# Chapter 5: Container Security Model

## Learning Goals
- Understand Linux capabilities and how containers drop privileges
- Learn seccomp BPF syscall filtering
- Know how LSMs (AppArmor, SELinux) protect containers
- Master rootless containers and user namespace security

---

## 1. Linux Capabilities

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Traditional Unix: binary root (UID 0) vs non-root      │
  │  Problem: root has ALL privileges — too coarse           │
  │                                                           │
  │  Capabilities: break root privileges into ~41 individual │
  │  flags that can be independently granted/revoked         │
  │                                                           │
  │  Per-thread capability sets:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Effective   — actually checked by kernel │            │
  │  │ Permitted   — ceiling for effective      │            │
  │  │ Inheritable — across execve()            │            │
  │  │ Bounding    — limit for gaining caps     │            │
  │  │ Ambient     — auto-raised after execve() │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Capabilities Docker keeps by default (14 of ~41):      │
  │  ┌────────────────────┬─────────────────────────────┐   │
  │  │ CAP_CHOWN          │ Change file ownership       │   │
  │  │ CAP_DAC_OVERRIDE   │ Bypass file read/write perms│   │
  │  │ CAP_FSETID         │ Set setuid/setgid bits      │   │
  │  │ CAP_FOWNER         │ Bypass permission checks    │   │
  │  │ CAP_MKNOD          │ Create special files        │   │
  │  │ CAP_NET_RAW        │ Raw sockets (ping)          │   │
  │  │ CAP_SETGID         │ Set group ID                │   │
  │  │ CAP_SETUID         │ Set user ID                 │   │
  │  │ CAP_SETFCAP        │ Set file capabilities       │   │
  │  │ CAP_SETPCAP        │ Modify process capabilities │   │
  │  │ CAP_NET_BIND_SERVICE│ Bind port < 1024           │   │
  │  │ CAP_SYS_CHROOT     │ Use chroot()               │   │
  │  │ CAP_KILL           │ Send signals                │   │
  │  │ CAP_AUDIT_WRITE    │ Write to audit log          │   │
  │  └────────────────────┴─────────────────────────────┘   │
  │                                                           │
  │  Capabilities Docker DROPS:                              │
  │  CAP_SYS_ADMIN    — mount, bpf, namespace ops           │
  │  CAP_SYS_PTRACE   — trace/debug processes               │
  │  CAP_SYS_MODULE   — load kernel modules                 │
  │  CAP_SYS_RAWIO    — direct I/O to /dev/mem              │
  │  CAP_SYS_BOOT     — reboot system                       │
  │  CAP_NET_ADMIN    — network configuration               │
  │  CAP_SYS_TIME     — set system clock                    │
  │  ... (~27 more dropped)                                  │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Drop all capabilities except those needed */
#include <sys/prctl.h>
#include <linux/capability.h>

void drop_capabilities(void) {
    /* PR_SET_NO_NEW_PRIVS: no child process can gain more privs */
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);

    /* Drop bounding set — limits caps that can ever be gained */
    for (int cap = 0; cap <= CAP_LAST_CAP; cap++) {
        /* Keep only specific capabilities */
        if (cap == CAP_NET_BIND_SERVICE || cap == CAP_SETUID ||
            cap == CAP_SETGID) {
            continue;  /* keep this capability */
        }
        prctl(PR_CAPBSET_DROP, cap, 0, 0, 0);
    }
}
```

---

## 2. Seccomp BPF Syscall Filtering

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Seccomp-BPF: attach a BPF program to a thread that     │
  │  filters every syscall before it executes                │
  │                                                           │
  │  Process                                                 │
  │    │                                                     │
  │    ▼ syscall(NR, arg1, arg2, ...)                       │
  │    │                                                     │
  │    ▼                                                     │
  │  ┌─────────────────────┐                                │
  │  │  Seccomp BPF filter │                                │
  │  │  ┌───────────────┐  │                                │
  │  │  │ if NR == 165  │  │  mount() → EPERM              │
  │  │  │ if NR == 175  │  │  init_module() → KILL         │
  │  │  │ if NR == 161  │  │  chroot() → EPERM             │
  │  │  │ if NR == 101  │  │  ptrace() → EPERM             │
  │  │  │ else          │  │  → ALLOW                      │
  │  │  └───────────────┘  │                                │
  │  └─────────┬───────────┘                                │
  │            │ allowed?                                    │
  │            ▼                                             │
  │    ┌───────────────┐                                    │
  │    │ Kernel executes│                                   │
  │    │ the syscall   │                                    │
  │    └───────────────┘                                    │
  │                                                           │
  │  Filter decisions:                                       │
  │  SECCOMP_RET_ALLOW   — let syscall proceed              │
  │  SECCOMP_RET_ERRNO   — return error (e.g., -EPERM)     │
  │  SECCOMP_RET_KILL    — kill the thread immediately      │
  │  SECCOMP_RET_TRAP    — send SIGSYS to process          │
  │  SECCOMP_RET_TRACE   — notify ptrace tracer            │
  │  SECCOMP_RET_LOG     — allow but log                   │
  │  SECCOMP_RET_USER_NOTIF — notify userspace handler     │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Install a seccomp filter blocking mount() and reboot() */
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <sys/prctl.h>
#include <sys/syscall.h>

void install_seccomp_filter(void) {
    struct sock_filter filter[] = {
        /* Load syscall number */
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
                 offsetof(struct seccomp_data, nr)),

        /* Block mount (165 on x86_64) */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_mount, 0, 1),
        BPF_STMT(BPF_RET | BPF_K,
                 SECCOMP_RET_ERRNO | (EPERM & SECCOMP_RET_DATA)),

        /* Block reboot (169 on x86_64) */
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_reboot, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),

        /* Allow everything else */
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    };

    struct sock_fprog prog = {
        .len    = sizeof(filter) / sizeof(filter[0]),
        .filter = filter,
    };

    /* Must set no-new-privs before installing filter */
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    syscall(__NR_seccomp, SECCOMP_SET_MODE_FILTER, 0, &prog);
}
```

---

## 3. LSM Protection (AppArmor, SELinux)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Linux Security Modules — hooks in every security-       │
  │  sensitive kernel path                                   │
  │                                                           │
  │  ┌─────────────┬───────────────────────────────────┐    │
  │  │ LSM         │ How it protects containers         │    │
  │  ├─────────────┼───────────────────────────────────┤    │
  │  │ AppArmor    │ Path-based MAC policy per container│    │
  │  │ (Ubuntu)    │ e.g., deny /etc/shadow rw          │    │
  │  │             │ deny mount,                        │    │
  │  │             │ deny ptrace                        │    │
  │  ├─────────────┼───────────────────────────────────┤    │
  │  │ SELinux     │ Label-based MAC policy              │    │
  │  │ (RHEL)      │ container_t → cannot access         │    │
  │  │             │ host files labeled host_t           │    │
  │  │             │ Even if root escapes namespace,     │    │
  │  │             │ SELinux still blocks access         │    │
  │  ├─────────────┼───────────────────────────────────┤    │
  │  │ Landlock    │ Unprivileged sandboxing (5.13+)    │    │
  │  │             │ Restrict file access without root  │    │
  │  │             │ Stackable with AppArmor/SELinux    │    │
  │  └─────────────┴───────────────────────────────────┘    │
  │                                                           │
  │  Defense in depth — layers of container security:        │
  │                                                           │
  │  ┌─────────────────────────────────────┐                 │
  │  │ Namespaces  — visibility isolation  │  Layer 1       │
  │  ├─────────────────────────────────────┤                 │
  │  │ Cgroups     — resource limits       │  Layer 2       │
  │  ├─────────────────────────────────────┤                 │
  │  │ Capabilities— privilege reduction   │  Layer 3       │
  │  ├─────────────────────────────────────┤                 │
  │  │ Seccomp     — syscall filtering     │  Layer 4       │
  │  ├─────────────────────────────────────┤                 │
  │  │ LSM         — mandatory access ctrl │  Layer 5       │
  │  ├─────────────────────────────────────┤                 │
  │  │ User NS     — UID remapping         │  Layer 6       │
  │  └─────────────────────────────────────┘                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Rootless Containers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Traditional (rootful):                                  │
  │  Docker daemon runs as root → container process is root  │
  │  UID 0 inside = UID 0 on host = DANGEROUS               │
  │                                                           │
  │  Rootless containers:                                    │
  │  User namespace maps container UID 0 → host UID 100000  │
  │  No root daemon needed                                   │
  │                                                           │
  │  ┌────────────────────────────┐                          │
  │  │ Container (User Namespace) │                          │
  │  │ UID 0 (root inside)       │                          │
  │  │ → mapped to host UID 100000                          │
  │  │                            │                          │
  │  │ UID 1 → host UID 100001  │                          │
  │  │ UID 2 → host UID 100002  │                          │
  │  │ ...                       │                          │
  │  └────────────────────────────┘                          │
  │                                                           │
  │  /proc/<pid>/uid_map:                                    │
  │  ┌──────────────────────────────────┐                   │
  │  │ Inside-UID  Host-UID  Count     │                   │
  │  │ 0           100000    65536     │                   │
  │  └──────────────────────────────────┘                   │
  │                                                           │
  │  Tools: Podman (rootless by default), Docker rootless    │
  │  mode, Buildah, runc --rootless                          │
  │                                                           │
  │  Limitations:                                            │
  │  - Cannot bind ports < 1024 without cap                 │
  │  - Some networking features limited (no raw sockets)    │
  │  - Overlayfs needs kernel 5.11+ for unprivileged use    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the defense-in-depth security model for containers.**
**A:** Container security uses 6 independent layers, each catching what others miss: (1) **Namespaces** — visibility isolation; container cannot see host processes, network, or filesystems (PID, net, mnt, UTS, IPC, user namespaces). (2) **Cgroups** — resource accounting and limits; prevents DoS (memory bomb, fork bomb, CPU starvation). (3) **Capabilities** — privilege reduction; drop from ~41 capabilities to ~14 (Docker default); blocks mount, kernel module loading, raw I/O. (4) **Seccomp BPF** — syscall filtering; Docker's default filter blocks ~44 dangerous syscalls (mount, reboot, init_module, swapon, etc.); BPF program checked on every syscall entry. (5) **LSM (AppArmor/SELinux)** — mandatory access control; even if all other layers fail, LSM policy restricts file access by path (AppArmor) or label (SELinux). (6) **User namespaces** — UID remapping; container root (UID 0) maps to unprivileged host UID; even if container escapes to host, it's unprivileged. Each layer is independent — a bypass of one layer doesn't defeat the others.

**Q2: What is PR_SET_NO_NEW_PRIVS and why is it required before seccomp?**
**A:** `prctl(PR_SET_NO_NEW_PRIVS, 1)` is an irreversible per-thread flag ensuring the thread (and its children) can never gain more privileges than currently held. It prevents: (1) setuid/setgid binaries from running with elevated privileges. (2) File capabilities from granting new capabilities on execve(). (3) LSM transitions to more privileged domains. The kernel requires this flag before installing an unprivileged seccomp filter because otherwise a malicious user could install a filter that maps specific syscall return values to signals, then exec a setuid binary — using the filter to probe the setuid program's behavior (a confused deputy attack). With no-new-privs, the setuid binary runs without privilege elevation, making the attack pointless.

---

## Summary

- Linux capabilities: 41 fine-grained privilege flags replacing monolithic root
- Docker keeps 14 caps (basic ops), drops 27 (mount, modules, raw I/O)
- Seccomp BPF: BPF program filters every syscall — ALLOW, ERRNO, KILL, TRAP
- PR_SET_NO_NEW_PRIVS: irreversible flag, required before unprivileged seccomp
- LSMs (AppArmor, SELinux): mandatory access control, last line of defense
- Rootless containers: user namespace remaps UID 0 → unprivileged host UID
- Defense in depth: 6 independent layers each block different attack vectors

---

[Previous: Building Blocks ←](Chapter_04_Building_Blocks.md) | [Next: Namespace Architecture →](Chapter_06_Namespace_Architecture.md)
