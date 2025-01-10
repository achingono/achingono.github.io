---
author: "Alfero Chingono"
title: "Conditionally Deploying Resources in Azure App Configuration Using Deployment Scripts"
date: 2025-01-10T15:43:12Z
draft: false
description: "This post explores how to use Azure deployment scripts and Bicep templates to dynamically query and conditionally deploy resources in Azure App Configuration, ensuring efficient and error-free configuration management."
slug: conditionally-deploying-resources-azure-app-configuration-using-deployment-scripts
tags: [
"Azure",
"Bicep",
"Deployment Scripts",
"Azure App Configuration",
"Infrastructure as Code"
]
categories: [
"Azure",
"DevOps",
"Infrastructure as Code"
]
image: "cover.jpg"
---

When managing configurations in [Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview), dynamically identifying and conditionally deploying resources can streamline your infrastructure and reduce manual intervention. In this post, I’ll walk you through how I used [Azure deployment scripts](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template) and Bicep templates to achieve this. Along the way, I’ll explain the logic behind the deployment and the steps required to ensure everything works seamlessly.

## Problem Statement

[Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview) allows us to manage configurations centrally, but as environments and configurations evolve, deploying defaults and ensuring environments are properly configured becomes a challenge. Manual processes are prone to errors and inefficiencies, and hardcoding resources isn’t scalable. We needed a solution that dynamically queried existing configurations and conditionally deployed only what was missing.

## Solution Overview

Using [Azure deployment scripts](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template) within Bicep, we can:

1. Query existing labels and keys in Azure App Configuration.
1. Conditionally deploy default settings, environments, and configurations based on the query results.
1. Automate the entire process while maintaining modularity and reusability.

### High-Level Architecture

- **Deployment Script Modules:** Query existing labels and keys in Azure App Configuration.
- **Conditional Logic in Bicep:** Deploy resources based on the query results.
- **Managed Identity with Role Assignments:** Securely enable deployment scripts to access Azure resources.

## Step 1: Set Up a Managed Identity with Role Assignments

Deployment scripts require permissions to interact with Azure App Configuration. To enable this, we assign a User-Assigned Managed Identity with the [App Configuration Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/integration#app-configuration-contributor) role.

```bicep
param name string
param location string
param role string = 'fe86443c-f201-4fc4-9d2a-ac61149fbda0'

resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2018-11-30' = {
  name: 'id-${name}'
  location: location
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2020-08-01-preview' = {
  name: guid(resourceGroup().id, identity.id, resourceId('Microsoft.Authorization/roleDefinitions', role))
  properties: {
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', role)
    principalType: 'ServicePrincipal'
    principalId: identity.properties.principalId
  }
}

output id string = identity.id
output principalId string = identity.properties.principalId
```

This Bicep template creates a User-Assigned Managed Identity and assigns the [App Configuration Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/integration#app-configuration-contributor) role, granting the identity the necessary permissions to query configuration labels and keys.

## Step 2: Query Existing Labels and Keys

### Querying Labels

The `configurationLabel.bicep` module dynamically retrieves all labels in Azure App Configuration using a deployment script:

```bicep
param storeName string
param location string = resourceGroup().location
param identityId string
param utcValue string = utcNow()

resource script 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: '${storeName}-label-query'
  location: location
  kind: 'AzureCLI'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identityId}': {}
    }
  }
  properties: {
    forceUpdateTag: utcValue
    azCliVersion: '2.61.0'
    timeout: 'PT10M'
    arguments: storeName
    scriptContent: '''
    #!/bin/bash
    set -e
    echo "Start Listing the Labels";
    result=$(az appconfig kv list --name $1 --query "[].label" | jq -r .[] | sort -u | jq -R . | jq -s "{result: .}");
    echo $result > $AZ_SCRIPTS_OUTPUT_PATH;
    echo $result;
    echo "End Listing the Labels"
    '''
    cleanupPreference: 'OnSuccess'
    retentionInterval: 'P1D'
  }
}

output result string[] = script.properties.outputs.result
```

### Querying Keys

Similarly, the `configurationKey.bicep` module fetches all unique keys:

```bicep
param storeName string
param location string = resourceGroup().location
param identityId string
param utcValue string = utcNow()

resource script 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: '${storeName}-key-query'
  location: location
  kind: 'AzureCLI'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identityId}': {}
    }
  }
  properties: {
    forceUpdateTag: utcValue
    azCliVersion: '2.61.0'
    timeout: 'PT10M'
    arguments: storeName
    scriptContent: '''
    #!/bin/bash
    set -e
    echo "Start Listing the Keys";
    result=$(az appconfig kv list --name $1 --query "[].key" | jq -r .[] | cut -d ":" -f 1 | sort -u | jq -R . | jq -s "{result: .}");
    echo $result > $AZ_SCRIPTS_OUTPUT_PATH;
    echo $result;
    echo "End Listing the Keys"
    '''
    cleanupPreference: 'OnSuccess'
    retentionInterval: 'P1D'
  }
}

output result string[] = script.properties.outputs.result
```

The command `az appconfig kv list --name $1 --query "[].key" | jq -r .[] | cut -d ":" -f 1 | sort -u | jq -R . | jq -s "{result: .}"` performs several operations to list and process keys from an Azure App Configuration store:

1. `az appconfig kv list --name $1 --query "[].key"`: This Azure CLI command lists all key-value pairs from the specified Azure App Configuration store (using the name provided as the first argument `$1`) and extracts only the keys.

2. `| jq -r .[]`: The output from the previous command is piped to `jq`, a lightweight and flexible command-line JSON processor. The `-r` flag outputs raw strings instead of JSON texts, and `.[]` iterates over each element in the array, outputting each key on a new line.

3. `| cut -d ":" -f 1`: The `cut` command splits each key at the colon (`:`) delimiter and extracts the first field. This is useful if the keys have a namespace or prefix separated by a colon.

4. `| sort -u`: The `sort` command sorts the keys and the `-u` flag ensures that only unique keys are kept.

5. `| jq -R .`: The sorted unique keys are then piped back to `jq` with the `-R` flag, which reads each line of input as a raw string and converts it to a JSON string.

6. `| jq -s "{result: .}"`: Finally, the `jq -s` command slurps all the input lines into a single JSON array and wraps it in an object with a `result` property.

The final output is a JSON object containing an array of unique keys, which is then stored in the `result` variable.

## Step 3: Conditionally Deploy Resources

The `configuration/environments.bicep` module leverages the queried labels and keys to conditionally deploy resources. For example:

- **Defaults:** Ensures a default environment is created if the label does not exist.
- **Environments:** Deploys specific environments for missing labels.
- **Settings:** Adds configurations if the keys are not present.

```bicep
module defaults 'defaults.bicep' = if (!contains(labels, 'default')) {
  name: '${name}-defaults'
  params: {
    name: name
    environmentName: 'default'
    storeName: store.name
    validationKey: validationKey
    decryptionKey: decryptionKey
    googleAPIKey: googleAPIKey
    mail: mail
  }
}

module environments 'environment.bicep' = [
  for environmentName in filter(
    environmentNames,
    environmentName => !contains(labels, environmentName)
  ): {
    name: '${name}-environments-${environmentName}'
    params: {
      name: name
      environmentName: environmentName
      pools: pools
      sqlPassword: sqlPassword
      storageAccountName: storageAccountName
      storeName: store.name
      domain: domain
      configurations: configurations
    }
  }
]
```

## Conclusion

This approach automates the process of identifying missing configurations and deploying only the necessary resources. By combining deployment scripts with Bicep’s conditional logic, we’ve achieved a dynamic and scalable solution for managing Azure App Configuration.

Feel free to try this out in your environment and share your feedback. Let me know if you encounter any challenges or have ideas for further optimization!

References:

- [Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview)
- [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Deployment Scripts](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/deployment-script-template)
- [Managed Identities for Azure Resources](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- [Role-Based Access Control (RBAC) in Azure](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/)
- [jq Manual](https://stedolan.github.io/jq/manual/)
- [Use deployment scripts in Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-script-bicep?tabs=CLI)