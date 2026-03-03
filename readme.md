<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; font-size: 14px; color: #333; }
        h2 { color: #0056b3; }
        h3 { color: #444; margin-bottom: 5px; }
        table { border-collapse: collapse; width: 100%; margin-bottom: 20px; font-size: 13px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; font-weight: bold; }
        .status-success { color: green; font-weight: bold; }
        .status-error { color: red; font-weight: bold; }
        .amt { text-align: right; }
    </style>
</head>
<body>
    <h2>Import Job Summary</h2>
    <table>
        <thead>
            <tr>
                <th>Company</th>
                <th>Batch ID</th>
                <th>Journal Entry</th>
                <th>Status</th>
                <th>Import Net</th>
                <th>GP Net</th>
                <th>Message</th>
            </tr>
        </thead>
        <tbody>
            {{SummaryRows}}
        </tbody>
    </table>

    <h2>Import vs. GL Detail Lines</h2>
    {{DetailTables}}
</body>
</html>

function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [Guid]$JobId,
        [Parameter(Mandatory)]
        [string]$TemplatePath # e.g., "C:\scripts\template.html"
    )

    # 1. Execute the newly updated audit stored procedure ONCE
    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    
    try {
        $auditData = Invoke-Db -Query $query
    } catch {
        Write-Log -Message "Failed to execute audit SP: $($_.Exception.Message)" -Level "ERROR"
        return "<h3>Error retrieving audit data.</h3>"
    }

    if (-not $auditData) {
        return "<h3>No audit data found for JobID: $JobId</h3>"
    }

    # 2. Initialize string builders for our HTML parts
    $summaryHtml = ""
    $detailHtml = ""

    # 3. Group the dataset by RunID (which separates it by Company/Batch/JE)
    $runs = $auditData | Group-Object -Property RunID

    foreach ($run in $runs) {
        # Extract header info from the first row of the group
        $firstRow = $run.Group
        $company  = $firstRow.CompanyA
        $batch    = $firstRow.BatchID
        $je       = $firstRow.JournalEntry
        $status   = $firstRow.STATUS
        $msg      = $firstRow.MESSAGE

        # Format status color
        $statusClass = if ($status -match "Error|Failed") { "status-error" } else { "status-success" }

        # Calculate Totals for the Summary
        # Note: Using the column names mapped in your SP (DebitAmt, gp_DebitAmount, etc.)
        $importNet = ($run.Group | Measure-Object -Property NetAmt -Sum).Sum
        $gpNetCalc = ($run.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum - ($run.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
        
        # Build the Summary Row
        $summaryHtml += "<tr>
            <td>$company</td>
            <td>$batch</td>
            <td>$je</td>
            <td class='$statusClass'>$status</td>
            <td class='amt'>$("{0:N2}" -f $importNet)</td>
            <td class='amt'>$("{0:N2}" -f $gpNetCalc)</td>
            <td>$msg</td>
        </tr>"

        # Build the Detail Table for this specific Run
        $detailHtml += "<h3>$company | Batch: $batch | JE: $je</h3>"
        $detailHtml += "<table>
            <thead>
                <tr>
                    <th rowspan='2'>Account</th>
                    <th colspan='2' style='text-align:center'>Import File</th>
                    <th colspan='2' style='text-align:center'>Dynamics GP</th>
                </tr>
                <tr>
                    <th class='amt'>Debit</th><th class='amt'>Credit</th>
                    <th class='amt'>Debit</th><th class='amt'>Credit</th>
                </tr>
            </thead>
            <tbody>"

        foreach ($line in $run.Group) {
            $detailHtml += "<tr>
                <td>$($line.Account)</td>
                <td class='amt'>$("{0:N2}" -f $line.DebitAmt)</td>
                <td class='amt'>$("{0:N2}" -f $line.CreditAmt)</td>
                <td class='amt'>$("{0:N2}" -f $line.gp_DebitAmount)</td>
                <td class='amt'>$("{0:N2}" -f $line.gp_CreditAmount)</td>
            </tr>"
        }
        $detailHtml += "</tbody></table>"
    }

    # 4. Load the template and inject the generated HTML
    $template = Get-Content -Path $TemplatePath -Raw
    $finalEmailBody = $template -replace '\{\{SummaryRows\}\}', $summaryHtml `
                                -replace '\{\{DetailTables\}\}', $detailHtml

    return $finalEmailBody
}
