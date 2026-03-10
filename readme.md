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
            $targetDir = $Paths.Archive
        }
        'PARTIAL' {
            $targetDir = if ($Paths.ContainsKey('Partial')) { $Paths.Partial } else { $Paths.ErrorDir }
        }
        default {
            $targetDir = $Paths.ErrorDir
        }
    }

    $moved = Move-File -Source $File.FullName -DestinationDir $targetDir
    if ($null -ne $moved) {
        Write-Log "File routed to: $targetDir based on status [$Status]"
    } else {
        Write-Log "ERROR: Failed to route file to $targetDir. File is stuck in the Proc folder."
    }
}
