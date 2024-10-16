## Symptoms
Solution update get hung at downloading status with action plan instance showing running for months
![Update.png](/TSG/Update/Update_Stuck_02.png)

![ECE.png](/TSG/Update/Update_Stuck_01.png)

## Issue Validation
We can validate the issue using below command and we can see that the actionplan has been running for over 4 months 

```Powershell
get-actionplaninstances | sort startdatetime | ? actionplanname -eq "GetCauDeviceInfo" | ft instanceid, status, startdatetime, enddatetime
```
![Items.png](/TSG/Update/Update_Stuck_03.png)

## Cause
Update service does the following for OS update: 
1. get device info 
2. use device info to scan for new os update 
3. download os update pkg to local disk. step #1 and #2 are invoked via cau run and action plan.

Before creating action plan of type 'GetCauDeviceInfo', update service is looking for any instance that's running and use that instance id if found. 

When instance is running, we assume it's not failed. Therefore, update service is stuck since that instance was not active in ECE.

## Resolution:
We removed the instance using the script and the update succeeded.
```Powershell
Import-Module ECEClient -DisableNameChecking
$eceClient = Create-ECEClusterServiceClient

# Cancel old running instance
$actionPlanInstanceID = "<ActionPlan Instance ID>"
Cancel-ActionPlanInstance -eceClient $eceClient -actionPlanInstanceID $actionPlanInstanceID

# remove old instance
$deleteActionPlanInstanceDescription = New-Object -TypeName 'GetCauDeviceInfo'
$deleteActionPlanInstanceDescription.ActionPlanInstanceID = $actionPlanInstanceID
$eceClient.DeleteActionPlanInstance($deleteActionPlanInstanceDescription).Wait()
```