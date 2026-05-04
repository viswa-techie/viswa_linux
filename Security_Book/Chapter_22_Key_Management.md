# Chapter 22: Key Management

## Learning Goals
- Understand the Linux kernel keyring subsystem
- Know key types, keyrings, and access control
- Understand hardware key storage (TPM, TEE)
- Know kernel and user-space key management workflows

---

## 22.1 Kernel Keyring Subsystem

```
The kernel provides an in-kernel key management facility.
Stores cryptographic keys, authentication tokens, and certificates.

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │  User Space                                           │
  │  keyctl(2) syscall / keyctl utility                   │
  │  request_key(2) — request key from user-space helper  │
  │  add_key(2) — add key to keyring                      │
  └───────────────────────┬──────────────────────────────┘
                           │
  ┌───────────────────────▼──────────────────────────────┐
  │  Kernel Keyring Subsystem (security/keys/)            │
  │                                                       │
  │  Key Types:                                           │
  │  ┌──────────────┬──────────────────────────────────┐  │
  │  │ Type         │ Purpose                           │  │
  │  ├──────────────┼──────────────────────────────────┤  │
  │  │ user         │ User-defined opaque blob          │  │
  │  │ logon        │ Like user, but never readable     │  │
  │  │              │ from user space (kernel-only)     │  │
  │  │ keyring      │ A keyring containing other keys   │  │
  │  │ asymmetric   │ RSA/EC public/private keys        │  │
  │  │ trusted      │ TPM-sealed keys                   │  │
  │  │ encrypted    │ Encrypted by master key           │  │
  │  │ big_key      │ Large keys (stored in tmpfs)      │  │
  │  │ dns_resolver │ DNS lookup results                │  │
  │  └──────────────┴──────────────────────────────────┘  │
  │                                                       │
  │  Keyring hierarchy:                                   │
  │    @s  — Session keyring (per login session)          │
  │    @u  — User keyring (per UID, persistent)           │
  │    @us — User session keyring                         │
  │    @p  — Process keyring (per process, not inherited) │
  │    @t  — Thread keyring (per thread)                  │
  │                                                       │
  │  Special keyrings:                                    │
  │    .builtin_trusted_keys  — Compiled-in trusted keys  │
  │    .secondary_trusted_keys — Runtime-added keys       │
  │    .machine               — MOK keys from UEFI        │
  │    .platform              — UEFI db keys              │
  │    .ima                   — IMA appraisal keys        │
  └──────────────────────────────────────────────────────┘
```

---

## 22.2 Key Operations

```bash
# keyctl utility examples:

# Add a key to session keyring
keyctl add user mykey "secret_data" @s

# List keys in session keyring
keyctl show @s

# Read a key's payload
keyctl read <key_id>
keyctl pipe <key_id>       # Binary output

# Set permissions on a key
keyctl setperm <key_id> 0x3f3f0000
# Permissions: possessor/user/group/other × view/read/write/search/link/setattr

# Set key timeout (auto-expire)
keyctl timeout <key_id> 3600   # Expire in 1 hour

# Revoke and unlink
keyctl revoke <key_id>
keyctl unlink <key_id> @s

# Request a key (triggers /sbin/request-key if not found)
keyctl request user mykey

# Create a linked keyring
keyctl newring myring @s
keyctl add user subkey "data" <myring_id>
```

---

## 22.3 Key Access Control

```
Each key has ownership and permissions:

  Owner:  UID/GID who created the key
  Permissions: 6 bits × 4 categories

  ┌────────────┬─────┬──────┬───────┬────────┬──────┬─────────┐
  │ Category   │ View│ Read │ Write │ Search │ Link │ Setattr │
  ├────────────┼─────┼──────┼───────┼────────┼──────┼─────────┤
  │ Possessor  │  ✓  │  ✓   │  ✓    │  ✓     │  ✓   │  ✓      │
  │ User       │  ✓  │  ✓   │  —    │  ✓     │  ✓   │  —      │
  │ Group      │  ✓  │  —   │  —    │  —     │  —   │  —      │
  │ Other      │  —  │  —   │  —    │  —     │  —   │  —      │
  └────────────┴─────┴──────┴───────┴────────┴──────┴─────────┘

Possessor: anyone who possesses the key (has it in their keyring).
  Different from owner — possession is about keyring linkage.

  Security note:
    "logon" type keys are NEVER readable from user space.
    The kernel can use them (e.g., for fscrypt, dm-crypt)
    but no process can read the raw key material.
    This prevents key extraction by compromised user-space.
```

---

## 22.4 Trusted and Encrypted Keys

```
Trusted keys: Sealed by TPM — key material never leaves TPM.

  ┌──────────────────────────────────────────────────┐
  │  Trusted Key Lifecycle:                           │
  │                                                   │
  │  Create:                                          │
  │    keyctl add trusted mykey "new 256" @s          │
  │    → TPM generates random key                    │
  │    → TPM encrypts (seals) with its Storage Key   │
  │    → Sealed blob stored in kernel keyring        │
  │                                                   │
  │  Use:                                             │
  │    Kernel uses unsealed key for crypto operations │
  │    Key material exists in kernel memory only      │
  │    User space never sees raw key                  │
  │                                                   │
  │  Save to disk:                                    │
  │    keyctl pipe <key_id> > /etc/keys/mykey.blob    │
  │    Blob is TPM-encrypted — useless without TPM    │
  │                                                   │
  │  Reload:                                          │
  │    keyctl add trusted mykey "load $(cat blob)" @s │
  │    → TPM unseals the key                         │
  └──────────────────────────────────────────────────┘

Encrypted keys: Encrypted by a master key (trusted or user key).

  # Create encrypted key using trusted master key
  keyctl add trusted master "new 256" @s
  keyctl add encrypted myekey "new ecryptfs master 256" @s

  Hierarchy:
    TPM → seals → trusted key → encrypts → encrypted key
    No key material ever in plaintext outside kernel + TPM.
```

---

## 22.5 TPM (Trusted Platform Module)

```
TPM: Hardware security module for key storage and attestation.

  Capabilities:
    - Key generation and storage (RSA, ECC)
    - Sealing/unsealing data to system state (PCRs)
    - Attestation (prove system configuration)
    - Random number generation
    - Monotonic counter

  ┌───────────────────────────────────────────────────┐
  │  TPM 2.0 Architecture                             │
  │                                                    │
  │  PCR (Platform Configuration Registers):           │
  │    PCR[0]:  BIOS/UEFI firmware hash               │
  │    PCR[1]:  Platform config                       │
  │    PCR[2]:  Option ROM code                       │
  │    PCR[3]:  Option ROM config                     │
  │    PCR[4]:  MBR / bootloader                      │
  │    PCR[5]:  MBR config / GPT table                │
  │    PCR[7]:  Secure Boot policy                    │
  │    PCR[8]:  Kernel command line                   │
  │    PCR[9]:  Kernel image                          │
  │                                                    │
  │  Each PCR is extend-only:                          │
  │    PCR[n] = Hash(PCR[n] || new_measurement)        │
  │    Cannot be set to arbitrary value                │
  │    Only reset at boot                              │
  │                                                    │
  │  Sealing: encrypt data bound to PCR values         │
  │    If PCR values change (different firmware/kernel) │
  │    → sealed data cannot be decrypted               │
  │    → detects tampering                             │
  └───────────────────────────────────────────────────┘

  Linux TPM interface:
    /dev/tpm0          — Direct TPM access
    /dev/tpmrm0        — Resource-managed TPM access
    tpm2_* commands    — User-space tools
```

---

## 22.6 Key Management for Filesystem Encryption

```
fscrypt: Per-file encryption using kernel keyring.

  Key provisioning flow:
    1. User enters passphrase
    2. KDF derives master key (Argon2/HKDF)
    3. Master key added to session keyring as "logon" type
    4. fscrypt derives per-file keys from master key
    5. Files encrypted/decrypted transparently

  ┌──────────┐  passphrase  ┌──────────┐  add_key  ┌──────────┐
  │ User     │ ────────────►│ KDF      │ ─────────►│ Keyring  │
  │          │              │ (Argon2) │           │ (logon)  │
  └──────────┘              └──────────┘           └──────┬───┘
                                                          │
                    ┌─────────────────────────────────────▼─┐
                    │ fscrypt                                │
                    │ Derive per-file key from master key    │
                    │ AES-256-XTS for file contents          │
                    │ AES-256-CTS-CBC for filenames          │
                    │ Per-file nonce for key derivation      │
                    └───────────────────────────────────────┘

  Android key management:
    Keymaster/KeyMint HAL → TEE (TrustZone)
    Hardware-bound keys + auth-bound keys
    Key material never leaves TEE
    Biometric/PIN unlock gates key access
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/keys/keyring.c | Keyring management |
| security/keys/key.c | Key lifecycle |
| security/keys/keyctl.c | keyctl syscall handlers |
| security/keys/trusted-keys/ | TPM-sealed trusted keys |
| security/keys/encrypted-keys/ | Encrypted key type |
| security/keys/process_keys.c | Process/session keyring |
| fs/crypto/ | fscrypt key management |
| drivers/char/tpm/ | TPM driver subsystem |

---

## Interview Questions

**Q1: How does the kernel keyring subsystem manage cryptographic keys?**
A: The keyring subsystem provides in-kernel key storage with keyrings organized hierarchically: thread → process → session → user. Keys have types (user, logon, trusted, encrypted, asymmetric) defining their behavior. Access control uses 24-bit permissions across 4 categories (possessor/user/group/other) with 6 operations (view/read/write/search/link/setattr). "logon" type keys never expose raw material to user space — the kernel uses them directly for crypto operations. The `keyctl(2)` syscall manages keys. Keys can auto-expire via timeouts. The system supports key request upcalls to user-space helpers when a key isn't found.

**Q2: What are trusted keys and why are they more secure than regular keys?**
A: Trusted keys have their material generated and sealed by a TPM (Trusted Platform Module). The raw key never exists outside the TPM and kernel memory — user space only sees the TPM-encrypted blob. Creating a trusted key: the TPM generates random key material, encrypts it with its storage root key, and returns the sealed blob. The kernel holds the unsealed key in memory for crypto operations. When saved to disk, only the sealed blob is stored — it's useless without the same TPM. Keys can be sealed to specific PCR values (platform state), so they're only usable if the system hasn't been tampered with. This provides hardware-rooted key protection.

**Q3: How does fscrypt use the kernel keyring for filesystem encryption?**
A: fscrypt stores master keys in the kernel keyring as "logon" type keys (never readable from user space). The user provides a passphrase, a KDF (Argon2/HKDF) derives the master key, and it's added to the session keyring via `add_key()`. fscrypt derives unique per-file keys from the master key using HKDF with per-file nonces stored in the file's extended attributes. File contents use AES-256-XTS, filenames use AES-256-CTS-CBC. The kernel handles encryption/decryption transparently on I/O. When the key is removed from the keyring (logout/lock), files become unreadable. Android uses this with hardware-bound keys from the TEE/KeyMint HAL.

---

## Summary

- Kernel keyring: in-kernel key storage with access control
- Key types: user (opaque), logon (kernel-only), trusted (TPM), encrypted, asymmetric
- Keyring hierarchy: thread → process → session → user
- Special keyrings: .builtin_trusted_keys, .machine (MOK), .platform (UEFI)
- Trusted keys: TPM-sealed, raw material never leaves TPM + kernel
- Encrypted keys: wrapped by a master key (trusted or user)
- TPM: PCRs, sealing, attestation, key generation
- fscrypt: per-file encryption with logon keys from keyring

---

Next: [Chapter 23 — Network Security in Kernel](Chapter_23_Network_Security.md)
