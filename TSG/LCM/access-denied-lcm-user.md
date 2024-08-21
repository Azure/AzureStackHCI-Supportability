# Facing "Access Denied" For the LCM User

### Description 
During deployment, update, or node addition, you may encounter an "Access Denied" error for the LCM user. This issue can arise when the LCM user credentials in Active Directory become out of sync with those stored in the ECE (Enterprise Cloud Engine) store. Users may update the LCM user password directly in Active Directory instead of using the [Set-AzureStackLCMUserPassword](https://learn.microsoft.com/en-us/azure-stack/hci/manage/manage-secrets-rotation#run-set-azurestacklcmuserpassword-cmdlet) cmdlet. This cmdlet handles updating the ECE store, so bypassing it can cause a synchronization issue between Active Directory and the ECE store.

### Identifying the Issue
If you encounter an "Access Denied" error during deployment, update, or node addition, verify that the credentials in the ECE store match the LCM user credentials in Active Directory. You can check this by running the PowerShell script provided in the [Verify the ECE Store Contains the LCM Credentials](#Verify-the-ECE-Store-Contains-the-LCM-Credentials) section below.

### Mitigating the Issue
To mitigate the issue, the LCM user credentials need to be updated in the ECE store to match the credentials in Active Directory. To do this, run the powershell script provided in the [Update the LCM Credentials in the ECE Store](#Update-the-LCM-Credentials-in-the-ECE-Store) section below.

### Verify the ECE Store Contains the LCM Credentials
Run the following script in PowerShell. Input your LCM user credentials when prompted.

```
# Prompt for credentials
$credential = Get-Credential

# Import necessary modules
Import-Module "ECEClient" 3>$null 4>$null
Import-Module "C:\Program Files\WindowsPowerShell\Modules\Microsoft.AS.Infra.Security.SecretRotation\Microsoft.AS.Infra.Security.ActionPlanExecution.psm1" -DisableNameChecking

# Convert the SecureString password to an encrypted standard string
$encryptedPassword = $credential.GetNetworkCredential().Password | Protect-CmsMessage -To "CN=DscEncryptionCert"

# Validate credentials in ECE
$ValidateParams = @{
    TimeoutInSecs = 10 * 60
    RetryCount = "2"
    ExclusiveLock = $true
    RolePath = "SecretRotation"
    ActionType = "ValidateCredentials"
    ActionPlanInstanceId = [Guid]::NewGuid()
}
$ValidateParams['RuntimeParameters'] = @{
    UserName = $credential.GetNetworkCredential().UserName
    Password = $encryptedPassword
}

Write-AzsSecurityVerbose -Message "Validating credentials in ECE.`r`nStarting action plan with Instance ID: $($ValidateParams.ActionPlanInstanceId)" -Verbose
$ValidateActionPlanInstance = Start-ActionPlan @ValidateParams 3>$null 4>$null

if ($ValidateActionPlanInstance -eq $null)
{
    Write-AzsSecurityWarning -Message "There was an issue running the action plan. Please reach out to Microsoft support for help" -Verbose
}
elseif ($ValidateActionPlanInstance.Status -eq 'Failed')
{
    Write-AzsSecurityWarning -Message "Could not find matching credentials in ECE store." -Verbose
}
elseif ($ValidateActionPlanInstance.Status -eq 'Completed')
{
    Write-AzsSecurityVerbose -Message "Found matching credentials in ECE store." -Verbose
}
```

### Update the LCM Credentials in the ECE Store
To update the LCM user credentials in the ECE store. Input your LCM user credentials when prompted.
```
# Prompt for credentials
$credential = Get-Credential

# Import the necessary module
Import-Module "C:\Program Files\WindowsPowerShell\Modules\Microsoft.AS.Infra.Security.SecretRotation\PasswordUtilities.psm1" -DisableNameChecking

# Check the status of the ECE cluster group
$eceClusterGroup = Get-ClusterGroup | Where-Object { $_.Name -eq "Azure Stack HCI Orchestrator Service Cluster Group" }
if ($eceClusterGroup.State -ne "Online") {
    Write-AzsSecurityError -Message "ECE cluster group is not in an Online state. Cannot continue with password rotation." -ErrRecord $_
}

# Update ECE with the new password
Write-AzsSecurityVerbose -Message "Updating password in ECE" -Verbose

$ECEContainersToUpdate = @(
    "DomainAdmin",
    "DeploymentDomainAdmin",
    "SecondaryDomainAdmin",
    "TemporaryDomainAdmin",
    "BareMetalAdmin",
    "FabricAdmin",
    "SecondaryFabric",
    "CloudAdmin"
)

foreach ($containerName in $ECEContainersToUpdate) {
    Set-ECEServiceSecret -ContainerName $containerName -Credential $credential 3>$null 4>$null
}

Write-AzsSecurityVerbose -Message "Finished updating credentials in ECE." -Verbose
```