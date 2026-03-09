
function Ensure-Directory {
    param([Parameter(Mandatory = $true)][string]$Path)
    if (-not (Test-Path -Path $Path)) {
        New-Item -ItemType Directory -Path $Path -Force | Out-Null
    }
}
function Send-JobEmail{
    param(
        [Parameter(Mandatory=$true)][String]$Subject,
        [Parameter(Mandatory=$true)][String]$Body

    )
    
    $EmailParams = @{
        SMTPServer = $EmailConfig.Server
        From       = $EmailConfig.From
        To         = $EmailConfig.To
        #CC         = $CC
        Subject    = $Subject
        Body       = $Body
        BodyAsHTML = $true
    }

    Send-MailMessage @EmailParams
}

function Get-JobResult{
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    $Query = "SELECT JobStatus FROM dbo.IC_ImportJob WHERE JobID= '$JobID'"
    $QueryResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    $JobStatus = $($QueryResult.JobStatus).ToString().Trim().ToUpper()

    return $JobStatus

}


function Write-Log{
    param(
        [Parameter(Mandatory=$true)]
        [String]$Message,
        [switch]$IncludeHeader,
        [switch]$IncludeFooter
    )

    $TimeStamp = Get-Date -Format "yyyyMMdd HH:mm:ss"
    
    if($IncludeHeader){
        # Log Username, Clustername, NodeName
        $User = $env:USERNAME
        $Hostname = $env:COMPUTERNAME
        $Node = [System.Environment]::MachineName
        $LogEntry = "$TimeStamp | User: $User | Hostname: $Hostname | Node: $Node `n$Timestamp | $Message"
    } else {
        # Log Message Only
        $LogEntry = "$TimeStamp | $Message"
    }

    # Add linebreak
    if($IncludeFooter){
        $LogEntry += "`n"
    }

    Add-Content -Path $Paths.Log -Value $LogEntry
}
function Move-File {
    param(
        [Parameter(Mandatory = $true)][string]$Source,
        [Parameter(Mandatory = $true)][string]$DestinationDir
    )
    try {
        Ensure-Directory -Path $DestinationDir
        $target = Join-Path -Path $DestinationDir -ChildPath (Split-Path -Path $Source -Leaf)
        return Move-Item -Path $Source -Destination $target -PassThru -Force -ErrorAction Stop
    }
    catch {
        Write-Log "Failed to move file [$Source] to [$DestinationDir]. Error: $($_.Exception.Message)"
        return $null
    }
}

function Invoke-IntracompanyImportSP {
    param([Parameter(Mandatory = $true)][string]$CsvFullPath)

    Write-Log "Beginning Main Procedure Call"

    $query = @"
        DECLARE @GeneratedJobID UNIQUEIDENTIFIER;

        EXEC $($SQLConfig.MainSP) 
          @CSVPath = '$CsvFullPath',
          @JobID = @GeneratedJobID OUTPUT;

        SELECT @GeneratedJobID AS JobID
"@
    
    Write-Log "Executing Query: $Query"

    try {

        $SQLResult = @(Invoke-Sqlcmd -ServerInstance $SQLConfig.server -Database $SQLConfig.database -Query $query -TrustServerCertificate)
        $JobID = $SQLResult.JobID
        if ($null -eq $JobID -or $SQLResult.Count -eq 0) {
            Write-Log "Did not find JobID - Result count: $($SQLResult.Count)"
            return "No JobID returned"
        }
        else {
            Write-Log "Found JobID: $JobID"
            return $JobID
        }   
    }
    catch {
        Write-Log "SQL/eConnect Error while processing [$CsvFullPath]: $($_.Exception.Message)"
        return $null
    }
}

function New-DynamicsImportJobEmailContent {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)][Guid]$JobId
    )

    # 1. Execute the stored procedure to get the raw data
    $Query = "EXEC $($SQLConfig.ReportSP) @JobId = '$JobId'"
    $Details = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    # ==============================================================================
    # 2. Build the Summary Rows
    # ==============================================================================
    $SummaryGroups = $Details | Group-Object Company, gp_BatchNumber, gp_Status
    $SummaryHtml = ""

    foreach ($Group in $SummaryGroups) {
        $Comp = $Group.Group[0].Company
        $Batch = $Group.Group[0].gp_BatchNumber
        $LineCount = $Group.Count

    # SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
        $RawStatus = $Group.Group[0].gp_Status
        $CleanStatus = if ($null -ne $RawStatus) { $RawStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

        # Strict Switch Evaluation
        switch ($CleanStatus) {
            'POSTED'   { $StatusClass = 'status-ok'; break }
            'UNPOSTED' { $StatusClass = 'status-pending'; break }
            default    { $StatusClass = 'status-fail'; break }
        }


        # Strict array evaluation
        $StatusClass = if ($CleanStatus -in 'POSTED', 'UNPOSTED') { "status-ok" } else { "status-fail" }

        $SummaryHtml += "<tr>
            <td>$Comp</td>
            <td>$Batch</td>
            <td class='center'><span class='$StatusClass'>$CleanStatus</span></td>
            <td class='right'>$LineCount</td>
        </tr>`n"
    }

    # ==============================================================================
    # 3. Build the Detail Rows (Grouped by Company with Subtotals)
    # ==============================================================================
    
    
    
    $DetailHtml = ""
    $GroupedDetails = $Details | Group-Object Company

    foreach ($CompGroup in $GroupedDetails) {
        $CompanyName = $CompGroup.Name
        
        $DetailHtml += "<tr class='company-header'><td colspan='9'>Company: $CompanyName</td></tr>`n"

        $SubUpDeb = 0; $SubUpCred = 0; $SubGpDeb = 0; $SubGpCred = 0; $SubVar = 0

        foreach ($Row in $CompGroup.Group) {
            $SubUpDeb  += $Row.Uploaded_Debit
            $SubUpCred += $Row.Uploaded_Credit
            $SubGpDeb  += $Row.gp_DebitAmount
            $SubGpCred += $Row.gp_CreditAmount
            $SubVar    += $Row.Net_Variance

            $UpDeb  = "{0:N2}" -f $Row.Uploaded_Debit
            $UpCred = "{0:N2}" -f $Row.Uploaded_Credit
            $GpDeb  = "{0:N2}" -f $Row.gp_DebitAmount
            $GpCred = "{0:N2}" -f $Row.gp_CreditAmount
            $Var    = "{0:N2}" -f $Row.Net_Variance

        # SANITIZE: Handle Nulls, strip trailing SQL spaces, and force uppercase
            $RawRowStatus = $Row.gp_Status
            $CleanRowStatus = if ($null -ne $RawRowStatus) { $RawRowStatus.ToString().Trim().ToUpper() } else { "UNKNOWN" }

            # Strict Switch Evaluation
            switch ($CleanRowStatus) {
                'POSTED'   { $StatusClass = 'status-ok'; break }
                'UNPOSTED' { $StatusClass = 'status-pending'; break }
                default    { $StatusClass = 'status-fail'; break }
            }

            # Strict array evaluation
            $StatusClass = if ($CleanRowStatus -in 'POSTED', 'UNPOSTED') { "status-ok" } else { "status-fail" }
            
            $VarClass = if ($Row.Net_Variance -ne 0) { "variance-bad right" } else { "variance-good right" }

            $DetailHtml += "<tr>
                <td>$($Row.Company)</td>
                <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td>
                <td>$($Row.Account)</td>
                <td class='right'>$UpDeb</td>
                <td class='right'>$UpCred</td>
                <td class='right'>$GpDeb</td>
                <td class='right'>$GpCred</td>
                <td class='$VarClass'>$Var</td>
                <td class='center'><span class='$StatusClass'>$CleanRowStatus</span></td>
            </tr>`n"
        }

        $DetailHtml += "<tr class='subtotal-row'>
            <td colspan='3' class='right'>$CompanyName Totals:</td>
            <td class='right'>$("{0:N2}" -f $SubUpDeb)</td>
            <td class='right'>$("{0:N2}" -f $SubUpCred)</td>
            <td class='right'>$("{0:N2}" -f $SubGpDeb)</td>
            <td class='right'>$("{0:N2}" -f $SubGpCred)</td>
            <td class='right'>$("{0:N2}" -f $SubVar)</td>
            <td></td>
        </tr>`n"
    }



    # 4. Load Template, Swap Tokens, and Save/Send
    $HtmlTemplate = Get-Content $Paths.EmailTemplate -Raw

    # Replace the tokens with our generated rows and metadata
    $FinalHtml = $HtmlTemplate -replace '{{JobId}}', $JobId `
                            -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                            -replace '{{SummaryRows}}', $SummaryHtml `
                            -replace '{{DetailRows}}', $DetailHtml


    Return $FinalHtml
}

# ----------------------------
# MAIN
# ----------------------------
function Invoke-Main {

    $Paths.Keys | ForEach-Object {
        if ($_ -ne 'Log') { Ensure-Directory -Path $Paths[$_] }
    }

    $files = @(Get-ChildItem -Path $Paths.In -Filter *.csv -File -ErrorAction SilentlyContinue)
    if ($files.Count -eq 0) { return }

    Write-Log "-- Import Begin --" -IncludeHeader

    foreach ($file in $files) {
        Write-Log "Found file: $($file.Name)" 

        $working = Move-File -Source $file.FullName -DestinationDir $Paths.Proc
        if ($null -eq $working) {
            Write-Log "Skipping: file locked/in use or failed to move. File: $($file.Name)"
            continue
        }
        
        try {
            Write-Log "Starting SQL import for: $($working.FullName)" 

            $jobId = Invoke-IntracompanyImportSP -CsvFullPath $working.FullName
            if (-not $jobId) {
                Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                Write-Log "No JobID returned. Moved to error: $($file.Name)" 
                continue
            }

            $jobResult = Get-JobResult -JobID $jobId
            if (-not $jobResult) {
                Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                Write-Log "JobResult not found for JobID [$jobId]. Moved to error: $($file.Name)"
                continue
            }

            switch ($jobResult.ToUpperInvariant()) {
                'POSTED' {
                    $archived = Move-File -Source $working.FullName -DestinationDir $Paths.Archive
                    if ($null -ne $archived) {
                        Write-Log "Success: $($file.Name) → archived to $($Paths.Archive)"
                    }
                }
                'PARTIAL' {
                    $partialDir = if ($Paths.ContainsKey('Partial')) { $Paths.Partial } else { $Paths.ErrorDir }
                    Move-File -Source $working.FullName -DestinationDir $partialDir | Out-Null
                    Write-Log "Partial result for JobID [$jobId]. Moved to: $partialDir"
                }
                default {
                    Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
                    Write-Log "Error result for JobID [$jobId]. Moved to error: $($file.Name)" 
                }
            }

        }
        catch {
            Move-File -Source $working.FullName -DestinationDir $Paths.ErrorDir | Out-Null
            Write-Log "Unhandled failure for $($file.Name): $($_.Exception.Message)"
        }
    }

    Write-Log "Composing Email"
    $subject = "GP Import Result"
    $Body = New-DynamicsImportJobEmailContent -JobId $jobId

    Send-JobEmail -Subject $Subject -Body $Body
    
    write-log "Email Sent"
    Write-Log "-- Import End --" -IncludeFooter
}
Invoke-Main
