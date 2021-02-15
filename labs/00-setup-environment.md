# Setup Kubernetes lab environment

In this lab, we will set up the Kubernetes lab environment used for the other labs during the workshop.

## Pre-reqs

In order to complete this lab, you will need the following:

* Read/write access to an [Azure subscription][azure-account]
* [Azure CLI][install-azurecli] installed
* [Docker][install-docker] installed
* WSL/Ubuntu/MacOS installed
* Git installed & configured

## Tasks

### Create a Resource Group

First, you'll need to authenticate against Azure:

```bash
az login
az account set -s <subscription-id>
```

> **Note:** If you don't know which location to use, `westeurope` is probably fine.

Create your resource group by running the following command:

```bash
group=<resource-group>
location=<location>
az group create -n $group -l $location
```

Set this resource group as your default one, together with the region you chose earlier:

```bash
az configure --defaults location=$location group=$group
```

### Create a container registry

> **Note:** The name of the container registry can only contain alphanumeric characters, and must be globally unique.

Create a container registry by running the following command:

```bash
acr=<acr-name>
az acr create \
    --name $acr \
    --sku Basic
```

Login to your registry, so you can use it with Docker:

```bash
az acr login --name $acr
```

### Create an AKS cluster

Find the latest supported version of Kubernetes for AKS:

```bash
version=$(az aks get-versions --query 'orchestrators[?isPreview == null].[orchestratorVersion][-1]' -o tsv)
```

Create an AKS cluster with the Kubernetes version you got in the previous step:

```bash
cluster=<cluster-name>
az aks create \
    --name $cluster \
    --kubernetes-version $version \
    --node-count 1 \
    --attach-acr $acr
```

The AKS cluster will be linked to the Azure Container Registry you created in the previous step.

### Install the Kubernetes CLI

> **Note:** You can also install Kubectl using the installation instructions from [Kubernetes documentation][install-kubectl].

Kubectl is used for interacting with the Kubernetes API. In order to have the right features, the version of Kubectl should match the Kubernetes version your cluster has.

Install Kubectl by running the following:

```bash
az aks install-cli --client-version $version
```

Verify that Kubectl has been installed correctly:

```bash
kubectl version
```

### Connect to the cluster

In order to authenticate against the Kubernetes API, you need to get your credentials. You can get the credentials for your cluster by running the following command:

```bash
az aks get-credentials --name $cluster
```

Verify that you can access the cluster:

```bash
kubectl get nodes
```

## Summary

After finishing this lab, you should have achieved the following:

* Created a resource group in your Azure subscription, and set it as a default.
* Created (and authenticated against) an Azure Container Registry which you will use for your Docker images.
* Created (and linked) an AKS cluster, which will act as your Kubernetes lab environment.
* Installed Kubectl, which is used for interacting with your Kubernetes cluster.
* Configured Kubectl, with credentials for your Kubernetes cluster.

## Next step

When done with this lab, you can proceed with [building and deploying an application](./01-deploy-application.md).

<!-- References -->

[azure-account]: https://azure.microsoft.com/free/
[install-azurecli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[install-docker]: https://docs.docker.com/get-docker/
