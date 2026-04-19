# Kubernetes + MLflow Commands (Local Setup)

This is a command-first record of how to create a local Kubernetes cluster and install MLflow using Helm.

## 1) Prerequisites

```bash
# Install kind (if missing)
brew install kind

# Verify tools
kind --version
kubectl version --client
helm version
```

## 2) Create a Local Kubernetes Cluster (kind)

```bash
kind create cluster --name basic-mlflow-cluster
```

Verify cluster:

```bash
kind get clusters
kubectl cluster-info --context kind-basic-mlflow-cluster
kubectl get nodes
kubectl get pods -A
```

## 3) Add Helm Repo for MLflow Charts

```bash
helm repo add community-charts https://community-charts.github.io/helm-charts
helm repo update
```

If `helm repo update` shows a 404 for an old repo, clean it:

```bash
helm repo list
helm repo remove kubernetes-dashboard
helm repo update
```

## 4) Install MLflow on Kubernetes

```bash
helm install mlflow-community community-charts/mlflow
```

Verify deployment:

```bash
helm list -n default
kubectl get pods -n default
kubectl get svc -n default
```

Expected state:

- Helm release status: `deployed`
- MLflow pod: `Running`
- Service: `mlflow-community` (ClusterIP)

## 5) Access MLflow UI from Local Machine

### Recommended: Port-forward service

```bash
kubectl -n default port-forward svc/mlflow-community 7001:80 --address 0.0.0.0
```

Open:

```text
http://127.0.0.1:7001
```

### Alternative: Port-forward pod directly

```bash
kubectl port-forward pod/mlflow-community-5f8f74d69b-rd5xx 7001:5000 --address 0.0.0.0
```

## 6) Troubleshooting Commands

### Port already in use

```bash
lsof -nP -iTCP:7000 -sTCP:LISTEN
```

Use another port (example 7001):

```bash
kubectl -n default port-forward svc/mlflow-community 7001:80 --address 0.0.0.0
```

### Check cluster and workloads quickly

```bash
kubectl get pods -A
kubectl get svc -A
kubectl describe pod -n default mlflow-community-5f8f74d69b-rd5xx
kubectl logs -n default mlflow-community-5f8f74d69b-rd5xx
```

## 7) Uninstall / Cleanup

```bash
helm uninstall mlflow-community -n default
kind delete cluster --name basic-mlflow-cluster
```

## 8) Useful References

- https://community-charts.github.io/docs/charts/mlflow/basic-installation
- https://community-charts.github.io/docs/charts/mlflow/postgresql-backend-installation
