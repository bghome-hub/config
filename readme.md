-- 4.2 Export Buffer → CSV using xp_cmdshell + bcp (with error handling)
CREATE PROCEDURE dbo.usp_BoA_AV_ExportBatchToFile
    @BatchId     UNIQUEIDENTIFIER = NULL OUTPUT,
    @OutFolder   NVARCHAR(4000),
    @FullPath    NVARCHAR(4000) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @FileName NVARCHAR(260);

    BEGIN TRY
        IF @BatchId IS NULL
            SELECT TOP (1) @BatchId = BatchId FROM dbo.BoA_AV_Batches WHERE Exported = 0 ORDER BY CreatedUtc DESC;

        IF @BatchId IS NULL
        BEGIN
            RAISERROR('No unexported batches found.', 11, 1);
            RETURN;
        END

        SELECT @FileName = FileName FROM dbo.BoA_AV_Batches WHERE BatchId = @BatchId;
        IF @FileName IS NULL
        BEGIN
            RAISERROR('Batch found but FileName is NULL.', 16, 1);
            RETURN;
        END

        SET @FullPath = @OutFolder + N'\' + @FileName;

        DECLARE @mkdir NVARCHAR(4000) = N'IF NOT EXIST "' + @OutFolder + N'" MKDIR "' + @OutFolder + N'"';
        EXEC master..xp_cmdshell @mkdir, NO_OUTPUT;

        DECLARE @cmd NVARCHAR(4000) = N'bcp "SELECT CsvLine FROM ' + QUOTENAME(DB_NAME()) + N'.dbo.BoA_AV_ExportBuffer WHERE BatchId=''' + CONVERT(NVARCHAR(36), @BatchId) + N''' ORDER BY LineNumber" queryout "' + @FullPath + N'" -c -T -S ' + QUOTENAME(@@SERVERNAME, '''');


        DECLARE @out TABLE(line NVARCHAR(4000));
        INSERT @out EXEC master..xp_cmdshell @cmd;

        INSERT dbo.BoA_AV_JobLog(Stepname, BatchID, FileName, Message)
        values('BCP Command', @BatchID, @Filename, @cmd)


        DECLARE @exists TABLE (FileExists INT, FileIsDir INT, ParentDirExists INT);
        INSERT @exists EXEC master..xp_fileexist @FullPath;
        IF NOT EXISTS (SELECT 1 FROM @exists WHERE FileExists = 1)
        BEGIN
            RAISERROR('BCP completed but output file not found: %s', 16, 1, @FullPath);
        END

        UPDATE dbo.BoA_AV_Batches
           SET Exported = 1,
               ExportPath = @FullPath,
               Notes = CONCAT(ISNULL(Notes,''), ' | Exported ', CONVERT(VARCHAR(19), GETUTCDATE(), 120), ' UTC')
         WHERE BatchId = @BatchId;

        UPDATE o
           SET o.Status = 'Exported',
               o.LastUpdatedUtc = SYSUTCDATETIME()
          FROM dbo.BoA_AV_Outbox o
         WHERE o.FileName = @FileName
           AND o.Status = 'ReadyForUpload';

        INSERT dbo.BoA_AV_JobLog(StepName, BatchId, FileName, Message)
        VALUES('ExportBatchToFile', @BatchId, @FileName, CONCAT('Exported to: ', @FullPath));

    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog(StepName, BatchId, FileName, Message, ErrorNumber, ErrorSeverity, ErrorState, ErrorLine, ErrorProc)
        VALUES('ExportBatchToFile', @BatchId, @FileName, ERROR_MESSAGE(), ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(), ERROR_LINE(), ERROR_PROCEDURE());
        THROW;
    END CATCH
END
GO
