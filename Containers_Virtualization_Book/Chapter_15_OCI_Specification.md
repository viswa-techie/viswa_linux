# Chapter 15: OCI Specification

## Learning Goals
- Understand the Open Container Initiative (OCI) standards
- Learn the OCI Runtime Specification (config.json)
- Master the OCI Image Specification (layers, manifests)
- Know the OCI Distribution Specification (registry API)

---

## 1. OCI Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  OCI = Open Container Initiative                         │
  │  Founded 2015 by Docker + CoreOS (now Linux Foundation) │
  │                                                           │
  │  Three specifications:                                   │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ 1. Runtime Spec  — how to RUN a container       │    │
  │  │    config.json + filesystem bundle               │    │
  │  │    Reference impl: runc                          │    │
  │  │                                                  │    │
  │  │ 2. Image Spec    — how to PACKAGE a container   │    │
  │  │    Layers, config, manifest                      │    │
  │  │    Used by: Docker images, Buildah, Podman      │    │
  │  │                                                  │    │
  │  │ 3. Distribution Spec — how to DISTRIBUTE images │    │
  │  │    Registry HTTP API (push/pull)                 │    │
  │  │    Used by: Docker Hub, GHCR, ECR, GCR          │    │
  │  └─────────────────────────────────────────────────┘    │
  │                                                           │
  │  Why OCI matters:                                        │
  │  - Before OCI: Docker-specific format, no standard      │
  │  - After OCI: any runtime can run any image              │
  │  - Kubernetes: uses CRI (Container Runtime Interface)    │
  │    which speaks OCI underneath                          │
  │                                                           │
  │  Ecosystem:                                              │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
  │  │ Build    │    │ Registry │    │ Runtime  │          │
  │  │ docker   │───►│ Docker   │───►│ runc     │          │
  │  │ buildah  │    │ Hub      │    │ crun     │          │
  │  │ kaniko   │    │ GHCR     │    │ youki    │          │
  │  │ buildkit │    │ Harbor   │    │ kata     │          │
  │  └──────────┘    └──────────┘    └──────────┘          │
  │   OCI Image       OCI Dist       OCI Runtime            │
  │   Spec            Spec           Spec                    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. OCI Runtime Spec — config.json

```json
{
    "ociVersion": "1.0.2",
    "process": {
        "terminal": false,
        "user": { "uid": 0, "gid": 0 },
        "args": ["/usr/sbin/nginx", "-g", "daemon off;"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin",
            "NGINX_VERSION=1.25.0"
        ],
        "cwd": "/",
        "capabilities": {
            "bounding": ["CAP_NET_BIND_SERVICE", "CAP_CHOWN"],
            "effective": ["CAP_NET_BIND_SERVICE", "CAP_CHOWN"],
            "permitted": ["CAP_NET_BIND_SERVICE", "CAP_CHOWN"],
            "ambient": ["CAP_NET_BIND_SERVICE"]
        },
        "rlimits": [
            { "type": "RLIMIT_NOFILE", "hard": 1024, "soft": 1024 }
        ],
        "noNewPrivileges": true
    },
    "root": {
        "path": "rootfs",
        "readonly": false
    },
    "hostname": "my-container",
    "mounts": [
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc",
            "options": ["nosuid", "noexec", "nodev"]
        },
        {
            "destination": "/sys",
            "type": "sysfs",
            "source": "sysfs",
            "options": ["nosuid", "noexec", "nodev", "ro"]
        },
        {
            "destination": "/dev",
            "type": "tmpfs",
            "source": "tmpfs",
            "options": ["nosuid", "strictatime", "mode=755",
                        "size=65536k"]
        }
    ],
    "linux": {
        "namespaces": [
            { "type": "pid" },
            { "type": "network" },
            { "type": "ipc" },
            { "type": "uts" },
            { "type": "mount" },
            { "type": "cgroup" }
        ],
        "resources": {
            "memory": { "limit": 268435456 },
            "cpu": {
                "shares": 1024,
                "quota": 100000,
                "period": 100000
            },
            "pids": { "limit": 100 }
        },
        "seccomp": {
            "defaultAction": "SCMP_ACT_ERRNO",
            "architectures": ["SCMP_ARCH_X86_64"],
            "syscalls": [
                {
                    "names": ["read", "write", "open", "close",
                              "stat", "fstat", "mmap", "mprotect",
                              "exit_group"],
                    "action": "SCMP_ACT_ALLOW"
                }
            ]
        },
        "maskedPaths": [
            "/proc/acpi", "/proc/kcore", "/proc/keys",
            "/proc/sched_debug", "/sys/firmware"
        ],
        "readonlyPaths": [
            "/proc/asound", "/proc/bus", "/proc/fs",
            "/proc/irq", "/proc/sys", "/proc/sysrq-trigger"
        ]
    }
}
```

---

## 3. OCI Runtime Lifecycle

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Container states:                                       │
  │                                                           │
  │  creating ──► created ──► running ──► stopped            │
  │                  │                       │                 │
  │                  └────────────────────────┘                │
  │                      (can also go directly                │
  │                       to stopped on error)                │
  │                                                           │
  │  Lifecycle commands (runc CLI):                          │
  │  ┌──────────┬──────────────────────────────────────┐    │
  │  │ create   │ Set up container (namespaces, cgroups,    │
  │  │          │ mounts) but don't start process.          │
  │  │          │ State: creating → created                  │
  │  ├──────────┼──────────────────────────────────────┤    │
  │  │ start    │ Execute container process (args from      │
  │  │          │ config.json). State: created → running    │
  │  ├──────────┼──────────────────────────────────────┤    │
  │  │ kill     │ Send signal to container process.         │
  │  │          │ State: running → stopped (after exit)     │
  │  ├──────────┼──────────────────────────────────────┤    │
  │  │ delete   │ Clean up container resources (cgroups,    │
  │  │          │ state files). Must be stopped first.      │
  │  ├──────────┼──────────────────────────────────────┤    │
  │  │ state    │ Query container state (JSON output)       │
  │  └──────────┴──────────────────────────────────────┘    │
  │                                                           │
  │  Why create/start are separate:                          │
  │  - create sets up the "jail" (namespaces, mounts)       │
  │  - Allows hooks to run between create and start         │
  │    (e.g., network setup by CNI plugin)                  │
  │  - start executes the user's process inside the jail    │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. OCI Image Specification

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  OCI Image = manifest + config + layers                  │
  │                                                           │
  │  Image Manifest (application/vnd.oci.image.manifest.v1): │
  │  {                                                        │
  │    "schemaVersion": 2,                                    │
  │    "mediaType": "...manifest.v1+json",                   │
  │    "config": {                                            │
  │      "mediaType": "...image.config.v1+json",             │
  │      "digest": "sha256:abc123...",                        │
  │      "size": 1470                                         │
  │    },                                                     │
  │    "layers": [                                            │
  │      {                                                    │
  │        "mediaType": "...layer.v1.tar+gzip",              │
  │        "digest": "sha256:layer1...",                      │
  │        "size": 32654321                                   │
  │      },                                                   │
  │      {                                                    │
  │        "mediaType": "...layer.v1.tar+gzip",              │
  │        "digest": "sha256:layer2...",                      │
  │        "size": 1234567                                    │
  │      }                                                    │
  │    ]                                                      │
  │  }                                                        │
  │                                                           │
  │  Content-addressable:                                    │
  │  - Every blob identified by its SHA256 digest            │
  │  - Manifest references config and layers by digest      │
  │  - Tamper-evident: changing any byte changes the digest  │
  │                                                           │
  │  Image Index (multi-arch):                               │
  │  {                                                        │
  │    "manifests": [                                         │
  │      { "platform": {"os":"linux","arch":"amd64"},        │
  │        "digest": "sha256:..." },                         │
  │      { "platform": {"os":"linux","arch":"arm64"},        │
  │        "digest": "sha256:..." }                          │
  │    ]                                                      │
  │  }                                                        │
  │  → One tag → multiple architectures                      │
  │  → Client picks the right one automatically              │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Image Layers and Content-Addressable Storage

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Dockerfile → Image Layers:                              │
  │                                                           │
  │  FROM ubuntu:22.04           → Layer 1 (base, ~29MB)    │
  │  RUN apt-get install nginx   → Layer 2 (deps, ~50MB)    │
  │  COPY app.conf /etc/nginx/   → Layer 3 (config, ~1KB)   │
  │  COPY html/ /var/www/html/   → Layer 4 (content, ~5MB)  │
  │                                                           │
  │  Content-Addressable Storage (CAS):                      │
  │  /var/lib/containerd/io.containerd.content.v1.content/   │
  │  └── blobs/sha256/                                       │
  │      ├── abc123...  (layer 1 tarball)                    │
  │      ├── def456...  (layer 2 tarball)                    │
  │      ├── ghi789...  (layer 3 tarball)                    │
  │      └── jkl012...  (layer 4 tarball)                    │
  │                                                           │
  │  Layer sharing:                                          │
  │  Image A: FROM ubuntu:22.04 → uses Layer 1              │
  │  Image B: FROM ubuntu:22.04 → shares same Layer 1!      │
  │  → Only stored ONCE on disk                              │
  │  → Only pulled ONCE from registry                        │
  │                                                           │
  │  Layer diffing:                                          │
  │  Each layer is a tar of filesystem CHANGES:              │
  │  - Added files: present in tar                           │
  │  - Modified files: new version in tar (replaces lower)  │
  │  - Deleted files: whiteout entry (.wh.filename)         │
  │  - Deleted dirs: opaque whiteout (.wh..wh..opq)         │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Describe the OCI Runtime Specification's config.json and the container lifecycle.**
**A:** The OCI Runtime Spec defines config.json, a JSON document describing everything needed to run a container: (1) `process` — the command to execute, environment variables, working directory, capabilities (bounding/effective/permitted/ambient sets), rlimits, and `noNewPrivileges` flag. (2) `root` — path to the rootfs and whether it's read-only. (3) `mounts` — mount points (/proc, /sys, /dev, bind mounts, volumes). (4) `linux.namespaces` — which namespaces to create (pid, network, mount, uts, ipc, user, cgroup). Can also join existing namespaces by specifying a path. (5) `linux.resources` — cgroup limits (memory.limit, cpu.shares/quota, pids.limit). (6) `linux.seccomp` — syscall filter (default action + allowlist/blocklist). (7) `maskedPaths`/`readonlyPaths` — /proc and /sys protection. The lifecycle: `create` → sets up namespaces, cgroups, mounts, runs `createRuntime` and `createContainer` hooks, state becomes "created". `start` → executes the process from config.json, runs `startContainer` hook, state becomes "running". `kill` → sends signal. `delete` → removes cgroup, state files, runs `poststop` hooks. The create/start split allows network plugins (CNI) to configure networking between creation and process start.

**Q2: How are OCI images structured, and why is content-addressable storage important?**
**A:** An OCI image consists of three parts: (1) **Image Index** (optional) — multi-architecture manifest list, mapping platform (os/arch) to the correct manifest. (2) **Image Manifest** — references the config blob and an ordered list of layer blobs, all by SHA256 digest. (3) **Image Config** — JSON with execution parameters (env, entrypoint, cmd, exposed ports, labels) and history of how each layer was created. (4) **Layers** — tar+gzip archives containing filesystem changes (diffs). Content-addressable storage (CAS) means every blob is stored by its cryptographic hash. Benefits: (a) Deduplication — identical layers from different images are stored once. If 100 images use ubuntu:22.04 base, only one copy exists. (b) Integrity — any modification changes the hash, detected immediately. Pull verification is automatic. (c) Caching — layer already present locally? Skip download. (d) Concurrent pull — layers can be pulled in parallel (independent blobs). (e) Garbage collection — unreferenced blobs can be safely removed.

---

## Summary

- OCI: three specs — Runtime (config.json), Image (layers+manifest), Distribution (registry API)
- config.json: process, root, mounts, namespaces, cgroup resources, seccomp, masked paths
- Container lifecycle: creating → created → running → stopped → deleted
- create/start split: allows hooks (network setup) between jail creation and process exec
- OCI Image: manifest references config + layers by SHA256 digest (CAS)
- Layers: tar of filesystem diffs; whiteout files for deletions
- Multi-arch: Image Index maps platform → manifest → layers

---

[Previous: IO Controller and systemd ←](Chapter_14_IO_Systemd.md) | [Next: runc Internals →](Chapter_16_Runc_Internals.md)
