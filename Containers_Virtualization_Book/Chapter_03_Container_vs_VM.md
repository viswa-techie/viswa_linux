# Chapter 3: Container vs VM Architecture

## Learning Goals
- Compare container and VM architectures at the kernel level
- Understand performance, security, and density trade-offs
- Know when to use containers, VMs, or hybrid approaches
- Learn microVM architecture (Firecracker, gVisor)

---

## 1. Architecture Comparison

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Virtual Machines:                                       │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
  │  │ App A    │  │ App B    │  │ App C    │              │
  │  │ Libs     │  │ Libs     │  │ Libs     │              │
  │  │ Guest OS │  │ Guest OS │  │ Guest OS │              │
  │  │ (kernel) │  │ (kernel) │  │ (kernel) │              │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
  │       └──────────────┼──────────────┘                    │
  │                      ▼                                    │
  │              ┌───────────────┐                           │
  │              │  Hypervisor   │  (KVM + QEMU)             │
  │              └───────┬───────┘                           │
  │                      ▼                                    │
  │              ┌───────────────┐                           │
  │              │  Host OS/HW   │                           │
  │              └───────────────┘                           │
  │                                                           │
  │  Containers:                                             │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
  │  │ App A    │  │ App B    │  │ App C    │              │
  │  │ Libs     │  │ Libs     │  │ Libs     │              │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
  │       └──────────────┼──────────────┘                    │
  │         namespaces + cgroups + seccomp                    │
  │                      ▼                                    │
  │              ┌───────────────┐                           │
  │              │  Host Kernel  │  (shared!)                │
  │              └───────┬───────┘                           │
  │                      ▼                                    │
  │              ┌───────────────┐                           │
  │              │  Hardware     │                           │
  │              └───────────────┘                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Quantitative Comparison

```
  ┌──────────────────┬──────────────┬────────────────────────┐
  │ Metric           │ Container    │ VM                     │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Startup time     │ ~100ms       │ ~3-30 seconds          │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Memory overhead  │ ~1-10 MB     │ ~100-512 MB per VM     │
  │                  │ (process)    │ (full OS in RAM)       │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ CPU overhead     │ ~0%          │ ~1-5% (VM exits)       │
  │                  │ (native)     │                        │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Disk image       │ ~50-200 MB   │ ~1-10 GB per VM image  │
  │                  │ (layered)    │                        │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Density          │ 100s-1000s   │ 10s per host           │
  │ (per host)       │              │                        │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Isolation        │ Kernel-level │ Hardware-level          │
  │                  │ (namespaces) │ (MMU, separate kernel) │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Kernel exploit   │ Escapes to   │ Contained within VM    │
  │ impact           │ host         │ (need hypervisor bug)  │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Different OS     │ No (same     │ Yes (Linux, Windows,   │
  │                  │ kernel)      │ any OS)                │
  ├──────────────────┼──────────────┼────────────────────────┤
  │ Syscall path     │ Direct to    │ Guest kernel → VMexit  │
  │                  │ host kernel  │ → hypervisor           │
  └──────────────────┴──────────────┴────────────────────────┘
```

---

## 3. MicroVMs — Best of Both Worlds

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  MicroVM (Firecracker, Cloud Hypervisor):                │
  │                                                           │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
  │  │ App      │  │ App      │  │ App      │              │
  │  │ Minimal  │  │ Minimal  │  │ Minimal  │              │
  │  │ kernel   │  │ kernel   │  │ kernel   │              │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
  │       └──────────────┼──────────────┘                    │
  │                      ▼                                    │
  │              ┌───────────────┐                           │
  │              │ KVM (in-kernel)                           │
  │              │ Firecracker   │  ← minimal VMM           │
  │              │ (~50K lines)  │  (no BIOS, no USB,       │
  │              │               │   no graphics)           │
  │              └───────────────┘                           │
  │                                                           │
  │  Firecracker (AWS Lambda, Fargate):                     │
  │  - Startup: < 125ms (vs 3-30s for QEMU VM)             │
  │  - Memory: < 5 MB overhead per microVM                  │
  │  - Security: hardware isolation (like VM)                │
  │  - Density: 1000s per host (like containers)            │
  │  - Written in Rust (memory-safe VMM)                    │
  │                                                           │
  │  gVisor (Google):                                       │
  │  - Application kernel in userspace                      │
  │  - Intercepts syscalls, re-implements in Go             │
  │  - No hardware virtualization needed                    │
  │  - Trade-off: syscall overhead but stronger isolation   │
  │                                                           │
  │  Kata Containers:                                       │
  │  - Standard container API (OCI-compatible)              │
  │  - Each container runs in a lightweight VM              │
  │  - Transparent to Kubernetes                            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Decision Matrix

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Use Containers when:                                    │
  │  - Same OS family (Linux apps on Linux host)            │
  │  - Fast scaling needed (100ms startup)                   │
  │  - High density needed (1000s of instances)              │
  │  - CI/CD pipelines (build → test → deploy)              │
  │  - Microservices architecture                            │
  │  - Trusted workloads (your own code)                     │
  │                                                           │
  │  Use VMs when:                                           │
  │  - Different OS needed (Windows on Linux host)          │
  │  - Strong isolation required (multi-tenant)             │
  │  - Running untrusted code                               │
  │  - Kernel-level differences needed                      │
  │  - Legacy applications requiring full OS                │
  │                                                           │
  │  Use MicroVMs when:                                     │
  │  - Container speed + VM isolation                       │
  │  - Serverless platforms (AWS Lambda)                    │
  │  - Multi-tenant with untrusted code                     │
  │  - Security-sensitive cloud workloads                   │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why are containers not as secure as VMs?**
**A:** Containers share the host kernel — all containers make syscalls to the same kernel. A kernel exploit in any container can compromise the host and all other containers. VMs have a much smaller attack surface: guest code runs in a separate address space enforced by hardware (EPT/NPT), and only interacts with the host through the hypervisor's narrow interface (VM exits for privileged operations). The container attack surface is ~400 Linux syscalls; the VM attack surface is ~30 VM exit reasons. Mitigations: (1) seccomp reduces syscalls from ~400 to ~250 (Docker default filter). (2) User namespaces map container root to unprivileged host user. (3) SELinux/AppArmor restrict file access. (4) gVisor intercepts syscalls in userspace (smaller kernel surface). (5) Kata/Firecracker run each container in a microVM for hardware isolation.

**Q2: What is a microVM and how does Firecracker achieve container-like startup times?**
**A:** A microVM is a ultra-lightweight virtual machine with a minimal VMM. Firecracker (Amazon): (1) Minimal device model — only virtio-net, virtio-blk, serial, and a minimal interrupt controller. No BIOS, USB, graphics, or PCI. (2) Written in Rust (~50K lines vs QEMU's ~2M lines). (3) Uses KVM for hardware virtualization (not emulation). (4) Boots a minimal guest kernel directly (no bootloader, no firmware). (5) Startup < 125ms, memory overhead < 5MB. This gives VM-level isolation (separate kernel, hardware MMU) with container-level density and startup time. Used by AWS Lambda (serverless functions) and Fargate (container service). Trade-off: guests must be Linux (no Windows/BSD support).

---

## Summary

- Containers: shared kernel, near-zero overhead, weak isolation (namespaces)
- VMs: separate kernel, hardware isolation (VT-x/EPT), stronger security, more overhead
- MicroVMs: VM isolation + container speed (Firecracker < 125ms, < 5MB)
- Container density: 1000s per host vs VMs: 10s per host
- Kernel exploit: containers → host compromise; VMs → contained
- Decision: speed/density → containers; isolation/security → VMs; both → microVMs

---

[Previous: History ←](Chapter_02_History.md) | [Next: Building Blocks →](Chapter_04_Building_Blocks.md)
