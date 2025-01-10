---
author: "Alfero Chingono"
title: "Dockerfile for containerizing Classic ASP"
date: 2023-06-16T12:14:12Z
draft: false
description: ""
slug: dockerfile-for-containerizing-classic-asp
tags: [
"docker",
"classic-asp",
"containerization",
"asp-net",
"windows-server",
"iis",
"web-development"
]
categories: [
"containerization",
"web-development",
"docker",
"asp-net"
]
image: "cover.jpg"
---

In this blog post, we will walk through a Dockerfile designed for containerizing a Classic ASP application. This Dockerfile is based on the `mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2019` image and includes several steps to configure the environment for running Classic ASP applications.

### Breakdown of the Dockerfile

#### Base Image and Shell Configuration
```dockerfile
ARG LOGMONITOR_VERSION=2.0
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2019
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
ARG LOGMONITOR_VERSION
```
We start by specifying the base image, which is a Windows Server Core image with ASP.NET 4.8. The `SHELL` instruction sets PowerShell as the default shell for subsequent instructions.

#### Installing Windows Features
```dockerfile
RUN Install-WindowsFeature Web-ASP; `
    Install-WindowsFeature Web-CGI; `
    Install-WindowsFeature Web-ISAPI-Ext; `
    Install-WindowsFeature Web-ISAPI-Filter; `
    Install-WindowsFeature Web-Includes; `
    Install-WindowsFeature Web-HTTP-Errors; `
    Install-WindowsFeature Web-Common-HTTP; `
    Install-WindowsFeature Web-Performance; `
    Install-WindowsFeature WAS; `
    Import-module IISAdministration;
```
This section installs various IIS features required for running Classic ASP applications, such as ASP, CGI, ISAPI extensions, and more.

#### Enabling IIS Features
```dockerfile
RUN Enable-WindowsOptionalFeature -Online -FeatureName IIS-DefaultDocument; `
    Enable-WindowsOptionalFeature -Online -FeatureName IIS-HttpErrors;
```
Here, we enable additional IIS features like default documents and HTTP errors.

#### Installing Dependencies
```dockerfile
RUN Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://vcredist.com/install.ps1'));
RUN md c:/msi;
```
We install the C++ 2017 redistributable and create a directory for MSI files.

#### Installing IIS Rewrite Module and SQL ODBC Driver
```dockerfile
RUN Invoke-WebRequest 'https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi' -OutFile c:/msi/rewrite_amd64_en-US.msi; `
    Start-Process 'c:/msi/rewrite_amd64_en-US.msi' '/qn' -PassThru | Wait-Process;

RUN Invoke-WebRequest 'https://download.microsoft.com/download/c/5/4/c54c2bf1-87d0-4f6f-b837-b78d34d4d28a/en-US/18.2.1.1/x64/msodbcsql.msi' -OutFile c:/msi/msodbcsql.msi; `
    Start-Process 'c:/msi/msodbcsql.msi' 'IACCEPTMSODBCSQLLICENSETERMS=YES' -PassThru | Wait-Process;
```
These commands download and install the IIS Rewrite Module and the SQL ODBC driver.

#### Downloading Log Monitor
```dockerfile
RUN Invoke-WebRequest -Uri "https://github.com/microsoft/windows-container-tools/releases/download/v$($env:LOGMONITOR_VERSION)/LogMonitor.exe" -OutFile C:\LogMonitor.exe;
RUN Invoke-WebRequest -Uri "https://github.com/microsoft/windows-container-tools/releases/download/v$($env:LOGMONITOR_VERSION)/LogMonitor.exe" -OutFile C:\LogMonitor.exe;
```
This command downloads the Log Monitor tool from the specified GitHub release URL and save it to the container's file system. Log Monitor is a tool used to monitor and manage logs within a Windows container. This step ensures that the Log Monitor tool is available within the container for log management purposes. 


#### Unlocking IIS Configuration Sections
```dockerfile
RUN & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/asp; `
    & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/handlers; `
    & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/modules;
```
This section unlocks various IIS configuration sections to allow modifications.

In IIS (Internet Information Services), certain configuration sections are locked by default to prevent unauthorized changes that could affect the stability and security of the server. Unlocking these sections is necessary when you need to customize the behavior of your web server beyond the default settings. 

- **system.webServer/asp**: Unlocking this section allows you to modify settings related to ASP (Active Server Pages), such as enabling or disabling ASP, configuring script timeouts, and setting debugging options.
- **system.webServer/handlers**: This section controls the handlers that process requests for specific file types or URL patterns. Unlocking it allows you to add, remove, or modify handlers to customize how requests are processed.
- **system.webServer/modules**: Modules are components that handle various stages of request processing in IIS. Unlocking this section allows you to enable, disable, or configure modules to extend the functionality of your web server.

By unlocking these sections, you gain the flexibility to tailor the IIS configuration to meet the specific needs of your applications.

#### Configuring Default Application Pool and Web Site
```dockerfile
RUN Import-Module WebAdministration; `
    $pool = Get-Item IIS:\\AppPools\\DefaultAppPool; `
    $pool.ManagedPipelineMode       = 'Integrated'; `
    $pool.ManagedRuntimeVersion     = 'v4.0'; `
    $pool.Enable32BitAppOnWin64     = $false; `
    $pool.AutoStart                 = $true; `
    $pool | Set-Item;

RUN Import-Module WebAdministration; `
    Set-WebConfigurationProperty -Location 'Default Web Site' -Filter "system.webServer/asp" -Name "enableParentPaths" -Value 'True';`
    Set-WebConfigurationProperty -Location 'Default Web Site' -Filter "system.webServer/asp/limits" -Name "maxRequestEntityAllowed" -Value 2000000000;
```
We configure the default application pool and set properties for the default web site.

#### Enabling Fusion Logs
```dockerfile
RUN Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name ForceLog         -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogFailures      -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogResourceBinds -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogPath          -Value 'C:\FusionLog\' -Type String; `
    mkdir C:/FusionLog -Force;
```
This section enables Fusion logs for troubleshooting .NET assembly binding issues.

#### Setting Up the Working Directory and Copying Files
```dockerfile
WORKDIR /inetpub/wwwroot
COPY src/ .

COPY cfg/ c:/
```
We set the working directory to the IIS web root and copy the application source and configuration files.

#### Setting the Entry Point
```dockerfile
ENTRYPOINT ["C:\\LogMonitor.exe", "C:\\ServiceMonitor.exe", "w3svc"]
```
Finally, we set the entry point to start the Log Monitor and Service Monitor for IIS.

This Dockerfile provides a comprehensive setup for running Classic ASP applications in a containerized environment, ensuring all necessary features and dependencies are installed and configured.

```dockerfile
# escape=`

ARG LOGMONITOR_VERSION=2.0
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2019
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]
ARG LOGMONITOR_VERSION

# Install Windows Features
RUN Install-WindowsFeature Web-ASP; `
    Install-WindowsFeature Web-CGI; `
    Install-WindowsFeature Web-ISAPI-Ext; `
    Install-WindowsFeature Web-ISAPI-Filter; `
    Install-WindowsFeature Web-Includes; `
    Install-WindowsFeature Web-HTTP-Errors; `
    Install-WindowsFeature Web-Common-HTTP; `
    Install-WindowsFeature Web-Performance; `
    Install-WindowsFeature WAS; `
    Import-module IISAdministration;

# Enable IIS Features
RUN Enable-WindowsOptionalFeature -Online -FeatureName IIS-DefaultDocument; `
    Enable-WindowsOptionalFeature -Online -FeatureName IIS-HttpErrors;

# Install C++ 2017 distributions
RUN Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://vcredist.com/install.ps1'));

RUN md c:/msi;

# Install IIS Rewrite Module
RUN Invoke-WebRequest 'https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi' -OutFile c:/msi/rewrite_amd64_en-US.msi; `
    Start-Process 'c:/msi/rewrite_amd64_en-US.msi' '/qn' -PassThru | Wait-Process;

# Install SQL ODBC Driver
RUN Invoke-WebRequest 'https://download.microsoft.com/download/c/5/4/c54c2bf1-87d0-4f6f-b837-b78d34d4d28a/en-US/18.2.1.1/x64/msodbcsql.msi' -OutFile c:/msi/msodbcsql.msi; `
    Start-Process 'c:/msi/msodbcsql.msi' 'IACCEPTMSODBCSQLLICENSETERMS=YES' -PassThru | Wait-Process;

# Download Log Monitor
RUN Invoke-WebRequest -Uri "https://github.com/microsoft/windows-container-tools/releases/download/v$($env:LOGMONITOR_VERSION)/LogMonitor.exe" -OutFile C:\LogMonitor.exe;

# Unlock IIS Configuration Sections
RUN & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/asp; `
    & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/handlers; `
    & c:\windows\system32\inetsrv\appcmd.exe `
      unlock config `
      /section:system.webServer/modules;

# Configure Default Application Pool
RUN Import-Module WebAdministration; `
    $pool = Get-Item IIS:\\AppPools\\DefaultAppPool; `
    $pool.ManagedPipelineMode       = 'Integrated'; `
    $pool.ManagedRuntimeVersion     = 'v4.0'; `
    $pool.Enable32BitAppOnWin64     = $false; `
    $pool.AutoStart                 = $true; `
    $pool | Set-Item;

# Configure Default Web Site
RUN Import-Module WebAdministration; `
    Set-WebConfigurationProperty -Location 'Default Web Site' -Filter "system.webServer/asp" -Name "enableParentPaths" -Value 'True';`
    Set-WebConfigurationProperty -Location 'Default Web Site' -Filter "system.webServer/asp/limits" -Name "maxRequestEntityAllowed" -Value 2000000000;

# Enable Fusion Logs
# https://stackoverflow.com/a/33013110
RUN Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name ForceLog         -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogFailures      -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogResourceBinds -Value 1               -Type DWord; `
    Set-ItemProperty -Path HKLM:\\Software\\Microsoft\\Fusion -Name LogPath          -Value 'C:\FusionLog\' -Type String; `
    mkdir C:/FusionLog -Force;

WORKDIR /inetpub/wwwroot
COPY src/ .

COPY cfg/ c:/

ENTRYPOINT ["C:\\LogMonitor.exe", "C:\\ServiceMonitor.exe", "w3svc"]
```