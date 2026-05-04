# Chapter 31: Container and VM Security Deep Dive

## Learning Goals
- Understand container escape vectors and mitigations
- Learn VM escape (guest-to-host) attack surface
- Master security hardening for containers and VMs
- Know confidential computing: SEV, TDX, CCA

---

## 1. Container Escape Attack Vectors

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Container escape = breaking out of namespace/cgroup     │
  │  isolation to access the host                           │
  │                                                           │
  │  Attack surfaces:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Kernel exploits:                      │            │
  │  │    Container shares host kernel           │            │
  │  │    Kernel vulnerability = escape          │            │
  │  │    CVE-2022-0185: heap overflow in FS     │            │
  │  │    CVE-2022-0492: cgroup v1 release_agent │            │
  │  │    Mitigation: seccomp, gVisor, Kata      │            │
  │  │                                          │            │
  │  │ 2. Dangerous capabilities:               │            │
  │  │    CAP_SYS_ADMIN: mount, namespace ops   │            │
  │  │    CAP_SYS_PTRACE: ptrace any process    │            │
  │  │    CAP_NET_RAW: raw packet crafting       │            │
  │  │    Mitigation: drop ALL, add only needed │            │
  │  │                                          │            │
  │  │ 3. Sensitive mount exposure:             │            │
  │  │    /proc/sys, /proc/sysrq-trigger        │            │
  │  │    /sys/fs/cgroup (v1 release_agent)     │            │
  │  │    Docker socket: /var/run/docker.sock   │            │
  │  │    Mitigation: read-only mounts, no sock │            │
  │  │                                          │            │
  │  │ 4. Privileged containers:                │            │
  │  │    --privileged = ALL capabilities +     │            │
  │  │    all devices + no seccomp + no apparmor│            │
  │  │    Effectively NO isolation              │            │
  │  │    Mitigation: NEVER use in production   │            │
  │  │                                          │            │
  │  │ 5. Container runtime bugs:               │            │
  │  │    CVE-2019-5736: runc overwrite via      │            │
  │  │    /proc/self/exe                        │            │
  │  │    Mitigation: keep runtime updated      │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VM Security and Guest-to-Host Attacks

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  VM attack surface:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Hardware isolation (much stronger):       │            │
  │  │ - Separate kernel                        │            │
  │  │ - CPU ring separation (VMX non-root)     │            │
  │  │ - IOMMU for device isolation             │            │
  │  │ - EPT for memory isolation               │            │
  │  │                                          │            │
  │  │ But attack surface still exists:         │            │
  │  │                                          │            │
  │  │ 1. QEMU device emulation:               │            │
  │  │    Guest sends crafted data to emulated  │            │
  │  │    device → buffer overflow in QEMU      │            │
  │  │    CVE-2020-14364: USB OOB write         │            │
  │  │    Mitigation: minimize emulated devices,│            │
  │  │    use virtio, sandbox QEMU              │            │
  │  │                                          │            │
  │  │ 2. Shared memory (virtio):               │            │
  │  │    Guest can write arbitrary data to     │            │
  │  │    shared virtqueue descriptors          │            │
  │  │    Backend must validate ALL guest data  │            │
  │  │    TOCTOU: guest modifies data after     │            │
  │  │    backend validates it                   │            │
  │  │    Mitigation: copy-then-validate        │            │
  │  │                                          │            │
  │  │ 3. Side channels:                        │            │
  │  │    Spectre/Meltdown: guest reads host    │            │
  │  │    memory via speculative execution      │            │
  │  │    L1TF: reads L1 cache across VMs       │            │
  │  │    Mitigation: microcode updates, flush  │            │
  │  │    L1 on VM entry, core scheduling       │            │
  │  │                                          │            │
  │  │ 4. KVM vulnerabilities:                  │            │
  │  │    Bugs in VMCS handling, nested virt    │            │
  │  │    CVE-2021-22555: Netfilter OOB write   │            │
  │  │    Mitigation: kernel updates, audit     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Defense-in-Depth Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Layer    Container          VM                          │
  │  ───────────────────────────────────────────────────     │
  │  L1       read-only rootfs   minimal guest image        │
  │  L2       drop capabilities  minimal QEMU devices       │
  │  L3       seccomp BPF        QEMU sandboxing            │
  │  L4       AppArmor/SELinux   SELinux sVirt labels       │
  │  L5       user namespaace    separate IOMMU groups      │
  │  L6       cgroup limits      memory/CPU limits          │
  │  L7       network policy     firewall rules             │
  │  L8       image scanning     image verification         │
  │                                                           │
  │  Container hardening checklist:                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ □ Non-root user in container              │            │
  │  │ □ Read-only root filesystem               │            │
  │  │ □ No new privileges (no_new_privs)        │            │
  │  │ □ Drop all capabilities, add only needed │            │
  │  │ □ Seccomp profile (block dangerous calls)│            │
  │  │ □ AppArmor/SELinux profile enabled        │            │
  │  │ □ No host PID/network/IPC namespace      │            │
  │  │ □ No privileged mode                      │            │
  │  │ □ No Docker socket mount                  │            │
  │  │ □ Resource limits (memory, CPU, pids)     │            │
  │  │ □ Network policies (ingress/egress)       │            │
  │  │ □ Image from trusted registry             │            │
  │  │ □ Vulnerability scanning in CI/CD         │            │
  │  │ □ Runtime security (Falco, Tracee)        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Kubernetes Pod Security Standards:                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Privileged: unrestricted (admin only)    │            │
  │  │ Baseline: prevent known escalations      │            │
  │  │ Restricted: hardened (best practice)     │            │
  │  │                                          │            │
  │  │ Restricted enforces:                     │            │
  │  │ - runAsNonRoot: true                     │            │
  │  │ - allowPrivilegeEscalation: false        │            │
  │  │ - capabilities.drop: [ALL]               │            │
  │  │ - seccompProfile: RuntimeDefault         │            │
  │  │ - no hostNetwork/hostPID/hostIPC         │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Confidential Computing

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Confidential VMs: protect guest from HOST               │
  │  (even hypervisor can't read guest memory)               │
  │                                                           │
  │  ┌──────────────┬─────────────────────────────────────┐ │
  │  │ Technology   │ Description                          │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ AMD SEV      │ Secure Encrypted Virtualization     │ │
  │  │              │ AES-128 encrypts guest memory       │ │
  │  │              │ Key per-VM, managed by PSP (secure  │ │
  │  │              │ processor)                          │ │
  │  │              │ Hypervisor sees ciphertext only     │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ AMD SEV-SNP  │ SEV Secure Nested Paging            │ │
  │  │              │ Adds integrity (prevent replay,     │ │
  │  │              │ remapping attacks)                  │ │
  │  │              │ Reverse Map Table (RMP) enforced    │ │
  │  │              │ by hardware                        │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ Intel TDX    │ Trust Domain Extensions             │ │
  │  │              │ TD (Trust Domain) per VM            │ │
  │  │              │ SEAM module (secure arbiter)        │ │
  │  │              │ Memory encryption + integrity       │ │
  │  │              │ Attestation support                 │ │
  │  ├──────────────┼─────────────────────────────────────┤ │
  │  │ ARM CCA      │ Confidential Compute Architecture  │ │
  │  │              │ Realm VMs (Armv9)                   │ │
  │  │              │ RMM (Realm Management Monitor)      │ │
  │  │              │ GPT (Granule Protection Tables)     │ │
  │  └──────────────┴─────────────────────────────────────┘ │
  │                                                           │
  │  Threat model shift:                                     │
  │  Traditional: trust hypervisor, protect from guest      │
  │  Confidential: protect guest FROM hypervisor            │
  │                                                           │
  │  Attestation flow:                                       │
  │  1. Guest measures its boot state (hash chain)          │
  │  2. Hardware signs measurement report                   │
  │  3. Remote verifier checks: is this VM running          │
  │     expected code on genuine hardware?                  │
  │  4. Verifier provisions secrets (keys, certs)           │
  │     only to attested VMs                                │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Compare the security boundaries of containers vs VMs and describe scenarios where each is appropriate.**
**A:** **Containers** share the host kernel — the attack surface is the entire kernel syscall interface (~300+ syscalls). Seccomp reduces this (Docker's default profile blocks ~50 syscalls), but a kernel vulnerability can break containment (CVE-2022-0185, CVE-2022-0492). Capabilities, LSM profiles, and user namespaces add layers but don't change the fundamental shared-kernel model. Best for: trusted workloads, same-tenant multi-service deployments, environments where fast startup and density matter. **VMs** provide hardware isolation: separate kernel, CPU mode separation (VMX root/non-root), EPT memory isolation, and IOMMU device isolation. Attack surface is much narrower: emulated devices in QEMU (which can be further reduced/sandboxed), KVM code in the host kernel, and hardware side channels (Spectre/Meltdown). Best for: untrusted workloads, multi-tenant environments, running different OS versions, workloads with strong compliance requirements. **Hybrid approaches**: Kata Containers (container API, VM isolation), gVisor (container API, separate syscall layer), Firecracker (lightweight VM, minimal device model). These give container developer experience with stronger isolation. **Confidential VMs** go further: AMD SEV/Intel TDX protect the guest even from a compromised hypervisor by encrypting guest memory with hardware-managed keys.

**Q2: Describe the container security problem with the Docker socket and how to mitigate it.**
**A:** The Docker socket (`/var/run/docker.sock`) is a Unix socket that provides full control over the Docker daemon — it's effectively root access to the host. When mounted into a container (`-v /var/run/docker.sock:/var/run/docker.sock`), the container can: (1) create new privileged containers with host filesystem mounted, (2) execute commands in other containers, (3) pull and run arbitrary images, (4) access host networking and devices. This is common in CI/CD (building images inside containers), monitoring tools, and Docker-in-Docker scenarios. Mitigations: (a) **Don't mount it** — use Kaniko or Buildah for in-container image builds (they don't need a daemon). (b) **Read-only socket proxy** — tools like Docker Socket Proxy expose only specific API endpoints (e.g., allow `docker ps` but not `docker run`). (c) **Use rootless Docker** — Docker daemon runs as non-root user, limiting what socket access can do. (d) **Pod security policies/standards** — Kubernetes PSS Restricted profile blocks hostPath mounts. (e) **Alternatives**: use `nerdctl` (containerd CLI) or `podman` (daemonless, rootless by default) which don't have a privileged daemon socket. The fundamental principle: never give a container access to a control plane that can create other containers.

---

## Summary

- Container escapes: kernel vulnerabilities, dangerous capabilities, privileged mode, runtime bugs
- VM escapes: QEMU device emulation bugs, shared memory TOCTOU, side channels
- Container hardening: non-root, drop caps, seccomp, LSM, read-only rootfs, no Docker socket
- K8s Pod Security Standards: Restricted profile enforces best practices
- Confidential computing: AMD SEV/SNP, Intel TDX, ARM CCA — encrypt guest memory from host
- Attestation: hardware-signed measurement report proves VM integrity to remote verifier

---

[Previous: Performance & Debugging ←](Chapter_30_Performance_Debug.md) | [Next: Interview Preparation →](Chapter_32_Interview_Prep.md)
