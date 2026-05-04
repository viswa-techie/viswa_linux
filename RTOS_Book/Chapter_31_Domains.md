# Chapter 31: RTOS in Different Domains

## Learning Goals
- Understand RTOS usage in automotive, aerospace, medical, and industrial domains
- Know domain-specific requirements, standards, and RTOS choices
- Learn IoT and robotics RTOS architectures
- Master domain-specific design patterns

---

## 1. Automotive Systems

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Automotive RTOS Architecture:                           │
  │                                                           │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │                  Vehicle ECU                     │    │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
  │  │  │ App SWC  │  │ App SWC  │  │ App SWC      │  │    │
  │  │  │(brake ctl)│ │(EPS steer)│ │(body control)│  │    │
  │  │  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │    │
  │  │       │              │               │           │    │
  │  │  ┌────▼──────────────▼───────────────▼───────┐  │    │
  │  │  │           AUTOSAR RTE                      │  │    │
  │  │  │       (Runtime Environment)                │  │    │
  │  │  └────────────────┬──────────────────────────┘  │    │
  │  │                   │                              │    │
  │  │  ┌────────────────▼──────────────────────────┐  │    │
  │  │  │  AUTOSAR OS (OSEK/VDX compliant)          │  │    │
  │  │  │  • Static configuration (OIL file)        │  │    │
  │  │  │  • No dynamic task creation               │  │    │
  │  │  │  • Priority ceiling protocol              │  │    │
  │  │  │  • Alarm/counter mechanism                │  │    │
  │  │  │  • Protection (timing + memory)           │  │    │
  │  │  └───────────────────────────────────────────┘  │    │
  │  │                                                  │    │
  │  │  RTOS Choices by Safety Level:                   │    │
  │  │  ASIL A-B: FreeRTOS (with qualification)        │    │
  │  │  ASIL C-D: SafeRTOS, QNX, INTEGRITY, ETAS RTA-OS│   │
  │  │                                                  │    │
  │  │  Typical timing requirements:                    │    │
  │  │  • ABS/ESC control loop: 5ms period             │    │
  │  │  • Engine management: 1ms period                │    │
  │  │  • Body control: 10-50ms period                 │    │
  │  │  • Infotainment: soft real-time (Linux/QNX)     │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

```c
/* AUTOSAR-style static task configuration */
/* OIL (OSEK Implementation Language) configuration: */
/*
 * TASK BrakeControlTask {
 *     PRIORITY = 10;
 *     SCHEDULE = FULL;         // Preemptive
 *     ACTIVATION = 1;          // Single activation
 *     AUTOSTART = TRUE;
 *     STACKSIZE = 512;
 *     EVENT = BrakeEvent;
 * };
 *
 * ALARM BrakeAlarm {
 *     COUNTER = SystemCounter;
 *     ACTION = ACTIVATETASK { TASK = BrakeControlTask; };
 *     AUTOSTART = TRUE {
 *         ALARMTIME = 0;
 *         CYCLETIME = 5;       // 5ms period
 *     };
 * };
 */

/* CAN-based brake control task */
TASK(BrakeControlTask) {
    SensorData_t wheel_speeds[4];
    BrakeCommand_t brake_cmd;

    /* Read wheel speed sensors */
    read_wheel_speeds(wheel_speeds);

    /* ABS algorithm: prevent wheel lockup */
    brake_cmd = abs_control_algorithm(wheel_speeds);

    /* Output brake pressure via CAN */
    can_send_brake_command(&brake_cmd);

    TerminateTask();  /* AUTOSAR: task terminates, re-activated by alarm */
}
```

---

## 2. Aerospace and Avionics

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Avionics RTOS — ARINC 653 Partitioning:                │
  │                                                           │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ Partition 1    │ Partition 2    │ Partition 3    │    │
  │  │ (Flight Nav)   │ (Autopilot)   │ (Display)      │    │
  │  │ DAL A          │ DAL A          │ DAL C          │    │
  │  │ Period: 20ms   │ Period: 10ms   │ Period: 50ms   │    │
  │  │                │                │                │    │
  │  │ Task1  Task2   │ Task1  Task2   │ Task1          │    │
  │  └───────┬────────┴───────┬────────┴───────┬────────┘    │
  │          │                │                │              │
  │  ┌───────▼────────────────▼────────────────▼────────┐    │
  │  │           ARINC 653 Partition Manager             │    │
  │  │  • Time partitioning: fixed schedule per partition│    │
  │  │  • Space partitioning: MMU/MPU isolation          │    │
  │  │  • Health monitoring: fault containment           │    │
  │  │  • Inter-partition communication: sampling/queue  │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  RTOS Choices:                                           │
  │  • VxWorks 653: DO-178C DAL A certified                 │
  │  • INTEGRITY-178B: DO-178C DAL A, EAL 6+               │
  │  • PikeOS: DO-178C DAL A, hypervisor                    │
  │  • LynxOS-178: DO-178C DAL A                            │
  │                                                           │
  │  Key requirements:                                       │
  │  • Partition cannot affect other partitions (spatial)    │
  │  • Fixed time window per partition (temporal)            │
  │  • Deterministic scheduling: cyclic executive            │
  │  • WCET analysis for all code paths                     │
  │  • MC/DC test coverage required for DAL A               │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Medical Devices

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Medical Device RTOS (IEC 62304):                       │
  │                                                           │
  │  Class A: No injury possible                             │
  │    → Any RTOS, basic testing                             │
  │                                                           │
  │  Class B: Non-serious injury possible                    │
  │    → Qualified RTOS, unit testing, code review           │
  │    → Example: infusion pump display                      │
  │                                                           │
  │  Class C: Death or serious injury possible               │
  │    → Certified RTOS (SafeRTOS, QNX, INTEGRITY)          │
  │    → Full requirements traceability                      │
  │    → Static analysis, MC/DC coverage                    │
  │    → Example: ventilator, defibrillator, infusion pump   │
  │                                                           │
  │  Example: Infusion Pump Architecture                     │
  │  ┌─────────────────────────────────────────────┐        │
  │  │  ┌──────────────┐   ┌──────────────┐       │        │
  │  │  │ Flow Control │   │ Safety Monitor│       │        │
  │  │  │ Task (10ms)  │   │ Task (5ms)   │       │        │
  │  │  │ • Motor PID  │   │ • Flow check │       │        │
  │  │  │ • Valve ctl  │   │ • Air detect │       │        │
  │  │  └──────┬───────┘   │ • Occlusion  │       │        │
  │  │         │            └──────┬───────┘       │        │
  │  │         │                   │                │        │
  │  │  ┌──────▼───────────────────▼───────┐       │        │
  │  │  │      SafeRTOS (SIL 3)            │       │        │
  │  │  │  Static allocation only          │       │        │
  │  │  │  Validated API with error codes  │       │        │
  │  │  └──────────────────────────────────┘       │        │
  │  └─────────────────────────────────────────────┘        │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Industrial Automation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PLC (Programmable Logic Controller) with RTOS:          │
  │                                                           │
  │  ┌───────────────────────────────────────────┐           │
  │  │  IEC 61131-3 Runtime                      │           │
  │  │  ┌──────────┐  ┌──────────┐  ┌────────┐  │           │
  │  │  │ Scan Task│  │ Motion  │  │ Comms  │  │           │
  │  │  │ (1-10ms) │  │ Task    │  │ Task   │  │           │
  │  │  │ Ladder   │  │ (250us) │  │ Modbus │  │           │
  │  │  │ logic    │  │ Servo   │  │ EtherCAT│ │           │
  │  │  └──────────┘  └─────────┘  └────────┘  │           │
  │  │                                           │           │
  │  │  ┌───────────────────────────────────┐   │           │
  │  │  │     VxWorks / QNX / Zephyr        │   │           │
  │  │  │  • Deterministic scan cycle       │   │           │
  │  │  │  • EtherCAT: 250us cycle time     │   │           │
  │  │  │  • Safety PLC: SIL 3 certified    │   │           │
  │  │  └───────────────────────────────────┘   │           │
  │  └───────────────────────────────────────────┘           │
  │                                                           │
  │  Industrial protocols:                                   │
  │  • EtherCAT: 250us cycle, distributed clocks            │
  │  • PROFINET IRT: 31.25us minimum cycle                  │
  │  • Modbus TCP: non-deterministic, ~10ms                  │
  │  • OPC UA: information model + pub/sub                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. IoT and Connected Devices

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  IoT Device Architecture:                                │
  │                                                           │
  │  ┌─────────────────────────────────────────────┐        │
  │  │  Application                                 │        │
  │  │  ├── Sensor reading (10-100ms)              │        │
  │  │  ├── Data processing (local ML inference)   │        │
  │  │  └── Cloud publish (MQTT/CoAP)              │        │
  │  ├──────────────────────────────────────────────┤        │
  │  │  Connectivity                                │        │
  │  │  ├── WiFi (lwIP + TLS)                      │        │
  │  │  ├── BLE (Zephyr BLE stack)                 │        │
  │  │  ├── LoRa/LoRaWAN                           │        │
  │  │  └── Cellular (LTE-M/NB-IoT)               │        │
  │  ├──────────────────────────────────────────────┤        │
  │  │  RTOS (FreeRTOS / Zephyr / ThreadX)         │        │
  │  │  ├── Low power: tickless idle               │        │
  │  │  ├── OTA updates: MCUboot                   │        │
  │  │  └── Security: TLS 1.3, secure boot         │        │
  │  ├──────────────────────────────────────────────┤        │
  │  │  Hardware: Cortex-M4 (256KB Flash, 64KB RAM)│        │
  │  └─────────────────────────────────────────────┘        │
  │                                                           │
  │  IoT RTOS Choices:                                       │
  │  • FreeRTOS + AWS IoT: Amazon's managed IoT platform    │
  │  • Zephyr: best for BLE + networking devices            │
  │  • Azure RTOS (ThreadX): Microsoft IoT integration      │
  │  • Mbed OS: ARM's IoT platform (EOL 2026)              │
  │                                                           │
  │  Key IoT challenges for RTOS:                            │
  │  • Battery life: years on coin cell (tickless idle)     │
  │  • OTA: secure firmware update in the field             │
  │  • Security: TLS on constrained MCU (RAM-limited)       │
  │  • Connectivity: multiple protocols + failover           │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Domain RTOS Selection Guide

```
  ┌────────────┬──────────────┬──────────────┬─────────────┐
  │ Domain     │ Primary RTOS │ Safety Std   │ Key Metric  │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ Automotive │ AUTOSAR OS,  │ ISO 26262    │ 1-5ms cycle │
  │ (ASIL D)   │ SafeRTOS,QNX│ ASIL A-D     │ CAN/CAN-FD  │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ Aerospace  │ VxWorks 653, │ DO-178C      │ 10-50ms     │
  │ (DAL A)    │ INTEGRITY    │ DAL A-E      │ ARINC 653   │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ Medical    │ SafeRTOS,    │ IEC 62304    │ 5-100ms     │
  │ (Class C)  │ QNX,INTEGRITY│ Class A-C    │ Reliability │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ Industrial │ VxWorks, QNX │ IEC 61508    │ 250us cycle │
  │ (SIL 3)    │ Zephyr       │ SIL 1-4      │ EtherCAT    │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ IoT        │ FreeRTOS,    │ (varies)     │ Battery life│
  │            │ Zephyr,ThreadX│              │ OTA, TLS    │
  ├────────────┼──────────────┼──────────────┼─────────────┤
  │ Robotics   │ FreeRTOS,    │ ISO 10218   │ 1ms loop    │
  │            │ QNX, Zephyr  │ (industrial) │ Sensor+Motor│
  └────────────┴──────────────┴──────────────┴─────────────┘
```

---

## Interview Questions

**Q1: How does RTOS usage differ between automotive and IoT applications?**
**A:** Automotive: (1) Safety-certified RTOS required (ASIL D needs SafeRTOS/QNX). (2) Static configuration — no dynamic task/memory creation (AUTOSAR OSEK model). (3) Hard real-time: 1-5ms control loops for ABS/ESC. (4) CAN bus communication with strict timing. (5) Lockstep CPU for fault detection. (6) Static allocation only at ASIL C/D. IoT: (1) No safety certification typically needed. (2) Dynamic behavior — tasks created/destroyed as needed. (3) Soft real-time acceptable for most IoT. (4) Focus on power efficiency (years on battery, tickless idle). (5) Connectivity stack dominates code (WiFi/BLE/cellular + TLS). (6) OTA firmware update is critical (devices deployed in field). (7) Resource-constrained: 256KB Flash typical vs 2MB+ automotive. Automotive prioritizes determinism and safety; IoT prioritizes power, connectivity, and updatability.

**Q2: What is ARINC 653 and how does it differ from standard RTOS scheduling?**
**A:** ARINC 653 is the avionics OS interface standard that defines temporal and spatial partitioning. Standard RTOS: priority-based preemptive scheduling — highest priority ready task always runs. ARINC 653: (1) Time partitioning: system has a fixed major frame (e.g., 100ms) divided into time windows. Each partition gets a guaranteed time slot (e.g., Partition 1: 0-20ms, Partition 2: 20-40ms). No partition can exceed its window. (2) Spatial partitioning: each partition has its own memory space enforced by MMU — a fault in one partition cannot corrupt another. (3) Within each partition: standard priority-based scheduling for tasks. (4) Inter-partition communication: via pre-configured sampling/queuing ports (not shared memory). This enables mixed-criticality: DAL A flight control and DAL C display run on the same CPU with guaranteed isolation. Standard RTOS cannot provide this level of fault containment.

---

## Summary

- Automotive: AUTOSAR OS (OSEK), static config, CAN bus, ASIL A-D certification
- Aerospace: ARINC 653 partitioning (temporal + spatial), DO-178C DAL A, VxWorks 653/INTEGRITY
- Medical: IEC 62304 Class C, SafeRTOS, no dynamic allocation, redundant safety monitoring
- Industrial: EtherCAT 250us cycles, SIL 3, PLC scan tasks, deterministic fieldbus protocols
- IoT: FreeRTOS/Zephyr, battery optimization, OTA, TLS, connectivity stack dominates
- RTOS selection: driven by safety standard, timing requirements, communication protocols

---

[Previous Chapter: Debug Tools ←](Chapter_30_Debug_Tools.md) | [Next Chapter: RTOS vs Linux →](Chapter_32_RTOS_vs_Linux.md)
