# Chapter 10: Namespace Lifecycle and Advanced Operations

## Learning Goals
- Master namespace persistence mechanisms
- Learn cross-namespace operations (nsenter, setns patterns)
- Understand pidfd and namespace file descriptors
- Know namespace best practices for production containers

---

## 1. Namespace Persistence

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  A namespace is destroyed when:                          │
  │  - No process is in it AND                               │
  │  - No file descriptor references it AND                  │
  │  - No bind mount holds it                                │
  │                                                           │
  │  Persistence mechanisms:                                 │
  │                                                           │
  │  1. Process membership (default):                        │
  │     Namespace stays alive while any process is in it     │
  │                                                           │
  │  2. Open file descriptor:                                │
  │     int fd = open("/proc/<pid>/ns/net", O_RDONLY);       │
  │     /* NS stays alive even after process exits */        │
  │     setns(fd, CLONE_NEWNET);  /* can join it later */    │
  │                                                           │
  │  3. Bind mount (most persistent):                        │
  │     mount --bind /proc/<pid>/ns/net /var/run/netns/myns  │
  │     /* NS survives even after all processes exit */      │
  │     /* ip netns add/exec uses this mechanism */          │
  │                                                           │
  │  4. pidfd (modern approach, kernel 5.3+):                │
  │     int pidfd = pidfd_open(pid, 0);                      │
  │     int nsfd = pidfd_getfd(pidfd, nsfd);                 │
  │     /* Race-free namespace fd acquisition */             │
  │                                                           │
  │  ┌────────────────────────────────────────────┐          │
  │  │ Why bind-mount /proc/PID/ns/TYPE?          │          │
  │  │                                             │          │
  │  │ Problem: /proc/PID disappears when PID dies│          │
  │  │ Solution: bind-mount before process exits  │          │
  │  │                                             │          │
  │  │ ip netns add myns                          │          │
  │  │   → unshare(CLONE_NEWNET)                  │          │
  │  │   → bind-mount /proc/self/ns/net           │          │
  │  │     to /var/run/netns/myns                 │          │
  │  │   → process exits, NS kept alive by mount │          │
  │  │                                             │          │
  │  │ ip netns exec myns <cmd>                   │          │
  │  │   → open("/var/run/netns/myns")            │          │
  │  │   → setns(fd, CLONE_NEWNET)                │          │
  │  │   → exec(<cmd>)                            │          │
  │  └────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. nsenter and Cross-Namespace Operations

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  nsenter: enter one or more namespaces of a target       │
  │  process and execute a command                           │
  │                                                           │
  │  # Enter all namespaces of container PID 12345:          │
  │  nsenter -t 12345 -m -u -i -n -p -- /bin/bash           │
  │                                                           │
  │  Flags:                                                  │
  │  ┌──────┬────────────────────────────────────────┐      │
  │  │ -t   │ Target PID                             │      │
  │  │ -m   │ Enter mount namespace                  │      │
  │  │ -u   │ Enter UTS namespace                    │      │
  │  │ -i   │ Enter IPC namespace                    │      │
  │  │ -n   │ Enter network namespace                │      │
  │  │ -p   │ Enter PID namespace                    │      │
  │  │ -U   │ Enter user namespace                   │      │
  │  │ -C   │ Enter cgroup namespace                 │      │
  │  │ -T   │ Enter time namespace                   │      │
  │  └──────┴────────────────────────────────────────┘      │
  │                                                           │
  │  "docker exec" implementation:                           │
  │  1. Find container init PID (via containerd API)        │
  │  2. For each namespace:                                  │
  │     fd = open("/proc/<pid>/ns/<type>", O_RDONLY)        │
  │     setns(fd, 0)                                        │
  │  3. fork() + exec() the requested command               │
  │  4. The new process is now "inside" the container       │
  │                                                           │
  │  Note: PID namespace join is special:                    │
  │  setns(fd, CLONE_NEWPID) affects CHILDREN only          │
  │  The calling process stays in its original PID NS       │
  │  Must fork() after setns to enter the PID namespace     │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. pidfd — Race-Free Process Operations

```c
/*
 * pidfd: file descriptor referring to a process (kernel 5.3+)
 * Solves PID reuse race condition
 *
 * Problem:
 *   pid_t pid = 1234;
 *   // ... PID 1234 exits and is reused by another process ...
 *   kill(pid, SIGTERM);  // kills the WRONG process!
 *
 * Solution:
 *   int pidfd = pidfd_open(1234, 0);
 *   // Even if PID 1234 is reused, pidfd still refers
 *   // to original process (or returns error if dead)
 *   pidfd_send_signal(pidfd, SIGTERM, NULL, 0);
 */

#include <sys/syscall.h>
#include <unistd.h>

/* Open a pidfd for a process */
int pidfd = syscall(SYS_pidfd_open, target_pid, 0);

/* Send signal via pidfd (race-free) */
syscall(SYS_pidfd_send_signal, pidfd, SIGTERM, NULL, 0);

/* Get namespace fd from pidfd */
/* setns() accepts pidfd directly (kernel 5.8+) */
setns(pidfd, CLONE_NEWNET);  /* join net NS of pidfd's process */

/* Wait for process via pidfd */
/* poll()/epoll() on pidfd → notified when process exits */
struct pollfd pfd = { .fd = pidfd, .events = POLLIN };
poll(&pfd, 1, -1);  /* blocks until process exits */
/* No zombie creation — unlike waitpid() */

/* clone3 with CLONE_PIDFD returns a pidfd for the child */
struct clone_args args = {
    .flags = CLONE_PIDFD | CLONE_NEWPID | CLONE_NEWNS,
    .pidfd = (uint64_t)&pidfd,
    .exit_signal = SIGCHLD,
};
pid_t child = syscall(SYS_clone3, &args, sizeof(args));
/* pidfd now refers to the child — race-free from birth */
```

---

## 4. Namespace Diagnostic Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Useful commands for namespace inspection:               │
  │                                                           │
  │  # List all namespaces on the system:                    │
  │  lsns                                                    │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │ NS TYPE   NPROCS PID  USER  COMMAND              │   │
  │  │ 40265.. mnt  100  1   root  /sbin/init           │   │
  │  │ 40265.. pid    2  500 root  /usr/bin/containerd  │   │
  │  │ 40265.. net    3  1001 root nginx: master        │   │
  │  └──────────────────────────────────────────────────┘   │
  │                                                           │
  │  # Show namespaces of a specific PID:                    │
  │  ls -la /proc/<pid>/ns/                                  │
  │                                                           │
  │  # Compare two processes' namespaces:                    │
  │  readlink /proc/1001/ns/net  → net:[4026532001]         │
  │  readlink /proc/1002/ns/net  → net:[4026532001]  SAME!  │
  │  readlink /proc/2000/ns/net  → net:[4026532100]  DIFF   │
  │                                                           │
  │  # Find which cgroup a container process is in:          │
  │  cat /proc/<pid>/cgroup                                  │
  │  0::/system.slice/docker-abc.scope  (from host view)    │
  │  0::/  (from container's cgroupns view)                 │
  │                                                           │
  │  # Inspect namespace relationships:                      │
  │  lsns --tree                                             │
  │                                                           │
  │  # Find all processes in a specific namespace:           │
  │  lsns -t net                                             │
  │                                                           │
  │  # Check current namespaces:                             │
  │  ls -la /proc/self/ns/                                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Namespace Security Considerations

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Common namespace escape vectors and mitigations:        │
  │                                                           │
  │  1. /proc and /sys exposure:                             │
  │     Attack: write to /proc/sys/kernel/* → change host   │
  │     Mitigation: mount /proc/sys read-only, mask         │
  │     sensitive paths with /dev/null bind mounts           │
  │                                                           │
  │  2. Device access:                                       │
  │     Attack: mount host disk device inside container     │
  │     Mitigation: drop CAP_SYS_ADMIN, restrict            │
  │     /dev/disk access via device cgroup controller       │
  │                                                           │
  │  3. User namespace bypass:                               │
  │     Attack: user NS gives CAP_SYS_ADMIN → mount fuse   │
  │             → exploit fuse filesystem bugs               │
  │     Mitigation: kernel restricts what can be mounted    │
  │     in user NS (no block devices, limited FS types)     │
  │                                                           │
  │  4. PID reuse:                                           │
  │     Attack: race condition between check and use of PID │
  │     Mitigation: use pidfd (race-free process references)│
  │                                                           │
  │  5. Shared kernel:                                       │
  │     Attack: kernel exploit escapes all namespaces        │
  │     Mitigation: seccomp (reduce syscall surface),       │
  │     LSM, gVisor/Kata (don't share kernel)               │
  │                                                           │
  │  Production checklist:                                   │
  │  [ ] All unnecessary capabilities dropped               │
  │  [ ] Seccomp profile applied                            │
  │  [ ] User namespaces enabled (rootless)                 │
  │  [ ] /proc/sys mounted read-only                        │
  │  [ ] Sensitive /proc paths masked                       │
  │  [ ] AppArmor/SELinux profile active                    │
  │  [ ] Resource limits set via cgroups                    │
  │  [ ] Network policy restricting traffic                 │
  │  [ ] Read-only rootfs where possible                    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does `docker exec` work internally in terms of namespaces?**
**A:** `docker exec` runs a new process inside an existing container's namespaces: (1) The Docker client sends an "exec" request to dockerd, which forwards to containerd, which calls the container runtime (runc). (2) runc finds the container init process PID (e.g., PID 1234 on the host). (3) For each namespace type, runc opens `/proc/1234/ns/<type>` (mnt, uts, ipc, net, user, cgroup, time) and calls `setns(fd, 0)` to join that namespace. (4) PID namespace is special: `setns(fd, CLONE_NEWPID)` only affects children, so runc must `fork()` after setns — the child will be in the container's PID namespace. (5) The child applies the same seccomp filter and capability set as the container init process. (6) The child calls `exec()` to run the user's command. The result: the new process shares all namespaces with the container, sees the container's filesystem, network, hostname, PIDs, etc. But it gets a new PID within the container.

**Q2: What is pidfd and what problems does it solve?**
**A:** pidfd (process file descriptor, kernel 5.3+) is a file descriptor that refers to a specific process instance, solving the PID reuse race condition. Without pidfd: you observe PID 1234, the process exits, the kernel reuses PID 1234 for a new process, and you accidentally signal/interact with the wrong process. With pidfd: `pidfd_open(1234)` returns an fd tied to that specific process; even if PID 1234 is reused, the fd still refers to the original (or returns an error if dead). Features: (1) `pidfd_send_signal()` — race-free signal delivery. (2) `poll()/epoll()` on pidfd — notification when process exits (no zombie needed). (3) `setns(pidfd, CLONE_NEWNET)` — join namespaces of pidfd's process (kernel 5.8+). (4) `clone3(CLONE_PIDFD)` — get pidfd for child at creation time, race-free from birth. This is critical for container runtimes managing many processes — the PID reuse window is a real security concern.

---

## Summary

- Namespace persistence: process membership, open fd, bind mount (most persistent)
- `ip netns` uses bind mounts at /var/run/netns/ for persistent network namespaces
- nsenter/setns: join existing namespaces; PID NS requires fork() after setns
- pidfd: race-free process reference (kernel 5.3+), replaces PID-based APIs
- clone3(CLONE_PIDFD): get pidfd for child at creation time
- Namespace diagnostics: lsns, /proc/PID/ns/, readlink comparison
- Security: mask /proc paths, drop caps, seccomp, LSM, read-only rootfs

---

[Previous: Other Namespaces ←](Chapter_09_Other_Namespaces.md) | [Next: Cgroups v1 Architecture →](Chapter_11_Cgroups_v1.md)
