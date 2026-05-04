# Chapter 3: Hardware Architecture for Power Management

## Learning Goals
- Understand the hardware components that enable power management
- Learn CPU power architecture including voltage domains and clock distribution
- Understand PMIC architecture and voltage rail management
- Know power domain design in modern SoCs
- Learn the role of thermal sensors and cooling hardware

---

## 1. CPU Power Architecture

### 1.1 Voltage and Frequency Domains

Modern CPUs partition into independent voltage/frequency domains for fine-grained PM:

```
  Modern SoC Power Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌──── CPU Cluster 0 ─────┐  ┌──── CPU Cluster 1 ─────┐│
  │  │  Voltage: V_cpu0        │  │  Voltage: V_cpu1        ││
  │  │  Frequency: F_cpu0      │  │  Frequency: F_cpu1      ││
  │  │  ┌─────┐  ┌─────┐     │  │  ┌─────┐  ┌─────┐     ││
  │  │  │Core0│  │Core1│     │  │  │Core4│  │Core5│     ││
  │  │  └─────┘  └─────┘     │  │  └─────┘  └─────┘     ││
  │  │  ┌─────┐  ┌─────┐     │  │  ┌─────┐  ┌─────┐     ││
  │  │  │Core2│  │Core3│     │  │  │Core6│  │Core7│     ││
  │  │  └─────┘  └─────┘     │  │  └─────┘  └─────┘     ││
  │  │  ┌─────────────────┐   │  │  ┌─────────────────┐   ││
  │  │  │  L2 Cache       │   │  │  │  L2 Cache       │   ││
  │  │  └─────────────────┘   │  │  └─────────────────┘   ││
  │  └────────────────────────┘  └────────────────────────┘│
  │                                                           │
  │  ┌──── GPU Domain ────────┐  ┌──── Memory Domain ──────┐│
  │  │  Voltage: V_gpu        │  │  Voltage: V_mem          ││
  │  │  Frequency: F_gpu      │  │  Frequency: F_mem        ││
  │  │  ┌──────────────────┐  │  │  ┌──────────────────┐   ││
  │  │  │   GPU Cores      │  │  │  │  DDR Controller  │   ││
  │  │  └──────────────────┘  │  │  │  + PHY           │   ││
  │  └────────────────────────┘  │  └──────────────────┘   ││
  │                               └────────────────────────┘│
  │                                                           │
  │  ┌──── Always-On Domain ──────────────────────────────┐  │
  │  │  RTC, Wake-up controller, Power management unit     │  │
  │  │  (Remains powered during deep sleep)                │  │
  │  └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

### 1.2 Per-Core vs Per-Cluster DVFS

```
  Per-Cluster DVFS (Common in ARM big.LITTLE)
  ┌──────────────────────────────────────────┐
  │  All cores in cluster share V/F          │
  │  Core0 ─┐                               │
  │  Core1 ─┼── Same voltage rail ── PLL    │
  │  Core2 ─┤                               │
  │  Core3 ─┘                               │
  │  Simpler HW, coarser granularity         │
  └──────────────────────────────────────────┘

  Per-Core DVFS (Intel, modern ARM)
  ┌──────────────────────────────────────────┐
  │  Each core has independent V/F           │
  │  Core0 ── VR0 ── PLL0                   │
  │  Core1 ── VR1 ── PLL1                   │
  │  Core2 ── VR2 ── PLL2                   │
  │  Core3 ── VR3 ── PLL3                   │
  │  Better efficiency, complex HW           │
  └──────────────────────────────────────────┘
```

---

## 2. Clock Distribution Architecture

### 2.1 Clock Tree

```
  Crystal Oscillator (e.g., 24 MHz)
  │
  ├──► PLL0 (CPU PLL)
  │    ├── ÷N0 ──► CPU Cluster 0 clock
  │    └── ÷N1 ──► CPU Cluster 1 clock
  │
  ├──► PLL1 (System PLL)
  │    ├── ÷N2 ──► AXI bus clock
  │    ├── ÷N3 ──► AHB bus clock
  │    └── ÷N4 ──► APB bus clock
  │
  ├──► PLL2 (GPU PLL)
  │    └── ÷N5 ──► GPU clock
  │
  ├──► PLL3 (DDR PLL)
  │    └── ÷N6 ──► DDR controller clock
  │
  └──► PLL4 (Audio/Video)
       ├── ÷N7 ──► I2S clock
       └── ÷N8 ──► Display clock

  Each PLL:
  ┌─────────────────────────────────────────┐
  │  Reference ──► Phase    ──► VCO ──► ÷N │
  │   Clock        Detector      │         │
  │                  ▲           │         │
  │                  │           │         │
  │                  └── ÷M ◄───┘         │
  │                                        │
  │  F_out = F_ref × (N / M)              │
  │  PLL can be powered down when unused   │
  └─────────────────────────────────────────┘
```

### 2.2 Clock Gating Hardware

```
  Clock Gating Cell
  ┌───────────────────────────────────────┐
  │                                       │
  │  Clock_in ──►┌──┐                    │
  │              │AND├──► Clock_out       │
  │  Enable ────►└──┘                    │
  │                                       │
  │  When Enable=0: Clock_out = 0 (gated) │
  │  When Enable=1: Clock_out = Clock_in  │
  │                                       │
  │  Integrated Clock Gating (ICG) cell   │
  │  includes latch for glitch-free gating│
  └───────────────────────────────────────┘

  Hierarchical Clock Gating:
  ┌─────────────────────────────────────────────────┐
  │  Module_clock ──► Submodule_A_gate ──► Sub_A    │
  │                ├► Submodule_B_gate ──► Sub_B    │
  │                └► Submodule_C_gate ──► Sub_C    │
  │                                                  │
  │  If all sub-gates off → parent gate turns off    │
  └─────────────────────────────────────────────────┘
```

---

## 3. PMIC Architecture

### 3.1 Power Management IC

```
  PMIC (Power Management Integrated Circuit)
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  Battery/DC ──► ┌────────────────────────────────┐   │
  │                 │  Buck Converters (step-down)    │   │
  │                 │  ┌─────┐  ┌─────┐  ┌─────┐    │   │
  │                 │  │BUCK1│  │BUCK2│  │BUCK3│    │   │
  │                 │  │1.0V │  │1.8V │  │3.3V │    │   │
  │                 │  │CPU  │  │I/O  │  │DDR  │    │   │
  │                 │  └─────┘  └─────┘  └─────┘    │   │
  │                 └────────────────────────────────┘   │
  │                                                       │
  │                 ┌────────────────────────────────┐   │
  │                 │  LDO Regulators (low noise)    │   │
  │                 │  ┌─────┐  ┌─────┐  ┌─────┐    │   │
  │                 │  │LDO1 │  │LDO2 │  │LDO3 │    │   │
  │                 │  │1.2V │  │2.8V │  │1.8V │    │   │
  │                 │  │PLL  │  │Codec│  │Sensor│   │   │
  │                 │  └─────┘  └─────┘  └─────┘    │   │
  │                 └────────────────────────────────┘   │
  │                                                       │
  │  Control Interface: I2C / SPI to SoC                 │
  │  GPIO: Power sequencing, enable/disable rails        │
  │  Interrupts: Over-current, thermal, undervoltage     │
  │                                                       │
  │  ┌────────────────────────────┐                      │
  │  │  Additional Functions:     │                      │
  │  │  - RTC (Real Time Clock)   │                      │
  │  │  - Battery charger         │                      │
  │  │  - Watchdog timer          │                      │
  │  │  - GPIO expander           │                      │
  │  │  - ADC (voltage monitor)   │                      │
  │  └────────────────────────────┘                      │
  └──────────────────────────────────────────────────────┘
```

### 3.2 Voltage Regulator Types

```
  ┌──────────────┬──────────────────┬──────────────────────┐
  │ Type         │ Efficiency       │ Use Case              │
  ├──────────────┼──────────────────┼──────────────────────┤
  │ Buck (step-  │ 85-95%           │ CPU, DDR, main rails  │
  │   down)      │ Higher current   │ Variable voltage      │
  ├──────────────┼──────────────────┼──────────────────────┤
  │ Boost (step- │ 80-90%           │ LED backlights,       │
  │   up)        │                  │ USB VBUS              │
  ├──────────────┼──────────────────┼──────────────────────┤
  │ LDO (linear) │ V_out/V_in       │ Low-noise: PLL,      │
  │              │ (can be <50%)    │ ADC, RF, audio        │
  ├──────────────┼──────────────────┼──────────────────────┤
  │ Charge Pump  │ ~90% at 2x      │ Small current, flash  │
  │              │                  │ LED, display bias     │
  └──────────────┴──────────────────┴──────────────────────┘
```

### 3.3 PMIC Communication

```c
/* PMIC register access via I2C (simplified example) */
struct pmic_regulator {
    struct i2c_client *client;
    u8 voltage_reg;    /* Register for voltage setting */
    u8 enable_reg;     /* Register for enable/disable */
    u8 enable_bit;
};

static int pmic_set_voltage(struct pmic_regulator *reg, int uV)
{
    u8 selector = (uV - 600000) / 12500;  /* 600mV base, 12.5mV steps */
    return i2c_smbus_write_byte_data(reg->client,
                                     reg->voltage_reg,
                                     selector);
}

static int pmic_enable(struct pmic_regulator *reg)
{
    u8 val = i2c_smbus_read_byte_data(reg->client, reg->enable_reg);
    val |= BIT(reg->enable_bit);
    return i2c_smbus_write_byte_data(reg->client, reg->enable_reg, val);
}
```

---

## 4. Power Domain Hardware

### 4.1 Power Switches

```
  Power Gating with Header Switch
  ┌──────────────────────────────────────────────┐
  │                                               │
  │  VDD_main ──┐                                │
  │             │                                │
  │        ┌────▼────┐                           │
  │        │ PMOS    │ ◄── Power_gate_ctrl       │
  │        │ Header  │                           │
  │        │ Switch  │                           │
  │        └────┬────┘                           │
  │             │                                │
  │        VDD_local (Virtual VDD)               │
  │             │                                │
  │  ┌──────────▼──────────────────────────┐     │
  │  │      Logic Block (Power Domain)      │     │
  │  │                                      │     │
  │  │  ┌──────────┐  ┌────────────────┐   │     │
  │  │  │ Combinat- │  │  Sequential   │   │     │
  │  │  │  ional    │  │  (flip-flops) │   │     │
  │  │  │  Logic    │  │               │   │     │
  │  │  └──────────┘  └────────────────┘   │     │
  │  │                                      │     │
  │  │  ┌──────────────────────────────┐   │     │
  │  │  │ Retention registers          │   │     │
  │  │  │ (powered by always-on rail)  │   │     │
  │  │  └──────────────────────────────┘   │     │
  │  └──────────────────────────────────────┘     │
  │                                               │
  │  Isolation cells at domain boundary           │
  │  prevent floating outputs from                │
  │  corrupting active domains                    │
  └──────────────────────────────────────────────┘
```

### 4.2 Power State Transitions

```
  Power Domain States:
  ┌────────┐   power_off()   ┌────────┐
  │   ON   │ ──────────────► │  OFF   │
  │        │                 │        │
  │ VDD=1  │ ◄────────────── │ VDD=0  │
  │ CLK=1  │   power_on()   │ CLK=0  │
  │ ISO=0  │                 │ ISO=1  │
  └────┬───┘                 └────────┘
       │
       │  save_state()
       ▼
  ┌─────────┐
  │RETENTION│  VDD=0 for logic
  │         │  Retention regs powered
  │ Context │  Lower power than ON
  │ Saved   │  Faster resume than OFF
  └─────────┘

  Power-up sequence:
  1. Assert isolation (ISO=1)
  2. Power on (VDD=1, wait for ramp)
  3. Release clamp
  4. De-assert reset
  5. Restore state (if retention)
  6. Release isolation (ISO=0)
  7. Enable clocks
```

### 4.3 SoC Power Domain Example

```
  Typical Mobile SoC Power Domains
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  ┌─ Always-On ────────────────────────────────┐  │
  │  │  PMU, RTC, Wakeup controller, SRAM         │  │
  │  │  (Never turned off except at G3)            │  │
  │  └────────────────────────────────────────────┘  │
  │                                                   │
  │  ┌─ CPU Domain ───┐  ┌─ GPU Domain ───────────┐ │
  │  │ ┌────┐ ┌────┐  │  │  Shader cores          │ │
  │  │ │CPU0│ │CPU1│  │  │  Texture units          │ │
  │  │ └────┘ └────┘  │  │  Frame buffer           │ │
  │  │ ┌────┐ ┌────┐  │  │                         │ │
  │  │ │CPU2│ │CPU3│  │  └─────────────────────────┘ │
  │  │ └────┘ └────┘  │                               │
  │  │  L2 cache      │  ┌─ Display Domain ────────┐ │
  │  └────────────────┘  │  Display controller      │ │
  │                       │  DSI/HDMI PHY            │ │
  │  ┌─ Modem Domain ──┐ └─────────────────────────┘ │
  │  │  Baseband        │                              │
  │  │  RF frontend     │  ┌─ Camera/ISP Domain ───┐ │
  │  └─────────────────┘  │  ISP, CSI PHY          │ │
  │                        └────────────────────────┘ │
  │  ┌─ Audio Domain ──┐  ┌─ Peripheral Domain ───┐ │
  │  │  DSP, Codec      │  │  UART, SPI, I2C, USB  │ │
  │  └─────────────────┘  └────────────────────────┘ │
  └──────────────────────────────────────────────────┘
```

---

## 5. CPU Power State Hardware

### 5.1 C-State Hardware Implementation

```
  C-State Entry/Exit Hardware
  ┌────────────────────────────────────────────────────┐
  │  C0 (Active):                                      │
  │    All clocks running, VDD at operating voltage     │
  │                                                     │
  │  C1 (Halt):                                        │
  │    Core clock gated (AND gate)                     │
  │    VDD unchanged, resume: ~1μs                     │
  │    HW: clock gating cell engaged                   │
  │                                                     │
  │  C3 (Sleep):                                       │
  │    Core clock gated                                │
  │    L1 cache flushed or snooped                     │
  │    VDD may be reduced                              │
  │    Resume: ~100μs (cache refill)                   │
  │                                                     │
  │  C6 (Deep Power Down):                             │
  │    Core VDD = 0 (power gated)                      │
  │    State saved to retention SRAM                   │
  │    PLL may be off                                  │
  │    Resume: ~200-500μs (power up + restore)         │
  │                                                     │
  │  C7 (Package Deep):                                │
  │    All cores in C6                                 │
  │    Shared L3 flushed to memory                     │
  │    Package PLL off                                 │
  │    Resume: ~1-5ms                                  │
  └────────────────────────────────────────────────────┘
```

### 5.2 DVFS Hardware (Voltage Regulator + PLL)

```
  DVFS Transition: Frequency Increase
  ┌────────────────────────────────────────────────────┐
  │                                                     │
  │  1. Request new OPP (higher V, higher F)           │
  │     │                                              │
  │  2. Increase voltage FIRST (via PMIC)              │
  │     │  VDD: 0.8V ──► 1.0V                         │
  │     │  Wait for voltage to stabilize (~50-200μs)   │
  │     │                                              │
  │  3. Then increase frequency (via PLL/divider)      │
  │     │  F: 1.0 GHz ──► 1.5 GHz                     │
  │     │  PLL lock time: ~10-50μs                     │
  │     │                                              │
  │  4. Done.                                          │
  │                                                     │
  │  DVFS Transition: Frequency Decrease               │
  │  1. Decrease frequency FIRST                       │
  │  2. Then decrease voltage                          │
  │     (Reverse order — voltage must always be        │
  │      sufficient for current frequency)             │
  └────────────────────────────────────────────────────┘
```

---

## 6. Thermal Sensor Hardware

### 6.1 On-Die Thermal Sensors

```
  Thermal Monitoring Architecture
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  ┌─ CPU Die ──────────────────────────────────┐  │
  │  │                                             │  │
  │  │  ┌────┐ [T1]  ┌────┐ [T2]                 │  │
  │  │  │Core│       │Core│                       │  │
  │  │  │ 0  │       │ 1  │                       │  │
  │  │  └────┘       └────┘                       │  │
  │  │                          [T1-T4] = Thermal  │  │
  │  │  ┌────┐ [T3]  ┌────┐ [T4]  sensors        │  │
  │  │  │Core│       │Core│       (diodes or      │  │
  │  │  │ 2  │       │ 3  │       BJT junctions)  │  │
  │  │  └────┘       └────┘                       │  │
  │  │                                             │  │
  │  │  ┌─────────────────────────────────────┐   │  │
  │  │  │  Thermal Management Unit (TMU)      │   │  │
  │  │  │  - ADC reads sensor diodes          │   │  │
  │  │  │  - Programmable trip points          │   │  │
  │  │  │  - Interrupt on threshold crossing   │   │  │
  │  │  │  - Hardware throttle (emergency)     │   │  │
  │  │  └─────────────────────────────────────┘   │  │
  │  └─────────────────────────────────────────────┘  │
  │                                                   │
  │  Thermal trip points:                             │
  │  ┌──────────────────────────────────────────┐    │
  │  │  T_passive:  85°C → software throttling  │    │
  │  │  T_critical: 100°C → hardware throttle   │    │
  │  │  T_shutdown: 110°C → emergency shutdown  │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
```

### 6.2 Cooling Hardware

```
  Cooling Methods:
  ┌─────────────────────────────────────────────────┐
  │                                                  │
  │  Passive Cooling:                                │
  │  ├── Heat sink (conduction → convection)         │
  │  ├── Heat pipe                                   │
  │  ├── Thermal pad/paste (gap filler)              │
  │  └── Metal chassis as heat spreader              │
  │                                                  │
  │  Active Cooling:                                 │
  │  ├── Fan (PWM-controlled speed)                  │
  │  ├── Peltier/TEC cooler                          │
  │  └── Liquid cooling loop                         │
  │                                                  │
  │  Software Cooling:                               │
  │  ├── CPU throttling (reduce max frequency)       │
  │  ├── Task migration (away from hot cores)        │
  │  ├── GPU throttling                              │
  │  └── Network bandwidth limiting                  │
  └─────────────────────────────────────────────────┘
```

---

## 7. Power Monitoring Hardware

### 7.1 Current Sensing

```
  Current Monitoring Circuit
  ┌──────────────────────────────────────────┐
  │                                           │
  │  VDD ──┬── R_sense ──┬── Load            │
  │        │  (1-10 mΩ)  │                   │
  │        │             │                   │
  │        └──► ADC+ ────┘                   │
  │              │                           │
  │              ▼                            │
  │        V_sense = I_load × R_sense        │
  │        P_load = V_load × I_load          │
  │                                           │
  │  Common ICs:                              │
  │  - INA219/INA226 (I2C power monitor)     │
  │  - INA3221 (3-channel)                   │
  │  - TI LM3 series                         │
  └──────────────────────────────────────────┘
```

### 7.2 Intel RAPL (Running Average Power Limit)

```
  RAPL Architecture
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  RAPL Domains:                                   │
  │  ├── Package (PKG): Entire processor package     │
  │  ├── Power Plane 0 (PP0): CPU cores              │
  │  ├── Power Plane 1 (PP1): GPU (desktop)          │
  │  ├── DRAM: Memory controller + DIMMs             │
  │  └── PSys: Entire SoC platform (mobile)          │
  │                                                   │
  │  MSR Registers per domain:                       │
  │  ┌────────────────────────────────────────┐      │
  │  │ MSR_PKG_POWER_LIMIT    — Set TDP limit │      │
  │  │ MSR_PKG_ENERGY_STATUS  — Read energy   │      │
  │  │ MSR_PKG_POWER_INFO     — TDP info      │      │
  │  │ MSR_PKG_PERF_STATUS    — Throttle time │      │
  │  └────────────────────────────────────────┘      │
  │                                                   │
  │  Energy unit: ~15.3 μJ (platform dependent)      │
  │  Read MSR, compute delta, divide by time = Watts │
  └──────────────────────────────────────────────────┘
```

```c
/* Reading Intel RAPL energy counter */
#include <stdio.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>

#define MSR_PKG_ENERGY_STATUS   0x611
#define MSR_RAPL_POWER_UNIT     0x606

static uint64_t read_msr(int cpu, uint32_t reg)
{
    char path[64];
    uint64_t data;

    snprintf(path, sizeof(path), "/dev/cpu/%d/msr", cpu);
    int fd = open(path, O_RDONLY);
    pread(fd, &data, sizeof(data), reg);
    close(fd);
    return data;
}

int main(void)
{
    uint64_t power_unit = read_msr(0, MSR_RAPL_POWER_UNIT);
    double energy_unit = 1.0 / (1 << ((power_unit >> 8) & 0x1F));

    uint64_t e1 = read_msr(0, MSR_PKG_ENERGY_STATUS) & 0xFFFFFFFF;
    sleep(1);
    uint64_t e2 = read_msr(0, MSR_PKG_ENERGY_STATUS) & 0xFFFFFFFF;

    double joules = (e2 - e1) * energy_unit;
    printf("Package power: %.2f watts\n", joules);
    return 0;
}
```

---

## 8. ARM Power Management Hardware

### 8.1 big.LITTLE / DynamIQ

```
  ARM big.LITTLE Architecture
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  ┌── LITTLE Cluster ──┐  ┌── big Cluster ──────┐│
  │  │  (Efficiency cores) │  │  (Performance cores) ││
  │  │  ┌────┐  ┌────┐    │  │  ┌────┐  ┌────┐    ││
  │  │  │A55 │  │A55 │    │  │  │A78 │  │A78 │    ││
  │  │  │    │  │    │    │  │  │    │  │    │    ││
  │  │  └────┘  └────┘    │  │  └────┘  └────┘    ││
  │  │  Low power, in-order│  │  High perf, OoO     ││
  │  │  0.1W per core     │  │  1-2W per core       ││
  │  └────────────────────┘  └─────────────────────┘│
  │                                                   │
  │  Cache Coherent Interconnect (CCI/DSU)           │
  │  Both clusters share same memory view            │
  │                                                   │
  │  PM Strategy:                                    │
  │  - Light tasks → LITTLE cores                    │
  │  - Heavy tasks → big cores                       │
  │  - Idle → power gate unused cluster              │
  └──────────────────────────────────────────────────┘

  ARM DynamIQ (modern):
  ┌──────────────────────────────────────────────────┐
  │  Single cluster with mixed core types            │
  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐   │
  │  │A55 │ │A55 │ │A55 │ │A55 │ │A78 │ │X3  │   │
  │  │    │ │    │ │    │ │    │ │    │ │    │   │
  │  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘   │
  │  Per-core DVFS supported                        │
  │  Shared L3 cache                                │
  │  Faster task migration between core types       │
  └──────────────────────────────────────────────────┘
```

### 8.2 ARM Power Control

```c
/* ARM PSCI (Power State Coordination Interface) */
/* Used by Linux to control ARM CPU power states */

/* PSCI calls are made via SMC (Secure Monitor Call) */
#define PSCI_FN_CPU_ON          0xC4000003
#define PSCI_FN_CPU_OFF         0x84000002
#define PSCI_FN_CPU_SUSPEND     0xC4000001
#define PSCI_FN_SYSTEM_OFF      0x84000008
#define PSCI_FN_SYSTEM_RESET    0x84000009

/* Linux calls PSCI to put CPU into idle */
static int psci_cpu_suspend(u32 state, unsigned long entry_point)
{
    return invoke_psci_fn(PSCI_FN_CPU_SUSPEND, state,
                          entry_point, 0);
    /* This results in an SMC instruction:
     * CPU traps to ATF (EL3) → ATF manages power HW
     * → core voltage/clock turned off
     * On wakeup: ATF restores state, returns to Linux */
}
```

---

## 9. Power Sequencing

### 9.1 Power-Up Sequence

```
  System Power-Up Sequence (typical SoC)
  ┌────────────────────────────────────────────────────┐
  │                                                     │
  │  Time ────────────────────────────────────────►    │
  │                                                     │
  │  1. PMIC POR (Power-On Reset)                      │
  │     └── Internal regulators start                  │
  │                                                     │
  │  2. Always-On domain powered                       │
  │     └── VDD_AON = 0.8V (RTC, PMU, wakeup ctrl)    │
  │                                                     │
  │  3. Memory domain powered                          │
  │     └── VDD_DDR = 1.1V (DDR controller + PHY)     │
  │                                                     │
  │  4. I/O domain powered                             │
  │     └── VDD_IO = 1.8V / 3.3V                      │
  │                                                     │
  │  5. CPU domain powered                             │
  │     └── VDD_CPU = 1.0V (boot core only)            │
  │                                                     │
  │  6. PLL lock + clock distribution                  │
  │                                                     │
  │  7. Reset de-asserted → CPU starts executing       │
  │     └── Boot ROM → Bootloader → Kernel             │
  │                                                     │
  │  8. Kernel powers on remaining domains as needed   │
  │     └── GPU, Camera, Display, etc.                 │
  │                                                     │
  │  IMPORTANT: Voltage sequencing must follow PMIC    │
  │  specification — wrong order can damage SoC        │
  └────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/regulator/` | Voltage regulator framework + PMIC drivers |
| `drivers/clk/` | Clock framework + SoC clock drivers |
| `drivers/base/power/domain.c` | Generic power domain implementation |
| `drivers/thermal/` | Thermal framework + zone drivers |
| `drivers/firmware/psci/psci.c` | ARM PSCI implementation |
| `drivers/firmware/arm_scmi/` | ARM SCMI protocol drivers |
| `drivers/cpufreq/` | CPU frequency scaling drivers |
| `arch/arm64/kernel/psci.c` | ARM64 PSCI CPU operations |
| `arch/x86/kernel/cpu/intel_rapl.c` | Intel RAPL driver (moved to powercap) |
| `drivers/powercap/intel_rapl_common.c` | Intel RAPL powercap driver |

---

## Interview Questions

**Q1: Explain the difference between a buck converter and an LDO regulator.**
**A:** A buck (switching) converter steps down voltage using an inductor, switching transistor, and feedback loop — achieving 85-95% efficiency. An LDO (Low Dropout) linear regulator drops voltage by dissipating excess as heat — efficiency is V_out/V_in (e.g., 60% for 1.8V from 3.3V). LDOs provide cleaner (less noisy) output and are used for sensitive analog circuits (PLL, ADC, RF). Buck converters are used for high-current rails where efficiency matters.

**Q2: Why must voltage increase before frequency during a DVFS transition?**
**A:** Transistors need sufficient voltage to switch at the target frequency — this is the timing margin. If frequency increases before voltage is adequate, setup/hold timing violations occur, causing signal corruption and system crashes. Conversely, when reducing frequency, you decrease frequency first (reducing voltage requirement) and then safely lower voltage. The rule is: voltage must always be sufficient for the current operating frequency.

**Q3: What is power gating and how does it differ from clock gating?**
**A:** Clock gating stops the clock signal to a block, eliminating dynamic power (αCV²f) but leakage current still flows through powered transistors. Power gating uses header or footer switches (PMOS/NMOS transistors) to cut VDD entirely, eliminating both dynamic and leakage power. Power gating requires isolation cells at domain boundaries, optional retention registers for state preservation, and a specific power-up sequence (assert isolation → power on → release clamp → de-assert reset → release isolation).

**Q4: Explain ARM big.LITTLE and its PM implications.**
**A:** big.LITTLE pairs high-performance out-of-order cores (big) with energy-efficient in-order cores (LITTLE). The LITTLE cores consume ~10x less power at ~40-60% performance. The PM strategy is: route light tasks (UI, background) to LITTLE cores and heavy tasks (gaming, compilation) to big cores. Idle cores/clusters can be power-gated. The scheduler (EAS) considers energy cost when placing tasks. DynamIQ improves on this with per-core DVFS and mixed cores in a single cluster.

**Q5: What is Intel RAPL and how is it used?**
**A:** RAPL (Running Average Power Limit) provides power monitoring and capping via MSR registers. It defines domains (Package, PP0/CPU cores, PP1/GPU, DRAM, PSys). The energy counter MSR accumulates energy in ~15.3μJ units — reading it periodically gives power consumption. Power limits can be set (PL1 for sustained TDP, PL2 for short-term boost). Linux exposes RAPL through the powercap framework at `/sys/class/powercap/intel-rapl/`.

---

## Summary

- Modern SoCs partition into independent voltage/frequency domains for fine-grained PM
- Clock tree uses PLLs and dividers; clock gating stops unused clocks to save dynamic power
- PMICs contain buck converters (efficient, high-current) and LDOs (low-noise, lower efficiency)
- Power gating HW uses header switches, isolation cells, and retention registers
- CPU C-states progress from clock gating (C1) to full power gating (C6/C7)
- DVFS HW requires voltage-before-frequency sequencing for safe transitions
- Thermal sensors (on-die diodes) feed TMU with programmable trip points
- ARM uses PSCI/SCMI for power control; Intel uses MSR-based interfaces (RAPL, HWP)
- Power sequencing order is critical — violations can damage hardware

---

[Previous Chapter: History and Evolution ←](Chapter_02_History_Evolution.md) | [Next Chapter: Power States →](Chapter_04_Power_States.md)
