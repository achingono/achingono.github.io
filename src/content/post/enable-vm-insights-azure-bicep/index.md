---
author: "Alfero Chingono"
title: "How to Enable VM Insights on an Azure Virtual Machine Using Bicep"
date: 2024-06-27T13:31:25Z
draft: false
description: "A step-by-step guide to enabling VM Insights on Azure Virtual Machines using a Bicep module to automate diagnostics and telemetry collection."
slug: enable-vm-insights-azure-bicep
tags: [
  Azure,
  Bicep,
  Application Insights,
  Monitoring,
  Diagnostics,
  Virtual Machines
]
categories: [
  Azure DevOps,
  Cloud Infrastructure
]
image: "cover.jpg"
---
To monitor your virtual machines in Azure, Application Insights gives you a solid way to collect telemetry data. In this post, I'll show you how to enable VM Insights on an Azure Virtual Machine with a Bicep module I recently worked on.

## The Goal

We aim to:

1. Enable diagnostics on an Azure VM.
2. Send logs and performance metrics to Application Insights.
3. Use a Bicep module to automate the setup.

## The Solution: A Bicep Module

Below is the Bicep module I crafted to enable VM Insights:

```bicep
param name string
param location string
param storageAccountName string
param instanceName string

resource appInsights 'Microsoft.Insights/components@2020-02-02-preview' existing = {
  name: 'appi-${name}'
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-06-01' existing = {
  name: storageAccountName
}

resource virtualMachine 'Microsoft.Compute/virtualMachines@2023-03-01' existing = {
  name: 'vm-${instanceName}-${name}'
}

resource extension 'Microsoft.Compute/virtualMachines/extensions@2018-10-01' = {
  location: location
  parent: virtualMachine
  name: 'IaaSDiagnostics'
  properties: {
    publisher: 'Microsoft.Azure.Diagnostics'
    type: 'IaaSDiagnostics'
    typeHandlerVersion: '1.5'
    autoUpgradeMinorVersion: true
    settings: {
      WadCfg: {
        DiagnosticMonitorConfiguration: {
          overallQuotaInMB: '4096'
          sinks: 'applicationInsights'
          Directories: {
            scheduledTransferPeriod: 'PT1M'
            sinks: 'applicationInsights'
            IISLogs: {
              containerName: 'iislogs'
            }
            FailedRequestLogs: {
              containerName: 'failedrequestlogs'
            }
          }
          PerformanceCounters: {
            scheduledTransferPeriod: 'PT1M'
            sinks: 'AzureMonitor'
            PerformanceCounterConfiguration: []
          }
          WindowsEventLog: {
            scheduledTransferPeriod: 'PT5M'
            sinks: 'applicationInsights'
            DataSource: [
              {
                name: 'Application!*[System[(Level <=3)]]'
              }
              {
                name: 'System!*[System[(Level <=3)]]'
              }
              {
                name: 'System!*[System[Provider[@Name=\'Microsoft Antimalware\']]]'
              }
              {
                name: 'Security!*[System[(Level <= 3)]'
              }
            ]
          }
          Logs: {
            sinks: 'applicationInsights'
            DataSource: [
              {
                name: 'Application!*'
              }
              {
                name: 'System!*'
              }
              {
                name: 'Security!*'
              }
            ]
          }
          Crashes: ''
          Syslog: ''
        }
        SinksConfig: {
          Sink: [
            {
              name: 'AzureMonitor'
              AzureMonitor: {}
            }
            {
              name: 'applicationInsights'
              ApplicationInsights: appInsights.properties.InstrumentationKey
            }
          ]
        }
      }
      StorageAccount: storageAccount.name
      StorageType: 'TableAndBlob'
    }
    protectedSettings: {
      storageAccountName: storageAccount.name
      storageAccountKey: storageAccount.listKeys().keys[0].value
    }
  }
}
```

## Key Components

1. **Application Insights**

  The appInsights resource is defined as an existing resource. Make sure to create an Application Insights instance before using this module.

1. **Storage Account**

  Used for storing diagnostic logs and data. The storage account is also referenced as an existing resource.

1. **Virtual Machine**

  The target VM must exist before deploying this module. The Bicep code references the VM as an existing resource.

1. **VM Extension**

  The IaaSDiagnostics extension is the critical piece that connects the VM to Application Insights. It configures various settings, including performance counters, logs, and event logs.

## How It Works

- **Diagnostics Configuration (WadCfg)**
  Defines settings for logs, performance counters, and event logs that are collected from the VM.

- **Sinks Configuration**
  Routes the collected telemetry data to both Azure Monitor and Application Insights.

- **Protected Settings**
  Includes sensitive data such as the storage account key to securely connect to the storage account.

## Conclusion

This Bicep module simplifies enabling VM Insights on Azure Virtual Machines, so you get detailed monitoring and diagnostics without manual setup. With telemetry routed to Application Insights, you can watch your VM’s health and performance in real time.

Do you have a different approach to enabling VM Insights? Let me know in the comments!

References:

- [Azure Monitor Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Application Insights Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Azure Virtual Machines Extensions](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/overview)
- [Bicep Language Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Diagnostics Extension](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/diagnostics-extension-overview)
- [Manage Virtual Machines with Azure CLI](https://learn.microsoft.com/en-us/cli/azure/vm)
- [How to Configure VM Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/vminsights-enable)
- [Install and configure the Azure Diagnostics extension for Windows (WAD)](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/diagnostics-extension-windows-install)
- [Troubleshooting Azure Diagnostics extension - Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/agents/diagnostics-extension-troubleshooting)
- [Windows diagnostics extension schema](https://docs.azure.cn/en-us/azure-monitor//agents/diagnostics-extension-schema-windows)