# DevSecOps-GitOps-End-to-End-Infra

GitOps infrastructure repository for the **voting application**. It holds Kubernetes manifests, Kustomize base and environment overlays (dev/prod), and Argo CD definitions. Changes pushed to this repo are synced to the cluster by Argo CD.

---

## Overview

| Component | Description |
|-----------|-------------|
| **vote** | Voting UI (image: `aryanatel/tourlyvote`) — 2 replicas in base |
| **result** | Results UI (image: `aryanatel/tourlyresult`) |
| **worker** | Background worker (image: `aryanatel/tourlyworker`) |
| **redis** | Redis cache |
| **db** | PostgreSQL 15 (with Secret for credentials) |

Environments:

- **dev** — namespace `dev`, image tags e.g. `v22`, name suffix `-dev`
- **prod** — namespace `prod`, image tags e.g. `v15`, name suffix `-prod`

---

## Repository Structure

```
DevSecOps-GitOps-End-to-End-Infra/
├── README.md
└── GitOps/
    ├── argoCD/
    │   ├── appliactionSet.yaml   # ApplicationSet: one Argo Application per env folder
    │   └── appProject.yaml      # AppProject "voting-project" (source repo, destinations, RBAC)
    ├── base/                     # Kustomize bases (shared manifests)
    │   ├── db-infra/             # Postgres deployment, service, secret
    │   ├── redis-infra/          # Redis deployment and service
    │   ├── vote-infra/           # Vote deployment and service
    │   ├── result-infra/         # Result deployment and service
    │   └── worker-infra/         # Worker deployment
    └── env/
        ├── dev/                  # Dev overlays (namespace: dev, nameSuffix: -dev)
        │   ├── db/
        │   ├── redis/
        │   ├── vote/
        │   ├── result/
        │   └── worker/
        └── prod/                 # Prod overlays (namespace: prod, nameSuffix: -prod)
            ├── db/
            ├── redis/
            ├── vote/
            ├── result/
            └── worker/
```

Each `env/<env>/<app>/kustomization.yaml` points at the corresponding `base/<app>-infra` and sets namespace, name suffix, and (for vote/result/worker) image tags.

---

## Prerequisites

- Kubernetes cluster (e.g. minikube, kind, EKS)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed and connected to the cluster
- (Optional) [Helm](https://helm.sh/) for monitoring stack

---

## GitOps with Argo CD

### 1. Bootstrap Argo CD (if not already installed)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Apply Argo CD project and ApplicationSet

From the repo root:

```bash
kubectl apply -f GitOps/argoCD/appProject.yaml
kubectl apply -f GitOps/argoCD/appliactionSet.yaml
```

This creates:

- **AppProject** `voting-project` — allows source repo and destinations (e.g. any namespace).
- **ApplicationSet** `voting-appset` — discovers `GitOps/env/dev/*` and `GitOps/env/prod/*` and creates one **Application** per directory (e.g. `vote`, `result`, `db`, `redis`, `worker` for dev and prod).  
  **Note:** The template currently sets `destination.namespace: dev`. To deploy prod overlays into the `prod` namespace, update the ApplicationSet to use a generator that sets namespace per path (e.g. from `path`).

Argo CD will sync each Application from this repo (branch `main`) with **auto-sync** and **CreateNamespace=true**.

### 3. Verify

```bash
kubectl get applications -n argocd
kubectl get pods -n dev
kubectl get pods -n prod
```

---

## Local / CI usage (without Argo CD)

Build and apply a single overlay locally:

```bash
# Dev vote overlay
kubectl apply -k GitOps/env/dev/vote

# Prod result overlay
kubectl apply -k GitOps/env/prod/result
```

Apply all dev components:

```bash
for dir in GitOps/env/dev/*/; do kubectl apply -k "$dir"; done
```

---

## Monitoring (Prometheus + Grafana)

Optional: deploy the Prometheus Community stack (Prometheus, Grafana, Alertmanager, node-exporter) for metrics and dashboards.

### Add Helm repo and install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### Access Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open http://localhost:3000 — default login: `admin` / `prom-operator`.

### If node-exporter has permission issues (e.g. OpenShift / restricted PSP)

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set prometheus-node-exporter.hostRootFsMount.enabled=false
```

---

## Changing image versions

- **Dev:** Edit image `newTag` in `GitOps/env/dev/vote/kustomization.yaml`, `GitOps/env/dev/result/kustomization.yaml`, and `GitOps/env/dev/worker/kustomization.yaml` (e.g. `v22` → `v23`).
- **Prod:** Same files under `GitOps/env/prod/` (e.g. `v15` → `v16`).

Commit and push; Argo CD will sync the new manifests.

---

## Related repository

Application source code and CI/CD (build, test, image push) typically live in a separate repo (e.g. **DevSecOps-GitOps-End-to-End**). This repo only defines **what** runs in the cluster (GitOps declarative infra).
