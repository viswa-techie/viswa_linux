# Chapter 2: History — From chroot to Modern Containers

## Learning Goals
- Trace the evolution from chroot to Docker to Kubernetes
- Understand each generation's contribution to containerization
- Know the key Linux kernel features that enabled containers

---

## 1. Timeline of Container Technologies

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1979  chroot()           Unix V7 — filesystem isolation │
  │  2000  FreeBSD Jails      First OS-level virtualization  │
  │  2001  Linux-VServer      Kernel patches for containers  │
  │  2002  Linux namespaces   mount namespace (first)        │
  │  2006  Process containers → renamed to cgroups (Google)  │
  │  2007  cgroups merged     Linux 2.6.24                   │
  │  2008  LXC                First complete Linux container │
  │  2009  PID namespace      Linux 2.6.24                   │
  │  2012  User namespace     Linux 3.8 (rootless containers)│
  │  2013  Docker             Container revolution begins    │
  │  2014  Kubernetes         Google open-sources container   │
  │                           orchestrator                    │
  │  2015  OCI founded        Open Container Initiative      │
  │        runc released      Reference container runtime    │
  │  2016  containerd         Docker donates to CNCF         │
  │  2017  CRI-O              Kubernetes-native runtime      │
  │  2018  cgroups v2         Unified hierarchy (Linux 4.5+) │
  │  2019  Podman             Daemonless, rootless containers│
  │  2020  Time namespace     Linux 5.6                      │
  │  2022  cgroups v2 default systemd and container runtimes │
  │  2024  User namespace     Enhanced security in 6.x       │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. chroot — The Beginning (1979)

```c
/*
 * chroot() changes the root directory for a process.
 * It was the first form of filesystem isolation.
 *
 * Limitations:
 * - Root can escape with relative paths (chroot escape)
 * - No PID isolation (sees all host processes)
 * - No network isolation
 * - No resource limits
 * - Never intended as a security boundary
 */

#include <unistd.h>
#include <stdio.h>

int main(void) {
    /* Set up a minimal root filesystem */
    /* /new_root/bin/sh, /new_root/lib/... etc. */

    chroot("/new_root");
    chdir("/");  /* Must cd after chroot */

    /* Now "/" is actually /new_root on the host */
    /* Cannot access files outside /new_root ... */
    /* ... unless you are root and use escape tricks */

    execl("/bin/sh", "sh", NULL);
    return 0;
}

/*
 * chroot escape (why chroot is NOT security):
 *
 * int fd = open(".", O_RDONLY);  // save reference to CWD
 * chroot("/tmp");                // enter a new chroot
 * fchdir(fd);                    // go back to original dir
 * // Now "." is outside chroot, keep doing chdir("..") until /
 * chroot(".");                   // escape!
 *
 * pivot_root() (used by containers) prevents this by
 * actually moving the old root to a subdirectory and
 * then unmounting it.
 */
```

---

## 3. Linux Namespaces Timeline

```
  ┌────────────────────────────────────────────────────────┐
  │                                                         │
  │  Namespace     │ Kernel  │ Year │ What it isolates     │
  │  ──────────────┼─────────┼──────┼──────────────────── │
  │  Mount (mnt)   │ 2.4.19  │ 2002 │ Filesystem mounts   │
  │  UTS           │ 2.6.19  │ 2006 │ Hostname, domain    │
  │  IPC           │ 2.6.19  │ 2006 │ System V IPC, POSIX │
  │  PID           │ 2.6.24  │ 2008 │ Process ID numbers  │
  │  Network (net) │ 2.6.29  │ 2009 │ Network stack       │
  │  User          │ 3.8     │ 2013 │ UID/GID mappings    │
  │  Cgroup        │ 4.6     │ 2016 │ Cgroup view         │
  │  Time          │ 5.6     │ 2020 │ Boot/monotonic clock│
  │                                                         │
  │  Each namespace was added to solve a specific           │
  │  isolation problem. Together, they form a "container."  │
  └────────────────────────────────────────────────────────┘
```

---

## 4. From LXC to Docker to OCI

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  LXC (2008): First complete container solution           │
  │  - Combined namespaces + cgroups + chroot                │
  │  - Configuration file per container                      │
  │  - "System containers" — felt like lightweight VMs       │
  │  - Complex to set up, no image distribution              │
  │                                                           │
  │  Docker (2013): Container revolution                     │
  │  - Initially used LXC, then replaced with libcontainer  │
  │  - Key innovation: IMAGES (layered, portable)           │
  │  - Dockerfile: reproducible image builds                │
  │  - Docker Hub: image distribution registry              │
  │  - "Application containers" — one process per container │
  │  - Made containers accessible to mainstream developers  │
  │                                                           │
  │  OCI (2015): Standardization                             │
  │  - Docker donated container format + runtime            │
  │  - OCI Runtime Spec: how to run a container             │
  │  - OCI Image Spec: how to package a container image     │
  │  - OCI Distribution Spec: how to distribute images      │
  │  - runc: reference implementation of OCI runtime        │
  │                                                           │
  │  Architecture evolution:                                 │
  │                                                           │
  │  2013: Docker = monolithic daemon                       │
  │  2017: Docker → containerd + runc (split)               │
  │  2020: Kubernetes deprecates dockershim                  │
  │  Now:  Kubernetes → CRI → containerd/CRI-O → runc      │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why did Docker succeed where LXC didn't?**
**A:** LXC provided the kernel mechanics (namespaces + cgroups) but was complex to configure and lacked an image format. Docker added three innovations: (1) Layered images using union filesystems (AUFS, later overlayfs) — each layer is read-only, only the top layer is writable, making images small and shareable. (2) Dockerfile — declarative build format that made images reproducible. (3) Docker Hub — centralized registry for sharing images. Docker also simplified the UX to `docker run ubuntu` instead of manually configuring namespaces, cgroups, and mount points. Docker shifted the mental model from "lightweight VM" to "portable application package."

**Q2: What kernel features had to exist before containers were possible?**
**A:** Containers require at least: (1) PID namespace (2008, Linux 2.6.24) — so container sees PID 1 as its init. (2) Mount namespace (2002, Linux 2.4.19) — so container has its own filesystem view. (3) Network namespace (2009, Linux 2.6.29) — so container has its own IP, interfaces, routing. (4) Cgroups (2007, Linux 2.6.24) — so container's CPU/memory usage can be limited. (5) User namespace (2013, Linux 3.8) — so container can run as "root" inside but unprivileged on host (rootless). Without any of these, you only have chroot (filesystem only, easily escaped). The mount+PID+net namespaces were the minimum viable container; user namespace added rootless security.

---

## Summary

- 1979: chroot = filesystem isolation only (not secure, easily escaped)
- 2002-2020: Linux added 8 namespace types over 18 years
- 2007: cgroups (Google) added resource limiting
- 2008: LXC = first complete container (namespaces + cgroups)
- 2013: Docker = images + Dockerfile + registry → mainstream adoption
- 2015: OCI standardization → runc, containerd, CRI-O
- Modern stack: Kubernetes → CRI → containerd → runc → kernel (clone3 + cgroups)

---

[Previous: Foundations ←](Chapter_01_Foundations.md) | [Next: Container vs VM →](Chapter_03_Container_vs_VM.md)
