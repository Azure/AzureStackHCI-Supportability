# How-to-Recreate-Customerconfig.xml-For-Environment-Validator
# Description 
## Do not use this TSG if a deployment action has been started

This article describes how to bring back the customer configuration if it was removed and not recreated during the environment validator action flow.
# Symptoms

## Expected Occurrence  
This issue is expected to happen when certain conditions are met. When a validation call is triggered and fails, and then a second validation call is triggered. You will likely see this issue. This is due to the seconds validation call trying to clean up the state of the first validation call but something is locking a file that needs to be cleaned up in the ECE store. So the script fails to clean up and throws an error before the ECEstore can be recreated.
## Signature 
On a re-run of the environment validator action the action is unable to be launched and an error like ```Item 'CustomerConfiguration.xml' not found in store 'Default'. The file path is C:\EceStore\efb61d70-47ed-8f44-5d63-bed6adc0fb0f\0b8ae09e-3e34-bd06-78a3-28c0d0d425ca ``` should appear in the azure portal or arm template response.
# Mitigation 
These steps will only mitigate a single run of validation until a RCA is discovered and a fix is provided. These steps may have to be run after every environment validator action.

1) Run the below code block to re-init LCM.
``` 
$ErrorActionPreference = "stop"
import-module C:\CloudDeployment\ECEngine\EnterpriseCloudEngine.psd1  -ErrorAction stop
$act = Get-ActionProgress -ActionType "CloudDeployment" -ErrorAction stop
if ($null -ne $act)
{
    throw "Deployment action started do not proceed with this tsg"
}
else
{
    Write-Host "Applying TSG to delete reg key to re-initialize LCM Controller"
    Remove-ItemProperty -Path "HKLM:\Software\Microsoft\LCMAzureStackStampInformation" -Name InitializationComplete -ErrorAction stop -Force
}


```
2) Restart the LCM service ``` Restart-Service LCMController ```
4) Wait until LCM initialization completes. This can be check by looking in the latest ```C:\MasLogs\LCMECELitelogs\InitializeDeploymentService-date.log.``` The last line in the file should say ``` Action: Action plan 'InitializeDeploymentService' completed. ``` This action takes about 15-20 minutes to complete
5) Re-run environment validation from portal
