# Day 6 - Secrets, Compliance & Frameworks

## Kubernetes Secrets: Best Practices and Limitations

### What are Kubernetes Secrets

Secrets are Kubernetes objects for storing sensitive data such as passwords, tokens, TLS keys, and more. They are similar to ConfigMaps, but with different access controls.

### How Secrets are stored (and the problem)

**By default, Secrets are stored as Base64 in etcd - they are NOT encrypted.**

```bash
# A Secret created like this:
kubectl create secret generic db-password --from-literal=password=MySecretPass123

# Is stored in etcd as:
# /registry/secrets/default/db-password
# { "password": "TXlTZWNyZXRQYXNzMTIz" }  <- Base64 of "MySecretPass123"

# Anyone with access to etcd can decode it:
echo "TXlTZWNyZXRQYXNzMTIz" | base64 -d
# MySecretPass123
```

### Secret Types

```yaml
# Opaque (generic, most common)
type: Opaque
data:
  username: YWRtaW4=   # base64 of "admin"
  password: cGFzc3dvcmQ=  # base64 of "password"

# TLS
type: kubernetes.io/tls
data:
  tls.crt: <BASE64_CERT>
  tls.key: <BASE64_KEY>

# Service Account Token
type: kubernetes.io/service-account-token

# Docker registry credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <BASE64_JSON>

# Basic Auth
type: kubernetes.io/basic-auth
data:
  username: <BASE64>
  password: <BASE64>

# SSH Auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <BASE64>
```

### Creating and using Secrets

```bash
# Create a Secret
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=s3cur3pass

# From a file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=cert.pem \
  --from-file=tls.key=key.pem

# Via YAML (avoid committing to git!)
kubectl apply -f secret.yaml
```

**Using a Secret in a Pod:**
```yaml
# As an environment variable
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    # Expose entire Secret as env vars
    envFrom:
    - secretRef:
        name: db-secret

---
# As a volume (more secure than env vars)
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
      defaultMode: 0400  # Owner read-only
  containers:
  - name: app
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
```

**Volume vs Environment Variable:**

| Aspect | Env Var | Volume |
|---|---|---|
| Visibility in `kubectl describe` | Does not show value | Does not show value |
| `kubectl exec -- env` | Visible! | Not visible |
| Application logs | Risk of leaking | Lower risk |
| Update without restart | No | Yes (with delay) |
| Child processes | Inherit env vars | Do not inherit |
| **Recommendation** | Avoid for passwords | Prefer |

### Kubernetes Secret Limitations

1. **Not encrypted by default** - only Base64 (encoding, not encryption)
2. **Visible to anyone with etcd access**
3. **Visible to anyone with RBAC `get secrets`**
4. **Appear as plain text in `kubectl get secret -o yaml`**
5. **Replicated to all nodes that need them** (kubelet copies them)
6. **Maximum size: 1 MB per Secret**
7. **No native versioning** - no change history
8. **No automatic rotation** - you manage rotation manually

### Best practices with Kubernetes Secrets

```bash
# 1. Enable encryption at rest (covered in Day 2)
# EncryptionConfiguration with aescbc or kms

# 2. Restrictive RBAC for Secrets
kubectl create role secret-reader \
  --verb=get,list \
  --resource=secrets \
  --resource-name=app-secret  # Only this specific secret

# 3. Avoid committing secrets to Git
# Use .gitignore for secret files
# Use git-secrets or detect-secrets to prevent accidental commits

# 4. Regular rotation
kubectl create secret generic db-secret-v2 --from-literal=password=NewPass456
# Update references in pods
# Delete old version

# 5. Avoid secrets in environment variables when possible
# Prefer volumes
```

---

## Vault and Sealed Secrets

### HashiCorp Vault

Vault is a specialized tool for secrets management, far more secure than native Kubernetes Secrets.

**Vault advantages:**
- Real encryption (AES-256-GCM)
- Automatic secret rotation
- Leasing: secrets with short TTLs
- Full audit log (who accessed what and when)
- Multiple backends (AWS, GCP, LDAP, etc.)
- Dynamic secrets (generates database credentials on-demand)

**Kubernetes integration:**

```
Pod                    Vault Agent          Vault Server
 │                        │                    │
 │  needs a secret        │                    │
 │───────────────────────►│                    │
 │                        │ authenticates with SA│
 │                        │───────────────────►│
 │                        │ ◄── secret ──────── │
 │ ◄── secret (file) ─────│                    │
```

**Vault Agent Injector (sidecar):**

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    # Enable Vault Agent injection
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app-role"
    # Which secret to inject and where
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/my-role"
    # Template to format the secret
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/my-role" -}}
      export DB_USER={{ .Data.data.username }}
      export DB_PASS={{ .Data.data.password }}
      {{- end -}}
spec:
  serviceAccountName: my-app-sa
  containers:
  - name: app
    image: my-app:1.0
    # Secret available at /vault/secrets/db-creds
```

**Kubernetes authentication in Vault:**

```bash
# Configure the Kubernetes auth method in Vault
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc:443 \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for the app
vault policy write my-app-policy - <<EOF
path "secret/data/my-app/*" {
  capabilities = ["read"]
}
path "database/creds/my-role" {
  capabilities = ["read"]
}
EOF

# Create a role binding SA to Vault
vault write auth/kubernetes/role/my-app-role \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=production \
  policies=my-app-policy \
  ttl=1h
```

**Dynamic Secrets for databases:**
```bash
# Vault generates temporary database credentials
vault secrets enable database

vault write database/config/my-postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
  allowed_roles=my-role \
  username=vault-admin \
  password=admin-pass

vault write database/roles/my-role \
  db_name=my-postgres \
  creation_statements="CREATE USER {{name}} WITH PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'" \
  revocation_statements="DROP USER IF EXISTS {{name}}" \
  default_ttl=1h \
  max_ttl=24h
```

### Sealed Secrets (Bitnami)

Sealed Secrets solves the problem of securely storing secrets in Git using asymmetric encryption.

**How it works:**

```
Secret YAML (plaintext) → kubeseal → SealedSecret YAML (encrypted)
                                          │
                              Can be committed to Git!
                                          │
                              Kubernetes Controller
                                          │
                              Decrypts and creates real Secret
```

**Installation:**
```bash
# Controller in the cluster
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# kubeseal CLI
brew install kubeseal
```

**Creating a Sealed Secret:**
```bash
# 1. Create a normal Secret (do NOT apply to the cluster)
kubectl create secret generic db-secret \
  --from-literal=password=MyPass123 \
  --dry-run=client -o yaml > secret.yaml

# 2. Encrypt with kubeseal
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 3. Commit sealed-secret.yaml to Git (safe!)
git add sealed-secret.yaml
git commit -m "add db sealed secret"

# 4. Apply to the cluster (controller decrypts)
kubectl apply -f sealed-secret.yaml
```

**Generated SealedSecret:**
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: production
spec:
  encryptedData:
    password: AgBzKj9m... # Encrypted with the cluster's public key
```

**Encryption scopes:**
- `strict` (default): bound to a specific name and namespace
- `namespace-wide`: can be renamed within the namespace
- `cluster-wide`: can be used in any namespace

---

## CIS Benchmarks for Kubernetes

### What are CIS Benchmarks

CIS (Center for Internet Security) publishes secure configuration guides. The CIS Benchmark for Kubernetes defines controls for hardening the cluster.

### Main categories of the CIS Kubernetes Benchmark

**1. Control Plane Components**
- kube-apiserver configurations
- etcd configurations
- kube-scheduler configurations
- kube-controller-manager configurations

**2. Control Plane Configuration**
- RBAC
- Pod Security Standards
- Network Policies

**3. Worker Nodes**
- kubelet configurations
- Node OS configurations

**4. Policies**
- Images, namespaces, pods

### kube-bench: automating CIS verification

kube-bench automates the CIS Benchmark check:

```bash
# Install kube-bench
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View results
kubectl logs job/kube-bench

# Typical output:
# [PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive
# [FAIL] 1.2.1 Ensure that the --anonymous-auth argument is set to false
# [WARN] 1.2.6 Ensure that the --kubelet-certificate-authority argument is set as appropriate
```

### Most important CIS controls for KCSA

| ID | Control | Priority |
|---|---|---|
| 1.2.1 | `--anonymous-auth=false` on the apiserver | CRITICAL |
| 1.2.6 | TLS between apiserver and kubelet | HIGH |
| 1.2.16 | `--audit-log-path` configured | HIGH |
| 1.2.22 | `--profiling=false` | MEDIUM |
| 2.1 | etcd with TLS | CRITICAL |
| 2.2 | etcd peer with TLS | HIGH |
| 3.2.1 | Audit policy file configured | HIGH |
| 4.2.1 | `--anonymous-auth=false` on kubelet | CRITICAL |
| 5.1.1 | ClusterAdmin used only when necessary | HIGH |
| 5.2 | Pod Security Standards | HIGH |

---

## Audit Logging

### Full audit log configuration

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Do not log requests at the RequestReceived stage
omitStages:
  - "RequestReceived"

rules:
  # Maximum level for secrets and configmaps with sensitive data
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]

  # Log RBAC changes
  - level: Request
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]

  # Log pod changes
  - level: Request
    verbs: ["create", "delete", "update", "patch"]
    resources:
    - group: ""
      resources: ["pods"]

  # Do not log reads of non-sensitive resources
  - level: None
    verbs: ["get", "watch", "list"]
    resources:
    - group: ""
      resources: ["pods", "services", "endpoints"]

  # Default: log metadata for everything else
  - level: Metadata
```

**Analyzing audit logs:**
```bash
# Find actions by a specific user
cat /var/log/kubernetes/audit.log | jq 'select(.user.username == "admin")'

# Find secret accesses
cat /var/log/kubernetes/audit.log | jq 'select(.objectRef.resource == "secrets")'

# Find delete operations
cat /var/log/kubernetes/audit.log | jq 'select(.verb == "delete")'

# Find authentication failures
cat /var/log/kubernetes/audit.log | jq 'select(.responseStatus.code == 401)'
```

---

## NIST and SOC 2 in the Kubernetes Context

### NIST Cybersecurity Framework (CSF)

The NIST CSF organizes security into 5 functions:

```
IDENTIFY → PROTECT → DETECT → RESPOND → RECOVER
    │           │         │         │         │
  Asset      Controls  Monitor  Incidents  BCP/DR
 Inventory   Access    Threats  Playbooks  Backups
```

**Mapping to Kubernetes:**

| NIST Function | Kubernetes Controls |
|---|---|
| **IDENTIFY** | Workload inventory, SBOM, asset discovery |
| **PROTECT** | RBAC, Pod Security, NetworkPolicy, Secrets encryption, TLS |
| **DETECT** | Falco, Audit Logging, anomaly alerts |
| **RESPOND** | Incident playbooks, pod isolation (NetworkPolicy) |
| **RECOVER** | etcd backup, GitOps (re-deploy), DR procedures |

### NIST SP 800-190: Application Container Security Guide

NIST's specific guide for container security:

**5 risk areas identified:**

1. **Image risks**: vulnerabilities, malware, secrets in images
   - Countermeasure: scanning, minimal base images, signing

2. **Registry risks**: unauthorized access to registry, compromised images
   - Countermeasure: registry authentication, scanning, signing

3. **Orchestrator risks**: Kubernetes misconfigurations
   - Countermeasure: RBAC, Pod Security, Network Policies, audit

4. **Container risks**: containers with excessive privileges
   - Countermeasure: SecurityContext, PSA, least privilege

5. **Host OS risks**: compromised nodes
   - Countermeasure: minimal OS images, patch management, hardening

### SOC 2 and Kubernetes

SOC 2 (Service Organization Control 2) is a compliance audit based on 5 Trust Service Criteria (TSC):

| TSC | Kubernetes Application |
|---|---|
| **Security** | RBAC, authentication, encryption, firewall |
| **Availability** | HPA, PDB, multi-zone, etcd backups |
| **Processing Integrity** | Admission controllers, CI/CD gates |
| **Confidentiality** | Encryption at rest, Secrets management, mTLS |
| **Privacy** | PII data access control, audit logs |

**SOC 2 controls mapped to Kubernetes:**

```yaml
# CC6.1 - Logical and Physical Access Controls
# → RBAC with least privilege
# → Pod Security Admission
# → NetworkPolicies

# CC6.3 - Removing stale access
# → Service Account rotation
# → Certificate rotation (kubeadm certs renew)

# CC6.6 - Vulnerability Management
# → Image scanning (Trivy/Grype)
# → Node OS patching
# → Kubernetes version updates

# CC7.1 - Detecting and Monitoring
# → Falco for runtime threats
# → Audit logging
# → Centralized logging (EFK/PLG stack)

# CC7.2 - Anomalies and Events
# → Falco alerts → SIEM
# → AlertManager rules
```

### Relevant NIST SP 800-53 Controls

The most relevant controls from NIST 800-53 for Kubernetes:

| Control | Description | K8s Implementation |
|---|---|---|
| AC-2 | Account Management | RBAC, ServiceAccounts |
| AC-3 | Access Enforcement | RBAC authorization |
| AC-6 | Least Privilege | Roles with minimum permissions |
| AU-2 | Audit Events | kube-apiserver audit logging |
| AU-9 | Protection of Audit Info | Immutable, centralized logs |
| CM-7 | Least Functionality | Pod Security, capabilities drop |
| IA-3 | Device Identification | mTLS, per-component certificates |
| SC-8 | Transmission Confidentiality | TLS between components |
| SC-28 | Protection at Rest | etcd encryption at rest |
| SI-3 | Malicious Code Protection | Image scanning, Falco |
| SI-7 | Software Integrity | Image signing (Cosign) |

---

## Key Takeaways for the KCSA Exam

- Kubernetes Secrets are **Base64, not encrypted** by default
- For real encryption: `EncryptionConfiguration` on the apiserver
- Volume > Env Var for secrets (env vars leak in logs and `ps`)
- Vault: secrets with TTL, automatic rotation, full audit log
- Sealed Secrets: encrypts for safe Git storage (GitOps-friendly)
- CIS Benchmark: hardening guide; kube-bench automates the check
- Audit log levels: `None < Metadata < Request < RequestResponse`
- NIST CSF: Identify → Protect → Detect → Respond → Recover
- NIST SP 800-190: 5 container risk areas
- SOC 2 Trust Criteria: Security, Availability, Processing Integrity, Confidentiality, Privacy
- For KCSA: understand the frameworks at a high level, don't memorize control numbers
- kube-bench automatically verifies CIS Benchmark compliance
- etcd backup = backup of all cluster state (including Secrets)
- Audit logging helps both NIST (AU-2, AU-9) and SOC 2 (CC7.1, CC7.2)
