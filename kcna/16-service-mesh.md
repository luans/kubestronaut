# Service Mesh

## What is a Service Mesh?

A **service mesh** is a dedicated infrastructure layer that handles **service-to-service communication** in a microservices architecture. It moves networking logic (retries, timeouts, circuit breaking, mTLS, observability) out of application code and into the infrastructure.

**Problems it solves:**
- How do services discover and communicate with each other reliably?
- How do we encrypt all internal traffic (mTLS) without changing application code?
- How do we get visibility into latency, errors, and traffic between services?
- How do we control traffic (canary deployments, A/B testing, circuit breaking)?

## Sidecar Proxy Pattern

Most service meshes use a **sidecar proxy** injected into each Pod alongside the application container. All network traffic to/from the application is intercepted and managed by the proxy - transparently, without code changes.

```
Pod
├── app container      ← your application
└── proxy (Envoy)      ← handles all network traffic in/out
```

The **data plane** consists of all the sidecar proxies.
The **control plane** configures and manages the proxies centrally.

```
Control Plane (Istiod / Linkerd control plane)
        ↓ configuration push
Data Plane (Envoy sidecars in each Pod)
```

## Key Features

| Feature | Description |
|---|---|
| **mTLS** | Mutual TLS - automatically encrypts and authenticates all service-to-service traffic |
| **Traffic management** | Load balancing, retries, timeouts, circuit breaking, fault injection |
| **Observability** | Automatic metrics (golden signals), distributed traces, and access logs for all traffic |
| **Traffic splitting** | Route a percentage of traffic to different versions (canary releases, A/B testing) |
| **Authorization policies** | Fine-grained L7 access control between services |

## Popular Service Meshes

| Mesh | Proxy | Control plane | Notes |
|---|---|---|---|
| **Istio** | Envoy | Istiod | Most feature-rich; CNCF graduated |
| **Linkerd** | Linkerd2-proxy (Rust) | Linkerd control plane | Lightweight; CNCF graduated; simpler than Istio |
| **Consul Connect** | Envoy | Consul | HashiCorp; multi-platform |
| **Kuma** | Envoy | Kuma control plane | CNCF; supports VMs and Kubernetes |

> **Exam tip:** Both **Istio** and **Linkerd** are **CNCF graduated** projects. Linkerd is known for being lightweight and easier to operate. Istio is more feature-rich but more complex.

## Envoy Proxy

**Envoy** is the most widely used data plane proxy in cloud native networking. It is a **CNCF graduated** project used by Istio, AWS App Mesh, Kuma, and others.

**Capabilities:**
- L3/L4/L7 load balancing
- HTTP/2 and gRPC support
- Observability (metrics, traces, access logs)
- TLS termination and origination
- Circuit breaking and outlier detection
- Dynamic configuration via **xDS API**

## SMI (Service Mesh Interface)

**SMI** is a specification that defines a common API for service meshes on Kubernetes. It allows tooling to work with multiple service mesh implementations without being tightly coupled to any one.

SMI APIs include:
- `TrafficSplit` - traffic splitting between services
- `TrafficTarget` - access control policies
- `HTTPRouteGroup` - HTTP traffic routing rules
- `TrafficMetrics` - standard metrics interface

## When to Use a Service Mesh

A service mesh adds operational complexity. Consider it when you need:
- Automatic **mTLS** between all services
- Detailed **observability** (request-level metrics and traces) without code changes
- Advanced **traffic control** (canary, circuit breaking) across many services
- Consistent **policy enforcement** across a large microservices fleet

> **Exam tip:** For KCNA, understand *what* a service mesh does and *why* it's used more than the specific configuration of any one mesh. Key concepts: sidecar proxy, data plane, control plane, mTLS, and traffic management.
