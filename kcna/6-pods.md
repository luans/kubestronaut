# Pods

## What is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It represents one or more containers that share the same network namespace, IPC namespace, and optionally storage volumes. Containers inside a Pod communicate with each other via `localhost`.

> **Exam tip:** You almost never create Pods directly in production. You use higher-level workload resources (Deployments, StatefulSets, etc.) that manage Pods for you. A standalone Pod is called a **static Pod** or a **naked Pod** - if it dies, nothing recreates it.

## Pod Lifecycle

```
Pending → Running → Succeeded / Failed
                 ↘ Unknown
```

| Phase | Description |
|---|---|
| `Pending` | Pod accepted by the cluster but containers not yet running (scheduling, image pull) |
| `Running` | At least one container is running |
| `Succeeded` | All containers exited with code 0 and will not restart |
| `Failed` | At least one container exited with a non-zero code and will not restart |
| `Unknown` | Pod state cannot be determined (usually a node communication problem) |

## Container States

Each container within a Pod also has its own state:

| State | Description |
|---|---|
| `Waiting` | Not yet running - pulling image, applying secrets, etc. |
| `Running` | Process is executing |
| `Terminated` | Process finished or was killed; has an exit code |

## Restart Policies

Controls what happens when a container exits:

| Policy | Behavior |
|---|---|
| `Always` | Always restart (default) - used for long-running apps |
| `OnFailure` | Restart only if exit code is non-zero - used for Jobs |
| `Never` | Never restart |

> **Exam tip:** `restartPolicy` applies to all containers in the Pod, not individual containers.

## Pod Spec - Key Fields

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    env:
    - name: ENV_VAR
      value: "hello"
  initContainers:
  - name: init-check
    image: busybox
    command: ["sh", "-c", "until nslookup db; do sleep 2; done"]
  restartPolicy: Always
```

## Init Containers

Init containers run **before** the main application containers start. They must complete successfully for the Pod to proceed. Use cases:
- Wait for a dependency (database, service) to be ready
- Pre-populate a volume with config files
- Register the Pod with an external system

**Properties:**
- Run sequentially - each must succeed before the next starts
- Share volumes with app containers but have a separate image and environment
- Do not support liveness/readiness probes (they must exit 0 to be considered done)

## Sidecar Containers

Sidecars are additional containers in the same Pod that augment the main application without modifying it. Common patterns:
- **Log shipping** - sidecar reads log files from a shared volume and sends to a logging backend
- **Proxy/service mesh** - Envoy sidecar handles all network traffic (used by Istio)
- **Secret injection** - sidecar fetches secrets from a vault and writes them to a shared volume

## Multi-container Pod Patterns

| Pattern | Description |
|---|---|
| **Sidecar** | Augments the main container (logging, proxying) |
| **Ambassador** | Proxies network traffic on behalf of the main container |
| **Adapter** | Transforms the main container's output for external consumers |

## Static Pods

Static Pods are managed directly by the **kubelet** on a node, not by the API server. The kubelet watches a directory (usually `/etc/kubernetes/manifests/`) for Pod manifest files.

- Control plane components (kube-apiserver, etcd, scheduler, controller-manager) run as static Pods in kubeadm clusters
- They appear in `kubectl get pods -n kube-system` as mirror Pods but cannot be deleted via kubectl

> **Exam tip:** Static Pods are tied to a specific node. If the node dies, those Pods are gone. The kubelet auto-restarts static Pods if they crash.

## Useful Commands

```bash
# Create a Pod
kubectl run nginx --image=nginx:1.25

# Get Pod details
kubectl get pods -o wide
kubectl describe pod my-pod

# Execute a command inside a container
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -c sidecar -- sh   # specific container

# View logs
kubectl logs my-pod
kubectl logs my-pod -c init-check          # init container logs
kubectl logs my-pod --previous             # previous crashed container

# Delete a Pod
kubectl delete pod my-pod

# Generate a Pod manifest without applying it
kubectl run nginx --image=nginx --dry-run=client -o yaml
```
