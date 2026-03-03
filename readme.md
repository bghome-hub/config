
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param([Parameter(Mandatory)][Guid]$JobId)

    # 1. Execute the stored procedure
    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    try {
        $auditData = Invoke-Db -Query $query
    } catch {
        return "<h3 style='color:red;'>Error retrieving audit data.</h3>"
    }

    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    # 2. Define Inline CSS Styles (This defeats Outlook's style-stripping)
    $tableStyle = "border-collapse: collapse; width: 100%; margin-bottom: 20px; font-family: 'Segoe UI', Tahoma, sans-serif; font-size: 13px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);"
    $thStyle    = "border: 1px solid #dddddd; padding: 10px; background-color: #2c3e50; color: #ffffff; text-align: left; text-transform: uppercase; font-size: 12px; font-weight: bold;"
    $thCenter   = "border: 1px solid #dddddd; padding: 10px; background-color: #34495e; color: #ffffff; text-align: center; text-transform: uppercase; font-size: 12px; font-weight: bold;"
    $tdStyle    = "border: 1px solid #dddddd; padding: 10px; text-align: left; color: #333333;"
    $tdAmtStyle = "border: 1px solid #dddddd; padding: 10px; text-align: right; font-family: 'Consolas', monospace; color: #333333;"
    $h2Style    = "color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 5px; font-family: 'Segoe UI', sans-serif; margin-top: 30px;"
    $h3Style    = "color: #34495e; background-color: #e8ecf1; padding: 8px; border-left: 4px solid #2980b9; font-family: 'Segoe UI', sans-serif; margin-bottom: 0;"

    # 3. Build the Summary Table
    $html = "<h2 style='$h2Style'>Dynamics GP Import Job Summary</h2>"
    $html += "<table style='$tableStyle'><thead><tr>"
    $html += "<th style='$thStyle'>Company</th><th style='$thStyle'>Batch ID</th><th style='$thStyle'>Journal Entry</th><th style='$thStyle'>Status</th><th style='$thStyle'>Message</th>"
    $html += "</tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    foreach ($run in $runs) {
        $first = $run.Group
        
        # Color code the status
        $statusColor = if ($first.STATUS -match "Error|Failed") { "color: #721c24; font-weight: bold;" } else { "color: #155724; font-weight: bold;" }

        $html += "<tr>"
        $html += "<td style='$tdStyle'>$($first.CompanyA)</td>"
        $html += "<td style='$tdStyle'>$($first.BatchID)</td>"
        $html += "<td style='$tdStyle'>$($first.JournalEntry)</td>"
        $html += "<td style='$tdStyle'><span style='$statusColor'>$($first.STATUS)</span></td>"
        $html += "<td style='$tdStyle'>$($first.MESSAGE)</td>"
        $html += "</tr>"
    }
    $html += "</tbody></table>"

    # 4. Build the Detail Tables (SQUASHING DUPLICATES)
    $html += "<h2 style='$h2Style'>Import vs. GL Detail Lines</h2>"

    foreach ($run in $runs) {
        $first = $run.Group
        $html += "<h3 style='$h3Style'>$($first.CompanyA) | Batch: $($first.BatchID) | JE: $($first.JournalEntry)</h3>"
        
        $html += "<table style='$tableStyle'><thead><tr>"
        $html += "<th style='$thStyle' rowspan='2'>Account</th>"
        $html += "<th style='$thCenter' colspan='2'>Import File</th>"
        $html += "<th style='$thCenter' colspan='2'>Dynamics GP</th>"
        $html += "</tr><tr>"
        $html += "<th style='$thStyle'>Debit</th><th style='$thStyle'>Credit</th>"
        $html += "<th style='$thStyle'>Debit</th><th style='$thStyle'>Credit</th>"
        $html += "</tr></thead><tbody>"

        # THIS IS THE MAGIC: Grouping by Account squashes all duplicate SQL rows into a single net line
        $accounts = $run.Group | Group-Object -Property Account
        
        foreach ($acc in $accounts) {
            # Sum the amounts to guarantee exactly one perfect line per account
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum

            $html += "<tr>"
            $html += "<td style='$tdStyle'>$($acc.Name)</td>"
            $html += "<td style='$tdAmtStyle'>$("{0:N2}" -f $impDeb)</td>"
            $html += "<td style='$tdAmtStyle'>$("{0:N2}" -f $impCred)</td>"
            $html += "<td style='$tdAmtStyle'>$("{0:N2}" -f $gpDeb)</td>"
            $html += "<td style='$tdAmtStyle'>$("{0:N2}" -f $gpCred)</td>"
            $html += "</tr>"
        }
        $html += "</tbody></table>"
    }

    return $html
}
