# Expose an application with an ingress

In this lab, we will go through the steps of exposing an application outside of your Kubernetes cluster.

## Pre-reqs

Before you begin with this lab, you have to complete the previous one:

* [Build and deploy an application](./01-deploy-application.md)

## Tasks

### Deploy an ingress controller

Deploy the ingress controller by running the command below. The deployment file includes the necessary namespace as well.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

Verify that the ingress controller has been deployed correctly:

```bash
# View the nginx-ingress deployment
kubectl get deployments -n ingress-nginx

# View the pods managed by the deployment
kubectl get pods -n ingress-nginx
```

Verify that the ingress controller has gotten a public IP-address (this can take a few minutes):

```bash
kubectl get svc -n ingress-nginx

NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.0.25.228    51.124.87.220   80:30880/TCP,443:32491/TCP   7m34s
ingress-nginx-controller-admission   ClusterIP      10.0.197.154   <none>          443/TCP                      7m34s
```

Write down the `EXTERNAL-IP` of `ingress-nginx-controller` for the next command.

### Create an ingress rule

In order to expose `gpip`, we need to create an ingress rule. The ingress rule should listen on the address [http://gpip.external-ip.nip.io](http://gpip.external-ip.nip.io), where `external-ip` is replaced with the `EXTERNAL-IP` address from the previous command.

Run the following to create your ingress rule:

```bash
external_ip=<external-ip> # EXTERNAL-IP from the previous command
kubectl create ingress gpip \
    --rule="gpip.$external_ip.nip.io/=gpip:80" \
    --namespace=gpip
```

Verify that you can reach the application:

```bash
curl http://gpip.$external_ip.nip.io -v
```

## Summary

After finishing this lab, you should have achieved the following:

* Deployed an ingress controller to your cluster.
* Exposed an application through the ingress controller.

## Next step

When done with this lab, you can proceed with [enabling SSL/TLS for your ingress controller](./03-secure-ingress.md).
