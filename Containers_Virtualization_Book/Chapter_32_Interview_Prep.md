# Chapter 32: Interview Preparation — Containers and Virtualization

## Learning Goals
- Review core concepts with rapid-fire Q&A
- Master architecture comparison questions
- Practice system design scenarios
- Build mental models for whiteboard discussions

---

## 1. Rapid-Fire Concept Review

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Q: What are the 8 Linux namespaces?                    │
  │  A: PID, Mount, Network, UTS, IPC, User, Cgroup, Time  │
  │                                                           │
  │  Q: What syscalls create namespaces?                    │
  │  A: clone(CLONE_NEW*), unshare(), setns()               │
  │                                                           │
  │  Q: Cgroups v1 vs v2 key difference?                    │
  │  A: v1 = per-controller hierarchies (process in         │
  │     different trees for CPU vs memory)                  │
  │     v2 = unified hierarchy (one tree, all controllers)  │
  │                                                           │
  │  Q: What is EPT?                                        │
  │  A: Extended Page Tables — hardware 2D page walk        │
  │     GVA → GPA (guest tables) → HPA (EPT tables)        │
  │     Eliminates shadow page table overhead               │
  │                                                           │
  │  Q: What is a VMCS?                                     │
  │  A: Virtual Machine Control Structure — per-vCPU        │
  │     hardware struct containing guest/host state,        │
  │     control fields, exit/entry information              │
  │                                                           │
  │  Q: KVM_RUN ioctl — what happens?                       │
  │  A: QEMU calls KVM_RUN → KVM does VMLAUNCH/VMRESUME    │
  │     → CPU runs guest code → VM exit → KVM handles      │
  │     → returns to QEMU userspace if KVM can't handle    │
  │                                                           │
  │  Q: What is virtio?                                     │
  │  A: Paravirtual I/O standard — guest knows it's in VM, │
  │     uses efficient shared-memory virtqueues instead of  │
  │     emulating real hardware register-by-register        │
  │                                                           │
  │  Q: Split ring vs packed ring?                          │
  │  A: Split = 3 areas (desc table + avail ring + used     │
  │     ring) — cache unfriendly                            │
  │     Packed = 1 ring with in-place AVAIL/USED flags      │
  │     — cache friendly, better hardware implementation    │
  │                                                           │
  │  Q: What is SR-IOV?                                     │
  │  A: PCIe spec — one Physical Function creates multiple  │
  │     Virtual Functions, each assignable to a VM via VFIO │
  │     Near-native performance, hardware queue isolation   │
  │                                                           │
  │  Q: What is vDPA?                                       │
  │  A: Hardware implements virtio queue format — standard  │
  │     virtio driver + native HW performance + migration   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Comparison Questions

**Q1: Container vs VM — when would you choose each?**

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Choose containers when:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ✓ Same OS (Linux) across all workloads   │            │
  │  │ ✓ Fast startup needed (< 1 second)       │            │
  │  │ ✓ High density (100s of instances)       │            │
  │  │ ✓ Microservices architecture             │            │
  │  │ ✓ CI/CD pipeline stages                  │            │
  │  │ ✓ Trusted workloads (same organization)  │            │
  │  │ ✓ Developers need reproducible envs      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Choose VMs when:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ✓ Different OS kernels needed             │            │
  │  │ ✓ Strong isolation (multi-tenant cloud)  │            │
  │  │ ✓ Compliance requires VM-level separation │            │
  │  │ ✓ Legacy applications                    │            │
  │  │ ✓ Hardware passthrough (GPU, NIC)        │            │
  │  │ ✓ Untrusted workloads                    │            │
  │  │ ✓ Running Windows alongside Linux        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Choose hybrid (Kata/Firecracker):                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ✓ Container API but VM isolation needed  │            │
  │  │ ✓ Serverless / FaaS (AWS Lambda)         │            │
  │  │ ✓ Multi-tenant container platforms       │            │
  │  │ ✓ CRI-compatible with Kubernetes         │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

**Q2: Explain the complete I/O path for a network packet from guest application to physical NIC — for each virtualization approach.**

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  A. Full emulation (e1000):                              │
  │  app → guest socket → guest TCP/IP → guest e1000 drv    │
  │  → MMIO write to e1000 register                         │
  │  → VM exit (EPT_VIOLATION)                              │
  │  → KVM → return to QEMU                                 │
  │  → QEMU e1000 emulation                                 │
  │  → QEMU writes to TAP fd (syscall)                      │
  │  → kernel TAP driver → bridge → host NIC driver → NIC   │
  │  Exits per packet: ~10-50 (register-level emulation)    │
  │                                                           │
  │  B. virtio-net + QEMU backend:                          │
  │  app → guest socket → guest TCP/IP → guest virtio drv   │
  │  → write descriptor to avail ring                       │
  │  → single MMIO doorbell write                           │
  │  → VM exit → KVM → return to QEMU                      │
  │  → QEMU reads virtqueue, writes to TAP fd              │
  │  → kernel TAP → bridge → host NIC → NIC                 │
  │  Exits per packet: 1 (notification) + 1 (interrupt)     │
  │                                                           │
  │  C. virtio-net + vhost-net:                             │
  │  app → guest socket → guest TCP/IP → guest virtio drv   │
  │  → doorbell write                                       │
  │  → VM exit → KVM → ioeventfd → vhost-net worker        │
  │  → kernel reads virtqueue → TAP → bridge → NIC         │
  │  → irqfd → KVM injects interrupt (no QEMU)             │
  │  Exits per packet: 1 + 1 (no QEMU involvement)         │
  │                                                           │
  │  D. SR-IOV (VFIO passthrough):                           │
  │  app → guest socket → guest TCP/IP → guest VF driver    │
  │  → write to VF TX queue (MMIO, no VM exit!)             │
  │  → NIC reads from VF TX ring (DMA via IOMMU)           │
  │  → NIC transmits packet                                 │
  │  → NIC interrupt → posted interrupt → guest directly    │
  │  Exits per packet: 0 (hardware handles everything)      │
  │                                                           │
  │  E. virtio + vhost-user + DPDK:                         │
  │  app → guest socket → guest TCP/IP → guest virtio drv   │
  │  → doorbell write → ioeventfd                           │
  │  → DPDK (vhost-user) polls shared memory virtqueue      │
  │  → DPDK PMD writes to NIC (userspace, no kernel)        │
  │  → NIC transmits                                        │
  │  Exits per packet: ~0 (polling, no exit for doorbell    │
  │  if vhost-user uses polling mode)                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. System Design Scenarios

**Q3: Design a multi-tenant container platform that runs untrusted code safely.**

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Requirements: isolation, performance, multi-tenancy     │
  │                                                           │
  │  Architecture:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Layer 1: Kubernetes cluster               │            │
  │  │   - Namespace per tenant                 │            │
  │  │   - ResourceQuotas per namespace         │            │
  │  │   - NetworkPolicy: deny all ingress/     │            │
  │  │     egress by default, whitelist needed   │            │
  │  │                                          │            │
  │  │ Layer 2: RuntimeClass = kata-containers  │            │
  │  │   - Each pod runs in a microVM           │            │
  │  │   - VM provides kernel-level isolation   │            │
  │  │   - Even container escape stays in VM    │            │
  │  │                                          │            │
  │  │ Layer 3: Pod Security Standards (Restricted)│         │
  │  │   - Non-root, no privilege escalation    │            │
  │  │   - Drop all capabilities                │            │
  │  │   - Default seccomp profile              │            │
  │  │   - No host namespaces                   │            │
  │  │                                          │            │
  │  │ Layer 4: Network isolation               │            │
  │  │   - Cilium CNI with eBPF                 │            │
  │  │   - Per-tenant network policies          │            │
  │  │   - Encrypted pod-to-pod traffic         │            │
  │  │                                          │            │
  │  │ Layer 5: Image security                  │            │
  │  │   - Signed images only (cosign/notary)   │            │
  │  │   - Vulnerability scanning in admission  │            │
  │  │   - No latest tag, immutable tags        │            │
  │  │                                          │            │
  │  │ Layer 6: Runtime monitoring              │            │
  │  │   - Falco for syscall monitoring         │            │
  │  │   - eBPF-based file/network/process audit│            │
  │  │   - Automatic pod termination on anomaly │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

**Q4: Design a high-performance virtualized network function (VNF) with guaranteed throughput.**

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Requirements: 40 Gbps, <10μs latency, deterministic    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Hardware:                                │            │
  │  │ - SR-IOV NIC (Mellanox ConnectX-6)       │            │
  │  │ - 2MB hugepages (host + guest)           │            │
  │  │ - NUMA-aware CPU pinning                 │            │
  │  │ - CPU isolation (isolcpus=4-15)          │            │
  │  │                                          │            │
  │  │ Host:                                    │            │
  │  │ - DPDK + OVS-DPDK on dedicated cores     │            │
  │  │ - vhost-user sockets (shared memory)     │            │
  │  │ - IRQ affinity set to non-VNF CPUs       │            │
  │  │ - RT kernel patches (optional)           │            │
  │  │                                          │            │
  │  │ Guest VM:                                │            │
  │  │ - DPDK inside guest (poll-mode)          │            │
  │  │ - vCPUs pinned to isolated host CPUs     │            │
  │  │ - Hugepage memory (prealloc, locked)     │            │
  │  │ - NOHZ_FULL on packet processing cores   │            │
  │  │ - virtio multiqueue (queues = vCPUs)     │            │
  │  │                                          │            │
  │  │ Alternative (maximum perf):              │            │
  │  │ - SR-IOV VF passed directly to VM        │            │
  │  │ - Guest DPDK polls VF queues             │            │
  │  │ - Zero host CPU usage for forwarding     │            │
  │  │ - Trade-off: no live migration           │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Common Pitfalls and Tricky Questions

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Pitfall 1: "Containers are just lightweight VMs"        │
  │  Wrong! Containers share the host kernel.                │
  │  VMs have their own kernel. Fundamentally different      │
  │  security model.                                        │
  │                                                           │
  │  Pitfall 2: "Docker IS containers"                       │
  │  Docker is ONE container runtime. Others: containerd,   │
  │  CRI-O, podman. The technology is Linux namespaces +    │
  │  cgroups. Docker just popularized it.                   │
  │                                                           │
  │  Pitfall 3: "KVM is a type 2 hypervisor"                │
  │  Debatable. KVM converts Linux into a hypervisor.       │
  │  The kernel IS the hypervisor (type 1).                 │
  │  QEMU is userspace, but KVM runs at ring 0.             │
  │  Best answer: "type 1.5" or explain the nuance.         │
  │                                                           │
  │  Pitfall 4: "virtio is slower than hardware"             │
  │  virtio is slower than passthrough, but faster than      │
  │  emulation. With vhost + hugepages + multiqueue,        │
  │  virtio achieves 80-95% of native performance.          │
  │                                                           │
  │  Pitfall 5: "Namespaces provide security"               │
  │  Namespaces provide ISOLATION (visibility), not          │
  │  security. Security comes from seccomp, capabilities,   │
  │  LSM. Namespaces + cgroups + seccomp + LSM = defense   │
  │  in depth.                                               │
  │                                                           │
  │  Pitfall 6: "SR-IOV replaces virtio"                    │
  │  SR-IOV gives best performance but: can't live migrate, │
  │  requires hardware support, vendor-specific drivers.    │
  │  virtio is default choice; SR-IOV for special cases.    │
  │                                                           │
  │  Tricky Q: "Can a container have its own kernel?"       │
  │  Not in the traditional sense. But:                     │
  │  - Kata Containers: container API, runs in VM with     │
  │    separate kernel                                      │
  │  - gVisor: container has its own "kernel" (Sentry)     │
  │    but it's a userspace implementation                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Complete Architecture Diagram

```
  ┌──────────────────────────────────────────────────────────┐
  │  COMPLETE CONTAINER + VIRTUALIZATION STACK               │
  │                                                           │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ User Applications                                │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │ Container Orchestration (Kubernetes)              │    │
  │  │ ┌──────────┐ ┌──────────┐ ┌────────────────┐    │    │
  │  │ │ kubelet  │ │kube-proxy│ │ CNI (Cilium)   │    │    │
  │  │ └────┬─────┘ └──────────┘ └────────────────┘    │    │
  │  │      │ CRI gRPC                                  │    │
  │  ├──────┼───────────────────────────────────────────┤    │
  │  │ Container Runtime                                │    │
  │  │ ┌────┴──────┐ ┌──────────┐ ┌────────────────┐   │    │
  │  │ │containerd │ │ CRI-O   │ │ image store    │   │    │
  │  │ └────┬──────┘ └──────────┘ │ (overlay2)     │   │    │
  │  │      │ OCI runtime                          │   │    │
  │  │ ┌────┴────┬──────────┬──────────┐           │   │    │
  │  │ │ runc   │  kata    │ gVisor  │            │   │    │
  │  │ │(native)│ (microVM)│ (sentry)│            │   │    │
  │  │ └────────┘ └────┬───┘ └────────┘            │   │    │
  │  ├─────────────────┼───────────────────────────────┤    │
  │  │ Virtualization   │                                │    │
  │  │ ┌────────────────┴──────────────────────────┐    │    │
  │  │ │ QEMU / cloud-hypervisor / Firecracker     │    │    │
  │  │ │ ┌────────┐ ┌────────┐ ┌──────────────┐   │    │    │
  │  │ │ │ virtio │ │ vhost  │ │ vhost-user   │   │    │    │
  │  │ │ │ backend│ │ kernel │ │ (DPDK/SPDK)  │   │    │    │
  │  │ │ └────────┘ └────────┘ └──────────────┘   │    │    │
  │  │ └───────────────────────────────────────────┘    │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │ Linux Kernel                                     │    │
  │  │ ┌────────┐ ┌────┐ ┌────────┐ ┌──────────────┐   │    │
  │  │ │Namespace│ │KVM │ │cgroups │ │ VFIO/IOMMU  │   │    │
  │  │ │(8 types)│ │    │ │ v2     │ │              │   │    │
  │  │ └────────┘ └────┘ └────────┘ └──────────────┘   │    │
  │  │ ┌────────┐ ┌──────────────┐ ┌──────────────┐   │    │
  │  │ │seccomp │ │ LSM (SELinux/│ │ netfilter/   │   │    │
  │  │ │ BPF    │ │  AppArmor)  │ │ eBPF         │   │    │
  │  │ └────────┘ └──────────────┘ └──────────────┘   │    │
  │  ├──────────────────────────────────────────────────┤    │
  │  │ Hardware                                         │    │
  │  │ ┌──────┐ ┌─────┐ ┌───────┐ ┌──────────────┐    │    │
  │  │ │VT-x/ │ │EPT/ │ │VT-d/  │ │ SR-IOV /    │    │    │
  │  │ │AMD-V │ │NPT  │ │AMD-Vi │ │ vDPA NIC    │    │    │
  │  │ └──────┘ └─────┘ └───────┘ └──────────────┘    │    │
  │  │ ┌──────┐ ┌──────────────┐                       │    │
  │  │ │APICv/│ │ SEV/TDX/CCA │                       │    │
  │  │ │AVIC  │ │ (confid.comp)│                       │    │
  │  │ └──────┘ └──────────────┘                       │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through what happens when you type `docker run -it ubuntu bash` at the OS level.**
**A:** (1) Docker CLI parses the command and sends a gRPC request to the Docker daemon (dockerd). (2) dockerd checks if the `ubuntu` image is locally available; if not, pulls layers from Docker Hub (OCI distribution). (3) dockerd calls containerd via gRPC (CRI-like API) to create a container. (4) containerd creates an OCI bundle: unpacks image layers, creates an overlay filesystem (lower=image layers, upper=container writable layer, merged=union view), generates a `config.json` per OCI spec with namespaces, capabilities, mounts, etc. (5) containerd starts a shim process (`containerd-shim-runc-v2`) which calls `runc create` with the OCI bundle. (6) runc calls `clone()` with flags `CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | CLONE_NEWUTS | CLONE_NEWIPC` to create new namespaces. Inside the clone, the `nsexec` C constructor (via `/proc/self/exe` re-exec) sets up namespaces from privileged context. (7) In the new mount namespace: mount overlay rootfs, mount /proc, /dev, /sys, /dev/pts; `pivot_root` to new rootfs. (8) Apply cgroup limits (cpu.max, memory.max, pids.max). (9) Set up seccomp BPF filter (Docker's default profile). (10) Apply capabilities (drop 27, keep 14 of 41). (11) Create veth pair: one end in container's netns (eth0), other end on host bridge (docker0). Assign IP from Docker's subnet. (12) runc execs `/bin/bash` as PID 1 inside the container. (13) The shim remains as the container's parent process (reaps zombies, forwards stdio). (14) `-it` flags: dockerd sets up a PTY and streams stdin/stdout over the API connection.

**Q2: You're troubleshooting a production VM that has 10x higher tail latency than expected. Walk through your diagnostic approach.**
**A:** Systematic approach by layer: (1) **Check VM exits**: `perf kvm stat live` — look for high-frequency exits. If IO_INSTRUCTION exits dominate, the guest is using emulated devices (replace with virtio). If PAUSE_INSTRUCTION exits are high, guest has spinlock contention (enable pvspinlocks, check guest CPU overprovisioning). (2) **Check CPU scheduling**: verify vCPU pinning — unpinned vCPUs migrate between host CPUs, causing cache/NUMA penalties. Check for CPU overcommit: if more vCPUs than physical cores, vCPUs compete. Use `schedstat` to check vCPU wait time. (3) **Check memory**: verify hugepages are active (avoid 4KB page faults through EPT). Check KSM — if active, COW faults add latency spikes. Check NUMA: `numastat` to verify memory is local to vCPU's NUMA node. (4) **Check I/O**: verify virtio + vhost-net is in use (not QEMU backend). Check `iostat` for host disk contention. If using qcow2, check for excessive COW overhead (convert to raw or preallocate). Check I/O scheduler: guest should use `none`, host should use `mq-deadline` or `none` for SSD. (5) **Check host contention**: `perf stat -A -C <vcpu_cores>` to see IPC on vCPU cores. Check for noisy neighbors: other VMs or host processes on the same NUMA node. Check `mpstat` for IRQ imbalance. (6) **Check guest**: inside the guest, use `perf record -g` to profile the application. Check for lock contention, page faults, context switches. Look at `vmstat` for swap activity. (7) **Tail latency specifics**: tail latency often caused by occasional VM exits (device emulation), timer vCPU wakeup delays (increase `halt_poll_ns`), or host scheduler decisions. Use tracing: `perf sched` on host to see vCPU scheduling delays.

---

## Summary

- Containers: namespaces + cgroups + seccomp + LSM (shared kernel)
- VMs: VMX/SVM + EPT/NPT + virtio/vhost + IOMMU (separate kernel)
- I/O progression: emulation → virtio → vhost → vhost-user/DPDK → SR-IOV/vDPA
- Security progression: containers < gVisor < microVM (Kata) < VM < confidential VM
- Performance tuning: CPU pinning, hugepages, NUMA awareness, minimize VM exits
- K8s integration: CRI runtime, CNI networking, RuntimeClass for multi-runtime

---

## Book Complete

This book covered the complete Linux containers and virtualization stack:

| Part | Chapters | Topics |
|------|----------|--------|
| I    | 1-5      | Container foundations, building blocks, security |
| II   | 6-10     | Linux namespaces (all 8 types), lifecycle |
| III  | 11-14    | Cgroups v1, v2, CPU/memory/IO controllers |
| IV   | 15-18    | OCI spec, runc, containerd, container networking |
| V    | 19-22    | Hardware virt, KVM, QEMU, memory virt |
| VI   | 23-26    | I/O virt, virtio, interrupts, network/storage |
| VII  | 27-28    | vhost, vhost-user, VFIO, SR-IOV |
| VIII | 29-32    | Kubernetes CRI, perf tuning, security, interview |

---

[Previous: Security Deep Dive ←](Chapter_31_Security_Deep_Dive.md) | [Back to Index →](00_Master_Index.md)
