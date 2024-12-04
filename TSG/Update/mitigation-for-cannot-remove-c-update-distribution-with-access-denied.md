# Overview
If update attempts fail in **'CauPreRequisites'** step with an error message that looks something like the below, then we need to manually clean up the directory **_C:\UpdateDisctribution_** directory on all the cluster nodes before proceeding.
```
Type 'CauPreRequisites' of Role 'CAU' raised an exception: Could not finish cau prerequisites due to error 'Cannot remove item C:\UpdateDistribution\<any_file_name>: Access to the path is denied.'
```

# How to mitigate and clean up state before retrying
1. Resolve any leftover state from previous runs

   `Invoke-CauRun -ForceRecovery -Force`

2. Then you will need to clean up the following directory on **all** the cluster member nodes.

   `C:\UpdateDistribution\`

3. If removing any item in the directory fails with the FILE_IN_USE error then you need to terminate the process using the file then remove the item. Example, if the file foobar.dll is in use, the following command will list the processes using it:
   
   `tasklist /m foobar.dll`

4. Then, retrieve the process ID and use it in the following command to stop the process:

   `Stop-Process <pid> -Force`

5. Retry above steps for any other file which are in use. Clean up the directory on all the cluster nodes:

   `Remove-Item C:\UpdateDistribution -Force -Recurse`

Once the directories are cleaned up on all nodes, it should be safe to retry the update. 
