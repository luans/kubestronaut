# Containers

## What is a Container?

A container is a lightweight, portable, and isolated unit of software that packages an application together with its dependencies (libraries, binaries, config files). Containers share the host OS kernel but are isolated from each other using Linux kernel features.

**Key Linux primitives behind containers:**
- **Namespaces** - isolate what a process can *see* (PID, network, mount, UTS, IPC, user namespaces)
- **cgroups (control groups)** - limit what a process can *use* (CPU, memory, disk I/O)
- **Union filesystems (OverlayFS)** - layer-based image storage enabling copy-on-write

> **Exam tip:** Containers are **not** VMs. They share the host kernel. Isolation is provided by namespaces and cgroups, not hardware virtualization.

## Container Images

A container image is a read-only, layered filesystem snapshot used to create containers. Each instruction in a Dockerfile adds a new layer on top of the previous one.

**Image naming format:**

```
[registry/][namespace/]name[:tag][@digest]

examples:
  nginx:1.25
  docker.io/library/nginx:latest
  gcr.io/myproject/myapp:v2.0.1
```

- **Registry** - where images are stored (Docker Hub, GCR, ECR, GHCR, etc.)
- **Tag** - mutable human-readable label (default: `latest`)
- **Digest** - immutable SHA256 reference to a specific image version (`@sha256:abc...`)

> **Exam tip:** Using `latest` in production is discouraged because it is mutable - the same tag can point to different images over time. Use a specific version tag or digest for reproducibility.

## Dockerfile

A Dockerfile is a text file with instructions to build a container image. Each instruction creates a new image layer.

**Common instructions:**

| Instruction | Purpose |
|---|---|
| `FROM` | Sets the base image |
| `RUN` | Executes a command during the build |
| `COPY` / `ADD` | Copies files into the image |
| `ENV` | Sets environment variables |
| `EXPOSE` | Documents the port the container listens on (informational only) |
| `CMD` | Default command to run when the container starts (overridable) |
| `ENTRYPOINT` | Main executable for the container (harder to override) |
| `WORKDIR` | Sets the working directory inside the container |
| `USER` | Sets the user for subsequent instructions and the container runtime |

**Multi-stage builds** - use multiple `FROM` statements to produce a smaller final image by copying only the compiled artifacts from a build stage:

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM gcr.io/distroless/static
COPY --from=builder /app/myapp /myapp
ENTRYPOINT ["/myapp"]
```

> **Exam tip:** Multi-stage builds are important for producing minimal production images. The final image only contains what is explicitly copied from earlier stages.

## Container Runtimes

The **Container Runtime Interface (CRI)** is a Kubernetes plugin API that allows the kubelet to use different container runtimes without recompilation.

**Runtime layers:**

```
kubelet → CRI (gRPC) → High-level runtime (containerd / CRI-O) → Low-level runtime (runc)
```

| Runtime | Role |
|---|---|
| **containerd** | High-level runtime; default in most Kubernetes distributions |
| **CRI-O** | Lightweight high-level runtime designed specifically for Kubernetes |
| **runc** | Low-level OCI-compliant runtime that actually creates and runs containers |
| **gVisor (runsc)** | Sandboxed runtime with a user-space kernel for stronger isolation |
| **Kata Containers** | Runs containers inside lightweight VMs for hardware-level isolation |

> **Exam tip:** Docker is **not** a supported CRI in Kubernetes (support was removed in v1.24). Kubernetes uses containerd or CRI-O directly. Docker images still work because they follow the OCI standard.

## OCI Standards

The **Open Container Initiative (OCI)** is a Linux Foundation project that defines open standards for containers.

| Specification | Defines |
|---|---|
| **OCI Image Spec** | How container images are built and stored (layers, manifests, config) |
| **OCI Runtime Spec** | How a container is started from an unpacked image filesystem (defines `runc` behavior) |
| **OCI Distribution Spec** | How images are pushed and pulled from registries (HTTP API) |

> **Exam tip:** OCI standards ensure portability - an image built with Docker can run with containerd or CRI-O, and be stored in any OCI-compliant registry.

## Container Security

**Key practices:**

- **Run as non-root** - set `USER` in Dockerfile or `securityContext.runAsNonRoot: true` in Kubernetes
- **Read-only filesystem** - `securityContext.readOnlyRootFilesystem: true` prevents writes to the container filesystem
- **Drop capabilities** - remove Linux capabilities not needed by the app (`securityContext.capabilities.drop: ["ALL"]`)
- **Avoid privileged containers** - `privileged: true` gives the container near-host-level access; avoid unless absolutely necessary
- **Use minimal base images** - distroless or scratch images reduce the attack surface

**securityContext example:**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

> **Exam tip:** Security context can be set at the **Pod level** (applies to all containers) or at the **container level** (overrides the Pod-level setting). Container-level takes precedence.

## Resource Requests and Limits

Resource management ensures containers get the CPU and memory they need without starving other workloads.

| Field | Meaning |
|---|---|
| `requests.cpu` | Minimum CPU guaranteed to the container; used by the scheduler |
| `requests.memory` | Minimum memory guaranteed; used by the scheduler |
| `limits.cpu` | Maximum CPU the container can use (throttled if exceeded) |
| `limits.memory` | Maximum memory the container can use (OOMKilled if exceeded) |

**QoS classes** - Kubernetes assigns a Quality of Service class based on requests/limits:

| QoS Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | requests == limits for all containers | Last to be evicted |
| **Burstable** | requests < limits (or only one set) | Middle priority |
| **BestEffort** | No requests or limits set | First to be evicted |

> **Exam tip:** A container that exceeds its memory **limit** is killed with `OOMKilled`. A container that exceeds its CPU **limit** is **throttled** (not killed). Always set requests so the scheduler can make informed placement decisions.

## Image Pull Policy

Controls when the kubelet pulls a container image from the registry:

| Policy | Behavior |
|---|---|
| `Always` | Always pull from registry (ensures latest tag is up to date) |
| `IfNotPresent` | Pull only if the image is not already on the node (default for versioned tags) |
| `Never` | Never pull; fail if image is not already present on the node |

> **Exam tip:** When the tag is `latest` or no tag is specified, the default pull policy is `Always`. For any other specific tag, the default is `IfNotPresent`.
