# Chapter 21: Cryptographic Services

## Learning Goals
- Understand the Linux Kernel Crypto API architecture
- Know symmetric, asymmetric, and hash algorithm support
- Understand hardware crypto acceleration
- Know /dev/random entropy and CSPRNG

---

## 21.1 Kernel Crypto API Architecture

```
The kernel provides a pluggable cryptographic framework.

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │  User Space                                           │
  │    AF_ALG socket interface (user-space crypto)        │
  │    /dev/random, /dev/urandom                          │
  │    OpenSSL / libgcrypt (user-space libraries)         │
  └───────────────────────┬──────────────────────────────┘
                           │
  ┌───────────────────────▼──────────────────────────────┐
  │  Kernel Crypto API (crypto/)                          │
  │                                                       │
  │  Algorithm types:                                     │
  │  ┌────────────┐ ┌────────────┐ ┌────────────┐        │
  │  │ Cipher     │ │ Hash/HMAC  │ │ AEAD       │        │
  │  │ (AES, DES) │ │ (SHA, MD5) │ │ (GCM, CCM) │        │
  │  └────────────┘ └────────────┘ └────────────┘        │
  │  ┌────────────┐ ┌────────────┐ ┌────────────┐        │
  │  │ Compress   │ │ RNG        │ │ KPP/Asym   │        │
  │  │ (LZO, LZ4) │ │ (DRBG)    │ │ (RSA, EC)  │        │
  │  └────────────┘ └────────────┘ └────────────┘        │
  │                                                       │
  │  Template system:                                     │
  │    cbc(aes)  — AES in CBC mode                        │
  │    gcm(aes)  — AES-GCM authenticated encryption       │
  │    hmac(sha256)  — HMAC with SHA-256                  │
  │    xts(aes)  — AES-XTS for disk encryption            │
  │                                                       │
  │  Algorithm priority system:                           │
  │    Hardware implementations have higher priority       │
  │    Fallback to software if hardware unavailable       │
  └───────────────────────┬──────────────────────────────┘
                           │
  ┌───────────────────────▼──────────────────────────────┐
  │  Hardware Acceleration                                │
  │    AES-NI (x86)      — AES in hardware                │
  │    ARM CE            — ARM Crypto Extensions           │
  │    CCP (AMD)         — Crypto Co-Processor             │
  │    QAT (Intel)       — QuickAssist Technology          │
  │    CAAM (NXP)        — Crypto Accel and Assurance      │
  └──────────────────────────────────────────────────────┘
```

---

## 21.2 Using the Crypto API in Kernel

```c
#include <crypto/hash.h>
#include <crypto/skcipher.h>

/*
 * SHA-256 hash example
 */
int compute_sha256(const u8 *data, unsigned int len, u8 *digest)
{
    struct crypto_shash *tfm;
    struct shash_desc *desc;
    int ret;

    tfm = crypto_alloc_shash("sha256", 0, 0);
    if (IS_ERR(tfm))
        return PTR_ERR(tfm);

    desc = kmalloc(sizeof(*desc) + crypto_shash_descsize(tfm),
                   GFP_KERNEL);
    if (!desc) {
        crypto_free_shash(tfm);
        return -ENOMEM;
    }

    desc->tfm = tfm;
    ret = crypto_shash_digest(desc, data, len, digest);

    kfree(desc);
    crypto_free_shash(tfm);
    return ret;
}

/*
 * AES encryption example (skcipher — symmetric key cipher)
 */
int encrypt_aes_cbc(const u8 *key, unsigned int keylen,
                    const u8 *iv, const u8 *plaintext,
                    unsigned int len, u8 *ciphertext)
{
    struct crypto_skcipher *tfm;
    struct skcipher_request *req;
    struct scatterlist sg_in, sg_out;
    DECLARE_CRYPTO_WAIT(wait);
    int ret;

    tfm = crypto_alloc_skcipher("cbc(aes)", 0, 0);
    if (IS_ERR(tfm))
        return PTR_ERR(tfm);

    ret = crypto_skcipher_setkey(tfm, key, keylen);
    if (ret)
        goto out;

    req = skcipher_request_alloc(tfm, GFP_KERNEL);
    sg_init_one(&sg_in, plaintext, len);
    sg_init_one(&sg_out, ciphertext, len);

    skcipher_request_set_crypt(req, &sg_in, &sg_out, len, (u8 *)iv);
    skcipher_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
                                  crypto_req_done, &wait);

    ret = crypto_wait_req(crypto_skcipher_encrypt(req), &wait);

    skcipher_request_free(req);
out:
    crypto_free_skcipher(tfm);
    return ret;
}
```

---

## 21.3 Supported Algorithms

```
Algorithm listing: /proc/crypto

  ┌─────────────────┬──────────────────────────────────┐
  │ Category        │ Algorithms                        │
  ├─────────────────┼──────────────────────────────────┤
  │ Block ciphers   │ AES, DES, 3DES, Blowfish,        │
  │                 │ Twofish, Camellia, SM4            │
  │                 │                                   │
  │ Modes           │ ECB, CBC, CTR, XTS, CFB, OFB      │
  │                 │                                   │
  │ AEAD            │ GCM, CCM, ChaCha20-Poly1305,      │
  │                 │ AES-GCM-SIV                        │
  │                 │                                   │
  │ Hash            │ SHA-1, SHA-256, SHA-384, SHA-512,  │
  │                 │ SHA3-256, SHA3-512, MD5, SM3,      │
  │                 │ BLAKE2b, BLAKE2s                   │
  │                 │                                   │
  │ MAC             │ HMAC, CMAC, VMAC, Poly1305         │
  │                 │                                   │
  │ Asymmetric      │ RSA, ECDSA, ECDH, SM2, Ed25519     │
  │                 │                                   │
  │ KDF             │ HKDF                               │
  │                 │                                   │
  │ RNG             │ DRBG (CTR, Hash, HMAC based)       │
  │                 │                                   │
  │ Compression     │ LZO, LZ4, ZSTD, Deflate            │
  └─────────────────┴──────────────────────────────────┘

Kernel crypto consumers:
  - dm-crypt (disk encryption)
  - IPsec/IKE (VPN)
  - TLS (kernel TLS offload)
  - eCryptfs/fscrypt (filesystem encryption)
  - Module signing verification
  - Integrity (IMA/EVM)
  - WireGuard VPN (ChaCha20-Poly1305)
```

---

## 21.4 Random Number Generation

```
Kernel entropy and CSPRNG:

  /dev/random  — blocks until sufficient entropy (pre-5.6)
  /dev/urandom — never blocks, always available
  getrandom()  — syscall, can block until seeded

  Linux 5.6+: /dev/random behaves like /dev/urandom
  (no longer blocks after initial seeding)

Entropy architecture:
  ┌─────────────────────────────────────────────┐
  │  Entropy Sources                             │
  │                                              │
  │  Hardware: RDRAND/RDSEED (CPU instruction)   │
  │  Interrupts: timing of IRQs                  │
  │  Disk: I/O timing jitter                     │
  │  Input: keyboard/mouse events                │
  │  Network: packet arrival times               │
  │  Boot: UEFI/bootloader seed                  │
  └──────────────────┬──────────────────────────┘
                      │
  ┌──────────────────▼──────────────────────────┐
  │  Kernel CSPRNG (drivers/char/random.c)       │
  │                                              │
  │  ChaCha20-based CSPRNG                       │
  │  Input pool → crng_state                     │
  │  Periodically re-seeded from entropy sources │
  │                                              │
  │  Outputs:                                    │
  │    get_random_bytes() — kernel internal       │
  │    /dev/urandom       — user-space            │
  │    getrandom()        — user-space syscall    │
  └──────────────────────────────────────────────┘

  Security considerations:
    - Early boot: insufficient entropy → weak keys possible
    - getrandom(GRND_RANDOM) blocks until CSPRNG seeded
    - Embedded systems: may need hardware RNG (hwrng)
    - VMs: virtio-rng for entropy from host
```

---

## 21.5 Disk Encryption (dm-crypt/LUKS)

```
dm-crypt: Kernel block device encryption layer.

  LUKS (Linux Unified Key Setup):
    Standard format for disk encryption
    Multiple key slots
    Key derivation with PBKDF2 or Argon2

  ┌────────────┐    ┌──────────┐    ┌────────────┐
  │ Filesystem │───►│ dm-crypt │───►│ Physical   │
  │ (ext4)     │    │ AES-XTS  │    │ Disk       │
  │ plaintext  │    │ encrypt/ │    │ ciphertext │
  │            │    │ decrypt  │    │            │
  └────────────┘    └──────────┘    └────────────┘

  Typical setup:
    cryptsetup luksFormat /dev/sda2
    cryptsetup open /dev/sda2 secure_vol
    mkfs.ext4 /dev/mapper/secure_vol
    mount /dev/mapper/secure_vol /mnt/secure

  File-based encryption (fscrypt):
    Per-file/per-directory encryption
    Used by Android (ext4/f2fs) and Chrome OS
    Different keys for different directories/users
    Filename encryption for metadata protection
```

---

## 21.6 AF_ALG User-Space Interface

```
AF_ALG: Socket-based interface for user-space crypto operations.
Uses kernel crypto algorithms from user space.

  /* User-space SHA-256 via AF_ALG */
  int sock = socket(AF_ALG, SOCK_SEQPACKET, 0);

  struct sockaddr_alg sa = {
      .salg_family = AF_ALG,
      .salg_type   = "hash",
      .salg_name   = "sha256"
  };

  bind(sock, (struct sockaddr *)&sa, sizeof(sa));
  int fd = accept(sock, NULL, NULL);

  write(fd, data, data_len);
  read(fd, digest, 32);

  close(fd);
  close(sock);

  Benefits:
    - Uses hardware-accelerated algorithms automatically
    - Zero-copy via splice/vmsplice
    - Same algorithms as kernel (dm-crypt, IPsec)
    - No need for user-space crypto libraries
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| crypto/api.c | Crypto API registration and lookup |
| crypto/skcipher.c | Symmetric cipher framework |
| crypto/hash.c, crypto/shash.c | Hash/digest framework |
| crypto/aead.c | Authenticated encryption framework |
| crypto/af_alg.c | AF_ALG user-space socket interface |
| drivers/char/random.c | /dev/random CSPRNG |
| drivers/md/dm-crypt.c | dm-crypt disk encryption |
| arch/x86/crypto/aesni-intel_glue.c | AES-NI acceleration |

---

## Interview Questions

**Q1: How does the Linux Kernel Crypto API work?**
A: The API is a pluggable framework with algorithm types (cipher, hash, AEAD, RNG) and a template system for constructing composite algorithms like `cbc(aes)` or `hmac(sha256)`. Each algorithm has a priority — hardware implementations (AES-NI, ARM CE) register with higher priority than software fallbacks. When kernel code requests `crypto_alloc_skcipher("cbc(aes)", 0, 0)`, the API selects the highest-priority implementation available. The API uses scatterlists for zero-copy I/O and supports asynchronous operations. Main consumers include dm-crypt (disk encryption), IPsec (VPN), fscrypt (file encryption), WireGuard, and module signature verification.

**Q2: Explain the difference between /dev/random, /dev/urandom, and getrandom().**
A: Historically, /dev/random blocked until sufficient entropy was available, while /dev/urandom never blocked. Since Linux 5.6, both behave identically after initial seeding — neither blocks once the CSPRNG (ChaCha20-based) is seeded. The `getrandom()` syscall is the modern preferred interface: `getrandom(buf, len, 0)` blocks until the CSPRNG is seeded (safe for key generation), while `getrandom(buf, len, GRND_NONBLOCK)` returns -EAGAIN if not yet seeded. For most applications, getrandom() or /dev/urandom is correct. The kernel CSPRNG is re-seeded periodically from hardware RNG, interrupt timing, and other entropy sources.

**Q3: How does dm-crypt provide disk encryption?**
A: dm-crypt is a device-mapper target that encrypts/decrypts block I/O transparently. It sits between the filesystem and the physical device: reads decrypt blocks from disk, writes encrypt blocks before writing. Typical cipher: AES-XTS (256-bit key). LUKS wraps dm-crypt with key management — the master key is encrypted with a user passphrase via PBKDF2/Argon2, supporting multiple key slots. The kernel Crypto API handles the actual encryption, automatically using hardware acceleration (AES-NI) if available. For file-level encryption, fscrypt encrypts individual files with per-file keys, used by Android for user data isolation.

---

## Summary

- Kernel Crypto API: pluggable framework with priority-based algorithm selection
- Template system: cbc(aes), hmac(sha256), gcm(aes), xts(aes)
- Hardware acceleration: AES-NI, ARM CE, QAT — auto-selected by priority
- Scatterlist I/O for zero-copy; async operation support
- /dev/random, /dev/urandom, getrandom() — ChaCha20-based CSPRNG
- dm-crypt/LUKS: transparent block device encryption
- fscrypt: per-file encryption (Android, Chrome OS)
- AF_ALG: socket interface for user-space crypto operations

---

Next: [Chapter 22 — Key Management](Chapter_22_Key_Management.md)
