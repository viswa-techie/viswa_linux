# Chapter 6: Namespace Architecture and Kernel Implementation

## Learning Goals
- Understand the internal kernel data structures for namespaces
- Learn how namespaces attach to tasks via nsproxy
- Master clone(), unshare(), setns() syscall internals
- Know procfs namespace interfaces

---

## 1. Namespace Kernel Data Structures

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  struct task_struct (per-process)                        │
  │  ┌────────────────────────────────────┐                  │
  │  │ pid_t pid;                        │                  │
  │  │ struct nsproxy *nsproxy; ─────────┼──┐               │
  │  │ struct cred *cred;                │  │               │
  │  │ ...                               │  │               │
  │  └────────────────────────────────────┘  │               │
  │                                          ▼               │
  │  struct nsproxy (shared, refcounted)                     │
  │  ┌────────────────────────────────────────────┐          │
  │  │ atomic_t count;                            │          │
  │  │ struct uts_namespace   *uts_ns;  ──► hostname        │
  │  │ struct ipc_namespace   *ipc_ns;  ──► SysV IPC       │
  │  │ struct mnt_namespace   *mnt_ns;  ──► mount tree     │
  │  │ struct pid_namespace   *pid_ns_for_children; ──► PID│
  │  │ struct net             *net_ns;  ──► network stack  │
  │  │ struct time_namespace  *time_ns; ──► clock offsets  │
  │  │ struct cgroup_namespace *cgroup_ns; ──► cgroup view │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  struct cred (per-process credentials)                   │
  │  ┌────────────────────────────────────────────┐          │
  │  │ struct user_namespace *user_ns; ──► UID mappings     │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  Note: user_namespace is in cred, not nsproxy,           │
  │  because credentials are shared differently than          │
  │  namespaces (COW semantics, security model)              │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. The 8 Linux Namespaces

```
  ┌──────────┬──────────┬──────────────────────────────────────┐
  │ Namespace│ Flag     │ What it isolates                     │
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ Mount    │CLONE_    │ Mount points — each NS has own mount │
  │          │NEWNS     │ tree. First namespace (2002, v2.4.19)│
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ UTS      │CLONE_    │ Hostname and NIS domain name         │
  │          │NEWUTS    │ sethostname() only affects own NS    │
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ IPC      │CLONE_    │ SysV IPC objects (semaphores, shared │
  │          │NEWIPC    │ memory, message queues), POSIX mqueues│
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ PID      │CLONE_    │ Process IDs — PID 1 inside container │
  │          │NEWPID    │ is different PID on host. Hierarchical│
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ Network  │CLONE_    │ Network stack: interfaces, routes,   │
  │          │NEWNET    │ iptables, sockets, /proc/net          │
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ User     │CLONE_    │ UID/GID mappings. Root inside NS can │
  │          │NEWUSER   │ be unprivileged on host. Owns other  │
  │          │          │ namespaces (capability-granting)      │
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ Cgroup   │CLONE_    │ View of cgroup hierarchy. Container  │
  │          │NEWCGROUP │ sees its cgroup as root. (v4.6, 2016)│
  ├──────────┼──────────┼──────────────────────────────────────┤
  │ Time     │CLONE_    │ CLOCK_MONOTONIC, CLOCK_BOOTTIME      │
  │          │NEWTIME   │ offsets. Container boot time differs  │
  │          │          │ from host. (v5.6, 2020)              │
  └──────────┴──────────┴──────────────────────────────────────┘
```

---

## 3. Syscall Interface: clone, unshare, setns

```c
/* Three ways to work with namespaces */

/* === 1. clone() / clone3() === */
/* Create a NEW process in NEW namespaces */
/* Parent stays in original namespaces */
struct clone_args args = {
    .flags = CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET |
             CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER,
    .exit_signal = SIGCHLD,
};
pid_t pid = syscall(__NR_clone3, &args, sizeof(args));
/* pid is in new namespaces, parent in old */

/* === 2. unshare() === */
/* Current process leaves shared namespaces and creates new ones */
/* No new process created */
unshare(CLONE_NEWNS | CLONE_NEWPID);
/* This process now has new mount and PID namespaces */
/* Note: PID NS applies to CHILDREN, not current process */

/* === 3. setns() === */
/* Join an EXISTING namespace (from another process) */
int fd = open("/proc/1234/ns/net", O_RDONLY);
setns(fd, CLONE_NEWNET);  /* join PID 1234's network namespace */
close(fd);
/* Now this process shares network namespace with PID 1234 */
```

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  /proc/<pid>/ns/ directory:                              │
  │  ┌────────────────────────────────┐                      │
  │  │ lrwxrwxrwx cgroup -> cgroup:[4026531835]             │
  │  │ lrwxrwxrwx ipc    -> ipc:[4026531839]               │
  │  │ lrwxrwxrwx mnt    -> mnt:[4026531840]               │
  │  │ lrwxrwxrwx net    -> net:[4026531992]               │
  │  │ lrwxrwxrwx pid    -> pid:[4026531836]               │
  │  │ lrwxrwxrwx pid_for_children -> pid:[4026531836]     │
  │  │ lrwxrwxrwx time   -> time:[4026531834]              │
  │  │ lrwxrwxrwx time_for_children -> time:[4026531834]   │
  │  │ lrwxrwxrwx user   -> user:[4026531837]              │
  │  │ lrwxrwxrwx uts    -> uts:[4026531838]               │
  │  └────────────────────────────────┘                      │
  │                                                           │
  │  Inode number = namespace identifier                     │
  │  Same inode → same namespace                             │
  │  Namespace stays alive while:                            │
  │    - Any process is in it, OR                            │
  │    - /proc/<pid>/ns/<type> fd is open, OR               │
  │    - Bind-mounted somewhere                              │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Kernel Implementation of clone + namespaces

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  clone3() syscall path:                                  │
  │                                                           │
  │  sys_clone3()                                            │
  │    └─► kernel_clone()                                    │
  │         └─► copy_process()                               │
  │              ├─► copy_creds()                            │
  │              │    └─► if CLONE_NEWUSER:                  │
  │              │         create_user_ns()                  │
  │              │         (new user namespace in cred)       │
  │              │                                           │
  │              └─► copy_namespaces()                       │
  │                   └─► create_new_namespaces()            │
  │                        ├─► copy_mnt_ns()                │
  │                        │    (clone mount tree)           │
  │                        ├─► copy_utsname()               │
  │                        │    (copy hostname struct)       │
  │                        ├─► copy_ipcs()                  │
  │                        │    (new IPC namespace)          │
  │                        ├─► copy_pid_ns()                │
  │                        │    (new PID namespace)          │
  │                        ├─► copy_net_ns()                │
  │                        │    (new network namespace)      │
  │                        ├─► copy_cgroup_ns()             │
  │                        │    (new cgroup namespace)       │
  │                        └─► copy_time_ns()               │
  │                             (new time namespace)         │
  │                                                           │
  │  Each copy_*_ns() function:                              │
  │  1. Allocates new namespace struct                       │
  │  2. Initializes with copy of parent's data               │
  │  3. Assigns to child's nsproxy                           │
  │  4. Sets refcount to 1                                   │
  │  5. Registers with ns_common for procfs                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Namespace Lifetime Management

```c
/*
 * The ns_common structure embedded in every namespace struct
 * provides a unified interface for procfs and lifecycle mgmt
 */
struct ns_common {
    atomic_long_t stashed;     /* for bind-mount persistence */
    const struct proc_ns_operations *ops;
    unsigned int inum;         /* inode number (unique ID) */
    refcount_t count;          /* reference count */
};

/*
 * Each namespace type implements these operations:
 */
struct proc_ns_operations {
    const char *name;          /* "pid", "net", "mnt", etc. */
    const char *real_ns_name;  /* for display */
    int type;                  /* CLONE_NEW* flag */
    struct ns_common *(*get)(struct task_struct *);
    void (*put)(struct ns_common *);
    int (*install)(struct nsset *, struct ns_common *);
    struct user_namespace *(*owner)(struct ns_common *);
    struct ns_common *(*get_parent)(struct ns_common *);
};

/* Namespace lifecycle:
 *
 * Creation:  clone(CLONE_NEW*) or unshare(CLONE_NEW*)
 *            → alloc + init namespace struct
 *            → refcount = 1
 *
 * Reference: open(/proc/PID/ns/TYPE) → refcount++
 *            bind mount /proc/PID/ns/TYPE → refcount++
 *            fork() sharing nsproxy → refcount++
 *
 * Release:   close(fd) → refcount--
 *            process exit → refcount--
 *            umount bind mount → refcount--
 *            refcount == 0 → free namespace
 *
 * Persistence: bind-mount /proc/PID/ns/net /var/run/netns/myns
 *              → namespace survives even if all processes exit
 *              → ip netns uses this mechanism
 */
```

---

## Interview Questions

**Q1: Explain the relationship between nsproxy and a process's namespaces.**
**A:** Every `task_struct` has a pointer to a `struct nsproxy`, which holds pointers to 7 namespace types: UTS, IPC, mount, PID, net, cgroup, and time. The nsproxy is reference-counted and can be shared between processes that are in the same set of namespaces. When `clone()` is called without any `CLONE_NEW*` flags, the child shares the parent's nsproxy (refcount incremented). When `CLONE_NEW*` flags are used, `copy_namespaces()` creates a new nsproxy, creates new namespace objects for each flag, and copies the rest from the parent. The 8th namespace (user) is stored separately in `struct cred` (the process credentials), because credential changes have different sharing semantics (copy-on-write rather than reference counting) and because user namespaces own/govern other namespaces' capability checks.

**Q2: What are the three syscall interfaces for namespace manipulation and when do you use each?**
**A:** (1) `clone()` / `clone3()` — creates a new child process in new namespaces. The parent stays in its original namespaces. Used by container runtimes (runc calls clone3 to create the init process). (2) `unshare()` — the calling process leaves its current namespaces and creates new ones. No new process is created. Used by tools like `unshare(1)` command. Special case: `unshare(CLONE_NEWPID)` only affects children (the calling process keeps its PID). (3) `setns()` — join an existing namespace identified by a file descriptor (from `/proc/PID/ns/TYPE`). Used by `nsenter(1)`, `docker exec`, and `ip netns exec`. Example use case: a monitoring agent opens `/proc/PID/ns/net` to temporarily enter a container's network namespace, inspect it, then return.

---

## Summary

- Each process has nsproxy → pointers to 7 namespace types (user NS is in cred separately)
- 8 namespace types: mnt, uts, ipc, pid, net, user, cgroup, time
- clone(CLONE_NEW*) → new child in new NS; unshare() → current process gets new NS
- setns(fd) → join existing NS via /proc/PID/ns/TYPE fd
- Namespace lifetime: refcounted; alive while process/fd/bind-mount exists
- /proc/PID/ns/ symlinks: inode number = namespace identifier

---

[Previous: Container Security ←](Chapter_05_Container_Security.md) | [Next: PID and Mount Namespaces →](Chapter_07_PID_Mount_NS.md)
