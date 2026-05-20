# Cloud Native Architecture

## What is Cloud Native?

Cloud native is an approach to building and running applications that fully exploits the advantages of cloud computing — scalability, resilience, and flexibility. The **Cloud Native Computing Foundation (CNCF)** defines cloud native as:

> Technologies that empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.

**Core principles:**
- **Containerization** — applications packaged as containers
- **Dynamic orchestration** — containers managed by a scheduler (Kubernetes)
- **Microservices** — small, independently deployable services
- **Declarative APIs** — desired state declared, system reconciles to match it
- **Immutable infrastructure** — replace rather than patch

---

## Fundamental Characteristics of Cloud Native Applications

Cloud native applications are defined by four fundamental characteristics:

### Resiliency
The ability to recover from failures and continue functioning. A resilient app expects failure and is designed to handle it gracefully.

- Self-healing: Kubernetes restarts failed Pods, reschedules on healthy nodes
- Redundancy: multiple replicas so a single failure doesn't cause downtime
- Patterns: circuit breakers, retries with backoff, bulkheads, timeouts
- Graceful degradation: partial failure doesn't bring down the whole system

### Agility
The ability to deploy quickly, frequently, and safely — and to respond to change fast.

- Microservices: each service is independently deployable, so teams don't coordinate big-bang releases
- CI/CD pipelines: automated build, test, and deploy on every commit
- Rolling updates and canary deployments: release without downtime
- Small, loosely coupled services reduce the blast radius of any change

### Operability
The app is designed to be easy to deploy, run, and manage in production.

- **Health probes**: `livenessProbe` (is it alive?) and `readinessProbe` (is it ready for traffic?) tell Kubernetes when to restart or route to a Pod
- **Graceful shutdown**: handle `SIGTERM`, finish in-flight requests, then exit — enables zero-downtime rolling updates
- **Externalised config**: config in ConfigMaps/Secrets, not baked into the image (12-factor Config)
- **Automation**: Operators and controllers automate operational tasks (backups, scaling, failover)

### Observability
The ability to understand the internal state of the system from its external outputs — without needing to attach a debugger or SSH into a node.

The **three pillars of observability**:

| Pillar | What it tells you | Kubernetes / CNCF tools |
|---|---|---|
| **Logs** | What happened, event by event | Fluentd, Fluent Bit, Loki |
| **Metrics** | How the system is performing over time | Prometheus, Grafana |
| **Traces** | How a request traveled across services | Jaeger, Zipkin, OpenTelemetry |

> **Exam tip:** The four characteristics — **Resiliency, Agility, Operability, Observability** — are the KCNA-tested framework for what makes an application truly cloud native. The other concepts in this file (microservices, 12-factor, immutable infrastructure) are *enablers* of these four properties.

---

## Microservices

Microservices is an architectural style where an application is built as a collection of **small, independent services**, each:
- Responsible for a single business capability
- Independently deployable and scalable
- Communicating over a network (HTTP/REST, gRPC, message queues)
- Owning its own data store

**Contrast with monolith:**

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment | Deploy entire app | Deploy individual services |
| Scaling | Scale everything | Scale only bottleneck services |
| Fault isolation | One bug can crash all | Failures are contained |
| Technology | One stack | Each service can use different stack |
| Complexity | Simple initially | Distributed system complexity |

> **Exam tip:** Microservices introduce distributed system challenges: network latency, partial failure, distributed tracing, and data consistency. These are solved by patterns like circuit breakers, retries, and service meshes.

---

## The 12-Factor App

A methodology for building cloud-native applications. Key factors relevant to Kubernetes:

| Factor | Principle | Kubernetes connection |
|---|---|---|
| **Config** | Store config in the environment | ConfigMaps and Secrets |
| **Backing services** | Treat backing services as attached resources | Services and DNS |
| **Build, release, run** | Strictly separate build and run stages | Container images + CI/CD |
| **Processes** | Execute as stateless processes | Pods (stateless by default) |
| **Port binding** | Export services via port binding | ContainerPort + Service |
| **Concurrency** | Scale out via the process model | HPA + replicas |
| **Disposability** | Fast startup and graceful shutdown | Probes + preStop hooks |
| **Logs** | Treat logs as event streams | stdout/stderr → log aggregator |

---

## Continuous Integration and Continuous Deployment

### Continuous Integration (CI)
Developers merge code changes into a shared branch frequently — multiple times a day. Each merge triggers an **automated pipeline** that builds and tests the code immediately.

**Goal:** detect integration problems early, before they compound.

**Typical CI pipeline:**
```
Code push → Build → Unit tests → Integration tests → Static analysis → Container image build → Push to registry
```

### Continuous Delivery vs Continuous Deployment

These terms are often confused — the distinction matters for the exam:

| | Continuous Delivery | Continuous Deployment |
|---|---|---|
| **Definition** | Every change is automatically tested and made *ready* to deploy | Every change that passes tests is *automatically deployed* to production |
| **Human gate** | A human approves the production release | No human gate — fully automated |
| **Risk** | Lower — humans decide when to ship | Higher — requires strong test coverage and observability |
| **Common in** | Regulated industries, large enterprises | High-velocity teams, SaaS products |

> **Exam tip:** Continuous Delivery ≠ Continuous Deployment. Delivery means *releasable at any time*. Deployment means *released automatically*. All Continuous Deployment pipelines practice Continuous Delivery, but not vice versa.

### How CI/CD Enables Cloud Native Agility

CI/CD is the primary enabler of the **Agility** characteristic:
- Microservices can be deployed independently — each service has its own pipeline
- Short feedback loops: a bug introduced at 9am is caught and fixed by 9:05am
- Rolling updates and canary releases make frequent deploys safe in Kubernetes
- GitOps (covered in [15-gitops-cicd.md](15-gitops-cicd.md)) extends CD by making Git the source of truth for the cluster state

### Key Deployment Strategies

| Strategy | How it works | Risk |
|---|---|---|
| **Recreate** | Terminate all old Pods, then start new ones | Downtime during switch |
| **Rolling update** | Replace Pods gradually, old and new run briefly in parallel | Default in Kubernetes |
| **Blue/Green** | Run two identical environments; switch traffic at once | Zero downtime; doubles resource cost briefly |
| **Canary** | Route a small % of traffic to new version; increase if stable | Lowest risk; requires traffic splitting |

---

## Serverless

Serverless is a cloud execution model where:
- The provider manages infrastructure automatically
- Code runs in response to events
- Billing is per-execution (not per-idle server)
- Scale to zero is supported

**In Kubernetes**, serverless is implemented by frameworks like:
- **Knative** — CNCF project; runs event-driven workloads; scales to zero; builds on Kubernetes primitives
- **OpenFaaS** — Functions as a Service on Kubernetes
- **KEDA** — event-driven autoscaling that scales to zero

> **Exam tip:** Serverless does **not** mean "no servers" — it means the developer doesn't manage servers. Kubernetes-based serverless still runs on nodes.

---

## CNCF Landscape and Project Maturity

The **CNCF** hosts and maintains open source cloud-native projects across the ecosystem.

### Project Maturity Levels

| Level | Description | Examples |
|---|---|---|
| **Sandbox** | Early stage; experimental | Many new projects |
| **Incubating** | Growing adoption; stable API | Argo, KEDA, OpenTelemetry (at various times) |
| **Graduated** | Proven; production-ready; high adoption | Kubernetes, Prometheus, Envoy, Fluentd, Jaeger, Vitess, etcd |

> **Exam tip:** Kubernetes, Prometheus, Envoy, CoreDNS, containerd, Fluentd, Jaeger, and Vitess are all **CNCF graduated** projects. Knowing the graduation status of key projects is tested in KCNA.

### Key CNCF Project Categories

| Category | Projects |
|---|---|
| Container runtime | containerd, CRI-O |
| Orchestration | Kubernetes |
| Monitoring | Prometheus, Grafana, Thanos |
| Logging | Fluentd, Fluent Bit, Loki |
| Tracing | Jaeger, Zipkin |
| Service mesh | Istio, Linkerd, Consul |
| Networking | Cilium, Calico, Flannel |
| Storage | Rook, Longhorn |
| Security | Falco, OPA, cert-manager |
| GitOps / CD | Argo CD, Flux |
| API gateway | Envoy, Contour |
| Observability | OpenTelemetry |

---

## Immutable Infrastructure

Instead of updating running servers/containers, you **replace** them with new ones. Benefits:
- No configuration drift
- Easy rollback (deploy previous image version)
- Consistent environments (same image in dev, staging, prod)

In Kubernetes: update a Deployment's image tag → rolling update creates new Pods → old Pods are terminated.

---

## Declarative vs Imperative

| Approach | Description | Kubernetes example |
|---|---|---|
| **Imperative** | You specify **how** to get there step by step | `kubectl run`, `kubectl create`, `kubectl scale` |
| **Declarative** | You specify **what** the desired end state is | `kubectl apply -f deployment.yaml` |

Kubernetes is fundamentally **declarative** — you describe desired state in YAML, and controllers reconcile current state to match. This enables GitOps and self-healing.

---

## Open Standards

Cloud native relies on open standards to ensure interoperability:

| Standard | Governs |
|---|---|
| **OCI** (Open Container Initiative) | Container image format and runtime |
| **CNI** (Container Network Interface) | Network plugin integration |
| **CSI** (Container Storage Interface) | Storage plugin integration |
| **CRI** (Container Runtime Interface) | Container runtime integration with kubelet |
| **SMI** (Service Mesh Interface) | Common API for service meshes |
| **OpenTelemetry** | Metrics, logs, and traces telemetry |

> **Exam tip:** These interfaces decouple Kubernetes from specific implementations — you can swap CNI plugins, storage backends, or container runtimes without changing the core system.
