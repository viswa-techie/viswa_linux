# Chapter 32: Embedded Linux Security

## Learning Goals
- Understand security challenges unique to embedded Linux systems
- Know secure boot, OTA updates, and hardware security for embedded
- Understand automotive and IoT security requirements
- Know practical hardening for resource-constrained devices

---

## 32.1 Embedded Security Challenges

```
Embedded Linux has unique security constraints:

  ┌──────────────────────────────────────────────────────┐
  │  Embedded Security Challenges                        │
  │                                                       │
  │  1. Physical access: Attacker has the device          │
  │     - JTAG/UART debug ports exposed                   │
  │     - Flash chips can be read/modified                │
  │     - Side-channel attacks possible                   │
  │                                                       │
  │  2. Long lifecycle: 10-15 years in automotive         │
  │     - Must receive security updates for years         │
  │     - Hardware cannot be upgraded                     │
  │                                                       │
  │  3. Limited resources:                                │
  │     - Small RAM/flash → minimal security stack        │
  │     - Slow CPU → lightweight crypto                   │
  │     - No TPM on some devices                          │
  │                                                       │
  │  4. Network exposure:                                 │
  │     - IoT devices constantly connected                │
  │     - Automotive: CAN bus, V2X, OBD-II                │
  │     - Often behind weak or no firewall                │
  │                                                       │
  │  5. Supply chain:                                     │
  │     - Third-party BSP (Board Support Package)         │
  │     - Vendor blobs, closed-source drivers             │
  │     - Difficult to audit entire stack                 │
  └──────────────────────────────────────────────────────┘
```

---

## 32.2 Secure Boot for Embedded

```
Embedded secure boot: Hardware-rooted trust chain.

  ┌──────────────────────────────────────────────────────┐
  │  Typical Embedded Secure Boot Chain                   │
  │                                                       │
  │  ROM Bootloader (SoC internal, immutable)             │
  │    Contains OEM public key hash (burned in eFuses)    │
  │    │                                                  │
  │    │ verify signature                                 │
  │    ▼                                                  │
  │  SPL / First-stage bootloader (e.g., U-Boot SPL)      │
  │    Signed with OEM private key                        │
  │    │                                                  │
  │    │ verify signature                                 │
  │    ▼                                                  │
  │  U-Boot (main bootloader)                              │
  │    FIT image with signatures                          │
  │    │                                                  │
  │    │ verify signature                                 │
  │    ▼                                                  │
  │  Linux Kernel + DTB                                    │
  │    Verified by U-Boot                                 │
  │    │                                                  │
  │    │ dm-verity                                        │
  │    ▼                                                  │
  │  Root Filesystem                                       │
  │    Block-level verification via dm-verity              │
  └──────────────────────────────────────────────────────┘

  U-Boot verified boot:
    FIT image format includes:
      - Kernel image
      - Device tree blob
      - Optional ramdisk
      - SHA256 hashes
      - RSA signature of hashes

  eFuses: One-time programmable bits in SoC
    - Store public key hash
    - Enable/disable JTAG
    - Enable/disable boot from USB/SD
    - Cannot be changed once blown
```

---

## 32.3 Secure OTA Updates

```
OTA (Over-The-Air) update security for embedded:

  Requirements:
    1. Authenticity: Update from legitimate source (signed)
    2. Integrity: Update not corrupted in transit (hash)
    3. Rollback protection: Cannot install older vulnerable version
    4. Atomic: Either fully applied or not at all (A/B partitioning)
    5. Fail-safe: Boot previous version if update fails

  A/B Partition Scheme:
  ┌────────────────────────────────────────────┐
  │  Flash Layout                               │
  │                                             │
  │  ┌──────────┐  Active (A)                   │
  │  │ boot_a   │  Currently running            │
  │  │ system_a │                               │
  │  │ vendor_a │                               │
  │  └──────────┘                               │
  │  ┌──────────┐  Standby (B)                  │
  │  │ boot_b   │  Receive update here          │
  │  │ system_b │  Verify before switching      │
  │  │ vendor_b │                               │
  │  └──────────┘                               │
  │  ┌──────────┐                               │
  │  │ userdata │  Preserved across updates     │
  │  └──────────┘                               │
  │  ┌──────────┐                               │
  │  │ vbmeta   │  Rollback index + hashes      │
  │  └──────────┘                               │
  └────────────────────────────────────────────┘

  Update flow:
    1. Download update package (encrypted channel)
    2. Verify package signature (RSA/ECDSA)
    3. Check rollback index ≥ current
    4. Write to standby partition (B)
    5. Verify written data (dm-verity hash tree)
    6. Mark B as active (atomic flag update)
    7. Reboot → boot from B
    8. If boot fails → rollback to A

  Tools: RAUC, SWUpdate, Mender, Android OTA
```

---

## 32.4 Hardware Security Features

```
SoC security features for embedded:

  ARM TrustZone:
    Two execution environments on same CPU:
    ┌──────────────────┬──────────────────┐
    │   Normal World   │  Secure World    │
    │   (Linux/RTOS)   │  (OP-TEE/ATF)    │
    │                  │                  │
    │   Applications   │  Trusted Apps    │
    │   Kernel         │  Secure OS       │
    │                  │  Crypto, Keys    │
    │                  │  DRM, Biometrics │
    └──────────────────┴──────────────────┘
    Hardware isolation — normal world cannot access secure world memory.

  OP-TEE (Open Portable Trusted Execution Environment):
    Open-source TEE for TrustZone.
    Provides: secure storage, crypto, key management.
    Linux communicates via TEE driver and GlobalPlatform API.

  Hardware crypto:
    Many SoCs have dedicated crypto accelerators:
      - NXP CAAM: random number gen + AES + SHA
      - Qualcomm CE: crypto engine
      - Microchip CryptoAuth: secure element
    Used for: key storage, encrypt/decrypt, secure boot verification.

  RPMB (Replay Protected Memory Block):
    eMMC/UFS feature for tamper-proof storage.
    Used for: rollback counters, secure flags.
    Authenticated read/write via HMAC.
```

---

## 32.5 Automotive Security

```
Automotive Linux security — safety + security combined.

  Standards:
    ISO/SAE 21434: Cybersecurity for road vehicles
    UNECE WP.29: Vehicle cybersecurity regulation
    ISO 26262: Functional safety (ASIL levels)
    AUTOSAR SecOC: Secure on-board communication

  Automotive attack surfaces:
  ┌──────────────────────────────────────────────────────┐
  │  ┌─────────┐  ┌─────────┐  ┌──────────┐             │
  │  │ V2X/5G  │  │ OBD-II  │  │ WiFi/BT  │  External   │
  │  └────┬────┘  └────┬────┘  └────┬─────┘  Interfaces │
  │       │            │            │                     │
  │  ┌────▼────────────▼────────────▼─────┐              │
  │  │    Gateway ECU (firewall)          │              │
  │  └────┬────────────┬─────────────┬────┘              │
  │       │            │             │                    │
  │  ┌────▼────┐  ┌────▼────┐  ┌────▼────┐              │
  │  │Infotain │  │  ADAS   │  │ Body    │  Internal    │
  │  │ment ECU │  │  ECU    │  │ Control │  Network     │
  │  │(Android)│  │(Safety) │  │  ECU    │  (CAN/ETH)   │
  │  └─────────┘  └─────────┘  └─────────┘              │
  └──────────────────────────────────────────────────────┘

  Security measures:
    - Secure boot for each ECU
    - Firewall between external and internal networks
    - Message authentication on CAN bus (SecOC)
    - OTA updates with rollback protection
    - Hypervisor isolation for mixed-criticality systems
    - IDS/IPS for network monitoring
    - HSM (Hardware Security Module) for key storage

  AGL (Automotive Grade Linux):
    Uses Smack as MAC mechanism
    Cynara: policy framework for service access
    D-Bus security policies
    Mandatory app signing
```

---

## 32.6 IoT Security Best Practices

```
Minimum security for IoT/embedded Linux devices:

  ┌────────────────────────────────────────────────────────┐
  │  Embedded Linux Security Checklist                      │
  │                                                         │
  │  Boot security:                                         │
  │  □ Secure boot with hardware root of trust              │
  │  □ Disable JTAG/UART debug ports in production          │
  │  □ Blow eFuses to lock boot configuration               │
  │  □ dm-verity on root filesystem                         │
  │                                                         │
  │  Kernel hardening:                                      │
  │  □ Enable STRICT_KERNEL_RWX, STACKPROTECTOR             │
  │  □ Disable unnecessary kernel features/modules          │
  │  □ Enable KASLR, FORTIFY_SOURCE                         │
  │  □ kptr_restrict=2, dmesg_restrict=1                    │
  │                                                         │
  │  User space:                                            │
  │  □ Minimize installed packages (BusyBox minimal)        │
  │  □ Read-only root filesystem                            │
  │  □ Separate writable partition (data only)              │
  │  □ Drop privileges for services (non-root)              │
  │  □ MAC (SELinux/Smack) for process isolation            │
  │                                                         │
  │  Network:                                               │
  │  □ TLS for all communications (no plain HTTP)           │
  │  □ Firewall: default deny, allow only needed ports      │
  │  □ Certificate pinning for cloud connections            │
  │  □ Disable unused network services                      │
  │                                                         │
  │  Updates:                                               │
  │  □ Signed OTA updates with rollback protection          │
  │  □ A/B partitioning for atomic updates                  │
  │  □ Automatic security patch delivery                    │
  │  □ SBOM (Software Bill of Materials) tracking           │
  │                                                         │
  │  Credentials:                                           │
  │  □ Unique per-device keys (not shared across devices)   │
  │  □ Hardware key storage (TEE/secure element)            │
  │  □ No hardcoded passwords/keys in firmware              │
  │  □ Certificate-based authentication                     │
  └────────────────────────────────────────────────────────┘
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| drivers/tee/ | TEE (TrustZone) driver framework |
| drivers/char/hw_random/ | Hardware RNG drivers |
| drivers/md/dm-verity-target.c | dm-verity for embedded rootfs |
| drivers/mmc/core/block.c | eMMC RPMB support |
| security/smack/ | Smack (used in AGL) |

---

## Interview Questions

**Q1: What are the unique security challenges for embedded Linux?**
A: (1) Physical access — attacker has the device, can use JTAG, read flash chips, perform side-channel attacks. (2) Long lifecycle — automotive devices need 10-15 years of security updates. (3) Limited resources — constrained RAM/CPU means lighter security stack, no full SELinux. (4) Supply chain — third-party BSPs and vendor blobs are hard to audit. (5) Network exposure — IoT devices often permanently connected with weak firewalling. (6) No user interaction — devices can't prompt for passwords or updates. Mitigations: secure boot with eFuses, dm-verity, signed OTA, TrustZone for key storage, minimal attack surface, Smack or minimal SELinux for MAC.

**Q2: How does secure boot work on an embedded ARM platform?**
A: The SoC ROM bootloader (immutable code in silicon) verifies the first-stage bootloader's (SPL) signature against a public key hash burned into eFuses. The SPL verifies U-Boot's signature. U-Boot verifies a FIT image containing the kernel, device tree, and optional ramdisk — each component has SHA256 hashes signed with RSA. The kernel then uses dm-verity to verify the root filesystem block-by-block using a hash tree whose root hash was verified at boot. eFuses are blown in production to: lock the boot key, disable JTAG debug, disable booting from USB/SD. This creates an unbreakable chain from silicon to userspace.

**Q3: How does A/B partitioning improve OTA update security?**
A: A/B partitioning maintains two complete copies of boot/system/vendor partitions. Updates are written to the standby (inactive) partition while the system continues running from the active partition. After writing, the update is verified (dm-verity hash tree check). Only then is the standby partition marked as the new active boot target. On reboot, if the new partition fails to boot (verified by a boot counter), the system automatically rolls back to the previous working partition. Combined with rollback index checking (stored in RPMB), this prevents: (1) bricked devices from failed updates, (2) boot attacks from partial writes, (3) rollback to older vulnerable versions. The update is atomic — the system is never in a half-updated state.

---

## Summary

- Embedded challenges: physical access, long lifecycle, limited resources, supply chain
- Secure boot: ROM → SPL → U-Boot → kernel → dm-verity rootfs, eFuses lock config
- OTA: signed packages, A/B partitioning, rollback protection (RPMB)
- Hardware security: TrustZone (TEE), RPMB, eFuses, crypto accelerators
- Automotive: ISO 21434, gateway firewalls, CAN bus auth, HSMs, Smack (AGL)
- IoT checklist: secure boot, read-only rootfs, TLS everywhere, no hardcoded keys
- Unique device keys + hardware key storage + certificate-based auth

---

Next: [Chapter 33 — Documentation and References](Chapter_33_References.md)
