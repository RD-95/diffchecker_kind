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

## Related Repos

- [diffchecker](https://github.com/RD-95/diffchecker) — Flask app source code, Dockerfile, and GitHub Actions CI/CD
