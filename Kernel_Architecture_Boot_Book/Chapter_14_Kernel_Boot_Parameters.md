# Chapter 14: Kernel Boot Parameters

## Learning Goals
- Understand how kernel command line parameters work
- Know the most important boot parameters and their effects
- Understand how parameters are passed from bootloader to kernel
- Grasp boot configuration for different scenarios

---

## 14.1 Kernel Command Line Parameters

The kernel command line is a string of parameters passed by the bootloader that configures kernel behavior at boot time.

```
How Boot Parameters Flow:

Bootloader                    Kernel
┌──────────────────────┐     ┌──────────────────────────────┐
│ setenv bootargs      │     │ start_kernel()               │
│ "console=ttyS0,115200│ ──► │   setup_arch()               │
│  root=/dev/mmcblk0p2 │     │     parse_early_param()      │
│  rootwait rw         │     │       setup_command_line()    │
│  loglevel=7"         │     │         parse each parameter  │
└──────────────────────┘     └──────────────────────────────┘

Parameters are accessible:
  - Kernel: saved_command_line, boot_command_line
  - User: /proc/cmdline
  - DT: /chosen/bootargs node
```

```bash
# View current kernel command line
cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-6.8.0 root=UUID=xxxx ro quiet splash

# Parameters are KEY=VALUE or just KEY (boolean flags)
# Parsed by kernel and passed to matching subsystems/drivers
```

---

## 14.2 Essential Boot Parameters

### Console and Debug Parameters

| Parameter | Effect | Example |
|-----------|--------|---------|
| `console=` | Kernel console output device | `console=ttyS0,115200` |
| `earlycon=` | Early console before driver init | `earlycon=uart,0x09000000` |
| `earlyprintk` | Enable early debug output | `earlyprintk=serial,ttyS0` |
| `loglevel=` | Kernel log level (0-7) | `loglevel=7` (all messages) |
| `quiet` | Suppress most boot messages | `quiet` |
| `debug` | Enable debug-level messages | `debug` |
| `ignore_loglevel` | Print ALL messages regardless | `ignore_loglevel` |

### Root Filesystem Parameters

| Parameter | Effect | Example |
|-----------|--------|---------|
| `root=` | Root filesystem device | `root=/dev/sda2` |
| `rootfstype=` | Force root FS type | `rootfstype=ext4` |
| `rootwait` | Wait for root device (essential for USB/MMC) | `rootwait` |
| `rootdelay=` | Wait N seconds for root device | `rootdelay=5` |
| `rw` / `ro` | Mount root read-write / read-only | `rw` |
| `init=` | Override init process | `init=/bin/bash` |

### Memory Parameters

| Parameter | Effect | Example |
|-----------|--------|---------|
| `mem=` | Limit usable memory | `mem=512M` |
| `memmap=` | Reserve/mark memory regions | `memmap=64M$0x100000000` |
| `hugepages=` | Pre-allocate huge pages | `hugepages=512` |
| `cma=` | Contiguous Memory Allocator size | `cma=128M` |
| `vmalloc=` | vmalloc area size (32-bit) | `vmalloc=256M` |

### CPU and Scheduling

| Parameter | Effect | Example |
|-----------|--------|---------|
| `maxcpus=` | Limit online CPUs | `maxcpus=4` |
| `nosmp` | Disable SMP (single CPU) | `nosmp` |
| `isolcpus=` | Isolate CPUs from scheduler | `isolcpus=2,3` |
| `nohz_full=` | Tickless CPUs for RT | `nohz_full=2,3` |
| `rcu_nocbs=` | Offload RCU callbacks | `rcu_nocbs=2,3` |

### Security Parameters

| Parameter | Effect | Example |
|-----------|--------|---------|
| `selinux=` | Enable/disable SELinux | `selinux=1` |
| `enforcing=` | SELinux enforcing mode | `enforcing=0` (permissive) |
| `apparmor=` | Enable AppArmor | `apparmor=1` |
| `nokaslr` | Disable kernel ASLR | `nokaslr` (debug only!) |
| `lockdown=` | Kernel lockdown mode | `lockdown=confidentiality` |

### Common Boot Scenarios

```bash
# Development / Debug
bootargs="console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootwait \
          loglevel=7 ignore_loglevel initcall_debug printk.devkmsg=on"

# Production / Android
bootargs="console=ttyMSM0,115200n8 androidboot.hardware=qcom \
          root=/dev/dm-0 rootwait ro selinux=1 enforcing=1 \
          androidboot.verifiedbootstate=green"

# Rescue / Emergency
bootargs="root=/dev/sda2 ro single init=/bin/bash"

# Network root (NFS)
bootargs="console=ttyS0,115200 root=/dev/nfs nfsroot=192.168.1.1:/nfs/rootfs \
          ip=192.168.1.100::192.168.1.1:255.255.255.0::eth0:off"

# Minimal boot (fast boot optimization)
bootargs="console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootwait quiet \
          lpj=1000000 raid=noautodetect"
```

---

## 14.3 Passing Parameters to Kernel

### From Bootloader

```bash
# U-Boot: via environment variable
=> setenv bootargs "console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootwait"
=> booti $kernel_addr - $dtb_addr

# GRUB: via grub.cfg
linux /vmlinuz root=/dev/sda2 ro quiet splash

# UEFI: in boot entry
efibootmgr -c -L "Linux" -l '\vmlinuz' \
    -u "root=UUID=xxxx ro quiet"
```

### From Device Tree

```dts
/* Device tree can also specify boot parameters */
/ {
    chosen {
        bootargs = "console=ttyS0,115200 earlycon";
        stdout-path = "serial0:115200n8";
    };
};
```

### Built-in Command Line

```c
/* Compile-time default command line */
/* Set via CONFIG_CMDLINE in kernel config */

CONFIG_CMDLINE_BOOL=y
CONFIG_CMDLINE="console=ttyS0,115200 root=/dev/mmcblk0p2"

/* CONFIG_CMDLINE_EXTEND: append to bootloader args */
/* CONFIG_CMDLINE_FORCE: ignore bootloader args, use only this */
```

### How Kernel Parses Parameters

```c
/* Kernel parameter parsing mechanism */

/* 1. Early parameters — parsed first (before most subsystems) */
early_param("earlycon", setup_earlycon);
early_param("console", setup_console);
early_param("mem", parse_memopt);

/* 2. Regular parameters — parsed by __setup or module_param */
__setup("root=", root_dev_setup);      /* Core parameters */
__setup("init=", init_setup);

/* 3. Module parameters (also work as boot params for built-in) */
module_param(debug_level, int, 0644);
/* Boot: my_driver.debug_level=3 */

/* Parsing flow in start_kernel(): */
void __init start_kernel(void)
{
    setup_arch(&command_line);           /* Arch gets command line */
    setup_command_line(command_line);     /* Save copies */
    parse_early_param();                 /* Parse early_param() */
    /* ... */
    parse_args("Booting kernel", ...);   /* Parse __setup(), module_param */
}
```

---

## Interview Questions

**Q1: What is the difference between `rootwait` and `rootdelay`?**
A: `rootwait` waits indefinitely until the root device appears — essential for USB, eMMC, and SD cards that need driver initialization time. `rootdelay=N` waits exactly N seconds then tries. `rootwait` is preferred because it adapts to variable hardware initialization times; `rootdelay` wastes time if the device appears early or fails if it's too short.

**Q2: How can you debug a kernel that won't boot?**
A: Add `earlycon=<type>,<addr>` for early serial output before normal console init. Add `earlyprintk`, `loglevel=7`, `ignore_loglevel`, and `initcall_debug` to see all initialization. Use `init=/bin/bash` to skip init and drop to a shell. If display works, `console=tty0` shows messages. Remove `quiet` and `splash` parameters.

**Q3: What does `init=/bin/bash` do and when is it useful?**
A: It tells the kernel to run `/bin/bash` as PID 1 instead of `/sbin/init` or systemd. You get a root shell immediately after kernel init. Useful for: recovering a system with broken init, debugging startup issues, filesystem repair, password recovery. Note: no services start, and you need to mount filesystems manually.

---

## Summary

- The kernel command line configures boot-time behavior: console, root FS, memory, CPU, security
- Parameters flow from bootloader (bootargs) or device tree (/chosen/bootargs) to kernel
- `rootwait` is essential for removable/slow storage; `init=` can override the init process
- Debug boot: use `earlycon`, `loglevel=7`, `initcall_debug`, `init=/bin/bash`
- Parameters are parsed by `early_param()` (first) and `__setup()` (later)
- Built-in command line (`CONFIG_CMDLINE`) provides a fallback or override

---

*Next: [Chapter 15 — Initial RAM Filesystem](Chapter_15_Initial_RAM_Filesystem.md)*
