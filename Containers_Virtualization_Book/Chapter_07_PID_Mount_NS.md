# Chapter 7: PID and Mount Namespaces Deep Dive

## Learning Goals
- Understand PID namespace hierarchy and multi-level PID mapping
- Learn mount namespace propagation types
- Master the kernel implementation of PID translation
- Know how /proc works across PID namespaces

---

## 1. PID Namespace Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PID namespaces are HIERARCHICAL (tree structure)        │
  │                                                           │
  │  Host PID NS (level 0)                                   │
  │  ├── PID 1 (systemd)                                     │
  │  ├── PID 500 (dockerd)                                   │
  │  ├── PID 1001 ←───── container init                     │
  │  │   │                                                    │
  │  │   └── Container PID NS (level 1)                      │
  │  │       ├── PID 1 (container init) ← same process!     │
  │  │       ├── PID 2 (app)                                 │
  │  │       └── PID 3 (worker)                              │
  │  │                                                        │
  │  └── PID 1050 ←───── nested container init              │
  │      │                                                    │
  │      └── Container PID NS (level 1)                      │
  │          ├── PID 1 (init)                                │
  │          └── PID 2                                       │
  │              │                                            │
  │              └── Nested PID NS (level 2)                 │
  │                  ├── PID 1                               │
  │                  └── PID 2                               │
  │                                                           │
  │  Key rules:                                              │
  │  - A process has one PID in each ancestor namespace      │
  │  - Parent NS can see child NS processes (with host PID) │
  │  - Child NS CANNOT see parent NS processes               │
  │  - PID 1 in a namespace: if it dies, all processes in    │
  │    that namespace are killed with SIGKILL                │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. PID Namespace Kernel Structures

```c
/*
 * Kernel PID representation (multi-namespace aware)
 * Source: include/linux/pid.h
 */
struct pid {
    refcount_t count;
    unsigned int level;           /* namespace depth */
    spinlock_t lock;
    struct hlist_head tasks[PIDTYPE_MAX]; /* linked to task_struct */
    struct hlist_head inodes;     /* /proc inodes */
    wait_queue_head_t wait_pidfd;
    struct rcu_head rcu;
    struct upid numbers[];        /* PID numbers per level */
};

struct upid {
    int nr;                       /* PID value in this namespace */
    struct pid_namespace *ns;     /* which namespace */
};

/*
 * A process in a level-2 PID namespace has 3 upid entries:
 *
 * numbers[0] = { nr=1050, ns=&init_pid_ns }  ← host PID
 * numbers[1] = { nr=2,    ns=&container_ns }  ← container PID
 * numbers[2] = { nr=1,    ns=&nested_ns    }  ← nested PID
 *
 * PID translation:
 *   task_pid_nr(task)          → returns PID in CURRENT ns
 *   task_pid_nr_ns(task, ns)   → returns PID in specified ns
 *   task_tgid_vnr(task)        → returns TGID in current ns
 */
```

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PID Translation Example:                                │
  │                                                           │
  │  Process "nginx" running in nested container:            │
  │                                                           │
  │  struct pid {                                            │
  │    level = 2,                                            │
  │    numbers = [                                           │
  │      { nr = 3847, ns = init_pid_ns     },  ← host sees  │
  │      { nr = 15,   ns = container_pid_ns },  ← container │
  │      { nr = 1,    ns = nested_pid_ns    },  ← itself    │
  │    ]                                                     │
  │  };                                                      │
  │                                                           │
  │  getpid() inside nested container → returns 1            │
  │  Host "ps aux" → shows PID 3847                          │
  │  Parent container "ps" → shows PID 15                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Mount Namespace and Propagation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Mount Propagation Types:                                │
  │                                                           │
  │  ┌────────────┬──────────────────────────────────────┐   │
  │  │ Type       │ Behavior                             │   │
  │  ├────────────┼──────────────────────────────────────┤   │
  │  │ shared     │ Mounts propagate bidirectionally     │   │
  │  │ (MS_SHARED)│ between peer groups                  │   │
  │  ├────────────┼──────────────────────────────────────┤   │
  │  │ slave      │ Mounts propagate from master → slave │   │
  │  │ (MS_SLAVE) │ but NOT slave → master               │   │
  │  ├────────────┼──────────────────────────────────────┤   │
  │  │ private    │ No propagation at all                │   │
  │  │(MS_PRIVATE)│ Mount/unmount events stay local      │   │
  │  ├────────────┼──────────────────────────────────────┤   │
  │  │ unbindable │ Private + cannot be bind-mounted     │   │
  │  │(MS_UNBIND) │ Prevents mount namespace explosion   │   │
  │  └────────────┴──────────────────────────────────────┘   │
  │                                                           │
  │  Example: shared propagation                             │
  │                                                           │
  │  Host NS         Container NS                            │
  │  /mnt (shared)   /mnt (shared)                           │
  │     │               │                                     │
  │     ├── mount USB   │                                     │
  │     │  at /mnt/usb  │                                     │
  │     │               ├── /mnt/usb appears automatically   │
  │     │               │                                     │
  │     │               ├── mount NFS at /mnt/nfs            │
  │     ├── /mnt/nfs ───┘   propagates back to host          │
  │                                                           │
  │  Containers typically use:                               │
  │  - rprivate (recursive private) for rootfs               │
  │  - rslave for /proc, /sys (see host updates, no leak)   │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Container Mount Setup in Detail

```c
/*
 * Typical mount setup for a container (what runc does)
 */
void setup_container_mounts(const char *rootfs) {
    /* 1. Recursively make everything private
     *    (prevent mount events leaking to host) */
    mount("", "/", NULL, MS_REC | MS_PRIVATE, NULL);

    /* 2. Bind-mount the rootfs (makes it a mount point) */
    mount(rootfs, rootfs, NULL, MS_BIND | MS_REC, NULL);

    /* 3. pivot_root into the new rootfs */
    char old_root[PATH_MAX];
    snprintf(old_root, sizeof(old_root), "%s/.pivot", rootfs);
    mkdir(old_root, 0700);
    pivot_root(rootfs, old_root);
    chdir("/");

    /* 4. Mount /proc (PID namespace aware)
     *    Shows only processes in this PID namespace */
    mount("proc", "/proc", "proc",
          MS_NOSUID | MS_NODEV | MS_NOEXEC, NULL);

    /* 5. Mount certain /proc paths read-only (security)
     *    Prevents container from modifying kernel params */
    mount("/proc/sys", "/proc/sys", NULL,
          MS_BIND | MS_REC, NULL);
    mount("/proc/sys", "/proc/sys", NULL,
          MS_BIND | MS_REC | MS_RDONLY | MS_REMOUNT, NULL);

    /* 6. Mask sensitive paths (prevent reading) */
    mount("tmpfs", "/proc/acpi", "tmpfs",
          MS_NOSUID | MS_NODEV | MS_NOEXEC, NULL);
    mount("/dev/null", "/proc/kcore", NULL, MS_BIND, NULL);
    mount("/dev/null", "/proc/keys", NULL, MS_BIND, NULL);
    mount("/dev/null", "/proc/sched_debug", NULL, MS_BIND, NULL);

    /* 7. Mount /sys (cgroup aware) */
    mount("sysfs", "/sys", "sysfs",
          MS_NOSUID | MS_NODEV | MS_NOEXEC | MS_RDONLY, NULL);

    /* 8. Mount /dev */
    mount("tmpfs", "/dev", "tmpfs",
          MS_NOSUID | MS_STRICTATIME, "mode=755,size=65536k");

    /* 9. Create essential devices */
    mknod("/dev/null",    S_IFCHR | 0666, makedev(1, 3));
    mknod("/dev/zero",    S_IFCHR | 0666, makedev(1, 5));
    mknod("/dev/random",  S_IFCHR | 0666, makedev(1, 8));
    mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));

    /* 10. Mount devpts for terminal devices */
    mkdir("/dev/pts", 0755);
    mount("devpts", "/dev/pts", "devpts",
          MS_NOSUID | MS_NOEXEC,
          "newinstance,ptmxmode=0666,mode=0620");

    /* 11. Unmount old root */
    umount2("/.pivot", MNT_DETACH);
    rmdir("/.pivot");
}
```

---

## 5. PID 1 and Signal Handling in Containers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PID 1 in a PID namespace has special behavior:          │
  │                                                           │
  │  1. Signal handling:                                     │
  │     - Cannot receive signals it hasn't registered for    │
  │     - SIGKILL/SIGSTOP from WITHIN the NS are ignored    │
  │     - SIGKILL/SIGSTOP from PARENT NS work normally      │
  │     - Default signal actions are IGNORED (not default)   │
  │                                                           │
  │  2. Orphan reaping:                                      │
  │     - PID 1 inherits orphaned child processes            │
  │     - Must call wait()/waitpid() to reap zombies        │
  │     - If PID 1 doesn't reap: zombie accumulation        │
  │                                                           │
  │  3. Namespace destruction:                               │
  │     - If PID 1 dies → ALL processes in NS get SIGKILL   │
  │     - This is why containers die when PID 1 exits       │
  │     - No other process can become PID 1                  │
  │                                                           │
  │  Common problem:                                         │
  │  - Running app directly as PID 1 → doesn't reap zombies │
  │  - Solution: tini, dumb-init as PID 1 → reaps + signals │
  │                                                           │
  │  Docker:                                                 │
  │    CMD ["myapp"]              → myapp IS PID 1           │
  │    docker run --init myimage  → tini IS PID 1            │
  │                                → tini forwards signals   │
  │                                → tini reaps zombies      │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does the kernel implement PID translation across namespace levels?**
**A:** Each process has a `struct pid` with a flexible array `numbers[]` containing one `struct upid` per namespace level. For a process at level N, there are N+1 entries: numbers[0] is the PID in the initial (host) namespace, numbers[N] is the PID in the process's own namespace. When `getpid()` is called, the kernel calls `task_pid_vnr()` which looks up the upid entry matching the caller's current PID namespace. When the host runs `ps`, it reads `/proc` entries using the init_pid_ns, seeing host PIDs. Each PID namespace has its own PID allocator (idr), so PID 1 exists independently in every PID namespace. A process in a parent namespace can see child namespace processes (with parent-level PIDs), but child namespace processes cannot see or signal parent namespace processes.

**Q2: What is mount propagation and why does it matter for containers?**
**A:** Mount propagation controls whether mount/unmount events in one mount namespace are visible in other namespaces. There are 4 types: `shared` (bidirectional), `slave` (one-way: master→slave), `private` (no propagation), `unbindable` (private + cannot be bind-mounted). For containers: (1) The rootfs is set to `rprivate` (recursive private) so container mounts don't leak to the host. (2) systemd sets `/` to `rshared` by default (so mount events propagate to all mount namespaces) — container runtimes must override this. (3) Kubernetes uses `rslave` for certain paths like `/var/lib/kubelet` so that host mount updates (e.g., CSI volume mounts) propagate into pods, but pod mounts don't leak out. (4) Docker's `--mount type=bind,bind-propagation=shared` allows explicit control. The `findmnt -o TARGET,PROPAGATION` command shows propagation settings.

---

## Summary

- PID namespaces: hierarchical tree; process has one PID per ancestor NS level
- struct pid.numbers[] array: one upid per level (host PID, container PID, nested PID)
- PID 1 in namespace: special signal handling, reaps orphans, death kills all NS processes
- Mount propagation: shared (bidir), slave (one-way), private (none), unbindable
- Container rootfs: set rprivate → no mount leaks; pivot_root → swap root safely
- /proc in container: PID-namespace-aware, shows only container processes
- Sensitive /proc paths (/proc/kcore, /proc/keys) masked with bind-mount /dev/null

---

[Previous: Namespace Architecture ←](Chapter_06_Namespace_Architecture.md) | [Next: Network and User Namespaces →](Chapter_08_Net_User_NS.md)
