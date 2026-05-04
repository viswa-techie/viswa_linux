# Chapter 21: Multiprocessor Boot

## Learning Goals
- Understand SMP boot architecture
- Know how secondary CPUs are brought online
- Understand PSCI and spin-table boot methods
- Grasp CPU hotplug support

---

## 21.1 SMP Systems Overview

```
SMP Boot Challenge:

At power on, only ONE CPU (the boot/primary CPU) executes.
All other CPUs are held in reset or a wait loop.
The kernel must explicitly bring each one online.

Boot CPU (CPU 0):
  ├── Runs firmware, bootloader, kernel init
  ├── Initializes ALL kernel subsystems
  ├── Sets up per-CPU data for ALL CPUs
  └── Triggers secondary CPU boot via smp_init()

Secondary CPUs (CPU 1, 2, 3...):
  ├── Wait in firmware or spin-table until signaled
  ├── Each secondary CPU:
  │   1. Receives wake signal from boot CPU
  │   2. Initializes its own state (MMU, caches, vectors)
  │   3. Calls secondary_start_kernel()
  │   4. Enters the idle loop
  └── Now available for scheduler to assign tasks
```

---

## 21.2 CPU Initialization

```
Primary CPU Init (during start_kernel):

start_kernel()
  ├── boot_cpu_init()         ← Mark CPU 0 as online
  ├── setup_per_cpu_areas()   ← Allocate per-CPU data for ALL CPUs
  ├── smp_prepare_boot_cpu()  ← Boot CPU-specific setup
  └── rest_init() → kernel_init() → smp_init()

smp_init():
  ┌─────────────────────────────────────────────────┐
  │  for_each_present_cpu(cpu) {                    │
  │      if (cpu == boot_cpu) continue;             │
  │      cpu_up(cpu);  ← Bring secondary online     │
  │  }                                              │
  │  pr_info("SMP: Total of %d processors activated"│
  └─────────────────────────────────────────────────┘
```

---

## 21.3 Secondary CPU Boot Process

```
Secondary CPU Boot (ARM64):

Boot CPU calls cpu_up(cpu_id):
     │
     ├── __cpu_up() → cpu_ops->cpu_boot(cpu_id)
     │   │
     │   ├── PSCI method: smc/hvc PSCI_CPU_ON
     │   │   (Firmware wakes the CPU)
     │   │
     │   └── Spin-table method: write to cpu-release-addr
     │       (CPU polling sees the write, starts executing)
     │
     ▼
Secondary CPU wakes up:
     │
     ├── secondary_entry (arch/arm64/kernel/head.S)
     │   ├── el2_setup    ← Configure EL2/EL1
     │   ├── set_cpu_boot_mode_flag
     │   ├── __enable_mmu ← Enable MMU with kernel page tables
     │   └── secondary_startup
     │
     ├── secondary_start_kernel() (C code)
     │   ├── cpu_init()           ← Per-CPU initialization
     │   ├── notify_cpu_starting() ← CPU starting notification
     │   ├── calibrate_delay()     ← BogoMIPS for this CPU
     │   ├── set_cpu_online()      ← Mark as online
     │   └── cpu_startup_entry()   ← Enter idle loop
     │
     └── CPU is now ONLINE and available for scheduling
```

### PSCI vs Spin-Table

```
PSCI (Power State Coordination Interface):
┌─────────────────────────────────────────────┐
│  Standardized ARM firmware interface        │
│  CPU_ON, CPU_OFF, SUSPEND, SYSTEM_RESET     │
│                                             │
│  Boot CPU:                                  │
│    smc #0 → PSCI_CPU_ON(cpu_id, entry_addr)│
│                                             │
│  Firmware:                                  │
│    Powers on target CPU                     │
│    Sets PC to entry_addr                    │
│    Returns success/failure                  │
│                                             │
│  Advantages:                                │
│    - Secure (firmware controls power)       │
│    - Standard (same for all ARM SoCs)       │
│    - Supports CPU hotplug and suspend       │
└─────────────────────────────────────────────┘

Spin-Table (simple/legacy):
┌─────────────────────────────────────────────┐
│  Secondary CPUs poll a memory address       │
│                                             │
│  Boot: cpu-release-addr = 0x12345678        │
│        Secondary CPU loops:                 │
│          while (*release_addr == 0)         │
│              wfe;     /* Wait For Event */  │
│                                             │
│  Boot CPU writes entry point to release_addr│
│  Boot CPU sends SEV (Signal Event)          │
│  Secondary CPU reads addr, jumps there      │
│                                             │
│  Simpler but less capable (no hotplug)      │
└─────────────────────────────────────────────┘
```

```dts
/* Device Tree CPU boot methods */

cpus {
    cpu@0 {
        compatible = "arm,cortex-a78";
        enable-method = "psci";       /* ← Uses PSCI */
    };
};

psci {
    compatible = "arm,psci-1.0";
    method = "smc";                   /* SMC call to EL3 firmware */
};
```

---

## 21.4 CPU Hotplug Support

```
CPU Hotplug — Online/Offline CPUs at Runtime:

# Take CPU 3 offline
echo 0 > /sys/devices/system/cpu/cpu3/online
# Scheduler migrates all tasks off CPU 3
# IRQs moved away from CPU 3
# CPU 3 enters deep idle / powered off

# Bring CPU 3 back online
echo 1 > /sys/devices/system/cpu/cpu3/online
# CPU re-initialized and available for scheduling

Use cases:
  - Power saving (disable unused cores)
  - Thermal management (disable overheating core)
  - System maintenance (isolate a core)
  - Android: keep big cores off for battery life
```

```c
/* CPU hotplug notifier chain — drivers can react to CPU events */

static int my_cpu_callback(unsigned int cpu)
{
    pr_info("CPU %u coming online\n", cpu);
    /* Set up per-CPU resources for this CPU */
    return 0;
}

/* Register during init */
cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "my-driver:online",
                  my_cpu_callback, my_cpu_teardown);
```

---

## Interview Questions

**Q1: How does Linux boot secondary CPUs on an ARM64 SoC?**
A: The boot CPU calls `smp_init()` → `cpu_up()` for each secondary CPU. Using PSCI, it makes an SMC call to firmware (EL3) with `PSCI_CPU_ON`, passing the CPU ID and kernel entry address. Firmware powers on the CPU and sets its PC. The secondary CPU runs `secondary_entry` (assembly), enables MMU, then `secondary_start_kernel()` (C), initializes per-CPU state, and enters the idle loop.

**Q2: What is the difference between PSCI and spin-table boot methods?**
A: PSCI is a standardized firmware interface — uses secure monitor calls to power CPUs on/off, supports hotplug and suspend. Spin-table is simpler — secondary CPUs poll a memory address until the boot CPU writes an entry point there. PSCI is preferred for production; spin-table is for simple/early bring-up.

**Q3: Why can't secondary CPUs just start running the kernel?**
A: Each CPU needs its own state: stack, per-CPU data, TLB, caches, exception vectors, timer. The boot CPU must set up shared data structures (page tables, scheduler run queues, per-CPU areas) before secondaries can use them. Without this coordination, secondaries would access uninitialized data, causing crashes.

---

## Summary

- Only one CPU (boot CPU) runs at power-on — it initializes the entire kernel
- Secondary CPUs are brought online during `smp_init()` after all subsystems are ready
- ARM64 uses PSCI (firmware standard) or spin-table (polling) to wake secondary CPUs
- Each secondary CPU initializes its own state then enters the idle loop
- CPU hotplug allows runtime online/offline for power management and maintenance

---

*Next: [Chapter 22 — Kernel Scheduling Initialization](Chapter_22_Scheduling_Initialization.md)*
