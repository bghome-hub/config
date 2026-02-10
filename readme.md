1. The Audit and Staging Tables
We need these to exist before the procedures can compile.

SQL
-- Staging table for the Bulk Insert
CREATE TABLE dbo.IC_Import_Staging (
    CompanyID     VARCHAR(5),
    AccountNumber VARCHAR(75),
    Amount        DECIMAL(19, 5),
    TransDate     DATETIME,
    Description   VARCHAR(30)
);

-- Audit log to track every file and Journal Entry
CREATE TABLE dbo.IC_Import_HeaderLog (
    LogID            INT IDENTITY(1, 1) PRIMARY KEY,
    FileName         VARCHAR(255),
    ImportDate       DATETIME DEFAULT GETDATE(),
    JournalEntry     INT,
    TotalAmount      DECIMAL(19, 5),
    RowCountAffected INT,
    CompanyA         VARCHAR(5),
    CompanyB         VARCHAR(5),
    Status           VARCHAR(20), -- 'PROCESSING', 'SUCCESS', 'ERROR'
    ErrorMessage     VARCHAR(MAX)
);
2. The Logging Helper
This handles all entries into the audit table. It uses full, descriptive variable names.

SQL
CREATE PROCEDURE dbo.usp_ICImport_UpdateLog
    @LogID                INT OUTPUT,
    @FileName             VARCHAR(255),
    @Status               VARCHAR(20),
    @JournalEntryNumber   INT           = NULL,
    @ErrorMessageText     VARCHAR(MAX)  = NULL,
    @TotalTransactionSum  DECIMAL(19,5) = 0,
    @TotalRowCount        INT           = 0
AS
BEGIN
    SET NOCOUNT ON;

    -- Create a new log entry if no LogID is provided
    IF @LogID IS NULL
    BEGIN
        INSERT INTO dbo.IC_Import_HeaderLog (
            FileName, 
            Status, 
            TotalAmount, 
            RowCountAffected
        )
        VALUES (
            @FileName, 
            @Status, 
            @TotalTransactionSum, 
            @TotalRowCount
        );
        
        SET @LogID = SCOPE_IDENTITY();
    END
    -- Update existing entry for Success or Failure
    ELSE
    BEGIN
        UPDATE dbo.IC_Import_HeaderLog
        SET Status       = @Status,
            JournalEntry = ISNULL(@JournalEntryNumber, JournalEntry),
            ErrorMessage = ISNULL(@ErrorMessageText, ErrorMessage)
        WHERE LogID = @LogID;
    END
END
GO
3. The Data Validator
This ensures the data is "sane" before we ever attempt to talk to the Dynamics GP logic.

SQL
CREATE PROCEDURE dbo.usp_ICImport_ValidateData
    @IsDataValid  BIT OUTPUT,
    @OutputError  VARCHAR(255) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @IsDataValid = 1;

    -- 1. Check for data existence
    IF NOT EXISTS (SELECT 1 FROM dbo.IC_Import_Staging)
    BEGIN
        SET @IsDataValid = 0;
        SET @OutputError = 'Validation Failed: No data found in staging table.';
        RETURN;
    END

    -- 2. Check for balance (Debits must equal Credits)
    IF (SELECT SUM(Amount) FROM dbo.IC_Import_Staging) <> 0
    BEGIN
        SET @IsDataValid = 0;
        SET @OutputError = 'Validation Failed: Transactions do not balance to zero.';
        RETURN;
    END

    -- 3. Check for company count (Requirement: exactly 2)
    DECLARE @DistinctCompanyCount INT;
    SELECT @DistinctCompanyCount = COUNT(DISTINCT CompanyID) FROM dbo.IC_Import_Staging;

    IF @DistinctCompanyCount > 2
    BEGIN
        SET @IsDataValid = 0;
        SET @OutputError = 'Validation Failed: Found ' + CAST(@DistinctCompanyCount AS VARCHAR) + ' companies. Limit is 2.';
        RETURN;
    END
END
GO
4. The Main Orchestrator
This brings it all together with a TRY...CATCH block and explicit transaction management.

SQL
CREATE PROCEDURE dbo.usp_ProcessICImport
    @FileName VARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    -- Logging and Status Variables
    DECLARE @AuditLogID              INT;
    DECLARE @DataIsValid             BIT;
    DECLARE @ValidationErrorText     VARCHAR(255);
    DECLARE @NextJournalEntryNumber  INT;
    DECLARE @eConnectErrorState      INT = 0;
    DECLARE @eConnectErrorString     VARCHAR(255);

    -- Cursor/Line Processing Variables
    DECLARE @CurrentLineCompany      VARCHAR(5);
    DECLARE @CurrentLineAccount      VARCHAR(75);
    DECLARE @CurrentLineAmount       DECIMAL(19, 5);
    DECLARE @CurrentLineDate         DATETIME;
    DECLARE @CurrentLineDescription  VARCHAR(30);
    DECLARE @CalculatedLineDebit     DECIMAL(19, 5);
    DECLARE @CalculatedLineCredit    DECIMAL(19, 5);

    -- Summary Variables
    DECLARE @TotalImportVolume       DECIMAL(19, 5);
    DECLARE @TotalImportRowCount     INT;
    DECLARE @CompanyID_A             VARCHAR(5);
    DECLARE @CompanyID_B             VARCHAR(5);

    BEGIN TRY
        -- -----------------------------------------------------
        -- 1. DATA PREP: Remove 0.00 entries and Validate
        -- -----------------------------------------------------
        DELETE FROM dbo.IC_Import_Staging WHERE Amount = 0;

        EXEC dbo.usp_ICImport_ValidateData 
            @IsDataValid = @DataIsValid OUTPUT, 
            @OutputError = @ValidationErrorText OUTPUT;

        IF @DataIsValid = 0
        BEGIN
            EXEC dbo.usp_ICImport_UpdateLog 
                @LogID               = @AuditLogID OUTPUT, 
                @FileName            = @FileName, 
                @Status              = 'ERROR', 
                @ErrorMessageText    = @ValidationErrorText;

            RAISERROR(@ValidationErrorText, 16, 1);
            RETURN;
        END

        -- -----------------------------------------------------
        -- 2. INITIALIZE LOGGING & RESERVE GP JOURNAL ENTRY
        -- -----------------------------------------------------
        SELECT 
            @TotalImportVolume   = SUM(CASE WHEN Amount > 0 THEN Amount ELSE 0 END), 
            @TotalImportRowCount = COUNT(*) 
        FROM dbo.IC_Import_Staging;

        SELECT 
            @CompanyID_A = MIN(CompanyID), 
            @CompanyID_B = MAX(CompanyID) 
        FROM dbo.IC_Import_Staging;

        EXEC dbo.usp_ICImport_UpdateLog 
            @LogID               = @AuditLogID OUTPUT, 
            @FileName            = @FileName, 
            @Status              = 'PROCESSING', 
            @TotalTransactionSum = @TotalImportVolume, 
            @TotalRowCount       = @TotalImportRowCount;
        
        UPDATE dbo.IC_Import_HeaderLog 
        SET CompanyA = @CompanyID_A, 
            CompanyB = @CompanyID_B 
        WHERE LogID = @AuditLogID;

        EXEC DYNAMICS..smGetNextNumber 
            @I_sNumberType = 2, 
            @O_vNumber     = @NextJournalEntryNumber OUTPUT;

        -- -----------------------------------------------------
        -- 3. BEGIN TRANSACTION & PROCESS LINES
        -- -----------------------------------------------------
        BEGIN TRANSACTION;

        DECLARE cur_GL_Lines CURSOR LOCAL FAST_FORWARD FOR 
        SELECT CompanyID, AccountNumber, Amount, TransDate, Description 
        FROM dbo.IC_Import_Staging;
        
        OPEN cur_GL_Lines;
        FETCH NEXT FROM cur_GL_Lines INTO 
            @CurrentLineCompany, @CurrentLineAccount, @CurrentLineAmount, @CurrentLineDate, @CurrentLineDescription;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- Explicit Debit/Credit assignment
            IF @CurrentLineAmount > 0
            BEGIN
                SET @CalculatedLineDebit  = @CurrentLineAmount;
                SET @CalculatedLineCredit = 0;
            END
            ELSE
            BEGIN
                SET @CalculatedLineDebit  = 0;
                SET @CalculatedLineCredit = ABS(@CurrentLineAmount);
            END

            EXEC dbo.taGLTransactionLineInsert 
                @I_vBACHNUMB       = 'IC_IMPORT', 
                @I_vJRNENTRY       = @NextJournalEntryNumber, 
                @I_vACTNUMST       = @CurrentLineAccount,
                @I_vDEBITAMT       = @CalculatedLineDebit,
                @I_vCRDTAMT        = @CalculatedLineCredit,
                @I_vTRXDATE        = @CurrentLineDate, 
                @I_vREFRENCE       = @CurrentLineDescription, 
                @I_vINTERID        = @CurrentLineCompany, 
                @I_vICMSOPROCEDURE = 1,
                @O_iErrorState     = @eConnectErrorState OUTPUT, 
                @oErrString        = @eConnectErrorString OUTPUT;

            IF @eConnectErrorState <> 0
            BEGIN
                RAISERROR(@eConnectErrorString, 16, 1);
            END

            FETCH NEXT FROM cur_GL_Lines INTO 
                @CurrentLineCompany, @CurrentLineAccount, @CurrentLineAmount, @CurrentLineDate, @CurrentLineDescription;
        END

        CLOSE cur_GL_Lines; 
        DEALLOCATE cur_GL_Lines;

        -- -----------------------------------------------------
        -- 4. FINALIZE HEADER & COMMIT
        -- -----------------------------------------------------
        EXEC dbo.taGLTransactionHeaderInsert
            @I_vBACHNUMB   = 'IC_IMPORT', 
            @I_vJRNENTRY   = @NextJournalEntryNumber, 
            @I_vREFRENCE   = 'Intercompany Import',
            @I_vTRXDATE    = @CurrentLineDate, 
            @I_vTRXTYPE    = 2, 
            @I_vSOURCDOC   = 'GJ',
            @O_iErrorState = @eConnectErrorState OUTPUT, 
            @oErrString    = @eConnectErrorString OUTPUT;

        IF @eConnectErrorState <> 0
        BEGIN
            RAISERROR(@eConnectErrorString, 16, 1);
        END

        COMMIT TRANSACTION;

        -- -----------------------------------------------------
        -- 5. POST-SUCCESS LOGGING
        -- -----------------------------------------------------
        EXEC dbo.usp_ICImport_UpdateLog 
            @LogID              = @AuditLogID, 
            @FileName           = @FileName, 
            @Status             = 'SUCCESS', 
            @JournalEntryNumber = @NextJournalEntryNumber;

        TRUNCATE TABLE dbo.IC_Import_Staging;

    END TRY
    BEGIN CATCH
        -- Ensure transaction is rolled back first
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        -- Ensure cursor is released
        IF CURSOR_STATUS('local', 'cur_GL_Lines') >= 0
        BEGIN
            CLOSE cur_GL_Lines;
            DEALLOCATE cur_GL_Lines;
        END

        DECLARE @SystemErrorMessage VARCHAR(MAX) = ERROR_MESSAGE();

        -- Log failure if the AuditLog record was successfully created
        IF @AuditLogID IS NOT NULL
        BEGIN
            EXEC dbo.usp_ICImport_UpdateLog 
                @LogID              = @AuditLogID, 
                @FileName           = @FileName, 
                @Status             = 'ERROR', 
                @ErrorMessageText   = @SystemErrorMessage;
        END

        -- Propagate error to calling process (SQL Agent)
        RAISERROR(@SystemErrorMessage, 16, 1);
    END CATCH
END
GO
