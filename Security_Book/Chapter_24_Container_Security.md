# Chapter 24: Container Security

## Learning Goals
- Understand container security architecture and attack surfaces
- Know how namespaces, cgroups, seccomp, and MAC combine for containers
- Understand container escape vectors and mitigations
- Know rootless containers and security best practices

---

## 24.1 Container Security Architecture

```
Container security: Multiple kernel mechanisms layered together.

  ┌────────────────────────────────────────────────────────┐
  │  Container Security Stack                               │
  │                                                         │
  │  Layer 7: Image Security                                │
  │    Image scanning, signed images, minimal base images   │
  │                                                         │
  │  Layer 6: MAC (AppArmor / SELinux)                      │
  │    Mandatory access control profiles per container      │
  │                                                         │
  │  Layer 5: Seccomp                                       │
  │    System call filtering (~44 dangerous syscalls blocked)│
  │                                                         │
  │  Layer 4: Capabilities                                  │
  │    Drop unnecessary capabilities. Default: 14 of 41.    │
  │                                                         │
  │  Layer 3: Cgroups                                       │
  │    Resource limits (CPU, memory, PIDs, I/O)             │
  │                                                         │
  │  Layer 2: Namespaces                                    │
  │    Mount, PID, Network, UTS, IPC, User, Cgroup          │
  │                                                         │
  │  Layer 1: Read-only Root Filesystem                     │
  │    Overlay/Union filesystem, read-only base layer       │
  │                                                         │
  │  Layer 0: Host Kernel (shared!)                         │
  │    All containers share the same kernel.                │
  │    Kernel vulnerability → all containers compromised.   │
  └────────────────────────────────────────────────────────┘

  Key insight: Containers are NOT VMs.
  VM: separate kernel per guest → hardware-level isolation
  Container: shared kernel → kernel is the attack surface
```

---

## 24.2 Docker Default Security

```
Docker applies these defaults:

  Capabilities dropped:
    Default Docker keeps only 14 capabilities:
      CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER,
      CAP_MKNOD, CAP_NET_RAW, CAP_SETGID, CAP_SETUID,
      CAP_SETFCAP, CAP_SETPCAP, CAP_NET_BIND_SERVICE,
      CAP_SYS_CHROOT, CAP_KILL, CAP_AUDIT_WRITE

    Dropped (blocked):
      CAP_SYS_ADMIN    ← most dangerous, many kernel features
      CAP_NET_ADMIN    ← network configuration
      CAP_SYS_PTRACE   ← debugging other processes
      CAP_SYS_MODULE   ← loading kernel modules
      CAP_SYS_RAWIO    ← raw I/O port access
      CAP_SYS_BOOT     ← rebooting
      ... (27 more)

  Seccomp profile:
    Blocks ~44 dangerous syscalls (mount, kexec_load, etc.)

  AppArmor profile (default):
    docker-default — restricts file access, mount, signal

  Namespaces: PID, Net, Mount, UTS, IPC (User NS optional)

  Read-only layers:
    OverlayFS — base image read-only, writable upper layer

  No new privileges:
    PR_SET_NO_NEW_PRIVS — cannot escalate via setuid binaries
```

---

## 24.3 Container Escape Vectors

```
Container escape: Break out of container to host.

  ┌──────────────────────────────────────────────────────────┐
  │  Escape Vector              │ Mitigation                  │
  ├─────────────────────────────┼────────────────────────────┤
  │ Kernel exploit              │ Keep kernel updated, use    │
  │ (shared kernel!)            │ gVisor/Kata for isolation   │
  │                             │                             │
  │ Privileged container        │ NEVER use --privileged      │
  │ (--privileged flag)         │ in production               │
  │                             │                             │
  │ Docker socket mount         │ Never mount /var/run/       │
  │ (-v /var/run/docker.sock)   │ docker.sock in container    │
  │                             │                             │
  │ CAP_SYS_ADMIN               │ Don't add this capability   │
  │ (can mount, namespace, etc.)│                             │
  │                             │                             │
  │ Host PID namespace          │ Don't use --pid=host        │
  │ (--pid=host)                │                             │
  │                             │                             │
  │ Host network                │ Don't use --network=host    │
  │ (--network=host)            │                             │
  │                             │                             │
  │ Sensitive mount             │ Don't mount /proc, /sys     │
  │ (proc/sys from host)        │ from host                   │
  │                             │                             │
  │ Device access               │ Restrict /dev access,       │
  │ (/dev/sda, /dev/mem)        │ use cgroup device controller│
  └─────────────────────────────┴────────────────────────────┘

  --privileged flag removes ALL security:
    All capabilities added
    Seccomp disabled
    AppArmor disabled
    All devices accessible
    Can mount host filesystems
    → Effectively root on host
```

---

## 24.4 Rootless Containers

```
Rootless: Run container runtime and containers as non-root user.

  Traditional Docker:
    dockerd runs as root → container escape = host root

  Rootless Docker / Podman:
    Runtime runs as unprivileged user
    Uses user namespaces: root inside = unprivileged outside
    Container escape = unprivileged user on host

  ┌──────────────────────────────────────────────┐
  │  Traditional (rootful):                       │
  │    Host root → dockerd → container root       │
  │    Escape: container root = host root ✗       │
  │                                               │
  │  Rootless:                                    │
  │    Host user (uid 1000)                       │
  │      → container runtime (uid 1000)           │
  │        → container root (mapped to uid 100000)│
  │    Escape: mapped to unprivileged uid ✓       │
  └──────────────────────────────────────────────┘

  Podman: daemon-less, rootless by default
    podman run --rm -it alpine sh
    # Running as unprivileged user, no daemon needed

  Limitations of rootless:
    Cannot bind to ports < 1024 (no CAP_NET_BIND_SERVICE)
    Performance overhead for UID mapping on overlayfs
    Cannot directly access host devices
    Some networking features require slirp4netns/pasta
```

---

## 24.5 Container Runtime Sandboxing

```
For stronger isolation beyond namespaces:

  gVisor (Google):
    User-space kernel that intercepts syscalls.
    Container syscalls → gVisor kernel → limited host syscalls.
    Reduces kernel attack surface dramatically.

    ┌──────────────┐   syscalls   ┌──────────────┐
    │ Container    │ ────────────►│ gVisor       │
    │ Application  │              │ (Sentry)     │
    └──────────────┘              │ User-space   │
                                  │ kernel       │
                                  └──────┬───────┘
                                         │ ~200 host syscalls
                                         ▼
                                  ┌──────────────┐
                                  │ Host Kernel   │
                                  └──────────────┘

  Kata Containers:
    Lightweight VM per container.
    Separate kernel per container → strongest isolation.
    Uses QEMU/Cloud Hypervisor with minimal overhead.

  Firecracker (AWS):
    MicroVM for serverless containers.
    Minimal VMM, boots in <125ms.
    Used by AWS Lambda and Fargate.

  Comparison:
    ┌──────────────┬──────────────┬──────────────┬───────────┐
    │              │ Namespaces   │ gVisor       │ Kata/VM   │
    ├──────────────┼──────────────┼──────────────┼───────────┤
    │ Isolation    │ Process      │ Syscall      │ Hardware  │
    │ Kernel       │ Shared       │ User-space   │ Separate  │
    │ Overhead     │ Minimal      │ Low-Medium   │ Medium    │
    │ Compatibility│ Full         │ ~95%         │ Full      │
    │ Escape risk  │ Kernel vuln  │ Low          │ Very low  │
    └──────────────┴──────────────┴──────────────┴───────────┘
```

---

## 24.6 Kubernetes Security

```
Kubernetes adds another layer of security controls:

  Pod Security Standards (PSS):
    Privileged: Unrestricted (development only)
    Baseline:   Minimally restrictive (prevent known escalations)
    Restricted: Heavily restricted (security best practices)

  Key security settings:
    securityContext:
      runAsNonRoot: true           # Never run as root
      readOnlyRootFilesystem: true # Read-only root
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]              # Drop all capabilities
        add: ["NET_BIND_SERVICE"]  # Add only what's needed
      seccompProfile:
        type: RuntimeDefault       # Enable seccomp

  Network Policies:
    Restrict pod-to-pod communication.
    Default: all pods can talk to all pods.
    Best practice: deny all, allow specific.

  RBAC (Role-Based Access Control):
    Control who can create/modify/delete resources.
    Least privilege for service accounts.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/nsproxy.c | Namespace proxy (container isolation) |
| kernel/cgroup/cgroup.c | Cgroup resource limits |
| kernel/seccomp.c | Seccomp system call filtering |
| security/apparmor/ | AppArmor container profiles |
| security/selinux/ | SELinux container policies |
| fs/overlayfs/ | OverlayFS (container filesystem) |

---

## Interview Questions

**Q1: How do containers achieve security isolation?**
A: Containers use multiple layered kernel mechanisms: (1) Namespaces isolate resources — PID (process visibility), Mount (filesystem), Network (network stack), User (UID mapping). (2) Cgroups limit resource consumption — prevent DoS via memory/CPU/PID exhaustion. (3) Seccomp filters dangerous system calls (~44 blocked by Docker). (4) AppArmor/SELinux profiles enforce mandatory access control. (5) Capabilities are dropped (only 14 of 41 kept by Docker). (6) Read-only root filesystem via OverlayFS. (7) NO_NEW_PRIVS prevents privilege escalation. These layer together in defense-in-depth, but all containers share the host kernel — a kernel vulnerability compromises all containers.

**Q2: Why should you never use --privileged containers in production?**
A: The `--privileged` flag removes ALL container security: all 41 capabilities are added, seccomp filtering is disabled, AppArmor/SELinux profiles are removed, all host devices become accessible, and the container can mount host filesystems. The container effectively has root-level access to the host. Any code running in the container can escape by mounting the host filesystem, loading kernel modules, or accessing /dev/mem. There's no isolation — it's equivalent to running as root on the host. Instead, add only specific capabilities needed (`--cap-add`) and keep all other security mechanisms in place.

**Q3: How do rootless containers improve security?**
A: Rootless containers run the container runtime and containers as an unprivileged user. User namespaces remap UID 0 inside the container to an unprivileged UID (e.g., 100000) on the host. If an attacker escapes the container, they land as an unprivileged user with no special host privileges. Traditional containers run docker daemon as root — container escape gives host root access. Podman is designed rootless from the start (no daemon). Trade-offs: rootless containers cannot bind privileged ports (<1024), have some filesystem performance overhead for UID mapping, and require slirp4netns/pasta for networking instead of veth bridge.

---

## Summary

- Containers layer: namespaces + cgroups + seccomp + MAC + capabilities
- Docker drops 27 capabilities, blocks ~44 syscalls, applies AppArmor profile
- Containers share host kernel — kernel vulnerability = all containers compromised
- NEVER use --privileged in production (removes all security)
- Rootless containers: user namespace maps root→unprivileged, escape = limited
- gVisor: user-space kernel intercepts syscalls (syscall-level isolation)
- Kata/Firecracker: lightweight VMs (hardware-level isolation)
- Kubernetes PSS: Privileged/Baseline/Restricted pod security standards

---

Next: [Chapter 25 — Kernel Security Hardening](Chapter_25_Kernel_Hardening.md)
