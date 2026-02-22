# Linux Kernel RCU (Read-Copy-Update) Deep Dive

## Table of Contents
1. [Introduction](#introduction)
2. [RCU Fundamentals](#rcu-fundamentals)
3. [Grace Periods](#grace-periods)
4. [RCU API](#rcu-api)
5. [RCU Flavors](#rcu-flavors)
6. [Tree RCU Implementation](#tree-rcu-implementation)
7. [RCU and the Scheduler](#rcu-and-the-scheduler)
8. [SRCU (Sleepable RCU)](#srcu-sleepable-rcu)
9. [RCU Debugging](#rcu-debugging)
10. [RCU Use Cases](#rcu-use-cases)

---

## Introduction

RCU (Read-Copy-Update) is a synchronization mechanism optimized for read-mostly
scenarios. It allows concurrent reads with updates by ensuring readers see
either the old or new version of data, never a partially updated state.

### RCU vs Traditional Locking

```
┌─────────────────────────────────────────────────────────────┐
│              RCU vs Traditional Locking                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Reader-Writer Lock:                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Writers: Exclusive access (blocks all)              │    │
│  │ Readers: Shared access (blocks writers)             │    │
│  │                                                     │    │
│  │ Problem: Reader-side overhead                       │    │
│  │ - Atomic operations for each read                   │    │
│  │ - Cache-line bouncing on reader count               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  RCU:                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Readers: No synchronization overhead!               │    │
│  │ - Just disable preemption (or nothing on UP)        │    │
│  │ - No atomic operations                              │    │
│  │ - No cache-line bouncing                            │    │
│  │                                                     │    │
│  │ Writers: Wait for all readers to finish             │    │
│  │ - Create new version                                │    │
│  │ - Atomically switch pointer                         │    │
│  │ - Wait for grace period                             │    │
│  │ - Free old version                                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Trade-off:                                                 │
│  - RCU readers: O(1), nearly zero overhead                  │
│  - RCU writers: More expensive (memory + grace period)      │
│  - Best for: Read-mostly, infrequent updates                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## RCU Fundamentals

### Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                   RCU Core Concepts                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Read-Side Critical Section                              │
│     ────────────────────────                                │
│     rcu_read_lock();                                        │
│     /* Access RCU-protected data */                         │
│     /* CANNOT BLOCK! */                                     │
│     rcu_read_unlock();                                      │
│                                                             │
│  2. Grace Period                                            │
│     ────────────                                            │
│     Period during which all pre-existing readers complete   │
│     After grace period: safe to free old data               │
│                                                             │
│     Time ─────────────────────────────────────────▶         │
│     │                                                       │
│     │ [Reader A starts]                                     │
│     │      │                                                │
│     │ [Update published] ─── Grace Period Starts ───        │
│     │      │                          │                     │
│     │      [Reader A ends]            │                     │
│     │                                 │                     │
│     │                    [Reader B starts/ends]             │
│     │                                 │                     │
│     │                   ─── Grace Period Ends ───           │
│     │                         [Safe to free old data]       │
│                                                             │
│  3. Publish-Subscribe Pattern                               │
│     ────────────────────────                                │
│     Writer:                                                 │
│       new = allocate_and_init();                            │
│       rcu_assign_pointer(ptr, new);  /* Publish */          │
│       synchronize_rcu();              /* Wait */            │
│       kfree(old);                     /* Reclaim */         │
│                                                             │
│     Reader:                                                 │
│       rcu_read_lock();                                      │
│       p = rcu_dereference(ptr);      /* Subscribe */        │
│       /* use p */                                           │
│       rcu_read_unlock();                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Memory Ordering

```c
/*
 * rcu_assign_pointer() - Publish a pointer to RCU-protected data
 * Contains a memory barrier to ensure:
 * 1. Initialization of new structure is complete
 * 2. Before pointer is visible to readers
 */
#define rcu_assign_pointer(p, v) \
    do { \
        smp_store_release(&(p), (v)); \
    } while (0)

/*
 * rcu_dereference() - Subscribe to RCU-protected pointer
 * Contains a data dependency barrier to ensure:
 * 1. Pointer is loaded before dereferencing
 * 2. All accesses through pointer see initialization
 */
#define rcu_dereference(p) \
    ({ \
        typeof(p) _p = READ_ONCE(p); \
        smp_read_barrier_depends(); \
        _p; \
    })

/* On most architectures, rcu_dereference is essentially free */
```

---

## Grace Periods

### Grace Period Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                    Grace Period                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU 0    CPU 1    CPU 2    CPU 3                           │
│    │        │        │        │                             │
│    ├────────┼────────┼────────┼────── Update published      │
│    │        │        │        │        (rcu_assign_pointer) │
│    │        │        │        │                             │
│    │   ┌────┴────┐   │        │   Pre-existing reader       │
│    │   │ Reader  │   │        │                             │
│    │   │ in      │   │        │                             │
│    │   │ progress│   │        │                             │
│    │   └────┬────┘   │        │                             │
│    │        │        │        │                             │
│    ├────────┼────────┼────────┼──────────────────           │
│    │        │        │        │                             │
│    │  QS    │   QS   │   QS   │   QS   Quiescent states     │
│    │        │        │        │        (all CPUs passed     │
│    │        │        │        │         through)            │
│    │        │        │        │                             │
│    ├────────┼────────┼────────┼────── Grace period ends    │
│    │        │        │        │        (synchronize_rcu     │
│    │        │        │        │         returns)            │
│    │        │        │        │                             │
│    │        │        ┌────┴────┐   New reader (after GP)   │
│    │        │        │ Reader  │   sees new version        │
│    │        │        └────┬────┘                           │
│    ▼        ▼        ▼    ▼                                │
│                                                             │
│  QS = Quiescent State                                       │
│       (context switch, idle, user mode)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Quiescent States

```
┌─────────────────────────────────────────────────────────────┐
│                   Quiescent States                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  A quiescent state (QS) indicates a CPU is not in an        │
│  RCU read-side critical section.                            │
│                                                             │
│  Quiescent states include:                                  │
│  ─────────────────────────                                  │
│  1. Context switch (switched to another task)               │
│  2. Returning to user mode                                  │
│  3. CPU idle (nothing running)                              │
│  4. Offline CPU                                             │
│  5. nohz_full CPU in userspace                              │
│                                                             │
│  NOT a quiescent state:                                     │
│  ──────────────────────                                     │
│  - Inside rcu_read_lock() section                           │
│  - Interrupt handler (might nest in read section)           │
│  - NMI handler                                              │
│                                                             │
│  Grace Period Completion:                                   │
│  ────────────────────────                                   │
│  When ALL CPUs have passed through at least one             │
│  quiescent state since the grace period started             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## RCU API

### Read-Side API

```c
/* include/linux/rcupdate.h */

/* Enter RCU read-side critical section */
void rcu_read_lock(void);
void rcu_read_unlock(void);

/* Check if in RCU read-side critical section */
bool rcu_read_lock_held(void);

/* Dereference RCU-protected pointer */
#define rcu_dereference(p)          /* Normal read-side */
#define rcu_dereference_bh(p)       /* BH context */
#define rcu_dereference_sched(p)    /* Scheduler context */
#define rcu_dereference_check(p, c) /* With custom check */
#define rcu_dereference_protected(p, c) /* Write-side only */

/* Example read-side usage */
struct my_data *data;

rcu_read_lock();
data = rcu_dereference(global_ptr);
if (data) {
    /* Access data->field1, data->field2, etc. */
    do_something(data);
}
rcu_read_unlock();
```

### Write-Side API

```c
/* include/linux/rcupdate.h */

/* Wait for grace period (blocking) */
void synchronize_rcu(void);

/* Queue callback for after grace period (non-blocking) */
void call_rcu(struct rcu_head *head,
              rcu_callback_t func);

/* Assign RCU-protected pointer */
#define rcu_assign_pointer(p, v)

/* Check if OK to access without RCU (holding update-side lock) */
#define rcu_dereference_protected(p, lockdep_cond)

/* Example write-side usage */
void update_data(struct my_data *new_data)
{
    struct my_data *old;

    /* Allocate and initialize new version */
    new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
    /* ... initialize new_data ... */

    /* Atomically publish new version */
    old = rcu_dereference_protected(global_ptr,
                                     lockdep_is_held(&my_lock));
    rcu_assign_pointer(global_ptr, new_data);

    /* Wait for readers and free old version */
    synchronize_rcu();
    kfree(old);
}

/* Asynchronous version (doesn't block) */
void update_data_async(struct my_data *new_data)
{
    struct my_data *old;

    new_data = kmalloc(sizeof(*new_data), GFP_KERNEL);
    /* ... initialize ... */

    old = rcu_replace_pointer(global_ptr, new_data, true);

    /* Free after grace period - doesn't block! */
    call_rcu(&old->rcu_head, my_free_callback);
}

static void my_free_callback(struct rcu_head *head)
{
    struct my_data *data = container_of(head, struct my_data, rcu_head);
    kfree(data);
}
```

### List Operations

```c
/* include/linux/rculist.h */

/* Add to RCU-protected list */
void list_add_rcu(struct list_head *new, struct list_head *head);
void list_add_tail_rcu(struct list_head *new, struct list_head *head);

/* Delete from RCU-protected list */
void list_del_rcu(struct list_head *entry);

/* Replace in RCU-protected list */
void list_replace_rcu(struct list_head *old, struct list_head *new);

/* Iterate over RCU-protected list (read-side) */
#define list_for_each_entry_rcu(pos, head, member)

/* Example list usage */
struct my_list_entry {
    struct list_head list;
    int value;
    struct rcu_head rcu;
};

/* Reader */
rcu_read_lock();
list_for_each_entry_rcu(entry, &my_list, list) {
    process_entry(entry);
}
rcu_read_unlock();

/* Writer (with external lock) */
spin_lock(&my_list_lock);
list_del_rcu(&old_entry->list);
spin_unlock(&my_list_lock);
call_rcu(&old_entry->rcu, free_entry_callback);
```

---

## RCU Flavors

Linux has several RCU flavors optimized for different contexts:

### RCU Flavor Comparison

```
┌─────────────────────────────────────────────────────────────┐
│                    RCU Flavors                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RCU (rcu_read_lock)                                        │
│  ───────────────────                                        │
│  - Preemptible (CONFIG_PREEMPT_RCU)                         │
│  - Most common flavor                                       │
│  - Read sections can be preempted (with PREEMPT_RCU)        │
│  - Cannot block/sleep                                       │
│                                                             │
│  RCU-bh (rcu_read_lock_bh)                                  │
│  ─────────────────────────                                  │
│  - Disables bottom halves                                   │
│  - Faster grace periods in softirq context                  │
│  - For networking code (was, now mostly unified)            │
│                                                             │
│  RCU-sched (rcu_read_lock_sched)                            │
│  ───────────────────────────────                            │
│  - Disables preemption                                      │
│  - For scheduler-related code                               │
│  - Non-preemptible critical sections                        │
│                                                             │
│  SRCU (srcu_read_lock)                                      │
│  ─────────────────────                                      │
│  - Sleepable RCU                                            │
│  - Read sections CAN block                                  │
│  - Higher overhead                                          │
│  - Per-structure srcu_struct                                │
│                                                             │
│  Tasks RCU                                                  │
│  ─────────                                                  │
│  - For trampoline code                                      │
│  - Voluntary context switch = quiescent state               │
│                                                             │
│  Since Linux 5.0: RCU, RCU-bh, and RCU-sched unified        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Flavor Selection

```c
/* Most common: regular RCU */
rcu_read_lock();
p = rcu_dereference(ptr);
rcu_read_unlock();

/* Softirq/bottom-half context */
rcu_read_lock_bh();
p = rcu_dereference_bh(ptr);
rcu_read_unlock_bh();

/* Scheduler context (non-preemptible) */
rcu_read_lock_sched();
p = rcu_dereference_sched(ptr);
rcu_read_unlock_sched();

/* When you need to sleep in read section */
int idx = srcu_read_lock(&my_srcu);
/* Can sleep here */
p = srcu_dereference(ptr, &my_srcu);
srcu_read_unlock(&my_srcu, idx);
```

---

## Tree RCU Implementation

Tree RCU is the scalable RCU implementation for large systems.

### Tree Structure

```
┌─────────────────────────────────────────────────────────────┐
│                   Tree RCU Structure                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  For a system with 16 CPUs and fanout of 4:                 │
│                                                             │
│                    ┌────────────┐                           │
│                    │   Root     │                           │
│                    │  rcu_node  │                           │
│                    └─────┬──────┘                           │
│            ┌─────────────┼─────────────┐                    │
│            ▼             ▼             ▼                    │
│      ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│      │ rcu_node │  │ rcu_node │  │ rcu_node │  ...          │
│      │  (leaf)  │  │  (leaf)  │  │  (leaf)  │               │
│      └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│    ┌──┬──┬──┐    ┌──┬──┬──┐    ┌──┬──┬──┐                   │
│    ▼  ▼  ▼  ▼    ▼  ▼  ▼  ▼    ▼  ▼  ▼  ▼                   │
│   CPU0 1 2 3    CPU4 5 6 7    CPU8 9 10 11   ...            │
│   rcu_data      rcu_data      rcu_data                      │
│                                                             │
│  Each CPU has an rcu_data structure                         │
│  Leaf rcu_node covers a group of CPUs                       │
│  Tree reduces lock contention for GP tracking               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Data Structures

The rcu_data structure is the per-CPU data structure that tracks RCU state for
each processor. Every CPU in the system has its own rcu_data instance, which
maintains quiescent state tracking, callback lists, and dyntick-idle
information. This structure is the workhorse of Tree RCU, handling the local
bookkeeping needed to determine when a CPU has passed through a quiescent state
and managing the segmented callback list for deferred work. The key fields
include gp_seq for tracking grace period progression, cblist for organizing
callbacks by the grace period they're waiting on, and mynode which links this
CPU's data to its leaf rcu_node in the hierarchy.

```c
/* kernel/rcu/tree.h */

/* Per-CPU RCU data */
struct rcu_data {
    /* Quiescent state tracking */
    unsigned long gp_seq;           /* Last GP seen */
    unsigned long gp_seq_needed;    /* GP required for CBs */
    bool cpu_no_qs;                 /* No QS since last GP */
    bool core_needs_qs;             /* Core waiting for QS */

    /* Callback handling */
    struct rcu_segcblist cblist;    /* Segmented CB list */
    long qlen_last_fqs_check;       /* CB count at last FQS */

    /* Dyntick-idle tracking */
    int dynticks_snap;              /* Snapshot for GP */
    unsigned long dynticks_nesting; /* Nesting depth */

    /* CPU index */
    int cpu;
    struct rcu_node *mynode;        /* My leaf rcu_node */

    /* ... more fields ... */
};

The rcu_node structure represents a node in the Tree RCU hierarchy, which is
used to scalably track quiescent states across many CPUs. The tree structure
reduces lock contention by organizing CPUs into groups, where each leaf
rcu_node covers a subset of CPUs and interior nodes aggregate quiescent state
information up to the root. When all CPUs covered by a leaf node have reported
quiescent states, that information propagates upward through the tree. Key
fields include qsmask which is a bitmask of CPUs or child nodes that still need
to report quiescent states, gp_seq for grace period tracking, and the parent
pointer which links nodes in the hierarchy.

/* Hierarchy node */
struct rcu_node {
    raw_spinlock_t lock;

    /* Grace period tracking */
    unsigned long gp_seq;
    unsigned long gp_seq_needed;

    /* Quiescent state mask */
    unsigned long qsmask;           /* CPUs needing QS */
    unsigned long qsmaskinit;       /* Initial QS mask */

    /* Children */
    unsigned long grpmask;          /* Mask in parent */
    int grplo;                      /* Low CPU number */
    int grphi;                      /* High CPU number */
    u8 grpnum;                      /* Group number */
    u8 level;                       /* Tree level */

    /* Parent and children */
    struct rcu_node *parent;

    /* ... more fields ... */
};

The rcu_state structure is the global RCU state container that manages the
entire Tree RCU subsystem. There is one rcu_state instance for the system, and
it holds the array of all rcu_node structures, the current grace period
sequence number, and the grace period kernel thread. This structure coordinates
grace period initiation, tracks the maximum observed grace period latency, and
manages expedited grace period requests. The level array provides quick access
to each level of the rcu_node hierarchy, while gp_kthread points to the kernel
thread responsible for driving grace period processing.

/* Global RCU state */
struct rcu_state {
    struct rcu_node *level[RCU_NUM_LVLS];
    struct rcu_node *node;          /* All nodes array */
    int ncpus;                      /* Number of CPUs */
    int n_online_cpus;              /* Online CPU count */

    /* Grace period state */
    unsigned long gp_seq;           /* Current GP sequence */
    unsigned long gp_max;           /* Max GP latency */

    /* GP kthread */
    struct task_struct *gp_kthread;
    struct swait_queue_head gp_wq;

    /* Expedited GP */
    atomic_t expedited_need_qs;

    /* ... more fields ... */
};
```

### Grace Period State Machine

```
┌─────────────────────────────────────────────────────────────┐
│               Grace Period State Machine                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                     RCU_GP_IDLE                     │    │
│  │              No grace period in progress            │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           │ Callbacks pending               │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   RCU_GP_WAIT_GPS                   │    │
│  │            Waiting for GP kthread to start          │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    RCU_GP_INIT                      │    │
│  │         Initialize GP, set up quiescent masks       │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  RCU_GP_WAIT_FQS                    │    │
│  │     Wait for Force Quiescent State (periodic)       │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           │                                 │
│              ┌────────────┴────────────┐                    │
│              ▼                         ▼                    │
│    ┌──────────────────┐    ┌──────────────────────────┐     │
│    │    RCU_GP_FQS    │    │ All QS reported          │     │
│    │ Force QS check   │    │                          │     │
│    └────────┬─────────┘    └────────────┬─────────────┘     │
│             │                           │                   │
│             └───────────┬───────────────┘                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  RCU_GP_CLEANUP                     │    │
│  │    Advance callbacks, prepare next GP if needed     │    │
│  └────────────────────────┬────────────────────────────┘    │
│                           ▼                                 │
│                    Back to RCU_GP_IDLE                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## RCU and the Scheduler

### Scheduler Integration

```
┌─────────────────────────────────────────────────────────────┐
│               RCU and Scheduler Integration                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Context Switch (schedule())                                │
│  ─────────────────────────                                  │
│  When a task switches off a CPU:                            │
│  1. rcu_note_context_switch() called                        │
│  2. Reports quiescent state for that CPU                    │
│  3. Checks for pending callbacks                            │
│                                                             │
│  Idle (cpuidle)                                             │
│  ─────────────                                              │
│  When CPU goes idle:                                        │
│  1. rcu_idle_enter() marks CPU as in extended QS            │
│  2. Dyntick-idle counter incremented                        │
│  3. CPU can sleep deeply (no tick needed)                   │
│                                                             │
│  Interrupt Entry/Exit                                       │
│  ────────────────────                                       │
│  On interrupt:                                              │
│  1. rcu_irq_enter() / rcu_irq_exit()                        │
│  2. Track when interrupts nest in idle                      │
│  3. Ensure idle CPU counted correctly                       │
│                                                             │
│  User Mode Return                                           │
│  ────────────────                                           │
│  When returning to user space:                              │
│  1. Implicit quiescent state                                │
│  2. rcu_user_enter() for nohz_full                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Preemptible RCU

```c
/* CONFIG_PREEMPT_RCU */

/*
 * With preemptible RCU, tasks can be preempted while
 * in an RCU read-side critical section.
 *
 * The task is tracked so the grace period waits for it.
 */

void rcu_read_lock(void)
{
    /* Increment nesting count */
    current->rcu_read_lock_nesting++;
    barrier();
}

void rcu_read_unlock(void)
{
    barrier();
    if (--current->rcu_read_lock_nesting == 0) {
        /* If preempted while in critical section,
         * need to report to RCU */
        if (unlikely(current->rcu_read_unlock_special.s))
            rcu_read_unlock_special(current);
    }
}

/*
 * When a task is preempted in RCU read-side CS:
 * 1. Task is queued on the rcu_node's blocked list
 * 2. Grace period cannot complete until task exits CS
 * 3. rcu_read_unlock() removes task from list
 */
```

---

## SRCU (Sleepable RCU)

SRCU allows sleeping inside read-side critical sections.

### SRCU Structure

The srcu_struct structure is the core data structure for Sleepable RCU (SRCU),
which allows read-side critical sections to block or sleep. Unlike regular RCU
which has a single global grace period domain, each srcu_struct defines its own
independent grace period domain, meaning synchronize_srcu() only waits for
readers of that specific srcu_struct instance. This structure contains its own
hierarchy of srcu_node structures for scalable grace period tracking, per-CPU
srcu_data for reader counting, and synchronization primitives like mutexes for
coordinating grace period processing. The srcu_idx field implements a two-phase
reader tracking scheme, and the work field supports workqueue-based callback
processing.

```c
/* include/linux/srcu.h */
struct srcu_struct {
    struct srcu_node *node;
    struct srcu_node *level[RCU_NUM_LVLS];
    struct mutex srcu_cb_mutex;
    spinlock_t lock;
    struct mutex srcu_gp_mutex;
    unsigned int srcu_idx;          /* Current read-side index */
    unsigned long srcu_gp_seq;      /* GP sequence number */
    unsigned long srcu_gp_seq_needed;
    unsigned long srcu_gp_seq_needed_exp;
    unsigned long srcu_last_gp_end;
    struct srcu_data __percpu *sda;
    unsigned long srcu_barrier_seq;
    struct mutex srcu_barrier_mutex;
    struct completion srcu_barrier_completion;
    atomic_t srcu_barrier_cpu_cnt;
    struct delayed_work work;
    struct lockdep_map dep_map;
};
```

### SRCU API

```c
/* Define SRCU structure */
DEFINE_SRCU(my_srcu);
/* or dynamically */
struct srcu_struct my_srcu;
int init_srcu_struct(&my_srcu);
void cleanup_srcu_struct(&my_srcu);

/* Read-side */
int idx;
idx = srcu_read_lock(&my_srcu);
/* Can sleep here! */
p = srcu_dereference(ptr, &my_srcu);
/* ... use p, can call kmalloc, mutex_lock, etc. ... */
srcu_read_unlock(&my_srcu, idx);

/* Write-side */
synchronize_srcu(&my_srcu);
/* or */
call_srcu(&my_srcu, &obj->rcu_head, callback);
```

### SRCU vs RCU

```
┌─────────────────────────────────────────────────────────────┐
│                    SRCU vs RCU                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Feature              RCU          SRCU                     │
│  ───────              ───          ────                     │
│  Sleep in read        No           Yes                      │
│  Read overhead        Very low     Per-CPU counter          │
│  Grace period scope   Global       Per srcu_struct          │
│  Max read duration    Bounded      Unbounded                │
│                                                             │
│  Use SRCU when:                                             │
│  - Read sections need to block                              │
│  - Can tolerate slightly higher overhead                    │
│  - Want separate GP domains                                 │
│                                                             │
│  Use RCU when:                                              │
│  - Maximum read performance needed                          │
│  - Read sections are short                                  │
│  - Cannot sleep in read section                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## RCU Debugging

### RCU Lockdep

```c
/* Assert that RCU read lock is held */
RCU_LOCKDEP_WARN(!rcu_read_lock_held(),
                 "RCU read lock required");

/* Assert caller holds update-side lock */
struct my_data *p = rcu_dereference_protected(ptr,
    lockdep_is_held(&my_lock));

/* Check-enabled dereference */
struct my_data *p = rcu_dereference_check(ptr,
    lockdep_is_held(&my_lock) || rcu_read_lock_held());

/* Sparse checking annotations */
void __rcu *ptr;  /* Pointer protected by RCU */
```

### RCU Stall Warnings

```
┌─────────────────────────────────────────────────────────────┐
│                  RCU Stall Detection                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RCU tracks grace period duration and warns if stuck:       │
│                                                             │
│  "rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:"    │
│  "    0-....: (1 GPs behind) idle=..."                      │
│  "    (detected by 3, t=2502 jiffies, g=12345, q=100)"      │
│                                                             │
│  Common causes:                                             │
│  ─────────────                                              │
│  1. Infinite loop in kernel code                            │
│  2. IRQs disabled for too long                              │
│  3. Preemption disabled for too long                        │
│  4. Blocking in RCU read-side critical section              │
│  5. RT task starving RCU kthreads                           │
│                                                             │
│  Tuning stall detection:                                    │
│  $ echo 60 > /sys/module/rcupdate/parameters/rcu_cpu_stall_timeout│
│                                                             │
│  Debugging:                                                 │
│  - Check dmesg for full stall report                        │
│  - Look for stack traces of stuck CPUs                      │
│  - Use RCU tracepoints                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### RCU Tracing

```bash
# Enable RCU tracepoints
cd /sys/kernel/debug/tracing
echo 1 > events/rcu/enable

# Key tracepoints:
# - rcu_grace_period: GP start/end
# - rcu_quiescent_state_report: QS reported
# - rcu_callback: Callback queued/invoked
# - rcu_stall_warning: Stall detected

# View trace
cat trace
```

---

## RCU Use Cases

### Use Case 1: Read-Mostly Data Structure

```c
/* Example: Configuration that's read frequently, rarely updated */

struct config {
    int setting1;
    int setting2;
    char name[64];
    struct rcu_head rcu;
};

static struct config __rcu *current_config;
static DEFINE_SPINLOCK(config_lock);

/* Reader - hot path, called frequently */
int read_setting1(void)
{
    struct config *cfg;
    int val;

    rcu_read_lock();
    cfg = rcu_dereference(current_config);
    val = cfg->setting1;
    rcu_read_unlock();

    return val;
}

/* Writer - cold path, called rarely */
void update_config(int new_setting1)
{
    struct config *new_cfg, *old_cfg;

    new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);

    spin_lock(&config_lock);
    old_cfg = rcu_dereference_protected(current_config,
                lockdep_is_held(&config_lock));

    /* Copy and modify */
    *new_cfg = *old_cfg;
    new_cfg->setting1 = new_setting1;

    rcu_assign_pointer(current_config, new_cfg);
    spin_unlock(&config_lock);

    /* Free old after grace period */
    call_rcu(&old_cfg->rcu, config_free_callback);
}
```

### Use Case 2: RCU-Protected List

```c
/* Example: List of network devices */

struct net_device_entry {
    struct list_head list;
    struct net_device *dev;
    struct rcu_head rcu;
};

static LIST_HEAD(device_list);
static DEFINE_SPINLOCK(device_list_lock);

/* Reader - lookup device */
struct net_device *find_device(const char *name)
{
    struct net_device_entry *entry;
    struct net_device *dev = NULL;

    rcu_read_lock();
    list_for_each_entry_rcu(entry, &device_list, list) {
        if (strcmp(entry->dev->name, name) == 0) {
            dev = entry->dev;
            dev_hold(dev);  /* Get reference */
            break;
        }
    }
    rcu_read_unlock();

    return dev;
}

/* Writer - add device */
void add_device(struct net_device *dev)
{
    struct net_device_entry *entry;

    entry = kmalloc(sizeof(*entry), GFP_KERNEL);
    entry->dev = dev;

    spin_lock(&device_list_lock);
    list_add_rcu(&entry->list, &device_list);
    spin_unlock(&device_list_lock);
}

/* Writer - remove device */
void remove_device(struct net_device *dev)
{
    struct net_device_entry *entry;

    spin_lock(&device_list_lock);
    list_for_each_entry(entry, &device_list, list) {
        if (entry->dev == dev) {
            list_del_rcu(&entry->list);
            spin_unlock(&device_list_lock);
            call_rcu(&entry->rcu, entry_free_callback);
            return;
        }
    }
    spin_unlock(&device_list_lock);
}
```

### Use Case 3: RCU with Reference Counting

```c
/* Pattern: RCU + refcount for safe object lifetime */

struct my_object {
    refcount_t refcount;
    struct rcu_head rcu;
    /* ... data ... */
};

/* Get reference (in RCU read section) */
struct my_object *get_object_rcu(void)
{
    struct my_object *obj;

    rcu_read_lock();
    obj = rcu_dereference(global_obj);
    if (obj && !refcount_inc_not_zero(&obj->refcount))
        obj = NULL;  /* Object being freed */
    rcu_read_unlock();

    return obj;  /* Caller owns reference */
}

/* Drop reference */
void put_object(struct my_object *obj)
{
    if (refcount_dec_and_test(&obj->refcount))
        call_rcu(&obj->rcu, object_free_callback);
}
```

---

## Summary

RCU is a powerful synchronization mechanism for read-mostly data:

| Feature | Description |
|---------|-------------|
| Read overhead | Nearly zero (disable preemption) |
| Write cost | Memory allocation + grace period |
| Blocking | Readers cannot block (except SRCU) |
| Grace period | All pre-existing readers must complete |
| Best for | Read-heavy, write-rare workloads |

Key APIs:
| Read Side | Write Side |
|-----------|------------|
| rcu_read_lock() | rcu_assign_pointer() |
| rcu_dereference() | synchronize_rcu() |
| rcu_read_unlock() | call_rcu() |

RCU is used extensively in the kernel for:
- Network routing tables
- File system dentries
- Module lists
- PID lookup
- Many more read-mostly data structures
