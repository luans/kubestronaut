# Autoscaling

Kubernetes provides multiple layers of autoscaling to handle varying workload demands automatically.

## Horizontal Pod Autoscaler (HPA)

The **HPA** automatically scales the **number of Pod replicas** in a Deployment, ReplicaSet, or StatefulSet based on observed metrics.

**Default metric:** CPU utilization (percentage of the container's CPU request).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # scale up when avg CPU > 70% of request
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
```

**How it works:**
```
metrics-server → HPA controller → adjusts replica count → Deployment
```

> **Exam tip:** HPA requires **metrics-server** to be installed. CPU-based HPA only works if containers have **CPU requests** defined - without requests, there is no baseline to calculate utilization percentage.

### Imperative creation:

```bash
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70
kubectl get hpa
kubectl describe hpa my-app-hpa
```

---

## Vertical Pod Autoscaler (VPA)

The **VPA** automatically adjusts **CPU and memory requests/limits** for containers based on historical usage. Instead of adding more replicas, it makes each Pod more (or less) resourceful.

**Modes:**

| Mode | Behavior |
|---|---|
| `Off` | Only provides recommendations; no changes applied |
| `Initial` | Sets requests only at Pod creation time |
| `Auto` | Automatically updates requests; may restart Pods to apply changes |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
```

> **Exam tip:** VPA and HPA **should not** be used together on the same CPU/memory metrics - they can conflict. VPA is not installed by default; it requires a separate installation.

---

## Cluster Autoscaler

The **Cluster Autoscaler** adjusts the **number of nodes** in a cluster:
- **Scale up** - when Pods are `Pending` due to insufficient node resources
- **Scale down** - when nodes have been underutilized for a period and their Pods can be rescheduled elsewhere

Cluster Autoscaler is cloud-provider-specific (AWS, GCP, Azure) and integrates with the cloud's node group/autoscaling group APIs.

> **Exam tip:** Cluster Autoscaler will not scale down a node if it has Pods that cannot be evicted (e.g., Pods with `PodDisruptionBudget`, local storage, or system Pods).

---

## KEDA (Kubernetes Event-Driven Autoscaling)

**KEDA** is a CNCF project that extends HPA to scale based on **external event sources** beyond CPU/memory:
- Message queues (Kafka, RabbitMQ, SQS, Azure Service Bus)
- Database query results
- HTTP request rate
- Cron schedules

KEDA can scale deployments all the way down to **zero replicas** (unlike standard HPA which requires at least 1).

---

## PodDisruptionBudget (PDB)

A **PodDisruptionBudget** limits the number of Pods that can be **voluntarily disrupted** at the same time (e.g., during node drains, rolling updates, or Cluster Autoscaler scale-downs).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2           # at least 2 Pods must always be available
  # maxUnavailable: 1       # OR: at most 1 Pod can be unavailable at a time
  selector:
    matchLabels:
      app: my-app
```

> **Exam tip:** PDB only applies to **voluntary** disruptions (drains, evictions). It does not protect against node failures (involuntary disruptions).

---

## Autoscaling Summary

| Mechanism | What it scales | Trigger |
|---|---|---|
| HPA | Number of Pod replicas | CPU, memory, custom metrics |
| VPA | CPU/memory requests per Pod | Historical resource usage |
| Cluster Autoscaler | Number of nodes | Pending Pods / underutilized nodes |
| KEDA | Number of Pod replicas (to 0) | External event sources |
