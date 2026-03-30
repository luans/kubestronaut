# Control Plane

## kube-apiserver

O **ponto de entrada central** do cluster — todo acesso à API do Kubernetes passa por ele, seja via `kubectl`, outros componentes do control plane, ou aplicações internas.

**Responsabilidades principais:**
- Expõe a **API REST do Kubernetes** (recursos como Pods, Deployments, Services, etc.)
- Autentica e autoriza cada requisição (integra com RBAC, certificados, tokens)
- Valida e persiste o estado dos objetos no **etcd** — é o único componente que lê e escreve diretamente no etcd
- Serve como hub de comunicação: o `kube-scheduler`, `kube-controller-manager` e os `kubelets` nos nodes se comunicam exclusivamente através do kube-apiserver

**Fluxo de uma requisição:**

```
Cliente (kubectl / app) → Autenticação → Autorização (RBAC) → Admission Controllers → etcd
```

> **Dica de prova:** O kube-apiserver é o único componente que se comunica com o etcd. Todos os outros componentes do control plane consultam e atualizam o estado do cluster **indiretamente**, através da API.

## etcd

etcd is a distributed key-value store used by Kubernetes for reliable configuration data and state storage. It is the cluster’s source of truth and provides strong consistency, high availability, and data durability.

A common deployment uses 3 etcd members (replicas) because etcd relies on the Raft consensus algorithm. Raft requires a majority of nodes (quorum) to be healthy and reachable to make progress. In a 3-member cluster, losing one node still leaves a majority (2/3) available; losing two nodes causes loss of quorum and cluster unavailability. For this reason, etcd clusters should use an odd number of members (3, 5, 7, ...), so you can tolerate failures while maintaining quorum.

## kube-scheduler

TBD

## kube-controller-manager

TBD

## cloud-controller-manager

TBD
