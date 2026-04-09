# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Kubernetes homelab running on Raspberry Pi 5s, managed through GitOps using ArgoCD. All cluster configurations are stored in this repository and automatically synced to the cluster.

## Architecture

**Infrastructure Stack:**
- Compute: Raspberry Pi 5s
- CRI: containerd
- CNI: Cilium (kube-proxy-free)
- GitOps: ArgoCD (syncs cluster state from this repo)
- Package Management: Helm charts

**Directory Structure:**
- `k8s/apps/`: Helm charts for applications deployed to the cluster
- `k8s/argocd/`: ArgoCD Application manifests that reference Helm charts in `k8s/apps/`

## GitOps Workflow

All changes to the cluster are made by modifying files in this repository. ArgoCD watches this repo and automatically applies changes to the cluster.

**To deploy a new application:**
1. Create a Helm chart in `k8s/apps/<app-name>/`
   - `Chart.yaml`: Chart metadata
   - `values.yaml`: Default values (can be mostly empty)
   - `templates/`: Kubernetes manifests (can use Helm templating)
2. Create an ArgoCD Application manifest in `k8s/argocd/<app-name>.yaml` that references the Helm chart
3. Commit and push - ArgoCD will automatically deploy

**ArgoCD Application Pattern:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  project: default
  source:
    repoURL: git@github.com:cobyforrester/homelab.git
    targetRevision: HEAD
    path: k8s/apps/<app-name>
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <app-name>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Common Commands

### Connecting to the Cluster from Laptop

```bash
# Copy kubeconfig from control node
ssh $USERNAME@$PI_IP_ADDRESS "sudo cat /etc/kubernetes/admin.conf" > ~/.kube/config-pi
export KUBECONFIG=~/.kube/config-pi

# Verify connection
kubectl get nodes
```

### ArgoCD Access

```bash
# Port forward to ArgoCD server
kubectl -n argocd port-forward svc/argocd-server 8443:443 --address 127.0.0.1

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo

# Access UI at https://localhost:8443
```

### Helm Operations

```bash
# Test rendering a chart locally before committing
helm template <release-name> k8s/apps/<app-name>

# Validate chart
helm lint k8s/apps/<app-name>
```

## Installed Operators

- **CloudNativePG (CNPG)**: Manages PostgreSQL clusters
  - Use `Cluster` CRD for PostgreSQL instances
  - Reference image catalog: `postgresql-minimal-trixie`
  - Example in `k8s/apps/vitagraphs/templates/cnpg.yaml`

## Notes

- The control plane node has its taint removed, allowing workloads to run on it
- Static IPs are assigned via the router for stable node connections
- Swap is disabled on all nodes (required for Kubernetes)
- cgroups are enabled for resource partitioning
