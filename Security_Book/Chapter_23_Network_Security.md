# Chapter 23: Network Security in Kernel

## Learning Goals
- Understand kernel network security mechanisms
- Know IPsec, Netfilter, and TLS offload
- Understand network namespaces for isolation
- Know Linux firewall architecture (nftables/iptables)

---

## 23.1 Netfilter / nftables Firewall

```
Netfilter: Kernel framework for packet filtering, NAT, and mangling.
nftables: Modern replacement for iptables (since Linux 3.13).

Netfilter hook points:
  ┌──────────────────────────────────────────────────────────┐
  │  Incoming      PREROUTING ──► ROUTING ──► FORWARD ──►   │
  │  Packet          │              │           │            │
  │                  ▼              ▼           ▼            │
  │              (DNAT/mangle)  INPUT      POSTROUTING ──►   │
  │                              │              ▲        Out │
  │                              ▼              │            │
  │                          Local Process ──► OUTPUT ──►    │
  └──────────────────────────────────────────────────────────┘

  5 hook points: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING
  Chains at each point process packets in rule order.

nftables example:
  # Create table and chains
  nft add table inet filter
  nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
  nft add chain inet filter forward { type filter hook forward priority 0 \; policy drop \; }
  nft add chain inet filter output { type filter hook output priority 0 \; policy accept \; }

  # Allow established/related connections
  nft add rule inet filter input ct state established,related accept
  # Allow SSH
  nft add rule inet filter input tcp dport 22 accept
  # Allow ICMP
  nft add rule inet filter input meta l4proto icmp accept
  # Drop everything else (chain policy)

Security best practices:
  - Default policy: DROP (deny all, allow specific)
  - Stateful filtering: track connection state
  - Rate limiting: prevent SYN floods
  - Log before drop: forensic analysis
  - Separate chains for different trust zones
```

---

## 23.2 IPsec (Kernel Implementation)

```
IPsec: Kernel-level encryption for IP traffic (VPN).

  Modes:
    Transport mode: Encrypt payload, keep IP header
    Tunnel mode:    Encrypt entire original packet, new IP header

  Protocols:
    AH (Authentication Header):  Integrity + authentication only
    ESP (Encapsulating Security Payload): Encryption + auth

  ┌───────────────────────────────────────────────────────┐
  │  IPsec Tunnel Mode:                                    │
  │                                                        │
  │  Original:  [IP_hdr][TCP_hdr][Data]                    │
  │                                                        │
  │  After ESP: [New_IP][ESP_hdr][IP_hdr][TCP_hdr][Data]   │
  │              ↑ plain  ↑ plain  └──── encrypted ────┘    │
  │              new dst  SPI/seq                           │
  └───────────────────────────────────────────────────────┘

Kernel components:
  ┌────────────────────────────────────────────┐
  │  XFRM (Transform) Framework                │
  │                                            │
  │  SA (Security Association) database:        │
  │    ip xfrm state — active ESP/AH states    │
  │    SPI, keys, algorithms, lifetime          │
  │                                            │
  │  SP (Security Policy) database:             │
  │    ip xfrm policy — which traffic to encrypt│
  │    Selectors: src/dst IP, port, protocol    │
  │                                            │
  │  IKE daemon (user space):                   │
  │    strongSwan / libreswan                   │
  │    Negotiates SAs, manages keying           │
  └────────────────────────────────────────────┘

  Algorithms used:
    ESP: AES-GCM (preferred), AES-CBC + HMAC-SHA256
    IKEv2: Diffie-Hellman key exchange
    XFRM uses kernel Crypto API for all operations
```

---

## 23.3 Kernel TLS (kTLS)

```
Kernel TLS: Offload TLS encryption to kernel for performance.

  Traditional:
    Application → user-space TLS lib → syscall → kernel → NIC
    Double copy + context switches

  With kTLS:
    Application → sendfile() → kernel TLS → NIC
    Zero-copy, no user-space crypto overhead

  ┌──────────────────────────────────────────────────┐
  │  Application                                      │
  │    setsockopt(sock, SOL_TLS, TLS_TX, &crypto_info)│
  │    send(sock, data, len, 0)                       │
  │         │                                         │
  │         ▼                                         │
  │  Kernel: net/tls/                                 │
  │    TLS record layer (tls_sw_sendmsg)              │
  │    Encrypt using kernel Crypto API                │
  │    Or hardware offload to NIC                     │
  │         │                                         │
  │         ▼                                         │
  │  NIC (optional hardware TLS offload)              │
  │    Mellanox ConnectX / Intel E810                  │
  └──────────────────────────────────────────────────┘

  Setup flow:
    1. User-space TLS library does handshake (negotiate cipher)
    2. Pass negotiated keys to kernel via setsockopt()
    3. Kernel handles record encryption/decryption
    4. sendfile() works with encrypted connection (zero-copy)
```

---

## 23.4 Network Security Features

```
Connection tracking (conntrack):
  Tracks TCP/UDP/ICMP connection state.
  Enables stateful firewalling:
    NEW → ESTABLISHED → RELATED
  Only accept packets belonging to known connections.

SYN cookies:
  /proc/sys/net/ipv4/tcp_syncookies = 1
  Mitigate SYN flood DoS attacks.
  Kernel doesn't allocate TCB until handshake completes.

TCP hardening:
  net.ipv4.tcp_rfc1337 = 1         # TIME-WAIT assassination fix
  net.ipv4.conf.all.rp_filter = 1  # Reverse path filtering
  net.ipv4.conf.all.accept_redirects = 0  # Ignore ICMP redirects
  net.ipv4.conf.all.send_redirects = 0
  net.ipv4.conf.all.accept_source_route = 0  # No source routing
  net.ipv4.icmp_echo_ignore_broadcasts = 1   # Ignore broadcast ping

IPv6 hardening:
  net.ipv6.conf.all.accept_redirects = 0
  net.ipv6.conf.all.accept_ra = 0          # No Router Advertisements
  net.ipv6.conf.all.accept_source_route = 0

Socket security:
  SO_BINDTODEVICE — bind socket to specific interface
  SO_MARK — mark packets for policy routing
  IP_TRANSPARENT — for transparent proxying
```

---

## 23.5 eBPF for Network Security

```
eBPF: Programmable security enforcement in the network stack.

  XDP (eXpress Data Path):
    Process packets at driver level (before kernel stack).
    Fastest possible filtering — millions of packets/sec.
    Use: DDoS mitigation, firewalling.

  TC (Traffic Control) eBPF:
    Per-packet processing at qdisc level.
    Rich packet inspection and modification.

  Socket-level eBPF:
    cgroup/connect4: Control which IPs/ports apps can connect to
    cgroup/bind4: Control which ports apps can bind
    cgroup/sendmsg4: Filter outgoing UDP messages

  ┌──────────────────────────────────────────────────┐
  │  eBPF Network Security Points                     │
  │                                                   │
  │  NIC → [XDP] → Driver → [TC ingress] →            │
  │    → Network stack → [Netfilter] →                 │
  │    → Socket → [cgroup/sock] → Application          │
  │                                                   │
  │  Application → [cgroup/sock] → Socket →            │
  │    → [TC egress] → Driver → NIC                   │
  │                                                   │
  │  Each point can:                                   │
  │    DROP: discard packet                            │
  │    PASS/ACCEPT: allow packet                       │
  │    REDIRECT: send to different interface/CPU       │
  └──────────────────────────────────────────────────┘

  Cilium (Kubernetes CNI):
    Uses eBPF for network policy enforcement.
    Replaces iptables with eBPF programs.
    Identity-based security (not just IP-based).
```

---

## 23.6 WireGuard VPN

```
WireGuard: Modern in-kernel VPN. Simple, fast, auditable.

  Protocol:
    Noise protocol framework
    ChaCha20 (encryption)
    Poly1305 (MAC)
    Curve25519 (key exchange)
    BLAKE2s (hashing)
    SipHash (hashtable keys)

  Kernel module: drivers/net/wireguard/

  ┌──────────────────────────────────────────────────┐
  │  WireGuard Architecture                           │
  │                                                   │
  │  Application                                      │
  │    │                                              │
  │    ▼                                              │
  │  wg0 interface (virtual)                          │
  │    │                                              │
  │    ▼                                              │
  │  WireGuard kernel module                          │
  │    Encrypt with ChaCha20-Poly1305                 │
  │    Encapsulate in UDP                             │
  │    │                                              │
  │    ▼                                              │
  │  eth0 (physical) → Network → Remote peer          │
  └──────────────────────────────────────────────────┘

  ~4000 lines of code (vs OpenVPN ~100,000+)
  Easy to audit, formally verified protocol.
  In mainline kernel since 5.6.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/netfilter/ | Netfilter framework and conntrack |
| net/netfilter/nf_tables*.c | nftables core |
| net/xfrm/ | XFRM/IPsec transform framework |
| net/ipv4/esp4.c | ESP (IPv4) implementation |
| net/tls/ | Kernel TLS |
| drivers/net/wireguard/ | WireGuard VPN |
| net/core/filter.c | BPF/eBPF socket filtering |

---

## Interview Questions

**Q1: How does Netfilter provide network security?**
A: Netfilter provides 5 hook points in the packet path: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING. Rules at each hook inspect packets and decide: ACCEPT, DROP, REJECT, or LOG. nftables (modern) or iptables (legacy) configure these rules. Connection tracking (conntrack) enables stateful filtering — only packets belonging to established connections pass. Best practice is default-DROP policy with explicit allow rules. Netfilter also supports NAT, rate limiting (SYN flood protection), and deep packet marking. eBPF-based filtering (XDP, TC) can replace or supplement Netfilter for higher performance.

**Q2: How does kernel TLS offload improve security and performance?**
A: Kernel TLS (kTLS) moves TLS record-layer encryption from user-space libraries into the kernel. After the user-space TLS library completes the handshake and negotiates keys, it passes the session keys to the kernel via `setsockopt(SOL_TLS, TLS_TX)`. The kernel then handles encryption for all subsequent data. This enables `sendfile()` to work with TLS — data goes directly from page cache to encrypted network output with zero copies to user space. Some NICs (Mellanox, Intel) can further offload TLS to hardware. The security benefit: fewer copies of plaintext data in memory, and the kernel enforces correct TLS record framing.

**Q3: Why is WireGuard considered more secure than older VPN solutions?**
A: WireGuard is ~4000 lines of code versus 100,000+ for OpenVPN or IPsec implementations. The small codebase is fully auditable — it's been formally verified. It uses modern, conservative cryptography: ChaCha20-Poly1305 (AEAD), Curve25519 (DH), BLAKE2s (hashing). There's no cipher agility — no negotiation of weak algorithms. The protocol uses the Noise framework with provable security properties. It runs in the kernel (not user space) for minimal attack surface and better performance. It's stateless (no connection state to DoS attack) and uses a cookie mechanism for DDoS protection. Included in mainline Linux since 5.6.

---

## Summary

- Netfilter/nftables: 5 hook points, stateful filtering, default-DROP best practice
- IPsec (XFRM): kernel-level VPN with ESP/AH, SA/SP databases
- Kernel TLS (kTLS): offload TLS record layer to kernel, enables sendfile() with TLS
- Network hardening: SYN cookies, rp_filter, disable redirects/source routing
- eBPF: programmable network security at XDP/TC/socket levels
- WireGuard: ~4000 LOC modern VPN, in kernel since 5.6
- Cilium: eBPF-based Kubernetes network policy enforcement

---

Next: [Chapter 24 — Container Security](Chapter_24_Container_Security.md)
