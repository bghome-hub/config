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
        Company VARCHAR(MAX), Account VARCHAR(MAX), AcctDesc VARCHAR(MAX),
        Amount VARCHAR(MAX), TrxDate VARCHAR(MAX), Reference VARCHAR(MAX), ReversalFlag VARCHAR(MAX)
    );

    BEGIN TRY
        -- 2. Execute Bulk Insert
        SET @StepName = 'BULK_INSERT';
        SET @LogMsg = 'Executing BULK INSERT with FIRSTROW = 1 to capture headers.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        DECLARE @Sql NVARCHAR(MAX) = N'
            BULK INSERT #RawStaging
            FROM ' + QUOTENAME(@CsvPath, '''') + N'
            WITH (
                FIRSTROW = 1, -- We MUST load row 1 to validate headers
                FIELDTERMINATOR = '','',
                ROWTERMINATOR = ''0x0d0a'',
                FIELDQUOTE = ''"''
            );';

        EXEC(@Sql);

        -- 3. Validate Headers (The Bouncer Logic)
        SET @StepName = 'VALIDATE_HEADERS';
        SET @LogMsg = 'Interrogating Row 1 for expected column schema.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);
        
        -- We clean the strings (replace quotes/spaces) just in case the CSV generator was sloppy
        IF NOT EXISTS (
            SELECT 1 FROM #RawStaging 
            WHERE REPLACE(Company, '"', '') = 'Company' 
              AND REPLACE(Account, '"', '') = 'Account'
              AND REPLACE(Amount, '"', '') = 'Amount'
              AND REPLACE(TrxDate, '"', '') = 'TrxDate'
              AND REPLACE(ReversalFlag, '"', '') = 'ReversalFlag'
        )
        BEGIN
            -- Headers are missing, out of order, or malformed
            UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
            
            SET @LogMsg = 'Invalid CSV Headers. Expected: Company, Account, AcctDesc, Amount, TrxDate, Reference, ReversalFlag';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
            VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
            
            DROP TABLE #RawStaging;
            RETURN;
        END

        -- 4. Clean the Header Row out of the Temp Table
        SET @StepName = 'REMOVE_HEADERS';
        DELETE FROM #RawStaging 
        WHERE REPLACE(Company, '"', '') = 'Company' AND REPLACE(Account, '"', '') = 'Account';

        -- 5. Move to Persistent Staging
        SET @StepName = 'INSERT_STAGING';
        SET @LogMsg = 'Headers verified. Pushing raw text to persistent IC_StagingRaw table.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        INSERT INTO dbo.IC_StagingRaw (JobId, Company, Account, AcctDesc, Amount, TrxDate, Reference, ReversalFlag)
        SELECT @JobId, Company, Account, AcctDesc, Amount, TrxDate, Reference, ReversalFlag
        FROM #RawStaging;

        DECLARE @RowCount INT = @@ROWCOUNT;

        -- 6. Success
        UPDATE dbo.IC_ImportJob SET JobStatus = 'LOADED' WHERE JobId = @JobId;
        
        SET @LogMsg = 'Successfully staged ' + CAST(@RowCount AS VARCHAR(20)) + ' data rows.';
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'INFO', @StepName, @LogMsg);

        DROP TABLE #RawStaging;

    END TRY
    BEGIN CATCH
        DECLARE @BulkErr VARCHAR(MAX) = ERROR_MESSAGE();
        
        UPDATE dbo.IC_ImportJob SET JobStatus = 'FAILED' WHERE JobId = @JobId;
        
        SET @LogMsg = 'Bulk Load Exception: ' + @BulkErr;
        INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage)
        VALUES (@JobId, 'FATAL', @StepName, @LogMsg);
        
        IF OBJECT_ID('tempdb..#RawStaging') IS NOT NULL DROP TABLE #RawStaging;
    END CATCH
END
GO
