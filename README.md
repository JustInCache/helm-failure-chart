<p align="center">
  <img src="https://img.shields.io/badge/Helm-v3-blue?logo=helm&logoColor=white" alt="Helm v3"/>
  <img src="https://img.shields.io/badge/Kubernetes-1.24+-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes"/>
  <img src="https://img.shields.io/badge/License-MIT-green" alt="MIT License"/>
  <img src="https://img.shields.io/github/stars/JustInCache/helm-failure-chart?style=social" alt="GitHub Stars"/>
</p>

<h1 align="center">💥 helm-failure-chart</h1>

<p align="center">
  <strong>A deliberately broken Helm chart with 10+ real-world Kubernetes failure scenarios.</strong><br/>
  Deploy. Break. Diagnose. Fix. Repeat.
</p>

<p align="center">
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-failure-scenarios">Scenarios</a> •
  <a href="#-usage-patterns">Usage</a> •
  <a href="#%EF%B8%8F-customization">Customize</a> •
  <a href="#-contributing">Contribute</a>
</p>

---

## 🤔 What Is This?

Ever wanted a **safe playground** of Kubernetes failures you can deploy on demand? This Helm chart ships with 10+ intentionally broken resources — each producing a different, real-world K8s failure.

Perfect for:

- 🤖 **AI Agent Demos** — Let your AI bot (n8n + Slack + EKS MCP) diagnose and fix live cluster issues
- 🎓 **SRE Training & Interviews** — Test troubleshooting skills on realistic failures
- 📊 **Monitoring Validation** — Verify your Prometheus/Grafana/PagerDuty alerts actually fire
- 🧪 **Chaos Engineering** — Controlled failure injection without the unpredictability

> 🔒 Everything is **namespace-scoped** and isolated — safe to deploy alongside production workloads.

---

## 🚀 Quick Start

```bash
# Clone the repo
git clone https://github.com/JustInCache/helm-failure-chart.git
cd helm-failure-chart

# Deploy to any namespace you like
helm install failure-demo . --namespace failure-demo --create-namespace

# 🍿 Sit back and watch the chaos unfold
kubectl get pods -n failure-demo -w
```

---

## 📦 What Gets Deployed

| Component | Image | Replicas | Purpose |
|-----------|-------|----------|---------|
| 🌐 **Frontend** | `nginx` | 2 | Web UI (static files) |
| ⚙️ **Backend** | `node:18-alpine` | 2 | REST API |
| 🔧 **Worker** | `python:3.11-slim` | 1 | Background job processor |
| 🗄️ **Redis** | `redis:7-alpine` | 1 | Cache / message queue |

**Also creates:** ConfigMap, Secret, ServiceAccount, Role, RoleBinding, Ingress, HPA, PVC, NetworkPolicy

> 💡 **Resource footprint:** ~850m CPU, ~900Mi memory in requests (most pods fail before consuming anything).

---

## 🔥 Want Most Common Scenarios ONLY?

If you want the lite chart covers the **5 most common** Kubernetes failures. You can switch to below: 

👉 **[helm-failure-chart-lite](https://github.com/JustInCache/helm-failure-chart-lite)** — the lite version

| | Lite | Full (this repo) |
|---|---|---|
| Scenarios | 5 | 10+ |
| Components | 3 (Frontend, Backend, Worker) | 4 (+ Redis) |
| Ingress / HPA / PVC / NetworkPolicy | ❌ | ✅ |
| Resource footprint | ~200m CPU, ~256Mi | ~850m CPU, ~900Mi |
| Best for | Quick demos, interviews | Comprehensive training, deep dives |

---

## 💣 Failure Scenarios

### 1️⃣ ImagePullBackOff

| | |
|---|---|
| 📍 **Component** | Frontend Deployment |
| 🐛 **Root Cause** | Image tag `nginx:v99.99.99` does not exist |
| 👀 **What You See** | Pod stuck in `ImagePullBackOff` / `ErrImagePull` |
| ✅ **How to Fix** | Change `frontend.image.tag` to a valid tag like `1.25-alpine` |

```bash
kubectl describe pod -l app=frontend -n failure-demo | grep -A5 "Events"
```

---

### 2️⃣ CrashLoopBackOff

| | |
|---|---|
| 📍 **Component** | Backend Deployment |
| 🐛 **Root Cause** | Liveness probe targets port `8080` but container only has port `3000`. Container runs a simple `setTimeout`, not an HTTP server. |
| 👀 **What You See** | Pod enters `CrashLoopBackOff` after repeated probe failures |
| ✅ **How to Fix** | Change probe ports to `3000` in `values.yaml`, or remove the HTTP probes |

```bash
kubectl logs -l app=backend -n failure-demo --previous
```

---

### 3️⃣ Service Has 0 Endpoints

| | |
|---|---|
| 📍 **Component** | Backend Service |
| 🐛 **Root Cause** | Service selector is `app: backend-api` but pods are labeled `app: backend` |
| 👀 **What You See** | `kubectl get endpoints` shows 0 endpoints — traffic never reaches pods |
| ✅ **How to Fix** | Change the service selector to `app: backend` |

```bash
kubectl get endpoints -n failure-demo
```

---

### 4️⃣ CreateContainerConfigError

| | |
|---|---|
| 📍 **Component** | Worker Deployment |
| 🐛 **Root Cause** | Env vars reference ConfigMap keys `DATABASE_HOST` / `DATABASE_PORT`, but the ConfigMap defines `DB_HOST` / `DB_PORT` |
| 👀 **What You See** | Pod stuck in `CreateContainerConfigError` |
| ✅ **How to Fix** | Align the key names — update the ConfigMap or the deployment env refs |

```bash
kubectl describe pod -l app=worker -n failure-demo | grep -A3 "Warning"
```

---

### 5️⃣ OOMKilled

| | |
|---|---|
| 📍 **Component** | Redis Deployment |
| 🐛 **Root Cause** | Memory limit is `5Mi` — Redis needs ~30-50Mi minimum to start |
| 👀 **What You See** | Pod gets `OOMKilled` immediately, restarts in a loop |
| ✅ **How to Fix** | Increase `redis.resources.limits.memory` to at least `128Mi` |

```bash
kubectl get pod -l app=redis -n failure-demo -o jsonpath='{.items[0].status.containerStatuses[0].lastState}'
```

---

### 6️⃣ Ingress Misconfiguration

| | |
|---|---|
| 📍 **Component** | Ingress |
| 🐛 **Root Cause** | Frontend path routes to service `xxx-web` (should be `xxx-frontend`). API path uses port `8080` (should be `3000`). |
| 👀 **What You See** | 503 errors, no healthy targets |
| ✅ **How to Fix** | Correct the service name and port in `ingress.yaml` |

```bash
kubectl describe ingress -n failure-demo
```

---

### 7️⃣ HPA Target Mismatch

| | |
|---|---|
| 📍 **Component** | HorizontalPodAutoscaler |
| 🐛 **Root Cause** | HPA targets deployment `xxx-backend-api` but actual name is `xxx-backend` |
| 👀 **What You See** | HPA shows `<unknown>` for current metrics |
| ✅ **How to Fix** | Fix `scaleTargetRef.name` in `hpa.yaml` to match the actual deployment |

```bash
kubectl get hpa -n failure-demo
```

---

### 8️⃣ PVC Stuck in Pending

| | |
|---|---|
| 📍 **Component** | PersistentVolumeClaim |
| 🐛 **Root Cause** | References StorageClass `gp3-encrypted-premium` which doesn't exist |
| 👀 **What You See** | PVC stays `Pending`, worker pod can't mount volume |
| ✅ **How to Fix** | Change `persistence.storageClassName` to an existing class (`gp2`, `gp3`, `standard`) |

```bash
kubectl get pvc -n failure-demo
kubectl get storageclass
```

---

### 9️⃣ RBAC Permission Denied

| | |
|---|---|
| 📍 **Component** | Role |
| 🐛 **Root Cause** | Role lists `deployments` under apiGroup `""` (core) — they belong to `"apps"` |
| 👀 **What You See** | `403 Forbidden` when the ServiceAccount tries to access deployments |
| ✅ **How to Fix** | Change `apiGroups: [""]` to `apiGroups: ["apps"]` for the deployments rule |

```bash
kubectl auth can-i list deployments \
  --as=system:serviceaccount:failure-demo:failure-demo-helm-failure-chart-sa \
  -n failure-demo
```

---

### 🔟 NetworkPolicy Blocks Database

| | |
|---|---|
| 📍 **Component** | NetworkPolicy |
| 🐛 **Root Cause** | Backend egress only allows Redis. No rule for PostgreSQL on port 5432. |
| 👀 **What You See** | Backend → database connections time out |
| ✅ **How to Fix** | Add an egress rule for the database service on port `5432` |

---

### 🎁 Bonus — Secret Key Mismatch

| | |
|---|---|
| 📍 **Component** | Backend Deployment + Secret |
| 🐛 **Root Cause** | Deployment references `DATABASE_USERNAME` from the secret, but the secret key is `DB_USERNAME` |
| 👀 **What You See** | `CreateContainerConfigError` on backend pods |
| ✅ **How to Fix** | Align the key name in either the secret or the deployment |

### 🎁 Bonus — Invalid IAM Role ARN

| | |
|---|---|
| 📍 **Component** | ServiceAccount |
| 🐛 **Root Cause** | IRSA annotation points to `arn:aws:iam::123456789012:role/app-nonexistent-role` |
| 👀 **What You See** | AWS API calls from pods fail with auth errors |
| ✅ **How to Fix** | Update the ARN to a valid IAM role, or remove the annotation |

---

## 🎯 Usage Patterns

### 🤖 AI Agent Demo (n8n + Slack + EKS MCP)

1. Deploy the chart to your EKS cluster
2. Ask in Slack: *"What's wrong with pods in the failure-demo namespace?"*
3. AI agent uses EKS MCP to inspect pods, events, services, etc.
4. Agent diagnoses failures and suggests fixes
5. Apply fixes iteratively and re-ask to validate

### 🎓 SRE Training / Interviews

Deploy the chart and ask candidates to:
- 🔍 Identify all failing resources and their root causes
- 🛠️ Propose fixes without looking at the source
- 📋 Prioritize which failures to fix first

### 📊 Monitoring & Alerting Validation

Use the chart to verify that your Prometheus/Grafana/PagerDuty pipeline correctly detects:
- `CrashLoopBackOff` and `OOMKilled` pod states
- Deployments with 0 available replicas
- PVCs stuck in Pending
- Services with 0 endpoints

---

## ⚙️ Customization

Override any value at install time:

```bash
# Disable ingress (e.g., if you use Kong and don't want ALB conflicts)
helm install failure-demo . -n failure-demo --create-namespace \
  --set ingress.enabled=false

# Disable persistence (skip PVC scenario)
helm install failure-demo . -n failure-demo --create-namespace \
  --set persistence.enabled=false

# Fix the frontend image to isolate other scenarios
helm install failure-demo . -n failure-demo --create-namespace \
  --set frontend.image.tag=1.25-alpine
```

---

## 🛡️ Safety

Everything is **namespace-scoped**. Nothing touches other namespaces or creates cluster-wide resources.

| Check | Status |
|---|---|
| 🔐 Cluster-scoped RBAC (ClusterRole) | ✅ None |
| 🌐 LoadBalancer / NodePort services | ✅ None — all ClusterIP |
| 🔗 Cross-namespace NetworkPolicy | ✅ None |
| 📦 CRDs / Webhooks | ✅ None |
| 🚪 Ingress controller side effects | ⚠️ Only if ALB controller exists — disable with `ingress.enabled=false` |

---

## 🧹 Cleanup

```bash
helm uninstall failure-demo -n failure-demo
kubectl delete namespace failure-demo
```

---

## 📁 Chart Structure

```
helm-failure-chart/
├── Chart.yaml
├── values.yaml
├── README.md
└── templates/
    ├── _helpers.tpl
    ├── backend-deployment.yaml    # Scenarios 2, Bonus (secret mismatch)
    ├── backend-service.yaml       # Scenario 3
    ├── configmap.yaml             # Scenario 4 (key mismatch source)
    ├── frontend-deployment.yaml   # Scenario 1
    ├── frontend-service.yaml
    ├── hpa.yaml                   # Scenario 7
    ├── ingress.yaml               # Scenario 6
    ├── network-policy.yaml        # Scenario 10
    ├── pvc.yaml                   # Scenario 8
    ├── rbac.yaml                  # Scenario 9
    ├── redis-deployment.yaml      # Scenario 5
    ├── redis-service.yaml
    ├── secret.yaml                # Bonus (key mismatch source)
    ├── serviceaccount.yaml        # Bonus (invalid IAM ARN)
    └── worker-deployment.yaml     # Scenario 4
```

---

## 🤝 Contributing

Contributions are welcome! Here's how you can help:

- 🐛 **Add new failure scenarios** — open a PR with a new template and update the README
- 📝 **Improve documentation** — typos, better explanations, diagrams
- 🧪 **Test on different clusters** — EKS, GKE, AKS, minikube, kind — report what works
- 💡 **Suggest ideas** — open an issue with your use case

### How to contribute

1. Fork the repo
2. Create a feature branch (`git checkout -b feat/new-scenario`)
3. Commit your changes (`git commit -m "Add new failure scenario"`)
4. Push to the branch (`git push origin feat/new-scenario`)
5. Open a Pull Request

---

## ⭐ Star This Repo

If this chart helped you demo, learn, or break things in a fun way — **give it a star!** ⭐

It helps others discover this project and motivates continued development.

[![GitHub stars](https://img.shields.io/github/stars/JustInCache/helm-failure-chart?style=for-the-badge&logo=github&label=Star%20This%20Repo)](https://github.com/JustInCache/helm-failure-chart)

---

## ☕ Buy Me a Coffee

If this project saved you time or sparked an idea, consider buying me a coffee!

<a href="https://buymeacoffee.com/connectankush">
  <img src="assets/bmc.png" alt="Buy Me a Coffee" width="200"/>
</a>

Your support keeps this project maintained and growing 🙏

<a href="https://buymeacoffee.com/connectankush">
  <img src="assets/bmc-qr-code.png" alt="Buy Me a Coffee QR Code" width="150"/>
</a>

**[☕ buymeacoffee.com/connectankush](https://buymeacoffee.com/connectankush)**

---

## 📄 License

MIT — do whatever you want with it. Break things responsibly. 💥
