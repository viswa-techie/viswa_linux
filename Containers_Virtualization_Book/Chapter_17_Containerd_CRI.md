# Chapter 17: containerd and CRI-O

## Learning Goals
- Understand containerd architecture and shim model
- Learn CRI (Container Runtime Interface) for Kubernetes
- Master CRI-O as a minimal CRI implementation
- Know the full stack: kubelet → CRI → containerd → runc

---

## 1. Container Runtime Architecture Stack

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Evolution of the container runtime stack:               │
  │                                                           │
  │  2013-2016 (monolithic Docker):                          │
  │  ┌─────────────────────────────┐                         │
  │  │ Docker daemon (dockerd)     │                         │
  │  │ - API server                │                         │
  │  │ - Image management          │                         │
  │  │ - Network management        │                         │
  │  │ - Volume management         │                         │
  │  │ - Container lifecycle       │ ← ALL in one process   │
  │  │ - libcontainer (→ runc)     │                         │
  │  └─────────────────────────────┘                         │
  │                                                           │
  │  2017+ (modular architecture):                           │
  │  ┌──────────┐                                            │
  │  │ kubelet  │ (or Docker CLI)                            │
  │  └────┬─────┘                                            │
  │       │ CRI gRPC API                                     │
  │       ▼                                                   │
  │  ┌──────────────────────┐                                │
  │  │ containerd           │ (or CRI-O)                     │
  │  │ - Image management   │                                │
  │  │ - Container lifecycle│                                │
  │  │ - Snapshot management│                                │
  │  │ - Content store (CAS)│                                │
  │  └────┬─────────────────┘                                │
  │       │ OCI Runtime Spec                                  │
  │       ▼                                                   │
  │  ┌──────────────────────┐                                │
  │  │ containerd-shim-v2   │ (per-container process)        │
  │  └────┬─────────────────┘                                │
  │       │ exec runc                                         │
  │       ▼                                                   │
  │  ┌──────────────────────┐                                │
  │  │ runc (or crun/kata)  │ → creates container           │
  │  │ - clone namespaces   │                                │
  │  │ - set up cgroups     │                                │
  │  │ - exec entrypoint    │                                │
  │  └──────────────────────┘                                │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. containerd Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  containerd: industry-standard container runtime         │
  │  CNCF graduated project, used by Docker + Kubernetes    │
  │                                                           │
  │  ┌──────────────────────────────────────────────┐       │
  │  │              containerd daemon                │       │
  │  │  ┌────────────────────────────────────────┐  │       │
  │  │  │ CRI Plugin (Kubernetes interface)      │  │       │
  │  │  │ gRPC server for kubelet                │  │       │
  │  │  └────────────────────────────────────────┘  │       │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │       │
  │  │  │ Content  │ │Snapshots │ │  Metadata    │ │       │
  │  │  │ Store    │ │(overlayfs│ │  (bolt DB)   │ │       │
  │  │  │ (blobs)  │ │ layers)  │ │              │ │       │
  │  │  └──────────┘ └──────────┘ └──────────────┘ │       │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │       │
  │  │  │ Images   │ │Container │ │  Tasks       │ │       │
  │  │  │ service  │ │ service  │ │  service     │ │       │
  │  │  └──────────┘ └──────────┘ └──────────────┘ │       │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │       │
  │  │  │ Events   │ │ Diff     │ │  Leases      │ │       │
  │  │  │ service  │ │ service  │ │  service     │ │       │
  │  │  └──────────┘ └──────────┘ └──────────────┘ │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Storage layout:                                         │
  │  /var/lib/containerd/                                    │
  │  ├── io.containerd.content.v1.content/                   │
  │  │   └── blobs/sha256/      ← image layers (CAS)       │
  │  ├── io.containerd.snapshotter.v1.overlayfs/             │
  │  │   └── snapshots/         ← mounted layer snapshots   │
  │  ├── io.containerd.metadata.v1.bolt/                     │
  │  │   └── meta.db            ← metadata (BoltDB)        │
  │  └── io.containerd.runtime.v2.task/                      │
  │      └── <ns>/<id>/         ← container state           │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. containerd Shim Model

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Why shims? Decouple container lifecycle from daemon     │
  │                                                           │
  │  Without shim:                                           │
  │  containerd restart → ALL containers die                 │
  │  (containers are children of containerd)                 │
  │                                                           │
  │  With shim:                                              │
  │  containerd restart → containers keep running!           │
  │  (shim is container's parent, independent of containerd) │
  │                                                           │
  │  ┌──────────────┐                                        │
  │  │ containerd   │                                        │
  │  │   daemon     │──── gRPC/ttrpc ────┐                  │
  │  └──────────────┘                     │                  │
  │        ▲                              ▼                  │
  │        │                   ┌──────────────────┐          │
  │   can restart              │ shim (per-ct)     │          │
  │   without killing          │ ┌──────────────┐ │          │
  │   containers               │ │ container    │ │          │
  │                            │ │ process      │ │          │
  │                            │ │ (PID 1)      │ │          │
  │                            │ └──────────────┘ │          │
  │                            │                   │          │
  │                            │ Responsibilities: │          │
  │                            │ - Reap zombies    │          │
  │                            │ - Forward signals │          │
  │                            │ - Report exit code│          │
  │                            │ - Serve stdio     │          │
  │                            │ - Report OOM events│         │
  │                            └──────────────────┘          │
  │                                                           │
  │  Shim binary:                                            │
  │  containerd-shim-runc-v2    ← for runc/crun             │
  │  containerd-shim-kata-v2    ← for kata containers       │
  │  containerd-shim-runsc-v2   ← for gVisor                │
  │                                                           │
  │  Shim starts runc, runc creates container and exits,    │
  │  shim remains as parent of container process             │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. CRI-O

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CRI-O: minimal CRI implementation (Kubernetes-only)    │
  │  CNCF incubating project, used by OpenShift             │
  │                                                           │
  │  ┌────────────────────────────────────────┐              │
  │  │           CRI-O                        │              │
  │  │  Purpose: CRI for Kubernetes ONLY     │              │
  │  │  No Docker CLI, no Docker API         │              │
  │  │  Minimal attack surface               │              │
  │  │                                        │              │
  │  │  Components:                           │              │
  │  │  - CRI gRPC server                    │              │
  │  │  - Image service (containers/image)   │              │
  │  │  - Storage (containers/storage)       │              │
  │  │  - conmon (container monitor process)  │              │
  │  │  - CNI for networking                 │              │
  │  │  - OCI runtime (runc/crun)            │              │
  │  └────────────────────────────────────────┘              │
  │                                                           │
  │  CRI-O vs containerd:                                    │
  │  ┌────────────┬────────────────┬───────────────────┐    │
  │  │            │ containerd     │ CRI-O             │    │
  │  ├────────────┼────────────────┼───────────────────┤    │
  │  │ Scope      │ General purpose│ Kubernetes only   │    │
  │  │ Docker CLI │ Yes (via shim)│ No                │    │
  │  │ API        │ gRPC (general)│ CRI gRPC only     │    │
  │  │ Codebase   │ ~100K lines   │ ~50K lines        │    │
  │  │ Monitor    │ shim          │ conmon             │    │
  │  │ Storage    │ snapshotter   │ containers/storage│    │
  │  │ Used by    │ Docker, K8s   │ OpenShift, K8s    │    │
  │  │ Default    │ Most K8s      │ OpenShift          │    │
  │  │ runtime    │ distros       │                    │    │
  │  └────────────┴────────────────┴───────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Full Kubernetes → Container Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  kubectl run nginx --image=nginx                         │
  │                                                           │
  │  1. kubectl → API server: create Pod                     │
  │  2. Scheduler → assign Pod to Node                       │
  │  3. kubelet → watches assigned Pods                      │
  │                                                           │
  │  4. kubelet → CRI RunPodSandbox()                        │
  │     ├── containerd creates pause container               │
  │     │   (network namespace holder)                       │
  │     └── CNI plugin configures networking                 │
  │                                                           │
  │  5. kubelet → CRI PullImage("nginx")                     │
  │     ├── containerd pulls from registry                   │
  │     ├── Verifies digests                                 │
  │     └── Unpacks layers to snapshotter                    │
  │                                                           │
  │  6. kubelet → CRI CreateContainer(podSandbox, config)    │
  │     ├── containerd creates OCI bundle (config.json)      │
  │     ├── Starts shim process                              │
  │     └── Shim calls runc create                           │
  │                                                           │
  │  7. kubelet → CRI StartContainer(containerID)            │
  │     ├── Shim calls runc start                            │
  │     └── nginx process starts as PID 1 in container      │
  │                                                           │
  │  8. kubelet → CRI ContainerStatus() (health checks)     │
  │                                                           │
  │  9. kubelet → CRI StopContainer() (on delete)           │
  │     ├── Shim sends SIGTERM to PID 1                     │
  │     ├── Grace period (30s default)                       │
  │     └── SIGKILL if still running                        │
  │                                                           │
  │  10. kubelet → CRI RemoveContainer() + RemovePodSandbox()│
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is the containerd shim and why is it necessary?**
**A:** The containerd shim is a small per-container process that acts as the parent of the container's init process. Without a shim, containers would be direct children of containerd — if containerd restarts (upgrade, crash), all containers would receive SIGKILL and die. With the shim model: (1) containerd starts a shim process for each container. (2) The shim forks runc, which creates the container environment and execs the entrypoint. (3) runc exits; the container process is reparented to the shim. (4) containerd can now restart freely — shim and container are independent processes. The shim also handles: (a) zombie reaping (wait() on container children), (b) signal forwarding (SIGTERM from containerd to container), (c) exit code reporting (via ttrpc/gRPC back to containerd), (d) stdout/stderr streaming (holds the stdio pipe fds), and (e) OOM event monitoring (from cgroup memory.events). containerd reconnects to shims after restart by reading their known socket paths. Shim variants exist per-runtime: `containerd-shim-runc-v2`, `containerd-shim-kata-v2`, etc.

**Q2: Trace the full path from `kubectl run` to a running container process.**
**A:** (1) kubectl sends a Pod spec to the API server. (2) The scheduler assigns the Pod to a node. (3) The kubelet on that node watches for new Pod assignments. (4) kubelet calls `RunPodSandbox()` via CRI gRPC — containerd creates a "pause" container (a minimal process that holds the network namespace; all containers in the pod share this namespace). containerd calls the CNI plugin to configure networking (create veth pair, assign IP, set routes). (5) kubelet calls `PullImage()` — containerd fetches the image manifest from the registry, downloads layers (content-addressable blobs), verifies SHA256 digests, unpacks layers into the snapshotter (overlayfs). (6) kubelet calls `CreateContainer()` — containerd generates an OCI config.json from the pod/container spec (image config + resource limits + security context), prepares the rootfs (overlayfs mount of layers + writable upper), starts a containerd-shim process, shim calls `runc create` (which sets up namespaces, cgroups, mounts via nsexec 2-stage init). Container is now in "created" state. (7) kubelet calls `StartContainer()` — shim calls `runc start`, runc signals the init process to continue, init execs the container entrypoint. Container is "running". (8) kubelet continuously calls `ContainerStatus()` for liveness/readiness probes. (9) On deletion: `StopContainer()` sends SIGTERM, waits grace period, then SIGKILL. (10) `RemoveContainer()` and `RemovePodSandbox()` clean up cgroups, state, snapshots.

---

## Summary

- containerd: CNCF graduated, general-purpose container runtime (images, snapshots, tasks)
- Shim model: per-container process; enables containerd restarts without killing containers
- Shim handles: zombie reaping, signal forwarding, exit codes, stdio, OOM monitoring
- CRI-O: minimal Kubernetes-only CRI implementation (used by OpenShift)
- CRI gRPC API: RunPodSandbox, PullImage, CreateContainer, StartContainer, StopContainer
- Pause container: holds pod's network namespace, all pod containers share it
- Full stack: kubelet → CRI → containerd → shim → runc → container process

---

[Previous: runc Internals ←](Chapter_16_Runc_Internals.md) | [Next: Container Networking Deep Dive →](Chapter_18_Container_Networking.md)
