---
author: "Alfero Chingono"
title: "Adding Azure Functions Support to a Devcontainer"
date: 2022-09-23T15:58:13Z
draft: false
description: ""
slug: adding-azure-functions-support-devcontainer
tags: [
"azure",
"functions",
"codespaces",
"devcontainer"
]
categories: [
"How To",
"Setup",
"Development"
]
image: "cover.png"
---

While working on the [Reddog Microservices Integration Application Sample](https://github.com/Azure-Samples/app-templates-microservices-integration), I wanted to add support for [Azure Functions](https://azure.microsoft.com/products/functions/) to an existing [Dev Container](https://learn.microsoft.com/training/modules/use-docker-container-dev-env-vs-code/).

The first thing I tried was to add the following to my `Dockerfile`:

```Dockerfile
RUN apt-get update \
    && apt-get -y install azure-functions-core-tools-4
```

Unfortunately, that resulted in errors when rebuilding the container. So after searching online, and reviewing documentation, I eventually landed on this RUN command:

```Dockerfile
RUN curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg \
    && sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg \
    && apt-get update \
    && sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/debian/$(lsb_release -rs | cut -d'.' -f 1)/prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list' \
    && sudo apt-get update \
    && apt-get -y install azure-functions-core-tools-4
```

This successfully rebuilt the container image and had the [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools) installed.

Next, I added the following extension to the `devcontainer.json` file:

```json
	"extensions": [
		"ms-azuretools.vscode-azurefunctions",
		"ms-vscode.azure-account",
		"ms-azuretools.vscode-azureresourcegroups",
		"azurite.azurite"
	]
```

With that, I had a working environment fully loaded with Azure Functions Core Tools and the necessary Visual Studio Code extensions installed. Hope that helps, dear future reader!

References:  
[Use a Docker container as a development environment with Visual Studio Code](https://learn.microsoft.com/en-us/training/modules/use-docker-container-dev-env-vs-code/)  
[Work with Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local?tabs=v4%2Clinux%2Ccsharp%2Cportal%2Cbash)  
[Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-core-tools-reference?tabs=v2)