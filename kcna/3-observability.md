# Observability

Observability is the ability to understand the internal state of a system from the outside by examining its outputs. In Kubernetes, observability is built on three pillars: **metrics**, **logs**, and **traces**.

## The Three Pillars

| Pillar | What it answers | Example tools |
|---|---|---|
| Metrics | Is the system healthy? What are the trends? | Prometheus, Grafana |
| Logs | What happened and when? | Fluentd, Loki, ELK stack |
| Traces | Where did this request spend its time? | Jaeger, Zipkin, OpenTelemetry |

## Prometheus

Prometheus is the de-facto standard monitoring system for Kubernetes. It uses a **pull model** — it scrapes metrics from targets at regular intervals over HTTP.

**Architecture:**

```
Targets (apps / exporters) ← scrape ← Prometheus Server → Alertmanager → notifications
                                              ↓
                                           Grafana (visualization)
```

**Key concepts:**
- **Scrape** — Prometheus periodically pulls metrics from an HTTP endpoint (default: `/metrics`)
- **Exporter** — a sidecar or standalone process that exposes metrics in Prometheus format (e.g., `node_exporter` for host metrics, `kube-state-metrics` for Kubernetes object state)
- **Alertmanager** — receives alerts fired by Prometheus and routes them (email, Slack, PagerDuty, etc.)
- **PromQL** — Prometheus Query Language, used to query and aggregate time-series data

### The 4 Metric Types

- **Counter**
  A metric that only increases (or resets to zero on restart). Good for counting events like requests served, jobs completed, or errors occurred.

- **Gauge**
  A metric that can go up and down. Used for current values like memory usage, temperature, or number of active connections.

- **Histogram**
  Tracks the distribution of observed values in configurable buckets (e.g., request duration). Exposes `_bucket`, `_sum`, and `_count` series. Useful for computing percentile approximations server-side.

- **Summary**
  Also tracks value distribution with client-side quantile calculation, plus `_sum` and `_count`. Quantiles are computed on the client and cannot be aggregated across instances.

> **Exam tip:** Prefer **Histogram** over Summary when you need to aggregate across multiple instances (e.g., across replicas). Summary quantiles are pre-computed on the client side and cannot be combined later.

## Kubernetes Probes

Probes allow the kubelet to check the health of containers. There are three types:

### Liveness Probe
Determines if the container is **alive**. If it fails, the kubelet kills the container and restarts it (according to the `restartPolicy`).

Use it for: detecting deadlocks or hung processes that cannot recover on their own.

### Readiness Probe
Determines if the container is **ready to serve traffic**. If it fails, the Pod's IP is removed from the Service Endpoints — no traffic is sent to it.

Use it for: waiting for the app to fully start, or temporarily removing it from rotation during heavy load or dependency unavailability.

### Startup Probe
Determines if the container **application has started**. While it is failing, liveness and readiness probes are disabled. Once it succeeds, liveness and readiness take over.

Use it for: slow-starting containers that would otherwise be killed by liveness probes before they finish booting.

### Probe Execution Order

```
Container starts
      ↓
Startup Probe (if defined)
  ├── keeps failing → kubelet restarts the container
  └── succeeds (or not defined) ↓
      ├── Liveness Probe  → failure = container restart
      └── Readiness Probe → failure = removed from Service endpoints
```

All three probes run **concurrently** once the startup probe succeeds — liveness and readiness are independent of each other after that point.

> **Exam tip:** If no startup probe is defined, liveness and readiness probes start immediately when the container starts. This can cause fast-failing liveness probes to kill a slow-starting container before it ever becomes ready — that is exactly the problem the startup probe solves.

### Probe mechanisms:

| Mechanism | How it works |
|---|---|
| `httpGet` | HTTP GET request; success if status code is 2xx or 3xx |
| `tcpSocket` | TCP connection attempt; success if the port is open |
| `exec` | Runs a command inside the container; success if exit code is 0 |
| `grpc` | gRPC health check protocol |

> **Exam tip:** A failing **liveness** probe causes a container restart. A failing **readiness** probe only removes the Pod from Service endpoints — the container keeps running.

## Logging in Kubernetes

Kubernetes does not provide a native cluster-wide logging solution. Logs are written to stdout/stderr and managed at the node level.

**Log access:**
```bash
kubectl logs <pod>                  # current container logs
kubectl logs <pod> -c <container>   # specific container in a multi-container Pod
kubectl logs <pod> --previous       # logs from a previously crashed container
```

**Logging architectures:**

| Pattern | Description |
|---|---|
| Node-level logging agent | A DaemonSet (e.g., Fluentd, Fluent Bit) runs on every node and ships logs to a backend |
| Sidecar container | A logging sidecar reads log files from a shared volume and streams them to a backend |
| Direct push | Application sends logs directly to a logging backend |

> **Exam tip:** Container logs are lost when a Pod is deleted unless you have a cluster-level logging solution in place. The `--previous` flag is essential for debugging crashed containers.

## kube-state-metrics vs. metrics-server

These two components are often confused:

| Component | What it exposes | Use case |
|---|---|---|
| **metrics-server** | Real-time CPU and memory usage from the kubelet | `kubectl top`, HorizontalPodAutoscaler |
| **kube-state-metrics** | State of Kubernetes objects (desired replicas, Pod phase, etc.) from the API server | Prometheus scraping, alerting on object state |

> **Exam tip:** `kubectl top` requires **metrics-server** to be installed. kube-state-metrics is required if you want Prometheus to alert on things like "Deployment has fewer ready replicas than desired."

## OpenTelemetry

OpenTelemetry (OTel) is the CNCF standard for collecting and exporting telemetry data (metrics, logs, and traces) in a vendor-neutral way.

**Key components:**
- **SDK** — instrumentation libraries for applications
- **Collector** — receives, processes, and exports telemetry to backends (Jaeger, Prometheus, etc.)
- **OTLP** — OpenTelemetry Protocol, the standard wire format for telemetry data

> **Exam tip:** OpenTelemetry is a CNCF **graduated** project and represents the convergence of OpenCensus and OpenTracing. It is the recommended standard for new observability instrumentation.
