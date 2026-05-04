# Chapter 18: File System Security

## Learning Goals
- Understand Unix permission model and its extensions (ACLs, capabilities)
- Master Linux filesystem encryption (fscrypt, dm-crypt)
- Know SELinux, AppArmor FS label/policy enforcement
- Understand integrity measurement: IMA, EVM, dm-verity

---

## 18.1 Unix Permission Model

### Traditional Permission Bits

```
Permission bits layout (mode_t):
┌─────────┬──────────┬──────────┬──────────┐
│ Special  │  Owner   │  Group   │  Other   │
├─────────┼──────────┼──────────┼──────────┤
│ suid     │  r  w  x │  r  w  x │  r  w  x │
│ sgid     │          │          │          │
│ sticky   │          │          │          │
└─────────┴──────────┴──────────┴──────────┘
  4  2  1    4  2  1    4  2  1    4  2  1

Examples:
  0755 = rwxr-xr-x   (typical directory / executable)
  0644 = rw-r--r--   (typical file)
  4755 = rwsr-xr-x   (setuid executable, e.g., /bin/passwd)
  2755 = rwxr-sr-x   (setgid directory)
  1777 = rwxrwxrwt   (sticky bit, e.g., /tmp)
```

### Kernel Permission Check Path

```
User process calls open("/etc/shadow", O_RDONLY)
│
├─→ do_sys_open()
│   └─→ do_filp_open()
│       └─→ path_openat()
│           └─→ may_open()
│               └─→ inode_permission()
│                   │
│                   ├─→ (1) security_inode_permission()  ← LSM hook
│                   │       └─→ SELinux / AppArmor check
│                   │
│                   ├─→ (2) do_inode_permission()
│                   │       └─→ generic_permission()
│                   │           │
│                   │           ├─→ Check capabilities (CAP_DAC_OVERRIDE, etc.)
│                   │           ├─→ Check ACLs if present
│                   │           └─→ Check traditional rwx bits
│                   │
│                   └─→ Return -EACCES or 0

Source: fs/namei.c — inode_permission(), generic_permission()
```

### Kernel Data Structures

```c
/* inode permission fields */
struct inode {
    kuid_t          i_uid;     /* Owner user ID */
    kgid_t          i_gid;     /* Owner group ID */
    umode_t         i_mode;    /* Permission bits + file type */
    /* ... */
    struct posix_acl *i_acl;   /* Extended ACL */
    struct posix_acl *i_default_acl; /* Default ACL (directories) */
    void            *i_security; /* LSM security label */
};

/* Permission check in generic_permission() */
static int acl_permission_check(struct inode *inode, int mask)
{
    unsigned int mode = inode->i_mode;

    /* Owner check */
    if (uid_eq(current_fsuid(), inode->i_uid)) {
        mode >>= 6;  /* Use owner bits */
    }
    /* Group check */
    else if (in_group_p(inode->i_gid)) {
        mode >>= 3;  /* Use group bits */
    }
    /* Other: use bottom 3 bits as-is */

    if ((mask & ~mode & (MAY_READ|MAY_WRITE|MAY_EXEC)) == 0)
        return 0;  /* Permitted */
    return -EACCES;
}
```

---

## 18.2 POSIX Access Control Lists (ACLs)

```
ACLs extend the owner/group/other model:

Traditional:    rw-r-----  (owner=rw, group=r, other=none)
With ACL:       Can add per-user or per-group entries

ACL entries:
  user::rwx        # File owner permissions
  user:john:rw-    # Specific user "john"
  group::r-x       # File group permissions
  group:devs:rw-   # Specific group "devs"
  mask::rwx        # Maximum effective permissions
  other::r--       # Everyone else
```

### ACL Commands

```bash
# Set ACL
$ setfacl -m u:john:rw /var/shared/file.txt

# Set default ACL on directory (inherited by new files)
$ setfacl -d -m g:developers:rwx /var/shared/

# View ACL
$ getfacl /var/shared/file.txt
# file: var/shared/file.txt
# owner: root
# group: root
user::rw-
user:john:rw-
group::r--
group:developers:rw-
mask::rw-
other::r--

# Remove all ACLs
$ setfacl -b /var/shared/file.txt
```

### Kernel ACL Implementation

```c
/* POSIX ACL structure (include/linux/posix_acl.h) */
struct posix_acl_entry {
    short               e_tag;   /* ACL_USER_OBJ, ACL_USER, ACL_GROUP_OBJ, ... */
    unsigned short      e_perm;  /* ACL_READ, ACL_WRITE, ACL_EXECUTE */
    union {
        kuid_t          e_uid;
        kgid_t          e_gid;
    };
};

struct posix_acl {
    refcount_t          a_refcount;
    struct rcu_head     a_rcu;
    unsigned int        a_count;   /* Number of entries */
    struct posix_acl_entry a_entries[];  /* Flexible array */
};

/* On-disk: stored as extended attribute */
/* system.posix_acl_access  — file ACL */
/* system.posix_acl_default — directory default ACL */

/* ext4 stores ACLs in xattr blocks */
/* XFS stores ACLs in its attribute fork */
```

---

## 18.3 Linux Capabilities

```
Capabilities break root (UID 0) into granular privileges:

Traditional:
  root can do ANYTHING → one compromised root process = total control

Capabilities model:
  Each process has a capability set
  Each capability grants a specific privilege

File-system relevant capabilities:
┌─────────────────────┬──────────────────────────────────────────┐
│ Capability           │ Grants                                   │
├─────────────────────┼──────────────────────────────────────────┤
│ CAP_DAC_OVERRIDE    │ Bypass file read/write/execute perms     │
│ CAP_DAC_READ_SEARCH │ Bypass read permission & directory search│
│ CAP_FOWNER          │ Bypass ownership checks (chmod, etc.)    │
│ CAP_CHOWN           │ Change file ownership                    │
│ CAP_FSETID          │ Set SUID/SGID without ownership          │
│ CAP_MKNOD           │ Create special files (mknod)             │
│ CAP_SYS_ADMIN       │ Mount/umount, many sysadmin operations   │
│ CAP_LINUX_IMMUTABLE │ Set/clear immutable flag                 │
│ CAP_SYS_CHROOT      │ Use chroot()                             │
└─────────────────────┴──────────────────────────────────────────┘
```

```bash
# Set capabilities on a file
$ setcap cap_net_bind_service+ep /usr/bin/myapp
# Now myapp can bind to port <1024 without root

# View capabilities
$ getcap /usr/bin/myapp
/usr/bin/myapp = cap_net_bind_service+ep

# Check process capabilities
$ cat /proc/self/status | grep ^Cap
CapInh: 0000000000000000
CapPrm: 000001ffffffffff
CapEff: 000001ffffffffff
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000
```

---

## 18.4 File System Encryption (fscrypt)

### Architecture

```
fscrypt (native filesystem encryption):

                    User Space
  ┌──────────────────────────────────┐
  │  Application → open("/data/foo") │
  └───────────────┬──────────────────┘
                  │
  ═══════════════ │ ═══════ VFS ═════════
                  │
  ┌───────────────▼──────────────────┐
  │         fscrypt layer            │
  │  Handles key management          │
  │  Encrypts filenames              │
  │  Sets up per-file crypto         │
  └───────────────┬──────────────────┘
                  │
  ┌───────────────▼──────────────────┐
  │      ext4 / f2fs / UBIFS         │
  │  bio submissions include         │
  │  crypto transform info           │
  └───────────────┬──────────────────┘
                  │
  ┌───────────────▼──────────────────┐
  │      Block layer / blk-crypto    │
  │  Inline encryption if HW capable │
  │  Software fallback otherwise     │
  └───────────────┬──────────────────┘
                  │
  ┌───────────────▼──────────────────┐
  │      Disk (ciphertext stored)    │
  └──────────────────────────────────┘

  Key hierarchy:
    Master key (from user/keyring)
        │
        ├─→ Per-file content encryption key
        │     └─→ AES-256-XTS (file contents)
        │
        └─→ Per-directory filename encryption key
              └─→ AES-256-CTS (filenames)
```

### fscrypt Commands

```bash
# Generate a key
$ fscryptctl add_key /mnt
Enter passphrase: ****
Added key with identifier 0x1234abcd

# Set encryption policy on directory
$ fscryptctl set_policy 0x1234abcd /mnt/encrypted_dir/

# Kernel API (simplified):
# 1. ioctl(fd, FS_IOC_SET_ENCRYPTION_POLICY, &policy)
# 2. ioctl(fd, FS_IOC_ADD_ENCRYPTION_KEY, &arg)
# 3. ioctl(fd, FS_IOC_REMOVE_ENCRYPTION_KEY, &arg)
```

### fscrypt Kernel Internals

```c
/* Per-file encryption context (stored in xattr) */
struct fscrypt_context_v2 {
    u8 version;            /* 2 */
    u8 contents_encryption_mode;  /* FSCRYPT_MODE_AES_256_XTS */
    u8 filenames_encryption_mode; /* FSCRYPT_MODE_AES_256_CTS */
    u8 flags;
    u8 __reserved[4];
    u8 master_key_identifier[FSCRYPT_KEY_IDENTIFIER_SIZE];
    u8 nonce[FSCRYPT_FILE_NONCE_SIZE]; /* Per-file random nonce */
};

/* On read path: fscrypt decrypts page after bio completion */
/* On write path: fscrypt encrypts page before bio submission */

/* Source: fs/crypto/ */
```

---

## 18.5 dm-crypt (Full Disk Encryption)

```
dm-crypt operates at the block layer:

    File System (ext4, XFS, ...)
         │
    ┌────▼────────────────────────┐
    │   dm-crypt (device mapper)  │
    │   Encrypts/decrypts every   │
    │   block I/O transparently   │
    │   Uses kernel crypto API    │
    └────┬────────────────────────┘
         │
    Physical Block Device (/dev/sda)

Setup with LUKS:
  $ cryptsetup luksFormat /dev/sda2
  $ cryptsetup luksOpen /dev/sda2 encrypted_vol
  $ mkfs.ext4 /dev/mapper/encrypted_vol
  $ mount /dev/mapper/encrypted_vol /mnt
```

### fscrypt vs dm-crypt

```
┌──────────────┬──────────────────────────┬─────────────────────────┐
│ Feature       │ fscrypt                  │ dm-crypt (LUKS)         │
├──────────────┼──────────────────────────┼─────────────────────────┤
│ Encryption   │ Per-file / per-directory  │ Full partition          │
│ Granularity  │ Fine (different keys/dir) │ All-or-nothing          │
│ Filename     │ Encrypted                │ N/A (encrypts blocks)   │
│ Metadata     │ Partial (timestamps leak) │ Full (everything)       │
│ Multi-user   │ Each user has own key    │ One key per volume      │
│ Performance  │ Inline HW crypto possible│ May bypass HW crypto    │
│ Android use  │ FBE (File-Based Encrypt) │ FDE (Full-Disk Encrypt, │
│              │ Default since Android 10 │ deprecated)             │
│ Key mgmt     │ Linux keyring            │ LUKS header             │
└──────────────┴──────────────────────────┴─────────────────────────┘
```

---

## 18.6 SELinux and File Systems

### Labels

```
SELinux assigns a label (security context) to every file:

$ ls -Z /etc/passwd
-rw-r--r--. root root system_u:object_r:passwd_file_t:s0 /etc/passwd

Label format: user:role:type:level
  user    = system_u (system), unconfined_u (user)
  role    = object_r (files always object_r)
  type    = passwd_file_t (determines access rules)
  level   = s0 (MLS/MCS level)
```

### How Labels Are Stored

```
ext4 / XFS / Btrfs:
  Labels stored as extended attributes:
    security.selinux = "system_u:object_r:passwd_file_t:s0\0"

  ┌──────────────┐
  │   inode       │
  │   i_security──┼──→ struct inode_security_struct
  │              │     ├── sid (security ID)
  │   xattrs:   │     ├── sclass
  │   security. │     └── initialized
  │   selinux   │
  └──────────────┘

New file label assignment:
  1. Transition rules in policy
  2. Parent directory type_transition rule
  3. Default: inherit from parent directory context
```

### SELinux FS Enforcement

```c
/* LSM hook for file access (in kernel) */
static int selinux_inode_permission(struct inode *inode, int mask)
{
    /* Get source SID (process context) */
    u32 sid = current_sid();

    /* Get target SID (file context) from i_security */
    struct inode_security_struct *isec = inode->i_security;
    u32 isid = isec->sid;

    /* Query AVC (Access Vector Cache) for permission */
    return avc_has_perm(sid, isid, isec->sclass, mask, ...);
    /*
     * Returns 0 (allowed) or -EACCES (denied)
     *
     * Example: process httpd_t accessing file httpd_sys_content_t
     * Policy rule: allow httpd_t httpd_sys_content_t : file { read open };
     */
}
```

---

## 18.7 AppArmor

```
AppArmor uses path-based access control (not labels):

Profile example (/etc/apparmor.d/usr.bin.myapp):
  /usr/bin/myapp {
      /etc/myapp.conf    r,          # read config
      /var/log/myapp.log w,          # write log
      /tmp/myapp.*       rw,         # read/write temp files
      /usr/lib/**        mr,         # memory-map and read libs
      /dev/null          rw,         # /dev/null access
      deny /etc/shadow   rwx,        # explicit deny
  }

Path-based vs. Label-based:
  AppArmor: checks path (/etc/shadow)
  SELinux:  checks label (shadow_t)

  ┌──────────────────────────────────────────────────────┐
  │ AppArmor advantage: easier to write, no relabeling   │
  │ SELinux advantage:  labels survive rename/hard links  │
  └──────────────────────────────────────────────────────┘
```

---

## 18.8 Integrity: IMA and EVM

### IMA (Integrity Measurement Architecture)

```
IMA measures (hashes) files before execution/reading:

Boot-time:
  1. Kernel measures itself (boot measurement)
  2. IMA policy loaded
  3. Every file opened matching policy → hash computed
  4. Hash recorded in kernel measurement list
  5. List extended into TPM PCR (hardware trust anchor)

                    ┌─────────────┐
                    │    TPM      │
                    │  PCR[10]    │←── extend(file hash)
                    └──────┬──────┘
                           │
        ┌──────────────────▼───────────────────┐
        │       IMA Measurement List            │
        │  /proc/integrity/ima/ascii_runtime_*  │
        │  [ hash | filename ]                  │
        └──────────────────────────────────────┘

IMA modes:
  measurement  — record hashes in measurement list (audit)
  appraisal    — verify hash matches stored xattr, DENY if mismatch
  audit        — log all measurements to audit log
```

### EVM (Extended Verification Module)

```
EVM protects security-critical extended attributes:

Protected xattrs:
  security.ima       (file hash)
  security.selinux   (SELinux label)
  security.capability (file capabilities)

EVM signs/verifies these xattrs → prevents offline tampering

Flow:
  1. EVM computes HMAC over protected xattrs
  2. Stores HMAC in security.evm xattr
  3. On file access: EVM recomputes HMAC, compares
  4. If mismatch → xattr has been tampered → DENY access
```

---

## 18.9 dm-verity (Read-Only Integrity)

```
dm-verity provides verified boot for read-only partitions:

┌─────────────────────────────────────────────────────────┐
│                dm-verity Architecture                     │
│                                                          │
│  Hash tree:                                              │
│       ┌──────────┐                                       │
│       │ Root hash │ (signed, stored in boot image)       │
│       └────┬─────┘                                       │
│            │                                             │
│      ┌─────┴────────┐                                    │
│      ▼              ▼                                    │
│  ┌───────┐    ┌───────┐                                  │
│  │ Hash  │    │ Hash  │   (intermediate hash blocks)     │
│  └──┬──┬─┘    └──┬──┬─┘                                  │
│     ▼  ▼        ▼  ▼                                     │
│  ┌──┐┌──┐    ┌──┐┌──┐    (leaf hashes)                   │
│  │h1││h2│    │h3││h4│                                    │
│  └──┘└──┘    └──┘└──┘                                    │
│   │   │       │   │                                      │
│   ▼   ▼       ▼   ▼                                      │
│  ┌──┐┌──┐   ┌──┐┌──┐    (data blocks on disk)           │
│  │D1││D2│   │D3││D4│                                    │
│  └──┘└──┘   └──┘└──┘                                    │
│                                                          │
│  On read: verify hash(block) == tree hash                │
│  Tampered block → I/O error                              │
└─────────────────────────────────────────────────────────┘

Android usage:
  system, vendor, product partitions → dm-verity protected
  Ensures read-only partitions haven't been tampered with
  Root hash verified at boot via AVB (Android Verified Boot)
```

---

## 18.10 Immutable and Append-Only Files

```bash
# Make a file immutable (cannot be modified, deleted, renamed, or linked)
$ chattr +i /etc/critical_config
$ lsattr /etc/critical_config
----i--------e-- /etc/critical_config

# Append-only (can only add data, not modify existing)
$ chattr +a /var/log/audit.log

# Requires CAP_LINUX_IMMUTABLE to set/clear

# Kernel implementation: uses FS_IMMUTABLE_FL in inode->i_flags
# Checked in inode_change_ok() and various write paths
```

---

## 18.11 Security Comparison Across File Systems

```
┌──────────────┬──────────┬──────────┬──────────┬───────────┐
│ Feature       │ ext4      │ XFS       │ Btrfs     │ SquashFS  │
├──────────────┼──────────┼──────────┼──────────┼───────────┤
│ ACLs          │ ✓        │ ✓        │ ✓        │ Limited   │
│ SELinux xattr │ ✓        │ ✓        │ ✓        │ ✓         │
│ fscrypt       │ ✓        │ ✗        │ ✗        │ N/A       │
│ dm-verity     │ ✓        │ ✓        │ ✗        │ ✓         │
│ Capabilities  │ ✓        │ ✓        │ ✓        │ ✓         │
│ IMA/EVM       │ ✓        │ ✓        │ ✓        │ ✓*        │
│ Checksums     │ Metadata │ Metadata │ Data+Meta│ N/A       │
│ Immutable flag│ ✓        │ ✓        │ ✓        │ Always RO │
│ Verity (fs)   │ ✓ (5.4+) │ ✗        │ ✗        │ ✓         │
└──────────────┴──────────┴──────────┴──────────┴───────────┘

* SquashFS: IMA can measure, but SquashFS is read-only
```

---

## 18.12 Automotive Security Considerations

```
Automotive file system security requirements:

1. Verified Boot Chain:
   Bootloader (signed) → Kernel (signed) → dm-verity (system)
   Any break → boot fails

2. Partition layout:
   /system    → SquashFS + dm-verity (read-only, verified)
   /vendor    → ext4 + dm-verity (read-only, verified)
   /data      → ext4 + fscrypt (encrypted per-user keys)
   /persist   → ext4 + IMA (tamper detection)

3. SELinux enforcing:
   All processes confined
   No unconfined domains in production
   neverallow rules prevent policy bypass

4. Key storage:
   Hardware-backed keystore (TEE / HSM)
   Keys never in plaintext on filesystem

5. OTA updates:
   A/B partitions → can verify new image before switch
   Rollback protection via hardware counters
```

---

## Kernel Source References

```
Permission checks:
  fs/namei.c               ← inode_permission(), generic_permission()
  fs/posix_acl.c           ← POSIX ACL implementation
  include/linux/posix_acl.h

fscrypt:
  fs/crypto/               ← fscrypt core implementation
  include/linux/fscrypt.h

SELinux:
  security/selinux/hooks.c ← selinux_inode_permission()
  security/selinux/avc.c   ← Access Vector Cache

IMA/EVM:
  security/integrity/ima/  ← IMA implementation
  security/integrity/evm/  ← EVM implementation

dm-verity:
  drivers/md/dm-verity*    ← dm-verity target
```

---

## Interview Questions

1. **Explain the complete permission check path when open() is called.**
2. **What's the difference between SELinux labels and AppArmor paths?**
3. **How does fscrypt differ from dm-crypt? When to use each?**
4. **What is the key hierarchy in fscrypt? How are per-file keys derived?**
5. **What file system security features does Android use?**
6. **How does dm-verity protect read-only partitions?**
7. **What are IMA and EVM? How do they complement each other?**
8. **How are POSIX ACLs stored on disk in ext4?**
9. **What capabilities relate to file system operations?**
10. **Design a secure partition layout for an automotive head unit.**

---

## Summary

- Unix permissions: classic owner/group/other + SUID/SGID/sticky
- ACLs: fine-grained per-user/per-group permissions via extended attributes
- Capabilities: break root into granular privileges (CAP_DAC_OVERRIDE, etc.)
- fscrypt: per-file/per-directory encryption at FS level; Android FBE default
- dm-crypt: full disk/partition encryption at block layer (LUKS)
- SELinux: mandatory access control via labels on files (security.selinux xattr)
- AppArmor: path-based mandatory access control (simpler, less robust)
- IMA/EVM: integrity measurement and verification of files and xattrs
- dm-verity: Merkle tree verification for read-only partitions (Android Verified Boot)
- Automotive: full trust chain from bootloader → kernel → dm-verity → fscrypt → SELinux

---

*Next: [Chapter 19 — Debugging Tools](Chapter_19_Debugging_Tools.md)*
