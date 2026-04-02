# ----------------------------
# CONFIGURATION & PATHS
# ----------------------------
# Note: Ensure $FSRoot and $FSMappedRoot are defined before this block.
$FSPaths = [PSCustomObject]@{
    POSTED     = [PSCustomObject]@{ Name = 'Archive';    UNC = Join-Path $FSRoot 'Archive';    Mapped = Join-Path $FSMappedRoot 'Archive' }
    PARTIAL    = [PSCustomObject]@{ Name = 'Partial';    UNC = Join-Path $FSRoot 'Partial';    Mapped = Join-Path $FSMappedRoot 'Partial' }
    ERROR      = [PSCustomObject]@{ Name = 'Error';      UNC = Join-Path $FSRoot 'Error';      Mapped = Join-Path $FSMappedRoot 'Error' }
    INCOMING   = [PSCustomObject]@{ Name = 'Incoming';   UNC = Join-Path $FSRoot 'Incoming';   Mapped = Join-Path $FSMappedRoot 'Incoming' }
    PROCESSING = [PSCustomObject]@{ Name = 'Processing'; UNC = Join-Path $FSRoot 'Processing'; Mapped = Join-Path $FSMappedRoot 'Processing' }
}

# ----------------------------
# UTILITIES
# ----------------------------

function Ensure-Directory {
    param([Parameter(Mandatory = $true)][string]$Path)
    if (-not (Test-Path -Path $Path)) {
        # Removed .ToLower() - Keep casing as defined in config for compatibility/readability
        New-Item -ItemType Directory -Path $Path -Force | Out-Null
    }
}

function Write-Log {
    param(
        [Parameter(Mandatory=$true)][string]$Message,
        [switch]$IncludeHeader,
        [switch]$IncludeFooter,
        [ValidateSet("INFO", "WARN", "ERROR")][string]$Level = "INFO",
        [string]$Module = "System"
    )

    $TimeStamp = Get-Date -Format "yyyyMMdd HH:mm:ss"
    
    if($IncludeHeader){
        $LogEntry = "$TimeStamp | User: $env:USERNAME | Host: $env:COMPUTERNAME`n$TimeStamp | $Message"
    } else {
        $LogEntry = "$TimeStamp | $Level | $Module: $Message"
    }

    if($IncludeFooter){ $LogEntry += "`n" }

    # Ensure log directory exists before writing
    Ensure-Directory -Path (Split-Path $ITPaths.Log -Parent)
    Add-Content -Path $ITPaths.Log -Value $LogEntry
}

function Send-JobEmail {
    param(
        [Parameter(Mandatory=$true)][string]$Subject,
        [Parameter(Mandatory=$true)][string]$Body
    )
    
    $EmailParams = @{
        SMTPServer = $EmailConfig.Server
        From       = $EmailConfig.From
        To         = $EmailConfig.To
        Bcc        = $EmailConfig.Bcc
        Subject    = $Subject
        Body       = $Body
        BodyAsHTML = $true
        ErrorAction = 'Stop'
    }

    try {
        Send-MailMessage @EmailParams
    } catch {
        Write-Log "Failed to send email: $($_.Exception.Message)" -Level "ERROR"
    }
}

# ----------------------------
# DATA & PROCESSING
# ----------------------------

function Get-JobResult {
    param([Parameter(Mandatory)][guid]$JobId)

    $Query = "SELECT JobStatus FROM dbo.IC_ImportJob WHERE JobID = '$($JobId.ToString())'"
    $QueryResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate

    if ($null -eq $QueryResult) { return "NOT_FOUND" }
    return $QueryResult.JobStatus.ToString().Trim().ToUpper()
}

function Move-File {
    param(
        [Parameter(Mandatory = $true)][string]$Source,
        [Parameter(Mandatory = $true)][string]$DestinationDir
    )
    
    Ensure-Directory -Path $DestinationDir
    $FileName = Split-Path -Path $Source -Leaf
    $TargetFile = Join-Path -Path $DestinationDir -ChildPath $FileName
    
    $retryCount = 0
    while ($retryCount -lt 3) {
        try {
            # Use -PassThru to return the file object for the next step
            return Move-Item -Path $Source -Destination $TargetFile -PassThru -Force -ErrorAction Stop
        }
        catch {
            $retryCount++
            Write-Log "Move attempt $retryCount failed for [$FileName]. Retrying..." -Level "WARN"
            Start-Sleep -Seconds 2
        }
    }
    
    Write-Log "FATAL: Could not move [$FileName] to [$DestinationDir] after 3 attempts." -Level "ERROR"
    Send-JobEmail -Subject "File System Error: $FileName" -Body "Could not move file to $DestinationDir. Check permissions or locks."
    return $null 
}

function Route-ProcessedFile {
    param(
        [Parameter(Mandatory)][System.IO.FileInfo]$File,
        [Parameter(Mandatory)][string]$Status,
        [Parameter(Mandatory)][guid]$JobId
    )

    $BranchInfo = switch ($Status.ToUpper()) {
        'POSTED'  { $FSPaths.POSTED }
        'PARTIAL' { $FSPaths.PARTIAL }
        default   { $FSPaths.ERROR }
    }

    # Use .ToString().Substring to keep the folder names clean
    $SubFolder = Join-Path (Get-Date -Format 'yyyy-MM-dd') ($JobId.ToString().Substring(0,8))
    $TargetDir = Join-Path $BranchInfo.UNC $SubFolder

    Write-Log "Routing file to: $TargetDir"
    return Move-File -Source $File.FullName -DestinationDir $TargetDir
}

# ----------------------------
# MAIN EXECUTION
# ----------------------------

function Invoke-Main {
    $Module = "Main"
    Write-Log "Starting Import Cycle" -IncludeHeader -Module $Module

    # 1. Validation & Setup
    foreach ($Prop in $FSPaths.PSObject.Properties) {
        Ensure-Directory -Path $Prop.Value.UNC
    }

    # 2. Scan Incoming
    $Files = @(Get-ChildItem -Path $FSPaths.INCOMING.UNC -Filter '*.csv' -File -ErrorAction SilentlyContinue)
    if ($Files.Count -eq 0) {
        Write-Log "No files found in Incoming." -Module $Module
        return
    }

    foreach ($File in $Files) {
        Write-Log "Processing: $($File.Name)" -Module $Module
        
        # 3. Move to Processing (Isolate)
        $WorkingFile = Move-File -Source $File.FullName -DestinationDir $FSPaths.PROCESSING.UNC
        if ($null -eq $WorkingFile) { continue }

        try {
            # 4. SQL Import
            $JobId = Invoke-IntracompanyImportSP -CsvFullPath $WorkingFile.FullName
            if ($null -eq $JobId -or $JobId -eq "No JobID returned") { 
                throw "SQL Import failed to return a valid JobId." 
            }

            # 5. Get Result & Determine Email
            $JobStatus = Get-JobResult -JobId $JobId
            
            if ($JobStatus -in @('POSTED', 'PARTIAL')) {
                $EmailBody = New-DynamicsImportSuccessEmail -JobId $JobId
            } else {
                $EmailBody = New-DynamicsImportErrorEmail -JobId $JobId -FileName $WorkingFile.Name
            }

            # 6. Route File & Send Notification
            Route-ProcessedFile -File $WorkingFile -Status $JobStatus -JobId $JobId
            Send-JobEmail -Subject "GP Import $JobStatus: $($WorkingFile.Name)" -Body $EmailBody
            
            Write-Log "Successfully processed $($WorkingFile.Name) with status $JobStatus" -Module $Module
        }
        catch {
            $ErrorMessage = $_.Exception.Message
            Write-Log "CRITICAL ERROR processing $($File.Name): $ErrorMessage" -Level "ERROR" -Module $Module
            
            # Ensure file doesn't sit in Processing forever on crash
            Route-ProcessedFile -File $WorkingFile -Status "ERROR" -JobId ([guid]::Empty)
            
            $ErrBody = "The process crashed while handling $($File.Name).<br>Error: $ErrorMessage"
            Send-JobEmail -Subject "GP Import CRASHED: $($File.Name)" -Body $ErrBody
        }
    }
    Write-Log "Import Cycle Complete" -IncludeFooter -Module $Module
}

# Run the script
Invoke-Main
