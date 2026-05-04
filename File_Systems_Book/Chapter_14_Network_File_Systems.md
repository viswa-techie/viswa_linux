# Chapter 14: Network File Systems — NFS, SMB/CIFS

## Learning Goals
- Understand how network file systems integrate with VFS
- Know NFS architecture, protocol versions, and kernel client
- Know SMB/CIFS basics and Linux ksmbd/cifs client
- Understand cache coherency challenges with network FS

---

## 14.1 Network FS Architecture

```
Network FS access remote storage over the network:

  Client Machine                  Server Machine
  ┌─────────────────────┐        ┌─────────────────────┐
  │ Application         │        │ NFS/SMB Server      │
  │   read(fd, ...)     │        │   (nfsd / smbd)     │
  │         │           │        │         │           │
  │    VFS Layer        │        │    VFS Layer        │
  │         │           │        │         │           │
  │   NFS/CIFS Client   │        │   Local FS (ext4)   │
  │   (kernel module)   │        │         │           │
  │         │           │  RPC/  │   Block Layer       │
  │    Network Stack    │◄─SMB──►│   Network Stack     │
  │    (TCP/IP)         │ over   │   (TCP/IP)          │
  │         │           │ network│         │           │
  │       NIC           │        │       NIC           │
  └─────────────────────┘        └─────────────────────┘

Key: Client kernel has a VFS FS module (nfs.ko / cifs.ko)
     that translates VFS operations into network requests.
```

---

## 14.2 NFS — Network File System

### NFS Versions

```
Version │ Year  │ Protocol  │ Key Features
────────┼───────┼───────────┼─────────────────────────────
NFSv2   │ 1989  │ UDP       │ Basic, 32-bit, 2GB file max
NFSv3   │ 1995  │ UDP/TCP   │ 64-bit offsets, async writes, READDIRPLUS
NFSv4   │ 2003  │ TCP only  │ Stateful, compound ops, ACLs,
        │       │           │ delegation, single port (2049)
NFSv4.1 │ 2010  │ TCP       │ pNFS (parallel NFS), sessions,
        │       │           │ directory delegations
NFSv4.2 │ 2016  │ TCP       │ Server-side copy, fallocate,
        │       │           │ labeled NFS (SELinux), sparse files
```

### NFS Architecture

```
NFS Client (Linux kernel):
  ┌──────────────────────────────────────────────┐
  │  VFS Interface                               │
  │    nfs_file_operations                       │
  │    nfs_dir_inode_operations                  │
  │    nfs_symlink_inode_operations              │
  │                                              │
  │  NFS Client Core                             │
  │    ├── RPC client (SUNRPC)                   │
  │    ├── Inode/dentry management               │
  │    ├── Page cache integration                │
  │    ├── Delegation handling                   │
  │    └── Lock management (NLM / NFSv4 locks)  │
  │                                              │
  │  SUNRPC Layer                                │
  │    ├── XDR encoding/decoding                 │
  │    ├── Connection management                 │
  │    ├── Authentication (RPCSEC_GSS/Kerberos)  │
  │    └── Transport (TCP/RDMA)                  │
  └──────────────────────────────────────────────┘

NFS Server (Linux kernel, nfsd):
  ┌──────────────────────────────────────────────┐
  │  NFSD (kernel threads)                       │
  │    ├── RPC dispatcher                        │
  │    ├── NFSv3/v4 protocol handlers            │
  │    ├── Export table (what's shared)           │
  │    ├── File handle → path resolution         │
  │    └── VFS calls on local FS                 │
  └──────────────────────────────────────────────┘
```

### NFS Client VFS Operations

```c
/* fs/nfs/file.c — NFS file operations */
const struct file_operations nfs_file_operations = {
    .llseek       = nfs_file_llseek,
    .read_iter    = nfs_file_read,
    .write_iter   = nfs_file_write,
    .mmap         = nfs_file_mmap,
    .open         = nfs_file_open,
    .flush        = nfs_file_flush,    /* NFS implements flush! */
    .release      = nfs_file_release,
    .fsync        = nfs_file_fsync,
    .lock         = nfs_lock,
    .flock        = nfs_flock,
    .splice_read  = nfs_file_splice_read,
    .splice_write = iter_file_splice_write,
    .check_flags  = nfs_check_flags,
};

/* NFS inode operations */
const struct inode_operations nfs_file_inode_operations = {
    .permission   = nfs_permission,
    .getattr      = nfs_getattr,
    .setattr      = nfs_setattr,
    .listxattr    = nfs_listxattr,
};
```

### NFS Read Path

```
read(fd, buf, 4096) on NFS-mounted file:
  │
  ▼
nfs_file_read()
  │
  ├── Check: is data in page cache AND still valid?
  │     NFS uses attribute cache timeout (default: 60s for files)
  │     If cache valid → return cached data (no network I/O!)
  │
  └── Cache miss or stale:
        │
        ├── nfs_readpage() or nfs_readahead()
        │     │
        │     ├── Build NFS READ RPC request
        │     │     ├── File handle
        │     │     ├── Offset
        │     │     ├── Count (bytes)
        │     │     └── Auth credentials
        │     │
        │     ├── Send via SUNRPC
        │     │     Client ──── READ(fh, off, cnt) ────► Server
        │     │     Client ◄── DATA(bytes) ────────────── Server
        │     │
        │     └── Store data in page cache
        │
        └── Return data to user

Caching in NFS:
  - Page cache: file data (as with local FS)
  - Attribute cache: inode attrs (size, mtime) with timeout
  - dentry cache: name lookup results with timeout
  - On timeout: revalidate with server (GETATTR RPC)
```

### NFSv4 Delegations

```
Delegation: Server grants client exclusive cache rights

  Without delegation:
    Client must revalidate with server for EVERY read
    → Network round-trip for every operation

  With delegation (NFSv4):
    Server → Client: "You have READ delegation for file X"
    Client can:
      - Cache data without revalidating
      - Read from cache freely
      - No network traffic needed!

    When another client opens file X:
      Server → Delegated Client: "Please return delegation"
      (CB_RECALL callback)
      Delegated Client: flush changes, return delegation
      → Server now coordinates normally

  Types:
    READ delegation:  client can cache reads
    WRITE delegation: client can cache reads + writes
                      (exclusive access)
```

### NFS Mounting

```bash
# NFSv4 mount
$ mount -t nfs4 server:/export/data /mnt/nfs
# Or with version specification:
$ mount -t nfs -o vers=4.2 server:/export/data /mnt/nfs

# Common options
$ mount -t nfs -o \
    vers=4.2,\          # NFS version
    rsize=1048576,\     # Read buffer size (1MB)
    wsize=1048576,\     # Write buffer size (1MB)
    hard,\              # Keep retrying on failure (vs soft)
    timeo=600,\         # RPC timeout (60 seconds)
    retrans=2,\         # RPC retransmissions
    sec=krb5,\          # Kerberos authentication
    noatime \           # Don't update access time
    server:/data /mnt

# Server export (/etc/exports):
/data  *(rw,sync,no_subtree_check,no_root_squash)

# Check NFS stats
$ nfsstat -c        # Client stats
$ nfsstat -s        # Server stats
$ mountstats /mnt   # Detailed per-mount stats
```

---

## 14.3 SMB/CIFS — Windows File Sharing

### Architecture

```
SMB (Server Message Block) / CIFS (Common Internet File System):
  - Native Windows file sharing protocol
  - Linux client: cifs.ko (kernel module)
  - Linux server: Samba (user-space) or ksmbd (kernel-space)

  ┌──────────────────┐        ┌──────────────────┐
  │ Linux Client     │        │ Windows Server   │
  │                  │        │ (or Samba/ksmbd) │
  │  cifs.ko         │ SMB3   │                  │
  │  (kernel VFS     │◄──────►│  NTFS / ext4     │
  │   module)        │ TCP    │                  │
  │                  │ 445    │                  │
  └──────────────────┘        └──────────────────┘

SMB versions:
  SMB 1.0  → Obsolete, security issues (EternalBlue)
  SMB 2.0  → Windows Vista, improved performance
  SMB 2.1  → Windows 7, oplock leases
  SMB 3.0  → Windows 8, encryption, multichannel
  SMB 3.1.1 → Windows 10, pre-auth integrity, AES-128-CCM
```

### SMB/CIFS Mounting on Linux

```bash
# Mount Windows/Samba share
$ mount -t cifs //server/share /mnt/smb \
    -o username=user,password=pass,\
       vers=3.0,\             # SMB version
       seal,\                 # Encrypt traffic
       cache=strict,\         # Cache mode
       file_mode=0644,\
       dir_mode=0755

# Using credentials file (more secure)
$ mount -t cifs //server/share /mnt/smb \
    -o credentials=/etc/samba/creds,vers=3.1.1

# /etc/samba/creds:
#   username=myuser
#   password=mypassword
#   domain=MYDOMAIN

# Automount via /etc/fstab:
# //server/share  /mnt/smb  cifs  credentials=/etc/samba/creds,vers=3.0  0  0
```

---

## 14.4 Cache Coherency

```
The fundamental challenge of network FS:
  Multiple clients may access the same file simultaneously.
  Each client has a local page cache.
  How to keep caches consistent?

Approaches:
  ┌──────────────────────────────────────────────────────────────┐
  │ Close-to-Open (CTO) — NFS default                           │
  │                                                              │
  │  open():  Revalidate file attributes with server             │
  │           If file changed: invalidate page cache              │
  │  read():  Use local page cache                               │
  │  write(): Buffer in page cache                               │
  │  close(): Flush dirty data to server                         │
  │                                                              │
  │  Guarantees: After close() returns, other clients             │
  │  opening the file will see all written data.                  │
  │                                                              │
  │  Gap: Concurrent access with both files open → stale data   │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │ Delegation — NFSv4                                           │
  │                                                              │
  │  Server grants delegation to single client                   │
  │  Client can cache freely until delegation recalled           │
  │  When another client accesses: delegation recalled            │
  │  → Stronger guarantee, better performance for single-client  │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │ Oplock / Lease — SMB                                         │
  │                                                              │
  │  Similar to delegation:                                      │
  │  Batch oplock: client can read, write, delete cached         │
  │  Exclusive oplock: client has exclusive access cached         │
  │  Level II oplock: multiple readers, no writers cached         │
  │  Break on conflict: server → client "release your oplock"   │
  └──────────────────────────────────────────────────────────────┘
```

---

## 14.5 Comparison: NFS vs SMB

```
Feature              │ NFS (v4.2)         │ SMB (3.1.1)
─────────────────────┼────────────────────┼────────────────────
Origin               │ Sun/Unix           │ Microsoft/Windows
Transport            │ TCP (RDMA for pNFS)│ TCP (RDMA, QUIC)
Port                 │ 2049               │ 445
Authentication       │ Kerberos, AUTH_SYS │ NTLM, Kerberos
Encryption           │ krb5p (RPC level)  │ AES-128-CCM/GCM
Case sensitivity     │ Yes (default)      │ No (preserving)
Symlinks             │ Full support       │ Limited
ACLs                 │ NFSv4 ACLs         │ Windows ACLs
Delegation           │ Read/Write deleg   │ Oplocks/Leases
Server-side copy     │ COPY operation     │ srv_copychunk
Linux client         │ nfs.ko (in-tree)   │ cifs.ko (in-tree)
Linux server         │ nfsd (kernel)      │ Samba or ksmbd
Best for             │ Unix/Linux env     │ Mixed Windows/Linux
```

---

## 14.6 9P — Lightweight Network FS

```
9P: Simple distributed file protocol from Plan 9:
  - Used by: QEMU virtio-9p, WSL2, ChromeOS
  - Very simple protocol (open, read, write, walk)
  - Low overhead, ideal for VMs

Example: Share host dir with QEMU VM
  $ qemu-system-x86_64 \
      -virtfs local,path=/shared,mount_tag=host_share,security_model=mapped

  Inside VM:
  $ mount -t 9p -o trans=virtio host_share /mnt/host

Linux implementation: fs/9p/
```

---

## Kernel Source References

```
NFS client:
  fs/nfs/                     ← NFS client implementation
  fs/nfs/file.c               ← File operations
  fs/nfs/dir.c                ← Directory operations
  fs/nfs/read.c               ← Read path
  fs/nfs/write.c              ← Write path
  fs/nfs/nfs4proc.c           ← NFSv4 procedures
  fs/nfs/delegation.c         ← Delegation handling
  net/sunrpc/                 ← SUNRPC framework

NFS server:
  fs/nfsd/                    ← NFS server (nfsd)

CIFS:
  fs/smb/client/              ← SMB/CIFS client
  fs/smb/client/file.c        ← File operations
  fs/smb/client/dir.c         ← Directory operations
  fs/smb/server/              ← ksmbd (in-kernel SMB server)

9P:
  fs/9p/                      ← 9P client
  net/9p/                     ← 9P transport
```

---

## Interview Questions

1. **How does NFS integrate with the Linux VFS? What VFS operations does NFS implement?**
2. **What is close-to-open cache consistency in NFS?**
3. **What are NFSv4 delegations? How do they improve performance?**
4. **Compare NFS and SMB/CIFS for a mixed Linux/Windows environment.**
5. **What happens when you read a NFS file — what network RPCs are involved?**
6. **What is a "hard" vs "soft" NFS mount? When would you use each?**
7. **How does the NFS client use the page cache?**
8. **What is pNFS (parallel NFS)? When is it useful?**
9. **What is 9P? Where is it commonly used?**
10. **What challenges does cache coherency pose for network file systems?**

---

## Summary

- Network FS: kernel VFS module translates file ops to network RPCs
- NFS: Unix-native, NFSv4.2 has delegation, server-side copy, labeled NFS
- SMB/CIFS: Windows-native, SMB3 has encryption, multichannel, oplocks
- Cache coherency: CTO (default), delegations (NFSv4), oplocks (SMB)
- NFS client uses page cache + attribute cache with timeouts
- NFS server (nfsd) runs in kernel; Samba/ksmbd serves SMB
- 9P: lightweight protocol used in QEMU/virtio, WSL2
- Key trade-off: aggressive caching → better performance but stale data risk

---

*Next: [Chapter 15 — Memory-Mapped Files and mmap](Chapter_15_Mmap.md)*
