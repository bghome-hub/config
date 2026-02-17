# 1. CONNECT WITH A STATIC NAMESPACE
# We verify the namespace is "GP" so we can rely on it later
$GPService = New-WebServiceProxy -Uri $GPWS_URL -Namespace "GP" -UseDefaultCredential

# 2. CREATE THE KEY OBJECT
# Now we can use [GP.ClassName] instead of the long auto-generated name
$WfStepKey = New-Object GP.WorkflowStepInstanceKey

# Assign the ID we found from SQL
$WfStepKey.Id = $WorkflowStepID

# Create and assign the Company Key
$WfStepKey.CompanyKey = New-Object GP.CompanyKey
$WfStepKey.CompanyKey.Id = -1 

# 3. DEFINE THE ACTION (APPROVE)
# We access the Enum directly from our static "GP" namespace
$CompletionStatus = [GP.WorkflowAction]::Approve

# 4. EXECUTE
try {
    $Result = $GPService.CompleteWorkflowTask($WfStepKey, $CompletionStatus, $ApprovalComment)
    
    if ($Result -eq $true) {
        Write-Host "Success! Batch Approved." -ForegroundColor Green
    } else {
        Write-Host "Command sent, but GP returned false. Check Workflow History." -ForegroundColor Yellow
    }
}
catch {
    Write-Error "Web Service Call Failed: $_"
}
