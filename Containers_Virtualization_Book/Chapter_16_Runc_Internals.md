# Chapter 16: runc Internals

## Learning Goals
- Understand runc architecture and code flow
- Learn the parent-child process dance during container creation
- Master the init process setup sequence
- Know alternative runtimes: crun, youki

---

## 1. runc Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  runc: reference implementation of OCI Runtime Spec      │
  │  Written in Go, ~30K lines of code                      │
  │  Used by: Docker (via containerd), Kubernetes            │
  │                                                           │
  │  runc create <container-id>:                             │
  │                                                           │
  │  runc (parent)                                           │
  │    │                                                     │
  │    ├── Parse config.json                                 │
  │    ├── Set up container state directory                  │
  │    │                                                     │
  │    ├── clone() → runc init [stage 1]                    │
  │    │               │                                     │
  │    │               ├── Set up user namespace             │
  │    │               ├── clone() → runc init [stage 2]    │
  │    │               │               │                     │
  │    │               │               ├── Enter namespaces  │
  │    │               │               ├── Mount rootfs      │
  │    │               │               ├── pivot_root        │
  │    │               │               ├── Mount /proc, /sys │
  │    │               │               ├── Set hostname      │
  │    │               │               ├── Set up network    │
  │    │               │               ├── Apply rlimits     │
  │    │               │               ├── Set up seccomp    │
  │    │               │               ├── Drop capabilities │
  │    │               │               ├── Close all fds     │
  │    │               │               └── STOP (wait for    │
  │    │               │                   "start" command)  │
  │    │               └── exit                              │
  │    │                                                     │
  │    ├── Wait for stage 2 ready                           │
  │    ├── Write container state                             │
  │    ├── Run createRuntime hooks                          │
  │    └── Return (container is "created")                   │
  │                                                           │
  │  runc start <container-id>:                              │
  │    │                                                     │
  │    ├── Signal stage 2 to continue                       │
  │    └── Stage 2:                                          │
  │        ├── Install seccomp filter (if not yet)          │
  │        └── exec() container entrypoint                  │
  │           (becomes PID 1 in container)                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. The Two-Stage Init Process (nsexec)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  runc uses a C helper called "nsexec" that runs BEFORE  │
  │  Go runtime starts in the child (via cgo constructor)    │
  │                                                           │
  │  Why C? Because Go's runtime creates threads, and       │
  │  clone(CLONE_NEWPID) only puts the calling thread in    │
  │  the new PID namespace. Go threads would be in wrong NS.│
  │                                                           │
  │  nsexec flow (C code, runs as init constructor):         │
  │  ┌─────────────────────────────────────────────┐        │
  │  │ Parent process (runc):                      │        │
  │  │   writes config to pipe → child             │        │
  │  │                                             │        │
  │  │ Stage 1 (child):                            │        │
  │  │   read config from pipe                     │        │
  │  │   unshare(CLONE_NEWUSER) if needed          │        │
  │  │   clone(CLONE_NEWPID) → stage 2            │        │
  │  │   wait for stage 2                          │        │
  │  │   exit                                      │        │
  │  │                                             │        │
  │  │ Stage 2 (grandchild):                       │        │
  │  │   setns() for each namespace                │        │
  │  │   unshare(CLONE_NEWNS) for mount namespace  │        │
  │  │   signal parent "ready"                     │        │
  │  │   wait for "start" signal                   │        │
  │  │   continue to Go runtime                    │        │
  │  │   Go code: set up rootfs, seccomp, exec()  │        │
  │  │                                             │        │
  │  │ Communication: socketpair + pipe (sync)     │        │
  │  └─────────────────────────────────────────────┘        │
  │                                                           │
  │  The nsexec C code is in:                                │
  │  runc/libcontainer/nsenter/nsexec.c                      │
  │  It uses __attribute__((constructor)) to run before main │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Container State Management

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  runc stores container state in:                         │
  │  /run/runc/<container-id>/state.json                     │
  │                                                           │
  │  {                                                        │
  │    "id": "my-container-123",                             │
  │    "init_process_pid": 4567,                             │
  │    "init_process_start": 1234567890,                     │
  │    "created": "2024-01-15T10:30:00Z",                   │
  │    "config": { /* full OCI config */ },                  │
  │    "rootless": false,                                     │
  │    "cgroup_paths": {                                      │
  │      "": "/sys/fs/cgroup/system.slice/runc-my-ct.scope" │
  │    },                                                     │
  │    "namespace_paths": {                                   │
  │      "NEWPID": "/proc/4567/ns/pid",                     │
  │      "NEWNS": "/proc/4567/ns/mnt",                      │
  │      "NEWNET": "/proc/4567/ns/net"                      │
  │    },                                                     │
  │    "external_descriptors": [],                            │
  │    "intel_rdt_path": ""                                  │
  │  }                                                        │
  │                                                           │
  │  File locking:                                           │
  │  - /run/runc/<container-id> directory is locked          │
  │  - Prevents concurrent runc operations on same container│
  │  - Uses flock() on the directory fd                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Alternative OCI Runtimes

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌────────┬────────────┬──────────────────────────────┐ │
  │  │Runtime │Language    │Characteristics               │ │
  │  ├────────┼────────────┼──────────────────────────────┤ │
  │  │runc    │Go          │Reference implementation      │ │
  │  │        │            │~30K lines, most widely used  │ │
  │  │        │            │Used by Docker, containerd    │ │
  │  ├────────┼────────────┼──────────────────────────────┤ │
  │  │crun    │C           │~7K lines, much faster        │ │
  │  │        │            │Lower memory (~50% less)      │ │
  │  │        │            │Default in Podman/Fedora      │ │
  │  │        │            │Supports WASM containers      │ │
  │  ├────────┼────────────┼──────────────────────────────┤ │
  │  │youki   │Rust        │Memory-safe, modern           │ │
  │  │        │            │Goal: replace runc            │ │
  │  │        │            │Full OCI compliance           │ │
  │  ├────────┼────────────┼──────────────────────────────┤ │
  │  │kata    │Go + microVM│VM-based isolation            │ │
  │  │runtime │            │Each container in a microVM   │ │
  │  │        │            │OCI-compatible interface      │ │
  │  ├────────┼────────────┼──────────────────────────────┤ │
  │  │gVisor  │Go          │userspace kernel (sentry)     │ │
  │  │(runsc) │            │Intercepts syscalls           │ │
  │  │        │            │Strong isolation, some compat │ │
  │  └────────┴────────────┴──────────────────────────────┘ │
  │                                                           │
  │  Performance comparison (container creation):            │
  │  ┌────────┬──────────────────────────────┐              │
  │  │crun    │ ~50ms  (fastest, C)          │              │
  │  │runc    │ ~100ms (baseline, Go)        │              │
  │  │youki   │ ~80ms  (Rust)                │              │
  │  │kata    │ ~500ms (microVM overhead)    │              │
  │  │runsc   │ ~200ms (userspace kernel)    │              │
  │  └────────┴──────────────────────────────┘              │
  │                                                           │
  │  All runtimes are plug-replaceable — same OCI interface │
  │  containerd config.toml:                                 │
  │  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes│
  │   .runc]                                                 │
  │    runtime_type = "io.containerd.runc.v2"               │
  │    [plugins...runc.options]                              │
  │      BinaryName = "/usr/bin/crun"  ← swap runtime      │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Hooks in OCI Lifecycle

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  OCI Hooks: external programs executed at specific       │
  │  points in the container lifecycle                       │
  │                                                           │
  │  ┌──────────────────┬───────────────────────────────┐   │
  │  │ Hook             │ When it runs                  │   │
  │  ├──────────────────┼───────────────────────────────┤   │
  │  │ createRuntime    │ After create, before start    │   │
  │  │                  │ In runtime namespace          │   │
  │  │                  │ Use: set up network, devices  │   │
  │  ├──────────────────┼───────────────────────────────┤   │
  │  │ createContainer  │ After create namespace setup  │   │
  │  │                  │ In container namespace        │   │
  │  │                  │ Use: final container setup    │   │
  │  ├──────────────────┼───────────────────────────────┤   │
  │  │ startContainer   │ Before exec of user process   │   │
  │  │                  │ In container namespace        │   │
  │  │                  │ Use: pre-exec initialization  │   │
  │  ├──────────────────┼───────────────────────────────┤   │
  │  │ poststart        │ After user process started    │   │
  │  │                  │ In runtime namespace          │   │
  │  │                  │ Use: logging, notification    │   │
  │  ├──────────────────┼───────────────────────────────┤   │
  │  │ poststop         │ After container stopped       │   │
  │  │                  │ In runtime namespace          │   │
  │  │                  │ Use: cleanup, deregister      │   │
  │  └──────────────────┴───────────────────────────────┘   │
  │                                                           │
  │  CNI (Container Network Interface) uses createRuntime   │
  │  hook to set up container networking                    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why does runc use a two-stage init process with C code (nsexec)?**
**A:** runc is written in Go, but Go's runtime creates multiple OS threads during initialization (goroutine scheduler, GC, etc.). This conflicts with `clone(CLONE_NEWPID)`, which only puts the calling thread into the new PID namespace — other Go runtime threads remain in the original PID namespace, causing inconsistency. The solution: `nsexec.c` is a C function marked with `__attribute__((constructor))`, so it runs before Go's runtime starts (before `main()`), in a single-threaded process. Stage 1: `nsexec` reads configuration from a pipe, calls `unshare(CLONE_NEWUSER)` if needed, then `clone()` with all namespace flags to create Stage 2. Stage 2: the grandchild process is in all new namespaces. It calls `setns()` for any pre-existing namespaces to join, signals the parent it's ready, and waits. When `runc start` is called, Stage 2 continues to the Go runtime which sets up the rootfs, mounts, seccomp, capabilities, and finally `exec()`s the container entrypoint. The two stages are needed because: (a) user namespace setup requires a helper process to write uid_map/gid_map from outside, (b) PID namespace requires `clone()` (not just `unshare()`), and (c) all this must happen before Go threads are created.

**Q2: Compare runc, crun, and kata-runtime. When would you choose each?**
**A:** **runc** (Go, ~30K lines): the reference OCI runtime, most widely used and tested. Default for Docker and containerd. Best choice for general-purpose containers where compatibility is paramount. ~100ms container creation. **crun** (C, ~7K lines): fastest OCI runtime. ~50ms creation, ~50% less memory than runc (no Go runtime overhead). Default on Fedora/Podman. Best for: high-density environments (many short-lived containers), serverless functions, CI/CD pipelines. Also supports experimental features like WASM containers. **kata-runtime** (Go + microVM): runs each container inside a lightweight VM (using QEMU or Cloud Hypervisor/Firecracker). OCI-compatible — transparent to container orchestrators. ~500ms creation, higher memory overhead. Best for: multi-tenant environments with untrusted workloads (strong isolation), compliance requirements (hardware-level separation), running code from untrusted sources. Cannot be used where VM overhead is unacceptable or where hardware virtualization is unavailable (nested VMs, some cloud instances). All three implement the same OCI interface and can be swapped in containerd's config.toml without changing container images or orchestration.

---

## Summary

- runc: reference OCI runtime, Go, ~30K lines, used by Docker/containerd
- Two-stage init: nsexec.c (C constructor) runs before Go to avoid thread/namespace conflicts
- Stage 1: unshare user NS → clone → Stage 2: enter all NS → wait for start signal
- Container state: /run/runc/<id>/state.json with PID, cgroup paths, namespace paths
- Alternatives: crun (C, fastest), youki (Rust), kata (microVM), gVisor (userspace kernel)
- All OCI runtimes are interchangeable — same config.json, same lifecycle API
- OCI hooks: createRuntime, createContainer, startContainer, poststart, poststop

---

[Previous: OCI Specification ←](Chapter_15_OCI_Specification.md) | [Next: containerd and CRI-O →](Chapter_17_Containerd_CRI.md)
