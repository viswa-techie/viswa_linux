# Chapter 19: Hardware Virtualization Foundations

## Learning Goals
- Understand hardware virtualization extensions (VT-x, AMD-V)
- Learn ring levels, VMX root/non-root operation modes
- Master VM exits and their performance implications
- Know Extended Page Tables (EPT) / Nested Page Tables (NPT)

---

## 1. Virtualization Problem and HW Solution

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  The virtualization challenge:                           │
  │  Guest OS expects to run at ring 0 (kernel mode)        │
  │  But only the host/hypervisor should have ring 0        │
  │                                                           │
  │  Before hardware support (binary translation / trap):   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Ring 3: applications                     │            │
  │  │ Ring 2: (unused)                         │            │
  │  │ Ring 1: guest kernel (deprivileged!)     │            │
  │  │ Ring 0: hypervisor                       │            │
  │  │                                          │            │
  │  │ Problem: guest kernel's privileged ops   │            │
  │  │ (MOV to CR3, LGDT, etc.) trap to ring 0 │            │
  │  │ Hypervisor emulates them → SLOW          │            │
  │  │ Some x86 instructions don't trap → must  │            │
  │  │ binary-translate (scan & rewrite code)   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  With hardware support (VT-x / AMD-V):                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ VMX non-root mode (guest):               │            │
  │  │   Ring 3: guest applications             │            │
  │  │   Ring 0: guest kernel (full privileges  │            │
  │  │           within non-root!)              │            │
  │  │                                          │            │
  │  │ ──── VM Exit ──► (automatic trap)        │            │
  │  │                                          │            │
  │  │ VMX root mode (host):                    │            │
  │  │   Ring 3: host applications              │            │
  │  │   Ring 0: hypervisor/host kernel (KVM)   │            │
  │  │                                          │            │
  │  │ ──── VM Entry ──► (resume guest)         │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Guest runs at REAL ring 0, but in "non-root" mode      │
  │  No binary translation needed!                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VT-x (Intel VMX) Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Key VT-x concepts:                                      │
  │                                                           │
  │  VMCS (Virtual Machine Control Structure):               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Per-vCPU data structure (4KB page)        │            │
  │  │                                          │            │
  │  │ Guest state area:                        │            │
  │  │   - CR0, CR3, CR4                        │            │
  │  │   - RSP, RIP, RFLAGS                     │            │
  │  │   - CS, SS, DS, ES, FS, GS selectors     │            │
  │  │   - GDTR, LDTR, IDTR, TR                 │            │
  │  │   - MSRs (IA32_EFER, etc.)               │            │
  │  │                                          │            │
  │  │ Host state area:                         │            │
  │  │   - CR0, CR3, CR4                        │            │
  │  │   - RSP, RIP (entry point after VM exit) │            │
  │  │   - Segment selectors                    │            │
  │  │                                          │            │
  │  │ VM-execution control fields:             │            │
  │  │   - Which operations cause VM exits      │            │
  │  │   - Exception bitmap (which exceptions   │            │
  │  │     trap to hypervisor)                  │            │
  │  │   - I/O bitmap (which ports trap)        │            │
  │  │   - MSR bitmaps (which MSRs trap)        │            │
  │  │   - EPT pointer (nested page tables)     │            │
  │  │                                          │            │
  │  │ VM-exit information:                     │            │
  │  │   - Exit reason (what caused the exit)   │            │
  │  │   - Exit qualification (additional info) │            │
  │  │   - Guest linear/physical address        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  VMX instructions:                                       │
  │  ┌──────────────┬──────────────────────────────────┐    │
  │  │ VMXON        │ Enable VMX operation              │    │
  │  │ VMXOFF       │ Disable VMX operation             │    │
  │  │ VMLAUNCH     │ First entry into guest            │    │
  │  │ VMRESUME     │ Subsequent entries into guest     │    │
  │  │ VMREAD/WRITE │ Read/write VMCS fields            │    │
  │  │ VMPTRLD      │ Load VMCS pointer                 │    │
  │  │ VMCLEAR      │ Clear VMCS state                  │    │
  │  │ INVEPT       │ Invalidate EPT TLB entries        │    │
  │  │ INVVPID      │ Invalidate VPID TLB entries       │    │
  │  └──────────────┴──────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. VM Exits

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  VM Exit: guest → hypervisor transition (expensive!)     │
  │  Each VM exit costs ~500-2000 CPU cycles                 │
  │                                                           │
  │  Common VM exit reasons:                                 │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │ Exit Reason     │ Cause                          │   │
  │  ├──────────────────────────────────────────────────┤   │
  │  │ EPT violation   │ Guest accesses unmapped GPA    │   │
  │  │                 │ (memory virtualization fault)  │   │
  │  │ I/O instruction │ IN/OUT to emulated port       │   │
  │  │ HLT             │ Guest CPU halted (idle)       │   │
  │  │ CPUID           │ Guest queries CPU features    │   │
  │  │ MSR access      │ Read/write model-specific reg │   │
  │  │ CR access       │ Guest modifies control regs   │   │
  │  │ External int    │ Host interrupt while in guest │   │
  │  │ Preemption timer│ Guest time slice expired      │   │
  │  │ VMCALL          │ Guest intentional hypercall   │   │
  │  │ Exception       │ Guest exception (if in bitmap)│   │
  │  │ XSETBV          │ Guest sets XCR0               │   │
  │  └──────────────────────────────────────────────────┘   │
  │                                                           │
  │  VM exit path:                                           │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │ 1. CPU saves guest state to VMCS guest area     │   │
  │  │ 2. CPU loads host state from VMCS host area     │   │
  │  │ 3. CPU jumps to host RIP (VM exit handler)      │   │
  │  │ 4. KVM handles the exit reason                  │   │
  │  │ 5. KVM does VMRESUME to re-enter guest          │   │
  │  └──────────────────────────────────────────────────┘   │
  │                                                           │
  │  Performance key: MINIMIZE VM exits                      │
  │  - Use EPT (avoid exits for page table walks)           │
  │  - Use MSR bitmaps (skip exits for safe MSRs)          │
  │  - Use I/O bitmaps (skip exits for safe ports)         │
  │  - Use posted interrupts (deliver without exit)        │
  │  - Use virtio (batched I/O, minimal exits)             │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. EPT (Extended Page Tables)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Without EPT (shadow page tables):                       │
  │  Guest VA → Guest PT → GPA → HYPERVISOR → HPA           │
  │  Hypervisor maintains "shadow" page tables               │
  │  Every guest PT change → VM exit → expensive!            │
  │                                                           │
  │  With EPT (hardware 2D page walk):                       │
  │  Guest VA → Guest PT → GPA → EPT → HPA                  │
  │  Both translations done IN HARDWARE (no VM exit)         │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest Virtual Address (GVA)              │            │
  │  │        │                                 │            │
  │  │        ▼ (guest page table walk)         │            │
  │  │ Guest Physical Address (GPA)             │            │
  │  │        │                                 │            │
  │  │        ▼ (EPT walk — hardware!)          │            │
  │  │ Host Physical Address (HPA)              │            │
  │  │        │                                 │            │
  │  │        ▼                                 │            │
  │  │ Physical RAM                             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  EPT structure (4-level, like regular x86 page tables): │
  │  PML4E → PDPTE → PDE → PTE → HPA                       │
  │                                                           │
  │  EPT huge pages:                                         │
  │  - 2MB pages: skip PTE level (1GB PDPT → 2MB PDE → HPA)│
  │  - 1GB pages: skip PDE + PTE levels                     │
  │  - Fewer TLB misses for large VMs                       │
  │                                                           │
  │  Cost: 2D page walk = up to 24 memory accesses          │
  │  (4 levels guest × (4 levels EPT + 1 each access) + 4) │
  │  Worst case from 4 (native) to 24 memory accesses       │
  │  Mitigated by: large EPT pages, VPID TLB tags           │
  │                                                           │
  │  VPID (Virtual Processor ID):                            │
  │  - Tags TLB entries with vCPU ID                         │
  │  - VM exit no longer needs TLB flush                     │
  │  - Both host and guest entries coexist in TLB            │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. ARM Virtualization (EL2)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM Exception Levels:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ EL0: Application (user mode)             │            │
  │  │ EL1: OS Kernel (privileged)              │            │
  │  │ EL2: Hypervisor (virtualization)         │            │
  │  │ EL3: Secure Monitor (TrustZone)          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  VHE (Virtualization Host Extensions, ARMv8.1):          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Without VHE:                             │            │
  │  │   Host kernel runs at EL2 (awkward)      │            │
  │  │   Guest kernel runs at EL1               │            │
  │  │   Host→Guest transition: EL2→EL1         │            │
  │  │                                          │            │
  │  │ With VHE:                                │            │
  │  │   Host kernel runs at EL2 (transparent)  │            │
  │  │   EL2 looks like EL1 to the host kernel  │            │
  │  │   Guest kernel at EL1 (below hypervisor) │            │
  │  │   No need for special host kernel build  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Stage 2 translation (ARM's EPT equivalent):             │
  │  Guest VA → Stage 1 (guest PT) → IPA →                  │
  │  Stage 2 (hypervisor PT) → PA                            │
  │  Same concept as Intel EPT, different terminology        │
  │                                                           │
  │  IPA = Intermediate Physical Address (≈ GPA in Intel)   │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain VMX root/non-root modes and how VM exits work.**
**A:** Intel VT-x introduces two CPU operating modes: VMX root mode (host/hypervisor) and VMX non-root mode (guest). Both modes have the full ring 0-3 privilege levels, so the guest kernel runs at actual ring 0 within non-root mode — no binary translation or ring deprivileging needed. Key data structure: VMCS (Virtual Machine Control Structure), a 4KB per-vCPU structure containing guest state (registers, CR3, segment descriptors), host state (entry point after exit), and control fields (what causes VM exits). A **VM exit** occurs when the guest performs an operation that must be handled by the hypervisor — the CPU automatically saves guest state into VMCS, loads host state, and jumps to the VM exit handler. Common exit reasons: EPT violation (unmapped memory), I/O instruction (emulated device), HLT (idle), CPUID (feature query), MSR access, external interrupt. Each exit costs 500-2000 cycles (context save/restore). Performance optimization focuses on minimizing exits: EPT eliminates exits for page table walks, MSR bitmaps skip exits for safe MSRs, posted interrupts deliver guest interrupts without exiting, and virtio batches I/O operations.

**Q2: How does EPT improve virtualization performance compared to shadow page tables?**
**A:** Without EPT, the hypervisor must maintain "shadow page tables" that map Guest Virtual Address (GVA) directly to Host Physical Address (HPA). Every time the guest modifies its page tables (CR3 load, PTE update), a VM exit occurs so the hypervisor can update the shadow. This is extremely expensive — page-heavy workloads (fork, mmap, memory allocation) generate thousands of exits per second. With EPT, the hardware performs a two-dimensional page walk: GVA → guest page table → Guest Physical Address (GPA) → EPT → Host Physical Address (HPA). Both translations happen in hardware without VM exits. The guest can freely modify its own page tables (no exits). EPT violations only occur for truly unmapped GPAs (lazy allocation, MMIO). The cost: a 2D walk can require up to 24 memory accesses (worst case) compared to 4 for native. Mitigations: (1) EPT huge pages (2MB/1GB) reduce walk depth. (2) VPID (Virtual Processor ID) tags TLB entries per-vCPU, so VM exits don't require TLB flushes — both host and guest translations coexist in hardware TLB. In practice, EPT's elimination of shadow page table exits far outweighs the 2D walk overhead.

---

## Summary

- VT-x/AMD-V: hardware CPU modes (root/non-root) let guest run at real ring 0
- VMCS: per-vCPU structure with guest state, host state, exit control fields
- VM exits: guest→host trap, costs 500-2000 cycles, minimize for performance
- Common exits: EPT violation, I/O, HLT, CPUID, MSR, external interrupts
- EPT: hardware 2-level page walk (GVA→GPA→HPA), eliminates shadow page table exits
- VPID: TLB tags per-vCPU, no TLB flush on VM exit
- ARM: EL2 for hypervisor, Stage 2 translation (like EPT), VHE for transparent host mode

---

[Previous: Container Networking ←](Chapter_18_Container_Networking.md) | [Next: KVM Architecture →](Chapter_20_KVM_Architecture.md)
