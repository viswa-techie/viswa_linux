# Chapter 14: IO Controller and systemd Integration

## Learning Goals
- Understand cgroups v2 IO controller and writeback accounting
- Learn systemd's cgroup management model (slices, scopes, services)
- Master systemd resource control directives
- Know how container runtimes integrate with systemd

---

## 1. IO Controller v2

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  v1 blkio problems:                                      │
  │  - Only tracked DIRECT I/O (bypass page cache)          │
  │  - Buffered writes (write() → page cache → writeback)   │
  │    charged to the FLUSHER thread, not the writing cgroup │
  │  - 90%+ of I/O is buffered → v1 blkio nearly useless   │
  │                                                           │
  │  v2 io controller fixes this:                            │
  │  - Writeback I/O charged to the ORIGINATING cgroup      │
  │  - Page cache pages remember their cgroup (memcg tag)   │
  │  - When writeback flushes dirty pages, charge goes to   │
  │    the cgroup that dirtied them                          │
  │                                                           │
  │  Config:                                                 │
  │  echo "+io" > /sys/fs/cgroup/<path>/cgroup.subtree_control│
  │                                                           │
  │  io.max — hard limits (per device):                      │
  │  echo "8:0 rbps=10485760 wbps=5242880 riops=1000 wiops=500" \
  │       > io.max                                           │
  │  8:0 = major:minor of device (e.g., /dev/sda)           │
  │  rbps = read bytes per second                            │
  │  wbps = write bytes per second                           │
  │  riops = read I/O operations per second                  │
  │  wiops = write I/O operations per second                 │
  │                                                           │
  │  io.weight — proportional weight (1-10000, default 100): │
  │  echo "default 200" > io.weight                          │
  │  echo "8:0 500" > io.weight   (per-device override)     │
  │                                                           │
  │  io.stat — current I/O stats:                            │
  │  8:0 rbytes=1048576 wbytes=524288 rios=100 wios=50      │
  │      dbytes=0 dios=0                                     │
  │                                                           │
  │  io.pressure — PSI for I/O:                              │
  │  some avg10=2.50 avg60=1.20 avg300=0.50 total=45000     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. systemd Cgroup Model

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  systemd manages cgroups as THREE unit types:            │
  │                                                           │
  │  ┌─────────────────────────────────────────────┐         │
  │  │ Slice  — hierarchical grouping (no processes)        │
  │  │ Scope  — externally created process groups           │
  │  │ Service — systemd-managed daemon processes           │
  │  └─────────────────────────────────────────────┘         │
  │                                                           │
  │  Default hierarchy:                                      │
  │  /sys/fs/cgroup/ (root slice: -.slice)                   │
  │  ├── init.scope          ← PID 1 (systemd itself)      │
  │  ├── system.slice/       ← system services              │
  │  │   ├── sshd.service/                                   │
  │  │   ├── nginx.service/                                  │
  │  │   ├── docker.service/                                 │
  │  │   └── containerd.service/                             │
  │  ├── user.slice/         ← user sessions                │
  │  │   └── user-1000.slice/                               │
  │  │       ├── session-1.scope/  ← login session          │
  │  │       └── user@1000.service/                         │
  │  │           └── app.slice/                              │
  │  │               └── my-app.service/                    │
  │  └── machine.slice/      ← VMs and containers          │
  │      ├── docker-abc.scope/   ← container via Docker    │
  │      └── libvirt-qemu-vm.scope/ ← VM via libvirt       │
  │                                                           │
  │  Resource distribution:                                  │
  │  system.slice vs user.slice vs machine.slice             │
  │  → each gets proportional share based on weight          │
  │  → prevents user login sessions from starving services   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. systemd Resource Control Directives

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  In unit file [Service] or [Slice] section:              │
  │                                                           │
  │  CPU:                                                    │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ CPUWeight=200         ← cpu.weight (v2)       │      │
  │  │ CPUQuota=150%         ← cpu.max (1.5 cores)   │      │
  │  │ AllowedCPUs=0-3       ← cpuset.cpus           │      │
  │  │ AllowedMemoryNodes=0  ← cpuset.mems (NUMA)    │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Memory:                                                 │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ MemoryMax=512M        ← memory.max            │      │
  │  │ MemoryHigh=400M       ← memory.high           │      │
  │  │ MemoryLow=100M        ← memory.low            │      │
  │  │ MemoryMin=50M         ← memory.min            │      │
  │  │ MemorySwapMax=256M    ← memory.swap.max       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  IO:                                                     │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ IOWeight=200          ← io.weight              │      │
  │  │ IOReadBandwidthMax=/dev/sda 100M               │      │
  │  │ IOWriteBandwidthMax=/dev/sda 50M               │      │
  │  │ IOReadIOPSMax=/dev/sda 1000                    │      │
  │  │ IOWriteIOPSMax=/dev/sda 500                    │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Process:                                                │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ TasksMax=100          ← pids.max               │      │
  │  │ Delegate=yes          ← allow sub-cgroup mgmt  │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Runtime override (no restart needed):                   │
  │  systemctl set-property nginx.service MemoryMax=1G       │
  │  → writes to /sys/fs/cgroup/system.slice/nginx.service/ │
  │  → persistent if --runtime flag not used                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Container Runtime + systemd Integration

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Docker/containerd with systemd cgroup driver:           │
  │                                                           │
  │  /etc/docker/daemon.json:                                │
  │  { "exec-opts": ["native.cgroupdriver=systemd"] }       │
  │                                                           │
  │  Without systemd driver (cgroupfs driver):               │
  │  Docker creates cgroups directly under /sys/fs/cgroup/   │
  │  → TWO cgroup managers (systemd + Docker) = conflicts!  │
  │  → Kubernetes requires systemd driver since v1.22       │
  │                                                           │
  │  With systemd driver:                                    │
  │  Docker asks systemd to create cgroups via D-Bus         │
  │  → systemd is the SINGLE cgroup manager                  │
  │  → No conflicts                                          │
  │                                                           │
  │  Kubernetes cgroup layout:                               │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ /sys/fs/cgroup/                              │       │
  │  │ └── kubepods.slice/                          │       │
  │  │     ├── kubepods-burstable.slice/            │       │
  │  │     │   ├── kubepods-pod-abc.slice/          │       │
  │  │     │   │   ├── cri-containerd-xyz.scope/    │       │
  │  │     │   │   │   ├── cgroup.procs             │       │
  │  │     │   │   │   ├── cpu.max                  │       │
  │  │     │   │   │   └── memory.max               │       │
  │  │     │   │   └── cri-containerd-init.scope/   │       │
  │  │     │   └── ...                              │       │
  │  │     ├── kubepods-guaranteed.slice/ (QoS)     │       │
  │  │     └── kubepods-besteffort.slice/ (QoS)     │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Kubernetes QoS → cgroup mapping:                        │
  │  - Guaranteed: requests == limits → dedicated resources  │
  │  - Burstable: requests < limits → can burst             │
  │  - BestEffort: no requests/limits → use leftover         │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Cgroup Monitoring and Debugging

```bash
# View cgroup hierarchy as tree
systemd-cgls

# Show resource usage per cgroup
systemd-cgtop

# Check a specific cgroup's controllers
cat /sys/fs/cgroup/system.slice/docker.service/cgroup.controllers

# Runtime resource stats
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
cat /sys/fs/cgroup/system.slice/nginx.service/memory.events
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.stat

# PSI monitoring
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.pressure
cat /sys/fs/cgroup/system.slice/nginx.service/memory.pressure

# Find which cgroup a process belongs to
cat /proc/<pid>/cgroup

# Query systemd for resource properties
systemctl show nginx.service -p MemoryMax,CPUQuota,TasksMax

# Docker stats (uses cgroup counters underneath)
docker stats --no-stream
```

---

## Interview Questions

**Q1: Why did the v1 blkio controller fail for buffered I/O, and how does v2 fix it?**
**A:** In v1, the blkio controller only tracked direct I/O because it hooked at the block layer (request submission). Buffered I/O follows a different path: the process writes to page cache (charged to memory controller), then the kernel's writeback flusher thread (`kworker/flush-*`) writes dirty pages to disk. The blkio controller charged the I/O to the flusher thread's cgroup (usually root), not the cgroup that generated the data. Since 90%+ of I/O is buffered, v1 blkio was nearly useless. v2 fixes this through memory controller integration: when a page is dirtied, it's tagged with the `mem_cgroup` that dirtied it (the page's `page->memcg_data` field). During writeback, the kernel retrieves this tag and charges the I/O to the correct cgroup, regardless of which kernel thread performs the write. This requires both memory and io controllers to be enabled in the same cgroup hierarchy — another reason for v2's unified hierarchy.

**Q2: How does systemd manage cgroups, and why is the systemd cgroup driver preferred for Kubernetes?**
**A:** systemd is the cgroup manager on most Linux systems — it creates and manages the cgroup hierarchy at boot (slices: system.slice, user.slice, machine.slice) and for every service/scope. When a container runtime (Docker, containerd) uses the "cgroupfs" driver, it creates cgroups directly by writing to /sys/fs/cgroup, bypassing systemd. This creates TWO cgroup managers that don't know about each other, leading to: (1) Resource accounting inconsistency (systemd doesn't see container resource usage). (2) Potential cgroup tree corruption. (3) systemd commands like `systemctl status` don't show container resources. The "systemd" driver instead asks systemd to create cgroups via D-Bus API, making systemd the single source of truth. Kubernetes mandated the systemd driver starting v1.22 because the kubelet also needs cgroup management for pod QoS classes (Guaranteed, Burstable, BestEffort), and having three cgroup managers (systemd + Docker + kubelet) was fragile.

---

## Summary

- v2 IO controller: charges writeback I/O to originating cgroup (fixes v1 blkio's blind spot)
- io.max: per-device BPS and IOPS hard limits
- io.weight: proportional I/O sharing (1-10000)
- systemd cgroup model: -.slice → system.slice + user.slice + machine.slice
- systemd unit types: slice (hierarchy), scope (external processes), service (daemons)
- systemd directives: CPUWeight, MemoryMax, MemoryHigh, IOWeight, TasksMax
- Container runtimes: use systemd cgroup driver (not cgroupfs) for single manager
- Kubernetes: kubepods.slice with QoS sub-slices (guaranteed, burstable, besteffort)

---

[Previous: CPU and Memory Controllers ←](Chapter_13_CPU_Memory_Controllers.md) | [Next: OCI Specification →](Chapter_15_OCI_Specification.md)
