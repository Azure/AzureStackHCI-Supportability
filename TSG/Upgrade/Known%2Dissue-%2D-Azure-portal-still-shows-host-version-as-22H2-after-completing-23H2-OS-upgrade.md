# Symptoms
The Azure portal may still reflect the OS on the node to be Azure Stack HCI 22H2 even though the 23H2 upgrade has been completed.

# Issue Validation
In the Azure portal, navigate to either the cluster overview page or the node-specific page within Machine - Azure Arc.  The operating system version may still be reflected at 10.20349.X.X (22H2).

# Cause
This issue seems to occur due to the timing of the cluster uploading census data to Azure.

# Mitigation Details
This issue seems to clear on its own after a few hours, but you can run `Sync-AzureStackHCI` from any of the cluster nodes to force the cluster to update its census data.  This should update the OS version to the correct 10.0.25398.X.X (23H2) version.