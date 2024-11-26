---
---

10.2411.0.24 update failure in the VersionControl UpdatePowerShellModules interface.

# Symptoms
When installing the 2411.0 Solution update, you might experience a failure with a message similar to the following:
```
Type 'UpdatePowerShellModules' of Role 'VersionControl' raised an exception:

Exception occured while uninstalling higher version of module [Az.Accounts] = Exception occured while uninstalling module Az.Accounts. Please make sure all PowerShell sessions that might be using the module are closed.
```
Screenshot of the failure message from Azure portal:
![psmoduleupdatefailure.png](/TSG/Update/PSModule-Uninstall-Failure.png)

# Issue Validation
The issue corresponds to the message above, which will include the name of the first module for which uninstallation failed.

# Cause
      
The update process attempts to downgrade installed PowerShell modules that are part of the Azure Local validated recipe but do not conform to the validated recipe for 2411.0.
A downgrade will be attempted for the following PowerShell modules, if necessary:
- Az.StackHCI
- Az.Attestation
- Az.Storage
- Az.ConnectedMachine
- Az.Resources
- Az.Accounts

If there is a PowerShell process on one of the cluster nodes that has imported one of these PowerShell modules, the update process will be unable to uninstall the module. Modules can be locked by an interactive PowerShell console (powershell.exe) or a remote PowerShell session to the node (wsmprovhost.exe).

# Mitigation Details

## Step 1: Determine nodes on which uninstallation failed
Use the following PowerShell script to find the latest update attempt and determine the machines where modules could not be downgraded. Open a new PowerShell session to run the below commands:

```Powershell
$latestUpdate = Get-ActionPlanInstances | where { $_.RuntimeParameters.updateId -match "Solution10.2411.0.24" } | sort EndDateTime | select -Last 1
$xml = [xml]($latestUpdate.ProgressAsXml)
$exceptions = $xml.SelectNodes("//Exception") | where Message -match "Please make sure all PowerShell"
$exceptions.MachineName
```

## Step 2: Examine and terminate processes holding the locks
On each machine identified in step 1, examine the PowerShell processes that may have the module imported blocking the uninstallation attempt. Run the script below to find all PowerShell processes on this node, excluding the current process and well-known processes belonging to the ALM agent.

```powershell
$processes = Get-Process -IncludeUserName | ? Name -in wsmprovhost, powershell | Where-Object { $_.UserName -notmatch "NT SERVICE\\AzureStack Agent Lifecycle Agent" } | where Id -ne $PID
$processes
```

To terminate the processes either remove them one at a time or use the following script to remove all the processes identified in the previous script.
```powershell
foreach ($p in $processes) {
    Write-Host "Stopping process with id : $($p.Id)"
    Stop-Process $p.Id -Force
    Write-Host "Stopped process with id : $($p.Id)"
}
```

Once the PowerShell processes have been cleaned up, resume the update.