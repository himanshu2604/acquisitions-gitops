<div align="center">

# 🔄 Acquisitions GitOps

### Kubernetes manifests and Helm values for the Acquisitions API  
Managed by **ArgoCD** — Git is the only source of truth

<br/>

![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)

<br/>

> ⚠️ **This repo contains no application code.**  
> It contains only Kubernetes YAML manifests and Helm values.  
> Application code lives in [acquisitions-api](https://github.com/himanshu2604/acquisitions-api).

</div>

---

## 📌 Table of Contents

- [Why a Separate Repo?](#-why-a-separate-repo)
- [How It Works](-#-how-it-works)
- [Repository Structure](#-repository-structure)
- [ArgoCD Applications](#-argocd-applications)
- [App Manifests](#-app-manifests)
- [Monitoring Stack](#-monitoring-stack)
- [Prometheus + Grafana Monitoring Diagram](-#-Prometheus-+-Grafana-Monitoring-Diagram)
- [Setting Up from Scratch](#-setting-up-from-scratch)
- [How Deployments Happen](#-how-deployments-happen)
- [Drift Detection](-#-drift-detection)

---

## 🤔 Why a Separate Repo?

This is the **two-repo GitOps pattern**. It is the standard approach used in production GitOps setups, and the reason it exists is important to understand.

**The problem with putting manifests in the app repo:**

If your Kubernetes manifests live in the same repo as your code, your CI pipeline ends up running `kubectl apply` directly. This means:

- The cluster state is controlled by a CI runner — an ephemeral, stateless process
- If someone manually changes something in the cluster, there's no automatic correction
- You can't tell the difference between a code change and an infrastructure change
- Rolling back infrastructure requires rolling back code too

**The GitOps solution:**

By keeping manifests in a separate repo, ArgoCD continuously watches **this repo** and ensures the cluster always reflects what is committed here. The CI pipeline in `acquisitions-api` never touches the cluster directly — it only updates the image tag in this repo, and ArgoCD handles the actual deployment.

| | Traditional CI/CD | GitOps (this approach) |
|---|---|---|
| Who deploys? | CI runner (`kubectl apply`) | ArgoCD (continuous reconciliation) |
| Source of truth | CI pipeline | Git repository (this repo) |
| Manual cluster changes | Persist silently | Auto-reverted within minutes |
| Audit trail | CI logs | Git commit history |
| Rollback | Re-run pipeline | `git revert` + push |

---

## ⚙️ How It Works

<img width="2012" height="863" alt="diagram-export-4-20-2026-5_52_48-PM" src="https://github.com/user-attachments/assets/1f052780-8d89-418b-9bc4-2f72496a959e" />


---

## 📁 Repository Structure

```
acquisitions-gitops/
│
├── app/                          # Acquisitions API — Kubernetes manifests
│   ├── namespace.yaml            # Creates the "acquisitions" namespace
│   ├── configmap.yaml            # Non-sensitive env vars (NODE_ENV, PORT)
│   ├── secret.yaml               # Sensitive values (DATABASE_URL, JWT_SECRET)
│   ├── deployment.yaml           # 3 replicas with rolling update strategy
│   └── service.yaml              # NodePort service exposing the API
│
└── monitoring/                   # Prometheus + Grafana — Helm values
    ├── namespace.yaml            # Creates the "monitoring" namespace
    ├── prometheus-values.yaml    # kube-prometheus-stack Helm configuration
    └── alert-rules.yaml          # PrometheusRule — crash, CPU, error alerts
```

---

## 🔵 ArgoCD Applications

There are **two ArgoCD Applications** watching this repo:

### 1. acquisitions-api
- **Watches:** `app/` folder in this repo
- **Deploys to:** `acquisitions` namespace
- **Sync policy:** Automatic + SelfHeal
- **What it manages:** The API deployment, service, configmap, secret

### 2. monitoring
- **Watches:** `monitoring/` folder in this repo for Helm values
- **Helm chart:** `kube-prometheus-stack` from prometheus-community
- **Deploys to:** `monitoring` namespace
- **Sync policy:** Automatic + SelfHeal
- **What it manages:** Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics

Both applications have `selfHeal: true`, which means ArgoCD will automatically revert any manual changes made to the cluster that deviate from what is defined in this repo.

---

## 📦 App Manifests

### deployment.yaml

Key production settings used:

```yaml
replicas: 3                  # Always 3 replicas running

strategy:
  type: RollingUpdate
  maxSurge: 1               # Spin up 1 extra pod before killing old ones
  maxUnavailable: 0         # Never drop below 3 during an update

livenessProbe:              # Restarts the pod if it becomes unresponsive
  httpGet:
    path: /health

readinessProbe:             # Removes pod from load balancer during startup
  httpGet:
    path: /health

resources:
  requests:
    memory: "128Mi"         # Minimum guaranteed to the pod
    cpu: "100m"
  limits:
    memory: "256Mi"         # Maximum the pod can use
    cpu: "500m"
```

> The image tag in `deployment.yaml` is automatically updated by the CI  
> pipeline in `acquisitions-api` on every successful push to `main`.  
> It is tagged with the full Git commit SHA for full traceability.

### Namespaces

| Namespace | Contents |
|---|---|
| `acquisitions` | API pods, service, configmap, secret |
| `monitoring` | Prometheus, Grafana, Alertmanager, exporters |
| `argocd` | ArgoCD itself (installed separately) |

---

## 📊 Monitoring Stack

Prometheus and Grafana are deployed via the `kube-prometheus-stack` Helm chart, with custom values defined in `monitoring/prometheus-values.yaml`.

---

## Prometheus + Grafana Monitoring Diagram

<img width="1382" height="807" alt="diagram-export-4-20-2026-5_53_38-PM" src="https://github.com/user-attachments/assets/c92c84dc-6c9c-4677-bc2a-4986f20cc412" />

---

### What gets deployed

| Component | Purpose |
|---|---|
| **Prometheus** | Scrapes metrics from all targets every 15 seconds |
| **Grafana** | Dashboards and visualisation |
| **Alertmanager** | Routes alerts to Slack |
| **node-exporter** | Host-level metrics (CPU, memory, disk per node) |
| **kube-state-metrics** | Kubernetes object metrics (pod health, replica counts) |

### Custom scrape target

Prometheus is configured to scrape the Acquisitions API directly:

```yaml
additionalScrapeConfigs:
  - job_name: 'acquisitions-api'
    static_configs:
      - targets:
          - 'acquisitions-api-service.acquisitions.svc.cluster.local:80'
    metrics_path: '/metrics'
    scrape_interval: 15s
```

### Alert rules (`alert-rules.yaml`)

| Alert Name | Condition | Severity | Action |
|---|---|---|---|
| `AcquisitionsPodCrashLooping` | Pod restarts > 3 in 2 min | Critical | Slack notification |
| `AcquisitionsHighCPU` | CPU > 80% for 5 min | Warning | Slack notification |
| `AcquisitionsHighErrorRate` | 5xx error rate > 5% for 2 min | Critical | Slack notification |

---

## 🚀 Setting Up from Scratch

If you are cloning this repo to set up your own cluster, follow these steps.

### Prerequisites

- A running Kubernetes cluster (Minikube, k3d, or cloud)
- `kubectl` configured to point to your cluster
- `helm` installed
- ArgoCD installed in the `argocd` namespace

### Step 1 — Install ArgoCD (if not already installed)

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s
```

### Step 2 — Create the API secrets

```bash
# Base64 encode your values
echo -n "your_database_url" | base64
echo -n "your_jwt_secret"   | base64
echo -n "your_arcjet_key"   | base64
echo -n "your_cookie_secret" | base64

# Edit app/secret.yaml with your encoded values
# Then apply it directly (secrets are not auto-synced for security)
kubectl apply -f app/secret.yaml
```

### Step 3 — Create ArgoCD Application for the API

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: acquisitions-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/himanshu2604/acquisitions-gitops
    targetRevision: HEAD
    path: app
  destination:
    server: https://kubernetes.default.svc
    namespace: acquisitions
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Step 4 — Create ArgoCD Application for monitoring

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: "58.0.0"
      helm:
        valueFiles:
          - $values/monitoring/prometheus-values.yaml
    - repoURL: https://github.com/himanshu2604/acquisitions-gitops
      targetRevision: HEAD
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF
```

### Step 5 — Verify everything is running

```bash
# Check API pods
kubectl get pods -n acquisitions
# Expected: 3 pods in Running state

# Check monitoring pods
kubectl get pods -n monitoring
# Expected: prometheus, grafana, alertmanager, node-exporter, kube-state-metrics

# Check ArgoCD applications
kubectl get applications -n argocd
# Expected: acquisitions-api   Synced   Healthy
#           monitoring          Synced   Healthy
```

### Step 6 — Access the dashboards

```bash
# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana \
  -n monitoring 3001:80
# Open: http://localhost:3001  (admin / admin123change)

# ArgoCD UI
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
# Open: https://localhost:8080

# Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus \
  -n monitoring 9090:9090
# Open: http://localhost:9090
```

---

## 🔄 How Deployments Happen

You **never need to touch this repo manually** for normal deployments.

When a developer pushes code to `acquisitions-api`:

```
1. GitHub Actions runs the CI pipeline
2. Lint, test, Trivy scan, SonarCloud all pass
3. New Docker image pushed to Docker Hub with commit SHA tag
4. CI bot clones this repo
5. Runs: sed -i "s|image: .*/acquisitions-api:.*|image: user/acquisitions-api:NEW_SHA|" app/deployment.yaml
6. Commits: "ci: bump image to abc1234"
7. Pushes to this repo
8. ArgoCD detects the new commit within ~3 minutes
9. ArgoCD applies the updated deployment.yaml to the cluster
10. Kubernetes performs a rolling update — zero downtime
```

**To manually trigger a sync:**

```bash
# Using ArgoCD CLI
argocd app sync acquisitions-api

# Or in the ArgoCD UI — click "Sync" on the application
```

**To roll back to a previous version:**

```bash
# Find the commit you want to go back to
git log --oneline

# Revert the image tag commit
git revert <commit-hash>
git push

# ArgoCD will automatically deploy the reverted version
```

---

## 🛡️ Drift Detection

Both ArgoCD applications have `selfHeal: true`.

This means if anyone manually changes the cluster state — for example by running `kubectl scale`, `kubectl edit`, or `kubectl delete` — ArgoCD will:

1. Detect the divergence between Git (desired state) and the cluster (actual state)
2. Mark the application as `OutOfSync`
3. Automatically apply the Git state back to the cluster within ~3 minutes
4. Mark the application as `Synced` again

**This was tested by:**

```bash
# Manually scaling down to simulate an accident
kubectl scale deployment acquisitions-api \
  --replicas=1 -n acquisitions

# ArgoCD detected drift and restored 3 replicas automatically
# Time to self-heal: ~3 minutes
# Human intervention required: zero
```

This is why Git is the source of truth, not someone's terminal.

---

## 📄 License

This project is for educational and portfolio purposes.

---

<div align="center">

**🔗 Application repo:** [acquisitions-api](https://github.com/himanshu2604/acquisitions-api) — Node.js API + CI pipeline

*Built by Himanshu as a DevOps portfolio project*

</div>
