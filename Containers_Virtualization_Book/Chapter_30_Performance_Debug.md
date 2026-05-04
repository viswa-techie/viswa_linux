# Chapter 30: Container and VM Performance Tuning

## Learning Goals
- Master container performance profiling with perf, bpftrace
- Understand VM performance bottlenecks and diagnostics
- Learn cgroup-aware monitoring and resource accounting
- Know NUMA, hugepages, and CPU pinning optimization

---

## 1. Container Performance Profiling

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Container-aware profiling tools:                        │
  │                                                           │
  │  perf (host can profile all containers):                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Profile specific container by cgroup:  │            │
  │  │ perf stat -G docker/abc123 -a sleep 10   │            │
  │  │                                          │            │
  │  │ # Record container workload:             │            │
  │  │ perf record -g -p $(docker inspect \     │            │
  │  │   --format '{{.State.Pid}}' myapp) \     │            │
  │  │   -- sleep 30                            │            │
  │  │                                          │            │
  │  │ # Flamegraph from container:             │            │
  │  │ perf script | stackcollapse-perf.pl | \  │            │
  │  │   flamegraph.pl > container.svg          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  bpftrace (eBPF tracing):                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Trace syscalls from specific container │            │
  │  │ # (by cgroup ID):                        │            │
  │  │ bpftrace -e '                            │            │
  │  │   tracepoint:raw_syscalls:sys_enter      │            │
  │  │   /cgroup == cgroupid("/sys/fs/cgroup/\  │            │
  │  │     system.slice/docker-abc.scope")/     │            │
  │  │   { @[comm, args->id] = count(); }'     │            │
  │  │                                          │            │
  │  │ # Container DNS latency:                 │            │
  │  │ bpftrace -e '                            │            │
  │  │   kprobe:udp_sendmsg                     │            │
  │  │   /cgroup == cgroupid("/sys/fs/cgroup/...")/│         │
  │  │   { @start[tid] = nsecs; }               │            │
  │  │   kretprobe:udp_recvmsg                  │            │
  │  │   /@start[tid]/                          │            │
  │  │   { @latency = hist(nsecs - @start[tid]);│            │
  │  │     delete(@start[tid]); }'              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Cgroup resource monitoring:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # CPU usage per container:               │            │
  │  │ cat /sys/fs/cgroup/system.slice/\        │            │
  │  │   docker-abc.scope/cpu.stat              │            │
  │  │ → usage_usec, throttled_usec,            │            │
  │  │   nr_throttled                           │            │
  │  │                                          │            │
  │  │ # Memory pressure:                       │            │
  │  │ cat .../memory.pressure                   │            │
  │  │ → some avg10=5.20 avg60=3.10 avg300=1.50 │            │
  │  │   full avg10=1.02 avg60=0.50 avg300=0.20│            │
  │  │                                          │            │
  │  │ # I/O stats:                             │            │
  │  │ cat .../io.stat                           │            │
  │  │ → 8:0 rbytes=1234 wbytes=5678 rios=100  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VM Performance Diagnostics

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KVM performance counters:                               │
  │                                                           │
  │  perf kvm stat live:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Real-time VM exit statistics:          │            │
  │  │ perf kvm stat live -p $(pgrep qemu)      │            │
  │  │                                          │            │
  │  │ Exit reason          Count    Time(%)    │            │
  │  │ ──────────────────────────────────────── │            │
  │  │ HLT                  45230    67.2%      │            │
  │  │ EPT_VIOLATION         1250     8.3%      │            │
  │  │ IO_INSTRUCTION        3420    12.1%      │            │
  │  │ EXTERNAL_INTERRUPT    2100     5.8%      │            │
  │  │ MSR_WRITE             1890     4.2%      │            │
  │  │ PAUSE_INSTRUCTION      340     0.8%      │            │
  │  │ CPUID                  120     0.3%      │            │
  │  │                                          │            │
  │  │ What each means:                         │            │
  │  │ HLT: guest is idle (good)                │            │
  │  │ EPT_VIOLATION: page fault (MMIO or       │            │
  │  │   memory mapping changes)                │            │
  │  │ IO_INSTRUCTION: port I/O (in/out)        │            │
  │  │ EXTERNAL_INTERRUPT: host interrupt       │            │
  │  │ MSR_WRITE: model-specific register       │            │
  │  │ PAUSE_INSTRUCTION: spinlock hint         │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  kvm_stat (debugfs counters):                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # /sys/kernel/debug/kvm/                 │            │
  │  │ cat exits           → total VM exits     │            │
  │  │ cat halt_exits      → HLT instruction    │            │
  │  │ cat io_exits        → I/O port access    │            │
  │  │ cat mmio_exits      → MMIO access        │            │
  │  │ cat irq_injections  → interrupts sent    │            │
  │  │ cat signal_exits    → signal to qemu     │            │
  │  │                                          │            │
  │  │ Red flags:                               │            │
  │  │ - High io_exits: use virtio (not IDE)    │            │
  │  │ - High mmio_exits: reduce emulated devs  │            │
  │  │ - High signal_exits: QEMU thread issues  │            │
  │  │ - Excessive halt_exits: vCPU overprovisioned│         │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. CPU and Memory Optimization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CPU pinning (containers):                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Docker: pin to specific CPUs           │            │
  │  │ docker run --cpuset-cpus="0-3" app       │            │
  │  │                                          │            │
  │  │ # cgroup v2:                             │            │
  │  │ echo "0-3" > cpuset.cpus                 │            │
  │  │ echo "0" > cpuset.mems  # NUMA node     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  CPU pinning (VMs):                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # QEMU: pin vCPU threads to host CPUs   │            │
  │  │ taskset -cp 4 $(pgrep -f "thread_id=0") │            │
  │  │ taskset -cp 5 $(pgrep -f "thread_id=1") │            │
  │  │                                          │            │
  │  │ # libvirt:                               │            │
  │  │ <vcpu placement='static'>4</vcpu>        │            │
  │  │ <cputune>                                │            │
  │  │   <vcpupin vcpu='0' cpuset='4'/>         │            │
  │  │   <vcpupin vcpu='1' cpuset='5'/>         │            │
  │  │   <vcpupin vcpu='2' cpuset='6'/>         │            │
  │  │   <vcpupin vcpu='3' cpuset='7'/>         │            │
  │  │ </cputune>                               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Hugepages (reduces TLB misses):                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Allocate 2MB hugepages:                │            │
  │  │ echo 4096 > /proc/sys/vm/\               │            │
  │  │   nr_hugepages  # = 8GB                  │            │
  │  │                                          │            │
  │  │ # QEMU with hugepages:                   │            │
  │  │ qemu-system-x86_64 \                     │            │
  │  │   -m 8G \                                │            │
  │  │   -mem-path /dev/hugepages \             │            │
  │  │   -mem-prealloc                          │            │
  │  │                                          │            │
  │  │ # Container with hugepages:              │            │
  │  │ docker run \                             │            │
  │  │   --shm-size=2g \                        │            │
  │  │   -v /dev/hugepages:/dev/hugepages \     │            │
  │  │   --ulimit memlock=-1:-1 app             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  NUMA topology awareness:                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ NUMA node 0: CPUs 0-7, memory bank 0    │            │
  │  │ NUMA node 1: CPUs 8-15, memory bank 1   │            │
  │  │                                          │            │
  │  │ Best practice: pin vCPU/container to ONE │            │
  │  │ NUMA node (avoid cross-node memory)      │            │
  │  │                                          │            │
  │  │ numactl --cpunodebind=0 --membind=0 \    │            │
  │  │   qemu-system-x86_64 ...                 │            │
  │  │                                          │            │
  │  │ Cross-NUMA penalty: 30-50% latency hit   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. I/O Performance Tuning

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  I/O scheduler and VM/container interaction:             │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Host I/O scheduler:                      │            │
  │  │   SSD: none/noop (no scheduling needed)  │            │
  │  │   HDD: mq-deadline or bfq               │            │
  │  │                                          │            │
  │  │ Guest I/O scheduler:                     │            │
  │  │   Always none/noop (host handles it)     │            │
  │  │                                          │            │
  │  │ Double scheduling = wasted CPU           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Performance checklist:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Containers:                              │            │
  │  │ □ Use overlay2 storage driver            │            │
  │  │ □ Minimize layers in Dockerfile          │            │
  │  │ □ Use tmpfs for temporary files          │            │
  │  │ □ Set appropriate memory limits          │            │
  │  │   (avoid OOM, avoid overprovision)       │            │
  │  │ □ CPU quota: avoid throttling            │            │
  │  │   (nr_throttled in cpu.stat)             │            │
  │  │ □ Use PSI metrics for pressure detection │            │
  │  │                                          │            │
  │  │ VMs:                                     │            │
  │  │ □ Use virtio devices (not ide/e1000)     │            │
  │  │ □ Enable vhost-net for networking        │            │
  │  │ □ Use hugepages for memory               │            │
  │  │ □ Pin vCPUs to physical CPUs             │            │
  │  │ □ Match NUMA topology                    │            │
  │  │ □ Use io_uring or AIO for disk I/O       │            │
  │  │ □ Minimize emulated devices              │            │
  │  │ □ Enable multiqueue virtio               │            │
  │  │ □ Guest scheduler: noop/none             │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How do you diagnose excessive VM exits and what are common optimizations?**
**A:** Use `perf kvm stat live -p $(pgrep qemu)` to see real-time VM exit counts by reason. Common issues: (1) **High IO_INSTRUCTION exits**: guest is using PIO (port I/O) for device access — switch to virtio devices (virtio-net instead of e1000, virtio-blk instead of IDE). Virtio uses MMIO and batched notifications. (2) **High MMIO exits (EPT_VIOLATION)**: too many emulated devices causing MMIO traps to QEMU. Minimize emulated devices; use passthrough for performance-critical devices. (3) **High EXTERNAL_INTERRUPT exits**: enable APICv/posted interrupts so external interrupts don't cause exits. Use `process-posted-interrupts` KVM parameter. (4) **High PAUSE_INSTRUCTION exits**: vCPU spinning on spinlock. Enable PLE (Pause Loop Exiting) with appropriate thresholds, or use pvspinlocks (paravirtual spinlocks where guest yields to host instead of spinning). (5) **High MSR_WRITE exits**: guest writing model-specific registers. Enable MSR bitmap in VMCS to pass through safe MSRs without exits. (6) **HLT exits when not idle**: guest using HLT for timer waits. Use `halt_poll_ns` to have KVM poll briefly before halting the vCPU (reduces wakeup time). General: reduce emulated device count, use virtio + vhost, enable hardware acceleration features (APICv, EPT, VPID, PLE).

**Q2: What container performance issues are invisible to standard monitoring tools and how do you find them using cgroup metrics and eBPF?**
**A:** Standard tools (top, htop) show host-wide metrics but miss container-specific issues: (1) **CPU throttling**: a container hitting its CFS quota limit gets throttled — its processes are runnable but not running. Invisible in CPU utilization (shows <100%). Detect via `cpu.stat`: if `nr_throttled` is high, the container needs more CPU quota or has a burst problem. (2) **Memory pressure without OOM**: a container under memory pressure reclaims aggressively (swapping, page cache eviction) — slows down but doesn't OOM kill. Detect via PSI metrics in `memory.pressure`: `some avg10` > 0 means some tasks are stalled waiting for memory. (3) **Noisy neighbor I/O**: containers sharing a host disk compete for I/O bandwidth. Detect via `io.stat` per container and `io.pressure` PSI metrics. (4) **Network namespace issues**: DNS resolution failures, iptables rules accumulated from many pods. Use eBPF: `bpftrace` to trace DNS latency per cgroup ID, or `kubectl debug` with nsenter. (5) **Kernel lock contention**: containers sharing kernel resources (mount namespace operations, cgroup operations) can contend on kernel locks. Use BPF tools: `runqlat` filtered by cgroup to see scheduling latency, `offcputime` to see where tasks sleep. The key insight: container performance requires cgroup-level metrics (cpu.stat, memory.stat, io.stat, PSI) and eBPF tracing with cgroup filters — standard host-level tools aggregate across all containers and miss per-container issues.

---

## Summary

- Container profiling: perf/bpftrace with cgroup filters; watch cpu.stat, memory.pressure, io.stat
- VM profiling: `perf kvm stat live` for VM exit analysis; minimize exits for performance
- CPU pinning: cpuset for containers, taskset/libvirt for VMs; match NUMA topology
- Hugepages: 2MB/1GB pages reduce TLB misses; critical for VM memory performance
- I/O tuning: virtio + vhost, guest scheduler=none, io_uring on host, multiqueue
- Key metrics: nr_throttled (CPU), PSI (memory/IO/CPU pressure), VM exit counts

---

[Previous: Kubernetes & CRI ←](Chapter_29_K8s_CRI.md) | [Next: Security Deep Dive →](Chapter_31_Security_Deep_Dive.md)
