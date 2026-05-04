# Chapter 9: Booting Fundamentals

## Learning Goals
- Understand what system booting is and why it's a multi-stage process
- Map the complete boot stages from power-on to user space
- Understand firmware's role in hardware initialization
- Know what happens at each stage of the boot process

---

## 9.1 What Is System Booting?

**Booting** (bootstrapping) is the process of bringing a computer from a powered-off state to a fully operational state running an operating system. The term comes from "pulling oneself up by one's bootstraps" — each stage loads the next, more capable stage.

```
The Bootstrap Problem:

  To run an OS, you need to load it into memory.
  To load it, you need a program to do the loading.
  To run that program, you need to load IT...
  
  Solution: Hardware provides a FIXED starting point.

  CPU reset vector → tiny ROM code → loads bigger code → loads OS

  Each stage is more capable than the previous:
  ┌──────┐     ┌──────────┐     ┌────────────┐     ┌────────┐
  │ ROM  │ ──► │ Firmware │ ──► │ Bootloader │ ──► │ Kernel │
  │(fixed│     │(HW init, │     │(loads kernel│     │(full OS│
  │ addr)│     │ minimal) │     │ from disk)  │     │ starts)│
  │ tiny │     │  larger  │     │  flexible   │     │ giant  │
  └──────┘     └──────────┘     └────────────┘     └────────┘
  
  Capability grows at each stage.
```

Why multi-stage boot?

| Reason | Explanation |
|--------|-------------|
| **Hardware constraints** | CPU starts with minimal state — can only run from a fixed ROM address |
| **Code size growth** | ROM is tiny (KB) → firmware is medium (MB) → kernel is large (MB-GB) |
| **Abstraction layers** | Each stage initializes hardware needed by the next stage |
| **Flexibility** | Bootloader can load different kernels, parameters, DTBs |
| **Security** | Each stage can verify the next (secure boot chain) |

---

## 9.2 Boot Process Overview

```
Complete Boot Process — All Architectures:

Power ON / Reset
     │
     ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 0: Hardware Reset                              │
│ - CPU starts at reset vector (fixed address)         │
│ - x86: 0xFFFFFFF0 (top of 4GB, mapped to BIOS ROM) │
│ - ARM: 0x00000000 or 0xFFFF0000 (configurable)     │
│ - RISC-V: Implementation defined                     │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 1: Firmware / Boot ROM                         │
│ - Embedded: SoC Boot ROM (mask ROM, not modifiable) │
│ - x86 PC: BIOS / UEFI firmware (on SPI flash)      │
│ - Initialize: clocks, DRAM controller, basic I/O     │
│ - Load next stage from boot media                    │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 2: Bootloader (SPL / U-Boot / GRUB)           │
│ - Initialize more hardware (display, USB, network)  │
│ - Read kernel image from storage (eMMC, NFS, TFTP) │
│ - Read device tree blob (DTB) — embedded only       │
│ - Read initramfs (if used)                          │
│ - Pass kernel command line parameters               │
│ - Jump to kernel entry point                         │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 3: Kernel Initialization                       │
│ - Decompress kernel (if zImage/bzImage)             │
│ - Initialize CPU, MMU, page tables                  │
│ - start_kernel() → subsystem initialization         │
│ - Mount root filesystem                             │
│ - Launch init process (PID 1)                       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│ STAGE 4: User Space Initialization                   │
│ - init / systemd starts                             │
│ - Mount filesystems, start services                 │
│ - Network configuration                            │
│ - Login prompt / GUI / Android framework            │
└─────────────────────────────────────────────────────┘
```

---

## 9.3 Boot Stages in Modern Systems

### x86 PC Boot Flow

```
x86 PC Boot Sequence:

Power ON → UEFI firmware (SPI flash)
              │
              ├─ POST (Power-On Self-Test)
              ├─ DRAM init (memory training)
              ├─ PCIe enumeration
              ├─ Boot device selection
              │
              ▼
         GRUB bootloader (from ESP partition)
              │
              ├─ Display boot menu
              ├─ Load vmlinuz
              ├─ Load initramfs
              ├─ Pass command line
              │
              ▼
         Linux Kernel → systemd → Desktop
```

### ARM/Embedded Boot Flow

```
ARM SoC Boot Sequence:

Power ON → Boot ROM (SoC internal, mask ROM)
              │
              ├─ Read boot pins / eFuses
              ├─ Determine boot media (eMMC, SD, NAND, UART)
              ├─ Load SPL (Secondary Program Loader)
              │
              ▼
         SPL / U-Boot SPL (from boot media)
              │
              ├─ DRAM controller initialization
              ├─ Load full U-Boot
              │
              ▼
         U-Boot (full bootloader)
              │
              ├─ Load kernel (Image/zImage)
              ├─ Load DTB (device tree blob)
              ├─ Load initramfs (optional)
              ├─ Boot command: bootm / booti
              │
              ▼
         Linux Kernel → init / Android init
```

### Android Boot Flow (Qualcomm SoC)

```
Qualcomm Android Boot:

Power ON → PBL (Primary Boot Loader, ROM)
              │
              ├─ Reads boot config from eFuses
              ├─ Loads XBL (eXtensible Boot Loader) from UFS
              │
              ▼
         XBL (replaces SBL, UEFI-based)
              │
              ├─ DRAM init, clock init
              ├─ Loads QTEE (TrustZone)
              ├─ Loads ABL (Android Boot Loader)
              │
              ▼
         ABL (Android Bootloader)
              │
              ├─ A/B slot selection
              ├─ Verified boot (dm-verity)
              ├─ Loads boot.img (kernel + ramdisk + DTB)
              │
              ▼
         Linux Kernel → Android init → Zygote → System Server
```

---

## 9.4 Firmware Initialization

What firmware does before the bootloader:

```
Firmware Initialization Sequence:

1. CPU Clock Setup
   └── PLL configuration, CPU frequency ramp

2. Memory Controller Init
   └── DRAM training (timing calibration)
   └── Memory test (optional)

3. Basic I/O Setup
   └── UART for debug console
   └── GPIO initial states

4. Boot Media Access
   └── eMMC/UFS/SPI/SD controller init
   └── Read boot partition

5. Security Initialization
   └── Trusted execution environment (TEE/TrustZone)
   └── Secure boot chain verification
   └── eFuse-based authentication

6. Load Next Stage
   └── Verify signature (secure boot)
   └── Copy to DRAM
   └── Jump to entry point
```

---

## 9.5 Hardware Initialization During Boot

Each boot stage initializes different hardware:

| Stage | Hardware Initialized | Why Now |
|-------|---------------------|---------|
| Boot ROM | CPU core, basic clocks, boot media | Minimum to load next stage |
| Firmware/SPL | DRAM, PLLs, power rails | Need RAM for bootloader |
| Bootloader | UART, display, USB, network, storage | Need to load kernel, user interaction |
| Kernel early | MMU, page tables, interrupt controller | Need virtual memory, interrupt handling |
| Kernel drivers | All remaining: GPU, audio, sensors, etc. | Full device support |

```
Progressive Hardware Initialization:

Boot ROM    ████░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~5% of hardware
Firmware    ████████░░░░░░░░░░░░░░░░░░░░░░░░  ~15%
Bootloader  ████████████████░░░░░░░░░░░░░░░░  ~30%
Kernel init ████████████████████████░░░░░░░░  ~60%
All drivers ████████████████████████████████  100%
```

---

## Interview Questions

**Q1: Why can't the CPU directly load and run the Linux kernel at power-on?**
A: At power-on, the CPU starts executing from a fixed ROM address with no DRAM initialized, no file system, and minimal hardware. The kernel is too large for ROM and needs DRAM. The boot process progressively initializes hardware: ROM → DRAM init → load bootloader → load kernel.

**Q2: What is the difference between booting on x86 PC vs ARM embedded?**
A: x86 PCs have standardized UEFI firmware on SPI flash that handles DRAM init, PCIe, and can access disks. ARM embedded uses SoC-specific Boot ROM → SPL → U-Boot chain, with device trees for hardware description instead of ACPI. ARM requires board-specific customization; x86 is standardized.

**Q3: What is a secure boot chain?**
A: Each boot stage cryptographically verifies the next stage before executing it. Boot ROM verifies firmware signature → firmware verifies bootloader → bootloader verifies kernel. The root of trust is in hardware (eFuses, ROM). If any stage fails verification, boot halts, preventing unauthorized code execution.

---

## Summary

- Booting is a multi-stage process: ROM → firmware → bootloader → kernel → user space
- Each stage initializes hardware needed by the next and is more capable than the previous
- x86 uses UEFI → GRUB; ARM embedded uses Boot ROM → SPL → U-Boot
- Android (Qualcomm) uses PBL → XBL → ABL with verified boot
- Firmware initializes clocks, DRAM, and basic I/O before loading the bootloader
- Hardware initialization is progressive — each stage adds more device support

---

*Next: [Chapter 10 — Firmware Stage](Chapter_10_Firmware_Stage.md)*
