
$ITPaths = @{
    BaseITDir       = $ITDir
    SuccessTemplate = Join-Path $ITDir 'html\success_template.html'
    ErrorTemplate   = Join-Path $ITDir 'html\error_template.html'
    Log             = Join-Path $ITDir 'log\log.txt'
}


$FSPaths = @{
    BaseFSDir = $FSDir
    In        = Join-Path $FSDir 'Incoming'
    Proc      = Join-Path $FSDir 'Processing'
    Archive   = Join-Path $FSDir 'Archive'
    Err       = Join-Path $FSDir 'Error'
    Partial   = Join-Path $FSDir 'Partial'
}

# ----------------------------
# UTILITIES
# ----------------------------

function Ensure-Directory {
    param([Parameter(Mandatory = $true)][string]$Path)
    if (-not (Test-Path -Path $Path)) {
        New-Item -ItemType Directory -Path $($Path).ToLower() -Force | Out-Null
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
        Bcc        = $EmailConfig.Bcc
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
        [switch]$IncludeFooter,
        [ValidateSet("INFO", "WARN", "ERROR")][string]$Level = "INFO",
        [string]$Module
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
        $LogEntry = "$TimeStamp | $Level | $Module`: $Message"
    }

    # Add linebreak
    if($IncludeFooter){
        $LogEntry += "`n"
    }

    Add-Content -Path $ITPaths.Log -Value $LogEntry
}
function Move-File {
    param(
        [Parameter(Mandatory = $true)][string]$Source,
        [Parameter(Mandatory = $true)][string]$DestinationDir
    )
    
    Ensure-Directory -Path $DestinationDir
    $target = Join-Path -Path $DestinationDir -ChildPath (Split-Path -Path $Source -Leaf)
    
    $retryCount = 0
    while ($retryCount -lt 3) {
        try {
            return Move-Item -Path $Source -Destination $target -PassThru -Force -ErrorAction Stop
        }
        catch {
            $retryCount++
            Write-Log "Move attempt $retryCount failed for [$Source]. File may be locked. Retrying in 2 seconds..."
            Start-Sleep -Seconds 2
        }
    }
    
    Write-Log "FATAL: Could not move file [$Source] to [$DestinationDir] after 3 attempts. Ensure the file is not open in Excel."
        $Subject = "GP Import Error: $($Source)"
        $Body = "Failed to process file: Could not move file [ $($Source)] to [$($DestinationDir)] after 3 attempts. Ensure the file is not open in Excel."
        Send-JobEmail -Subject $Subject -Body $Body

    return $null 
}

function Route-ProcessedFile {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][System.IO.FileInfo]$File,
        [Parameter(Mandatory=$true)][string]$Status
    )
    
    switch ($Status.ToUpperInvariant()) {
        'POSTED' {
            $targetDir = $FSPaths.Archive
        }
        'PARTIAL' {
            $targetDir = if ($FSPaths.ContainsKey('Partial')) { $FSPaths.Partial } else { $FSPaths.Err }
        }
        default {
            $targetDir = $FSPaths.Err
        }
    }

    $moved = Move-File -Source $File.FullName -DestinationDir $targetDir
    if ($null -ne $moved) {
        Write-Log "File routed to: $targetDir based on status [$Status]"
    } else {
        Write-Log "ERROR: Failed to route file to $targetDir. File is stuck in the Proc folder."
    }
}


function New-DynamicsImportErrorEmail {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][Guid]$JobId,
        [Parameter(Mandatory=$true)][string]$FileName
    )
    Write-Log "Generating Error HTML Body for JobID: $JobId"
    
    $Query = "SELECT TOP 1 LogMessage FROM dbo.IC_SystemLog WHERE JobId = '$JobId' AND LogLevel IN ('FATAL', 'ERROR') ORDER BY LogId ASC"
    $SqlResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate
    $FatalError = if ($null -ne $SqlResult) { $SqlResult.LogMessage } else { "Unknown fatal error occurred. Check system logs." }

    $HtmlTemplate = Get-Content $ITPaths.ErrorTemplate -Raw
    return $HtmlTemplate -replace '{{JobId}}', $JobId `
                         -replace '{{FileName}}', $FileName `
                         -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                         -replace '{{ErrorMessage}}', $FatalError
}

function New-DynamicsImportSuccessEmail {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][Guid]$JobId
    )
    Write-Log "Generating Report HTML Body for JobID: $JobId"
    
    $Query = "EXEC $($SQLConfig.ReportSP) @JobId = '$JobId'"
    $Details = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    # Build Summary Rows
    $SummaryGroups = $Details | Group-Object Company, gp_BatchNumber, gp_Status
    $SummaryHtml = ""
    foreach ($Group in $SummaryGroups) {
    $FirstRow = $Group.Group[0]
    $CleanStatus = if ($null -ne $FirstRow.gp_Status) { $FirstRow.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
    $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }

    # The Indicator: Shows 'Y (MM/DD/YY)' if reversal, or 'N' if not [cite: 153, 155]
    $RevIndicator = if ($FirstRow.Reversal -eq 'Y') { 
        "Y ($($FirstRow.ReversalDate.ToString('MM/dd/yy')))" 
    } else { 
        "N" 
    }

    $SummaryHtml += "<tr> 
        <td>$($FirstRow.Company)</td> 
        <td>$($FirstRow.gp_BatchNumber)</td> 
        <td class='center'>$RevIndicator</td> 
        <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> 
        <td class='right'>$($Group.Count)</td> 
    </tr>`n"
}

    # Build Detail Rows
    $DetailHtml = ""
    $GroupedDetails = $Details | Group-Object Company
    foreach ($CompGroup in $GroupedDetails) {
        $DetailHtml += "<tr class='company-header'><td colspan='9'>Company: $($CompGroup.Name)</td></tr>`n"
        $SubUpDeb = 0; $SubUpCred = 0; $SubGpDeb = 0; $SubGpCred = 0; $SubVar = 0

        foreach ($Row in $CompGroup.Group) {
            $SubUpDeb += $Row.Uploaded_Debit; $SubUpCred += $Row.Uploaded_Credit
            $SubGpDeb += $Row.gp_DebitAmount; $SubGpCred += $Row.gp_CreditAmount; $SubVar += $Row.Net_Variance

            $CleanRowStatus = if ($null -ne $Row.gp_Status) { $Row.gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
            $StatusClass = switch ($CleanRowStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
            $VarClass = if ($Row.Net_Variance -ne 0) { "variance-bad right" } else { "variance-good right" }

            $DetailHtml += "<tr> <td>$($Row.Company)</td> `
                                 <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td> `
                                 <td>$($Row.Account)</td> `
                                 <td class='right'>$("{0:N2}" -f $Row.Uploaded_Debit)</td> `
                                 <td class='right'>$("{0:N2}" -f $Row.Uploaded_Credit)</td> `
                                 <td class='right'>$("{0:N2}" -f $Row.gp_DebitAmount)</td> `
                                 <td class='right'>$("{0:N2}" -f $Row.gp_CreditAmount)</td> `
                                 <td class='$VarClass'>$("{0:N2}" -f $Row.Net_Variance)</td> `
                                 <td class='center'><span class='$StatusClass'>$CleanRowStatus</span></td> `
                            </tr>`n"
        }
        $DetailHtml +=      "<tr class='subtotal-row'> <td colspan='3' class='right'>$($CompGroup.Name) Totals:</td> `
                                <td class='right'>$("{0:N2}" -f $SubUpDeb)</td> `
                                <td class='right'>$("{0:N2}" -f $SubUpCred)</td> `
                                <td class='right'>$("{0:N2}" -f $SubGpDeb)</td> `
                                <td class='right'>$("{0:N2}" -f $SubGpCred)</td> `
                                <td class='right'>$("{0:N2}" -f $SubVar)</td> <td></td> `
                                </tr>`n"
    }

    $HtmlTemplate = Get-Content $ITPaths.SuccessTemplate -Raw
    return $HtmlTemplate -replace '{{JobId}}', $JobId `
                         -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                         -replace '{{SummaryRows}}', $SummaryHtml `
                         -replace '{{DetailRows}}', $DetailHtml
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

function Invoke-Main {

    # Module Definition
    $Module = "Invoke-Main"

    # Make sure all directories exist before any operation
    Ensure-Directory -Path (Split-Path $ITPaths.Log -Parent)

    $FSPaths.GetEnumerator() | ForEach-Object { Ensure-Directory -Path $_.Value }

    $ITPaths.GetEnumerator() | ForEach-Object {
        $val = $_.Value
        $dir = if (Test-Path $val -PathType Container) { $val } else { Split-Path $val -Parent }
        if ($dir) { Ensure-Directory -Path $dir }
    }
    
    # 1. Scan for files
    $files = @(Get-ChildItem -Path $FSPaths.In -Filter *.csv -File -ErrorAction SilentlyContinue)
    if ($files.Count -eq 0) { return }
    
    
    Write-Log "Files(s) found. Move each to Processing folder and begin import." -Module $Module

    foreach ($file in $files) {
        Write-Log "Found file: $($file.Name)" -Module $Module
        
        # 1.5 Isolate the file to prevent locking/double-processing
        $working = Move-File -Source $file.FullName -DestinationDir $FSPaths.Proc
        if ($null -eq $working) {
                Write-Log "Skipping: file locked/in use or failed to move. File: $($file.Name)" -Module $Module
            continue
        }
        try {
                Write-Log "Starting SQL import for: $($working.FullName)" -Module $Module
            
            # 2 & 3. Run the file and get the Job ID back
                Write-Log "Calling Invoke-IntracompanyImport for $working.FullName" -Module $Module
            $jobId = Invoke-IntracompanyImportSP -CsvFullPath $working.FullName
            if (-not $jobId) { throw "No JobID returned from SQL." }
                Write-Log "Received JobID: $JobID" -Module $Module

            # 4. Check if it ran success/partial/fail
                Write-Log "Calling Get-JobResult for $JobID" -Module $Module
            $jobResult = Get-JobResult -JobID $jobId
            if (-not $jobResult) { $jobResult = "FAILED" }
                Write-Log "Received Result: $JobResult" -Module $Module

                Write-Log "Calling New-DynamicsImport. Determining which Email to Send" -Module $Module
            # 5. Go to the function that creates the appropriate email
            if ($jobResult -in 'POSTED', 'PARTIAL') {
                Write-Log "Calling SuccessEmail for JobID: $JobID" -Module $Module
                $emailBody = New-DynamicsImportSuccessEmail -JobId $jobId
            } else {
                Write-Log "Calling FailedEmail for JobID: $JobID" -Module $Module
                $emailBody = New-DynamicsImportErrorEmail -JobId $jobId -FileName $working.Name
            }
                Write-Log "Email should be determined." -Module $Module

            # 6. Move file
                Write-Log "Calling Route-ProcessedFile" -Module $Module
            Route-ProcessedFile -File $working -Status $jobResult

            # 7. Send email
                Write-Log "Calling Send-JobEmail" -Module $Module
            $subject = "GP Import $($jobResult): $($working.Name)"
            Send-JobEmail -Subject $subject -Body $emailBody

        } catch {
            Write-Log "Unhandled failure for $($file.Name): $($_.Exception.Message)" -Level "ERROR" -Module $Module
            Route-ProcessedFile -File $working -Status "FAILED"
            
            # Fallback email if the pipeline completely explodes
            $emailBody = New-DynamicsImportErrorEmail -JobId ([Guid]::Empty) -FileName $working.Name
            
            Send-JobEmail -Subject "GP Import CRASHED: $($working.Name) - Are the procedures compiled?" -Body $emailBody
        }
    }
    
    Write-Log "-- Import End --" -IncludeFooter -Module $Module
}

Invoke-Main
