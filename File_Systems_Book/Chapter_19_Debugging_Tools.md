# Chapter 19: Debugging Tools

## Learning Goals
- Master filesystem debugging tools: strace, debugfs, fsck, blktrace
- Understand kernel tracing for filesystem issues (ftrace, perf)
- Know how to diagnose common FS problems: corruption, hangs, performance
- Use /proc and /sys interfaces for FS diagnostics

---

## 19.1 Debugging Tool Overview

```
Layer                  │ Tool                    │ Purpose
───────────────────────┼─────────────────────────┼──────────────────
Application / Syscall  │ strace, ltrace          │ Trace system calls
VFS / Path resolution  │ ftrace, perf            │ Kernel function tracing
File System internals  │ debugfs (ext4), xfs_db  │ FS-specific debugging
Block layer            │ blktrace, blkparse      │ Block I/O tracing
I/O statistics         │ iostat, iotop, pidstat  │ I/O performance
Corruption repair      │ fsck, e2fsck, xfs_repair│ Filesystem check/fix
Memory / Cache         │ vmstat, /proc/meminfo   │ Page cache stats
Kernel logging         │ dmesg, journalctl       │ Kernel messages
General profiling      │ perf, bpftrace          │ Performance analysis
```

---

## 19.2 strace — System Call Tracing

### Basic Usage

```bash
# Trace all syscalls of a command
$ strace ls /mnt/
execve("/usr/bin/ls", ["ls", "/mnt/"], ...) = 0
openat(AT_FDCWD, "/mnt/", O_RDONLY|O_NONBLOCK|O_DIRECTORY) = 3
fstat(3, {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
getdents64(3, [{d_ino=2, d_type=DT_DIR, d_name="."},
               {d_ino=2, d_type=DT_DIR, d_name=".."},
               {d_ino=12, d_type=DT_REG, d_name="file.txt"}], 32768) = 80
close(3)                                = 0

# Filter specific syscalls
$ strace -e trace=open,read,write,close cat /etc/hostname
open("/etc/hostname", O_RDONLY)         = 3
read(3, "myhost\n", 131072)            = 7
write(1, "myhost\n", 7)                = 7
close(3)                                = 0

# Trace a running process
$ strace -p 1234 -e trace=file
# Shows all file-related syscalls of PID 1234

# Time each syscall
$ strace -T ls /mnt/
openat(AT_FDCWD, "/mnt/", ...) = 3 <0.000012>
getdents64(3, ...) = 80 <0.000245>
# Time in angle brackets: <seconds>

# Count syscalls (summary)
$ strace -c ls /mnt/
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 42.16    0.000089           8        11           mmap
 18.01    0.000038           5         7           close
 12.32    0.000026           4         7           fstat
  8.53    0.000018          18         1           getdents64
  6.16    0.000013          13         1           openat
```

### Diagnosing Common Issues with strace

```bash
# Permission denied — which file?
$ strace -e trace=openat myapp 2>&1 | grep EACCES
openat(AT_FDCWD, "/etc/secret.conf", O_RDONLY) = -1 EACCES

# File not found — which path?
$ strace -e trace=openat,stat myapp 2>&1 | grep ENOENT
openat(AT_FDCWD, "/usr/lib/libmissing.so", ...) = -1 ENOENT

# Slow I/O — which file operations are slow?
$ strace -T -e trace=read,write,fsync mydb 2>&1 | sort -t'<' -k2 -rn | head
fsync(5)                    = 0 <0.125432>
write(5, "...", 8192)       = 8192 <0.001234>
read(5, "...", 4096)        = 4096 <0.000089>

# Hung process — what's it waiting for?
$ strace -p 5678
read(3, <unfinished ...>
# Process blocked on read from fd 3
$ ls -la /proc/5678/fd/3
lrwx------ 1 root root 64 ... /proc/5678/fd/3 -> /mnt/nfs/bigfile
# Blocked on NFS read → network issue
```

---

## 19.3 debugfs (ext4) — Filesystem Debugger

```bash
# Open filesystem in read-only mode (SAFE)
$ debugfs /dev/sda1

# Examine inode
debugfs: stat /etc/passwd
Inode: 131073   Type: regular    Mode:  0644   Flags: 0x80000
Generation: 12345   Version: 0x0000001a
User:     0   Group:     0   Size: 2048
File ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
ctime: 0x64a12345 -- Tue Jul  1 10:00:00 2025
atime: 0x64a12345 -- Tue Jul  1 10:00:00 2025
mtime: 0x64a12300 -- Tue Jul  1 09:58:00 2025
crtime: 0x63000000 -- Sat Jan  1 00:00:00 2023
EXTENTS:
(0-3): 524288-524291

# List directory contents by inode
debugfs: ls -l /lost+found
      2   40700      0      0    16384  1-Jan-2023 00:00 .
      2   40755      0      0     4096  1-Jan-2023 00:00 ..

# Dump file by inode number
debugfs: dump <131073> /tmp/recovered_file

# Show block allocation
debugfs: blocks /etc/passwd
524288 524289 524290 524291

# Show superblock info
debugfs: show_super_stats
Filesystem volume name:   rootfs
Block count:              26214400
Free blocks:              15000000
Block size:               4096
Inode count:              6553600
Free inodes:              6400000

# Examine journal
debugfs: logdump -a
Journal starts at block 1, transaction 12345
  FS block 100 logged at sequence 12345
  FS block 101 logged at sequence 12345
```

### xfs_db — XFS Debugger

```bash
$ xfs_db /dev/sda1

# Show superblock
xfs_db> sb 0
xfs_db> p
magicnum = 0x58465342   ("XFSB")
blocksize = 4096
dblocks = 26214400
agcount = 4

# Examine an inode
xfs_db> inode 131073
xfs_db> p
core.mode = 0100644
core.nlink = 1
core.size = 2048
core.nblocks = 4

# Show allocation group free space
xfs_db> freesp -s
   from      to extents  blocks    pct
      1       1     234     234   0.01
      2       4      89     267   0.01
      5      16      45     450   0.02
```

---

## 19.4 fsck — Filesystem Check and Repair

```bash
# ext4 filesystem check (read-only check)
$ e2fsck -n /dev/sda1
e2fsck 1.46.5 (30-Dec-2021)
/dev/sda1: clean, 156789/6553600 files, 11214400/26214400 blocks

# Force full check
$ e2fsck -f /dev/sda1
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sda1: 156789/6553600 files, 11214400/26214400 blocks

# Repair (answer yes to all)
$ e2fsck -y /dev/sda1

# XFS repair
$ xfs_repair /dev/sda1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
Phase 3 - for each AG...
Phase 4 - check for duplicate blocks...
Phase 5 - rebuild AG headers and trees...
Phase 6 - check inode connectivity...
Phase 7 - verify and correct link counts...
done

# Btrfs check
$ btrfs check /dev/sda1
# WARNING: btrfs check --repair is dangerous, use with caution
```

### fsck Decision Flow

```
System won't boot / FS errors in dmesg?
│
├─→ Read-only check first
│   $ e2fsck -n /dev/sda1
│   $ xfs_repair -n /dev/sda1
│
├─→ Errors found?
│   ├─→ Minor (orphan inodes, bitmap errors):
│   │   $ e2fsck -y /dev/sda1    (safe auto-fix)
│   │
│   ├─→ Moderate (directory errors, link counts):
│   │   $ e2fsck -y /dev/sda1    (fix and check lost+found)
│   │
│   └─→ Severe (superblock corruption):
│       $ e2fsck -n -b 32768 /dev/sda1  (try backup superblock)
│       $ mount -o sb=32768 /dev/sda1 /mnt (mount with backup SB)
│
└─→ No errors but still issues?
    → Check hardware: smartctl -a /dev/sda
    → Check dmesg for I/O errors
    → Consider backup and reformat
```

---

## 19.5 blktrace — Block Layer Tracing

```bash
# Start block trace
$ blktrace -d /dev/sda -o trace &

# Run workload
$ dd if=/dev/sda of=/dev/null bs=1M count=100

# Stop trace (Ctrl-C blktrace)
$ kill %1

# Parse trace
$ blkparse -i trace.blktrace.0

# Output format:
#  8,0  1  1  0.000000000  1234  A   R 1048576 + 2048 <- (8,1) 1048576
#  8,0  1  2  0.000001234  1234  Q   R 1048576 + 2048
#  8,0  1  3  0.000002345  1234  G   R 1048576 + 2048
#  8,0  1  4  0.000003456  1234  I   R 1048576 + 2048
#  8,0  1  5  0.000234567  1234  D   R 1048576 + 2048
#  8,0  1  6  0.002345678  1234  C   R 1048576 + 2048

# Action codes:
#  A = remap    Q = queued      G = get request
#  I = inserted D = dispatched  C = completed

# Visualize with btt
$ btt -i trace.blktrace.0

# iowatcher (generates SVG/video)
$ iowatcher -t trace.blktrace.0 -o trace.svg
```

### blktrace ASCII Diagram

```
Application I/O
│
▼  ┌─────────────┐
   │  Q (Queue)  │  Block I/O request created
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  G (Get)    │  Allocate request structure
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  M (Merge)  │  Merged with adjacent request? (optional)
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  I (Insert) │  Inserted into I/O scheduler queue
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  D (Dispatch)│  Sent to device driver
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  C (Complete)│  Device signals completion
   └─────────────┘

Timing between D and C = device service time
Timing between Q and C = total I/O latency
```

---

## 19.6 iostat — I/O Statistics

```bash
# Basic I/O stats, every 1 second
$ iostat -xz 1
Device  r/s    w/s    rkB/s   wkB/s  rrqm/s wrqm/s %util await r_await w_await
sda     125.0  450.0  15000   55000  10.0   85.0   78.5  2.1   0.8     2.5
nvme0n1 5000   12000  250000  480000 0.0    0.0    45.0  0.05  0.03    0.06

Key columns:
  r/s, w/s      = Read/write operations per second (IOPS)
  rkB/s, wkB/s  = Read/write throughput (KB/s)
  rrqm/s,wrqm/s = Merge rate (adjacent I/Os combined)
  %util         = Device utilization (100% = saturated for HDD)
  await         = Average I/O wait time (ms)
  r_await       = Read wait time
  w_await       = Write wait time

Interpreting %util:
  HDD:  %util ~100% = fully saturated, queue building up
  SSD:  %util can be misleading (parallel queues)
        Look at await instead — if growing, device constrained
```

---

## 19.7 ftrace — Kernel Function Tracing

```bash
# Enable function_graph tracer for FS functions
$ cd /sys/kernel/debug/tracing
$ echo 0 > tracing_on
$ echo function_graph > current_tracer
$ echo 'ext4_*' > set_ftrace_filter
$ echo 1 > tracing_on

# Trigger some FS activity
$ cat /mnt/test.txt > /dev/null

# Read trace
$ cat trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)   0.123 us    |  ext4_file_read_iter();
 0)               |  ext4_readpage() {
 0)   0.089 us    |    ext4_mpage_readpages();
 0)   0.456 us    |  }
 0)               |  ext4_file_open() {
 0)   0.034 us    |    ext4_journal_check_start();
 0)   0.567 us    |  }

# Trace specific events
$ echo 1 > events/ext4/ext4_da_write_begin/enable
$ echo 1 > events/ext4/ext4_da_write_end/enable
$ echo 1 > events/ext4/ext4_sync_file_enter/enable
$ echo 1 > events/ext4/ext4_sync_file_exit/enable

# Read event trace
$ cat trace_pipe
 myapp-1234  [001] .... 1234.567890: ext4_da_write_begin: dev 8,1 ino 131073 pos 0 len 4096
 myapp-1234  [001] .... 1234.567900: ext4_da_write_end: dev 8,1 ino 131073 pos 0 len 4096
 myapp-1234  [001] .... 1234.890000: ext4_sync_file_enter: dev 8,1 ino 131073
 myapp-1234  [001] .... 1234.990000: ext4_sync_file_exit: dev 8,1 ino 131073 ret 0

# Available FS tracepoints
$ ls events/ext4/
ext4_alloc_da_blocks      ext4_da_write_begin
ext4_da_write_end         ext4_discard_blocks
ext4_es_insert_extent     ext4_ext_map_blocks_enter
ext4_ext_map_blocks_exit  ext4_fallocate_enter
ext4_fallocate_exit       ext4_free_blocks
ext4_free_inode           ext4_journal_start
ext4_mark_inode_dirty     ext4_mballoc_alloc
ext4_read_block_bitmap    ext4_sync_file_enter
ext4_sync_file_exit       ext4_writepages
```

---

## 19.8 perf — Performance Profiling

```bash
# Profile filesystem operations
$ perf record -g -a -e 'ext4:*' -- sleep 10

# Analyze results
$ perf report
# Shows call stacks for ext4 tracepoints

# Count specific events
$ perf stat -e 'ext4:ext4_da_write_begin' \
            -e 'ext4:ext4_sync_file_enter' \
            -e 'block:block_rq_issue' \
            -- dd if=/dev/zero of=/mnt/test bs=4k count=10000

Performance counter stats for 'dd ...':
         10,000      ext4:ext4_da_write_begin
              1      ext4:ext4_sync_file_enter
            320      block:block_rq_issue
    0.456789012 seconds time elapsed

# Trace I/O latency distribution
$ perf trace -s -e 'syscalls:sys_enter_write' \
                  -e 'syscalls:sys_exit_write' -- myapp

# BPF-based tools (bcc/bpftrace)
$ bpftrace -e 'tracepoint:ext4:ext4_sync_file_enter {
    printf("fsync on inode %d\\n", args->ino);
}'
```

---

## 19.9 /proc and /sys Diagnostic Interfaces

```bash
# Mount information
$ cat /proc/mounts
/dev/sda1 / ext4 rw,relatime 0 0
/dev/sda2 /home ext4 rw,noatime 0 0
tmpfs /tmp tmpfs rw,nosuid,nodev 0 0

# Detailed mount info (mount IDs, parent, options, propagation)
$ cat /proc/self/mountinfo
22 1 8:1 / / rw,relatime shared:1 - ext4 /dev/sda1 rw

# File descriptor info for a process
$ ls -la /proc/1234/fd/
lrwx------ 1 root root 64 ... 0 -> /dev/null
lrwx------ 1 root root 64 ... 1 -> /var/log/myapp.log
lrwx------ 1 root root 64 ... 3 -> /var/lib/myapp/data.db

# Open files count
$ cat /proc/sys/fs/file-nr
5376    0    1048576
# allocated    free    maximum

# Inode cache stats
$ cat /proc/sys/fs/inode-nr
123456    456
# allocated    free

# Dentry cache stats
$ cat /proc/sys/fs/dentry-state
234567    123456    45    0    0    0
# total  unused  age_limit  want_pages

# Page cache stats
$ cat /proc/meminfo | grep -E 'Cached|Dirty|Writeback|Buffers'
Buffers:          234567 kB
Cached:          8765432 kB
Dirty:             12345 kB
Writeback:             0 kB

# ext4 runtime stats
$ cat /sys/fs/ext4/sda1/session_write_kbytes
1234567
$ cat /sys/fs/ext4/sda1/lifetime_write_kbytes
98765432

# Block device stats
$ cat /sys/block/sda/stat
   12345    5678    1234567  45678    67890    12345    8765432  23456  0  34567  69134
# Field documentation: Documentation/block/stat.rst
```

---

## 19.10 Diagnosing Common Problems

### Problem: "No space left on device" but df shows free space

```bash
# Check inode usage
$ df -i /mnt
Filesystem     Inodes   IUsed   IFree IUse% Mounted on
/dev/sda1    6553600  6553600       0  100% /mnt
# ← All inodes used! (many small files)

# Solutions:
# 1. Find and remove unneeded small files
$ find /mnt -type f | wc -l   # Count files
$ find /mnt -type f -size 0 -delete   # Remove zero-byte files

# 2. Reformat with more inodes
$ mkfs.ext4 -i 1024 /dev/sda1   # One inode per 1KB (more inodes)
# Default is one inode per 16KB
```

### Problem: FS mounted read-only unexpectedly

```bash
# Check dmesg for errors
$ dmesg | grep -i 'ext4\|error\|remount'
EXT4-fs error (device sda1): ext4_lookup:1234: inode #131073: comm ls: ...
EXT4-fs (sda1): Remounting filesystem read-only

# Kernel detected corruption and remounted RO for safety
# Fix:
$ umount /mnt
$ e2fsck -y /dev/sda1
$ mount /dev/sda1 /mnt

# Prevent auto-remount (not recommended for production):
$ tune2fs -e continue /dev/sda1  # Continue on error
$ tune2fs -e panic /dev/sda1     # Kernel panic on error (automotive safety)
```

### Problem: Hung processes in D state (uninterruptible sleep)

```bash
# Find hung processes
$ ps aux | grep ' D'
root  5678  1.0  0.1 12345 1234 ?  D  10:00 0:30 myapp

# Check what they're waiting for
$ cat /proc/5678/wchan
ext4_write_begin

# Or use stack trace
$ cat /proc/5678/stack
[<0>] ext4_write_begin+0x123/0x456
[<0>] generic_perform_write+0x89/0x234
[<0>] ext4_buffered_write_iter+0x45/0x123
[<0>] vfs_write+0x234/0x345
[<0>] ksys_write+0x56/0x78

# Usually caused by:
# 1. Disk I/O timeout (check dmesg for ata/scsi errors)
# 2. NFS server unreachable (NFS mounts)
# 3. dm-crypt CPU exhaustion
# 4. Filesystem bug (rare)
```

### Problem: Slow file deletion of large directory

```bash
# Deleting millions of files from a directory

# BAD (slow):
$ rm -rf /mnt/huge_dir/    # One unlink() syscall per file

# BETTER (parallel):
$ find /mnt/huge_dir -type f -print0 | xargs -0 -P 4 rm

# BEST (if emptying entire FS):
$ umount /mnt && mkfs.ext4 /dev/sda1 && mount /dev/sda1 /mnt

# Why slow?
# Each delete: lookup dentry → remove dir entry → free inode → free blocks
# Journaling adds overhead (each delete = journal transaction)
# For ext4: dir HTree rebalancing on every delete
```

---

## 19.11 eBPF/bpftrace FS Tracing

```bash
# Trace all ext4 file opens
$ bpftrace -e 'kprobe:ext4_file_open {
    printf("%s opened file on ext4\\n", comm);
}'

# Histogram of read sizes
$ bpftrace -e 'tracepoint:syscalls:sys_enter_read {
    @read_size = hist(args->count);
}'

# Trace fsync latency
$ bpftrace -e '
kprobe:ext4_sync_file { @start[tid] = nsecs; }
kretprobe:ext4_sync_file /@start[tid]/ {
    @fsync_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

# BCC tools for FS analysis
$ /usr/share/bcc/tools/ext4slower 1
# Shows ext4 operations slower than 1ms

$ /usr/share/bcc/tools/filetop
# Top files by I/O activity

$ /usr/share/bcc/tools/cachestat 1
# Page cache hit/miss stats per second
HITS   MISSES  DIRTIES HITRATIO   BUFFERS_MB  CACHED_MB
5432      12       45    99.78%      123        8765
```

---

## 19.12 Debugging Quick Reference

```
Symptom                      │ First Tool              │ Next Steps
─────────────────────────────┼─────────────────────────┼─────────────────
Permission denied            │ strace -e openat        │ ls -laZ, getfacl
File not found               │ strace -e openat,stat   │ Check paths, symlinks
Slow reads                   │ iostat -xz 1            │ blktrace, perf record
Slow writes / fsync          │ strace -T, iostat       │ ftrace ext4 events
Process hung (D state)       │ cat /proc/PID/stack     │ dmesg, check disk
No space (but df shows free) │ df -i                   │ Find inode-heavy dirs
FS gone read-only            │ dmesg | grep ext4       │ e2fsck, check disk
Corruption suspected         │ e2fsck -n (dry run)     │ Backup first, then -y
High I/O latency             │ iostat await             │ blktrace, BFQ tuning
Cache performance            │ /proc/meminfo Cached    │ bcc cachestat
Boot-time FS issues          │ journalctl -b           │ Check fstab, fsck
```

---

## Kernel Source References

```
Debugging interfaces:
  fs/proc/           ← /proc filesystem implementation
  fs/debugfs/        ← debugfs implementation
  fs/ext4/ioctl.c    ← ext4 debug ioctls
  kernel/trace/      ← ftrace, tracepoints

ext4 tracepoints:
  include/trace/events/ext4.h   ← All ext4 tracepoint definitions
  include/trace/events/block.h  ← Block layer tracepoints

Block stats:
  block/blk-core.c   ← /sys/block/*/stat generation
  block/genhd.c      ← Disk statistics
```

---

## Interview Questions

1. **A process is stuck in D state after calling write(). How do you diagnose?**
2. **How do you determine if I/O latency is an application, FS, or disk issue?**
3. **Explain the difference between iostat %util and await for SSDs vs HDDs.**
4. **How do you trace all file operations of a specific process?**
5. **What is blktrace? What do the action codes Q, G, I, D, C mean?**
6. **"No space left on device" but df shows 50% free. What happened?**
7. **How do you use ftrace to trace ext4 write operations?**
8. **What eBPF tools are useful for filesystem debugging?**
9. **How do you recover from an ext4 filesystem that remounted read-only?**
10. **Compare strace, ltrace, ftrace, perf for filesystem debugging.**

---

## Summary

- strace: first tool for application-level FS issues (permission, ENOENT, slow syscalls)
- debugfs/xfs_db: low-level FS inspection (examine inodes, blocks, journal)
- fsck/e2fsck/xfs_repair: corruption diagnosis and repair
- blktrace: block layer I/O tracing (latency analysis, I/O patterns)
- iostat: quick I/O performance overview (%util, await, throughput)
- ftrace: kernel-level FS function tracing (ext4 events, function_graph)
- perf: sampling profiler for FS performance analysis
- eBPF/bpftrace: advanced dynamic tracing (latency histograms, cache stats)
- /proc, /sys: always-available stats (file-nr, meminfo, block stats)
- Key diagnostic pattern: symptom → most likely layer → appropriate tool → root cause

---

*Next: [Chapter 20 — Kernel Source Code Navigation](Chapter_20_Kernel_Source.md)*
