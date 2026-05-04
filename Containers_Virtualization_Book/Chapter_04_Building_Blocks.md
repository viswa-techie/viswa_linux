# Chapter 4: Linux Container Building Blocks

## Learning Goals
- Understand the kernel structures behind namespaces and cgroups
- Learn the system call interface for container creation
- Master overlayfs for container image layers
- Know pivot_root vs chroot for filesystem isolation

---

## 1. Creating a Container Step by Step

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  What runc does to create a container:                   │
  │                                                           │
  │  1. clone3(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET    │
  │           | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER) │
  │     → child process in new namespaces                    │
  │                                                           │
  │  2. Set up user namespace UID/GID mappings               │
  │     → write to /proc/PID/uid_map and gid_map            │
  │                                                           │
  │  3. Mount container filesystem                           │
  │     → overlayfs layers (lower=image, upper=writable)    │
  │     → mount proc, sysfs, devtmpfs, tmpfs                │
  │                                                           │
  │  4. pivot_root(new_root, put_old)                        │
  │     → switch root filesystem                             │
  │     → unmount old root                                   │
  │                                                           │
  │  5. Set up network (veth pair + bridge)                  │
  │     → create veth in host net namespace                  │
  │     → move one end into container net namespace          │
  │                                                           │
  │  6. Set hostname (sethostname)                           │
  │                                                           │
  │  7. Apply cgroup limits                                  │
  │     → write PID to /sys/fs/cgroup/container_cg/cgroup.procs│
  │     → set memory.max, cpu.max, pids.max                 │
  │                                                           │
  │  8. Apply seccomp filter                                 │
  │     → install BPF program to restrict syscalls           │
  │                                                           │
  │  9. Drop capabilities                                    │
  │     → keep only needed capabilities                      │
  │     → set PR_SET_NO_NEW_PRIVS                            │
  │                                                           │
  │  10. exec() the container entrypoint                     │
  │     → becomes PID 1 inside the container                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Overlayfs for Container Images

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Container image = stack of read-only layers             │
  │  + one writable layer on top                             │
  │                                                           │
  │  ┌─────────────────────────────┐  upperdir (writable)   │
  │  │  Modified/new files         │  ← container writes    │
  │  ├─────────────────────────────┤  here                   │
  │  │  Layer 3: app code (RO)    │  lowerdir (read-only)  │
  │  ├─────────────────────────────┤                         │
  │  │  Layer 2: dependencies (RO)│                          │
  │  ├─────────────────────────────┤                         │
  │  │  Layer 1: base OS (RO)     │  e.g., ubuntu:22.04    │
  │  └─────────────────────────────┘                         │
  │                                                           │
  │  ┌─────────────────────────────┐  merged view           │
  │  │  Unified filesystem view   │  ← what the container  │
  │  │  (union of all layers)     │     actually sees       │
  │  └─────────────────────────────┘                         │
  │                                                           │
  │  How overlayfs works:                                    │
  │  - Read: search upperdir first, then lowerdirs          │
  │  - Write new file: created in upperdir                  │
  │  - Modify existing: copy-up from lower to upper, modify│
  │  - Delete: whiteout file in upperdir (hides lower)      │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Mount overlayfs manually (what container runtimes do)
mount -t overlay overlay \
    -o lowerdir=/layers/base:/layers/deps:/layers/app,\
       upperdir=/container/upper,\
       workdir=/container/work \
    /container/merged

# Result: /container/merged shows unified view
# Writes go to /container/upper
# Reads fall through to lower layers
```

---

## 3. pivot_root vs chroot

```c
/*
 * chroot: changes the root directory reference
 * - Old root is still accessible (can escape)
 * - Does not affect mount namespace
 * - Processes can break out if they have CAP_SYS_CHROOT
 *
 * pivot_root: swaps the root mount
 * - Old root is moved to a subdirectory (then unmounted)
 * - Old root is completely inaccessible
 * - Used by ALL modern container runtimes
 */

#include <sys/mount.h>
#include <sys/syscall.h>
#include <unistd.h>

static int pivot_root(const char *new_root, const char *put_old) {
    return syscall(SYS_pivot_root, new_root, put_old);
}

void setup_container_rootfs(const char *rootfs) {
    /* 1. Make rootfs a mount point (required by pivot_root) */
    mount(rootfs, rootfs, NULL, MS_BIND | MS_REC, NULL);

    /* 2. Create directory for old root */
    char put_old[256];
    snprintf(put_old, sizeof(put_old), "%s/.old_root", rootfs);
    mkdir(put_old, 0700);

    /* 3. Pivot: new_root becomes /, old root goes to .old_root */
    pivot_root(rootfs, put_old);

    /* 4. Change to new root */
    chdir("/");

    /* 5. Unmount old root (now at /.old_root) — completely gone */
    umount2("/.old_root", MNT_DETACH);
    rmdir("/.old_root");

    /* 6. Mount pseudo-filesystems */
    mount("proc",  "/proc",  "proc",    0, NULL);
    mount("sysfs", "/sys",   "sysfs",   0, NULL);
    mount("tmpfs", "/tmp",   "tmpfs",   0, NULL);
    mount("devtmpfs", "/dev", "devtmpfs", 0, NULL);
}
```

---

## 4. Container Networking (veth + bridge)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Host Network Namespace:                                 │
  │  ┌──────────────────────────────────────────┐           │
  │  │  eth0 (physical NIC)                      │           │
  │  │    │                                      │           │
  │  │  docker0 (bridge)                         │           │
  │  │    │          │          │                 │           │
  │  │  vethA1     vethB1     vethC1             │           │
  │  └────┬──────────┬──────────┬────────────────┘           │
  │       │          │          │                              │
  │  ┌────▼────┐┌────▼────┐┌────▼────┐                      │
  │  │Container││Container││Container│                      │
  │  │   A     ││   B     ││   C     │                      │
  │  │ vethA0  ││ vethB0  ││ vethC0  │                      │
  │  │(=eth0)  ││(=eth0)  ││(=eth0)  │                      │
  │  │172.17.  ││172.17.  ││172.17.  │                      │
  │  │ 0.2     ││ 0.3     ││ 0.4     │                      │
  │  └─────────┘└─────────┘└─────────┘                      │
  │                                                           │
  │  veth pair: virtual Ethernet cable                       │
  │  - One end in host namespace → attached to bridge       │
  │  - Other end moved to container namespace → becomes eth0│
  │  - Bridge does L2 switching between containers          │
  │  - NAT (iptables) for outgoing traffic                  │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through the steps a container runtime takes to create a container.**
**A:** (1) `clone3()` with CLONE_NEWPID, CLONE_NEWNS, CLONE_NEWNET, CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWUSER — creates child process in new namespaces. (2) Write UID/GID mappings to `/proc/PID/uid_map` and `gid_map` (map container root=0 to host unprivileged user). (3) Mount overlayfs: combine read-only image layers (lowerdir) with a writable layer (upperdir) into a merged view. (4) `pivot_root()` — swap the root filesystem to the container's merged view; unmount old root. (5) Mount pseudo-filesystems: /proc, /sys, /dev, /tmp. (6) Create veth pair, attach one end to host bridge, move other end to container network namespace. (7) Write container PID to cgroup procs file; set resource limits (memory.max, cpu.max, pids.max). (8) Install seccomp BPF filter to restrict syscalls. (9) Drop unneeded capabilities, set `PR_SET_NO_NEW_PRIVS`. (10) `exec()` the container entrypoint — becomes PID 1 inside the container.

**Q2: How does overlayfs work for container images?**
**A:** Overlayfs creates a union filesystem from multiple layers: multiple read-only lower directories (the image layers) and one writable upper directory (the container's changes). The merged directory shows a unified view. Read operations search upperdir first, then each lowerdir in order. New file writes go directly to upperdir. Modifying an existing file triggers a "copy-up": the file is copied from lowerdir to upperdir, then modified in place. Deleting a file creates a "whiteout" (character device 0,0) in upperdir — this hides the lower file from the merged view. This design enables image layer sharing: if 100 containers use the same `ubuntu:22.04` base layer, only one copy exists on disk; each container only stores its own modifications in its upperdir.

---

## Summary

- Container creation: clone3 + mount overlayfs + pivot_root + cgroup limits + seccomp + exec
- Overlayfs: union of read-only image layers + writable upper layer, copy-up on modify
- pivot_root: moves old root to subdirectory then unmounts it (unlike chroot which is escapable)
- Networking: veth pair (virtual cable) + bridge (L2 switch) + NAT (iptables)
- All steps are ordinary Linux syscalls — container runtimes are just userspace programs

---

[Previous: Container vs VM ←](Chapter_03_Container_vs_VM.md) | [Next: Container Security →](Chapter_05_Container_Security.md)
