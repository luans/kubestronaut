# Networking

## Kubernetes Networking Model

Kubernetes enforces a flat networking model with three rules:
1. Every Pod gets its own IP address
2. All Pods can communicate with all other Pods **without NAT**
3. Nodes can communicate with all Pods (and vice versa) without NAT

This model simplifies service discovery and removes the need for port mapping between Pods.

## CNI (Container Network Interface)

The **Container Network Interface** is a CNCF standard that defines how network plugins integrate with container runtimes. The kubelet calls the CNI plugin when a Pod is created or deleted to set up or tear down its network interface.

**Popular CNI plugins:**

| Plugin | Features |
|---|---|
| **Flannel** | Simple overlay network; limited NetworkPolicy support |
| **Calico** | BGP-based networking; full NetworkPolicy support; high performance |
| **Cilium** | eBPF-based; NetworkPolicy + Layer 7 policies; observability |
| **Weave Net** | Mesh overlay; built-in encryption |
| **AWS VPC CNI** | Native AWS VPC IPs for Pods; used in EKS |

> **Exam tip:** Kubernetes does not include a CNI plugin by default. You must install one or the cluster will not schedule Pods (they will stay `Pending`). `kubeadm` clusters require a CNI plugin to be installed after `kubeadm init`.

## DNS in Kubernetes

CoreDNS is deployed as a Deployment in the `kube-system` namespace and serves as the cluster DNS server.

**DNS records created automatically:**

| Resource | DNS name |
|---|---|
| Service | `<service>.<namespace>.svc.cluster.local` |
| Pod | `<pod-ip-dashes>.<namespace>.pod.cluster.local` |
| Headless Service Pod | `<pod-name>.<service>.<namespace>.svc.cluster.local` |

From within a Pod, you can use:
- `my-service` — same namespace
- `my-service.other-namespace` — cross-namespace
- `my-service.other-namespace.svc.cluster.local` — fully qualified

The DNS search path inside Pods is configured via `/etc/resolv.conf` automatically.

## kube-proxy

**kube-proxy** runs on every node and maintains network rules (iptables or IPVS) that implement the Service abstraction — forwarding traffic from a Service's ClusterIP to one of its backing Pods.

**Modes:**

| Mode | Description |
|---|---|
| `iptables` | Default; uses iptables rules; scales to thousands of services |
| `ipvs` | Uses Linux IPVS (kernel-level LB); better performance at scale |
| `userspace` | Legacy; not used in modern clusters |

> **Exam tip:** With Cilium's eBPF mode, kube-proxy can be replaced entirely. But in standard clusters, kube-proxy is always present.

## NetworkPolicy

A **NetworkPolicy** is a Kubernetes resource that controls **ingress and egress traffic** for Pods using label selectors. Without any NetworkPolicy, all Pods can talk to all other Pods freely.

**Important:** NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium, Weave). Flannel does not support NetworkPolicy.

### Default behavior:
- No NetworkPolicy selected → **all traffic allowed**
- At least one NetworkPolicy selects a Pod → **only explicitly allowed traffic is permitted** (default deny for the selected policy types)

### Example — deny all ingress, allow only from frontend:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend           # this policy applies to backend Pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend      # only allow ingress from frontend Pods
    ports:
    - protocol: TCP
      port: 8080
```

### Selectors in NetworkPolicy:

| Selector | Matches |
|---|---|
| `podSelector` | Pods in the same namespace |
| `namespaceSelector` | All Pods in matching namespaces |
| `ipBlock` | CIDR ranges (for external traffic) |

> **Exam tip:** NetworkPolicy is **additive** — multiple policies selecting the same Pod are unioned together. There are no explicit deny rules; you restrict by limiting what is allowed.

### Default deny all (ingress and egress):

```yaml
spec:
  podSelector: {}         # selects all Pods in the namespace
  policyTypes:
  - Ingress
  - Egress
  # no ingress/egress rules = deny everything
```

## Pod-to-Pod Communication Flow

```
Pod A (node 1) → veth pair → bridge (cbr0/cni0) → CNI overlay/routing → bridge (node 2) → veth pair → Pod B (node 2)
```

## Useful Commands

```bash
# Inspect DNS resolution from inside a Pod
kubectl run -it --rm debug --image=busybox -- nslookup my-service

# View NetworkPolicies
kubectl get networkpolicy
kubectl describe networkpolicy allow-frontend

# Check kube-proxy mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```
