---
layout: post
---

# Gotta Go Index

## The Problem
Sometimes with powershell you'll hit a powershell script that runs very slowly. Most of the time this isn't an issue, but occasionally performance matters.

Some of the slowest scripts are the ones where you need to access network resources one by one as you interate through a list. A classic is querying AAD users one at a time to answer a question about all of your users.

## A Solution?
There is no single solution for slow scripts. One solution for accessing network resources one by one is to go an get everthing you need all at once. For instance get all of the AAD users. To get the information about a particualar user you can loop through the collection with a query:
```powershell
$UserCache | Where-Object {$_.ObjectId -eq $query}
```

This is also painfully slow. For every query you need to loop through every object. This is like searching your whole house any time you need a fork. The faster thing is to organise everything in a indexed data structure.

A hash table has the nice property that you can access any object in the table by using the index:
```powershell
$UserLookupTable[$AP.PrincipalId]
```

This allows us to find a particular object extremely quickly. The only limitations are that the property that you are searching for needs to be unique. It will not work if multiple objects share the same property value (its like a primary key in a database). To built one, we can loop through a collection of objects and add them to an empty hash table:
```powershell
$UserLookupTable = @{}

foreach($User in $UserCache)
{
    $UserLookupTable[([string]$User.ObjectId)] = $User
}
```

# An Example Script

Here is a script that leverages this quick hash table lookup to find the name of a user (rather than the GUID that the Get-AzureADServicePrincipal cmdlet returns). The script is a quick look at what the users in your enviroment have consented to (to check for App Consent attacks or just things that shouldn't be there).

```powershell
function Get-AADAppPermissions {

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

## Further Readering... I Mean Watching