You are completely right. Regex is completely unreadable and a nightmare to maintain. If someone else (or you in 6 months) looks at that script, they will have no idea what those `(?s)` regex blocks are doing.

We can rip the regex out completely. The absolute easiest, cleanest way to do this in one file is to just use a simple `<!--===SPLIT===-->` tag between your sections, and let PowerShell chop the file into an array.

Here is the final, regex-free setup. 

### 1. `template.html`
Save this file. It is the exact same compressed layout, but instead of complex hidden regex tags at the bottom, it just uses `<!--===SPLIT===-->` to separate the chunks.

```html
<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: 'Segoe UI', Tahoma, Arial, sans-serif; font-size: 10px; color: #000; }
    .meta-block { margin-bottom: 10px; }
    .meta-block div { margin-bottom: 1px; }
    
    table { border-collapse: collapse; width: 100%; margin-bottom: 15px; }
    th, td { padding: 3px 5px; text-align: left; border-bottom: 1px solid #eee; }
    
    .sum-table th { border-bottom: 1px solid #c8d8c8; background-color: #e2f0d9; font-weight: bold; }
    .sum-table tr:nth-child(even) { background-color: #ffffff; }
    .sum-table tr:nth-child(odd) { background-color: #f4f9f4; }
    
    .det-header { font-size: 13px; font-weight: bold; margin-bottom: 5px; border-bottom: 1px solid #ccc; padding-bottom: 2px;}
    .det-meta { border: none; margin-bottom: 5px; width: auto; }
    .det-meta td { padding: 1px 10px 1px 0; border: none; }
    .det-meta td:first-child { font-weight: bold; width: 90px; }
    
    .det-table th { background-color: #f0f0f0; border-bottom: 1px solid #ccc; font-weight: bold; }
    .det-table th.center { text-align: center; }
    .det-table th.right, .det-table td.right { text-align: right; font-family: 'Consolas', monospace; }
    .det-table tr.totals td { border-top: 1px solid #333; font-weight: bold; background-color: #fafafa;}
    
    .posted-msg { color: #d9534f; font-weight: bold; font-size: 10px; margin-bottom: 5px; }
</style>
</head>
<body>
    <div class="meta-block">
        <div>Filename: <strong>{{FILENAME}}</strong></div>
        <div>Result: <strong>{{STATUS}}</strong></div>
        <div>Job ID: <strong>{{JOBID}}</strong></div>
    </div>

    <table class="sum-table">
        <thead>
            <tr><th>Company</th><th>Batch</th><th>JE</th><th>Reference</th><th>TrxDate</th><th>#Rows</th><th>RunStatus</th><th>Reversal</th><th>Error</th></tr>
        </thead>
        <tbody>
            {{SUMMARY_ROWS}}
        </tbody>
    </table>

    {{DETAIL_TABLES}}
</body>
</html>
<!--===SPLIT===-->
<tr><td>{{COMPANY}}</td><td>{{BATCH}}</td><td>{{JE}}</td><td>{{REF}}</td><td>{{TRXDATE}}</td><td>{{ROWS}}</td><td>{{STATUS}}</td><td>{{REVERSAL}}</td><td>{{ERROR}}</td></tr>
<!--===SPLIT===-->
<div class="det-header">{{COMPANY}}</div>
<table class="det-meta">
    <tr><td>Journal Entry</td><td>{{JE}}</td></tr>
    <tr><td>TxDate</td><td>{{TRXDATE}}</td></tr>
    <tr><td>BatchID</td><td>{{BATCH}}</td></tr>
    <tr><td>Reference</td><td>{{REF}}</td></tr>
    <tr><td>JobID</td><td>{{JOBID}}</td></tr>
</table>
{{POSTED_MSG}}
<table class="det-table">
    <thead>
        <tr><th rowspan="2">Account</th><th colspan="3" class="center">Import File</th><th colspan="3" class="center">Dynamics GP</th></tr>
        <tr><th class="right">Debit</th><th class="right">Credit</th><th class="right">Total</th><th class="right">Debit</th><th class="right">Credit</th><th class="right">Total</th></tr>
    </thead>
    <tbody>
        {{DETAIL_ROWS}}
        <tr class="totals"><td>Totals</td><td class="right">{{TOT_IMP_DEB}}</td><td class="right">{{TOT_IMP_CRED}}</td><td class="right">{{TOT_IMP_NET}}</td><td class="right">{{TOT_GP_DEB}}</td><td class="right">{{TOT_GP_CRED}}</td><td class="right">{{TOT_GP_NET}}</td></tr>
    </tbody>
</table>
<!--===SPLIT===-->
<tr><td>{{ACCOUNT}}</td><td class="right">{{IMP_DEB}}</td><td class="right">{{IMP_CRED}}</td><td class="right">{{IMP_NET}}</td><td class="right">{{GP_DEB}}</td><td class="right">{{GP_CRED}}</td><td class="right">{{GP_NET}}</td></tr>
<!--===SPLIT===-->
<div class="posted-msg">&#9888; Note: This batch has been POSTED. Unposted Dynamics GP details are no longer available.</div>
```

### 2. The PowerShell Script (Regex removed)
Look how clean the extraction block is now. It just splits the file into an array of 5 items. No regex, no string manipulation, and still zero HTML inside the script. It uses your requested `Invoke-Sqlcmd` and `usp_ia_job_report`.

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    $TemplatePath = "$PSScriptRoot\template.html"
    if (-not (Test-Path $TemplatePath)) { return "<h3>Template file not found at $TemplatePath</h3>" }
    
    # 1. Read the template and split it into an array (No Regex)
    $rawHtml = Get-Content -Path $TemplatePath -Raw
    $tplArray = $rawHtml -split '<!--===SPLIT===-->'

    $mainTpl      = $tplArray.Trim()
    $sumRowTpl    = $tplArray.Trim()
    $detBlockTpl  = $tplArray.Trim()
    $detRowTpl    = $tplArray.Trim()
    $postedMsgTpl = $tplArray.Trim()

    # 2. Query Database
    $query = "EXEC dbo.usp_ia_job_report @JobID = '$JobId'"
    try { 
        $auditData = Invoke-Sqlcmd -ServerInstance $sqlconfig.server -Database $sqlconfig.database -TrustServerCertificate -Query $query 
    } catch { 
        return "<h3 style='color:red;'>Error retrieving data: $($_.Exception.Message)</h3>" 
    }
    
    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    $firstGlobal = $auditData | Select-Object -First 1
    $fileName = if ($firstGlobal.FILENAME) { $firstGlobal.FILENAME } else { "Unknown" }
    $jobResult = if ($firstGlobal.STATUS) { $firstGlobal.STATUS } else { "Unknown" }

    $summaryRows = ""
    $detailTables = ""

    # Group runs for processing
    $runs = $auditData | Group-Object -Property RunID
    
    foreach ($run in $runs) {
        $group = $run.Group
        $first = $group | Select-Object -First 1
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }
        $rowCount = ($group | Measure-Object).Count

        # --- Generate Summary Row ---
        $summaryRows += $sumRowTpl.Replace('{{COMPANY}}', $first.CompanyA).Replace('{{BATCH}}', $first.BatchID).Replace('{{JE}}', $first.JournalEntry).Replace('{{REF}}', $first.Reference).Replace('{{TRXDATE}}', $trxDateStr).Replace('{{ROWS}}', $rowCount).Replace('{{STATUS}}', $first.STATUS).Replace('{{REVERSAL}}', $first.ReversalFlag).Replace('{{ERROR}}', $first.MESSAGE)

        # --- Handle POSTED Logic ---
        $isPosted = ($first.STATUS -match 'POSTED') -or ([string]::IsNullOrWhiteSpace($first.gp_Journal) -and $first.STATUS -notmatch 'ERROR')
        $injectPostedMsg = if ($isPosted) { $postedMsgTpl } else { "" }

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0
        $detailRowsOut = ""

        # --- Loop Lines & Safely Sum ---
        $accounts = $group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $accName = ($acc.Group | Select-Object -First 1).Account
            $impDeb = 0; $impCred = 0; $impNet = 0
            $gpDeb = 0; $gpCred = 0; $gpNet = 0

            foreach ($row in $acc.Group) {
                if ($null -ne $row.DebitAmt -and $row.DebitAmt -isnot [System.DBNull]) { $impDeb += $row.DebitAmt }
                if ($null -ne $row.CreditAmt -and $row.CreditAmt -isnot [System.DBNull]) { $impCred += $row.CreditAmt }
                if ($null -ne $row.NetAmt -and $row.NetAmt -isnot [System.DBNull]) { $impNet += $row.NetAmt }
                
                if ($null -ne $row.gp_DebitAmount -and $row.gp_DebitAmount -isnot [System.DBNull]) { $gpDeb += $row.gp_DebitAmount }
                if ($null -ne $row.gp_CreditAmount -and $row.gp_CreditAmount -isnot [System.DBNull]) { $gpCred += $row.gp_CreditAmount }
            }
            $gpNet = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet

            # Format amounts or substitute hyphens if posted
            $gpDebStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $gpDeb }
            $gpCredStr = if ($isPosted) { "-" } else { "{0:N2}" -f $gpCred }
            $gpNetStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $gpNet }

            # Inject row data
            $detailRowsOut += $detRowTpl.Replace('{{ACCOUNT}}', $accName).Replace('{{IMP_DEB}}', ("{0:N2}" -f $impDeb)).Replace('{{IMP_CRED}}', ("{0:N2}" -f $impCred)).Replace('{{IMP_NET}}', ("{0:N2}" -f $impNet)).Replace('{{GP_DEB}}', $gpDebStr).Replace('{{GP_CRED}}', $gpCredStr).Replace('{{GP_NET}}', $gpNetStr)
        }

        # Format Subtotals
        $totGpDebStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpDeb }
        $totGpCredStr = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpCred }
        $totGpNetStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpNet }

        # --- Generate Entire Detail Block ---
        $detailTables += $detBlockTpl.Replace('{{COMPANY}}', $first.CompanyA).Replace('{{JE}}', $first.JournalEntry).Replace('{{TRXDATE}}', $trxDateStr).Replace('{{BATCH}}', $first.BatchID).Replace('{{REF}}', $first.Reference).Replace('{{JOBID}}', $JobId.ToString()).Replace('{{POSTED_MSG}}', $injectPostedMsg).Replace('{{DETAIL_ROWS}}', $detailRowsOut).Replace('{{TOT_IMP_DEB}}', ("{0:N2}" -f $sumImpDeb)).Replace('{{TOT_IMP_CRED}}', ("{0:N2}" -f $sumImpCred)).Replace('{{TOT_IMP_NET}}', ("{0:N2}" -f $sumImpNet)).Replace('{{TOT_GP_DEB}}', $totGpDebStr).Replace('{{TOT_GP_CRED}}', $totGpCredStr).Replace('{{TOT_GP_NET}}', $totGpNetStr)
    }

    # 3. Final Output Injection
    $finalHtml = $mainTpl.Replace('{{FILENAME}}', $fileName).Replace('{{STATUS}}', $jobResult).Replace('{{JOBID}}', $JobId.ToString()).Replace('{{SUMMARY_ROWS}}', $summaryRows).Replace('{{DETAIL_TABLES}}', $detailTables)

    return $finalHtml
}
```
