# Day 3 - Pod & Workload Security

## SecurityContext

The `SecurityContext` defines privileges and access controls for pods and containers. It can be configured at the Pod level (affects all containers) or at the Container level (overrides the Pod).

### SecurityContext at the Pod level

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true        # Fails if the image runs as root (UID 0)
    runAsUser: 1000           # UID of the main process
    runAsGroup: 3000          # GID of the main process
    fsGroup: 2000             # GID applied to mounted volumes
    fsGroupChangePolicy: "OnRootMismatch"  # When to change fsGroup permissions
    seccompProfile:
      type: RuntimeDefault    # Apply the container runtime's default seccomp profile
    supplementalGroups: [4000]  # Additional GIDs for the process
  containers:
  - name: app
    image: my-app:1.0
    securityContext:          # Overrides the Pod level
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

### runAsNonRoot

```yaml
securityContext:
  runAsNonRoot: true
```

**What it does:** The kubelet verifies that the image does not run as root (UID 0). If it does, the pod fails before starting.

**Important:**
- Checks the effective UID of the process, not just the USER in the Dockerfile
- `runAsNonRoot: true` without `runAsUser` depends on the image defining a non-root USER
- Ideal combination: `runAsNonRoot: true` + `runAsUser: 1000` (explicit)

```yaml
# GOOD: explicit and secure
securityContext:
  runAsNonRoot: true
  runAsUser: 65534  # nobody user
```

### readOnlyRootFilesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

**What it does:** Mounts the container's root filesystem as read-only.

**Why it matters:**
- Prevents malware from writing executables into the container
- Detects persistence attempts
- Forces temporary data to use explicit volumes

**How to handle apps that need to write:**
```yaml
spec:
  securityContext:
    readOnlyRootFilesystem: true
  volumes:
  - name: tmp-dir
    emptyDir: {}
  - name: cache
    emptyDir: {}
  containers:
  - name: app
    volumeMounts:
    - name: tmp-dir
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
```

---

## allowPrivilegeEscalation and Linux Capabilities

### allowPrivilegeEscalation

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

**What it does:** Controls whether the process can gain more privileges than its parent process. Internally, it controls the Linux `no_new_privs` bit.

**Why it matters:**
- Without this flag, a process can execute setuid binaries (e.g., `sudo`)
- Prevents privilege escalation via exploits that use setuid/setgid

**Default:** `true` when the container has `CAP_SYS_ADMIN` or is privileged; `false` otherwise in modern versions.

**Recommendation:** always set explicitly to `false`.

### Linux Capabilities

Linux capabilities divide root privileges into smaller, more granular units.

**Common capabilities and their risks:**

| Capability | What it allows | Risk |
|---|---|---|
| `CAP_SYS_ADMIN` | System admin operations | Nearly equivalent to root |
| `CAP_NET_ADMIN` | Configure network interfaces, firewall | Can manipulate traffic |
| `CAP_NET_RAW` | Create raw sockets | Packet sniffing |
| `CAP_SYS_PTRACE` | Trace processes | Can inject code into other processes |
| `CAP_DAC_OVERRIDE` | Bypass file permissions | Read/write any file |
| `NET_BIND_SERVICE` | Bind to ports < 1024 | Safe for web servers |
| `CAP_CHOWN` | Change file ownership | Can alter file ownership |

**Capabilities configuration:**

```yaml
securityContext:
  capabilities:
    drop:
      - ALL               # Remove ALL capabilities (best practice)
    add:
      - NET_BIND_SERVICE  # Add back only what is needed
```

**Default capabilities added by Docker/containerd:**
`CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW, SETGID, SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT, KILL, AUDIT_WRITE`

**Recommendation:** drop ALL, add only what is necessary.

```yaml
# Most secure possible configuration
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

### Seccomp Profiles

Seccomp (Secure Computing Mode) filters the syscalls a process can make.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault    # Container runtime's default profile
    # type: Localhost       # Custom profile at /var/lib/kubelet/seccomp/
    # localhostProfile: my-profile.json
    # type: Unconfined      # No seccomp (default before K8s 1.27, INSECURE)
```

---

## Pod Security Admission (PSA)

Pod Security Admission replaces Pod Security Policies (PSP), which was removed in Kubernetes 1.25.

### How PSA works

PSA is implemented as a built-in Admission Controller. It evaluates pods against a **Policy Level** defined by a **label on the namespace**.

### The three policy levels

**1. privileged**
- No restrictions
- Allows privileged containers
- Used for system workloads (kube-system)

**2. baseline**
- Prevents known privilege escalations
- Does not allow privileged containers
- Does not allow `hostNetwork`, `hostPID`, `hostIPC`
- Does not allow dangerous hostPath volumes
- Restricts some capabilities

**3. restricted**
- Maximum security, follows all best practices
- Includes everything from baseline, plus:
- Enforces `runAsNonRoot: true`
- Enforces `allowPrivilegeEscalation: false`
- Requires seccomp RuntimeDefault or Localhost
- Drop ALL capabilities is mandatory

### Enforcement modes

For each level, you can define the **mode**:

- `enforce`: rejects pods that violate the policy
- `audit`: allows pods but adds an audit annotation
- `warn`: allows pods but shows a warning to the user

### Configuring PSA via namespace labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted in production
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest

    # Audit to see what would be blocked at the next level
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest

    # Warn for developer feedback
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

```yaml
# Staging namespace with baseline
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/enforce: baseline
```

### PSP vs PSA comparison

| Aspect | PSP (removed) | PSA (current) |
|---|---|---|
| Scope | Cluster-wide | Per namespace |
| Mechanism | Separate Admission Controller | Built into apiserver |
| Configuration | PSP object + RBAC | Namespace labels |
| Complexity | High (hard to understand) | Low |
| Granularity | High (many options) | Three fixed levels |
| Version | Removed in 1.25 | Available since 1.23, GA in 1.25 |

### Testing PSA without enforcement

```bash
# Check if a pod would pass a given level without applying it
kubectl label namespace test pod-security.kubernetes.io/warn=restricted --overwrite

# Create the pod and see the warnings
kubectl apply -f pod.yaml
# Warning: would violate PodSecurity "restricted:latest": ...
```

---

## Privileged vs Unprivileged Containers

### Privileged Container

```yaml
securityContext:
  privileged: true  # ← EXTREMELY DANGEROUS
```

**What happens:**
- Container gains near-complete access to the host
- Can see and modify host devices (`/dev`)
- Can load kernel modules
- Can access host memory
- Essentially root on the host node

**When it is necessary:**
- Containers managing host networking (CNI plugins)
- Containers managing storage (CSI drivers in some cases)
- Low-level debugging tools
- **Never for application workloads**

### Containers with hostPID, hostNetwork, hostIPC

```yaml
spec:
  hostPID: true      # Shares the host's PID namespace (can see all processes)
  hostNetwork: true  # Shares the host's network interface
  hostIPC: true      # Shares the host's IPC namespace
```

**Risks:**
- `hostPID`: can see and interact with host processes, including other containers
- `hostNetwork`: bypasses network isolation, accesses internal node services
- `hostIPC`: can interact with processes via shared memory

### hostPath Volumes (dangerous)

```yaml
volumes:
- name: host-vol
  hostPath:
    path: /etc      # ← Mounting /etc from the host into the container!
    type: Directory
```

**Risks:**
- Reading sensitive host files (`/etc/shadow`, `/etc/kubernetes/`)
- Writing to host files
- `hostPath: /var/run/docker.sock` = full control of the Docker daemon

**Especially dangerous paths:**
- `/` - root filesystem
- `/etc` - system configuration
- `/var/run/docker.sock` - Docker socket
- `/proc` - process information
- `/sys` - kernel interfaces

### Security comparison

```yaml
# INSECURE: privileged container
spec:
  containers:
  - name: app
    securityContext:
      privileged: true

# SECURE: restricted container
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

---

## Complete Example: Pod with Secure Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: production
spec:
  serviceAccountName: app-sa  # Dedicated SA, not default
  automountServiceAccountToken: false  # does not need the API
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:1.2.3  # specific tag, not :latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## Key Takeaways for the KCSA Exam

- `runAsNonRoot: true` + `runAsUser: UID` is the recommended combination
- `readOnlyRootFilesystem: true` requires volumes for writable directories
- `allowPrivilegeEscalation: false` controls the Linux `no_new_privs` bit
- Capabilities: `drop: ["ALL"]` + add back only what is needed
- PSA replaces PSP (removed in K8s 1.25)
- PSA has 3 levels: `privileged` < `baseline` < `restricted`
- PSA has 3 modes: `enforce` (blocks), `audit` (annotates), `warn` (warns)
- PSA is configured via **namespace labels**
- `privileged: true` = near root on the host - never for normal workloads
- `hostPID`, `hostNetwork`, `hostIPC` break container isolation
- `hostPath: /var/run/docker.sock` = full control of the Docker daemon
- SecurityContext at the Container level overrides the Pod level
