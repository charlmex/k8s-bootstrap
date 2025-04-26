# Kubernetes GitOps Bootstrap


# Kubernetes Project Documentation

## Overview

This project sets up a Kubernetes environment with ArgoCD for Continuous Delivery (CD). It includes configurations for deploying and managing multiple applications such as Prometheus, Grafana, Traefik, and more, within a Kubernetes cluster. The setup uses Helm charts, Kubernetes resources, and ArgoCD to manage deployments, monitor metrics, and configure traffic routing via Ingress.

## Prerequisites

Before getting started, ensure you have the following:

- **Minikube**: For setting up the Kubernetes cluster locally.
- **Helm**: For deploying Helm charts.
- **kubectl**: For managing Kubernetes resources.
- **ArgoCD**: For continuous delivery and application lifecycle management.
- **Docker**: If you're using Docker as a driver in Minikube.

## Setup Instructions

### 1. Start Minikube

To start a local Kubernetes cluster using Minikube, run:

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

This will create a Kubernetes cluster with 2 CPUs and 4GB of RAM using Docker as the driver.

### 2. Install ArgoCD

Install ArgoCD in your Kubernetes cluster to manage applications:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Expose ArgoCD API Server

Forward the ArgoCD API Server port to access it locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Once done, you can access ArgoCD UI at `https://localhost:8080`.

### 4. Log in to ArgoCD

To log in to the ArgoCD web interface, get the initial admin password:

```bash
kubectl -n argocd admin initial-password
```

Use this password along with the `admin` username to log in.

### 5. Deploy Applications with ArgoCD

Create an ArgoCD Application manifest to deploy your Kubernetes applications:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/charlmex/k8s-bootstrap.git
    targetRevision: master
    path: clusters/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
```

Apply the Application manifest:

```bash
kubectl apply -f my-app.yaml
```

This will trigger ArgoCD to deploy the resources from the Git repository into your cluster.

### 6. Set Resource Limits for ArgoCD

You can optimize the resource usage of ArgoCD components by patching the deployments:

```bash
kubectl patch deployment -n argocd argocd-repo-server   --patch '{"spec": {"template": {"spec": {"containers": [{"name": "argocd-repo-server", "resources": {"limits": {"cpu": "1", "memory": "1Gi"}, "requests": {"cpu": "500m", "memory": "512Mi"}}}]}}}}'

kubectl patch deployment -n argocd argocd-server   --patch '{"spec": {"template": {"spec": {"containers": [{"name": "argocd-server", "resources": {"limits": {"cpu": "800m", "memory": "512Mi"}, "requests": {"cpu": "300m", "memory": "256Mi"}}}]}}}}'
```

### 7. Deploy Prometheus, Grafana, Traefik, and Other Apps

You can deploy various monitoring tools like **Prometheus** and **Grafana** by using Helm charts. To deploy these tools, use:

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring

# Install Grafana
helm install grafana grafana/grafana --namespace monitoring

# Install Traefik
helm install traefik traefik/traefik --namespace traefik
```

### 8. Port Forward for Grafana, Prometheus, and Traefik

To access the services locally, use `kubectl port-forward`:

```bash
# Grafana
kubectl port-forward svc/grafana -n monitoring 3000:80

# Prometheus
kubectl port-forward svc/prometheus-server -n monitoring 9090:80

# Traefik
kubectl port-forward svc/traefik -n traefik 9000:9000
```

You can now access:

- Grafana: `http://localhost:3000`
- Prometheus: `http://localhost:9090`
- Traefik: `http://localhost:9000`

### 9. Troubleshooting

- If `kubectl top` commands show "Metrics API not available", ensure that **metrics-server** is correctly installed and patched:
  
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
  ```

- If you face issues with services not being found by Traefik (e.g., `error="kubernetes service not found"`), ensure the services are correctly deployed and accessible in the specified namespaces.

## Conclusion

This setup provides a comprehensive way to manage your Kubernetes resources using **ArgoCD** and **Helm charts**. It also includes **monitoring** with **Prometheus** and **Grafana**, and **Ingress management** via **Traefik**. With these tools, you can automate deployments, monitor resources, and expose services in a secure manner.

---


