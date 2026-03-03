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

    # Extract Job-Level Header Info
    $firstGlobalRow = $auditData
    $fileName  = $firstGlobalRow.FileName
    $jobResult = $firstGlobalRow.JobResult

    # ---------------------------------------------------------
    # PART 1: SUMMARY TABLE (Mimicking summary.png)
    # ---------------------------------------------------------
    $font       = "font-family: 'Segoe UI', Arial, sans-serif; font-size: 11px; color: #000;"
    $thSumStyle = "border-bottom: 1px solid #c8d8c8; padding: 6px 10px; background-color: #e2f0d9; text-align: left; font-weight: bold;"
    $tdSumBase  = "padding: 6px 10px; text-align: left;"
    
    $html = "<div style='$font'>"
    $html += "<div style='margin-bottom: 2px;'>Filename: <strong>$fileName</strong></div>"
    $html += "<div style='margin-bottom: 2px;'>Result: <strong>$jobResult</strong></div>"
    $html += "<div style='margin-bottom: 15px;'>Job ID: <strong>$JobId</strong></div>"

    $html += "<table style='border-collapse: collapse; width: 100%; $font'>"
    $html += "<thead><tr>"
    $html += "<th style='$thSumStyle'>Company</th><th style='$thSumStyle'>Batch</th>"
    $html += "<th style='$thSumStyle'>JE</th><th style='$thSumStyle'>Reference</th>"
    $html += "<th style='$thSumStyle'>TrxDate</th><th style='$thSumStyle'>#Rows</th>"
    $html += "<th style='$thSumStyle'>RunStatus</th><th style='$thSumStyle'>Reversal</th>"
    $html += "<th style='$thSumStyle'>Error</th>"
    $html += "</tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    $rowIndex = 0

    foreach ($run in $runs) {
        $first = $run.Group
        
        # Zebra Striping
        $bgColor = if ($rowIndex % 2 -eq 0) { "#ffffff" } else { "#f4f9f4" }
        $trxDateStr = if ($first.TxDate -is [DateTime]) { $first.TxDate.ToString("yyyy-MM-dd") } else { $first.TxDate }

        $html += "<tr style='background-color: $bgColor;'>"
        $html += "<td style='$tdSumBase'>$($first.CompanyA)</td><td style='$tdSumBase'>$($first.BatchID)</td>"
        $html += "<td style='$tdSumBase'>$($first.JournalEntry)</td><td style='$tdSumBase'>$($first.Reference)</td>"
        $html += "<td style='$tdSumBase'>$trxDateStr</td><td style='$tdSumBase'>$($first.RowCountLine)</td>"
        $html += "<td style='$tdSumBase'>$($first.STATUS)</td><td style='$tdSumBase'>$($first.ReversalFlag)</td>"
        $html += "<td style='$tdSumBase'>$($first.MESSAGE)</td>"
        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # ---------------------------------------------------------
    # PART 2: DETAIL TABLES (Mimicking detail.png)
    # ---------------------------------------------------------
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        $first = $run.Group
        $trxDateStr = if ($first.TxDate -is [DateTime]) { $first.TxDate.ToString("yyyy-MM-dd") } else { $first.TxDate }
        
        # Company Header
        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        
        # Metadata Block (Borderless)
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        # Detail Lines Table
        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr>"
        $html += "<th style='$thDet' rowspan='2'>Account</th>"
        $html += "<th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th>"
        $html += "<th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th>"
        $html += "</tr><tr>"
        $html += "<th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th>"
        $html += "<th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th>"
        $html += "</tr></thead><tbody>"

        $accounts = $run.Group | Group-Object -Property Account
        
        # Initialize Grand Totals for this run
        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        foreach ($acc in $accounts) {
            # Roll up amounts per account
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = $impDeb - $impCred
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            $gpNet   = $gpDeb - $gpCred

            # Add to Grand Totals
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr>"
            $html += "<td style='$tdDet'>$($acc.Name)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $impCred)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $impNet)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td>"
            $html += "<td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td>"
            $html += "</tr>"
        }

        # The 'Totals' row at the bottom
        $html += "<tr>"
        $html += "<td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td>"
        $html += "<td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td>"
        $html += "</tr>"

        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}
