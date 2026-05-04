# Chapter 18: Networking in RTOS

## Learning Goals
- Understand TCP/IP stacks for embedded RTOS (lwIP, Zephyr net)
- Learn socket API usage in RTOS context
- Know network stack architecture and task integration
- Understand real-time networking constraints
- Compare networking across RTOS platforms

---

## 1. Embedded Network Stack Architecture

```
  lwIP (Lightweight IP) Stack — most common RTOS TCP/IP
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Application Layer                                       │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ Socket API (BSD-like) or Raw API or Netconn API │     │
  │  └──────────────────┬──────────────────────────────┘     │
  │                      │                                    │
  │  Transport Layer                                         │
  │  ┌──────────┬────────┴───┬────────────┐                  │
  │  │ TCP      │ UDP        │ ICMP       │                  │
  │  │ (stream) │ (datagram) │ (ping/err) │                  │
  │  └──────────┴────────────┴────────────┘                  │
  │                      │                                    │
  │  Network Layer                                           │
  │  ┌──────────┬────────┴───┬────────────┐                  │
  │  │ IPv4     │ IPv6       │ ARP/NDP    │                  │
  │  └──────────┴────────────┴────────────┘                  │
  │                      │                                    │
  │  Network Interface                                       │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Ethernet MAC driver (DMA-driven)              │       │
  │  │ Receive: DMA → pbuf → tcpip_input()           │       │
  │  │ Transmit: pbuf → DMA → wire                   │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  lwIP footprint: ~40KB Flash, ~20KB RAM (typical config) │
  │  Runs on: FreeRTOS, Zephyr, bare-metal, ThreadX          │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. lwIP with FreeRTOS

```c
/* lwIP + FreeRTOS integration: tcpip_thread handles stack processing */

/* Initialize lwIP with FreeRTOS */
void network_init(void) {
    /* lwIP creates its own tcpip_thread internally */
    tcpip_init(NULL, NULL);

    /* Configure network interface */
    struct netif netif;
    ip4_addr_t ipaddr, netmask, gw;
    IP4_ADDR(&ipaddr,  192,168,1,100);
    IP4_ADDR(&netmask, 255,255,255,0);
    IP4_ADDR(&gw,      192,168,1,1);

    netif_add(&netif, &ipaddr, &netmask, &gw, NULL,
              ethernetif_init,   /* Driver init function */
              tcpip_input);      /* Input function (to tcpip_thread) */
    netif_set_default(&netif);
    netif_set_up(&netif);

    /* Or use DHCP */
    dhcp_start(&netif);
}

/* TCP server using Socket API (thread-safe, blocking) */
void vTcpServerTask(void *pvParameters) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(8080),
        .sin_addr.s_addr = INADDR_ANY,
    };

    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 5);

    for (;;) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        int client_fd = accept(server_fd,
            (struct sockaddr *)&client_addr, &client_len);

        if (client_fd >= 0) {
            /* Spawn a task to handle this client */
            xTaskCreate(vClientHandler, "Client",
                512, (void *)(intptr_t)client_fd, 2, NULL);
        }
    }
}

void vClientHandler(void *pvParameters) {
    int fd = (int)(intptr_t)pvParameters;
    char buf[256];

    for (;;) {
        int n = recv(fd, buf, sizeof(buf), 0);
        if (n <= 0) break;  /* Connection closed or error */

        /* Process and respond */
        send(fd, buf, n, 0);  /* Echo back */
    }
    close(fd);
    vTaskDelete(NULL);  /* Delete self */
}

/*
 * lwIP API choices:
 *
 * Raw API:    Callback-based, no threading, lowest overhead
 *             Used in tcpip_thread context only
 *             Best for: protocol implementations within lwIP
 *
 * Netconn API: Sequential, blocking, uses mailboxes
 *              Requires tcpip_thread + application threads
 *              Medium overhead
 *
 * Socket API:  BSD-like, fully blocking, thread-safe
 *              Built on top of Netconn
 *              Most portable, highest overhead
 *              Best for: application code, portability
 */
```

---

## 3. Ethernet MAC Driver Integration

```c
/*
 * Ethernet driver: DMA receives frames → lwIP → application
 *
 * ┌─────────┐    DMA    ┌──────────┐  tcpip_input  ┌──────┐
 * │ Network │──────────►│ RX Desc  │──────────────►│ lwIP │
 * │ Wire    │           │ Ring     │                │ stack│
 * │         │◄──────────│ TX Desc  │◄──────────────│      │
 * └─────────┘    DMA    │ Ring     │  low_level_out └──────┘
 *                        └──────────┘
 */

/* Receive task: waits for Ethernet DMA interrupt */
void vEthReceiveTask(void *pvParameters) {
    for (;;) {
        /* Block until Ethernet DMA signals frame received */
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        while (eth_rx_frame_available()) {
            /* Allocate lwIP packet buffer */
            struct pbuf *p = low_level_input(&netif);
            if (p != NULL) {
                /* Pass to lwIP stack (thread-safe) */
                if (netif.input(p, &netif) != ERR_OK) {
                    pbuf_free(p);
                }
            }
        }
    }
}

void ETH_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    if (ETH->DMASR & ETH_DMASR_RS) {
        ETH->DMASR = ETH_DMASR_RS;  /* Clear RX interrupt */
        vTaskNotifyGiveFromISR(xEthRxTaskHandle, &xWoken);
    }
    portYIELD_FROM_ISR(xWoken);
}
```

---

## 4. Real-Time Networking Considerations

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  RT networking challenges:                               │
  │                                                           │
  │  1. Non-deterministic latency:                           │
  │     Standard TCP/IP has variable latency (retransmits,   │
  │     congestion, buffering). Not suitable for hard RT.    │
  │                                                           │
  │  2. Network stack runs as task:                          │
  │     tcpip_thread priority affects RT behavior.           │
  │     Too high: delays application tasks.                  │
  │     Too low: network responses delayed.                  │
  │                                                           │
  │  3. Solutions for RT networking:                         │
  │     ┌────────────────────────────────────────────┐      │
  │     │ TSN (Time Sensitive Networking):            │      │
  │     │   IEEE 802.1 standards for deterministic    │      │
  │     │   Ethernet. Time-aware shaper (Qbv),        │      │
  │     │   frame preemption (Qbu). Used in          │      │
  │     │   automotive and industrial Ethernet.      │      │
  │     │                                             │      │
  │     │ EtherCAT:                                   │      │
  │     │   Industrial Ethernet with hardware         │      │
  │     │   processing. Sub-microsecond cycle times.  │      │
  │     │                                             │      │
  │     │ PROFINET IRT:                               │      │
  │     │   Isochronous real-time Ethernet for        │      │
  │     │   industrial automation. Reserved time      │      │
  │     │   slots for deterministic frames.           │      │
  │     │                                             │      │
  │     │ CAN FD / Automotive Ethernet:               │      │
  │     │   Priority-based, bounded latency           │      │
  │     │   CAN: 1Mbps, SOME/IP over Ethernet        │      │
  │     └────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Networking Comparison Across RTOS

| Feature | FreeRTOS+lwIP | Zephyr | QNX | VxWorks |
|---------|---------------|--------|-----|---------|
| **Stack** | lwIP | Native (based on BSD) | io-pkt (BSD) | WIND NET (BSD) |
| **API** | Socket/Netconn/Raw | Socket (POSIX-like) | Full POSIX | Full POSIX |
| **IPv6** | Optional | Yes | Yes | Yes |
| **TLS** | mbedTLS / wolfSSL | mbedTLS | OpenSSL | wolfSSL |
| **Footprint** | ~40KB Flash | ~80KB Flash | Larger (full BSD) | Larger |
| **TSN** | No | Experimental | io-pkt TSN | Yes |
| **MQTT/CoAP** | Libraries | Built-in | Libraries | Libraries |

---

## Interview Questions

**Q1: How does lwIP integrate with FreeRTOS?**
**A:** lwIP creates a `tcpip_thread` (task) that processes all TCP/IP stack events. Application tasks communicate with this thread via mailbox/queue. Socket API calls (send/recv) block the calling task and send a message to tcpip_thread, which processes the request and signals completion. Ethernet RX ISR notifies a receive task, which reads DMA descriptors, wraps frames in pbuf structures, and passes them to `tcpip_input()`. The tcpip_thread processes received packets up the stack. TX path: application calls send() → socket layer creates pbuf → lwIP processes → low_level_output() feeds Ethernet DMA. Critical config: tcpip_thread priority, pbuf pool size, memp pool sizes, TCP window size.

**Q2: Why is standard TCP/IP not suitable for hard real-time networking?**
**A:** TCP/IP has multiple sources of non-deterministic latency: (1) TCP retransmissions — lost packets cause variable delays (RTO can be seconds). (2) Congestion control — TCP slows down under load (AIMD). (3) Buffer bloat — queuing delays at switches/routers. (4) ARP resolution — first packet to a new host delayed by ARP request/response. (5) DHCP — address acquisition takes variable time. (6) OS scheduling — network stack runs as a task, competes with other tasks. For hard RT communication: use CAN bus (priority-based, bounded latency), TSN Ethernet (time-aware scheduling), or industrial protocols (EtherCAT, PROFINET IRT). UDP is more deterministic than TCP (no retransmits) but still affected by queuing.

---

## Summary

- lwIP: most common embedded TCP/IP stack (~40KB Flash), three API levels
- FreeRTOS+lwIP: tcpip_thread processes stack events, socket API for application tasks
- Ethernet MAC: DMA descriptors for zero-copy RX/TX, ISR notifies receive task
- Standard TCP/IP not suitable for hard RT due to retransmissions and variable latency
- RT networking solutions: CAN/CAN FD, TSN, EtherCAT, PROFINET IRT
- TLS integration via mbedTLS or wolfSSL for secure communication

---

[Previous Chapter: I/O Management ←](Chapter_17_IO_Management.md) | [Next Chapter: File Systems →](Chapter_19_File_Systems.md)
