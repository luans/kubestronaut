# Storage

## Volumes

A **Volume** in Kubernetes is a directory accessible to containers in a Pod. Unlike a container's filesystem, volumes have a lifetime tied to the **Pod** (not the container) — data survives container restarts within the same Pod.

### Common Volume Types

| Type | Description |
|---|---|
| `emptyDir` | Empty directory created when Pod starts; deleted when Pod is removed; shared between containers in the Pod |
| `hostPath` | Mounts a file or directory from the **host node's filesystem** into the Pod |
| `configMap` | Mounts ConfigMap data as files |
| `secret` | Mounts Secret data as files (base64-decoded) |
| `persistentVolumeClaim` | Mounts a PersistentVolume claimed by the Pod |
| `nfs` | Network File System mount |
| `projected` | Combines multiple volume sources into a single directory |

```yaml
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: app
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

> **Exam tip:** `emptyDir` is ideal for sharing data between containers in the same Pod (e.g., main app + sidecar). Data is lost when the Pod is deleted.

---

## Persistent Volumes (PV)

A **PersistentVolume** is a piece of storage in the cluster provisioned by an admin or dynamically by a StorageClass. It exists **independently of any Pod** — data persists even after the Pod that used it is deleted.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  reclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data
```

### Access Modes

| Mode | Abbreviation | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted as read-write by a **single node** |
| `ReadOnlyMany` | ROX | Mounted as read-only by **many nodes** |
| `ReadWriteMany` | RWX | Mounted as read-write by **many nodes** |
| `ReadWriteOncePod` | RWOP | Mounted as read-write by a **single Pod** (Kubernetes 1.22+) |

> **Exam tip:** Most cloud block storage (AWS EBS, GCP PD) only supports `ReadWriteOnce`. For shared read-write access across multiple nodes, you need a distributed filesystem (NFS, CephFS, Azure Files).

### Reclaim Policies

| Policy | Behavior when PVC is deleted |
|---|---|
| `Retain` | PV is not deleted; data preserved; requires manual cleanup |
| `Delete` | PV and the underlying storage asset are deleted |
| `Recycle` | *(Deprecated)* Scrubs data and makes PV available again |

---

## PersistentVolumeClaims (PVC)

A **PersistentVolumeClaim** is a request for storage by a user. Kubernetes binds the PVC to a matching PV based on size, access modes, and StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

**Using a PVC in a Pod:**

```yaml
spec:
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
  containers:
  - name: app
    volumeMounts:
    - name: storage
      mountPath: /data
```

### PVC Binding

| PVC state | Description |
|---|---|
| `Pending` | No matching PV found yet |
| `Bound` | Successfully bound to a PV |
| `Lost` | Bound PV no longer exists |

---

## StorageClass

A **StorageClass** enables **dynamic provisioning** — when a PVC is created, Kubernetes automatically provisions a new PV using the StorageClass's provisioner.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs    # or ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**`volumeBindingMode`:**
- `Immediate` — PV is provisioned as soon as PVC is created
- `WaitForFirstConsumer` — delays provisioning until a Pod using the PVC is scheduled (respects topology constraints)

> **Exam tip:** If a PVC does not specify a `storageClassName`, it uses the **default** StorageClass (marked with the annotation `storageclass.kubernetes.io/is-default-class: "true"`). If no default exists and no class is specified, the PVC stays `Pending`.

---

## CSI (Container Storage Interface)

The **Container Storage Interface** is a standard for exposing storage systems to container orchestrators. CSI drivers replace the older in-tree volume plugins.

- All major cloud storage providers offer CSI drivers (AWS EBS CSI, GCP PD CSI, Azure Disk CSI)
- CSI enables dynamic provisioning, snapshots, volume expansion, and cloning

---

## Storage Summary

```
StorageClass → (dynamic provisioning) → PersistentVolume ← bound ← PersistentVolumeClaim ← used by ← Pod
```

| Resource | Role |
|---|---|
| PersistentVolume (PV) | Actual storage resource in the cluster |
| PersistentVolumeClaim (PVC) | User's request for storage |
| StorageClass | Template for dynamic PV provisioning |

## Useful Commands

```bash
kubectl get pv
kubectl get pvc
kubectl get storageclass

kubectl describe pvc my-pvc           # check binding status and events
kubectl get pvc -o wide               # see which PV is bound

# Check default StorageClass
kubectl get storageclass
# The one with (default) next to the name is the default
```
