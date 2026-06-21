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
- Dev and prod clusters sharing the same AWS account (blast radius amplification)
- Kubernetes API server publicly accessible from the internet

**Best practices:**
- Least privilege principle for node IAM roles
- Block access to the metadata endpoint on nodes (or use IMDSv2)
- Use a private VPC with nodes that have no public IP
- Enable cloud provider audit logging
- Separate dev and prod into different AWS accounts (AWS Organizations)
- Expose the Kubernetes API only within the VPC, restricted to specific subnets/CIDRs

### Multi-account strategy: isolating environments

Running dev and prod in the same AWS account is a critical risk. A misconfigured IAM role, a leaked credential, or an accidental `kubectl delete` in the wrong context can affect production.

```
❌ BAD: single account for everything

  Account: 123456789012
  ├── EKS cluster: dev
  ├── EKS cluster: staging
  └── EKS cluster: prod       ← same IAM boundary, same blast radius

✅ GOOD: one account per environment (AWS Organizations)

  Organization
  ├── Management Account (billing, governance only)
  ├── Account: dev    (111111111111)  ← EKS dev cluster
  ├── Account: staging (222222222222) ← EKS staging cluster
  └── Account: prod   (333333333333)  ← EKS prod cluster
```

**Why separate accounts:**

| Risk | Single account | Separate accounts |
|---|---|---|
| Leaked dev credential | Can reach prod resources | Isolated to dev account |
| Accidental `terraform destroy` | Destroys prod | Destroys only its own env |
| Overly permissive dev IAM role | Can escalate to prod | No cross-account access |
| Compliance scope (PCI, SOC2) | Entire account in scope | Only prod account in scope |
| Cost visibility | Mixed, hard to attribute | Per-environment billing |
| Security groups / VPCs | Must be carefully separated | Completely separate by default |

**AWS Organizations structure:**

```
Root
└── Organization Units (OUs)
    ├── OU: Security
    │   └── Account: security-tooling (GuardDuty, SecurityHub, CloudTrail aggregation)
    ├── OU: Production
    │   ├── Account: prod-eks       ← production cluster
    │   └── Account: prod-data      ← databases, S3
    ├── OU: Non-Production
    │   ├── Account: staging-eks
    │   └── Account: dev-eks
    └── OU: Shared Services
        └── Account: shared-services (ECR, internal tooling)
```

**Cross-account access when needed (e.g., CI/CD deploying to prod):**

```yaml
# In the prod account: IAM role that allows the CI/CD account to assume it
# Trust policy on the prod role:
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::CICD_ACCOUNT_ID:role/github-actions-role"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-external-id"  # prevents confused deputy
    }
  }
}
```

**Service Control Policies (SCPs) — enforced at the OU level:**

```json
// Example SCP: prevent anyone in non-prod OUs from touching prod resources
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceAccount": "333333333333"  // prod account ID
    }
  }
}
```

### Network isolation: keeping the Kubernetes API private

The Kubernetes API server (port 6443) should **never be publicly accessible**. Any exposure to the internet makes the cluster a target for credential stuffing, CVE exploits, and brute-force attacks.

**EKS endpoint access modes:**

```
Public endpoint (default — INSECURE for production):
  Internet → EKS API (6443) ← accessible from anywhere

Public + Private (transition):
  Internet → EKS API (accessible, but restrict with authorized CIDRs)
  VPC      → EKS API (also accessible via private endpoint)

Private endpoint only (RECOMMENDED for production):
  Internet ✗ (no public endpoint)
  VPC      → EKS API via AWS PrivateLink (stays inside the VPC)
```

**Configuring private endpoint on EKS (Terraform):**

```hcl
resource "aws_eks_cluster" "main" {
  name     = "prod-cluster"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true   # ← enable private endpoint
    endpoint_public_access  = false  # ← disable public endpoint

    # If public access must remain on temporarily, restrict to known CIDRs:
    # endpoint_public_access  = true
    # public_access_cidrs     = ["203.0.113.10/32"]  # VPN/office IP only
  }
}
```

**Configuring private endpoint with eksctl:**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: prod-cluster
  region: us-east-1

vpc:
  clusterEndpoints:
    privateAccess: true   # API accessible inside VPC
    publicAccess: false   # no internet exposure
  # Optional: restrict which subnets can reach the API
  subnets:
    private:
      us-east-1a: { id: subnet-aaa }
      us-east-1b: { id: subnet-bbb }
```

**Network architecture: what should and should not be accessible from the internet:**

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────┐
│  VPC (private, no internet gateway for nodes)        │
│                                                      │
│  Public subnets (only ALB/NLB for app traffic):      │
│  ┌─────────────────────┐                             │
│  │  Load Balancer      │ ← port 80/443 (app only)   │
│  └──────────┬──────────┘                             │
│             │                                        │
│  Private subnets (nodes, control plane):             │
│  ┌──────────▼──────────┐  ┌─────────────────────┐   │
│  │  Worker Nodes       │  │  EKS Control Plane  │   │
│  │  (no public IP)     │  │  (private endpoint) │   │
│  └─────────────────────┘  └─────────────────────┘   │
│             ↑                        ↑               │
│             │         VPC only       │               │
│  ┌──────────────────────────────┐    │               │
│  │  Kubectl access:             │    │               │
│  │  - VPN → private subnet      │────┘               │
│  │  - AWS Systems Manager (SSM) │                    │
│  │  - Bastion host (jump box)   │                    │
│  └──────────────────────────────┘                    │
└──────────────────────────────────────────────────────┘

✗ kube-apiserver:6443   → NOT accessible from internet
✗ kubelet:10250         → NOT accessible from internet
✗ etcd:2379-2380        → NOT accessible from internet
✓ App LoadBalancer:443  → accessible from internet (only app traffic)
```

**Security groups: restricting who can reach the API:**

```hcl
# Security group for EKS control plane (applied to the API endpoint)
resource "aws_security_group_rule" "eks_api_ingress" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id

  # Allow only from VPN subnet and CI/CD subnet — not from internet
  cidr_blocks = [
    "10.0.0.0/8",       # internal VPC range
    "172.16.10.0/24",   # corporate VPN subnet
  ]
}
```

**Accessing the private API: recommended approaches:**

```
Option 1: VPN (site-to-site or client VPN)
  Developer laptop → VPN → corporate network → VPC → EKS private API

Option 2: AWS Client VPN (managed)
  Developer laptop → AWS Client VPN endpoint → VPC → EKS private API

Option 3: Systems Manager Session Manager (no VPN needed)
  Developer → SSM Session Manager → bastion EC2 → kubectl from inside VPC

Option 4: Bastion host (jump box)
  Developer → SSH to bastion (in VPC) → kubectl from bastion
  Security: use EC2 Instance Connect or SSM instead of permanent key pairs
```

**kubeconfig for private clusters:**

```yaml
# kubeconfig pointing to internal/VPN-only endpoint
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://internal-EKS-endpoint.us-east-1.eks.amazonaws.com
    # ↑ This resolves to a private IP — only reachable inside the VPC
    certificate-authority-data: <base64-ca>
  name: prod-cluster
```

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

## Shared Responsibility Model in Kubernetes

"Someone manages it" does not mean "it is secure by default." The shared responsibility model defines *who is responsible for what* — and the boundary shifts depending on whether you run Kubernetes yourself or use a managed service.

### The fundamental division

```
┌───────────────────────────────────────────────────────────┐
│  CLOUD PROVIDER responsibility                            │
│  Physical infrastructure, hypervisor, network fabric,    │
│  managed service availability (SLA)                      │
├───────────────────────────────────────────────────────────┤
│  CUSTOMER responsibility                                  │
│  What runs ON TOP: OS config, cluster config, workloads, │
│  RBAC, network policies, secrets, application code        │
└───────────────────────────────────────────────────────────┘
```

The boundary moves inward (toward the customer) as you use more managed services — but it never disappears completely. Security of workloads, RBAC, network policies, and application code is **always the customer's responsibility**.

---

### Deployment models and responsibility boundaries

#### Model 1: Self-managed Kubernetes (EC2 / bare metal)

You provision EC2 instances and install Kubernetes yourself (e.g., with kubeadm). You own everything.

```
AWS manages:                      Customer manages:
─────────────────                 ──────────────────────────────────────
Physical hardware                 EC2 instance OS (patches, hardening)
Hypervisor (Nitro)                Control plane (apiserver, etcd, scheduler)
Network fabric                    Worker node OS
Data center security              Container runtime (containerd, CRI-O)
                                  CNI plugin (Calico, Cilium, etc.)
                                  RBAC and authentication
                                  Network policies
                                  Cluster certificates (rotation)
                                  etcd backups
                                  All application workloads
                                  Secrets management
                                  Security patches for all components
```

**Risk:** maximum flexibility, maximum operational and security burden. A misconfigured apiserver, an unpatched etcd, or a forgotten certificate rotation is entirely your problem.

---

#### Model 2: EKS with self-managed nodes

AWS manages the control plane. You still manage the worker nodes.

```
AWS manages:                      Customer manages:
─────────────────                 ──────────────────────────────────────
Control plane HA and patches      Worker node OS (AMI, patches)
  - kube-apiserver                Node IAM roles and instance profiles
  - etcd (multi-AZ, encrypted)    Container runtime on nodes
  - kube-scheduler                CNI plugin configuration
  - kube-controller-manager       RBAC and authentication
  - Control plane certificates    Network policies
  - Control plane audit logs      Pod security (SecurityContext, PSA)
EKS API endpoint availability     Application workloads
                                  Secrets encryption (EncryptionConfig)
                                  Cluster add-ons (CoreDNS, kube-proxy)
                                  Node security groups
```

**Key insight:** AWS guarantees the control plane is running and patched. It does *not* guarantee that your RBAC is correct, your pods are non-privileged, or your secrets are encrypted. A misconfigured `ClusterRoleBinding` granting `cluster-admin` to all ServiceAccounts is still your problem.

---

#### Model 3: EKS with managed node groups

AWS manages the node lifecycle (provisioning, patching, draining, replacing). You still manage what runs on the nodes.

```
AWS manages:                      Customer manages:
─────────────────                 ──────────────────────────────────────
Everything from Model 2, plus:    Container runtime security config
  Node AMI selection              Pod-level security (SecurityContext)
  Node OS patching                RBAC and authentication
  Node draining on update         Network policies
  Node replacement on failure     Application workloads
  Node group scaling              Secrets management
                                  Namespace isolation
                                  Image scanning and signing
```

**Key insight:** AWS updates the node OS, but it does not validate what your pods are doing on those nodes. A pod with `privileged: true` or `hostNetwork: true` is your responsibility.

---

#### Model 4: EKS with Fargate

No nodes to manage. Each pod runs in its own isolated micro-VM (Firecracker). AWS manages the infrastructure under each pod.

```
AWS manages:                      Customer manages:
─────────────────                 ──────────────────────────────────────
Everything from Model 3, plus:    Pod specifications (SecurityContext)
  Pod-level infrastructure        RBAC and authentication
  Kernel and OS for each pod      Network policies (required — no node SG)
  Node isolation (Firecracker)    Application workloads
  Runtime patching                Secrets management
  No shared nodes between tenants Namespace isolation
                                  Resource limits (required on Fargate)
                                  Image security (scanning, signing)
```

**Key insight:** Fargate eliminates the node attack surface for you (no SSH to nodes, no node OS to patch). But it does not protect you from a malicious or misconfigured container. RBAC, network policies, and pod security context are still entirely your responsibility.

---

### Comparison table across models

| Responsibility area | Self-managed | EKS + self nodes | EKS + managed NG | EKS Fargate |
|---|---|---|---|---|
| Physical hardware | AWS | AWS | AWS | AWS |
| Control plane (apiserver, etcd) | **Customer** | AWS | AWS | AWS |
| Control plane patching | **Customer** | AWS | AWS | AWS |
| Worker node OS patching | **Customer** | **Customer** | AWS | AWS |
| Container runtime patching | **Customer** | **Customer** | AWS (partial) | AWS |
| Node provisioning/replacement | **Customer** | **Customer** | AWS | AWS |
| RBAC configuration | **Customer** | **Customer** | **Customer** | **Customer** |
| Network policies | **Customer** | **Customer** | **Customer** | **Customer** |
| Pod security (SecurityContext) | **Customer** | **Customer** | **Customer** | **Customer** |
| Secrets encryption (etcd) | **Customer** | **Customer** | **Customer** | **Customer** |
| Application workload security | **Customer** | **Customer** | **Customer** | **Customer** |
| Image scanning | **Customer** | **Customer** | **Customer** | **Customer** |
| Compliance verification | **Customer** | **Customer** | **Customer** | **Customer** |

**The columns get shorter toward the right — but the customer's row never disappears.**

---

### Equivalents in other cloud providers

| Concern | AWS | GCP | Azure |
|---|---|---|---|
| Managed control plane | EKS | GKE | AKS |
| Managed nodes | Managed Node Groups | Node Pools (auto-upgrade) | Node Pools |
| Serverless pods | Fargate | Autopilot / Cloud Run | ACI (Virtual Kubelet) |
| Private endpoint | EKS private endpoint | Private cluster | Private cluster |
| Audit logs | CloudTrail + EKS audit | Cloud Audit Logs | Azure Monitor |
| Multi-account isolation | AWS Organizations | GCP folders/projects | Azure Management Groups |
| Node OS (managed) | Bottlerocket (EKS) | Container-Optimized OS | AKS node images |

---

### What "managed" does NOT mean

This is a critical exam concept. Managed Kubernetes gives you a managed control plane, not a secure cluster.

```
❌ Common misconception:
"We use EKS, so Kubernetes security is AWS's problem."

✅ Reality:
AWS manages: control plane availability, OS patching on managed nodes, SLA
Customer manages: everything that makes the cluster *actually secure*:
  - Who can access what (RBAC)
  - What pods can do (SecurityContext, PSA)
  - How pods communicate (NetworkPolicy)
  - Where secrets are stored and who can read them
  - What images are running (scanning, signing)
  - How workloads are isolated (namespaces, node pools)
  - Audit log retention and alerting
```

**Analogy:** a managed Kubernetes cluster is like a managed apartment building. The landlord maintains the structure, heating, and elevator. But *you* are responsible for locking your door, not leaving the stove on, and not letting strangers in.

---

### Security checklist by responsibility layer

**AWS's responsibility (verify the service is configured correctly):**
```
✓ EKS control plane is running (check cluster status)
✓ etcd is encrypted by default in EKS (AWS-managed KMS key)
✓ Control plane logs are enabled (API, authenticator, audit, scheduler, controllerManager)
✓ Managed node group AMIs are up to date (check for pending updates)
```

**Customer's responsibility (you must configure these explicitly):**
```
✓ Private endpoint enabled, no public access
✓ RBAC: no wildcard roles, no unnecessary cluster-admin bindings
✓ Pod Security Admission enforced per namespace
✓ NetworkPolicies defined (default deny + explicit allow)
✓ Secrets encrypted with customer-managed KMS key
✓ Container images scanned and signed
✓ Node IAM roles follow least privilege (no AdministratorAccess)
✓ CloudTrail and EKS audit logs shipped to centralized logging
✓ Falco or equivalent runtime detection running
✓ Resource limits defined on all pods (CPU, memory)
```

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
- Dev and prod must be in **separate AWS accounts**: different IAM boundary, different blast radius, compliance scope isolation
- AWS Organizations + SCPs enforce account-level guardrails that even account admins cannot bypass
- EKS private endpoint (`endpoint_public_access: false`) keeps the API inside the VPC — never expose it to the internet
- Authorized CIDRs (`public_access_cidrs`) as a transitional measure, private endpoint as the target state
- Access to private clusters: VPN → VPC, AWS Client VPN, or SSM Session Manager (no permanent bastion key pairs)
- Nodes should have **no public IP** and sit in private subnets; only the Load Balancer lives in the public subnet
- **Shared responsibility**: the cloud provider manages infrastructure; RBAC, network policies, pod security, and application security are always the customer's responsibility
- EKS manages the control plane (apiserver, etcd, patching) — it does NOT manage your RBAC, NetworkPolicies, or SecurityContexts
- EKS Fargate eliminates node management but does NOT eliminate pod-level security responsibility
- "We use a managed cluster" ≠ "security is the provider's problem" — the responsibility boundary moves but never reaches zero for the customer
- Managed node groups patch node OS automatically; self-managed nodes require the customer to track and apply AMI/OS updates
- etcd in EKS is encrypted by default with an AWS-managed key; customer-managed KMS key is an additional hardening step
