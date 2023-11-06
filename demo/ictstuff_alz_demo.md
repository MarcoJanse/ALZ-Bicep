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
    - [Management Groups Diagnostic Settings](#management-groups-diagnostic-settings)
    - [Hub networking](#hub-networking)

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
New-AzManagementGroupDeployment @parPolDefDeploy -Verbose
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
New-AzManagementGroupDeployment @parRoleDefsDeploy -Verbose
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
New-AzResourceGroupDeployment @parLogDeploy -Verbose
```

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
 New-AzManagementGroupDeployment @parMgDiagDeploy -Verbose
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

