# Chapter 20: Real-Time Communication Protocols

## Learning Goals
- Master CAN bus protocol and RTOS integration
- Understand automotive Ethernet and SOME/IP
- Learn industrial protocols (Modbus, PROFINET)
- Know protocol selection criteria for embedded systems

---

## 1. CAN Bus Protocol

```
  CAN (Controller Area Network) — Automotive Standard
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CAN 2.0B Frame Format:                                  │
  │  ┌───┬─────────┬───┬───┬──────────┬────┬───┬───┬───┐   │
  │  │SOF│   ID    │RTR│IDE│   DLC    │DATA│CRC│ACK│EOF│   │
  │  │ 1 │  11/29  │ 1 │ 1 │   4     │0-64│15 │ 2 │ 7 │   │
  │  └───┴─────────┴───┴───┴──────────┴────┴───┴───┴───┘   │
  │                                                           │
  │  Key properties:                                         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Multi-master bus (no central controller)     │      │
  │  │ · Priority-based arbitration (lower ID = higher)│     │
  │  │ · Deterministic: known worst-case latency      │      │
  │  │ · Error detection: CRC, bit stuffing, ACK      │      │
  │  │ · Speeds: CAN 2.0 (1Mbps), CAN FD (8Mbps data)│      │
  │  │ · Max payload: 8 bytes (CAN), 64 bytes (CAN FD)│      │
  │  │ · Bus length: up to 500m at 125kbps            │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Arbitration (non-destructive):                          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Node A sends ID 0x100: dominant bits win        │      │
  │  │ Node B sends ID 0x200: loses arbitration        │      │
  │  │ Node B backs off, retries next bus-free         │      │
  │  │                                                  │      │
  │  │ Lower ID = higher priority = guaranteed first   │      │
  │  │ This is why CAN is deterministic!               │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

```c
/* CAN driver with FreeRTOS */

typedef struct {
    uint32_t id;
    uint8_t  dlc;
    uint8_t  data[8];
} CanMessage_t;

static QueueHandle_t xCanRxQueue;
static QueueHandle_t xCanTxQueue;

/* CAN RX ISR */
void CAN1_RX0_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    CanMessage_t msg;

    msg.id  = (CAN1->sFIFOMailBox[0].RIR >> 21) & 0x7FF;
    msg.dlc = CAN1->sFIFOMailBox[0].RDTR & 0x0F;
    msg.data[0] = CAN1->sFIFOMailBox[0].RDLR & 0xFF;
    msg.data[1] = (CAN1->sFIFOMailBox[0].RDLR >> 8) & 0xFF;
    msg.data[2] = (CAN1->sFIFOMailBox[0].RDLR >> 16) & 0xFF;
    msg.data[3] = (CAN1->sFIFOMailBox[0].RDLR >> 24) & 0xFF;
    msg.data[4] = CAN1->sFIFOMailBox[0].RDHR & 0xFF;
    msg.data[5] = (CAN1->sFIFOMailBox[0].RDHR >> 8) & 0xFF;
    msg.data[6] = (CAN1->sFIFOMailBox[0].RDHR >> 16) & 0xFF;
    msg.data[7] = (CAN1->sFIFOMailBox[0].RDHR >> 24) & 0xFF;

    CAN1->RF0R |= CAN_RF0R_RFOM0;  /* Release FIFO */

    xQueueSendFromISR(xCanRxQueue, &msg, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* CAN message processing task */
void vCanRxTask(void *pvParameters) {
    CanMessage_t msg;
    for (;;) {
        if (xQueueReceive(xCanRxQueue, &msg, portMAX_DELAY) == pdPASS) {
            switch (msg.id) {
                case 0x100: handle_engine_rpm(&msg); break;
                case 0x200: handle_wheel_speed(&msg); break;
                case 0x300: handle_battery_voltage(&msg); break;
                default: break;
            }
        }
    }
}
```

---

## 2. Automotive Ethernet and SOME/IP

```
  Automotive Ethernet Stack
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Application (e.g., Camera stream, Diagnostics)│       │
  │  ├──────────────────────────────────────────────┤       │
  │  │ SOME/IP (Service-Oriented Middleware)         │       │
  │  │ - Service discovery (SD)                      │       │
  │  │ - Request/Response and Publish/Subscribe      │       │
  │  │ - Serialization over TCP/UDP                  │       │
  │  ├──────────────────────────────────────────────┤       │
  │  │ TCP / UDP                                     │       │
  │  ├──────────────────────────────────────────────┤       │
  │  │ IPv4 / IPv6                                   │       │
  │  ├──────────────────────────────────────────────┤       │
  │  │ 100BASE-T1 / 1000BASE-T1                     │       │
  │  │ (single twisted pair — automotive grade)      │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  SOME/IP vs CAN:                                         │
  │  ┌───────────┬──────────────┬──────────────────┐        │
  │  │           │ CAN          │ SOME/IP (Eth)    │        │
  │  ├───────────┼──────────────┼──────────────────┤        │
  │  │ Bandwidth │ 1Mbps        │ 100Mbps-1Gbps    │        │
  │  │ Payload   │ 8/64 bytes   │ Unlimited (TCP)  │        │
  │  │ Latency   │ Deterministic│ Best-effort*      │        │
  │  │ Topology  │ Bus          │ Switch/star       │        │
  │  │ Discovery │ Static       │ Dynamic (SD)     │        │
  │  │ Use       │ Control      │ Camera, ADAS     │        │
  │  └───────────┴──────────────┴──────────────────┘        │
  │  * With TSN, automotive Ethernet becomes deterministic   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Protocol Selection Guide

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Application           Recommended Protocol              │
  │  ───────────────────── ────────────────────────────────  │
  │  Safety-critical ctrl  CAN (deterministic, robust)       │
  │  High-bandwidth sensor Automotive Ethernet + SOME/IP     │
  │  Industrial automation Modbus RTU/TCP, PROFINET          │
  │  IoT cloud connect     MQTT over TCP/TLS                 │
  │  Sensor to gateway     BLE, Zigbee, LoRa                 │
  │  Inter-MCU (board)     SPI, UART, shared memory          │
  │  Debug/diagnostics     UDS over CAN (ISO 14229)          │
  │  Audio/video stream    AVB/TSN over Ethernet              │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why is CAN bus deterministic while Ethernet is not?**
**A:** CAN uses non-destructive bitwise arbitration: when multiple nodes transmit simultaneously, the one with the lowest ID (dominant bits) wins, and losers detect this and back off without corrupting the winning frame. Worst-case latency for the highest-priority message is bounded — it wins arbitration within one bit time. Ethernet uses CSMA/CD (or switches now): collisions cause backoff with random delays, making latency unpredictable. Even with switches (no collisions), Ethernet has store-and-forward latency, TCP retransmissions, and buffering delays. TSN (Time Sensitive Networking) adds time-aware scheduling to Ethernet for deterministic behavior, but requires specialized hardware and configuration.

**Q2: Compare CAN and CAN FD. When would you use CAN FD?**
**A:** CAN FD (Flexible Data-rate) extends CAN 2.0: (1) Data phase can run at higher bit rate (2-8Mbps vs 1Mbps arbitration rate). (2) Payload up to 64 bytes (vs 8 bytes CAN). (3) Same arbitration mechanism and backward compatible on the bus. Use CAN FD when: need more data per message (reduce bus load by sending fewer frames), need faster throughput (firmware updates over CAN), modern ECU development. Keep CAN 2.0 when: legacy compatibility required, simple nodes with limited controller support, cost-sensitive applications. CAN FD is standard in new automotive platforms (since ~2017).

---

## Summary

- CAN: deterministic, priority-based arbitration, 1Mbps, 8-byte payload — automotive standard
- CAN FD: up to 8Mbps data rate, 64-byte payload — next-gen automotive
- Automotive Ethernet: 100Mbps-1Gbps, SOME/IP for service-oriented communication
- TSN makes Ethernet deterministic with time-aware scheduling
- Protocol selection: CAN for control, Ethernet for bandwidth, MQTT for IoT

---

[Previous Chapter: File Systems ←](Chapter_19_File_Systems.md) | [Next Chapter: Power Management →](Chapter_21_Power_Management.md)
