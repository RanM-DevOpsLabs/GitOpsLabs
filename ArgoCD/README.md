# GitOps Lab Setup Guide

This guide walks you through setting up a local Kubernetes cluster using Kind, installing ArgoCD, and deploying applications using GitOps principles.

## Prerequisites

Before starting, ensure you have the following tools installed:

- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

## Step 1: Setting up Local Kind Cluster

Create a local Kubernetes cluster using the provided Kind configuration:

```bash
kind create cluster --config kind-config.yaml --name gitops-lab
```

This will create a cluster with:
- One control-plane node with ingress capabilities
- One worker node
- Port mapping from host port 8080 to container port 80 for ingress access

Verify the cluster is running:
```bash
kubectl cluster-info --context kind-gitops-lab
```

## Step 2: Create Namespaces

Apply the namespace configuration to create the required namespaces:

```bash
kubectl apply -f namespaces.yaml
```

This creates two namespaces:
- `lab` - for development environment
- `staging` - for staging environment

Verify the namespaces were created:
```bash
kubectl get namespaces
```

## Step 3: Installing ArgoCD Core Version

Install ArgoCD core version in your cluster:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
```

Wait for ArgoCD components to be ready:
```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-application-controller -n argocd
kubectl wait --for=condition=available --timeout=300s deployment/argocd-repo-server -n argocd
```

## Step 4: ArgoCD Login (Core Mode)

Login to ArgoCD using the core mode:

```bash
argocd login --core
```

Note: In core mode, ArgoCD doesn't expose a web UI or API server. All operations are performed through the CLI directly against the Kubernetes API.

## Step 5: Create ArgoCD Applications

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