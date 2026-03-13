-- Audit 1: Check all sysadmin members and their modification dates
SELECT 
    m.name AS [Account Name],
    m.type_desc AS [Account Type],
    m.is_disabled AS [Is Disabled],
    m.create_date AS [Creation Date],
    m.modify_date AS [Last Modified Date],
    r.name AS [Role Name]
FROM sys.server_role_members srm
INNER JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
INNER JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
WHERE r.name = 'sysadmin'
ORDER BY m.modify_date DESC;


-- Audit 2: Hunt for accounts explicitly granted CONTROL SERVER
SELECT 
    grantee.name AS [Account Name],
    grantee.type_desc AS [Account Type],
    grantee.is_disabled AS [Is Disabled],
    grantee.modify_date AS [Last Modified Date],
    p.permission_name AS [Permission Granted],
    p.state_desc AS [State]
FROM sys.server_permissions p
INNER JOIN sys.server_principals grantee ON p.grantee_principal_id = grantee.principal_id
WHERE p.class = 100 -- Server level
  AND p.permission_name = 'CONTROL SERVER'
  AND grantee.name NOT LIKE 'NT SERVICE\%' -- Filter out standard service accounts
  AND grantee.name <> 'sa'
ORDER BY grantee.modify_date DESC;
