# GitOps and CI/CD

## CI/CD Fundamentals

### Continuous Integration (CI)
The practice of automatically building and testing code changes as soon as they are committed. Goal: detect integration problems early.

**Typical CI pipeline:**
```
Code push → Build → Unit tests → Static analysis → Container image build → Push to registry
```

### Continuous Delivery (CD)
Automatically deploying tested artifacts to one or more environments. The deployment may be triggered manually (delivery) or automatically (deployment).

**CD pipeline in Kubernetes context:**
```
Image pushed to registry → Update manifest → kubectl apply / Helm upgrade → Kubernetes
```

### Common CI/CD Tools
- **Jenkins** — widely used; highly configurable
- **GitHub Actions** — integrated with GitHub repositories
- **GitLab CI** — integrated with GitLab
- **Tekton** — Kubernetes-native CI/CD; CNCF project
- **Argo Workflows** — Kubernetes-native workflow engine

---

## GitOps

**GitOps** is an operational framework where **Git is the single source of truth** for the desired state of infrastructure and applications.

**Core principles:**
1. **Declarative** — the entire system is described declaratively (YAML manifests)
2. **Versioned and immutable** — desired state is stored in Git; changes are tracked
3. **Pulled automatically** — software agents (operators) automatically apply the desired state
4. **Continuously reconciled** — agents detect and correct drift between desired and actual state

**Push-based vs Pull-based:**

| Model | How it works | Example |
|---|---|---|
| **Push** | CI pipeline pushes changes directly to the cluster via `kubectl` or API | Jenkins + kubectl |
| **Pull** | An agent inside the cluster watches Git and pulls changes | Argo CD, Flux |

> **Exam tip:** GitOps favors the **pull model** — the cluster agent pulls from Git, so the cluster initiates the connection. This is more secure (no external write access to the cluster needed) and provides continuous reconciliation.

---

## Argo CD

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes. It is a **CNCF graduated project**.

**How it works:**
1. You push manifest changes to Git
2. Argo CD detects the change (polls or webhook)
3. Argo CD compares desired state (Git) with actual state (cluster)
4. Argo CD syncs the cluster to match Git

```
Git repo (desired state) ← watches ← Argo CD → applies → Kubernetes cluster
```

**Key concepts:**
- **Application** — Argo CD resource linking a Git repo + path to a Kubernetes cluster + namespace
- **Sync** — the action of making the cluster match Git
- **Health status** — Argo CD evaluates if resources are actually healthy, not just applied
- **App of Apps** — pattern for managing multiple applications declaratively

---

## Flux

**Flux** is another CNCF GitOps tool. It is a **CNCF graduated project** and takes a more modular, controller-based approach than Argo CD.

**Flux controllers:**
- `source-controller` — watches Git repos, Helm repos, OCI registries
- `kustomize-controller` — applies Kustomize manifests
- `helm-controller` — manages Helm releases
- `notification-controller` — sends alerts and receives webhooks
- `image-automation-controller` — updates Git with new image tags

---

## Helm

**Helm** is the package manager for Kubernetes. It uses **Charts** — packages of pre-configured Kubernetes resources.

**Key concepts:**

| Concept | Description |
|---|---|
| **Chart** | A collection of YAML templates + default values |
| **Release** | A deployed instance of a Chart in a cluster |
| **Repository** | A place where Charts are stored and shared |
| **Values** | Configuration overrides for a Chart (`values.yaml`) |

```bash
# Add a chart repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a chart
helm install my-release bitnami/nginx

# Install with custom values
helm install my-release bitnami/nginx -f custom-values.yaml
helm install my-release bitnami/nginx --set replicaCount=3

# Upgrade a release
helm upgrade my-release bitnami/nginx --set replicaCount=5

# Rollback
helm rollback my-release 1

# List releases
helm list

# Uninstall
helm uninstall my-release
```

> **Exam tip:** Helm is not a GitOps tool by itself — it's a packaging/templating tool. GitOps tools like Argo CD and Flux have native Helm support and can manage Helm releases declaratively.

---

## Kustomize

**Kustomize** is a Kubernetes-native configuration management tool (built into `kubectl`). It lets you customize raw YAML manifests without templating — using **overlays** on top of a base configuration.

**Structure:**
```
base/
  deployment.yaml
  service.yaml
  kustomization.yaml

overlays/
  production/
    kustomization.yaml   # references base + adds production patches
  staging/
    kustomization.yaml   # references base + adds staging patches
```

```bash
kubectl apply -k overlays/production/
kubectl kustomize overlays/production/   # preview output
```

> **Exam tip:** Kustomize is built into `kubectl` (no separate install needed). Argo CD and Flux both natively support Kustomize. Unlike Helm, Kustomize doesn't use a templating language — it patches existing YAML.

---

## GitOps Summary

| Tool | Type | CNCF status |
|---|---|---|
| Argo CD | GitOps (pull) | Graduated |
| Flux | GitOps (pull) | Graduated |
| Tekton | CI/CD pipelines | Incubating |
| Helm | Package manager | CNCF (graduated) |
| Kustomize | Config management | Built into kubectl |
