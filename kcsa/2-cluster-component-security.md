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

### Network-level access restriction

Hardening flags alone are not enough if the API server is reachable from the internet. The network layer must enforce who can even attempt a connection.

**On managed clusters (EKS, GKE, AKS): private endpoint**

The strongest control is disabling the public endpoint entirely so port 6443 is only reachable inside the VPC:

```hcl
# Terraform — EKS: no public endpoint, private only
vpc_config {
  endpoint_private_access = true
  endpoint_public_access  = false
  # Access requires being inside the VPC (VPN, SSM, bastion)
}
```

If a public endpoint must stay on during a transition, restrict it to known CIDRs:

```hcl
# Only the corporate VPN and CI/CD runner egress IPs can reach the API
public_access_cidrs = ["203.0.113.10/32", "198.51.100.0/24"]
```

**On self-managed clusters (kubeadm): bind address and firewall**

```bash
# Bind the apiserver only to the internal network interface
--bind-address=10.0.1.5        # internal node IP, not 0.0.0.0

# Firewall rule (iptables / cloud security group):
# Allow 6443 only from: control-plane nodes, worker nodes, VPN CIDR
# Deny 6443 from: 0.0.0.0/0 (internet)
```

**kubelet API (port 10250) — also must be restricted:**

```bash
# kubelet flags
--address=10.0.1.5             # bind to internal IP only
--anonymous-auth=false         # require authentication
--authorization-mode=Webhook   # delegate authorization to apiserver

# Security group / firewall: allow 10250 only from control-plane subnet
# Workers should never expose kubelet to the internet
```

**etcd (ports 2379–2380) — must be completely internal:**

```bash
# etcd listens only on the internal interface
--listen-client-urls=https://10.0.1.5:2379
--listen-peer-urls=https://10.0.1.5:2380
# Never bind to 0.0.0.0 — etcd has no defense against direct read access
# besides TLS client certificates
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

## Node OS: Choosing a Secure Operating System for Kubernetes

The operating system running on Kubernetes nodes is a critical security layer. A general-purpose OS brings thousands of packages, services, and attack vectors that are irrelevant for running containers. Container-optimized OSes reduce this surface drastically.

### Why OS choice matters

```
General-purpose OS (Ubuntu/Debian/RHEL):
  - Full package manager (apt/yum)
  - Hundreds of pre-installed services
  - SSH, cron, syslog daemons
  - Shell environment with tools (curl, wget, gcc...)
  - Large attack surface on every node

Container-optimized OS (Bottlerocket, Flatcar, Talos):
  - No package manager
  - Minimal or no interactive shell
  - Only what is needed to run containers
  - Immutable root filesystem
  - Significantly reduced attack surface
```

### Comparison of node OS options

| OS | Package manager | Shell on node | Root FS | Update model | Best for |
|---|---|---|---|---|---|
| Ubuntu/Debian | apt | Yes | Mutable | Traditional | General purpose, flexibility |
| Amazon Linux 2/2023 | yum/dnf | Yes | Mutable | Traditional | AWS workloads |
| **Bottlerocket** | **None** | **Admin container only** | **Read-only** | **Atomic (A/B)** | **AWS EKS, security-focused** |
| Flatcar Container Linux | None | Limited (via toolbox) | Read-only | Atomic (A/B) | Any cloud, CoreOS successor |
| Talos Linux | None | None (API-only) | Immutable | Atomic | Maximum security, air-gapped |

---

## Bottlerocket: Purpose-Built OS for Containers

Bottlerocket is an open source, Linux-based OS developed by AWS specifically for running containers. It is the recommended node OS for Amazon EKS and a strong security choice in any environment.

### Core security design principles

**1. Read-only root filesystem**

The OS partition is mounted read-only. No process (including root inside a container) can modify the OS binaries, libraries, or configuration.

```
/  (read-only)          ← OS files, immutable
├── /etc                ← OS config, read-only
├── /usr                ← Binaries, read-only
└── /local              ← Writable, data partition
    ├── /var            ← Container data, logs
    └── /opt            ← User data
```

**2. No package manager**

There is no `apt`, `yum`, `dnf`, or any package manager. It is impossible to install new software on the node at runtime. This eliminates:
- Post-compromise package installation
- Dependency confusion attacks on the node
- Accidental software installation

**3. No interactive shell by default**

SSH is not enabled by default. There is no shell available via normal paths. Accessing the node requires enabling the **admin container** (a privileged, separate container with an SSH server).

```bash
# To access the node (must be enabled explicitly):
# 1. Enable admin container via SSM or bootstrap config
# 2. SSH into the admin container (not the host directly)
# 3. Use "enter-admin-container" to get a host shell
```

**4. Atomic (A/B) updates**

Bottlerocket maintains two OS partitions (A and B). Updates apply to the inactive partition, then flip. Rollback is instant — just reboot to the previous partition.

```
┌─────────────────────────────────────────┐
│  Disk layout                            │
├─────────────┬───────────────────────────┤
│  Partition A│  Partition B              │
│  (active)   │  (update target)          │
│  v1.15.0    │  v1.16.0 ← applied here  │
├─────────────┴───────────────────────────┤
│  Data partition (writable)              │
│  /local/var, container data, config     │
└─────────────────────────────────────────┘
             ↓ reboot to activate
┌─────────────────────────────────────────┐
│  Partition A│  Partition B              │
│  (fallback) │  (active) v1.16.0        │
└─────────────────────────────────────────┘
```

**5. dm-verity (kernel-level integrity)**

Bottlerocket uses Linux `dm-verity` to cryptographically verify the integrity of the OS partition at boot. If any bit of the OS is tampered with, the node will refuse to boot. This prevents persistent rootkits from surviving a reboot.

**6. SELinux enforced**

SELinux is enabled in enforcing mode by default. This provides mandatory access control (MAC) over all processes, including the container runtime. Containers cannot access files or resources outside their defined policy, even if they run as root.

**7. Measured boot and TPM attestation**

Bottlerocket supports measured boot, recording the boot state in the TPM. This enables remote attestation — verifying that a node booted the expected, unmodified OS.

### Architecture: control channel instead of SSH

Management is done through a declarative API, not a shell:

```
┌──────────────────────────────────────────────┐
│  Management paths in Bottlerocket            │
│                                              │
│  1. apiclient (local, on the node)           │
│     → UNIX socket to the Bottlerocket API    │
│                                              │
│  2. Control container                         │
│     → Enabled by default                    │
│     → Lightweight container with apiclient   │
│     → Access via AWS SSM Session Manager     │
│                                              │
│  3. Admin container (disabled by default)    │
│     → Full shell access (emergency use)      │
│     → Requires explicit enablement           │
│     → SSH into the container, then           │
│        "enter-admin-container" for host ns   │
└──────────────────────────────────────────────┘
```

### Bootstrap configuration (user data)

Bottlerocket is configured via TOML in the instance user-data (on AWS), not by running commands:

```toml
# Bottlerocket node configuration (user data / bootstrap)
[settings.kubernetes]
cluster-name = "my-cluster"
api-server = "https://xxxxx.gr7.us-east-1.eks.amazonaws.com"

[settings.kubernetes.node-labels]
"node.kubernetes.io/role" = "worker"

[settings.kubernetes.node-taints]
"dedicated" = "gpu:NoSchedule"

# Enable control container for management via SSM
[settings.host-containers.control]
enabled = true
superpowered = false

# Admin container: disabled by default, enable only when needed
[settings.host-containers.admin]
enabled = false
```

### Using Bottlerocket with EKS managed node groups

```yaml
# Terraform: EKS managed node group with Bottlerocket
resource "aws_eks_node_group" "bottlerocket" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "bottlerocket-workers"
  ami_type        = "BOTTLEROCKET_x86_64"  # or BOTTLEROCKET_ARM_64

  scaling_config {
    desired_size = 3
    min_size     = 1
    max_size     = 10
  }

  # Bottlerocket configuration via launch template
  launch_template {
    id      = aws_launch_template.bottlerocket.id
    version = "$Latest"
  }
}
```

```yaml
# eksctl cluster configuration with Bottlerocket
managedNodeGroups:
  - name: bottlerocket-ng
    amiFamily: Bottlerocket  # ← specify Bottlerocket
    instanceType: m5.large
    minSize: 2
    maxSize: 10
    bottlerocket:
      settings:
        kubernetes:
          cluster-dns-ip: "10.100.0.10"
```

### Security features summary

| Feature | Bottlerocket | Ubuntu | Flatcar | Talos |
|---|---|---|---|---|
| Read-only root FS | Yes | No | Yes | Yes |
| Package manager | No | apt | No | No |
| Interactive shell | Admin container | Yes | toolbox | No (API only) |
| Atomic updates | A/B partitions | No | A/B partitions | Yes |
| SELinux enforced | Yes | Optional | Yes | No (AppArmor-like) |
| dm-verity | Yes | No | No | Yes |
| Default SSH | No | Yes | No | No |
| Managed by | AWS (open source) | Canonical | Kinvolk/Microsoft | Sidero Labs |
| CIS benchmark | Yes (CIS Bottlerocket) | CIS Ubuntu | CIS Flatcar | N/A |

### When to choose Bottlerocket

**Choose Bottlerocket when:**
- Running on AWS EKS (native integration, managed AMIs)
- Security posture requires minimal attack surface on nodes
- Compliance requirements mandate immutable infrastructure
- You want atomic updates with instant rollback
- You need SELinux + dm-verity without manual configuration

**Consider alternatives when:**
- Need to run non-containerized workloads on nodes (use Ubuntu/Amazon Linux)
- On-premise or non-AWS cloud (consider Flatcar or Talos)
- Need maximum control via API-only (consider Talos)
- Legacy infrastructure that requires SSH-based management tools

### Bottlerocket vs general-purpose OS: security impact

```
Attack scenario: compromised container escapes to node

General-purpose OS node:
  Container escape → Root shell on node → Install malware → Persist across reboots
                                        → Read /etc/shadow → Lateral movement
                                        → Install backdoor → Modify cron

Bottlerocket node:
  Container escape → Root shell on admin container (if enabled) → SELinux restricts
                   → Cannot write to OS partition (dm-verity + read-only)
                   → Malware removed on reboot (immutable OS)
                   → No package manager to install tools
                   → No persistent shell without explicit admin container
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
- Container-optimized OSes (Bottlerocket, Flatcar, Talos) reduce node attack surface drastically
- Bottlerocket: read-only root FS + no package manager + no SSH by default + SELinux enforced
- Bottlerocket uses dm-verity: tampering with the OS partition prevents the node from booting
- A/B atomic updates: updates apply to inactive partition, reboot activates — instant rollback
- Bottlerocket is managed via API (apiclient), not SSH; admin container is disabled by default
- On AWS EKS: `amiFamily: Bottlerocket` (eksctl) or `ami_type: "BOTTLEROCKET_x86_64"` (Terraform)
