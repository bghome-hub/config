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

    $SubFolder  = Join-Path (Get-Date -Format 'yyyy-MM-dd') ($JobId.ToString().Substring(0,8))
    $TargetDir  = Join-Path $BranchInfo.UNC $SubFolder
    $MappedDir  = Join-Path $BranchInfo.Mapped $SubFolder # This is the user-friendly path
    
    # Perform the move
    $MovedFile = Move-File -Source $File.FullName -DestinationDir $TargetDir

    if ($null -ne $MovedFile) {
        # Return an object so the main loop has the Mapped Path for the email
        return [PSCustomObject]@{
            File       = $MovedFile
            MappedPath = Join-Path $MappedDir $MovedFile.Name
        }
    }
    return $null
}
