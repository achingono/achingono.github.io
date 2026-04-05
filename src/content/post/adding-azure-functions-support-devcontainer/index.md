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

I already had a dev container set up and needed to add Azure Functions Core Tools to it. This is the approach that finally worked for me.

## The Challenge

I needed to add support for [Azure Functions](https://azure.microsoft.com/products/functions/) to my existing [Dev Container](https://learn.microsoft.com/training/modules/use-docker-container-dev-env-vs-code/). I tried a few approaches and hit errors along the way. After some digging through the docs and a bit of trial and error, I landed on a version that rebuilt cleanly.

## The Solution

To begin, I attempted to add the following lines to my `Dockerfile`:

That version failed when I rebuilt the container. After searching around and comparing a few examples, I landed on this `RUN` command:

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

With that in place, I had a working dev container with Azure Functions Core Tools and the Visual Studio Code extensions I needed.

References:  
[Use a Docker container as a development environment with Visual Studio Code](https://learn.microsoft.com/en-us/training/modules/use-docker-container-dev-env-vs-code/)  
[Work with Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local?tabs=v4%2Clinux%2Ccsharp%2Cportal%2Cbash)  
[Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-core-tools-reference?tabs=v2)
