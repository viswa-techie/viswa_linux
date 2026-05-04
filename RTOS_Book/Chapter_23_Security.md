# Chapter 23: Security in RTOS

## Learning Goals
- Understand embedded security threats and attack surfaces
- Learn secure boot chain and firmware update mechanisms
- Master ARM TrustZone for RTOS security isolation
- Know crypto hardware accelerators and secure storage

---

## 1. Embedded Security Threat Model

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Attack Surface for Embedded RTOS Devices                │
  │                                                           │
  │  ┌─────────────────┐                                    │
  │  │ Physical Attacks │                                    │
  │  │ • JTAG/SWD debug │◄── Disable in production!         │
  │  │ • Bus probing     │                                   │
  │  │ • Fault injection │                                   │
  │  │ • Side-channel    │                                   │
  │  └─────────────────┘                                    │
  │                                                           │
  │  ┌─────────────────┐                                    │
  │  │ Network Attacks  │                                    │
  │  │ • Buffer overflow │◄── Stack canaries + MPU           │
  │  │ • Protocol fuzzing│                                   │
  │  │ • Man-in-middle   │◄── TLS + mutual auth             │
  │  │ • Replay attacks  │◄── Nonce/timestamp                │
  │  └─────────────────┘                                    │
  │                                                           │
  │  ┌─────────────────┐                                    │
  │  │ Firmware Attacks │                                    │
  │  │ • Code extraction│◄── Read-out protection (RDP)      │
  │  │ • Tampered update│◄── Signed firmware                │
  │  │ • Rollback attack│◄── Anti-rollback counter          │
  │  └─────────────────┘                                    │
  │                                                           │
  │  Security Principles for RTOS:                           │
  │  1. Least privilege (MPU regions per task)               │
  │  2. Defense in depth (multiple layers)                   │
  │  3. Secure by default (disable debug, encrypt keys)     │
  │  4. Minimal attack surface (disable unused peripherals) │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Secure Boot Chain

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Root of Trust → Bootloader → RTOS Kernel → Application │
  │                                                           │
  │  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
  │  │  ROM Boot │───►│ Stage 1  │───►│ Stage 2  │          │
  │  │ (immutable│    │ Bootloader│   │ RTOS     │          │
  │  │  in SoC)  │    │ (signed)  │   │ (signed) │          │
  │  └──────────┘    └──────────┘    └──────────┘          │
  │       │               │               │                  │
  │       ▼               ▼               ▼                  │
  │  OTP key hash    Verify sig      Verify sig             │
  │  (fused in HW)   with pub key    with pub key           │
  │                                                           │
  │  Chain of Trust: each stage verifies the next            │
  │  If ANY verification fails → halt (do not boot)         │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Simplified secure boot verification */
#include "mbedtls/sha256.h"
#include "mbedtls/pk.h"

typedef struct {
    uint32_t magic;          /* 0xSECB0071 */
    uint32_t version;
    uint32_t image_size;
    uint32_t rollback_counter;
    uint8_t  signature[256]; /* RSA-2048 or ECDSA-P256 */
} FirmwareHeader_t;

bool verify_firmware(const FirmwareHeader_t *hdr, const uint8_t *image) {
    uint8_t hash[32];
    mbedtls_pk_context pk;

    /* Step 1: compute SHA-256 of firmware image */
    mbedtls_sha256(image, hdr->image_size, hash, 0);

    /* Step 2: verify signature against public key */
    mbedtls_pk_init(&pk);
    mbedtls_pk_parse_public_key(&pk, pub_key_der, pub_key_len);

    int ret = mbedtls_pk_verify(&pk, MBEDTLS_MD_SHA256,
                                 hash, 32,
                                 hdr->signature, sizeof(hdr->signature));
    mbedtls_pk_free(&pk);

    if (ret != 0) return false;

    /* Step 3: anti-rollback check */
    uint32_t stored_counter = read_otp_rollback_counter();
    if (hdr->rollback_counter < stored_counter) return false;

    return true;
}
```

---

## 3. ARM TrustZone for RTOS

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  TrustZone-M (Cortex-M23, M33, M55):                   │
  │                                                           │
  │  ┌────────────────────┬─────────────────────┐           │
  │  │    Secure World     │  Non-Secure World   │           │
  │  │                     │                      │           │
  │  │  Crypto keys        │  Application RTOS   │           │
  │  │  Secure boot code   │  (FreeRTOS/Zephyr)  │           │
  │  │  Root of trust      │                      │           │
  │  │  Secure storage     │  User tasks          │           │
  │  │                     │  Network stack       │           │
  │  │  Secure APIs ◄──────┤  Non-secure calls   │           │
  │  │  (NSC region)       │  via SG instruction  │           │
  │  └────────────────────┴─────────────────────┘           │
  │                                                           │
  │  SAU (Security Attribution Unit):                        │
  │  - Configures memory as Secure / Non-Secure / NSC       │
  │  - NSC (Non-Secure Callable): gateway region            │
  │  - Transition: SG instruction in NSC → secure function  │
  │                                                           │
  │  Implementation:                                         │
  │  - Secure: TF-M (Trusted Firmware-M) or custom          │
  │  - Non-Secure: FreeRTOS runs in NS world                │
  │  - Secure functions exported via veneer table            │
  └──────────────────────────────────────────────────────────┘
```

```c
/* TrustZone: Non-Secure Callable function (secure side) */
/* Marked with __attribute__((cmse_nonsecure_entry)) */
#include <arm_cmse.h>

__attribute__((cmse_nonsecure_entry))
int secure_encrypt(uint8_t *data, size_t len, uint8_t *out) {
    /* Validate pointer comes from non-secure memory */
    if (cmse_check_address_range(data, len, CMSE_NONSECURE) == NULL) {
        return -1;  /* Invalid: pointer in secure memory */
    }

    /* Perform encryption with secure key (never exposed) */
    aes_encrypt(secure_key, data, len, out);
    return 0;
}

/* Non-Secure side (FreeRTOS task) calls secure function */
void vEncryptTask(void *pvParameters) {
    uint8_t plaintext[16] = "Hello RTOS!";
    uint8_t ciphertext[16];

    /* This call transitions to secure world via SG instruction */
    int ret = secure_encrypt(plaintext, 16, ciphertext);
    if (ret != 0) {
        /* Handle error */
    }
}
```

---

## 4. Secure Firmware Update (OTA)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  A/B Update Scheme:                                      │
  │                                                           │
  │  Flash Layout:                                           │
  │  ┌────────────────────────────────────────────┐         │
  │  │ Bootloader (protected, read-only)           │         │
  │  ├────────────────────────────────────────────┤         │
  │  │ Slot A: Active firmware (currently running) │         │
  │  ├────────────────────────────────────────────┤         │
  │  │ Slot B: Update firmware (new image)         │         │
  │  ├────────────────────────────────────────────┤         │
  │  │ Swap metadata (which slot is active)        │         │
  │  └────────────────────────────────────────────┘         │
  │                                                           │
  │  Update flow:                                            │
  │  1. Download signed image → write to Slot B             │
  │  2. Verify signature + hash                             │
  │  3. Set "test boot" flag in metadata                    │
  │  4. Reboot → bootloader loads Slot B                    │
  │  5. If app confirms OK → mark Slot B as permanent       │
  │  6. If boot fails → bootloader reverts to Slot A        │
  │                                                           │
  │  MCUboot (Zephyr/FreeRTOS):                             │
  │  - Open source secure bootloader                         │
  │  - Supports: swap, overwrite, revert                    │
  │  - Image signing: RSA-2048, ECDSA-P256                  │
  │  - Anti-rollback with security counter                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Crypto Hardware Accelerators

```c
/*
 * Many MCUs have hardware crypto engines:
 * STM32: CRYP (AES, DES), HASH (SHA), RNG
 * NXP LPC: CASPER (RSA/ECC), HASHCRYPT
 * Nordic nRF: CryptoCell CC310
 *
 * Benefits:
 * - 10-100x faster than software
 * - Constant-time (resistant to timing attacks)
 * - DPA resistant (side-channel protection)
 * - Free CPU for real-time tasks
 */

/* STM32 hardware AES example */
#include "stm32_hal_cryp.h"

CRYP_HandleTypeDef hcryp;

void hw_aes_encrypt(const uint8_t key[16],
                     const uint8_t *input, size_t len,
                     uint8_t *output) {
    hcryp.Instance = AES;
    hcryp.Init.DataType = CRYP_DATATYPE_8B;
    hcryp.Init.KeySize = CRYP_KEYSIZE_128B;
    hcryp.Init.Algorithm = CRYP_AES_CBC;
    memcpy(hcryp.Init.pKey, key, 16);

    HAL_CRYP_Init(&hcryp);
    HAL_CRYP_Encrypt(&hcryp, (uint32_t *)input,
                      len / 4, (uint32_t *)output, HAL_MAX_DELAY);
    HAL_CRYP_DeInit(&hcryp);
}

/*
 * Secure key storage:
 * - OTP (One-Time Programmable) fuses: burned in manufacturing
 * - Secure element (ATECC608, SE050): tamper-resistant IC
 * - TrustZone secure memory: accessible only from secure world
 * - NEVER store keys in plaintext in firmware binary
 */
```

---

## 6. Cross-RTOS Security Comparison

```
  ┌──────────┬─────────────┬──────────────┬──────────────┐
  │ Feature  │ FreeRTOS    │ Zephyr       │ QNX          │
  ├──────────┼─────────────┼──────────────┼──────────────┤
  │ MPU      │ Optional    │ Built-in     │ Process      │
  │ support  │ (port dep.) │ (userspace)  │ isolation    │
  ├──────────┼─────────────┼──────────────┼──────────────┤
  │ TrustZone│ NS world    │ TF-M + NS   │ N/A (uses    │
  │          │ support     │ integration  │ hypervisor)  │
  ├──────────┼─────────────┼──────────────┼──────────────┤
  │ Secure   │ OTA library │ MCUboot      │ QNX secure   │
  │ boot     │ (AWS IoT)   │ (native)     │ boot chain   │
  ├──────────┼─────────────┼──────────────┼──────────────┤
  │ Crypto   │ mbedTLS     │ mbedTLS/     │ OpenSSL      │
  │ library  │             │ tinycrypt    │ (full)       │
  ├──────────┼─────────────┼──────────────┼──────────────┤
  │ Cert     │ PSA L1      │ PSA L1-2     │ EAL 4+       │
  │          │ (FreeRTOS+) │ (in progress)│ (CC cert)    │
  └──────────┴─────────────┴──────────────┴──────────────┘
```

---

## Interview Questions

**Q1: How does secure boot work on an MCU running RTOS?**
**A:** Secure boot establishes a chain of trust from hardware to application: (1) ROM bootloader (immutable in SoC silicon) contains hash of Stage 1 public key, burned into OTP fuses during manufacturing. (2) ROM loads Stage 1 bootloader from flash, computes SHA-256 hash of image, verifies RSA/ECDSA signature against public key hash in OTP. If mismatch → halt. (3) Stage 1 (e.g., MCUboot) verifies Stage 2 (RTOS + app) signature. (4) Anti-rollback counter in OTP prevents downgrade attacks — new firmware must have counter >= stored value. (5) JTAG/SWD debug ports are disabled via option bytes (STM32 RDP Level 2 is permanent). This ensures only authorized, unmodified firmware can execute.

**Q2: How does ARM TrustZone-M isolate security-critical code from RTOS tasks?**
**A:** TrustZone-M (Cortex-M33) divides the processor into two worlds: (1) Secure world: runs crypto operations, key storage, secure boot — has exclusive access to secure memory/peripherals. (2) Non-Secure world: runs FreeRTOS/Zephyr with application tasks — cannot access secure memory. The SAU (Security Attribution Unit) configures memory regions as Secure, Non-Secure, or Non-Secure Callable (NSC). Transition from NS→S happens via SG (Secure Gateway) instruction in NSC region, calling veneer functions marked with `cmse_nonsecure_entry`. The secure side validates all pointers from NS world using `cmse_check_address_range()` to prevent confused deputy attacks. The NS RTOS never sees secure keys or code — it calls secure services through a well-defined API.

---

## Summary

- Security threats: physical (JTAG), network (overflow), firmware (tampering)
- Secure boot: chain of trust from ROM → bootloader → RTOS, signature verification at each stage
- TrustZone-M: hardware isolation of crypto keys/operations from RTOS application code
- Secure OTA: A/B slots, signed images, anti-rollback, MCUboot for Zephyr/FreeRTOS
- Hardware crypto: AES/SHA accelerators — faster, constant-time, frees CPU for RT tasks
- Key storage: OTP fuses, secure elements, TrustZone secure memory — never in plaintext

---

[Previous Chapter: Safety and Reliability ←](Chapter_22_Safety.md) | [Next Chapter: Debugging →](Chapter_24_Debugging.md)
