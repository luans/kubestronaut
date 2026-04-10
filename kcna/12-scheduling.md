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
