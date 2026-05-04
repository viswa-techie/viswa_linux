# Chapter 6: Kernel Space vs User Space

## Learning Goals
- Understand how address space separation works at hardware and kernel level
- Deep-dive into the system call interface and mechanism
- Know the kernel APIs available to kernel-space code
- Understand all user-kernel communication mechanisms

---

## 6.1 Address Space Separation

The kernel maintains strict separation between kernel and user address spaces. This is enforced by hardware (MMU) and is the foundation of system security and stability.

```
64-bit Linux Virtual Address Space:

0xFFFFFFFFFFFFFFFF ┌───────────────────────────────┐
                   │  Fixmap, KASAN shadow          │
                   │  vmemmap (struct page array)    │
                   │  vmalloc / ioremap space        │
                   │  Direct map (all physical RAM)  │
                   │  Kernel text + data + BSS       │
                   │  Module space                   │
                   │  (All shared by every process)  │
0xFFFF000000000000 ├───────────────────────────────┤  ← Kernel/User split
                   │                               │
                   │   Non-canonical hole           │  ← Hardware enforced gap
                   │   (accessing here = fault)     │     Sign extension gap
                   │                               │
0x0000FFFFFFFFFFFF ├───────────────────────────────┤
                   │                               │
                   │   USER SPACE                  │
                   │                               │
                   │   Stack       (grows ↓)       │  ← Top of user space
                   │   ...                         │
                   │   mmap region (libs, shm)     │
                   │   ...                         │
                   │   Heap        (grows ↑)       │  ← brk() grows this
                   │   BSS         (zero-init)     │
                   │   Data        (initialized)   │
                   │   Text        (code, r-x)     │  ← ELF loaded here
                   │                               │
0x0000000000000000 └───────────────────────────────┘
```

How the kernel protects its memory:

```c
/* Page table entries have permission bits */

/* Kernel page: */
PTE = physical_addr | PTE_VALID | PTE_KERNEL | PTE_RW | PTE_NX
/*                                 ^^^^^^^^
   PTE_KERNEL = accessible only from Ring 0 / EL1
   User mode access → page fault → SIGSEGV */

/* User page: */
PTE = physical_addr | PTE_VALID | PTE_USER | PTE_RW
/*                                 ^^^^^^^^
   PTE_USER = accessible from Ring 3 / EL0 AND Ring 0
   But SMAP/PAN prevents kernel from accidentally accessing */
```

Hardware protection features:

| Feature | Architecture | Purpose |
|---------|-------------|---------|
| **SMEP** (Supervisor Mode Execution Prevention) | x86 | Kernel cannot execute user-space code |
| **SMAP** (Supervisor Mode Access Prevention) | x86 | Kernel cannot read/write user memory directly |
| **PAN** (Privileged Access Never) | ARM64 | Same as SMAP — kernel can't directly access user pages |
| **PXN** (Privileged Execute Never) | ARM64 | Same as SMEP — kernel can't execute user code |
| **KASLR** | Both | Randomize kernel address layout |
| **KPTI** (Kernel Page Table Isolation) | Both | Separate page tables for user/kernel (Meltdown fix) |

---

## 6.2 System Call Interface

System calls are the **only** legitimate way for user-space code to request kernel services.

```
System Call Mechanism (x86_64):

User Space                              Kernel Space
┌──────────────────┐                   ┌──────────────────────┐
│                  │                   │                      │
│  write(1, "hi",2)│                   │                      │
│       │          │                   │                      │
│  glibc wrapper:  │                   │                      │
│   mov $1, %rax   │  (syscall #)      │                      │
│   mov $1, %rdi   │  (fd)             │                      │
│   lea "hi", %rsi │  (buf)            │                      │
│   mov $2, %rdx   │  (count)          │                      │
│   syscall        │  ──────────────►  │  entry_SYSCALL_64    │
│                  │  (Ring 3 → Ring 0)│   │                  │
│       waits...   │                   │   ▼                  │
│                  │                   │  sys_call_table[1]   │
│                  │                   │   = ksys_write()     │
│                  │                   │   │                  │
│                  │                   │   ▼                  │
│                  │                   │  VFS → driver → HW  │
│                  │                   │   │                  │
│                  │  ◄────────────────│   return             │
│   return value   │  (Ring 0 → Ring 3)│  (sysret)           │
│   in %rax        │                   │                      │
└──────────────────┘                   └──────────────────────┘


System Call Mechanism (ARM64):

User Space                              Kernel Space
┌──────────────────┐                   ┌──────────────────────┐
│                  │                   │                      │
│  Bionic wrapper: │                   │                      │
│   mov x8, #64    │  (syscall # for   │                      │
│   mov x0, #1     │   write on arm64) │                      │
│   adr x1, "hi"   │                   │                      │
│   mov x2, #2     │                   │                      │
│   svc #0         │  ──────────────►  │  el0_svc handler    │
│                  │  (EL0 → EL1)      │   │                  │
│                  │                   │   ▼                  │
│                  │                   │  sys_call_table[64]  │
│                  │                   │   = ksys_write()     │
│                  │  ◄────────────────│   eret               │
│                  │  (EL1 → EL0)      │                      │
└──────────────────┘                   └──────────────────────┘
```

System call table organization:

```c
/* arch/x86/entry/syscall_64.c */
const sys_call_ptr_t sys_call_table[] = {
    [0]   = __x64_sys_read,
    [1]   = __x64_sys_write,
    [2]   = __x64_sys_open,
    [3]   = __x64_sys_close,
    /* ... ~450 system calls */
    [435] = __x64_sys_cachestat,
};
```

Common system calls:

| # (x86_64) | Name | Purpose |
|------------|------|---------|
| 0 | `read` | Read from file descriptor |
| 1 | `write` | Write to file descriptor |
| 2 | `open` | Open file |
| 3 | `close` | Close file descriptor |
| 9 | `mmap` | Map memory |
| 56 | `clone` | Create process/thread |
| 57 | `fork` | Create child process |
| 59 | `execve` | Execute program |
| 60 | `exit` | Terminate process |
| 62 | `kill` | Send signal |
| 16 | `ioctl` | Device control |

---

## 6.3 Kernel APIs

The kernel provides APIs for internal use by kernel code (drivers, modules, subsystems):

```c
/* Memory allocation */
void *kmalloc(size_t size, gfp_t flags);      /* General allocation */
void *kzalloc(size_t size, gfp_t flags);      /* Zero-filled */
void kfree(const void *ptr);                   /* Free */
void *vmalloc(unsigned long size);             /* Virtually contiguous */
unsigned long __get_free_pages(gfp_t, order);  /* Page-aligned */

/* String / memory operations */
void *memcpy(void *dst, const void *src, size_t n);
void *memset(void *s, int c, size_t n);
int strcmp(const char *s1, const char *s2);
size_t strlen(const char *s);
int snprintf(char *buf, size_t size, const char *fmt, ...);

/* Synchronization */
DEFINE_SPINLOCK(my_lock);
spin_lock(&my_lock);                      spin_unlock(&my_lock);
DEFINE_MUTEX(my_mutex);
mutex_lock(&my_mutex);                    mutex_unlock(&my_mutex);

/* User-space data transfer */
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
int get_user(x, ptr);                    /* Read single value */
int put_user(x, ptr);                    /* Write single value */

/* Logging */
printk(KERN_INFO "msg: %d\n", val);
pr_info("msg: %d\n", val);              /* Preferred */
dev_info(dev, "msg: %d\n", val);         /* Device-context */
```

---

## 6.4 User-Kernel Communication Mechanisms

```
User ↔ Kernel Communication Methods:

┌──────────────────────────────────────────────────────────┐
│  1. SYSTEM CALLS                                         │
│     read(), write(), ioctl(), mmap()                     │
│     Primary mechanism, ~450 calls                        │
├──────────────────────────────────────────────────────────┤
│  2. /proc FILESYSTEM                                     │
│     /proc/cpuinfo, /proc/meminfo, /proc/[pid]/status    │
│     Read-only process/system information                 │
├──────────────────────────────────────────────────────────┤
│  3. /sys FILESYSTEM (sysfs)                              │
│     /sys/class/leds/red/brightness                       │
│     Device/driver attributes, read-write                 │
├──────────────────────────────────────────────────────────┤
│  4. /dev DEVICE FILES                                    │
│     open() + read()/write()/ioctl()                      │
│     Access device drivers                                │
├──────────────────────────────────────────────────────────┤
│  5. NETLINK SOCKETS                                      │
│     socket(AF_NETLINK, ...)                              │
│     Async kernel-to-user notifications (udev, iproute2) │
├──────────────────────────────────────────────────────────┤
│  6. ioctl()                                              │
│     Device-specific commands via file descriptors        │
│     Flexible but type-unsafe                             │
├──────────────────────────────────────────────────────────┤
│  7. mmap()                                               │
│     Map device memory or kernel buffers to user space    │
│     Zero-copy data transfer                              │
├──────────────────────────────────────────────────────────┤
│  8. SIGNALS                                              │
│     Kernel → User asynchronous notifications             │
│     SIGKILL, SIGSEGV, SIGCHLD, etc.                     │
├──────────────────────────────────────────────────────────┤
│  9. DEBUGFS                                              │
│     /sys/kernel/debug/                                   │
│     Debug-only interfaces, not for production            │
├──────────────────────────────────────────────────────────┤
│  10. eBPF                                                │
│      User-space programs run safely in kernel            │
│      Tracing, networking, security                       │
└──────────────────────────────────────────────────────────┘
```

```c
/* Example: sysfs attribute — user reads/writes via /sys/... */

static ssize_t brightness_show(struct device *dev,
                               struct device_attribute *attr, char *buf)
{
    struct my_led *led = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", led->brightness);
}

static ssize_t brightness_store(struct device *dev,
                                struct device_attribute *attr,
                                const char *buf, size_t count)
{
    struct my_led *led = dev_get_drvdata(dev);
    int val;
    if (kstrtoint(buf, 10, &val))
        return -EINVAL;
    led->brightness = val;
    return count;
}

static DEVICE_ATTR_RW(brightness);
/* Creates: /sys/class/leds/my-led/brightness */
```

---

## Interview Questions

**Q1: What happens when a user-space program dereferences a kernel address?**
A: The MMU detects the access violation (PTE has kernel-only permission), generates a page fault, and the kernel delivers `SIGSEGV` to the process, killing it. On ARM64 with PAN enabled, even the kernel itself can't accidentally access user memory without using `copy_from_user()`.

**Q2: Why does `copy_from_user()` exist instead of direct pointer dereference?**
A: User pointers may be invalid, unmapped, or deliberately crafted to access kernel memory. `copy_from_user()`: (1) verifies the address range is in user space, (2) uses special faulting instructions that the kernel can recover from, (3) prevents the kernel from accessing its own memory through user-supplied pointers (SMAP/PAN).

**Q3: What is the difference between ioctl and sysfs for user-kernel communication?**
A: `ioctl()` passes arbitrary commands through file descriptors — flexible but opaque and hard to document. sysfs exposes individual attributes as files — each attribute is one value, self-documenting (path = meaning), and works with shell scripts. Modern kernel drivers prefer sysfs for simple values and ioctl for complex struct-based commands.

**Q4: How does Netlink differ from regular system calls?**
A: System calls are synchronous (user waits for result). Netlink sockets provide asynchronous, message-based communication — the kernel can send notifications to user space without being asked. Used by udev (device events), iproute2 (network config), and audit subsystem.

---

## Summary

- Linux separates virtual address space: kernel in upper half, user in lower half, with a non-canonical gap
- Hardware (SMEP/SMAP/PAN) prevents accidental or malicious cross-boundary access
- System calls are the only controlled entry from user to kernel mode
- The kernel provides its own APIs (kmalloc, printk, spinlock) — no libc in kernel space
- `copy_from_user()`/`copy_to_user()` safely transfer data across the boundary
- Communication methods: syscalls, /proc, /sys, /dev, netlink, ioctl, mmap, signals, debugfs, eBPF

---

*Next: [Chapter 7 — Kernel Data Structures](Chapter_07_Kernel_Data_Structures.md)*
