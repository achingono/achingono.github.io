---
author: "Alfero Chingono"
title: "Consume Secrets in Azure Key vault From Kubernetes On-prem"
date: 2022-11-07T18:32:15Z
draft: false
description: "Using a Service Principal, it is possible to leverage Azure Key Vault on on-prem/non-Azure k8s clusters."
slug: consume-secrets-in-azure-keyvault-from-kubernetes-onprem
tags: [
  "kubernetes",
  "azure",
  "keyvault",
  "secrets"
]
categories: [
  "containers",
  "security"
]
image: "cover.png"
---

Recently, I had a customer ask if there is a way to leverage Azure Key Vault on on-prem/non-Azure k8s clusters. In this blog post, I will show the steps I followed to demonstrate the process of mounting Azure Key Vault secrets inside an on-prem Kubernetes cluster.

The Secrets Store CSI Driver on Azure Kubernetes Service (AKS) provides the following methods of identity-based access to your Azure key vault.

* An [Azure Active Directory pod identity](https://learn.microsoft.com/en-us/azure/aks/use-azure-ad-pod-identity) (preview)
* An [Azure Active Directory workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) (preview)
* A [user-assigned or system-assigned managed identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
* An [Azure Active Directory (AD) service principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)  

> **NOTE:** Service principals eventually expire and must be renewed to keep the cluster working. In addition, managing service principals adds complexity, thus it's recommended to use managed identities when working with Azure resources. Managed identities are the default authentication method for an AKS cluster.

Given that the use-case in question is not AKS and the cluster is not hosted in Azure, we are left with no other choice but to use service principals.

## Create a new Kubernetes Cluster

Since I did not have a running on-prem cluster, I followed instructions here at [How to Install Kubernetes Cluster on Debian 11 with Kubeadm](https://www.linuxtechi.com/install-kubernetes-cluster-on-debian/) to create a cluster on Hyper-V.  

## Create a new Azure key vault

In addition to a Kubernetes cluster, you'll need an Azure key vault resource that stores the secret content in the cloud.  

### Create a new Resource Group

To create a new resource group, execute the following command:

```bash
$GROUP_NAME=secret-store-example
$LOCATION=canadacentral

az group create -n $GROUP_NAME -l $LOCATION
```

### Create a new Azure Key Vault resource

Execute the following command to create a new Azure Key vault resource:

```bash
KEYVAULT_NAME=secret-store-6d9e5b0a
az keyvault create -n $KEYVAULT_NAME -g $GROUP_NAME -l $LOCATION
```

> **NOTE:** The key vault's name must be globally unique.

### Create Key Vault secrets

In this example, we'll create two plain-text secrets in our Key vault as follows:

```bash
az keyvault secret set --vault-name $KEYVAULT_NAME -n Username --value atomic_fifth
az keyvault secret set --vault-name $KEYVAULT_NAME -n Password --value Forty7&Scale
```
## Create a Service Principal

Next, let us create a service principal with the following commands:
```bash
$ SP_NAME=secret-store-service-principal
$ az ad sp create-for-rbac --skip-assignment --name $SP_NAME
```

The command should output a JSON object similar to this::

```json
{
  "appId": "<GUID>",
  "displayName": "secret-store-service-principal",
  "password": "<STRING>",
  "tenant": "<GUID>"
}
```

We will need all four values in later steps, so it's important to note them down at this point.

## Assign Permissions to Key vault

 Next, we assign the service principal `get` permissions to our key vault:

```bash
# Set environment variables
CLIENT_ID=<appId from previous step>

az keyvault set-policy -n $KEYVAULT_NAME --key-permissions get --spn $CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get --spn $CLIENT_ID
az keyvault set-policy -n $KEYVAULT_NAME --certificate-permissions get --spn $CLIENT_ID
```

The rest of this process is completed in our on-prem Kubernetes cluster.


## Install the Secrets Store CSI Driver

To install the [Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) on a fresh Kubernetes installation, execute the following script:

```bash
BASE_URL=https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy

kubectl apply -f $BASE_URL/rbac-secretproviderclass.yaml
kubectl apply -f $BASE_URL/csidriver.yaml
kubectl apply -f $BASE_URL/secrets-store.csi.x-k8s.io_secretproviderclasses.yaml
kubectl apply -f $BASE_URL/secrets-store.csi.x-k8s.io_secretproviderclasspodstatuses.yaml
kubectl apply -f $BASE_URL/secrets-store-csi-driver.yaml

# If using the driver to sync secrets-store content as Kubernetes Secrets, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f $BASE_URL/rbac-secretprovidersyncing.yaml

# If using the secret rotation feature, deploy the additional RBAC permissions
# required to enable this feature
kubectl apply -f $BASE_URL/rbac-secretproviderrotation.yaml

# If using the CSI Driver token requests feature (https://kubernetes-csi.github.io/docs/token-requests.html) to use
# pod/workload identity to request a token and use with providers
kubectl apply -f $BASE_URL/rbac-secretprovidertokenrequest.yaml

# [OPTIONAL] To deploy driver on windows nodes
kubectl apply -f $BASE_URL/secrets-store-csi-driver-windows.yaml
```

This will produce the following output:

```
serviceaccount/secrets-store-csi-driver created
clusterrole.rbac.authorization.k8s.io/secretproviderclasses-role created
clusterrolebinding.rbac.authorization.k8s.io/secretproviderclasses-rolebinding created
csidriver.storage.k8s.io/secrets-store.csi.k8s.io created
customresourcedefinition.apiextensions.k8s.io/secretproviderclasses.secrets-store.csi.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/secretproviderclasspodstatuses.secrets-store.csi.x-k8s.io created
daemonset.apps/csi-secrets-store created
clusterrole.rbac.authorization.k8s.io/secretprovidersyncing-role created
clusterrolebinding.rbac.authorization.k8s.io/secretprovidersyncing-rolebinding created
clusterrole.rbac.authorization.k8s.io/secretproviderrotation-role created
clusterrolebinding.rbac.authorization.k8s.io/secretproviderrotation-rolebinding created
clusterrole.rbac.authorization.k8s.io/secretprovidertokenrequest-role created
clusterrolebinding.rbac.authorization.k8s.io/secretprovidertokenrequest-rolebinding created
daemonset.apps/csi-secrets-store-windows created
```

## Install the Azure Key Vault Provider for Secrets Store CSI Driver

[Azure Key Vault provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure) allows you to get secret contents stored in an Azure Key Vault instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods. To install the Azure Key Vault provider for Secrets Store CSI Driver, execute the following command on the cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/deployment/provider-azure-installer.yaml
```

This will produce output as follows:

```
serviceaccount/csi-secrets-store-provider-azure created
daemonset.apps/csi-secrets-store-provider-azure created
```

## Create Service Principal Kubernetes Secret

In order for our on-prem Kubernetes cluster to connect to Azure Key Vault, we will need to create Kubernetes secrets accessible to the Secret Store CSI Driver. To create the secrets, execute the following commands against the cluster:

```bash
$SECRET_NAME=secrets-store-creds
# $CLIENT_ID is the App ID of your service principal
# $CLIENT_SECRET is the Password of your service principal
kubectl create secret generic $SECRET_NAME --from-literal clientid=$CLIENT_ID --from-literal clientsecret=$CLIENT_SECRET

# Label the secret
# Refer to https://secrets-store-csi-driver.sigs.k8s.io/load-tests.html for more details on why this is necessary in future releases.
kubectl label secret $SECRET_NAME secrets-store.csi.k8s.io/used=true
```

> **NOTE:** The Kubernetes secret is stored as plaintext in etcd. Additional security measures are needed to ensure access is restricted.

## Deploy the Secret Provider Class

Create a YAML file:

```bash
nano secret-provider-class.yaml
```

Paste the following contents and save the file:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault                                # The Name of the Secret Provider Class
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManadedIdentity: "false"
    userAssignedIdentityID: ""
    keyvaultName: "secret-store-6d9e5b0a"             # The name of the Key Vault we created in earlier steps
    tenantId: "c6a70df6-0848-491e-9ae6-038dc1d12160"  # The tenant Id returned from the `az ad sp create-for-rbac` command
    objects:  |
      array:
        - |
          objectName: Username                        # The Key Vault secret we created in previous step
          objectType: secret
          objectVersion: ""
        - |
          objectName: Password                        # The Key Vault secret we created in previous step
          objectType: secret
          objectVersion: ""
```

Deploy the Secret Provider Class:

```bash
kubectl apply -f secret-provider-class.yaml
```

## Create a Pod to consume Azure Key Vault Secrets


Create a YAML file:

```bash
nano busybox-secrets-store-inline.yaml
```

Paste the following contents and save the file:

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: busybox-secrets-store-inline
spec:
  containers:
  - name: busybox
    image: k8s.gcr.io/e2e-test-images/busybox:1.29
    command:
      - "/bin/sleep"
      - "10000"
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-keyvault"     # The Name of the Secret Provider Class created in previous step
        nodePublishSecretRef:                       # Only required when using service principal mode
          name: secrets-store-creds                 # The name of the Kubernetes secret created in previous step.
```

Deploy the pod:

```bash
kubectl apply -f busybox-secrets-store-inline.yaml
```

The command should produce the following output:

```
pod/busybox-secrets-store-inline created
```

## Verify Azure Key Vault Secrets in container

To verify that our secrets are accessible inside our container, we execute the following commands:

```bash
$ kubectl exec busybox-secrets-store-inline -- ls /mnt/secrets-store
Username
Password

$ kubectl exec busybox-secrets-store-inline -- cat /mnt/secrets-store/Username
atomic_fifth

$ kubectl exec busybox-secrets-store-inline -- cat /mnt/secrets-store/Username
Forty7&Scale
```

And there you have it! We managed to retrieve Azure Key Vault secrets from an on-prem Kubernetes cluster using a Service Principal. Hope the proves valuable to you, dear reader.

References:  
[Use Azure KeyVaults in BC OnPrem](https://www.j3ns.de/d365-business-central/use-azure-keyvaults-in-bc-onprem/)  
[Service Principal | Azure Key Vault Provider for Secrets Store CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/configurations/identity-access-modes/service-principal-mode/)  
[Use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)  
[Kubernetes Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver/)  
[Installation - Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)  
[Installation | Azure Key Vault Provider for Secrets Store CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/getting-started/installation/)  
[Use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)  
[Service Principal Examples for Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure/tree/master/examples/service-principal)  