# Workloads

Workload resources manage Pods for you — they handle replication, rolling updates, scheduling guarantees, and restarts. You describe the desired state and the controller reconciles reality to match it.

## ReplicaSet

Ensures a specified number of Pod replicas are running at any given time. If a Pod dies, the ReplicaSet creates a new one. If there are too many, it deletes the extras.

> **Exam tip:** You rarely create ReplicaSets directly. Deployments manage ReplicaSets for you and add rolling update / rollback capabilities on top.

## Deployment

The standard way to run **stateless** applications. A Deployment manages a ReplicaSet and adds:
- **Rolling updates** — gradually replace old Pods with new ones
- **Rollback** — revert to a previous revision
- **Scaling** — change the number of replicas declaratively

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # max extra Pods above desired count during update
      maxUnavailable: 0  # max Pods that can be unavailable during update
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v2
```

### Update Strategies

| Strategy | Behavior |
|---|---|
| `RollingUpdate` | Gradually replaces old Pods; zero-downtime if configured correctly |
| `Recreate` | Kills all old Pods before creating new ones; causes downtime |

### Rollback

```bash
kubectl rollout history deployment/my-app          # view revision history
kubectl rollout undo deployment/my-app             # rollback to previous revision
kubectl rollout undo deployment/my-app --to-revision=2
kubectl rollout status deployment/my-app           # watch rollout progress
kubectl rollout pause deployment/my-app            # pause a rollout
kubectl rollout resume deployment/my-app           # resume a paused rollout
```

> **Exam tip:** Each `kubectl apply` that changes the Pod template creates a new revision. Changes only to metadata (labels, annotations) outside the template do **not** create a new revision.

## DaemonSet

Ensures that **one Pod runs on every node** (or a subset defined by a node selector). As nodes are added to the cluster, Pods are automatically added to them. As nodes are removed, Pods are garbage collected.

**Use cases:**
- Log collectors (Fluentd, Fluent Bit)
- Monitoring agents (Prometheus node_exporter, Datadog agent)
- Network plugins (CNI agents, kube-proxy)
- Storage daemons (Ceph, GlusterFS)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: fluentbit
        image: fluent/fluent-bit:2.1
```

> **Exam tip:** DaemonSets bypass the normal scheduler for node assignment. They use `nodeName` directly. You can use `nodeSelector` or `affinity` to limit which nodes get the Pod.

## StatefulSet

Manages **stateful** applications where each instance needs a stable identity and/or persistent storage.

**Guarantees provided:**
- **Stable network identity** — each Pod gets a predictable DNS name: `pod-0.service.namespace.svc.cluster.local`
- **Ordered deployment** — Pods are created sequentially (0, 1, 2, ...) and deleted in reverse order
- **Persistent storage** — each Pod gets its own PVC via `volumeClaimTemplates` that survives Pod rescheduling

**Use cases:** databases (MySQL, PostgreSQL, Cassandra), distributed systems (Zookeeper, Kafka, etcd)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"       # must reference a Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

> **Exam tip:** StatefulSets require a **Headless Service** (`clusterIP: None`) to manage the network identity of each Pod. Without it, stable DNS names won't work.

## Job

Runs a Pod (or multiple Pods) **to completion**. When the specified number of successful completions is reached, the Job is done. If a Pod fails, the Job creates a new one.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1       # number of successful completions required
  parallelism: 1       # number of Pods running at the same time
  backoffLimit: 4      # number of retries before marking Job as failed
  template:
    spec:
      restartPolicy: OnFailure   # Jobs must use OnFailure or Never
      containers:
      - name: migrate
        image: migrate-tool:v1
        command: ["./migrate.sh"]
```

> **Exam tip:** Jobs must use `restartPolicy: OnFailure` or `restartPolicy: Never` — never `Always`. Use `parallelism > 1` to run multiple Pods in parallel (work queues).

## CronJob

Runs Jobs on a **cron schedule**. Creates a new Job object at each scheduled time.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"           # every day at 02:00
  concurrencyPolicy: Forbid        # don't start new Job if previous is still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:v1
```

**Concurrency policies:**

| Policy | Behavior |
|---|---|
| `Allow` | Allow concurrent Jobs (default) |
| `Forbid` | Skip new Job if previous is still running |
| `Replace` | Cancel the running Job and start a new one |

> **Exam tip:** CronJob schedules use standard cron syntax: `minute hour day-of-month month day-of-week`. The schedule is in the **cluster's timezone** (UTC by default).

## Workload Summary

| Resource | Use case | Pod identity | Storage |
|---|---|---|---|
| Deployment | Stateless apps | Ephemeral (random names) | Shared or none |
| DaemonSet | Per-node agents | One per node | Usually none |
| StatefulSet | Stateful apps | Stable, ordered | Dedicated PVC per Pod |
| Job | One-off batch tasks | Ephemeral | Usually none |
| CronJob | Scheduled batch tasks | Ephemeral | Usually none |

## Useful Commands

```bash
# Deployments
kubectl create deployment my-app --image=nginx:1.25 --replicas=3
kubectl scale deployment my-app --replicas=5
kubectl set image deployment/my-app app=nginx:1.26
kubectl rollout status deployment/my-app

# DaemonSets
kubectl get daemonset -n kube-system
kubectl rollout restart daemonset/log-collector

# StatefulSets
kubectl get statefulset
kubectl scale statefulset mysql --replicas=5

# Jobs
kubectl create job my-job --image=busybox -- echo "hello"
kubectl get jobs
kubectl logs job/my-job

# CronJobs
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo "tick"
kubectl get cronjobs
```
