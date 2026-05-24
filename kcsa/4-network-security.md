# Day 4 - Network Security

## NetworkPolicies: Ingress and Egress

By default, all pods in a Kubernetes cluster can communicate with all other pods, in any namespace. NetworkPolicies allow you to restrict this traffic.

### Core concept

**Without NetworkPolicy:** traffic is free between all pods.
**With NetworkPolicy:** a pod selected by the policy has its traffic restricted - **only what is explicitly allowed is accepted**.

**Important:** NetworkPolicy is implemented by the CNI plugin. Not all CNIs support NetworkPolicy. Plugins that do: Calico, Cilium, Weave Net, Antrea. **Flannel does NOT support it**.

### NetworkPolicy structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: production
spec:
  podSelector:         # Which pods this policy applies to
    matchLabels:
      app: backend
  policyTypes:         # Which traffic types to control
  - Ingress            # Traffic ENTERING the selected pods
  - Egress             # Traffic LEAVING the selected pods
  ingress:
  - from:              # Allow traffic FROM:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:                # Allow traffic TO:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
```

### Ingress (incoming traffic)

```yaml
# Allow only traffic from the "monitoring" namespace on port 9090
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```

**Source selectors (ingress from):**

```yaml
ingress:
- from:
  # 1. By pod in the same namespace
  - podSelector:
      matchLabels:
        role: frontend

  # 2. By namespace
  - namespaceSelector:
      matchLabels:
        environment: production

  # 3. Combination: pod AND namespace (logical AND)
  - namespaceSelector:
      matchLabels:
        environment: staging
    podSelector:           # Same block = AND
      matchLabels:
        app: frontend

  # 4. By CIDR (external IP)
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.1.0/24       # Exclude subnet
```

**CAREFUL with OR vs AND:**
```yaml
# This is OR: pod OR namespace
from:
- podSelector: {...}      # Rule 1 (separate block)
- namespaceSelector: {...} # Rule 2 (separate block)

# This is AND: pod AND namespace (same block)
from:
- podSelector: {...}
  namespaceSelector: {...}
```

### Egress (outgoing traffic)

```yaml
# Allow only DNS and database access
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # DNS (required for name resolution)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### "Default deny all" policy (network zero trust)

```yaml
# Block ALL traffic in the namespace by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # {} = selects ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# Block only ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

# Block only egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Example: Three-tier architecture with NetworkPolicy

```yaml
# Frontend can receive from the external world
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}  # Allow all (traffic from ingress controller arrives via IP)
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080

---
# Backend only receives from frontend, only talks to the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - ports:    # DNS
    - protocol: UDP
      port: 53

---
# Database only receives from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

---

## TLS Between Cluster Components

Kubernetes uses TLS for communication between all its components. Understanding the cluster PKI is fundamental.

### Kubernetes CA hierarchy

```
Cluster CA (ca.crt / ca.key)
├── kube-apiserver (apiserver.crt)
├── kube-apiserver-kubelet-client (apiserver-kubelet-client.crt)
├── kube-controller-manager (controller-manager.crt)
├── kube-scheduler (scheduler.crt)
├── admin (admin.crt) - for kubectl
└── kubelet (kubelet.crt - per node)

etcd CA (etcd/ca.crt) - separate CA
├── etcd server (etcd/server.crt)
├── etcd peer (etcd/peer.crt)
└── etcd client (apiserver-etcd-client.crt)

Front Proxy CA (front-proxy-ca.crt)
└── front-proxy-client (front-proxy-client.crt)
```

### Certificates and their locations

```bash
# kubeadm default: /etc/kubernetes/pki/
ls /etc/kubernetes/pki/
# apiserver.crt           apiserver.key
# apiserver-etcd-client.crt  apiserver-etcd-client.key
# apiserver-kubelet-client.crt  apiserver-kubelet-client.key
# ca.crt                  ca.key
# etcd/
#   ca.crt  ca.key  server.crt  server.key  peer.crt  peer.key
# front-proxy-ca.crt      front-proxy-ca.key
# front-proxy-client.crt  front-proxy-client.key
# sa.key                  sa.pub  (service account signing)
```

### Inspecting certificates

```bash
# Check certificate expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 Validity

# With kubeadm
kubeadm certs check-expiration

# Renew certificates
kubeadm certs renew all
```

### How kubelet uses TLS

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false    # Disable anonymous authentication
  webhook:
    enabled: true     # Use apiserver for authentication
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook       # Authorization delegated to the apiserver
tlsCertFile: /var/lib/kubelet/pki/kubelet.crt
tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.key
```

---

## Secure Ingress with TLS Termination

### TLS Termination at the Ingress

The Ingress Controller (nginx, traefik, etc.) terminates TLS and forwards traffic internally.

```yaml
# Secret with the TLS certificate
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <BASE64_CERT>
  tls.key: <BASE64_KEY>

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretRef: tls-secret      # References the Secret with the certificate
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Creating a TLS certificate with cert-manager

```yaml
# ClusterIssuer with Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx

---
# Certificate (automatically managed)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: production
spec:
  secretName: tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
```

### TLS best practices for Ingress

```yaml
annotations:
  # Force HTTPS (redirect HTTP → HTTPS)
  nginx.ingress.kubernetes.io/ssl-redirect: "true"

  # Security headers
  nginx.ingress.kubernetes.io/configuration-snippet: |
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

  # Minimum TLS version
  nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"

  # Strong ciphers
  nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
```

---

## Service Mesh and mTLS (Concepts)

### What is a Service Mesh

A Service Mesh is a dedicated infrastructure layer for controlling service-to-service communication. Most common service meshes: **Istio**, **Linkerd**, **Consul Connect**.

### How it works: Sidecar Proxy

```
┌─────────────────────────────────────────────────────┐
│  Pod A                        Pod B                 │
│  ┌─────────────┐              ┌─────────────┐       │
│  │ App         │  mTLS/TLS    │ App         │       │
│  │ Container   │◄────────────►│ Container   │       │
│  ├─────────────┤              ├─────────────┤       │
│  │ Sidecar     │              │ Sidecar     │       │
│  │ (Envoy)     │◄────────────►│ (Envoy)     │       │
│  └─────────────┘              └─────────────┘       │
└─────────────────────────────────────────────────────┘
        ↑                              ↑
   Intercepts traffic            Intercepts traffic
   automatically                 automatically
```

The sidecar intercepts all pod traffic, automatically applies TLS, and manages authentication/authorization.

### mTLS (Mutual TLS)

In standard TLS, only the server authenticates to the client (server certificate).
In mTLS, **both** parties authenticate each other.

```
Standard TLS:
  Client ──── verifies server cert ──── Server
  Client ←─── encrypted data ──────── Server

mTLS:
  Client ──── verifies server cert ──── Server
  Server ──── verifies client cert ───── Client
  Client ←─── encrypted data ──────── Server
```

**In the Service Mesh context:**
- The mesh issues certificates for each service (SPIFFE/SVID)
- Sidecars negotiate mTLS automatically
- The application does not need to know mTLS exists

### SPIFFE and SVID

**SPIFFE (Secure Production Identity Framework For Everyone):**
- Open standard for workload identities
- Each workload receives a unique identifier: SPIFFE URI
- Format: `spiffe://<trust-domain>/<path>`
- Example: `spiffe://cluster.local/ns/production/sa/payment-service`

**SVID (SPIFFE Verifiable Identity Document):**
- Concrete implementation of the SPIFFE identity
- Usually an X.509 certificate with the SPIFFE URI in the SAN field

### Security policies with Istio

```yaml
# PeerAuthentication: require mTLS in a namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # STRICT = only mTLS allowed
                  # PERMISSIVE = accepts mTLS and plaintext

---
# AuthorizationPolicy: L7 access control
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/production/sa/frontend-sa
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

### Service Mesh vs NetworkPolicy

| Aspect | NetworkPolicy | Service Mesh |
|---|---|---|
| Layer | L3/L4 (IP/port) | L7 (HTTP, gRPC) |
| mTLS | No | Yes (automatic) |
| Observability | No | Yes (metrics, traces) |
| Complexity | Low | High |
| Overhead | Minimal | Significant (sidecars) |
| Granularity | IP/port | HTTP method, path, headers |
| Identity | By IP | By SPIFFE identity |

**KCSA recommendation:** understand the concepts, not the detailed implementation. Know that a service mesh provides automatic mTLS, observability, and L7 control.

---

## Key Takeaways for the KCSA Exam

- By default: no NetworkPolicy = free traffic between all pods
- NetworkPolicy is implemented by the CNI - Flannel does NOT support it
- `podSelector: {}` = selects ALL pods in the namespace
- Two items in the same `from` block = logical AND; separate blocks = OR
- Always include a DNS rule (UDP/TCP 53) in egress policies
- Default deny: `policyTypes: [Ingress, Egress]` with `podSelector: {}`
- TLS termination in Ingress: Secret of type `kubernetes.io/tls`
- Mandatory HTTPS: annotation `ssl-redirect: "true"`
- mTLS = both client and server authenticate with certificates
- SPIFFE: standardized identity for workloads - URI `spiffe://trust-domain/path`
- Service mesh (Istio/Linkerd) automates mTLS via sidecar proxy
- `PeerAuthentication: STRICT` in Istio = mandatory mTLS
- NetworkPolicy operates at L3/L4; Service Mesh operates at L7
- Cluster certificates: `/etc/kubernetes/pki/` (kubeadm)
- Check expiration: `kubeadm certs check-expiration`
