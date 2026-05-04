# Chapter 21: Memory Mapping in Drivers (mmap Deep Dive)

## Chapter Overview

The `mmap()` system call allows userspace processes to map files, devices, or anonymous memory into their virtual address space. For drivers, mmap provides zero-copy access to device memory, DMA buffers, and shared memory regions. This chapter dives deep into implementing and using mmap in kernel drivers.

---

## 21.1 mmap() System Call Internals

```
Userspace call:
  void *addr = mmap(NULL, length, prot, flags, fd, offset);

Kernel path:
  sys_mmap() → ksys_mmap_pgoff() → do_mmap()
    → get_unmapped_area()     ← Find free VA range
    → mmap_region()           ← Create VMA
      → file->f_op->mmap()   ← Call driver's mmap handler
      → vma_link()            ← Insert VMA in process mm

┌─────────────────────────────────────────────────────────────┐
│ Step 1: Find free virtual address range                     │
│   get_unmapped_area() searches the VMA gap tree             │
│                                                             │
│ Step 2: Create VMA (vm_area_struct)                         │
│   vm_start = chosen addr                                    │
│   vm_end   = addr + length                                  │
│   vm_flags = translate(prot, flags)                         │
│   vm_pgoff = offset >> PAGE_SHIFT                           │
│                                                             │
│ Step 3: Call driver's f_op->mmap(file, vma)                │
│   Driver sets up page table entries or vm_ops               │
│                                                             │
│ Step 4: Link VMA into process address space                │
│   Insert into mm->mm_mt (maple tree, 6.1+) or rb-tree     │
└─────────────────────────────────────────────────────────────┘
```

---

## 21.2 Driver mmap Implementation Approaches

### Approach 1: remap_pfn_range() — Map Physical Pages at mmap Time

```c
/*
 * Maps a contiguous range of physical pages into user VMA.
 * All pages mapped immediately (no page faults).
 * Best for: DMA buffers, device MMIO, framebuffers.
 */
static int my_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct my_device *dev = filp->private_data;
    unsigned long size = vma->vm_end - vma->vm_start;
    unsigned long pfn;

    /* Validate size */
    if (size > dev->buf_size)
        return -EINVAL;

    /* Get physical frame number */
    pfn = virt_to_phys(dev->dma_buf) >> PAGE_SHIFT;

    /* Set non-cacheable for device memory */
    vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
    
    /* Prevent the VMA from being merged or core-dumped */
    vm_flags_set(vma, VM_IO | VM_DONTEXPAND | VM_DONTDUMP);

    return remap_pfn_range(vma, vma->vm_start, pfn, size,
                           vma->vm_page_prot);
}
```

### Approach 2: vm_ops->fault() — Map Pages on Demand

```c
/*
 * Pages mapped on-demand via page fault handler.
 * More flexible: can map different pages at different offsets,
 * handle sparse mappings, track access patterns.
 */
static vm_fault_t my_fault(struct vm_fault *vmf)
{
    struct my_device *dev = vmf->vma->vm_private_data;
    unsigned long offset = vmf->pgoff << PAGE_SHIFT;
    struct page *page;

    if (offset >= dev->buf_size)
        return VM_FAULT_SIGBUS;

    /* Get the page for this offset */
    page = virt_to_page(dev->buf + offset);
    get_page(page);     /* Increment refcount */

    vmf->page = page;
    return 0;           /* VM_FAULT_NOPAGE if using vmf_insert_pfn() */
}

static const struct vm_operations_struct my_vm_ops = {
    .fault = my_fault,
    .open  = my_vma_open,
    .close = my_vma_close,
};

static int my_mmap(struct file *filp, struct vm_area_struct *vma)
{
    vma->vm_ops = &my_vm_ops;
    vma->vm_private_data = filp->private_data;
    return 0;   /* No pages mapped yet — fault handler will do it */
}
```

### Approach 3: dma_mmap_coherent() — DMA Buffer Shortcut

```c
/*
 * Kernel helper for mapping DMA coherent allocations to userspace.
 * Handles cache attribute setup and address translation automatically.
 */
static int my_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct my_device *dev = filp->private_data;

    return dma_mmap_coherent(dev->dev, vma,
                             dev->dma_buf,    /* CPU virtual addr */
                             dev->dma_addr,   /* DMA address */
                             dev->dma_size);  /* Size */
}
```

---

## 21.3 Comparison: mmap Approaches

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature          │ remap_pfn_range  │ vm_ops->fault    │ dma_mmap_coherent│
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ When mapped      │ All at mmap time │ On demand        │ All at mmap time │
│ Flexibility      │ Contiguous only  │ Any page pattern │ DMA buffers only │
│ Initial overhead │ Higher           │ Lower            │ Medium           │
│ Fault overhead   │ None             │ Per-page         │ None             │
│ Memory type      │ Any physical     │ Any kernel page  │ DMA coherent     │
│ Cache setup      │ Manual           │ Manual           │ Automatic        │
│ Complexity       │ Simple           │ Complex          │ Simplest         │
│ Use case         │ Framebuffer,MMIO │ Sparse, on-demand│ DMA shared bufs  │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## 21.4 VMA Operations (vm_operations_struct)

```c
struct vm_operations_struct {
    void (*open)(struct vm_area_struct *vma);   /* VMA duplicated (fork) */
    void (*close)(struct vm_area_struct *vma);  /* VMA destroyed (munmap) */
    
    /* Page fault: called when process accesses unmapped page */
    vm_fault_t (*fault)(struct vm_fault *vmf);
    
    /* Huge page fault */
    vm_fault_t (*huge_fault)(struct vm_fault *vmf, unsigned int order);
    
    /* Called after fault for page accounting */
    vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);
    /* Called when clean page about to become writable (mmap shared files) */
    
    /* Called before page writeback */
    vm_fault_t (*pfn_mkwrite)(struct vm_fault *vmf);
    
    /* Access tracking for NUMA balancing */
    int (*access)(struct vm_area_struct *vma, unsigned long addr,
                  void *buf, int len, int write);
};
```

### VMA Lifecycle with fork():

```
Process A:                           After fork():
┌──────────────┐                    Process A:          Process B:
│ VMA          │                    ┌──────────┐       ┌──────────┐
│ vm_ops→open  │                    │ VMA (orig)│       │ VMA (copy)│
│ vm_ops→close │                    └──────────┘       └──────────┘
│ refcount = 1 │                     refcount = 2       vm_ops->open()
└──────────────┘                                        called with
                                                        new VMA
                                    
                                    On munmap/exit:
                                    vm_ops->close() called
                                    refcount decremented
```

---

## 21.5 Page Cache Attributes for mmap

```c
/* Cache attribute helpers for vm_page_prot */

/* Normal (write-back) caching — default for RAM */
pgprot_t prot = vma->vm_page_prot;

/* Uncacheable — for MMIO registers */
pgprot_t prot = pgprot_noncached(vma->vm_page_prot);

/* Write-combining — for framebuffers, GPU memory */
pgprot_t prot = pgprot_writecombine(vma->vm_page_prot);

/* Write-through — special cases */
pgprot_t prot = pgprot_writethrough(vma->vm_page_prot);

/*
 * Why cache attributes matter:
 *
 * Write-back (WB):   CPU caches writes, flushes later → fast, coherency issues
 * Write-through (WT): CPU writes to cache AND memory → moderate, simpler coherency
 * Write-combining (WC): CPU batches writes, no caching of reads → good for FB
 * Uncacheable (UC):   Every access goes to device → slow, fully ordered/coherent
 *
 * x86 PAT (Page Attribute Table):
 * ┌─────┬─────┬─────┬─────────────┐
 * │ PAT │ PCD │ PWT │ Memory Type │
 * ├─────┼─────┼─────┼─────────────┤
 * │  0  │  0  │  0  │ WB          │
 * │  0  │  0  │  1  │ WT          │
 * │  0  │  1  │  0  │ UC-         │
 * │  0  │  1  │  1  │ UC          │
 * │  1  │  0  │  0  │ WC          │
 * └─────┴─────┴─────┴─────────────┘
 */
```

---

## 21.6 Security Considerations

```
mmap Security Checklist for Driver Developers:

1. VALIDATE OFFSET AND SIZE
   if (offset + size > dev->buf_size) return -EINVAL;
   
2. CHECK PERMISSIONS
   if ((vma->vm_flags & VM_WRITE) && !user_can_write(dev))
       return -EPERM;

3. SET APPROPRIATE VM FLAGS
   VM_IO         — Not backed by struct page (device memory)
   VM_PFNMAP     — Pages tracked by PFN, not struct page
   VM_DONTEXPAND — Prevent mremap() from expanding
   VM_DONTDUMP   — Exclude from core dumps (security)
   VM_DONTCOPY   — Don't copy on fork
   
4. PREVENT INFORMATION LEAKS
   - Zero-fill buffers before mapping to userspace
   - Use memset() on DMA buffers after allocation
   
5. HANDLE CONCURRENT ACCESS
   - Device removal while mmap active
   - Multiple processes mapping same buffer
   - fork() creating VMA copies
```

---

## 21.7 Real-World Example: Video4Linux2 mmap

```c
/*
 * V4L2 uses mmap to share video frames between kernel and userspace.
 * This is the standard zero-copy video capture path.
 */

/* Userspace: */
struct v4l2_requestbuffers req = {
    .count  = 4,
    .type   = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &req);

/* Query and mmap each buffer */
for (int i = 0; i < 4; i++) {
    struct v4l2_buffer buf = { .index = i, .type = req.type };
    ioctl(fd, VIDIOC_QUERYBUF, &buf);
    
    buffers[i] = mmap(NULL, buf.length,
                       PROT_READ | PROT_WRITE, MAP_SHARED,
                       fd, buf.m.offset);
}

/* Queue buffers, start streaming, dequeue when frame ready */
/* Camera DMA writes directly to mmap'd buffer — ZERO COPY! */

/*
 * Flow:
 * Camera sensor → ISP → DMA → Physical buffer → mmap'd to userspace
 * No copies! Userspace reads frame data directly from DMA buffer.
 */
```

---

## 21.8 dma-buf: Cross-Driver Buffer Sharing

```
dma-buf: Framework for sharing DMA buffers between drivers

Example: Camera captures frame, GPU processes it, Display shows it

┌─────────┐         ┌──────────┐        ┌──────────────┐
│ Camera   │ export  │  dma-buf │ import │ GPU          │
│ driver   │────────→│ (fd)     │←───────│ driver       │
│(exporter)│         └────┬─────┘        │(importer)    │
└─────────┘              │              └──────────────┘
                          │ import
                    ┌─────▼──────┐
                    │ Display    │
                    │ driver     │
                    │ (importer) │
                    └────────────┘

/* Exporter: */
DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
exp_info.ops = &my_dmabuf_ops;
exp_info.size = buf_size;
exp_info.priv = my_buffer;
struct dma_buf *dbuf = dma_buf_export(&exp_info);
int fd = dma_buf_fd(dbuf, O_CLOEXEC);  /* Return fd to userspace */

/* Importer: */
struct dma_buf *dbuf = dma_buf_get(fd);
struct dma_buf_attachment *attach = dma_buf_attach(dbuf, dev);
struct sg_table *sgt = dma_buf_map_attachment(attach, DMA_FROM_DEVICE);
/* Now device can DMA from shared buffer */
```

---

## Interview Questions

1. **Q: How does a driver's mmap handler work?**
   A: When userspace calls `mmap()` on a device fd, the kernel creates a VMA and calls the driver's `file_operations->mmap()`. The driver can either immediately map physical pages using `remap_pfn_range()`, set up a fault handler via `vm_operations_struct` for on-demand mapping, or use `dma_mmap_coherent()` for DMA buffers.

2. **Q: What is the difference between `remap_pfn_range()` and the fault handler approach?**
   A: `remap_pfn_range()` maps all pages immediately at mmap time (eager), while the fault handler maps pages on demand as they're accessed (lazy). Fault handler is more flexible (can map different pages at different offsets, handle sparse mappings) but has per-page fault overhead.

3. **Q: What is dma-buf and why is it important?**
   A: dma-buf is a kernel framework for sharing DMA buffers between drivers (e.g., camera → GPU → display) without copying. One driver exports a dma-buf, others import it via file descriptor. Critical for zero-copy multimedia pipelines on Android and embedded systems.

4. **Q: Why set VM_IO and VM_DONTDUMP on device mmap VMAs?**
   A: VM_IO indicates the VMA maps device I/O memory (not backed by regular struct pages), preventing the kernel from treating it as normal RAM. VM_DONTDUMP excludes the region from core dumps, preventing sensitive device data leaks.

---

## Summary

1. Driver `mmap()` enables zero-copy userspace access to kernel/device memory.
2. Three approaches: `remap_pfn_range()` (eager), `vm_ops->fault()` (lazy), `dma_mmap_coherent()` (DMA helper).
3. Cache attributes (`pgprot_noncached`, `pgprot_writecombine`) must match device requirements.
4. Security: validate offsets/sizes, set proper VM flags, prevent info leaks.
5. `dma-buf` enables zero-copy buffer sharing between multiple drivers.
6. Real-world: V4L2 video capture, GPU rendering, display output all use mmap.

---

*Next: [Chapter 22 — Memory Security Mechanisms](Chapter_22_Memory_Security.md)*
