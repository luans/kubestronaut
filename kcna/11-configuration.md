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

> **Exam tip:** When mounted as a volume, ConfigMap updates are eventually reflected in the mounted files (after a short delay). Environment variable injections are **not** updated — the Pod must be restarted.

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
