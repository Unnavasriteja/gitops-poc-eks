# GitOps POC for EKS Workloads

## The Problem This Solves

Traditional CI/CD pipelines push changes directly to Kubernetes clusters. This means:
- The pipeline needs cluster credentials — a security risk
- Manual `kubectl apply` commands leave no audit trail
- There is no single source of truth for what is actually running in the cluster
- Rollbacks require pipeline re-runs or manual intervention

## The Solution: GitOps with ArgoCD and Helm

This project makes **Git the single source of truth** for all Kubernetes deployments. ArgoCD runs inside the cluster and continuously reconciles the cluster state toward what is defined in Git.

- No pipeline ever touches the cluster directly
- Every change is a Git commit — fully auditable
- Drift is detected and auto-corrected by ArgoCD
- Rollback = `git revert` — one command, instant

## Architecture
Developer → Git Push → GitHub Repo → ArgoCD detects change → Cluster reconciles

### App of Apps Pattern

A single root ArgoCD Application watches the `apps/` folder and automatically manages all environment Applications. Adding a new environment means adding a folder in Git — nothing else.

root-app (ArgoCD)

├── nginx-dev

├── nginx-staging

└── nginx-prod

## Repository Structure

gitops-poc-eks/

├── apps/                        # ArgoCD Application manifests

│   ├── root-app.yaml            # App of Apps entry point

│   ├── dev/application.yaml

│   ├── staging/application.yaml

│   └── prod/application.yaml

├── charts/nginx-app/            # Helm chart for nginx

│   ├── Chart.yaml

│   ├── values.yaml              # Base defaults

│   └── templates/

│       ├── deployment.yaml      # With liveness and readiness probes

│       ├── service.yaml

│       ├── ingress.yaml

│       ├── hpa.yaml             # Auto-scaling based on CPU

│       └── networkpolicy.yaml   # Least-privilege pod networking

└── envs/                        # Per-environment value overrides

├── dev/values.yaml

├── staging/values.yaml

└── prod/values.yaml

## Environment Differences

| Setting | Dev | Staging | Prod |
|---|---|---|---|
| Replicas | 1 | 2 | 3 |
| CPU Request | 100m | 150m | 200m |
| Max Replicas (HPA) | 2 | 5 | 10 |
| CPU Scale Threshold | 80% | 70% | 60% |

## Key Design Decisions

**Why App of Apps?**
Registering each environment Application manually in ArgoCD doesn't scale. The root app watches the `apps/` folder — new environments are added by creating a folder in Git, not by clicking in the UI.

**Why per-environment values files?**
The Helm chart is the template — it defines the shape of the app. The `envs/` folder defines the differences. Dev, staging, and prod run the same chart with different resource allocations and replica counts.

**Why NetworkPolicy?**
By default all pods can communicate with each other. NetworkPolicy enforces least-privilege networking — nginx only accepts traffic from the ingress controller.

**Why HPA?**
Fixed replica counts don't handle real traffic patterns. HPA watches CPU utilization and scales pods automatically within defined boundaries — no manual intervention needed.

## Local Setup

### Prerequisites
- k3d
- kubectl
- helm

### Create a local cluster

```bash
k3d cluster create gitops-poc --agents 2
```

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Register the root app

```bash
kubectl apply -f apps/root-app.yaml
```

ArgoCD will automatically discover and deploy all environment Applications.

### Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Open: https://localhost:8080

## Deployment Flow

1. Developer raises a PR to update image tag or config in `envs/<environment>/values.yaml`
2. PR is reviewed and merged to `master`
3. ArgoCD detects the Git change within 3 minutes
4. ArgoCD reconciles the cluster — no pipeline, no manual apply
5. If drift is detected at any point, ArgoCD self-heals automatically