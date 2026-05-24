# Day 7 - Review & Practice Tests

## KCSA Exam Domains and Weights

According to the official CNCF curriculum:

| Domain | Approximate Weight |
|---|---|
| Overview of Cloud Native Security | ~14% |
| Kubernetes Cluster Component Security | ~22% |
| Kubernetes Security Fundamentals | ~22% |
| Kubernetes Threat Model | ~16% |
| Platform Security | ~16% |
| Compliance and Security Frameworks | ~10% |

---

## Review Checklist by Domain

### Domain 1: Overview of Cloud Native Security

- [ ] Explain the 4Cs and the dependency relationship between them
- [ ] Describe the Kubernetes attack surface (which components, which ports)
- [ ] Apply STRIDE to Kubernetes scenarios
  - S: pod spoofing, stolen tokens
  - T: image tampering, etcd without encryption
  - R: audit logs for non-repudiation
  - I: exposed Secrets, unencrypted etcd
  - D: pods without ResourceQuota, apiserver flood
  - E: privileged containers, excessive RBAC
- [ ] Describe Defense in Depth with examples per layer
- [ ] Least privilege principle applied to Kubernetes

### Domain 2: Kubernetes Cluster Component Security

- [ ] Critical kube-apiserver flags:
  - `--anonymous-auth=false`
  - `--authorization-mode=Node,RBAC`
  - `--audit-log-path` and `--audit-policy-file`
  - `--encryption-provider-config`
  - `--enable-admission-plugins`
- [ ] etcd encryption:
  - EncryptionConfiguration with aescbc/kms
  - How to verify it works (etcdctl + hexdump)
  - Re-encrypting existing secrets
- [ ] Full RBAC:
  - Role vs ClusterRole
  - RoleBinding vs ClusterRoleBinding
  - ServiceAccount: creation, use, automount
  - `kubectl auth can-i --list --as=...`
- [ ] ServiceAccount tokens:
  - Legacy (< 1.24): Secret, no expiration
  - Modern: projected volume, with expiration
- [ ] `automountServiceAccountToken: false` and when to use it

### Domain 3: Kubernetes Security Fundamentals (Pod & Network)

- [ ] SecurityContext:
  - `runAsNonRoot`, `runAsUser`, `runAsGroup`
  - `readOnlyRootFilesystem`
  - `allowPrivilegeEscalation: false`
  - `capabilities: {drop: [ALL], add: [...]}`
  - `seccompProfile: RuntimeDefault`
- [ ] Pod Security Admission:
  - 3 levels: privileged, baseline, restricted
  - 3 modes: enforce, audit, warn
  - Configuration via namespace labels
- [ ] NetworkPolicy:
  - Default deny all
  - Ingress and egress rules
  - podSelector, namespaceSelector, ipBlock
  - AND vs OR in selectors
  - CNIs that support it: Calico, Cilium, Weave
- [ ] Privileged vs unprivileged containers
- [ ] Risks of hostPID, hostNetwork, hostIPC, hostPath

### Domain 4: Kubernetes Threat Model

- [ ] STRIDE applied to each component
- [ ] Attack vectors:
  - Via compromised container
  - Via supply chain
  - Via control plane
  - Lateral movement
- [ ] CIS Benchmark and kube-bench
- [ ] Auditing with `kubectl auth can-i`

### Domain 5: Platform Security (Supply Chain & Runtime)

- [ ] Trivy vs Grype: differences and use cases
- [ ] SBOM: what it is, Syft to generate, SPDX/CycloneDX formats
- [ ] Cosign:
  - `cosign sign --key` and `cosign verify --key`
  - Keyless signing with OIDC + Rekor
  - Attestations
- [ ] Admission Controllers:
  - Mutating → Validating → etcd
  - Gatekeeper: ConstraintTemplate + Constraint, Rego
  - Kyverno: validate, mutate, generate
  - `validationFailureAction: Enforce vs Audit`
- [ ] Falco:
  - eBPF / kernel module
  - Rules, macros, lists
  - Alert priorities
  - Falcosidekick for integration

### Domain 6: Compliance and Security Frameworks

- [ ] Kubernetes Secrets: limitations, types, secure usage
- [ ] Volume vs env var for secrets
- [ ] Vault: dynamic secrets, agent injector, k8s auth method
- [ ] Sealed Secrets: flow, kubeseal, scopes
- [ ] CIS Benchmark categories and main controls
- [ ] NIST CSF: 5 functions (Identify, Protect, Detect, Respond, Recover)
- [ ] NIST SP 800-190: 5 container risk areas
- [ ] SOC 2: 5 Trust Service Criteria
- [ ] Audit log levels (None, Metadata, Request, RequestResponse)

---

## Practice Test 1 - 30 Questions

### Section A: Cloud Native Security & Threat Model

**Q1.** What is the correct order of the 4Cs of cloud native security, from outermost to innermost?
- A) Code, Container, Cluster, Cloud
- B) Cloud, Code, Container, Cluster
- C) Cloud, Cluster, Container, Code ✓
- D) Container, Cluster, Cloud, Code

**Q2.** An attacker compromises a container and uses the automatically mounted ServiceAccount token to list secrets in the namespace. Which STRIDE category best describes this threat?
- A) Spoofing
- B) Elevation of Privilege ✓
- C) Information Disclosure
- D) Tampering

**Q3.** Which kube-apiserver flag disables anonymous authentication?
- A) `--auth-mode=none`
- B) `--disable-anonymous=true`
- C) `--anonymous-auth=false` ✓
- D) `--unauthenticated-deny=true`

**Q4.** Which EncryptionConfiguration provider is recommended for encrypting Secrets in etcd?
- A) identity
- B) base64
- C) aescbc ✓
- D) md5

**Q5.** You need to verify whether a secret is stored encrypted in etcd. What would indicate that encryption is working when using `etcdctl get`?
- A) The data appears as Base64
- B) The data appears with the prefix `k8s:enc:aescbc:v1:` ✓
- C) The data does not appear (empty return)
- D) The data appears as `ENCRYPTED:true`

---

### Section B: Cluster and Pod Security

**Q6.** Which securityContext setting prevents a process inside the container from gaining more privileges than its parent process?
- A) `runAsNonRoot: true`
- B) `readOnlyRootFilesystem: true`
- C) `allowPrivilegeEscalation: false` ✓
- D) `privileged: false`

**Q7.** A pod with `privileged: true` can do which of the following that a normal pod cannot?
- A) Make external HTTP requests
- B) Mount emptyDir volumes
- C) Load kernel modules from the host ✓
- D) Use more than 1 CPU

**Q8.** Which Pod Security Admission mode allows the pod to be created but shows a warning to the user?
- A) enforce
- B) audit
- C) warn ✓
- D) permissive

**Q9.** How do you configure the "restricted" PSA level on a namespace?
- A) Create a PodSecurityPolicy named "restricted"
- B) Add the label `pod-security.kubernetes.io/enforce: restricted` to the namespace ✓
- C) Set `securityLevel: restricted` in the namespace spec
- D) Create a NetworkPolicy named "restricted"

**Q10.** Which statement about Linux capabilities is CORRECT?
- A) `CAP_SYS_ADMIN` is safe for production containers
- B) `drop: ["ALL"]` removes all capabilities, and `add` adds the ones needed ✓
- C) Capabilities can only be added, not removed
- D) Containers cannot have capabilities without `privileged: true`

---

### Section C: Network Security

**Q11.** Which CNI plugin does NOT support NetworkPolicy?
- A) Calico
- B) Cilium
- C) Flannel ✓
- D) Weave Net

**Q12.** A namespace has a NetworkPolicy with `podSelector: {}` and `policyTypes: [Ingress, Egress]` with no ingress/egress rules. What happens to traffic?
- A) Traffic is free (as if there were no policy)
- B) Only ingress is blocked
- C) All traffic is blocked ✓
- D) Only cross-namespace traffic is blocked

**Q13.** In a NetworkPolicy `from` rule, two selectors in the SAME block (podSelector and namespaceSelector together) represent which logic?
- A) OR - allows from either one
- B) AND - the pod must satisfy both conditions ✓
- C) NOT - excludes what matches
- D) XOR - only one can be true

**Q14.** What is the correct Secret type for storing TLS certificates for an Ingress?
- A) `Opaque`
- B) `kubernetes.io/tls` ✓
- C) `kubernetes.io/dockerconfigjson`
- D) `kubernetes.io/certificate`

**Q15.** What is mTLS and what is its main advantage over standard TLS?
- A) mTLS is faster than TLS
- B) mTLS authenticates both the client and the server ✓
- C) mTLS does not require certificates
- D) mTLS only supports HTTP/2

---

### Section D: Supply Chain & Runtime Security

**Q16.** Which tool is used to sign container images in the Sigstore/CNCF ecosystem?
- A) Notary
- B) Trivy
- C) Cosign ✓
- D) Grype

**Q17.** What is a SBOM (Software Bill of Materials)?
- A) A vulnerability scanner
- B) An inventory of all software components and dependencies ✓
- C) A type of admission controller
- D) A digital signature format

**Q18.** In Gatekeeper (OPA), which object defines the policy logic in Rego?
- A) Constraint
- B) ConstraintTemplate ✓
- C) PolicyRule
- D) AdmissionPolicy

**Q19.** In Kyverno, which `validationFailureAction` blocks the creation of resources that violate the policy?
- A) Audit
- B) Block
- C) Enforce ✓
- D) Deny

**Q20.** Falco uses which Linux kernel mechanism to monitor syscalls?
- A) kprobes only
- B) iptables
- C) eBPF or kernel module ✓
- D) seccomp only

---

### Section E: Secrets, RBAC, and Compliance

**Q21.** Why is using volumes more secure than environment variables for injecting Secrets into pods?
- A) Volumes are automatically encrypted, env vars are not
- B) Env vars are visible via `kubectl exec -- env` and can leak into logs ✓
- C) Volumes are updated in real time, env vars are immutable
- D) There is no security difference

**Q22.** For which ServiceAccount is `automountServiceAccountToken: false` most important?
- A) ServiceAccounts for pods that need to access the Kubernetes API
- B) ServiceAccounts for pods that do NOT need to access the Kubernetes API ✓
- C) Only for the `default` ServiceAccount
- D) Only for ServiceAccounts in kube-system

**Q23.** You want a Role defined in one namespace to be reused across multiple namespaces. What is the correct approach?
- A) Create a ClusterRole and use RoleBindings in each namespace ✓
- B) Create a ClusterRole and a ClusterRoleBinding
- C) Duplicate the Role in each namespace
- D) Use a Role template with the namespace as a variable

**Q24.** kube-bench checks compliance with which standard/framework?
- A) NIST SP 800-53
- B) SOC 2
- C) CIS Benchmark ✓
- D) ISO 27001

**Q25.** What is the main limitation of a native Kubernetes Secret compared to HashiCorp Vault?
- A) Native Secrets do not support different types
- B) Native Secrets are stored as Base64 without real encryption by default ✓
- C) Native Secrets cannot be used in volumes
- D) Native Secrets expire automatically

**Q26.** Which audit logging level captures only the metadata of a request, WITHOUT the body?
- A) None
- B) Metadata ✓
- C) Request
- D) RequestResponse

**Q27.** A container with `hostPath: /etc` mounted has access to what?
- A) Only the /etc directory inside the container's namespace
- B) The /etc files on the host node ✓
- C) An empty volume created at /etc
- D) The cluster's etcd

**Q28.** In the context of the NIST CSF, which function covers using Falco for detecting anomalous behavior?
- A) IDENTIFY
- B) PROTECT
- C) DETECT ✓
- D) RESPOND

**Q29.** What does the annotation `vault.hashicorp.com/agent-inject: "true"` do on a pod?
- A) Enables Vault to encrypt the pod's Secrets
- B) Injects a Vault Agent sidecar that fetches and mounts secrets from Vault ✓
- C) Allows the pod to directly access etcd
- D) Authenticates the pod to Vault via password

**Q30.** What is the main purpose of Sealed Secrets (Bitnami)?
- A) To automatically rotate secrets
- B) To allow encrypted secrets to be safely stored in Git ✓
- C) To replicate secrets across namespaces
- D) To integrate Kubernetes with HashiCorp Vault

---

## Commented Answer Key - Practice Test 1

| Q | A | Topic | Review if wrong |
|---|---|---|---|
| 1 | C | 4Cs | Day 1: Cloud > Cluster > Container > Code |
| 2 | B | STRIDE | Day 1: E = Elevation of Privilege (SA token to escalate) |
| 3 | C | apiserver flags | Day 2: --anonymous-auth=false |
| 4 | C | etcd encryption | Day 2: aescbc recommended (kms = best, but more complex) |
| 5 | B | etcd encryption | Day 2: prefix k8s:enc:aescbc:v1: |
| 6 | C | SecurityContext | Day 3: allowPrivilegeEscalation controls no_new_privs |
| 7 | C | Privileged containers | Day 3: privileged = host access, including kernel modules |
| 8 | C | PSA modes | Day 3: warn = allows but warns |
| 9 | B | PSA configuration | Day 3: label on the namespace |
| 10 | B | Capabilities | Day 3: drop ALL + add what's needed |
| 11 | C | CNI and NetworkPolicy | Day 4: Flannel does not support it |
| 12 | C | NetworkPolicy | Day 4: default deny all |
| 13 | B | NetworkPolicy selectors | Day 4: same block = AND |
| 14 | B | TLS Secret | Day 4: kubernetes.io/tls |
| 15 | B | mTLS | Day 4: mutual authentication |
| 16 | C | Image signing | Day 5: Cosign/Sigstore |
| 17 | B | SBOM | Day 5: inventory of components |
| 18 | B | OPA/Gatekeeper | Day 5: ConstraintTemplate defines Rego |
| 19 | C | Kyverno | Day 5: Enforce blocks, Audit only logs |
| 20 | C | Falco | Day 5: eBPF or kernel module |
| 21 | B | Secrets volumes vs env | Day 6: env leaks via `env` and logs |
| 22 | B | automountServiceAccountToken | Day 2: pods that don't use the API |
| 23 | A | RBAC | Day 2: ClusterRole + RoleBinding per namespace |
| 24 | C | kube-bench | Day 6: CIS Benchmark |
| 25 | B | Vault vs K8s Secrets | Day 6: Base64 ≠ encryption |
| 26 | B | Audit logging | Day 6: Metadata = no body |
| 27 | B | hostPath | Day 3: access to host, not the container |
| 28 | C | NIST CSF | Day 6: Falco = DETECT |
| 29 | B | Vault Agent Injector | Day 6: sidecar injects secrets |
| 30 | B | Sealed Secrets | Day 6: Git-friendly encrypted secrets |

---

## Practice Test 2 - Critical Point Questions

**Q1.** To re-encrypt all existing Secrets after adding EncryptionConfiguration, which command should you use?
```bash
A) kubectl encrypt secrets --all-namespaces
B) kubectl get secrets --all-namespaces -o json | kubectl replace -f -  ✓
C) etcdctl compact --all
D) kubectl rollout restart deployment --all
```

**Q2.** A pod is failing with the error "container has runAsNonRoot and image will run as root". What causes this error?
- A) The pod does not have a SecurityContext defined
- B) `runAsNonRoot: true` is set but the image uses UID 0 ✓
- C) The namespace does not have PSA configured
- D) The container does not have enough capabilities

**Q3.** Which `from` combination in a NetworkPolicy allows traffic ONLY from pods with label `app=frontend` IN the `production` namespace?
```yaml
A) from:
   - podSelector:
       matchLabels:
         app: frontend
   - namespaceSelector:  # ← separate blocks = OR
       matchLabels:
         name: production

B) from:
   - podSelector:
       matchLabels:
         app: frontend
     namespaceSelector:  # ← same block = AND ✓
       matchLabels:
         kubernetes.io/metadata.name: production
```
Answer: **B** ✓

**Q4.** What is the difference between `Role` + `ClusterRoleBinding` and `ClusterRole` + `RoleBinding`?
- A) They are equivalent
- B) `Role` + `ClusterRoleBinding` = cluster-wide permissions; `ClusterRole` + `RoleBinding` = namespace-scoped permissions ✓
- C) `ClusterRole` + `RoleBinding` = cluster-wide permissions
- D) A `Role` cannot be used with a `ClusterRoleBinding`

**Q5.** What happens if you set `automountServiceAccountToken: false` on the ServiceAccount but `automountServiceAccountToken: true` on the Pod?
- A) The pod follows the ServiceAccount configuration (false)
- B) There is a conflict and the pod does not start
- C) The pod follows its own configuration (true) - the token is mounted ✓
- D) The administrator needs to resolve the conflict manually

**Q6.** Which Falco rule priority would indicate a critical threat such as access to the Docker socket?
- A) WARNING
- B) NOTICE
- C) ERROR
- D) CRITICAL ✓

**Q7.** In Kyverno, which type of rule would automatically create a "default-deny" NetworkPolicy whenever a new namespace is created?
- A) validate
- B) mutate
- C) generate ✓
- D) enforce

**Q8.** What is Rekor in the context of Cosign/Sigstore?
- A) A private image registry
- B) A vulnerability scanner
- C) A public, immutable transparency log for signatures ✓
- D) An admission controller

**Q9.** Which of the following configurations is NOT covered by the "baseline" level of Pod Security Admission?
- A) Prohibiting privileged containers
- B) Prohibiting hostNetwork
- C) Requiring runAsNonRoot ✓ (this is "restricted")
- D) Prohibiting hostPID

**Q10.** To satisfy the NIST 800-53 "Transmission Confidentiality" control (SC-8) in Kubernetes, which control is most direct?
- A) Enable PSA at the restricted level
- B) Configure NetworkPolicies
- C) Ensure TLS on all communications between components ✓
- D) Use Sealed Secrets

---

## Final Checklist Before the Exam

### The day before the exam

**Quick review - the 10 most tested items:**

1. **4Cs**: Cloud > Cluster > Container > Code (outside to inside)
2. **kube-apiserver**: `--anonymous-auth=false`, `--authorization-mode=Node,RBAC`
3. **etcd**: Secrets are Base64 without extra encryption. Use `EncryptionConfiguration` to encrypt
4. **PSA**: 3 levels (privileged/baseline/restricted) × 3 modes (enforce/audit/warn) = namespace labels
5. **SecurityContext**: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, `capabilities: drop ALL`
6. **NetworkPolicy**: `podSelector: {}` = all pods; separate blocks = OR; same block = AND
7. **Cosign**: signs images; Trivy: scans for vulnerabilities; Falco: runtime detection
8. **RBAC**: Role/ClusterRole define permissions; RoleBinding/ClusterRoleBinding bind subjects
9. **Secrets**: Base64 ≠ encryption; volume > env var; Vault for production
10. **STRIDE**: S(poofing) T(ampering) R(epudiation) I(nfo Disclosure) D(oS) E(levation)

### During the exam

- KCSA is a **theoretical** exam (multiple choice), not practical like CKA/CKS
- Duration: 90 minutes, ~60 questions
- Passing score: ~75%
- You are allowed to flag questions for review
- If unsure, use process of elimination on obviously wrong answers
- Questions with "NEVER" or "ALWAYS" tend to be false (exceptions exist)
- Read all options before answering

### Resources for extra practice

- **killer.sh** - practice test closest to the real exam (2 sessions included with purchase)
- **Linux Foundation Learning** - official material
- **KodeKloud KCSA** - practice with scenarios
- **A Cloud Guru / Pluralsight** - explanatory videos

---

## Mental Map - Quick Decisions During the Exam

```
"How to encrypt Secrets in etcd?"
→ EncryptionConfiguration + --encryption-provider-config on apiserver
→ Provider: aescbc or kms

"How to block root containers?"
→ runAsNonRoot: true in SecurityContext
→ PSA restricted level

"How to block traffic between pods?"
→ NetworkPolicy with podSelector: {} and policyTypes without rules

"How to see what a ServiceAccount can do?"
→ kubectl auth can-i --list --as=system:serviceaccount:ns:sa-name

"How to scan an image for vulnerabilities?"
→ trivy image name:tag
→ Add --exit-code 1 --severity CRITICAL for CI

"How to sign an image?"
→ cosign sign --key cosign.key registry/image:tag

"How to detect malicious behavior at runtime?"
→ Falco (rules, eBPF/kernel module)

"How to store secrets in Git securely?"
→ Sealed Secrets (kubeseal)

"How to verify CIS compliance?"
→ kube-bench

"Which framework uses Identify→Protect→Detect→Respond→Recover?"
→ NIST CSF
```
