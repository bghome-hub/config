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
            SELECT HeaderId, Company, BatchId, TrxDate, ReversalDate, ReversalFlag
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
            -- CORRECTED: Using the outer @Company variable
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

            -- CORRECTED: Removed @pCompany from parameters entirely.
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
