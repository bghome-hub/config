USE master;
GO

SELECT 
    [Current LSN],
    [Operation],
    [Transaction Name],
    [Transaction SID],
    SUSER_SNAME([Transaction SID]) AS [Executed By],
    [Begin Time],
    [End Time]
FROM fn_dblog(NULL, NULL)
WHERE [Transaction Name] LIKE '%LOGIN%' 
   OR [Transaction Name] LIKE '%PASSWORD%'
   OR [Transaction Name] = 'user transaction';
