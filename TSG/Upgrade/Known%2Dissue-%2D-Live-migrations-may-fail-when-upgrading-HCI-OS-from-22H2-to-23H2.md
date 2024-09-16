During the upgrade of Azure Stack HCI 22H2 to 23H2, customers may see that live migrations start failing.  If customers are upgrading via Cluster Aware Updating (CAU), this will result in a failed CAU run as live migrations will eventually start to fail. 

# Symptoms
Live migrations will not complete.  These may hang at a certain percentage, or they may stay queued.  CAU may fail due to the node failing to drain.

# Cause
This appears to be related to an issue with virtual machines that utilize dynamic memory.  

# Mitigation Details

To fix this issue, the following registry key needs to be added to each node and the node rebooted for the change to take effect.

> :ledger: **NOTE**
>
> This registry key should be added to the 22H2 nodes prior to performing the upgrade to avoid this issue.  A reboot is required for the change to take effect.

- Path: HKLM\System\CurrentControlSet\Services\Vid\Parameters.
- Subkey name: SkipSmallLocalAllocations
- Type: REG_DWORD
- Value : 0

The following PowerShell commands can be used to create the registry value.

> :exclamation: **IMPORTANT**
>
> The first command may fail with an error that the item already exists which may be expected.  The command is included to ensure the item does indeed exist so the value will be properly created.  Do not append the `-Force` parameter to the command as this would overwrite the existing key which could unintentionally remove existing subkeys or values.

```PowerShell
    New-Item -Path HKLM:\SYSTEM\CurrentControlSet\Services\Vid\Parameters
    New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\Vid\Parameters -Name SkipSmallLocalAllocations -Value 0 -PropertyType DWord

```