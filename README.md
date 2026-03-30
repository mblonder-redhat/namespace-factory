# GitOps Namespace Factory

This repository implements a "Namespace Factory" that provisions namespaces and baseline controls using ArgoCD dynamically.

## The Folder Structure
```
.
├── bootstrap/
│   ├── project.yaml
│   └── root-app.yaml
├── namespace-factory/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── namespace.yaml
│       ├── quota.yaml
│       ├── limitrange.yaml
│       └── rbac.yaml
├── tenants/
│   └── team-a.yaml
└── README.md
```

## Architectural Decisions

### ApplicationSet
I chose to use an ArgoCD ApplicationSet so that instead of manually creating individual ArgoCD Applications for every team, the ApplicationSet monitors the tenants/ directory. 
In order to add a new team, a single file needs to be pushed into that folder. This file must contain all the required values for the team, just as in the given example of "team-a". 
The factory then detects the new file and creates the entire environment automatically.

### Avoiding cluster-admin
A requirement for this project was to avoid assuming cluster-admin privileges for the automation. I've achieved this by using an ArgoCD AppProject (project.yaml).
This way I've explicitly restricted the factory to only manage Namespaces, ResourceQuotas, LimitRanges, and RoleBindings.
In addition, by using a dedicated Project, I've ensured that the automation cannot accidentally modify cluster-wide resources, keeping the cluster secure.

### Maintainability
By using a Helm chart there is not need to deal with the Kubernetes logic. 
The user only needs to provide a simple key-value file defining their requirements for each new team they want to add. This makes the system simple and maintainable without needing to handle the Kubernetes in the background.

### Sequence Enforcement
In order to enforce the order of the resources creation in a logical order, I've used ArgoCD Sync Waves:
1. Wave 0: The namespace is the first resource to be created, since all other resources must reside in an existing namespace.
2. Wave 1: ResourceQuotas and LimitRanges
3. Wave 2: RoleBindings

## Prerequisites
1. Minikube 
2. Podman or Docker
3. kubectl

## Setup
I've provided a short script in order to get the system up and running.
```
#!/bin/bash
git clone https://github.com/mblonder-redhat/namespace-factory.git
cd namespace-factory/

minikube start --driver=<podman/docker>

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)

kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/appset.yaml

echo "------------------------------------------------"
echo "ArgoCD URL: https://localhost:8080"
echo "Username: admin"
echo "Password: $PASSWORD"
echo "------------------------------------------------"

kubectl port-forward svc/argocd-server -n argocd 8080:443
```
