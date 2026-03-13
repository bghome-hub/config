SELECT 
    sp.name AS [Login Name],
    sp.is_disabled AS [Is Disabled],
    sl.is_policy_checked AS [Enforce Password Policy],
    sl.is_expiration_checked AS [Enforce Password Expiration],
    sp.default_database_name AS [Default Database],
    sp.modify_date AS [Last Modified Date]
FROM sys.server_principals sp
INNER JOIN sys.sql_logins sl ON sp.principal_id = sl.principal_id
WHERE sp.name = 'sa';
