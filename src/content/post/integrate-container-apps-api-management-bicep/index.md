---
author: "Alfero Chingono"
title: "Integrate Container Apps With Api Management using Bicep"
date: 2022-09-28T15:58:11Z
draft: false
description: "With Azure API Management it is easy to expose Backend APIs hosted in Azure Container Apps and integrate them with other Azure services like Azure App Service or Azure Functions"
slug: integrate-container-apps-api-management-bicep
tags: [
"bicep",
"containers",
"api-management",
"integration",
"api"
]
categories: [
"Azure",
"Development"
]
image: "cover.png"
---

While working on the [Reddog Microservices Integration Application Sample](https://github.com/Azure-Samples/app-templates-microservices-integration), our objective was to demonstrate a development scenario that uses [Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview) to integrate [Azure API Management](https://azure.microsoft.com/products/api-management/) with [Azure Container Apps](https://azure.microsoft.com/services/container-apps/).

> **DISCLAIMER:** This project is not my original work. The repository leverages the [Reddog codebase](https://github.com/Azure/reddog-code) and the [Reddog Container Apps](https://github.com/Azure/reddog-containerapps) bicep modules generously contributed by the Cloud Native Global Black Belt Team. I do not own any rights to the project an do not take credit for any of the amazing design work done by the team.

- **[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)** is a domain-specific language (DSL) that uses declarative syntax to deploy Azure resources.

- **[Azure API Management](https://azure.microsoft.com/services/api-management)** is a managed service that allows you to manage services across hybrid and multi-cloud environments. API management acts as a facade to abstract the backend architecture, and it provides control and security for API observability and consumption for both internal and external users.

- **[Azure Container Apps](https://azure.microsoft.com/services/container-apps)** is a fully managed, serverless container service used to build and deploy modern apps at scale. In this solution, you're hosting microservices on Azure Container Apps and deploying them into a single Container App environment. This environment acts as a secure boundary around the system.  

## Creating the APIM resource

The first step was to create the main [APIM Service resource](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service?pivots=deployment-language-bicep):

```Bicep
@description('The name of the API Management resource to be created.')
param apimName string

@description('The email address of the publisher of the APIM resource.')
@minLength(1)
param publisherEmail string = 'apim@contoso.com'

@description('Company name of the publisher of the APIM resource.')
@minLength(1)
param publisherName string = 'Contoso'

@description('The pricing tier of the APIM resource.')
param skuName string = 'Developer'

@description('The instance size of the APIM resource.')
param capacity int = 1

@description('Location for Azure resources.')
param location string = resourceGroup().location

resource apimResource 'Microsoft.ApiManagement/service@2020-12-01' = {
  name: apimName
  location: location
  sku: {
    capacity: capacity
    name: skuName
  }
  properties: {
    virtualNetworkType: 'External'
    publisherEmail: publisherEmail
    publisherName: publisherName
  }
}
```

This will deploy a APIM instance. Of particular note here is `virtualNetworkType` declaration. This represents the type of VPN in which API Management service needs to be configured in:

- **None (Default Value)** means the API Management service is not part of any Virtual Network, 
- **External** means the API Management deployment is set up inside a Virtual Network having an Internet Facing Endpoint, and 
- **Internal** means that API Management deployment is setup inside a Virtual Network having an Intranet Facing Endpoint only.

> **NOTE:** Please bear in mind that APIM takes more than an hour to provision. 

## Applying a global policy

Next step, we created a global policy definition file `apimPolicies/global.xml`:

```xml
<policies>
    <inbound>
        <rate-limit-by-key  calls="1000"
              renewal-period="60"
              increment-condition="@(context.Response.StatusCode == 200)"
              counter-key="@(context.Request.IpAddress)"
              remaining-calls-variable-name="remainingCallsPerIP"/>
        <!-- Distributed tracing support by adding correlationId using COMB format-->
        <!-- NOTE: If COMB format is not needed, context.RequestId should be used as a value of correlation id. -->
        <!-- context.RequestId is unique for each request and is stored as part of gateway log records. -->
        <!-- https://learn.microsoft.com/en-us/azure/api-management/policies/add-correlation-id -->
        <set-header name="correlationid" exists-action="skip">
            <value>@{ 
                var guidBinary = new byte[16];
                Array.Copy(Guid.NewGuid().ToByteArray(), 0, guidBinary, 0, 10);
                long time = DateTime.Now.Ticks;
                byte[] bytes = new byte[6];
                unchecked
                {
                       bytes[5] = (byte)(time >> 40);
                       bytes[4] = (byte)(time >> 32);
                       bytes[3] = (byte)(time >> 24);
                       bytes[2] = (byte)(time >> 16);
                       bytes[1] = (byte)(time >> 8);
                       bytes[0] = (byte)(time);
                }
                Array.Copy(bytes, 0, guidBinary, 10, 6);
                return new Guid(guidBinary).ToString();
            }</value>
        </set-header>
    </inbound>
    <backend>
        <forward-request />
    </backend>
    <outbound>
        <redirect-content-urls />
    </outbound>
    <on-error />
</policies>
```

Then add the [policy resource](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/policies?pivots=deployment-language-bicep) to the Bicep module:

```Bicep
resource apimPolicy 'Microsoft.ApiManagement/service/policies@2021-12-01-preview' = {
  name: 'policy'
  parent: apimResource
  properties: {
    format: 'rawxml'
    value: loadTextContent('apimPolicies/global.xml')
  }
}
```

The [Policy Contract](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/policies?pivots=deployment-language-bicep#policycontractproperties) here is defined in xml format, and of particular note here is the [loadTextContent](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions-files#loadtextcontent) function which loads the content of a specified file as a string.

## Adding a Backend API

A [*backend*](https://learn.microsoft.com/en-us/azure/api-management/backends) (or *API backend*) in API Management is an HTTP service that implements your front-end API and its operations. Sometimes backend APIs are referred to simply as backends. API Management supports using other Azure resources as an API backend, such as Container Apps. A custom backend has several benefits, including:

* Abstracts information about the backend service, promoting reusability across APIs and improved governance.  
* Easily used by configuring a transformation policy on an existing API.
* Takes advantage of API Management functionality to maintain secrets in Azure Key Vault if [named values](api-management-howto-properties.md) are configured for header or query parameter authentication.

To create a [Microsoft.ApiManagement/service/backends](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/backends?pivots=deployment-language-bicep) resource, we added the following Bicep to my template.

```Bicep
resource orderService 'Microsoft.App/containerApps@2022-03-01' existing = {
  name: 'order-service'
}

resource orderBackendResource 'Microsoft.ApiManagement/service/backends@2021-12-01-preview' = {
  name: 'order-service-backend'
  parent: apimResource
  dependsOn: [
    orderService
  ]
  properties: {
    description: orderService.name
    url: 'https://${orderService.properties.configuration.ingress.fqdn}'
    protocol: 'http'
    resourceId: '${environment().resourceManager}${orderService.id}'
  }
}
```

Here we reference an existing Container App in our backend resource and use the ingress URL as the backend URL. The `resourceId` needs to be a URL for the deployment to work. Here we use the [`environment()`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-deployment#environment) function, returns properties for the current Azure environment, to get the management URL instead of hard-coding `"https://management.azure.com/"`.

## Adding a Fronted API

API Management serves as mediation layer over the backend APIs. Frontend API is an API that is exposed to API consumers from API Management. You can customize the shape and behavior of a frontend API in API Management without making changes to the backend API(s) that it represents. Sometimes frontend APIs are referred to simply as APIs.

To create a [Microsoft.ApiManagement/service/apis](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/apis?pivots=deployment-language-bicep) resource, we added the following Bicep to my template.

```Bicep
resource orderApiResource 'Microsoft.ApiManagement/service/apis@2021-12-01-preview' = {
  parent: apimResource
  name: 'order-service'
  properties: {
    displayName: 'OrderService'
    subscriptionRequired: false
    path: 'orders'
    protocols: [
      'https'
    ]
    isCurrent: true
  }
}
```

- `subscriptionRequired`: Specifies whether an API or Product subscription is required for accessing the API.
- `path`: Relative URL uniquely identifying this API and all of its resource paths within the API Management service instance. It is appended to the API endpoint base URL specified during the service instance creation to form a public URL for this API.

## Adding a Fronted API Policy

A policy is a reusable and composable component, implementing some commonly used API-related functionality. API Management offers over 50 built-in policies that take care of critical but undifferentiated horizontal concerns - for example, request transformation, routing, security, protection, caching. The policies can be applied at various scopes, which determine the affected APIs or operations and dynamically configured using policy expressions. For more information, see [Policies in Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies).

In order to make the Fronted API policy reusable, we saved it in a XML file `apimPolicies/api.xml`:

```xml
<policies>
    <inbound>
        <base />
        <set-backend-service id="apim-generated-policy" backend-id="{backendName}" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

This policy does one thing: Use the `set-backend-service` policy to redirect an incoming request to the specified backend for all operations in this Frontend API. For more information,  see [Set backend service](https://learn.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetBackendService).

In this case, `backend-id` is the Identifier (name) of the backend to route requests to, and since we are going to reuse this xml, we will add the `{backendName}` token to be replaced later.

To create a [Microsoft.ApiManagement/service/apis/policies](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/apis/policies?pivots=deployment-language-bicep) resource, we then added the following Bicep to my template:

```Bicep
resource orderApiPolicy 'Microsoft.ApiManagement/service/apis/policies@2021-12-01-preview' = {
  name: 'policy'
  parent: orderApiResource
  properties: {
    value: replace(loadTextContent('apimPolicies/api.xml'), '{backendName}', orderBackendResource.name)
    format: 'xml'
  }
}
```

Here we are basically leveraging a combination of the [`replace`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions-string#replace) function and the [`loadTextContent`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions-files#loadtextcontent) function to read the XML file and replace the `{backendName}` token with the actual Backend API we want to route requests to.

## Adding a Fronted API Operation

A frontend API in API Management can define multiple operations. An operation is a combination of an HTTP verb and a URL template uniquely resolvable within the frontend API. Often operations map one-to-one to backend API endpoints. For more information, see [Mock API responses](https://learn.microsoft.com/en-us/azure/api-management/mock-api-responses).

To create a [Microsoft.ApiManagement/service/apis/operations](https://learn.microsoft.com/en-us/azure/templates/microsoft.apimanagement/service/apis/operations?pivots=deployment-language-bicep) resource, we added the following Bicep to my template:

```Bicep
resource getOrdersOperationResource 'Microsoft.ApiManagement/service/apis/operations@2021-12-01-preview' = {
  name: 'get-orders-storeid'
  parent: orderApiResource
  properties: {
    displayName: '/orders/{storeId} - GET'
    method: 'GET'
    urlTemplate: '/orders/{storeId}'
    templateParameters: [
      {
        name: 'storeId'
        type: 'string'
        required: true
      }
    ]
    responses: [
      {
        statusCode: 200
        description: 'Success'
      }
    ]
  }
}
```

## Adding a Operation policy

In order to make the Operation policy reusable, we saved it in a XML file `apimPolicies/operation.xml`:

```xml
In order to make the Fronted API policy reusable, we saved it in a XML file `apimPolicies/api.xml`:

```xml
<policies>
    <inbound>
        <base />
        <set-method>{method}</set-method>
        <rewrite-uri id="apim-generated-policy" template="{template}" />
        <set-header id="apim-generated-policy" name="Ocp-Apim-Subscription-Key" exists-action="delete" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
</outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

This policy does three things:

- `set-method`: sets the HTTP request method for a request. See [Set request method](https://learn.microsoft.com/en-us/azure/api-management/api-management-advanced-policies#SetRequestMethod)
- `rewrite-url`: converts a request URL from its public form to the form expected by the web service. See [Rewrite URL](https://learn.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#RewriteURL)
- `set-header`: assigns a value to an existing response and/or request header or adds/removes a new response and/or request header. See [Set HTTP header](https://learn.microsoft.com/en-us/azure/api-management/api-management-transformation-policies#SetHTTPheader)

## Repeat

Steps 3 - 7 are repeated for all Backend APIs, Frontend APIs, Policies, and operations that need to be defined on Azure API Management.

Hope this proves valuable to you, dear reader.

References:  
[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)  
[Azure API Management](https://azure.microsoft.com/services/api-management)  
[Azure Container Apps](https://azure.microsoft.com/services/container-apps)  
[Azure API Management terminology](https://learn.microsoft.com/en-us/azure/api-management/api-management-terminology)  
[Bicep functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-functions)  
[API Management policy reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)  
[Define resources with Bicep, ARM templates, and Terraform AzAPI provider](https://learn.microsoft.com/en-us/azure/templates/)  
[RedDog Codebase](https://github.com/Azure/reddog-code)  
[https://github.com/Azure/reddog-containerapps](https://github.com/Azure/reddog-containerappsc)