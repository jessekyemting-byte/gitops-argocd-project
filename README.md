# GitOps Deployment with Argo CD and Kubernetes

An automated GitOps continuous delivery pipeline that deploys a Flask web application to a Kubernetes cluster using **Argo CD**. The project demonstrates automated synchronization, continuous state management, and self-healing (drift correction).

---

## Table of Contents

- [Architecture & Stack](#architecture--stack)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [GitOps Configuration](#gitops-configuration)
- [Self-Healing & Drift Correction Test](#self-healing--drift-correction-test)
- [Verification](#verification)

---

## Architecture & Stack

| Component        | Technology                                  |
| ---------------- | ------------------------------------------- |
| Application      | Python Flask web server                     |
| Containerization | Docker (`jessekyemting/gitops-demo:1.0`)    |
| Orchestration    | Kubernetes (Minikube / WSL2)                |
| GitOps Engine    | Argo CD                                     |
| Target Namespace | `gitops-demo`                               |

**How it works:** Git is the single source of truth. Argo CD continuously compares the manifests in the `k8s/` directory of this repository with the live state of the cluster, and reconciles any difference automatically.

---

## Repository Structure

```text
.
├── k8s/
│   ├── deployment.yaml   # Kubernetes Deployment manifest (2 replicas)
│   └── service.yaml      # ClusterIP Service manifest (port 5000)
├── argocd-app.yaml       # Argo CD Application custom resource
├── app.py                # Flask application source code
└── README.md             # Project documentation
```

---

## Prerequisites

- Docker
- Minikube (or any Kubernetes cluster) and `kubectl`
- Argo CD installed in the cluster (see below)

---

## Getting Started

**1. Start the cluster**

```bash
minikube start
```

**2. Install Argo CD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**3. Register the application with Argo CD**

```bash
kubectl apply -f argocd-app.yaml
```

**4. (Optional) Access the Argo CD UI**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open <https://localhost:8080>. The initial admin password can be retrieved with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## GitOps Configuration

The application is declared with an Argo CD `Application` custom resource that has automated sync enabled:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/jessekyemting-byte/gitops-argocd-project.git'
    targetRevision: main
    path: k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: gitops-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Key settings**

| Setting                | Effect                                                                 |
| ---------------------- | ---------------------------------------------------------------------- |
| `automated`            | Changes pushed to Git are applied to the cluster without manual action |
| `prune: true`          | Resources removed from Git are deleted from the cluster                |
| `selfHeal: true`       | Manual changes to the cluster are reverted to match Git                |
| `CreateNamespace=true` | The `gitops-demo` namespace is created automatically                   |

---

## Self-Healing & Drift Correction Test

This test demonstrates GitOps resilience.

**1. Simulate drift.** Manually scale the deployment down in the cluster, bypassing Git:

```bash
kubectl scale deployment gitops-demo --replicas=0 -n gitops-demo
```

**2. Observe automated reconciliation.** Because `selfHeal: true` is configured, Argo CD detects the divergence between the live state (0 replicas) and the desired state in Git (2 replicas), then triggers a sync that restores the missing pods.

```bash
kubectl get pods -n gitops-demo -w
```

---

## Verification

Confirm the application is serving traffic from inside the cluster:

```bash
kubectl exec -n gitops-demo deploy/gitops-demo -- python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:5000').read().decode('utf-8'))"
```

**Expected output:**

```text
GitOps deployment with Argo CD is working!
```