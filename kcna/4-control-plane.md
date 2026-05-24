# Control Plane

## kube-apiserver

The **central entry point** of the cluster - all access to the Kubernetes API goes through it, whether via `kubectl`, other control plane components, or internal applications.

**Main responsibilities:**
- Exposes the **Kubernetes REST API** (resources like Pods, Deployments, Services, etc.)
- Authenticates and authorizes each request (integrates with RBAC, certificates, tokens)
- Validates and persists object state in **etcd** - it is the only component that reads and writes directly to etcd
- Serves as a communication hub: `kube-scheduler`, `kube-controller-manager`, and `kubelets` on nodes communicate exclusively through the kube-apiserver

**Request flow:**

```
Client (kubectl / app) → Authentication → Authorization (RBAC) → Admission Controllers → etcd
```

> **Exam tip:** The kube-apiserver is the only component that communicates with etcd. All other control plane components query and update cluster state **indirectly**, through the API.

## Admission Controllers

Admission Controllers are plugins that intercept API requests **after** authentication and authorization, but **before** the object is persisted to etcd. They can **validate**, **mutate**, or **reject** requests.

**Updated request flow:**

```
Client → Authentication → Authorization (RBAC) → Mutating Admission → Validating Admission → etcd
```

### Two types of Admission Controllers

| Type | What it does |
|---|---|
| **Mutating** | Can modify the request object (e.g., inject a sidecar, set default values) |
| **Validating** | Can only approve or reject the request - cannot modify it |

Mutating controllers always run **before** validating controllers, so validators see the final modified object.

### Key built-in Admission Controllers

| Controller | Purpose |
|---|---|
| `NamespaceLifecycle` | Rejects requests to create resources in non-existent or terminating namespaces |
| `LimitRanger` | Enforces LimitRange defaults and constraints on Pods and containers |
| `ResourceQuota` | Enforces ResourceQuota limits in a namespace |
| `ServiceAccount` | Automatically mounts the default ServiceAccount token into Pods |
| `DefaultStorageClass` | Assigns the default StorageClass to PVCs that don't specify one |
| `MutatingAdmissionWebhook` | Calls external webhooks to mutate objects |
| `ValidatingAdmissionWebhook` | Calls external webhooks to validate objects |
| `PodSecurity` | Enforces Pod Security Standards (replaced the deprecated PodSecurityPolicy) |

### Admission Webhooks

Webhooks allow you to extend admission control with **custom logic** without modifying Kubernetes itself. The API server calls an external HTTP endpoint you control.

- **MutatingAdmissionWebhook** - your webhook receives the object and can return a modified version (e.g., inject an Envoy sidecar, add default labels)
- **ValidatingAdmissionWebhook** - your webhook inspects the object and returns allow or deny (e.g., enforce naming conventions, block images without a digest)

```
API request → kube-apiserver → calls webhook endpoint (your service) → allow / deny / mutate
```

> **Exam tip:** Admission controllers are the mechanism behind many Kubernetes features you use daily - LimitRange defaults, ResourceQuota enforcement, and service mesh sidecar injection all rely on admission controllers. If a validating webhook is unavailable and its `failurePolicy` is `Fail`, requests will be rejected - this is a common cause of cluster issues during upgrades.

## etcd

etcd is a distributed key-value store used by Kubernetes for reliable configuration data and state storage. It is the cluster's source of truth and provides strong consistency, high availability, and data durability.

A common deployment uses 3 etcd members (replicas) because etcd relies on the Raft consensus algorithm. Raft requires a majority of nodes (quorum) to be healthy and reachable to make progress. In a 3-member cluster, losing one node still leaves a majority (2/3) available; losing two nodes causes loss of quorum and cluster unavailability. For this reason, etcd clusters should use an odd number of members (3, 5, 7, ...), so you can tolerate failures while maintaining quorum.

> **Exam tip:** etcd stores all cluster state - if you lose etcd without a backup, you lose the entire cluster configuration. Regular etcd backups (`etcdctl snapshot save`) are a critical operational practice.

## kube-scheduler

The **kube-scheduler** is responsible for watching for newly created Pods that have no Node assigned, and selecting the best Node for them to run on.

**How scheduling works:**

1. **Filtering** - eliminates Nodes that do not satisfy the Pod's requirements (e.g., insufficient CPU/memory, missing labels, taints without matching tolerations)
2. **Scoring** - ranks the remaining Nodes using priority functions (e.g., spread workloads evenly, prefer Nodes with the image already pulled)
3. **Binding** - assigns the Pod to the highest-scoring Node by writing the `nodeName` field via the API server

**Factors considered during scheduling:**
- Resource requests and limits (`requests.cpu`, `requests.memory`)
- Node selectors and affinity/anti-affinity rules
- Taints and tolerations
- Pod topology spread constraints
- Available ports and volume requirements

> **Exam tip:** The scheduler does **not** run Pods - it only decides **where** they should run. The kubelet on the chosen Node is responsible for actually starting the container. If no Node satisfies the constraints, the Pod stays in `Pending` state.

## kube-controller-manager

The **kube-controller-manager** runs a collection of controllers - each one is a control loop that watches the current state of the cluster and works to drive it toward the desired state.

**Key built-in controllers:**

| Controller | Responsibility |
|---|---|
| Node Controller | Monitors Node health; marks nodes as `NotReady` and evicts Pods when nodes become unreachable |
| Replication Controller | Ensures the correct number of Pod replicas are running |
| Endpoints Controller | Populates the Endpoints object that links Services to Pods |
| Service Account Controller | Creates default ServiceAccounts in new namespaces |
| Job Controller | Tracks Jobs to completion and creates Pods as needed |
| Namespace Controller | Handles cleanup when namespaces are deleted |

**Control loop pattern:**

```
Watch (observe current state) → Compare (diff vs desired state) → Act (reconcile)
```

All controllers communicate exclusively through the kube-apiserver - they never write to etcd directly.

> **Exam tip:** Each controller runs as a separate goroutine inside a single binary (`kube-controller-manager`). This design means a single process manages the entire lifecycle reconciliation of most built-in Kubernetes resources.

## cloud-controller-manager

The **cloud-controller-manager** integrates Kubernetes with the underlying cloud provider infrastructure. It was separated from `kube-controller-manager` to allow cloud-specific logic to evolve independently of the core Kubernetes codebase.

**Cloud-specific controllers it runs:**

| Controller | Responsibility |
|---|---|
| Node Controller | Checks the cloud provider to determine if a node has been deleted after it stops responding |
| Route Controller | Configures routes in the cloud infrastructure so Pods on different nodes can communicate |
| Service Controller | Creates, updates, and deletes cloud load balancers for `LoadBalancer`-type Services |

**Key points:**
- Only runs if you are using a supported cloud provider (AWS, GCP, Azure, etc.)
- On bare-metal or local clusters (e.g., kubeadm without a cloud), this component is not present
- Cloud providers can implement the cloud controller interface to support their own infrastructure

> **Exam tip:** The separation of `cloud-controller-manager` from `kube-controller-manager` follows the principle of keeping cloud-provider-specific code out of the core Kubernetes release cycle. This is why `LoadBalancer` Services work differently on cloud clusters vs. bare-metal clusters.
