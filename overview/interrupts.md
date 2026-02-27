# Linux Kernel Interrupt Subsystem

## Overview

The Linux interrupt subsystem handles hardware-generated signals and
software-deferred work through a layered architecture. Hardware signals enter
through architecture-specific entry stubs, pass through the Generic IRQ layer
(`kernel/irq/`), invoke per-IRQ chip operations, and execute registered action
handlers. Deferred work is split among softirqs, tasklets, workqueues, and
threaded IRQ handlers using a top-half/bottom-half design.

## Architecture

### Top-Half / Bottom-Half Split

**Top half (hardirq)**: Executes immediately in interrupt context with
interrupts disabled on the local CPU. Must be fast — typically acknowledges the
hardware and schedules deferred work.

**Bottom half**: Deferred work that runs with interrupts re-enabled. Four mechanisms:
- **Softirqs** — statically allocated, per-CPU, highest performance
- **Tasklets** — built on softirqs, per-tasklet serialization (deprecated for new code)
- **Workqueues** — process context, may sleep, managed thread pools
- **Threaded IRQs** — dedicated per-IRQ kernel thread, may sleep

### Context Tracking via `preempt_count`

The `preempt_count` word (`include/linux/preempt.h`) encodes the current
execution context in distinct bit fields:

| Field | Meaning |
|-------|---------|
| NMI bits | In NMI handler |
| HARDIRQ bits | In hardirq handler |
| SOFTIRQ bits | In softirq or BH-disabled section |
| Preempt count | Preemption disable depth |

Context query macros:
```c
in_hardirq()         // currently in hardirq handler
in_softirq()         // in softirq or BH-disabled section
in_serving_softirq() // currently executing a softirq handler
in_interrupt()       // any of the above
in_nmi()             // in NMI context
in_atomic()          // preempt_count != 0
irqs_disabled()      // local IRQs currently disabled
```

### x86 IDT Vector Layout

Defined in `arch/x86/include/asm/irq_vectors.h`:

```
Vectors   0 ..  31 : CPU exceptions and traps (hardcoded)
Vectors  32 .. 127 : device interrupts (FIRST_EXTERNAL_VECTOR = 0x20)
Vector  128 (0x80) : legacy int80 syscall interface
Vectors 129 .. 0xeb: device interrupts
Vectors 0xec.. 0xff: system/special interrupts
```

Key special vectors:
```c
LOCAL_TIMER_VECTOR          0xec   // APIC timer
IRQ_WORK_VECTOR             0xf6   // deferred IRQ work
POSTED_INTR_VECTOR          0xf2   // KVM posted interrupts
CALL_FUNCTION_SINGLE_VECTOR 0xfb   // SMP single-target IPI
CALL_FUNCTION_VECTOR        0xfc   // SMP broadcast IPI
RESCHEDULE_VECTOR           0xfd   // scheduler IPI
ERROR_APIC_VECTOR           0xfe   // APIC error
SPURIOUS_APIC_VECTOR        0xff   // spurious APIC
```

### Generic IRQ Architecture

The Generic IRQ layer (`kernel/irq/`) provides architecture-independent
interrupt management. The architecture registers a top-level handler:

```c
// kernel/irq/handle.c
void (*handle_arch_irq)(struct pt_regs *) __ro_after_init;
int __init set_handle_irq(void (*handle_irq)(struct pt_regs *));
```

On entry, `generic_handle_arch_irq()` (`kernel/irq/handle.c:232`) calls
`irq_enter()`, dispatches via `handle_arch_irq(regs)`, then calls `irq_exit()`.

---

## Key Data Structures

### `struct irq_desc` — Per-IRQ Descriptor

`include/linux/irqdesc.h:67-120`

The central descriptor, one per Linux IRQ number:

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;
    struct irq_data         irq_data;          // per-chip: hwirq, chip, domain
    struct irqstat __percpu *kstat_irqs;        // per-CPU statistics
    irq_flow_handler_t      handle_irq;         // flow handler (level/edge/fasteoi)
    struct irqaction        *action;            // linked list of handlers
    unsigned int            depth;             // nested disable depth
    unsigned int            irq_count;         // for stuck IRQ detection
    unsigned int            irqs_unhandled;    // spurious count
    raw_spinlock_t          lock;              // SMP protection
    const struct cpumask    *affinity_hint;
    cpumask_var_t           pending_mask;       // pending rebalance
    unsigned long           threads_oneshot;    // oneshot thread bitmask
    atomic_t                threads_active;
    wait_queue_head_t       wait_for_threads;   // synchronize_irq()
    struct proc_dir_entry   *dir;              // /proc/irq/N/
    struct mutex            request_mutex;      // serializes request/free
    const char              *name;             // flow handler name
};
```

With `CONFIG_SPARSE_IRQ` (default), descriptors are dynamically allocated and
stored in a maple tree accessed via `irq_to_desc(irq)`. Without it, a static
array `irq_desc[NR_IRQS]` is used.

### `struct irq_chip` — Interrupt Controller Operations

`include/linux/irq.h:501-551`

```c
struct irq_chip {
    const char  *name;              // shown in /proc/interrupts
    unsigned int (*irq_startup)(struct irq_data *data);
    void (*irq_shutdown)(struct irq_data *data);
    void (*irq_enable)(struct irq_data *data);
    void (*irq_disable)(struct irq_data *data);
    void (*irq_ack)(struct irq_data *data);       // acknowledge start
    void (*irq_mask)(struct irq_data *data);
    void (*irq_mask_ack)(struct irq_data *data);
    void (*irq_unmask)(struct irq_data *data);
    void (*irq_eoi)(struct irq_data *data);        // end of interrupt
    int  (*irq_set_affinity)(struct irq_data *data,
             const struct cpumask *dest, bool force);
    int  (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
    int  (*irq_set_wake)(struct irq_data *data, unsigned int on);
    void (*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
    void (*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);
    unsigned long flags;             // IRQCHIP_* flags
};
```

Key `IRQCHIP_*` flags:
```c
IRQCHIP_SET_TYPE_MASKED     // mask before set_type
IRQCHIP_EOI_IF_HANDLED      // only eoi if handled
IRQCHIP_MASK_ON_SUSPEND     // mask during suspend
IRQCHIP_ONESHOT_SAFE        // no mask/unmask for oneshot
IRQCHIP_EOI_THREADED        // eoi on unmask in threaded mode
IRQCHIP_SUPPORTS_NMI        // can handle NMIs
IRQCHIP_IMMUTABLE           // never modify this chip
```

### `struct irq_data` — Per-Chip Data

`include/linux/irq.h:179-190`

```c
struct irq_data {
    u32                  mask;        // precomputed bitmask
    unsigned int         irq;         // Linux virtual IRQ number
    irq_hw_number_t      hwirq;       // hardware IRQ in this domain
    struct irq_common_data *common;   // shared across chip levels
    struct irq_chip      *chip;       // serving irqchip
    struct irq_domain    *domain;     // translation domain
    struct irq_data      *parent_data; // hierarchical parent
    void                 *chip_data;  // chip-private data
};
```

### `struct irq_domain` — Hardware-to-Linux IRQ Translation

`include/linux/irqdomain.h:169-198`

Maps hardware IRQ numbers to Linux virtual IRQ numbers:

```c
struct irq_domain {
    const char              *name;
    const struct irq_domain_ops *ops;
    void                    *host_data;    // controller private data
    struct fwnode_handle    *fwnode;       // firmware node (DT/ACPI)
    struct irq_domain       *parent;      // hierarchical parent
    irq_hw_number_t         hwirq_max;
    struct radix_tree_root  revmap_tree;  // sparse reverse map
    struct irq_data __rcu   *revmap[];    // linear reverse map
};
```

Domain ops (`irq_domain_ops`):
- `map(d, virq, hwirq)` — create/update mapping
- `unmap(d, virq)` — dispose of mapping
- `xlate(d, node, intspec, ...)` — decode DT interrupt specifier
- `alloc(d, virq, nr_irqs, arg)` — allocate IRQs (hierarchical)
- `free(d, virq, nr_irqs)` — free IRQs (hierarchical)
- `activate(d, irqd, reserve)` — activate in hardware
- `translate(d, fwspec, ...)` — decode `irq_fwspec` (preferred over xlate)

### `struct irqaction` — Registered Handler

`include/linux/interrupt.h:122-136`

One per registered handler (multiple for shared IRQs, linked via `next`):

```c
struct irqaction {
    irq_handler_t       handler;       // primary (hardirq) handler
    void                *dev_id;       // device cookie
    struct irqaction    *next;         // shared IRQ chain
    irq_handler_t       thread_fn;     // threaded handler
    struct task_struct  *thread;       // IRQ thread task
    struct irqaction    *secondary;    // secondary action
    unsigned int        irq;
    unsigned int        flags;         // IRQF_* flags
    unsigned long       thread_flags;  // IRQTF_* flags
    unsigned long       thread_mask;   // oneshot tracking
    const char          *name;
};
```

---

## IRQ Flow: Hardware Signal to Handler

### Complete Path

```
1. Device raises interrupt (pin assert or MSI write)
       |
2. CPU enters IDT assembly stub (arch/x86/include/asm/idtentry.h)
   - Saves registers → struct pt_regs
       |
3. irq_enter_rcu()  (kernel/softirq.c:601)
   - preempt_count += HARDIRQ_OFFSET
   - in_hardirq() now returns true
       |
4. handle_arch_irq(regs)  (kernel/irq/handle.c:232)
   - Architecture handler reads HW to determine IRQ
       |
5. generic_handle_domain_irq(domain, hwirq)
   - Translates hwirq → Linux virq via domain reverse map
       |
6. desc->handle_irq(desc)  — the flow handler
   |-- handle_fasteoi_irq  (modern APICs)
   |-- handle_edge_irq     (edge-triggered)
   |-- handle_level_irq    (level-triggered)
       |
7. handle_irq_event(desc)  (kernel/irq/handle.c:202)
   - Releases desc->lock
   - Calls handle_irq_event_percpu()
       |
8. __handle_irq_event_percpu(desc)  (kernel/irq/handle.c:139)
   - Iterates irqaction chain
   - Calls action->handler(irq, dev_id)
   - If IRQ_WAKE_THREAD: wakes action->thread
       |
9. irq_exit()  (kernel/softirq.c:676)
   - preempt_count -= HARDIRQ_OFFSET
   - If !in_interrupt() && local_softirq_pending():
     invokes softirq processing
```

### Flow Handlers

All in `kernel/irq/chip.c`:

**`handle_fasteoi_irq`**: Used for modern APICs. Runs handlers, then calls
`irq_chip->irq_eoi()`. No explicit mask/unmask — the hardware holds off
re-delivery.

**`handle_edge_irq`**: Acks immediately via `irq_ack`, runs handlers.
Re-triggers if a new edge arrived during handling. Must handle the case where
edges queue while handling.

**`handle_level_irq`**: Masks on entry, runs handlers, unmasks on exit. Handles
the case where the interrupt line remains asserted.

---

## Softirqs

### The 10 Softirq Types

`include/linux/interrupt.h:556-569`:

```c
enum {
    HI_SOFTIRQ      = 0,   // high-priority tasklets
    TIMER_SOFTIRQ   = 1,   // timer wheel processing
    NET_TX_SOFTIRQ  = 2,   // network transmit
    NET_RX_SOFTIRQ  = 3,   // network receive
    BLOCK_SOFTIRQ   = 4,   // block device completions
    IRQ_POLL_SOFTIRQ = 5,  // IRQ polling
    TASKLET_SOFTIRQ = 6,   // normal-priority tasklets
    SCHED_SOFTIRQ   = 7,   // scheduler rebalancing
    HRTIMER_SOFTIRQ = 8,   // high-resolution timers
    RCU_SOFTIRQ     = 9,   // RCU callbacks (always last)
    NR_SOFTIRQS     = 10
};
```

The handler array: `static struct softirq_action softirq_vec[NR_SOFTIRQS]` at
`kernel/softirq.c:60`.

### Raising and Processing

**Raising** (must be called with IRQs disabled):
```c
void raise_softirq(unsigned int nr);           // saves/restores IRQ state
void raise_softirq_irqoff(unsigned int nr);    // when IRQs already disabled
void __raise_softirq_irqoff(unsigned int nr);  // raw, no wakeup
```

Sets `local_softirq_pending() |= (1 << nr)` — a per-CPU pending bitmask.

**Registration** (boot time only):
```c
void open_softirq(int nr, void (*action)(void));
```

**Processing** — `handle_softirqs()` at `kernel/softirq.c:518-596`:

1. Read `local_softirq_pending()` into local variable
2. Set SOFTIRQ_OFFSET in preempt_count (BH context)
3. Clear pending bitmask
4. Re-enable IRQs
5. Loop through set bits via `ffs(pending)`, calling each handler
6. Re-check pending; restart if time < 2ms and restarts < 10
7. If still pending, wake `ksoftirqd`

Softirqs run from two contexts:
- **`irq_exit()`** — after every hardirq, if softirqs are pending and we're
  returning to non-interrupt context
- **`ksoftirqd`** — per-CPU kernel thread (`ksoftirqd/%u`), runs at
  `SCHED_OTHER` priority

### `ksoftirqd`

Per-CPU kernel thread registered at `kernel/softirq.c:990-995`:
```c
static struct smp_hotplug_thread softirq_threads = {
    .thread_should_run = ksoftirqd_should_run,  // local_softirq_pending()
    .thread_fn         = run_ksoftirqd,
    .thread_comm       = "ksoftirqd/%u",
};
```

Handles overflow when inline softirq processing runs too long (>2ms or >10 restarts).

---

## Tasklets

Built on `HI_SOFTIRQ` and `TASKLET_SOFTIRQ`. Provide per-tasklet serialization
— a given tasklet runs on only one CPU at a time.

**Note**: Tasklets are deprecated for new code
(`include/linux/interrupt.h:674-676`). Use threaded IRQs instead.

### Data Structure

`include/linux/interrupt.h:696-707`:
```c
struct tasklet_struct {
    struct tasklet_struct *next;     // per-CPU list linkage
    unsigned long         state;     // TASKLET_STATE_SCHED | _RUN
    atomic_t              count;     // disable count; 0 = enabled
    bool                  use_callback;
    union {
        void (*func)(unsigned long data);           // old API
        void (*callback)(struct tasklet_struct *t);  // new API
    };
    unsigned long         data;
};
```

### Key Operations

```c
// Initialize (new API):
tasklet_setup(t, callback);

// Schedule:
tasklet_schedule(t);       // TASKLET_SOFTIRQ
tasklet_hi_schedule(t);    // HI_SOFTIRQ

// Disable/Enable (reference counted):
tasklet_disable(t);        // increment count + wait for completion
tasklet_disable_nosync(t); // increment count only
tasklet_enable(t);         // decrement count

// Destroy:
tasklet_kill(t);           // wait + prevent re-scheduling
```

### Execution Path

`tasklet_action_common()` at `kernel/softirq.c:790-832`:
1. Atomically takes the entire per-CPU tasklet list
2. For each tasklet: tries `tasklet_trylock()` (sets `TASKLET_STATE_RUN`)
3. If locked and enabled (`count == 0`): clears `TASKLET_STATE_SCHED`, calls handler
4. If lock fails or disabled: re-queues and re-raises softirq

---

## Workqueues

Execute deferred work in process context via managed kernel thread pools
(`kworker/...`). The primary mechanism for deferred work that needs to sleep.

### Core Structures

**`struct work_struct`** (`include/linux/workqueue_types.h:16-23`):
```c
struct work_struct {
    atomic_long_t   data;   // pool ID, flags, or pwq pointer
    struct list_head entry; // linkage in pool worklist
    work_func_t     func;   // function to execute
};
```

**`struct worker_pool`** (`kernel/workqueue.c:188-235`):
- Per-CPU pools (2 per CPU: normal + high-priority)
- Unbound pools (per-NUMA node, shared)
- Manages `worklist`, `nr_running`, idle workers, concurrency

**`struct workqueue_struct`** (`kernel/workqueue.c:339-387`):
- Named queue (`name[WQ_NAME_LEN]`)
- Links to per-CPU or per-NUMA `pool_workqueue` structures
- Controls `max_active`, flags, flush coordination

### Workqueue Flags

`include/linux/workqueue.h:370-399`:
```c
WQ_BH              // execute in softirq context
WQ_UNBOUND          // not bound to any CPU
WQ_FREEZABLE        // freeze during suspend
WQ_MEM_RECLAIM      // may be used during memory reclaim (has rescuer)
WQ_HIGHPRI          // high priority workers
WQ_CPU_INTENSIVE    // excluded from concurrency management
WQ_SYSFS            // visible in sysfs
WQ_POWER_EFFICIENT  // becomes unbound if power_efficient=y
```

### APIs

```c
// Create:
alloc_workqueue(fmt, flags, max_active, ...);
create_workqueue(name);                // WQ_MEM_RECLAIM, max_active=1
create_singlethread_workqueue(name);   // ordered

// Initialize work:
INIT_WORK(work, func);
INIT_DELAYED_WORK(work, func);

// Queue:
queue_work(wq, work);
queue_delayed_work(wq, dwork, delay);
schedule_work(work);                   // on system_wq

// Synchronize:
flush_workqueue(wq);
flush_work(work);
cancel_work_sync(work);
cancel_delayed_work_sync(dwork);

// Destroy:
destroy_workqueue(wq);
```

### System Workqueues

Predefined in `kernel/workqueue.c`:
- `system_wq` — general purpose, per-CPU, unbounded concurrency
- `system_highpri_wq` — high priority workers
- `system_long_wq` — long-running work (watchdog exempt)
- `system_unbound_wq` — unbound, no CPU affinity
- `system_freezable_wq` — freezes during suspend
- `system_power_efficient_wq` — power-efficient variant

Worker threads are named `kworker/N:M` (CPU-bound) or `kworker/uN:M` (unbound).

---

## Threaded IRQs

### `request_threaded_irq()`

`include/linux/interrupt.h:151`:
```c
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,      // hardirq handler
                         irq_handler_t thread_fn,    // threaded handler
                         unsigned long flags,
                         const char *name, void *dev);
```

- `handler` runs in hardirq context, returns `IRQ_WAKE_THREAD` to schedule thread
- `thread_fn` runs in a dedicated kernel thread (`irq/%d-%s`), may sleep
- If `handler == NULL`: uses `irq_default_primary_handler` (always returns `IRQ_WAKE_THREAD`); requires `IRQF_ONESHOT`

### IRQ Thread

Created in `setup_irq_thread()` at `kernel/irq/manage.c:1458`:
- Named `irq/%d-%s` (or `irq/%d-s-%s` for secondary)
- Runs at `SCHED_FIFO` priority via `sched_set_fifo(current)`
- Waits in `irq_wait_for_interrupt()` until `IRQTF_RUNTHREAD` is set by hardirq

The thread function (`irq_thread()` at `kernel/irq/manage.c:1300`):
```c
while (!irq_wait_for_interrupt(desc, action)) {
    action_ret = handler_fn(desc, action);
    if (action_ret == IRQ_WAKE_THREAD)
        irq_wake_secondary(desc, action);
    wake_threads_waitq(desc);  // wakes synchronize_irq() waiters
}
```

### ONESHOT Handling

Required for level-triggered threaded IRQs (`IRQF_ONESHOT`):
1. Flow handler keeps IRQ masked after ack
2. `thread_mask` bit set in `desc->threads_oneshot`
3. After thread handler completes, `irq_finalize_oneshot()` clears the bit
4. IRQ unmasked only when all `threads_oneshot` bits clear

### Forced Threading

With `CONFIG_IRQ_FORCED_THREADING` + `threadirqs` kernel parameter (or `CONFIG_PREEMPT_RT`):
- Non-`IRQF_NO_THREAD` handlers automatically converted to threaded
- Original handler becomes `thread_fn`
- `irq_default_primary_handler` becomes the hardirq handler
- Activated in `irq_setup_forced_threading()` at `kernel/irq/manage.c:1368`

---

## IRQ Affinity

### Setting Affinity

```c
int irq_set_affinity(unsigned int irq, const struct cpumask *cpumask);
int irq_force_affinity(unsigned int irq, const struct cpumask *cpumask);
```

`irq_do_set_affinity()` (`kernel/irq/manage.c:223`) calls `chip->irq_set_affinity()` and updates `desc->irq_common_data.affinity`.

### Managed Interrupts (`IRQD_AFFINITY_MANAGED`)

For multi-queue devices (NVMe, RDMA):
- Kernel auto-assigns vectors to CPUs
- Shuts down when target CPUs go offline
- Prevents userspace affinity override
- Set via `irq_update_affinity_desc()` with `affinity->is_managed = 1`

### `/proc/irq/` Interface

`kernel/irq/proc.c` — For each IRQ N:
- `affinity` — current mask (hex bitmap), read/write
- `affinity_list` — CPU list format
- `effective_affinity` — actual effective mask
- `node` — NUMA node
- `spurious` — unhandled count

### Affinity Spreading for MSI-X

`irq_create_affinity_masks()` (`kernel/irq/affinity.c`) distributes vectors across CPUs:
```c
struct irq_affinity {
    unsigned int pre_vectors;    // reserved at start
    unsigned int post_vectors;   // reserved at end
    unsigned int nr_sets;        // number of interrupt sets
    unsigned int set_size[];     // per-set sizes
};
```

---

## Key APIs Reference

### Registration

```c
// Standard:
int request_irq(irq, handler, flags, name, dev);
int request_threaded_irq(irq, handler, thread_fn, flags, name, dev);
const void *free_irq(irq, dev_id);

// Device-managed (auto-freed on device removal):
int devm_request_irq(dev, irq, handler, flags, name, dev_id);
int devm_request_threaded_irq(dev, irq, handler, thread_fn, flags, name, dev_id);

// Per-CPU:
int request_percpu_irq(irq, handler, name, percpu_dev_id);
void free_percpu_irq(irq, percpu_dev_id);

// NMI:
int request_nmi(irq, handler, flags, name, dev);

// Smart context (auto-selects hardirq or nested threaded):
int request_any_context_irq(irq, handler, flags, name, dev_id);
```

### Enable / Disable

```c
void disable_irq(unsigned int irq);         // disable + wait for in-progress
void disable_irq_nosync(unsigned int irq);  // disable without wait
bool disable_hardirq(unsigned int irq);     // disable + wait for hardirq only
void enable_irq(unsigned int irq);          // re-enable (depth-counted)

void synchronize_irq(unsigned int irq);     // wait for hardirq + threads
bool synchronize_hardirq(unsigned int irq); // wait for hardirq only
```

### Local CPU Control

```c
local_irq_disable();         // disable IRQs on local CPU
local_irq_enable();          // re-enable
local_irq_save(flags);      // save state + disable
local_irq_restore(flags);   // restore saved state

local_bh_disable();          // disable softirq processing
local_bh_enable();           // re-enable (runs pending softirqs)
```

### IRQF Flags

`include/linux/interrupt.h:31-90`:
```c
IRQF_SHARED          0x00000080  // allow sharing
IRQF_TIMER           // timer interrupt (no suspend, no threading)
IRQF_PERCPU          0x00000400  // per-CPU interrupt
IRQF_NOBALANCING     0x00000800  // exclude from balancing
IRQF_ONESHOT         0x00002000  // keep masked until thread completes
IRQF_NO_SUSPEND      0x00004000  // don't disable on suspend
IRQF_NO_THREAD       0x00010000  // cannot be threaded
IRQF_NO_AUTOEN       0x00080000  // don't auto-enable on request
IRQF_TRIGGER_RISING  0x00000001  // edge triggers
IRQF_TRIGGER_FALLING 0x00000002
IRQF_TRIGGER_HIGH    0x00000004  // level triggers
IRQF_TRIGGER_LOW     0x00000008
```

### Return Values

```c
typedef enum irqreturn {
    IRQ_NONE        = 0,  // not from this device
    IRQ_HANDLED     = 1,  // handled
    IRQ_WAKE_THREAD = 2,  // wake handler thread
} irqreturn_t;
```

---

## MSI / MSI-X

Message Signaled Interrupts replace pin-based signaling. The device writes a
data word to a memory address (LAPIC on x86) instead of asserting a physical
line. MSI-X supports up to 2048 vectors per device with independent masking.

### MSI Message

`include/linux/msi.h:61-74`:
```c
struct msi_msg {
    u32 address_lo;  // destination LAPIC address (0xFEExx000 on x86)
    u32 address_hi;
    u32 data;        // vector number + delivery mode
};
```

### MSI Domain Hierarchy

MSI uses hierarchical IRQ domains:
1. `pci_alloc_irq_vectors()` triggers `msi_domain_alloc_irqs()`
2. Parent domain (e.g., x86 APIC vector domain) allocates hardware vectors
3. `irq_chip->irq_compose_msi_msg()` or `irq_write_msi_msg()` programs the device
4. Device uses the programmed message to trigger interrupts

---

## Advanced Topics

### Hierarchical IRQ Domains

Modern interrupt controllers form hierarchies (GPIO → GIC → CPU). Each level
has its own domain and chip. `irq_data->parent_data` links the levels.

Activation walks up the chain calling each domain's `activate()`. Helper
functions for hierarchical chips:
```c
irq_chip_enable_parent()      irq_chip_mask_parent()
irq_chip_disable_parent()     irq_chip_unmask_parent()
irq_chip_ack_parent()         irq_chip_eoi_parent()
irq_chip_set_affinity_parent()
```

### Generic IRQ Chip

`kernel/irq/generic-chip.c` — reusable implementation for simple memory-mapped
controllers. Controllers allocate `struct irq_chip_generic` and fill in
register offsets for enable/disable/mask/ack.

### Spurious IRQ Detection

`note_interrupt()` in `kernel/irq/spurious.c` — tracks `IRQ_NONE` returns. If
unhandled rate exceeds threshold (10,000 out of 100,000), the IRQ is disabled
with a warning. Controlled by `noirqdebug` kernel parameter.

### `irq_matrix` — Vector Allocation

`kernel/irq/matrix.c` — per-CPU bit matrix for allocating interrupt vectors.
Used by x86's vector allocation domain. Supports managed reservations that
pre-allocate across a CPU set.

### `/proc/interrupts`

Generated by `show_interrupts()` in `kernel/irq/proc.c`. Shows per-CPU counts,
chip name, flow handler, and action names. Architecture-specific entries (NMI,
LOC, etc.) appended by `arch_show_interrupts()`.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `kernel/softirq.c` | Softirq engine, tasklets, ksoftirqd |
| `kernel/irq/handle.c` | `handle_irq_event`, `generic_handle_arch_irq` |
| `kernel/irq/manage.c` | `request_threaded_irq`, `free_irq`, enable/disable, affinity |
| `kernel/irq/chip.c` | Flow handlers: level, edge, fasteoi |
| `kernel/irq/irqdesc.c` | `irq_desc` allocation, maple tree |
| `kernel/irq/irqdomain.c` | Domain creation, mapping, hierarchy |
| `kernel/irq/proc.c` | `/proc/irq/` and `/proc/interrupts` |
| `kernel/irq/affinity.c` | MSI affinity spreading |
| `kernel/irq/msi.c` | MSI/MSI-X descriptor management |
| `kernel/irq/generic-chip.c` | Generic chip framework |
| `kernel/irq/spurious.c` | Spurious IRQ detection |
| `kernel/irq/matrix.c` | Vector allocation matrix |
| `kernel/workqueue.c` | Workqueue implementation |
| `include/linux/interrupt.h` | IRQ flags, irqaction, softirq types, APIs |
| `include/linux/irq.h` | irq_chip, irq_data, flow handlers |
| `include/linux/irqdesc.h` | irq_desc |
| `include/linux/irqdomain.h` | irq_domain, domain ops |
| `include/linux/hardirq.h` | irq_enter/exit, context macros |
| `arch/x86/include/asm/irq_vectors.h` | x86 IDT vector layout |

---

## Subsystem Invariants

- `desc->lock` (raw_spinlock_t) protects all `irq_desc` modifications; held during flow handlers
- `desc->request_mutex` (mutex) serializes `request_irq`/`free_irq`
- Softirq handlers run with IRQs enabled but preemption disabled
- A tasklet's `TASKLET_STATE_RUN` bit ensures single-CPU execution
- `ksoftirqd` runs at `SCHED_OTHER`; IRQ threads run at `SCHED_FIFO`
- `depth` in `irq_desc` is the nested disable counter; `enable_irq()` only starts the IRQ when depth reaches 0
- Managed affinity (`IRQD_AFFINITY_MANAGED`) cannot be overridden from userspace
- Softirq processing is bounded: max 2ms or 10 restarts before deferring to `ksoftirqd`
