SELECT 
    name AS [Account Name],
    type_desc AS [Account Type],
    is_disabled AS [Is Disabled],
    create_date AS [Creation Date],
    modify_date AS [Last Modified Date],
    default_database_name AS [Default Database]
FROM sys.server_principals
WHERE create_date >= '2026-03-08'
  AND type IN ('S', 'U', 'G') -- S = SQL Login, U = Windows User, G = Windows Group
  AND name NOT LIKE 'NT SERVICE\%'
  AND name NOT LIKE 'NT AUTHORITY\%'
  AND name NOT LIKE '##MS_%' -- Filter out internal certificate-mapped logins
ORDER BY create_date DESC;
