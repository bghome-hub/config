<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #333; background-color: #f4f7f6; margin: 0; padding: 20px; }
        .container { background-color: #fff; padding: 20px; border-top: 5px solid #e74c3c; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); max-width: 800px; margin: auto; }
        h2 { color: #e74c3c; margin-top: 0; border-bottom: 2px solid #ecf0f1; padding-bottom: 10px; }
        .meta-data { background: #f9fbfd; padding: 15px; border: 1px solid #e0e6ed; margin-bottom: 20px; font-size: 14px; }
        .error-box { background: #f8d7da; color: #721c24; padding: 15px; border: 1px solid #f5c6cb; border-radius: 4px; font-family: 'Courier New', Courier, monospace; font-size: 14px; line-height: 1.5; }
    </style>
</head>
<body>
    <div class="container">
        <h2>Intracompany Upload Failed</h2>
        
        <div class="meta-data">
            <strong>Job ID:</strong> {{JobId}}<br>
            <strong>File:</strong> {{FileName}}<br>
            <strong>Time:</strong> {{RunDate}}
        </div>

        <h3>Error Details:</h3>
        <div class="error-box">
            {{ErrorMessage}}
        </div>
        
        <p><em>Please correct the file and drop it back into the inbound folder.</em></p>
    </div>
</body>
</html>





# ==============================================================================
# 3. REPORTING & PRESENTATION ROUTING
# ==============================================================================

# 1. Ask SQL for the final status and any fatal error messages
$StatusQuery = @"
SELECT 
    JobStatus,
    (SELECT TOP 1 LogMessage FROM dbo.IC_SystemLog WHERE JobId = '$JobId' AND LogLevel = 'FATAL' ORDER BY LogId DESC) AS FatalError
FROM dbo.IC_ImportJob 
WHERE JobId = '$JobId'
"@

$JobState = Invoke-Sqlcmd -ServerInstance $ServerInstance -Database $Database -Query $StatusQuery
$CleanState = if ($null -ne $JobState.JobStatus) { $JobState.JobStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }
$FatalError = $JobState.FatalError

if ($CleanState -eq 'FAILED' -and [string]::IsNullOrWhiteSpace($FatalError) -eq $false) {
    # ---------------------------------------------------------
    # ROUTE A: FILE FAILED (Bad Headers, Deadlock, etc.)
    # ---------------------------------------------------------
    Write-Host "File level failure detected. Generating Error Email."
    
    $HtmlTemplate = Get-Content "C:\Scripts\ErrorTemplate.html" -Raw
    $FinalHtml = $HtmlTemplate -replace '{{JobId}}', $JobId `
                               -replace '{{FileName}}', (Split-Path $CsvPath -Leaf) `
                               -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                               -replace '{{ErrorMessage}}', $FatalError

    $FinalHtml | Set-Content $OutputPath
    Move-Item -Path $CsvPath -Destination $ErrorArchive -Force
    Write-Host "File moved to Error folder."

} else {
    # ---------------------------------------------------------
    # ROUTE B: NORMAL RECONCILIATION REPORT
    # ---------------------------------------------------------
    Write-Host "Generating standard reconciliation report."
    
    $Query = "EXEC dbo.usp_IC_ReportJobReconciliation @JobId = '$JobId'"
    $Details = Invoke-Sqlcmd -ServerInstance $ServerInstance -Database $Database -Query $Query

    # ... [PASTE YOUR EXISTING GROUPING & DETAIL LOOP CODE HERE] ...

    $HtmlTemplate = Get-Content "C:\Scripts\ReportTemplate.html" -Raw
    $FinalHtml = $HtmlTemplate -replace '{{JobId}}', $JobId `
                               -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                               -replace '{{SummaryRows}}', $SummaryHtml `
                               -replace '{{DetailRows}}', $DetailHtml

    $FinalHtml | Set-Content $OutputPath
    Move-Item -Path $CsvPath -Destination $SuccessArchive -Force
    Write-Host "File moved to Archive folder."
}

# -> Execute your Send-MailMessage function here using $OutputPath



