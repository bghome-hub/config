# --- <Configuration> ---
$GPWS_URL        = "http://GPSERVER:48620/Dynamics/GPService.asmx"
# The ID you found from SQL earlier (e.g. '946306CA-...')
$WorkflowStepID  = "PASTE_YOUR_GUID_HERE" 
$ApprovalComment = "Approved via PowerShell"
# -----------------------

try {
    # 1. CONNECT WITH STATIC NAMESPACE
    $GPService = New-WebServiceProxy -Uri $GPWS_URL -Namespace "GP" -UseDefaultCredential

    # 2. CREATE THE *CORRECT* KEY OBJECT
    # The method 'CompleteTask' expects a 'WorkflowTaskKey', not a 'StepInstanceKey'
    $TaskKey = New-Object GP.WorkflowTaskKey
    $TaskKey.Id = $WorkflowStepID
    $TaskKey.CompanyKey = New-Object GP.CompanyKey
    $TaskKey.CompanyKey.Id = -1 

    # 3. DEFINE THE ACTION
    # We use the static namespace 'GP' to get the Enum
    $Action = [GP.WorkflowAction]::Approve

    # 4. EXECUTE 'CompleteTask'
    Write-Host "Attempting to approve using CompleteTask..."
    
    # Signature: CompleteTask(WorkflowTaskKey key, WorkflowAction action, string comment)
    $Result = $GPService.CompleteTask($TaskKey, $Action, $ApprovalComment)
    
    if ($Result) {
        Write-Host "Success! Task Approved." -ForegroundColor Green
    } else {
        Write-Host "Command sent, but returned false." -ForegroundColor Yellow
    }

} catch {
    Write-Error "Error: $_"
    
    # 5. SELF-DIAGNOSIS (If this fails again)
    Write-Host "`n--- DEBUG: AVAILABLE METHODS ---" -ForegroundColor Cyan
    $GPService | Get-Member -MemberType Method | Where-Object { $_.Name -like "*Task*" } | Select-Object Name
}
