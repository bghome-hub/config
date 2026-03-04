Final Code Fix: Dynamics GP Import Reporting Utility

<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-size: 10pt; line-height: 1.5; color: #333; }
    table { border-collapse: collapse; width: 100%; margin-top: 10px; margin-bottom: 25px; }
    th { background-color: #E2EFDA; border: 1px solid #A9D08E; padding: 6px; text-align: left; font-size: 9pt; }
    td { border: 1px solid #A9D08E; padding: 6px; font-size: 9pt; vertical-align: top; }
    .right { text-align: right; }
    .totals { font-weight: bold; background-color: #F2F2F2; }
    .posted-msg { border-left: 4px solid #2E75B6; background-color: #DDEBF7; padding: 10px; margin-bottom: 15px; font-weight: bold; color: #1E4E79; }
    h2 { color: #2E75B6; border-bottom: 2px solid #2E75B6; padding-bottom: 5px; margin-top: 30px; }
    .meta { margin-bottom: 20px; }
</style>
</head>
<body>
    <div class="meta">
        <strong>Filename:</strong> {{FILENAME}}<br>
        <strong>Status:</strong> {{STATUS}}<br>
        <strong>Job ID:</strong> {{JOBID}}
    </div>
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
</body>
</html>
@@@SPLIT@@@
<tr>
    <td>{{COMPANY}}</td>
    <td>{{BATCH}}</td>
    <td>{{JE}}</td>
    <td>{{REFERENCE}}</td>
    <td>{{TRXDATE}}</td>
    <td class="right">{{ROWS}}</td>
    <td>{{STATUS}}</td>
    <td style="text-align:center">{{REVERSAL}}</td>
    <td style="color:red">{{ERROR}}</td>
</tr>
@@@SPLIT@@@
<h2>{{COMPANY}} - Detail Report</h2>
{{POSTED_BANNER}}
<table style="width:auto">
    <tr><th>Journal Entry</th><td>{{JE}}</td></tr>
    <tr><th>TrxDate</th><td>{{TRXDATE}}</td></tr>
    <tr><th>BatchID</th><td>{{BATCH}}</td></tr>
    <tr><th>Reference</th><td>{{REFERENCE}}</td></tr>
    <tr><th>RunID</th><td>{{RUNID}}</td></tr>
</table>
<table>
    <thead>
        <tr>
            <th>Account</th>
            <th class="right">Import Debit</th>
            <th class="right">Import Credit</th>
            <th class="right">Import Net</th>
            <th class="right">GP Debit</th>
            <th class="right">GP Credit</th>
            <th class="right">GP Net</th>
        </tr>
    </thead>
    <tbody>
        {{DETAIL_ROWS}}
        <tr class="totals">
            <td>Totals</td>
            <td class="right">{{TOT_IMP_DEB}}</td>
            <td class="right">{{TOT_IMP_CRE}}</td>
            <td class="right">{{TOT_IMP_NET}}</td>
            <td class="right">{{TOT_GP_DEB}}</td>
            <td class="right">{{TOT_GP_CRE}}</td>
            <td class="right">{{TOT_GP_NET}}</td>
        </tr>
    </tbody>
</table>
@@@SPLIT@@@
<tr>
    <td>{{ACCOUNT}}</td>
    <td class="right">{{IMP_DEB}}</td>
    <td class="right">{{IMP_CRE}}</td>
    <td class="right">{{IMP_NET}}</td>
    <td class="right">{{GP_DEB}}</td>
    <td class="right">{{GP_CRE}}</td>
    <td class="right">{{GP_NET}}</td>
</tr>
@@@SPLIT@@@
<div class="posted-msg">&#9888; Note: This batch has been POSTED. Unposted Dynamics GP values are shown for comparison.</div>

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

    $templateRaw = Get-Content -Path $TemplatePath -Raw
    $parts = $templateRaw -split '@@@SPLIT@@@'

    # Execute Audit Reporting Procedure via Invoke-Sqlcmd
    try {
        $results = Invoke-Sqlcmd -Query "EXEC dbo.usp_ia_job_report @JobID = '$JobId'" -ErrorAction Stop
    }
    catch {
        return "<h3>Error executing report query: $($_.Exception.Message)</h3>"
    }

    if ($null -eq $results -or $results.Count -eq 0) {
        return "<h3>No data found for JobID: $JobId</h3>"
    }

    $summaryRows = ""
    $detailSections = ""

    # Global Metadata from first record for main header
    $first = $results[0]
    $htmlMain = $parts[0].Replace('{{FILENAME}}', $first.FileName) `
                         .Replace('{{STATUS}}', $first.JobResult) `
                         .Replace('{{JOBID}}', $first.JobID.ToString())

    # Group results by RunID for sectional detail rendering
    $runGroups = $results | Group-Object -Property RunID

    foreach ($group in $runGroups) {
        $run = $group.Group[0]

        # 1. Map and Build Summary Table Row
        $summaryRows += $parts[1].Replace('{{COMPANY}}', $run.CompanyA) `
            .Replace('{{BATCH}}', $run.BatchID) `
            .Replace('{{JE}}', $run.JournalEntry) `
            .Replace('{{REFERENCE}}', $run.Reference) `
            .Replace('{{TRXDATE}}', (Get-Date $run.TxDate -Format "yyyy-MM-dd")) `
            .Replace('{{ROWS}}', $run.RowCountLine) `
            .Replace('{{STATUS}}', $run.Status) `
            .Replace('{{REVERSAL}}', $run.ReversalFlag) `
            .Replace('{{ERROR}}', $run.Message)

        # 2. Build Detail Section and handle Posted Status
        $postedBanner = ""
        if ($run.gp_Journal -isnot [System.DBNull]) {
            $postedBanner = $parts[4]
        }

        $detailRows = ""
        $tImpDeb = 0; $tImpCre = 0; $tImpNet = 0
        $tGpDeb = 0; $tGpCre = 0; $tGpNet = 0

        foreach ($line in $group.Group) {
            # Data binding with specific type checks for DBNull
            $detailRows += $parts[3].Replace('{{ACCOUNT}}', $line.Account) `
                .Replace('{{IMP_DEB}}', ("{0:N2}" -f $line.DebitAmt)) `
                .Replace('{{IMP_CRE}}', ("{0:N2}" -f $line.CreditAmt)) `
                .Replace('{{IMP_NET}}', ("{0:N2}" -f $line.NetAmt)) `
                .Replace('{{GP_DEB}}',  ("{0:N2}" -f (if($line.gp_DebitAmount -is [System.DBNull]){0}else{$line.gp_DebitAmount}))) `
                .Replace('{{GP_CRE}}',  ("{0:N2}" -f (if($line.gp_CreditAmount -is [System.DBNull]){0}else{$line.gp_CreditAmount}))) `
                .Replace('{{GP_NET}}',  ("{0:N2}" -f (if($line.gp_NetAmt -is [System.DBNull]){0}else{$line.gp_NetAmt})))

            $tImpDeb += $line.DebitAmt
            $tImpCre += $line.CreditAmt
            $tImpNet += $line.NetAmt
            
            if ($line.gp_DebitAmount -isnot [System.DBNull]) { $tGpDeb += $line.gp_DebitAmount }
            if ($line.gp_CreditAmount -isnot [System.DBNull]) { $tGpCre += $line.gp_CreditAmount }
            if ($line.gp_NetAmt -isnot [System.DBNull]) { $tGpNet += $line.gp_NetAmt }
        }

        # Apply Run-level placeholders to the detail section template
        $section = $parts[2].Replace('{{COMPANY}}', $run.CompanyA) `
            .Replace('{{POSTED_BANNER}}', $postedBanner) `
            .Replace('{{JE}}', $run.JournalEntry) `
            .Replace('{{TRXDATE}}', (Get-Date $run.TxDate -Format "yyyy-MM-dd")) `
            .Replace('{{BATCH}}', $run.BatchID) `
            .Replace('{{REFERENCE}}', $run.Reference) `
            .Replace('{{RUNID}}', $run.RunID.ToString()) `
            .Replace('{{DETAIL_ROWS}}', $detailRows) `
            .Replace('{{TOT_IMP_DEB}}', ("{0:N2}" -f $tImpDeb)) `
            .Replace('{{TOT_IMP_CRE}}', ("{0:N2}" -f $tImpCre)) `
            .Replace('{{TOT_IMP_NET}}', ("{0:N2}" -f $tImpNet)) `
            .Replace('{{TOT_GP_DEB}}',  ("{0:N2}" -f $tGpDeb)) `
            .Replace('{{TOT_GP_CRE}}',  ("{0:N2}" -f $tGpCre)) `
            .Replace('{{TOT_GP_NET}}',  ("{0:N2}" -f $tGpNet))

        $detailSections += $section
    }

    # Final HTML Assembly for return
    return $htmlMain.Replace('{{SUMMARY_ROWS}}', $summaryRows).Replace('{{DETAIL_SECTIONS}}', $detailSections)
}
Path</h3>" }
    
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
