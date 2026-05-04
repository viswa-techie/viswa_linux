# Chapter 9: Block Device Drivers

## Chapter Overview

Block device drivers manage storage devices where data is transferred in fixed-size blocks. They integrate with the block layer's request queue and I/O scheduler for optimal performance.

---

## 9.1 Block Device Driver Overview

```
User: read(fd, buf, 4096)
       │
       ▼
  VFS → Filesystem (ext4) → Page Cache
       │
       ▼ (cache miss)
  Block Layer → I/O Scheduler → Request Queue
       │
       ▼
  block_device_operations → Driver
       │
       ▼
  Hardware (DMA to/from disk)
```

### Block vs Character

| Feature | Block | Character |
|---------|-------|-----------|
| Transfer unit | Fixed blocks (512B/4KB) | Byte stream |
| Buffering | Page cache + I/O scheduler | None (driver manages) |
| Random access | First-class support | Optional |
| Request merging | Yes (scheduler merges adjacent) | No |
| Typical devices | HDD, SSD, eMMC, NVMe | UART, sensors, cameras |

---

## 9.2-9.3 Block I/O Subsystem

### Multi-Queue Block Layer (blk-mq)

Linux 5.x+ uses blk-mq exclusively (the legacy single-queue path was removed in 5.0):

```
           submit_bio()
               │
               ▼
         ┌──────────────┐
         │  Software     │  Per-CPU software queues
         │  Staging      │  (for request accounting)
         │  Queues       │
         └──────┬───────┘
                │  merge, sort
                ▼
         ┌──────────────┐
         │  Hardware     │  Maps to hardware submission queues
         │  Dispatch     │  (NVMe: up to 64K queues)
         │  Queues       │
         └──────┬───────┘
                │
                ▼
         Driver queue_rq() callback
                │
                ▼
         Hardware DMA
```

---

## 9.4-9.5 Block Layer Architecture

### Key Data Structures

```c
/* The disk representation */
struct gendisk {
    int major, first_minor;
    int minors;                     /* max partitions */
    char disk_name[DISK_NAME_LEN];  /* e.g., "sda" */
    struct block_device_operations *fops;
    struct request_queue *queue;
    void *private_data;
    /* ... */
};

/* Block device operations (simpler than char fops) */
struct block_device_operations {
    int (*open)(struct gendisk *disk, blk_mode_t mode);
    void (*release)(struct gendisk *disk);
    int (*ioctl)(struct block_device *bdev, blk_mode_t mode,
                 unsigned cmd, unsigned long arg);
    int (*getgeo)(struct block_device *, struct hd_geometry *);
    /* ... */
};

/* blk-mq operations — the heart of throughput */
struct blk_mq_ops {
    blk_status_t (*queue_rq)(struct blk_mq_hw_ctx *hctx,
                             const struct blk_mq_queue_data *bd);
    void (*complete)(struct request *rq);
    int (*init_hctx)(struct blk_mq_hw_ctx *, void *, unsigned int);
    /* ... */
};
```

---

## 9.6 Complete RAM Disk Driver Example

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * simple_blk.c — Minimal RAM-backed block device using blk-mq
 */
#include <linux/module.h>
#include <linux/blkdev.h>
#include <linux/blk-mq.h>
#include <linux/hdreg.h>

#define DEVICE_NAME   "simpleblk"
#define SECTOR_SIZE   512
#define NSECTORS      (1024 * 1024 * 2)  /* 1GB in sectors */

struct simple_blk {
    struct gendisk *gd;
    struct blk_mq_tag_set tag_set;
    u8 *data;   /* RAM backing store */
};
static struct simple_blk sblk;

static blk_status_t sblk_queue_rq(struct blk_mq_hw_ctx *hctx,
                                   const struct blk_mq_queue_data *bd)
{
    struct request *rq = bd->rq;
    struct bio_vec bvec;
    struct req_iterator iter;
    loff_t pos = blk_rq_pos(rq) * SECTOR_SIZE;

    blk_mq_start_request(rq);

    rq_for_each_segment(bvec, rq, iter) {
        void *buf = page_address(bvec.bv_page) + bvec.bv_offset;
        unsigned int len = bvec.bv_len;

        if (rq_data_dir(rq) == WRITE)
            memcpy(sblk.data + pos, buf, len);
        else
            memcpy(buf, sblk.data + pos, len);
        pos += len;
    }

    blk_mq_end_request(rq, BLK_STS_OK);
    return BLK_STS_OK;
}

static const struct blk_mq_ops sblk_mq_ops = {
    .queue_rq = sblk_queue_rq,
};

static const struct block_device_operations sblk_fops = {
    .owner = THIS_MODULE,
};

static int __init sblk_init(void)
{
    sblk.data = vzalloc((size_t)NSECTORS * SECTOR_SIZE);
    if (!sblk.data)
        return -ENOMEM;

    sblk.tag_set.ops = &sblk_mq_ops;
    sblk.tag_set.nr_hw_queues = 1;
    sblk.tag_set.queue_depth = 128;
    sblk.tag_set.numa_node = NUMA_NO_NODE;
    sblk.tag_set.flags = BLK_MQ_F_SHOULD_MERGE;
    if (blk_mq_alloc_tag_set(&sblk.tag_set))
        goto err_data;

    sblk.gd = blk_mq_alloc_disk(&sblk.tag_set, NULL, NULL);
    if (IS_ERR(sblk.gd))
        goto err_tag;

    sblk.gd->major = 0;  /* dynamic major */
    sblk.gd->first_minor = 0;
    sblk.gd->minors = 1;
    sblk.gd->fops = &sblk_fops;
    snprintf(sblk.gd->disk_name, DISK_NAME_LEN, DEVICE_NAME);
    set_capacity(sblk.gd, NSECTORS);

    if (add_disk(sblk.gd))
        goto err_disk;

    pr_info("simpleblk: created %llu MB RAM disk\n",
            (u64)NSECTORS * SECTOR_SIZE / 1024 / 1024);
    return 0;

err_disk:
    put_disk(sblk.gd);
err_tag:
    blk_mq_free_tag_set(&sblk.tag_set);
err_data:
    vfree(sblk.data);
    return -ENOMEM;
}

static void __exit sblk_exit(void)
{
    del_gendisk(sblk.gd);
    put_disk(sblk.gd);
    blk_mq_free_tag_set(&sblk.tag_set);
    vfree(sblk.data);
    pr_info("simpleblk: removed\n");
}

module_init(sblk_init);
module_exit(sblk_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Simple RAM block device");
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `block/blk-mq.c` | Multi-queue block layer core |
| `block/blk-core.c` | Block core: submit_bio, etc. |
| `include/linux/blkdev.h` | `struct gendisk`, `struct request` |
| `include/linux/blk-mq.h` | `struct blk_mq_ops`, tag set |
| `drivers/block/brd.c` | Kernel's built-in RAM disk (reference) |
| `drivers/nvme/host/` | NVMe driver (production blk-mq) |

---

## Interview Questions

**Q1: What is the blk-mq framework?**
A: Multi-queue block I/O layer introduced in Linux 3.13, mandatory since 5.0. Maps software queues (per-CPU) to hardware queues (per-NVMe/SCSI queue), enabling parallel I/O submission without contention.

**Q2: What is the I/O path from userspace read to disk?**
A: read() → VFS → filesystem → page cache check → cache miss → submit_bio() → blk-mq software queue → hardware dispatch queue → driver queue_rq() → DMA to device → completion interrupt → blk_mq_end_request().

**Q3: What are BIOs and requests?**
A: A `struct bio` represents a single I/O operation (set of pages). The block layer merges adjacent BIOs into a `struct request` for efficiency, which is what the driver actually processes via queue_rq().

---

*Next: [Chapter 10 — Network Device Drivers](Chapter_10_Network_Device_Drivers.md)*
