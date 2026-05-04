# Chapter 27: RTOS Build Systems

## Learning Goals
- Understand cross-compilation toolchains for embedded RTOS
- Learn CMake for RTOS projects (FreeRTOS, Zephyr)
- Master Makefile-based builds for bare-metal + RTOS
- Know firmware image generation and flashing

---

## 1. Cross-Compilation Toolchain

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Host (x86_64 Linux/Windows)  →  Target (ARM Cortex-M)  │
  │                                                           │
  │  Toolchain components:                                   │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ arm-none-eabi-gcc    → Cross compiler        │       │
  │  │ arm-none-eabi-as     → Assembler             │       │
  │  │ arm-none-eabi-ld     → Linker                │       │
  │  │ arm-none-eabi-objcopy→ ELF → BIN/HEX         │       │
  │  │ arm-none-eabi-objdump→ Disassembly           │       │
  │  │ arm-none-eabi-size   → Section sizes          │       │
  │  │ arm-none-eabi-gdb    → Debugger               │       │
  │  │ arm-none-eabi-nm     → Symbol table           │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  "arm-none-eabi" meaning:                                │
  │  arm  = target architecture                              │
  │  none = no OS (bare-metal)                               │
  │  eabi = Embedded ABI (calling convention)                │
  │                                                           │
  │  Newlib C library variants:                              │
  │  --specs=nosys.specs  → stubs for syscalls (no OS)      │
  │  --specs=nano.specs   → minimal printf/scanf (smaller)  │
  │  --specs=rdimon.specs → semihosting (debug via JTAG)    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Makefile-Based Build

```makefile
# Makefile for STM32F4 + FreeRTOS project

# Cross-compiler
CC      = arm-none-eabi-gcc
AS      = arm-none-eabi-as
LD      = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy
SIZE    = arm-none-eabi-size

# Target MCU
MCU = -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16

# Compiler flags
CFLAGS  = $(MCU) -Os -Wall -Werror
CFLAGS += -ffunction-sections -fdata-sections
CFLAGS += -DSTM32F407xx
CFLAGS += -I./include
CFLAGS += -I./FreeRTOS/Source/include
CFLAGS += -I./FreeRTOS/Source/portable/GCC/ARM_CM4F

# Linker flags
LDFLAGS  = $(MCU) -T linker.ld
LDFLAGS += --specs=nosys.specs --specs=nano.specs
LDFLAGS += -Wl,--gc-sections    # Remove unused sections
LDFLAGS += -Wl,-Map=output.map  # Generate map file

# Source files
SRCS  = src/main.c src/tasks.c
SRCS += bsp/system_stm32f4.c
SRCS += bsp/startup_stm32f407.s
SRCS += FreeRTOS/Source/tasks.c
SRCS += FreeRTOS/Source/queue.c
SRCS += FreeRTOS/Source/list.c
SRCS += FreeRTOS/Source/timers.c
SRCS += FreeRTOS/Source/portable/GCC/ARM_CM4F/port.c
SRCS += FreeRTOS/Source/portable/MemMang/heap_4.c

OBJS = $(SRCS:.c=.o)
OBJS := $(OBJS:.s=.o)

TARGET = firmware

all: $(TARGET).bin

$(TARGET).elf: $(OBJS)
	$(LD) $(LDFLAGS) -o $@ $^
	$(SIZE) $@

$(TARGET).bin: $(TARGET).elf
	$(OBJCOPY) -O binary $< $@

$(TARGET).hex: $(TARGET).elf
	$(OBJCOPY) -O ihex $< $@

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

%.o: %.s
	$(AS) $(MCU) -o $@ $<

flash: $(TARGET).bin
	st-flash write $< 0x08000000

clean:
	rm -f $(OBJS) $(TARGET).elf $(TARGET).bin $(TARGET).hex

.PHONY: all flash clean
```

---

## 3. CMake for RTOS Projects

```cmake
# CMakeLists.txt for STM32F4 + FreeRTOS

cmake_minimum_required(VERSION 3.20)

# Cross-compilation toolchain file
set(CMAKE_TOOLCHAIN_FILE ${CMAKE_SOURCE_DIR}/cmake/arm-gcc-toolchain.cmake)

project(rtos_app C ASM)

# MCU-specific flags
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_C_FLAGS "${MCU_FLAGS} -Os -Wall -ffunction-sections -fdata-sections")
set(CMAKE_EXE_LINKER_FLAGS "${MCU_FLAGS} -T${CMAKE_SOURCE_DIR}/linker.ld \
    --specs=nosys.specs --specs=nano.specs -Wl,--gc-sections \
    -Wl,-Map=${PROJECT_NAME}.map")

# FreeRTOS kernel sources
set(FREERTOS_DIR ${CMAKE_SOURCE_DIR}/FreeRTOS/Source)
add_library(freertos STATIC
    ${FREERTOS_DIR}/tasks.c
    ${FREERTOS_DIR}/queue.c
    ${FREERTOS_DIR}/list.c
    ${FREERTOS_DIR}/timers.c
    ${FREERTOS_DIR}/event_groups.c
    ${FREERTOS_DIR}/portable/GCC/ARM_CM4F/port.c
    ${FREERTOS_DIR}/portable/MemMang/heap_4.c
)
target_include_directories(freertos PUBLIC
    ${FREERTOS_DIR}/include
    ${FREERTOS_DIR}/portable/GCC/ARM_CM4F
    ${CMAKE_SOURCE_DIR}/include  # FreeRTOSConfig.h
)

# Application
add_executable(${PROJECT_NAME}.elf
    src/main.c
    src/tasks.c
    bsp/system_stm32f4.c
    bsp/startup_stm32f407.s
)
target_link_libraries(${PROJECT_NAME}.elf PRIVATE freertos)

# Post-build: generate .bin and .hex
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND arm-none-eabi-objcopy -O binary
        ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
    COMMAND arm-none-eabi-objcopy -O ihex
        ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
    COMMAND arm-none-eabi-size ${PROJECT_NAME}.elf
)
```

```cmake
# cmake/arm-gcc-toolchain.cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER   arm-none-eabi-gcc)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY      arm-none-eabi-objcopy)
set(CMAKE_SIZE         arm-none-eabi-size)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
```

---

## 4. Linker Script

```
/* linker.ld for STM32F407 (1MB Flash, 192KB SRAM) */

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    SRAM  (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    CCM   (rwx) : ORIGIN = 0x10000000, LENGTH = 64K
}

/* Entry point */
ENTRY(Reset_Handler)

/* Initial stack pointer = top of SRAM */
_estack = ORIGIN(SRAM) + LENGTH(SRAM);

SECTIONS
{
    /* Vector table and code → Flash */
    .isr_vector : {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } >FLASH

    .text : {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
        _etext = .;
    } >FLASH

    /* Initialized data: stored in Flash, copied to SRAM */
    _sidata = LOADADDR(.data);
    .data : {
        . = ALIGN(4);
        _sdata = .;
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } >SRAM AT>FLASH

    /* Uninitialized data: zeroed at startup */
    .bss : {
        . = ALIGN(4);
        _sbss = .;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
    } >SRAM

    /* FreeRTOS heap in CCM (Core Coupled Memory) */
    .ccm_heap (NOLOAD) : {
        . = ALIGN(8);
        _heap_start = .;
        . = . + 60K;
        _heap_end = .;
    } >CCM

    /* Stack (main stack before RTOS starts) */
    ._stack (NOLOAD) : {
        . = ALIGN(8);
        . = . + 2K;
        _stack_top = .;
    } >SRAM
}
```

---

## 5. Zephyr West Build System

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Zephyr uses "west" meta-tool + CMake + Kconfig:        │
  │                                                           │
  │  # Initialize Zephyr workspace                           │
  │  $ west init ~/zephyrproject                             │
  │  $ cd ~/zephyrproject && west update                     │
  │                                                           │
  │  # Build for specific board                              │
  │  $ west build -b nucleo_f411re app/                      │
  │                                                           │
  │  # Flash to target                                       │
  │  $ west flash                                            │
  │                                                           │
  │  # Debug with GDB                                        │
  │  $ west debug                                            │
  │                                                           │
  │  Build flow:                                             │
  │  1. west invokes CMake with board DTS + Kconfig          │
  │  2. Kconfig generates autoconf.h (feature toggles)      │
  │  3. Device Tree compiled → devicetree_generated.h       │
  │  4. CMake builds application + kernel + drivers          │
  │  5. Linker produces zephyr.elf → zephyr.bin             │
  │                                                           │
  │  Application CMakeLists.txt:                             │
  │  cmake_minimum_required(VERSION 3.20.0)                  │
  │  find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})  │
  │  project(my_app)                                         │
  │  target_sources(app PRIVATE src/main.c)                  │
  │                                                           │
  │  Kconfig (prj.conf):                                     │
  │  CONFIG_SERIAL=y                                         │
  │  CONFIG_GPIO=y                                           │
  │  CONFIG_HEAP_MEM_POOL_SIZE=4096                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Firmware Image Formats

```
  ┌──────────┬──────────────────────────────────────────────┐
  │ Format   │ Description                                  │
  ├──────────┼──────────────────────────────────────────────┤
  │ ELF      │ Debug symbols + code + data. Used by GDB.   │
  │          │ NOT flashed directly (too large).            │
  ├──────────┼──────────────────────────────────────────────┤
  │ BIN      │ Raw binary image. Byte-for-byte flash       │
  │          │ content. Smallest. Used by st-flash, dd.     │
  ├──────────┼──────────────────────────────────────────────┤
  │ HEX      │ Intel HEX: ASCII with addresses. Supports   │
  │          │ non-contiguous regions. Used by J-Link.      │
  ├──────────┼──────────────────────────────────────────────┤
  │ UF2      │ USB Flashing Format. Drag-and-drop to mass  │
  │          │ storage bootloader. Used by RP2040, nRF52.   │
  ├──────────┼──────────────────────────────────────────────┤
  │ S-Record │ Motorola format. Similar to HEX. Used in    │
  │ (SREC)   │ automotive/industrial tools.                 │
  └──────────┴──────────────────────────────────────────────┘

  Conversion: arm-none-eabi-objcopy
  ELF → BIN:  objcopy -O binary firmware.elf firmware.bin
  ELF → HEX:  objcopy -O ihex firmware.elf firmware.hex
  ELF → SREC: objcopy -O srec firmware.elf firmware.srec
```

---

## Interview Questions

**Q1: What are the key compiler and linker flags for building an RTOS project for Cortex-M4?**
**A:** Compiler flags: `-mcpu=cortex-m4` (target CPU), `-mthumb` (Thumb-2 instructions, smaller code), `-mfloat-abi=hard -mfpu=fpv4-sp-d16` (hardware FPU), `-Os` (optimize for size), `-Wall -Werror` (all warnings as errors), `-ffunction-sections -fdata-sections` (each function/data in own section for dead-code removal). Linker flags: `-T linker.ld` (memory layout script), `--specs=nosys.specs` (no OS syscall stubs), `--specs=nano.specs` (minimal C library for small flash), `-Wl,--gc-sections` (garbage collect unused sections), `-Wl,-Map=output.map` (map file for size analysis). Post-build: `arm-none-eabi-objcopy -O binary` converts ELF to flashable BIN, `arm-none-eabi-size` shows text/data/bss usage.

**Q2: What is the purpose of a linker script in an RTOS project?**
**A:** The linker script defines: (1) Memory regions — Flash base/size (e.g., 0x08000000, 1MB), SRAM base/size (0x20000000, 128KB), and special regions (CCM, DTCM). (2) Section placement — `.isr_vector` at Flash start (vector table must be at reset address), `.text` and `.rodata` in Flash, `.data` in SRAM (with Flash copy for initialization), `.bss` in SRAM (zeroed at startup). (3) Initial stack pointer `_estack` at top of SRAM. (4) Symbols for startup code: `_sdata/_edata/_sidata` (for copying .data from Flash to SRAM), `_sbss/_ebss` (for zeroing BSS). (5) Heap region for FreeRTOS memory management (can be placed in specific SRAM bank). Without a correct linker script, the vector table is wrong (no boot), or .data isn't copied (variables have garbage values).

---

## Summary

- Cross-compiler toolchain: `arm-none-eabi-gcc` (compiler) + `objcopy` (format conversion)
- Makefile: explicit control over sources, flags, and build rules — simple projects
- CMake: modern, out-of-tree builds, better dependency management — larger projects
- Linker script: defines memory layout, section placement, startup symbols — MCU-specific
- Zephyr: west meta-tool + CMake + Kconfig + Device Tree — most automated
- Firmware formats: ELF (debug), BIN (flash), HEX (programmer), UF2 (drag-and-drop)

---

[Previous Chapter: Porting ←](Chapter_26_Porting.md) | [Next Chapter: Kernel Source Structure →](Chapter_28_Kernel_Source.md)
