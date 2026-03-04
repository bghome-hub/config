I am so sorry. The logic I added to hide "POSTED" batches was too aggressive—it accidentally triggered and overwrote your actual GP values with hyphens (`-`). That is exactly why your GL values disappeared.

I have completely ripped that out. This will show your GL values again. I also removed **all** backticks so you won't get any PowerShell syntax errors, and kept every line extremely short so nothing truncates off the edge of your screen. 

Copy and drop this in right now for your meeting. Your `template.html` is fine, just update the PowerShell:

```powershell
function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    $TemplatePath = Join-Path $PSScriptRoot "template.html"
    
    if (-not (Test-Path $TemplatePath)) { 
        return "<h3>Template not found: $TemplatePath</h3>" 
    }
    
    $rawHtml = Get-Content -Path $TemplatePath -Raw
    
    $mainTpl, $sumRowTpl, $detBlockTpl, $detRowTpl, $postedMsgTpl = $rawHtml -split '@@@SPLIT@@@'

    $mainTpl      = $mainTpl.Trim()
    $sumRowTpl    = $sumRowTpl.Trim()
    $detBlockTpl  = $detBlockTpl.Trim()
    $detRowTpl    = $detRowTpl.Trim()
    $postedMsgTpl = $postedMsgTpl.Trim()

    $query = "EXEC dbo.usp_ia_job_report @JobID = '$JobId'"
    $srv = $sqlconfig.server
    $db = $sqlconfig.database
    
    try { 
        $auditData = Invoke-Sqlcmd -ServerInstance $srv -Database $db -TrustServerCertificate -Query $query 
    } catch { 
        return "<h3>Error: $($_.Exception.Message)</h3>" 
    }
    
    if (-not $auditData) { 
        return "<h3>No data for JobID: $JobId</h3>" 
    }

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

        $rowTpl = $sumRowTpl
        $rowTpl = $rowTpl.Replace('{{COMPANY}}', "$($first.CompanyA)")
        $rowTpl = $rowTpl.Replace('{{BATCH}}', "$($first.BatchID)")
        $rowTpl = $rowTpl.Replace('{{JE}}', "$($first.JournalEntry)")
        $rowTpl = $rowTpl.Replace('{{REF}}', "$($first.Reference)")
        $rowTpl = $rowTpl.Replace('{{TRXDATE}}', "$trxDateStr")
        $rowTpl = $rowTpl.Replace('{{ROWS}}', "$rowCount")
        $rowTpl = $rowTpl.Replace('{{STATUS}}', "$($first.STATUS)")
        $rowTpl = $rowTpl.Replace('{{REVERSAL}}', "$($first.ReversalFlag)")
        $rowTpl = $rowTpl.Replace('{{ERROR}}', "$($first.MESSAGE)")
        $summaryRows += $rowTpl

        $injectPostedMsg = ""
        if ($first.STATUS -match 'POSTED') { 
            $injectPostedMsg = $postedMsgTpl 
        }

        [decimal]$sumImpDeb = 0; [decimal]$sumImpCred = 0; [decimal]$sumImpNet = 0
        [decimal]$sumGpDeb = 0;  [decimal]$sumGpCred = 0;  [decimal]$sumGpNet = 0
        $detailRowsOut = ""

        $accounts = $group | Group-Object -Property Account
        foreach ($acc in $accounts) {
            $accName = ($acc.Group | Select-Object -First 1).Account
            [decimal]$impDeb = 0; [decimal]$impCred = 0; [decimal]$impNet = 0
            [decimal]$gpDeb = 0;  [decimal]$gpCred = 0;  [decimal]$gpNet = 0

            foreach ($row in $acc.Group) {
                if ($row.DebitAmt -as [decimal]) { $impDeb += [decimal]$row.DebitAmt }
                if ($row.CreditAmt -as [decimal]) { $impCred += [decimal]$row.CreditAmt }
                if ($row.NetAmt -as [decimal]) { $impNet += [decimal]$row.NetAmt }
                if ($row.gp_DebitAmount -as [decimal]) { $gpDeb += [decimal]$row.gp_DebitAmount }
                if ($row.gp_CreditAmount -as [decimal]) { $gpCred += [decimal]$row.gp_CreditAmount }
            }
            $gpNet = $gpDeb - $gpCred
            
            $sumImpDeb += $impDeb
            $sumImpCred += $impCred
            $sumImpNet += $impNet
            $sumGpDeb += $gpDeb
            $sumGpCred += $gpCred
            $sumGpNet += $gpNet

            $dRow = $detRowTpl
            $dRow = $dRow.Replace('{{ACCOUNT}}', "$accName")
            $dRow = $dRow.Replace('{{IMP_DEB}}', ("{0:N2}" -f $impDeb))
            $dRow = $dRow.Replace('{{IMP_CRED}}', ("{0:N2}" -f $impCred))
            $dRow = $dRow.Replace('{{IMP_NET}}', ("{0:N2}" -f $impNet))
            $dRow = $dRow.Replace('{{GP_DEB}}', ("{0:N2}" -f $gpDeb))
            $dRow = $dRow.Replace('{{GP_CRED}}', ("{0:N2}" -f $gpCred))
            $dRow = $dRow.Replace('{{GP_NET}}', ("{0:N2}" -f $gpNet))
            $detailRowsOut += $dRow
        }

        $detTpl = $detBlockTpl
        $detTpl = $detTpl.Replace('{{COMPANY}}', "$($first.CompanyA)")
        $detTpl = $detTpl.Replace('{{JE}}', "$($first.JournalEntry)")
        $detTpl = $detTpl.Replace('{{TRXDATE}}', "$trxDateStr")
        $detTpl = $detTpl.Replace('{{BATCH}}', "$($first.BatchID)")
        $detTpl = $detTpl.Replace('{{REF}}', "$($first.Reference)")
        $detTpl = $detTpl.Replace('{{JOBID}}', "$JobId")
        $detTpl = $detTpl.Replace('{{POSTED_MSG}}', "$injectPostedMsg")
        $detTpl = $detTpl.Replace('{{DETAIL_ROWS}}', "$detailRowsOut")
        $detTpl = $detTpl.Replace('{{TOT_IMP_DEB}}', ("{0:N2}" -f $sumImpDeb))
        $detTpl = $detTpl.Replace('{{TOT_IMP_CRED}}', ("{0:N2}" -f $sumImpCred))
        $detTpl = $detTpl.Replace('{{TOT_IMP_NET}}', ("{0:N2}" -f $sumImpNet))
        $detTpl = $detTpl.Replace('{{TOT_GP_DEB}}', ("{0:N2}" -f $sumGpDeb))
        $detTpl = $detTpl.Replace('{{TOT_GP_CRED}}', ("{0:N2}" -f $sumGpCred))
        $detTpl = $detTpl.Replace('{{TOT_GP_NET}}', ("{0:N2}" -f $sumGpNet))
        $detailTables += $detTpl
    }

    $finalHtml = $mainTpl
    $finalHtml = $finalHtml.Replace('{{FILENAME}}', "$fileName")
    $finalHtml = $finalHtml.Replace('{{STATUS}}', "$jobResult")
    $finalHtml = $finalHtml.Replace('{{JOBID}}', "$JobId")
    $finalHtml = $finalHtml.Replace('{{SUMMARY_ROWS}}', "$summaryRows")
    $finalHtml = $finalHtml.Replace('{{DETAIL_TABLES}}', "$detailTables")

    return $finalHtml
}
```
