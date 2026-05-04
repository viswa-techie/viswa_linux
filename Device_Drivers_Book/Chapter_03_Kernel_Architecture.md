# Chapter 3: Linux Kernel Architecture Overview

## Chapter Overview

Before writing drivers, you must understand the kernel's overall architecture — where drivers sit, how they interact with other subsystems, and the execution environment they operate in.

---

## 3.1 Linux Kernel Subsystems Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINUX KERNEL SUBSYSTEMS                      │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Process  │  │ Memory   │  │ Virtual  │  │   Network    │   │
│  │Scheduler │  │ Manager  │  │File Sys  │  │    Stack     │   │
│  │          │  │(mm/)     │  │(fs/)     │  │  (net/)      │   │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│        │            │             │               │             │
│        └────────────┴──────┬──────┴───────────────┘             │
│                            │                                     │
│              ┌─────────────▼──────────────┐                     │
│              │     DRIVER SUBSYSTEM       │                     │
│              │                            │                     │
│              │  char │ block │ net │ plat │                     │
│              │  USB │ PCI │ I2C │ SPI    │                     │
│              │  input │ sound │ video    │                     │
│              └─────────────┬──────────────┘                     │
│                            │                                     │
│              ┌─────────────▼──────────────┐                     │
│              │    ARCH-SPECIFIC LAYER     │                     │
│              │  (arch/arm64, arch/x86)     │                     │
│              │  IRQ, timer, MMU, cache     │                     │
│              └────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### Subsystem Interactions with Drivers

| Subsystem | How It Interacts with Drivers |
|-----------|------------------------------|
| **VFS** | Routes open/read/write to char/block driver via `file_operations` |
| **Memory Manager** | Provides `kmalloc`, `vmalloc`, DMA allocation, page cache |
| **Scheduler** | Manages driver threads, workqueues, completions |
| **Network Stack** | Calls `net_device_ops` for xmit, passes received packets up |
| **Block Layer** | Submits I/O requests to block driver via `request_queue` |
| **IRQ Subsystem** | Dispatches hardware interrupts to driver handlers |
| **Power Manager** | Calls driver PM callbacks (suspend, resume, runtime_pm) |

---

## 3.2 Position of Device Drivers Inside Kernel

```
User Space
───────────────────────────────────────────────
Kernel Space

  System Call Interface (arch/x86/entry/, arch/arm64/kernel/entry.S)
         │
         ▼
  ┌──────────────────────────────┐
  │ Core Frameworks              │
  │  VFS    ← routes to driver  │
  │  net_rx ← routes to driver  │
  │  blk    ← routes to driver  │
  └──────────┬───────────────────┘
             │
  ┌──────────▼───────────────────┐
  │ Driver Core (drivers/base/)  │
  │  bus_type, device, driver    │
  │  class, kobject, sysfs       │
  └──────────┬───────────────────┘
             │
  ┌──────────▼───────────────────┐
  │ Subsystem Drivers            │
  │  drivers/char/               │
  │  drivers/block/              │
  │  drivers/net/                │
  │  drivers/gpu/                │
  │  drivers/i2c/                │
  │  drivers/spi/                │
  │  drivers/usb/                │
  └──────────┬───────────────────┘
             │  readl()/writel()/in*/out*
  ┌──────────▼───────────────────┐
  │ Hardware Abstraction         │
  │  MMIO, PIO, DMA, IRQ        │
  └──────────────────────────────┘
```

---

## 3.3 Kernel Layers Interacting with Drivers

### Call Path: User Read → Character Driver

```
  userspace: read(fd, buf, 4096)
       │
       ▼  (syscall trap)
  ksys_read()                     ← fs/read_write.c
       │
       ▼
  vfs_read()                      ← fs/read_write.c
       │
       ▼
  new_sync_read()                 ← calls f_op->read_iter or f_op->read
       │
       ▼
  my_driver_read(file, buf, count, ppos)    ← YOUR DRIVER
       │
       ▼
  copy_to_user(buf, kernel_buf, count)      ← return data to user
```

### Call Path: I2C Sensor Read

```
  userspace: read(fd, buf, 2)
       │
       ▼
  VFS → char driver read()
       │
       ▼
  i2c_smbus_read_byte_data(client, reg)     ← drivers/i2c/i2c-core-smbus.c
       │
       ▼
  i2c_transfer()                              ← drivers/i2c/i2c-core-base.c
       │
       ▼
  adapter->algo->master_xfer()               ← I2C controller driver
       │
       ▼
  Hardware I2C controller registers           ← writel(), readl()
```

---

## 3.4 Kernel Modules vs Built-in Drivers

### Comparison

| Feature | Built-in (y) | Module (m) |
|---------|-------------|------------|
| When loaded | At boot (part of vmlinux) | Runtime (insmod/modprobe) |
| File | vmlinux binary | .ko file (ELF relocatable) |
| Can be unloaded | No | Yes (rmmod) |
| Init function | Called during boot via `initcall` | Called on `insmod` |
| When to use | Boot-critical (root FS, console) | Everything else |
| Binary size impact | Increases vmlinux | Separate file |
| Configuration | `CONFIG_FOO=y` | `CONFIG_FOO=m` |

### Module Loading Mechanism

```
insmod my_driver.ko
       │
       ▼
  sys_init_module()               ← kernel/module/main.c
       │
       ▼
  load_module()
       │
       ├── Verify ELF format
       ├── Check module signature (if enabled)
       ├── Allocate kernel memory
       ├── Copy sections (.text, .data, .bss, .rodata)
       ├── Resolve symbol references (kallsyms)
       ├── Apply relocations
       └── Call module->init()    ← YOUR module_init() function
              │
              ▼
         Driver registers with bus, creates devices
```

### initcall Levels (for built-in drivers)

```c
/* Built-in drivers use initcall levels to control init order */
pure_initcall(fn)           /* 0 — earliest */
core_initcall(fn)           /* 1 — core subsystems */
postcore_initcall(fn)       /* 2 */
arch_initcall(fn)           /* 3 — architecture setup */
subsys_initcall(fn)         /* 4 — bus/subsystem init */
fs_initcall(fn)             /* 5 — filesystems */
device_initcall(fn)         /* 6 — default (= module_init for built-in) */
late_initcall(fn)           /* 7 — after everything else */
```

---

## 3.5 Kernel Space Execution Environment

### Key Differences from User Space

| Aspect | User Space | Kernel Space |
|--------|-----------|-------------|
| **Address space** | Private per-process | Shared across all kernel code |
| **Stack size** | 8 MB (default) | 16 KB (x86_64), 16 KB (ARM64) |
| **Memory allocation** | malloc (can fail gracefully) | kmalloc (must handle failure) |
| **Sleeping** | Always allowed | Only in process context (not in IRQ) |
| **Preemption** | Always preemptible | CONFIG_PREEMPT dependent |
| **Error handling** | Signal, exit | Must not crash — handle and return error |
| **Floating point** | Available | Forbidden (no FPU context saved) |
| **Standard library** | libc available | No libc — use kernel equivalents |

### Execution Contexts in the Kernel

```
┌─────────────────────────────────────────────────────────────┐
│ PROCESS CONTEXT (can sleep)                                  │
│ ─────────────────────────────                               │
│ • System call handlers (read, write, ioctl)                 │
│ • Workqueue handlers                                        │
│ • Kernel threads (kworker, kswapd)                          │
│ • probe() / remove() functions                              │
│ • Can use: mutex, kmalloc(GFP_KERNEL), msleep()            │
├─────────────────────────────────────────────────────────────┤
│ INTERRUPT CONTEXT (cannot sleep)                             │
│ ─────────────────────────────                               │
│ • Hardirq handler (top half)                                │
│ • Softirq handler                                           │
│ • Tasklet handler                                           │
│ • Timer callback                                            │
│ • Must use: spin_lock, kmalloc(GFP_ATOMIC)                 │
│ • NEVER: mutex_lock, msleep, copy_from_user                │
├─────────────────────────────────────────────────────────────┤
│ ATOMIC CONTEXT (sleeping prohibited)                         │
│ ─────────────────────────────                               │
│ • Inside spinlock critical section                          │
│ • With preemption disabled                                  │
│ • In RCU read-side critical section                         │
└─────────────────────────────────────────────────────────────┘
```

### The Kernel Stack Limitation

```c
/* User stack: 8 MB — can have large local variables */
void user_func(void) {
    char buffer[1048576];   /* 1 MB — fine in user space */
}

/* Kernel stack: 16 KB — MUST be careful */
static int driver_func(void) {
    char buffer[8192];   /* BAD — half the stack! */
    /* Use kmalloc instead: */
    char *buffer = kmalloc(8192, GFP_KERNEL);
    if (!buffer)
        return -ENOMEM;
    /* ... use buffer ... */
    kfree(buffer);
    return 0;
}
```

---

## 3.6 System Call Interface

### How User → Kernel Transition Works

```
User Space                              Kernel Space
─────────                               ────────────
write(fd, buf, len)
    │
    ▼
glibc wrapper:
    movq $1, %rax          ← syscall number for write
    syscall                 ← trap to kernel
                                │
                                ▼
                            entry_SYSCALL_64     ← arch/x86/entry/entry_64.S
                                │
                                ▼
                            sys_call_table[1]    ← points to ksys_write()
                                │
                                ▼
                            ksys_write()
                                │
                                ▼
                            vfs_write() → f_op->write()
                                │
                                ▼
                            YOUR DRIVER's write function
```

### ARM64 System Call Entry
```
User: SVC #0                    ← supervisor call
Kernel: vectors → el0_sync → el0_svc → invoke_syscall()
       → sys_call_table[__NR_write] → ksys_write()
```

### Key System Calls for Drivers

| System Call | Driver Function Called | Purpose |
|-------------|----------------------|---------|
| `open()` | `fops->open()` | Initialize per-file state |
| `read()` | `fops->read()` / `read_iter()` | Transfer data to user |
| `write()` | `fops->write()` / `write_iter()` | Accept data from user |
| `ioctl()` | `fops->unlocked_ioctl()` | Device control |
| `mmap()` | `fops->mmap()` | Map device memory to user |
| `close()` | `fops->release()` | Cleanup per-file state |
| `poll()` | `fops->poll()` | Event readiness check |

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `kernel/module/main.c` | Module loading/unloading |
| `include/linux/init.h` | `module_init()`, `initcall` macros |
| `arch/x86/entry/entry_64.S` | x86_64 syscall entry |
| `arch/arm64/kernel/entry.S` | ARM64 syscall entry |
| `fs/read_write.c` | `vfs_read()`, `vfs_write()` |
| `include/linux/syscalls.h` | System call declarations |
| `init/main.c` | `start_kernel()`, initcall processing |

---

## Debugging: Exploring Kernel Architecture

```bash
# See kernel version
uname -r

# See kernel configuration
zcat /proc/config.gz | grep CONFIG_MODULES

# See all loaded modules
lsmod

# See module dependencies
modprobe --show-depends my_driver

# See initcall order (boot log)
dmesg | grep "initcall"

# See system call table (if kallsyms available)
sudo cat /proc/kallsyms | grep sys_call_table

# Trace a system call
strace -e trace=open,read,write,ioctl ./my_app

# See which driver handles a device
udevadm info /dev/ttyS0
```

---

## OS Comparison

| Aspect | Linux | Windows | macOS | QNX |
|--------|-------|---------|-------|-----|
| Kernel type | Monolithic + modules | Hybrid (NT kernel) | Hybrid (XNU = Mach + BSD) | Microkernel |
| Module format | .ko (ELF) | .sys (PE) | .kext (Mach-O) bundle | Shared lib (.so) |
| Syscall entry | `syscall` (x86), `svc` (ARM) | `ntdll.dll` → `syscall` | Mach trap + BSD syscall | Message passing |
| Driver context | Kernel threads, IRQ, process | IRQL levels (PASSIVE, DISPATCH, DIRQL) | IOWorkLoop event loop | Thread context always |
| Stack size | 16 KB | 12-24 KB (per IRQL) | 16 KB | Thread-dependent |

---

## Interview Questions

**Q1: What execution contexts exist in the Linux kernel, and what can you do in each?**
A: Process context (syscalls, workqueues, kthreads) — can sleep, use mutexes, allocate with GFP_KERNEL. Interrupt context (hardirq, softirq, tasklet) — cannot sleep, must use spinlocks, GFP_ATOMIC. Atomic context (inside spinlock/preempt_disable) — cannot sleep.

**Q2: Why is the kernel stack only 16 KB?**
A: Every thread (including kernel threads) gets its own kernel stack. If stacks were 8 MB like user space, 1000 tasks × 8 MB = 8 GB just for stacks. 16 KB is a trade-off between functionality and memory usage. Drivers must avoid large stack allocations; use kmalloc instead.

**Q3: What happens if a driver calls `mutex_lock()` in interrupt context?**
A: The kernel will sleep the current context waiting for the mutex. But interrupt context cannot sleep — this causes a scheduling-while-atomic bug, leading to a kernel warning/panic. Use `spin_lock_irqsave()` in interrupt context instead.

**Q4: What is an initcall level and why does it matter?**
A: Built-in drivers use `device_initcall()` (level 6) by default. Buses must be registered before devices — bus subsystems use `subsys_initcall()` (level 4). If your driver depends on another subsystem, it must init at the same or later level.

**Q5: How does `modprobe` differ from `insmod`?**
A: `insmod` loads a single .ko file with no dependency resolution. `modprobe` reads `/lib/modules/$(uname -r)/modules.dep` and loads all dependencies first. `modprobe` also handles module aliases (matching devices to modules).

**Q6: Can you use floating-point math in a Linux driver?**
A: No. The kernel does not save/restore FPU registers on kernel entry. Using float/double in a driver corrupts the user process's FPU state. Use `kernel_fpu_begin()/kernel_fpu_end()` if absolutely necessary (very rare, e.g., crypto, RAID XOR).

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Kernel subsystems | Drivers interact with VFS, MM, net stack, block layer, IRQ |
| Driver position | Between core frameworks (VFS/net/block) and hardware |
| Modules vs built-in | Modules (.ko) loaded at runtime; built-in compiled into vmlinux |
| Execution contexts | Process (can sleep) vs interrupt (cannot sleep) vs atomic |
| Kernel stack | 16 KB — never allocate large buffers on stack |
| System calls | User → kernel boundary; VFS routes to driver file_operations |
| initcall levels | Control built-in driver init ordering |

---

*Next: [Chapter 4 — Linux Device Model](Chapter_04_Linux_Device_Model.md)*
