ELSE
        BEGIN
            -- The file structure is valid, even if the individual batches failed business rules.
            -- Keep the pipeline moving so the Reconciliation Report generates.
            UPDATE dbo.IC_ImportJob SET JobStatus = 'VALIDATED' WHERE JobId = @JobId;
            SET @LogMsg = 'Validation complete. Zero batches survived, moving to reconciliation.';
            INSERT INTO dbo.IC_SystemLog (JobId, LogLevel, StepName, LogMessage) VALUES (@JobId, 'WARN', @StepName, @LogMsg);
        END
