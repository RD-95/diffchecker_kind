# diffchecker_kind

Kubernetes deployment for the [diffchecker](https://github.com/RD-95/diffchecker) Flask app using **kind** (Kubernetes in Docker) and **Helm**.

## Overview

- **App**: A web-based text diff checker — paste two texts, click Compare, see differences highlighted
- **Image**: Pulled from GHCR (`ghcr.io/rd-95/diffchecker:latest`)
- **Cluster**: Local kind cluster
- **Ingress**: nginx ingress controller
- **Autoscaling**: HPA (2–10 pods, scales at 70% CPU)

## Repo Structure

```
diffchecker_kind/
  kind-config.yaml              # kind cluster config with port mappings
  charts/diffchecker/
    Chart.yaml                  # Helm chart metadata
    values.yaml                 # All configurable values
    templates/
      namespace.yaml            # Creates diffchecker namespace
      deployment.yaml           # Deploys the Flask app
      service.yaml              # ClusterIP service
      ingress.yaml              # nginx ingress (host: localhost)
      hpa.yaml                  # HorizontalPodAutoscaler
```

## Prerequisites

Install the following inside WSL2:

### Docker Engine
```bash
curl -fsSL https://get.docker.com | sh
sudo service docker start
sudo usermod -aG docker $USER
newgrp docker
```

### kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

### helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Setup & Deployment

### 1. Clone the repo
```bash
mkdir -p ~/repos
cd ~/repos
git clone https://github.com/RD-95/diffchecker_kind.git
cd diffchecker_kind
```

### 2. Create kind cluster
```bash
kind create cluster --name diffchecker --config kind-config.yaml
```

### 3. Install nginx ingress controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for it to be ready:
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 4. Install metrics-server (required for HPA)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 5. Deploy with Helm
```bash
helm install diffchecker ./charts/diffchecker
```

## Accessing the App

Open your browser and go to:
```
http://localhost
```

If that doesn't work (WSL2 networking), use port-forward:
```bash
kubectl port-forward svc/diffchecker-service 8080:80 -n diffchecker
```
Then open `http://localhost:8080`.

## Useful Commands

```bash
# Check pods
kubectl get pods -n diffchecker

# Check ingress
kubectl get ingress -n diffchecker

# Check all resources
kubectl get all -n diffchecker

# Check HPA status
kubectl get hpa -n diffchecker

# View pod logs
kubectl logs -l app=diffchecker -n diffchecker

# Upgrade helm release after changes
helm upgrade diffchecker ./charts/diffchecker

# Uninstall
helm uninstall diffchecker

# Delete kind cluster
kind delete cluster --name diffchecker
```

## Configuration

All values are in `charts/diffchecker/values.yaml`:

| Key | Default | Description |
|---|---|---|
| `image.repository` | `ghcr.io/rd-95/diffchecker` | Container image |
| `image.tag` | `latest` | Image tag |
| `replicas` | `2` | Number of pod replicas |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.port` | `80` | Service port |
| `service.targetPort` | `5000` | Flask app port |
| `ingress.host` | `localhost` | Ingress hostname |
| `hpa.minReplicas` | `2` | Minimum pods |
| `hpa.maxReplicas` | `10` | Maximum pods |
| `hpa.cpuUtilization` | `70` | CPU % threshold for scaling |

---

## ArgoCD Concepts

> **What is ArgoCD?**
> ArgoCD is a **GitOps continuous delivery tool** for Kubernetes. It watches a Git repository and automatically syncs your cluster to match whatever is declared there — your Git repo becomes the single source of truth.

---

### ArgoCD Server

The **ArgoCD server** (`argocd-server`) is the **main application component** of ArgoCD. Think of it as the brain and the face of the system.

**What it does:**
- Serves the **web UI** (the dashboard you open in the browser)
- Exposes the **REST API** and **gRPC API** (used by the `argocd` CLI)
- Handles **authentication and RBAC** (who can do what)
- Shows application sync status, diffs, and deployment history

**In Kubernetes, it appears as:**
```bash
kubectl get pods -n argocd
# NAME                                READY   STATUS
# argocd-server-7d5b9f4c6-xxxxx      1/1     Running
```

---

### ArgoCD Components (all pods)

ArgoCD is not just one pod — it is a set of microservices, each doing a specific job:

| Component | Pod Name | What it does |
|---|---|---|
| **ArgoCD Server** | `argocd-server` | Web UI + REST/gRPC API. The front door for users and CLI. |
| **Repo Server** | `argocd-repo-server` | Clones Git repos, renders Helm charts / Kustomize / plain YAML into final manifests. |
| **Application Controller** | `argocd-application-controller` | The reconciliation engine — compares *desired state* (Git) vs *live state* (cluster) and syncs them. |
| **Dex Server** | `argocd-dex-server` | Optional SSO/OIDC provider for login (GitHub, Google, LDAP). |
| **Redis** | `argocd-redis` | In-memory cache used by other components for performance. |
| **Notifications Controller** | `argocd-notifications-controller` | Sends alerts (Slack, email, etc.) on sync events. |

**How they work together (flow):**

```
You (browser / argocd CLI)
          │
          ▼
   argocd-server              ← you interact here (UI + API)
          │
          ├──► argocd-repo-server         ← fetches Git repo, renders Helm/Kustomize manifests
          │
          └──► argocd-application-controller  ← compares Git vs cluster, applies changes
                        │
                        ▼
                 Kubernetes API            ← actual resources (Deployments, Services, etc.)
```

---

### ArgoCD Service (`svc`)

In Kubernetes, a **Service (`svc`)** is a stable network endpoint that routes traffic to one or more pods. Without a Service, pod IPs change on every restart — Services give a fixed, reliable address.

ArgoCD installs several Services in the `argocd` namespace:

```bash
kubectl get svc -n argocd
```

| Service Name | Type | Port(s) | Purpose |
|---|---|---|---|
| `argocd-server` | ClusterIP | 80, 443 | Exposes the Web UI and API inside the cluster |
| `argocd-repo-server` | ClusterIP | 8081 | Internal — used by argocd-server to call the repo server |
| `argocd-dex-server` | ClusterIP | 5556, 5557 | Internal — SSO/OIDC login flow |
| `argocd-redis` | ClusterIP | 6379 | Internal — cache backend |
| `argocd-metrics` | ClusterIP | 8082, 8083 | Prometheus metrics scraping |

> All services are **ClusterIP** by default (only reachable inside the cluster). You use `port-forward` to reach them locally.

**Access the ArgoCD UI from your local machine:**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then open: `https://localhost:8080`

**Get the initial admin password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

### Quick Concept Summary

| Term | Simple Explanation |
|---|---|
| **ArgoCD** | A tool that keeps your Kubernetes cluster in sync with a Git repo (GitOps) |
| **ArgoCD Server** | The main pod — provides the web dashboard and API |
| **ArgoCD Components** | All the pods ArgoCD runs: server, repo-server, controller, redis, dex |
| **Service (`svc`)** | A Kubernetes resource that gives pods a stable network address |
| **`argocd-server` svc** | The Service that exposes the ArgoCD UI/API for access |
| **`port-forward svc/argocd-server`** | Tunnels the ArgoCD UI to your local machine so you can open it in a browser |

---

### Why ArgoCD (GitOps) instead of just `helm install`?

| Plain `helm install` | ArgoCD (GitOps) |
|---|---|
| You manually run `helm install` / `helm upgrade` | ArgoCD watches Git and auto-syncs on every commit |
| No audit trail of who deployed what | Full Git history = full audit trail |
| Drift goes undetected (someone edits the cluster manually) | ArgoCD detects drift and can auto-heal |
| No UI to visualise app health | Dashboard shows live status of every resource |
| Easy to forget what version is deployed | Deployed version always matches a Git commit |

---

## Related Repos

- [diffchecker](https://github.com/RD-95/diffchecker) — Flask app source code, Dockerfile, and GitHub Actions CI/CD
