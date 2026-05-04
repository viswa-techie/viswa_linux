# Chapter 20: Secure Boot

## Learning Goals
- Understand the UEFI Secure Boot chain of trust
- Know how Linux implements verified boot
- Understand kernel module signing and lockdown
- Know Android Verified Boot (AVB)

---

## 20.1 Secure Boot Chain of Trust

```
Secure Boot: Verify each boot stage before executing it.
Prevents bootkits, rootkits, and unauthorized OS loading.

Chain of trust:
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Hardware Root of Trust                                 │
  │    ROM code / UEFI firmware (immutable)                 │
  │    Contains Platform Key (PK)                           │
  │         │                                               │
  │         ▼  verify signature                             │
  │  UEFI Firmware                                          │
  │    Verifies bootloader against Key Exchange Key (KEK)   │
  │    db (allowed) / dbx (denied) signature databases      │
  │         │                                               │
  │         ▼  verify signature                             │
  │  Bootloader (GRUB2 / systemd-boot / shim)              │
  │    Signed with vendor key (Microsoft or distro)         │
  │    Loads and verifies kernel                            │
  │         │                                               │
  │         ▼  verify signature                             │
  │  Linux Kernel                                           │
  │    Signed kernel image                                  │
  │    Kernel verifies modules (CONFIG_MODULE_SIG)          │
  │         │                                               │
  │         ▼  verify signature                             │
  │  Kernel Modules                                         │
  │    Each .ko file signed with kernel build key           │
  │         │                                               │
  │         ▼  (optional)                                   │
  │  IMA/EVM verify user-space binaries                     │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  Each stage verifies the next before executing it.
  Compromise at any stage breaks the chain.
```

---

## 20.2 UEFI Secure Boot Keys

```
UEFI key hierarchy:

  Platform Key (PK):
    - Single key, owned by hardware vendor (OEM)
    - Controls who can modify KEK database
    - Factory-installed in firmware

  Key Exchange Key (KEK):
    - Can modify db/dbx databases
    - Typically: Microsoft KEK + vendor KEK
    - Allows authorized parties to update allowed signatures

  Signature Database (db):
    - List of allowed signing certificates/hashes
    - Bootloaders signed by these keys can execute
    - Microsoft UEFI CA: signs third-party bootloaders
    - Distro keys: signs distro-specific bootloaders

  Forbidden Database (dbx):
    - Revoked certificates/hashes
    - Known-compromised bootloaders blocked here
    - Updated via firmware updates

  ┌──────────────────────────────────────────────┐
  │  UEFI Key Hierarchy                           │
  │                                               │
  │  PK (Platform Key)                            │
  │    └── Authorizes changes to KEK              │
  │                                               │
  │  KEK (Key Exchange Key)                       │
  │    └── Authorizes changes to db/dbx           │
  │                                               │
  │  db (Allowed signatures)                      │
  │    ├── Microsoft Windows Production PCA       │
  │    ├── Microsoft UEFI CA (third-party)        │
  │    └── Distro signing key                     │
  │                                               │
  │  dbx (Denied signatures)                      │
  │    └── Revoked bootloaders                    │
  └──────────────────────────────────────────────┘
```

---

## 20.3 Linux Shim Bootloader

```
Problem: Linux distros need Microsoft's signature for Secure Boot.
Solution: "shim" — small bootloader signed by Microsoft.

Boot flow with shim:
  UEFI firmware
    ↓ verify against db
  shim.efi (signed by Microsoft UEFI CA)
    ↓ verify against MOK (Machine Owner Key) or shim's built-in key
  grubx64.efi (signed by distro key)
    ↓ verify
  vmlinuz (signed by distro key)
    ↓ verify
  kernel modules (.ko signed by kernel build key)

MOK (Machine Owner Key):
  User-managed key database stored in UEFI variables.
  Allows users to sign their own kernels/modules.

  # Enroll a MOK:
  mokutil --import my-key.der
  # Reboot → MokManager prompts to confirm enrollment

  # Sign a kernel module with MOK:
  /usr/src/kernels/$(uname -r)/scripts/sign-file \
    sha256 my-key.priv my-key.der mymodule.ko
```

---

## 20.4 Kernel Module Signing

```
Kernel can enforce that only signed modules are loaded.

Config:
  CONFIG_MODULE_SIG=y           # Enable module signature verification
  CONFIG_MODULE_SIG_FORCE=y     # Reject unsigned modules
  CONFIG_MODULE_SIG_SHA256=y    # Hash algorithm
  CONFIG_MODULE_SIG_KEY="certs/signing_key.pem"

Module signing at build:
  Scripts automatically sign modules during make modules_install
  Key pair generated at build time or provided externally

Verification flow:
  ┌──────────────┐   insmod/modprobe  ┌────────────────┐
  │ User request │ ─────────────────► │ Kernel module   │
  │ load module  │                    │ loader          │
  └──────────────┘                    └───────┬────────┘
                                              │
                                    ┌─────────▼────────┐
                                    │ Check signature:  │
                                    │ 1. Extract sig    │
                                    │ 2. Verify against │
                                    │    trusted keyring│
                                    │ 3. Verify hash    │
                                    └─────────┬────────┘
                                              │
                                    ┌─────────▼────────┐
                                    │ SIG_FORCE=y:     │
                                    │   Bad sig → REJECT│
                                    │ SIG_FORCE=n:     │
                                    │   Bad sig → WARN  │
                                    │   (taint kernel)  │
                                    └──────────────────┘

Kernel keyrings:
  .builtin_trusted_keys — keys compiled into kernel
  .secondary_trusted_keys — keys added at runtime
  .platform — UEFI db keys
  .machine — MOK keys (from shim)
```

---

## 20.5 Kernel Lockdown

```
Lockdown: Restrict what root can do when Secure Boot is active.

  Without lockdown:
    Root can: load unsigned modules, access /dev/mem,
    write to MSRs, use kprobes, modify ACPI tables
    → Root can subvert Secure Boot by patching the running kernel

  With lockdown:
    These dangerous operations are BLOCKED even for root.

  Two lockdown levels:
    integrity:  Prevent modifications to running kernel
    confidentiality: Also prevent reading kernel secrets

  ┌──────────────────┬─────────────┬───────────────────┐
  │ Operation        │ integrity   │ confidentiality    │
  ├──────────────────┼─────────────┼───────────────────┤
  │ Unsigned modules │ Blocked     │ Blocked            │
  │ /dev/mem access  │ Blocked     │ Blocked            │
  │ /dev/kmem        │ Blocked     │ Blocked            │
  │ kexec unsigned   │ Blocked     │ Blocked            │
  │ MSR writes       │ Blocked     │ Blocked            │
  │ kprobes          │ Allowed     │ Blocked            │
  │ /proc/kcore      │ Allowed     │ Blocked            │
  │ BPF read kernel  │ Allowed     │ Blocked            │
  │ perf events      │ Allowed     │ Blocked            │
  └──────────────────┴─────────────┴───────────────────┘

Config:
  CONFIG_SECURITY_LOCKDOWN_LSM=y
  CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT=y  # Auto-enable when UEFI SB on
  Boot param: lockdown=integrity|confidentiality
```

---

## 20.6 Android Verified Boot (AVB)

```
Android Verified Boot: Ensures device runs authorized firmware/OS.

  AVB 2.0 (dm-verity based):
  ┌─────────────────────────────────────────────────┐
  │  Boot Flow                                       │
  │                                                  │
  │  Bootloader (verified by hardware root of trust) │
  │    ↓ verify vbmeta signature                     │
  │  vbmeta partition                                │
  │    Contains hashes of: boot, system, vendor      │
  │    Signed with OEM key                           │
  │    ↓                                             │
  │  boot partition (kernel + ramdisk)               │
  │    Verified by hash in vbmeta                    │
  │    ↓                                             │
  │  system/vendor partitions                        │
  │    dm-verity: block-level hash verification      │
  │    Every 4KB block verified on read              │
  └─────────────────────────────────────────────────┘

  dm-verity:
    Hash tree built during image creation.
    Root hash stored in vbmeta.
    On read: kernel computes block hash, compares with tree.
    Mismatch → I/O error → partition corruption detected.

  Boot states:
    Green:   Fully verified, OEM keys
    Yellow:  Verified with user-installed root of trust
    Orange:  Unlocked bootloader (unverified)
    Red:     Verification failed

  Rollback protection:
    vbmeta contains rollback index.
    Stored in tamper-evident storage (RPMB).
    Cannot install older (potentially vulnerable) firmware.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/lockdown/lockdown.c | Kernel lockdown LSM |
| certs/system_keyring.c | Trusted key management |
| kernel/module/signing.c | Module signature verification |
| drivers/md/dm-verity-target.c | dm-verity implementation |
| arch/x86/kernel/setup.c | UEFI Secure Boot detection |
| include/linux/security.h | Lockdown hooks |

---

## Interview Questions

**Q1: How does UEFI Secure Boot establish a chain of trust?**
A: The chain starts with the Platform Key (PK) in firmware ROM — the hardware root of trust. PK authorizes the Key Exchange Key (KEK), which manages the signature database (db/dbx). On boot, UEFI firmware verifies the bootloader's signature against the db. The bootloader (typically shim, signed by Microsoft) verifies GRUB, which verifies the kernel. The kernel verifies loadable modules against its built-in keys. Each stage cryptographically verifies the next before executing it. If any signature check fails, boot halts. The dbx database contains revoked signatures for known-compromised components.

**Q2: Why is kernel lockdown necessary with Secure Boot?**
A: Without lockdown, root can bypass the entire Secure Boot chain on a running system: load unsigned kernel modules, write to /dev/mem to patch kernel code, modify MSRs, or use kexec to load an unsigned kernel. Lockdown blocks these operations even for root when Secure Boot is active. The integrity level prevents kernel modification (no unsigned modules, no /dev/mem, no kexec unsigned). The confidentiality level additionally prevents reading kernel secrets (no /proc/kcore, restricted BPF, restricted perf). Lockdown closes the gap between verified boot and runtime integrity.

**Q3: How does Android Verified Boot use dm-verity?**
A: AVB computes a hash tree over the system/vendor partitions during image creation. The root hash is stored in the vbmeta partition, which is signed with the OEM's key. At boot, the bootloader verifies vbmeta's signature. The kernel then sets up dm-verity on the system partition using the verified root hash. On every 4KB block read, the kernel computes the block's hash and verifies it against the hash tree. Any mismatch (corruption or tampering) returns an I/O error. This provides continuous runtime verification — every block is checked when accessed, not just at boot. Combined with rollback protection (rollback index in RPMB), AVB prevents both modification and downgrade attacks.

---

## Summary

- Secure Boot: each boot stage verifies the next via cryptographic signatures
- UEFI keys: PK → KEK → db (allowed) / dbx (denied)
- Shim: Microsoft-signed bootloader that chains to distro bootloaders
- Module signing: kernel rejects unsigned modules (CONFIG_MODULE_SIG_FORCE)
- Lockdown: blocks root from bypassing Secure Boot at runtime
- MOK: Machine Owner Key for user-managed signing
- Android AVB: vbmeta + dm-verity for block-level runtime verification
- Rollback protection: prevents installing older vulnerable firmware

---

Next: [Chapter 21 — Cryptographic Services](Chapter_21_Crypto_Services.md)
