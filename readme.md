  DECLARE @UserName NVARCHAR(128) = 'Domain\UserName';  -- Replace with the actual username

-- Loop through each database and check for owned objects
DECLARE @SQL NVARCHAR(MAX);

-- Temporary table to store results
CREATE TABLE #OwnedObjects (
    DatabaseName NVARCHAR(128),
    ObjectName NVARCHAR(128),
    ObjectType NVARCHAR(128)
);

DECLARE db_cursor CURSOR FOR
SELECT name
FROM sys.databases
WHERE state_desc = 'ONLINE' AND name NOT IN ('master', 'tempdb', 'model', 'msdb');

OPEN db_cursor;
DECLARE @DatabaseName NVARCHAR(128);

FETCH NEXT FROM db_cursor INTO @DatabaseName;
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = N'
    USE ' + QUOTENAME(@DatabaseName) + ';
    INSERT INTO #OwnedObjects (DatabaseName, ObjectName, ObjectType)
    SELECT ''' + @DatabaseName + ''' AS DatabaseName,
           o.name AS ObjectName,
           o.type_desc AS ObjectType
    FROM sys.objects o
    JOIN sys.database_principals dp ON o.principal_id = dp.principal_id
    WHERE dp.name = @UserName;';

    EXEC sp_executesql @SQL, N'@UserName NVARCHAR(128)', @UserName;

    FETCH NEXT FROM db_cursor INTO @DatabaseName;
END

CLOSE db_cursor;
DEALLOCATE db_cursor;

-- View owned objects
SELECT * FROM #OwnedObjects;

-- Drop temporary table
DROP TABLE #OwnedObjects;
