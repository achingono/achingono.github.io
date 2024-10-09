---
author: "Alfero Chingono"
title: "Adding Azure Functions Support to a Devcontainer"
date: 2022-09-23T15:58:13Z
draft: false
description: "A container image for developing Azure Functions with Azure Functions Core Tools exists. What if one wants to add Azure Functions Core Tools to an existing dev container?"
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

# Adding Azure Functions Support to a Devcontainer

Are you looking to develop Azure Functions with ease? Do you already have a dev container set up and want to add Azure Functions Core Tools to it? Look no further! In this guide, we'll walk you through the steps to seamlessly integrate Azure Functions support into your existing dev container.

## The Challenge

Recently, I faced the task of adding support for [Azure Functions](https://azure.microsoft.com/products/functions/) to my trusted [Dev Container](https://learn.microsoft.com/training/modules/use-docker-container-dev-env-vs-code/). I tried a few approaches, but encountered errors along the way. After extensive research and experimentation, I finally found a solution that worked flawlessly.

## The Solution

To begin, I attempted to add the following lines to my `Dockerfile`:

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

Next, I added the following extensions to the `devcontainer.json` file:

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