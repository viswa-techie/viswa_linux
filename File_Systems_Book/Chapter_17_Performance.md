# Chapter 17: File System Performance

## Learning Goals
- Understand performance factors: I/O patterns, caching, fragmentation
- Master benchmarking tools (fio, bonnie++, iozone)
- Know tuning parameters for ext4, XFS, Btrfs
- Optimize for SSD vs HDD vs embedded storage

---

## 17.1 Performance Factors

```
Factor               │ Impact                          │ Tunable
─────────────────────┼─────────────────────────────────┼──────────────────
Page cache hit rate  │ 100x+ speed difference          │ RAM size, workload
I/O pattern          │ Sequential >> random (HDD)      │ Application design
Block size           │ Larger = fewer I/Os, more waste │ mkfs -b
Readahead            │ Huge for sequential reads       │ read_ahead_kb
Writeback delay      │ Batches writes, risks data loss │ dirty_* tunables
I/O scheduler        │ Reorders for device type        │ scheduler sysfs
Fragmentation        │ Scattered extents = more seeks  │ defrag tools
Journal mode         │ Safety vs speed trade-off       │ mount -o data=
Filesystem type      │ ext4 vs XFS vs Btrfs            │ mkfs choice
Alignment            │ Misaligned = read-modify-write  │ mkfs -E stride
Compression          │ CPU for less I/O (Btrfs, EROFS) │ mount -o compress
noatime              │ Eliminates access time writes   │ mount -o noatime
```

---

## 17.2 Benchmarking Tools

### fio (Flexible I/O Tester)

```bash
# Sequential read throughput
$ fio --name=seqread --rw=read --bs=1M --size=4G \
      --numjobs=1 --direct=1 --filename=/mnt/testfile

# Random read IOPS
$ fio --name=randread --rw=randread --bs=4k --size=1G \
      --numjobs=4 --iodepth=32 --direct=1 --filename=/mnt/testfile

# Sequential write throughput
$ fio --name=seqwrite --rw=write --bs=1M --size=4G \
      --numjobs=1 --direct=1 --filename=/mnt/testfile

# Mixed random read/write (database workload)
$ fio --name=randrw --rw=randrw --rwmixread=70 --bs=4k --size=1G \
      --numjobs=8 --iodepth=32 --direct=1 --filename=/mnt/testfile

# Buffered write (through page cache)
$ fio --name=bufwrite --rw=write --bs=4k --size=1G --numjobs=1 \
      --direct=0 --fsync_on_close=1 --filename=/mnt/testfile

# fio output interpretation:
#   read: IOPS=125k, BW=489MiB/s
#   lat (usec): min=8, max=1234, avg=31.5
#   clat percentiles (usec):
#     |  1.00th=[  10],  5.00th=[  12], 50.00th=[  28]
#     | 95.00th=[  65], 99.00th=[ 125], 99.99th=[1100]
```

### Other Benchmarking Tools

```bash
# bonnie++ — classic FS benchmark
$ bonnie++ -d /mnt -s 4G -n 100 -u root

# iozone — comprehensive I/O benchmark
$ iozone -a -s 1G -r 4k -r 1M -i 0 -i 1 -i 2 -f /mnt/test

# dd — simple throughput test
$ dd if=/dev/zero of=/mnt/testfile bs=1M count=4096 conv=fdatasync
# 4294967296 bytes (4.3 GB) copied, 12.5 s, 344 MB/s

# iostat — real-time I/O statistics
$ iostat -xz 1
Device  r/s    w/s    rMB/s  wMB/s  rrqm/s wrqm/s %util await
sda     0.50   125.0  0.01   48.5   0.00   24.5   45.2  0.8

# blktrace — detailed block layer tracing
$ blktrace -d /dev/sda -o trace
$ blkparse -i trace.blktrace.0
```

---

## 17.3 Tuning Parameters

### Page Cache / Writeback Tuning

```bash
# Writeback tunables
echo 5  > /proc/sys/vm/dirty_background_ratio  # Start writeback at 5%
echo 15 > /proc/sys/vm/dirty_ratio              # Block writers at 15%
echo 300 > /proc/sys/vm/dirty_writeback_centisecs # Wake every 3s
echo 1500 > /proc/sys/vm/dirty_expire_centisecs   # Expire dirty at 15s

# For databases (lower latency, less dirty data):
echo 3  > /proc/sys/vm/dirty_background_ratio
echo 10 > /proc/sys/vm/dirty_ratio

# For streaming writes (higher throughput):
echo 20 > /proc/sys/vm/dirty_background_ratio
echo 40 > /proc/sys/vm/dirty_ratio

# Readahead tuning
echo 256 > /sys/block/sda/queue/read_ahead_kb  # 256KB readahead

# For SSDs (less readahead needed):
echo 128 > /sys/block/nvme0n1/queue/read_ahead_kb

# For HDD sequential workloads:
echo 4096 > /sys/block/sda/queue/read_ahead_kb  # 4MB readahead
```

### ext4 Mount Options for Performance

```bash
# High-performance ext4 mount
mount -t ext4 -o \
    noatime,\                    # Skip access time updates
    nodiratime,\                 # Skip directory access time
    commit=60,\                  # Journal commit every 60s (risk!)
    data=writeback,\             # Fastest journal mode (risk!)
    delalloc,\                   # Delayed allocation (default)
    nobarrier,\                  # Skip write barriers (risk!)
    discard \                    # TRIM for SSDs
    /dev/sda1 /mnt

# Safe high-performance ext4
mount -t ext4 -o \
    noatime,\
    data=ordered,\               # Default safe mode
    delalloc,\
    barrier=1,\                  # Write barriers ON
    commit=5 \                   # Default commit interval
    /dev/sda1 /mnt

# noatime saves ~30-50% of metadata writes on read-heavy workloads
```

### XFS Tuning

```bash
# XFS mount options
mount -t xfs -o \
    noatime,\
    logbufs=8,\                  # Log buffers (more = less log I/O)
    logbsize=256k,\              # Log buffer size
    allocsize=64m,\              # Speculative preallocation size
    inode64 \                    # Place inodes anywhere on disk
    /dev/sda1 /mnt

# XFS is generally well-tuned out of the box for large files
```

### I/O Scheduler Selection

```bash
# For NVMe SSDs:
echo "none" > /sys/block/nvme0n1/queue/scheduler
# No scheduling needed — device is fast enough

# For SATA SSDs:
echo "mq-deadline" > /sys/block/sda/queue/scheduler

# For HDDs:
echo "bfq" > /sys/block/sda/queue/scheduler
# BFQ provides fair bandwidth allocation

# For embedded/automotive:
echo "mq-deadline" > /sys/block/mmcblk0/queue/scheduler
# Low overhead, good latency for eMMC/UFS
```

---

## 17.4 SSD Optimization

```
SSD-specific optimizations:

1. TRIM/Discard:
   mount -o discard           ← Continuous TRIM (some latency)
   systemctl enable fstrim.timer ← Weekly TRIM (preferred)

2. I/O Scheduler: none or mq-deadline
   echo none > /sys/block/nvme0n1/queue/scheduler

3. Reduce write amplification:
   - noatime mount option
   - Appropriate dirty_ratio settings
   - ext4: journal_async_commit

4. Queue depth:
   echo 256 > /sys/block/nvme0n1/queue/nr_requests

5. NCQ (Native Command Queuing):
   Enabled by default for SATA/NVMe

6. File system alignment:
   mkfs.ext4 -E stride=1,stripe-width=1 /dev/sda1
   (Most modern mkfs auto-detects alignment)

Performance profile (NVMe SSD):
  Sequential read:  3500 MB/s
  Sequential write: 3000 MB/s
  Random 4K read:   500K IOPS
  Random 4K write:  400K IOPS
  Latency (avg):    20-50 μs
```

---

## 17.5 HDD Optimization

```
HDD-specific optimizations:

1. Maximize sequential access:
   - Large readahead: echo 4096 > /sys/block/sda/queue/read_ahead_kb
   - Delayed allocation (ext4 default)
   - Large block I/O sizes

2. I/O Scheduler: bfq or mq-deadline
   echo bfq > /sys/block/sda/queue/scheduler

3. Reduce seeks:
   - noatime (avoid read-triggered writes)
   - ext4 flex_bg (group metadata together)
   - Keep related data in same block group

4. NCQ: enable (usually default)
   Allows disk to reorder commands for less head movement

5. Write caching: enable (with barriers)
   hdparm -W1 /dev/sda

Performance profile (7200 RPM HDD):
  Sequential read:  150-200 MB/s
  Sequential write: 130-180 MB/s
  Random 4K read:   75-150 IOPS
  Random 4K write:  75-150 IOPS
  Latency (avg):    5-15 ms
```

---

## 17.6 Automotive / Embedded Performance

```
Automotive constraints:
  - eMMC/UFS storage (not NVMe)
  - Limited write endurance
  - Must not write continuously
  - Boot time critical (< 2 seconds for display)

Optimizations:
  1. Read-only partitions: SquashFS or EROFS
     - Compressed: less I/O, less storage
     - No writes: no wear, no corruption risk

  2. Minimize writes to /data:
     - noatime on all mounts
     - Increase commit interval
     - Use tmpfs for temporary files/logs

  3. Fast boot:
     - SquashFS with LZ4 compression (fast decompression)
     - dm-verity for integrity (no fsck needed)
     - Parallel mount of independent partitions

  4. Write endurance:
     - Monitor /sys/block/mmcblk0/device/life_time
     - Limit logging to RAM, batch writes
     - f2fs for /data (flash-friendly)

  5. Power-loss resilience:
     - ext4 with data=journal for critical data
     - A/B partition scheme for OTA
     - Atomic writes where possible
```

---

## 17.7 Performance Anti-Patterns

```
Common mistakes that kill FS performance:

1. ❌ sync after every write
   fsync(fd) after every small write → serializes I/O
   ✓ Batch writes, fsync periodically

2. ❌ Opening many small files
   open() + read 100 bytes + close() × 10,000 = slow
   ✓ Pack small items into larger files or database

3. ❌ stat() storms
   ls -la on directory with 100K files
   ✓ Use readdir() only, avoid stat() per file

4. ❌ Ignoring noatime
   Every read updates atime → metadata write per read!
   ✓ mount -o noatime (or relatime)

5. ❌ Wrong I/O scheduler for device type
   BFQ on NVMe → unnecessary overhead
   ✓ Match scheduler to device

6. ❌ O_DIRECT for small random reads
   Bypasses page cache → every read hits disk
   ✓ Use buffered I/O unless you have your own cache

7. ❌ Not pre-allocating for known-size files
   Append writes → fragmented allocation
   ✓ fallocate(fd, 0, 0, known_size) before writing
```

---

## Kernel Source References

```
Performance-related code:
  mm/filemap.c             ← Page cache read/write performance
  mm/readahead.c           ← Readahead algorithms
  mm/page-writeback.c      ← Dirty page thresholds, writeback
  block/blk-mq.c           ← Multi-queue block layer
  block/mq-deadline.c      ← mq-deadline scheduler
  block/bfq-iosched.c      ← BFQ scheduler

Tuning interfaces:
  /proc/sys/vm/dirty_*     ← Writeback tunables
  /sys/block/*/queue/*     ← Block device tunables
  /sys/fs/ext4/*/          ← ext4 runtime stats
```

---

## Interview Questions

1. **What are the top 3 things you'd tune for ext4 performance on an SSD?**
2. **How does readahead improve sequential read performance?**
3. **What is the impact of noatime? Why is it so important?**
4. **Compare dirty_background_ratio and dirty_ratio tuning strategies.**
5. **When would you use O_DIRECT? When would it hurt performance?**
6. **How do you benchmark file system performance? Name 3 tools.**
7. **What I/O scheduler would you use for NVMe? HDD? eMMC?**
8. **How does delayed allocation improve write performance?**
9. **What FS optimizations are important for automotive/embedded systems?**
10. **Name 3 common performance anti-patterns and their solutions.**

---

## Summary

- Page cache hit rate is the #1 performance factor (RAM → I/O avoidance)
- noatime: simple mount option that eliminates unnecessary metadata writes
- Readahead: kernel prefetches sequential pages; tunable via read_ahead_kb
- Writeback: dirty_background_ratio starts background flush; dirty_ratio blocks writers
- SSD: use `none` scheduler, TRIM, low dirty thresholds
- HDD: use `bfq`, large readahead, maximize sequential access
- Benchmark with fio (industry standard); monitor with iostat, blktrace
- Embedded: read-only FS (SquashFS), minimize writes, monitor flash life

---

*Next: [Chapter 18 — File System Security](Chapter_18_Security.md)*
