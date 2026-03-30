CREATE PROCEDURE dbo.usp_BoA_AV_Req_S04_WriteFile
(
    @BatchId UNIQUEIDENTIFIER,
    @Environment NVARCHAR(10) = N'PREPROD',
    @FullPath NVARCHAR(4000) OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @OutFolder NVARCHAR(4000),
            @FileName NVARCHAR(260),
            @cmd NVARCHAR(4000),
            @cmdSql NVARCHAR(4000),
            @proc SYSNAME = N'usp_BoA_AV_Req_S04_WriteFile';

    BEGIN TRY
        ------------------------------------------------------------
        -- 1) Validate batch
        ------------------------------------------------------------
        SELECT @FileName = FileName 
        FROM dbo.BoA_AV_Req_Batches 
        WHERE BatchId = @BatchId AND Exported = 0;

        IF @FileName IS NULL
        BEGIN
            RAISERROR('No unexported batch found for supplied BatchId.', 16, 1);
            RETURN;
        END

        ------------------------------------------------------------
        -- 2) Resolve output folder
        ------------------------------------------------------------
        SET @OutFolder = CASE 
            WHEN @Environment = 'PROD' THEN N'\\VLOBPRODFS\PRODUCTION_PASS\' 
            ELSE N'\\VLOBPREPRODFS\PREPROD_PASS\' 
        END + N'GP2018Data\FileTransfer\boa_av\sent_files';
        
        SET @FullPath = @OutFolder + N'\' + @FileName;

        ------------------------------------------------------------
        -- 3) Write CSV via BCP
        ------------------------------------------------------------
        SET @cmd = N'bcp "SELECT CsvLine FROM ' + QUOTENAME(DB_NAME()) + 
                   N'.dbo.BoA_AV_Req_ExportBuffer WHERE BatchId = ''' + 
                   CONVERT(NVARCHAR(36), @BatchId) + 
                   N''' ORDER BY LineNumber" queryout "' + 
                   @FullPath + N'" -c -T -S ' + @@SERVERNAME;
                   
        SET @cmdSql = N'EXEC master..xp_cmdshell @c';
        
        EXEC sp_executesql @cmdSql, N'@c NVARCHAR(4000)', @c = @cmd;

        ------------------------------------------------------------
        -- 4) Verify file existence
        ------------------------------------------------------------
        DECLARE @exists TABLE (
            FileExists INT, 
            FileIsDirectory INT, 
            ParentDirExists INT
        );
        
        INSERT @exists EXEC master..xp_fileexist @FullPath;
        
        IF NOT EXISTS (SELECT 1 FROM @exists WHERE FileExists = 1)
        BEGIN
            RAISERROR('BCP reported success but file not found: %s', 16, 1, @FullPath);
            RETURN;
        END

        ------------------------------------------------------------
        -- 5) Mark batch + rows exported
        ------------------------------------------------------------
        UPDATE dbo.BoA_AV_Req_Batches 
        SET 
            Exported = 1, 
            ExportPath = @FullPath, 
            Notes = CONCAT(ISNULL(Notes,''), ' Exported ', CONVERT(VARCHAR(19), SYSUTCDATETIME(), 120), ' UTC')
        WHERE BatchId = @BatchId;

        UPDATE o 
        SET 
            o.Status = 'Exported', 
            o.LastUpdatedUtc = SYSUTCDATETIME()
        FROM dbo.BoA_AV_Req_Outbox o 
        WHERE o.FileName = @FileName 
          AND o.Status = 'ReadyForUpload';

        ------------------------------------------------------------
        -- 6) Advance the Lifecycle Tracker
        ------------------------------------------------------------
        UPDATE l
        SET 
            l.RequestBatchId = @BatchId,
            l.RequestFileName = @FileName,
            l.FileSentUtc = SYSUTCDATETIME(),
            l.ProcessingStatus = 'SENT',
            l.LastUpdatedUtc = SYSUTCDATETIME()
        FROM dbo.BoA_AV_Lifecycle l
        JOIN dbo.BoA_AV_Req_Outbox o ON o.EndToEndId = l.EndToEndId
        WHERE o.FileName = @FileName 
          AND o.Status = 'Exported';

        ------------------------------------------------------------
        -- 7) Notify
        ------------------------------------------------------------
        EXEC dbo.usp_BoA_AV_Req_S05_SendEmail 
            @BatchId = @BatchId, 
            @FileName = @FileName, 
            @ExportPath = @FullPath, 
            @Status = 'SUCCESS';

    END TRY
    BEGIN CATCH
        INSERT dbo.BoA_AV_JobLog (
            StepName, BatchId, FileName, Message, ErrorNumber, 
            ErrorSeverity, ErrorState, ErrorLine, ErrorProc
        ) 
        VALUES (
            @proc, @BatchId, @FileName, ERROR_MESSAGE(), ERROR_NUMBER(), 
            ERROR_SEVERITY(), ERROR_STATE(), ERROR_LINE(), ERROR_PROCEDURE()
        );

        SET @FileName = ISNULL(@FileName, 'UNKNOWN');
        DECLARE @err_msg NVARCHAR(MAX) = ERROR_MESSAGE();
        
        EXEC dbo.usp_BoA_AV_Req_S05_SendEmail 
            @BatchId = @BatchId, 
            @FileName = @FileName, 
            @ExportPath = NULL, 
            @Status = 'FAILED', 
            @ErrorMessage = @err_msg;
            
        THROW;
    END CATCH
END;
GO
