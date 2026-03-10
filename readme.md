CREATE PROCEDURE dbo.usp_IC_LoadStaging
(
    @JobId UNIQUEIDENTIFIER,
    @CsvPath NVARCHAR(4000)
)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @StepName VARCHAR(50) = 'LOAD_INIT';
    DECLARE @LogMsg VARCHAR(MAX);

    SET @LogMsg = 'Creating temporary staging table for BULK INSERT.';
    INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

    -- 1. Concurrency-Safe Temp Table matching exactly the columns we expect
    CREATE TABLE #RawStaging (
        Company VARCHAR(MAX), 
        Account VARCHAR(MAX),
        AccountDesc VARCHAR(MAX),
        Amount VARCHAR(MAX), 
        TrxDate VARCHAR(MAX), 
        Reference VARCHAR(MAX), 
        Reversal VARCHAR(MAX)
    );

    BEGIN TRY
        -- 2. Execute Bulk Insert
        SET @StepName = 'BULK_INSERT';
        SET @LogMsg = 'Executing BULK INSERT with FIRSTROW = 1 to capture headers.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

            DECLARE @sql nvarchar(MAX) = N'BULK INSERT #RawStaging
            FROM ' + QUOTENAME(@CsvPath, '''') + N'
            WITH (
                FIRSTROW = 1, -- We MUST load row 1 to validate headers
                FIELDTERMINATOR = '','',
                ROWTERMINATOR = ''0x0d0a'',
                FIELDQUOTE = ''"''
            );';

        EXEC(@Sql);

        -- 3. Validate Headers 
        SET @StepName = 'VALIDATE_HEADERS';
        SET @LogMsg = 'Interrogating Row 1 for expected column schema.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        
        DECLARE @ExpectedHeaders VARCHAR(500)= 'Company,Account,AccountDesc,Amount,TrxDate,Reference,Reversal'
        DECLARE @ActualHeaders VARCHAR(MAX)

		/* Get the headers and compare */
		SET @ActualHeaders = (SELECT TOP 1 CONCAT(Company,',',Account,',',AccountDesc,',',Amount,',',TrxDate,',',Reference,',',Reversal) FROM #RawStaging)
        INSERT INTO dbo.IC_SystemLog (JobID, Loglevel, StepName, LogMessage) Values (@JobId, 'INFO', @StepName, @ActualHeaders)
        
		IF @ActualHeaders <> @ExpectedHeaders
		BEGIN
            -- Headers are missing, out of order, or malformed
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            
            SET @LogMsg = 'Invalid CSV Headers. Expected: Company,Account,AccountDesc,Amount,TrxDate,Reference,Reversal';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
            VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            
            DROP TABLE #RawStaging;
            RETURN;
        END

		SET @LogMsg = 'Completed Row interrogation.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);


        -- 4. Clean the Header Row out of the Temp Table
        SET @StepName = 'REMOVE_HEADERS';
		DELETE FROM #RawStaging WHERE CONCAT(Company,',',Account,',',AccountDesc,',',Amount,',',TrxDate,',',Reference,',',Reversal) = @ExpectedHeaders

        -- 5. Move to Persistent Staging
        SET @StepName = 'INSERT_STAGING';
        SET @LogMsg = 'Headers verified. Pushing raw text to persistent IC_StagingRaw table.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        INSERT INTO dbo.IC_StagingRaw (JobId, Company, Account, AccountDesc, Amount, TrxDate, Reference, Reversal)
        SELECT @JobId, Company, Account, AccountDesc, Amount, TrxDate, Reference, Reversal
        FROM #RawStaging;

        DECLARE @RowCount INT = @@ROWCOUNT;

        -- 6. Success
        UPDATE dbo.IC_ImportJob SET JobStatus = 'LOADED' WHERE JobId = @JobId;
        
        SET @LogMsg = 'Successfully staged ' + CAST(@RowCount AS VARCHAR(20)) + ' data rows.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        DROP TABLE IF exists #RawStaging;

    END TRY
    BEGIN CATCH
        DECLARE @BulkErr VARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
        
        SET @LogMsg = 'Bulk Load Exception: ' + @BulkErr;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
        
		DROP TABLE IF EXISTS #RawStaging
    END CATCH
END
GO
CREATE PROCEDURE dbo.usp_IC_PostEconnect
(
    @JobId UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StepName VARCHAR(50) = 'POST_INIT';
    DECLARE @LogMsg VARCHAR(MAX);

    DECLARE @HeaderId INT, @Company VARCHAR(20), @BatchId CHAR(15);
    DECLARE @TrxDate DATE, @RevDate DATE, @RevFlag CHAR(1);
    DECLARE @DynSql NVARCHAR(MAX);

    BEGIN TRY
        SET @LogMsg = 'Starting eConnect execution phase for JobId: ' + CAST(@JobId AS VARCHAR(36));
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        -- Open the cursor ONLY for pristine, VALIDATED batches
        SET @StepName = 'OPEN_CURSOR';
        DECLARE PostCursor CURSOR LOCAL FAST_FORWARD FOR
            SELECT HeaderId, Company, BatchId, TrxDate, ReversalDate, Reversal
            FROM dbo.IC_CompanyHeader
            WHERE JobId = @JobId AND HeaderStatus = 'VALIDATED';

        OPEN PostCursor;
        FETCH NEXT FROM PostCursor INTO @HeaderId, @Company, @BatchId, @TrxDate, @RevDate, @RevFlag;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            SET @StepName = 'EXEC_DYN_ECONNECT';
            SET @LogMsg = 'Attempting to post BatchId ' + LTRIM(RTRIM(@BatchId)) + ' to Company ' + LTRIM(RTRIM(@Company));
            INSERT INTO dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage) VALUES (@JobId, @HeaderId, 'INFO', @StepName, @LogMsg);

            -- Wrap the entire eConnect logic in dynamic SQL targeting the specific company DB

            SET @DynSql = N'
            USE ' + QUOTENAME(@Company) + N';
            
            DECLARE @Err INT = 0, @ErrStr VARCHAR(255) = '''';
            DECLARE @JE CHAR(13);
            DECLARE @Seq INT = 16384;
            DECLARE @DynStep VARCHAR(50) = ''GET_JE'';
            DECLARE @DynLog VARCHAR(MAX);
            
            BEGIN TRY
                -- 1. Get Next Journal Entry Number
                EXEC taGetNextJournalEntry @I_vInc_Dec = 1, @O_vJournalEntryNumber = @JE OUTPUT, @O_iErrorState = @Err OUTPUT;
                IF (@Err <> 0 OR @JE IS NULL) THROW 50001, ''taGetNextJournalEntry failed to return a valid Journal Entry.'', 1;

                -- 2. Insert Detail Lines via Cursor
                SET @DynStep = ''INSERT_LINES'';
                DECLARE @Acct CHAR(129), @Ref VARCHAR(30), @Deb NUMERIC(19,5), @Cred NUMERIC(19,5);
                
                DECLARE LineCursor CURSOR LOCAL FAST_FORWARD FOR
                    SELECT Account, Reference, DebitAmount, CreditAmount
                    FROM DYNAMICS.dbo.IC_DetailLine
                    WHERE HeaderId = @pHeaderId;
                    
                OPEN LineCursor;
                FETCH NEXT FROM LineCursor INTO @Acct, @Ref, @Deb, @Cred;
                
                WHILE @@FETCH_STATUS = 0
                BEGIN
                    SET @Err = 0; SET @ErrStr = '''';
                    EXEC taGLTransactionLineInsert
                        @I_vBACHNUMB = @pBatch, @I_vJRNENTRY = @JE, @I_vSQNCLINE = @Seq,
                        @I_vACTNUMST = @Acct, @I_vDEBITAMT = @Deb, @I_vCRDTAMNT = @Cred,
                        @I_vDOCDATE = @pTrxDate, @I_vDSCRIPTN = @Ref,
                        @O_iErrorState = @Err OUTPUT, @oErrString = @ErrStr OUTPUT;
                    
                    IF (@Err <> 0) THROW 50002, ''taGLTransactionLineInsert failed. Review eConnect Error State.'', 1;
                    
                    SET @Seq = @Seq + 16384;
                    FETCH NEXT FROM LineCursor INTO @Acct, @Ref, @Deb, @Cred;
                END
                CLOSE LineCursor; DEALLOCATE LineCursor;

                -- 3. Insert Header
                SET @DynStep = ''INSERT_HEADER'';
                SET @Err = 0; SET @ErrStr = '''';
                
                IF @pRevFlag = ''Y''
                BEGIN
                    EXEC taGLTransactionHeaderInsert
                        @I_vBACHNUMB = @pBatch, @I_vJRNENTRY = @JE, @I_vREFRENCE = ''Intracompany Upload'',
                        @I_vTRXDATE = @pTrxDate, @I_vTRXTYPE = 1, @I_vSOURCDOC = ''GJ'', @I_vRVRSNGDT = @pRevDate,
                        @O_iErrorState = @Err OUTPUT, @oErrString = @ErrStr OUTPUT;
                END
                ELSE
                BEGIN
                    EXEC taGLTransactionHeaderInsert
                        @I_vBACHNUMB = @pBatch, @I_vJRNENTRY = @JE, @I_vREFRENCE = ''Intracompany Upload'',
                        @I_vTRXDATE = @pTrxDate, @I_vTRXTYPE = 0, @I_vSOURCDOC = ''GJ'',
                        @O_iErrorState = @Err OUTPUT, @oErrString = @ErrStr OUTPUT;
                END
                
                IF (@Err <> 0) THROW 50003, ''taGLTransactionHeaderInsert failed. Review eConnect Error State.'', 1;

                -- 4. Success: Stamp the JE back to the Header table
                SET @DynStep = ''POST_SUCCESS'';
                UPDATE DYNAMICS.dbo.IC_CompanyHeader 
                SET JournalEntry = CAST(LTRIM(RTRIM(@JE)) AS INT), HeaderStatus = ''POSTED''
                WHERE HeaderId = @pHeaderId;

                SET @DynLog = ''Successfully posted Journal Entry '' + LTRIM(RTRIM(@JE)) + '' to batch '' + LTRIM(RTRIM(@pBatch));
                INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
                VALUES (@pJobId, @pHeaderId, ''INFO'', @DynStep, @DynLog);

            END TRY
            BEGIN CATCH
                DECLARE @GPError VARCHAR(MAX) = ERROR_MESSAGE();
                
                UPDATE DYNAMICS.dbo.IC_CompanyHeader 
                SET HeaderStatus = ''FAILED''
                WHERE HeaderId = @pHeaderId;

                SET @DynLog = ''eConnect Exception: '' + @GPError;
                INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
                VALUES (@pJobId, @pHeaderId, ''ERROR'', @DynStep, @DynLog);
                
                IF CURSOR_STATUS(''local'', ''LineCursor'') >= -1
                BEGIN
                    CLOSE LineCursor; DEALLOCATE LineCursor;
                END
            END CATCH
            ';

  
            EXEC sp_executesql @DynSql,
                N'@pJobId UNIQUEIDENTIFIER, @pHeaderId INT, @pBatch CHAR(15), @pTrxDate DATE, @pRevDate DATE, @pRevFlag CHAR(1)',
                @pJobId = @JobId, @pHeaderId = @HeaderId, @pBatch = @BatchId, @pTrxDate = @TrxDate, @pRevDate = @RevDate, @pRevFlag = @RevFlag;

            FETCH NEXT FROM PostCursor INTO @HeaderId, @Company, @BatchId, @TrxDate, @RevDate, @RevFlag;
        END

        CLOSE PostCursor; DEALLOCATE PostCursor;

        -- ==============================================================================
        -- PHASE 5: EVALUATE FINAL JOB OUTCOME
        -- ==============================================================================
        SET @StepName = 'POST_FINALIZE';
        
        IF NOT EXISTS (SELECT 1 FROM dbo.IC_CompanyHeader WHERE JobId = @JobId AND HeaderStatus IN ('FAILED', 'PENDING'))
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'POSTED' WHERE JobId = @JobId;
            SET @LogMsg = 'All batches successfully pushed to Dynamics GP.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        END
        ELSE
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'PARTIAL' WHERE JobId = @JobId;
            SET @LogMsg = 'Execution finished with mixed results. Check HeaderStatus for FAILED batches.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'WARN', @StepName, @LogMsg);
        END

    END TRY
    BEGIN CATCH
        DECLARE @ErrMessage VARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;

        SET @LogMsg = 'Post eConnect Exception: ' + @ErrMessage;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
        
        IF CURSOR_STATUS('local', 'PostCursor') >= -1
        BEGIN
            CLOSE PostCursor; DEALLOCATE PostCursor;
        END
    END CATCH
END
GO
CREATE PROCEDURE dbo.usp_IC_ProcessCsv
(
    @CsvPath NVARCHAR(4000),
    @JobId UNIQUEIDENTIFIER OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StepName VARCHAR(50) = 'INIT_JOB';
    DECLARE @LogMsg VARCHAR(MAX);
    DECLARE @FileName NVARCHAR(260) = RIGHT(@CsvPath, CHARINDEX('\', REVERSE(@CsvPath)) - 1);

    BEGIN TRY
        SET @JobId = NEWID();

        -- 1. Create the Master Job Record
        INSERT INTO dbo.IC_ImportJob (JobId, FileName, FilePath, JobStatus)
        VALUES (@JobId, @FileName, @CsvPath, 'LOADING');

        SET @LogMsg = 'Master pipeline initialized for file: ' + @FileName;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        -- 2. Execute Staging Load
        SET @StepName = 'EXEC_LOAD_STAGING';
        SET @LogMsg = 'Handing off to dbo.usp_IC_LoadStaging.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        
        EXEC dbo.usp_IC_LoadStaging @JobId = @JobId, @CsvPath = @CsvPath;

        IF EXISTS (SELECT 1 FROM dbo.IC_ImportJob WHERE JobId = @JobId AND JobStatus = 'FAILED')
        BEGIN
            SET @LogMsg = 'Pipeline aborted. dbo.usp_IC_LoadStaging reported a FATAL failure.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'ERROR', @StepName, @LogMsg);
            RETURN;
        END

        -- 3. Execute Validation Rules
        SET @StepName = 'EXEC_VALIDATE_RULES';
        SET @LogMsg = 'Staging successful. Handing off to dbo.usp_IC_ValidateRules.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        
        EXEC dbo.usp_IC_ValidateRules @JobId = @JobId;

        IF EXISTS (SELECT 1 FROM dbo.IC_ImportJob WHERE JobId = @JobId AND JobStatus = 'FAILED')
        BEGIN
            SET @LogMsg = 'Pipeline aborted. dbo.usp_IC_ValidateRules reported a FATAL file-level failure.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'ERROR', @StepName, @LogMsg);
            RETURN;
        END

        -- 4. Execute eConnect Posting
        SET @StepName = 'EXEC_POST_ECONNECT';
        SET @LogMsg = 'Validation complete. Handing off VALIDATED batches to dbo.usp_IC_PostEconnect.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        
        EXEC dbo.usp_IC_PostEconnect @JobId = @JobId;

        SET @StepName = 'PIPELINE_COMPLETE';
        SET @LogMsg = 'Master pipeline execution finished for JobId: ' + CAST(@JobId AS VARCHAR(36));
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

  

    END TRY
    BEGIN CATCH
        DECLARE @ErrMessage VARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;

        SET @LogMsg = 'Unhandled Pipeline Exception: ' + @ErrMessage;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
    END CATCH
END
GO
CREATE  PROCEDURE dbo.usp_IC_ReportJobReconciliation
(
    @JobId UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT EXISTS (SELECT 1 FROM dbo.IC_CompanyHeader WHERE JobId = @JobId)
    BEGIN
        DECLARE @FatalError VARCHAR(MAX);
        
        -- Grab the exact error message from the system log
        SELECT TOP 1 @FatalError = LogMessage 
        FROM dbo.IC_SystemLog 
        WHERE JobId = @JobId AND LogLevel IN ('FATAL', 'ERROR')
        ORDER BY LogId DESC; -- Assuming an identity column, or use CreateDateTime DESC

        -- Return a formatted dummy row that perfectly matches the PowerShell expectations
        SELECT 
            'FILE_REJECTED' AS Company,
            'N/A' AS gp_BatchNumber,
            0 AS gp_Journal,
            LEFT(ISNULL(@FatalError, 'Fatal error during file load. No data staged.'), 129) AS Account,
            0.00 AS Uploaded_Debit,
            0.00 AS Uploaded_Credit,
            0.00 AS gp_DebitAmount,
            0.00 AS gp_CreditAmount,
            0.00 AS Net_Variance,
            'FAILED' AS gp_Status;

        RETURN; -- Exit the procedure immediately so the rest of the recon doesn't run
    END

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

		-- ==============================================================================
        -- OUTPUT: UPLOADED (RAW) vs DYNAMICS GP (NETTED)
        -- ==============================================================================
        SELECT 
            Company,
            BatchId AS gp_BatchNumber,
            JournalEntry AS gp_Journal,
            Account,
            
            -- 1. The Uploaded Side (Raw, exactly as it was in the CSV/Staging)
            IntDebit AS Uploaded_Debit,
            IntCredit AS Uploaded_Credit,
            
            -- 2. The GP Actual Side (Netted, exactly as it appears in the GP UI)
            CASE WHEN (GpDebit - GpCredit) > 0 THEN (GpDebit - GpCredit) ELSE 0 END AS gp_DebitAmount,
            CASE WHEN (GpCredit - GpDebit) > 0 THEN (GpCredit - GpDebit) ELSE 0 END AS gp_CreditAmount,
            
            -- 3. The True Variance (Proves the raw upload mathematically matches the netted GP state)
            ((IntDebit - IntCredit) - (GpDebit - GpCredit)) AS Net_Variance,
            
            GpState AS gp_Status
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
CREATE PROCEDURE dbo.usp_IC_ValidateRules
(
    @JobId UNIQUEIDENTIFIER
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StepName VARCHAR(50) = 'VAL_INIT';
    DECLARE @LogMsg VARCHAR(MAX);
    DECLARE @FileName NVARCHAR(260);

    BEGIN TRY
        SELECT @FileName = FileName FROM dbo.IC_ImportJob WHERE JobId = @JobId;

        SET @LogMsg = 'Starting validation rules for JobId: ' + CAST(@JobId AS VARCHAR(36));
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        -- ==============================================================================
        -- PHASE 1: JOB-LEVEL FATAL CHECKS
        -- ==============================================================================
        SET @StepName = 'VAL_FILE_FLAGS';
        
        IF EXISTS (
            SELECT 1 FROM dbo.IC_StagingRaw 
            WHERE JobId = @JobId AND UPPER(LTRIM(RTRIM(Reversal))) NOT IN ('Y', 'N')
        )
        OR 
        (SELECT COUNT(DISTINCT UPPER(LTRIM(RTRIM(Reversal)))) FROM dbo.IC_StagingRaw WHERE JobId = @JobId) > 1
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            
            SET @LogMsg = 'Mixed or invalid ReversalFlag values. Entire job rejected.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            RETURN;
        END

        SET @StepName = 'VAL_AMOUNT_PRECISION';
        
        IF EXISTS (
            SELECT 1 FROM dbo.IC_StagingRaw 
            WHERE JobId = @JobId AND Amount LIKE '%.[0-9][0-9][0-9]%'
        )
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            
            SET @LogMsg = 'Amount value must have no more than 2 decimal precision. Entire job rejected.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            RETURN;
        END

        SET @StepName = 'VAL_AMOUNT_FORMAT';
        
        IF EXISTS (
            SELECT 1 FROM dbo.IC_StagingRaw 
            WHERE JobId = @JobId AND TRY_CAST(Amount AS DECIMAL(19,2)) = null
        )
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            
            SET @LogMsg = 'Amount value is in incorrect format. Entire job rejected.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            RETURN;
        END

    



        DECLARE @JobRevFlag CHAR(1) = (SELECT TOP 1 UPPER(LTRIM(RTRIM(Reversal))) FROM dbo.IC_StagingRaw WHERE JobId = @JobId);

        -- ==============================================================================
        -- PHASE 2: PROMOTION (STAGING -> HEADER & DETAIL)
        -- ==============================================================================
        SET @StepName = 'PROMOTE_HEADERS';
        
        -- 2A. Build the Company Headers (FileSection built here)
        INSERT INTO dbo.IC_CompanyHeader (JobId, Company, FileSection, Reversal, TrxDate, HeaderStatus)
        SELECT 
            @JobId,
            LTRIM(RTRIM(Company)),
            CONCAT(@FileName, ' | Company=', LTRIM(RTRIM(Company))),
            @JobRevFlag,
            MIN(TRY_CAST(TrxDate AS DATE)), 
            'PENDING'
        FROM dbo.IC_StagingRaw
        WHERE JobId = @JobId
        GROUP BY LTRIM(RTRIM(Company));

        IF @JobRevFlag = 'Y'
        BEGIN
            UPDATE dbo.IC_CompanyHeader
            SET ReversalDate = DATEADD(DAY, 1, EOMONTH(TrxDate))
            WHERE JobId = @JobId;
        END

        -- 2B. Build the Detail Lines
        SET @StepName = 'PROMOTE_LINES';
        INSERT INTO dbo.IC_DetailLine (HeaderId, JobId, Account, AccountDesc, TrxDate, Reference, DebitAmount, CreditAmount)
        SELECT 
            h.HeaderId,
            @JobId,
            CONVERT(CHAR(129), LTRIM(RTRIM(s.Account))), 
            s.AccountDesc,
            TRY_CAST(s.TrxDate AS DATE),
            s.Reference,
            CASE WHEN TRY_CAST(REPLACE(s.Amount, ',', '') AS NUMERIC(19,5)) > 0 THEN TRY_CAST(REPLACE(s.Amount, ',', '') AS NUMERIC(19,5)) ELSE 0 END,
            CASE WHEN TRY_CAST(REPLACE(s.Amount, ',', '') AS NUMERIC(19,5)) < 0 THEN ABS(TRY_CAST(REPLACE(s.Amount, ',', '') AS NUMERIC(19,5))) ELSE 0 END
        FROM dbo.IC_StagingRaw s
        INNER JOIN dbo.IC_CompanyHeader h ON LTRIM(RTRIM(s.Company)) = h.Company AND h.JobId = @JobId
        WHERE s.JobId = @JobId;

        -- 2C. Update Header Aggregates for Reporting
        SET @StepName = 'UPDATE_AGGREGATES';
        UPDATE h
        SET LineCount = (SELECT COUNT(*) FROM dbo.IC_DetailLine l WHERE l.HeaderId = h.HeaderId),
            TotalAmount = (SELECT SUM(DebitAmount) FROM dbo.IC_DetailLine l WHERE l.HeaderId = h.HeaderId)
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId;



        -- ==============================================================================
        -- PHASE 3: THE DUPLICATE GUARD (CONTENT HASHING)
        -- ==============================================================================
        SET @StepName = 'HASH_PAYLOAD';
        
        UPDATE h
        SET ContentHash = HASHBYTES('SHA2_256', 
            (
                SELECT Account, AccountDesc, TrxDate, DebitAmount, CreditAmount, Reference
                FROM dbo.IC_DetailLine l
                WHERE l.HeaderId = h.HeaderId
                ORDER BY Account, AccountDesc, DebitAmount, CreditAmount, Reference
                FOR JSON PATH, INCLUDE_NULL_VALUES
            )
        )
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId;

        UPDATE current_h
        SET HeaderStatus = 'SKIPPED'
        FROM dbo.IC_CompanyHeader current_h
        INNER JOIN dbo.IC_CompanyHeader past_h 
            ON current_h.ContentHash = past_h.ContentHash
            AND past_h.HeaderStatus = 'POSTED'
            AND current_h.Company = past_h.Company
        WHERE current_h.JobId = @JobId;

        IF EXISTS (SELECT 1 FROM dbo.IC_CompanyHeader WHERE JobId = @JobId AND HeaderStatus = 'SKIPPED')
        BEGIN
            SET @LogMsg = 'One or more company payloads match a previously POSTED batch. Marked SKIPPED.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'WARN', @StepName, @LogMsg);
        END

        -- ==============================================================================
        -- PHASE 4: MATH & BUSINESS RULES
        -- ==============================================================================
        SET @StepName = 'VAL_MATH_RULES';
        
        -- Rule 1: Exactly one TrxDate per company
        UPDATE h
        SET HeaderStatus = 'FAILED'
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId AND h.HeaderStatus = 'PENDING'
        AND (SELECT COUNT(DISTINCT TrxDate) FROM dbo.IC_DetailLine l WHERE l.HeaderId = h.HeaderId) <> 1;

        SET @LogMsg = 'Company batch contains multiple Transaction Dates.';
        INSERT INTO dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
        SELECT @JobId, HeaderId, 'ERROR', @StepName, @LogMsg
        FROM dbo.IC_CompanyHeader WHERE JobId = @JobId AND HeaderStatus = 'FAILED' 
        AND HeaderId NOT IN (SELECT HeaderId FROM dbo.IC_SystemLog WHERE JobId = @JobId);

        -- Rule 2: Debits must equal Credits
        UPDATE h
        SET HeaderStatus = 'FAILED'
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId AND h.HeaderStatus = 'PENDING'
        AND (SELECT SUM(DebitAmount) - SUM(CreditAmount) FROM dbo.IC_DetailLine l WHERE l.HeaderId = h.HeaderId) <> 0;

        declare @hs NVARCHAR(300) 
        set @hs = (SELECT headerstatus from IC_CompanyHeader where jobid = @jobid)

        insert into dbo.IC_SystemLog values (@jobid, 'hid', 'INFO', 'Sum check', @hs )


        SET @LogMsg = 'Company batch does not balance (Debits <> Credits).';
        INSERT INTO dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
        SELECT @JobId, HeaderId, 'ERROR', @StepName, @LogMsg
        FROM dbo.IC_CompanyHeader WHERE JobId = @JobId AND HeaderStatus = 'FAILED'
        AND HeaderId NOT IN (SELECT HeaderId FROM dbo.IC_SystemLog WHERE JobId = @JobId);


        -- ==============================================================================
        -- PHASE 5: DYNAMICS GP CROSS-DATABASE CHECKS
        -- ==============================================================================
        SET @StepName = 'VAL_GP_CROSSCHECK';
        
        DECLARE @DynamicCo VARCHAR(20);
        DECLARE @DynSql NVARCHAR(MAX);

        DECLARE CoCursor CURSOR LOCAL FAST_FORWARD FOR 
            SELECT DISTINCT Company FROM dbo.IC_CompanyHeader 
            WHERE JobId = @JobId AND HeaderStatus = 'PENDING';

        OPEN CoCursor;
        FETCH NEXT FROM CoCursor INTO @DynamicCo;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            SET @DynSql = N'
            USE ' + QUOTENAME(@DynamicCo) + N';

            -- Check 1: TrxDate Period
            UPDATE h
            SET h.HeaderStatus = ''FAILED''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND NOT EXISTS (
                SELECT 1 FROM dbo.SY40100 p 
                WHERE p.SERIES = 2 AND p.CLOSED = 0 
                AND h.TrxDate BETWEEN p.PERIODDT AND p.PERDENDT
            );

            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, HeaderId, ''ERROR'', ''VAL_PERIOD'', ''Transaction Date period is closed or does not exist.''
            FROM DYNAMICS.dbo.IC_CompanyHeader 
            WHERE JobId = @pJobId AND Company = @pCompany AND HeaderStatus = ''FAILED''
            AND HeaderId NOT IN (SELECT HeaderId FROM DYNAMICS.dbo.IC_SystemLog WHERE JobId = @pJobId);

            -- Check 2: ReversalDate Period
            UPDATE h
            SET h.HeaderStatus = ''FAILED''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND h.Reversal = ''Y''
            AND NOT EXISTS (
                SELECT 1 FROM dbo.SY40100 p 
                WHERE p.SERIES = 2 AND p.CLOSED = 0 
                AND h.ReversalDate BETWEEN p.PERIODDT AND p.PERDENDT
            );

            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, HeaderId, ''ERROR'', ''VAL_REV_PERIOD'', ''Reversal Date period is closed or does not exist.''
            FROM DYNAMICS.dbo.IC_CompanyHeader 
            WHERE JobId = @pJobId AND Company = @pCompany AND HeaderStatus = ''FAILED''
            AND HeaderId NOT IN (SELECT HeaderId FROM DYNAMICS.dbo.IC_SystemLog WHERE JobId = @pJobId);

            -- Check 3: Account Exists & Active
            UPDATE h
            SET h.HeaderStatus = ''FAILED''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND EXISTS (
                SELECT 1 FROM DYNAMICS.dbo.IC_DetailLine l
                LEFT JOIN dbo.GL00105 a ON l.Account = a.ACTNUMST
                LEFT JOIN dbo.GL00100 m ON a.ACTINDX = m.ACTINDX
                WHERE l.HeaderId = h.HeaderId
                AND (a.ACTNUMST IS NULL OR m.ACTIVE = 0)
            );

            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, HeaderId, ''ERROR'', ''VAL_ACCOUNT'', ''One or more accounts do not exist or are inactive in GP.''
            FROM DYNAMICS.dbo.IC_CompanyHeader 
            WHERE JobId = @pJobId AND Company = @pCompany AND HeaderStatus = ''FAILED''
            AND HeaderId NOT IN (SELECT HeaderId FROM DYNAMICS.dbo.IC_SystemLog WHERE JobId = @pJobId);
            ';

            EXEC sp_executesql @DynSql, 
                N'@pJobId UNIQUEIDENTIFIER, @pCompany VARCHAR(20)', 
                @pJobId = @JobId, @pCompany = @DynamicCo;

            FETCH NEXT FROM CoCursor INTO @DynamicCo;
        END

        CLOSE CoCursor;
        DEALLOCATE CoCursor;

        -- ==============================================================================
        -- PHASE 6: FINALIZE
        -- ==============================================================================
        SET @StepName = 'VAL_FINALIZE';
        
        -- Promote survivors
        UPDATE dbo.IC_CompanyHeader
        SET HeaderStatus = 'VALIDATED'
        WHERE JobId = @JobId AND HeaderStatus = 'PENDING';

        -- Generate BatchIds
        UPDATE dbo.IC_CompanyHeader
        SET BatchId = 'GLTX' + RIGHT(REPLICATE('0', 8) + CAST(NEXT VALUE FOR dbo.IC_Batch_Seq AS VARCHAR(32)), 8)
        WHERE JobId = @JobId AND HeaderStatus = 'VALIDATED';

        -- Final Job Status
        IF EXISTS (SELECT 1 FROM dbo.IC_CompanyHeader WHERE JobId = @JobId AND HeaderStatus = 'VALIDATED')
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'VALIDATED' WHERE JobId = @JobId;
            SET @LogMsg = 'Validation complete. Found VALIDATED batches ready for GP.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        END
        ELSE
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            SET @LogMsg = 'Validation complete. Zero batches survived the validation gauntlet.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'WARN', @StepName, @LogMsg);
        END

    END TRY
    BEGIN CATCH
        DECLARE @ErrMessage VARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;

        SET @LogMsg = 'Validation Exception Unable to Load File: ' + @ErrMessage;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
        
        IF CURSOR_STATUS('local', 'CoCursor') >= -1
        BEGIN
            CLOSE CoCursor; DEALLOCATE CoCursor;
        END
    END CATCH
END
GO
# Requires -Version 5.1
# Requires -Modules SqlServer
# Test Comment
# Run in an elevated PowerShell session (not in SQLPS)
#Install-Module -Name SqlServer -Scope AllUsers -Force
# or update
#Update-Module -Name SqlServer



# ----------------------------
# UTILITIES
# ----------------------------

function Ensure-Directory {
    param([Parameter(Mandatory = $true)][string]$Path)
    if (-not (Test-Path -Path $Path)) {
        New-Item -ItemType Directory -Path $Path -Force | Out-Null
    }
}
function Send-JobEmail{
    param(
        [Parameter(Mandatory=$true)][String]$Subject,
        [Parameter(Mandatory=$true)][String]$Body

    )
    
    $EmailParams = @{
        SMTPServer = $EmailConfig.Server
        From       = $EmailConfig.From
        To         = $EmailConfig.To
        #CC         = $CC
        Subject    = $Subject
        Body       = $Body
        BodyAsHTML = $true
    }

    Send-MailMessage @EmailParams
}

function Get-JobResult{
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    $Query = "SELECT JobStatus FROM dbo.IC_ImportJob WHERE JobID= '$JobID'"
    $QueryResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    $JobStatus = $($QueryResult.JobStatus).ToString().Trim().ToUpper()

    return $JobStatus

}


function Write-Log{
    param(
        [Parameter(Mandatory=$true)]
        [String]$Message,
        [switch]$IncludeHeader,
        [switch]$IncludeFooter
    )

    $TimeStamp = Get-Date -Format "yyyyMMdd HH:mm:ss"
    
    if($IncludeHeader){
        # Log Username, Clustername, NodeName
        $User = $env:USERNAME
        $Hostname = $env:COMPUTERNAME
        $Node = [System.Environment]::MachineName
        $LogEntry = "$TimeStamp | User: $User | Hostname: $Hostname | Node: $Node `n$Timestamp | $Message"
    } else {
        # Log Message Only
        $LogEntry = "$TimeStamp | $Message"
    }

    # Add linebreak
    if($IncludeFooter){
        $LogEntry += "`n"
    }

    Add-Content -Path $Paths.Log -Value $LogEntry
}
function Move-File {
    param(
        [Parameter(Mandatory = $true)][string]$Source,
        [Parameter(Mandatory = $true)][string]$DestinationDir
    )
    try {
        Ensure-Directory -Path $DestinationDir
        $target = Join-Path -Path $DestinationDir -ChildPath (Split-Path -Path $Source -Leaf)
        return Move-Item -Path $Source -Destination $target -PassThru -Force -ErrorAction Stop
    }
    catch {
        Write-Log "Failed to move file [$Source] to [$DestinationDir]. Error: $($_.Exception.Message)"
        return $null
    }
}


function Route-ProcessedFile {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][System.IO.FileInfo]$File,
        [Parameter(Mandatory=$true)][string]$Status
    )
    
    switch ($Status.ToUpperInvariant()) {
        'POSTED' {
            $targetDir = $Paths.Archive
        }
        'PARTIAL' {
            $targetDir = if ($Paths.ContainsKey('Partial')) { $Paths.Partial } else { $Paths.ErrorDir }
        }
        default {
            $targetDir = $Paths.ErrorDir
        }
    }

    Move-File -Source $File.FullName -DestinationDir $targetDir | Out-Null
    Write-Log "File routed to: $targetDir based on status [$Status]"
}


function New-DynamicsImportErrorEmail {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][Guid]$JobId,
        [Parameter(Mandatory=$true)][string]$FileName
    )
    Write-Log "Generating Error HTML Body for JobID: $JobId"
    
    $Query = "SELECT TOP 1 LogMessage FROM dbo.IC_SystemLog WHERE JobId = '$JobId' AND LogLevel = 'FATAL' ORDER BY LogId DESC"
    $SqlResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate
    $FatalError = if ($null -ne $SqlResult) { $SqlResult.LogMessage } else { "Unknown fatal error occurred. Check system logs." }

    $HtmlTemplate = Get-Content $Paths.ErrorTemplate -Raw
    return $HtmlTemplate -replace '{{JobId}}', $JobId `
                         -replace '{{FileName}}', $FileName `
                         -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                         -replace '{{ErrorMessage}}', $FatalError
}

function New-DynamicsImportSuccessEmail {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][Guid]$JobId
    )
    Write-Log "Generating Report HTML Body for JobID: $JobId"
    
    $Query = "EXEC $($SQLConfig.ReportSP) @JobId = '$JobId'"
    $Details = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    # Build Summary Rows
    $SummaryGroups = $Details | Group-Object Company, gp_BatchNumber, gp_Status
    $SummaryHtml = ""
    foreach ($Group in $SummaryGroups) {
        $CleanStatus = if ($null -ne $Group.Group[0].gp_Status) { $Group.Group[0].gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
        $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
        $SummaryHtml += "<tr> <td>$($Group.Group[0].Company)</td> <td>$($Group.Group[0].gp_BatchNumber)</td> <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> <td class='right'>$($Group.Count)</td> </tr>`n"
    }

    # Build Detail Rows
    $DetailHtml = ""
    $GroupedDetails = $Details | Group-Object Company
    foreach ($CompGroup in $GroupedDetails) {
        $DetailHtml += "<tr class='company-header'><td colspan='9'>Company: $($CompGroup.Name)</td></tr>`n"
        $SubUpDeb = 0; $SubUpCred = 0; $SubGpDeb = 0; $SubGpCred = 0; $SubVar = 0

        foreach ($Row in $CompGroup.Group) {
            $SubUpDeb += $Row.Uploaded_Debit; $SubUpCred += $Row.Uploaded_Credit
            $SubGpDeb += $Row.gp_DebitAmount; $SubGpCred += $Row.gp_CreditAmount; $SubVar += $Row.Net_Variance

            $CleanRowStatus = if ($null -ne $Row.gp_Status) { $Row.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
            $StatusClass = switch ($CleanRowStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
            $VarClass = if ($Row.Net_Variance -ne 0) { "variance-bad right" } else { "variance-good right" }

            $DetailHtml += "<tr> <td>$($Row.Company)</td> <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td> <td>$($Row.Account)</td> <td class='right'>$("{0:N2}" -f $Row.Uploaded_Debit)</td> <td class='right'>$("{0:N2}" -f $Row.Uploaded_Credit)</td> <td class='right'>$("{0:N2}" -f $Row.gp_DebitAmount)</td> <td class='right'>$("{0:N2}" -f $Row.gp_CreditAmount)</td> <td class='$VarClass'>$("{0:N2}" -f $Row.Net_Variance)</td> <td class='center'><span class='$StatusClass'>$CleanRowStatus</span></td> </tr>`n"
        }
        $DetailHtml += "<tr class='subtotal-row'> <td colspan='3' class='right'>$($CompGroup.Name) Totals:</td> <td class='right'>$("{0:N2}" -f $SubUpDeb)</td> <td class='right'>$("{0:N2}" -f $SubUpCred)</td> <td class='right'>$("{0:N2}" -f $SubGpDeb)</td> <td class='right'>$("{0:N2}" -f $SubGpCred)</td> <td class='right'>$("{0:N2}" -f $SubVar)</td> <td></td> </tr>`n"
    }

    $HtmlTemplate = Get-Content $Paths.SuccessTemplate -Raw
    return $HtmlTemplate -replace '{{JobId}}', $JobId `
                         -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                         -replace '{{SummaryRows}}', $SummaryHtml `
                         -replace '{{DetailRows}}', $DetailHtml
}



function Invoke-IntracompanyImportSP {
    param([Parameter(Mandatory = $true)][string]$CsvFullPath)

    Write-Log "Beginning Main Procedure Call"

    $query = @"
        DECLARE @GeneratedJobID UNIQUEIDENTIFIER;

        EXEC $($SQLConfig.MainSP) 
          @CSVPath = '$CsvFullPath',
          @JobID = @GeneratedJobID OUTPUT;

        SELECT @GeneratedJobID AS JobID
"@
    
    Write-Log "Executing Query: $Query"

    try {

        $SQLResult = @(Invoke-Sqlcmd -ServerInstance $SQLConfig.server -Database $SQLConfig.database -Query $query -TrustServerCertificate)
        $JobID = $SQLResult.JobID
        if ($null -eq $JobID -or $SQLResult.Count -eq 0) {
            Write-Log "Did not find JobID - Result count: $($SQLResult.Count)"
            return "No JobID returned"
        }
        else {
            Write-Log "Found JobID: $JobID"
            return $JobID
        }   
    }
    catch {
        Write-Log "SQL/eConnect Error while processing [$CsvFullPath]: $($_.Exception.Message)"
        return $null
    }
}

function Invoke-Main {
    $Paths.Keys | ForEach-Object {
        if ($_ -ne 'Log') { Ensure-Directory -Path $Paths[$_] }
    }

    # 1. Scan for files
    $files = @(Get-ChildItem -Path $Paths.In -Filter *.csv -File -ErrorAction SilentlyContinue)
    if ($files.Count -eq 0) { return }

    Write-Log "-- Import Begin --" -IncludeHeader

    foreach ($file in $files) {
        Write-Log "Found file: $($file.Name)"
        
        # 1.5 Isolate the file to prevent locking/double-processing
        $working = Move-File -Source $file.FullName -DestinationDir $Paths.Proc
        if ($null -eq $working) {
            Write-Log "Skipping: file locked/in use or failed to move. File: $($file.Name)"
            continue
        }

        try {
            Write-Log "Starting SQL import for: $($working.FullName)"
            
            # 2 & 3. Run the file and get the Job ID back
            $jobId = Invoke-IntracompanyImportSP -CsvFullPath $working.FullName
            if (-not $jobId) { throw "No JobID returned from SQL." }

            # 4. Check if it ran success/partial/fail
            $jobResult = Get-JobResult -JobID $jobId
            if (-not $jobResult) { $jobResult = "FAILED" }

            # 5. Go to the function that creates the appropriate email
            if ($jobResult -in 'POSTED', 'PARTIAL') {
                $emailBody = New-DynamicsImportSuccessEmail -JobId $jobId
            } else {
                $emailBody = New-DynamicsImportErrorEmail -JobId $jobId -FileName $working.Name
            }

            # 6. Move file
            Route-ProcessedFile -File $working -Status $jobResult

            # 7. Send email
            $subject = "GP Import $($jobResult): $($working.Name)"
            Send-JobEmail -Subject $subject -Body $emailBody

        } catch {
            Write-Log "Unhandled failure for $($file.Name): $($_.Exception.Message)"
            Route-ProcessedFile -File $working -Status "FAILED"
            
            # Fallback email if the pipeline completely explodes
            $emailBody = New-DynamicsImportErrorEmail -JobId ([Guid]::Empty) -FileName $working.Name
            Send-JobEmail -Subject "GP Import CRASHED: $($working.Name)" -Body $emailBody
        }
    }
    
    Write-Log "-- Import End --" -IncludeFooter
}

Invoke-Main
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #333; background-color: #f4f7f6; margin: 0; padding: 20px; }
        .container { background-color: #fff; padding: 20px; border-top: 5px solid #e74c3c; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); max-width: 800px; margin: auto; }
        h2 { color: #e74c3c; margin-top: 0; border-bottom: 2px solid #ecf0f1; padding-bottom: 10px; }
        .meta-data { background: #f9fbfd; padding: 15px; border: 1px solid #e0e6ed; margin-bottom: 20px; font-size: 14px; }
        .error-box { background: #f8d7da; color: #721c24; padding: 15px; border: 1px solid #f5c6cb; border-radius: 4px; font-family: 'Courier New', Courier, monospace; font-size: 14px; line-height: 1.5; }
    </style>
</head>
<body>
    <div class="container">
        <h2>Intracompany Upload Failed</h2>
        
        <div class="meta-data">
            <strong>Job ID:</strong> {{JobId}}<br>
            <strong>File:</strong> {{FileName}}<br>
            <strong>Time:</strong> {{RunDate}}
        </div>

        <h3>Error Details:</h3>
        <div class="error-box">
            {{ErrorMessage}}
        </div>
        
        <p><em>Please correct the file and drop it back into the inbound folder.</em></p>
    </div>
</body>
</html>

<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #333; background-color: #f4f7f6; margin: 0; padding: 20px; }
        .container { background-color: #fff; padding: 20px; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); max-width: 1200px; margin: auto; }
        h2 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; margin-top: 0; }
        h3 { color: #34495e; margin-top: 30px; }
        .meta-data { font-size: 14px; color: #555; margin-bottom: 20px; }
        table { border-collapse: collapse; width: 100%; font-size: 13px; margin-bottom: 20px; }
        th { background-color: #ecf0f1; color: #2c3e50; font-weight: bold; text-align: left; padding: 10px; border: 1px solid #bdc3c7; }
        td { padding: 8px 10px; border: 1px solid #e0e6ed; }
        tr:nth-child(even) { background-color: #f9fbfd; }
        
        /* Alignment */
        .right { text-align: right; font-family: 'Courier New', Courier, monospace; }
        .center { text-align: center; }
        
        /* Dynamic Status Colors */
        .status-ok { color: #155724; background-color: #d4edda; font-weight: bold; padding: 3px 6px; border-radius: 3px; display: inline-block;}
        .status-fail { color: #721c24; background-color: #f8d7da; font-weight: bold; padding: 3px 6px; border-radius: 3px; display: inline-block;}
        
        /* Dynamic Variance Colors */
        .variance-bad { color: #fff; background-color: #e74c3c; font-weight: bold; }
        .variance-good { color: #27ae60; }

	/* Grouping & Totals */
        .company-header { background-color: #2c3e50; color: #ffffff; font-weight: bold; font-size: 14px; }
        .company-header td { padding: 10px; border-bottom: none; }
        .subtotal-row { background-color: #ecf0f1; font-weight: bold; }
        .subtotal-row td { border-top: 2px solid #bdc3c7; }


    </style>
</head>
<body>
    <div class="container">
        <h2>Intracompany Upload Reconciliation</h2>
        
        <div class="meta-data">
            <strong>Job ID:</strong> {{JobId}}<br>
            <strong>Run Date:</strong> {{RunDate}}
        </div>

        <h3>Execution Summary</h3>
        <table>
            <thead>
                <tr>
                    <th>Company</th>
                    <th>Batch ID</th>
                    <th class="center">Status</th>
                    <th class="right">Total Uploaded Lines</th>
                </tr>
            </thead>
            <tbody>
                {{SummaryRows}}
            </tbody>
        </table>

        <h3>Detail Reconciliation</h3>
        <table>
            <thead>
                <tr>
                    <th>Company</th>
                    <th>Batch / Journal</th>
                    <th>Account</th>
                    <th class="right">Uploaded Debit</th>
                    <th class="right">Uploaded Credit</th>
                    <th class="right">GP Debit</th>
                    <th class="right">GP Credit</th>
                    <th class="right">Net Variance</th>
                    <th class="center">GP Status</th>
                </tr>
            </thead>
            <tbody>
                {{DetailRows}}
            </tbody>
        </table>
    </div>
</body>
</html>

