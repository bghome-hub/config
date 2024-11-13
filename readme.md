DECLARE @LoginName SYSNAME = 'YourLoginName'; -- Replace with your login name
DECLARE @DBName SYSNAME;
DECLARE @SQL NVARCHAR(MAX);

DECLARE db_cursor CURSOR FOR
SELECT name FROM sys.databases
WHERE state_desc = 'ONLINE' AND database_id > 4;  -- Exclude system databases

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DBName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = N'USE [' + QUOTENAME(@DBName) + '];

    SELECT
        DB_NAME() AS DatabaseName,
        s.name AS SchemaName,
        o.name AS ObjectName,
        o.type_desc AS ObjectType
    FROM sys.objects o
    INNER JOIN sys.schemas s ON o.schema_id = s.schema_id
    INNER JOIN sys.database_principals dp ON s.principal_id = dp.principal_id
    WHERE dp.sid = SUSER_SID(''' + @LoginName + ''');';

    EXEC sp_executesql @SQL;

    FETCH NEXT FROM db_cursor INTO @DBName;
END

CLOSE db_cursor;
DEALLOCATE db_cursor;
