DECLARE @LoginName SYSNAME = 'YourLoginName'; -- Replace with your login name
DECLARE @LoginSID VARBINARY(85);

-- Get the SID of the specified login
SELECT @LoginSID = sid FROM sys.server_principals WHERE name = @LoginName;

DECLARE @DBName SYSNAME;
DECLARE @SQL NVARCHAR(MAX);

-- Cursor to iterate over all user databases
DECLARE db_cursor CURSOR FOR
SELECT name FROM sys.databases
WHERE state_desc = 'ONLINE' AND database_id > 4;  -- Exclude system databases

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DBName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = N'
    USE ' + QUOTENAME(@DBName) + ';

    DECLARE @UserID INT = (SELECT principal_id FROM sys.database_principals WHERE sid = @LoginSID);

    IF @UserID IS NOT NULL
    BEGIN
        -- Objects directly owned by the user
        SELECT
            DB_NAME() AS DatabaseName,
            dp.name AS DatabaseUserName,
            s.name AS SchemaName,
            o.name AS ObjectName,
            o.type_desc AS ObjectType,
            ''Object Owner'' AS OwnershipType
        FROM sys.objects o
        INNER JOIN sys.schemas s ON o.schema_id = s.schema_id
        INNER JOIN sys.database_principals dp ON o.principal_id = dp.principal_id
        WHERE o.principal_id = @UserID

        UNION ALL

        -- Objects in schemas owned by the user
        SELECT
            DB_NAME() AS DatabaseName,
            dp.name AS DatabaseUserName,
            s.name AS SchemaName,
            o.name AS ObjectName,
            o.type_desc AS ObjectType,
            ''Schema Owner'' AS OwnershipType
        FROM sys.objects o
        INNER JOIN sys.schemas s ON o.schema_id = s.schema_id
        INNER JOIN sys.database_principals dp ON s.principal_id = dp.principal_id
        WHERE s.principal_id = @UserID;
    END
    ';

    EXEC sp_executesql @SQL, N'@LoginSID VARBINARY(85)', @LoginSID;

    FETCH NEXT FROM db_cursor INTO @DBName;
END

CLOSE db_cursor;
DEALLOCATE db_cursor;
