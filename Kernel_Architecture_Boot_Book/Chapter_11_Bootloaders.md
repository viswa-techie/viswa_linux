# Chapter 11: Bootloaders

## Learning Goals
- Understand the bootloader concept and why it exists
- Know the bootloader's role in the Linux boot process
- Understand multi-stage bootloader design
- Grasp bootloader responsibilities in detail

---

## 11.1 Bootloader Concept

A **bootloader** is a program that loads and starts the operating system kernel. It bridges the gap between firmware (which can only run simple code) and the kernel (which needs a specific environment to start).

```
Why Bootloaders Exist:

Firmware knows HOW to read storage, but NOT what to load.
The kernel is a complex binary that needs:
  - To be loaded at a specific memory address
  - CPU in a specific state (MMU off, interrupts off)
  - Hardware information (memory map, DTB)
  - Configuration (command line parameters)

The bootloader handles all of this.

┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  Firmware    │         │  Bootloader  │         │   Kernel    │
│             │         │              │         │             │
│ "I know how │  ──►    │"I know WHERE │  ──►    │"I need DTB, │
│  to read    │         │ the kernel   │         │ command line│
│  the boot   │         │ is and how   │         │ and correct │
│  media"     │         │ to set up    │         │ CPU state"  │
│             │         │ for it"      │         │             │
└─────────────┘         └──────────────┘         └─────────────┘
```

---

## 11.2 Role of Bootloaders in Linux Systems

```
Bootloader's Critical Jobs:

1. LOAD the kernel image into RAM
   ├── From local storage (eMMC, NVMe, SD card)
   ├── From network (TFTP, HTTP, NFS)
   └── From USB, serial, or other media

2. LOAD the Device Tree Blob (DTB) — embedded systems
   └── Hardware description for the kernel

3. LOAD initial ramdisk (initramfs/initrd)
   └── Early user-space with essential drivers

4. PASS boot parameters to kernel
   └── console=ttyMSM0,115200 root=/dev/sda2 ...

5. SET UP required CPU/hardware state
   ├── CPU registers to defined values
   ├── MMU off (ARM) — kernel will set it up
   ├── Interrupts disabled
   └── Cache state as required

6. JUMP to kernel entry point
   └── Never returns — kernel takes over completely
```

---

## 11.3 Bootloader Stages

Many embedded systems use a two-stage bootloader design:

```
Two-Stage Bootloader (Embedded):

┌──────────────────────────────────────────────────┐
│  Boot ROM (SoC internal)                          │
│  - Reads boot pins to determine media            │
│  - Limited SRAM available (64-256 KB)             │
│  - Loads SPL from boot media into SRAM            │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│  Stage 1: SPL (Secondary Program Loader)          │
│                                                  │
│  Size: 50-200 KB (fits in SRAM)                  │
│  Runs from: SRAM (before DRAM is ready)          │
│                                                  │
│  Jobs:                                           │
│  ├── Initialize DRAM controller                  │
│  ├── Initialize clocks / power                   │
│  ├── Minimal hardware init                       │
│  └── Load full bootloader into DRAM              │
└───────────────────────┬──────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────┐
│  Stage 2: Full Bootloader (U-Boot)                │
│                                                  │
│  Size: 500 KB - 2 MB                             │
│  Runs from: DRAM                                 │
│                                                  │
│  Jobs:                                           │
│  ├── Full hardware initialization                │
│  ├── Interactive shell (optional)                │
│  ├── Network boot (TFTP, NFS)                    │
│  ├── Load kernel + DTB + initramfs               │
│  ├── Boot scripting (boot.scr)                   │
│  └── Jump to kernel                              │
└──────────────────────────────────────────────────┘
```

---

## 11.4 Bootloader Responsibilities

### Setting Up Kernel Boot Requirements

```
ARM64 Kernel Entry Requirements (documented in kernel):
(Documentation/arm64/booting.rst)

Register Setup:
  x0 = physical address of device tree blob (DTB)
  x1 = 0 (reserved)
  x2 = 0 (reserved)
  x3 = 0 (reserved)

CPU State:
  - All cores except primary: held in firmware or spin-table
  - MMU: OFF
  - D-cache: OFF (or clean)
  - I-cache: ON or OFF
  - Interrupts: Masked (PSTATE.DAIF all set)
  - Exception level: EL2 (if hypervisor) or EL1
  - Little-endian mode

Memory:
  - Kernel image loaded at TEXT_OFFSET from DRAM base
    (2 MB aligned, typically 0x80080000 for many SoCs)
  - DTB in memory (must not overlap kernel)
  - initramfs in memory (if provided)
```

```
x86_64 Kernel Entry Requirements:

For bzImage (Linux Boot Protocol):
  - Kernel loaded per boot protocol (header at offset 0x1F1)
  - Protected mode or long mode (64-bit)
  - Boot parameters in struct boot_params
  - GDT set up
  - Interrupts disabled

For EFI Stub:
  - UEFI application loaded by firmware
  - Kernel calls ExitBootServices() itself
  - EFI system table pointer passed
```

### Memory Layout at Boot Time

```
Memory Layout When Bootloader Passes Control to Kernel (ARM64):

0x00000000  ┌─────────────────────────────┐
            │  (Reserved / peripheral)     │
0x40000000  ├─────────────────────────────┤  ← DRAM start (example)
            │  U-Boot code & data          │
            │  (bootloader will be         │
            │   overwritten eventually)    │
            ├─────────────────────────────┤
            │                             │
            │  Kernel Image               │ ← Loaded by bootloader
            │  (Image file, ~20-30 MB)    │
            │                             │
            ├─────────────────────────────┤
            │  Device Tree Blob (DTB)     │ ← Loaded by bootloader
            │  (50-200 KB)                │
            ├─────────────────────────────┤
            │  initramfs                  │ ← Loaded by bootloader
            │  (optional, varies)         │    (if specified)
            ├─────────────────────────────┤
            │                             │
            │  Free memory                │ ← Kernel will manage this
            │                             │
0xBFFFFFFF  └─────────────────────────────┘  ← DRAM end (2 GB example)
```

---

## Interview Questions

**Q1: Why do embedded systems need a two-stage bootloader?**
A: Boot ROM can only load a small program into limited SRAM (64-256 KB) — DRAM isn't initialized yet. SPL (Stage 1) fits in SRAM, initializes DRAM, then loads the full bootloader into DRAM. The full bootloader (Stage 2) has all features: network boot, scripting, device tree loading.

**Q2: What CPU state must the bootloader set before jumping to the ARM64 kernel?**
A: x0 = DTB physical address, x1-x3 = 0. MMU off, D-cache off or clean, interrupts masked (PSTATE.DAIF), EL2 or EL1 exception level, little-endian mode. Secondary CPUs held in spin-table or PSCI. Violating these requirements causes undefined behavior or immediate crash.

**Q3: Can a bootloader load the kernel from a network?**
A: Yes. Bootloaders support multiple boot sources: TFTP (U-Boot's `tftpboot`), NFS root, HTTP (UEFI HTTP Boot), PXE. This is essential for development (quick kernel iteration) and diskless systems (factory, thin clients).

---

## Summary

- Bootloaders bridge firmware and kernel — loading the kernel with correct parameters and CPU state
- Embedded systems use two-stage bootloaders: SPL (runs from SRAM, inits DRAM) → full bootloader (runs from DRAM)
- The bootloader loads kernel image, DTB, and initramfs into defined memory locations
- ARM64 kernel expects: x0=DTB, MMU off, interrupts disabled, EL2/EL1
- Bootloaders support multiple boot sources: storage, network, USB, serial

---

*Next: [Chapter 12 — Common Bootloaders](Chapter_12_Common_Bootloaders.md)*
