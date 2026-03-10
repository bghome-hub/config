$Query = "SELECT TOP 1 LogMessage FROM dbo.IC_SystemLog WHERE JobId = '$JobId' AND LogLevel IN ('FATAL', 'ERROR') ORDER BY LogId ASC"
