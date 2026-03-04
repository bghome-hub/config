
<!DOCTYPE html>
<html>
<head>
<style>
    body { font-family: 'Segoe UI', Tahoma, Arial, sans-serif; font-size: 11px; color: #000; }
    .meta-block { margin-bottom: 15px; }
    .meta-block div { margin-bottom: 2px; }
    
    table { border-collapse: collapse; width: 100%; margin-bottom: 30px; }
    th, td { padding: 6px 10px; text-align: left; }
    
    /* Summary Table Styles (Green Theme) */
    .sum-table th { border-bottom: 1px solid #c8d8c8; background-color: #e2f0d9; font-weight: bold; }
    .sum-table tr:nth-child(even) { background-color: #ffffff; }
    .sum-table tr:nth-child(odd) { background-color: #f4f9f4; }
    
    /* Detail Table Styles */
    .det-header { font-size: 16px; font-weight: bold; margin-bottom: 10px; }
    .det-meta { border: none; margin-bottom: 15px; width: auto; }
    .det-meta td { padding: 2px 10px 2px 0; border: none; }
    .det-meta td:first-child { font-weight: bold; width: 100px; }
    
    .det-table th { border-bottom: 2px solid #ccc; font-weight: bold; }
    .det-table td { border-bottom: 1px solid #eee; }
    .det-table th.center { text-align: center; border-bottom: 1px solid #ccc; }
    .det-table th.right, .det-table td.right { text-align: right; font-family: 'Consolas', monospace; }
    .det-table tr.totals td { border-top: 2px solid #333; font-weight: bold; }
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
```

### Step 2: The Cleaned-Up PowerShell Function
Now, replace your `New-DynamicsImportJobEmailContent` function with this updated version. 

Look how much smaller and cleaner this is! There is **zero** inline CSS here. It just reads your `template.html`, hits the database, builds the raw `<tr>` data rows using the columns from your `usp_IA_Run_Audit` output (`DebitAmt`, `gp_DebitAmount`, etc.), and injects them into the template.

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId,
        # Looks for template.html in the same directory as this script
        [Parameter()][string]$TemplatePath = "$PSScriptRoot\template.html" 
    )

    # 1. Read the external HTML Template
    if (-not (Test-Path $TemplatePath)) { return "<h3>Template file not found at $TemplatePath</h3>" }
    $htmlTemplate = Get-Content -Path $TemplatePath -Raw

    # 2. Query your Database
    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    try { $auditData = Invoke-Db -Query $query } catch { return "<h3 style='color:red;'>Error retrieving data.</h3>" }
    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    # Grab top-level metadata safely
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

        # --- Build Summary Row <tr> ---
        $summaryRows += "<tr><td>$($first.CompanyA)</td><td>$($first.BatchID)</td><td>$($first.JournalEntry)</td><td>$($first.Reference)</td><td>$trxDateStr</td><td>$rowCount</td><td>$($first.STATUS)</td><td>$($first.ReversalFlag)</td><td>$($first.MESSAGE)</td></tr>"
        
        # --- Build Detail Table Layout ---
        $detailTables += "<div class='det-header'>$($first.CompanyA)</div>"
        $detailTables += "<table class='det-meta'>"
        $detailTables += "<tr><td>Journal Entry</td><td>$($first.JournalEntry)</td></tr>"
        $detailTables += "<tr><td>TxDate</td><td>$trxDateStr</td></tr>"
        $detailTables += "<tr><td>BatchID</td><td>$($first.BatchID)</td></tr>"
        $detailTables += "<tr><td>Reference</td><td>$($first.Reference)</td></tr>"
        $detailTables += "<tr><td>JobID</td><td>$JobId</td></tr>"
        $detailTables += "</table>"

        $detailTables += "<table class='det-table'>"
        $detailTables += "<thead><tr><th rowspan='2'>Account</th><th colspan='3' class='center'>Import File</th><th colspan='3' class='center'>Dynamics GP</th></tr>"
        $detailTables += "<tr><th class='right'>Debit</th><th class='right'>Credit</th><th class='right'>Total</th><th class='right'>Debit</th><th class='right'>Credit</th><th class='right'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0

        # --- Build Detail Lines ---
        $accounts = $group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $accName = ($acc.Group | Select-Object -First 1).Account
            
            $impDeb = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            
            # Catch null amounts
            if ($null -eq $impDeb) { $impDeb = 0 }
            if ($null -eq $impCred) { $impCred = 0 }
            if ($null -eq $impNet) { $impNet = 0 }
            if ($null -eq $gpDeb) { $gpDeb = 0 }
            if ($null -eq $gpCred) { $gpCred = 0 }
            
            $gpNet = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet

            $detailTables += "<tr><td>$accName</td><td class='right'>$("{0:N2}" -f $impDeb)</td><td class='right'>$("{0:N2}" -f $impCred)</td><td class='right'>$("{0:N2}" -f $impNet)</td><td class='right'>$("{0:N2}" -f $gpDeb)</td><td class='right'>$("{0:N2}" -f $gpCred)</td><td class='right'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $detailTables += "<tr class='totals'><td>Totals</td><td class='right'>$("{0:N2}" -f $sumImpDeb)</td><td class='right'>$("{0:N2}" -f $sumImpCred)</td><td class='right'>$("{0:N2}" -f $sumImpNet)</td><td class='right'>$("{0:N2}" -f $sumGpDeb)</td><td class='right'>$("{0:N2}" -f $sumGpCred)</td><td class='right'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $detailTables += "</tbody></table>"
    }

    # 3. Inject Data into HTML Template and return
    $finalHtml = $htmlTemplate.Replace('{{FILENAME}}', $fileName).Replace('{{STATUS}}', $jobResult).Replace('{{JOBID}}', $JobId.ToString()).Replace('{{SUMMARY_ROWS}}', $summaryRows).Replace('{{DETAIL_TABLES}}', $detailTables)

    return $finalHtml
}
```

This acts completely like a template engine. If you ever want to adjust padding, font sizes, or colors later, you just tweak the `.css` block in `template.html` and don't ever have to touch the messy PowerShell strings again! Test this out and tell me how it looks!            <tr><th>Company</th><th>Batch</th><th>JE</th><th>Reference</th><th>TrxDate</th><th>#Rows</th><th>RunStatus</th><th>Reversal</th><th>Error</th></tr>
        </thead>
        <tbody>
            {{SUMMARY_ROWS}}
        </tbody>
    </table>

    {{DETAIL_TABLES}}
</body>
</html>
```

### 2. Update your PowerShell Function
Now, replace your `New-DynamicsImportJobEmailContent` function with this. 

It reads your new `template.html` file using `Get-Content`, loops through your data to build the raw `<tr>` rows, and uses a simple `.Replace()` command to inject the data into your template. **No more inline HTML styling.**

*(Note: I added a parameter `$TemplatePath` so you can point it to wherever you saved the HTML file. It defaults to looking in the exact same folder as your script.)*

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId,
        [Parameter()][string]$TemplatePath = "$PSScriptRoot\template.html"
    )

    # 1. Read the external HTML Template
    if (-not (Test-Path $TemplatePath)) { return "<h3>Template file not found at $TemplatePath</h3>" }
    $htmlTemplate = Get-Content -Path $TemplatePath -Raw

    # 2. Query your Database
    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    try { $auditData = Invoke-Db -Query $query } catch { return "<h3 style='color:red;'>Error retrieving data.</h3>" }
    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    # Grab top-level metadata safely
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

        # --- Build Summary Row <tr> ---
        $summaryRows += "<tr><td>$($first.CompanyA)</td><td>$($first.BatchID)</td><td>$($first.JournalEntry)</td><td>$($first.Reference)</td><td>$trxDateStr</td><td>$rowCount</td><td>$($first.STATUS)</td><td>$($first.ReversalFlag)</td><td>$($first.MESSAGE)</td></tr>"
        
        # --- Build Detail Table Layout ---
        $detailTables += "<div class='det-header'>$($first.CompanyA)</div>"
        $detailTables += "<table class='det-meta'>"
        $detailTables += "<tr><td>Journal Entry</td><td>$($first.JournalEntry)</td></tr>"
        $detailTables += "<tr><td>TxDate</td><td>$trxDateStr</td></tr>"
        $detailTables += "<tr><td>BatchID</td><td>$($first.BatchID)</td></tr>"
        $detailTables += "<tr><td>Reference</td><td>$($first.Reference)</td></tr>"
        $detailTables += "<tr><td>JobID</td><td>$JobId</td></tr>"
        $detailTables += "</table>"

        $detailTables += "<table class='det-table'>"
        $detailTables += "<thead><tr><th rowspan='2'>Account</th><th colspan='3' class='center'>Import File</th><th colspan='3' class='center'>Dynamics GP</th></tr>"
        $detailTables += "<tr><th class='right'>Debit</th><th class='right'>Credit</th><th class='right'>Total</th><th class='right'>Debit</th><th class='right'>Credit</th><th class='right'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0

        # --- Build Detail Lines ---
        $accounts = $group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $accName = ($acc.Group | Select-Object -First 1).Account
            
            $impDeb = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            
            # Catch null amounts
            if ($null -eq $impDeb) { $impDeb = 0 }
            if ($null -eq $impCred) { $impCred = 0 }
            if ($null -eq $impNet) { $impNet = 0 }
            if ($null -eq $gpDeb) { $gpDeb = 0 }
            if ($null -eq $gpCred) { $gpCred = 0 }
            
            $gpNet = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet

            $detailTables += "<tr><td>$accName</td><td class='right'>$("{0:N2}" -f $impDeb)</td><td class='right'>$("{0:N2}" -f $impCred)</td><td class='right'>$("{0:N2}" -f $impNet)</td><td class='right'>$("{0:N2}" -f $gpDeb)</td><td class='right'>$("{0:N2}" -f $gpCred)</td><td class='right'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $detailTables += "<tr class='totals'><td>Totals</td><td class='right'>$("{0:N2}" -f $sumImpDeb)</td><td class='right'>$("{0:N2}" -f $sumImpCred)</td><td class='right'>$("{0:N2}" -f $sumImpNet)</td><td class='right'>$("{0:N2}" -f $sumGpDeb)</td><td class='right'>$("{0:N2}" -f $sumGpCred)</td><td class='right'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $detailTables += "</tbody></table>"
    }

    # 3. Inject Data into HTML Template
    $finalHtml = $htmlTemplate.Replace('{{FILENAME}}', $fileName).Replace('{{STATUS}}', $jobResult).Replace('{{JOBID}}', $JobId.ToString()).Replace('{{SUMMARY_ROWS}}', $summaryRows).Replace('{{DETAIL_TABLES}}', $detailTables)

    return $finalHtml
}
```

This completely uncouples your HTML from PowerShell. If you ever want to adjust padding, font sizes, or colors later, you just tweak `template.html` and don't ever have to touch the PowerShell code again.
