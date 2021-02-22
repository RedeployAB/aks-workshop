# Build and deploy an application

In this lab, we will go through the process of building and deploying an application to Kubernetes.

## Pre-reqs

Before you begin with this lab, you have to complete the previous one:

* [Setup Kubernetes lab environment](./00-setup-environment.md)

## Tasks

### Clone application repository

Clone the repository of [gpip][gpip], which we will use during the lab:

```bash
# cd ~/repositories
git clone https://github.com/RedeployAB/gpip.git
cd gpip
```

### Build Docker image

Build gpip's Docker image by running the following:

```bash
gpip_version=1.1.1
docker build --tag gpip:$gpip_version --tag gpip:latest --build-arg VERSION=$gpip_version .
```

Verify that the container runs as expected:

```bash
# Run the container
docker run -d -p 5050:5050 --name gpip gpip
curl localhost:5050

# Stop the container
docker stop gpip
```

### Push Docker image to registry

Before we push the image to our registry, we need to tag it with its fully qualified name:

```bash
docker image tag gpip $acr.azurecr.io/gpip:$gpip_version
docker image tag gpip $acr.azurecr.io/gpip:latest
```

> **Note:** If `--all-tags` isn't available for you, you can push each image tag instead.

When this is done, we can proceed with pushing the image to our repository:

```bash
docker image push --all-tags $acr.azurecr.io/gpip
```

### Deploy the application to Kubernetes

Create a namespace for the application to run within:

```bash
kubectl create namespace gpip
```

Deploy the application:

```bash
kubectl create deployment gpip \
    --namespace=gpip \
    --image=$acr.azurecr.io/gpip:$gpip_version \
    --port=5050
```

Verify that the deployment was successful:

```bash
kubectl get deployments --namespace=gpip
```

### Test the application within the cluster

Make the application discoverable within the cluster by creating a service:

```bash
kubectl create service clusterip gpip --tcp=80:5050 --namespace=gpip
```

Create a new pod running `curl` to verify that the service responds:

```bash
kubectl run curl --image=radial/busyboxplus:curl -i --tty --namespace=gpip
curl gpip
```

After verifying that the service works, you can delete the `curl` pod:

```bash
kubectl delete pod curl --namespace=gpip
```

## Summary

After finishing this lab, you should have achieved the following:

* Built a Docker image locally
* Tagged and pushed the image to a remote registry
* Deployed the image to a Kubernetes cluster
* Made the application discoverable within the cluster
* Connected to the application within the cluster

## Next step

When done with this lab, you can proceed with [exposing the application via an ingress controller](./02-deploy-ingress.md).

<!-- References -->

[gpip]: https://github.com/RedeployAB/gpip
