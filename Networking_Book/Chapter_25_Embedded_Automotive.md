# Chapter 25: Embedded and Automotive Networking

## Learning Goals
- Understand CAN bus and SocketCAN in Linux
- Know automotive Ethernet (100BASE-T1, 1000BASE-T1)
- Understand TSN (Time-Sensitive Networking) for real-time
- Know SOME/IP and DDS for automotive service-oriented communication
- Understand network challenges in embedded/constrained devices

---

## 25.1 CAN Bus in Linux (SocketCAN)

```
CAN (Controller Area Network): Serial bus for automotive ECUs.
Arbitration-based, multi-master, real-time capable.

SocketCAN: Linux kernel CAN subsystem using standard socket API.

Architecture:
  ┌────────────────────────────────────────────────────┐
  │  User Application                                  │
  │    socket(AF_CAN, SOCK_RAW, CAN_RAW)              │
  │    bind(s, &addr, sizeof(addr))  /* addr.can_ifindex */
  │    read(s, &frame, sizeof(frame))                  │
  │    write(s, &frame, sizeof(frame))                 │
  │                                                    │
  │  ┌──────────────────────────────────────────┐      │
  │  │ CAN Protocol Family (AF_CAN)            │      │
  │  │   CAN_RAW:   Raw CAN frames             │      │
  │  │   CAN_BCM:   Broadcast Manager          │      │
  │  │   CAN_ISOTP: ISO 15765-2 Transport      │      │
  │  │   CAN_J1939: SAE J1939 Protocol         │      │
  │  └──────────────────────────────────────────┘      │
  │                                                    │
  │  ┌──────────────────────────────────────────┐      │
  │  │ CAN Network Device (net/can/)            │      │
  │  │   can_rx_register() — filter by CAN ID   │      │
  │  │   can_send() — transmit frame             │      │
  │  └──────────────────────────────────────────┘      │
  │                                                    │
  │  ┌──────────────────────────────────────────┐      │
  │  │ CAN Controller Driver                    │      │
  │  │   mcp251x (SPI), flexcan, peak_pci, etc. │      │
  │  └──────────────────────────────────────────┘      │
  │                    │                               │
  │                    ▼                               │
  │              [CAN Bus Hardware]                     │
  └────────────────────────────────────────────────────┘

CAN frame:
  ┌──────┬───────┬─────┬─────┬──────────────────┐
  │ ID   │ RTR   │ IDE │ DLC │ Data (0-8 bytes) │
  │11/29b│1 bit  │1 bit│4 bit│                  │
  └──────┴───────┴─────┴─────┴──────────────────┘

CAN FD frame: up to 64 bytes data, higher bitrate data phase
```

```c
/* SocketCAN: Send and receive CAN frames */
#include <linux/can.h>
#include <linux/can/raw.h>
#include <net/if.h>

int s = socket(PF_CAN, SOCK_RAW, CAN_RAW);

struct ifreq ifr;
strncpy(ifr.ifr_name, "can0", IFNAMSIZ);
ioctl(s, SIOCGIFINDEX, &ifr);

struct sockaddr_can addr = {
    .can_family  = AF_CAN,
    .can_ifindex = ifr.ifr_ifindex,
};
bind(s, (struct sockaddr *)&addr, sizeof(addr));

/* Send */
struct can_frame frame = {
    .can_id  = 0x123,
    .can_dlc = 4,
    .data    = {0xDE, 0xAD, 0xBE, 0xEF},
};
write(s, &frame, sizeof(frame));

/* Receive */
struct can_frame rx;
read(s, &rx, sizeof(rx));
printf("ID: 0x%X DLC: %d Data: %02X %02X ...\n",
       rx.can_id, rx.can_dlc, rx.data[0], rx.data[1]);

/* Filter: only receive ID 0x200-0x2FF */
struct can_filter rfilter = {
    .can_id   = 0x200,
    .can_mask = 0x700,
};
setsockopt(s, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter, sizeof(rfilter));
```

```bash
# CAN utilities (can-utils):
ip link set can0 type can bitrate 500000
ip link set can0 up

cansend can0 123#DEADBEEF        # Send frame
candump can0                      # Receive/display frames
cangen can0                       # Generate random traffic
canbusload can0                   # Monitor bus load
```

---

## 25.2 Automotive Ethernet

```
Automotive Ethernet: Ethernet for in-vehicle networks.
Replaces CAN for high-bandwidth domains (ADAS, infotainment).

Physical Layer:
  ┌───────────────────────────────────────────────────────────┐
  │ Standard        │ Speed    │ Cable   │ Use Case           │
  ├─────────────────┼──────────┼─────────┼────────────────────┤
  │ 100BASE-T1      │ 100 Mbps │ 1 pair  │ Body, chassis      │
  │ 1000BASE-T1     │ 1 Gbps   │ 1 pair  │ ADAS, backbone     │
  │ 10BASE-T1S      │ 10 Mbps  │ 1 pair  │ Low-cost sensors   │
  │                 │          │ multidrop│                     │
  │ 2.5/5/10GBASE-T1│ Multi-Gb │ 1 pair  │ Future backbone    │
  └─────────────────┴──────────┴─────────┴────────────────────┘

  Single unshielded twisted pair: lighter, cheaper, automotive-grade
  BroadR-Reach / OPEN Alliance TC standards

Vehicle network topology:
  ┌─────────────────────────────────────────────────┐
  │                                                  │
  │  ┌─────────┐       ┌──────────────┐             │
  │  │ Camera  │──T1──►│ Central      │             │
  │  │ (ADAS)  │ 1Gbps │ Gateway/     │             │
  │  └─────────┘       │ Domain       │             │
  │                    │ Controller   │             │
  │  ┌─────────┐       │              │             │
  │  │ Radar   │──T1──►│  Ethernet    │             │
  │  └─────────┘ 100M  │  Switch      │──── CAN ───│
  │                    │  + Router    │   (legacy)  │
  │  ┌─────────┐       │              │             │
  │  │ IVI     │──T1──►│              │             │
  │  │ (HU)    │ 1Gbps │              │             │
  │  └─────────┘       └──────────────┘             │
  └─────────────────────────────────────────────────┘

  Protocols over automotive Ethernet:
    SOME/IP:  Service-Oriented Middleware
    DoIP:     Diagnostics over IP (ISO 13400)
    AVB/TSN:  Audio/Video Bridging, Time-Sensitive Networking
    TCP/UDP:  Standard IP stack
```

---

## 25.3 TSN (Time-Sensitive Networking)

```
TSN: IEEE 802.1 standards for deterministic, low-latency Ethernet.
Critical for automotive, industrial, and audio/video applications.

Key TSN standards:
  ┌─────────────────────────────────────────────────────────────┐
  │ Standard      │ Purpose                                     │
  ├───────────────┼─────────────────────────────────────────────┤
  │ 802.1AS       │ gPTP: Time synchronization (< 1 us)        │
  │ 802.1Qbv      │ TAS: Time-Aware Shaper (scheduled traffic) │
  │ 802.1Qav      │ CBS: Credit-Based Shaper (AVB traffic)     │
  │ 802.1Qci      │ PSFP: Per-Stream Filtering and Policing    │
  │ 802.1CB       │ FRER: Frame Replication and Elimination     │
  │ 802.1Qcc      │ SRP: Stream Reservation Protocol enhanced  │
  └───────────────┴─────────────────────────────────────────────┘

Time-Aware Shaper (802.1Qbv):
  Opens/closes traffic gates on schedule (gate control list)
  ┌───────────────────── Time ────────────────────────┐
  │ T0      T1      T2      T3      T4      T5       │
  │ ┌──────┐        ┌──────┐        ┌──────┐         │
  │ │GATE  │ closed │GATE  │ closed │GATE  │          │
  │ │OPEN  │        │OPEN  │        │OPEN  │          │
  │ │Queue │        │Queue │        │Queue │          │
  │ │  0   │        │  0   │        │  0   │          │
  │ └──────┘        └──────┘        └──────┘          │
  │        ┌──────┐        ┌──────┐        ┌──────┐   │
  │        │GATE  │        │GATE  │        │GATE  │   │
  │        │OPEN  │        │OPEN  │        │OPEN  │   │
  │        │Queue │        │Queue │        │Queue │   │
  │        │ 1-7  │        │ 1-7  │        │ 1-7  │   │
  │        └──────┘        └──────┘        └──────┘   │
  └───────────────────────────────────────────────────┘

  Queue 0: Critical real-time (e.g., control messages)
  Queue 1-7: Best effort (standard traffic)
  Deterministic latency for scheduled traffic

Linux TSN support:
  tc qdisc add dev eth0 parent root handle 100 taprio \
     num_tc 3 \
     map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 \
     queues 1@0 1@1 2@2 \
     base-time 1000000000 \
     sched-entry S 01 300000 \
     sched-entry S 06 700000 \
     clockid CLOCK_TAI

  ptp4l: IEEE 1588 / 802.1AS time sync daemon
  phc2sys: Sync PTP clock to system clock
```

---

## 25.4 SOME/IP (Service-Oriented Middleware over IP)

```
SOME/IP: AUTOSAR-standardized middleware for automotive Ethernet.

Concepts:
  - Services: Named interfaces (e.g., "VehicleSpeed")
  - Methods: Request/Response (RPC)
  - Events: Publish/Subscribe
  - Fields: Get/Set/Notify
  - Service Discovery (SD): Find services on network

SOME/IP message format:
  ┌──────────────────────────────────────────────┐
  │ Service ID (16b) │ Method ID (16b)           │
  ├──────────────────┼───────────────────────────┤
  │ Length (32b)                                  │
  ├──────────────────┼───────────────────────────┤
  │ Client ID (16b)  │ Session ID (16b)          │
  ├──────────────────┼───────────────────────────┤
  │ Proto Ver │ If Ver│ Msg Type  │ Return Code  │
  ├──────────────────┴───────────────────────────┤
  │ Payload (serialized data)                     │
  └──────────────────────────────────────────────┘

SOME/IP-SD (Service Discovery):
  - Multicast offer/find messages
  - Service instances identified by ServiceID + InstanceID
  - Subscribe/Unsubscribe for event groups

Linux implementation:
  - vsomeip: COVESA open-source SOME/IP stack
  - Runs over UDP (unicast/multicast) and TCP
  - Configuration via JSON files
  - No kernel module needed — user-space library
```

---

## 25.5 Embedded Linux Networking Considerations

```
Resource-constrained environments:

Memory optimization:
  - Reduce sk_buff overhead: CONFIG_SLUB_TINY
  - Minimize socket buffer sizes (rmem/wmem)
  - Disable unused protocols (CONFIG_IPV6=n if not needed)
  - Use lightweight TCP/IP stacks for microcontrollers (lwIP, uIP)

CPU optimization:
  - Disable GRO/TSO if hardware doesn't support
  - Use NAPI properly (appropriate budget)
  - Pin network IRQs to specific CPU
  - Interrupt coalescing for power/performance balance

Real-time networking:
  - PREEMPT_RT kernel for deterministic latency
  - IRQ threading: threaded interrupts with RT priority
  - Network stack priority: sched_setscheduler for ksoftirqd
  - TSN for deterministic Ethernet
  - CAN for deterministic serial bus

Power management:
  - Wake-on-LAN (WoL): NIC wakes system on magic packet
  - Network interface power states (D0-D3)
  - PHY power down when cable unplugged
  - Energy Efficient Ethernet (802.3az EEE)

Boot-time networking:
  - Network console (netconsole): kernel logs over UDP
  - NFS root filesystem over network
  - PXE/TFTP boot sequence
  - DHCP for initial configuration
```

---

## 25.6 IoT Protocols

```
Constrained Application Protocol (CoAP):
  - REST-like protocol over UDP (port 5683)
  - Lightweight alternative to HTTP for IoT
  - Confirmable/Non-confirmable messages
  - Observe pattern for event notification

MQTT:
  - Publish/Subscribe over TCP
  - Broker-based (mosquitto, HiveMQ)
  - QoS levels: 0 (fire-and-forget), 1 (at-least-once), 2 (exactly-once)
  - Lightweight: 2-byte fixed header

6LoWPAN:
  - IPv6 over Low-Power Wireless Personal Area Networks
  - Header compression for IEEE 802.15.4
  - Mesh routing (RPL protocol)
  - Linux support: net/6lowpan/, net/ieee802154/

Thread / Matter:
  - Smart home networking standards
  - Thread: mesh networking over 802.15.4
  - Matter: application layer, runs over Thread, Wi-Fi, Ethernet
  - Linux: OpenThread border router
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/can/af_can.c | CAN socket family |
| net/can/raw.c | CAN_RAW protocol |
| net/can/bcm.c | CAN broadcast manager |
| net/can/isotp.c | ISO-TP transport protocol |
| net/can/j1939/ | SAE J1939 protocol |
| drivers/net/can/ | CAN controller drivers |
| net/sched/sch_taprio.c | TSN Time-Aware shaper |
| net/sched/sch_cbs.c | TSN Credit-Based shaper |
| drivers/ptp/ | PTP hardware clock drivers |
| net/6lowpan/ | 6LoWPAN protocol |

---

## Interview Questions

**Q1: What is SocketCAN and how does it differ from traditional CAN access?**
A: SocketCAN integrates CAN into the Linux networking stack using the standard socket API (AF_CAN). Unlike traditional character device approaches (like /dev/can0), SocketCAN allows: (1) Multiple applications reading the same bus simultaneously (multicast). (2) Standard networking tools (tcpdump, Wireshark). (3) Socket filtering (can_filter). (4) Higher-layer protocols as socket types (CAN_RAW, CAN_BCM, CAN_ISOTP, CAN_J1939). (5) Network namespace support for containers. Applications use socket(), bind(), read(), write() — same API as TCP/UDP.

**Q2: What is TSN and why is it important for automotive?**
A: TSN (Time-Sensitive Networking) is a set of IEEE 802.1 standards that add deterministic, low-latency capabilities to standard Ethernet. Key features: time synchronization (802.1AS, sub-microsecond), scheduled traffic (802.1Qbv, time gates), credit-based shaping (802.1Qav), and stream redundancy (802.1CB). For automotive: critical messages (brake, steering) need guaranteed delivery within microseconds — TSN provides this over standard Ethernet, enabling the transition from CAN to Ethernet for safety-critical functions while sharing the same network with infotainment traffic.

**Q3: How does SOME/IP enable service-oriented architecture in vehicles?**
A: SOME/IP provides RPC (request/response methods) and publish/subscribe (events) over standard UDP/TCP. Service Discovery (SD) uses multicast to offer and find services dynamically. Each service has a unique ID and version. This enables loose coupling — ECUs can be added/removed without rewiring the bus or updating routing tables. Unlike CAN's signal-oriented model (fixed IDs), SOME/IP's service model supports dynamic discovery, versioning, and complex data serialization, matching modern software architecture needs.

---

## Summary

- SocketCAN: Standard socket API for CAN bus, supports raw, BCM, ISO-TP, J1939
- Automotive Ethernet: 100BASE-T1 / 1000BASE-T1 over single twisted pair
- TSN: Deterministic Ethernet — time sync, scheduled traffic, bounded latency
- SOME/IP: Service-oriented middleware for automotive (RPC + pub/sub)
- Embedded considerations: memory, CPU, real-time, power management
- IoT protocols: CoAP, MQTT, 6LoWPAN, Thread/Matter for constrained devices

---

Next: [Chapter 26 — Networking in Other Operating Systems](Chapter_26_Other_OS_Networking.md)
