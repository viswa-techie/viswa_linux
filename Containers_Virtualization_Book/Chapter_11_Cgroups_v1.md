# Chapter 11: Cgroups v1 Architecture

## Learning Goals
- Understand cgroups v1 hierarchy and controller model
- Learn the filesystem interface for cgroup management
- Master CPU, memory, and PID controllers
- Know cgroups v1 limitations that led to v2

---

## 1. Cgroups v1 Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cgroups (Control Groups): organize processes into       │
  │  hierarchical groups and apply resource limits           │
  │                                                           │
  │  Two functions:                                          │
  │  1. Resource LIMITING — restrict CPU, memory, I/O, PIDs │
  │  2. Resource ACCOUNTING — measure usage per group        │
  │                                                           │
  │  cgroups v1: each controller has its OWN hierarchy       │
  │                                                           │
  │  /sys/fs/cgroup/                                         │
  │  ├── cpu/                     ← cpu controller           │
  │  │   ├── docker/                                         │
  │  │   │   ├── container_a/                               │
  │  │   │   │   ├── cgroup.procs   (list of PIDs)          │
  │  │   │   │   ├── cpu.shares     (relative weight)       │
  │  │   │   │   └── cpu.cfs_quota_us (hard limit)          │
  │  │   │   └── container_b/                               │
  │  │   └── system.slice/                                   │
  │  ├── memory/                  ← memory controller        │
  │  │   ├── docker/                                         │
  │  │   │   ├── container_a/                               │
  │  │   │   │   ├── cgroup.procs                           │
  │  │   │   │   ├── memory.limit_in_bytes                  │
  │  │   │   │   └── memory.usage_in_bytes                  │
  │  │   │   └── container_b/                               │
  │  │   └── system.slice/                                   │
  │  ├── pids/                    ← pids controller          │
  │  ├── blkio/                   ← block I/O controller     │
  │  ├── cpuset/                  ← CPU pinning              │
  │  ├── devices/                 ← device access control    │
  │  ├── freezer/                 ← pause/resume processes   │
  │  ├── net_cls,net_prio/        ← network classifier       │
  │  └── hugetlb/                 ← huge pages               │
  │                                                           │
  │  Problem: container_a has DIFFERENT paths in each        │
  │  controller's hierarchy! Hard to manage consistently.    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. CPU Controller (v1)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CPU controller: CFS bandwidth control                   │
  │                                                           │
  │  Proportional sharing (cpu.shares):                      │
  │  ┌────────────────────────────────────────────┐          │
  │  │ Container A: cpu.shares = 1024 (default)  │          │
  │  │ Container B: cpu.shares = 512             │          │
  │  │ Container C: cpu.shares = 2048            │          │
  │  │                                            │          │
  │  │ When all compete for CPU:                  │          │
  │  │ A gets 1024/(1024+512+2048) = 28.6%       │          │
  │  │ B gets 512/(1024+512+2048) = 14.3%        │          │
  │  │ C gets 2048/(1024+512+2048) = 57.1%       │          │
  │  │                                            │          │
  │  │ When B is idle: A and C split B's share   │          │
  │  │ (work-conserving — no wasted CPU)         │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  Hard limit (CFS quota):                                 │
  │  ┌────────────────────────────────────────────┐          │
  │  │ cpu.cfs_period_us = 100000  (100ms)       │          │
  │  │ cpu.cfs_quota_us  = 50000   (50ms)        │          │
  │  │                                            │          │
  │  │ → Container gets 50ms out of every 100ms  │          │
  │  │ → Equivalent to 0.5 CPU cores             │          │
  │  │                                            │          │
  │  │ docker run --cpus=2.5 ...                 │          │
  │  │ → period=100000, quota=250000             │          │
  │  │ → 250ms out of every 100ms = 2.5 cores   │          │
  │  │                                            │          │
  │  │ quota=-1 → no limit (unlimited CPU)       │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  CPU pinning (cpuset controller):                        │
  │  ┌────────────────────────────────────────────┐          │
  │  │ cpuset.cpus = 0-3     ← use cores 0,1,2,3│          │
  │  │ cpuset.mems = 0       ← NUMA node 0 only │          │
  │  │                                            │          │
  │  │ docker run --cpuset-cpus="0,2" ...        │          │
  │  │ → process can only run on cores 0 and 2   │          │
  │  └────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Controller (v1)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Memory controller files:                                │
  │  ┌────────────────────────────────┬──────────────────┐   │
  │  │ File                          │ Purpose           │   │
  │  ├────────────────────────────────┼──────────────────┤   │
  │  │ memory.limit_in_bytes         │ Hard limit (OOM)  │   │
  │  │ memory.soft_limit_in_bytes    │ Reclaim target    │   │
  │  │ memory.usage_in_bytes         │ Current usage     │   │
  │  │ memory.max_usage_in_bytes     │ Peak usage        │   │
  │  │ memory.memsw.limit_in_bytes   │ Memory+swap limit │   │
  │  │ memory.kmem.limit_in_bytes    │ Kernel mem limit  │   │
  │  │ memory.oom_control            │ OOM behavior      │   │
  │  │ memory.stat                   │ Detailed stats    │   │
  │  │ memory.failcnt                │ # of limit hits  │   │
  │  └────────────────────────────────┴──────────────────┘   │
  │                                                           │
  │  OOM (Out of Memory) behavior:                           │
  │  ┌────────────────────────────────────────────┐          │
  │  │ 1. Process in container allocates memory   │          │
  │  │ 2. Usage reaches memory.limit_in_bytes    │          │
  │  │ 3. Kernel tries to reclaim pages           │          │
  │  │ 4. If reclaim fails → OOM killer triggered │          │
  │  │ 5. OOM kills process with highest oom_score│          │
  │  │    within the cgroup (not host processes!) │          │
  │  │ 6. dmesg: "Memory cgroup out of memory"   │          │
  │  │                                            │          │
  │  │ docker run --memory=512m --oom-kill-disable│          │
  │  │ → memory.oom_control: 1 (disabled)        │          │
  │  │ → processes PAUSE instead of being killed  │          │
  │  │ → can cause system hang if not monitored!  │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  # Set 256MB memory limit for a cgroup:                  │
  │  echo 268435456 > memory.limit_in_bytes                  │
  │  echo 268435456 > memory.memsw.limit_in_bytes            │
  │  echo $PID > cgroup.procs                                │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Other v1 Controllers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  pids controller (prevent fork bombs):                   │
  │  ┌────────────────────────────────────────────┐          │
  │  │ pids.max = 100                             │          │
  │  │ → cgroup can have at most 100 processes    │          │
  │  │ → fork() returns -EAGAIN when limit hit    │          │
  │  │ pids.current → current process count       │          │
  │  │                                            │          │
  │  │ docker run --pids-limit=100 ...            │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  blkio controller (I/O throttling):                      │
  │  ┌────────────────────────────────────────────┐          │
  │  │ blkio.throttle.read_bps_device             │          │
  │  │   = "8:0 10485760"  ← 10MB/s read on sda  │          │
  │  │ blkio.throttle.write_bps_device            │          │
  │  │   = "8:0 5242880"   ← 5MB/s write on sda  │          │
  │  │ blkio.throttle.read_iops_device            │          │
  │  │   = "8:0 1000"      ← 1000 IOPS read      │          │
  │  │                                            │          │
  │  │ docker run --device-read-bps /dev/sda:10mb │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  devices controller (device access whitelist):           │
  │  ┌────────────────────────────────────────────┐          │
  │  │ devices.allow = "c 1:3 rwm"  ← /dev/null  │          │
  │  │ devices.allow = "c 1:5 rwm"  ← /dev/zero  │          │
  │  │ devices.allow = "c 1:8 rwm"  ← /dev/random│          │
  │  │ devices.deny  = "a"          ← deny all    │          │
  │  │                                            │          │
  │  │ docker run --device=/dev/video0 ...        │          │
  │  │ → adds "c 81:0 rwm" to devices.allow      │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  freezer controller:                                     │
  │  ┌────────────────────────────────────────────┐          │
  │  │ freezer.state = FROZEN   ← pause all procs│          │
  │  │ freezer.state = THAWED   ← resume all     │          │
  │  │                                            │          │
  │  │ Used by: docker pause/unpause              │          │
  │  │          CRIU checkpoint/restore           │          │
  │  └────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. cgroups v1 Limitations

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Why cgroups v2 was needed:                              │
  │                                                           │
  │  1. Multiple hierarchies = nightmare                     │
  │     v1: each controller has separate hierarchy           │
  │     → container has different paths in cpu/ vs memory/   │
  │     → inconsistent, hard to manage                       │
  │                                                           │
  │  2. No unified resource model                            │
  │     → cannot express "this group gets 50% of all         │
  │       resources" — must set each controller separately   │
  │                                                           │
  │  3. Thread-level granularity missing                     │
  │     → v1 cannot put threads of the same process in      │
  │       different cpu cgroups easily                       │
  │                                                           │
  │  4. No delegation model                                  │
  │     → container manager cannot safely delegate cgroup    │
  │       management to container processes                  │
  │                                                           │
  │  5. Internal nodes can have processes                    │
  │     → parent cgroup has processes AND children           │
  │     → makes resource distribution ambiguous              │
  │     → v2 enforces "no internal process" rule            │
  │                                                           │
  │  6. Notification mechanism limited                       │
  │     → v1 uses eventfd, hard to use correctly            │
  │     → v2 uses inotify + readable state files            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does the CPU CFS bandwidth controller work in cgroups v1?**
**A:** The CFS bandwidth controller provides two mechanisms: (1) **Proportional sharing** via `cpu.shares` — when multiple cgroups compete for CPU, each gets a proportion based on its shares weight. Default is 1024. If group A has 1024 shares and group B has 512 shares, A gets 2/3 and B gets 1/3 of CPU time when both are busy. This is work-conserving: if B is idle, A gets 100%. (2) **Hard limit** via `cpu.cfs_quota_us` and `cpu.cfs_period_us` — the group gets at most `quota` microseconds of CPU time per `period` microseconds. Example: quota=200000, period=100000 → 200ms per 100ms → 2 CPU cores maximum. If the group exhausts its quota, the scheduler throttles all tasks until the next period. The throttle count is visible in `cpu.stat`. Docker's `--cpus=2.5` sets quota=250000 and period=100000. Setting quota=-1 removes the hard limit. Note: CFS quota has caused latency issues in production (short bursts get throttled) — kernel 5.6 introduced "burst" to allow temporary overuse.

**Q2: Explain OOM handling in the memory cgroup controller.**
**A:** When a process in a cgroup tries to allocate memory beyond `memory.limit_in_bytes`: (1) The kernel first attempts reclaim — reclaiming page cache, swapping anonymous pages (if swap is available and `memory.memsw.limit_in_bytes` permits). (2) If reclaim insufficient, the cgroup-aware OOM killer is triggered. It only kills processes WITHIN the cgroup (not host processes or other containers). (3) The OOM killer selects the process with the highest `oom_score_adj` + memory usage combination. (4) `dmesg` logs: "Memory cgroup out of memory: Killed process <pid>". (5) `memory.failcnt` increments. (6) In Docker, the container exits with code 137 (128+SIGKILL=137). Alternatives: `memory.oom_control` can disable the OOM killer (processes are paused/blocked on allocation instead), but this risks system hangs. `memory.soft_limit_in_bytes` sets a reclaim pressure point (kernel prefers reclaiming from groups exceeding soft limit) but doesn't enforce a hard limit.

---

## Summary

- Cgroups v1: each controller (cpu, memory, blkio, pids, devices, freezer) in separate hierarchy
- CPU: cpu.shares (proportional), cpu.cfs_quota_us/period_us (hard limit), cpuset.cpus (pinning)
- Memory: memory.limit_in_bytes (hard limit), OOM killer within cgroup only
- PIDs: pids.max prevents fork bombs
- Devices: whitelist model (devices.allow/deny) for /dev access
- Freezer: FROZEN/THAWED states for pause/resume
- v1 limitations: multiple hierarchies, no delegation, inconsistent management → led to v2

---

[Previous: Namespace Lifecycle ←](Chapter_10_NS_Lifecycle.md) | [Next: Cgroups v2 Unified Hierarchy →](Chapter_12_Cgroups_v2.md)
