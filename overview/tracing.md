# Linux Kernel Tracing Subsystem

## Overview

The Linux tracing subsystem is a multi-layered infrastructure for kernel and
user-space observation. At its core is the Ftrace framework (`kernel/trace/`),
which owns the ring buffer, the tracefs filesystem, and a plugin architecture
for tracers. Four orthogonal instrumentation mechanisms feed into this
    infrastructure: static tracepoints, dynamic kprobes, user-space uprobes,
    and perf events. BPF programs can attach to all of these.

---

## Ftrace: The Function Tracer Framework

### Core Data Structures

#### `struct trace_array`

`kernel/trace/trace.h:320`

The top-level object for one tracing instance (global or named instance under
`/sys/kernel/tracing/instances/`):

```c
struct trace_array {
    struct list_head    list;
    char                *name;              // NULL for global
    struct array_buffer array_buffer;       // main ring buffer + per-cpu data
    struct array_buffer max_buffer;         // snapshot buffer (latency tracers)
    struct tracer       *current_trace;     // active tracer plugin
    unsigned int        trace_flags;        // TRACE_ITER_* bitmask
    struct dentry       *dir;               // tracefs directory
    struct ftrace_ops   *ops;               // function tracer ops
    struct fgraph_ops   *gops;              // function_graph ops
    struct trace_pid_list __rcu *filtered_pids;
    cpumask_var_t       tracing_cpumask;
};
```

Global instance: `global_trace` at `kernel/trace/trace.c:499`.

#### `struct tracer`

`kernel/trace/trace.h:576`

Plugin interface — every tracer (nop, function, function_graph, irqsoff,
wakeup, etc.):

```c
struct tracer {
    const char *name;
    int  (*init)(struct trace_array *tr);
    void (*reset)(struct trace_array *tr);
    void (*start)(struct trace_array *tr);
    void (*stop)(struct trace_array *tr);
    enum print_line_t (*print_line)(struct trace_iterator *iter);
    struct tracer *next;
    bool allow_instances;
    bool use_max_tr;    // uses snapshot buffer
};
```

Registered via `register_tracer()`. Selected via `current_tracer` tracefs file.

#### `struct trace_entry` — Universal Event Header

`include/linux/trace_events.h:82`

```c
struct trace_entry {
    unsigned short type;           // event type ID
    unsigned char  flags;          // TRACE_FLAG_* (irq, preempt state)
    unsigned char  preempt_count;
    int            pid;            // current->pid at trace time
};
```

### `struct ftrace_ops` — Callback Registration

`include/linux/ftrace.h:349`

```c
struct ftrace_ops {
    ftrace_func_t    func;          // callback(ip, parent_ip, op, fregs)
    unsigned long    flags;         // FTRACE_OPS_FL_*
    struct ftrace_ops_hash *func_hash;  // filter/notrace hashes
    unsigned long    trampoline;    // custom trampoline address
};
```

Registration: `register_ftrace_function(ops)` / `unregister_ftrace_function(ops)`.

Key flags:
- `FTRACE_OPS_FL_SAVE_REGS` — full register save
- `FTRACE_OPS_FL_IPMODIFY` — may rewrite instruction pointer (live patching)
- `FTRACE_OPS_FL_PERMANENT` — not disabled when `ftrace_enabled=0`
- `FTRACE_OPS_FL_PID` — affected by `set_ftrace_pid`

### Dynamic Patching

At build time, `gcc -pg` or `-mfentry` inserts a 5-byte `call __fentry__` at
every function entry. The kernel collects these addresses into the
`__mcount_loc` ELF section. At boot, `ftrace_init()` builds the `dyn_ftrace`
array.

```c
struct dyn_ftrace {
    unsigned long   ip;     // call site address
    unsigned long   flags;  // FTRACE_FL_* state
    struct dyn_arch_ftrace arch;
};
```

When no callbacks are registered: `ftrace_make_nop()` patches to NOP. When
callbacks register and filter matches: `ftrace_make_call()` patches back.
Patching uses `stop_machine()` on x86.

### Filter Hashes

```c
struct ftrace_ops_hash {
    struct ftrace_hash __rcu *notrace_hash;  // exclude functions
    struct ftrace_hash __rcu *filter_hash;   // include functions
};
```

Empty `filter_hash` = all functions traced. Each hash is a hashtable of `ftrace_func_entry` keyed by function IP.

### The tracefs Filesystem

Mounted at `/sys/kernel/tracing/`:

| File | Description |
|------|-------------|
| `current_tracer` | Active tracer name |
| `available_tracers` | Registered tracers |
| `tracing_on` | Enable/disable recording |
| `trace` | Formatted output (seekable) |
| `trace_pipe` | Streaming output (consuming) |
| `trace_marker` | Write strings into ring buffer |
| `trace_marker_raw` | Write binary data |
| `trace_clock` | Clock source selection |
| `buffer_size_kb` | Per-CPU ring buffer size |
| `trace_options` | TRACE_ITER_* options |
| `set_ftrace_filter` | Function include filter |
| `set_ftrace_notrace` | Function exclude filter |
| `set_ftrace_pid` | PID-based filtering |
| `available_filter_functions` | All instrumentable functions |
| `events/` | Per-event subsystem directories |
| `instances/` | Named trace instances |
| `per_cpu/cpuN/` | Per-CPU buffer files |

---

## Ring Buffer

`kernel/trace/ring_buffer.c` (~7500 lines)

### Structure Hierarchy

```
struct trace_buffer
  └── struct ring_buffer_per_cpu **buffers  (per CPU)
        ├── buffer_page *head_page      (oldest data)
        ├── buffer_page *tail_page      (current write)
        ├── buffer_page *commit_page    (last committed)
        ├── buffer_page *reader_page    (isolated for reader)
        └── struct list_head *pages     (circular page list)
```

### `struct ring_buffer_per_cpu`

`kernel/trace/ring_buffer.c:473`

```c
struct ring_buffer_per_cpu {
    struct trace_buffer     *buffer;
    raw_spinlock_t          reader_lock;    // serializes readers
    arch_spinlock_t         lock;           // writer page advance
    struct buffer_page      *head_page;     // oldest unconsumed
    struct buffer_page      *tail_page;     // current write target
    struct buffer_page      *commit_page;
    struct buffer_page      *reader_page;   // private reader page
    unsigned long           nr_pages;
    local_t                 entries;
    local_t                 overrun;        // overwritten events
    local_t                 dropped_events;
    local_t                 committing;     // nesting depth
    rb_time_t               write_stamp;
};
```

### Event Encoding

`struct ring_buffer_event` — packed in page data:
```c
struct ring_buffer_event {
    u32 type_len:5;     // 0-28: data length/4; 29=PADDING; 30=TIME_EXTEND; 31=TIME_STAMP
    u32 time_delta:27;  // ns delta from page timestamp
    u32 array[];        // length word if type_len==0, then data
};
```

### Lock-Free Write Path

`ring_buffer_lock_reserve()` at `ring_buffer.c:4482`:

1. `preempt_disable_notrace()` — prevent CPU migration
2. Check `record_disabled` (atomic, no lock)
3. `trace_recursive_lock()` — detect recursion across NMI/hardirq/softirq/task levels
4. `rb_reserve_next_event()`: `local_cmpxchg()` on `tail_page->write` to claim space; advance page if full
5. Returns pointer to event data for caller to fill

`ring_buffer_unlock_commit()`:
1. `rb_commit()` — updates `commit_page->page->commit` (makes event visible to readers)
2. `rb_wakeups()` — wake poll/wait waiters
3. `preempt_enable_notrace()`

### Overwrite vs Producer-Consumer

- **Overwrite** (`RB_FL_OVERWRITE`): write pointer advances past read pointer, incrementing `overrun`
- **Producer-Consumer**: returns NULL when full, incrementing `dropped_events`

### Read Path

Reader uses a private `reader_page`. `rb_get_reader_page()` atomically swaps
`reader_page` with `head_page` using `local_cmpxchg()`. Reader iterates events
on its private page without locking.

---

## Tracepoints: Static Instrumentation

### `struct tracepoint`

`include/linux/tracepoint-defs.h:39`

```c
struct tracepoint {
    const char              *name;
    struct static_key_false  key;           // disabled by default (NOP branch)
    struct static_call_key  *static_call_key;
    void                    *iterator;      // __traceiter_NAME
    struct tracepoint_func __rcu *funcs;    // RCU-protected probe array
};
```

When no probes: `static_key_false` makes the callsite a NOP. On first probe:
`static_key_enable()` patches the branch.

### TRACE_EVENT Macro

Generates both a tracepoint AND ring buffer event format:

```c
TRACE_EVENT(sched_switch,
    TP_PROTO(bool preempt, struct task_struct *prev, struct task_struct *next,
             unsigned int prev_state),
    TP_ARGS(preempt, prev, next, prev_state),
    TP_STRUCT__entry(
        __array(char, prev_comm, TASK_COMM_LEN)
        __field(pid_t, prev_pid)
        __field(int, prev_prio)
        __field(long, prev_state)
        __array(char, next_comm, TASK_COMM_LEN)
        __field(pid_t, next_pid)
        __field(int, next_prio)
    ),
    TP_fast_assign(
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->prev_pid = prev->pid;
        // ...
    ),
    TP_printk("prev_comm=%s ...", __entry->prev_comm, ...)
);
```

This generates:
1. `DECLARE_TRACE(sched_switch, ...)` — inline callsite
2. `struct trace_event_class` — shared metadata
3. `struct trace_event_call` — per-event, registered in `ftrace_events`
4. Probe function: `trace_event_buffer_reserve()` → fill `__entry` → `trace_event_buffer_commit()`

### `struct trace_event_call`

`include/linux/trace_events.h:363`

```c
struct trace_event_call {
    struct list_head        list;
    struct trace_event_class *class;
    struct tracepoint       *tp;
    struct trace_event      event;          // type ID + print functions
    char                    *print_fmt;
    int                     flags;
    int                     perf_refcount;
    struct bpf_prog_array __rcu *prog_array;  // BPF programs attached
};
```

### Probe Registration

`tracepoint_probe_register()` in `kernel/tracepoint.c`:
1. Acquire `tracepoints_mutex`
2. Allocate new `tracepoint_func[]` array, insert sorted by priority
3. `rcu_assign_pointer()` to publish
4. `static_key_enable()` on first probe

---

## Kprobes: Dynamic Instrumentation

`include/linux/kprobes.h`, `kernel/kprobes.c`

### `struct kprobe`

```c
struct kprobe {
    struct hlist_node     hlist;
    unsigned long         nmissed;
    kprobe_opcode_t      *addr;           // probe address
    const char           *symbol_name;
    unsigned int          offset;
    kprobe_pre_handler_t  pre_handler;    // called before probed instruction
    kprobe_post_handler_t post_handler;   // called after (INT3 path only)
    kprobe_opcode_t       opcode;         // saved original byte
    u32                   flags;          // KPROBE_FLAG_*
};
```

### INT3-Based Mechanism

1. `register_kprobe(p)`: resolve address, check blacklist, save original instruction, write `0xCC` (INT3) via `text_poke()`
2. INT3 fires → `kprobe_int3_handler()` → call `pre_handler` → single-step original from insn slot → `post_handler`
3. `unregister_kprobe(p)`: `text_poke()` restores original bytes

### Ftrace-Based Kprobes

When probe is at function entry (offset 0) and `CONFIG_KPROBES_ON_FTRACE`: uses
ftrace callback instead of INT3. No single-stepping, lower overhead.
`KPROBE_FLAG_FTRACE` set.

### `struct kretprobe`

```c
struct kretprobe {
    struct kprobe       kp;
    kretprobe_handler_t handler;        // return handler
    kretprobe_handler_t entry_handler;  // optional entry handler
    int                 maxactive;
    struct rethook     *rh;
};
```

Saves real return address, replaces with `__kretprobe_trampoline`. On return:
trampoline → `kretprobe_trampoline_handler()` → user handler → restore.

### Blacklist

`NOKPROBE_SYMBOL()` macro places functions in `__kprobe_blacklist` section.
Critical sections, kprobe infrastructure, exception handlers excluded.

---

## Uprobes: User-Space Probes

`include/linux/uprobes.h`, `kernel/events/uprobes.c`

### `struct uprobe_consumer`

```c
struct uprobe_consumer {
    int (*handler)(struct uprobe_consumer *, struct pt_regs *, __u64 *data);
    int (*ret_handler)(struct uprobe_consumer *, unsigned long func,
                       struct pt_regs *, __u64 *data);
    bool (*filter)(struct uprobe_consumer *, struct mm_struct *mm);
};
```

### Mechanism

1. `uprobe_register(inode, offset, uc)`: find/create uprobe for (inode, offset), insert INT3 at mapped virtual address via copy-on-write
2. Process hits INT3 → identified as uprobe → call consumer handlers → single-step original from XOL (execute-out-of-line) area → restore
3. XOL area: per-thread scratch VMA in process address space

### `struct uprobe_task`

Per-task state during single-step: `state`, `depth` (uretprobe nesting),
`return_instances`, `xol_vaddr`, `active_uprobe`.

---

## Perf Events

`include/linux/perf_event.h`, `kernel/events/core.c`

### `struct perf_event` (key fields)

```c
struct perf_event {
    struct pmu              *pmu;
    enum perf_event_state   state;
    struct perf_event_attr  attr;       // user configuration
    struct hw_perf_event    hw;         // hardware state
    struct perf_buffer      *rb;        // mmap'd ring buffer
    perf_overflow_handler_t overflow_handler;
    struct bpf_prog         *prog;      // BPF filter program
    struct trace_event_call *tp_event;  // for tracepoint events
};
```

### `struct perf_event_attr`

```c
struct perf_event_attr {
    __u32 type;          // PERF_TYPE_HARDWARE/SOFTWARE/TRACEPOINT/RAW/BREAKPOINT
    __u64 config;        // event-specific (e.g., PERF_COUNT_HW_CPU_CYCLES or tracepoint ID)
    __u64 sample_period; // or sample_freq
    __u64 sample_type;   // PERF_SAMPLE_IP|TID|TIME|CALLCHAIN|...
    // bitfields: disabled, inherit, pinned, exclude_user/kernel, mmap, etc.
};
```

### `struct pmu` — PMU Driver Interface

```c
struct pmu {
    const char *name;
    int  (*event_init)(struct perf_event *);
    int  (*add)(struct perf_event *, int flags);
    void (*del)(struct perf_event *, int flags);
    void (*start)(struct perf_event *, int flags);
    void (*stop)(struct perf_event *, int flags);
    void (*read)(struct perf_event *);
};
```

`event_init()` validates config, `add()`/`del()` schedule hardware, `read()` updates count.

### Perf Ring Buffer

Separate from ftrace ring buffer. Shared with userspace via `mmap()`.
`perf_output_begin()` claims space, `perf_output_sample()` writes sample,
`perf_output_end()` finalizes.

---

## BPF Integration

### Program Types for Tracing

| Type | Attach | Context |
|------|--------|---------|
| `BPF_PROG_TYPE_KPROBE` | perf_event_open + kprobe | `struct pt_regs *` |
| `BPF_PROG_TYPE_TRACEPOINT` | perf_event_open + tracepoint ID | tracepoint args buffer |
| `BPF_PROG_TYPE_RAW_TRACEPOINT` | `BPF_RAW_TRACEPOINT_OPEN` | raw `u64 args[]` |
| `BPF_PROG_TYPE_PERF_EVENT` | perf_event_open + hw/sw event | `struct bpf_perf_event_data` |

### `trace_call_bpf()`

`kernel/trace/bpf_trace.c:110`

Called from event handlers when `call->prog_array` is non-NULL:

```c
unsigned int trace_call_bpf(struct trace_event_call *call, void *ctx) {
    if (unlikely(__this_cpu_inc_return(bpf_prog_active) != 1))
        return 0;  // prevent recursion
    ret = bpf_prog_run_array(rcu_dereference(call->prog_array), ctx, bpf_prog_run);
    __this_cpu_dec(bpf_prog_active);
    return ret;  // 0=discard, 1=keep
}
```

### Raw Tracepoints

`struct bpf_raw_event_map` in `__bpf_raw_tp` ELF section links tracepoints to
BPF wrappers (`__bpf_trace_NAME`). Raw tracepoints pass arguments directly as
`u64 args[]` — no ring buffer formatting overhead.

### Key BPF Helpers for Tracing

| Helper | Description |
|--------|-------------|
| `bpf_probe_read_kernel()` | Safe kernel memory read |
| `bpf_probe_read_user()` | Safe user memory read |
| `bpf_perf_event_output()` | Write to perf ring buffer |
| `bpf_get_stackid()` | Get stack ID into stackmap |
| `bpf_get_stack()` | Get stack trace into buffer |
| `bpf_get_current_task_btf()` | BTF-typed `task_struct *` |
| `bpf_override_return()` | Override function return value |
| `bpf_get_func_ip()` | IP of probed function |

---

## Function Graph Tracer

`kernel/trace/fgraph.c`, `kernel/trace/trace_functions_graph.c`

### Data Structures

```c
struct ftrace_graph_ent {
    unsigned long func;    // function being entered
    int           depth;   // call depth
};

struct ftrace_graph_ret {
    unsigned long       func;
    unsigned long       retval;      // CONFIG_FUNCTION_GRAPH_RETVAL
    int                 depth;
    unsigned long long  calltime;    // entry timestamp (ns)
    unsigned long long  rettime;     // return timestamp (ns)
};

struct fgraph_ops {
    trace_func_graph_ent_t  entryfunc;   // entry callback
    trace_func_graph_ret_t  retfunc;     // return callback
    struct ftrace_ops       ops;         // filter support
    int                     idx;         // 0-15 in fgraph_array
};
```

### Shadow Stack

Each task has `task_struct->ret_stack` (array of `unsigned long`). Up to 16
`fgraph_ops` can be active simultaneously.

**On entry** (`function_graph_enter()`):
1. Save real return address
2. Replace with `return_to_handler` trampoline
3. Push `ftrace_ret_stack` onto shadow stack
4. Call each matching `fgraph_ops[i]->entryfunc()`

**On return** (via trampoline → `ftrace_return_to_handler()`):
1. Pop shadow stack, recover real return address
2. Call matching `retfunc()` callbacks with timing data
3. Return to real caller

---

## Event Triggers

`kernel/trace/trace_events_trigger.c`

Attached to `trace_event_file->triggers` list. Executed by
`event_triggers_call()` from event handlers.

### Available Triggers

| Trigger | Description |
|---------|-------------|
| `traceon` / `traceoff` | Enable/disable ring buffer |
| `snapshot` | Copy to snapshot buffer |
| `stacktrace` | Record kernel stack trace |
| `enable_event` / `disable_event` | Control another event |
| `hist:` | In-kernel histogram aggregation |

### Histogram Triggers

`kernel/trace/trace_events_hist.c`

```
echo 'hist:keys=pid:vals=hitcount' > events/sched/sched_switch/trigger
```

Uses `struct tracing_map` — a lock-free hashtable in
`kernel/trace/tracing_map.c`. Supports variables (`$var = field`), conditional
actions (`onmax()`, `onchange()`), and synthetic events.

---

## Recursion Protection

Critical across all tracing paths:
- **Ring buffer**: `trace_recursive_lock()` tracks NMI/hardirq/softirq/task levels via `current_context`
- **BPF**: `bpf_prog_active` per-CPU counter prevents re-entrant BPF execution
- **Kprobes**: `kprobe_busy_begin()` / `kprobe_busy_end()`
- **Raw tracepoints**: `bpf_raw_tp_nest_level` per-CPU counter

---

## Data Flow Example: Tracepoint to BPF

```
1. trace_sched_switch(preempt, prev, next, prev_state)
   └─ static_branch check → near-zero cost when disabled

2. Slow path (probes registered):
   └─ static_call(tp_func_sched_switch)

3. Probe: ftrace_raw_event_sched_switch()
   ├─ trace_event_buffer_reserve() → ring_buffer_lock_reserve()
   ├─ Fill __entry from TP_fast_assign
   ├─ Check call->prog_array → trace_call_bpf()
   │   └─ bpf_prog_run_array() → run BPF program
   └─ trace_event_buffer_commit() → ring_buffer_unlock_commit()

4. BPF program:
   ├─ Read event fields via bpf_probe_read_kernel()
   └─ Emit to userspace via bpf_perf_event_output()

5. Reader: trace_pipe / perf mmap / BPF ringbuf
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| `kernel/trace/trace.c` | Core: trace_array, tracefs, ring buffer mgmt |
| `kernel/trace/trace.h` | Internal structs: trace_array, tracer |
| `kernel/trace/ftrace.c` | Function tracer: dyn_ftrace, filter hashes, trampolines |
| `kernel/trace/fgraph.c` | Function graph: shadow stack, entry/return |
| `kernel/trace/ring_buffer.c` | Ring buffer implementation |
| `kernel/trace/trace_events.c` | Event infrastructure, enabling/disabling |
| `kernel/trace/trace_events_trigger.c` | Trigger framework |
| `kernel/trace/trace_events_hist.c` | Histogram triggers |
| `kernel/trace/trace_kprobe.c` | Kprobe events |
| `kernel/trace/trace_uprobe.c` | Uprobe events |
| `kernel/trace/trace_event_perf.c` | Perf/BPF attachment |
| `kernel/trace/bpf_trace.c` | BPF helpers, trace_call_bpf() |
| `kernel/trace/tracing_map.c` | Lock-free hash map for histograms |
| `kernel/kprobes.c` | Kprobe core |
| `kernel/events/uprobes.c` | Uprobe core |
| `kernel/events/core.c` | Perf event lifecycle |
| `include/linux/ftrace.h` | ftrace_ops, fgraph_ops, dyn_ftrace |
| `include/linux/trace_events.h` | trace_event_call, trace_event_class |
| `include/linux/tracepoint.h` | DECLARE_TRACE, TRACE_EVENT |
| `include/linux/kprobes.h` | kprobe, kretprobe |
| `include/linux/uprobes.h` | uprobe_consumer, uprobe_task |
| `include/linux/perf_event.h` | perf_event, pmu, perf_sample_data |
| `include/linux/ring_buffer.h` | Ring buffer API |
| `include/trace/events/` | Tracepoint definitions (sched.h, mm.h, etc.) |
