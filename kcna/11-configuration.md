# Configuration

## ConfigMaps

A **ConfigMap** stores non-sensitive configuration data as key-value pairs. It decouples environment-specific configuration from container images, making applications portable.

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config --from-literal=ENV=production --from-literal=LOG_LEVEL=info

# From a file
kubectl create configmap app-config --from-file=config.properties

# From a directory (each file becomes a key)
kubectl create configmap app-config --from-file=./configs/
```

### Using ConfigMaps in Pods

**As environment variables:**

```yaml
spec:
  containers:
  - name: app
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    envFrom:                    # inject all keys as env vars
    - configMapRef:
        name: app-config
```

**As a volume (files):**

```yaml
spec:
  volumes:
  - name: config-vol
    configMap:
      name: app-config
  containers:
  - name: app
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config    # each key becomes a file
```

> **Exam tip:** When mounted as a volume, ConfigMap updates are eventually reflected in the mounted files (after a short delay). Environment variable injections are **not** updated - the Pod must be restarted.

---

## Secrets

A **Secret** stores sensitive data such as passwords, tokens, and certificates. Data is stored **base64-encoded** (not encrypted by default) in etcd.

> **Exam tip:** Base64 is **encoding**, not encryption. Secrets are only as secure as your etcd and RBAC configuration. Enable **encryption at rest** for etcd in production.

### Secret Types

| Type | Use case |
|---|---|
| `Opaque` | Default; arbitrary user-defined key-value data |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key (`tls.crt`, `tls.key`) |
| `kubernetes.io/service-account-token` | ServiceAccount tokens |
| `kubernetes.io/ssh-auth` | SSH credentials |
| `kubernetes.io/basic-auth` | Basic authentication |

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-secret --from-literal=password=s3cr3t

# From files
kubectl create secret generic tls-secret --from-file=tls.crt --from-file=tls.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=user \
  --docker-password=pass
```

### Using Secrets in Pods

```yaml
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    volumeMounts:
    - name: tls
      mountPath: /etc/tls
      readOnly: true
  volumes:
  - name: tls
    secret:
      secretName: tls-secret
  imagePullSecrets:             # for pulling from private registries
  - name: regcred
```

---

## Namespaces

**Namespaces** provide a mechanism for isolating groups of resources within a single cluster. They are used to divide cluster resources between multiple users or teams.

### Default Namespaces

| Namespace | Purpose |
|---|---|
| `default` | Default namespace for resources with no namespace specified |
| `kube-system` | Kubernetes system components (CoreDNS, kube-proxy, etc.) |
| `kube-public` | Publicly readable; used for cluster info (`kubectl cluster-info`) |
| `kube-node-lease` | Node heartbeat lease objects (improves node failure detection) |

### Namespace-scoped vs Cluster-scoped Resources

**Namespaced:** Pods, Deployments, Services, ConfigMaps, Secrets, PVCs, ServiceAccounts, Roles, RoleBindings

**Cluster-scoped:** Nodes, PersistentVolumes, StorageClasses, Namespaces, ClusterRoles, ClusterRoleBindings

```bash
# List all namespaced resource types
kubectl api-resources --namespaced=true

# List all cluster-scoped resource types
kubectl api-resources --namespaced=false
```

### Resource Quotas

**ResourceQuota** limits the total resource consumption in a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "5"
```

### LimitRange

**LimitRange** sets **default** and **maximum** resource values for containers in a namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
```

> **Exam tip:** If a namespace has a LimitRange and a container does not specify requests/limits, the LimitRange **default** values are applied automatically. This prevents containers from running without resource constraints.

---

## Environment Variables

Pods can receive configuration via environment variables from multiple sources:

```yaml
env:
- name: LITERAL_VALUE
  value: "hello"
- name: FROM_CONFIGMAP
  valueFrom:
    configMapKeyRef:
      name: my-config
      key: some-key
- name: FROM_SECRET
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
- name: POD_NAME                    # Downward API
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
```

## Downward API

The **Downward API** allows Pods to access information about themselves (Pod name, namespace, node name, labels, resource limits) without calling the Kubernetes API.

Available as environment variables (`fieldRef`, `resourceFieldRef`) or as volume files.

## Labels and Annotations

Both labels and annotations are **key-value pairs** attached to Kubernetes objects in the `metadata` section. They serve different purposes.

### Labels

Labels are **identifying metadata** used to organize objects and to select groups of them. They are the primary mechanism for Kubernetes internals (Services, Deployments, NetworkPolicies, etc.) to target specific Pods.

```yaml
metadata:
  name: my-pod
  labels:
    app: my-app
    version: v2
    environment: production
    tier: frontend
```

**Label key format:** `[prefix/]name`
- Prefix is optional and must be a valid DNS subdomain (e.g., `app.kubernetes.io/name`)
- Name is required; max 63 characters
- Well-known prefix: `app.kubernetes.io/` - used by Kubernetes tooling and Helm

**Common recommended labels (`app.kubernetes.io/`):**

| Label | Example value | Description |
|---|---|---|
| `app.kubernetes.io/name` | `mysql` | Name of the application |
| `app.kubernetes.io/version` | `5.7.21` | Current version |
| `app.kubernetes.io/component` | `database` | Component within the architecture |
| `app.kubernetes.io/part-of` | `my-app` | Higher-level app this is part of |
| `app.kubernetes.io/managed-by` | `helm` | Tool used to manage the app |

#### Label Selectors

Used by controllers and Services to select which Pods they manage:

**Equality-based:**
```yaml
selector:
  matchLabels:
    app: my-app
    environment: production
```

**Set-based:**
```yaml
selector:
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: tier
    operator: NotIn
    values: [frontend]
  - key: version
    operator: Exists
```

| Operator | Behavior |
|---|---|
| `In` | Key's value must be in the list |
| `NotIn` | Key's value must not be in the list |
| `Exists` | Key must be present (any value) |
| `DoesNotExist` | Key must not be present |

> **Exam tip:** Services use the older `selector:` field (equality-based only). Deployments, Jobs, and NetworkPolicies use `matchLabels` / `matchExpressions` (supports set-based). You cannot change a Deployment's `selector` after creation.

### Annotations

Annotations are **non-identifying metadata** - arbitrary key-value pairs used to attach information to objects for tools, libraries, and humans. They are **not** used for selection.

```yaml
metadata:
  name: my-pod
  annotations:
    description: "Main application pod for the payments service"
    contact: "team-payments@example.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
```

**Common uses:**
- Build/release metadata (Git commit SHA, image tag, CI run ID)
- Configuration for tools and controllers (Ingress annotations, Prometheus scrape config)
- Documentation (description, owner, runbook URL)
- `kubectl apply` stores the last-applied config as an annotation

**Key format:** same as labels - `[prefix/]name`. Annotation values can be **any string**, including JSON or YAML blobs (unlike label values which have character restrictions).

### Labels vs Annotations

| Aspect | Labels | Annotations |
|---|---|---|
| Purpose | Identifying; used for selection | Non-identifying; used for metadata |
| Used by selectors | Yes | No |
| Value restrictions | Max 63 chars, alphanumeric + `-_.` | Any string, no size limit* |
| Queryable with kubectl | Yes (`-l app=my-app`) | No |
| Example use | Group Pods for a Service | Store Prometheus scrape config |

*Annotations with very large values can impact API server performance.

### Useful Commands

```bash
# Filter resources by label
kubectl get pods -l app=my-app
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production,staging)'

# Add / update a label on a running object
kubectl label pod my-pod version=v2
kubectl label pod my-pod version=v3 --overwrite

# Remove a label (trailing dash)
kubectl label pod my-pod version-

# Add / update an annotation
kubectl annotate pod my-pod description="payments service"
kubectl annotate pod my-pod description="updated" --overwrite

# Remove an annotation
kubectl annotate pod my-pod description-

# Show labels in output
kubectl get pods --show-labels
kubectl get pods -L app,environment   # show specific labels as columns
```

---

## Useful Commands

```bash
# ConfigMaps
kubectl get configmap
kubectl describe configmap app-config
kubectl edit configmap app-config

# Secrets
kubectl get secrets
kubectl describe secret db-secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# Namespaces
kubectl get namespaces
kubectl create namespace staging
kubectl config set-context --current --namespace=staging   # switch default namespace

# Resource Quotas
kubectl get resourcequota -n development
kubectl describe resourcequota dev-quota -n development
```
