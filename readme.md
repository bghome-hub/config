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
```}
.sum-table th { 
    border-bottom: 1px solid #c8d8c8; 
    background-color: #e2f0d9; 
}
.sum-table tr:nth-child(even) { background-color: #ffffff; }
.sum-table tr:nth-child(odd) { background-color: #f4f9f4; }
.det-header { 
    font-size: 13px; 
    font-weight: bold; 
    margin-bottom: 5px; 
}
.det-meta td:first-child { 
    font-weight: bold; 
    width: 90px; 
}
.det-table th { background-color: #f0f0f0; }
.center { text-align: center; }
.right { 
    text-align: right; 
    font-family: 'Consolas', monospace; 
}
.totals td { 
    border-top: 1px solid #333; 
    font-weight: bold; 
}
.posted-msg { 
    color: #d9534f; 
    font-weight: bold; 
    margin-bottom: 5px; 
}
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
    <tr>
        <td>Journal Entry</td>
        <td>{{JE}}</td>
    </tr>
    <tr>
        <td>TxDate</td>
        <td>{{TRXDATE}}</td>
    </tr>
    <tr>
        <td>BatchID</td>
        <td>{{BATCH}}</td>
    </tr>
    <tr>
        <td>Reference</td>
        <td>{{REF}}</td>
    </tr>
    <tr>
        <td>JobID</td>
        <td>{{JOBID}}</td>
    </tr>
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

### 2. PowerShell Script (Valid syntax, short lines)
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
    
    $mainTpl, $sumRowTpl, $detBlockTpl, $detRowTpl, $postedMsgTpl = 
        $rawHtml -split '@@@SPLIT@@@'

    $mainTpl      = $mainTpl.Trim()
    $sumRowTpl    = $sumRowTpl.Trim()
    $detBlockTpl  = $detBlockTpl.Trim()
    $detRowTpl    = $detRowTpl.Trim()
    $postedMsgTpl = $postedMsgTpl.Trim()

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
    
    if (-not $auditData) { 
        return "<h3>No data for JobID: $JobId</h3>" 
    }

    $firstGlobal = $auditData | Select-Object -First 1
    
    $fileName = "Unknown"
    if ($firstGlobal.FILENAME) { 
        $fileName = $firstGlobal.FILENAME 
    }
    
    $jobResult = "Unknown"
    if ($firstGlobal.STATUS) { 
        $jobResult = $firstGlobal.STATUS 
    }

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

        # Valid Sequential Reassignment (No Backticks Needed)
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

        $isPosted = ($first.STATUS -match 'POSTED') -or 
                    ([string]::IsNullOrWhiteSpace($first.gp_Journal) -and 
                    $first.STATUS -notmatch 'ERROR')
        
        $injectPostedMsg = ""
        if ($isPosted) { 
            $injectPostedMsg = $postedMsgTpl 
        }

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
            
            $sumImpDeb += $impDeb
            $sumImpCred += $impCred
            $sumImpNet += $impNet

            $sumGpDeb += $gpDeb
            $sumGpCred += $gpCred
            $sumGpNet += $gpNet

            $gpDebStr = "{0:N2}" -f $gpDeb
            $gpCredStr = "{0:N2}" -f $gpCred
            $gpNetStr = "{0:N2}" -f $gpNet

            if ($isPosted) {
                $gpDebStr = "-"
                $gpCredStr = "-"
                $gpNetStr = "-"
            }

            $dRow = $detRowTpl
            $dRow = $dRow.Replace('{{ACCOUNT}}', "$accName")
            $dRow = $dRow.Replace('{{IMP_DEB}}', ("{0:N2}" -f $impDeb))
            $dRow = $dRow.Replace('{{IMP_CRED}}', ("{0:N2}" -f $impCred))
            $dRow = $dRow.Replace('{{IMP_NET}}', ("{0:N2}" -f $impNet))
            $dRow = $dRow.Replace('{{GP_DEB}}', "$gpDebStr")
            $dRow = $dRow.Replace('{{GP_CRED}}', "$gpCredStr")
            $dRow = $dRow.Replace('{{GP_NET}}', "$gpNetStr")
            $detailRowsOut += $dRow
        }

        $totGpDebStr = "{0:N2}" -f $sumGpDeb
        $totGpCredStr = "{0:N2}" -f $sumGpCred
        $totGpNetStr = "{0:N2}" -f $sumGpNet

        if ($isPosted) {
            $totGpDebStr = "-"
            $totGpCredStr = "-"
            $totGpNetStr = "-"
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
        $detTpl = $detTpl.Replace('{{TOT_GP_DEB}}', "$totGpDebStr")
        $detTpl = $detTpl.Replace('{{TOT_GP_CRED}}', "$totGpCredStr")
        $detTpl = $detTpl.Replace('{{TOT_GP_NET}}', "$totGpNetStr")
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
```    </div>
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
