# Chapter 1: Foundations of OS-Level Virtualization

## Learning Goals
- Understand the difference between OS-level virtualization and hardware virtualization
- Learn why containers are lighter than VMs
- Know the kernel primitives that enable containerization
- Grasp isolation vs security trade-offs

---

## 1. What is OS-Level Virtualization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Traditional Computing:                                  │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
  │  │   App A    │  │   App B    │  │   App C    │        │
  │  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘        │
  │         └────────────────┼────────────────┘              │
  │                          ▼                                │
  │                 ┌─────────────────┐                      │
  │                 │  Single OS      │                      │
  │                 │  (shared kernel)│                      │
  │                 └────────┬────────┘                      │
  │                          ▼                                │
  │                 ┌─────────────────┐                      │
  │                 │    Hardware      │                      │
  │                 └─────────────────┘                      │
  │                                                           │
  │  OS-Level Virtualization (Containers):                   │
  │  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
  │  │ Container A│  │ Container B│  │ Container C│        │
  │  │ (isolated  │  │ (isolated  │  │ (isolated  │        │
  │  │  view of   │  │  view of   │  │  view of   │        │
  │  │  the OS)   │  │  the OS)   │  │  the OS)   │        │
  │  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘        │
  │         └────────────────┼────────────────┘              │
  │                          ▼                                │
  │                 ┌─────────────────┐                      │
  │                 │  SINGLE Kernel  │  ← shared!           │
  │                 │  (namespaces +  │                      │
  │                 │   cgroups)      │                      │
  │                 └────────┬────────┘                      │
  │                          ▼                                │
  │                 ┌─────────────────┐                      │
  │                 │    Hardware      │                      │
  │                 └─────────────────┘                      │
  │                                                           │
  │  Key insight: containers share the HOST kernel           │
  │  They are just processes with restricted views           │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. The Three Pillars of Containers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Pillar 1: NAMESPACES — What a process can SEE          │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ PID namespace:   sees only its own process tree  │    │
  │  │ Mount namespace: sees its own filesystem layout  │    │
  │  │ Net namespace:   sees its own network interfaces │    │
  │  │ User namespace:  sees its own UID/GID mappings   │    │
  │  │ UTS namespace:   sees its own hostname            │    │
  │  │ IPC namespace:   sees its own shared memory       │    │
  │  │ Cgroup namespace:sees its own cgroup hierarchy   │    │
  │  │ Time namespace:  sees its own boot/monotonic time│    │
  │  └─────────────────────────────────────────────────┘    │
  │                                                           │
  │  Pillar 2: CGROUPS — What a process can USE             │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ cpu controller:    limit CPU time/shares         │    │
  │  │ memory controller: limit RAM usage               │    │
  │  │ io controller:     limit disk I/O bandwidth      │    │
  │  │ pids controller:   limit number of processes     │    │
  │  │ cpuset controller: pin to specific CPU cores     │    │
  │  └─────────────────────────────────────────────────┘    │
  │                                                           │
  │  Pillar 3: SECURITY — What a process can DO             │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ Capabilities:  fine-grained root privileges      │    │
  │  │ Seccomp:       restrict allowed system calls     │    │
  │  │ LSM (SELinux): mandatory access control          │    │
  │  │ Read-only rootfs: prevent filesystem modification│    │
  │  │ No new privileges: prevent privilege escalation  │    │
  │  └─────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Container vs Process vs VM

```c
/*
 * A container IS a process (or group of processes)
 * with restricted views and resource limits.
 *
 * Proof: you can see container processes from the host:
 *
 *   Host PID namespace:
 *   PID 1     init
 *   PID 1000  containerd
 *   PID 1234  /bin/nginx    ← container process!
 *   PID 1235  nginx: worker ← container child!
 *
 *   Container PID namespace:
 *   PID 1     /bin/nginx    ← sees itself as PID 1
 *   PID 2     nginx: worker
 *
 * The container processes run directly on the host CPU.
 * No CPU emulation or virtualization.
 * Syscalls go directly to the host kernel.
 */

/* Creating a minimal "container" with just clone() */
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int child_func(void *arg) {
    /* This process has its own PID and UTS namespace */
    sethostname("container", 9);

    printf("Inside container: PID=%d hostname=container\n", getpid());
    /* PID is 1 inside the new PID namespace */

    execl("/bin/sh", "sh", NULL);
    return 0;
}

int main(void) {
    char stack[65536];

    int flags = CLONE_NEWPID   /* New PID namespace */
              | CLONE_NEWUTS   /* New hostname */
              | CLONE_NEWNS    /* New mount namespace */
              | SIGCHLD;

    pid_t pid = clone(child_func, stack + sizeof(stack), flags, NULL);
    if (pid < 0) { perror("clone"); return 1; }

    printf("Host: container PID=%d\n", pid);  /* Host PID */
    waitpid(pid, NULL, 0);
    return 0;
}
```

---

## 4. Kernel Primitives for Containers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  System calls used by container runtimes:                │
  │                                                           │
  │  ┌────────────────┬────────────────────────────────┐    │
  │  │ clone3()       │ Create process in new           │    │
  │  │                │ namespaces (CLONE_NEW*)          │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ unshare()      │ Move current process to new     │    │
  │  │                │ namespaces                       │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ setns()        │ Join an existing namespace       │    │
  │  │                │ (docker exec uses this)         │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ pivot_root()   │ Change root filesystem           │    │
  │  │                │ (stronger than chroot)           │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ mount()        │ Set up container filesystem      │    │
  │  │                │ (proc, sys, tmpfs, overlayfs)   │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ seccomp()      │ Install syscall filter           │    │
  │  │                │ (BPF program)                   │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ prctl()        │ Set NO_NEW_PRIVS                 │    │
  │  │                │ Drop capabilities                │    │
  │  ├────────────────┼────────────────────────────────┤    │
  │  │ cgroup fs      │ Write to /sys/fs/cgroup/ to     │    │
  │  │ operations     │ set resource limits              │    │
  │  └────────────────┴────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is a container at the kernel level?**
**A:** A container is one or more regular Linux processes running with restricted views (namespaces) and resource limits (cgroups). The container shares the host kernel — there is no separate kernel or OS. The kernel uses namespaces to give each container its own view of PID space, network interfaces, mounts, hostname, etc. Cgroups limit how much CPU, memory, and I/O the container can consume. Security is enforced via capabilities (dropping root privileges), seccomp (restricting system calls), and optionally SELinux/AppArmor (mandatory access control). The container runtime (runc) creates a process using `clone3()` with `CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | ...`, sets up the filesystem via `pivot_root()`, applies cgroup limits, installs a seccomp filter, and then `exec()`s the container's entrypoint. From the host's perspective, container processes are just regular processes visible via `ps` and `/proc`.

**Q2: How does OS-level virtualization differ from hardware virtualization?**
**A:** OS-level (containers): processes share the host kernel, no CPU virtualization overhead, startup in milliseconds, memory overhead is negligible (just process memory), cannot run a different OS kernel. Hardware virtualization (VMs): each VM runs its own kernel on emulated/virtualized hardware, CPU traps on privileged instructions (VM-exit), startup takes seconds, memory overhead of full OS (100MB+ per VM), can run any OS (Linux, Windows, etc.). Containers are ~10-100x faster to start, use less memory, and have near-native CPU performance. VMs provide stronger isolation (separate kernel, MMU isolation) — a kernel vulnerability in one VM doesn't affect another. Containers share one kernel, so a kernel exploit affects all containers. For security-critical workloads: VMs or microVMs (Firecracker). For density and speed: containers.

---

## Summary

- Container = process(es) with restricted views (namespaces) and resource limits (cgroups)
- Three pillars: namespaces (what you see), cgroups (what you can use), security (what you can do)
- Containers share the host kernel — no virtualization overhead
- Key syscalls: clone3/unshare (namespaces), pivot_root (filesystem), seccomp (security)
- 8 namespace types: PID, Mount, Net, User, UTS, IPC, Cgroup, Time
- Trade-off: containers = faster + lighter; VMs = stronger isolation

---

[Next Chapter: History →](Chapter_02_History.md)
