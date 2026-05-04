# Chapter 29: Kubernetes and Container Runtime Interface

## Learning Goals
- Understand Kubernetes pod lifecycle and sandbox model
- Learn CRI gRPC API and its protocol
- Master RuntimeClass for multiple container runtimes
- Know pod networking and CNI integration with K8s

---

## 1. Kubernetes Architecture and Pod Model

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Kubernetes node architecture:                           │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Control Plane                            │            │
  │  │   kube-apiserver                          │            │
  │  │       │                                  │            │
  │  │       ▼                                  │            │
  │  │   kube-scheduler → assigns pod to node  │            │
  │  │   controller-manager                     │            │
  │  │   etcd (state store)                     │            │
  │  └──────────────┬───────────────────────────┘            │
  │                  │ kubelet watches API server            │
  │                  ▼                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Worker Node                              │            │
  │  │                                          │            │
  │  │ kubelet (node agent)                     │            │
  │  │   │                                      │            │
  │  │   ├── CRI gRPC ──→ containerd / CRI-O   │            │
  │  │   │                    │                 │            │
  │  │   │                    ├── OCI runtime    │            │
  │  │   │                    │   (runc, kata)   │            │
  │  │   │                    │                 │            │
  │  │   │                    └── container shim │            │
  │  │   │                                      │            │
  │  │   ├── CNI ──→ network plugin             │            │
  │  │   │            (Calico, Cilium, Flannel)  │            │
  │  │   │                                      │            │
  │  │   └── CSI ──→ storage plugin             │            │
  │  │                (volume mount)            │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Pod = group of containers sharing:                     │
  │  - Network namespace (same IP, same ports)              │
  │  - IPC namespace (shared memory, semaphores)            │
  │  - Volumes (shared filesystems)                         │
  │  - Cgroup (resource limits applied together)            │
  │  Containers in a pod: own PID, mount, UTS namespaces    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. CRI (Container Runtime Interface)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CRI = gRPC API between kubelet and container runtime    │
  │                                                           │
  │  Two service interfaces:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ RuntimeService:                          │            │
  │  │   PodSandbox operations:                 │            │
  │  │     RunPodSandbox(config)                │            │
  │  │     StopPodSandbox(id)                   │            │
  │  │     RemovePodSandbox(id)                 │            │
  │  │     PodSandboxStatus(id)                 │            │
  │  │     ListPodSandbox(filter)               │            │
  │  │                                          │            │
  │  │   Container operations:                  │            │
  │  │     CreateContainer(sandbox, config)      │            │
  │  │     StartContainer(id)                   │            │
  │  │     StopContainer(id, timeout)           │            │
  │  │     RemoveContainer(id)                  │            │
  │  │     ListContainers(filter)               │            │
  │  │     ContainerStatus(id)                  │            │
  │  │     ExecSync(id, cmd)                    │            │
  │  │     Exec(id, cmd) → streaming            │            │
  │  │     Attach(id) → streaming               │            │
  │  │     PortForward(id, port)                │            │
  │  │                                          │            │
  │  │ ImageService:                            │            │
  │  │     PullImage(image, auth)               │            │
  │  │     RemoveImage(image)                   │            │
  │  │     ImageStatus(image)                   │            │
  │  │     ListImages(filter)                   │            │
  │  │     ImageFsInfo()                        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Pod creation flow:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. kubelet → RunPodSandbox(config)       │            │
  │  │    Runtime creates:                       │            │
  │  │    - Network namespace                   │            │
  │  │    - Calls CNI ADD (gets pod IP)          │            │
  │  │    - Creates pause container (holds NS)  │            │
  │  │    - Sets up cgroup                       │            │
  │  │                                          │            │
  │  │ 2. kubelet → PullImage(image)            │            │
  │  │    Runtime pulls image layers             │            │
  │  │                                          │            │
  │  │ 3. kubelet → CreateContainer(sandbox,cfg)│            │
  │  │    Runtime creates container in sandbox:  │            │
  │  │    - Joins sandbox's network NS          │            │
  │  │    - Creates own PID/mount NS            │            │
  │  │    - Sets up rootfs (overlay mount)      │            │
  │  │                                          │            │
  │  │ 4. kubelet → StartContainer(id)          │            │
  │  │    OCI runtime (runc) starts container   │            │
  │  │                                          │            │
  │  │ 5. kubelet monitors container status     │            │
  │  │    via ListContainers / ContainerStatus  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. RuntimeClass and Multi-Runtime

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  RuntimeClass: run different pods with different         │
  │  container runtimes on the same node                    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ apiVersion: node.k8s.io/v1               │            │
  │  │ kind: RuntimeClass                       │            │
  │  │ metadata:                                │            │
  │  │   name: kata-containers                  │            │
  │  │ handler: kata                            │            │
  │  │ ---                                      │            │
  │  │ apiVersion: node.k8s.io/v1               │            │
  │  │ kind: RuntimeClass                       │            │
  │  │ metadata:                                │            │
  │  │   name: gvisor                           │            │
  │  │ handler: runsc                           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Using in a pod:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ apiVersion: v1                           │            │
  │  │ kind: Pod                                │            │
  │  │ metadata:                                │            │
  │  │   name: secure-workload                  │            │
  │  │ spec:                                    │            │
  │  │   runtimeClassName: kata-containers      │            │
  │  │   containers:                            │            │
  │  │   - name: app                            │            │
  │  │     image: myapp:latest                  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  containerd config (/etc/containerd/config.toml):       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ [plugins."io.containerd.grpc.v1.cri"     │            │
  │  │   .containerd.runtimes.runc]             │            │
  │  │   runtime_type = "...runc-v2"            │            │
  │  │                                          │            │
  │  │ [plugins."io.containerd.grpc.v1.cri"     │            │
  │  │   .containerd.runtimes.kata]             │            │
  │  │   runtime_type = "...kata-v2"            │            │
  │  │                                          │            │
  │  │ [plugins."io.containerd.grpc.v1.cri"     │            │
  │  │   .containerd.runtimes.runsc]            │            │
  │  │   runtime_type = "...runsc-v1"           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Decision matrix:                                        │
  │  ┌───────────────┬─────────────────────────────────────┐│
  │  │ Runtime       │ When to use                          ││
  │  ├───────────────┼─────────────────────────────────────┤│
  │  │ runc          │ Default, trusted workloads           ││
  │  │ kata          │ Untrusted, strong isolation (VM)     ││
  │  │ gVisor (runsc)│ Untrusted, no HW virt available     ││
  │  │ crun          │ Performance-sensitive (C, faster)    ││
  │  │ youki         │ Performance (Rust)                    ││
  │  └───────────────┴─────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Pod Networking and CNI in K8s

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  K8s networking model:                                   │
  │  1. Every pod gets its own IP address                   │
  │  2. Pods can communicate without NAT                    │
  │  3. Nodes can communicate with pods without NAT         │
  │                                                           │
  │  CNI integration:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ kubelet → RunPodSandbox                  │            │
  │  │   → CRI runtime creates network NS      │            │
  │  │   → CRI runtime calls CNI ADD:          │            │
  │  │                                          │            │
  │  │ CNI_COMMAND=ADD                          │            │
  │  │ CNI_CONTAINERID=abc123                   │            │
  │  │ CNI_NETNS=/var/run/netns/abc123          │            │
  │  │ CNI_IFNAME=eth0                          │            │
  │  │                                          │            │
  │  │ CNI plugin (e.g., calico):               │            │
  │  │ 1. Creates veth pair                     │            │
  │  │ 2. Moves one end into pod netns          │            │
  │  │ 3. Assigns IP from IPAM                  │            │
  │  │ 4. Sets up routes                        │            │
  │  │ 5. Programs BGP/VXLAN/eBPF               │            │
  │  │ 6. Returns: { "ips": ["10.244.1.5/24"] } │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Popular CNI plugins:                                    │
  │  ┌──────────────┬─────────────────────────────────────┐ │
  │  │ Plugin       │ Mechanism                            │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ Flannel      │ VXLAN overlay, simple                │ │
  │  │ Calico       │ BGP routing or VXLAN, NetworkPolicy │ │
  │  │ Cilium       │ eBPF datapath, no iptables          │ │
  │  │ Weave        │ Mesh overlay                        │ │
  │  │ AWS VPC CNI  │ ENI + secondary IPs (native)        │ │
  │  │ Azure CNI    │ Azure VNET integration               │ │
  │  └──────────────┴─────────────────────────────────────┘ │
  │                                                           │
  │  Service networking:                                     │
  │  Pod IPs are ephemeral → Services provide stable VIP   │
  │  kube-proxy: iptables/ipvs rules for service routing    │
  │  Cilium: replaces kube-proxy entirely with eBPF        │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the CRI (Container Runtime Interface) and how kubelet creates a pod.**
**A:** CRI is a gRPC API that decouples kubelet from container runtime implementation. It defines two services: RuntimeService (pod and container lifecycle) and ImageService (image pull/removal). Pod creation: (1) kubelet receives pod spec from API server. (2) kubelet calls `RunPodSandbox(PodSandboxConfig)` — the runtime creates a "sandbox": allocates a new network namespace, creates a cgroup, runs the CNI plugin to assign a pod IP, and starts a pause container (an empty container that holds namespaces alive). (3) For each container in the pod spec, kubelet calls `PullImage()` if needed, then `CreateContainer(sandboxId, ContainerConfig)` — the runtime creates the container within the sandbox, joining the sandbox's network and IPC namespaces but getting its own PID and mount namespaces. The rootfs is an overlay mount of image layers. (4) kubelet calls `StartContainer(containerId)` — the runtime invokes the OCI runtime (runc) to start the container process. (5) kubelet monitors via `ContainerStatus()` and `ListContainers()`. Container readiness/liveness probes are executed via `ExecSync()`. This abstraction allows Kubernetes to work with containerd, CRI-O, or any CRI-compliant runtime without code changes.

**Q2: How does Kubernetes RuntimeClass enable multi-tenancy with different isolation levels?**
**A:** RuntimeClass is a Kubernetes API resource that maps a handler name to a container runtime configuration on each node. It enables running different workloads with different runtimes on the same cluster. Example setup: create RuntimeClass objects for `runc` (default, Linux namespaces/cgroups), `kata-containers` (microVM per pod via Kata), and `gvisor` (user-space kernel via runsc). In containerd's `config.toml`, configure each handler name to a corresponding shim binary. Pods specify `spec.runtimeClassName` — kubelet passes this handler name to the CRI runtime, which selects the appropriate OCI runtime. Multi-tenancy scenario: trusted first-party workloads run with `runc` (fast, low overhead). Untrusted third-party or multi-tenant workloads run with `kata-containers` (hardware VM isolation — guest can't see host kernel, even with container escape). CI/CD pipelines running arbitrary code use `gvisor` (syscall interception without hardware virtualization requirement). Each RuntimeClass can also specify `overhead` (additional memory/CPU the runtime itself consumes) and `scheduling` (node selector to schedule pods only to nodes with the runtime installed). This provides defense-in-depth: standard namespace isolation for trusted code, VM-level isolation for untrusted code.

---

## Summary

- CRI: gRPC API with RuntimeService (pod/container ops) and ImageService (image ops)
- Pod sandbox: shared network/IPC namespace, pause container holds namespaces
- Pod creation: RunPodSandbox → CNI ADD → PullImage → CreateContainer → StartContainer
- RuntimeClass: maps handler name to OCI runtime; enables runc/kata/gvisor per-pod
- CNI integration: called during sandbox creation; assigns pod IP, sets up networking
- Service networking: kube-proxy (iptables/ipvs) or Cilium (eBPF) for stable service IPs

---

[Previous: VFIO & SR-IOV ←](Chapter_28_VFIO_SRIOV.md) | [Next: Performance and Debugging →](Chapter_30_Performance_Debug.md)
