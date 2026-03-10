ALTER PROCEDURE dbo.usp_IC_ValidateRules
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

        -- STRICT FORMAT GATEKEEPER: Will naturally catch commas, letters, and blanks.
        SET @StepName = 'VAL_AMOUNT_FORMAT';
        IF EXISTS (
            SELECT 1 FROM dbo.IC_StagingRaw 
            WHERE JobId = @JobId AND TRY_CAST(Amount AS DECIMAL(19,2)) IS NULL
        )
        BEGIN
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            SET @LogMsg = 'Amount value is in incorrect format (Non-numeric or contains illegal characters like commas). Entire job rejected.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            RETURN;
        END

        DECLARE @JobRevFlag CHAR(1) = (SELECT TOP 1 UPPER(LTRIM(RTRIM(Reversal))) FROM dbo.IC_StagingRaw WHERE JobId = @JobId);

        -- ==============================================================================
        -- PHASE 2: PROMOTION (STAGING -> HEADER & DETAIL)
        -- ==============================================================================
        SET @StepName = 'PROMOTE_HEADERS';
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

        -- STRIPPED THE "REPLACE" COMMAS HACK.
        -- We now rely entirely on the clean, raw string passing the Phase 1 gate.
        SET @StepName = 'PROMOTE_LINES';
        INSERT INTO dbo.IC_DetailLine (HeaderId, JobId, Account, AccountDesc, TrxDate, Reference, DebitAmount, CreditAmount)
        SELECT 
            h.HeaderId,
            @JobId,
            CONVERT(CHAR(129), LTRIM(RTRIM(s.Account))), 
            s.AccountDesc,
            TRY_CAST(s.TrxDate AS DATE),
            s.Reference,
            CASE WHEN TRY_CAST(s.Amount AS NUMERIC(19,5)) > 0 THEN TRY_CAST(s.Amount AS NUMERIC(19,5)) ELSE 0 END,
            CASE WHEN TRY_CAST(s.Amount AS NUMERIC(19,5)) < 0 THEN ABS(TRY_CAST(s.Amount AS NUMERIC(19,5))) ELSE 0 END
        FROM dbo.IC_StagingRaw s
        INNER JOIN dbo.IC_CompanyHeader h ON LTRIM(RTRIM(s.Company)) = h.Company AND h.JobId = @JobId
        WHERE s.JobId = @JobId;

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
        -- PHASE 4: MATH & BUSINESS RULES (ERROR-DRIVEN)
        -- ==============================================================================
        SET @StepName = 'VAL_MATH_RULES';
        
        -- Rule 1: Exactly one TrxDate per company
        INSERT INTO dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
        SELECT 
            @JobId, 
            h.HeaderId, 
            'ERROR', 
            @StepName, 
            'Company batch contains multiple Transaction Dates.'
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId AND h.HeaderStatus = 'PENDING'
        AND (SELECT COUNT(DISTINCT TrxDate) FROM dbo.IC_DetailLine l WHERE l.HeaderId = h.HeaderId) <> 1;

        -- Rule 2: Debits must equal Credits
        INSERT INTO dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
        SELECT 
            @JobId, 
            h.HeaderId, 
            'ERROR', 
            @StepName, 
            'Company batch does not balance. Debits: ' + 
               CAST(ISNULL(SUM(l.DebitAmount),0) AS VARCHAR(50)) + 
               ' Credits: ' + CAST(ISNULL(SUM(l.CreditAmount),0) AS VARCHAR(50))
        FROM dbo.IC_CompanyHeader h
        INNER JOIN dbo.IC_DetailLine l ON h.HeaderId = l.HeaderId
        WHERE h.JobId = @JobId AND h.HeaderStatus = 'PENDING'
        GROUP BY h.HeaderId
        HAVING ISNULL(SUM(l.DebitAmount),0) - ISNULL(SUM(l.CreditAmount),0) <> 0;

        -- ==============================================================================
        -- PHASE 5: DYNAMICS GP CROSS-DATABASE CHECKS (ERROR-DRIVEN)
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
            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, h.HeaderId, ''ERROR'', ''VAL_PERIOD'', ''Transaction Date period is closed or does not exist.''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND NOT EXISTS (
                SELECT 1 FROM dbo.SY40100 p 
                WHERE p.SERIES = 2 AND p.CLOSED = 0 
                AND h.TrxDate BETWEEN p.PERIODDT AND p.PERDENDT
            );

            -- Check 2: ReversalDate Period
            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, h.HeaderId, ''ERROR'', ''VAL_REV_PERIOD'', ''Reversal Date period is closed or does not exist.''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND h.Reversal = ''Y''
            AND NOT EXISTS (
                SELECT 1 FROM dbo.SY40100 p 
                WHERE p.SERIES = 2 AND p.CLOSED = 0 
                AND h.ReversalDate BETWEEN p.PERIODDT AND p.PERDENDT
            );

            -- Check 3: Account Exists & Active
            INSERT INTO DYNAMICS.dbo.IC_SystemLog (JobId, HeaderId, LogLevel, StepName, LogMessage)
            SELECT @pJobId, h.HeaderId, ''ERROR'', ''VAL_ACCOUNT'', ''One or more accounts do not exist or are inactive in GP.''
            FROM DYNAMICS.dbo.IC_CompanyHeader h
            WHERE h.JobId = @pJobId AND h.Company = @pCompany AND h.HeaderStatus = ''PENDING''
            AND EXISTS (
                SELECT 1 FROM DYNAMICS.dbo.IC_DetailLine l
                LEFT JOIN dbo.GL00105 a ON l.Account = a.ACTNUMST
                LEFT JOIN dbo.GL00100 m ON a.ACTINDX = m.ACTINDX
                WHERE l.HeaderId = h.HeaderId
                AND (a.ACTNUMST IS NULL OR m.ACTIVE = 0)
            );
            ';

            EXEC sp_executesql @DynSql, 
                N'@pJobId UNIQUEIDENTIFIER, @pCompany VARCHAR(20)', 
                @pJobId = @JobId, @pCompany = @DynamicCo;

            FETCH NEXT FROM CoCursor INTO @DynamicCo;
        END

        CLOSE CoCursor;
        DEALLOCATE CoCursor;

        -- ==============================================================================
        -- PHASE 6: FINALIZE (THE MASTER STATE MUTATION)
        -- ==============================================================================
        SET @StepName = 'VAL_FINALIZE';
        
        -- 1. Fail any header that caught an error during Phase 4 or 5
        UPDATE h
        SET h.HeaderStatus = 'FAILED'
        FROM dbo.IC_CompanyHeader h
        WHERE h.JobId = @JobId AND h.HeaderStatus = 'PENDING'
        AND EXISTS (
            SELECT 1 FROM dbo.IC_SystemLog log 
            WHERE log.JobId = @JobId AND log.HeaderId = h.HeaderId AND log.LogLevel = 'ERROR'
        );

        -- 2. Promote true survivors
        UPDATE dbo.IC_CompanyHeader
        SET HeaderStatus = 'VALIDATED'
        WHERE JobId = @JobId AND HeaderStatus = 'PENDING';

        -- 3. Generate BatchIds for survivors
        UPDATE dbo.IC_CompanyHeader
        SET BatchId = 'GLTX' + RIGHT(REPLICATE('0', 8) + CAST(NEXT VALUE FOR dbo.IC_Batch_Seq AS VARCHAR(32)), 8)
        WHERE JobId = @JobId AND HeaderStatus = 'VALIDATED';

        -- 4. Final Job Status
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
        
        SET @LogMsg = 'Validation Exception: ' + @ErrMessage;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
        
        IF CURSOR_STATUS('local', 'CoCursor') >= -1
        BEGIN
            CLOSE CoCursor;
            DEALLOCATE CoCursor;
        END
    END CATCH
END
GO
