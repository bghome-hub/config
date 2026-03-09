    -- ==============================================================================
    -- CATCH EARLY FATAL FAILURES (e.g., Bad Headers, Empty File)
    -- ==============================================================================
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

    -- ... (The rest of your normal recon code continues here) ...
