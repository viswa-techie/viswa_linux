# Chapter 7: Kernel Data Structures

## Learning Goals
- Understand the core kernel data structures every kernel developer must know
- Deep-dive into task_struct, mm_struct, and device structures
- Master kernel linked lists, trees, and hash tables
- Know when and why each data structure is used

---

## 7.1 Core Kernel Structures

```
Key Kernel Data Structures and Their Relationships:

task_struct ────► mm_struct ────► vm_area_struct (VMA list)
     │                │               │
     │                └──► pgd_t* ────► Page Tables → Physical Pages
     │
     ├──► files_struct ──► fdtable ──► file ──► inode ──► address_space
     │
     ├──► signal_struct ──► sighand_struct ──► sigaction[]
     │
     ├──► cred ──► uid, gid, capabilities
     │
     ├──► nsproxy ──► pid_namespace, net_namespace, mnt_namespace
     │
     └──► sched_entity ──► cfs_rq (scheduler run queue)
```

---

## 7.2 Process Descriptors — task_struct

`task_struct` is the most important data structure in the kernel. Every process and thread has one.

```c
/* include/linux/sched.h — simplified */
struct task_struct {
    /* === Scheduler === */
    volatile long           state;          /* TASK_RUNNING, TASK_INTERRUPTIBLE, etc. */
    int                     prio;           /* Dynamic priority (0-139) */
    int                     static_prio;    /* Set by nice value */
    int                     normal_prio;    /* Computed priority */
    unsigned int            rt_priority;    /* Real-time priority (0-99) */
    const struct sched_class *sched_class;  /* CFS, RT, DL, idle */
    struct sched_entity     se;             /* CFS scheduling entity */
    struct sched_rt_entity  rt;             /* RT scheduling entity */
    unsigned int            policy;         /* SCHED_NORMAL, SCHED_FIFO, etc. */
    int                     on_cpu;         /* Currently running on a CPU */
    int                     on_rq;          /* On run queue */
    struct list_head        tasks;          /* Global process list */
    cpumask_t               cpus_mask;      /* CPU affinity mask */

    /* === Identity === */
    pid_t                   pid;            /* Process ID */
    pid_t                   tgid;           /* Thread Group ID (= PID for main thread) */
    struct task_struct      *real_parent;   /* Real parent process */
    struct task_struct      *parent;        /* Current parent (ptrace) */
    struct list_head        children;       /* Child process list */
    struct list_head        sibling;        /* Sibling process list */
    struct task_struct      *group_leader;  /* Thread group leader */
    char                    comm[TASK_COMM_LEN]; /* Process name (16 chars) */

    /* === Memory === */
    struct mm_struct        *mm;            /* User-space memory descriptor */
    struct mm_struct        *active_mm;     /* Active address space */

    /* === File System === */
    struct files_struct     *files;         /* Open file table */
    struct fs_struct        *fs;            /* Root dir, pwd */
    struct nsproxy          *nsproxy;       /* Namespaces */

    /* === Credentials === */
    const struct cred       *cred;          /* uid, gid, capabilities */

    /* === Signals === */
    struct signal_struct    *signal;        /* Signal shared state */
    struct sighand_struct   *sighand;       /* Signal handlers */
    sigset_t                blocked;        /* Blocked signal mask */
    struct sigpending       pending;        /* Pending signals */

    /* === Timing === */
    u64                     utime;          /* User-space CPU time */
    u64                     stime;          /* Kernel-space CPU time */
    u64                     start_time;     /* Process start time (monotonic) */

    /* === Stack === */
    void                    *stack;         /* Kernel stack pointer */
    /* ... many more fields (~700+ bytes total) */
};
```

```
task_struct Layout in Memory:

┌──────────────────────────────────────────┐
│            task_struct                    │
│                                          │
│  state          ┌─ TASK_RUNNING          │
│  pid = 1234     │  TASK_INTERRUPTIBLE    │
│  tgid = 1230    │  TASK_UNINTERRUPTIBLE  │
│  comm = "nginx" │  __TASK_STOPPED        │
│                 └─ EXIT_ZOMBIE           │
│                                          │
│  mm ──────────► mm_struct (user VM)      │
│  files ───────► files_struct (fd table)  │
│  cred ────────► cred (uid/gid/caps)      │
│  signal ──────► signal_struct            │
│  stack ───────► kernel stack (16 KB)     │
│                                          │
│  se.vruntime    (CFS virtual runtime)    │
│  se.load        (scheduling weight)      │
│                                          │
│  tasks ◄──────► (global task list)       │
│  children ◄───► (child list)             │
│  sibling ◄────► (sibling list)           │
└──────────────────────────────────────────┘
```

---

## 7.3 Memory Descriptors

```c
/* include/linux/mm_types.h — simplified */
struct mm_struct {
    struct maple_tree   mm_mt;        /* VMA tree (was rb_root in older kernels) */
    unsigned long       mmap_base;    /* mmap region start */
    unsigned long       task_size;    /* User address space size */

    pgd_t              *pgd;          /* Page Global Directory (top-level page table) */

    atomic_t            mm_users;     /* How many users (processes) share this mm */
    atomic_t            mm_count;     /* Reference count */

    unsigned long       start_code;   /* Text segment start */
    unsigned long       end_code;     /* Text segment end */
    unsigned long       start_data;   /* Data segment start */
    unsigned long       end_data;     /* Data segment end */
    unsigned long       start_brk;    /* Heap start */
    unsigned long       brk;          /* Current heap end */
    unsigned long       start_stack;  /* Stack start */
    unsigned long       arg_start;    /* Argv start */
    unsigned long       env_start;    /* Environment start */

    unsigned long       total_vm;     /* Total pages mapped */
    unsigned long       locked_vm;    /* Locked (non-swappable) pages */
};

/* VMA — describes one contiguous virtual memory region */
struct vm_area_struct {
    unsigned long       vm_start;     /* Region start address */
    unsigned long       vm_end;       /* Region end address (exclusive) */
    pgoff_t             vm_pgoff;     /* Offset in file (for file-backed mappings) */
    unsigned long       vm_flags;     /* VM_READ | VM_WRITE | VM_EXEC | ... */
    struct file         *vm_file;     /* File being mapped (NULL for anonymous) */
    const struct vm_operations_struct *vm_ops;  /* Page fault handlers */
};
```

```
Process Memory Layout with VMAs:

mm_struct
   │
   └──► VMA tree (maple_tree)
         │
         ├── VMA: 0x400000 - 0x410000  [r-x]  /usr/bin/app (text)
         ├── VMA: 0x610000 - 0x612000  [rw-]  /usr/bin/app (data)
         ├── VMA: 0x612000 - 0x700000  [rw-]  [heap]
         ├── VMA: 0x7f000000 - 0x7f200000  [r-x]  /lib/libc.so.6
         ├── VMA: 0x7f400000 - 0x7f402000  [rw-]  [anon] (malloc)
         └── VMA: 0x7fffe000 - 0x80000000  [rw-]  [stack]
```

---

## 7.4 Device Structures

```c
/* include/linux/device.h — Device model core structures */

/* Bus type — represents a bus (PCI, USB, I2C, platform) */
struct bus_type {
    const char  *name;            /* "pci", "usb", "i2c", "platform" */
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    /* ... */
};

/* Device — represents a hardware device */
struct device {
    struct device       *parent;         /* Parent device */
    struct bus_type     *bus;            /* Bus this device is on */
    struct device_driver *driver;        /* Bound driver */
    void                *platform_data;  /* Platform-specific data */
    void                *driver_data;    /* Driver private data */
    struct device_node  *of_node;        /* Device tree node */
    dev_t               devt;            /* Major:minor number */
    struct class        *class;          /* Device class */
    const char          *init_name;      /* Initial device name */
    /* ... */
};

/* Driver — represents a device driver */
struct device_driver {
    const char          *name;           /* Driver name */
    struct bus_type     *bus;            /* Bus type */
    int (*probe)(struct device *dev);    /* Called when device matched */
    void (*remove)(struct device *dev);  /* Called on unbind */
    const struct of_device_id *of_match_table;  /* DT compatible list */
    /* ... */
};
```

```
Device Model Hierarchy (sysfs):

/sys/
├── bus/
│   ├── platform/
│   │   ├── devices/
│   │   │   ├── a0010000.gpio → ../../../devices/platform/a0010000.gpio
│   │   │   └── a0020000.i2c  → ../../../devices/platform/a0020000.i2c
│   │   └── drivers/
│   │       ├── gpio-sa8155/
│   │       └── i2c-qcom/
│   ├── i2c/
│   └── pci/
├── class/
│   ├── leds/
│   ├── net/
│   └── input/
└── devices/
    └── platform/
        └── a0010000.gpio/
            ├── driver → ../../../../bus/platform/drivers/gpio-sa8155
            ├── of_node → ../../../../firmware/devicetree/base/...
            └── subsystem → ../../../../bus/platform
```

---

## 7.5 Kernel Linked Lists

The kernel uses an intrusive circular doubly-linked list — the list nodes are embedded inside the data structures.

```c
/* include/linux/list.h */

/* The list head (embedded in your struct) */
struct list_head {
    struct list_head *next, *prev;
};

/* Usage example: a list of tasks */
struct my_task {
    int                 id;
    char                name[32];
    struct list_head    list;    /* ← Embedded list node */
};

/* Initialize */
LIST_HEAD(task_list);  /* Static: declares and init empty list */

/* Or dynamic: */
struct list_head task_list;
INIT_LIST_HEAD(&task_list);

/* Add to list */
struct my_task *t = kmalloc(sizeof(*t), GFP_KERNEL);
t->id = 1;
list_add(&t->list, &task_list);        /* Add at head */
list_add_tail(&t->list, &task_list);   /* Add at tail */

/* Iterate */
struct my_task *entry;
list_for_each_entry(entry, &task_list, list) {
    pr_info("task: %d %s\n", entry->id, entry->name);
}

/* Safe iteration (allows deletion during iteration) */
struct my_task *tmp;
list_for_each_entry_safe(entry, tmp, &task_list, list) {
    if (entry->id == 42) {
        list_del(&entry->list);
        kfree(entry);
    }
}

/* Get containing struct from list_head pointer */
struct my_task *t = list_entry(ptr, struct my_task, list);
/* Same as: container_of(ptr, struct my_task, list) */
```

```
Kernel Linked List (intrusive, circular):

                 ┌──────────────────────────┐
                 │                          │
    ┌────────────▼──┐    ┌──────────────┐   │    ┌──────────────┐
    │  task_list    │    │  my_task A   │   │    │  my_task B   │
    │  (head)       │    │  id = 1      │   │    │  id = 2      │
    │  next ─────────►   │  list.next ──────►   │  list.next ──────┐
    │  prev ◄────────    │  list.prev ◄──────   │  list.prev ◄─   │
    └───────────────┘    └──────────────┘        └──────────────┘  │
         ▲                                                         │
         └─────────────────────────────────────────────────────────┘
```

Other kernel data structures:

| Data Structure | Header | Use Case |
|---------------|--------|----------|
| `list_head` | `linux/list.h` | Doubly-linked list (most common) |
| `hlist_head` | `linux/list.h` | Hash table bucket list (singly-linked) |
| `rb_root` / `rb_node` | `linux/rbtree.h` | Red-black tree (VMAs, scheduler) |
| `maple_tree` | `linux/maple_tree.h` | Range-based tree (VMAs in 6.x) |
| `radix_tree_root` | `linux/radix-tree.h` | Radix tree (page cache) |
| `xarray` | `linux/xarray.h` | eXtensible Array (replacing radix tree) |
| `idr` | `linux/idr.h` | ID to pointer mapping |
| `kfifo` | `linux/kfifo.h` | Lock-free FIFO queue |
| `bitmap` | `linux/bitmap.h` | Bit arrays (CPU masks, IRQ masks) |

---

## Interview Questions

**Q1: Why does Linux use intrusive linked lists instead of traditional ones?**
A: Intrusive lists embed the `list_head` inside the data structure. This means: (1) no separate allocation for list nodes, (2) a single object can be on multiple lists simultaneously, (3) `container_of()` retrieves the parent struct from the list pointer. Traditional lists would require extra allocation and indirection.

**Q2: What is `container_of()` and how does it work?**
A: `container_of(ptr, type, member)` takes a pointer to a member of a struct and returns a pointer to the containing struct. It works by subtracting the member's offset from the pointer: `(type *)((char *)ptr - offsetof(type, member))`. It's fundamental to kernel linked lists and the driver model.

**Q3: What is the difference between mm_struct and vm_area_struct?**
A: `mm_struct` describes the entire address space of a process (page tables, segment boundaries, VM statistics). `vm_area_struct` describes one contiguous virtual memory region within that address space (start, end, permissions, backing file). A process has one `mm_struct` containing many `vm_area_struct` entries.

**Q4: Why are kernel threads' mm pointer NULL?**
A: Kernel threads don't have their own user-space address space — they only run in kernel space. They "borrow" the `active_mm` of whatever process was running before (lazy TLB switching) to avoid unnecessary page table switches. This saves the cost of loading a new page table.

---

## Summary

- `task_struct` is the central process descriptor (~700+ bytes) containing all process state
- `mm_struct` describes a process's entire virtual address space; VMAs describe individual regions
- The device model uses `bus_type`, `device`, and `device_driver` with match/probe binding
- Kernel linked lists are intrusive (embedded `list_head`) with circular doubly-linked design
- `container_of()` / `list_entry()` converts member pointers back to parent struct pointers
- Red-black trees, maple trees, radix trees, and hash lists serve specialized lookup needs

---

*Next: [Chapter 8 — Linux Kernel Modules](Chapter_08_Kernel_Modules.md)*
