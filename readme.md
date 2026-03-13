DECLARE @TracePath NVARCHAR(260);

-- Get the path of the current default trace
SELECT @TracePath = path 
FROM sys.traces 
WHERE is_default = 1;

-- String manipulation to grab the base trace file name so we read the entire rollover chain
SET @TracePath = REVERSE(@TracePath);
SET @TracePath = SUBSTRING(@TracePath, CHARINDEX('_', @TracePath), LEN(@TracePath));
SET @TracePath = REVERSE(@TracePath) + '.trc';

-- Query the trace for any modifications to the sa account
SELECT 
    t.StartTime AS [Execution Time],
    te.name AS [Event Name],
    t.LoginName AS [Executed By],
    t.TargetLoginName AS [Target Account],
    t.HostName AS [Originating Host],
    t.ApplicationName AS [Application Used],
    t.ClientProcessID AS [PID]
FROM sys.fn_trace_gettable(@TracePath, DEFAULT) t
INNER JOIN sys.trace_events te ON t.EventClass = te.trace_event_id
WHERE t.TargetLoginName = 'sa'
  AND te.name IN ('Audit Login Change Property Event', 'Audit Addlogin Event')
ORDER BY t.StartTime DESC;
