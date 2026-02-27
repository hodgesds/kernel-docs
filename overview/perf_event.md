# Linux Kernel Perf Event Subsystem

## Overview

The perf_event subsystem (`kernel/events/`) provides a unified interface for
performance monitoring and observability in the Linux kernel. It abstracts
hardware Performance Monitoring Units (PMUs), software counters, tracepoints,
and breakpoint events behind a single file-descriptor-based API exposed through
the `perf_event_open()` system call. Events can operate in counting mode
(accumulate a counter value) or sampling mode (write records into a memory-mapped
ring buffer on overflow).

The subsystem is the kernel-side foundation for the `perf` tool, and serves as
a primary attachment point for BPF programs that need hardware counter access,
tracepoint-driven sampling, or a high-performance channel to stream data to
user space.

### Key Source Files

| File | Purpose |
|------|---------|
| `kernel/events/core.c` | Core subsystem: syscall, scheduling, overflow, ioctls (~15k lines) |
| `kernel/events/ring_buffer.c` | mmap ring buffer allocation and I/O |
| `kernel/events/internal.h` | `struct perf_buffer` and internal helpers |
| `include/linux/perf_event.h` | Kernel-internal structs (`perf_event`, `pmu`, etc.) |
| `include/uapi/linux/perf_event.h` | UAPI structs, enums, ioctl definitions |
| `kernel/trace/bpf_trace.c` | BPF helper implementations and attachment |
| `kernel/bpf/arraymap.c` | `BPF_MAP_TYPE_PERF_EVENT_ARRAY` map operations |
| `arch/x86/events/core.c` | x86 PMU driver (Intel/AMD hardware counters) |

---

## Architecture

### Object Hierarchy

Every perf event lives within a context, is serviced by a PMU, and produces
data through either direct reads or ring buffer writes.

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                     struct task_struct                           │
 │  (or per-CPU context when task == NULL)                          │
 │                                                                  │
 │  ┌────────────────────────────────────────────────────────────┐  │
 │  │  struct perf_event_context                                 │  │
 │  │  ┌──────────────────────────────────────────────────────┐  │  │
 │  │  │  pinned_groups   (rb-tree, PERF_EV_CAP_PINNED)       │  │  │
 │  │  │  flexible_groups (rb-tree, rotate on multiplex)      │  │  │
 │  │  └──────────────────────────────────────────────────────┘  │  │
 │  │                                                            │  │
 │  │  ┌──────────────────────────┐  ┌──────────────────────┐    │  │
 │  │  │ perf_event_pmu_context   │  │ perf_event_pmu_ctx   │    │  │
 │  │  │  (PMU A events)          │  │  (PMU B events)      │    │  │
 │  │  │  ┌────────────────────┐  │  │  ┌────────────────┐  │    │  │
 │  │  │  │ struct perf_event  │  │  │  │ struct perf_event │    │  │
 │  │  │  │  ├─ attr           │  │  │  │  ├─ attr      │   │    │  │
 │  │  │  │  ├─ hw (hw_perf)   │  │  │  │  ├─ hw        │   │    │  │
 │  │  │  │  ├─ pmu ──────────────┼──┼──┤  ├─ pmu       │   │    │  │
 │  │  │  │  ├─ rb (ring buf)  │  │  │  │  └─ rb        │   │    │  │
 │  │  │  │  └─ prog (BPF)     │  │  │  └───────────────┘   │    │  │
 │  │  │  └────────────────────┘  │  └──────────────────────┘    │  │
 │  │  └──────────────────────────┘                              │  │
 │  └────────────────────────────────────────────────────────────┘  │
 └──────────────────────────────────────────────────────────────────┘

 ┌──────────────────────────┐
 │  struct pmu              │   PMU driver vtable
 │  ├─ event_init()         │   (one per PMU type: cpu, software,
 │  ├─ add() / del()        │    tracepoint, breakpoint, kprobe, etc.)
 │  ├─ start() / stop()     │
 │  ├─ read()               │
 │  ├─ pmu_enable/disable() │
 │  └─ sched_task()         │
 └──────────────────────────┘
```

### Event State Machine

Defined in `include/linux/perf_event.h:679`:

```
                         perf_event_open()
                               │
                               ▼
 ┌─────────┐  enable   ┌────────────┐  sched_in   ┌────────┐
 │   OFF   │──────────▶│  INACTIVE  │────────────▶│ ACTIVE │
 │  (-1)   │◀──────────│    (0)     │◀────────────│  (1)   │
 └─────────┘  disable  └────────────┘  sched_out  └────────┘
      │                      ▲                         │
      │  error               │ enable                  │ error
      │                      │                         │
      ▼                ┌─────┴────┐                    │
 ┌─────────┐           │  ERROR   │◀───────────────────┘
 │  EXIT   │           │  (-2)    │  (scheduling failure)
 │  (-3)   │           └──────────┘
 │ (task   │
 │  died)  │
 └────┬────┘
      │               ┌──────────┐
      │               │ REVOKED  │  PMU hot-unplug
      │               │  (-4)    │
      │               └──────────┘
      ▼
 ┌─────────┐
 │  DEAD   │  final cleanup
 │  (-5)   │
 └─────────┘
```

State transitions:
- `sched_in`: `INACTIVE` → `ACTIVE` (or → `ERROR` on scheduling failure)
- `sched_out`: `ACTIVE` → `INACTIVE`
- `disable`: `{ACTIVE, INACTIVE}` → `OFF`
- `enable`: `{OFF, ERROR}` → `INACTIVE`
- Task exit: → `EXIT` (event persists for `inherit` readout)
- PMU removal: → `REVOKED`

---

## Key Data Structures

### struct perf_event_attr

`include/uapi/linux/perf_event.h:393` — The user-space-visible event specification,
passed to `perf_event_open()`. This structure is versioned via its `size` field
for forward and backward compatibility.

```c
struct perf_event_attr {
    __u32 type;           /* enum perf_type_id */
    __u32 size;           /* sizeof this struct, for compat */
    __u64 config;         /* type-specific configuration */

    union {
        __u64 sample_period;  /* sample every N events */
        __u64 sample_freq;    /* sample at N Hz (if freq=1) */
    };

    __u64 sample_type;    /* PERF_SAMPLE_* bitmask */
    __u64 read_format;    /* PERF_FORMAT_* bitmask */

    /* Bitfield flags (partial list): */
    __u64 disabled       :  1, /* off by default        */
          inherit        :  1, /* children inherit it   */
          pinned         :  1, /* must always be on PMU */
          exclusive      :  1, /* only group on PMU     */
          exclude_user   :  1, /* don't count user      */
          exclude_kernel :  1, /* don't count kernel    */
          exclude_hv     :  1, /* don't count hypervisor*/
          exclude_idle   :  1, /* don't count when idle */
          mmap           :  1, /* include mmap data     */
          freq           :  1, /* use freq, not period  */
          enable_on_exec :  1, /* enable on next exec   */
          task           :  1, /* trace fork/exit       */
          watermark      :  1, /* use wakeup_watermark  */
          precise_ip     :  2, /* skid constraint 0-3   */
          sample_id_all  :  1, /* sample_type on all    */
          context_switch :  1, /* context switch data   */
          sigtrap        :  1, /* send SIGTRAP on event */
          ...;

    union {
        __u32 wakeup_events;    /* wakeup every N events */
        __u32 wakeup_watermark; /* bytes before wakeup   */
    };

    __u64 config1;        /* extension of config   */
    __u64 config2;        /* extension of config1  */

    __u64 branch_sample_type;
    __u64 sample_regs_user;
    __u32 sample_stack_user;
    __s32 clockid;
    __u64 sample_regs_intr;
    __u32 aux_watermark;
    __u16 sample_max_stack;
    __u32 aux_sample_size;
    __u64 sig_data;       /* siginfo_t::si_perf_data */
    __u64 config3;
    __u64 config4;
};
```

Version size progression: `PERF_ATTR_SIZE_VER0` (64 bytes) through
`PERF_ATTR_SIZE_VER9` (144 bytes). The kernel zero-extends older structs and
silently truncates unknown future fields whose values are zero.

### struct perf_event

`include/linux/perf_event.h:764` — The primary kernel-internal representation
of a single event. One is allocated per `perf_event_open()` call.

```c
struct perf_event {
    /* Group linkage (protected by ctx->lock): */
    struct list_head       event_entry;
    struct list_head       sibling_list;
    struct rb_node         group_node;
    struct perf_event     *group_leader;
    int                    nr_siblings;

    struct pmu            *pmu;
    enum perf_event_state  state;
    local64_t              count;          /* accumulated count */

    u64                    total_time_enabled;
    u64                    total_time_running;

    struct perf_event_attr attr;           /* user specification */
    struct hw_perf_event   hw;             /* PMU-private state */

    struct perf_event_context     *ctx;    /* owning context */
    struct perf_event_pmu_context *pmu_ctx;

    /* Ring buffer: */
    struct perf_buffer    *rb;

    /* Overflow handler and BPF: */
    perf_overflow_handler_t  overflow_handler;
    struct bpf_prog         *prog;         /* attached BPF program */
    u64                      bpf_cookie;

    int  oncpu;           /* CPU currently executing on (-1 if not) */
    int  cpu;             /* target CPU (-1 = any) */

    struct task_struct    *owner;
    struct perf_event     *parent;         /* for inherited events */
    struct list_head       child_list;

    u64  id;              /* unique event ID */
    u64 (*clock)(void);

    /* Tracepoint / filter: */
    struct trace_event_call *tp_event;
    struct event_filter     *filter;
};
```

### struct perf_event_context

`include/linux/perf_event.h:1007` — Container for a set of events, either
per-task (`ctx->task != NULL`) or per-CPU (`ctx->task == NULL`).

```c
struct perf_event_context {
    raw_spinlock_t lock;       /* IRQ-safe; protects event list, nr_active */
    struct mutex   mutex;      /* sleepable; protects slow-path modifications */

    struct list_head          pmu_ctx_list;
    struct perf_event_groups  pinned_groups;    /* rb-tree */
    struct perf_event_groups  flexible_groups;  /* rb-tree */
    struct list_head          event_list;

    int  nr_events;
    int  is_active;
    int  nr_freq;
    int  rotate_disable;

    struct task_struct *task;   /* NULL for per-CPU contexts */
    u64  time;                  /* context clock */
    u64  timestamp;

    struct perf_event_context *parent_ctx;
    u64  parent_gen;
    u64  generation;

    local_t nr_no_switch_fast; /* events preventing fast context switch */
};
```

Locking discipline:
- **Modify** the event list: hold **both** `ctx->mutex` and `ctx->lock`.
- **Read** the event list: hold **either** `ctx->mutex` or `ctx->lock`.
- `ctx->lock` is used by the scheduler path (hardirq context).
- `ctx->mutex` is used by `perf_event_open()`, ioctls, and other slow paths.

### struct pmu

`include/linux/perf_event.h:329` — The PMU driver vtable. Each PMU type
(hardware counters, software counters, tracepoints, breakpoints, kprobes)
registers one instance.

```c
struct pmu {
    struct list_head  entry;
    const char       *name;
    int               type;           /* PMU type ID (in pmu_idr) */
    int               capabilities;   /* PERF_PMU_CAP_* flags */

    void (*pmu_enable)(struct pmu *pmu);
    void (*pmu_disable)(struct pmu *pmu);

    int  (*event_init)(struct perf_event *event);

    int  (*add)(struct perf_event *event, int flags);   /* schedule onto HW */
    void (*del)(struct perf_event *event, int flags);   /* remove from HW */
    void (*start)(struct perf_event *event, int flags); /* start counting */
    void (*stop)(struct perf_event *event, int flags);  /* stop counting */
    void (*read)(struct perf_event *event);             /* read HW counter */

    /* Group scheduling: */
    void (*start_txn)(struct pmu *pmu, unsigned int txn_flags);
    int  (*commit_txn)(struct pmu *pmu);
    void (*cancel_txn)(struct pmu *pmu);

    /* Context switch notification: */
    void (*sched_task)(struct perf_event_pmu_context *pmu_ctx,
                       struct task_struct *task, bool sched_in);

    /* AUX area (Intel PT, ARM CoreSight): */
    void *(*setup_aux)(struct perf_event *event, void **pages,
                       int nr_pages, bool overwrite);
    void (*free_aux)(void *aux);
};
```

Key `PERF_EF_*` flags passed to `add`/`del`/`start`/`stop`:

| Flag | Value | Meaning |
|------|-------|---------|
| `PERF_EF_START` | 0x01 | Start counting immediately when adding |
| `PERF_EF_RELOAD` | 0x02 | Reload counter value when starting |
| `PERF_EF_UPDATE` | 0x04 | Read and update count when stopping |

### struct hw_perf_event

`include/linux/perf_event.h:147` — PMU-private hardware state embedded in
every `perf_event` as `event->hw`. Uses a union for different event types.

```c
struct hw_perf_event {
    union {
        struct { /* hardware PMU */
            u64  config;
            u64  last_tag;
            int  idx;              /* hardware counter index */
            int  last_cpu;
            int  flags;
        };
        struct { /* software */
            struct hrtimer hrtimer;
        };
        struct { /* tracepoint */
            struct list_head tp_list;
        };
        struct { /* breakpoint */
            struct arch_hw_breakpoint info;
        };
    };

    struct task_struct *target;
    int       state;               /* PERF_HES_STOPPED | PERF_HES_UPTODATE */
    local64_t prev_count;          /* last observed hardware value */
    u64       sample_period;

    union {
        struct {
            u64       last_period;
            local64_t period_left;
        };
        struct { /* topdown */
            u64 saved_metric;
            u64 saved_slots;
        };
    };

    u64 interrupts_seq;            /* throttle state */
    u64 interrupts;
    u64 freq_time_stamp;
    u64 freq_count_stamp;
};
```

### struct perf_buffer

`kernel/events/internal.h:13` — The ring buffer backing `mmap()`. One is
allocated per ring buffer and shared across events via `PERF_EVENT_IOC_SET_OUTPUT`.

```c
struct perf_buffer {
    refcount_t refcount;
    int        nr_pages;      /* data pages (power-of-2) */
    int        overwrite;     /* 1 = overwrite mode (kernel ignores tail) */
    int        paused;

    atomic_t   poll;          /* EPOLLIN readiness */
    local_t    head;          /* kernel write position (lock-free) */
    unsigned int nest;        /* re-entrancy depth (NMI safety) */
    local_t    events;        /* event count for wakeup */
    local_t    wakeup;
    local_t    lost;          /* dropped record count */

    long       watermark;     /* wakeup threshold in bytes */
    spinlock_t event_lock;
    struct list_head event_list;

    struct perf_event_mmap_page *user_page; /* mmap'd control page */
    void *data_pages[];                     /* flexible array of page pointers */
};
```

---

## Event Types

`include/uapi/linux/perf_event.h:29` — The `type` field of `perf_event_attr`
selects the event class:

| Type | Value | Description |
|------|-------|-------------|
| `PERF_TYPE_HARDWARE` | 0 | CPU hardware counter (cycles, instructions, etc.) |
| `PERF_TYPE_SOFTWARE` | 1 | Kernel software counter (page faults, context switches) |
| `PERF_TYPE_TRACEPOINT` | 2 | Kernel tracepoint (config = tracepoint ID from tracefs) |
| `PERF_TYPE_HW_CACHE` | 3 | Hardware cache event (encoded cache/op/result triple) |
| `PERF_TYPE_RAW` | 4 | Raw PMU event code (architecture-specific) |
| `PERF_TYPE_BREAKPOINT` | 5 | Hardware breakpoint (data/instruction watchpoint) |

Types ≥ `PERF_TYPE_MAX` are dynamically assigned PMU type IDs (e.g., Intel PT,
uncore PMUs). Userspace discovers these via `/sys/bus/event_source/devices/`.

### Hardware Event IDs

`enum perf_hw_id` — Generic hardware events mapped by each PMU driver:

| ID | Value | Event |
|----|-------|-------|
| `PERF_COUNT_HW_CPU_CYCLES` | 0 | CPU cycles |
| `PERF_COUNT_HW_INSTRUCTIONS` | 1 | Retired instructions |
| `PERF_COUNT_HW_CACHE_REFERENCES` | 2 | Cache references |
| `PERF_COUNT_HW_CACHE_MISSES` | 3 | Cache misses |
| `PERF_COUNT_HW_BRANCH_INSTRUCTIONS` | 4 | Branch instructions |
| `PERF_COUNT_HW_BRANCH_MISSES` | 5 | Branch mispredictions |
| `PERF_COUNT_HW_BUS_CYCLES` | 6 | Bus cycles |
| `PERF_COUNT_HW_STALLED_CYCLES_FRONTEND` | 7 | Front-end stalls |
| `PERF_COUNT_HW_STALLED_CYCLES_BACKEND` | 8 | Back-end stalls |
| `PERF_COUNT_HW_REF_CPU_CYCLES` | 9 | Reference cycles (unscaled) |

### Software Event IDs

`enum perf_sw_ids` — Kernel-maintained software counters:

| ID | Value | Event |
|----|-------|-------|
| `PERF_COUNT_SW_CPU_CLOCK` | 0 | CPU clock (hrtimer-based) |
| `PERF_COUNT_SW_TASK_CLOCK` | 1 | Task clock (per-task CPU time) |
| `PERF_COUNT_SW_PAGE_FAULTS` | 2 | Page faults (major + minor) |
| `PERF_COUNT_SW_CONTEXT_SWITCHES` | 3 | Context switches |
| `PERF_COUNT_SW_CPU_MIGRATIONS` | 4 | CPU migrations |
| `PERF_COUNT_SW_PAGE_FAULTS_MIN` | 5 | Minor page faults |
| `PERF_COUNT_SW_PAGE_FAULTS_MAJ` | 6 | Major page faults |
| `PERF_COUNT_SW_ALIGNMENT_FAULTS` | 7 | Alignment faults |
| `PERF_COUNT_SW_EMULATION_FAULTS` | 8 | Instruction emulation faults |
| `PERF_COUNT_SW_DUMMY` | 9 | Dummy event (no counting, for mmap/BPF only) |
| `PERF_COUNT_SW_BPF_OUTPUT` | 10 | Used by `bpf_perf_event_output()` |
| `PERF_COUNT_SW_CGROUP_SWITCHES` | 11 | Cgroup switches |

---

## The perf_event_open Syscall

`kernel/events/core.c:13471`

```c
int perf_event_open(struct perf_event_attr *attr_uptr,
                    pid_t pid, int cpu,
                    int group_fd, unsigned long flags);
```

**Parameters:**

| Parameter | Meaning |
|-----------|---------|
| `attr` | Event specification (type, config, sample_type, etc.) |
| `pid` | `0` = calling process; `> 0` = target process; `-1` = per-CPU |
| `cpu` | CPU index (`-1` = any CPU, requires `pid >= 0`) |
| `group_fd` | `-1` = new group leader; `>= 0` = join existing group |
| `flags` | `PERF_FLAG_FD_CLOEXEC`, `PERF_FLAG_PID_CGROUP`, etc. |

**Returns:** a file descriptor on success, or a negative error code.

### Execution Flow

```
perf_event_open()                                    core.c:13471
 1. perf_copy_attr()         copy + validate attr from userspace
 2. security_perf_event_open() LSM permission check
 3. perf_allow_kernel()      paranoid check if !exclude_kernel
 4. find_get_context()       look up or create perf_event_context for pid/cpu
 5. perf_event_alloc()       allocate struct perf_event
 6.   perf_init_event()      find PMU, call pmu->event_init()
 7. group validation         verify group_fd leader compatibility
 8. anon_inode_getfile()     create anonymous file for the event fd
 9. perf_install_in_context() add event to context, may IPI target CPU
10. fd_install()             make fd visible to userspace
```

### PMU Discovery via perf_init_event()

`kernel/events/core.c:12752` — Finds the PMU that claims the event:

1. If inheriting from a parent, try the parent's PMU first.
2. For `PERF_TYPE_HARDWARE` / `PERF_TYPE_HW_CACHE`: extract PMU type from upper
   32 bits of `attr.config` and look up via `pmu_idr`.
3. Fall back to a linear scan of the `pmus` list, calling each PMU's
   `event_init()` until one returns 0 (success).
4. Returns `ERR_PTR(-ENOENT)` if no PMU claims the event.

---

## PMU Abstraction Layer

### Registration

PMU drivers register via `perf_pmu_register(pmu, name, type)` in
`kernel/events/core.c`. Each registered PMU receives a type ID stored in the
global `pmu_idr` and is added to the `pmus` list. Sysfs entries appear under
`/sys/bus/event_source/devices/<name>/`.

Built-in PMUs:
- `perf_swevent` — software events (context switches, page faults)
- `perf_cpu_clock` / `perf_task_clock` — clock-based software PMUs
- `perf_tracepoint` — tracepoint events
- `perf_breakpoint` — hardware breakpoints
- Architecture-specific hardware PMUs (e.g., `x86_pmu` in `arch/x86/events/`)

### Operation Lifecycle

During event scheduling, the core calls PMU operations in a specific sequence
with IRQs disabled:

```
pmu->pmu_disable()          // disable PMU-wide (batch begin)
  pmu->add(event, PERF_EF_START)   // schedule event onto counter
  pmu->start(event, PERF_EF_RELOAD) // (re)start counting
  ...
  pmu->stop(event, PERF_EF_UPDATE)  // stop, read counter
  pmu->del(event, 0)                // remove from counter
pmu->pmu_enable()           // re-enable PMU (batch commit)
```

### Transaction Support

When scheduling an event group, the core uses PMU transactions to test whether
all group members can be scheduled simultaneously:

```c
pmu->start_txn(pmu, PERF_PMU_TXN_ADD);
for each event in group:
    pmu->add(event, PERF_EF_START);
if (pmu->commit_txn(pmu))  // hardware has enough counters?
    success;
else
    pmu->cancel_txn(pmu);  // roll back, group cannot be scheduled
```

---

## Counting vs Sampling

### Counting Mode

In counting mode, the event accumulates a value in `event->count`. Userspace
reads it via:

1. **`read(fd)`** — Returns the count and optional `time_enabled`/`time_running`
   scaling values. For group reads (`PERF_FORMAT_GROUP`), returns all sibling
   counts atomically.

2. **RDPMC fast path** — On x86, when `perf_event_mmap_page.cap_user_rdpmc` is
   set, userspace can read the hardware counter directly via the `RDPMC`
   instruction without a syscall. The mmap page provides `index` and `offset`
   for self-monitoring:

```c
/* Userspace self-monitoring loop: */
u32 seq, idx;
u64 count;
do {
    seq = READ_ONCE(mmap_page->lock);   /* even = unlocked */
    barrier();
    idx = mmap_page->index;
    count = mmap_page->offset;
    if (idx)
        count += rdpmc(idx - 1);        /* read HW counter */
    barrier();
} while (mmap_page->lock != seq);
```

3. **Scaling** — When events are multiplexed, the true count is estimated:
   `scaled_count = raw_count * time_enabled / time_running`.

### Sampling Mode

When `attr.sample_period` or `attr.sample_freq` is set, the event triggers a
sample on counter overflow. The PMU generates a Performance Monitoring
Interrupt (PMI), and the overflow handler writes a `PERF_RECORD_SAMPLE` record
into the ring buffer:

```
Hardware counter overflow
  → PMI / NMI
    → perf_event_overflow()                 core.c:10483
      → __perf_event_overflow()             core.c:10391
        → event->overflow_handler()         (perf_event_output_forward)
          → perf_output_begin()             ring_buffer.c
          → perf_output_sample()            writes PERF_RECORD_SAMPLE
          → perf_output_end()               publishes data_head, wakes poll
```

The `PERF_SAMPLE_*` bitmask in `attr.sample_type` controls which fields are
included in each sample record (IP, TID, time, callchain, registers, etc.).

### Frequency Mode

When `attr.freq = 1`, the kernel dynamically adjusts `sample_period` to
maintain the requested `sample_freq` (in Hz). The adjustment algorithm runs in
`perf_swevent_hrtimer()` and `perf_adjust_period()`: it measures the actual
event rate over a time window and scales the period proportionally. This trades
deterministic sampling for a predictable overhead budget.

---

## Ring Buffer / mmap Interface

### struct perf_event_mmap_page

`include/uapi/linux/perf_event.h:596` — The first page of the mmap region is
a control page shared between kernel and userspace:

```c
struct perf_event_mmap_page {
    __u32 version;
    __u32 compat_version;

    /* Userspace counter read (seqlock): */
    __u32 lock;              /* even = consistent */
    __u32 index;             /* HW counter index (0 = stopped) */
    __s64 offset;            /* add to RDPMC result */
    __u64 time_enabled;
    __u64 time_running;

    /* Capabilities: */
    __u64 cap_user_rdpmc  : 1;  /* RDPMC available */
    __u64 cap_user_time   : 1;  /* TSC conversion valid */
    ...;

    /* TSC → nanosecond conversion: */
    __u16 pmc_width;
    __u16 time_shift;
    __u32 time_mult;
    __u64 time_offset;
    __u64 time_zero;

    /* Ring buffer control: */
    __u64 data_head;    /* kernel write position (read with smp_rmb()) */
    __u64 data_tail;    /* userspace read position (write with smp_mb()) */
    __u64 data_offset;  /* offset of data ring from mmap start */
    __u64 data_size;    /* size of data region */

    /* AUX area (for Intel PT, etc.): */
    __u64 aux_head;
    __u64 aux_tail;
    __u64 aux_offset;
    __u64 aux_size;
};
```

### Producer/Consumer Protocol

The ring buffer uses a single-producer (kernel), single-consumer (userspace)
model with explicit memory barriers:

```
Kernel (producer):                     Userspace (consumer):

  reserve space via                      read data_head with
  local_cmpxchg(&rb->head)              smp_rmb() after

  write sample data                     process records from
                                        data_tail to data_head

  perf_output_end():                    advance data_tail with
  smp_wmb()                             smp_mb() before write
  WRITE_ONCE(data_head, head)
  wake_up(waitq)                        poll()/epoll() for EPOLLIN
```

In overwrite mode (`rb->overwrite = 1`), the kernel does not check `data_tail`
and will overwrite old records. Userspace has no flow control.

### Write Path

`kernel/events/ring_buffer.c`:

1. **`perf_output_begin()`** — Reserves `size` bytes using
   `local_try_cmpxchg(&rb->head)` (lock-free). Returns `-ENOSPC` if the buffer
   is full (increments `rb->lost`). Automatically prepends a `PERF_RECORD_LOST`
   record if events were dropped.

2. **`perf_output_sample()`** — Writes the `perf_event_header` and sample
   fields based on `sample_type`.

3. **`perf_output_end()`** — Commits the write: `smp_wmb()`, publishes
   `data_head` to the user page, decrements `rb->nest`. Wakes pollers when the
   watermark threshold is crossed.

### Record Types

`enum perf_event_type` — Each ring buffer record starts with a
`perf_event_header`:

```c
struct perf_event_header {
    __u32 type;   /* PERF_RECORD_* */
    __u16 misc;   /* PERF_RECORD_MISC_* cpumode, flags */
    __u16 size;   /* total record size in bytes */
};
```

| Record | Value | Description |
|--------|-------|-------------|
| `PERF_RECORD_MMAP` | 1 | Executable mapping (for symbol resolution) |
| `PERF_RECORD_LOST` | 2 | Count of dropped records |
| `PERF_RECORD_COMM` | 3 | Process name change |
| `PERF_RECORD_EXIT` | 4 | Process exit |
| `PERF_RECORD_THROTTLE` | 5 | Sampling throttled |
| `PERF_RECORD_UNTHROTTLE` | 6 | Sampling resumed |
| `PERF_RECORD_FORK` | 7 | Process fork |
| `PERF_RECORD_READ` | 8 | Counter group read |
| `PERF_RECORD_SAMPLE` | 9 | Sample record (main data record) |
| `PERF_RECORD_MMAP2` | 10 | Extended mmap with inode/build-id |
| `PERF_RECORD_AUX` | 11 | AUX area data available |
| `PERF_RECORD_SWITCH` | 14 | Context switch |
| `PERF_RECORD_SWITCH_CPU_WIDE` | 15 | CPU-wide context switch with prev/next pid |
| `PERF_RECORD_NAMESPACES` | 16 | Namespace information |
| `PERF_RECORD_KSYMBOL` | 17 | Kernel symbol register/unregister |
| `PERF_RECORD_BPF_EVENT` | 18 | BPF program load/unload notification |
| `PERF_RECORD_CGROUP` | 19 | Cgroup creation |
| `PERF_RECORD_TEXT_POKE` | 20 | Kernel text modification |

---

## Event Groups and Multiplexing

### Group Creation

Events are grouped by passing the leader's fd as `group_fd` to
`perf_event_open()`. The leader is always read/scheduled first. Group members
are linked via `sibling_list` and share the leader's scheduling fate — either
all members are on the PMU simultaneously, or none are.

### Pinned vs Flexible Groups

Events are organized in the context's rb-trees:

- **Pinned groups** (`attr.pinned = 1`): Must always be scheduled. If the PMU
  lacks counters, the group enters `ERROR` state rather than waiting. Scheduled
  first.
- **Flexible groups**: Best-effort scheduling. Rotated among available counters
  via multiplexing when the PMU is oversubscribed.

### Multiplexing and Rotation

When more events exist than hardware counters, the kernel multiplexes flexible
groups using an hrtimer. On each tick (`perf_event_mux_interval_ms`, default
1ms), `perf_rotate_context()` removes the current set of flexible events and
installs the next set from the rb-tree.

### Scaling

Because multiplexed events do not run continuously, each event tracks:
- `total_time_enabled` — wall time since the event was enabled
- `total_time_running` — time the event was actually on a counter

Userspace computes the estimated true count:

```
estimated_count = raw_count × (time_enabled / time_running)
```

These values are available via `read()` (with `PERF_FORMAT_TOTAL_TIME_ENABLED`
and `PERF_FORMAT_TOTAL_TIME_RUNNING`) and in the mmap control page.

---

## Scheduler Integration

### Context Switch Hooks

The core scheduler calls perf_event hooks on every context switch:

**`__perf_event_task_sched_out()`** — `kernel/events/core.c:3837`

Called when `task` is being descheduled:
1. `perf_pmu_sched_task(task, next, false)` — invokes `pmu->sched_task()` for
   PMUs that registered the callback (e.g., Intel LBR save).
2. `perf_event_switch()` — emits `PERF_RECORD_SWITCH` records.
3. `perf_event_context_sched_out()` — stops and saves the task's events from
   hardware counters.
4. `perf_cgroup_switch()` — handles cgroup perf event transitions.

**`__perf_event_task_sched_in()`** — `kernel/events/core.c:4174`

Called when `task` is being scheduled in:
1. `perf_event_context_sched_in()` — programs the task's events into hardware.
2. `perf_event_switch()` — emits switch-in records.
3. `perf_pmu_sched_task(prev, task, true)` — PMU sched_task callback.

### Clone Context Optimization

When two tasks share the same `perf_event_context` (detected via
`ctx->generation` matching), the scheduler can use a fast path that swaps
contexts without fully removing and re-adding all events. The
`nr_no_switch_fast` counter tracks events that prevent this optimization
(e.g., events with address filters or AUX output).

---

## Overflow and Throttling

### PMI Path

`kernel/events/core.c:10391` — When a hardware counter overflows, the
architecture-specific PMI handler calls into:

```c
static int __perf_event_overflow(struct perf_event *event,
                                  int throttle,
                                  struct perf_sample_data *data,
                                  struct pt_regs *regs)
```

Sequence:
1. `__perf_event_account_interrupt()` — Increments `hw.interrupts`. If the
   interrupt rate exceeds `max_samples_per_tick`, reduces
   `sysctl_perf_event_sample_rate` globally.
2. If `event->prog` is set, calls `bpf_overflow_handler()`.
3. If `attr.sigtrap`, queues a `task_work` to deliver SIGTRAP on return to
   userspace, with `si_perf_data` set from `attr.sig_data`.
4. Calls `event->overflow_handler()` — typically `perf_event_output_forward()`,
   which writes a `PERF_RECORD_SAMPLE` into the ring buffer.

### Throttle Mechanism

The kernel throttles high-frequency sampling to prevent PMI storms from
locking up the system:

- **`sysctl_perf_event_sample_rate`** — Maximum samples/sec across all events
  (default: 100,000 Hz). Dynamically reduced when interrupt accounting detects
  overload.
- **`perf_cpu_time_max_percent`** — Maximum CPU time percentage spent handling
  perf interrupts (default: 25%). When exceeded, events are throttled and
  `PERF_RECORD_THROTTLE` records are emitted.
- Per-event throttle: if an event generates more than `max_samples_per_tick`
  interrupts in one scheduler tick, it is disabled until the next tick
  (`PERF_RECORD_UNTHROTTLE`).

### SIGTRAP Delivery

When `attr.sigtrap = 1`, counter overflow triggers a SIGTRAP signal to the
monitored task instead of (or in addition to) writing a sample record. The
signal carries `si_perf_data` from `attr.sig_data`, allowing userspace to
identify which event triggered the signal. This is used for hardware
breakpoint-like behavior on PMU events.

---

## BPF Integration

The perf_event subsystem is one of BPF's primary integration points, serving
both as an attachment target for BPF programs and as a data channel from BPF
programs to userspace.

### BPF Program Types for Perf Events

Three BPF program types interact with perf events:

| Program Type | Attach To | Context Struct |
|-------------|-----------|----------------|
| `BPF_PROG_TYPE_PERF_EVENT` | Hardware/software PMU events | `struct bpf_perf_event_data` |
| `BPF_PROG_TYPE_KPROBE` | kprobe/uprobe perf events | `struct pt_regs` |
| `BPF_PROG_TYPE_TRACEPOINT` | Tracepoint perf events | Tracepoint-specific struct |

### Attaching BPF Programs to Perf Events

`kernel/events/core.c:11293` — `perf_event_set_bpf_prog()` handles attachment
via two paths depending on event type:

**Non-tracing path** (hardware/software PMU events):
- Requires `BPF_PROG_TYPE_PERF_EVENT`.
- Sets `event->prog` and installs `bpf_overflow_handler` as the overflow
  handler.
- On each counter overflow, the BPF program runs with a `bpf_perf_event_data`
  context providing access to the sample data and registers.

**Tracing path** (kprobe/uprobe/tracepoint events):
- Requires matching program type (`BPF_PROG_TYPE_KPROBE` for kprobes/uprobes,
  `BPF_PROG_TYPE_TRACEPOINT` for tracepoints).
- Calls `perf_event_attach_bpf_prog()` in `kernel/trace/bpf_trace.c`.
- Multiple BPF programs can attach to the same tracepoint event.

**Attachment methods:**
1. `ioctl(perf_fd, PERF_EVENT_IOC_SET_BPF, bpf_fd)` — Legacy path.
2. `bpf(BPF_LINK_CREATE)` with `target_fd = perf_fd` — Modern BPF link API,
   provides lifecycle management and `bpf_cookie` support.

### bpf_overflow_handler()

`kernel/events/core.c:10294` — Called from `__perf_event_overflow()` when
`event->prog` is set:

```c
static int bpf_overflow_handler(struct perf_event *event,
                                 struct perf_sample_data *data,
                                 struct pt_regs *regs)
```

Builds a `bpf_perf_event_data_kern` context:
```c
struct bpf_perf_event_data_kern ctx = {
    .data  = data,    /* struct perf_sample_data * */
    .event = event,
    .regs  = perf_arch_bpf_user_pt_regs(regs),
};
```

Runs the BPF program via `bpf_prog_run(prog, &ctx)`. Guards against
re-entrancy with `bpf_prog_active` per-CPU counter. If the BPF program returns
0, the default ring buffer output is suppressed.

### bpf_perf_event_output() Helper

`kernel/trace/bpf_trace.c:657` — Allows BPF programs to emit arbitrary data
into a perf ring buffer, delivering it to userspace consumers.

```
BPF program                    Kernel                        Userspace
─────────────────────────────────────────────────────────────────────────
bpf_perf_event_output(         Look up perf_event from       poll()/
  ctx,                         PERF_EVENT_ARRAY[cpu]         epoll_wait()
  map,                            │                             │
  BPF_F_CURRENT_CPU,              ▼                             │
  &data,                       Validate event type ==           │
  sizeof(data))                SW/BPF_OUTPUT, oncpu == cpu      │
                                  │                             │
                                  ▼                             │
                               perf_event_output()              │
                                  │                             │
                                  ▼                             │
                               perf_output_begin()  ──────────▶ │
                               write PERF_RECORD_SAMPLE         │
                               perf_output_end()                │
                                  │                             ▼
                                  └────────────────────▶ read(mmap ring)
```

**BPF_MAP_TYPE_PERF_EVENT_ARRAY** (`kernel/bpf/arraymap.c:1247`):
- Per-CPU array where each slot holds a reference to a `PERF_COUNT_SW_BPF_OUTPUT`
  perf event.
- Userspace creates one perf event per CPU, stores the fd in the map via
  `bpf_map_update_elem()`, then `mmap()`s each event's ring buffer.
- `perf_event_fd_array_get_ptr()` validates the perf event via
  `perf_event_read_local()` and creates a `bpf_event_entry` linking the perf
  file and map file.
- Entries are freed after an RCU grace period via `perf_event_fd_array_put_ptr()`.

**Nesting safety**: `bpf_perf_event_output()` uses per-CPU
`bpf_trace_sds` arrays with 3 nesting slots (normal context, IRQ, NMI),
allowing the helper to be called from any interrupt context without
data corruption.

### Reading Hardware PMCs from BPF

Two BPF helpers read hardware performance counters from within BPF programs:

**`bpf_perf_event_read(map, flags)`** — `kernel/trace/bpf_trace.c:559`

Reads a hardware counter value from a `BPF_MAP_TYPE_PERF_EVENT_ARRAY` entry.
Uses `BPF_F_CURRENT_CPU` or an explicit CPU index. Internally calls
`perf_event_read_local()`. Returns the raw counter value, but negative error
codes overlap with small positive counter values (known limitation).

**`bpf_perf_event_read_value(map, flags, buf, size)`** —
`kernel/trace/bpf_trace.c:582`

Improved version that fills a `struct bpf_perf_event_value`:
```c
struct bpf_perf_event_value {
    __u64 counter;   /* current count */
    __u64 enabled;   /* time_enabled (ns) */
    __u64 running;   /* time_running (ns) */
};
```

This avoids the sign ambiguity of `bpf_perf_event_read()` and provides scaling
information.

**`perf_event_read_local()`** — `include/linux/perf_event.h` — NMI-safe fast
path that reads the counter without IPIs. Returns `-EOPNOTSUPP` for inherited
events, `-EBUSY` if the event is not on the current CPU.

### Perf Ring Buffer vs BPF Ring Buffer

The kernel provides two ring buffer mechanisms for BPF-to-userspace data
transfer. The BPF ring buffer (`kernel/bpf/ringbuf.c`) was introduced in
Linux 5.8 as a purpose-built alternative.

| Feature | Perf Ring Buffer | BPF Ring Buffer |
|---------|-----------------|-----------------|
| Topology | Per-CPU (one ring per CPU) | Single shared ring |
| Map type | `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | `BPF_MAP_TYPE_RINGBUF` |
| Ordering | Per-CPU ordering only | Global ordering (single producer lock) |
| Zero-copy | No (kernel copies sample data) | Yes (reserve/commit API) |
| Overwrite mode | Yes (`write_backward`) | No |
| Variable-length records | Yes | Yes |
| Wakeup control | Watermark or N events | Watermark or manual notification |
| Backpressure | Drops with `PERF_RECORD_LOST` | Returns `-EAGAIN` on reserve |
| Memory overhead | N × ring_size (N = nr_cpus) | Single ring_size |
| Kernel version | 4.4+ | 5.8+ |

**When to use which:**
- Use **perf ring buffer** when: you need per-CPU isolation (no cross-CPU
  contention), overwrite mode, or compatibility with older kernels.
- Use **BPF ring buffer** when: you need global event ordering, zero-copy
  output, or want to minimize total memory usage.

### Tooling: How bpftrace and BCC Use Perf Events

End-to-end flow for a kprobe-based BPF tool:

```
bpftrace 'kprobe:do_sys_open { printf("open: %s\n", str(arg1)); }'

1. Create perf_event:
   perf_event_open(type=TRACEPOINT/kprobe, config=..., pid=-1, cpu=N)
   → returns perf_fd

2. Attach BPF program:
   bpf(BPF_PROG_LOAD, type=BPF_PROG_TYPE_KPROBE, ...)
   → returns prog_fd
   ioctl(perf_fd, PERF_EVENT_IOC_SET_BPF, prog_fd)
   ioctl(perf_fd, PERF_EVENT_IOC_ENABLE)

3. Set up output channel:
   bpf(BPF_MAP_CREATE, type=BPF_MAP_TYPE_PERF_EVENT_ARRAY, ...)
   For each CPU:
     perf_event_open(type=SOFTWARE, config=BPF_OUTPUT, cpu=N)
     mmap(perf_fd, 1 + 2^n pages)   // control page + data ring
     bpf_map_update_elem(map, &cpu, &perf_fd)

4. On kprobe hit:
   BPF program runs → bpf_perf_event_output(ctx, map, BPF_F_CURRENT_CPU,
                                              &data, sizeof(data))
   → perf_event_output() → writes PERF_RECORD_SAMPLE to ring buffer

5. Userspace poll loop:
   epoll_wait() on perf fds → read samples from mmap rings → print output
```

---

## Ioctls Reference

`include/uapi/linux/perf_event.h:576` — Operations on perf event file
descriptors:

| Ioctl | Argument | Description |
|-------|----------|-------------|
| `PERF_EVENT_IOC_ENABLE` | `0` or `PERF_IOC_FLAG_GROUP` | Enable event (or entire group) |
| `PERF_EVENT_IOC_DISABLE` | `0` or `PERF_IOC_FLAG_GROUP` | Disable event (or entire group) |
| `PERF_EVENT_IOC_REFRESH` | `int count` | Enable for `count` overflows, then disable |
| `PERF_EVENT_IOC_RESET` | `0` or `PERF_IOC_FLAG_GROUP` | Reset counter to zero |
| `PERF_EVENT_IOC_PERIOD` | `__u64 *period` | Set new sample period |
| `PERF_EVENT_IOC_SET_OUTPUT` | `int fd` | Redirect output to another event's ring buffer |
| `PERF_EVENT_IOC_SET_FILTER` | `char *filter` | Set tracepoint filter expression |
| `PERF_EVENT_IOC_ID` | `__u64 *id` | Read event's unique ID |
| `PERF_EVENT_IOC_SET_BPF` | `__u32 bpf_fd` | Attach BPF program to event |
| `PERF_EVENT_IOC_PAUSE_OUTPUT` | `__u32 pause` | Pause/resume ring buffer output |
| `PERF_EVENT_IOC_QUERY_BPF` | `struct perf_event_query_bpf *` | Query attached BPF program IDs |
| `PERF_EVENT_IOC_MODIFY_ATTRIBUTES` | `struct perf_event_attr *` | Modify event attributes in place |

---

## Global Sysctls

`kernel/events/core.c:468`

| Sysctl | Default | Description |
|--------|---------|-------------|
| `kernel.perf_event_paranoid` | 2 | Access control level: `-1` = unrestricted; `0` = disallow raw tracepoint for unpriv; `1` = disallow CPU events for unpriv; `2` = disallow kernel profiling for unpriv |
| `kernel.perf_event_max_sample_rate` | 100000 | Maximum samples/sec system-wide; dynamically reduced under overload |
| `kernel.perf_cpu_time_max_percent` | 25 | Maximum CPU% for perf interrupt handling; 0 = unlimited, 100 = disable throttling |
| `kernel.perf_event_max_contexts_per_stack` | 8 | Maximum nested contexts in callchain capture |
| `kernel.perf_event_mlock_kb` | 516 | Per-user mlock limit for perf ring buffers (KB) |
| `kernel.perf_event_max_stack` | 127 | Maximum callchain depth |

---

## Locking Summary

The perf_event subsystem uses a layered locking strategy to handle concurrent
access from syscalls, ioctls, scheduler hooks, and NMI-context PMI handlers:

| Lock | Type | Scope | Used By |
|------|------|-------|---------|
| `ctx->mutex` | `struct mutex` | Per-context; sleepable | `perf_event_open`, ioctls, BPF attach, event destruction |
| `ctx->lock` | `raw_spinlock_t` | Per-context; IRQ-safe | Scheduler hooks (`sched_in`/`sched_out`), event state transitions |
| `rb->event_lock` | `spinlock_t` | Per-ring-buffer | Ring buffer event list management |
| `event->mmap_mutex` | `struct mutex` | Per-event | mmap/munmap serialization |
| `event->child_mutex` | `struct mutex` | Per-event | Inherited event child list |
| `bpf_event_mutex` | Global mutex | System-wide | BPF program attachment to tracing events |
| RCU | Read-side | Various | Ring buffer access (`event->rb`), BPF prog references (`event->prog`), `bpf_event_entry` lifetime |
| `local_t rb->head` | Lock-free atomic | Per-ring-buffer | Ring buffer write position (supports NMI nesting) |

Key discipline:
- To **modify** `ctx->event_list`: hold **both** `ctx->mutex` and `ctx->lock`.
- To **read** `ctx->event_list`: hold **either** `ctx->mutex` or `ctx->lock`.
- Ring buffer writes use lock-free `local_cmpxchg()` with nesting depth
  tracking (`rb->nest`) for NMI safety.
- BPF program pointers on events are RCU-protected: `rcu_assign_pointer()` on
  attach, `rcu_dereference()` on overflow, `synchronize_rcu()` on detach.
