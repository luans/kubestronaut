# Services and Ingress

## Why Services?

Pods are ephemeral - they get new IP addresses every time they restart or are rescheduled. A **Service** provides a stable network endpoint (IP + DNS name) that load-balances traffic to a dynamic set of Pods selected by labels.

## How Services Work

Services use **label selectors** to find target Pods. The kube-proxy (or an eBPF equivalent) on each node programs the network rules (iptables/ipvs) to forward traffic from the Service's ClusterIP to one of the backing Pods.

```
Client → Service (stable ClusterIP:port) → kube-proxy rules → Pod (dynamic IP:port)
```

The **Endpoints** (or **EndpointSlices**) object is automatically populated with the IPs of Pods that match the selector and pass their readiness probe.

## Service Types

### ClusterIP (default)
Exposes the Service on a **cluster-internal IP**. Only reachable from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80          # port on the Service
    targetPort: 8080  # port on the Pod
```

### NodePort
Exposes the Service on a **static port on every node's IP**. Accessible from outside the cluster via `<NodeIP>:<NodePort>`.

- NodePort range: **30000–32767**
- Also creates a ClusterIP automatically

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # optional; auto-assigned if omitted
```

### LoadBalancer
Provisions an **external load balancer** from the cloud provider (AWS ELB, GCP LB, Azure LB). The external IP is assigned by the cloud.

- Superset of NodePort - also creates a NodePort and ClusterIP
- Only works on cloud clusters or with a bare-metal LB solution (MetalLB)

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
```

### ExternalName
Maps the Service to an **external DNS name** - no proxying, just a CNAME record.

```yaml
spec:
  type: ExternalName
  externalName: database.external.com
```

> **Exam tip:** Service type hierarchy: `ClusterIP ⊂ NodePort ⊂ LoadBalancer`. Each builds on the previous. `ExternalName` is the odd one out - it does not select Pods.

## Headless Services

A Service with `clusterIP: None`. No load balancing - DNS returns the IPs of all matching Pods directly. Required by StatefulSets to give each Pod a stable DNS name.

```yaml
spec:
  clusterIP: None
  selector:
    app: mysql
```

DNS for a headless service Pod: `pod-name.service-name.namespace.svc.cluster.local`

## Service DNS

Kubernetes DNS (CoreDNS) automatically creates DNS records for every Service:

```
<service-name>.<namespace>.svc.cluster.local
```

From within the same namespace, you can use just the service name. From another namespace, use `<service>.<namespace>`.

> **Exam tip:** CoreDNS is the cluster DNS server deployed as a Deployment in `kube-system`. It resolves Service names and Pod names (for headless Services).

## Service Summary

| Type | Accessible from | Use case |
|---|---|---|
| ClusterIP | Inside cluster only | Internal microservice communication |
| NodePort | Outside cluster via node IP | Dev/testing, bare-metal without LB |
| LoadBalancer | Outside cluster via cloud LB | Production external access on cloud |
| ExternalName | Inside cluster only | Alias for external service |

---

## Ingress

An **Ingress** is an API object that manages **external HTTP/HTTPS access** to Services inside the cluster. It provides:
- **Host-based routing** - route `app.example.com` to one Service, `api.example.com` to another
- **Path-based routing** - route `/api` to one Service, `/web` to another
- **TLS termination** - handle HTTPS at the ingress layer

> **Exam tip:** Ingress is not a Service type. An Ingress requires an **Ingress Controller** to be running in the cluster (Nginx, Traefik, HAProxy, AWS ALB, etc.). The Ingress object alone does nothing without a controller.

### Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret      # Secret with tls.crt and tls.key
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Path Types

| pathType | Behavior |
|---|---|
| `Exact` | Matches exactly the specified path |
| `Prefix` | Matches paths that start with the specified prefix |
| `ImplementationSpecific` | Matching is up to the Ingress controller |

## Useful Commands

```bash
# Services
kubectl get services
kubectl get svc -o wide
kubectl describe service my-service
kubectl expose deployment my-app --port=80 --target-port=8080 --type=ClusterIP

# Endpoints
kubectl get endpoints my-service   # see which Pod IPs back the service

# Ingress
kubectl get ingress
kubectl describe ingress my-ingress
```
