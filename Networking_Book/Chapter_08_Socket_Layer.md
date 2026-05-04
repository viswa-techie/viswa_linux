# Chapter 8: Socket Layer

## Learning Goals
- Understand the socket abstraction and its VFS integration
- Know the relationship between struct socket and struct sock
- Understand protocol family and type dispatch
- Know the socket system call flow from user space to kernel
- Understand socket options and buffer management

---

## 8.1 Socket Abstraction

Sockets are the primary user-space API for network communication. In Linux, a socket is a file descriptor backed by a special filesystem.

```
User Space:
  fd = socket(AF_INET, SOCK_STREAM, 0);
  bind(fd, ...);
  listen(fd, backlog);
  client_fd = accept(fd, ...);
  send(client_fd, data, len, 0);
  recv(client_fd, buf, len, 0);
  close(fd);

Kernel Structures:
  ┌──────────────────────────────────────────────────────────┐
  │ File Descriptor Table (per process)                     │
  │  fd[3] ──► struct file ──► struct socket                │
  │                              │                           │
  │                              ├── type (SOCK_STREAM)     │
  │                              ├── ops (inet_stream_ops)  │
  │                              └── sk ──► struct sock     │
  │                                          │               │
  │                                          ├── sk_prot     │
  │                                          │   (tcp_prot)  │
  │                                          ├── sk_rcvbuf   │
  │                                          ├── sk_sndbuf   │
  │                                          ├── sk_state    │
  │                                          └── (TCP state) │
  └──────────────────────────────────────────────────────────┘
```

---

## 8.2 Key Structures

### struct socket (User-Facing)

```c
struct socket {
    socket_state         state;    /* SS_UNCONNECTED, SS_CONNECTED, etc. */
    short                type;     /* SOCK_STREAM, SOCK_DGRAM, SOCK_RAW */
    unsigned long        flags;    /* SOCK_ASYNC_NOSPACE, etc. */
    struct file         *file;     /* Back-pointer to file descriptor */
    struct sock         *sk;       /* Protocol-level socket state */
    const struct proto_ops *ops;   /* Protocol operations */
};
```

### struct sock (Protocol-Level)

```c
struct sock {
    /* Network addresses */
    struct sock_common  __sk_common;  /* includes family, state, hash */

    /* Receive queue */
    struct sk_buff_head  sk_receive_queue;  /* Incoming data */
    int                  sk_rcvbuf;         /* Receive buffer limit */

    /* Send queue */
    struct sk_buff_head  sk_write_queue;    /* Outgoing data */
    int                  sk_sndbuf;         /* Send buffer limit */

    /* Socket state */
    unsigned char        sk_state;          /* TCP state (ESTABLISHED, etc.) */
    unsigned short       sk_type;           /* SOCK_STREAM, SOCK_DGRAM */

    /* Protocol */
    struct proto        *sk_prot;           /* Protocol callbacks */

    /* Callbacks */
    void (*sk_data_ready)(struct sock *sk);     /* Data available */
    void (*sk_write_space)(struct sock *sk);    /* Write space available */
    void (*sk_state_change)(struct sock *sk);   /* State changed */

    /* Timestamps, filters, options */
    /* ... many more fields ... */
};
```

---

## 8.3 Protocol Family and Type Dispatch

```
socket(domain, type, protocol)
   │
   ▼
┌─────────────────────────────────────────────────────────┐
│ Domain (Address Family)     Protocol Operations         │
├─────────────────────────────────────────────────────────┤
│ AF_INET  (IPv4)       →    inet_create()               │
│ AF_INET6 (IPv6)       →    inet6_create()              │
│ AF_UNIX  (local)      →    unix_create()               │
│ AF_PACKET (raw L2)    →    packet_create()             │
│ AF_NETLINK (kernel)   →    netlink_create()            │
│ AF_CAN   (CAN bus)    →    can_create()                │
└─────────────────────────────────────────────────────────┘
   │
   ▼ For AF_INET:
┌─────────────────────────────────────────────────────────┐
│ Type                    Protocol Struct     proto_ops   │
├─────────────────────────────────────────────────────────┤
│ SOCK_STREAM (TCP)  →   tcp_prot        inet_stream_ops │
│ SOCK_DGRAM  (UDP)  →   udp_prot        inet_dgram_ops │
│ SOCK_RAW    (raw)  →   raw_prot        inet_sockraw_ops│
└─────────────────────────────────────────────────────────┘
```

### proto_ops (Socket-Level Operations)

```c
/* inet_stream_ops for TCP */
const struct proto_ops inet_stream_ops = {
    .family     = PF_INET,
    .bind       = inet_bind,
    .connect    = inet_stream_connect,
    .listen     = inet_listen,
    .accept     = inet_accept,
    .sendmsg    = inet_sendmsg,       /* → tcp_sendmsg */
    .recvmsg    = inet_recvmsg,       /* → tcp_recvmsg */
    .setsockopt = sock_common_setsockopt,
    .getsockopt = sock_common_getsockopt,
    .poll       = tcp_poll,
    .shutdown   = inet_shutdown,
    .release    = inet_release,
    .ioctl      = inet_ioctl,
    .mmap       = sock_no_mmap,
    .sendpage   = inet_sendpage,
};
```

---

## 8.4 Socket System Call Flow

### socket() — Create Socket

```
User: fd = socket(AF_INET, SOCK_STREAM, 0)
   │
   ▼
sys_socket()                    [net/socket.c]
   │
   ├── sock_create()
   │   ├── Lookup PF_INET in net_families[]
   │   ├── Allocate struct socket
   │   └── inet_create()          [net/ipv4/af_inet.c]
   │       ├── Lookup SOCK_STREAM → tcp_prot
   │       ├── sk_alloc()         → allocate struct sock
   │       ├── sock_init_data()   → init queues, callbacks
   │       └── tcp_v4_init_sock() → TCP-specific init
   │
   ├── sock_map_fd()
   │   ├── get_unused_fd()
   │   ├── sock_alloc_file()      → create struct file
   │   └── fd_install()           → link fd → file
   │
   └── Return fd to user space
```

### send() — Transmit Data

```
User: send(fd, buf, len, 0)
   │
   ▼
sys_sendto()                      [net/socket.c]
   │
   ├── sockfd_lookup_light(fd)    → find struct socket from fd
   │
   ├── sock_sendmsg()
   │   └── sock->ops->sendmsg()
   │       └── inet_sendmsg()
   │           └── sk->sk_prot->sendmsg()
   │               └── tcp_sendmsg()     [net/ipv4/tcp.c]
   │                   ├── Copy data from user to skb
   │                   ├── Add to sk_write_queue
   │                   └── tcp_push()
   │                       └── tcp_write_xmit()
   │                           └── ip_queue_xmit()
   │                               └── ... → driver → wire
   │
   └── Return bytes sent
```

### recv() — Receive Data

```
User: recv(fd, buf, len, 0)
   │
   ▼
sys_recvfrom()                    [net/socket.c]
   │
   ├── sock->ops->recvmsg()
   │   └── tcp_recvmsg()          [net/ipv4/tcp.c]
   │       ├── Lock socket
   │       ├── If sk_receive_queue empty:
   │       │   └── sk_wait_data()  → sleep (wait for data)
   │       ├── Copy data from skb to user buffer
   │       ├── Update TCP window
   │       └── Free consumed sk_buffs
   │
   └── Return bytes received
```

---

## 8.5 Socket Buffers and Flow Control

```
                     ┌───────────────────────┐
                     │      struct sock      │
                     │                       │
  send()/write() ──► │  sk_write_queue       │ ──► TCP output
  (user data)        │  sk_sndbuf (limit)    │
                     │  sk_wmem_alloc (used)  │
                     │                       │
  recv()/read()  ◄── │  sk_receive_queue     │ ◄── TCP input
  (user buffer)      │  sk_rcvbuf (limit)    │
                     │  sk_rmem_alloc (used)  │
                     └───────────────────────┘

Flow control:
  TX: If sk_wmem_alloc > sk_sndbuf → send() blocks or returns EAGAIN
  RX: If sk_rmem_alloc > sk_rcvbuf → TCP advertises zero window → sender stops
```

### Socket Buffer Sizes

```bash
# Default and max buffer sizes
sysctl net.core.rmem_default    # Default receive buffer (208KB)
sysctl net.core.rmem_max        # Max receive buffer (8MB)
sysctl net.core.wmem_default    # Default send buffer (208KB)
sysctl net.core.wmem_max        # Max send buffer (8MB)

# TCP auto-tuning (min, default, max)
sysctl net.ipv4.tcp_rmem        # "4096 131072 6291456"
sysctl net.ipv4.tcp_wmem        # "4096 16384 4194304"

# Per-socket override
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &size, sizeof(size));
```

---

## 8.6 Socket Options

```c
/* Important socket options */

/* SOL_SOCKET level */
SO_REUSEADDR    /* Allow bind to address in TIME_WAIT */
SO_REUSEPORT    /* Multiple sockets bind to same port */
SO_KEEPALIVE    /* Enable TCP keepalive probes */
SO_RCVBUF       /* Receive buffer size */
SO_SNDBUF       /* Send buffer size */
SO_LINGER       /* Linger on close */
SO_RCVTIMEO     /* Receive timeout */
SO_SNDTIMEO     /* Send timeout */
SO_BINDTODEVICE /* Bind to specific interface */
SO_PRIORITY     /* QoS priority */
SO_MARK         /* Netfilter mark */
SO_TIMESTAMP    /* Receive timestamps */

/* IPPROTO_TCP level */
TCP_NODELAY     /* Disable Nagle algorithm */
TCP_CORK        /* Delay sends to accumulate data */
TCP_KEEPIDLE    /* Time before first keepalive */
TCP_KEEPINTVL   /* Interval between keepalives */
TCP_KEEPCNT     /* Max keepalive probes */
TCP_MAXSEG      /* Maximum segment size */
TCP_FASTOPEN    /* TFO support */
TCP_CONGESTION  /* Congestion control algorithm */

/* IPPROTO_IP level */
IP_TOS          /* Type of service */
IP_TTL          /* Time to live */
IP_MULTICAST_IF /* Multicast interface */
IP_ADD_MEMBERSHIP /* Join multicast group */
```

---

## 8.7 Socket VFS Integration

```
Socket as a File:

  Sockets are files in Linux. socket() returns an fd.
  The special "sockfs" filesystem backs socket files.

  ┌──────────┐     ┌──────────┐     ┌──────────────┐
  │    fd    │────►│ struct   │────►│ struct socket │
  │ (int)    │     │ file     │     │  .ops = ...   │
  └──────────┘     │ .f_op=   │     │  .sk = ...    │
                   │socket_ops│     └──────────────┘
                   └──────────┘

  Because sockets are files:
    - read(fd)/write(fd) work (dispatch to recvmsg/sendmsg)
    - poll(fd)/epoll work (dispatch to socket poll)
    - close(fd) works (release socket)
    - dup(fd) works (share socket)
    - sendfile(fd_out, fd_in) works (zero-copy from file to socket)
```

---

## 8.8 Multiplexing: select, poll, epoll

```
select/poll:
  - O(n) scan of all fds each call
  - Copy fd sets to/from kernel each call
  - Limited scalability (1024 fds for select)

epoll (Linux-specific, preferred):
  - O(1) per event
  - epoll_create() → epoll instance (backed by red-black tree)
  - epoll_ctl(ADD/MOD/DEL) → register interest
  - epoll_wait() → block until events ready

  Internal mechanism:
    When data arrives on a socket:
    1. sk_data_ready() callback fires
    2. Wakes up wait queue
    3. epoll checks socket's poll status
    4. Reports as ready to user in epoll_wait()

  Scalability: handles 100K+ connections (C10K/C100K problem)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/socket.c | Socket system calls, VFS integration |
| net/ipv4/af_inet.c | AF_INET socket creation, inet_stream_ops |
| net/ipv6/af_inet6.c | AF_INET6 socket creation |
| net/unix/af_unix.c | AF_UNIX (local) sockets |
| include/net/sock.h | struct sock definition |
| include/linux/net.h | struct socket, proto_ops |
| net/core/sock.c | sock_init_data, buffer management |
| fs/eventpoll.c | epoll implementation |

---

## Interview Questions

**Q1: What is the relationship between struct socket and struct sock?**
A: struct socket is the user-facing VFS-integrated object — it has a file descriptor, type (STREAM/DGRAM), and proto_ops for system call dispatch. struct sock (sk) is the protocol-level state — it holds send/receive queues, TCP state machine, buffer sizes, timers, and the protocol-specific callbacks (sk_prot). socket->sk points to the sock. Multiple protocol families reuse the sock infrastructure with different sk_prot implementations.

**Q2: How does the kernel dispatch socket calls to the right protocol?**
A: Two levels of dispatch. socket() looks up the address family (AF_INET) in net_families[] to call the family's create function (inet_create). inet_create looks up the socket type (SOCK_STREAM) to find the protocol struct (tcp_prot). Subsequently, socket operations like send() go through proto_ops (inet_stream_ops)→sendmsg → sk_prot (tcp_prot)→sendmsg → tcp_sendmsg().

**Q3: What is SO_REUSEPORT and why is it useful?**
A: SO_REUSEPORT allows multiple sockets (even in different processes) to bind to the same IP:port. The kernel distributes incoming connections across the sockets using a hash of the source IP/port. This enables multi-process servers where each process has its own accept() loop — no need for a single listener distributing connections. Used by nginx, HAProxy for better multi-core scaling.

**Q4: How does epoll differ from select/poll and why is it more scalable?**
A: select/poll are O(n): every call copies fd sets to kernel, kernel scans all fds, copies back results. epoll is O(active): epoll_create creates a kernel data structure, epoll_ctl registers interest once, epoll_wait returns only ready fds. Internally, callbacks from socket wake-ups add entries to a ready list. With 100K sockets where 10 are active, select scans 100K; epoll returns 10.

**Q5: How does the kernel wait when recv() is called and no data is available?**
A: tcp_recvmsg() checks sk_receive_queue. If empty (and socket is blocking), it calls sk_wait_data() which adds the current task to the socket's wait queue and calls schedule() (puts task to sleep). When data arrives, tcp_v4_rcv() queues the skb and calls sk_data_ready() which wakes up the sleeping task via wake_up_interruptible(). The task resumes in tcp_recvmsg() and copies data to user buffer.

---

## Summary

- Sockets are file descriptors backed by sockfs — VFS integration enables read/write/poll
- struct socket is user-facing; struct sock holds protocol state and queues
- Two-level dispatch: proto_ops for system calls, sk_prot for protocol operations
- Buffer management: sk_sndbuf/sk_rcvbuf limit queue sizes; TCP auto-tunes
- epoll provides O(1) scalable event notification for high-connection-count servers
- Socket options control behavior: TCP_NODELAY, SO_REUSEPORT, SO_KEEPALIVE

---

Next: [Chapter 9 — Transport Layer: TCP Deep Dive](Chapter_09_TCP.md)
