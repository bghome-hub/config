# ... inside the foreach ($File in $Files) loop ...

try {
    $JobId = Invoke-IntracompanyImportSP -CsvFullPath $WorkingFile.FullName
    if ($null -eq $JobId -or $JobId -eq "No JobID returned") { throw "SQL Import failed." }

    $JobStatus = Get-JobResult -JobId $JobId
    
    # 1. Route the file FIRST to get the final path
    $RoutingResult = Route-ProcessedFile -File $WorkingFile -Status $JobStatus -JobId $JobId
    $UserPath = if ($null -ne $RoutingResult) { $RoutingResult.MappedPath } else { "Unknown (Move Failed)" }

    # 2. Pass that path into your email functions
    if ($JobStatus -in @('POSTED', 'PARTIAL')) {
        # Update your Success email function to accept -MappedPath
        $EmailBody = New-DynamicsImportSuccessEmail -JobId $JobId -MappedPath $UserPath
    } else {
        # Update your Error email function to accept -MappedPath
        $EmailBody = New-DynamicsImportErrorEmail -JobId $JobId -FileName $WorkingFile.Name -MappedPath $UserPath
    }

    Send-JobEmail -Subject "GP Import $JobStatus: $($WorkingFile.Name)" -Body $EmailBody
}
catch {
    # Fallback error handling...
}
