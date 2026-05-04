# Chapter 16: Namespaces and Security Isolation

## Learning Goals
- Understand all 8 Linux namespace types and their security roles
- Know how namespaces isolate processes from each other
- Understand user namespaces and privilege isolation
- Know how containers combine namespaces for security

---

## 16.1 Namespace Overview

```
Namespaces: Partition kernel resources so processes see different views.
Foundation of container security — each container gets its own namespaces.

8 Namespace Types:
  ┌──────────────┬─────────────────────────────────────────┐
  │ Namespace    │ Isolates                                 │
  ├──────────────┼─────────────────────────────────────────┤
  │ Mount (mnt)  │ Filesystem mount points                  │
  │ PID          │ Process IDs                              │
  │ Network (net)│ Network stack (interfaces, routes, etc.) │
  │ UTS          │ Hostname and domain name                 │
  │ IPC          │ System V IPC, POSIX message queues       │
  │ User         │ UIDs/GIDs (privilege isolation)          │
  │ Cgroup       │ Cgroup root directory view               │
  │ Time         │ System clock offsets (5.6+)              │
  └──────────────┴─────────────────────────────────────────┘

APIs:
  clone(flags)    Create child in new namespace(s)
  unshare(flags)  Move current process to new namespace(s)
  setns(fd, type) Join an existing namespace

  Flags: CLONE_NEWNS, CLONE_NEWPID, CLONE_NEWNET,
         CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWUSER,
         CLONE_NEWCGROUP, CLONE_NEWTIME

Process namespace membership:
  /proc/<pid>/ns/ — symlinks to namespace inodes
    ls -la /proc/self/ns/
    lrwxrwxrwx  user -> user:[4026531837]
    lrwxrwxrwx  mnt  -> mnt:[4026531840]
    lrwxrwxrwx  pid  -> pid:[4026531836]
    lrwxrwxrwx  net  -> net:[4026531992]
    ...
```

---

## 16.2 Mount Namespace

```
Mount namespace: Each process sees its own mount tree.

  ┌─────────────────┐    ┌─────────────────┐
  │ Namespace A      │    │ Namespace B      │
  │ /                │    │ /                │
  │ ├── /proc        │    │ ├── /proc        │
  │ ├── /sys         │    │ ├── /sys         │
  │ ├── /home (host) │    │ ├── /app         │
  │ └── /var         │    │ └── /data        │
  └─────────────────┘    └─────────────────┘

Security features:
  - Container sees only its own filesystem
  - Cannot access host files not explicitly mounted
  - pivot_root: change root filesystem entirely

  Mount propagation:
    MS_SHARED:   Mounts propagate between namespaces
    MS_PRIVATE:  Mounts do NOT propagate (default for containers)
    MS_SLAVE:    One-way propagation (host→container)
    MS_UNBINDABLE: Cannot be bind-mounted

  Bind mounts for selective sharing:
    mount --bind /host/dir /container/dir
    mount --bind -o ro /host/config /container/config  # read-only
```

---

## 16.3 PID Namespace

```
PID namespace: Process sees different PIDs inside vs outside.

  Host PID namespace:        Container PID namespace:
  PID 1 (init/systemd)       PID 1 (container init)  ← PID 1 inside
  PID 100 (sshd)             PID 2 (app)             ← PID 842 outside
  PID 841 (container runtime)
  PID 842 (container init)   ← This is PID 1 inside container

  ┌──────────────────────────────────────────────┐
  │  Host PID Namespace                           │
  │  PID 1 ─── PID 841                           │
  │               │                               │
  │  ┌────────────▼───────────────────────┐       │
  │  │  Container PID Namespace            │       │
  │  │  PID 1 (= host PID 842)            │       │
  │  │  PID 2 (= host PID 843)            │       │
  │  │  Cannot see host PIDs               │       │
  │  │  Can only signal PIDs in this NS    │       │
  │  └────────────────────────────────────┘       │
  └──────────────────────────────────────────────┘

Security:
  - Container process cannot see or signal host processes
  - PID 1 in container receives orphaned children
  - kill(-1, sig) only kills processes in same PID namespace
  - /proc shows only processes in same PID namespace
```

---

## 16.4 Network Namespace

```
Network namespace: Separate network stack per namespace.

  Each network namespace has its own:
    - Network interfaces (lo, eth0, etc.)
    - IP addresses
    - Routing tables
    - Firewall rules (iptables/nftables)
    - /proc/net
    - Ports (each NS has its own port 80)

  Container networking:
  ┌──────────────────────────┐
  │ Host Network Namespace    │
  │ eth0: 192.168.1.10        │
  │ docker0: 172.17.0.1       │
  │ vethXXXX ────────────┐    │
  │                       │    │
  │  ┌────────────────────▼─┐  │
  │  │ Container Net NS      │  │
  │  │ eth0: 172.17.0.2      │  │
  │  │ (veth pair endpoint)  │  │
  │  │ lo: 127.0.0.1         │  │
  │  │ Own routing table     │  │
  │  │ Own iptables rules    │  │
  │  └──────────────────────┘  │
  └──────────────────────────┘

  veth pair: virtual ethernet cable between namespaces
  Bridge (docker0): connects multiple container veth pairs

  Security:
    - Container cannot sniff host network traffic
    - Container has its own firewall rules
    - Port conflicts impossible (each NS has own ports)
    - Network policies control cross-container traffic
```

---

## 16.5 User Namespace (Security Critical)

```
User namespace: Remap UIDs/GIDs. Root inside, unprivileged outside.

  THE most important namespace for security.

  UID mapping:
    Inside container:  UID 0 (root)     → Outside: UID 100000
    Inside container:  UID 1 (daemon)   → Outside: UID 100001
    Inside container:  UID 1000 (user)  → Outside: UID 101000

  Config files:
    /proc/<pid>/uid_map:   0 100000 65536
    /proc/<pid>/gid_map:   0 100000 65536
    Format: <inside_start> <outside_start> <count>

  Security model:
  ┌──────────────────────────────────────────────┐
  │  Host: UID 100000 (unprivileged)              │
  │                                               │
  │  ┌────────────────────────────────────┐       │
  │  │  User Namespace                     │       │
  │  │  UID 0 (root inside = no host privs)│       │
  │  │                                     │       │
  │  │  Capabilities: FULL SET inside NS   │       │
  │  │  But only applies to resources      │       │
  │  │  OWNED by this namespace.           │       │
  │  │                                     │       │
  │  │  Can create other namespaces ✓      │       │
  │  │  Can mount proc/sysfs ✓             │       │
  │  │  Can modify host files ✗            │       │
  │  │  Can load kernel modules ✗          │       │
  │  │  Can access host devices ✗          │       │
  │  └────────────────────────────────────┘       │
  └──────────────────────────────────────────────┘

  User namespace enables rootless containers:
    - Process is UID 0 inside container (full capabilities)
    - Process is UID 100000+ outside (no special privileges)
    - Container escape → unprivileged user on host
    - Huge security improvement over root containers

  Capability scoping:
    CAP_NET_ADMIN inside user NS → only affects that NS's network
    CAP_SYS_ADMIN inside user NS → limited to NS resources
    Capabilities do NOT grant host-level privileges
```

---

## 16.6 Container Namespace Combination

```
A typical container uses ALL namespaces together:

  docker run --name myapp myimage

  Creates:
    ┌─────────────────────────────────────────────┐
    │  Container                                   │
    │                                              │
    │  Mount NS:  Own root filesystem              │
    │  PID NS:    PID 1 = container entrypoint     │
    │  Net NS:    Own IP, ports, routing           │
    │  UTS NS:    Own hostname                     │
    │  IPC NS:    Own shared memory, semaphores    │
    │  User NS:   UID 0 inside = unprivileged out  │
    │  Cgroup NS: Own cgroup view                  │
    │                                              │
    │  + Seccomp filter (blocks dangerous syscalls) │
    │  + AppArmor/SELinux profile (MAC)            │
    │  + Cgroup limits (CPU, memory, I/O)          │
    │  + Dropped capabilities                      │
    │  + Read-only root filesystem                 │
    │  + No new privileges (PR_SET_NO_NEW_PRIVS)   │
    └─────────────────────────────────────────────┘

  Defense in depth: namespaces + seccomp + MAC + cgroups + capabilities
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/nsproxy.c | Namespace proxy management |
| kernel/user_namespace.c | User namespace implementation |
| fs/namespace.c | Mount namespace |
| kernel/pid_namespace.c | PID namespace |
| net/core/net_namespace.c | Network namespace |
| kernel/utsname.c | UTS namespace |
| ipc/namespace.c | IPC namespace |
| include/linux/nsproxy.h | nsproxy structure |

---

## Interview Questions

**Q1: How do namespaces provide security isolation?**
A: Namespaces partition kernel resources so each process group sees its own isolated view. Mount NS hides host files. PID NS prevents seeing/signaling host processes. Network NS isolates network stacks. User NS remaps UIDs so container root is unprivileged on host. IPC NS isolates shared memory. Together, they create the illusion of a separate machine. A process in a container cannot enumerate host processes (PID NS), access host files (Mount NS), sniff host network (Net NS), or escalate to host root (User NS). Containers combine all namespace types with Seccomp and MAC for defense in depth.

**Q2: Why is the user namespace the most important for container security?**
A: The user namespace remaps UIDs — root (UID 0) inside the container maps to an unprivileged UID (e.g., 100000) on the host. This means: (1) Process has full capabilities inside the container for normal operation. (2) If an attacker escapes the container, they land as an unprivileged user on the host. (3) Capabilities are scoped — CAP_NET_ADMIN only affects the container's network namespace, not the host. (4) Enables rootless containers — the entire container runtime runs without host root. Without user namespaces, container root equals host root, and container escape means full host compromise.

**Q3: What is the security difference between MS_SHARED and MS_PRIVATE mount propagation?**
A: MS_SHARED propagates mount/unmount events between namespaces — if a USB drive is mounted in the host, containers with shared propagation also see it. This is convenient but a security risk — it expands the container's attack surface to include new host mounts. MS_PRIVATE (default for containers) prevents propagation — mount events in one namespace are invisible to others. Containers see only explicitly configured mounts. MS_SLAVE allows one-way propagation (host→container). For security, containers should use MS_PRIVATE to minimize exposure to host filesystem changes.

---

## Summary

- 8 namespace types isolate: mounts, PIDs, network, hostname, IPC, users, cgroups, time
- APIs: clone(CLONE_NEW*), unshare(), setns()
- User namespace: UID remapping — root inside = unprivileged outside
- PID namespace: container has its own PID 1
- Network namespace: separate interfaces, IPs, ports, firewall rules
- Mount namespace: separate filesystem view with propagation control
- Containers combine ALL namespaces + seccomp + MAC + cgroups
- User namespace enables rootless containers (major security gain)

---

Next: [Chapter 17 — Control Groups (cgroups)](Chapter_17_Cgroups.md)
