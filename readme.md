CASE 
                    WHEN i.HeaderStatus IN (''FAILED'', ''PENDING'', ''SKIPPED'') THEN 
                        CAST(''NOT SUBMITTED - '' + ISNULL((
                            SELECT TOP 1 log.LogMessage 
                            FROM DYNAMICS.dbo.IC_SystemLog log 
                            INNER JOIN DYNAMICS.dbo.IC_CompanyHeader ch ON log.HeaderId = ch.HeaderId
                            WHERE log.JobId = @pJobId AND ch.Company = @pCompany AND log.LogLevel = ''ERROR''
                        ), ''Check logs'') AS VARCHAR(500))
                    WHEN g.GpState IS NULL THEN ''DELETED FROM GP''
                    ELSE SUBSTRING(g.GpState, 3, 20) -- Strip the sorting prefix
                END AS GpState,
