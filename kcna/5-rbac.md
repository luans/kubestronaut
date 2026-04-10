# RBAC (Role-Based Access Control)

RBAC is the authorization mechanism used by Kubernetes to control who can do what on which resources. It is enabled by default since Kubernetes 1.8 and is the recommended approach for access control.

> **Key point:** Kubernetes does **not manage** user accounts or groups. Instead, it expects certificates to be created and signed by a trusted Certificate Authority (CA). Users and groups are represented by fields in those certificates, not as objects in the cluster.

---

## Authentication vs Authorization

Kubernetes processes every API request through three stages:

```
Request → Authentication → Authorization (RBAC) → Admission Controllers → etcd
```

| Stage | Question answered | Mechanism |
|---|---|---|
| **Authentication** | Who are you? | Certificates, tokens, OIDC, basic auth |
| **Authorization** | Are you allowed to do this? | RBAC, ABAC, Node, Webhook |
| **Admission Control** | Should this request be allowed/mutated? | Built-in and custom admission webhooks |

> **Exam tip:** Authentication proves identity; authorization decides permissions. RBAC is the standard authorization mode in Kubernetes.

---

## Identities in Kubernetes

### Users
Individuals or applications that interact with the cluster — admins, developers, or automated systems. They are **managed externally** to Kubernetes; inside the cluster they are represented only by a string (e.g., `alice` or `alice@example.com`).

> In certificates, the user is identified by the **CN (Common Name)** field.

### Groups
Also managed outside Kubernetes. A group aggregates multiple users and lets you assign a set of permissions at once — all members of the group automatically inherit those permissions.

> In certificates, the group is identified by the **O (Organisation)** field.

> **Exam tip:** Kubernetes has built-in groups: `system:masters` grants cluster-admin level access, `system:authenticated` matches any authenticated user, and `system:unauthenticated` matches unauthenticated requests.

### ServiceAccounts
Used by **applications running inside the cluster** (not by humans). Unlike users and groups, ServiceAccounts are **Kubernetes objects** managed by the cluster itself. They grant the necessary permissions for a Pod to interact with the Kubernetes API.

- Every namespace has a `default` ServiceAccount created automatically
- Pods are assigned the `default` ServiceAccount of their namespace unless specified otherwise
- The ServiceAccount token is automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`

> **Exam tip:** To prevent a Pod from having any API access, set `automountServiceAccountToken: false` in the Pod spec or on the ServiceAccount itself.

---

## RBAC Objects

### Role
Defines a set of permissions (rules) scoped to a **specific namespace**. A Role only grants access — RBAC is purely additive; there are no "deny" rules.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]        # "" means the core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### RoleBinding
Associates a Role with one or more subjects (User, Group, or ServiceAccount) **within a namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole
A **non-namespaced** resource — applies cluster-wide and can define permissions over:
- Resources in all namespaces (e.g., `pods`, `deployments`)
- Non-namespaced resources (e.g., `nodes`, `persistentvolumes`, `namespaces`)
- Non-resource URLs (e.g., `/healthz`, `/metrics`)

### ClusterRoleBinding
Associates a ClusterRole with a subject with **cluster-wide** scope.

> **Exam tip:** A **RoleBinding** can reference a **ClusterRole** — this is a common pattern to define permissions once in a ClusterRole and reuse them in specific namespaces via RoleBindings. The binding scope (namespace vs cluster) always wins.

---

## Verbs and Resources

RBAC rules are composed of **resources** and **verbs**:

| Verb | HTTP method | Description |
|---|---|---|
| `get` | GET | Retrieve a single resource |
| `list` | GET | List resources |
| `watch` | GET (streaming) | Watch for changes |
| `create` | POST | Create a new resource |
| `update` | PUT | Replace a resource |
| `patch` | PATCH | Partially modify a resource |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete a collection of resources |

Resources can be further scoped to **subresources**:

```yaml
resources: ["pods", "pods/log", "pods/exec"]
```

And to specific **resource names**:

```yaml
resources: ["configmaps"]
resourceNames: ["my-config"]   # only this specific ConfigMap
```

---

## Quick Reference

| Object | Scope | Description |
|---|---|---|
| Role | Namespace | Defines permissions within a namespace |
| RoleBinding | Namespace | Binds a Role to an identity within a namespace |
| ClusterRole | Cluster | Defines permissions cluster-wide |
| ClusterRoleBinding | Cluster | Binds a ClusterRole to an identity cluster-wide |

| Identity | Managed by | Certificate field | Primary use case |
|---|---|---|---|
| User | External (certs/OIDC) | CN (Common Name) | Humans or external systems accessing the cluster |
| Group | External (certs/OIDC) | O (Organisation) | Set of users sharing permissions |
| ServiceAccount | Kubernetes | — | Pods that need to call the Kubernetes API |

---

## Aggregated ClusterRoles

ClusterRoles can be composed using **aggregation rules** — labels select other ClusterRoles and their rules are merged automatically. This is how `admin`, `edit`, and `view` built-in roles work.

```yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
```

> **Exam tip:** The built-in ClusterRoles `cluster-admin`, `admin`, `edit`, and `view` are available in every cluster. `cluster-admin` grants unrestricted access to everything.

---

## Checking Permissions

```bash
# Check if the current user can perform an action
kubectl auth can-i create pods
kubectl auth can-i delete deployments -n production

# Check permissions for another user or ServiceAccount
kubectl auth can-i list secrets --as=alice
kubectl auth can-i list secrets --as=system:serviceaccount:default:my-sa

# List all permissions the current user has
kubectl auth can-i --list
kubectl auth can-i --list -n kube-system
```

---

## Useful Commands

```bash
# List all ClusterRoleBindings with details
kubectl get clusterrolebindings -o wide

# List Roles and RoleBindings in a namespace
kubectl get roles,rolebindings -n default

# Describe a role to see its rules
kubectl describe clusterrole cluster-admin

# Create a Role imperatively
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default

# Create a RoleBinding imperatively
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=alice \
  -n default

# Create a ClusterRole with full access
kubectl create clusterrole cluster-superhero --verb='*' --resource='*'

# Create a ClusterRoleBinding for a group
kubectl create clusterrolebinding cluster-superhero \
  --clusterrole=cluster-superhero \
  --group=cluster-superheroes

# Bind a ClusterRole to a ServiceAccount
kubectl create rolebinding dev-access \
  --clusterrole=edit \
  --serviceaccount=default:my-sa \
  -n development
```

> **Exam tip:** `--verb='*'` and `--resource='*'` grant full access — equivalent to a superuser. In real environments, always apply the principle of **least privilege**. Use `kubectl auth can-i` to verify effective permissions before and after applying RBAC changes.
