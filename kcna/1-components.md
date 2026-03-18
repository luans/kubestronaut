# Components

## Control Plane

- `kube-apiserver`: exposes the k8s API, basically it is a frontend for the control plane. This is the only component that have access directly to the `etcd`.
- `etcd`: it is a consistenty key-value database with high availability. This is the source of truth for the k8s.
- `kube-scheduler`: a component that watch new pods on the cluster and assign them to the most suitable node into the k8s, according to the specific parameters such as resources, taints, tolerations, afinity, eg.
- `kube-controller-manager`: responsible to execute the reconcile looping/control looping, basically this component watches the current status and try to adjust to the desirable status.
- `cloud-controller-manager`: opcional component used to integrate the cluster with cloud providers such as AWS, Digital Ocean, Azure, eg.

## Node Components

- `kubelet`: principal agent that runs on each node, the component that receives the PodSpecs and ensure that the pods are running and healthy.
- `kube-proxy`: manage the network rules on nodes, it is responsible to implement the service concepts and keeps the rules for network communication.
- `Container Runtime`: software responsible for running containers (CRI interface).