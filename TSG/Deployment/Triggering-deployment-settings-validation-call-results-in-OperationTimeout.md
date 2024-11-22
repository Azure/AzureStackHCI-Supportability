
# Symptoms
  
When deploying a new 23h2 cluster based on the newly available 2411 image, there may be occurrences where you will experience a timeout while initiating the validation process "**Triggering deployment settings validation call....**"



# Issue Validation
To confirm the scenario that you are encountering is the issue documented in this article, please confirm that you see a similar error as showed below:

"**Could not complete the operation. 200: OperationTimeout , No updates received from device for oeration (...) beyond timeout of [600000] ms**.


<img width="392" alt="Picture1" src="https://github.com/user-attachments/assets/82dc34cf-034b-44bc-a982-2186244da046">


# Cause
The issue has been identified and the permanent mitigation will be offered in a future LCM extension version. **The current workaround is to make sure that all the nodes are set to time zone "UTC"**.

# Mitigation Details

```
1.  Log into each of the Azure Local nodes and change the time zone to UTC using the command `Set-TimeZone -Id "UTC"`, then reboot the nodes.

2.  Once they are up, make sure LCM service is running using the command `Get-Service LcmController` as shown in the example below.  
    
PS C:\> Get-Service LcmController

Status   Name               DisplayName
------   ----               -----------
Running  LcmController      LcmController

3. Please retry the validation process.
```
