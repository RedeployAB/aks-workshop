# Set up CI/CD with Flux

In this lab, we will set up CI/CD against your cluster with [Flux][flux].

## Pre-reqs

Before you begin with this lab, you need to have performed the following labs:

* [Setup Kubernetes lab environment](./00-setup-environment.md)
* [Build and deploy an application](./01-deploy-application.md)
* [Expose an application with an ingress](./02-deploy-ingress.md)

You also need a GitHub account where you can create a repository.

## Tasks

### Generate a personal access token

Sign in to GitHub with the account that you'll be using for the lab.

Create a personal access token that you can use together with Flux. Follow the [GitHub guide][github-pat] in order to do so. Flux CLI needs to be able to create repositories for you (check everything under `repo`).

Export your GitHub personal access token and username as environment variables:

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

### Install the Flux CLI

If you have Homebrew installed on your computer, you can install Flux with the following command:

```bash
brew install fluxcd/tap/flux
```

Otherwise, install flux using their installer bash script:

```bash
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

Verify that flux has been installed correctly:

```bash
flux --version
```

### Install Flux components

First, check that you are connected to the right cluster:

```bash
kubectl config get-contexts
```

You should see a little `*` showing the current cluster.

Run the pre-flight checks for Flux:

```bash
flux check --pre
```

If everything is good you should get a message saying that the pre-req checks passed.

Run the bootstrap command:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-cluster \
  --branch=main \
  --path=./clusters/lab-cluster \
  --personal
```

This will create a repository if it doesn't already exist. It will also commit the manifests for the Flux components to the `main` branch. Then it configures your cluster to synchronize with the specified path in the repository.

If everything went smoothly you should get the message "bootstrap finished" at the end of the command output.

### Export resource manifests

In this lab, we will be deploying the `gpip` application again. Start by forking the [repository][gpip] to your GitHub account. Then clone the forked repository to your computer.

```bash
git clone https://github.com/$GITHUB_USER/gpip
cd ./gpip
```

Create a new directory where we can put our deployment manifests for `gpip`:

```bash
mkdir deploy && cd deploy
```

We will export the manifests we previously created and migrate to a declarative object configuration for our cluster. Start by exporting the `namespace` resource:

```bash
kubectl get namespace gpip -o yaml > namespace.yaml
```

The current resource config will be exported to `namespace.yaml`. Edit the file and manually remove the fields `status`, `spec`, and all fields inside `metadata` except for `name`.

Your file should look like this:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gpip

```

Now export the deployment resource to `deployment.yaml`:

```bash
kubectl get deployments gpip -n gpip -o yaml > deployment.yaml
```

Edit the file and remove the fields `status` and everything within `metadata` except for `labels`, `name` and `namespace`. Your file should look similar to this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: gpip
  name: gpip
  namespace: gpip
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: gpip
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: gpip
    spec:
      containers:
      - image: <registry>.azurecr.io/gpip:1.1.1
        imagePullPolicy: IfNotPresent
        name: gpip
        ports:
        - containerPort: 5050
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

```

Export the service resource to `service.yaml` followed by the ingress resource to `ingress.yaml`:

```bash
kubectl get svc gpip -n gpip -o yaml > service.yaml
kubectl get ingress gpip -n gpip -o yaml > ingress.yaml
```

Remove the fields all fields within `metadata` except for `labels`, `name` and `namespace`, and the field `status` inside both `service.yaml` and `ingress.yaml`.

The files should look something like these:

#### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: gpip
  name: gpip
  namespace: gpip
spec:
  clusterIP: 10.0.254.75
  ports:
  - name: 80-5050
    port: 80
    protocol: TCP
    targetPort: 5050
  selector:
    app: gpip
  sessionAffinity: None
  type: ClusterIP

```

#### `ingress.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gpip
  namespace: gpip
spec:
  rules:
  - host: gpip.<external-ip>.nip.io
    http:
      paths:
      - backend:
          serviceName: gpip
          servicePort: 80
        path: /
        pathType: Exact

```

You should now have the following files inside the `./deploy` directory:

```bash
tree
.
├── deployment.yaml
├── ingress.yaml
├── namespace.yaml
└── service.yaml

0 directories, 4 files
```

> **Note:** You might get a warning about an API deprecation, but this is expected and can safely be ignored (for now).

Validate that the files are still correctly formatted by running the following command:

```bash
kubectl apply -f . --dry-run=true
```

Commit and push the manifests to your repository:

```bash
git add -A && git commit -m "Add deployment manifests"
git push
```

### Add application to Flux

Our application is now ready to added to Flux. Clone the repository Flux bootstraped for you earlier:

```bash
git clone https://github.com/$GITHUB_USER/flux-cluster
cd ./flux-cluster
```

Create a GitRepository manifest pointing to your `gpip` repository:

```bash
flux create source git gpip \
  --url=https://github.com/$GITHUB_USER/gpip \
  --branch=main \
  --interval=30s \
  --export > ./clusters/lab-cluster/gpip-source.yaml
```

Commit and push the manifest to your `flux-cluster` repository:

```bash
git add -A && git commit -m "Add gpip GitRepository"
git push
```

### Deploy the application with Flux

In order for Flux to fetch and deploy our application, we need to create a Kustomization manifest for `gpip`.

Run the following command to create the Kustomization manifest:

```bash
flux create kustomization gpip \
  --source=gpip \
  --path="./deploy" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/lab-cluster/gpip-kustomization.yaml
```

Delete the current `service` resource since it will conflict with this deployment:

```bash
kubectl delete svc gpip -n gpip
```

Commit and push the manifest to your cluster repository:

```bash
git add -A && git commit -m "Add gpip Kustomization"
git push
```

Check that Flux deploys the manifests:

```bash
flux get kustomizations
```

If it doesn't happen anything immediately, you can trigger a reconcile with the following command:

```bash
flux reconcile kustomization gpip
```

You have now set up CI/CD against the `gpip` repository. Well done!

## Summary

After finishing this lab, you should have achieved the following:

* Installed the Flux CLI to your computer
* Installed Flux to your cluster
* Exported cluster resources as manifests and deployed it to your application repository
* Configured Flux to deploy your application based on the application repo manifests

## Next step

When done with this lab, you can proceed with [scaling and monitoring your application](./05-scale-and-monitor.md).

<!-- References -->

[gpip]: https://github.com/RedeployAB/gpip
[github-pat]: https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token
[flux]: https://toolkit.fluxcd.io/
