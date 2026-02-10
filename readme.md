/* Staging table for the BULK INSERT process */
DROP TABLE IF EXISTS dbo.IC_Import_Staging;

CREATE TABLE dbo.IC_Import_Staging (
    CompanyID     VARCHAR(5),
    AccountNumber VARCHAR(75),
    Amount        DECIMAL(19, 5),
    TransDate     DATETIME,
    Description   VARCHAR(30)
);

/* Audit log for historical tracking */
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
        INSERT INTO dbo.IC_Import_HeaderLog (
            FileName, 
            Status, 
            TotalAmount, 
            RowCountAffected
        )
        VALUES (
            @FileName, 
            @Status, 
            @TotalDebitVolume, 
            @TotalLineCount
        );
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





CREATE PROCEDURE dbo.usp_ICImport_ValidateData
    @IsDataValid           BIT OUTPUT,
    @ValidationErrorMessage VARCHAR(255) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @IsDataValid = 1;

    /* 1. Verify staging data exists */
    IF NOT EXISTS (SELECT 1 FROM dbo.IC_Import_Staging)
    BEGIN
        SET @IsDataValid = 0;
        SET @ValidationErrorMessage = 'Validation Failed: Staging table is empty.';
        RETURN;
    END

    /* 2. Ensure exactly two companies are present */
    DECLARE @DistinctCompanyCount INT;
    SELECT @DistinctCompanyCount = COUNT(DISTINCT CompanyID) FROM dbo.IC_Import_Staging;

    IF @DistinctCompanyCount <> 2
    BEGIN
        SET @IsDataValid = 0;
        SET @ValidationErrorMessage = 'Validation Failed: File must contain exactly 2 companies.';
        RETURN;
    END

    /* 3. Check Intercompany Relationship (IC40100) */
    DECLARE @PrimaryCompanyID   VARCHAR(5);
    DECLARE @SecondaryCompanyID VARCHAR(5);

    SELECT 
        @PrimaryCompanyID   = MIN(CompanyID), 
        @SecondaryCompanyID = MAX(CompanyID) 
    FROM dbo.IC_Import_Staging;

    /* Verify relationship exists in either direction using LEFT(5) for safety */
    IF NOT EXISTS (
        SELECT 1 
        FROM DYNAMICS..IC40100 
        WHERE (ORCOMID = LEFT(@PrimaryCompanyID, 5) AND DSTCOMID = LEFT(@SecondaryCompanyID, 5))
           OR (ORCOMID = LEFT(@SecondaryCompanyID, 5) AND DSTCOMID = LEFT(@PrimaryCompanyID, 5))
    )
    BEGIN
        SET @IsDataValid = 0;
        SET @ValidationErrorMessage = 'Validation Failed: Relationship not found in DYNAMICS..IC40100.';
        RETURN;
    END

    /* 4. Check for zero balance */
    IF (SELECT SUM(Amount) FROM dbo.IC_Import_Staging) <> 0
    BEGIN
        SET @IsDataValid = 0;
        SET @ValidationErrorMessage = 'Validation Failed: Transactions do not balance to zero.';
        RETURN;
    END
END
GO









CREATE PROCEDURE dbo.usp_ProcessICImport
    @FileName VARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    /* Orchestration Variables */
    DECLARE @AuditLogID              INT;
    DECLARE @IsDataValid             BIT;
    DECLARE @ValidationErrorMessage  VARCHAR(255);
    DECLARE @NextJournalEntryNumber  INT;
    DECLARE @eConnectErrorState      INT = 0;
    DECLARE @eConnectErrorString     VARCHAR(255);

    /* Cursor Context Variables */
    DECLARE @CurrentRowCompany       VARCHAR(5);
    DECLARE @CurrentRowAccount       VARCHAR(75);
    DECLARE @CurrentRowAmount        DECIMAL(19, 5);
    DECLARE @CurrentRowDate          DATETIME;
    DECLARE @CurrentRowDescription   VARCHAR(30);
    DECLARE @CalculatedDebitAmount   DECIMAL(19, 5);
    DECLARE @CalculatedCreditAmount  DECIMAL(19, 5);

    /* Summary Variables */
    DECLARE @TotalDebitVolume        DECIMAL(19, 5);
    DECLARE @TotalLineCount          INT;
    DECLARE @PrimaryCompanyID        VARCHAR(5);
    DECLARE @SecondaryCompanyID      VARCHAR(5);

    BEGIN TRY
        /* Clean zero-amount records immediately */
        DELETE FROM dbo.IC_Import_Staging WHERE Amount = 0;

        /* Step 1: Validate Data */
        EXEC dbo.usp_ICImport_ValidateData 
            @IsDataValid           = @IsDataValid OUTPUT, 
            @ValidationErrorMessage = @ValidationErrorMessage OUTPUT;

        IF @IsDataValid = 0
        BEGIN
            EXEC dbo.usp_ICImport_UpdateLog 
                @AuditLogID             = @AuditLogID OUTPUT, 
                @FileName               = @FileName, 
                @Status                 = 'ERROR', 
                @ValidationErrorMessage = @ValidationErrorMessage;

            RAISERROR(@ValidationErrorMessage, 16, 1);
            RETURN;
        END

        /* Step 2: Initialize Audit Stats */
        SELECT @TotalDebitVolume = SUM(CASE WHEN Amount > 0 THEN Amount ELSE 0 END), @TotalLineCount = COUNT(*) FROM dbo.IC_Import_Staging;
        SELECT @PrimaryCompanyID = MIN(CompanyID), @SecondaryCompanyID = MAX(CompanyID) FROM dbo.IC_Import_Staging;

        /* Step 3: Log Processing Start */
        EXEC dbo.usp_ICImport_UpdateLog 
            @AuditLogID       = @AuditLogID OUTPUT, 
            @FileName         = @FileName, 
            @Status           = 'PROCESSING', 
            @TotalDebitVolume = @TotalDebitVolume, 
            @TotalLineCount   = @TotalLineCount;
        
        UPDATE dbo.IC_Import_HeaderLog SET CompanyA = @PrimaryCompanyID, CompanyB = @SecondaryCompanyID WHERE LogID = @AuditLogID;

        /* Step 4: Get Next JE Number */
        EXEC dbo.taGetNextJournalEntry 
            @I_vBatchNumber        = 'IC_IMPORT',
            @O_vJournalEntryNumber = @NextJournalEntryNumber OUTPUT,
            @O_iErrorState         = @eConnectErrorState OUTPUT;

        IF @eConnectErrorState <> 0 OR @NextJournalEntryNumber IS NULL
        BEGIN
            RAISERROR('eConnect Error: JE Number Retrieval Failed.', 16, 1);
        END

        /* Step 5: Process Lines (Transaction Scope) */
        BEGIN TRANSACTION;

        DECLARE cur_TransactionLines CURSOR LOCAL FAST_FORWARD FOR 
        SELECT CompanyID, AccountNumber, Amount, TransDate, Description FROM dbo.IC_Import_Staging;

        OPEN cur_TransactionLines;
        FETCH NEXT FROM cur_TransactionLines INTO @CurrentRowCompany, @CurrentRowAccount, @CurrentRowAmount, @CurrentRowDate, @CurrentRowDescription;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            SET @CalculatedDebitAmount  = CASE WHEN @CurrentRowAmount > 0 THEN @CurrentRowAmount ELSE 0 END;
            SET @CalculatedCreditAmount = CASE WHEN @CurrentRowAmount < 0 THEN ABS(@CurrentRowAmount) ELSE 0 END;

            /* VERIFIED PARAMETERS FOR taGLTransactionLineInsert */
            EXEC dbo.taGLTransactionLineInsert 
                @I_vBACHNUMB       = 'IC_IMPORT', 
                @I_vJRNENTRY       = @NextJournalEntryNumber, 
                @I_vACTNUMST       = @CurrentRowAccount,
                @I_vDEBITAMT       = @CalculatedDebitAmount, 
                @I_vCRDTAMNT       = @CalculatedCreditAmount, /* Note: CRDTAMNT (with N) */
                @I_vTRXDATE        = @CurrentRowDate, 
                @I_vDSCRIPTN       = @CurrentRowDescription,  /* Note: DSCRIPTN (not REFERENCE) */
                @I_vINTERID        = @CurrentRowCompany, 
                @I_vICMSOPROCEDURE = 1,
                @O_iErrorState     = @eConnectErrorState OUTPUT, 
                @oErrString        = @eConnectErrorString OUTPUT;

            IF @eConnectErrorState <> 0 RAISERROR(@eConnectErrorString, 16, 1);

            FETCH NEXT FROM cur_TransactionLines INTO @CurrentRowCompany, @CurrentRowAccount, @CurrentRowAmount, @CurrentRowDate, @CurrentRowDescription;
        END

        CLOSE cur_TransactionLines; 
        DEALLOCATE cur_TransactionLines;

        /* Step 6: Finalize Header */
        EXEC dbo.taGLTransactionHeaderInsert
            @I_vBACHNUMB   = 'IC_IMPORT', 
            @I_vJRNENTRY   = @NextJournalEntryNumber, 
            @I_vREFRENCE   = 'Intercompany Import', /* Note: REFRENCE (Header only) */
            @I_vTRXDATE    = @CurrentRowDate, 
            @I_vTRXTYPE    = 2, 
            @I_vSOURCDOC   = 'GJ',
            @O_iErrorState = @eConnectErrorState OUTPUT, 
            @oErrString    = @eConnectErrorString OUTPUT;

        IF @eConnectErrorState <> 0 RAISERROR(@eConnectErrorString, 16, 1);

        COMMIT TRANSACTION;

        /* Step 7: Final Success Logging */
        EXEC dbo.usp_ICImport_UpdateLog 
            @AuditLogID             = @AuditLogID, 
            @FileName               = @FileName, 
            @Status                 = 'SUCCESS', 
            @NextJournalEntryNumber = @NextJournalEntryNumber;

        TRUNCATE TABLE dbo.IC_Import_Staging;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;

        IF CURSOR_STATUS('local', 'cur_TransactionLines') >= 0 
        BEGIN 
            CLOSE cur_TransactionLines; 
            DEALLOCATE cur_TransactionLines; 
        END
        
        SET @eConnectErrorString = ERROR_MESSAGE();
        
        IF @AuditLogID IS NOT NULL
        BEGIN
            EXEC dbo.usp_ICImport_UpdateLog 
                @AuditLogID             = @AuditLogID, 
                @FileName               = @FileName, 
                @Status                 = 'ERROR', 
                @ValidationErrorMessage = @eConnectErrorString;
        END
        RAISERROR(@eConnectErrorString, 16, 1);
    END CATCH
END
GO
