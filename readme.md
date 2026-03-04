I see exactly what went wrong in your screenshot. The chat interface is aggressively cutting off the right side of the code blocks. `\template.html"` got truncated into `\l"`, and all the long `.Replace()` lines ran completely off your screen, making it impossible to copy. 

I also cannot put this code into a generated Studio Document for you. The document generator strictly reads from your originally uploaded files (like your `audit.txt` and images). It cannot read our chat history. When I triggered it, it had no access to our new code, which is why it spit out an empty template with broken placeholders.

Since we are locked into this chat window, **I have reformatted both files to be completely vertical.** I used PowerShell line-continuations (backticks `` ` ``) so that absolutely no line goes past the middle of your screen. Nothing will truncate, and you can copy this cleanly.

### 1. template.html (Vertical Format)
```html
<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: 'Segoe UI', Tahoma, sans-serif; font-size: 10px; color: #000; }
    .meta-block { margin-bottom: 10px; }
    .meta-block div { margin-bottom: 1px; }
    table { border-collapse: collapse; width: 100%; margin-bottom: 15px; }
    th, td { padding: 3px 5px; text-align: left; border-bottom: 1px solid #eee; }
    .sum-table th { border-bottom: 1px solid #c8d8c8; background-color: #e2f0d9; }
    .sum-table tr:nth-child(even) { background-color: #ffffff; }
    .sum-table tr:nth-child(odd) { background-color: #f4f9f4; }
    .det-header { font-size: 13px; font-weight: bold; margin-bottom: 5px; }
    .det-meta td:first-child { font-weight: bold; width: 90px; }
    .det-table th { background-color: #f0f0f0; }
    .center { text-align: center; }
    .right { text-align: right; font-family: 'Consolas', monospace; }
    .totals td { border-top: 1px solid #333; font-weight: bold; }
    .posted-msg { color: #d9534f; font-weight: bold; margin-bottom: 5px; }
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
            <tr>
                <th>Company</th>
                <th>Batch</th>
                <th>JE</th>
                <th>Reference</th>
                <th>TrxDate</th>
                <th>#Rows</th>
                <th>RunStatus</th>
                <th>Reversal</th>
                <th>Error</th>
            </tr>
        </thead>
        <tbody>
            {{SUMMARY_ROWS}}
        </tbody>
    </table>
    {{DETAIL_TABLES}}
</body>
</html>
@@@SPLIT@@@
<tr>
    <td>{{COMPANY}}</td>
    <td>{{BATCH}}</td>
    <td>{{JE}}</td>
    <td>{{REF}}</td>
    <td>{{TRXDATE}}</td>
    <td>{{ROWS}}</td>
    <td>{{STATUS}}</td>
    <td>{{REVERSAL}}</td>
    <td>{{ERROR}}</td>
</tr>
@@@SPLIT@@@
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
        <tr>
            <th rowspan="2">Account</th>
            <th colspan="3" class="center">Import File</th>
            <th colspan="3" class="center">Dynamics GP</th>
        </tr>
        <tr>
            <th class="right">Debit</th>
            <th class="right">Credit</th>
            <th class="right">Total</th>
            <th class="right">Debit</th>
            <th class="right">Credit</th>
            <th class="right">Total</th>
        </tr>
    </thead>
    <tbody>
        {{DETAIL_ROWS}}
        <tr class="totals">
            <td>Totals</td>
            <td class="right">{{TOT_IMP_DEB}}</td>
            <td class="right">{{TOT_IMP_CRED}}</td>
            <td class="right">{{TOT_IMP_NET}}</td>
            <td class="right">{{TOT_GP_DEB}}</td>
            <td class="right">{{TOT_GP_CRED}}</td>
            <td class="right">{{TOT_GP_NET}}</td>
        </tr>
    </tbody>
</table>
@@@SPLIT@@@
<tr>
    <td>{{ACCOUNT}}</td>
    <td class="right">{{IMP_DEB}}</td>
    <td class="right">{{IMP_CRED}}</td>
    <td class="right">{{IMP_NET}}</td>
    <td class="right">{{GP_DEB}}</td>
    <td class="right">{{GP_CRED}}</td>
    <td class="right">{{GP_NET}}</td>
</tr>
@@@SPLIT@@@
<div class="posted-msg">
    &#9888; Note: This batch has been POSTED. Unposted details unavailable.
</div>
```

### 2. PowerShell Script (Vertical Format, No Array Brackets)
I fixed the `$TemplatePath` so it won't truncate to `\l`, and separated every single `.Replace()` onto its own line so the right side of the text cannot hide off your screen.

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    $TemplatePath = Join-Path $PSScriptRoot "template.html"
    
    if (-not (Test-Path $TemplatePath)) { 
        return "<h3>Template file not found at $TemplatePath</h3>" 
    }
    
    $rawHtml = Get-Content -Path $TemplatePath -Raw
    
    # 1. Unpack templates safely
    $mainTpl, $sumRowTpl, $detBlockTpl, $detRowTpl, $postedMsgTpl = 
        $rawHtml -split '@@@SPLIT@@@'

    $mainTpl      = $mainTpl.Trim()
    $sumRowTpl    = $sumRowTpl.Trim()
    $detBlockTpl  = $detBlockTpl.Trim()
    $detRowTpl    = $detRowTpl.Trim()
    $postedMsgTpl = $postedMsgTpl.Trim()

    # 2. Query Database
    $query = "EXEC dbo.usp_ia_job_report @JobID = '$JobId'"
    try { 
        $auditData = Invoke-Sqlcmd `
            -ServerInstance $sqlconfig.server `
            -Database $sqlconfig.database `
            -TrustServerCertificate `
            -Query $query 
    } catch { 
        return "<h3>Error: $($_.Exception.Message)</h3>" 
    }
    
    if (-not $auditData) { return "<h3>No data for JobID: $JobId</h3>" }

    $firstGlobal = $auditData | Select-Object -First 1
    
    $fileName = "Unknown"
    if ($firstGlobal.FILENAME) { $fileName = $firstGlobal.FILENAME }
    
    $jobResult = "Unknown"
    if ($firstGlobal.STATUS) { $jobResult = $firstGlobal.STATUS }

    $summaryRows = ""
    $detailTables = ""

    $runs = $auditData | Group-Object -Property RunID
    
    foreach ($run in $runs) {
        $group = $run.Group
        $first = $group | Select-Object -First 1
        
        $trxDateStr = $first.TrxDate
        if ($first.TrxDate -is [DateTime]) { 
            $trxDateStr = $first.TrxDate.ToString("yyyy-MM-dd") 
        }
        
        $rowCount = ($group | Measure-Object).Count

        # Replace wrapped safely so it doesn't run off screen
        $summaryRows += $sumRowTpl `
            .Replace('{{COMPANY}}',  "$($first.CompanyA)") `
            .Replace('{{BATCH}}',    "$($first.BatchID)") `
            .Replace('{{JE}}',       "$($first.JournalEntry)") `
            .Replace('{{REF}}',      "$($first.Reference)") `
            .Replace('{{TRXDATE}}',  "$trxDateStr") `
            .Replace('{{ROWS}}',     "$rowCount") `
            .Replace('{{STATUS}}',   "$($first.STATUS)") `
            .Replace('{{REVERSAL}}', "$($first.ReversalFlag)") `
            .Replace('{{ERROR}}',    "$($first.MESSAGE)")

        $isPosted = ($first.STATUS -match 'POSTED') -or 
                    ([string]::IsNullOrWhiteSpace($first.gp_Journal) -and 
                    $first.STATUS -notmatch 'ERROR')
        
        $injectPostedMsg = ""
        if ($isPosted) { $injectPostedMsg = $postedMsgTpl }

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0
        $sumGpDeb = 0;  $sumGpCred = 0;  $sumGpNet = 0
        $detailRowsOut = ""

        $accounts = $group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $accName = ($acc.Group | Select-Object -First 1).Account
            $impDeb = 0; $impCred = 0; $impNet = 0
            $gpDeb = 0;  $gpCred = 0;  $gpNet = 0

            foreach ($row in $acc.Group) {
                if ($row.DebitAmt -isnot [System.DBNull]) { 
                    $impDeb += $row.DebitAmt 
                }
                if ($row.CreditAmt -isnot [System.DBNull]) { 
                    $impCred += $row.CreditAmt 
                }
                if ($row.NetAmt -isnot [System.DBNull]) { 
                    $impNet += $row.NetAmt 
                }
                if ($row.gp_DebitAmount -isnot [System.DBNull]) { 
                    $gpDeb += $row.gp_DebitAmount 
                }
                if ($row.gp_CreditAmount -isnot [System.DBNull]) { 
                    $gpCred += $row.gp_CreditAmount 
                }
            }
            $gpNet = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet
            $sumGpDeb  += $gpDeb;  $sumGpCred  += $gpCred;  $sumGpNet  += $gpNet

            $gpDebStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $gpDeb }
            $gpCredStr = if ($isPosted) { "-" } else { "{0:N2}" -f $gpCred }
            $gpNetStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $gpNet }

            $detailRowsOut += $detRowTpl `
                .Replace('{{ACCOUNT}}',  "$accName") `
                .Replace('{{IMP_DEB}}',  ("{0:N2}" -f $impDeb)) `
                .Replace('{{IMP_CRED}}', ("{0:N2}" -f $impCred)) `
                .Replace('{{IMP_NET}}',  ("{0:N2}" -f $impNet)) `
                .Replace('{{GP_DEB}}',   "$gpDebStr") `
                .Replace('{{GP_CRED}}',  "$gpCredStr") `
                .Replace('{{GP_NET}}',   "$gpNetStr")
        }

        $totGpDebStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpDeb }
        $totGpCredStr = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpCred }
        $totGpNetStr  = if ($isPosted) { "-" } else { "{0:N2}" -f $sumGpNet }

        $detailTables += $detBlockTpl `
            .Replace('{{COMPANY}}',      "$($first.CompanyA)") `
            .Replace('{{JE}}',           "$($first.JournalEntry)") `
            .Replace('{{TRXDATE}}',      "$trxDateStr") `
            .Replace('{{BATCH}}',        "$($first.BatchID)") `
            .Replace('{{REF}}',          "$($first.Reference)") `
            .Replace('{{JOBID}}',        "$JobId") `
            .Replace('{{POSTED_MSG}}',   "$injectPostedMsg") `
            .Replace('{{DETAIL_ROWS}}',  "$detailRowsOut") `
            .Replace('{{TOT_IMP_DEB}}',  ("{0:N2}" -f $sumImpDeb)) `
            .Replace('{{TOT_IMP_CRED}}', ("{0:N2}" -f $sumImpCred)) `
            .Replace('{{TOT_IMP_NET}}',  ("{0:N2}" -f $sumImpNet)) `
            .Replace('{{TOT_GP_DEB}}',   "$totGpDebStr") `
            .Replace('{{TOT_GP_CRED}}',  "$totGpCredStr") `
            .Replace('{{TOT_GP_NET}}',   "$totGpNetStr")
    }

    $finalHtml = $mainTpl `
        .Replace('{{FILENAME}}',      "$fileName") `
        .Replace('{{STATUS}}',        "$jobResult") `
        .Replace('{{JOBID}}',         "$JobId") `
        .Replace('{{SUMMARY_ROWS}}',  "$summaryRows") `
        .Replace('{{DETAIL_TABLES}}', "$detailTables")

    return $finalHtml
}
```
