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
