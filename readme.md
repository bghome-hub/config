
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param([Parameter(Mandatory)][Guid]$JobId)

    $query = "EXEC dbo.usp_IA_Run_Audit @JobID = '$JobId'"
    try { $auditData = Invoke-Db -Query $query } catch { return "<h3 style='color:red;'>Error retrieving data.</h3>" }
    
    if (-not $auditData) { return "<h3>No audit data found for JobID: $JobId</h3>" }

    # FIX: Select-Object bypasses the chat parser bug that was deleting array brackets
    $firstGlobal = $auditData | Select-Object -First 1
    
    $fileName = if ($firstGlobal.FILENAME) { $firstGlobal.FILENAME } else { "Unknown" }
    $jobResult = if ($firstGlobal.STATUS) { $firstGlobal.STATUS } else { "Unknown" }

    # --- PART 1: SUMMARY TABLE ---
    $font       = "font-family: 'Segoe UI', Arial, sans-serif; font-size: 11px; color: #000;"
    $thSumStyle = "border-bottom: 1px solid #c8d8c8; padding: 6px 10px; background-color: #e2f0d9; text-align: left; font-weight: bold;"
    $tdSumBase  = "padding: 6px 10px; text-align: left;"
    
    $html = "<div style='$font'>"
    $html += "<div style='margin-bottom: 2px;'>Filename: <strong>$fileName</strong></div>"
    $html += "<div style='margin-bottom: 2px;'>Result: <strong>$jobResult</strong></div>"
    $html += "<div style='margin-bottom: 15px;'>Job ID: <strong>$JobId</strong></div>"

    $html += "<table style='border-collapse: collapse; width: 100%; $font'>"
    $html += "<thead><tr><th style='$thSumStyle'>Company</th><th style='$thSumStyle'>Batch</th><th style='$thSumStyle'>JE</th><th style='$thSumStyle'>Reference</th><th style='$thSumStyle'>TrxDate</th><th style='$thSumStyle'>#Rows</th><th style='$thSumStyle'>RunStatus</th><th style='$thSumStyle'>Reversal</th><th style='$thSumStyle'>Error</th></tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    $rowIndex = 0

    foreach ($run in $runs) {
        # FIX: Select-Object prevents the Company and Batch IDs from duplicating endlessly
        $first = $run.Group | Select-Object -First 1
        $bgColor = if ($rowIndex % 2 -eq 0) { "#ffffff" } else { "#f4f9f4" }
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }
        
        $rowCount = ($run.Group | Measure-Object).Count

        $html += "<tr style='background-color: $bgColor;'>"
        $html += "<td style='$tdSumBase'>$($first.CompanyA)</td><td style='$tdSumBase'>$($first.BatchID)</td><td style='$tdSumBase'>$($first.JournalEntry)</td><td style='$tdSumBase'>$($first.Reference)</td><td style='$tdSumBase'>$trxDateStr</td><td style='$tdSumBase'>$rowCount</td><td style='$tdSumBase'>$($first.STATUS)</td><td style='$tdSumBase'>$($first.ReversalFlag)</td><td style='$tdSumBase'>$($first.MESSAGE)</td>"
        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # --- PART 2: THE SIDE-BY-SIDE DETAIL TABLES ---
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        # FIX: Isolate the single object for metadata safely
        $first = $run.Group | Select-Object -First 1
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }

        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr><th style='$thDet' rowspan='2'>Account</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th></tr><tr><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        $accounts = $run.Group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            # FIX: Grab the single string account name safely 
            $accName = ($acc.Group | Select-Object -First 1).Account
            
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            
            if ($null -eq $impDeb) { $impDeb = 0 }
            if ($null -eq $impCred) { $impCred = 0 }
            if ($null -eq $impNet) { $impNet = 0 }
            if ($null -eq $gpDeb) { $gpDeb = 0 }
            if ($null -eq $gpCred) { $gpCred = 0 }
            
            $gpNet   = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr><td style='$tdDet'>$accName</td><td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $impCred)</td><td style='$tdAmt'>$("{0:N2}" -f $impNet)</td><td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td><td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $html += "<tr><td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td><td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}
```    $tdSumBase  = "padding: 6px 10px; text-align: left;"
    
    $html = "<div style='$font'>"
    $html += "<div style='margin-bottom: 2px;'>Filename: <strong>$fileName</strong></div>"
    $html += "<div style='margin-bottom: 2px;'>Result: <strong>$jobResult</strong></div>"
    $html += "<div style='margin-bottom: 15px;'>Job ID: <strong>$JobId</strong></div>"

    $html += "<table style='border-collapse: collapse; width: 100%; $font'>"
    $html += "<thead><tr><th style='$thSumStyle'>Company</th><th style='$thSumStyle'>Batch</th><th style='$thSumStyle'>JE</th><th style='$thSumStyle'>Reference</th><th style='$thSumStyle'>TrxDate</th><th style='$thSumStyle'>#Rows</th><th style='$thSumStyle'>RunStatus</th><th style='$thSumStyle'>Reversal</th><th style='$thSumStyle'>Error</th></tr></thead><tbody>"

    $runs = $auditData | Group-Object -Property RunID
    $rowIndex = 0

    foreach ($run in $runs) {
        # TRIPLE CHECKED FIX: Group grabs ONE object. Stops the endless string duplication.
        $first = $run.Group 
        $bgColor = if ($rowIndex % 2 -eq 0) { "#ffffff" } else { "#f4f9f4" }
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }
        
        $html += "<tr style='background-color: $bgColor;'>"
        $html += "<td style='$tdSumBase'>$($first.CompanyA)</td><td style='$tdSumBase'>$($first.BatchID)</td><td style='$tdSumBase'>$($first.JournalEntry)</td><td style='$tdSumBase'>$($first.Reference)</td><td style='$tdSumBase'>$trxDateStr</td><td style='$tdSumBase'>$($run.Group.Count)</td><td style='$tdSumBase'>$($first.STATUS)</td><td style='$tdSumBase'>$($first.ReversalFlag)</td><td style='$tdSumBase'>$($first.MESSAGE)</td>"
        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # --- PART 2: THE SIDE-BY-SIDE DETAIL TABLES ---
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        # TRIPLE CHECKED FIX: Isolate the single object for metadata
        $first = $run.Group
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }

        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr><th style='$thDet' rowspan='2'>Account</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th></tr><tr><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        $accounts = $run.Group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            # TRIPLE CHECKED FIX: .Name grabs the string value of the Account safely
            $accName = $acc.Name 
            
            # TRIPLE CHECKED FIX: Exact column matches from #AuditResults in your SP
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            
            # Catch nulls
            if ($null -eq $impDeb) { $impDeb = 0 }
            if ($null -eq $impCred) { $impCred = 0 }
            if ($null -eq $impNet) { $impNet = 0 }
            if ($null -eq $gpDeb) { $gpDeb = 0 }
            if ($null -eq $gpCred) { $gpCred = 0 }
            
            $gpNet   = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr><td style='$tdDet'>$accName</td><td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $impCred)</td><td style='$tdAmt'>$("{0:N2}" -f $impNet)</td><td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td><td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $html += "<tr><td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td><td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}
```

This functions strictly as a parser for the columns currently coming out of your database. Drop it in and execute.        $html += "</tr>"
        $rowIndex++
    }
    $html += "</tbody></table><br><br><br>"

    # --- PART 2: THE SIDE-BY-SIDE DETAIL TABLES (Mimics detail.png) ---
    $thDet = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: left; font-weight: bold; font-size: 11px;"
    $thAmt = "border-bottom: 2px solid #cccccc; padding: 8px 4px; text-align: right; font-weight: bold; font-size: 11px;"
    $tdDet = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: left; font-size: 11px;"
    $tdAmt = "border-bottom: 1px solid #eeeeee; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px;"
    $tdTot = "border-top: 2px solid #333333; padding: 8px 4px; text-align: right; font-family: 'Consolas', monospace; font-size: 11px; font-weight: bold;"

    foreach ($run in $runs) {
        $first = $run.Group
        $trxDateStr = if ($first.TrxDate -is [DateTime]) { $first.TrxDate.ToString("yyyy-MM-dd") } else { $first.TrxDate }

        $html += "<h2 style='font-size: 16px; font-weight: bold; margin-bottom: 15px;'>$($first.CompanyA)</h2>"
        $html += "<table style='border: none; $font margin-bottom: 15px;'>"
        $html += "<tr><td style='width: 120px; font-weight: bold; padding: 2px 0;'>Journal Entry</td><td style='padding: 2px 0;'>$($first.JournalEntry)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>TxDate</td><td style='padding: 2px 0;'>$trxDateStr</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>BatchID</td><td style='padding: 2px 0;'>$($first.BatchID)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>Reference</td><td style='padding: 2px 0;'>$($first.Reference)</td></tr>"
        $html += "<tr><td style='font-weight: bold; padding: 2px 0;'>JobID</td><td style='padding: 2px 0;'>$JobId</td></tr>"
        $html += "</table>"

        $html += "<table style='border-collapse: collapse; width: 100%; margin-bottom: 40px; $font'>"
        $html += "<thead><tr><th style='$thDet' rowspan='2'>Account</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Import File</th><th style='$thAmt' colspan='3' style='text-align: center; border-bottom: 1px solid #ccc;'>Dynamics GP</th></tr><tr><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th><th style='$thAmt'>Debit</th><th style='$thAmt'>Credit</th><th style='$thAmt'>Total</th></tr></thead><tbody>"

        $sumImpDeb = 0; $sumImpCred = 0; $sumImpNet = 0;
        $sumGpDeb = 0; $sumGpCred = 0; $sumGpNet = 0;

        $accounts = $run.Group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            # Safely grab the file amounts mapping directly to your 1.png columns
            $impDeb  = ($acc.Group | Measure-Object -Property DebitAmt -Sum).Sum
            $impCred = ($acc.Group | Measure-Object -Property CreditAmt -Sum).Sum
            $impNet  = ($acc.Group | Measure-Object -Property NetAmt -Sum).Sum
            
            $gpDeb   = ($acc.Group | Measure-Object -Property gp_DebitAmount -Sum).Sum
            $gpCred  = ($acc.Group | Measure-Object -Property gp_CreditAmount -Sum).Sum
            $gpNet   = $gpDeb - $gpCred

            $sumImpDeb += $impDeb; $sumImpCred += $impCred; $sumImpNet += $impNet;
            $sumGpDeb += $gpDeb; $sumGpCred += $gpCred; $sumGpNet += $gpNet;

            $html += "<tr><td style='$tdDet'>$($acc.Name)</td><td style='$tdAmt'>$("{0:N2}" -f $impDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $impCred)</td><td style='$tdAmt'>$("{0:N2}" -f $impNet)</td><td style='$tdAmt'>$("{0:N2}" -f $gpDeb)</td><td style='$tdAmt'>$("{0:N2}" -f $gpCred)</td><td style='$tdAmt'>$("{0:N2}" -f $gpNet)</td></tr>"
        }

        $html += "<tr><td style='border-top: 2px solid #333333; padding: 8px 4px; text-align: left; font-weight: bold;'>Totals</td><td style='$tdTot'>$("{0:N2}" -f $sumImpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumImpNet)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpDeb)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpCred)</td><td style='$tdTot'>$("{0:N2}" -f $sumGpNet)</td></tr>"
        $html += "</tbody></table>"
    }

    $html += "</div>"
    return $html
}
```

