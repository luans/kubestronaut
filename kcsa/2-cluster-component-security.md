# Day 2 - Cluster Component Security

## kube-apiserver: Hardening and Critical Flags

The kube-apiserver is the heart of the cluster. All communication with Kubernetes goes through it. It is the primary attack surface and the most critical component to harden.

### Authentication and authorization architecture

```
Client Request
      │
      ▼
┌─────────────┐
│ Authentication│  Who are you?
│ (AuthN)      │  (certificates, tokens, OIDC)
└──────┬──────┘
       │ ✓
       ▼
┌─────────────┐
│ Authorization │  Are you allowed to do this?
│ (AuthZ/RBAC) │  (RBAC, ABAC, Webhook)
└──────┬──────┘
       │ ✓
       ▼
┌──────────────────┐
│ Admission Control │  Is this valid/permitted?
│ (Webhooks)        │  (validating/mutating)
└──────┬───────────┘
       │ ✓
       ▼
   etcd (persists)
```

### Critical kube-apiserver flags

**Authentication:**
```bash
# Disable anonymous authentication (CRITICAL)
--anonymous-auth=false

# Enable OIDC for external provider authentication
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=kubernetes
--oidc-username-claim=email
--oidc-groups-claim=groups

# Static token file (AVOID in production)
--token-auth-file=/etc/kubernetes/tokens.csv  # LEGACY, do not use
```

**Authorization:**
```bash
# Use Node + RBAC (recommended default)
--authorization-mode=Node,RBAC

# Node: authorizes kubelets to access resources for their pods
# RBAC: role-based access control
# ABAC: avoid (static file, hard to manage)
# Webhook: delegate to an external service
```

**TLS and encryption:**
```bash
# Server certificates
--tls-cert-file=/etc/kubernetes/pki/apiserver.crt
--tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# CA for verifying clients
--client-ca-file=/etc/kubernetes/pki/ca.crt

# Recommended ciphers (disable weak ones)
--tls-min-version=VersionTLS12
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**Audit logging:**
```bash
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

**Admission controllers:**
```bash
# Enable critical admission controllers
--enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota,LimitRanger

# Disable dangerous admission controllers
--disable-admission-plugins=AlwaysAdmit
```

**Other important flags:**
```bash
# Disable profiling (exposes internal information)
--profiling=false

# Service account key file
--service-account-key-file=/etc/kubernetes/pki/sa.pub

# Restrict kubelet access
--kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
--kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

### Audit Logging Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all operations on secrets at the Request level
  - level: Request
    resources:
    - group: ""
      resources: ["secrets"]

  # Log at Metadata level for pods (does not log body)
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods"]

  # Do not log health checks
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services", "services/status"]

  # Default: log Metadata for everything else
  - level: Metadata
```

**Audit levels:**
- `None`: do not log
- `Metadata`: log only metadata (who, when, what) - no body
- `Request`: log metadata + request body
- `RequestResponse`: log metadata + request body + response body

---

## etcd: Encryption at Rest and in Transit

etcd stores all cluster state: deployments, services, configmaps, **and Secrets in plain text by default**.

### Encryption in Transit

etcd communication uses TLS. Verify it is configured:

```bash
# etcd flags for TLS
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--client-cert-auth=true  # require client certificate

# Peer TLS (communication between etcd cluster members)
--peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
--peer-key-file=/etc/kubernetes/pki/etcd/peer.key
--peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--peer-client-cert-auth=true
```

### Encryption at Rest

By default, Secrets are stored in plaintext in etcd. To encrypt them:

**Step 1: Create the EncryptionConfiguration**

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps  # optional, encrypt as well
    providers:
      # First provider is used to encrypt new data
      - aescbc:
          keys:
            - name: key1
              # Generate with: head -c 32 /dev/urandom | base64
              secret: <BASE64_ENCODED_32_BYTE_KEY>
      # identity = no encryption (for reading old data)
      - identity: {}
```

**Step 2: Configure kube-apiserver**

```bash
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

**Step 3: Verify it works**

```bash
# Create a secret
kubectl create secret generic test-secret --from-literal=key=value

# Check directly in etcd (should appear encrypted)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/test-secret | hexdump -C

# If encrypted, you will see k8s:enc:aescbc:v1:key1 at the beginning
```

**Step 4: Re-encrypt existing Secrets**

```bash
# Force re-write of all secrets to apply encryption
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

**Encryption providers (in order of preference):**

| Provider | Strength | Notes |
|---|---|---|
| `aescbc` | AES-CBC 256-bit | Recommended, widely supported |
| `aesgcm` | AES-GCM 256-bit | More performant, nonce reuse risk |
| `secretbox` | XSalsa20 + Poly1305 | Modern, good choice |
| `kms` | Delegated to KMS | Best option (keys managed externally) |
| `identity` | None | Plaintext, used for decrypting old data |

---

## Deep Dive RBAC: ServiceAccounts and Tokens

### RBAC core concepts

**Main resources:**

```
Role / ClusterRole              → DEFINES permissions
RoleBinding / ClusterRoleBinding → BINDS permissions to subjects
```

- `Role`: permissions within a namespace
- `ClusterRole`: cluster-wide permissions (or reusable across namespaces)
- `RoleBinding`: binds a Role or ClusterRole to subjects in a namespace
- `ClusterRoleBinding`: binds a ClusterRole to subjects cluster-wide

**Subjects:**
- `User`: human user (managed externally to the cluster)
- `Group`: group of users
- `ServiceAccount`: identity for pods/processes inside the cluster

### Complete RBAC example

```yaml
# Role: can only read pods in the "production" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
# RoleBinding: binds the role to ServiceAccount "monitoring-sa"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production
roleRef:
  kind: Role
  apiRef: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Common verbs and resources

```yaml
# Available verbs
verbs: ["get", "list", "watch", "create", "update", "patch", "delete", "deletecollection"]

# Core resources (apiGroups: [""])
resources: ["pods", "services", "endpoints", "configmaps", "secrets",
            "serviceaccounts", "namespaces", "nodes", "persistentvolumes"]

# Apps resources (apiGroups: ["apps"])
resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]

# Sub-resources (e.g., exec on pods)
resources: ["pods/exec", "pods/log", "pods/portforward"]
```

### ServiceAccounts

Every pod has a ServiceAccount. By default, it uses the `default` ServiceAccount of the namespace.

```yaml
# Create a dedicated ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production

---
# Use in a Pod
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: my-app:1.0
```

### ServiceAccount Tokens

**Automatically mounted token:**
- Mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- JWT used by the pod to authenticate with the Kubernetes API
- Includes claims such as `namespace` and `serviceaccount`

**Inspect a pod's token:**
```bash
kubectl exec -it <pod-name> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d | python3 -m json.tool
```

**Token types:**
1. **Legacy tokens** (Kubernetes < 1.24): no expiration, stored as a Secret
2. **Bound Service Account Tokens** (Kubernetes >= 1.22): expirable, with audience, mounted as a projected volume

```yaml
# Modern token with expiration (projected volume)
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        audience: my-api
        expirationSeconds: 3600
        path: token
```

---

## automountServiceAccountToken and Best Practices

### The automount problem

By default, Kubernetes mounts the ServiceAccount token in **all pods**, even if they don't need to access the Kubernetes API. This increases the attack surface.

```yaml
# Pod with automatically mounted token (default behavior)
# An attacker who compromises the container can use this token
# to authenticate against the Kubernetes API
```

### Disabling automount

**On the ServiceAccount (affects all pods using it):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
  namespace: default
automountServiceAccountToken: false  # ← Disable here
```

**On the Pod (overrides the ServiceAccount configuration):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  automountServiceAccountToken: false  # ← Or disable on the pod
  containers:
  - name: app
    image: my-app:1.0
```

**Priority:** the Pod configuration overrides the ServiceAccount configuration.

### RBAC and ServiceAccount best practices

**1. One ServiceAccount per application:**
```yaml
# BAD: share the "default" ServiceAccount
# GOOD: dedicated ServiceAccount with minimum permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: payments
```

**2. Principle of least privilege:**
```yaml
# Grant only what is necessary
# BAD: ClusterAdmin for everyone
# GOOD:
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]                     # Read-only, configmaps only
  resourceNames: ["app-config"]      # Only this specific configmap
```

**3. Avoid unnecessary ClusterRoleBindings:**
```yaml
# Use RoleBinding + ClusterRole when possible
# (ClusterRole is reusable, but restricted to the namespace via RoleBinding)
kind: RoleBinding  # Restricted to the namespace
roleRef:
  kind: ClusterRole  # Reusable role
  name: pod-reader
```

**4. Permission auditing:**
```bash
# See what a ServiceAccount can do
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa

# Check specific permission
kubectl auth can-i get secrets --as=system:serviceaccount:default:my-sa -n production

# List all RoleBindings
kubectl get rolebindings,clusterrolebindings --all-namespaces -o wide
```

**5. Dangerous roles to avoid:**
```yaml
# NEVER grant these permissions except when absolutely necessary:
# - verb: "*" (wildcard) on any resource
# - resources: "*" (all resources)
# - get/list/watch on "secrets" cluster-wide
# - create on "pods/exec" (remote execution)
# - bind/escalate on roles (enables privilege escalation)
```

### RBAC security checklist example

```bash
# 1. Check bindings with ClusterAdmin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# 2. Check ServiceAccounts with access to secrets
kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
  jq '.items[] | select(.rules[]?.resources[]? == "secrets")'

# 3. List all SA tokens (Kubernetes < 1.24)
kubectl get secrets --all-namespaces | grep service-account-token
```

---

## Key Takeaways for the KCSA Exam

- `--anonymous-auth=false` on kube-apiserver is mandatory for security
- `--authorization-mode=Node,RBAC` is the recommended configuration
- etcd without encryption at rest = Secrets in plaintext accessible to anyone with disk access
- EncryptionConfiguration: the first provider encrypts new data; the rest are fallback for reads
- To re-encrypt existing data: `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`
- The `default` ServiceAccount is automatically assigned if not specified
- `automountServiceAccountToken: false` when the pod does not need to talk to the API
- `kubectl auth can-i --list --as=...` to audit permissions
- ClusterRole + RoleBinding = namespace-scoped permission (better than ClusterRoleBinding)
- Legacy tokens (< 1.24): no expiration, stored as Secret. Modern tokens: expirable, projected volume
