# Chapter 12: Cgroups v2 Unified Hierarchy

## Learning Goals
- Understand cgroups v2 unified hierarchy design
- Learn the "no internal process" rule and domain model
- Master v2 controller interfaces (cpu, memory, io, pids)
- Know v1 to v2 migration and hybrid mode

---

## 1. Cgroups v2 Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  v2: ONE unified hierarchy for ALL controllers           │
  │                                                           │
  │  /sys/fs/cgroup/  (unified, cgroup2 filesystem)          │
  │  ├── cgroup.controllers    "cpu memory io pids"          │
  │  ├── cgroup.subtree_control  "cpu memory io pids"        │
  │  ├── cgroup.procs          (PIDs in root cgroup)         │
  │  │                                                        │
  │  ├── system.slice/                                       │
  │  │   ├── cgroup.controllers  "cpu memory io pids"        │
  │  │   ├── cgroup.subtree_control                          │
  │  │   ├── sshd.service/                                   │
  │  │   │   ├── cgroup.procs                                │
  │  │   │   ├── cpu.max           "max 100000"              │
  │  │   │   ├── cpu.weight        100                       │
  │  │   │   ├── memory.max        max                       │
  │  │   │   └── pids.max          max                       │
  │  │   └── docker.service/                                 │
  │  │                                                        │
  │  └── docker/                                             │
  │      ├── cgroup.subtree_control  "+cpu +memory +io +pids"│
  │      ├── container_a/                                    │
  │      │   ├── cgroup.procs        "12345"                 │
  │      │   ├── cpu.max             "50000 100000"          │
  │      │   ├── cpu.weight          100                     │
  │      │   ├── memory.max          268435456               │
  │      │   ├── memory.current      134217728               │
  │      │   ├── io.max              "8:0 rbps=10485760"     │
  │      │   └── pids.max            100                     │
  │      └── container_b/                                    │
  │          └── ...                                          │
  │                                                           │
  │  Key difference: SAME hierarchy path for ALL controllers │
  │  container_a is at /docker/container_a for cpu AND       │
  │  memory AND io AND pids                                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. No Internal Process Rule

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  v1 problem:                                             │
  │  /cgroup/A/                                              │
  │    ├── cgroup.procs  → [P1, P2]  ← processes here!     │
  │    ├── child_1/                                          │
  │    │   └── cgroup.procs → [P3]                          │
  │    └── child_2/                                          │
  │        └── cgroup.procs → [P4]                          │
  │                                                           │
  │  How much CPU does P1 get vs child_1?                   │
  │  → Ambiguous! P1 competes with child_1 AND child_2     │
  │  → Different controllers resolve this differently       │
  │                                                           │
  │  v2 rule: NO INTERNAL PROCESSES                          │
  │  Processes can only live in LEAF cgroups                 │
  │                                                           │
  │  /cgroup/A/                                              │
  │    ├── cgroup.procs  → []  ← EMPTY (has children)      │
  │    ├── cgroup.subtree_control = "+cpu +memory"           │
  │    ├── child_1/                                          │
  │    │   └── cgroup.procs → [P1, P2, P3]  ← leaf OK     │
  │    └── child_2/                                          │
  │        └── cgroup.procs → [P4]            ← leaf OK     │
  │                                                           │
  │  Resource distribution is unambiguous:                   │
  │  A distributes resources between child_1 and child_2    │
  │  child_1's resources are shared among P1, P2, P3        │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. v2 Controller Interfaces

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CPU controller (v2):                                    │
  │  ┌────────────────────────────────────────────┐          │
  │  │ cpu.weight      = 100  (default, range 1-10000)      │
  │  │   Replaces cpu.shares (1024 default in v1)           │
  │  │   Normalized: 100 = normal priority                  │
  │  │                                                       │
  │  │ cpu.max         = "200000 100000"                    │
  │  │   Replaces cfs_quota_us / cfs_period_us              │
  │  │   Format: "$MAX $PERIOD" (quota period)              │
  │  │   "200000 100000" → 2 cores max                     │
  │  │   "max 100000" → unlimited                          │
  │  │                                                       │
  │  │ cpu.stat:        usage_usec, user_usec, system_usec  │
  │  │                  nr_periods, nr_throttled             │
  │  │                  throttled_usec                       │
  │  │                                                       │
  │  │ cpu.pressure:    PSI (Pressure Stall Information)    │
  │  │   some avg10=0.00 avg60=0.00 avg300=0.00 total=0    │
  │  │   full avg10=0.00 avg60=0.00 avg300=0.00 total=0    │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  Memory controller (v2):                                 │
  │  ┌────────────────────────────────────────────┐          │
  │  │ memory.max      = 268435456  (hard limit, in bytes)  │
  │  │   Replaces memory.limit_in_bytes                     │
  │  │   "max" → no limit                                   │
  │  │                                                       │
  │  │ memory.high     = 134217728  (throttle point)        │
  │  │   NEW in v2! No v1 equivalent.                       │
  │  │   When exceeded: kernel slows allocations            │
  │  │   (reclaim pressure, NOT OOM)                        │
  │  │   Preferred over memory.max for most use cases       │
  │  │                                                       │
  │  │ memory.low      = 67108864  (best-effort protection) │
  │  │   Below this: kernel avoids reclaiming from cgroup   │
  │  │                                                       │
  │  │ memory.min      = 33554432  (hard protection)        │
  │  │   Below this: kernel NEVER reclaims from cgroup      │
  │  │                                                       │
  │  │ memory.current  → current usage (read-only)          │
  │  │ memory.swap.max → swap limit                         │
  │  │ memory.swap.current → current swap usage             │
  │  │ memory.events   → oom_kill, max, high counters       │
  │  │ memory.pressure → PSI information                    │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  IO controller (v2):                                     │
  │  ┌────────────────────────────────────────────┐          │
  │  │ io.max = "8:0 rbps=10485760 wbps=5242880" │          │
  │  │   Per-device: read bytes/s, write bytes/s  │          │
  │  │   Also: riops, wiops (IOPS limits)         │          │
  │  │                                            │          │
  │  │ io.weight = "default 100"                  │          │
  │  │   Proportional I/O weight (1-10000)        │          │
  │  │                                            │          │
  │  │ io.stat → per-device I/O statistics        │          │
  │  │ io.pressure → PSI for I/O                  │          │
  │  └────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. PSI (Pressure Stall Information)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PSI: added in kernel 4.20, integrated with cgroups v2  │
  │  Measures resource contention per-cgroup                 │
  │                                                           │
  │  cat /sys/fs/cgroup/docker/container_a/cpu.pressure      │
  │  some avg10=4.50 avg60=2.10 avg300=0.85 total=125000    │
  │  full avg10=0.00 avg60=0.00 avg300=0.00 total=0         │
  │                                                           │
  │  "some": percentage of time at least ONE task was        │
  │          stalled (waiting for resource)                   │
  │  "full": percentage of time ALL tasks were stalled       │
  │          (cgroup completely unproductive)                 │
  │                                                           │
  │  Available for: cpu.pressure, memory.pressure, io.pressure│
  │                                                           │
  │  Use case: auto-scaling                                  │
  │  - memory.pressure some > 10% → container needs more RAM│
  │  - cpu.pressure full > 5% → CPU bottleneck              │
  │  - io.pressure some > 20% → I/O contention              │
  │                                                           │
  │  PSI triggers (event-based monitoring):                  │
  │  echo "some 50000 1000000" > cpu.pressure                │
  │  → trigger when "some" exceeds 50ms per 1s window       │
  │  → kernel sends notification via poll/epoll              │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Cgroup Delegation (v2)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Delegation: allow non-root to manage sub-cgroups        │
  │  Critical for: rootless containers, systemd user services│
  │                                                           │
  │  /sys/fs/cgroup/user.slice/user-1000.slice/              │
  │    owned by UID 1000                                      │
  │    ├── cgroup.procs              (user can write)        │
  │    ├── cgroup.subtree_control    (user can write)        │
  │    ├── cgroup.threads            (user can write)        │
  │    ├── my_container_a/           (user can create)       │
  │    │   ├── cgroup.procs                                  │
  │    │   ├── cpu.max                                       │
  │    │   └── memory.max                                    │
  │    └── my_container_b/                                   │
  │                                                           │
  │  Requirements for safe delegation:                       │
  │  1. Delegate directory (not files) to unprivileged user  │
  │  2. User can create subdirectories (sub-cgroups)         │
  │  3. User can write cgroup.procs (move own processes)     │
  │  4. User CANNOT write cpu.max in delegated root          │
  │     (but CAN write cpu.max in their sub-cgroups)         │
  │  5. Resource limits set by root at delegation boundary   │
  │                                                           │
  │  systemd v240+ supports cgroup delegation:               │
  │  [Service]                                                │
  │  Delegate=yes                                            │
  │  → systemd makes the service's cgroup delegatable       │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What are the key differences between cgroups v1 and v2?**
**A:** (1) **Hierarchy**: v1 has one hierarchy per controller (cpu/, memory/, blkio/ are separate trees); v2 has ONE unified hierarchy for all controllers. A cgroup has the same path for all controllers. (2) **No internal process rule**: v2 requires processes only in leaf cgroups; v1 allows processes at any level, causing ambiguous resource distribution. (3) **Controller interface**: v2 uses cleaner interfaces — `cpu.max` replaces separate `cfs_quota_us` and `cfs_period_us`; `cpu.weight` (1-10000, default 100) replaces `cpu.shares` (default 1024). (4) **memory.high**: v2 adds a throttle boundary (slow allocations instead of OOM) — no v1 equivalent. (5) **PSI (Pressure Stall Information)**: v2 integrates PSI per-cgroup for monitoring resource contention. (6) **Delegation**: v2 has a proper model for safely delegating cgroup management to unprivileged users (critical for rootless containers). (7) **Thread mode**: v2 supports threaded cgroups (separate thread-level cpu scheduling within a process). (8) **IO controller**: v2's `io.max`/`io.weight` replaces v1's `blkio.*` with a simpler interface and works with writeback I/O (v1's blkio didn't account for buffered writes).

**Q2: Explain memory.high vs memory.max in cgroups v2.**
**A:** `memory.max` is the hard limit — when usage reaches this, the OOM killer is triggered (same as v1's `memory.limit_in_bytes`). `memory.high` is a throttle boundary (new in v2, no v1 equivalent) — when usage exceeds this, the kernel applies reclaim pressure: memory allocations are slowed down (the process is put into direct reclaim, scanning its own pages), but no OOM kill. This creates back-pressure: the process slows down instead of crashing. Best practice: set `memory.high` to your expected usage (e.g., 450MB) and `memory.max` slightly higher (e.g., 500MB) as a safety net. This way: (1) Normal operation: usage < memory.high, no impact. (2) Under pressure: usage between high and max, process slows but continues. (3) Emergency: usage hits max, OOM kill prevents system-wide impact. Additionally, `memory.low` (best-effort protection) and `memory.min` (hard protection) protect cgroups from reclaim — the kernel avoids reclaiming from cgroups below their `memory.low` unless there's no other option.

---

## Summary

- Cgroups v2: unified hierarchy — all controllers share one tree at /sys/fs/cgroup/
- No internal process rule: processes only in leaf cgroups → clear resource distribution
- cpu.weight (1-10000) replaces cpu.shares; cpu.max ("quota period") replaces cfs_quota/period
- memory.high (throttle) + memory.max (OOM) — two-level memory management
- memory.low/min protect working sets from reclaim
- PSI: per-cgroup CPU/memory/IO pressure metrics for auto-scaling decisions
- Delegation: safe model for rootless containers (unprivileged cgroup management)
- IO controller: io.max (BPS/IOPS limits), io.weight (proportional), supports writeback

---

[Previous: Cgroups v1 Architecture ←](Chapter_11_Cgroups_v1.md) | [Next: CPU and Memory Controllers Deep Dive →](Chapter_13_CPU_Memory_Controllers.md)
