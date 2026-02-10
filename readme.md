/* 1. TABLE DEFINITIONS */
DROP TABLE IF EXISTS dbo.IC_Import_Staging;
CREATE TABLE dbo.IC_Import_Staging (
    CompanyID     VARCHAR(5),
    AccountNumber VARCHAR(75),
    Amount        DECIMAL(19, 5),
    TransDate     DATETIME,
    Description   VARCHAR(30)
);

DROP TABLE IF EXISTS dbo.IC_Import_HeaderLog;
CREATE TABLE dbo.IC_Import_HeaderLog (
    LogID            INT IDENTITY(1, 1) PRIMARY KEY,
    FileName         VARCHAR(255),
    ImportDate       DATETIME DEFAULT GETDATE(),
    JournalEntry     INT,
    TotalAmount      DECIMAL(19, 5),
    RowCountAffected INT,
    CompanyA         VARCHAR(5),
    CompanyB         VARCHAR(5),
    Status           VARCHAR(20), /* 'PROCESSING', 'SUCCESS', 'ERROR' */
    ErrorMessage     VARCHAR(MAX)
);
GO

/* 2. LOGGING HELPER */
CREATE PROCEDURE dbo.usp_ICImport_UpdateLog
    @AuditLogID              INT OUTPUT,
    @FileName                VARCHAR(255),
    @Status                  VARCHAR(20),
    @NextJournalEntryNumber  INT           = NULL,
    @ValidationErrorMessage  VARCHAR(MAX)  = NULL,
    @TotalDebitVolume        DECIMAL(19,5) = 0,
    @TotalLineCount          INT           = 0
AS
BEGIN
    SET NOCOUNT ON;
    IF @AuditLogID IS NULL
    BEGIN
        INSERT INTO dbo.IC_Import_HeaderLog (FileName, Status, TotalAmount, RowCountAffected)
        VALUES (@FileName, @Status, @TotalDebitVolume, @TotalLineCount);
        SET @AuditLogID = SCOPE_IDENTITY();
    END
    ELSE
    BEGIN
        UPDATE dbo.IC_Import_HeaderLog
        SET Status       = @Status,
            JournalEntry = ISNULL(@NextJournalEntryNumber, JournalEntry),
            ErrorMessage = ISNULL(@ValidationErrorMessage, ErrorMessage)
        WHERE LogID = @AuditLogID;
    END
END
GO

/* 3. VALIDATION (IC40100) */
CREATE PROCEDURE dbo.usp_ICImport_ValidateData
    @IsDataValid           BIT OUTPUT,
    @ValidationErrorMessage VARCHAR(255) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @IsDataValid = 1;

    IF NOT EXISTS (SELECT 1 FROM dbo.IC_Import_Staging)
    BEGIN
        SET @IsDataValid = 0; SET @ValidationErrorMessage = 'Validation Failed: Staging table is empty.';
        RETURN;
    END

    DECLARE @DistinctCompanyCount INT;
    SELECT @DistinctCompanyCount = COUNT(DISTINCT CompanyID) FROM dbo.IC_Import_Staging;

    IF @DistinctCompanyCount <> 2
    BEGIN
        SET @IsDataValid = 0; SET @ValidationErrorMessage = 'Validation Failed: File must contain exactly 2 companies.';
        RETURN;
    END

    DECLARE @PrimaryCompanyID VARCHAR(5), @SecondaryCompanyID VARCHAR(5);
    SELECT @PrimaryCompanyID = MIN(CompanyID), @SecondaryCompanyID = MAX(CompanyID) FROM dbo.IC_Import_Staging;

    IF NOT EXISTS (
        SELECT 1 FROM DYNAMICS..IC40100 
        WHERE (ORCOMID = LEFT(@PrimaryCompanyID, 5) AND DSTCOMID = LEFT(@SecondaryCompanyID, 5))
           OR (ORCOMID = LEFT(@SecondaryCompanyID, 5) AND DSTCOMID = LEFT(@PrimaryCompanyID, 5))
    )
    BEGIN
        SET @IsDataValid = 0; SET @ValidationErrorMessage = 'Validation Failed: IC Link missing in DYNAMICS..IC40100.';
        RETURN;
    END

    IF (SELECT SUM(Amount) FROM dbo.IC_Import_Staging) <> 0
    BEGIN
        SET @IsDataValid = 0; SET @ValidationErrorMessage = 'Validation Failed: Transactions do not balance.';
        RETURN;
    END
END
GO

/* 4. MAIN ORCHESTRATOR */
CREATE PROCEDURE dbo.usp_ProcessICImport
    @FileName VARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @AuditLogID              INT;
    DECLARE @IsDataValid             BIT;
    DECLARE @ValidationErrorMessage  VARCHAR(255);
    DECLARE @NextJournalEntryNumber  INT;
    DECLARE @eConnectErrorState      INT = 0;
    DECLARE @eConnectErrorString     VARCHAR(255);

    DECLARE @CurrentRowCompany       VARCHAR(5);
    DECLARE @CurrentRowAccount       VARCHAR(75);
    DECLARE @CurrentRowAmount        DECIMAL(19, 5);
    DECLARE @CurrentRowDate          DATETIME;
    DECLARE @CurrentRowDescription   VARCHAR(30);
    DECLARE @CalculatedDebitAmount   DECIMAL(19, 5);
    DECLARE @CalculatedCreditAmount  DECIMAL(19, 5);

    DECLARE @TotalDebitVolume        DECIMAL(19, 5);
    DECLARE @TotalLineCount          INT;
    DECLARE @PrimaryCompanyID        VARCHAR(5);
    DECLARE @SecondaryCompanyID      VARCHAR(5);

    BEGIN TRY
        DELETE FROM dbo.IC_Import_Staging WHERE Amount = 0;

        EXEC dbo.usp_ICImport_ValidateData @IsDataValid = @IsDataValid OUTPUT, @ValidationErrorMessage = @ValidationErrorMessage OUTPUT;

        IF @IsDataValid = 0
        BEGIN
            EXEC dbo.usp_ICImport_UpdateLog @AuditLogID = @AuditLogID OUTPUT, @FileName = @FileName, @Status = 'ERROR', @ValidationErrorMessage = @ValidationErrorMessage;
            RAISERROR(@ValidationErrorMessage, 16, 1); RETURN;
        END

        SELECT @TotalDebitVolume = SUM(CASE WHEN Amount > 0 THEN Amount ELSE 0 END), @TotalLineCount = COUNT(*) FROM dbo.IC_Import_Staging;
        SELECT @PrimaryCompanyID = MIN(CompanyID), @SecondaryCompanyID = MAX(CompanyID) FROM dbo.IC_Import_Staging;

        EXEC dbo.usp_ICImport_UpdateLog @AuditLogID = @AuditLogID OUTPUT, @FileName = @FileName, @Status = 'PROCESSING', @TotalDebitVolume = @TotalDebitVolume, @TotalLineCount = @TotalLineCount;
        UPDATE dbo.IC_Import_HeaderLog SET CompanyA = @PrimaryCompanyID, CompanyB = @SecondaryCompanyID WHERE LogID = @AuditLogID;

        EXEC dbo.taGetNextJournalEntry @I_vBatchNumber = 'IC_IMPORT', @O_vJournalEntryNumber = @NextJournalEntryNumber OUTPUT, @O_iErrorState = @eConnectErrorState OUTPUT;

        IF @eConnectErrorState <> 0 OR @NextJournalEntryNumber IS NULL RAISERROR('eConnect Error: JE Number Retrieval Failed.', 16, 1);

        BEGIN TRANSACTION;

        DECLARE cur_TransactionLines CURSOR LOCAL FAST_FORWARD FOR SELECT CompanyID, AccountNumber, Amount, TransDate, Description FROM dbo.IC_Import_Staging;
        OPEN cur_TransactionLines;
        FETCH NEXT FROM cur_TransactionLines INTO @CurrentRowCompany, @CurrentRowAccount, @CurrentRowAmount, @CurrentRowDate, @CurrentRowDescription;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            SET @CalculatedDebitAmount = CASE WHEN @CurrentRowAmount > 0 THEN @CurrentRowAmount ELSE 0 END;
            SET @CalculatedCreditAmount = CASE WHEN @CurrentRowAmount < 0 THEN ABS(@CurrentRowAmount) ELSE 0 END;

            /* -------------------------------------------------------------------------
               CORRECTED PARAMETERS:
               1. @I_vDEBITAMT (No N)
               2. @I_vCRDTAMNT (With N)
               3. @I_vDSCRIPTN (Description)
               4. REMOVED: @I_vINTERID, @I_vTRXDATE, @I_vICMSOPROCEDURE (Invalid)
               ------------------------------------------------------------------------- */
            EXEC dbo.taGLTransactionLineInsert 
                @I_vBACHNUMB       = 'IC_IMPORT', 
                @I_vJRNENTRY       = @NextJournalEntryNumber, 
                @I_vACTNUMST       = @CurrentRowAccount,
                @I_vDEBITAMT       = @CalculatedDebitAmount, 
                @I_vCRDTAMNT       = @CalculatedCreditAmount,
                @I_vDSCRIPTN       = @CurrentRowDescription,
                @O_iErrorState     = @eConnectErrorState OUTPUT, 
                @oErrString        = @eConnectErrorString OUTPUT;

            IF @eConnectErrorState <> 0 RAISERROR(@eConnectErrorString, 16, 1);
            FETCH NEXT FROM cur_TransactionLines INTO @CurrentRowCompany, @CurrentRowAccount, @CurrentRowAmount, @CurrentRowDate, @CurrentRowDescription;
        END

        CLOSE cur_TransactionLines; DEALLOCATE cur_TransactionLines;

        EXEC dbo.taGLTransactionHeaderInsert
            @I_vBACHNUMB = 'IC_IMPORT', @I_vJRNENTRY = @NextJournalEntryNumber, @I_vREFRENCE = 'IC Import',
            @I_vTRXDATE = @CurrentRowDate, @I_vTRXTYPE = 0, @I_vSOURCDOC = 'GJ', /* Note: TRXTYPE 0 = Standard */
            @O_iErrorState = @eConnectErrorState OUTPUT, @oErrString = @eConnectErrorString OUTPUT;

        IF @eConnectErrorState <> 0 RAISERROR(@eConnectErrorString, 16, 1);
        COMMIT TRANSACTION;

        EXEC dbo.usp_ICImport_UpdateLog @AuditLogID = @AuditLogID, @FileName = @FileName, @Status = 'SUCCESS', @NextJournalEntryNumber = @NextJournalEntryNumber;
        TRUNCATE TABLE dbo.IC_Import_Staging;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        IF CURSOR_STATUS('local', 'cur_TransactionLines') >= 0 BEGIN CLOSE cur_TransactionLines; DEALLOCATE cur_TransactionLines; END
        SET @eConnectErrorString = ERROR_MESSAGE();
        IF @AuditLogID IS NOT NULL EXEC dbo.usp_ICImport_UpdateLog @AuditLogID = @AuditLogID, @FileName = @FileName, @Status = 'ERROR', @ValidationErrorMessage = @eConnectErrorString;
        RAISERROR(@eConnectErrorString, 16, 1);
    END CATCH
END
GO


