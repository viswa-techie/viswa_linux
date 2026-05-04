# Chapter 20: eBPF Maps and Helpers

## Learning Goals
- Understand eBPF map types and operations
- Learn helper functions for tracing, networking, and data access
- Master map pinning, per-CPU maps, and ring buffer
- Know kernel-userspace communication patterns

---

## 1. eBPF Maps

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Maps = shared data structures between eBPF programs    │
  │  and between eBPF and userspace                         │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ Userspace          Kernel                │            │
  │  │ ┌─────────┐      ┌─────────────┐        │            │
  │  │ │ bpf()   │ ←──→ │ eBPF Map    │        │            │
  │  │ │ syscall │      │ (hash, array│        │            │
  │  │ │ lookup/ │      │  ringbuf..) │        │            │
  │  │ │ update/ │      └──────┬──────┘        │            │
  │  │ │ delete  │             │               │            │
  │  │ └─────────┘      ┌──────▼──────┐        │            │
  │  │                  │ eBPF Program│        │            │
  │  │                  │ (kprobe,    │        │            │
  │  │                  │  XDP, etc.) │        │            │
  │  │                  └─────────────┘        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Map types:                                              │
  │  ┌─────────────────┬────────────────────────────────────┐│
  │  │ Type            │ Description                        ││
  │  ├─────────────────┼────────────────────────────────────┤│
  │  │ BPF_MAP_TYPE_   │                                    ││
  │  │ HASH            │ Hash table (key → value)           ││
  │  │ ARRAY           │ Fixed-size array (index → value)   ││
  │  │ PERCPU_HASH     │ Per-CPU hash (no lock contention)  ││
  │  │ PERCPU_ARRAY    │ Per-CPU array                      ││
  │  │ LRU_HASH        │ LRU eviction hash table           ││
  │  │ RINGBUF         │ Ring buffer (efficient bulk data)  ││
  │  │ PERF_EVENT_ARRAY│ Per-CPU perf event output          ││
  │  │ STACK_TRACE     │ Stack trace storage                ││
  │  │ PROG_ARRAY      │ Tail call table (prog → prog)     ││
  │  │ LPM_TRIE        │ Longest prefix match (IP routing)  ││
  │  │ QUEUE / STACK   │ FIFO queue / LIFO stack            ││
  │  │ BLOOM_FILTER    │ Probabilistic membership test      ││
  │  │ CGROUP_STORAGE  │ Per-cgroup data                    ││
  │  │ TASK_STORAGE    │ Per-task data                      ││
  │  └─────────────────┴────────────────────────────────────┘│
  │                                                           │
  │  Map definition (libbpf style):                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ struct {                                 │            │
  │  │   __uint(type, BPF_MAP_TYPE_HASH);       │            │
  │  │   __uint(max_entries, 10240);            │            │
  │  │   __type(key, u32);    /* PID */         │            │
  │  │   __type(value, u64);  /* count */       │            │
  │  │ } my_map SEC(".maps");                   │            │
  │  │                                          │            │
  │  │ /* In eBPF program: */                   │            │
  │  │ u32 key = bpf_get_current_pid_tgid() >> 32;│         │
  │  │ u64 *val = bpf_map_lookup_elem(&my_map, &key);│      │
  │  │ if (val) {                               │            │
  │  │   __sync_fetch_and_add(val, 1);          │            │
  │  │ } else {                                 │            │
  │  │   u64 init = 1;                          │            │
  │  │   bpf_map_update_elem(&my_map, &key,     │            │
  │  │     &init, BPF_ANY);                     │            │
  │  │ }                                        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Ring buffer (preferred for events):                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ BPF_MAP_TYPE_RINGBUF (since 5.8):       │            │
  │  │ - Single shared buffer (not per-CPU)     │            │
  │  │ - Preserves event ordering               │            │
  │  │ - Efficient: no per-event wakeup needed  │            │
  │  │ - Supports reserve/commit pattern        │            │
  │  │                                          │            │
  │  │ /* Producer (eBPF program): */           │            │
  │  │ struct event *e;                         │            │
  │  │ e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);│        │
  │  │ if (!e) return 0;                        │            │
  │  │ e->pid = bpf_get_current_pid_tgid() >> 32;│          │
  │  │ e->ts = bpf_ktime_get_ns();             │            │
  │  │ bpf_ringbuf_submit(e, 0);               │            │
  │  │                                          │            │
  │  │ /* Consumer (userspace): */              │            │
  │  │ ring_buffer__poll(rb, 100 /* ms */);     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Helper Functions

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Helpers: kernel functions callable from eBPF programs  │
  │  (validated by verifier per program type)               │
  │                                                           │
  │  Key helpers:                                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Map operations:                          │            │
  │  │   bpf_map_lookup_elem(&map, &key)        │            │
  │  │   bpf_map_update_elem(&map, &key, &val,  │            │
  │  │     flags)                               │            │
  │  │   bpf_map_delete_elem(&map, &key)        │            │
  │  │                                          │            │
  │  │ Process context:                         │            │
  │  │   bpf_get_current_pid_tgid()   → pid/tgid│           │
  │  │   bpf_get_current_uid_gid()    → uid/gid │            │
  │  │   bpf_get_current_comm(buf, sz) → name   │            │
  │  │   bpf_get_current_task()       → task_struct│         │
  │  │                                          │            │
  │  │ Time:                                    │            │
  │  │   bpf_ktime_get_ns()           → monotonic│           │
  │  │   bpf_ktime_get_boot_ns()      → boot    │            │
  │  │                                          │            │
  │  │ Output:                                  │            │
  │  │   bpf_printk(fmt, ...)  → trace_pipe     │            │
  │  │   bpf_ringbuf_output/reserve/submit      │            │
  │  │   bpf_perf_event_output()                │            │
  │  │                                          │            │
  │  │ Stack traces:                            │            │
  │  │   bpf_get_stackid(ctx, &stack_map, flags)│            │
  │  │   bpf_get_stack(ctx, buf, sz, flags)     │            │
  │  │                                          │            │
  │  │ Networking:                              │            │
  │  │   bpf_skb_load_bytes()                   │            │
  │  │   bpf_redirect()                         │            │
  │  │   bpf_xdp_adjust_head/tail()             │            │
  │  │   bpf_fib_lookup()                       │            │
  │  │                                          │            │
  │  │ Memory:                                  │            │
  │  │   bpf_probe_read_kernel(dst, sz, src)    │            │
  │  │   bpf_probe_read_user(dst, sz, src)      │            │
  │  │   bpf_core_read()  (CO-RE, BTF-based)   │            │
  │  │                                          │            │
  │  │ Tail calls:                              │            │
  │  │   bpf_tail_call(ctx, &prog_array, idx)   │            │
  │  │   → jump to another eBPF program        │            │
  │  │   → max 33 chain depth                  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Map pinning (persist across program exit):              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ bpf_obj_pin(map_fd,                      │            │
  │  │   "/sys/fs/bpf/my_map");                 │            │
  │  │                                          │            │
  │  │ /* Later, another program: */            │            │
  │  │ bpf_obj_get("/sys/fs/bpf/my_map");       │            │
  │  │                                          │            │
  │  │ → maps/programs survive creator exit    │            │
  │  │ → enables multi-program data sharing    │            │
  │  │ → bpffs: mount -t bpf none /sys/fs/bpf  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does eBPF communicate data from kernel to userspace?**
**A:** Multiple mechanisms, used for different patterns: (1) **BPF_MAP_TYPE_HASH/ARRAY**: userspace uses `bpf_map_lookup_elem()` (via bpf syscall) to poll map values. Good for aggregated statistics (counters, histograms). Userspace reads when ready — no push notification. (2) **BPF_MAP_TYPE_PERF_EVENT_ARRAY**: per-CPU event buffers. eBPF program calls `bpf_perf_event_output(ctx, &map, BPF_F_CURRENT_CPU, &data, sizeof(data))`. Userspace reads using `perf_buffer__poll()`. Ordered per-CPU, but cross-CPU ordering not guaranteed. Can lose events if buffer full. (3) **BPF_MAP_TYPE_RINGBUF** (preferred since 5.8): single shared ring buffer. eBPF uses `bpf_ringbuf_reserve()`/`bpf_ringbuf_submit()`. Userspace uses `ring_buffer__poll()`. Advantages: globally ordered events, no per-CPU overhead, supports back-pressure (reserve fails when full, unlike perf_event which drops silently). More memory-efficient (one buffer vs N per-CPU buffers). (4) **bpf_printk()**: writes to tracefs trace_pipe. Only for debug — not for production (global trace buffer, contention, string formatting overhead). (5) **BPF iterators**: eBPF programs that iterate kernel data structures and output results sequentially via `bpf_seq_write()` — read as regular files from `/sys/fs/bpf/`. Best practice: use ringbuf for event streams, hash/array maps for aggregations, perf_event_array only if you need per-CPU isolation.

**Q2: What are per-CPU maps and when should you use them?**
**A:** Per-CPU maps (`BPF_MAP_TYPE_PERCPU_HASH`, `BPF_MAP_TYPE_PERCPU_ARRAY`) maintain a separate copy of each value for each CPU core. When an eBPF program writes to key K, it only writes to CPU N's copy — no lock contention, no cache bouncing. Use per-CPU maps when: (1) **High-frequency counters**: each CPU increments its own counter → userspace sums all CPUs at read time. No atomic operations needed. (2) **Temporary scratch space**: per-CPU arrays as scratch buffers for eBPF programs (avoids stack size limit of 512 bytes). (3) **Histograms**: each CPU maintains its own histogram buckets → merged at read time. When NOT to use per-CPU maps: (1) **Global state**: if you need program A on CPU 0 to see updates from program B on CPU 1 (inter-CPU communication), use regular (shared) maps. (2) **Small number of entries**: per-CPU maps use N times more memory (N = number of CPUs). On a 128-core machine, a per-CPU array with 10K entries of 8 bytes = 10K * 8 * 128 = 10MB instead of 80KB. (3) **Event ordering**: per-CPU implies no global ordering — use ringbuf if order matters. In userspace, reading per-CPU maps returns an array of values (one per CPU) — you typically sum or aggregate them.

---

## Summary

- eBPF maps: shared data between programs and userspace (hash, array, ringbuf, per-CPU, etc.)
- Ring buffer (BPF_MAP_TYPE_RINGBUF): preferred for events — ordered, efficient, back-pressure
- Per-CPU maps: lockless contention-free counters/histograms — sum values in userspace
- Helpers: bpf_map_lookup/update, bpf_get_current_pid_tgid, bpf_ktime_get_ns, bpf_probe_read
- Map pinning: /sys/fs/bpf/ — persists maps beyond program lifetime for sharing
- Tail calls: chain eBPF programs (max 33 deep) for complex logic

---

[Previous: eBPF Architecture ←](Chapter_19_eBPF.md) | [Next: BCC and bpftrace →](Chapter_21_BCC_bpftrace.md)
