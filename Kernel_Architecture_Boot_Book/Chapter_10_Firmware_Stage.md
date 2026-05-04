# Chapter 10: Firmware Stage

## Learning Goals
- Understand BIOS and UEFI firmware architectures
- Know what firmware is responsible for during boot
- Understand firmware hardware initialization steps
- Grasp firmware boot services and runtime services

---

## 10.1 BIOS Architecture

**BIOS** (Basic Input/Output System) is the legacy x86 firmware. While being replaced by UEFI, understanding BIOS is important for historical context.

```
BIOS Boot Flow:

CPU Reset Vector: 0xFFFFFFF0
        │
        ▼
┌────────────────────────────────┐
│  BIOS ROM (SPI Flash)          │
│                                │
│  1. POST (Power-On Self-Test)  │
│     - CPU self-test            │
│     - Memory detection & test  │
│     - Keyboard controller init │
│                                │
│  2. Hardware Init              │
│     - Interrupt vector table   │
│     - PCI device enumeration   │
│     - Video BIOS (VGA init)    │
│                                │
│  3. Boot Device Selection      │
│     - Check boot order         │
│     - Read MBR (first 512 bytes│
│       of boot disk)            │
│                                │
│  4. Load MBR → Jump to it      │
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│  MBR (Master Boot Record)      │
│  512 bytes:                    │
│  ┌──────────────────┐          │
│  │ Bootstrap code   │ 446 bytes│
│  │ (stage 1 loader) │          │
│  ├──────────────────┤          │
│  │ Partition table  │ 64 bytes │
│  ├──────────────────┤          │
│  │ Boot signature   │ 2 bytes  │
│  │ (0x55AA)         │          │
│  └──────────────────┘          │
└────────────────────────────────┘
        │
        ▼
   Stage 1.5 / Stage 2 bootloader (GRUB)
```

BIOS limitations:

| Limitation | Impact |
|-----------|--------|
| 16-bit real mode operation | Limited memory access (1 MB) |
| MBR partition table | Max 4 primary partitions, 2 TB disk limit |
| No standard driver model | Each BIOS vendor implements differently |
| No built-in networking | Cannot boot from network without PXE ROM |
| No security framework | No secure boot chain |

---

## 10.2 UEFI Architecture

**UEFI** (Unified Extensible Firmware Interface) is the modern replacement for BIOS.

```
UEFI Architecture:

┌──────────────────────────────────────────────────────────┐
│                    UEFI Firmware                          │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  SEC Phase (Security)                              │  │
│  │  - CPU init, cache-as-RAM setup                    │  │
│  │  - Verify PEI volume                               │  │
│  └────────────────────────┬───────────────────────────┘  │
│                           ▼                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │  PEI Phase (Pre-EFI Initialization)                │  │
│  │  - Memory (DRAM) initialization                    │  │
│  │  - Discover boot mode (normal, recovery, S3)       │  │
│  │  - Create HOBs (Hand-Off Blocks)                   │  │
│  └────────────────────────┬───────────────────────────┘  │
│                           ▼                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │  DXE Phase (Driver Execution Environment)          │  │
│  │  - Load and execute UEFI drivers                   │  │
│  │  - Initialize all hardware (PCIe, USB, SATA, NVMe) │  │
│  │  - Install protocol interfaces                     │  │
│  │  - Set up Boot Services and Runtime Services       │  │
│  └────────────────────────┬───────────────────────────┘  │
│                           ▼                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │  BDS Phase (Boot Device Selection)                 │  │
│  │  - Evaluate boot options                           │  │
│  │  - Load boot loader from ESP (EFI System Partition)│  │
│  │  - Transfer control to OS loader                   │  │
│  └────────────────────────┬───────────────────────────┘  │
│                           ▼                              │
│  ┌────────────────────────────────────────────────────┐  │
│  │  TSL Phase (Transient System Load)                 │  │
│  │  - OS loader runs using Boot Services              │  │
│  │  - ExitBootServices() called                       │  │
│  │  - Boot Services memory freed                      │  │
│  │  - Only Runtime Services remain                    │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

BIOS vs UEFI comparison:

| Feature | Legacy BIOS | UEFI |
|---------|------------|------|
| Mode | 16-bit real mode | 32/64-bit protected mode |
| Boot code | MBR (512 bytes) | EFI application (unlimited) |
| Partition | MBR (2 TB limit) | GPT (9.4 ZB limit) |
| Disk count | 4 primary partitions | 128 partitions default |
| Drivers | Vendor-specific | Standardized EFI drivers |
| Secure Boot | None | Built-in (signature verification) |
| Boot speed | Slower (sequential init) | Faster (parallel init) |
| Network | PXE boot only | HTTP boot, iPXE |
| UI | Text-mode setup | Graphics-capable |
| OS interface | INT 13h | EFI system table |

---

## 10.3 Firmware Responsibilities

```
Firmware Must Establish This State Before Bootloader:

┌──────────────────────────────────────────────────────┐
│  CPU                                                 │
│  ├── Running at defined frequency                    │
│  ├── Caches enabled                                  │
│  ├── MMU may be OFF (ARM) or in identity-map mode    │
│  └── Interrupts disabled                             │
│                                                      │
│  Memory                                              │
│  ├── DRAM controller initialized and trained         │
│  ├── Memory size detected and reported               │
│  ├── Memory map available (what regions are usable)  │
│  └── Stack set up for bootloader                     │
│                                                      │
│  Boot Media                                          │
│  ├── Storage controller initialized                  │
│  ├── Can read bootloader from storage                │
│  └── Boot media type determined                      │
│                                                      │
│  Console                                             │
│  ├── UART initialized (embedded) or VGA init (PC)    │
│  └── Basic output for debug messages                 │
│                                                      │
│  Security                                            │
│  ├── Secure world initialized (ARM TrustZone)        │
│  ├── Root of trust established                       │
│  └── Next stage signature verified (secure boot)     │
└──────────────────────────────────────────────────────┘
```

---

## 10.4 Firmware Hardware Initialization

### SoC Firmware (Embedded ARM — e.g., Qualcomm, NXP, TI)

```
SoC Firmware Init Sequence:

1. Power Management IC (PMIC)
   └── Set voltage rails: CPU core, I/O, DRAM, peripherals
   └── Power-on sequence (order matters for safety)

2. Clock Tree Setup
   └── Crystal oscillator → PLL multiplication
   └── CPU PLL: 24 MHz → 1.8 GHz
   └── DDR PLL: 24 MHz → 1066 MHz
   └── Peripheral clocks: UART, SPI, I2C
   
3. Pin Multiplexing
   └── Configure IOMUX: which pins serve which function
   └── UART TX/RX, SPI MOSI/MISO/CLK/CS, I2C SDA/SCL

4. DRAM Controller
   └── DDR PHY training (read/write leveling)
   └── Timing calibration (tCAS, tRAS, tRP, tRCD)
   └── Memory test (optional, adds boot time)
   └── Report memory size to next stage

5. Boot Media Controller
   └── eMMC: HS400 training
   └── UFS: link startup
   └── SPI NOR: QSPI mode

6. Debug UART
   └── Set baud rate (typically 115200)
   └── First "signs of life" message
```

---

## 10.5 Firmware Boot Services

UEFI provides two types of services:

```
UEFI Services:

Boot Services (available until ExitBootServices()):
┌──────────────────────────────────────────────────────┐
│  Memory Services                                     │
│  ├── AllocatePages(), FreePages()                    │
│  ├── AllocatePool(), FreePool()                      │
│  └── GetMemoryMap()                                  │
│                                                      │
│  Protocol Services                                   │
│  ├── HandleProtocol()                                │
│  ├── LocateProtocol()                                │
│  └── OpenProtocol()                                  │
│                                                      │
│  Image Services                                      │
│  ├── LoadImage()                                     │
│  ├── StartImage()                                    │
│  └── ExitBootServices() ← Point of no return!        │
│                                                      │
│  Event/Timer Services                                │
│  ├── CreateEvent()                                   │
│  ├── SetTimer()                                      │
│  └── WaitForEvent()                                  │
└──────────────────────────────────────────────────────┘

Runtime Services (available after OS takes control):
┌──────────────────────────────────────────────────────┐
│  Variable Services                                   │
│  ├── GetVariable()         ← Read NVRAM variables    │
│  ├── SetVariable()         ← Write NVRAM variables   │
│  └── GetNextVariableName()                           │
│                                                      │
│  Time Services                                       │
│  ├── GetTime()             ← RTC access              │
│  └── SetTime()                                       │
│                                                      │
│  Virtual Memory Services                             │
│  └── SetVirtualAddressMap() ← Switch to virtual addr │
│                                                      │
│  Reset Services                                      │
│  └── ResetSystem()         ← Reboot/shutdown         │
└──────────────────────────────────────────────────────┘
```

```c
/* Linux kernel using UEFI Runtime Services */

/* Reading a UEFI variable from Linux */
/* /sys/firmware/efi/efivars/ exposes these */

/* The kernel's EFI stub — loaded by UEFI, calls ExitBootServices */
efi_status_t efi_main(efi_handle_t handle, efi_system_table_t *sys_table)
{
    /* Get memory map */
    status = efi_get_memory_map(&map);

    /* Exit boot services — firmware stops managing hardware */
    status = sys_table->boottime->exit_boot_services(handle, map_key);

    /* NOW the kernel owns all hardware */
    /* Jump to kernel proper */
}
```

---

## Interview Questions

**Q1: What is the difference between BIOS and UEFI?**
A: BIOS runs in 16-bit real mode, uses MBR partitioning (2 TB limit), has no standard driver model or security framework. UEFI runs in 32/64-bit mode, uses GPT (9.4 ZB limit), provides standardized driver model, Secure Boot, and proper boot services. UEFI is the modern standard.

**Q2: What does ExitBootServices() do and why is it important?**
A: `ExitBootServices()` is the handoff point from firmware to OS. After this call, all UEFI boot services are unavailable, boot services memory can be reclaimed by the OS, and the OS takes full control of hardware. Only UEFI Runtime Services (variables, time, reset) remain accessible. It's a point of no return.

**Q3: Why does DRAM initialization happen in firmware, not the kernel?**
A: The kernel is stored on disk/flash and needs to be loaded into DRAM before it can run. DRAM initialization (training, timing calibration) must happen before any large code can be loaded. The Boot ROM/firmware runs from on-chip SRAM or ROM to set up DRAM, then loads the bootloader/kernel into DRAM.

**Q4: What is DRAM training and why does it affect boot time?**
A: DRAM training calibrates signal timing between the memory controller and DRAM chips — read/write leveling, DQ/DQS alignment. This is needed because PCB trace lengths vary per board. Training takes 100-500 ms and is a significant chunk of boot time. Some SoCs cache training results in NVRAM to skip it on subsequent boots.

---

## Summary

- BIOS is legacy 16-bit firmware; UEFI is modern 32/64-bit firmware with standardized services
- UEFI boot phases: SEC → PEI (DRAM init) → DXE (drivers) → BDS (boot selection) → TSL (OS load)
- Firmware must initialize: CPU clocks, DRAM, boot media, debug console, security
- SoC firmware handles PMIC, PLL, pin mux, DDR training, and boot media controllers
- UEFI provides Boot Services (until ExitBootServices) and Runtime Services (persist after OS boot)
- DRAM training is essential but time-consuming — affects boot time significantly

---

*Next: [Chapter 11 — Bootloaders](Chapter_11_Bootloaders.md)*
