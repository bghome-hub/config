<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #333; background-color: #f4f7f6; margin: 0; padding: 20px; }
        .container { background-color: #fff; padding: 20px; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); max-width: 1200px; margin: auto; }
        h2 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; margin-top: 0; }
        h3 { color: #34495e; margin-top: 30px; }
        .meta-data { font-size: 14px; color: #555; margin-bottom: 20px; }
        table { border-collapse: collapse; width: 100%; font-size: 13px; margin-bottom: 20px; }
        th { background-color: #ecf0f1; color: #2c3e50; font-weight: bold; text-align: left; padding: 10px; border: 1px solid #bdc3c7; }
        td { padding: 8px 10px; border: 1px solid #e0e6ed; }
        tr:nth-child(even) { background-color: #f9fbfd; }
        
        /* Alignment */
        .right { text-align: right; font-family: 'Courier New', Courier, monospace; }
        .center { text-align: center; }
        
        /* Dynamic Status Colors */
        .status-ok { color: #155724; background-color: #d4edda; font-weight: bold; padding: 3px 6px; border-radius: 3px; display: inline-block;}
        .status-fail { color: #721c24; background-color: #f8d7da; font-weight: bold; padding: 3px 6px; border-radius: 3px; display: inline-block;}
        
        /* Dynamic Variance Colors */
        .variance-bad { color: #fff; background-color: #e74c3c; font-weight: bold; }
        .variance-good { color: #27ae60; }
    </style>
</head>
<body>
    <div class="container">
        <h2>Intracompany Upload Reconciliation</h2>
        
        <div class="meta-data">
            <strong>Job ID:</strong> {{JobId}}<br>
            <strong>Run Date:</strong> {{RunDate}}
        </div>

        <h3>Execution Summary</h3>
        <table>
            <thead>
                <tr>
                    <th>Company</th>
                    <th>Batch ID</th>
                    <th class="center">Status</th>
                    <th class="right">Total Uploaded Lines</th>
                </tr>
            </thead>
            <tbody>
                {{SummaryRows}}
            </tbody>
        </table>

        <h3>Detail Reconciliation</h3>
        <table>
            <thead>
                <tr>
                    <th>Company</th>
                    <th>Batch / Journal</th>
                    <th>Account</th>
                    <th class="right">Uploaded Debit</th>
                    <th class="right">Uploaded Credit</th>
                    <th class="right">GP Debit</th>
                    <th class="right">GP Credit</th>
                    <th class="right">Net Variance</th>
                    <th class="center">GP Status</th>
                </tr>
            </thead>
            <tbody>
                {{DetailRows}}
            </tbody>
        </table>
    </div>
</body>
</html>







# Parameters
$ServerInstance = "YourSQLServer"
$Database = "YourIntegrationDB"
$JobId = "YOUR-JOB-ID-HERE" # This would be passed in from your orchestrator
$TemplatePath = "C:\Scripts\ReportTemplate.html"
$OutputPath = "C:\Scripts\FinalReport.html"

# 1. Execute the stored procedure to get the raw data
$Query = "EXEC dbo.usp_IC_ReportJobReconciliation @JobId = '$JobId'"
$Details = Invoke-Sqlcmd -ServerInstance $ServerInstance -Database $Database -Query $Query

# 2. Build the Summary Rows (Grouping the Detail data in PowerShell)
$SummaryGroups = $Details | Group-Object Company, gp_BatchNumber, gp_Status
$SummaryHtml = ""

foreach ($Group in $SummaryGroups) {
    # Extract the values from the group name
    $Comp = $Group.Group[0].Company
    $Batch = $Group.Group[0].gp_BatchNumber
    $Status = $Group.Group[0].gp_Status
    $LineCount = $Group.Count

    # Determine CSS badge color for status
    $StatusClass = if ($Status -like "*POSTED*") { "status-ok" } else { "status-fail" }

    $SummaryHtml += "<tr>
        <td>$Comp</td>
        <td>$Batch</td>
        <td class='center'><span class='$StatusClass'>$Status</span></td>
        <td class='right'>$LineCount</td>
    </tr>`n"
}

# 3. Build the Detail Rows (With color coding for bad variances)
$DetailHtml = ""

foreach ($Row in $Details) {
    # Formatting money
    $UpDeb  = "{0:N2}" -f $Row.Uploaded_Debit
    $UpCred = "{0:N2}" -f $Row.Uploaded_Credit
    $GpDeb  = "{0:N2}" -f $Row.gp_DebitAmount
    $GpCred = "{0:N2}" -f $Row.gp_CreditAmount
    $Var    = "{0:N2}" -f $Row.Net_Variance

    # If variance is not 0.00, flag it red using the CSS class from the template
    $VarClass = if ($Row.Net_Variance -ne 0) { "variance-bad right" } else { "variance-good right" }
    $StatusClass = if ($Row.gp_Status -like "*POSTED*") { "status-ok" } else { "status-fail" }

    $DetailHtml += "<tr>
        <td>$($Row.Company)</td>
        <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td>
        <td>$($Row.Account)</td>
        <td class='right'>$UpDeb</td>
        <td class='right'>$UpCred</td>
        <td class='right'>$GpDeb</td>
        <td class='right'>$GpCred</td>
        <td class='$VarClass'>$Var</td>
        <td class='center'><span class='$StatusClass'>$($Row.gp_Status)</span></td>
    </tr>`n"
}

# 4. Load Template, Swap Tokens, and Save/Send
$HtmlTemplate = Get-Content $TemplatePath -Raw

# Replace the tokens with our generated rows and metadata
$FinalHtml = $HtmlTemplate -replace '{{JobId}}', $JobId `
                           -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                           -replace '{{SummaryRows}}', $SummaryHtml `
                           -replace '{{DetailRows}}', $DetailHtml

# Save to disk (Or pass $FinalHtml directly to your Send-MailMessage function)
$FinalHtml | Set-Content $OutputPath

Write-Host "Report generated: $OutputPath"
