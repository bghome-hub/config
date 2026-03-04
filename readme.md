Dynamics GP Import Reporting: PowerShell and HTML Templates

<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-size: 10px; }
    table { border-collapse: collapse; width: 100%; margin-bottom: 20px; font-size: 10px; }
    th { background-color: #e2efda; border: 1px solid #ffffff; padding: 4px; text-align: left; font-size: 10px; }
    td { border: 1px solid #ffffff; padding: 4px; font-size: 10px; }
    tr:nth-child(even) { background-color: #f2f2f2; }
    .right { text-align: right; }
    .totals { font-weight: bold; background-color: #ffffff !important; border-top: 1px solid #000000; }
    .posted-msg { color: #000000; font-weight: bold; margin-top: 10px; }
</style>
</head>
<body>
    <h3>Import Job Summary</h3>
    <table>
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
    {{DETAIL_SECTIONS}}
    {{POSTED_FOOTER}}
</body>
</html>

@@@SPLIT@@@

<div class="detail-section">
    <h3>{{COMPANY_ID}}</h3>
    <table>
        <tr><td><b>Journal Entry</b></td><td>{{JE}}</td></tr>
        <tr><td><b>TxDate</b></td><td>{{TRX_DATE}}</td></tr>
        <tr><td><b>BatchID</b></td><td>{{BATCH_ID}}</td></tr>
        <tr><td><b>Reference</b></td><td>{{REFERENCE}}</td></tr>
        <tr><td><b>JobID</b></td><td>{{JOB_ID}}</td></tr>
    </table>
    <table>
        <thead>
            <tr>
                <th>Account</th>
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
                <td class="right">{{TOT_IMP_CRE}}</td>
                <td class="right">{{TOT_NET}}</td>
            </tr>
        </tbody>
    </table>
</div>

@@@SPLIT@@@

<tr>
    <td>{{COMPANY}}</td>
    <td>{{BATCH}}</td>
    <td>{{JE}}</td>
    <td>{{REFERENCE}}</td>
    <td>{{TRXDATE}}</td>
    <td>{{ROWS}}</td>
    <td>{{STATUS}}</td>
    <td>{{REVERSAL}}</td>
    <td>{{ERROR}}</td>
</tr>

@@@SPLIT@@@

<tr>
    <td>{{ACCOUNT}}</td>
    <td class="right">{{IMP_DEB}}</td>
    <td class="right">{{IMP_CRE}}</td>
    <td class="right">{{IMP_NET}}</td>
</tr>

@@@SPLIT@@@

<div class="posted-msg">&#9888; Note: This batch has been POSTED. Unposted Dynamic GP data remains in the audit tables.</div>


function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [Guid]$JobId
    )

    $TemplatePath = "$PSScriptRoot\template.html"
    if (-not (Test-Path $TemplatePath)) {
        return "<h3>Template file not found at $TemplatePath</h3>"
    }

    $html = Get-Content -Path $TemplatePath -Raw
    $parts = $html -split '@@@SPLIT@@@'
    
    $layoutTpl = $parts[0].Trim()
    $detailContainerTpl = $parts[1].Trim()
    $summaryRowTpl = $parts[2].Trim()
    $detailRowTpl = $parts[3].Trim()
    $postedFooterTpl = $parts[4].Trim()

    # Define complex date logic and fetch headers from Source Context (Pasted Text)
    $qHeaders = @"
SELECT 
    r.RunID, 
    r.JobID, 
    r.FileName, 
    r.CompanyA, 
    r.BatchID, 
    r.JournalEntry, 
    r.Reference, 
    r.Status, 
    r.Message, 
    r.RowCountLine, 
    r.JobResult, 
    r.ReversalFlag, 
    TxDate = ISNULL(r.TrxDateBasis, (SELECT MIN(il.TrxDate) FROM dbo.IA_Line il WHERE il.RunID = r.RunID))
FROM dbo.IA_Run r 
WHERE r.JobID = '$JobId' 
ORDER BY r.StartedDT ASC;
"@
    $headerData = Invoke-Sqlcmd -Query $qHeaders

    $summaryRows = ""
    $detailSections = ""
    $isPosted = $false

    foreach ($header in $headerData) {
        # Check if any part of the job is posted for the footer warning
        if ($header.Status -eq "POSTED") { $isPosted = $true }

        # Generate Summary Table Row
        $sRow = $summaryRowTpl.Replace('{{COMPANY}}', $header.CompanyA)
        $sRow = $sRow.Replace('{{BATCH}}', $header.BatchID)
        $sRow = $sRow.Replace('{{JE}}', $header.JournalEntry)
        $sRow = $sRow.Replace('{{REFERENCE}}', $header.Reference)
        $sRow = $sRow.Replace('{{TRXDATE}}', (Get-Date $header.TxDate -Format "yyyy-MM-dd"))
        $sRow = $sRow.Replace('{{ROWS}}', $header.RowCountLine)
        $sRow = $sRow.Replace('{{STATUS}}', $header.Status)
        $sRow = $sRow.Replace('{{REVERSAL}}', $header.ReversalFlag)
        $sRow = $sRow.Replace('{{ERROR}}', $header.Message)
        $summaryRows += $sRow

        # Fetch Lines using Source Context field names (1.png / audit.txt)
        $rid = $header.RunID
        $qLines = @"
SELECT 
    il.Account, 
    SUM(ISNULL(il.Debit,0)) AS DebitAmt, 
    SUM(ISNULL(il.Credit,0)) AS CreditAmt, 
    SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0)) AS NetAmt 
FROM dbo.IA_Line il
WHERE il.RunID = '$rid' 
GROUP BY il.Account 
HAVING ABS(SUM(ISNULL(il.Debit,0)) + SUM(ISNULL(il.Credit,0))) > 0 
   OR ABS(SUM(ISNULL(il.Debit,0) - ISNULL(il.Credit,0))) > 0.00001 
ORDER BY il.Account;
"@
        $lineData = Invoke-Sqlcmd -Query $qLines

        $detailRows = ""
        $runDebit = 0
        $runCredit = 0
        $runNet = 0

        foreach ($line in $lineData) {
            $dRow = $detailRowTpl.Replace('{{ACCOUNT}}', $line.Account)
            $dRow = $dRow.Replace('{{IMP_DEB}}', $line.DebitAmt.ToString('N2'))
            $dRow = $dRow.Replace('{{IMP_CRE}}', $line.CreditAmt.ToString('N2'))
            $dRow = $dRow.Replace('{{IMP_NET}}', $line.NetAmt.ToString('N2'))
            $detailRows += $dRow
            
            $runDebit += $line.DebitAmt
            $runCredit += $line.CreditAmt
            $runNet += $line.NetAmt
        }

        # Build Detail Section
        $section = $detailContainerTpl.Replace('{{COMPANY_ID}}', $header.CompanyA)
        $section = $section.Replace('{{JE}}', $header.JournalEntry)
        $section = $section.Replace('{{TRX_DATE}}', (Get-Date $header.TxDate -Format "yyyy-MM-dd"))
        $section = $section.Replace('{{BATCH_ID}}', $header.BatchID)
        $section = $section.Replace('{{REFERENCE}}', $header.Reference)
        $section = $section.Replace('{{JOB_ID}}', $JobId.ToString())
        $section = $section.Replace('{{DETAIL_ROWS}}', $detailRows)
        $section = $section.Replace('{{TOT_IMP_DEB}}', $runDebit.ToString('N2'))
        $section = $section.Replace('{{TOT_IMP_CRE}}', $runCredit.ToString('N2'))
        $section = $section.Replace('{{TOT_NET}}', $runNet.ToString('N2'))
        
        $detailSections += $section
    }

    # Final Assembly
    $finalContent = $layoutTpl.Replace('{{SUMMARY_ROWS}}', $summaryRows)
    $finalContent = $finalContent.Replace('{{DETAIL_SECTIONS}}', $detailSections)
    
    if ($isPosted) {
        $finalContent = $finalContent.Replace('{{POSTED_FOOTER}}', $postedFooterTpl)
    } else {
        $finalContent = $finalContent.Replace('{{POSTED_FOOTER}}', '')
    }

    return $finalContent
}
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
