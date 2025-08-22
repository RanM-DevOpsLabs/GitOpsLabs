![download](https://github.com/user-attachments/assets/4612c096-61d2-4aa0-9d53-fa22bd2a32ac)
## Gitops with ArgoCD

This guide walks you through setting up a local Kubernetes cluster using Kind, installing ArgoCD, and deploying applications using GitOps principles.

## What is GitOps ?
GitOps is a practice of managing the realase life cycle of an application within the Git repo.
It is a on-growing adoption method by many Development teams, as it ease the deployment process, allowing the developers to stay focused with the source code repository. 
Narrowing down the applications overload often necessary for the CD process.

## What is ArgoCD
ArgoCD is a continuous delivery tool that implements the GitOps concept for Kubernetes resources.
As you'll see in this guide, by creating argocd CRD called Application, we can manager a group of k8s resources, fairly easily.
This by telling the ArgoCD controller the path of which our resouce definition files will be stored. 
When the controller "picks" them up, it will reconcile into the k8s environment:
- if they're not exist there, it would create them
- if they've been modified in default branch, it would update them
- if they've been removed - it would either:
  - remove then if prune is set
  - will mark them as "out of sync"

## Prerequisites

Before starting, ensure you have the following tools installed:

- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)


## End result
This guide will walk you through creating argocd application (see: https://argo-cd.readthedocs.io/en/stable/core_concepts/) to sync a specific repo and folderpath for continuously deploying resources into a k8s cluster.<br><br>
![argo-sync](https://github.com/user-attachments/assets/ba32b45d-0c47-4867-b821-e689e359d02e)

# GitOps Lab Setup Guide

## Step 1: Setting up Local Kind Cluster

Create a local Kubernetes cluster using the provided Kind configuration:

```bash
kind create cluster --config kind-config.yaml --name gitops-lab
```

This will create a cluster with:
- One control-plane node with ingress capabilities
- One worker node
- Port mapping from host port 433 to container port 443 for ingress access

Verify the cluster is running:
```bash
kubectl cluster-info --context kind-gitops-lab
```

## Step 2: Create Namespaces

Apply the namespace configuration to create the required namespaces:

```bash
kubectl apply -f namespaces.yaml
```

This creates three namespaces:
- `argocd` - for argo components
- `lab` - for development environment
- `staging` - for staging environment

Verify the namespaces were created:
```bash
kubectl get namespaces
```

## Step 3: Installing ArgoCD Core Version
Note: if you don't care for UI you can always download the core version, just change the URL to: `.../manifests/core-install.yaml`

Install ArgoCD full version in your cluster (with ui supported):

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Handle argocd-dex-server ImagePullBackOff issue:
Because we're running this setup locally, there'll be an issue with pulling this image.
To resolve this, simply:
1. Pull locally using docker pull: `docker pull ghcr.io/dexidp/dex:v2.43.0`
2. Edit deployment not to pull the image (force it to use locally mounted one) 
3. Then load it into kind cluster: `kind load docker-image ghcr.io/dexidp/dex:v2.43.0 --name gitops-lab` </br>
The argocd-dex-server pod should now be running.

(Optional) Wait for ArgoCD components to be ready:
```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-application-controller -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-repo-server -n argocd
```

## Step 4: Install NGINX Ingress Controller

Login to ArgoCD using the core mode:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/kind/deploy.yaml
```

## Step 5: Accessing argocd UI
## Creating a self-signed certificate for https
Although the argocd server is exposed in both http (port 80) and https (port 443 - as it can be viewed by its service configs), we'll create a self-signed certificate to connect to its UI using HTTPS.

This is for the sake of practicing this configuration :)

## Step 5A: Generate the cert and key
 - cd into certificates folder: `cd certificates`
 - create a self-sign certificate using Subject Alternative Name (SAN) (which is configured in openssl-san.cnf). This is a requirement by the k8s client.
```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout argocd.key \
  -out argocd.crt \
  -config openssl-san.cnf \
  -extensions req_ext
```
## Step 5B: Create the Kubernetes TLS secret
```bash
kubectl create secret tls argocd-tls \
  --key argocd.key \
  --cert argocd.crt \
  -n argocd
```

## Step 6: Configuring ingress and access argocd UI
`kubectl apply -f argo-server-ing.yaml` </br>
This ingress routes all external calls to the cluster on https (port 443) to argocd-server, allowing us to access its UI.

### Login to argo client
at: `https://localhost` you should be able to view argo login page (mind that we're on HTTPS!)

### Extract admin password
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d` </br>
username is `admin`

## Step 7: Create ArgoCD Applications

Create the two ArgoCD applications using the provided configurations:

### Create Lab Application
```bash
kubectl apply -f ArgoCD/applications/lab-app.yml
```

### Create Staging Application
```bash
kubectl apply -f ArgoCD/applications/stg-app.yml
```

Verify the applications were created:
```bash
kubectl get applications -n argocd
```

You can also check the application status using ArgoCD CLI:
```bash
argocd app list
argocd app get lab-app
argocd app get stg-app
```

## Step 6: Inspect Pods in Both Namespaces

Monitor the pods being deployed in both namespaces:

### Check Lab Namespace
```bash
kubectl get pods -n lab
kubectl describe pods -n lab
```

### Check Staging Namespace
```bash
kubectl get pods -n staging
kubectl describe pods -n staging
```

### Watch Pods in Real-time
You can watch the pods as they're being created and updated:
```bash
# Watch lab namespace
kubectl get pods -n lab -w

# In another terminal, watch staging namespace
kubectl get pods -n staging -w
```

## Application Details

### Lab Application (`lab-app.yml`)
- **Name**: lab-app
- **Namespace**: argocd
- **Source**: https://github.com/RanM-DevOpsLabs/GitOpsLabs.git
- **Path**: ArgoCD/.gitops/lab
- **Destination**: lab namespace
- **Sync Policy**: Automated with prune and self-heal enabled

### Staging Application (`stg-app.yml`)
- **Name**: stg-app  
- **Namespace**: argocd
- **Source**: https://github.com/RanM-DevOpsLabs/GitOpsLabs.git
- **Path**: ArgoCD/.gitops/staging
- **Destination**: staging namespace
- **Sync Policy**: Automated with prune and self-heal enabled

## Troubleshooting

### Check ArgoCD Application Status
```bash
argocd app get lab-app
argocd app get stg-app
```

### Force Sync Applications
If applications are not syncing automatically:
```bash
argocd app sync lab-app
argocd app sync stg-app
```

### Check ArgoCD Logs
```bash
kubectl logs -n argocd deployment/argocd-application-controller
kubectl logs -n argocd deployment/argocd-repo-server
```

### Check Events
```bash
kubectl get events -n lab --sort-by='.lastTimestamp'
kubectl get events -n staging --sort-by='.lastTimestamp'
```

## Cleanup

To clean up the resources when you're done:

```bash
# Delete applications
kubectl delete -f ArgoCD/applications/lab-app.yml
kubectl delete -f ArgoCD/applications/stg-app.yml

# Delete namespaces
kubectl delete -f namespaces.yaml

# Delete ArgoCD
kubectl delete namespace argocd

# Delete Kind cluster
kind delete cluster --name gitops-lab
```

## Next Steps

- Explore the ArgoCD applications and their sync status
- Make changes to the GitOps repository and observe automatic deployments
- Experiment with different sync policies and configurations
- Set up monitoring and observability for your applications
