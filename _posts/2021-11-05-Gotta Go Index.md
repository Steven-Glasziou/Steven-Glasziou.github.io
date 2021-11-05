---
layout: post
---

# Gotta Go Index

## The Problem

## The Solution

```powershell
function Audit-AADAppPermissions {

    $TennantDetails = Get-AzureADTenantDetail

    $EnterpriseApp = Get-AzureADServicePrincipal -All:$true | Where-Object {$_.tags -eq "WindowsAzureActiveDirectoryIntegratedApp"}

    $UserCache = Get-AzureADUser -All:$true

    $UserLookupTable = @{}

    foreach($User in $UserCache)
    {
        $UserLookupTable[([string]$User.ObjectId)] = $User.DisplayName
    }

    foreach ($EA in $EnterpriseApp) {
        $AppPermission = Get-AzureADServicePrincipalOAuth2PermissionGrant -ObjectId $EA.ObjectId

        foreach ($AP in $AppPermission) {

            [PSCustomObject]@{
                AppName = $EA.DisplayName
                AppPublisher = $EA.PublisherName
                User = If($AP.PrincipalId){$UserLookupTable[$AP.PrincipalId]} Else {$null}
                ConsentType = $AP.ConsentType
                Scope = $AP.Scope
                Tennant = $TennantDetails.DisplayName
                ReplyUrls = $EA.ReplyUrls
            }
        }
    }
}
```
