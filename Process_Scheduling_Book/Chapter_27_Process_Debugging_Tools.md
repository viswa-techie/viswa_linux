# Chapter 27: Process Debugging Tools

## Learning Goals
- Master essential process monitoring and debugging tools
- Learn to diagnose scheduling issues using ps, top, htop, perf
- Understand /proc filesystem for process inspection
- Know strace, ltrace, and gdb for process debugging

---

## 27.1 ps — Process Status

```bash
# Essential ps commands for scheduling analysis

# All processes with scheduling info
ps -eo pid,ppid,pgid,sid,ni,pri,rtprio,cls,psr,stat,wchan,comm
#  PID  PPID  PGID   SID  NI PRI RTPRIO CLS PSR STAT WCHAN      COMMAND
#    1     0     1     1   0  19      - TS    0 Ss   -          systemd
#  100    1   100   100 -10  29      - TS    2 S<   futex_wait python
# 1000    1  1000  1000   -  41     99 FF    1 S    -          irq/18-i2c

# Columns explained:
#   NI:     nice value
#   PRI:    display priority (higher = more priority)
#   RTPRIO: RT priority (1-99); - for non-RT
#   CLS:    TS=NORMAL, FF=FIFO, RR=RR, B=BATCH, IDL=IDLE, DLN=DEADLINE
#   PSR:    Processor (CPU) last executed on
#   STAT:   R=running, S=sleeping, D=uninterruptible, Z=zombie, T=stopped
#           s=session leader, +=foreground, l=multi-threaded
#   WCHAN:  Kernel function where task is sleeping

# Process tree
ps -ejH   # or: ps axjf

# Threads of a specific process
ps -T -p <pid>
ps -eLf

# Sort by CPU usage
ps aux --sort=-%cpu | head -20

# Processes in D state (uninterruptible — often I/O blocked)
ps aux | awk '$8 ~ /D/'
```

---

## 27.2 top / htop — Real-Time Monitoring

```bash
# top essentials
top
# Key presses:
#   1     → show per-CPU utilization
#   H     → show threads
#   f     → add/remove fields (add: NI, PR, P, SWAP)
#   </>   → change sort column
#   c     → show full command line
#   z     → color mode

# top output interpretation:
# %Cpu(s): 25.0 us,  5.0 sy,  0.0 ni, 65.0 id,  3.0 wa,  0.0 hi,  2.0 si
#   us = user       — application CPU time
#   sy = system     — kernel CPU time
#   ni = nice       — nice'd user task time
#   id = idle       — truly idle
#   wa = iowait     — waiting for I/O (NOT a real CPU state!)
#   hi = hardware irq
#   si = software irq (softirq)

# htop advantages:
htop
# - Color-coded CPU bars per core
# - Tree view (F5)
# - Search/filter (F3/F4)
# - Send signals (F9)
# - Setup fields (F2)
# - Sort (F6)
```

---

## 27.3 /proc Filesystem Deep Dive

```bash
# Per-process information:
/proc/<pid>/
├── stat        # One-line status (machine-parseable)
├── status      # Human-readable status
├── sched       # Scheduler-specific statistics
├── schedstat   # run_time, wait_time, nr_timeslices
├── io          # I/O accounting
├── maps        # Memory mappings
├── smaps       # Detailed memory map info
├── stack       # Kernel stack trace (if CONFIG_STACKTRACE)
├── wchan       # Current wait channel (kernel function)
├── syscall     # Current/last syscall number + args
├── fd/         # Open file descriptors
├── fdinfo/     # File descriptor details
├── task/       # Per-thread subdirectories
│   ├── <tid>/sched    # Per-thread scheduling stats
│   └── <tid>/status   # Per-thread status
├── cgroup      # cgroup membership
├── cpuset      # Allowed CPUs and memory nodes
├── limits      # Resource limits
├── oom_score   # OOM kill priority
└── autogroup   # Autogroup info

# System-wide scheduling info:
/proc/sched_debug       # Full scheduler state dump
/proc/schedstat         # Per-CPU scheduling statistics
/proc/loadavg           # Load averages (1, 5, 15 min)
/proc/stat              # System CPU statistics
```

### Reading /proc/sched_debug

```bash
cat /proc/sched_debug
# Sched Debug Version: v0.11
# cpu#0:
#   .nr_running           : 2
#   .load                 : 2048
#   .nr_switches          : 5000000
#   .curr->pid            : 1234
#
# cfs_rq[0]:
#   .nr_running           : 2
#   .load                 : 2048
#   .min_vruntime         : 123456789.012345
#   .spread               : 2.345678
#
# rt_rq[0]:
#   .rt_nr_running        : 0

# Per-task detail:
# task   PID  tree-key   switches  prio  sum-exec    sum-sleep
# my_app 1234 98765432   1500      120   5000.000    10000.000
```

---

## 27.4 perf — Performance Analysis

```bash
# Count scheduling events
perf stat -e sched:sched_switch,sched:sched_wakeup,\
context-switches,cpu-migrations ./my_app

# Record scheduler events
perf sched record -- sleep 10
perf sched latency      # Show scheduling latency per task
perf sched map          # Visual CPU timeline
perf sched timehist     # Detailed wakeup/sleep timeline

# perf sched latency output:
#  Task                  |   Runtime ms | Switches | Avg delay ms |
# ─────────────────────────────────────────────────────────────────
# my_app:1234            |    5000.000  |     1500 |        0.050 |
# bash:500               |     100.000  |      800 |        0.020 |
# kworker/0:1234         |      50.000  |      200 |        0.010 |

# perf sched timehist output:
#  time      cpu  task                wait    sch delay   run time
# ──────────────────────────────────────────────────────────────────
# 1000.001  [001] my_app:1234         0.5ms     0.05ms      2.0ms
# 1002.001  [001] bash:500            1.0ms     0.02ms      0.5ms

# CPU frequency and scheduling
perf stat -e task-clock,cycles,instructions,cache-misses ./my_app

# Profile scheduler overhead
perf top -e sched:sched_switch    # Live view of switches
```

---

## 27.5 strace — System Call Tracing

```bash
# Trace all syscalls of a running process
strace -p <pid>

# Trace specific scheduling-related syscalls
strace -e trace=sched_setscheduler,sched_getscheduler,\
sched_setaffinity,sched_getaffinity,\
sched_yield,clone,fork,execve ./my_app

# Trace with timing (for identifying slow syscalls)
strace -T ./my_app
# write(1, "hello\n", 6)     = 6 <0.000015>
# nanosleep({1, 0}, NULL)    = 0 <1.000234>   ← 1 second sleep
# read(0, ..., 1024)         = 5 <2.345678>   ← waited 2.3s for input

# Count syscalls by frequency
strace -c ./my_app
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- -------
#  50.00    0.500000          10     50000           read
#  30.00    0.300000           5     60000           write
#  10.00    0.100000         100      1000           futex
#   5.00    0.050000          50      1000           nanosleep

# Follow forks (trace child processes too)
strace -f ./my_app
```

---

## 27.6 pidstat — Per-Process Statistics

```bash
# CPU usage per process (every 1 second, 10 samples)
pidstat 1 10
#  PID   %usr  %system  %guest  %wait  %CPU  CPU  Command
# 1234  45.00     5.00    0.00   2.00  50.00    2  my_app

# Context switches per process
pidstat -w 1
#  PID   cswch/s  nvcswch/s  Command
# 1234    150.00     20.00   my_app

# Thread-level stats
pidstat -t 1
#  TGID  TID    %usr  %system  %CPU  Command
# 1234    -    45.00     5.00  50.0  my_app
#    -  1235   20.00     2.00  22.0  |__worker1
#    -  1236   25.00     3.00  28.0  |__worker2

# Stack of blocked thread
pidstat -s 1

# Combined: CPU + switches + I/O
pidstat -u -w -d 1
```

---

## 27.7 Diagnosing Common Scheduling Issues

### High Involuntary Context Switches

```bash
# Symptom: application runs slower than expected
pidstat -w 1 -p <pid>
#   nvcswch/s: 500  ← Too high for CPU-bound work

# Diagnosis: task is being preempted too often
# Solutions:
chrt -b 0 -p <pid>           # SCHED_BATCH: no wakeup preemption
taskset -c 4 <pid>           # Pin to CPU: avoid migration
nice -n -5 <pid>             # Higher priority
```

### D-State (Uninterruptible) Process

```bash
# Symptom: process stuck, can't be killed
ps aux | awk '$8 ~ /D/ {print}'
#  PID ... D ...  stuck_process

# Get kernel stack to find WHERE it's stuck
cat /proc/<pid>/stack
# [<0>] nfs_wait_bit_killable+0x30/0x40
# [<0>] __wait_on_bit+0x60/0x90
# [<0>] out_of_line_wait_on_bit+0x80/0xa0

# Diagnosis: waiting for I/O (NFS, disk, etc.)
# "Killable" → SIGKILL will work when I/O returns
# If not killable → must fix the I/O subsystem
```

### Schedule Latency Issues

```bash
# Symptom: task wakes but doesn't run for too long
perf sched record -- sleep 10
perf sched latency --sort max
#  Task              |  Avg delay  |  Max delay  |
# ─────────────────────────────────────────────────
# audio_thread:200   |    0.050ms  |   15.000ms  | ← Problem!

# Diagnosis: something blocking audio thread from running
# Check: RT throttling, CPU saturation, lock contention
cat /proc/sys/kernel/sched_rt_runtime_us   # RT throttling?
mpstat -P ALL 1                             # CPU saturation?
perf lock record -- sleep 5                 # Lock contention?
```

---

## 27.8 Quick Reference

```
┌─────────────────────┬──────────────────────────────────────┐
│ Tool                 │ Best for                             │
├─────────────────────┼──────────────────────────────────────┤
│ ps                   │ Snapshot of process state            │
│ top/htop             │ Live CPU usage monitoring            │
│ perf sched           │ Scheduling latency/timeline analysis │
│ perf stat            │ Context switch counting              │
│ pidstat              │ Per-process CPU/IO/ctx switch stats  │
│ strace               │ Syscall tracing (what process does)  │
│ /proc/<pid>/sched    │ Detailed scheduler accounting        │
│ /proc/<pid>/stack    │ Where kernel thread is stuck         │
│ /proc/sched_debug    │ Full system scheduler dump           │
│ vmstat               │ System-wide ctx switch rate          │
│ mpstat               │ Per-CPU utilization                  │
│ chrt                 │ View/change scheduling policy        │
│ taskset              │ View/change CPU affinity             │
└─────────────────────┴──────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you diagnose why an Android app is dropping frames?**
A: 1) Check `systrace`/`perfetto` for scheduling delays in the UI thread. 2) Look for RT throttling (`/proc/sys/kernel/sched_rt_runtime_us`). 3) Check cgroup bandwidth throttling (`cpu.stat` nr_throttled). 4) Use `pidstat -w` for excessive context switches on the render thread. 5) Check for priority inversion (binder transactions to lower-priority services). 6) Verify CPU frequency is scaling up via `schedutil`. 7) Check for D-state blocks in the graphics pipeline.

**Q2: A process shows 0% CPU in top but high voluntary context switches in pidstat. What does this mean?**
A: The process is doing many short operations that block quickly — likely heavy I/O or IPC. It sends a request, sleeps, gets woken, processes briefly, sleeps again. "0% CPU" in top (sampled) means it runs for such short durations it's rarely caught running. The high `cswch/s` confirms frequent sleep/wake cycles. Check `strace -T` to find which syscalls are blocking, and `pidstat -d` for I/O rates.

**Q3: How do you use `/proc/PID/stack` to debug a hung process?**
A: `/proc/PID/stack` shows the kernel call stack of a sleeping/blocked task. It reveals exactly which kernel function the task is waiting in. Common patterns: `futex_wait` (waiting on lock/condvar), `pipe_read/write` (waiting on pipe), `wait_for_completion` (waiting for hardware), `io_schedule` (waiting for disk). This tells you what resource the task needs, which guides the fix (e.g., if stuck in NFS → check network; if stuck in futex → check for deadlock with other threads).

---

*Next: [Chapter 28 — Kernel Tracing for Scheduling](Chapter_28_Kernel_Tracing.md)*
