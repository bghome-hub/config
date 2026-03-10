$Query = "
        SELECT ISNULL(STRING_AGG(CAST(StepName + ' - ' + LogMessage AS NVARCHAR(MAX)), '<br/>&bull; ') WITHIN GROUP (ORDER BY LogId ASC), 'No specific error details found. Check raw logs.') AS LogMessage 
        FROM dbo.IC_SystemLog 
        WHERE JobId = '$JobId' 
          AND LogLevel IN ('FATAL', 'ERROR')
    "
