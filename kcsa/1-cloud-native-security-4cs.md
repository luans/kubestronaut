# Day 1 - Cloud Native Security & the 4Cs

## The 4Cs of Cloud Native Security

Cloud native security is built in layers. Each layer depends on the one beneath it: a vulnerability in an outer layer compromises all inner layers, regardless of how well you protect the layers closest to the code.

```
┌─────────────────────────────┐
│           CODE              │  ← Innermost layer
├─────────────────────────────┤
│         CONTAINER           │
├─────────────────────────────┤
│          CLUSTER            │
├─────────────────────────────┤
│           CLOUD             │  ← Outermost layer
└─────────────────────────────┘
```

### Cloud
The infrastructure layer: the cloud provider (AWS, GCP, Azure) or the on-premise data center.

**What to protect:**
- Provider API access (IAM roles, cloud service accounts)
- Network segmentation at the provider level (VPC, security groups, firewall rules)
- Physical and logical access to nodes (node pools, EC2/GKE node instances)
- Control over who can create/destroy clusters and infrastructure resources

**Typical risks:**
- Overly permissive IAM roles (e.g., nodes with AdministratorAccess)
- Instance metadata endpoint accessible without restriction (IMDSv1 on AWS)
- Public S3/GCS buckets containing sensitive cluster data
- Direct SSH to nodes enabled without a bastion host

**Best practices:**
- Least privilege principle for node IAM roles
- Block access to the metadata endpoint on nodes (or use IMDSv2)
- Use a private VPC with nodes that have no public IP
- Enable cloud provider audit logging

---

### Cluster
The Kubernetes layer: control plane, worker nodes, and their components.

**What to protect:**
- kube-apiserver (primary attack surface)
- etcd (stores all cluster state, including Secrets)
- kubelet (agent on nodes, exposes an API)
- kube-scheduler and kube-controller-manager
- Communication between components

**Typical risks:**
- kube-apiserver exposed publicly without strong authentication
- etcd without encryption at rest
- kubelet with anonymous authentication enabled
- Misconfigured RBAC (ClusterAdmin for everyone)
- Kubernetes Dashboard exposed without authentication

**Best practices:**
- Restrict apiserver access by IP (authorized networks)
- Encrypt etcd at rest
- Disable anonymous authentication on kubelet
- Apply RBAC with least privilege

---

### Container
The containerization layer: runtime, images, and isolation.

**What to protect:**
- Container images (vulnerabilities, malware)
- Container runtime (containerd, CRI-O)
- Isolation between containers (Linux namespaces, cgroups)
- Container privileges (root, capabilities)

**Typical risks:**
- Containers running as root
- Images with known vulnerabilities (CVEs)
- Using images without a tag or with the `latest` tag
- Mounting the Docker socket inside the container
- Unnecessary `privileged: true`

**Best practices:**
- Use minimal base images (distroless, alpine)
- Scan images for vulnerabilities (Trivy, Grype)
- Run containers as non-root
- `readOnlyRootFilesystem: true`
- Drop unnecessary capabilities

---

### Code
The application layer: the code running inside the containers.

**What to protect:**
- Application code vulnerabilities (OWASP Top 10)
- Dependencies and libraries (SCA - Software Composition Analysis)
- Hardcoded secrets in code
- Exposed APIs and endpoints

**Typical risks:**
- SQL injection, XSS, SSRF in application code
- Dependencies with known CVEs
- Tokens, passwords, and keys in git repositories
- SSRF exploiting internal Kubernetes endpoints

**Best practices:**
- SAST (Static Application Security Testing) in CI/CD
- SCA for dependencies
- Never hardcode secrets (use environment variables or Vault)
- Validate and sanitize all inputs

---

## Attack Surface in Kubernetes

The attack surface is the complete set of all points where an attacker can try to enter or extract data.

### Exposed components and their risks

| Component | Default Port | Risk if exposed |
|---|---|---|
| kube-apiserver | 6443 | Full cluster control |
| etcd | 2379-2380 | Read/write of all cluster state |
| kubelet API | 10250 | Command execution on pods |
| kube-scheduler | 10259 | Influence pod scheduling |
| kube-controller-manager | 10257 | Control over controllers |
| NodePort Services | 30000-32767 | Access to internal services |

### Common attack vectors

**1. Container Compromise**
- Exploit a vulnerability in the application
- Gain a shell inside the container
- Attempt container escape to the node
- Use the ServiceAccount token to access the API

**2. Supply Chain Compromise**
- Malicious image in the registry
- Backdoored dependency
- CI/CD pipeline compromise

**3. Control Plane Attack**
- Unauthorized access to kube-apiserver
- Direct access to etcd
- Compromise of admin credentials

**4. Lateral Movement**
- Exploit excessive RBAC permissions
- Leverage missing NetworkPolicies to reach other pods
- Escalate privileges via ServiceAccount

### Minimizing the attack surface

```yaml
# Example: Restrict access to kube-apiserver
# In the cluster manifest (kubeadm config)
apiServer:
  certSANs:
    - "10.0.0.1"
  extraArgs:
    anonymous-auth: "false"
    authorization-mode: "Node,RBAC"
    audit-log-path: "/var/log/audit.log"
```

---

## Threat Modeling with STRIDE

STRIDE is a framework for systematically identifying threats. Each letter represents a threat category.

### S - Spoofing
**What it is:** An attacker impersonates another entity.

**In Kubernetes:**
- A pod spoofing the identity of another pod
- A user using stolen credentials
- A malicious image pretending to be a legitimate one

**Countermeasures:**
- mTLS between services (service mesh)
- Strong authentication on the apiserver (OIDC, certificates)
- Image signing (Cosign)

---

### T - Tampering
**What it is:** Modifying data or code without authorization.

**In Kubernetes:**
- Modifying manifests in etcd
- Altering images in the registry
- Modifying ConfigMaps or Secrets

**Countermeasures:**
- Restrictive RBAC (who can write to resources)
- etcd encryption at rest
- Admission controllers to validate resources
- Image signing and verification at deploy time

---

### R - Repudiation
**What it is:** Denying having performed an action.

**In Kubernetes:**
- A user denies deleting a deployment
- A pod denies making a malicious API call

**Countermeasures:**
- kube-apiserver audit logging
- Immutable and centralized logs
- Identity traceability for all actions

---

### I - Information Disclosure
**What it is:** Exposing data to entities that should not have access.

**In Kubernetes:**
- Secrets accessible by unauthorized pods
- Logs containing sensitive data
- etcd without encryption exposing tokens and passwords
- Environment variables with credentials

**Countermeasures:**
- Restrictive RBAC for Secrets
- etcd encryption at rest
- Encryption at rest for Secrets via EncryptionConfiguration
- Avoid secrets in environment variables visible in logs

---

### D - Denial of Service
**What it is:** Making a service unavailable.

**In Kubernetes:**
- A pod consuming all node resources (CPU/memory)
- Request flood to the kube-apiserver
- A container filling the node disk with logs

**Countermeasures:**
- ResourceQuota and LimitRange per namespace
- PodDisruptionBudgets
- Rate limiting on the apiserver
- Horizontal Pod Autoscaler

---

### E - Elevation of Privilege
**What it is:** Gaining permissions beyond what was granted.

**In Kubernetes:**
- A root container escaping to the host node
- A ServiceAccount with excessive RBAC accessing secrets from other namespaces
- Using `privileged: true` to escape the container
- Exploiting `hostPID: true` or `hostNetwork: true`

**Countermeasures:**
- `allowPrivilegeEscalation: false`
- `runAsNonRoot: true`
- Pod Security Admission
- RBAC with least privilege
- Drop unnecessary Linux capabilities

---

## Defense in Depth in the Cloud Native Context

Defense in Depth means having multiple layers of security controls, so that a failure in one layer does not compromise the entire system.

### Defense layers in Kubernetes

```
┌──────────────────────────────────────────────┐
│  CLOUD PROVIDER SECURITY                      │
│  (IAM, VPC, firewall, audit logs)             │
│  ┌────────────────────────────────────────┐   │
│  │  CLUSTER SECURITY                      │   │
│  │  (RBAC, apiserver hardening, etcd enc) │   │
│  │  ┌──────────────────────────────────┐  │   │
│  │  │  NETWORK SECURITY                │  │   │
│  │  │  (NetworkPolicy, mTLS, ingress)  │  │   │
│  │  │  ┌────────────────────────────┐  │  │   │
│  │  │  │  WORKLOAD SECURITY         │  │  │   │
│  │  │  │  (PodSecurity, secCtx)     │  │  │   │
│  │  │  │  ┌──────────────────────┐  │  │  │   │
│  │  │  │  │  APP SECURITY        │  │  │  │   │
│  │  │  │  │  (SAST, SCA, input)  │  │  │  │   │
│  │  │  │  └──────────────────────┘  │  │  │   │
│  │  │  └────────────────────────────┘  │  │   │
│  │  └──────────────────────────────────┘  │   │
│  └────────────────────────────────────────┘   │
└──────────────────────────────────────────────┘
```

### Defense in Depth Principles

**1. Least Privilege**
- Each component has only the minimum permissions needed
- ServiceAccounts with workload-specific RBAC
- Minimal IAM roles for cluster nodes

**2. Segmentation**
- Namespaces as logical boundaries
- NetworkPolicies to isolate workloads
- Dedicated nodes for critical workloads

**3. Assume Breach**
- Act as if a layer has already been compromised
- Monitoring and detection at all layers (Falco)
- Audit logs for forensic traceability

**4. Immutability**
- Immutable containers (`readOnlyRootFilesystem: true`)
- Infrastructure as code (GitOps)
- Signed and verified images

**5. Zero Trust**
- Never trust, always verify
- mTLS between all services
- Authentication and authorization on every call

---

## Key Takeaways for the KCSA Exam

- The 4Cs are: **Cloud, Cluster, Container, Code** (from outside to inside)
- A failure in an outer layer compromises the inner ones
- STRIDE: **S**poofing, **T**ampering, **R**epudiation, **I**nformation Disclosure, **D**enial of Service, **E**levation of Privilege
- Defense in Depth = multiple independent layers of controls
- Kubernetes does not guarantee security by default - defaults are permissive
- The kube-apiserver is the primary entry point and attack surface
- etcd contains ALL cluster state including Secrets in plaintext (without additional encryption)
