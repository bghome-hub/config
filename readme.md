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
