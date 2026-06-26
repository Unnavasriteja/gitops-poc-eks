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