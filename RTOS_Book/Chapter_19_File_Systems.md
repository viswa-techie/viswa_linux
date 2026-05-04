# Chapter 19: File Systems in RTOS

## Learning Goals
- Understand embedded file system requirements and constraints
- Learn FAT, LittleFS, and SPIFFS architectures
- Know wear leveling and power-loss resilience
- Master flash memory characteristics (NOR, NAND)
- Compare file systems across RTOS platforms

---

## 1. Flash Memory Fundamentals

```
  ┌──────────────────────────────────────────────────────────┐
  │  NOR Flash (typical MCU internal + external SPI Flash):  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Byte-addressable for reads (execute-in-place) │     │
  │  │ · Write: must erase before write                │      │
  │  │ · Erase granularity: SECTOR (4KB typical)       │      │
  │  │ · Write granularity: page (256 bytes typical)   │      │
  │  │ · Erase sets all bits to 1 (0xFF)               │      │
  │  │ · Write can only change 1→0 (not 0→1)           │      │
  │  │ · Endurance: 100K erase cycles per sector       │      │
  │  │ · Size: 1MB - 64MB typical (SPI NOR)            │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Erase-before-write constraint:                          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │  Sector (4KB):                                  │      │
  │  │  [0xFF 0xFF 0xFF 0xFF ...] ← After erase        │      │
  │  │  [0xAB 0xCD 0xFF 0xFF ...] ← After write        │      │
  │  │  [0xAB 0xCD 0x12 0xFF ...] ← Another write OK   │      │
  │  │  [0xAB 0xCD 0x12 0xFF ...] ← Can't change 0xAB! │     │
  │  │  Must erase entire 4KB sector first              │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Wear leveling: distribute erases across all sectors     │
  │  Without it: hot sector wears out → permanent failure    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. LittleFS — Embedded File System

```c
/* LittleFS: designed for NOR flash with power-loss resilience */
/*
 * Properties:
 * - Bounded RAM/ROM usage
 * - Power-loss resilient (copy-on-write)
 * - Dynamic wear leveling
 * - No external tools needed (no mkfs required)
 * - Footprint: ~15KB Flash, ~4KB RAM
 */

#include "lfs.h"

/* Configuration for SPI NOR flash */
lfs_t lfs;
lfs_file_t file;

const struct lfs_config cfg = {
    .read  = spi_flash_read,     /* Platform-specific */
    .prog  = spi_flash_write,
    .erase = spi_flash_erase,
    .sync  = spi_flash_sync,

    .read_size      = 256,       /* Minimum read size */
    .prog_size      = 256,       /* Minimum write size (page) */
    .block_size     = 4096,      /* Erase block size (sector) */
    .block_count    = 1024,      /* Total blocks (4MB flash) */
    .cache_size     = 256,       /* Per-file cache */
    .lookahead_size = 16,        /* Block allocator lookahead */
    .block_cycles   = 500,       /* Wear leveling trigger */
};

/* Mount (or format on first use) */
void fs_init(void) {
    int err = lfs_mount(&lfs, &cfg);
    if (err) {
        lfs_format(&lfs, &cfg);  /* First-time format */
        lfs_mount(&lfs, &cfg);
    }
}

/* File operations */
void log_sensor_data(uint32_t value) {
    lfs_file_open(&lfs, &file, "sensor.log",
                  LFS_O_WRONLY | LFS_O_CREAT | LFS_O_APPEND);
    lfs_file_write(&lfs, &file, &value, sizeof(value));
    lfs_file_close(&lfs, &file);
    /* Power-safe: if power lost during write, file is not corrupted */
}

/*
 * LittleFS copy-on-write (power-loss resilience):
 *
 * Update file "config.dat":
 * 1. Write new data to a FREE block (don't modify original)
 * 2. Update metadata to point to new block
 * 3. Mark old block as free
 *
 * If power lost at step 1: old data intact (new block incomplete)
 * If power lost at step 2: old data intact (metadata not updated)
 * If power lost at step 3: old block leaked (recovered on mount)
 */
```

---

## 3. FAT File System

```c
/* FatFs: FAT12/16/32 for SD cards and USB drives */
/* Most compatible with PC — SD card readable on any computer */

#include "ff.h"

FATFS fs;
FIL file;
FRESULT res;

void fat_init(void) {
    /* Mount SD card */
    res = f_mount(&fs, "0:", 1);  /* "0:" = drive 0, 1 = mount now */

    if (res == FR_NO_FILESYSTEM) {
        /* Format SD card */
        uint8_t work[FF_MAX_SS];
        f_mkfs("0:", NULL, work, sizeof(work));
        f_mount(&fs, "0:", 1);
    }
}

void write_log(const char *msg) {
    UINT bw;
    f_open(&file, "0:log.txt", FA_WRITE | FA_CREATE_ALWAYS);
    f_write(&file, msg, strlen(msg), &bw);
    f_close(&file);
}

/*
 * FAT vs LittleFS for RTOS:
 *
 * ┌──────────────┬───────────────┬──────────────────┐
 * │ Feature      │ FAT (FatFs)   │ LittleFS          │
 * ├──────────────┼───────────────┼──────────────────┤
 * │ Media        │ SD, USB, disk │ NOR/NAND flash    │
 * │ PC compat    │ Yes (standard)│ No                │
 * │ Power-safe   │ No            │ Yes (CoW)         │
 * │ Wear level   │ No            │ Yes (dynamic)     │
 * │ RAM usage    │ ~4-8KB        │ ~4KB              │
 * │ Flash usage  │ ~15KB         │ ~15KB             │
 * │ Directories  │ Full tree     │ Full tree         │
 * │ Max file size│ 4GB (FAT32)   │ Limited by flash  │
 * │ Use case     │ Data exchange │ Config storage    │
 * └──────────────┴───────────────┴──────────────────┘
 */
```

---

## 4. File System Thread Safety

```c
/* Making file system operations thread-safe in RTOS */

/* FatFs: built-in RTOS support via ff_conf.h */
/* #define FF_FS_REENTRANT  1 */
/* #define FF_FS_TIMEOUT    1000 */

/* FatFs calls these (must implement for your RTOS): */
int ff_cre_syncobj(BYTE vol, FF_SYNC_t *sobj) {
    *sobj = xSemaphoreCreateMutex();
    return (*sobj != NULL) ? 1 : 0;
}

int ff_req_grant(FF_SYNC_t sobj) {
    return (xSemaphoreTake(sobj, pdMS_TO_TICKS(FF_FS_TIMEOUT))
            == pdTRUE) ? 1 : 0;
}

void ff_rel_grant(FF_SYNC_t sobj) {
    xSemaphoreGive(sobj);
}

int ff_del_syncobj(FF_SYNC_t sobj) {
    vSemaphoreDelete(sobj);
    return 1;
}

/* LittleFS: not thread-safe by default — wrap with mutex */
static SemaphoreHandle_t xFsMutex;

int fs_write_safe(const char *path, const void *data, size_t len) {
    xSemaphoreTake(xFsMutex, portMAX_DELAY);

    lfs_file_t file;
    int err = lfs_file_open(&lfs, &file, path,
                            LFS_O_WRONLY | LFS_O_CREAT | LFS_O_TRUNC);
    if (err == 0) {
        lfs_file_write(&lfs, &file, data, len);
        lfs_file_close(&lfs, &file);
    }

    xSemaphoreGive(xFsMutex);
    return err;
}
```

---

## 5. File System Comparison

| Feature | LittleFS | FatFs | SPIFFS | NVS (Zephyr) |
|---------|----------|-------|--------|-------------|
| **Target** | NOR flash | SD/USB | SPI NOR | Flash KV store |
| **Power-safe** | Yes (CoW) | No | Yes | Yes |
| **Wear level** | Dynamic | No | Static | Yes |
| **Directories** | Yes | Yes | No (flat) | No (flat KV) |
| **PC compat** | No | Yes | No | No |
| **RAM** | ~4KB | ~4-8KB | ~1KB | ~1KB |
| **RTOS support** | Wrap with mutex | Built-in | Wrap | Thread-safe |
| **Best for** | Config, logs | Data exchange | Web content | Settings |

---

## Interview Questions

**Q1: Why can't you use FAT on internal NOR flash?**
**A:** Two reasons: (1) No wear leveling — FAT writes to the same sectors repeatedly (FAT table, directory entries). Internal NOR flash has limited erase cycles (~100K). Without wear leveling, those hot sectors wear out quickly, causing permanent data loss. (2) Not power-safe — if power is lost during a multi-step FAT update (allocate cluster → write data → update directory → update FAT), the file system can be corrupted. FAT has no journaling or copy-on-write. LittleFS solves both: copy-on-write prevents corruption, and dynamic wear leveling distributes erases. FAT is fine for SD cards because SD controllers have built-in wear leveling and error correction.

**Q2: How does LittleFS achieve power-loss resilience?**
**A:** LittleFS uses copy-on-write (CoW) for all metadata and file updates. When modifying a file: (1) New data is written to a previously free block — the original block is not modified. (2) Parent metadata (directory entry) is updated to point to the new block, also via CoW. (3) Old blocks are marked free. If power is lost at any step: either the old data is intact (metadata not updated) or the new data is complete (metadata updated). On mount, LittleFS scans for orphaned blocks (written but never linked) and reclaims them. The only cost is needing spare blocks for CoW — LittleFS requires some free space to operate.

---

## Summary

- NOR flash: erase-before-write, sector-level erase, limited endurance (100K cycles)
- LittleFS: power-safe (CoW), wear leveling, ideal for internal flash config/logging
- FAT (FatFs): PC-compatible, good for SD cards, not power-safe or wear-leveling
- SPIFFS: flat file system for SPI NOR, power-safe but no directories
- Thread safety: wrap file operations with mutex or use built-in FS reentrancy
- Always use wear-leveling FS on flash with limited endurance

---

[Previous Chapter: Networking ←](Chapter_18_Networking.md) | [Next Chapter: Real-Time Communication Protocols →](Chapter_20_RT_Protocols.md)
