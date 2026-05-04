# Chapter 5: Linux Kernel Subsystems

## Learning Goals
- Deep-dive into each major kernel subsystem
- Understand how subsystems interact during kernel operation
- Know the key data structures and APIs for each subsystem
- Grasp the role of each subsystem in both desktop and embedded Linux

---

## 5.1 Process Management

The process management subsystem handles process creation, destruction, scheduling, and state management.

```
Process Management Architecture:

                    User Space
                        │  fork() / clone() / exec()
                        ▼
              ┌──────────────────┐
              │  System Call     │
              │  Entry           │
              └────────┬─────────┘
                       │
         ┌─────────────┼──────────────────┐
         │             │                  │
  ┌──────▼──────┐ ┌────▼─────┐  ┌────────▼────────┐
  │   fork()    │ │ exec()   │  │   Scheduler     │
  │ kernel_clone│ │ do_execve│  │                 │
  │ copy_process│ │ load_elf │  │ ┌─────────────┐ │
  │             │ │          │  │ │  CFS (fair)  │ │
  └──────┬──────┘ └────┬─────┘  │ │  RT (fifo/rr)│ │
         │             │        │ │  DL (deadline)│ │
         ▼             ▼        │ │  IDLE         │ │
  ┌─────────────────────────┐   │ └─────────────┘ │
  │     task_struct          │   └────────────────┘
  │  (one per process/thread)│
  │  PID, state, priority,  │
  │  mm, files, signals...  │
  └─────────────────────────┘
```

Key process management functions:

```c
/* Process creation */
pid_t kernel_clone(struct kernel_clone_args *args);  /* Core clone */
long do_fork(unsigned long clone_flags, ...);         /* fork() path */
int do_execve(struct filename *filename, ...);        /* exec() path */

/* Process termination */
void do_exit(long code);                              /* Process exit */
long do_wait(struct wait_opts *wo);                    /* Parent wait */

/* Scheduling */
void schedule(void);                                   /* Context switch */
void wake_up_process(struct task_struct *p);           /* Wake a task */
void set_current_state(long state);                    /* Set task state */
```

---

## 5.2 Memory Management

The MM subsystem manages virtual memory, physical page allocation, and memory mapping.

```
Memory Management Layers:

     User process calls malloc()
              │
              ▼
     ┌─────────────────┐
     │  C Library       │  brk() or mmap() system call
     │  (glibc/Bionic)  │
     └────────┬─────────┘
              │
     ┌────────▼─────────┐
     │   VMA Manager    │  vm_area_struct — tracks virtual regions
     │   (mmap, brk)    │
     └────────┬─────────┘
              │ Page fault
     ┌────────▼─────────┐
     │   Page Fault     │  Allocates physical page on demand
     │   Handler        │  (demand paging / COW)
     └────────┬─────────┘
              │
     ┌────────▼─────────┐
     │  Page Allocator  │  Buddy system — alloc 2^n pages
     │  (Buddy System)  │
     └────────┬─────────┘
              │ Small objects
     ┌────────▼─────────┐
     │  Slab Allocator  │  SLUB — caches for fixed-size objects
     │  (SLUB)          │  (task_struct, inode, dentry, etc.)
     └────────┬─────────┘
              │
     ┌────────▼─────────┐
     │  Physical Memory │  Actual RAM pages (struct page)
     └──────────────────┘
```

Memory zones:

| Zone | Description | Typical Range |
|------|-------------|---------------|
| `ZONE_DMA` | Legacy DMA (ISA) | 0 – 16 MB |
| `ZONE_DMA32` | 32-bit DMA capable | 0 – 4 GB |
| `ZONE_NORMAL` | Regular kernel memory | Direct mapped region |
| `ZONE_HIGHMEM` | Beyond direct map (32-bit only) | > 896 MB (32-bit) |
| `ZONE_MOVABLE` | Movable pages for compaction | Configurable |

---

## 5.3 Device Drivers

Device drivers translate generic kernel APIs into hardware-specific operations.

```
Driver Classification:

┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Character   │   │    Block     │   │   Network    │
│  Devices     │   │   Devices    │   │   Devices    │
├──────────────┤   ├──────────────┤   ├──────────────┤
│ /dev/ttyS0   │   │ /dev/sda     │   │ eth0, wlan0  │
│ /dev/i2c-0   │   │ /dev/nvme0n1 │   │ can0         │
│ /dev/video0  │   │ /dev/mmcblk0 │   │ lo           │
├──────────────┤   ├──────────────┤   ├──────────────┤
│ Byte stream  │   │ Fixed-size   │   │ Packet-based │
│ Sequential   │   │ blocks       │   │ Async        │
│ No seek (typ)│   │ Random access│   │ No /dev entry│
│ file_ops     │   │ block_ops    │   │ net_device_ops│
└──────────────┘   └──────────────┘   └──────────────┘

Linux Driver Model (bus-device-driver):

┌─────────────────────────────────────────────────┐
│                  Driver Core                     │
│           (drivers/base/)                        │
├─────────────────────────────────────────────────┤
│                                                 │
│   Bus          Device          Driver            │
│  ┌──────┐    ┌──────────┐   ┌──────────┐       │
│  │ PCI  │    │ PCI dev  │   │ PCI drv  │       │
│  │ USB  │◄──│ USB dev  │──►│ USB drv  │       │
│  │ I2C  │    │ I2C dev  │   │ I2C drv  │       │
│  │ SPI  │    │ platform │   │ platform │       │
│  │ plat │    │ device   │   │ driver   │       │
│  └──────┘    └──────────┘   └──────────┘       │
│       │           │              │               │
│       └───────────┼──────────────┘               │
│              match()                             │
│       (compatible string / ID table)             │
└─────────────────────────────────────────────────┘
```

---

## 5.4 Interrupt Subsystem

```
Interrupt Subsystem Architecture:

Hardware IRQ Line
      │
      ▼
┌──────────────────┐
│  Interrupt       │   GIC (ARM), APIC (x86)
│  Controller      │   Manages priorities, routing
│  Driver          │
└────────┬─────────┘
         │
┌────────▼─────────┐
│  Generic IRQ     │   include/linux/irq.h
│  Layer           │   Chip-independent abstraction
│  (irq_desc[])    │ 
└────────┬─────────┘
         │
┌────────▼─────────┐
│  Action Chain    │   Handlers registered via request_irq()
│  (irqaction)     │   Each IRQ can have multiple handlers (shared)
└────────┬─────────┘
         │
         ├──► Top half: hardirq handler (fast, no sleep)
         │
         └──► Bottom half: softirq / tasklet / workqueue / threaded IRQ
```

```c
/* Register an interrupt handler */
int ret = devm_request_irq(&pdev->dev,
    irq,                      /* IRQ number */
    my_irq_handler,           /* Top-half handler */
    IRQF_SHARED,              /* Flags: shared IRQ line */
    "my-device",              /* Name for /proc/interrupts */
    my_dev_data               /* Private data (cookie) */
);

/* IRQ handler — runs in interrupt context */
static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_dev *dev = data;
    u32 status = readl(dev->base + STATUS_REG);

    if (!(status & MY_IRQ_FLAG))
        return IRQ_NONE;       /* Not our interrupt (shared) */

    writel(status, dev->base + STATUS_REG);  /* Acknowledge */
    /* Schedule deferred work */
    return IRQ_HANDLED;
}
```

---

## 5.5 I/O Subsystem

```
I/O Stack:

  Application: read(fd, buf, size)
       │
       ▼
  ┌──────────────┐
  │  VFS Layer   │   Dispatches to correct FS or device
  └──────┬───────┘
         │
    ┌────┴────────────────┬────────────────────┐
    │                     │                    │
┌───▼───────┐    ┌───────▼──────┐    ┌────────▼────────┐
│ File System│    │ Char Device  │    │  Network Socket │
│ (ext4/f2fs)│    │ (/dev/ttyS0) │    │  (TCP/UDP)      │
└───┬───────┘    └──────────────┘    └─────────────────┘
    │
┌───▼───────┐
│Block Layer│   I/O scheduler (mq-deadline, bfq, none)
│  (bio)    │   Merging, sorting, plugging
└───┬───────┘
    │
┌───▼───────┐
│Block Driver│  SCSI, NVMe, MMC, virtio
└───┬───────┘
    │
┌───▼───────┐
│ Hardware  │  DMA, interrupts, command queues
└───────────┘
```

---

## 5.6 File Systems

```
VFS — The Unifying Abstraction:

struct super_block  ←── Mounted filesystem instance
       │
struct inode        ←── On-disk file metadata (permissions, size, blocks)
       │
struct dentry       ←── Directory entry (filename → inode mapping, cached)
       │
struct file         ←── Open file instance (position, flags, f_op)
       │
file_operations     ←── Callback table: read, write, ioctl, mmap, poll

Every filesystem implements these callbacks:
┌─────────┬────────────────────────────────────────────────┐
│  VFS    │  ext4_file_read()   f2fs_file_read()          │
│  calls  │  ext4_file_write()  f2fs_file_write()         │
│  ───►   │  ext4_lookup()      f2fs_lookup()             │
│         │  ext4_mkdir()       f2fs_mkdir()              │
└─────────┴────────────────────────────────────────────────┘
```

Common Linux file systems:

| File System | Type | Key Feature | Use Case |
|-------------|------|------------|----------|
| ext4 | Disk | Journaling, extents | Default Linux desktop/server |
| f2fs | Disk | Flash-friendly, log-structured | Android, embedded with eMMC/UFS |
| xfs | Disk | High-performance, scalable | Large files, servers |
| btrfs | Disk | COW, snapshots, checksums | Advanced data management |
| tmpfs | RAM | Memory-backed | /tmp, /dev/shm |
| procfs | Virtual | Process information | /proc |
| sysfs | Virtual | Device/driver model | /sys |
| debugfs | Virtual | Debug information | /sys/kernel/debug |
| devtmpfs | Virtual | Device nodes | /dev (auto-populated) |

---

## 5.7 Networking Subsystem

```
Network Stack Architecture:

  Application: send(sockfd, data, len, 0)
       │
  ┌────▼──────────────────┐
  │  Socket Layer          │  socket(), bind(), connect(), send()
  │  (struct socket)       │  Protocol family selection
  └────┬──────────────────┘
       │
  ┌────▼──────────────────┐
  │  Transport Layer       │  TCP (stream), UDP (datagram)
  │  (struct sock)         │  Congestion control, reliability
  └────┬──────────────────┘
       │
  ┌────▼──────────────────┐
  │  Network Layer         │  IPv4/IPv6 routing
  │  (Netfilter hooks)     │  iptables/nftables filtering here
  └────┬──────────────────┘
       │
  ┌────▼──────────────────┐
  │  Link Layer            │  ARP, neighbor discovery
  │                        │  Ethernet framing
  └────┬──────────────────┘
       │
  ┌────▼──────────────────┐
  │  Network Driver        │  NIC driver (e1000, igb, stmmac)
  │  (struct net_device)   │  NAPI for high-performance RX
  └────┬──────────────────┘
       │
       ▼  Hardware (NIC, PHY)
```

---

## 5.8 Security Subsystem

```
Linux Security Architecture:

  System Call
       │
       ▼
  ┌──────────────────┐
  │  DAC Check       │  Traditional Unix permissions
  │  (uid/gid/mode)  │  Read/Write/Execute × Owner/Group/Other
  └────┬─────────────┘
       │ passes
       ▼
  ┌──────────────────┐
  │  LSM Hooks       │  Linux Security Module framework
  │  security_*()    │  Pluggable security checks
  └────┬─────────────┘
       │
  ┌────┴─────────────────────────────────────┐
  │                                          │
  ▼                                          ▼
┌──────────────┐                    ┌──────────────┐
│  SELinux      │                    │  AppArmor    │
│  (Label-based │                    │  (Path-based │
│   MAC)        │                    │   MAC)       │
└──────────────┘                    └──────────────┘

Additional security mechanisms:
  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐
  │ Capabilities│  │  Seccomp     │  │  Namespaces     │
  │ (fine-grain │  │  (syscall    │  │  (PID/Net/Mount │
  │  privilege) │  │   filtering) │  │   isolation)    │
  └─────────────┘  └──────────────┘  └─────────────────┘
```

---

## Kernel Source References

| File | Content |
|------|---------|
| `kernel/fork.c` | Process creation |
| `kernel/sched/core.c` | Scheduler core |
| `mm/page_alloc.c` | Buddy page allocator |
| `mm/slub.c` | SLUB slab allocator |
| `fs/namei.c` | VFS path resolution |
| `net/socket.c` | Socket layer |
| `drivers/base/core.c` | Driver model core |
| `security/security.c` | LSM framework |

---

## Interview Questions

**Q1: How does the VFS allow different file systems to coexist?**
A: VFS defines abstract operations (`file_operations`, `inode_operations`, `super_operations`). Each file system implements these callbacks. When user code calls `read()`, VFS dispatches to the correct FS-specific function (e.g., `ext4_file_read()`). This polymorphism lets the kernel support any file system through a single interface.

**Q2: What is the difference between character and block devices?**
A: Character devices transfer data as a stream of bytes (serial ports, sensors, `/dev/null`). Block devices transfer data in fixed-size blocks with random access (disks, eMMC). Block devices go through the block I/O layer with scheduling; character devices go directly through `file_operations`.

**Q3: What are Linux capabilities and why are they important?**
A: Capabilities break the all-or-nothing root privilege into fine-grained permissions (e.g., `CAP_NET_RAW` for raw sockets, `CAP_SYS_ADMIN` for admin operations). This allows processes to have only the specific privileges they need, following the principle of least privilege — much safer than running as full root.

---

## Summary

- Process management: `task_struct`, scheduler (CFS/RT/DL), fork/exec/exit lifecycle
- Memory management: page allocator (buddy), slab allocator (SLUB), demand paging, COW
- Device drivers: char/block/net classification, bus-device-driver model with match
- Interrupt subsystem: controller → generic IRQ layer → handlers, top/bottom half
- VFS: unified file interface, inode/dentry/file abstractions, supports all FS types
- Networking: socket → transport → network → link → driver, with Netfilter hooks
- Security: DAC + LSM (SELinux/AppArmor) + capabilities + seccomp + namespaces

---

*Next: [Chapter 6 — Kernel Space vs User Space](Chapter_06_Kernel_vs_User_Space.md)*
