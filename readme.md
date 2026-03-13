SELECT 
    j.name AS [Job Name],
    jh.step_name AS [Step Name],
    jh.run_date AS [Run Date (YYYYMMDD)],
    STUFF(STUFF(RIGHT('000000' + CAST(jh.run_time AS VARCHAR(6)), 6), 3, 0, ':'), 6, 0, ':') AS [Run Time],
    jh.run_status AS [Status (1=Success)],
    js.subsystem AS [Subsystem],
    js.command AS [Command Executed]
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
INNER JOIN msdb.dbo.sysjobsteps js ON jh.job_id = js.job_id AND jh.step_id = js.step_id
WHERE jh.run_date = 20260308 -- Specifically Sunday, March 8th
  AND (
      j.name LIKE '%sync%' 
      OR j.name LIKE '%login%' 
      OR js.subsystem IN ('PowerShell', 'CmdExec')
      OR js.command LIKE '%ALTER LOGIN%'
      OR js.command LIKE '%dbatools%'
  )
ORDER BY jh.run_time DESC;
