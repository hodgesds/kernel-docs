# Linux Kernel SMP and CPU Topology

## Overview

The Linux kernel's Symmetric Multiprocessing (SMP) subsystem manages
multiple CPUs as a cooperating system. It handles CPU discovery and
bringup, inter-processor communication via IPIs, per-CPU data isolation,
CPU hotplug lifecycle, NUMA-aware memory placement, and hierarchical
scheduling domains. The core implementation lives in `kernel/smp.c`,
`kernel/cpu.c`, `kernel/smpboot.c`, and `kernel/sched/topology.c`, with
architecture-specific code (e.g., `arch/x86/kernel/smpboot.c`) providing
the low-level CPU startup and IPI delivery mechanisms.

## Architecture

### CPU Topology Hierarchy

Modern systems expose a multi-level CPU topology. The kernel tracks these
levels and exposes them through accessor macros defined in
`include/linux/topology.h`:

```
System (Package/Socket)
  |
  +-- Die (topology_die_id / topology_die_cpumask)
        |
        +-- NUMA Node (cpu_to_node / cpumask_of_node)
              |
              +-- Cluster (topology_cluster_id / topology_cluster_cpumask)
                    |
                    +-- Core (topology_core_id / topology_core_cpumask)
                          |
                          +-- SMT Thread (topology_sibling_cpumask)
```

Default topology accessor macros (`include/linux/topology.h:193-231`):
```c
#define topology_physical_package_id(cpu)  ((void)(cpu), -1)
#define topology_die_id(cpu)               ((void)(cpu), -1)
#define topology_cluster_id(cpu)           ((void)(cpu), -1)
#define topology_core_id(cpu)              ((void)(cpu), 0)
#define topology_sibling_cpumask(cpu)      cpumask_of(cpu)
#define topology_core_cpumask(cpu)         cpumask_of(cpu)
#define topology_cluster_cpumask(cpu)      cpumask_of(cpu)
#define topology_die_cpumask(cpu)          cpumask_of(cpu)
```

These are weak defaults; architectures override them. On x86, per-CPU
maps track siblings at each level (`arch/x86/kernel/smpboot.c:95-104`):

```c
DEFINE_PER_CPU_READ_MOSTLY(cpumask_var_t, cpu_sibling_map);  // HT siblings
DEFINE_PER_CPU_READ_MOSTLY(cpumask_var_t, cpu_core_map);     // core siblings
DEFINE_PER_CPU_READ_MOSTLY(cpumask_var_t, cpu_die_map);      // die siblings
```

### SMP Execution Model

All CPUs run the same kernel image. One CPU (the boot processor, or BSP)
performs early initialization, then brings up application processors (APs)
through the hotplug state machine. Once online, each CPU runs its own
scheduler instance, services its own interrupts, and accesses its own
per-CPU data. Cross-CPU coordination uses IPIs, shared memory with
appropriate barriers, and synchronization primitives.

## Core Data Structures

### cpumask

A `cpumask` is a bitmap representing a set of CPUs, one bit per possible
CPU ID (`include/linux/cpumask_types.h:9`):

```c
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

The kernel maintains several global cpumasks
(`include/linux/cpumask.h:117-128`):

| Mask | Meaning |
|------|---------|
| `cpu_possible_mask` | CPUs that could ever be plugged in (fixed at boot) |
| `cpu_present_mask`  | CPUs currently physically present |
| `cpu_online_mask`   | CPUs available to the scheduler |
| `cpu_active_mask`   | CPUs available for task migration |
| `cpu_enabled_mask`  | CPUs that can be brought online |
| `cpu_dying_mask`    | CPUs currently going offline |

Common iteration macros:
```c
for_each_possible_cpu(cpu)   // all populatable CPUs
for_each_online_cpu(cpu)     // scheduler-visible CPUs
for_each_present_cpu(cpu)    // physically present CPUs
for_each_cpu(cpu, mask)      // custom mask iteration
```

Dynamic allocation (`cpumask_var_t`) is used when `CONFIG_CPUMASK_OFFSTACK`
is set, to avoid large stack-allocated bitmaps on systems with high
`NR_CPUS` (`include/linux/cpumask_types.h:60-64`):

```c
#ifdef CONFIG_CPUMASK_OFFSTACK
typedef struct cpumask *cpumask_var_t;        // heap-allocated pointer
#else
typedef struct cpumask cpumask_var_t[1];      // stack-embedded array
#endif
```

### call_single_data (CSD)

The CSD structure is the fundamental unit for IPI-based cross-CPU function
calls (`include/linux/smp.h:23-27`):

```c
struct __call_single_data {
    struct __call_single_node node;
    smp_call_func_t func;
    void *info;
};
typedef struct __call_single_data call_single_data_t
    __aligned(sizeof(struct __call_single_data));
```

Each CPU has a lockless linked list (`call_single_queue`) of pending CSD
entries, drained by `generic_smp_call_function_single_interrupt()`
(`kernel/smp.c:48`):

```c
static DEFINE_PER_CPU_SHARED_ALIGNED(struct llist_head, call_single_queue);
```

### sched_domain

The `sched_domain` structure represents one level of the scheduling
topology hierarchy (`include/linux/sched/topology.h:73-151`):

```c
struct sched_domain {
    struct sched_domain __rcu *parent;   // up toward system-wide
    struct sched_domain __rcu *child;    // down toward individual CPUs
    struct sched_group *groups;          // circular list of groups at this level
    unsigned long min_interval;          // minimum balance interval (ms)
    unsigned long max_interval;          // maximum balance interval (ms)
    unsigned int busy_factor;            // reduce balancing when busy
    unsigned int imbalance_pct;          // imbalance threshold percentage
    unsigned int cache_nice_tries;       // cache hot migration resistance
    unsigned int imb_numa_nr;            // NUMA imbalance task threshold
    int flags;                           // SD_* flags
    int level;                           // depth in the hierarchy
    unsigned long last_balance;          // last balance timestamp (jiffies)
    unsigned int balance_interval;       // current balance interval (ms)
    struct sched_domain_shared *shared;  // shared idle tracking data
    unsigned int span_weight;            // number of CPUs in this domain
    unsigned long span[];                // variable-length CPU bitmap
};
```

### sched_domain_topology_level

Describes how to construct one level of the scheduling domain hierarchy
(`include/linux/sched/topology.h:179-185`):

```c
struct sched_domain_topology_level {
    sched_domain_mask_f mask;       // function returning CPUs at this level
    sched_domain_flags_f sd_flags;  // function returning SD_* flags
    int             numa_level;     // NUMA hop distance
    struct sd_data  data;           // per-CPU allocated domain/group data
    char            *name;          // debug name
};
```

## CPU Bringup Sequence

### Boot CPU Initialization

The boot CPU (BSP) initializes the SMP subsystem during kernel startup.
The key entry point is `smp_init()` (`kernel/smp.c:992-1010`):

```c
void __init smp_init(void)
{
    int num_nodes, num_cpus;

    idle_threads_init();       // fork idle tasks for all possible CPUs
    cpuhp_threads_init();      // create per-CPU hotplug threads

    pr_info("Bringing up secondary CPUs ...\n");
    bringup_nonboot_cpus(setup_max_cpus);

    num_nodes = num_online_nodes();
    num_cpus  = num_online_cpus();
    pr_info("Brought up %d node%s, %d CPU%s\n",
        num_nodes, str_plural(num_nodes),
        num_cpus, str_plural(num_cpus));
    smp_cpus_done(setup_max_cpus);
}
```

The `bringup_nonboot_cpus()` function iterates over present CPUs and calls
the hotplug state machine to bring each to `CPUHP_ONLINE`
(`kernel/cpu.c:1881-1892`):

```c
void __init bringup_nonboot_cpus(unsigned int max_cpus)
{
    if (!max_cpus)
        return;
    if (cpuhp_bringup_cpus_parallel(max_cpus))  // try parallel bringup
        return;
    cpuhp_bringup_mask(cpu_present_mask, max_cpus, CPUHP_ONLINE);
}
```

### Application Processor Startup (x86)

On x86, each AP starts in real mode and transitions through protected/long
mode to reach the `start_secondary()` entry point
(`arch/x86/kernel/smpboot.c:229-313`):

```
1. BSP sends INIT-SIPI-SIPI sequence to target AP via APIC
2. AP starts executing trampoline code in real mode
3. AP transitions to protected mode, then long mode (64-bit)
4. AP calls start_secondary():
     a. cr4_init() -- set up control registers
     b. cpu_init_exception_handling() -- IDT, GDT setup
     c. load_ucode_ap() -- load microcode
     d. cpuhp_ap_sync_alive() -- synchronize with BSP
     e. cpu_init() -- per-CPU state initialization
     f. fpu__init_cpu() -- FPU initialization
     g. rcutree_report_cpu_starting() -- notify RCU
     h. ap_starting() -- identify CPU, set sibling maps
     i. set_cpu_online(smp_processor_id(), true)
     j. local_irq_enable()
     k. cpu_startup_entry(CPUHP_AP_ONLINE_IDLE) -- enter idle loop
```

The `ap_starting()` function (`arch/x86/kernel/smpboot.c:170-209`)
establishes the CPU's topology by calling `set_cpu_sibling_map()` and runs
the hotplug state callbacks via `notify_cpu_starting()`.

### Idle Thread Setup

Each CPU needs its own idle thread. These are pre-created during
`idle_threads_init()` (`kernel/smpboot.c:64-74`):

```c
void __init idle_threads_init(void)
{
    unsigned int cpu, boot_cpu;
    boot_cpu = smp_processor_id();
    for_each_possible_cpu(cpu) {
        if (cpu != boot_cpu)
            idle_init(cpu);  // calls fork_idle(cpu)
    }
}
```

## Inter-Processor Interrupts (IPIs)

### smp_call_function API

The IPI subsystem enables one CPU to execute a function on other CPUs.
The core API is defined in `include/linux/smp.h` and implemented in
`kernel/smp.c`:

| Function | Behavior |
|----------|----------|
| `smp_call_function_single(cpu, func, info, wait)` | Run on one specific CPU |
| `smp_call_function(func, info, wait)` | Run on all other online CPUs |
| `smp_call_function_many(mask, func, info, wait)` | Run on a set of CPUs |
| `smp_call_function_any(mask, func, info, wait)` | Run on any one CPU in mask |
| `on_each_cpu(func, info, wait)` | Run on all online CPUs including self |
| `on_each_cpu_mask(mask, func, info, wait)` | Run on mask including self |
| `on_each_cpu_cond(cond, func, info, wait)` | Conditional execution |
| `smp_call_function_single_async(cpu, csd)` | Async, caller-managed CSD |

The synchronous `smp_call_function_single()` (`kernel/smp.c:636-693`)
places a CSD on the target CPU's `call_single_queue` and optionally waits:

```c
int smp_call_function_single(int cpu, smp_call_func_t func,
                             void *info, int wait)
{
    call_single_data_t csd_stack = {
        .node = { .u_flags = CSD_FLAG_LOCK | CSD_TYPE_SYNC, },
    };
    // ...
    csd->func = func;
    csd->info = info;
    err = generic_exec_single(cpu, csd);
    if (wait)
        csd_lock_wait(csd);  // spin until remote CPU completes
    // ...
}
```

### IPI Delivery Mechanism

When a CSD is queued, an IPI is sent to the target CPU via
`arch_send_call_function_single_ipi()`. The target CPU handles it in
`generic_smp_call_function_single_interrupt()` (`kernel/smp.c:461-464`),
which calls `__flush_smp_call_function_queue()`. This function processes
entries in priority order (`kernel/smp.c:480-595`):

1. **SYNC callbacks first** -- callers are blocking, prioritize them
2. **ASYNC and IRQ_WORK callbacks** -- unlock CSD before execution (async)
3. **TTWU (try-to-wake-up) batched wakeups** -- processed last

### Reschedule IPI

The scheduler uses a dedicated IPI vector to trigger rescheduling on
remote CPUs (`include/linux/smp.h:139-142`):

```c
#define smp_send_reschedule(cpu) ({             \
    trace_ipi_send_cpu(cpu, _RET_IP_, NULL);    \
    arch_smp_send_reschedule(cpu);              \
})
```

On x86, this uses `RESCHEDULE_VECTOR (0xfd)` from
`arch/x86/include/asm/irq_vectors.h`.

## Per-CPU Data

### Declaration and Definition

Per-CPU variables give each CPU its own instance, eliminating cache-line
contention. The macros are in `include/linux/percpu-defs.h`:

```c
// Basic declaration/definition (percpu-defs.h:110-114)
#define DECLARE_PER_CPU(type, name)
#define DEFINE_PER_CPU(type, name)

// Cacheline-aligned to prevent false sharing (percpu-defs.h:140-146)
#define DEFINE_PER_CPU_SHARED_ALIGNED(type, name)

// Read-mostly optimization (percpu-defs.h:173-174)
#define DEFINE_PER_CPU_READ_MOSTLY(type, name)

// Cache-hot, frequently accessed (percpu-defs.h:123-127)
#define DEFINE_PER_CPU_CACHE_HOT(type, name)

// Page-aligned (percpu-defs.h:163-165)
#define DEFINE_PER_CPU_PAGE_ALIGNED(type, name)
```

### Accessors

Per-CPU variable access uses pointer arithmetic based on per-CPU offsets
(`include/linux/percpu-defs.h:237-273`):

```c
// Get pointer to variable for specific CPU
#define per_cpu_ptr(ptr, cpu)                       \
({                                                  \
    __verify_pcpu_ptr(ptr);                         \
    SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu))); \
})

// Get pointer to variable for current CPU (preemption-safe)
#define this_cpu_ptr(ptr) ...

// Dereference: get the value for a specific CPU
#define per_cpu(var, cpu)   (*per_cpu_ptr(&(var), cpu))
```

The `this_cpu_*` operations are single-instruction on x86 (using `%gs`
segment prefix), making them extremely efficient:

```c
this_cpu_read(var)       // read current CPU's copy
this_cpu_write(var, val) // write current CPU's copy
this_cpu_add(var, val)   // atomic add on current CPU
this_cpu_inc(var)        // atomic increment on current CPU
```

### Per-CPU Allocator

Dynamic per-CPU allocation uses a chunk-based allocator
(`include/linux/percpu.h:75-95`):

```c
struct pcpu_group_info {
    int           nr_units;      // aligned number of units
    unsigned long base_offset;   // base address offset
    unsigned int  *cpu_map;      // unit->cpu map
};

struct pcpu_alloc_info {
    size_t  static_size;   // static per-CPU data size
    size_t  reserved_size; // reserved area size
    size_t  dyn_size;      // dynamic allocation area size
    size_t  unit_size;     // total unit size
    size_t  atom_size;     // allocation atom size
    int     nr_groups;     // number of groups (NUMA nodes)
    struct pcpu_group_info groups[];
};
```

Allocation API (`include/linux/percpu.h:139-156`):

```c
alloc_percpu(type)                  // allocate per-CPU variable
alloc_percpu_gfp(type, gfp)        // with GFP flags
free_percpu(ptr)                    // free per-CPU variable
```

The first chunk setup modes (`include/linux/percpu.h:97-103`):

```c
enum pcpu_fc {
    PCPU_FC_AUTO,    // auto-select based on configuration
    PCPU_FC_EMBED,   // embed in large page mappings
    PCPU_FC_PAGE,    // individual page allocations
};
```

## CPU Hotplug State Machine

### State Organization

The CPU hotplug subsystem uses a linear state machine defined in
`include/linux/cpuhotplug.h`. States are divided into three sections:

```
CPUHP_OFFLINE (0)
  |
  |  PREPARE section: callbacks run on control CPU
  |    CPUHP_CREATE_THREADS
  |    CPUHP_PERF_X86_PREPARE
  |    ...
  |    CPUHP_SMPCFD_PREPARE
  |    CPUHP_RCUTREE_PREP
  |    CPUHP_TIMERS_PREPARE
  |    CPUHP_BP_PREPARE_DYN .. CPUHP_BP_PREPARE_DYN_END  (20 dynamic slots)
  |    CPUHP_BP_KICK_AP
  |    CPUHP_BRINGUP_CPU
  |
  |  STARTING section: callbacks run on hotplugged CPU, IRQs disabled
  |    CPUHP_AP_IDLE_DEAD
  |    CPUHP_AP_OFFLINE
  |    CPUHP_AP_SCHED_STARTING
  |    CPUHP_AP_RCUTREE_DYING
  |    ...timer/IRQ controller callbacks...
  |    CPUHP_AP_SMPCFD_DYING
  |    CPUHP_AP_ONLINE
  |    CPUHP_TEARDOWN_CPU
  |
  |  ONLINE section: callbacks run on hotplugged CPU, preemptible
  |    CPUHP_AP_ONLINE_IDLE
  |    CPUHP_AP_SMPBOOT_THREADS
  |    CPUHP_AP_WORKQUEUE_ONLINE
  |    CPUHP_AP_RCUTREE_ONLINE
  |    CPUHP_AP_ONLINE_DYN .. CPUHP_AP_ONLINE_DYN_END    (40 dynamic slots)
  |    CPUHP_AP_ACTIVE
  v
CPUHP_ONLINE
```

### Per-CPU State Tracking

Each CPU's hotplug progress is tracked in `struct cpuhp_cpu_state`
(`kernel/cpu.c:67-85`):

```c
struct cpuhp_cpu_state {
    enum cpuhp_state    state;          // current state
    enum cpuhp_state    target;         // target state
    enum cpuhp_state    fail;           // failed state for injection
    struct task_struct  *thread;        // per-CPU hotplug kthread
    bool                should_run;     // thread needs to run
    bool                rollback;       // performing rollback
    bool                single;         // single callback invocation
    bool                bringup;        // bringup vs teardown
    struct hlist_node   *node;          // multi-instance node
    enum cpuhp_state    cb_state;       // callback state
    int                 result;         // operation result
    atomic_t            ap_sync_state;  // AP synchronization state
    struct completion   done_up;        // bringup completion
    struct completion   done_down;      // teardown completion
};
```

### Step Registration

Each hotplug state has a `struct cpuhp_step` with startup and teardown
callbacks (`kernel/cpu.c:126-143`):

```c
struct cpuhp_step {
    const char      *name;
    union {
        int (*single)(unsigned int cpu);
        int (*multi)(unsigned int cpu, struct hlist_node *node);
    } startup;
    union {
        int (*single)(unsigned int cpu);
        int (*multi)(unsigned int cpu, struct hlist_node *node);
    } teardown;
    struct hlist_head    list;           // multi-instance list
    bool                 cant_stop;      // not stoppable
    bool                 multi_instance; // supports multiple instances
};
```

### Registration API

Subsystems register hotplug callbacks using
(`include/linux/cpuhotplug.h:271-277`):

```c
// Register and invoke startup on already-online CPUs
cpuhp_setup_state(state, name, startup_fn, teardown_fn);

// Register without invoking on current CPUs
cpuhp_setup_state_nocalls(state, name, startup_fn, teardown_fn);

// Use dynamic state slot (returns allocated state number)
cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, name, startup_fn, teardown_fn);

// Multi-instance (e.g., per-device callbacks)
cpuhp_setup_state_multi(state, name, startup_fn, teardown_fn);
```

### CPU Online/Offline Flow

CPU online path (`kernel/cpu.c:1627-1690`):

```
cpu_up(cpu)
  -> _cpu_up(cpu, tasks_frozen, CPUHP_ONLINE)
       -> cpus_write_lock()
       -> idle_thread_get(cpu)        // get pre-forked idle thread
       -> cpuhp_up_callbacks()        // walk PREPARE states on control CPU
            CPUHP_CREATE_THREADS
            ...
            CPUHP_BRINGUP_CPU         // arch kicks the AP alive
       -> cpuhp_kick_ap_work()        // AP hotplug thread runs STARTING+ONLINE
            CPUHP_AP_SCHED_STARTING   // scheduler notified
            ...
            CPUHP_AP_ONLINE
            CPUHP_AP_SMPBOOT_THREADS  // unpark per-CPU kthreads
            CPUHP_AP_WORKQUEUE_ONLINE
            CPUHP_AP_RCUTREE_ONLINE
            CPUHP_AP_ACTIVE           // set cpu_active_mask
            CPUHP_ONLINE
       -> cpus_write_unlock()
```

CPU offline reverses the sequence, tearing down from `CPUHP_ONLINE` back
to `CPUHP_OFFLINE`, with `stop_machine` used at `CPUHP_TEARDOWN_CPU` to
safely remove the CPU from the scheduler.

### Hotplug Locking

The `cpu_hotplug_lock` is a percpu-rwsem protecting CPU state transitions
(`kernel/cpu.c:485-505`):

```c
DEFINE_STATIC_PERCPU_RWSEM(cpu_hotplug_lock);

void cpus_read_lock(void)   { percpu_down_read(&cpu_hotplug_lock); }
void cpus_read_unlock(void) { percpu_up_read(&cpu_hotplug_lock); }
void cpus_write_lock(void)  { percpu_down_write(&cpu_hotplug_lock); }
void cpus_write_unlock(void){ percpu_up_write(&cpu_hotplug_lock); }
```

Readers (`cpus_read_lock`) prevent CPU state changes. This is the
recommended way for subsystems to ensure CPUs remain online during
operations.

## NUMA Topology

### Nodes and Distances

NUMA systems have multiple memory nodes with non-uniform access latencies.
The kernel uses ACPI SLIT (System Locality Information Table) distance
values (`include/linux/topology.h:46-51`):

```c
#define LOCAL_DISTANCE      10   // same-node access
#define REMOTE_DISTANCE     20   // different-node access
#define RECLAIM_DISTANCE    30   // threshold for node reclaim

#ifndef node_distance
#define node_distance(from, to)  ((from) == (to) ? LOCAL_DISTANCE : REMOTE_DISTANCE)
#endif
```

The `node_reclaim_distance` tunable (`include/linux/topology.h:73`) allows
platforms like AMD EPYC to override the default, since 2-hop distances
(value 32) can still benefit from cross-node reclaim.

### CPU-to-Node Mapping

Each CPU's NUMA node is tracked via a per-CPU variable
(`include/linux/topology.h:80-109`):

```c
DECLARE_PER_CPU(int, numa_node);

static inline int numa_node_id(void)
{
    return raw_cpu_read(numa_node);
}

static inline int cpu_to_node(int cpu)
{
    return per_cpu(numa_node, cpu);
}

static inline void set_cpu_numa_node(int cpu, int node)
{
    per_cpu(numa_node, cpu) = node;
}
```

For systems with memoryless nodes, `cpu_to_mem()` returns the nearest node
with memory, tracked separately (`include/linux/topology.h:130-158`).

### NUMA Iteration

The kernel provides iterators for NUMA-aware operations
(`include/linux/topology.h:307-330`):

```c
// Iterate over nodes in increasing distance order
for_each_node_numadist(node, unvisited) { ... }

// Iterate over cpumasks of increasing NUMA distance
for_each_numa_hop_mask(mask, node) { ... }

// Find nth CPU in mask closest to a given node
int sched_numa_find_nth_cpu(const struct cpumask *cpus, int cpu, int node);

// Get cpumask of CPUs within N hops of a node
const struct cpumask *sched_numa_hop_mask(unsigned int node, unsigned int hops);
```

## Scheduling Domains

### Domain Hierarchy

The scheduler constructs a per-CPU tree of `sched_domain` structures
representing the hardware topology. The default levels are defined in
`kernel/sched/topology.c:1784-1798`:

```c
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
    SDTL_INIT(tl_smt_mask, cpu_smt_flags, SMT),       // HT siblings
#endif
#ifdef CONFIG_SCHED_CLUSTER
    SDTL_INIT(tl_cls_mask, cpu_cluster_flags, CLS),    // cluster (L2/LLC tags)
#endif
#ifdef CONFIG_SCHED_MC
    SDTL_INIT(tl_mc_mask, cpu_core_flags, MC),         // multicore (LLC shared)
#endif
    SDTL_INIT(tl_pkg_mask, NULL, PKG),                 // package/socket
    { NULL, },
};
```

For NUMA systems, additional levels are automatically appended for each
hop distance. The resulting hierarchy (on a 2-socket, 2-core, 2-thread
system) looks like (`kernel/sched/topology.c:1145-1157`):

```
CPU   0   1   2   3   4   5   6   7

PKG  [                             ]
MC   [             ] [             ]
SMT  [     ] [     ] [     ] [     ]

PKG  0-7 0-7 0-7 0-7 0-7 0-7 0-7 0-7    (all CPUs)
MC   0-3 0-3 0-3 0-3 4-7 4-7 4-7 4-7    (same LLC)
SMT  0-1 0-1 2-3 2-3 4-5 4-5 6-7 6-7    (same core)
```

### SD_* Flags

Scheduling domain flags control load balancing behavior. They are defined
in `include/linux/sched/sd_flags.h` with metaflags indicating inheritance
rules:

| Flag | Metaflags | Purpose |
|------|-----------|---------|
| `SD_BALANCE_NEWIDLE` | SHARED_CHILD, NEEDS_GROUPS | Pull tasks when becoming idle |
| `SD_BALANCE_EXEC` | SHARED_CHILD, NEEDS_GROUPS | Balance on exec() |
| `SD_BALANCE_FORK` | SHARED_CHILD, NEEDS_GROUPS | Balance on fork() |
| `SD_BALANCE_WAKE` | SHARED_CHILD, NEEDS_GROUPS | Balance on wakeup |
| `SD_WAKE_AFFINE` | SHARED_CHILD | Consider waking CPU for placement |
| `SD_ASYM_CPUCAPACITY` | SHARED_PARENT, NEEDS_GROUPS | Heterogeneous CPU capacities |
| `SD_SHARE_CPUCAPACITY` | SHARED_CHILD, NEEDS_GROUPS | SMT capacity sharing |
| `SD_CLUSTER` | NEEDS_GROUPS | Cluster-level grouping |
| `SD_SHARE_LLC` | SHARED_CHILD, NEEDS_GROUPS | Shared last-level cache |
| `SD_SERIALIZE` | SHARED_PARENT, NEEDS_GROUPS | Single load-balance instance |
| `SD_ASYM_PACKING` | NEEDS_GROUPS | Pack tasks on preferred CPUs |
| `SD_PREFER_SIBLING` | NEEDS_GROUPS | Spread tasks to siblings |
| `SD_NUMA` | SHARED_PARENT, NEEDS_GROUPS | Cross-NUMA-node domain |

Metaflag semantics (`include/linux/sched/sd_flags.h:32-43`):
- `SDF_SHARED_CHILD` (0x1): If set, all child domains must also have it
- `SDF_SHARED_PARENT` (0x2): If set, all parent domains must also have it
- `SDF_NEEDS_GROUPS` (0x4): Only meaningful when domain has 2+ groups

### Load Balancing

Each `sched_domain` has associated `sched_group` structures forming a
circular list. The load balancer periodically compares load across groups
within a domain and migrates tasks to reduce imbalance. Key parameters:

```c
// include/linux/sched/topology.h:66-71
struct sched_domain_shared {
    atomic_t    ref;
    atomic_t    nr_busy_cpus;   // CPUs with runnable tasks
    int         has_idle_cores; // fast idle-CPU search hint
    int         nr_idle_scan;   // limit for idle scan
};
```

The balancing interval adapts: `min_interval` to `max_interval`, scaled by
`busy_factor` when CPUs are loaded. The `imbalance_pct` threshold (e.g.,
125 = 25% imbalance required) prevents unnecessary migrations.

### Domain Rebuild

Scheduling domains are rebuilt when topology changes via
`partition_sched_domains()` or `rebuild_sched_domains()`. This happens on:
- CPU hotplug events
- cpuset configuration changes
- Energy model updates
- Architecture topology updates (`arch_update_cpu_topology()`)

## stop_machine

### Mechanism

The `stop_machine` facility provides a way to execute code on specific CPUs
with all other CPUs held in a known state (interrupts disabled, spinning).
It is essential for operations requiring system-wide quiescence.

The per-CPU stopper infrastructure (`kernel/stop_machine.c:37-47`):

```c
struct cpu_stopper {
    struct task_struct  *thread;     // migration/N kthread
    raw_spinlock_t      lock;
    bool                enabled;    // stopper active?
    struct list_head    works;      // pending stop work items
    struct cpu_stop_work stop_work; // for stop_cpus
    unsigned long       caller;    // debug: who queued work
    cpu_stop_fn_t       fn;        // debug: current function
};

static DEFINE_PER_CPU(struct cpu_stopper, cpu_stopper);
```

The stopper thread runs as `migration/%u` at the highest priority
(`kernel/stop_machine.c:559-567`):

```c
static struct smp_hotplug_thread cpu_stop_threads = {
    .store              = &cpu_stopper.thread,
    .thread_should_run  = cpu_stop_should_run,
    .thread_fn          = cpu_stopper_thread,
    .thread_comm        = "migration/%u",
    .create             = cpu_stop_create,
    .park               = cpu_stop_park,
    .selfparking        = true,
};
```

### stop_machine State Machine

The multi-CPU stop operation uses a state machine to synchronize all CPUs
(`kernel/stop_machine.c:155-166`):

```c
enum multi_stop_state {
    MULTI_STOP_NONE,        // initial state
    MULTI_STOP_PREPARE,     // all CPUs scheduled
    MULTI_STOP_DISABLE_IRQ, // all CPUs disable interrupts
    MULTI_STOP_RUN,         // execute the function on active CPUs
    MULTI_STOP_EXIT,        // done, re-enable interrupts
};
```

The `multi_cpu_stop()` function (`kernel/stop_machine.c:201-258`)
implements the state machine on each CPU. CPUs acknowledge each state
transition, and the last CPU to acknowledge advances the global state.

### API

```c
// Stop one CPU, wait for completion (kernel/stop_machine.c:137-152)
int stop_one_cpu(unsigned int cpu, cpu_stop_fn_t fn, void *arg);

// Stop one CPU, don't wait (kernel/stop_machine.c:385-390)
bool stop_one_cpu_nowait(unsigned int cpu, cpu_stop_fn_t fn, void *arg,
                         struct cpu_stop_work *work_buf);

// Coordinate two CPUs (used by task migration) (kernel/stop_machine.c:335-365)
int stop_two_cpus(unsigned int cpu1, unsigned int cpu2,
                  cpu_stop_fn_t fn, void *arg);

// Stop ALL CPUs, run fn on @cpus with IRQs disabled (kernel/stop_machine.c:623-633)
int stop_machine(cpu_stop_fn_t fn, void *data, const struct cpumask *cpus);

// stop_machine with cpu_hotplug_lock already held (kernel/stop_machine.c:587-621)
int stop_machine_cpuslocked(cpu_stop_fn_t fn, void *data,
                            const struct cpumask *cpus);

// SMT-aware: stop all siblings in a core (kernel/stop_machine.c:636-653)
int stop_core_cpuslocked(unsigned int cpu, cpu_stop_fn_t fn, void *data);
```

### Usage

`stop_machine` is used for operations requiring global consistency:
- **CPU hotplug**: `CPUHP_TEARDOWN_CPU` uses it to safely remove a CPU
  from the scheduler
- **Module loading/unloading**: text patching requires all CPUs quiesced
- **ftrace/kprobes**: dynamic code patching
- **Memory hotplug**: page table modifications
- **Clock source changes**: TSC synchronization

## Integration with Subsystems

### Scheduler Integration

The scheduler is deeply intertwined with SMP topology:

- **Per-CPU runqueues**: Each CPU has its own `struct rq` with independent
  scheduling decisions
- **Load balancing**: Periodic and idle-time balancing across scheduling
  domains using `sched_balance_rq()` in `kernel/sched/fair.c`
- **Task placement**: `select_task_rq()` uses topology to pick optimal CPUs
  for wakeups, considering cache affinity (`SD_WAKE_AFFINE`), NUMA
  distances, and CPU capacity
- **NUMA balancing**: Automatic page migration and task placement based on
  memory access patterns (`task_numa_work()`)
- **CPU capacity**: `arch_scale_cpu_capacity()` reports per-CPU compute
  capacity for heterogeneous systems (big.LITTLE)
- **Hotplug callbacks**: `CPUHP_AP_SCHED_STARTING` and
  `CPUHP_AP_SCHED_WAIT_EMPTY` manage scheduler state during CPU
  transitions

### RCU Integration

RCU (Read-Copy-Update) depends heavily on per-CPU state and SMP
coordination:

- **Per-CPU RCU data**: Each CPU tracks its quiescent state for grace
  period detection
- **Hotplug states**: `CPUHP_RCUTREE_PREP` (prepare), `CPUHP_AP_RCUTREE_DYING`
  (CPU going offline, IRQs disabled), `CPUHP_AP_RCUTREE_ONLINE` (CPU
  online)
- **`rcutree_report_cpu_starting()`**: Called from `start_secondary()` to
  register the AP with the RCU tree before callbacks fire
- **Grace period detection**: RCU monitors per-CPU quiescent states; a CPU
  going offline must report a quiescent state
- **`rcu_momentary_eqs()`**: Called during `stop_machine` spinning to
  suppress RCU CPU stall warnings

### Workqueue Integration

Workqueues maintain per-CPU worker pools that respond to CPU topology:

- **Per-CPU pools**: Each online CPU has bound worker pools (normal and
  high-priority)
- **Unbound pools**: Use NUMA-aware CPU affinity for placement
- **Hotplug states**: `CPUHP_WORKQUEUE_PREP` (prepare worker pools),
  `CPUHP_AP_WORKQUEUE_ONLINE` (activate pools when CPU comes online)
- **CPU affinity**: `queue_work_on(cpu, wq, work)` targets specific CPUs;
  unbound work is distributed across NUMA-local CPUs

### sched_ext (Extensible Scheduler)

sched_ext allows BPF programs to implement custom scheduling policies while
building on the SMP topology infrastructure:

- **Topology access**: BPF schedulers can query `cpu_to_node()`,
  `cpumask` membership, and scheduling domains to make NUMA-aware
  placement decisions
- **Per-CPU dispatch queues**: sched_ext uses per-CPU local DSQs (dispatch
  queues) analogous to the per-CPU runqueue model
- **CPU hotplug awareness**: BPF schedulers receive callbacks for CPU
  online/offline events to adjust their scheduling decisions
- **Domain-based balancing**: BPF schedulers can implement custom load
  balancing that respects or overrides the kernel's scheduling domain
  hierarchy
- **stop_machine interaction**: sched_ext transitions (enable/disable)
  use `stop_machine` to safely switch between scheduling classes

### SMP Boot Threads (smpboot)

The `smpboot` framework manages per-CPU kthreads that are hotplug-aware
(`kernel/smpboot.c:80-84`):

```c
struct smpboot_thread_data {
    unsigned int                cpu;
    unsigned int                status;    // HP_THREAD_NONE/ACTIVE/PARKED
    struct smp_hotplug_thread   *ht;
};
```

Per-CPU kthreads registered via `smpboot_register_percpu_thread()` are
automatically parked when their CPU goes offline and unparked when it
comes back online. This is used by:
- **migration/N** (stop_machine stopper threads)
- **ksoftirqd/N** (softirq processing)
- **kcompactd/N** (memory compaction)
- **watchdog/N** (NMI watchdog)

## Key Source Files

| File | Purpose |
|------|---------|
| `kernel/smp.c` | IPI-based cross-CPU function calls |
| `kernel/smpboot.c` | Per-CPU hotplug thread management, idle thread init |
| `kernel/cpu.c` | CPU hotplug state machine, online/offline paths |
| `kernel/stop_machine.c` | stop_machine and per-CPU stopper threads |
| `kernel/sched/topology.c` | Scheduling domain construction and management |
| `include/linux/smp.h` | SMP API declarations |
| `include/linux/topology.h` | CPU/NUMA topology accessors |
| `include/linux/cpumask.h` | cpumask type and operations |
| `include/linux/cpumask_types.h` | cpumask and cpumask_var_t type definitions |
| `include/linux/sched/topology.h` | sched_domain, sched_group structures |
| `include/linux/sched/sd_flags.h` | SD_* flag definitions with metaflags |
| `include/linux/percpu.h` | Per-CPU allocator and group info |
| `include/linux/percpu-defs.h` | DEFINE_PER_CPU macros and accessors |
| `include/linux/cpuhotplug.h` | cpuhp_state enum and registration API |
| `arch/x86/kernel/smpboot.c` | x86 AP startup, sibling map construction |
