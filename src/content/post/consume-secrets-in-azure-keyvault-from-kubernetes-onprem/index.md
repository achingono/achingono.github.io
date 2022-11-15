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
  "cloud",
  "key management",
  "security"
]
image: "cover.png"
---

Recently, a customer inquired if there is a way to leverage Azure Key Vault on on-prem/non-Azure k8s clusters. In this blog post, I will show the steps we followed to demonstrate the process of mounting Azure Key Vault secrets inside an on-prem Kubernetes cluster.

- [**Azure Key Vault**](https://learn.microsoft.com/en-us/azure/key-vault/general/overview) is one of several key management solutions in Azure, and can be used to Securely store and tightly control access to tokens, passwords, [certificates](https://learn.microsoft.com/en-us/azure/key-vault/certificates/), [API keys](https://learn.microsoft.com/en-us/azure/key-vault/keys/), and other [secrets](https://learn.microsoft.com/en-us/azure/key-vault/secrets/). Centralizing storage of application secrets in Azure Key Vault allows you to control their distribution. Key Vault greatly reduces the chances that secrets may be accidentally leaked. When using Key Vault, application developers no longer need to store security information in their application. Not having to store security information in applications eliminates the need to make this information part of the code.
- [**Kubernetes**](https://kubernetes.io/docs/concepts/overview/) is a portable, extensible, open source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem and focuses on the application workloads, not the underlying infrastructure components.
- [**Kubernetes Secrets Store CSI Driver**](https://secrets-store-csi-driver.sigs.k8s.io/) integrates secrets stores with Kubernetes via a [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/) volume. It allows Kubernetes to mount multiple secrets, keys, and certificates stored in enterprise-grade external secrets stores, such as Azure Key Vault, into pods as a volume. Once the Volume is attached, the data in it is mounted into the container’s file system.

The Secrets Store CSI Driver on Azure Kubernetes Service (AKS) provides the following methods of identity-based access to your Azure key vault.

- An [Azure Active Directory pod identity](https://learn.microsoft.com/en-us/azure/aks/use-azure-ad-pod-identity) (preview)
- An [Azure Active Directory workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) (preview)
- A [user-assigned or system-assigned managed identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- An [Azure Active Directory (AD) service principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)  

> **NOTE:** Service principals eventually expire and must be renewed to keep the cluster working. In addition, managing service principals adds complexity, thus it's recommended to use managed identities when working with Azure resources. Managed identities are the default authentication method for an AKS cluster.

Given that the use-case in question is not AKS and the cluster is not hosted in Azure, we are left with no other option but to use service principals to [access Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features#key-vault-authentication-options).

## Create a new Kubernetes Cluster

If you do not have an on-prem Kubernetes cluster, you can follow instructions at [How to Install Kubernetes Cluster on Debian 11 with Kubeadm](https://www.linuxtechi.com/install-kubernetes-cluster-on-debian/) to create a cluster on Hyper-V.  

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

In this example, you'll create two plain-text secrets in our Key vault as follows:

```bash
az keyvault secret set --vault-name $KEYVAULT_NAME -n Username --value atomic_fifth
az keyvault secret set --vault-name $KEYVAULT_NAME -n Password --value Forty7&Scale
```

## Create a Service Principal

Next, create a service principal with the following commands:

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

You will need all four values in later steps, so it's important to note them down at this point.

## Assign Permissions to Key vault

Access to a key vault is controlled through two interfaces: the management plane and the data plane. The management plane is where you manage Key Vault itself. Operations in this plane include creating and deleting key vaults, retrieving Key Vault properties, and updating access policies. The data plane is where you work with the data stored in a key vault. You can add, delete, and modify keys, secrets, and certificates. To grant our service principal permissions to read keys, secrets, and certificates, we execute the following commands:

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

In order for your on-prem Kubernetes cluster to connect to Azure Key Vault, you will need to create [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) accessible to the Secret Store CSI Driver. To create the secrets, execute the following commands against the cluster:

```bash
$SECRET_NAME=secrets-store-creds
# $CLIENT_ID is the App ID of your service principal
# $CLIENT_SECRET is the Password of your service principal
kubectl create secret generic $SECRET_NAME --from-literal clientid=$CLIENT_ID --from-literal clientsecret=$CLIENT_SECRET

# Label the secret
# Refer to https://secrets-store-csi-driver.sigs.k8s.io/load-tests.html for more details on why this is necessary in future releases.
kubectl label secret $SECRET_NAME secrets-store.csi.k8s.io/used=true
```

> **NOTE:**  
> Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.
>
> In order to restrict access to Kubernetes secrets, consider enabling or configuring [RBAC rules](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) with least-privilege access to Secrets.
>
> See [Information security for Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#information-security-for-secrets) for more details.

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
    useVMManagedIdentity: "false"
    userAssignedIdentityID: ""
    keyvaultName: "secret-store-6d9e5b0a"             # The name of the Key Vault we created in earlier steps
    tenantId: "<GUID>"                                # The tenant Id returned from the `az ad sp create-for-rbac` command
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

To verify that your secrets are accessible inside our container, you execute the following commands:

```bash
$ kubectl exec busybox-secrets-store-inline -- ls /mnt/secrets-store
Username
Password

$ kubectl exec busybox-secrets-store-inline -- cat /mnt/secrets-store/Username
atomic_fifth

$ kubectl exec busybox-secrets-store-inline -- cat /mnt/secrets-store/Username
Forty7&Scale
```

> **Important:**  
> You can reduce the exposure of your vaults by specifying which IP addresses have access to them. The [Virtual network service endpoints for Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints) allow you to restrict access to a list of IPv4 (internet protocol version 4) address ranges. Any user connecting to your key vault from outside those sources is denied access.  
>
> For more information, see [Configure Azure Key Vault firewalls and virtual networks](https://learn.microsoft.com/en-us/azure/key-vault/general/network-security)

And there you have it! We managed to retrieve Azure Key Vault secrets from an on-prem Kubernetes cluster using a Service Principal. Hope the proves valuable to you, dear reader.

Credits:  
Thanks to my colleagues [Ravi Yadav](https://mvp.microsoft.com/en-us/PublicProfile/5002189) and [Hammad Aslam](https://hammadaslam.com/) for the important pointers.

References:  
[Use Azure KeyVaults in BC OnPrem](https://www.j3ns.de/d365-business-central/use-azure-keyvaults-in-bc-onprem/)  
[Service Principal | Azure Key Vault Provider for Secrets Store CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/configurations/identity-access-modes/service-principal-mode/)  
[Use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)  
[Kubernetes Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver/)  
[Installation - Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/getting-started/installation.html)  
[Installation | Azure Key Vault Provider for Secrets Store CSI Driver](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/getting-started/installation/)  
[Use the Azure Key Vault Provider for Secrets Store CSI Driver in an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)  
[Service Principal Examples for Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure/tree/master/examples/service-principal)  
[Azure Key Vault security](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features)  
