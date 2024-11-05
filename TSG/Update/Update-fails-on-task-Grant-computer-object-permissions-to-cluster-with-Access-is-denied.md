# Symptoms

Update failure during task:
**Grant computer object permissions to cluster**

**Type 'AddComputerObjectPermissionToClusterForUpdate' of Role 'Domain' raised an exception: Access is denied at <ScriptBlock>, <No file>: line 21**

This occurs during updates from 2408 to >=2408.1

# Cause

This is caused by a change that was added to support brownfield upgrade, where an interface to ECE was added to grant permissions to CAU. In this case, this happened due to the customer's configuration, where the LCM user did not have the permissions to grant other permissions.

# Mitigation Details

To mitigate this, we need to manually grant the required permissions to CAU or CAU role is already configured on the cluster, and then skip the failing step in the update action plan.

To skip the failing steps and resume the update, save the following script to a file, e.g. `sample.ps1`

```Powershell
$tempPath = "C:\temp"
if (!(Test-Path -Path $path))
{
    New-Item -ItemType Directory -Path $path
}

function Invoke-ActionPlanInstanceWithNewXml
{
    param(
        [Parameter(Mandatory = $true)]
        [string]
        $ActionPlanPath,

        [Parameter(Mandatory = $true)]
        [Guid]
        $ReferenceActionPlanInstanceID
    )

    $ErrorActionPreference = 'Stop'

    $eceServiceClient = Create-ECEClusterServiceClient
    $inst = Get-ActionPlanInstance -eceClient $eceServiceClient -actionPlanInstanceId $ReferenceActionPlanInstanceID
    if ($inst -eq $null)
    {
        throw "Reference action plan instance not found: $ReferenceActionPlanInstanceID"
    }

    $lock = $inst.LockType -eq [Microsoft.AzureStack.Solution.Deploy.EnterpriseCloudEngine.Controllers.Models.ActionPlanInstanceExecutionLock]::ExclusiveLock
    Invoke-ActionPlanInstance -eceClient $eceServiceClient -ActionPlanPath $ActionPlanPath -Retries $inst.Retries -RuntimeParameters $inst.RuntimeParameters -ExclusiveLock:$lock | Out-Null
}

# This function MUST be invoked manually and should NOT be used in any automated scripts.
function Skip-FailedStepsInActionPlan
{
    [CmdletBinding(SupportsShouldProcess,
                   ConfirmImpact = 'High')]
    param(
        [Parameter(Mandatory = $true)]
        [Guid]
        $ActionPlanInstanceID
    )

    $ErrorActionPreference = 'Stop'

    $eceServiceClient = Create-ECEClusterServiceClient
    $inst = Get-ActionPlanInstance -eceClient $eceServiceClient -actionPlanInstanceId $ActionPlanInstanceID
    if ($inst.Status -ne "Failed")
    {
        Write-Warning "Instance is not in Failed state. Cannot skip failed steps."
        return
    }

    [xml]$progressXml = $inst.ProgressAsXml
    $failedInterfaceTasks = $progressXml.SelectNodes("//Task[@InterfaceType and @Status='Error']")
    if ($failedInterfaceTasks.Count -eq 0)
    {
        Write-Warning "Did not find InterfaceTask in 'Error' state in action XML."
        return
    }

    Write-Host "Failed Interface(s):" -ForegroundColor Yellow
    $failedInterfaceTasks | Select RolePath, InterfaceType | Format-Table
    if ($PSCmdlet.ShouldProcess($ActionPlanInstanceID))
    {
        $failedRemoteActions = $progressXml.SelectNodes("//*[RemoteConfig and @Status='Error']")

        Write-Verbose "Marking failed interface Tasks as Skipped." -Verbose
        $failedInterfaceTasks | foreach { $_.Status = 'Skipped' }

        # Delete relevant remote Action plan instance because ECE service will use remote XML.
        foreach ($remoteAction in $failedRemoteActions)
        {
            $remoteNode = $remoteAction.RemoteNodeName
            $remoteActionId = $remoteAction.RemoteTaskId
            $eceAgentClient = Create-ECEAgentClient -NodeName $remoteNode
            if ($eceAgentClient.GetActionPlanInstance($remoteActionId).GetAwaiter().GetResult())
            {
                Write-Verbose "Deleting associated failed remote action plan instance $remoteActionId from $remoteNode." -Verbose
                $deleteActionPlanInstanceDescription = New-Object -TypeName 'Microsoft.AzureStack.Solution.Deploy.EnterpriseCloudEngine.Controllers.Models.DeleteActionPlanInstanceDescription'
                $deleteActionPlanInstanceDescription.ActionPlanInstanceID = $remoteActionId
                $eceAgentClient.DeleteActionPlanInstance($deleteActionPlanInstanceDescription).Wait()
            }
        }

        # Save to file and invoke new action plan
        $tempFile = Join-Path "C:\Temp" ([IO.Path]::GetRandomFileName())
        Write-Verbose "Saving modified XML to temp file $tempFile." -Verbose
        $progressXml.Save($tempFile)
        Invoke-ActionPlanInstanceWithNewXml -ActionPlanPath $tempFile -ReferenceActionPlanInstanceID $ActionPlanInstanceID | Out-Null
    }
}
```

Then run:
1. `Import-Module sample.ps1`
2. ```Skip-FailedStepsInActionPlan -ActionPlanInstanceID <REPLACE FAILED UPDATE ACTIONPLAN INSTANCE ID>```

This will automatically resume update.