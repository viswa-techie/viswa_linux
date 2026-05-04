# Chapter 24: Process Accounting

## Learning Goals
- Understand how Linux tracks CPU and resource usage per process
- Learn kernel accounting data structures and update mechanisms
- Master user-space tools for viewing process statistics
- Know cgroup resource accounting

---

## 24.1 What is Process Accounting?

```
Process accounting = tracking resource usage per process:

  ┌───────────────────────────────────────────────┐
  │  Resource                │ Tracked where       │
  ├──────────────────────────┼─────────────────────┤
  │  CPU time (user/system)  │ task_struct          │
  │  Voluntary ctx switches  │ task_struct          │
  │  Involuntary ctx switches│ task_struct          │
  │  Memory (RSS, virtual)   │ mm_struct            │
  │  I/O (read/write bytes)  │ task_io_accounting   │
  │  Page faults (minor/major)│ task_struct         │
  │  Signals received/sent   │ signal_struct        │
  │  Wall clock time         │ start_time in task   │
  │  Scheduling statistics   │ sched_entity/sched_info│
  └──────────────────────────┴─────────────────────┘
```

---

## 24.2 CPU Time Accounting

### Fields in task_struct

```c
/* include/linux/sched.h */
struct task_struct {
    /* CPU time counters (nanoseconds) */
    u64                     utime;      /* User mode CPU time */
    u64                     stime;      /* Kernel mode CPU time */
    u64                     gtime;      /* Guest (VM) CPU time */

    /* Children's accumulated time (after wait()) */
    u64                     cutime;     /* Children's user time */
    u64                     cstime;     /* Children's system time */

    /* Start time */
    u64                     start_time;       /* Monotonic boot time */
    u64                     start_boottime;   /* Includes suspend time */

    /* Scheduling statistics */
    struct sched_info       sched_info;
    /* in sched_entity: */
    /*   sum_exec_runtime — total CPU time on this CPU */
    /*   vruntime — virtual runtime */
};

/* sched_info — scheduling statistics */
struct sched_info {
    unsigned long           pcount;     /* # times scheduled */
    unsigned long long      run_delay;  /* Total wait time on runqueue */
    unsigned long long      last_arrival;
    unsigned long long      last_queued;
};
```

### How CPU Time is Updated

```
Timer interrupt (scheduler_tick):
  account_process_tick(current, user_mode(regs))
    → If user mode:  current->utime += tick_period
    → If kernel mode: current->stime += tick_period
    → If guest mode:  current->gtime += tick_period

With VIRT_CPU_ACCOUNTING (fine-grained):
  Exact accounting using timestamp at kernel entry/exit
  Not dependent on timer tick sampling
  More accurate but slightly more overhead

  context_tracking_enter(CONTEXT_USER)  → record kernel→user timestamp
  context_tracking_exit(CONTEXT_USER)   → record user→kernel timestamp
  vtime_account_user(current)           → add exact user time
```

---

## 24.3 Context Switch Accounting

```c
/* Per-task context switch counters */
struct task_struct {
    unsigned long           nvcsw;   /* Voluntary context switches */
    unsigned long           nivcsw;  /* Involuntary context switches */
};

/* Updated in schedule(): */
if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
    /* Task voluntarily slept (called schedule from wait) */
    prev->nvcsw++;
} else {
    /* Task was preempted (timer or higher priority) */
    prev->nivcsw++;
}
```

```
Voluntary vs Involuntary context switches:

  Voluntary (nvcsw):
    Task calls: sleep(), wait(), read() on empty pipe, mutex_lock()
    → Task CHOSE to give up CPU
    High nvcsw = I/O-bound task (frequently waits)

  Involuntary (nivcsw):
    Timer tick expired timeslice, higher priority task arrived
    → Task was FORCED off CPU
    High nivcsw = CPU-bound task (runs until preempted)

  Viewing:
    /proc/<pid>/status  → voluntary_ctxt_switches / nonvoluntary_ctxt_switches
    /proc/<pid>/sched   → nr_switches, nr_voluntary_switches, nr_involuntary_switches
```

---

## 24.4 I/O Accounting

```c
/* include/linux/task_io_accounting.h */
struct task_io_accounting {
    u64 rchar;           /* Bytes read (including page cache hits) */
    u64 wchar;           /* Bytes written */
    u64 syscr;           /* Read syscall count */
    u64 syscw;           /* Write syscall count */
    u64 read_bytes;      /* Bytes actually read from storage */
    u64 write_bytes;     /* Bytes actually written to storage */
    u64 cancelled_write_bytes; /* Writes cancelled (truncate before flush) */
};
```

```bash
# View per-process I/O
cat /proc/<pid>/io
rchar: 1234567        # Total bytes read (inc. cache)
wchar: 987654         # Total bytes written
syscr: 1500           # Read syscalls
syscw: 800            # Write syscalls
read_bytes: 512000    # Actual disk reads
write_bytes: 256000   # Actual disk writes
cancelled_write_bytes: 4096
```

---

## 24.5 /proc Filesystem Accounting Interface

### /proc/\<pid\>/stat (one-line summary)

```bash
cat /proc/1234/stat
# Decoding key fields:
# Field  1: pid (1234)
# Field  2: comm (program_name)
# Field  3: state (R/S/D/Z/T)
# Field  4: ppid
# Field  5: pgid
# Field  6: sid
# Field 10: minflt (minor page faults)
# Field 12: majflt (major page faults — disk I/O)
# Field 14: utime (clock ticks in user mode)
# Field 15: stime (clock ticks in kernel mode)
# Field 16: cutime (children's user time)
# Field 17: cstime (children's system time)
# Field 18: priority (kernel priority)
# Field 19: nice value
# Field 20: num_threads
# Field 22: starttime (since boot, in ticks)
# Field 23: vsize (virtual memory bytes)
# Field 24: rss (pages in physical memory)
```

### /proc/\<pid\>/status (human-readable)

```bash
cat /proc/1234/status
Name:   my_program
State:  S (sleeping)
Tgid:   1234
Pid:    1234
PPid:   500
Threads:        4
VmPeak:     52000 kB
VmSize:     48000 kB
VmRSS:      12000 kB
VmData:      8000 kB
VmStk:        200 kB
Cpus_allowed:   ff
voluntary_ctxt_switches:        1500
nonvoluntary_ctxt_switches:     200
```

### /proc/\<pid\>/sched (scheduler-specific)

```bash
cat /proc/1234/sched
se.exec_start                     :    123456789.012345
se.vruntime                       :     98765432.100000
se.sum_exec_runtime               :      5000000.000000
se.nr_migrations                  :                  42
nr_switches                       :                1700
nr_voluntary_switches             :                1500
nr_involuntary_switches           :                 200
se.load.weight                    :                1024
se.avg.load_avg                   :                 500
se.avg.util_avg                   :                 300
policy                            :                   0  # SCHED_NORMAL
prio                              :                 120  # nice 0
```

---

## 24.6 User-Space Accounting Tools

### getrusage()

```c
#include <sys/resource.h>

struct rusage usage;
getrusage(RUSAGE_SELF, &usage);

printf("User CPU:   %ld.%06ld sec\n", usage.ru_utime.tv_sec, usage.ru_utime.tv_usec);
printf("System CPU: %ld.%06ld sec\n", usage.ru_stime.tv_sec, usage.ru_stime.tv_usec);
printf("Max RSS:    %ld KB\n", usage.ru_maxrss);
printf("Minor faults: %ld\n", usage.ru_minflt);
printf("Major faults: %ld\n", usage.ru_majflt);
printf("Vol switches: %ld\n", usage.ru_nvcsw);
printf("Invol switches: %ld\n", usage.ru_nivcsw);
```

### time command

```bash
time make -j4
# real    0m45.123s    ← wall clock time
# user    2m30.456s    ← total user CPU (all threads)
# sys     0m12.789s    ← total kernel CPU

# Note: user > real means multi-threaded parallelism
# real > user + sys means mostly I/O waiting
```

### pidstat (sysstat package)

```bash
# Per-process CPU breakdown every 1 second
pidstat 1
#  PID  %usr  %system  %guest  %CPU  CPU  Command
# 1234  45.0     5.0     0.0  50.0    2  my_app
# 5678  10.0     2.0     0.0  12.0    0  other_app

# Context switches per process
pidstat -w 1
#  PID  cswch/s  nvcswch/s  Command
# 1234    150       20      my_app

# I/O per process
pidstat -d 1
#  PID  kB_rd/s  kB_wr/s  kB_ccwr/s  Command
# 1234   500.0    100.0      0.0     my_app
```

---

## 24.7 cgroup Resource Accounting

```bash
# cgroup v2 CPU accounting
cat /sys/fs/cgroup/mygroup/cpu.stat
usage_usec 1500000           # Total CPU time (microseconds)
user_usec 1200000            # User mode
system_usec 300000           # Kernel mode
nr_periods 150               # Enforcement periods
nr_throttled 5               # Times group was throttled
throttled_usec 25000         # Total throttled time

# Memory accounting
cat /sys/fs/cgroup/mygroup/memory.stat
anon 8388608                 # Anonymous memory
file 4194304                 # File-backed memory
pgfault 1500                 # Page faults
pgmajfault 10                # Major page faults

# I/O accounting per cgroup
cat /sys/fs/cgroup/mygroup/io.stat
8:0 rbytes=1048576 wbytes=524288 rios=100 wios=50
```

---

## 24.8 BSD Process Accounting (acct)

Traditional Unix accounting — writes record when process exits:

```c
/* Enable BSD process accounting */
#include <unistd.h>
acct("/var/log/pacct");  /* Write accounting records to file */

/* Each exit writes struct acct_v3 to the file:
 *   ac_comm    — command name
 *   ac_utime   — user CPU time
 *   ac_stime   — system CPU time
 *   ac_etime   — elapsed time
 *   ac_mem     — average memory usage
 *   ac_io      — I/O count
 *   ac_uid     — user ID
 *   ac_exitcode — exit status
 */
```

```bash
# Enable
echo 1 > /proc/sys/kernel/acct   # or use accton command
accton /var/log/pacct

# View records
sa -c /var/log/pacct              # Summary
lastcomm -f /var/log/pacct        # Recent commands
```

---

## 24.9 Schedstat — Scheduler Statistics

```bash
# System-wide scheduler stats
cat /proc/schedstat
# version 15
# timestamp 123456789
# cpu0 <yld_count> <array_expires> <schedule_count> <sched_goidle> ...
# domain0 <cpumask> <lb_count> <lb_balanced> <lb_failed> ...

# Per-task scheduling delay
cat /proc/<pid>/schedstat
# run_time  wait_time  nr_timeslices
# 5000000   1200000    1700
```

---

## Interview Questions

**Q1: How does the kernel track whether CPU time is spent in user mode or kernel mode?**
A: On each timer tick (`scheduler_tick()`), the kernel checks the saved registers from the interrupted context to determine if it was in user mode or kernel mode, then increments `utime` or `stime` accordingly. With fine-grained accounting (`CONFIG_VIRT_CPU_ACCOUNTING`), the kernel records timestamps at every kernel entry/exit point for exact measurement rather than statistical sampling.

**Q2: What's the difference between minor and major page faults?**
A: **Minor fault**: the page exists in memory but the page table entry isn't set up (e.g., first access after mmap, COW). Resolved without disk I/O — ~1-10µs. **Major fault**: the page must be read from disk (swapped out or memory-mapped file not cached). Involves disk I/O — ~1-10ms. High major faults indicate memory pressure and thrashing.

**Q3: How can cgroup CPU accounting help diagnose scheduling issues on Android?**
A: Android puts each app in a cgroup. `cpu.stat` shows: 1) Total CPU time per app — identify CPU hogs. 2) `nr_throttled` — app hitting CPU quota limits (causes jank). 3) `throttled_usec` — how much latency throttling adds. Combined with `cpuacct.usage_percpu`, this shows if an app is poorly balanced across CPUs. For automotive Android, deadline tasks in RT cgroups must not exceed bandwidth allocation.

---

*Next: [Chapter 25 — Scheduler Performance Optimization](Chapter_25_Scheduler_Performance.md)*
