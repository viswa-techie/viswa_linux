# Chapter 13: Special File Systems — procfs, sysfs, tmpfs, debugfs

## Learning Goals
- Understand pseudo file systems that exist only in memory
- Know how procfs exports process and kernel info
- Master sysfs as the device model interface
- Use tmpfs for high-performance temporary storage
- Use debugfs for kernel debugging interfaces

---

## 13.1 Overview of Special File Systems

```
Special (pseudo/virtual) file systems:
  - No backing block device — exist purely in kernel memory
  - Created by the kernel, not by mkfs
  - Provide kernel ↔ user space interfaces
  - Mounted automatically at boot

  FS          │ Mount Point      │ Purpose
  ────────────┼──────────────────┼──────────────────────────
  procfs      │ /proc            │ Process + kernel info
  sysfs       │ /sys             │ Device model, drivers
  tmpfs       │ /tmp, /run, /dev/shm │ RAM-backed temp storage
  debugfs     │ /sys/kernel/debug│ Kernel debugging info
  devtmpfs    │ /dev             │ Device nodes
  configfs    │ /sys/kernel/config│ User-driven kernel config
  securityfs  │ /sys/kernel/security │ Security module interfaces
  cgroup2     │ /sys/fs/cgroup   │ Control groups
  tracefs     │ /sys/kernel/tracing │ ftrace interface
  bpffs       │ /sys/fs/bpf     │ BPF maps and programs
```

---

## 13.2 procfs — Process and Kernel Information

### Architecture

```
procfs provides a file-based interface to kernel data:

  /proc/
  ├── [pid]/                    ← Per-process directories
  │   ├── status               ← Process status (human-readable)
  │   ├── stat                 ← Process statistics (for tools)
  │   ├── maps                 ← Memory mappings
  │   ├── fd/                  ← Open file descriptors
  │   ├── cmdline              ← Command line arguments
  │   ├── exe → /path/to/binary ← Symlink to executable
  │   ├── cwd → /current/dir  ← Current working directory
  │   ├── root → /            ← Root directory
  │   ├── mountinfo            ← Mount information
  │   ├── io                   ← I/O statistics
  │   └── smaps                ← Detailed memory maps
  │
  ├── meminfo                   ← System memory information
  ├── cpuinfo                   ← CPU information
  ├── filesystems               ← Registered file systems
  ├── mounts → self/mounts     ← Mount points
  ├── partitions                ← Disk partitions
  ├── sys/                      ← Sysctl tunables
  │   ├── fs/                  ← FS-related sysctls
  │   │   ├── file-max         ← Max open files system-wide
  │   │   ├── dentry-state     ← Dcache statistics
  │   │   └── inode-state      ← Icache statistics
  │   ├── vm/                  ← Virtual memory
  │   │   ├── dirty_ratio      ← Writeback threshold
  │   │   └── swappiness       ← Swap aggressiveness
  │   └── kernel/              ← Kernel parameters
  └── slabinfo                  ← Slab allocator stats
```

### Creating procfs Entries (Kernel Module)

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

/* Simple proc file using seq_file */
static int my_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "My driver status: active\n");
    seq_printf(m, "Operations count: %lu\n", my_ops_count);
    return 0;
}

static int my_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, my_proc_show, NULL);
}

static const struct proc_ops my_proc_ops = {
    .proc_open    = my_proc_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

/* In module init: */
proc_create("my_driver_status", 0444, NULL, &my_proc_ops);

/* In module exit: */
remove_proc_entry("my_driver_status", NULL);

/* Result: cat /proc/my_driver_status */
```

### procfs for FS Debugging

```bash
# File system related proc entries
$ cat /proc/filesystems           # Registered FS types
$ cat /proc/mounts                # Current mounts
$ cat /proc/self/mountinfo        # Detailed mount info
$ cat /proc/sys/fs/file-nr        # Open files: allocated, free, max
$ cat /proc/sys/fs/dentry-state   # Dcache statistics
$ cat /proc/sys/fs/inode-state    # Icache statistics

# Per-process FS info
$ cat /proc/self/fd               # Open file descriptors
$ cat /proc/self/fdinfo/3         # Details about fd 3
$ cat /proc/self/io               # I/O bytes read/written
$ cat /proc/self/mountinfo        # Process mount namespace
$ readlink /proc/self/cwd         # Current working directory
$ readlink /proc/self/root        # Root directory
```

---

## 13.3 sysfs — Device Model Interface

### Architecture

```
sysfs exposes the kernel's device model:

  /sys/
  ├── block/                     ← Block devices
  │   ├── sda/
  │   │   ├── queue/            ← I/O scheduler, settings
  │   │   │   ├── scheduler     ← Current I/O scheduler
  │   │   │   ├── read_ahead_kb ← Readahead size
  │   │   │   ├── nr_requests   ← Queue depth
  │   │   │   └── rotational    ← 0=SSD, 1=HDD
  │   │   ├── size              ← Device size (sectors)
  │   │   └── sda1/, sda2/     ← Partitions
  │   └── dm-0/                 ← Device mapper
  │
  ├── class/                     ← Devices by class
  │   ├── block/
  │   ├── net/
  │   └── tty/
  │
  ├── devices/                   ← Device hierarchy (bus topology)
  │   ├── platform/
  │   ├── pci0000:00/
  │   └── virtual/
  │
  ├── fs/                        ← File system attributes
  │   ├── ext4/
  │   │   └── sda1/             ← Per-mounted-FS attributes
  │   │       ├── delayed_allocation_blocks
  │   │       ├── mb_groups
  │   │       └── es_shrinker_info
  │   ├── btrfs/
  │   └── fuse/
  │
  ├── module/                    ← Loaded kernel modules
  │   ├── ext4/
  │   │   └── parameters/
  │   └── xfs/
  │
  └── kernel/
      ├── debug/ → debugfs
      └── mm/
          └── transparent_hugepage/
```

### sysfs for I/O Tuning

```bash
# I/O scheduler
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber bfq none

$ echo "bfq" > /sys/block/sda/queue/scheduler

# Readahead
$ cat /sys/block/sda/queue/read_ahead_kb
128
$ echo 256 > /sys/block/sda/queue/read_ahead_kb

# Device type
$ cat /sys/block/sda/queue/rotational
0    # SSD

# Queue parameters
$ cat /sys/block/sda/queue/nr_requests
256

# ext4 specific stats
$ cat /sys/fs/ext4/sda1/delayed_allocation_blocks
0
$ cat /sys/fs/ext4/sda1/lifetime_write_kbytes
12345678
```

### Creating sysfs Entries

```c
#include <linux/kobject.h>
#include <linux/sysfs.h>

static int my_value = 42;

static ssize_t my_value_show(struct kobject *kobj,
                              struct kobj_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "%d\n", my_value);
}

static ssize_t my_value_store(struct kobject *kobj,
                               struct kobj_attribute *attr,
                               const char *buf, size_t count)
{
    sscanf(buf, "%d", &my_value);
    return count;
}

static struct kobj_attribute my_value_attr =
    __ATTR(my_value, 0644, my_value_show, my_value_store);

/* Result: echo 100 > /sys/my_driver/my_value */
```

---

## 13.4 tmpfs — RAM-Backed File System

### How tmpfs Works

```
tmpfs stores files in:
  1. Page cache (RAM) — for file data
  2. Swap — if memory pressure moves pages to swap
  
  No disk I/O for normal operations → extremely fast!

  ┌──────────────────────────────────────────┐
  │             tmpfs                         │
  │                                          │
  │  File data → Page cache pages            │
  │  Metadata → struct inode (in memory)      │
  │  Dentries → dcache (in memory)            │
  │                                          │
  │  Under memory pressure:                   │
  │    Pages moved to swap (if swap enabled)  │
  │                                          │
  │  On reboot: ALL DATA LOST                │
  └──────────────────────────────────────────┘

Common mounts:
  /tmp                ← Temporary files (fast)
  /run                ← Runtime data (PID files, sockets)
  /dev/shm            ← POSIX shared memory
  /sys/fs/cgroup      ← cgroup filesystem

Mounting:
  $ mount -t tmpfs -o size=2G,mode=1777 tmpfs /tmp
  # size=2G → max 2 GB (can overcommit!)
  # mode=1777 → sticky bit (only owner can delete)
```

### tmpfs vs ramfs

```
Feature          │ tmpfs             │ ramfs
─────────────────┼───────────────────┼──────────────────
Size limit       │ Yes (configurable)│ No (grows until OOM)
Swap support     │ Yes               │ No
Data on reboot   │ Lost              │ Lost
Default mount    │ Yes               │ No
Use case         │ /tmp, /run        │ Initramfs only
Memory reclaim   │ Pages can swap    │ Pages pinned in RAM
```

### tmpfs Implementation

```c
/* mm/shmem.c — tmpfs is built on shared memory ("shmem") */

static const struct inode_operations shmem_dir_inode_operations = {
    .create     = shmem_create,
    .lookup     = simple_lookup,
    .link       = shmem_link,
    .unlink     = shmem_unlink,
    .symlink    = shmem_symlink,
    .mkdir      = shmem_mkdir,
    .rmdir      = shmem_rmdir,
    .mknod      = shmem_mknod,
    .rename     = shmem_rename2,
    .tmpfile    = shmem_tmpfile,
};

/* Key: no block device, no disk I/O, all in page cache */
/* shmem_get_folio() allocates pages on demand */
```

---

## 13.5 debugfs — Kernel Debugging Interface

### Purpose and Usage

```
debugfs provides kernel developers with a debug-only interface:
  - NOT for production use (no stability guarantees)
  - Mounted at /sys/kernel/debug
  - Requires root access
  - Can expose internal kernel state

Common debug files:
  /sys/kernel/debug/
  ├── block/                     ← Block layer debug
  │   └── sda/                  ← Per-device stats
  ├── ext4/                      ← ext4 debug info
  │   └── sda1/
  │       ├── mb_groups         ← mballoc group info
  │       └── es_shrinker_info  ← Extent status shrinker
  ├── bdi/                       ← Backing device info
  ├── kmemleak                   ← Memory leak detector
  ├── tracing/                   ← ftrace interface
  │   ├── trace                 ← Trace output
  │   ├── trace_pipe            ← Streaming trace
  │   ├── available_tracers     ← Available tracers
  │   └── events/               ← Tracepoint events
  │       └── ext4/             ← ext4 tracepoints
  │           ├── ext4_da_write_begin/
  │           ├── ext4_sync_file_enter/
  │           └── ext4_writepages/
  └── sleep_time                 ← Sleep profiling
```

### Creating debugfs Entries

```c
#include <linux/debugfs.h>

static struct dentry *my_debugfs_dir;
static u32 my_debug_counter;
static bool my_debug_enabled;

static int __init my_init(void)
{
    /* Create directory */
    my_debugfs_dir = debugfs_create_dir("my_driver", NULL);
    
    /* Create files */
    debugfs_create_u32("counter", 0644, my_debugfs_dir, &my_debug_counter);
    debugfs_create_bool("enabled", 0644, my_debugfs_dir, &my_debug_enabled);
    
    /* Custom file with fops */
    debugfs_create_file("status", 0444, my_debugfs_dir, NULL, &my_status_fops);
    
    return 0;
}

/* Usage:
 *   $ cat /sys/kernel/debug/my_driver/counter
 *   42
 *   $ echo 0 > /sys/kernel/debug/my_driver/counter
 *   $ cat /sys/kernel/debug/my_driver/enabled
 *   Y
 */
```

### Using ext4 debugfs for Troubleshooting

```bash
# Mount debugfs
$ sudo mount -t debugfs none /sys/kernel/debug

# ext4 mballoc group statistics
$ cat /sys/kernel/debug/ext4/sda1/mb_groups
#group: free  frags first [ 2^0   2^1   2^2   ... ]
#    0: 12288    3  1024 [ 0     0     0     ... ]

# Block I/O tracing via debugfs
$ echo 1 > /sys/kernel/debug/tracing/events/ext4/ext4_da_write_begin/enable
$ cat /sys/kernel/debug/tracing/trace
```

---

## 13.6 devtmpfs — Auto-Created Device Nodes

```
devtmpfs automatically creates /dev entries when devices are detected:

  Kernel detects new device
    │
    ▼
  device_add() → devtmpfs_create_node()
    │
    ▼
  Create /dev/sda, /dev/ttyS0, etc.
  (major:minor from device registration)

  Before devtmpfs (historic):
    udev ran in user space, created nodes after delay
    Boot was slower, race conditions possible

  With devtmpfs:
    Basic /dev nodes available immediately
    udev still runs for permissions, symlinks, rules

  Mount at boot:
    devtmpfs mounted on /dev by kernel itself
    Kernel config: CONFIG_DEVTMPFS=y, CONFIG_DEVTMPFS_MOUNT=y
```

---

## 13.7 Other Virtual File Systems

### configfs

```
configfs: User-driven kernel object configuration

  /sys/kernel/config/
  └── target/                  ← SCSI target subsystem
      └── iscsi/               ← iSCSI targets
          └── iqn.2024-01.com.example:storage/
              └── tpg1/
                  ├── attrib/
                  └── lun/

  Unlike sysfs (kernel creates entries),
  configfs lets USER create directories → kernel creates objects.

  Example: Create iSCSI target by mkdir
    $ mkdir /sys/kernel/config/target/iscsi/iqn.2024-01.../
```

### securityfs

```
securityfs: Interface for Linux Security Modules (LSM)

  /sys/kernel/security/
  ├── apparmor/          ← AppArmor profiles
  ├── selinux/           ← SELinux status
  │   ├── enforce        ← 0=permissive, 1=enforcing
  │   └── policy         ← Loaded policy binary
  ├── integrity/         ← IMA/EVM
  └── tomoyo/            ← TOMOYO profiles
```

### tracefs

```
tracefs: ftrace tracing interface (split from debugfs)

  /sys/kernel/tracing/            (or /sys/kernel/debug/tracing/)
  ├── current_tracer              ← Active tracer
  ├── trace                       ← Trace buffer output
  ├── trace_pipe                  ← Streaming output
  ├── tracing_on                  ← Enable/disable
  ├── buffer_size_kb              ← Per-CPU buffer size
  ├── available_filter_functions  ← Functions for ftrace
  └── events/
      ├── ext4/                   ← ext4 tracepoints
      │   ├── ext4_writepages/
      │   │   └── enable
      │   └── ext4_da_write_begin/
      ├── block/                  ← Block layer events
      │   └── block_rq_complete/
      └── filemap/                ← Page cache events
          └── filemap_fault/
```

---

## Kernel Source References

```
procfs:
  fs/proc/                    ← procfs implementation
  fs/proc/base.c              ← /proc/[pid]/ entries
  fs/proc/meminfo.c           ← /proc/meminfo
  fs/proc/stat.c              ← /proc/stat

sysfs:
  fs/sysfs/                   ← sysfs implementation
  fs/sysfs/file.c             ← sysfs file operations
  fs/sysfs/dir.c              ← sysfs directory ops

tmpfs:
  mm/shmem.c                  ← tmpfs/shmem implementation

debugfs:
  fs/debugfs/                 ← debugfs implementation
  fs/debugfs/inode.c          ← debugfs_create_file(), etc.

devtmpfs:
  drivers/base/devtmpfs.c    ← devtmpfs implementation

tracefs:
  fs/tracefs/                 ← tracefs implementation
```

---

## Interview Questions

1. **What is a pseudo/virtual file system? How does it differ from disk-based FS?**
2. **What information does /proc provide? Give 5 useful files.**
3. **How do you create a new procfs entry from a kernel module?**
4. **What is the relationship between sysfs and the kernel device model?**
5. **How does tmpfs store file data? What happens under memory pressure?**
6. **What is the difference between tmpfs and ramfs?**
7. **What is debugfs used for? Why shouldn't it be used in production?**
8. **How does devtmpfs create device nodes automatically?**
9. **Name 3 ways to tune file system behavior via /proc/sys/.**
10. **How would you use tracefs to debug ext4 writeback issues?**

---

## Summary

- **procfs** (/proc): Process info (/proc/[pid]/) + kernel tunables (/proc/sys/)
- **sysfs** (/sys): Device model hierarchy, I/O tuning, per-FS attributes
- **tmpfs**: RAM-backed FS for /tmp, /run; data lost on reboot; can use swap
- **debugfs**: Kernel debug interface; unstable API; requires root; not for production
- **devtmpfs**: Auto-creates /dev entries when devices are detected
- **tracefs**: ftrace interface for kernel tracing/profiling
- All pseudo FS: no block device, data in kernel memory, mounted at boot
- Key for FS work: /proc/sys/vm/dirty_*, /sys/block/*/queue/*, debugfs ext4/

---

*Next: [Chapter 14 — Network File Systems: NFS, SMB/CIFS](Chapter_14_Network_File_Systems.md)*
