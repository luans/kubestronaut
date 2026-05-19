# Scheduling

## How the Scheduler Works

The `kube-scheduler` watches for Pods with no assigned node and selects the best node for each Pod through two phases:

1. **Filtering** — eliminates nodes that do not meet the Pod's requirements
2. **Scoring** — ranks remaining nodes; the highest-scoring node wins

## nodeSelector

The simplest way to constrain a Pod to specific nodes — requires an exact label match.

```yaml
spec:
  nodeSelector:
    disktype: ssd
    region: us-east-1
```

Nodes must have `disktype=ssd` and `region=us-east-1` labels to be eligible.

```bash
# Add a label to a node
kubectl label node node-1 disktype=ssd
```

## Node Affinity

More expressive than `nodeSelector` — supports operators (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`) and two enforcement modes.

| Type | Behavior |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | **Hard** requirement — Pod won't schedule if no node matches |
| `preferredDuringSchedulingIgnoredDuringExecution` | **Soft** preference — scheduler tries to match but will place elsewhere if needed |

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd", "nvme"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: region
            operator: In
            values: ["us-east-1"]
```

> **Exam tip:** `IgnoredDuringExecution` means the rule is only evaluated at scheduling time — if a node's labels change after the Pod is scheduled, the Pod is **not** evicted.

## Pod Affinity and Anti-Affinity

Controls Pod placement **relative to other Pods** (not nodes). Useful for co-locating related Pods or spreading replicas.

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: kubernetes.io/hostname   # "same node" as cache Pods
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname  # spread replicas across nodes
```

> **Exam tip:** `topologyKey` defines the domain of co-location. `kubernetes.io/hostname` means "same node". `topology.kubernetes.io/zone` means "same availability zone".

## Taints and Tolerations

**Taints** are applied to **nodes** to repel Pods. **Tolerations** are applied to **Pods** to allow them to be scheduled on tainted nodes.

```bash
# Add a taint to a node
kubectl taint node node-1 key=value:effect

# Remove a taint
kubectl taint node node-1 key=value:effect-
```

### Taint Effects

| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without the toleration will **not** be scheduled on the node |
| `PreferNoSchedule` | Scheduler tries to avoid placing Pods without the toleration |
| `NoExecute` | New Pods won't be scheduled **and** existing Pods without the toleration are **evicted** |

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # tolerate for 300s before eviction
```

> **Exam tip:** Control plane nodes have a `node-role.kubernetes.io/control-plane:NoSchedule` taint by default — this is why regular workloads don't land on master nodes. System Pods (CoreDNS, kube-proxy) have matching tolerations.

### Toleration Operators

A toleration uses one of two `operator` values to match a taint:

| Operator | Behavior |
|---|---|
| `Equal` | `key`, `value`, and `effect` must all match the taint exactly (default) |
| `Exists` | Only `key` (and optionally `effect`) must match — `value` is ignored |

```yaml
spec:
  tolerations:
  # Equal: matches only dedicated=gpu:NoSchedule
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

  # Exists: matches any taint with key "dedicated", regardless of value
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"

  # Empty key + Exists: matches ALL taints on the node (super-toleration)
  - operator: "Exists"
```

> **Exam tip:** An empty `key` with `operator: Exists` tolerates every taint on the node. This is used by system components that must run anywhere (e.g., `kube-proxy`).

### Multiple Taints — How They Interact

When a node has **multiple taints**, Kubernetes evaluates each one independently. A Pod must tolerate **all** taints or face the strictest untolerated effect.

**Example:**
- Node taints: `env=prod:NoSchedule` + `hardware=gpu:NoExecute`
- Pod tolerates only `env=prod:NoSchedule`
- Result: the `NoExecute` taint is not tolerated → existing Pods are **evicted**, new Pods are **not scheduled**

The rule: *the most severe untolerated effect wins.*

### Taints Repel — Tolerations Don't Attract

A common misunderstanding: tolerations **allow** a Pod to be placed on a tainted node, but they do **not guarantee** it will land there. The scheduler can still place the Pod on any other untainted node.

For a **dedicated node** pattern (e.g., reserving nodes exclusively for GPU workloads), you need **both**:
1. A taint on the node — to keep non-GPU Pods off
2. Node affinity on the Pod — to pull GPU Pods toward those nodes

```yaml
# Node: kubectl taint node gpu-node-1 hardware=gpu:NoSchedule

# Pod spec: toleration + node affinity together
spec:
  tolerations:
  - key: "hardware"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: hardware
            operator: In
            values: ["gpu"]
```

### Node Condition Auto-Taints

Kubernetes automatically adds taints to nodes when certain conditions are detected. You don't need to add these manually — the node lifecycle controller manages them.

| Taint (key) | Effect | Triggered by |
|---|---|---|
| `node.kubernetes.io/not-ready` | `NoExecute` | Node `Ready` condition is `False` |
| `node.kubernetes.io/unreachable` | `NoExecute` | Node `Ready` condition is `Unknown` |
| `node.kubernetes.io/memory-pressure` | `NoSchedule` | Node reports memory pressure |
| `node.kubernetes.io/disk-pressure` | `NoSchedule` | Node reports disk pressure |
| `node.kubernetes.io/pid-pressure` | `NoSchedule` | Node reports PID pressure |
| `node.kubernetes.io/unschedulable` | `NoSchedule` | Node is cordoned (`kubectl cordon`) |
| `node.kubernetes.io/network-unavailable` | `NoSchedule` | Node reports no network route |

> **Exam tip:** DaemonSet Pods automatically receive `NoExecute` tolerations for `not-ready` and `unreachable` (with no `tolerationSeconds`). This ensures DaemonSet Pods stay running even on degraded nodes — they are designed to run on every node regardless of health.

### `tolerationSeconds` — Delayed Eviction

When using `NoExecute`, you can delay eviction with `tolerationSeconds`. The Pod is allowed to keep running on the node for that many seconds after the taint appears, then evicted.

```yaml
spec:
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300   # stay 5 minutes before eviction
```

Omitting `tolerationSeconds` on a `NoExecute` toleration means the Pod is **never evicted** due to that taint.

### Real-World Use Cases

**Dedicated GPU nodes** — reserve nodes exclusively for GPU workloads:
```bash
kubectl taint node gpu-node-1 hardware=gpu:NoSchedule
kubectl label node gpu-node-1 hardware=gpu
```
GPU Pods get the toleration + node affinity (shown above); all other Pods are blocked.

**Spot / preemptible nodes** — allow only batch/fault-tolerant workloads:
```bash
kubectl taint node spot-node-1 cloud.google.com/gke-spot=true:NoSchedule
```
Batch jobs tolerate this taint; stateful services do not.

**Soft maintenance drain** — temporarily discourage scheduling without a full `kubectl drain`:
```bash
kubectl taint node node-1 maintenance=true:NoSchedule
# Later, when done:
kubectl taint node node-1 maintenance=true:NoSchedule-
```

## Topology Spread Constraints

Controls how Pods are **spread** across topology domains (nodes, zones, regions). More flexible than pod anti-affinity for even distribution.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                              # max difference in Pod count between domains
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule        # or ScheduleAnyway
    labelSelector:
      matchLabels:
        app: my-app
```

## Priority and Preemption

**PriorityClasses** assign a priority value to Pods. Higher-priority Pods can **preempt** (evict) lower-priority Pods if the cluster is under resource pressure.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical workloads"
```

```yaml
spec:
  priorityClassName: high-priority
```

> **Exam tip:** Built-in system priority classes: `system-cluster-critical` and `system-node-critical` are used by core system Pods. Never assign these to regular workloads.

## nodeName

The most direct way to assign a Pod to a specific node — bypasses the scheduler entirely.

```yaml
spec:
  nodeName: node-1
```

> **Exam tip:** `nodeName` skips scheduling (no filtering, no scoring). If the node doesn't exist or can't run the Pod, it stays `Pending` with no retry from the scheduler.

## Scheduling Summary

| Mechanism | Applied to | Purpose |
|---|---|---|
| `nodeSelector` | Pod | Simple label-based node constraint |
| Node Affinity | Pod | Expressive node constraint with operators |
| Pod Affinity | Pod | Co-locate with other Pods |
| Pod Anti-Affinity | Pod | Spread away from other Pods |
| Taints | Node | Repel Pods from a node |
| Tolerations | Pod | Allow scheduling on tainted nodes |
| Topology Spread | Pod | Even distribution across topology domains |
| `nodeName` | Pod | Bypass scheduler, assign directly |

## Useful Commands

```bash
# View node labels
kubectl get nodes --show-labels
kubectl label node node-1 disktype=ssd

# View taints
kubectl describe node node-1 | grep Taint

# Add / remove taints
kubectl taint node node-1 dedicated=gpu:NoSchedule
kubectl taint node node-1 dedicated=gpu:NoSchedule-

# Check why a Pod is not scheduling
kubectl describe pod my-pod | grep -A 10 Events
```
