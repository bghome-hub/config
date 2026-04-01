function Route-ProcessedFile {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][System.IO.FileInfo]$File,
        [Parameter(Mandatory=$true)][string]$Status,
        [Parameter(Mandatory=$true)][Guid]$JobId
    )
    
    # 1. Determine the category (Archive, Partial, or Error)
    switch ($Status.ToUpperInvariant()) {
        'POSTED'  { $baseDir = $FSPaths.Archive }
        'PARTIAL' { $baseDir = if ($FSPaths.ContainsKey('Partial')) { $FSPaths.Partial } else { $FSPaths.Err } }
        default   { $baseDir = $FSPaths.Err }
    }

    # 2. Build the nested path: Category \ yyyy-MM-dd \ JobID
    $dateFolder = Get-Date -Format "yyyy-MM-dd"
    
    # Use $JobId.ToString().Substring(0,8) if you strictly want "part of it"
    $shortJobId = $JobId.ToString().Substring(0,8) 
    
    $targetDir = Join-Path -Path $baseDir -ChildPath $dateFolder
    $targetDir = Join-Path -Path $targetDir -ChildPath $shortJobId
    
    # 3. Move the file using your existing Move-File utility
    $moved = Move-File -Source $File.FullName -DestinationDir $targetDir
    
    if ($null -ne $moved) {
        # 4. LOCK THE FILE: Ensure no one can edit the evidence
        $moved.IsReadOnly = $true
        Write-Log "File routed and LOCKED in: $targetDir"
    } else {
        Write-Log "ERROR: Failed to route file to $targetDir." -Level "ERROR"
    }
}
