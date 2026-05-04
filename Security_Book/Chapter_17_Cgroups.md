# Chapter 17: Control Groups (cgroups)

## Learning Goals
- Understand cgroup v1 and v2 architecture
- Know how cgroups enforce resource limits for security
- Understand cgroup controllers and their security implications
- Know how containers use cgroups for resource isolation

---

## 17.1 Cgroup Architecture

```
Cgroups: Organize processes into hierarchical groups.
Apply resource limits, accounting, and isolation per group.

Security role: Prevent denial-of-service via resource exhaustion.

  ┌──────────────────────────────────────────────────────┐
  │                  Cgroup Hierarchy                     │
  │                                                       │
  │  / (root cgroup)                                      │
  │  ├── system.slice/       (system services)            │
  │  │   ├── sshd.service    CPU: 10%, MEM: 256MB         │
  │  │   ├── nginx.service   CPU: 30%, MEM: 1GB           │
  │  │   └── docker.service  CPU: 50%, MEM: 8GB           │
  │  ├── user.slice/         (user sessions)              │
  │  │   ├── user-1000.slice CPU: 20%, MEM: 4GB           │
  │  │   └── user-1001.slice CPU: 10%, MEM: 2GB           │
  │  └── container/          (container workloads)        │
  │      ├── container_1/    CPU: 2 cores, MEM: 512MB     │
  │      ├── container_2/    CPU: 1 core, MEM: 256MB      │
  │      └── container_3/    CPU: 4 cores, MEM: 2GB       │
  └──────────────────────────────────────────────────────┘

Cgroup v1 vs v2:
  ┌──────────────────┬──────────────┬──────────────┐
  │ Feature          │ v1           │ v2           │
  ├──────────────────┼──────────────┼──────────────┤
  │ Hierarchy        │ Multiple     │ Single       │
  │ Controller mount │ Per-ctrlr    │ Unified      │
  │ Delegation       │ Difficult    │ Designed in  │
  │ Mount point      │ /sys/fs/cgroup/ctrl │ /sys/fs/cgroup │
  │ Thread control   │ No           │ Yes          │
  │ Pressure info    │ No           │ PSI support  │
  └──────────────────┴──────────────┴──────────────┘

Cgroup v2 is the modern standard. v1 is legacy.
```

---

## 17.2 Resource Controllers

```
Controllers limit specific resource types:

  ┌──────────────┬──────────────────────────────────────┐
  │ Controller   │ Controls                              │
  ├──────────────┼──────────────────────────────────────┤
  │ cpu          │ CPU time scheduling weight/max        │
  │ cpuset       │ Pin to specific CPU cores             │
  │ memory       │ Memory usage limits + swap            │
  │ io           │ Block I/O bandwidth and IOPS          │
  │ pids         │ Maximum number of processes           │
  │ hugetlb      │ Huge page allocation limits           │
  │ rdma         │ RDMA resource limits                  │
  │ misc         │ Miscellaneous scalar resources        │
  └──────────────┴──────────────────────────────────────┘

Security-critical controllers:

  Memory controller:
    memory.max = 536870912     # 512MB hard limit
    memory.high = 268435456    # 256MB throttle point
    memory.swap.max = 0        # No swap (prevent swapping secrets)
    OOM kill: kernel kills process when exceeding memory.max

  PID controller:
    pids.max = 100             # Max 100 processes
    Prevents fork bomb: :(){ :|:& };:
    Container cannot exhaust host PID space

  CPU controller:
    cpu.max = "200000 100000"  # 200ms per 100ms period = 2 cores
    cpu.weight = 100           # Scheduling weight (1-10000)
    Prevents CPU starvation of other containers

  I/O controller:
    io.max = "8:0 rbps=1048576 wbps=524288"  # Limit disk I/O
    Prevents I/O starvation
```

---

## 17.3 Cgroup v2 Interface

```bash
# Cgroup v2 filesystem interface:
/sys/fs/cgroup/                     # Root cgroup
/sys/fs/cgroup/cgroup.controllers   # Available controllers
/sys/fs/cgroup/cgroup.subtree_control  # Enabled for children

# Create a cgroup:
mkdir /sys/fs/cgroup/myapp

# Enable controllers for children:
echo "+memory +pids +cpu" > /sys/fs/cgroup/cgroup.subtree_control

# Set limits:
echo 536870912 > /sys/fs/cgroup/myapp/memory.max      # 512MB
echo 100 > /sys/fs/cgroup/myapp/pids.max               # 100 pids
echo "200000 100000" > /sys/fs/cgroup/myapp/cpu.max    # 2 cores

# Add process to cgroup:
echo $PID > /sys/fs/cgroup/myapp/cgroup.procs

# Monitor:
cat /sys/fs/cgroup/myapp/memory.current    # Current usage
cat /sys/fs/cgroup/myapp/pids.current      # Current PIDs
cat /sys/fs/cgroup/myapp/cpu.stat          # CPU statistics
cat /sys/fs/cgroup/myapp/memory.events     # OOM events count
```

---

## 17.4 Cgroup Security Use Cases

```
1. Fork bomb protection:
   pids.max = 100
   Container cannot create more than 100 processes.
   Without this: fork bomb exhausts host PID table → all services fail.

2. Memory exhaustion protection:
   memory.max = 512M
   Container OOM-killed before affecting host.
   memory.swap.max = 0 → secrets never written to swap.

3. CPU fairness:
   cpu.weight = 100 (default)
   Malicious container cannot starve others of CPU.
   cpu.max limits burst usage.

4. I/O throttling:
   io.max rate limiting prevents container from
   saturating disk bandwidth.

5. Device access control (v1 devices controller):
   Restrict which /dev/ devices container can access.
   Whitelist only needed devices.

  ┌────────────────────────────────────────────┐
  │  Without cgroups:                          │
  │  Process A: malloc(ALL_RAM)  → OOM host    │
  │  Process B: fork() infinite  → PID exhaust │
  │  Process C: while(1) {}      → CPU starve  │
  │                                            │
  │  With cgroups:                             │
  │  Process A: OOM killed at 512MB limit      │
  │  Process B: fork fails at 100 pid limit    │
  │  Process C: throttled to 2 CPU cores       │
  │  Host remains stable.                      │
  └────────────────────────────────────────────┘
```

---

## 17.5 Cgroup Delegation and Containers

```
Cgroup delegation: Allow unprivileged users to manage sub-cgroups.

  Container runtime creates cgroup:
    /sys/fs/cgroup/containers/container_1/

  Delegates to container by giving ownership:
    chown -R container_uid /sys/fs/cgroup/containers/container_1/

  Container can then create sub-cgroups within its allocation
  but cannot exceed parent's limits.

  ┌─────────────────────────────────────────────┐
  │  /sys/fs/cgroup/                             │
  │  └── containers/            (root-owned)     │
  │      ├── container_1/       (delegated)      │
  │      │   memory.max = 2G                     │
  │      │   ├── app/           (container mgd)  │
  │      │   │   memory.max = 1G  (≤ parent 2G)  │
  │      │   └── worker/                         │
  │      │       memory.max = 512M               │
  │      └── container_2/       (delegated)      │
  │          memory.max = 4G                     │
  └─────────────────────────────────────────────┘

  Security: container can subdivide its resources
  but cannot escape its allocated limits.

Docker resource limits:
  docker run --memory=512m --cpus=2 --pids-limit=100 myimage

Kubernetes resource limits:
  resources:
    limits:
      memory: "512Mi"
      cpu: "2"
    requests:
      memory: "256Mi"
      cpu: "1"
```

---

## 17.6 Pressure Stall Information (PSI)

```
PSI: Cgroup v2 feature for detecting resource pressure.

  Files:
    /sys/fs/cgroup/myapp/cpu.pressure
    /sys/fs/cgroup/myapp/memory.pressure
    /sys/fs/cgroup/myapp/io.pressure

  Output:
    some avg10=0.00 avg60=0.00 avg300=0.00 total=0
    full avg10=0.00 avg60=0.00 avg300=0.00 total=0

    some: percentage of time at least one task was stalled
    full: percentage of time ALL tasks were stalled

  Security use: detect resource exhaustion attacks.
  If a cgroup shows high pressure, it may be under attack
  or misconfigured. Monitoring tools can alert and throttle.

  PSI triggers (poll-based alerts):
    # Trigger when >50% stalled for >1 second in 2s window
    echo "some 500000 2000000" > /sys/fs/cgroup/myapp/memory.pressure
    # Application can poll() the fd for alerts
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| kernel/cgroup/cgroup.c | Core cgroup v2 implementation |
| kernel/cgroup/cgroup-v1.c | Cgroup v1 compatibility |
| mm/memcontrol.c | Memory controller |
| kernel/sched/core.c | CPU controller integration |
| kernel/cgroup/pids.c | PID controller |
| block/blk-cgroup.c | I/O controller |
| include/linux/cgroup.h | Cgroup data structures |

---

## Interview Questions

**Q1: How do cgroups prevent denial-of-service attacks?**
A: Cgroups limit resource consumption per process group. The memory controller kills processes exceeding their memory limit (OOM) before they can exhaust host RAM. The PID controller caps the number of processes — preventing fork bombs from exhausting the host PID table. The CPU controller limits CPU time so one process group can't starve others. The I/O controller rate-limits disk operations. Without cgroups, a single compromised container could exhaust any host resource, causing system-wide denial of service. With cgroups, each container is confined to its allocated resources and killed or throttled when exceeding limits.

**Q2: Explain cgroup delegation and its security model.**
A: Cgroup delegation allows a privileged process (container runtime) to give an unprivileged process (container) ownership of a sub-cgroup. The container can create further sub-cgroups within its allocation and set their limits, but can never exceed the parent cgroup's limits. For example, a container delegated 2GB memory can create sub-cgroups totaling at most 2GB. The security model relies on: (1) the parent sets hard limits before delegation, (2) cgroup v2's single hierarchy prevents manipulation, (3) the kernel enforces that child limits cannot exceed parent limits. This enables rootless containers to manage their own resource allocation safely.

**Q3: Why set memory.swap.max to 0 for security?**
A: Setting `memory.swap.max = 0` prevents the cgroup's memory from being swapped to disk. This is important for security because: (1) Sensitive data (encryption keys, passwords, tokens) in process memory could be written to swap, persisting on disk after the process exits. (2) Swap data can be recovered by other processes or forensic analysis. (3) Performance becomes more predictable — no swap means the OOM killer triggers cleanly when the memory limit is reached, rather than the process thrashing on swap. For containers handling sensitive data, disabling swap ensures secrets stay in RAM only.

---

## Summary

- Cgroups organize processes into groups with resource limits
- Controllers: memory (OOM), CPU (scheduling), PIDs (fork bomb), I/O (bandwidth)
- Cgroup v2: unified hierarchy, PSI, delegation support
- Memory.max: hard limit → OOM kill; memory.swap.max = 0: disable swap
- PID.max: prevents fork bombs; CPU.max: prevents CPU starvation
- Delegation: container manages sub-cgroups within allocated limits
- PSI: pressure stall information for detecting resource exhaustion
- Combined with namespaces + seccomp + MAC = complete container security

---

Next: [Chapter 18 — Kernel Address Space Protection](Chapter_18_Address_Space_Protection.md)
