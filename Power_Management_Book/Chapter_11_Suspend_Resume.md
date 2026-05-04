# Chapter 11: System Suspend and Resume

## Learning Goals
- Understand the complete system suspend/resume flow
- Learn the suspend-to-RAM (S3) and suspend-to-idle (s2idle) mechanisms
- Know the hibernate (S4) lifecycle
- Understand freeze, suspend, and hibernate differences
- Learn to debug suspend/resume failures

---

## 1. System Suspend Overview

```
  System Suspend Types in Linux
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  "freeze" (Suspend-to-Idle / s2idle):                   │
  │  ┌───────────────────────────────────────────────────┐   │
  │  │  Software-only — no platform firmware involved     │   │
  │  │  Freezes tasks, suspends devices, enters CPU idle  │   │
  │  │  Power: ~100-500mW (depends on device states)      │   │
  │  │  Resume: Very fast (<1s)                           │   │
  │  │  Works on ALL platforms                             │   │
  │  └───────────────────────────────────────────────────┘   │
  │                                                           │
  │  "mem" (Suspend-to-RAM / S3 / deep):                    │
  │  ┌───────────────────────────────────────────────────┐   │
  │  │  Platform-assisted — firmware controls power rails  │   │
  │  │  Most hardware powered off, only RAM refreshed     │   │
  │  │  Power: ~1-5W                                      │   │
  │  │  Resume: ~3-5 seconds                              │   │
  │  │  Requires platform ACPI/PSCI support               │   │
  │  └───────────────────────────────────────────────────┘   │
  │                                                           │
  │  "disk" (Hibernate / S4):                               │
  │  ┌───────────────────────────────────────────────────┐   │
  │  │  RAM image written to swap partition/file           │   │
  │  │  All power can be removed                          │   │
  │  │  Power: 0W                                         │   │
  │  │  Resume: ~10-30 seconds (read image from disk)     │   │
  │  │  Survives power failure                            │   │
  │  └───────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Suspend-to-RAM (S3) Complete Flow

### 2.1 Suspend Path

```
  System Suspend-to-RAM Flow
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  User: echo mem > /sys/power/state                       │
  │  Or: systemctl suspend                                   │
  │    │                                                      │
  │    ▼                                                      │
  │  1. PM NOTIFIERS                                         │
  │     pm_notifier_call_chain(PM_SUSPEND_PREPARE)           │
  │     → Subsystems prepare (sync filesystems, etc.)        │
  │    │                                                      │
  │    ▼                                                      │
  │  2. FREEZE PROCESSES                                     │
  │     freeze_processes() → suspend_freeze_processes()      │
  │     → All userspace tasks frozen (TASK_FROZEN)           │
  │     → Kernel threads with PF_NOFREEZE continue           │
  │    │                                                      │
  │    ▼                                                      │
  │  3. CHECK WAKEUP EVENTS                                  │
  │     pm_wakeup_pending() → abort if wakeup occurred       │
  │    │                                                      │
  │    ▼                                                      │
  │  4. DEVICE PREPARE                                       │
  │     dpm_prepare(PMSG_SUSPEND)                            │
  │     → Call prepare() for all devices                     │
  │    │                                                      │
  │    ▼                                                      │
  │  5. DEVICE SUSPEND                                       │
  │     dpm_suspend(PMSG_SUSPEND)                            │
  │     → Call suspend() for all devices (children first)    │
  │     → Async suspend for enabled devices                  │
  │    │                                                      │
  │    ▼                                                      │
  │  6. DEVICE SUSPEND_LATE                                  │
  │     dpm_suspend_late(PMSG_SUSPEND)                       │
  │     → Call suspend_late() (no scheduling allowed)        │
  │    │                                                      │
  │    ▼                                                      │
  │  7. DISABLE NON-BOOT CPUS                               │
  │     suspend_disable_secondary_cpus()                     │
  │     → Hotplug off all CPUs except CPU0                   │
  │    │                                                      │
  │    ▼                                                      │
  │  8. DEVICE SUSPEND_NOIRQ                                 │
  │     dpm_suspend_noirq(PMSG_SUSPEND)                      │
  │     → Call suspend_noirq() (IRQs disabled)               │
  │    │                                                      │
  │    ▼                                                      │
  │  9. SYSCORE SUSPEND                                      │
  │     syscore_suspend()                                    │
  │     → Low-level: interrupt controllers, timers           │
  │    │                                                      │
  │    ▼                                                      │
  │  10. PLATFORM SUSPEND                                    │
  │      suspend_ops->enter(state)                           │
  │      → ACPI: write to PM1a_CNT register                 │
  │      → ARM: PSCI CPU_SUSPEND to ATF                     │
  │                                                           │
  │  ════════ SYSTEM IS NOW IN S3 (SLEEPING) ════════       │
  │  Only RAM powered, waiting for wakeup event              │
  └──────────────────────────────────────────────────────────┘
```

### 2.2 Resume Path

```
  System Resume Flow (reverse of suspend)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Wakeup event → platform firmware resumes CPU            │
  │    │                                                      │
  │    ▼                                                      │
  │  1. PLATFORM RESUME                                      │
  │     suspend_ops->wake() / ->finish()                     │
  │    │                                                      │
  │    ▼                                                      │
  │  2. SYSCORE RESUME                                       │
  │     syscore_resume()                                     │
  │     → Restore interrupt controllers, timers              │
  │    │                                                      │
  │    ▼                                                      │
  │  3. DEVICE RESUME_NOIRQ                                  │
  │     dpm_resume_noirq(PMSG_RESUME)                        │
  │     → Early HW init with IRQs disabled                   │
  │    │                                                      │
  │    ▼                                                      │
  │  4. ENABLE SECONDARY CPUS                                │
  │     suspend_enable_secondary_cpus()                      │
  │    │                                                      │
  │    ▼                                                      │
  │  5. DEVICE RESUME_EARLY                                  │
  │     dpm_resume_early(PMSG_RESUME)                        │
  │    │                                                      │
  │    ▼                                                      │
  │  6. DEVICE RESUME                                        │
  │     dpm_resume(PMSG_RESUME)                              │
  │     → Restore all devices (parents first)                │
  │    │                                                      │
  │    ▼                                                      │
  │  7. DEVICE COMPLETE                                      │
  │     dpm_complete(PMSG_RESUME)                            │
  │    │                                                      │
  │    ▼                                                      │
  │  8. THAW PROCESSES                                       │
  │     thaw_processes()                                     │
  │     → All tasks resume execution                         │
  │    │                                                      │
  │    ▼                                                      │
  │  9. PM NOTIFIERS                                         │
  │     pm_notifier_call_chain(PM_POST_SUSPEND)              │
  │                                                           │
  │  System is now fully awake                                │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Suspend-to-Idle (s2idle)

```c
/* kernel/power/suspend.c — s2idle implementation */

/*
 * s2idle doesn't use platform firmware —
 * it freezes tasks, suspends devices, then enters CPU idle
 * in a loop until a wakeup event occurs.
 */

static int suspend_enter(suspend_state_t state, bool *wakeup)
{
    if (state == PM_SUSPEND_TO_IDLE) {
        /* s2idle path — no platform firmware involved */

        /* Freeze tasks and suspend devices (same as S3) */
        /* ... */

        /* Enter idle loop instead of platform suspend */
        s2idle_loop();
        /* Returns when wakeup event detected */

        /* Resume devices and thaw tasks (same as S3) */
    }
}

static void s2idle_loop(void)
{
    do {
        /* Put all CPUs into deepest idle state */
        /* Platform may enter S0ix if all conditions met */
        cpuidle_enter_s2idle();

        /* Check for wakeup events */
        if (pm_wakeup_pending())
            break;

    } while (!need_resched());
}
```

```
  s2idle vs S3 Comparison
  ┌──────────────────┬────────────────────┬──────────────────┐
  │ Aspect           │ s2idle             │ S3 (deep)        │
  ├──────────────────┼────────────────────┼──────────────────┤
  │ Implementation   │ Software-only      │ Platform firmware│
  │ CPU state        │ Deep idle (C10+)   │ Powered off      │
  │ RAM              │ Powered (refresh)  │ Self-refresh     │
  │ Devices          │ Runtime suspended  │ Full suspend     │
  │ Power            │ ~100-500mW         │ ~1-5W            │
  │ Resume time      │ <1 second          │ 3-5 seconds      │
  │ Network          │ Can stay connected │ Disconnected     │
  │ Wakeup sources   │ Any IRQ            │ Platform wakeup  │
  │ Platform support │ Always available   │ ACPI/PSCI needed │
  └──────────────────┴────────────────────┴──────────────────┘
```

---

## 4. Hibernate (Suspend-to-Disk)

### 4.1 Hibernate Flow

```
  Hibernate (S4) Flow
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  echo disk > /sys/power/state                            │
  │    │                                                      │
  │    ▼                                                      │
  │  1. FREEZE ALL TASKS                                     │
  │    │                                                      │
  │    ▼                                                      │
  │  2. CREATE SNAPSHOT                                      │
  │     hibernate_snapshot()                                  │
  │     → snapshot_additional_pages()                         │
  │     → Create memory bitmap of pages to save              │
  │     → Copy snapshot into allocated pages                 │
  │    │                                                      │
  │    ▼                                                      │
  │  3. WRITE IMAGE TO SWAP                                  │
  │     swsusp_write()                                       │
  │     → Compress pages (LZO/LZ4)                           │
  │     → Write to swap partition or swap file               │
  │     → Write header with resume info                      │
  │    │                                                      │
  │    ▼                                                      │
  │  4. POWER OFF                                            │
  │     Based on /sys/power/disk setting:                    │
  │     ├── "platform" — ACPI S4 (can wake from RTC)        │
  │     ├── "shutdown" — Complete power off                  │
  │     ├── "reboot"   — Reboot immediately                 │
  │     └── "suspend"  — Hybrid: S3 with disk image          │
  │                                                           │
  │  ════════ POWER IS OFF ════════                          │
  │                                                           │
  │  RESUME (at next boot):                                  │
  │  1. Bootloader loads kernel                              │
  │  2. Kernel checks swap for hibernate image header        │
  │  3. Loads and decompresses image                         │
  │  4. Restores memory state                                │
  │  5. Jumps to restore point                               │
  │  6. Resume devices and thaw processes                    │
  │                                                           │
  │  Resume kernel cmdline:                                  │
  │  resume=/dev/sda2  (swap partition with hibernate image) │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 Hibernate Kernel Configuration

```
  Required kernel config for hibernate:
  ┌──────────────────────────────────────────────────────┐
  │  CONFIG_HIBERNATION=y                                │
  │  CONFIG_SUSPEND=y                                    │
  │  CONFIG_PM=y                                         │
  │  CONFIG_SWAP=y             # Need swap space          │
  │  CONFIG_CRYPTO_LZO=y      # Compression (optional)   │
  │                                                       │
  │  Swap space must be ≥ physical RAM size               │
  │                                                       │
  │  Kernel command line:                                 │
  │  resume=/dev/sdaN          # Swap partition           │
  │  resume=UUID=xxxx          # Or by UUID               │
  │  resume_offset=NNN         # If using swap file       │
  └──────────────────────────────────────────────────────┘
```

---

## 5. Process Freezer

```c
/* kernel/power/process.c — Process freezing for suspend */

int freeze_processes(void)
{
    /* Set system_freezing_cnt so tasks check for freeze */
    /* Each task checks in schedule() if it should freeze */

    /* Wait for all tasks to enter frozen state */
    error = try_to_freeze_tasks(true);
    /* true = freeze userspace only */

    if (error)
        /* Some task didn't freeze in time — abort suspend */
        thaw_processes();

    return error;
}

int freeze_kernel_threads(void)
{
    /* Freeze kernel threads (except PF_NOFREEZE ones) */
    error = try_to_freeze_tasks(false);
    return error;
}

/* Tasks check for freezing in key scheduling points:
 * - schedule()
 * - signal delivery
 * - Various explicit try_to_freeze() calls
 *
 * PF_NOFREEZE tasks: critical kernel threads that must
 * continue during suspend (e.g., suspend thread itself)
 */
```

```
  Process Freeze Order
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Freeze order:                                           │
  │  1. Userspace tasks (freeze_processes)                   │
  │     → All user processes and their threads frozen        │
  │     → Workqueues drained                                 │
  │  2. Kernel threads (freeze_kernel_threads)               │
  │     → Freezable kthreads frozen                          │
  │     → PF_NOFREEZE threads continue                       │
  │                                                           │
  │  Thaw order (reverse):                                   │
  │  1. Kernel threads (thaw_kernel_threads)                 │
  │  2. Userspace tasks (thaw_processes)                     │
  │                                                           │
  │  Timeout: FREEZE_TIMEOUT = 20 seconds                    │
  │  If any task doesn't freeze in time → abort              │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Wakeup Events and Abort

```c
/* Wakeup event detection during suspend */
/* kernel/power/wakeup_count.c */

bool pm_wakeup_pending(void)
{
    unsigned long flags;
    bool ret = false;

    raw_spin_lock_irqsave(&events_lock, flags);
    if (events_check_enabled) {
        /* A wakeup event occurred after suspend started */
        ret = ((unsigned int)atomic_read(&event_count) !=
                saved_count);
    }
    raw_spin_unlock_irqrestore(&events_lock, flags);
    return ret;
}

/* Suspend checks for wakeup events at multiple points:
 *
 * 1. After freezing processes
 * 2. After suspending devices
 * 3. Before entering platform suspend
 *
 * If wakeup detected → abort suspend → resume everything
 *
 * Common abort scenarios:
 * - User presses key during suspend
 * - Network packet arrives (WoL)
 * - Timer expires
 * - Hardware alarm (RTC)
 */
```

---

## 7. Suspend/Resume Timing

```bash
# Enable suspend/resume timing in kernel log
echo 1 > /sys/power/pm_print_times

# Suspend and check timing
echo mem > /sys/power/state
dmesg | grep "suspend"

# Example output:
# [  100.123] PM: suspend entry (deep)
# [  100.234] PM: Syncing filesystems ... done.
# [  100.345] Freezing user space processes ... (elapsed 0.010 seconds) done.
# [  100.456] Freezing remaining freezable tasks ... (elapsed 0.001 seconds) done.
# [  100.500] PM: suspend of devices complete after 43.567 msecs
# [  100.520] PM: late suspend of devices complete after 19.234 msecs
# [  100.540] PM: noirq suspend of devices complete after 18.890 msecs
# ... S3 ...
# [  105.000] PM: noirq resume of devices complete after 15.678 msecs
# [  105.020] PM: early resume of devices complete after 18.345 msecs
# [  105.100] PM: resume of devices complete after 79.234 msecs
# [  105.110] PM: suspend exit

# Identify slow devices
dmesg | grep -E "call .+\+.*returned" | sort -t'+' -k2 -n -r | head
# Shows devices sorted by suspend/resume time
```

---

## 8. Debugging Suspend/Resume Failures

### 8.1 Common Failure Points

```
  Suspend Failure Diagnosis
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Failure Type        │ Diagnosis                          │
  │  ────────────────────┼──────────────────────────────────  │
  │  Process won't freeze│ Check: dmesg "Freezing...failed"  │
  │                      │ Some task ignoring freeze request  │
  │                      │ Look for TASK_UNINTERRUPTIBLE      │
  │                      │                                    │
  │  Device suspend fails│ Check: dmesg "PM: Device X failed" │
  │                      │ Driver bug in suspend callback     │
  │                      │ Missing clock/power dependency     │
  │                      │                                    │
  │  Platform entry fails│ ACPI/PSCI error                    │
  │                      │ Missing wakeup source              │
  │                      │ Firmware bug                        │
  │                      │                                    │
  │  Wakeup abort        │ Wakeup event during suspend        │
  │                      │ Check: /sys/power/wakeup_count     │
  │                      │ Spurious IRQ                       │
  │                      │                                    │
  │  Resume hang         │ Device resume callback stuck       │
  │                      │ Timeout on I2C/SPI access          │
  │                      │ Clock/regulator not restored       │
  └──────────────────────────────────────────────────────────┘
```

### 8.2 Test Modes

```bash
# Test suspend without entering S3
echo freezer > /sys/power/pm_test
echo mem > /sys/power/state
# Only freezes processes, then resumes

echo devices > /sys/power/pm_test
echo mem > /sys/power/state
# Freezes processes + suspends devices, then resumes

echo platform > /sys/power/pm_test
echo mem > /sys/power/state
# Full flow except actual hardware entry

echo processors > /sys/power/pm_test
echo mem > /sys/power/state
# Includes CPU hotplug

echo core > /sys/power/pm_test
echo mem > /sys/power/state
# Full flow including syscore, but no platform entry

# Reset to normal operation
echo none > /sys/power/pm_test

# Trace suspend/resume
echo 1 > /sys/kernel/debug/tracing/events/power/suspend_resume/enable
echo mem > /sys/power/state
cat /sys/kernel/debug/tracing/trace
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/suspend.c` | System suspend core (enter/resume_s3) |
| `kernel/power/hibernate.c` | Hibernate implementation |
| `kernel/power/process.c` | Process freezer |
| `kernel/power/main.c` | PM core, sysfs, pm_test |
| `kernel/power/wakeup_count.c` | Wakeup event tracking |
| `kernel/power/snapshot.c` | Hibernate snapshot creation |
| `kernel/power/swap.c` | Hibernate image I/O |
| `drivers/base/power/main.c` | Device PM list, dpm_suspend/resume |
| `drivers/acpi/sleep.c` | ACPI suspend operations |
| `drivers/firmware/psci/psci.c` | ARM PSCI suspend |

---

## Interview Questions

**Q1: Walk through the complete suspend-to-RAM (S3) flow.**
**A:** (1) PM notifiers fired (PM_SUSPEND_PREPARE). (2) Freeze userspace processes and kernel threads. (3) Check for pending wakeup events (abort if detected). (4) Prepare all devices (dpm_prepare). (5) Suspend all devices in child-before-parent order (dpm_suspend → suspend_late → suspend_noirq). (6) Disable secondary CPUs (hotplug off). (7) Syscore suspend (IRQ controllers, timers). (8) Platform enter (ACPI PM1a_CNT write or ARM PSCI). System sleeps. On wakeup: reverse order — syscore resume, enable CPUs, resume devices (parent-before-child), thaw processes, PM notifiers.

**Q2: What is the difference between s2idle and S3 suspend?**
**A:** s2idle (suspend-to-idle) is software-only: it freezes tasks, suspends devices, then enters a CPU idle loop. No platform firmware is involved. Power savings are moderate (~100-500mW) but resume is instant (<1s). S3 (suspend-to-RAM) uses platform firmware (ACPI/PSCI) to power off most hardware, keeping only RAM in self-refresh. Power is lower (~1-5W) but resume takes 3-5s. s2idle works everywhere; S3 requires platform support. Modern laptops often prefer s2idle for connected standby.

**Q3: How does the process freezer work?**
**A:** The freezer sets a global flag (system_freezing_cnt). Userspace tasks are frozen first — they check for the freeze flag in schedule() and signal delivery paths, then enter TASK_FROZEN state. A 20-second timeout applies; tasks that don't freeze in time cause suspend to abort. Then freezable kernel threads are frozen. Threads with PF_NOFREEZE flag continue (they're critical for the suspend process itself). On resume, thawing happens in reverse order.

**Q4: How would you debug a device that fails to suspend?**
**A:** (1) Enable PM timing: `echo 1 > /sys/power/pm_print_times` and check dmesg for "Device X failed to suspend." (2) Use PM test mode: `echo devices > /sys/power/pm_test` to test device suspend without entering S3. (3) Enable PM trace: add `pm_trace` to kernel cmdline — on resume failure, dmesg shows the last device being suspended. (4) Check ftrace for suspend/resume events. (5) Review the driver's .suspend callback for bugs — missing error handling, timeout on hardware access, or wrong power sequencing.

**Q5: Explain hibernate and how it differs from suspend-to-RAM.**
**A:** Hibernate (S4) saves the entire RAM contents to a swap partition/file, then powers off completely (0W). On boot, the kernel detects the hibernate image (via `resume=` cmdline), loads and decompresses it, restores memory state, and jumps back to the restore point. Key differences from S3: (1) survives complete power loss, (2) takes much longer (10-30s for image I/O), (3) needs swap space ≥ RAM size, (4) uses compression (LZO/LZ4) to reduce image size. Hybrid mode (suspend-to-both) writes image to disk AND enters S3, providing fast wake with power-failure safety.

---

## Summary

- Three Linux suspend modes: s2idle (software idle loop), S3 (platform suspend-to-RAM), S4 (hibernate to disk)
- Suspend flow: notify → freeze processes → suspend devices (child→parent) → disable CPUs → platform enter
- Resume is exact reverse: platform wake → enable CPUs → resume devices (parent→child) → thaw processes
- s2idle is software-only, fast resume, always available; S3 needs platform support but saves more power
- Hibernate writes RAM snapshot to swap, survives power loss, needs swap ≥ RAM
- Process freezer sets global flag that tasks check in schedule() — 20 second timeout
- Wakeup events can abort suspend at multiple checkpoints
- Debug with: pm_print_times, pm_test modes, ftrace suspend_resume events

---

[Previous Chapter: Runtime PM ←](Chapter_10_Runtime_PM.md) | [Next Chapter: Power Domains →](Chapter_12_Power_Domains.md)
