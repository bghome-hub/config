DECLARE @CurrentTracePath NVARCHAR(260);
DECLARE @BaseTracePath NVARCHAR(260);

-- Get the exact path of the currently active default trace
SELECT @CurrentTracePath = path 
FROM sys.traces 
WHERE is_default = 1;

-- Isolate the directory and append the base 'log.trc' to read all rollover files
SET @BaseTracePath = LEFT(@CurrentTracePath, LEN(@CurrentTracePath) - PATINDEX('%\%', REVERSE(@CurrentTracePath)) + 1) + 'log.trc';

-- Query the entire trace chain for modifications to the sa account
SELECT 
    t.StartTime AS [Execution Time],
    te.name AS [Event Name],
    t.LoginName AS [Executed By],
    t.TargetLoginName AS [Target Account],
    t.HostName AS [Originating Host],
    t.ApplicationName AS [Application Used]
FROM sys.fn_trace_gettable(@BaseTracePath, DEFAULT) t
INNER JOIN sys.trace_events te ON t.EventClass = te.trace_event_id
WHERE t.TargetLoginName = 'sa'
  AND te.name IN ('Audit Login Change Property Event', 'Audit Addlogin Event')
ORDER BY t.StartTime DESC;
