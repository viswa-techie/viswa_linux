# Chapter 9: UTS, IPC, Cgroup, and Time Namespaces

## Learning Goals
- Understand UTS namespace hostname isolation
- Learn IPC namespace and SysV/POSIX IPC isolation
- Master cgroup namespace and virtualized /proc/self/cgroup view
- Know time namespace for container boot time isolation

---

## 1. UTS Namespace

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  UTS = "Unix Time-Sharing" (historical name)             │
  │  Isolates: hostname and NIS domain name                  │
  │                                                           │
  │  Host:                                                   │
  │  # hostname → "production-server-01"                     │
  │                                                           │
  │  Container A (own UTS namespace):                        │
  │  # hostname → "webapp-container"                         │
  │  # sethostname("new-name") → only changes container     │
  │                                                           │
  │  struct uts_namespace {                                   │
  │      struct new_utsname name;  /* hostname, domainname */│
  │      struct user_namespace *user_ns;  /* owner */        │
  │      struct ucounts *ucounts;                            │
  │      struct ns_common ns;                                 │
  │  };                                                       │
  │                                                           │
  │  struct new_utsname {                                    │
  │      char sysname[65];    /* "Linux"           */        │
  │      char nodename[65];   /* hostname          */        │
  │      char release[65];    /* "5.15.0"          */        │
  │      char version[65];    /* "#1 SMP ..."      */        │
  │      char machine[65];    /* "x86_64"          */        │
  │      char domainname[65]; /* NIS domain        */        │
  │  };                                                       │
  │                                                           │
  │  Note: sysname, release, version, machine are NOT        │
  │  namespace-aware — they reflect the host kernel.         │
  │  Only nodename and domainname are per-namespace.         │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. IPC Namespace

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  IPC namespace isolates inter-process communication      │
  │  objects:                                                 │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ SysV IPC:                                │            │
  │  │   - Shared memory segments (shmget)      │            │
  │  │   - Semaphore arrays (semget)            │            │
  │  │   - Message queues (msgget)              │            │
  │  │                                          │            │
  │  │ POSIX message queues:                    │            │
  │  │   - mq_open(), mq_send(), mq_receive()  │            │
  │  │   - Mounted at /dev/mqueue               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Without IPC NS:                                         │
  │    Container A: shmget(key=1234) → segment S1            │
  │    Container B: shmget(key=1234) → attaches to SAME S1! │
  │    → Data leak between containers!                       │
  │                                                           │
  │  With IPC NS:                                            │
  │    Container A: shmget(key=1234) → segment S1 (A's NS)  │
  │    Container B: shmget(key=1234) → segment S2 (B's NS)  │
  │    → Completely independent IPC objects                   │
  │                                                           │
  │  struct ipc_namespace {                                  │
  │      struct ipc_ids  ids[3]; /* sem, msg, shm */         │
  │      int sem_ctls[4];        /* semaphore limits */      │
  │      int msg_ctlmax;         /* max msg size */          │
  │      int msg_ctlmnb;         /* max queue bytes */       │
  │      int msg_ctlmni;         /* max queues */            │
  │      int shm_ctlmax;         /* max segment size */      │
  │      int shm_ctlall;         /* max total shared mem */  │
  │      int shm_ctlmni;         /* max segments */          │
  │      struct user_namespace *user_ns;                     │
  │      struct ns_common ns;                                 │
  │  };                                                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Cgroup Namespace

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cgroup namespace: virtualizes /proc/self/cgroup         │
  │  Added in kernel 4.6 (2016)                              │
  │                                                           │
  │  Without cgroup NS:                                      │
  │  Container reads /proc/self/cgroup and sees:             │
  │    0::/system.slice/docker-abc123.scope                  │
  │    → Reveals host's cgroup structure!                    │
  │    → Container knows it's in Docker                     │
  │    → Container knows host's systemd layout              │
  │                                                           │
  │  With cgroup NS:                                         │
  │  Container reads /proc/self/cgroup and sees:             │
  │    0::/                                                   │
  │    → Thinks it's at the root of the cgroup tree         │
  │    → Cannot discover host cgroup layout                 │
  │                                                           │
  │  Kernel implementation:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ struct cgroup_namespace {                 │            │
  │  │     struct ns_common ns;                  │            │
  │  │     struct user_namespace *user_ns;       │            │
  │  │     struct ucounts *ucounts;              │            │
  │  │     struct css_set *root_cset;            │            │
  │  │     /* ↑ the cgroup that appears as "/" */│            │
  │  │ };                                        │            │
  │  │                                           │            │
  │  │ When reading /proc/self/cgroup:           │            │
  │  │ 1. Get task's actual cgroup path          │            │
  │  │ 2. Get cgroup NS root_cset path           │            │
  │  │ 3. Strip root_cset prefix from path       │            │
  │  │ 4. Return relative path (or "/" if same)  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Real path: /sys/fs/cgroup/system.slice/docker-abc.scope│
  │  NS root:   /sys/fs/cgroup/system.slice/docker-abc.scope│
  │  Displayed:  /                                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Time Namespace

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Time namespace: added in kernel 5.6 (2020)              │
  │  Isolates: CLOCK_MONOTONIC and CLOCK_BOOTTIME            │
  │                                                           │
  │  Problem:                                                │
  │  - Container migrated from host A to host B              │
  │  - Host B has been up for 100 days                       │
  │  - Container expects CLOCK_BOOTTIME to be 5 days        │
  │  - Without time NS: container sees 100 days → confusion │
  │                                                           │
  │  With time NS:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /proc/<pid>/timens_offsets:              │            │
  │  │   monotonic  -8640000  0                 │            │
  │  │   boottime   -8208000  0                 │            │
  │  │                                          │            │
  │  │ offset = (container time) - (host time)  │            │
  │  │ Applied to clock_gettime() results       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  NOT isolated (still shows host time):                   │
  │  - CLOCK_REALTIME (wall clock) — would break TLS certs  │
  │  - CLOCK_TAI (atomic time) — not virtualized             │
  │                                                           │
  │  struct time_namespace {                                 │
  │      struct user_namespace *user_ns;                     │
  │      struct ucounts *ucounts;                            │
  │      struct ns_common ns;                                 │
  │      struct timens_offsets offsets;                       │
  │      struct page *vvar_page;  /* VDSO optimization */   │
  │  };                                                       │
  │                                                           │
  │  VDSO integration:                                       │
  │  - VDSO normally reads time without syscall              │
  │  - Time NS patches VDSO vvar page per-namespace        │
  │  - clock_gettime() in container still uses VDSO         │
  │  - No performance penalty for time virtualization!       │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Complete Namespace Isolation Example

```c
/*
 * Create a fully isolated container with ALL 8 namespaces
 */
#define _GNU_SOURCE
#include <sched.h>
#include <sys/mount.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <signal.h>

#define ALL_NS_FLAGS (CLONE_NEWUSER | CLONE_NEWPID | CLONE_NEWNS | \
                      CLONE_NEWNET | CLONE_NEWUTS | CLONE_NEWIPC | \
                      CLONE_NEWCGROUP | CLONE_NEWTIME)

static int container_main(void *arg) {
    const char *rootfs = (const char *)arg;

    /* Set hostname (UTS namespace) */
    sethostname("container", 9);

    /* Set up mount namespace */
    mount("", "/", NULL, MS_REC | MS_PRIVATE, NULL);
    mount(rootfs, rootfs, NULL, MS_BIND | MS_REC, NULL);

    /* pivot_root */
    char old_root[256];
    snprintf(old_root, sizeof(old_root), "%s/.old", rootfs);
    mkdir(old_root, 0700);
    syscall(SYS_pivot_root, rootfs, old_root);
    chdir("/");

    /* Mount /proc (PID namespace aware) */
    mount("proc", "/proc", "proc", 0, NULL);

    /* Mount /sys (read-only) */
    mount("sysfs", "/sys", "sysfs", MS_RDONLY, NULL);

    /* Unmount old root */
    umount2("/.old", MNT_DETACH);
    rmdir("/.old");

    /* Now fully isolated in all 8 namespaces */
    execl("/bin/sh", "sh", NULL);
    return 1;
}

int main(int argc, char *argv[]) {
    char stack[1024 * 1024];

    pid_t pid = clone(container_main,
                      stack + sizeof(stack),
                      ALL_NS_FLAGS | SIGCHLD,
                      argv[1]);  /* rootfs path */

    /* Write UID/GID mappings for user namespace */
    char path[64], map[64];
    snprintf(path, sizeof(path), "/proc/%d/uid_map", pid);
    snprintf(map, sizeof(map), "0 %d 1\n", getuid());
    int fd = open(path, O_WRONLY);
    write(fd, map, strlen(map));
    close(fd);

    snprintf(path, sizeof(path), "/proc/%d/setgroups", pid);
    fd = open(path, O_WRONLY);
    write(fd, "deny\n", 5);
    close(fd);

    snprintf(path, sizeof(path), "/proc/%d/gid_map", pid);
    snprintf(map, sizeof(map), "0 %d 1\n", getgid());
    fd = open(path, O_WRONLY);
    write(fd, map, strlen(map));
    close(fd);

    waitpid(pid, NULL, 0);
    return 0;
}
```

---

## Interview Questions

**Q1: Why was the cgroup namespace added, and what does it virtualize?**
**A:** Before cgroup namespace (kernel 4.6), a container reading `/proc/self/cgroup` would see the full host cgroup path, e.g., `0::/system.slice/docker-abc123.scope`. This leaks information: the container knows it's running in Docker, knows the host uses systemd, and can potentially infer the host's cgroup layout. The cgroup namespace virtualizes this view by establishing a root point: the container's cgroup becomes "/" from its perspective. Implementation: `struct cgroup_namespace` stores a `root_cset` pointer to the cgroup that should appear as root. When the kernel renders `/proc/self/cgroup`, it strips the `root_cset` path prefix from the process's actual cgroup path. If the process is at the root cset, it sees "/". This also prevents container processes from navigating above their cgroup via `/sys/fs/cgroup` (they can only see their subtree).

**Q2: Explain time namespace — what clocks are virtualized and why not CLOCK_REALTIME?**
**A:** Time namespace (kernel 5.6) virtualizes CLOCK_MONOTONIC and CLOCK_BOOTTIME by applying per-namespace offsets stored in `/proc/PID/timens_offsets`. Use case: container migration (checkpoint/restore with CRIU). When a container is migrated from host A (uptime 5 days) to host B (uptime 100 days), CLOCK_MONOTONIC and CLOCK_BOOTTIME would jump — breaking applications that measure elapsed time or container uptime. Time NS applies a negative offset so the container sees consistent monotonic time. CLOCK_REALTIME is NOT virtualized because: (1) It would break TLS/SSL certificate validation (certificates have real-time expiry). (2) It would break distributed systems that rely on wall-clock ordering. (3) It would cause log timestamp confusion. (4) Applications already handle CLOCK_REALTIME changes (NTP adjustments). Performance: the VDSO vvar page is per-namespace, so `clock_gettime()` still avoids syscalls — no performance penalty.

---

## Summary

- UTS NS: isolates hostname and NIS domain (only nodename/domainname, not uname release)
- IPC NS: isolates SysV IPC (shm, sem, msg) and POSIX mqueues — prevents cross-container data leaks
- Cgroup NS: virtualizes /proc/self/cgroup — container sees "/" instead of full host path
- Time NS: offsets CLOCK_MONOTONIC and CLOCK_BOOTTIME for container migration (CRIU)
- CLOCK_REALTIME NOT virtualized — would break TLS certs and distributed systems
- Time NS uses VDSO vvar page patching — zero performance overhead
- All 8 namespaces: mnt, uts, ipc, pid, net, user, cgroup, time

---

[Previous: Network and User Namespaces ←](Chapter_08_Net_User_NS.md) | [Next: Namespace Lifecycle Management →](Chapter_10_NS_Lifecycle.md)
