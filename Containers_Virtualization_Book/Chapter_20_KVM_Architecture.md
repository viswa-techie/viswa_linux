# Chapter 20: KVM Architecture

## Learning Goals
- Understand KVM's position as a Linux kernel module
- Learn the KVM ioctl interface and vcpu lifecycle
- Master the KVM-QEMU split (kernel vs userspace)
- Know vCPU scheduling and the KVM run loop

---

## 1. KVM Architecture Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KVM = Kernel-based Virtual Machine                      │
  │  Type 1.5 hypervisor (kernel module, not standalone)     │
  │                                                           │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ User Space (QEMU process per VM)             │       │
  │  │  ┌────────────────────────────────────────┐  │       │
  │  │  │ QEMU                                   │  │       │
  │  │  │ ┌──────────┐ ┌──────────┐             │  │       │
  │  │  │ │ Device   │ │ Device   │ ...         │  │       │
  │  │  │ │ emulation│ │ emulation│ (virtio-net │  │       │
  │  │  │ │ (e1000)  │ │ (IDE)    │  virtio-blk)│  │       │
  │  │  │ └──────────┘ └──────────┘             │  │       │
  │  │  │ ┌──────────┐ ┌──────────┐             │  │       │
  │  │  │ │ vCPU     │ │ vCPU     │             │  │       │
  │  │  │ │ thread 0 │ │ thread 1 │             │  │       │
  │  │  │ └────┬─────┘ └────┬─────┘             │  │       │
  │  │  └──────┼─────────────┼───────────────────┘  │       │
  │  └─────────┼─────────────┼──────────────────────┘       │
  │            │ ioctl       │ ioctl                          │
  │  ══════════╪═════════════╪══════════════════════════     │
  │            ▼             ▼                                │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Kernel Space                                  │       │
  │  │  ┌────────────────────────────────────────┐  │       │
  │  │  │ KVM module (kvm.ko + kvm-intel.ko)     │  │       │
  │  │  │                                        │  │       │
  │  │  │ ┌──────┐  ┌──────┐                    │  │       │
  │  │  │ │ vCPU │  │ vCPU │  ← runs on host CPU│  │       │
  │  │  │ │  0   │  │  1   │  in VMX non-root    │  │       │
  │  │  │ └──────┘  └──────┘                    │  │       │
  │  │  │                                        │  │       │
  │  │  │ EPT management, interrupt injection,   │  │       │
  │  │  │ VMCS setup, VM exit handling           │  │       │
  │  │  └────────────────────────────────────────┘  │       │
  │  │                                               │       │
  │  │  Linux scheduler, memory management, drivers │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Key insight: KVM turns Linux itself into a hypervisor  │
  │  - Linux scheduler schedules vCPU threads               │
  │  - Linux memory management manages guest memory (EPT)   │
  │  - Linux device drivers provide host I/O                │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. KVM ioctl Interface

```c
/*
 * KVM uses three levels of file descriptors:
 * 1. /dev/kvm — system-level (create VMs)
 * 2. VM fd — per-VM (create vCPUs, set memory)
 * 3. vCPU fd — per-vCPU (run, get/set regs)
 */

#include <linux/kvm.h>
#include <sys/ioctl.h>
#include <fcntl.h>

/* Step 1: Open KVM device */
int kvm_fd = open("/dev/kvm", O_RDWR);

/* Check API version */
int api_ver = ioctl(kvm_fd, KVM_GET_API_VERSION, 0);
/* Must return 12 */

/* Step 2: Create a VM */
int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);

/* Step 3: Set up guest memory */
struct kvm_userspace_memory_region region = {
    .slot = 0,
    .flags = 0,
    .guest_phys_addr = 0x0,       /* GPA start */
    .memory_size = 256 * 1024 * 1024, /* 256 MB */
    .userspace_addr = (uint64_t)mmap(NULL,
        256 * 1024 * 1024,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS, -1, 0),
};
ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);
/* KVM creates EPT mapping: GPA → HPA (backing mmap) */

/* Step 4: Create a vCPU */
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);

/* Step 5: Map the kvm_run structure */
int mmap_size = ioctl(kvm_fd, KVM_GET_VCPU_MMAP_SIZE, 0);
struct kvm_run *run = mmap(NULL, mmap_size,
    PROT_READ | PROT_WRITE, MAP_SHARED, vcpu_fd, 0);

/* Step 6: Set initial registers */
struct kvm_regs regs = {
    .rip = 0x1000,  /* entry point */
    .rsp = 0xFFFF0, /* stack pointer */
    .rflags = 0x2,  /* mandatory bit */
};
ioctl(vcpu_fd, KVM_SET_REGS, &regs);

/* Step 7: Run the vCPU (the main loop) */
while (1) {
    ioctl(vcpu_fd, KVM_RUN, 0);
    /* Returns on VM exit */

    switch (run->exit_reason) {
    case KVM_EXIT_IO:
        /* Handle port I/O (IN/OUT) */
        handle_io(run);
        break;
    case KVM_EXIT_MMIO:
        /* Handle memory-mapped I/O */
        handle_mmio(run);
        break;
    case KVM_EXIT_HLT:
        /* Guest executed HLT — vCPU idle */
        return 0;
    case KVM_EXIT_SHUTDOWN:
        /* Guest triple-faulted */
        return 1;
    }
}
```

---

## 3. The KVM Run Loop

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  QEMU vCPU thread lifecycle:                             │
  │                                                           │
  │  QEMU vCPU thread                                        │
  │    │                                                     │
  │    ├──► ioctl(vcpu_fd, KVM_RUN)                         │
  │    │     │                                               │
  │    │     ▼ (kernel space — KVM module)                   │
  │    │     ├── Load guest VMCS                             │
  │    │     ├── VMLAUNCH / VMRESUME                         │
  │    │     │     │                                         │
  │    │     │     ▼ (VMX non-root — guest running)          │
  │    │     │     ├── Guest executes code                   │
  │    │     │     ├── Guest does I/O or privileged op       │
  │    │     │     └── VM Exit occurs                        │
  │    │     │                                               │
  │    │     ├── VM exit handler (still in kernel)           │
  │    │     │   ├── If KVM can handle: do it                │
  │    │     │   │   (e.g., EPT fault → allocate page,      │
  │    │     │   │    MSR → emulate, timer → inject)        │
  │    │     │   │   → VMRESUME (re-enter guest)            │
  │    │     │   │                                           │
  │    │     │   └── If QEMU must handle: return to user    │
  │    │     │       (e.g., I/O to emulated device,         │
  │    │     │        MMIO to emulated hardware)            │
  │    │     │                                               │
  │    │     └── Return from ioctl (run->exit_reason set)   │
  │    │                                                     │
  │    ├── Handle exit in QEMU userspace                    │
  │    │   (emulate device, update state)                    │
  │    │                                                     │
  │    └── Loop: ioctl(KVM_RUN) again                        │
  │                                                           │
  │  Three execution contexts for a vCPU:                    │
  │  1. Guest mode (VMX non-root) — running guest code      │
  │  2. Kernel mode (KVM) — handling VM exits               │
  │  3. User mode (QEMU) — device emulation                 │
  │                                                           │
  │  Performance: most exits handled in KVM (kernel)        │
  │  Only I/O to emulated devices exits to QEMU (userspace) │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. vCPU Scheduling

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Each vCPU = one Linux thread (in QEMU process)         │
  │  Scheduled by Linux CFS like any other thread           │
  │                                                           │
  │  Host CPU 0    Host CPU 1    Host CPU 2    Host CPU 3   │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
  │  │ VM1-vCPU0│ │ VM1-vCPU1│ │ VM2-vCPU0│ │ host     │  │
  │  │ (QEMU    │ │ (QEMU    │ │ (QEMU    │ │ processes│  │
  │  │  thread) │ │  thread) │ │  thread) │ │          │  │
  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
  │                                                           │
  │  vCPU pinning (for performance):                         │
  │  - Pin vCPU thread to specific host CPU                 │
  │  - Avoids migration overhead (TLB, cache)               │
  │  - taskset -c 0 qemu-vCPU-0-thread                     │
  │  - Or: <vcpupin vcpu='0' cpuset='2'/>  (libvirt)       │
  │                                                           │
  │  Overcommit:                                             │
  │  - More vCPUs than physical CPUs                        │
  │  - Linux scheduler time-slices between vCPU threads     │
  │  - Causes "steal time" — guest sees CPU stolen          │
  │  - /proc/stat: st (steal) column                        │
  │  - KVM reports steal time to guest via PV clock          │
  │                                                           │
  │  NUMA awareness:                                         │
  │  - Pin vCPUs and memory to same NUMA node               │
  │  - Avoid cross-NUMA memory access (2-3x slower)        │
  │  - numactl --membind=0 --cpunodebind=0 qemu ...        │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. KVM Kernel Module Structure

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KVM kernel modules:                                     │
  │                                                           │
  │  kvm.ko                                                  │
  │  ├── /dev/kvm character device                          │
  │  ├── VM lifecycle management                            │
  │  ├── Memory management (memory slots, dirty tracking)   │
  │  ├── Interrupt injection (APIC emulation)               │
  │  ├── Timer management                                    │
  │  └── Architecture-independent code                      │
  │                                                           │
  │  kvm-intel.ko (or kvm-amd.ko)                           │
  │  ├── VMX/SVM hardware setup                             │
  │  ├── VMCS/VMCB management                               │
  │  ├── VM entry/exit code                                 │
  │  ├── EPT/NPT management                                 │
  │  ├── Posted interrupts                                   │
  │  └── Architecture-specific handling                     │
  │                                                           │
  │  Key kernel structures:                                  │
  │  struct kvm {                                            │
  │      struct kvm_memslots *memslots[KVM_ADDRESS_SPACE_NUM];│
  │      struct kvm_vcpu *vcpus[KVM_MAX_VCPUS];             │
  │      struct kvm_io_bus *buses[KVM_NR_BUSES];            │
  │      struct mm_struct *mm;  /* userspace memory */       │
  │      spinlock_t mmu_lock;                                │
  │  };                                                       │
  │                                                           │
  │  struct kvm_vcpu {                                       │
  │      struct kvm *kvm;                                    │
  │      int vcpu_id;                                        │
  │      struct kvm_run *run;    /* shared with userspace */ │
  │      struct kvm_vcpu_arch arch; /* VMCS, regs, etc. */  │
  │      struct task_struct *task; /* host thread */         │
  │  };                                                       │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does KVM turn Linux into a hypervisor?**
**A:** KVM is a Linux kernel module (`kvm.ko` + `kvm-intel.ko`/`kvm-amd.ko`) that leverages existing Linux infrastructure instead of building a standalone hypervisor. It exposes `/dev/kvm` — a character device with an ioctl interface at three levels: (1) System fd: `KVM_CREATE_VM` creates a new VM. (2) VM fd: `KVM_SET_USER_MEMORY_REGION` maps guest physical memory to host virtual memory (QEMU's mmap'd region → KVM sets up EPT), `KVM_CREATE_VCPU` creates virtual CPUs. (3) vCPU fd: `KVM_RUN` enters guest mode (VMLAUNCH/VMRESUME), returns on VM exit. Each vCPU is a QEMU thread scheduled by Linux's CFS scheduler — KVM doesn't need its own scheduler. Guest memory is managed by Linux's memory subsystem (mmap, page cache, swap, NUMA policies, KSM deduplication) — KVM doesn't need its own memory manager. Host device drivers provide I/O. This design means KVM benefits from all Linux improvements (scheduler, memory, networking) automatically. The trade-off: Linux's general-purpose scheduler isn't always optimal for real-time VM workloads (vs. dedicated hypervisor schedulers), but this is mitigated with vCPU pinning and PREEMPT_RT.

**Q2: What are the three execution contexts of a KVM vCPU, and what happens in each?**
**A:** A vCPU alternates between three contexts: (1) **Guest mode** (VMX non-root): the guest OS runs natively on the CPU. Guest code executes at hardware speed, including ring 0 kernel code. This is where the VM spends most of its time. Ends when a VM exit occurs. (2) **Kernel mode** (KVM in host kernel): the VM exit handler in KVM processes the exit reason. Most exits are handled entirely in kernel: EPT violations (allocate a page, update EPT), timer interrupts (inject virtual timer to guest), MSR accesses (emulate), CPUID (return sanitized values). After handling, KVM immediately does VMRESUME to re-enter guest — no return to userspace. (3) **User mode** (QEMU process): for exits that require device emulation (I/O to emulated NIC, disk, USB). KVM returns from the `KVM_RUN` ioctl with `run->exit_reason` set. QEMU processes the exit (e.g., reads from virtual disk image, computes device state), then calls `KVM_RUN` again. Performance optimization: virtio devices minimize user-mode exits by using shared memory rings (virtqueues) and eventfd-based notification (ioeventfd/irqfd), allowing much I/O to be handled in kernel without exiting to QEMU.

---

## Summary

- KVM: kernel module that makes Linux a hypervisor (not a standalone hypervisor)
- /dev/kvm → ioctl API: system fd → vm fd → vcpu fd (three levels)
- Each vCPU = one QEMU thread, scheduled by Linux CFS
- KVM_RUN: enters guest (VMLAUNCH), returns on VM exit
- Three contexts: guest mode (fast), kernel KVM (handles most exits), user QEMU (device emulation)
- Guest memory: QEMU mmap → KVM EPT mapping → guest accesses at hardware speed
- KVM reuses Linux scheduler, memory manager, device drivers — all Linux improvements apply

---

[Previous: HW Virtualization Foundations ←](Chapter_19_HW_Virtualization.md) | [Next: QEMU and Device Emulation →](Chapter_21_QEMU_Devices.md)
