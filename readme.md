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

    Move-File -Source $File.FullName -DestinationDir $targetDir | Out-Null
    Write-Log "File routed to: $targetDir based on status [$Status]"
}

function New-DynamicsImportErrorEmail {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][Guid]$JobId,
        [Parameter(Mandatory=$true)][string]$FileName
    )
    Write-Log "Generating Error HTML Body for JobID: $JobId"
    
    $Query = "SELECT TOP 1 LogMessage FROM dbo.IC_SystemLog WHERE JobId = '$JobId' AND LogLevel = 'FATAL' ORDER BY LogId DESC"
    $SqlResult = Invoke-Sqlcmd -ServerInstance $SQLConfig.Server -Database $SQLConfig.Database -Query $Query -TrustServerCertificate
    $FatalError = if ($null -ne $SqlResult) { $SqlResult.LogMessage } else { "Unknown fatal error occurred. Check system logs." }

    $HtmlTemplate = Get-Content $Paths.ErrorTemplate -Raw
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
        $CleanStatus = if ($null -ne $Group.Group[0].gp_Status) { $Group.Group[0].gp_Status.ToString().Trim().ToUpper() } else { "UNKNOWN" }
        $StatusClass = switch ($CleanStatus) { 'POSTED' {'status-ok'} 'UNPOSTED' {'status-pending'} default {'status-fail'} }
        $SummaryHtml += "<tr> <td>$($Group.Group[0].Company)</td> <td>$($Group.Group[0].gp_BatchNumber)</td> <td class='center'><span class='$StatusClass'>$CleanStatus</span></td> <td class='right'>$($Group.Count)</td> </tr>`n"
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

            $DetailHtml += "<tr> <td>$($Row.Company)</td> <td>$($Row.gp_BatchNumber) / JE: $($Row.gp_Journal)</td> <td>$($Row.Account)</td> <td class='right'>$("{0:N2}" -f $Row.Uploaded_Debit)</td> <td class='right'>$("{0:N2}" -f $Row.Uploaded_Credit)</td> <td class='right'>$("{0:N2}" -f $Row.gp_DebitAmount)</td> <td class='right'>$("{0:N2}" -f $Row.gp_CreditAmount)</td> <td class='$VarClass'>$("{0:N2}" -f $Row.Net_Variance)</td> <td class='center'><span class='$StatusClass'>$CleanRowStatus</span></td> </tr>`n"
        }
        $DetailHtml += "<tr class='subtotal-row'> <td colspan='3' class='right'>$($CompGroup.Name) Totals:</td> <td class='right'>$("{0:N2}" -f $SubUpDeb)</td> <td class='right'>$("{0:N2}" -f $SubUpCred)</td> <td class='right'>$("{0:N2}" -f $SubGpDeb)</td> <td class='right'>$("{0:N2}" -f $SubGpCred)</td> <td class='right'>$("{0:N2}" -f $SubVar)</td> <td></td> </tr>`n"
    }

    $HtmlTemplate = Get-Content $Paths.EmailTemplate -Raw
    return $HtmlTemplate -replace '{{JobId}}', $JobId `
                         -replace '{{RunDate}}', (Get-Date -Format 'yyyy-MM-dd HH:mm:ss') `
                         -replace '{{SummaryRows}}', $SummaryHtml `
                         -replace '{{DetailRows}}', $DetailHtml
}


function Invoke-Main {
    $Paths.Keys | ForEach-Object {
        if ($_ -ne 'Log') { Ensure-Directory -Path $Paths[$_] }
    }

    # 1. Scan for files
    $files = @(Get-ChildItem -Path $Paths.In -Filter *.csv -File -ErrorAction SilentlyContinue)
    if ($files.Count -eq 0) { return }

    Write-Log "-- Import Begin --" -IncludeHeader

    foreach ($file in $files) {
        Write-Log "Found file: $($file.Name)"
        
        # 1.5 Isolate the file to prevent locking/double-processing
        $working = Move-File -Source $file.FullName -DestinationDir $Paths.Proc
        if ($null -eq $working) {
            Write-Log "Skipping: file locked/in use or failed to move. File: $($file.Name)"
            continue
        }

        try {
            Write-Log "Starting SQL import for: $($working.FullName)"
            
            # 2 & 3. Run the file and get the Job ID back
            $jobId = Invoke-IntracompanyImportSP -CsvFullPath $working.FullName
            if (-not $jobId) { throw "No JobID returned from SQL." }

            # 4. Check if it ran success/partial/fail
            $jobResult = Get-JobResult -JobID $jobId
            if (-not $jobResult) { $jobResult = "FAILED" }

            # 5. Go to the function that creates the appropriate email
            if ($jobResult -in 'POSTED', 'PARTIAL') {
                $emailBody = New-DynamicsImportSuccessEmail -JobId $jobId
            } else {
                $emailBody = New-DynamicsImportErrorEmail -JobId $jobId -FileName $working.Name
            }

            # 6. Move file
            Route-ProcessedFile -File $working -Status $jobResult

            # 7. Send email
            $subject = "GP Import $($jobResult): $($working.Name)"
            Send-JobEmail -Subject $subject -Body $emailBody

        } catch {
            Write-Log "Unhandled failure for $($file.Name): $($_.Exception.Message)"
            Route-ProcessedFile -File $working -Status "FAILED"
            
            # Fallback email if the pipeline completely explodes
            $emailBody = New-DynamicsImportErrorEmail -JobId ([Guid]::Empty) -FileName $working.Name
            Send-JobEmail -Subject "GP Import CRASHED: $($working.Name)" -Body $emailBody
        }
    }
    
    Write-Log "-- Import End --" -IncludeFooter
}
