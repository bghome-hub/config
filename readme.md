CREATE PROCEDURE dbo.usp_IC_ReportJobReconciliation
(
    @JobId UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StepName VARCHAR(50) = 'RECON_INIT';
    DECLARE @LogMsg VARCHAR(MAX);

    SET @LogMsg = 'Running job reconciliation report for JobId: ' + CAST(@JobId AS VARCHAR(36));
    INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

    -- 1. Create a container to hold the cross-database results
    CREATE TABLE #JobRecon (
        Company VARCHAR(20),
        FileSection NVARCHAR(300),
        HeaderStatus VARCHAR(20),
        BatchId CHAR(15),
        JournalEntry INT,
        Account CHAR(129),
        IntDebit NUMERIC(19,5),
        IntCredit NUMERIC(19,5),
        GpState VARCHAR(20),
        GpDebit NUMERIC(19,5),
        GpCredit NUMERIC(19,5)
    );

    BEGIN TRY
        DECLARE @DynamicCo VARCHAR(20);
        DECLARE @DynSql NVARCHAR(MAX);

        SET @StepName = 'OPEN_CURSOR';
        -- Only loop through companies that actually exist in this specific Job
        DECLARE CoCursor CURSOR LOCAL FAST_FORWARD FOR 
            SELECT DISTINCT Company FROM dbo.IC_CompanyHeader 
            WHERE JobId = @JobId;

        OPEN CoCursor;
        FETCH NEXT FROM CoCursor INTO @DynamicCo;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            SET @StepName = 'EXEC_RECON_QUERY';
            
            -- We aggregate both sides (Int vs GP) by Journal Entry and Account to safely catch manual modifications
            SET @DynSql = N'
            USE ' + QUOTENAME(@DynamicCo) + N';

            WITH IntLines AS (
                SELECT 
                    h.Company, h.FileSection, h.HeaderStatus, h.BatchId, h.JournalEntry, l.Account,
                    SUM(l.DebitAmount) AS IntDebit, SUM(l.CreditAmount) AS IntCredit
                FROM DYNAMICS.dbo.IC_CompanyHeader h
                INNER JOIN DYNAMICS.dbo.IC_DetailLine l ON h.HeaderId = l.HeaderId
                WHERE h.JobId = @pJobId AND h.Company = @pCompany
                GROUP BY h.Company, h.FileSection, h.HeaderStatus, h.BatchId, h.JournalEntry, l.Account
            ),
            GpLines AS (
                SELECT 
                    JRNENTRY, ACTINDX, MAX(GpState) AS GpState, 
                    SUM(DEBITAMT) AS GpDebit, SUM(CRDTAMNT) AS GpCredit
                FROM (
                    SELECT JRNENTRY, ACTINDX, ''1_UNPOSTED'' AS GpState, DEBITAMT, CRDTAMNT FROM dbo.GL10001 
                    WHERE JRNENTRY IN (SELECT JournalEntry FROM DYNAMICS.dbo.IC_CompanyHeader WHERE JobId = @pJobId AND Company = @pCompany AND JournalEntry IS NOT NULL)
                    UNION ALL
                    SELECT JRNENTRY, ACTINDX, ''2_POSTED'' AS GpState, DEBITAMT, CRDTAMNT FROM dbo.GL20000 
                    WHERE JRNENTRY IN (SELECT JournalEntry FROM DYNAMICS.dbo.IC_CompanyHeader WHERE JobId = @pJobId AND Company = @pCompany AND JournalEntry IS NOT NULL)
                    UNION ALL
                    SELECT JRNENTRY, ACTINDX, ''3_HISTORICAL'' AS GpState, DEBITAMT, CRDTAMNT FROM dbo.GL30000 
                    WHERE JRNENTRY IN (SELECT JournalEntry FROM DYNAMICS.dbo.IC_CompanyHeader WHERE JobId = @pJobId AND Company = @pCompany AND JournalEntry IS NOT NULL)
                ) u
                GROUP BY JRNENTRY, ACTINDX
            )
            INSERT INTO #JobRecon (Company, FileSection, HeaderStatus, BatchId, JournalEntry, Account, IntDebit, IntCredit, GpState, GpDebit, GpCredit)
            SELECT 
                i.Company, i.FileSection, i.HeaderStatus, i.BatchId, i.JournalEntry, i.Account,
                i.IntDebit, i.IntCredit,
                CASE 
                    WHEN i.HeaderStatus IN (''FAILED'', ''PENDING'', ''SKIPPED'') THEN ''NOT SUBMITTED''
                    WHEN g.GpState IS NULL THEN ''DELETED FROM GP''
                    ELSE SUBSTRING(g.GpState, 3, 20) -- Strip the sorting prefix
                END AS GpState,
                COALESCE(g.GpDebit, 0), 
                COALESCE(g.GpCredit, 0)
            FROM IntLines i
            LEFT JOIN dbo.GL00105 a ON i.Account = a.ACTNUMST
            LEFT JOIN GpLines g ON i.JournalEntry = g.JRNENTRY AND a.ACTINDX = g.ACTINDX;
            ';

            EXEC sp_executesql @DynSql, 
                N'@pJobId UNIQUEIDENTIFIER, @pCompany VARCHAR(20)', 
                @pJobId = @JobId, @pCompany = @DynamicCo;

            FETCH NEXT FROM CoCursor INTO @DynamicCo;
        END

        CLOSE CoCursor;
        DEALLOCATE CoCursor;

        SET @StepName = 'OUTPUT_REPORT';

        -- Output the final report with calculated variances for the accountants
        SELECT 
            Company,
            FileSection,
            HeaderStatus,
            BatchId,
            JournalEntry,
            Account,
            IntDebit AS Uploaded_Debit,
            GpDebit  AS GP_Actual_Debit,
            (IntDebit - GpDebit) AS Debit_Variance,
            IntCredit AS Uploaded_Credit,
            GpCredit  AS GP_Actual_Credit,
            (IntCredit - GpCredit) AS Credit_Variance,
            GpState AS Current_GP_Status
        FROM #JobRecon
        ORDER BY Company, JournalEntry, Account;

        DROP TABLE #JobRecon;

    END TRY
    BEGIN CATCH
        DECLARE @ErrMessage VARCHAR(MAX) = ERROR_MESSAGE();
        
        SET @LogMsg = 'Reconciliation Report Exception: ' + @ErrMessage;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'ERROR', @StepName, @LogMsg);
        
        IF CURSOR_STATUS('local', 'CoCursor') >= -1
        BEGIN
            CLOSE CoCursor; DEALLOCATE CoCursor;
        END
        IF OBJECT_ID('tempdb..#JobRecon') IS NOT NULL DROP TABLE #JobRecon;
        
        THROW;
    END CATCH
END
GO
