# XTAPStressTest

This sample script will allow admins to test the limits of the Cross Tenant Access Settings policy object by adding dummy partner tenants to the policy. It also includes a script to backup the current configuration, clean up the dummy partners that are created during the test, and restore the original configuration.

## Connect to Microsoft Graph
Open PowerShell and run the following. Insert the Tenant ID GUID of the tenant you want to run the test in. If needed, authenticate with Global Admin or Security Admin credentials and consent to the permission.

```
connect-graph -TenantId <insert tenant id> -Scopes Policy.ReadWrite.CrossTenantAccess
```

## Backup existing XTAP configuration
Run the following and define a file path of where to save the txt file. For example, "C:\Users\jeffbley\Desktop\XTAPBackup.txt".

```
$path = "TODO" #Enter the file path where you want the txt file exported
$XTAP = Invoke-MgGraphRequest -Method GET -Uri https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners?top=999
$XTAPBackup = $XTAP.Value | ConvertTo-Json -Depth 10
$XTAPBackup | out-file $path 
```

## Stress test
Run the following and define the number of dummy partner tenants you want to add. Starting with zero partner tenants, this should fail around 650 partner tenants. The policy is intentionally large to take up more space and require fewer partners before hitting the limit.

```
$NumberTenantIds = 700 #Enter the number of Tenants the want to add to your Cross Tenant Access Policy
$i="" #initilizing variable/resetting counter if running more than once
for ($i=1; $i -le $NumberTenantIds; $i++) 
{
$newGUID = [guid]::NewGuid().ToString()
$i
$params = @{
	TenantId = "$newGUID"
    inboundTrust = {
        isMfaAccepted = true
        isCompliantDeviceAccepted = true
        isHybridAzureADJoinedDeviceAccepted = true
    }
    b2bCollaborationOutbound = {
        usersAndGroups = {
            accessType = "allowed"
            targets = [
                {
                    target = "750c9b7f-1bf7-478e-b72c-a2af8a2bf0f8"
                    targetType = "group"
                },
                {
                    target = "2f6f0bcc-51da-4271-81c1-97d1dafbe22c",
                    targetType: "group"
                }
            ]
        },
        applications = {
            accessType = "allowed",
            targets = [
                {
                    target = "3ce7f118-a012-4cc0-9232-f13ba92caa29",
                    targetType = "application"
                },
                {
                    target = "1da12f7c-9c59-4ab9-80d0-e6aefc6be0c6",
                    targetType = "application"
                }
            ]
        }
    }
    b2bCollaborationInbound = {
        usersAndGroups = {
            accessType = "allowed",
            targets = [
                {
                    target = "48a88dae-056a-45d6-9362-acfd2e245ca4",
                    targetType = "group"
                },
                {
                    target = "df216ca3-f561-46d8-b068-187a8943c4ee",
                    targetType = "group"
                }
            ]
        },
        applications = {
            accessType = "allowed",
            targets = [
                {
                    target = "a566b413-46c5-43ee-bf9b-9eb728cb35d4",
                    targetType = "application"
                },
                {
                    target = "ca5e3e37-a7ae-4b4a-af7c-e2ba2c3f3626",
                    targetType = "application"
                },
                {
                    target = "26612b82-68f2-4bca-97e7-03f3b8c42cc7",
                    targetType = "application"
                },
                {
                    target = "fe890a58-e83f-4a0a-9e71-d0e95083f515",
                    targetType = "application"
                },
                {
                    target = "c9a559d2-7aab-4f13-a6ed-e7e9c52aec87",
                    targetType = "application"
                },
                {
                    target = "Office365",
                    targetType = "application"
                }
            ]
        }
    }
    b2bDirectConnectOutbound = {
        usersAndGroups = {
            accessType = "allowed",
            targets = [
                {
                    target = "dabccd43-50c6-4a73-b975-6e1484ddd760",
                    targetType = "group"
                },
                {
                    target = "2f6f0bcc-51da-4271-81c1-97d1dafbe22c",
                    targetType = "group"
                },
                {
                    target = "750c9b7f-1bf7-478e-b72c-a2af8a2bf0f8",
                    targetType = "group"
                }
            ]
        },
        applications = {
            accessType = "allowed",
            targets = [
                {
                    target = "1da12f7c-9c59-4ab9-80d0-e6aefc6be0c6",
                    targetType = "application"
                },
                {
                    target = "Office365",
                    targetType = "application"
                }
            ]
        }
    }
    b2bDirectConnectInbound = {
        usersAndGroups = {
            accessType = "allowed",
            targets = [
                {
                    target = "AllUsers",
                    targetType = "user"
                }
            ]
        },
        applications = {
            accessType = "allowed",
            targets = [
                {
                    target = "Office365",
                    targetType = "application"
                }
            ]
        }
    }
}

New-MgPolicyCrossTenantAccessPolicyPartner -BodyParameter $params -ErrorAction Stop
}
```

## Count number of partners
If you want to know how many partner tenants are currently configured, run the following:

```
$XTAPCount = Invoke-MgGraphRequest -Method GET -Uri https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners?top=999
$XTAPCount.value | measure-object | select count
```

## Clean up dummy partner tenants
Run the following to delete all the partner tenants. Specify the path to the back txt file created earlier. 
Note: this script can run slowly due to MS Graph API throttling limits. You may need to run this script more than once if it times out. To know if all the partner tenants have been deleted, use the above cmdlets to count the current number.

```
#Delete all XTAP partners
$CurrentXTAP = Invoke-MgGraphRequest -Method GET -Uri https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners?top=999
$tenantid = $CurrentXTAP.value.tenantid
Foreach($id in $tenantid){
$id
Invoke-MgGraphRequest -Method DELETE -Uri https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners/$id
}
```

## Restore XTAP configuration
Run the following to restore the XTAP configuration to backed-up state. Specify the path to the backup txt file created earlier. 

```
$path = "TODO" #Enter the file path of your backup txt file
$XTAPRestore = Get-Content -Path $path | convertfrom-json
foreach($tenant in $XTAPRestore){
$body = $tenant | ConvertTo-Json -Depth 5
$XTAP = Invoke-MgGraphRequest -Method POST -Uri https://graph.microsoft.com/beta/policies/crossTenantAccessPolicy/partners -Body $body
}
```
