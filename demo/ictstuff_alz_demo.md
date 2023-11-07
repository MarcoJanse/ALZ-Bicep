# ICTStuff Azure LandingZone Demo

## Index

- [ICTStuff Azure LandingZone Demo](#ictstuff-azure-landingzone-demo)
  - [Index](#index)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Steps](#steps)
    - [Policies](#policies)
    - [Custom Role Definitions](#custom-role-definitions)
    - [Logging, Automation and Sentinel](#logging-automation-and-sentinel)
    - [Hub networking](#hub-networking)
    - [Role Assignments for Management Groups and Subscriptions](#role-assignments-for-management-groups-and-subscriptions)
    - [Subscription Placement](#subscription-placement)
    - [Built-in and Custom Policy assignments](#built-in-and-custom-policy-assignments)
      - [alzDefaultPolicyAssignments.ictstuff.bicep - Changes](#alzdefaultpolicyassignmentsictstuffbicep---changes)
      - [alzDefaultPolicyAssignments.parameters.ictstuff.json](#alzdefaultpolicyassignmentsparametersictstuffjson)

## Introduction

This page can be used for demoing the Azure Landing Zone deployment in ICTStuff Azure tenant

## Prerequisites

- [ ] Forked the Azure/ALZ-Bicep repo to my own repo as [MarcoJanse/ALZ-Bicep](https://github.com/MarcoJanse/ALZ-Bicep)
- [ ] Cloned the git repo locally
- [ ] Created a branch [ictstuff-landingzone](https://github.com/MarcoJanse/ALZ-Bicep/tree/ictstuff-landingzone).
- [ ] In Azure, elevated my account in Azure AD.
- [ ] Added myself as an owner on the root management group using Azure PowerShell

```powershell
Connect-AZAccount

$user = Get-AzADUser -UserPrincipalName (Get-AzContext).Account
New-AzRoleAssignment -Scope '/' -RoleDefinitionName 'Owner' -ObjectId $user.Id
```

- After that step, removed the elevation in Azure AD
- Signed in again

## Steps

- Navigate to locally cloned git repo of (forked) ALZ-Bicep: `ALZ-Bicep`.
- Open VScode here: `code .`.
- Switch to the `ictstuff-landingzone`-branch.
- Copy the `ALZ-Bicep\infra-as-code\bicep\modules\managementGroups\parameters\managementGroups.parameters.all.json` and rename it.
  - This way, you can keep syncing with the upstream branch and not run into any merge conflicts when the original files get updated.
  - In my case I named it `ALZ-Bicep\infra-as-code\bicep\modules\managementGroups\parameters\managementGroups.parameters.ictstuff.json`.
- Update the parameters below with your own, for example:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "parTopLevelManagementGroupPrefix": {
      "value": "alz"
    },
    "parTopLevelManagementGroupDisplayName": {
      "value": "Azure Landing Zones"
    },
    "parTopLevelManagementGroupParentId": {
      "value": "-mg"
    },
    "parLandingZoneMgAlzDefaultsEnable": {
      "value": true
    },
    "parLandingZoneMgConfidentialEnable": {
      "value": false
    },
    "parLandingZoneMgChildren": {
      "value": {}
    },
    "parTelemetryOptOut": {
      "value": false
    }
  }
}
```

- Log in using `Connect-AzAccount`
- Make sure a subscription is selected that's part of the correct Azure tenant.
  - You can use `Get-AzContext` to view that
    - If not, use the following to set the right subscription:

```powershell
Set-AzContext -Subscription (Get-AzSubscription | Where-Object Name -eq 'VSES–MPN-3fifty-marco-01-platform').Id
```

- Create the below variable that will make a hashtable of al the cmdlet parameters and values:

```powershell
$parMgDeploy = @{
  DeploymentName          = 'ictstuff-MgDeploy-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location                = 'westeurope'
  TemplateFile            = 'infra-as-code\bicep\modules\managementGroups\managementGroups.bicep'
  TemplateParameterFile  = 'infra-as-code\bicep\modules\managementGroups\parameters\managementGroups.parameters.ictstuff.json'
}
```

- Deploy the bicep file using the hash of the parameters to create the management group structure.

```powershell
New-AzTenantDeployment @parMgDeploy -WhatIf
```

> WhatIf is recommended to run first, and you could replace `-WhatIf` with `-Verbose` to follow the deployment along.

### Policies

- Copy the parameter file `\infra-as-code\bicep\modules\policy\definitions\parameters\customPolicyDefinitions.parameters.all.json` and rename it.
  - In my case, I named it `\infra-as-code\bicep\modules\policy\definitions\parameters\customPolicyDefinitions.parameters.ictstuff.json`
  - Change `parTargetManagementGroupId` value to to match your `parTopLevelManagementGroupPrefix` from the [Management group deployment](#management-groups).
    - Don't forget the suffix if you have added one.
- Create the below variable that will make a hash table of al the cmdlet parameters and values:

```powershell
$parPolDefDeploy = @{
  DeploymentName        = 'ictstuff-PolicyDefsDeploy-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location              = 'westeurope'
  ManagementGroupId     = 'alz-mg'
  TemplateFile          = "infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep"
  TemplateParameterFile = 'infra-as-code/bicep/modules/policy/definitions/parameters/customPolicyDefinitions.parameters.ictstuff.json'
}
```

- After that, run the `New-AzManagementGroupDeployment` using the created variable for all parameters

```powershell
New-AzManagementGroupDeployment @parPolDefDeploy -WhatIf
```

### Custom Role Definitions

> Some custom roles that are best practices that can be deployed, like
>
> - Application Owner
> - NetOps
> - SecOps
> - Subscription Owner

- Copy the parameter file `\infra-as-code\bicep\modules\customRoleDefinitions\parameters\customRoleDefinitions.parameters.all.json` and rename it.
  - In my case, I named it `\infra-as-code\bicep\modules\customRoleDefinitions\parameters\customRoleDefinitions.parameters.ictstuff.json`
  - Change `parTargetManagementGroupId` value to to match your `parTopLevelManagementGroupPrefix` from the [Management group deployment](#management-groups).
    - Don't forget the suffix if you have added one.
- Create the below variable that will make a hash table of al the cmdlet parameters and values:

```powershell
$parRoleDefsDeploy = @{
  DeploymentName        = 'ictstuff-RoleDefsDeployment-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location              = 'westeurope'
  ManagementGroupId     = 'alz-mg'
  TemplateFile          = "infra-as-code/bicep/modules/customRoleDefinitions/customRoleDefinitions.bicep"
  TemplateParameterFile = 'infra-as-code/bicep/modules/customRoleDefinitions/parameters/customRoleDefinitions.parameters.ictstuff.json'
}
```

- After that, run the `New-AzManagementGroupDeployment` using the created variable for all parameters

```powershell
New-AzManagementGroupDeployment @parRoleDefsDeploy -WhatIf
```

### Logging, Automation and Sentinel

> Please make sure you know which subscription you want to deploy your resources under.

- Copy the parameter file `\infra-as-code\bicep\modules\logging\parameters\logging.parameters.all.json` and rename it.
  - In my case, I named it `\infra-as-code\bicep\modules\logging\parameters\logging.parameters.ictstuff.json`
  - [Optional]: Change `parLogAnalyticsWorkspaceName` value if you want.
  - Change `parLogAnalyticsWorkspaceLocation` value to `westeurope`.
  - Change `parAutomationAccountLocation` value to `westeurope`.
  - [Optional]: Change `parAutomationAccountName` value if you want
  - Under `parTags`, change the `Environment` to your desired value. (I used Shared)
- Use the below scripts to define required parameters and create the necessary resource group:

```powershell
# Set the top level MG Prefix in accordance to your environment.
$TopLevelMGPrefix = "alz"

# Parameters necessary for deployment
$parLogDeploy = @{
  DeploymentName        = 'ictstuff-LogDeploy-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  ResourceGroupName     = "rg-logging-shd-001"
  TemplateFile          = "infra-as-code/bicep/modules/logging/logging.bicep"
  TemplateParameterFile = "infra-as-code/bicep/modules/logging/parameters/logging.parameters.ictstuff.json"
}
```

- Once you have the variables defined, use the blocks below to create the required resource group and after that, deploy the logging Bicep module:

```powershell
# Create Resource Group - optional when using an existing resource group
New-AzResourceGroup `
  -Name $parLogDeploy.ResourceGroupName `
  -Location westeurope
```

- Deploy the Logging bicep module using the command below:

```powershell
New-AzResourceGroupDeployment @parLogDeploy -WhatIf

### Management Groups Diagnostic Settings

- Copy the parameter file  `/infra-as-code\bicep\orchestration\mgDiagSettingsAll\parameters\mgDiagSettingsAll.parameters.all.json` and rename it.
  - In my case, I named it `/infra-as-code\bicep\orchestration\mgDiagSettingsAll\parameters\mgDiagSettingsAll.parameters.ictstuff.json`
  - [optional] Change `parTopLevelManagementGroupPrefix` to your desired value
  - [optional] Change `parTopLevelManagementGroupSuffix` to your desired value.
    - I used `-mg` here
  - Change `parLogAnalyticsWorkspaceResourceId` to include your subscription ID, the resourcegroup that holds the log analytics workspace and the log analytics workspace name.
  **HINT:** get the subscription ID with this cmdlet:

```powershell
Get-AzSubscription | Where-Object { $_.Name -match '<unique part of subscription name>' } | Select-Object -ExpandProperty Id
```

- Next, create a hashtable with the parameters and deploy the bicep template.

```powershell
$parMgDiagDeploy = @{
  TemplateFile          = "infra-as-code/bicep/orchestration/mgDiagSettingsAll/mgDiagSettingsAll.bicep"
  TemplateParameterFile = "infra-as-code/bicep/orchestration/mgDiagSettingsAll/parameters/mgDiagSettingsAll.parameters.ictstuff.json"
  Location              = "westeurope"
  ManagementGroupId     = "alz-mg"
}
```

Deploy the Diagnostics settings bicep template using the command below:

```powershell
 New-AzManagementGroupDeployment @parMgDiagDeploy -WhatIf
 ```

### Hub networking

- Copy the parameters file `infra-as-code\bicep\modules\hubNetworking\parameters\hubNetworking.parameters.all.json` and rename it
  - I used `infra-as-code\bicep\modules\hubNetworking\parameters\hubNetworking.parameters.ictstuff.json` and edited the following values:
    - `parLocation`
    - `parHubNetworkName`
    - `parHubNetworkAddressPrefix`
      - I used `172.20.0.0/16` and updated the following `parSubnets`

```json
"parSubnets": {
      "value": [
        {
          "name": "AzureBastionSubnet",
          "ipAddressRange": "172.20.0.0/24"
        },
        {
          "name": "GatewaySubnet",
          "ipAddressRange": "172.20.254.0/24"
        },
        {
          "name": "AzureFirewallSubnet",
          "ipAddressRange": "10.20.255.0/24"
        }
      ]
    },
```

> Please note that you cannot rename these subnets. These names is mandatory
>
> I prefer to add additional subnets with NSG's using a separate bicep file, referencing the existing vNet.

- Continue with the rest of the `hubNetworking.parameters.ictstuff.json` file
  - After the `parSubnets, verify or change these values:
    - `parAzBastionName`
    - `parDdosPlanName`
    - `parAzFirewallName`
    - `parAzFirewallPoliciesName`
    - `parHubRouteTableName`
    - `parPrivateDnsZones`
      - `privatelink.westeurope.azmk8s.io`
      - `privatelink.westeurope.batch.azure.com`
      - `privatelink.westeurope.kusto.windows.net`
      - `privatelink.we.backup.windowsazure.com`
- Do not deploy an VPN gateway and ExpressRoute by removing all parameters inside the following arrays. (make them empty):
  - `parVpnGatewayConfig`
  - `parExpressRouteGatewayConfig`

After that, first select the right subscription. You might have a separate subscription for connectivity/networking, but I will use the Visual Studio subscription called `VSES–MPN-3fifty-marco-01-platform`

Next, create a hash table of all parameters to deploy the hub networking bicep template

```powershell
# Parameters necessary for deployment
$parHubNetworkDeploy = @{
  DeploymentName        = 'ictstuff-HubNetworkingDeploy-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  ResourceGroupName     = "rg-hubnetworking-shd-001"
  TemplateFile          = "infra-as-code/bicep/modules/hubNetworking/hubNetworking.bicep"
  TemplateParameterFile = "infra-as-code/bicep/modules/hubNetworking/parameters/hubNetworking.parameters.ictstuff.minimal.json"
}
```

- Now, create the Azure resource group to hold all resources in the region you prefer

```powershell
New-AzResourceGroup `
  -Name $parHubNetworkDeploy.ResourceGroupName `
  -Location 'westeurope'
```

- Finally, deploy the resources in the bicep file using the variables and the `inputobject` you created.

```powershell
New-AzResourceGroupDeployment @parHubNetworkDeploy -WhatIf
```

### Role Assignments for Management Groups and Subscriptions

> The example below is to assign a Readers role to a security group: **sg-ictstuff-readers**

- Copy the bicep file `infra-as-code\bicep\modules\roleAssignments\parameters\roleAssignmentManagementGroup.securityGroup.parameters.all.json` and rename it.
  - In mÿ case, I renamed it to `infra-as-code\bicep\modules\roleAssignments\parameters\roleAssignmentManagementGroup.securityGroup.parameters.ictstuff.json`
    - Remove the `parRoleAssignmentNameGuid` code block. This is optional and will be constructed by the `roleAssignmentResourceGroup.bicep`-file.
    - `parRoleDefinitionId` with the GUID of the (built-in) role definition. See [Azure built-in roles | MS-Learn](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles). For example, the built-in Reader role has the following GUID: `acdd72a7-3385-48ef-bd42-f606fba81ae7`
    - `parAssigneeObjectId` with the objectID of your security group. In my example, I use `sg-ictstuff-readers`. (But I won't display the GUID here...)
- Making sure you are logged in to Azure with the Az PowerShell module, create the `inputobject` below:

```powershell
$parRoleAssignDeploy    = @{
  DeploymentName        = 'ictstuff-RoleAssignmentsDeploy-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location              = 'westeurope'
  ManagementGroupId     = 'alz-mg'
  TemplateFile          = "infra-as-code/bicep/modules/roleAssignments/roleAssignmentManagementGroup.bicep"
  TemplateParameterFile = 'infra-as-code/bicep/modules/roleAssignments/parameters/roleAssignmentManagementGroup.securityGroup.parameters.ictstuff.json'
}
```

Next, create the role assignment:

```powershell
New-AzManagementGroupDeployment @parRoleAssignDeploy -WhatIf
```

### Subscription Placement

- Copy the bicep file `infra-as-code\bicep\orchestration\subPlacementAll\parameters\subPlacementAll.parameters.all.json` and rename it.
  - In mÿ case, I renamed it to `infra-as-code\bicep\orchestration\subPlacementAll\parameters\subPlacementAll.parameters.ictstuff.json`
  - Review/change the following parameters
    - `parTopLevelManagementGroupPrefix`
      - I kept mine set to `alz`
    - `parTopLevelManagementGroupSuffix`
      - Changed this to `-mg`
    - Next, I added my subscription ID's under the following management groups:
      - `parIntRootMgSubs`
      - `parDecommissionedMgSubs`
- Making sure you are logged in to Azure with the Az PowerShell module, create the `inputobject` below:

```powershell
$parSubPlacementDeploy  = @{
  DeploymentName        = 'ictstuff-SubPlacementAll-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location              = 'westeurope'
  ManagementGroupId     = 'alz-mg'
  TemplateFile          = "infra-as-code/bicep/orchestration/subPlacementAll/subPlacementAll.bicep"
  TemplateParameterFile = 'infra-as-code/bicep/orchestration/subPlacementAll/parameters/subPlacementAll.parameters.ictstuff.json'
}
```

Next, deploy the subscription placement using below code

```powershell
New-AzManagementGroupDeployment @parSubPlacementDeploy -WhatIf
```

### Built-in and Custom Policy assignments

> **NOTE** Because I only have 2 subscriptions to deploy resources to, I do the following:
>
> 1. Put my first subscription under the platform-mg management group
> 2. Assign all platform-management, platform-identity, and platform-connectivity policies to the platform top-level management group
> 3. Place the other subscription under the landingzone-corp-mg management group.

- On your system, make sure you are in the root of the ALZ-Bicep git repo.
- Open Code in this folder: `code .`
- Copy the `infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.bicep`
  - In my case I renamed it to `infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.ictstuff.bicep`
  - Edit this file and change line 332-334 and change the last part of each line:
    - Change `platform-management$` to `platform$`
    - Change `platform-connectivity$` to `platform$`
    - Change `platform-identity$` to `platform$`
- Copy the bicep file `infra-as-code\bicep\modules\policy\assignments\alzDefaults\parameters\alzDefaultPolicyAssignments.parameters.all.json` and rename it.
  - In mÿ case, I renamed it to `infra-as-code\bicep\modules\policy\assignments\alzDefaults\parameters\alzDefaultPolicyAssignments.parameters.ictstuff.json`
    - Exclude the policies you do not want to assign using the `parExcludedPolicyAssignments`-array in the parameter file.

Below you find the 3 lines that changed:

#### alzDefaultPolicyAssignments.ictstuff.bicep - Changes

```powershell
  platformManagement: parPlatformMgAlzDefaultsEnable ? '${parTopLevelManagementGroupPrefix}-platform-management${parTopLevelManagementGroupSuffix}' : '${parTopLevelManagementGroupPrefix}-platform${parTopLevelManagementGroupSuffix}'
  platformConnectivity: parPlatformMgAlzDefaultsEnable ? '${parTopLevelManagementGroupPrefix}-platform-connectivity${parTopLevelManagementGroupSuffix}' : '${parTopLevelManagementGroupPrefix}-platform${parTopLevelManagementGroupSuffix}'
  platformIdentity: parPlatformMgAlzDefaultsEnable ? '${parTopLevelManagementGroupPrefix}-platform-identity${parTopLevelManagementGroupSuffix}' : '${parTopLevelManagementGroupPrefix}-platform${parTopLevelManagementGroupSuffix}'
```

#### alzDefaultPolicyAssignments.parameters.ictstuff.json

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "parTopLevelManagementGroupPrefix": {
      "value": "alz"
    },
    "parTopLevelManagementGroupSuffix": {
      "value": "-mg"
    },
    "parLogAnalyticsWorkSpaceAndAutomationAccountLocation": {
      "value": "westeurope"
    },
    "parLogAnalyticsWorkspaceResourceId": {
      "value": "/subscriptions/30ff91cf-356d-4e30-b506-7687c4599923/resourceGroups/rg-pltf-logging-shd-001/providers/Microsoft.OperationalInsights/workspaces/log-alz-shd-001"
    },
    "parLogAnalyticsWorkspaceLogRetentionInDays": {
      "value": "365"
    },
    "parAutomationAccountName": {
      "value": "aa-alz-shd-001"
    },
    "parMsDefenderForCloudEmailSecurityContact": {
      "value": "marco.janse@ictstuff.info"
    },
    "parDdosProtectionPlanId": {
      "value": ""
    },
    "parPrivateDnsResourceGroupId": {
      "value": "/subscriptions/30ff91cf-356d-4e30-b506-7687c4599923/resourceGroups/rg-hubnetworking-shared-001"
    },
    "parPrivateDnsZonesNamesToAuditInCorp": {
      "value": []
    },
    "parDisableAlzDefaultPolicies": {
      "value": false
    },
    "parVmBackupExclusionTagName" : {
      "value": "BackupPolicy"
    },
    "parVmBackupExclusionTagValue" : {
      "value": [
        "ExcludedFromBackup"
      ]
    },
    "parExcludedPolicyAssignments": {
      "value": [
        "Deny-Public-IP",
        "Enable-DDoS-VNET"
      ]
    },
    "parTelemetryOptOut": {
      "value": false
    }
  }
}
```

```powershell
$parPolicyAssignmentDeploy  = @{
  DeploymentName            = 'alz-alzPolicyAssignmentDefaultsDeployment-{0}' -f (-join (Get-Date -Format 'yyyyMMddTHHMMssffffZ')[0..63])
  Location                  = 'westeurope'
  ManagementGroupId         = 'alz-mg'
  TemplateFile              = "infra-as-code/bicep/modules/policy/assignments/alzDefaults/alzDefaultPolicyAssignments.ictstuff.bicep"
  TemplateParameterFile     = 'infra-as-code/bicep/modules/policy/assignments/alzDefaults/parameters/alzDefaultPolicyAssignments.parameters.ictstuff.json'
}
```

Verify the deployment

```powershell
New-AzManagementGroupDeployment @parPolicyAssignmentDeploy -WhatIf
```
