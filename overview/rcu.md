# Linux Kernel RCU (Read-Copy-Update) Overview

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
11. [Tree RCU Node Hierarchy](#tree-rcu-node-hierarchy)
12. [Grace Period Machinery](#grace-period-machinery)
13. [Quiescent State Reporting](#quiescent-state-reporting)
14. [Callback Management](#callback-management)
15. [Dynticks Integration](#dynticks-integration)
16. [Locking Summary](#locking-summary)
17. [Source File Reference](#source-file-reference)

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

## Tree RCU Node Hierarchy

### `struct rcu_state` — Key Fields
**File:** `kernel/rcu/tree.h:340–427`

```c
struct rcu_state {
    struct rcu_node node[NUM_RCU_NODES];    /* All rcu_node structs (heap-order dense array) */
    struct rcu_node *level[RCU_NUM_LVLS + 1]; /* First node at each level */
    int             ncpus;                  /* # CPUs seen so far */
    int             n_online_cpus;          /* # CPUs currently online for RCU */

    /* Guarded by root rcu_node's lock: */
    unsigned long   gp_seq;                 /* Grace-period sequence number */
    unsigned long   gp_max;                 /* Maximum GP duration in jiffies */
    struct task_struct *gp_kthread;         /* The grace-period kthread */
    struct swait_queue_head gp_wq;          /* Where GP kthread waits */
    short           gp_flags;              /* Commands for GP kthread */
    short           gp_state;              /* GP kthread sleep state */
    unsigned long   expedited_sequence;     /* Expedited GP sequence counter */
    atomic_t        expedited_need_qs;      /* # CPUs left to check in for exp GP */
    struct swait_queue_head expedited_wq;   /* Wait for expedited GP check-ins */

    /* Barrier tracking: */
    struct mutex    barrier_mutex;          /* Guards barrier fields */
    unsigned long   barrier_sequence;       /* Incremented at barrier start/end */
    raw_spinlock_t  barrier_lock;           /* Protects ->barrier_seq_snap */

    /* Expedited serialization: */
    struct mutex    exp_mutex;              /* Serialize expedited GPs */
    struct mutex    exp_wake_mutex;         /* Serialize expedited GP wakeup */

    /* FQS timing: */
    unsigned long   jiffies_force_qs;       /* Time to call force_quiescent_state() */
    unsigned long   n_force_qs;             /* FQS call count */
    unsigned long   gp_start;              /* GP start time in jiffies */
    unsigned long   gp_end;                /* Last GP end time in jiffies */

    arch_spinlock_t ofl_lock;              /* Synchronize offline with GP init */
};
```

Key `gp_flags` values (`tree.h:430–432`):
- `RCU_GP_FLAG_INIT` (`0x1`) — need grace-period initialization
- `RCU_GP_FLAG_FQS`  (`0x2`) — need quiescent-state forcing
- `RCU_GP_FLAG_OVLD` (`0x4`) — callback overload

Key `gp_state` values (`tree.h:435–443`):
- `RCU_GP_IDLE` (`0`) — no GP in progress
- `RCU_GP_WAIT_GPS` (`1`) — waiting for GP start
- `RCU_GP_INIT` (`4`) — GP initialization
- `RCU_GP_WAIT_FQS` (`5`) — waiting for FQS time
- `RCU_GP_CLEANUP` (`7`) — GP cleanup started
- `RCU_GP_CLEANED` (`8`) — GP cleanup complete

### `struct rcu_node` — Key Fields
**File:** `kernel/rcu/tree.h:41–139`

```c
struct rcu_node {
    raw_spinlock_t  lock;           /* Primary lock; root lock also protects
                                       some rcu_state fields */
    unsigned long   gp_seq;         /* Local shadow of rcu_state.gp_seq */
    unsigned long   gp_seq_needed;  /* Furthest future GP requested */
    unsigned long   completedqs;    /* All QSes done for this node */

    /* Quiescent-state tracking bitmasks: */
    unsigned long   qsmask;         /* CPUs/groups still needing QS this GP.
                                       Leaf: one bit per rcu_data.
                                       Internal: one bit per child rcu_node */
    unsigned long   qsmaskinit;     /* Per-GP initial value of qsmask
                                       (copied from qsmaskinitnext at GP start) */
    unsigned long   qsmaskinitnext; /* Online CPUs for next GP */
    unsigned long   rcu_gp_init_mask; /* Mask of offline CPUs at GP init */

    /* Expedited GP bitmasks: */
    unsigned long   expmask;        /* CPUs/groups needing to check in for exp GP */
    unsigned long   expmaskinit;    /* Per-GP initial value of expmask */
    unsigned long   expmaskinitnext;/* Online CPUs for next expedited GP */

    unsigned long   grpmask;        /* Single-bit mask to apply to parent->qsmask */
    int             grplo;          /* Lowest-numbered CPU in this node's range */
    int             grphi;          /* Highest-numbered CPU in this node's range */
    u8              grpnum;         /* Group number for the next level up */
    u8              level;          /* 0 = root, increases toward leaves */
    struct rcu_node *parent;        /* Parent node; NULL at root */

    /* Priority boosting (PREEMPT_RCU): */
    struct list_head blkd_tasks;    /* Tasks blocked in RCU read-side CS */
    struct list_head *gp_tasks;     /* First task blocking current GP */
    struct list_head *exp_tasks;    /* First task blocking current exp GP */

    /* FQS lock (separate cacheline): */
    raw_spinlock_t  fqslock;        /* Used by rcu_force_quiescent_state() funnel */

    /* Expedited GP per-node state: */
    spinlock_t      exp_lock;
    wait_queue_head_t exp_wq[4];    /* Wait queues indexed by gp_seq low bits */
    struct rcu_exp_work rew;        /* Work item for expedited GP */
    raw_spinlock_t  exp_poll_lock;  /* For polled expedited grace periods */
} ____cacheline_internodealigned_in_smp;
```

### `struct rcu_data` — Key Fields
**File:** `kernel/rcu/tree.h:178–285`

```c
struct rcu_data {
    /* 1) Quiescent-state and grace-period handling: */
    unsigned long       gp_seq;          /* Local shadow of rcu_state.gp_seq */
    unsigned long       gp_seq_needed;   /* Furthest future GP requested by this CPU */
    union rcu_noqs      cpu_no_qs;       /* No QSes yet for this CPU (.b.norm/.b.exp) */
    bool                core_needs_qs;   /* Core is waiting for a quiescent state */
    bool                gpwrap;          /* Possible gp_seq wraparound detected */
    struct rcu_node    *mynode;          /* This CPU's leaf rcu_node */
    unsigned long       grpmask;         /* Bitmask for this CPU in mynode->qsmask */

    /* 2) Callback batch handling: */
    struct rcu_segcblist cblist;         /* Segmented callback list */
    long                blimit;          /* Max callbacks per batch invocation */
    unsigned long       n_cbs_invoked;   /* Callbacks invoked since boot */
    long                qlen_last_fqs_check; /* cblist length at last FQS check */

    /* 3) Dynticks / nohz interface: */
    int                 watching_snap;   /* Per-GP snapshot of ct_rcu_watching() */
    bool                rcu_need_heavy_qs; /* GP is old; need heavy quiescent state */
    bool                rcu_urgent_qs;   /* GP is old; need light quiescent state */
    bool                rcu_forced_tick; /* Tick forced to provide QS */
    bool                rcu_forced_tick_exp; /* Tick forced for expedited GP */

    /* 4) rcu_barrier(), OOM: */
    unsigned long       barrier_seq_snap; /* Snapshot of rcu_state.barrier_sequence */
    struct rcu_head     barrier_head;

    /* 5) Callback offloading (CONFIG_RCU_NOCB_CPU): */
    struct task_struct *nocb_gp_kthread; /* GP kthread for this nocb group */
    raw_spinlock_t      nocb_lock;       /* Guards nocb fields */
    raw_spinlock_t      nocb_bypass_lock;
    struct rcu_cblist   nocb_bypass;     /* Lock-contention-bypass CB list */
    struct task_struct *nocb_cb_kthread; /* CB kthread for this CPU */

    /* 7) Diagnostics: */
    unsigned long       last_fqs_resched; /* Time of last rcu_resched() */
    int                 cpu;
};
```

The `union rcu_noqs` (`tree.h:152–158`):
```c
union rcu_noqs {
    struct {
        u8 norm;  /* Normal GP: no QS yet */
        u8 exp;   /* Expedited GP: no QS yet */
    } b;
    u16 s;        /* Aggregate: non-zero means "still needs QS" */
};
```

### Tree Structure: Fan-out and Levels

**File:** `include/linux/rcu_node_tree.h`

```
RCU_FANOUT_LEAF  = 16   (default, CONFIG_RCU_FANOUT_LEAF)
RCU_FANOUT       = 64   (default on 64-bit, CONFIG_RCU_FANOUT)

RCU_FANOUT_1 = RCU_FANOUT_LEAF           (max CPUs: 16  → 1 level)
RCU_FANOUT_2 = RCU_FANOUT_1 * RCU_FANOUT (max CPUs: 1024 → 2 levels)
RCU_FANOUT_3 = RCU_FANOUT_2 * RCU_FANOUT (max CPUs: 65536 → 3 levels)
RCU_FANOUT_4 = RCU_FANOUT_3 * RCU_FANOUT (max CPUs: 4M → 4 levels)
```

The tree is stored as a dense array in `rcu_state.node[]`. `rcu_state.level[i]` points to the first node at level `i`. Level 0 is the root; the deepest level contains leaf nodes, each covering `RCU_FANOUT_LEAF` CPUs. The `qsmask` at each level has one bit per child: leaf nodes have one bit per CPU, internal nodes have one bit per child `rcu_node`.

```
Example: 64 CPUs, RCU_FANOUT_LEAF=16, RCU_FANOUT=64

Level 0 (root):    [rcu_node 0]  qsmask bits 0–3 (4 children)
                   /    |    |    \
Level 1 (leaf):  [1]  [2]  [3]  [4]   each covers 16 CPUs
                 CPUs  CPUs CPUs CPUs
                 0-15  16-31 32-47 48-63
```

---

## Grace Period Machinery

### State Machine

`rcu_state.gp_state` (`kernel/rcu/tree.h:434-443`) cycles through these values:

| Value | Constant | Meaning |
|-------|----------|---------|
| 0 | `RCU_GP_IDLE` | No GP in progress; kthread sleeping on `gp_wq` |
| 1 | `RCU_GP_WAIT_GPS` | Waiting for `RCU_GP_FLAG_INIT` in `gp_flags` |
| 2 | `RCU_GP_DONE_GPS` | Wakeup received; about to call `rcu_gp_init()` |
| 3 | `RCU_GP_ONOFF` | Hotplug processing inside `rcu_gp_init()` |
| 4 | `RCU_GP_INIT` | Tree walk setting `qsmask` from `qsmaskinit` |
| 5 | `RCU_GP_WAIT_FQS` | Sleeping in `swait_event_idle_timeout_exclusive()` |
| 6 | `RCU_GP_DOING_FQS` | Actively forcing quiescent states |
| 7 | `RCU_GP_CLEANUP` | Walking tree to propagate `new_gp_seq` |
| 8 | `RCU_GP_CLEANED` | Cleanup done; about to loop back to IDLE |

### `rcu_gp_kthread()` Main Loop (`kernel/rcu/tree.c:2221-2253`)

```
for (;;) {
    // Phase 1: wait for GP request
    gp_state = RCU_GP_WAIT_GPS
    swait_event on gp_wq until gp_flags & RCU_GP_FLAG_INIT
    gp_state = RCU_GP_DONE_GPS
    loop: rcu_gp_init() until returns true

    // Phase 2: quiescent-state forcing
    rcu_gp_fqs_loop()

    // Phase 3: cleanup
    gp_state = RCU_GP_CLEANUP
    rcu_gp_cleanup()
    gp_state = RCU_GP_CLEANED
    // gp_cleanup() itself writes RCU_GP_IDLE at the end
}
```

The kthread calls `rcu_bind_gp_kthread()` at start to pin itself to a housekeeping CPU.

### `rcu_gp_init()` (`kernel/rcu/tree.c:1796-1948`)

1. Acquires root `rnp->lock`; if `gp_flags == 0`, returns false (spurious wakeup).
2. Calls `rcu_seq_start(&rcu_state.gp_seq)` — odd value = GP in progress.
3. Sets `gp_state = RCU_GP_ONOFF`; iterates leaf nodes with `rcu_for_each_leaf_node()`. For each leaf, holds `ofl_lock` + `rnp->lock` and applies pending hotplug: copies `qsmaskinitnext` into `qsmaskinit`, propagates changes upward via `rcu_init_new_rnp()` / `rcu_cleanup_dead_rnp()`.
4. Sets `gp_state = RCU_GP_INIT`; iterates **all** nodes breadth-first with `rcu_for_each_node_breadth_first()`. For each node: acquires `rnp->lock`, copies `rnp->qsmask = rnp->qsmaskinit`, writes `rnp->gp_seq = rcu_state.gp_seq`. For offline CPUs present in `qsmask & ~qsmaskinitnext`, immediately calls `rcu_report_qs_rnp()` to clear their bits.
5. Returns `true`.

### `rcu_gp_fqs_loop()` (`kernel/rcu/tree.c:2014-2094`)

Timing is driven by two module parameters:

- `jiffies_till_first_fqs` — delay before first FQS scan (default 1-3 jiffies based on HZ)
- `jiffies_till_next_fqs` — delay between subsequent scans

On each iteration:

1. Computes deadline: `rcu_state.jiffies_force_qs = jiffies + j`. Under callback overload (`cbovld`), `j` is compressed to `(j+2)/3`.
2. Sets `gp_state = RCU_GP_WAIT_FQS`; sleeps with timeout on `gp_wq`.
3. Sets `gp_state = RCU_GP_DOING_FQS`; checks root `rnp->qsmask == 0` — if so, breaks (GP done).
4. If `jiffies >= jiffies_force_qs` or `RCU_GP_FLAG_FQS` set: calls `rcu_gp_fqs(first_time)`.
   - First call: `force_qs_rnp(rcu_watching_snap_save)` — takes dynticks snapshots for each CPU still pending.
   - Subsequent calls: `force_qs_rnp(rcu_watching_snap_recheck)` — compares current counter to saved snapshot; if changed, reports QS.
5. Sets `j = jiffies_till_next_fqs` for next sleep.

### `rcu_gp_cleanup()` (`kernel/rcu/tree.c:2100-2219`)

1. Records `gp_end = jiffies`; updates `gp_max` if this was the longest GP.
2. Computes `new_gp_seq` via `rcu_seq_end()` (even value = GP complete).
3. Iterates all nodes breadth-first: acquires lock, writes `rnp->gp_seq = new_gp_seq`, calls `__note_gp_changes()` to advance callbacks.
4. Re-acquires root lock, finalizes `rcu_seq_end(&rcu_state.gp_seq)`, writes `gp_state = RCU_GP_IDLE`.
5. If another GP is needed, sets `gp_flags = RCU_GP_FLAG_INIT`.

---

## Quiescent State Reporting

### `rcu_note_context_switch()` (`kernel/rcu/tree_plugin.h:904-918`)

Called by the scheduler from `__schedule()` on every context switch:

1. `rcu_qs()` — clears `rdp->cpu_no_qs.b.norm` (the normal-QS-needed flag).
2. If `rdp->rcu_urgent_qs` and `rcu_need_heavy_qs` are set, calls `rcu_momentary_eqs()` which atomically increments `ct->state` by `2 * CT_RCU_WATCHING` — passes through an even value (EQS) and back, satisfying any remote watcher.
3. Calls `rcu_tasks_qs()` for RCU-tasks.

### `note_gp_changes()` Fast Path (`kernel/rcu/tree.c:1314-1333`)

Called from `rcu_check_quiescent_state()` (tick path). Uses `raw_spin_trylock_rcu_node()` — if lock unavailable, returns silently (deferred to next tick). If acquired:

1. `__note_gp_changes(rnp, rdp)`:
   - If `rdp->gp_seq` lags `rnp->gp_seq`: GP ended — calls `rcu_advance_cbs()`, clears `core_needs_qs`.
   - If new GP started: sets `rdp->cpu_no_qs.b.norm = true` and `rdp->core_needs_qs = true` if CPU's bit is in `rnp->qsmask`.
   - Updates `rdp->gp_seq = rnp->gp_seq`.
2. If callbacks need a new GP, wakes the GP kthread.

### `rcu_report_qs_rdp()` (`kernel/rcu/tree.c:2393-2438`)

Entry point for a CPU to report its own QS:

1. Acquires leaf `rnp->lock`.
2. Validates: `rdp->cpu_no_qs.b.norm` must be clear and `rdp->gp_seq == rnp->gp_seq`.
3. If CPU's bit is in `rnp->qsmask`, calls `rcu_accelerate_cbs()` then `rcu_report_qs_rnp()`.

### `rcu_report_qs_rnp()` Tree Walk (`kernel/rcu/tree.c:2289-2351`)

Walks from leaf to root, releasing each lock and acquiring the parent's:

```
for (;;):
    if (rnp->qsmask & mask) == 0: return    // bit already cleared
    if (rnp->gp_seq != gps): return          // GP ended
    rnp->qsmask &= ~mask                    // clear this CPU/group bit
    if (rnp->qsmask != 0): return            // other bits remain
    rnp->completedqs = rnp->gp_seq          // node fully done
    mask = rnp->grpmask                      // become this node's bit in parent
    if no parent: break                      // holding root lock
    unlock(rnp); lock(rnp->parent); rnp = rnp->parent
```

When root is reached with `qsmask == 0`, wakes the GP kthread.

---

## Callback Management

### `struct rcu_segcblist` (`include/linux/rcu_segcblist.h:190-201`)

```c
struct rcu_segcblist {
    struct rcu_head  *head;                      /* oldest callback */
    struct rcu_head **tails[RCU_CBLIST_NSEGS];  /* tail pointers per segment */
    unsigned long     gp_seq[RCU_CBLIST_NSEGS]; /* GP number at which segment expires */
    atomic_long_t     len;                       /* total CB count (atomic for NOCB) */
    long              seglen[RCU_CBLIST_NSEGS];  /* per-segment count */
    u8                flags;                     /* SEGCBLIST_ENABLED, SEGCBLIST_OFFLOADED */
};
```

The four segments (`include/linux/rcu_segcblist.h:60-63`):

| Index | Constant | Contents |
|-------|----------|----------|
| 0 | `RCU_DONE_TAIL` | Ready to invoke now |
| 1 | `RCU_WAIT_TAIL` | Waiting for current GP |
| 2 | `RCU_NEXT_READY_TAIL` | Waiting for next GP (accelerated) |
| 3 | `RCU_NEXT_TAIL` | Newly enqueued, GP unknown |

The `tails[]` array holds `**` pointers to the `next` field of the last element in each segment. When a segment is empty, `tails[seg-1] == tails[seg]`.

### Callback Advancement

`rcu_segcblist_advance(rsclp, seq)` (`kernel/rcu/rcu_segcblist.c:469-508`): Scans `WAIT_TAIL` through `NEXT_READY_TAIL`. For each segment whose `gp_seq[i] <= seq`, collapses it into `DONE_TAIL`. Then compacts remaining segments downward.

`rcu_segcblist_accelerate(rsclp, seq)` (`kernel/rcu/rcu_segcblist.c:526-580`): When a GP number becomes available earlier than estimated, merges `NEXT_TAIL` into `NEXT_READY_TAIL` (or `WAIT_TAIL`) and stamps with `seq`.

`rcu_accelerate_cbs()` (`kernel/rcu/tree.c:1135`): Snaps `rcu_state.gp_seq` and calls `rcu_segcblist_accelerate()`. If the accelerated GP is not yet requested, calls `rcu_start_this_gp()`.

`rcu_advance_cbs()` (`kernel/rcu/tree.c:1211`): Calls `rcu_segcblist_advance()` with `rnp->gp_seq` then `rcu_accelerate_cbs()` to label remaining pending callbacks.

### `rcu_do_batch()` (`kernel/rcu/tree.c:2490-2600`)

Invoked from `rcu_core()` (softirq) or the `rcuoc` kthread:

1. Returns if DONE segment is empty.
2. Computes batch limit `bl = max(rdp->blimit, pending >> rcu_divisor)`.
3. Calls `rcu_segcblist_extract_done_cbs()` to move DONE segment into local list.
4. Loops calling each `rhp->func(rhp)`.
5. Breaks early if `count >= bl && (need_resched() || !is_idle_task(current))` or 3ms time limit exceeded.
6. Re-inserts unprocessed callbacks via `rcu_segcblist_insert_done_cbs()`.

### NOCB Architecture (`kernel/rcu/tree_nocb.h`)

Enabled with `rcu_nocbs=<cpulist>` boot parameter. Two kthreads per offloaded CPU:

**`rcu_nocb_gp_kthread`** (`tree_nocb.h:862`) — one per NOCB group. Calls `nocb_gp_wait()`:
- Scans associated CPUs for pending callbacks
- Sleeps on `rdp->nocb_gp_wq` until callbacks arrive or GP completes
- Wakes CB kthreads after GP ends

**`rcu_nocb_cb_kthread`** (`tree_nocb.h:950`) — one per NOCB CPU. Calls `nocb_cb_wait()`:
- Sleeps on `rdp->nocb_cb_wq`
- On wake: `rcu_momentary_eqs()` for a QS, then `rcu_do_batch(rdp)`

**Bypass list** (`rdp->nocb_bypass`, type `struct rcu_cblist`): Lock-contention bypass for high `call_rcu()` rates exceeding `nocb_nobypass_lim_per_jiffy` (default `16000/HZ`). Callbacks enqueued under `nocb_bypass_lock` (lighter than `nocb_lock`). Flushed by GP kthread via `rcu_nocb_try_flush_bypass()`.

---

## Dynticks Integration

### `context_tracking.state` Encoding (`include/linux/context_tracking_state.h`)

```c
struct context_tracking {
    atomic_t state;   /* upper bits: RCU watching counter; lower 2 bits: context state */
    long     nesting; /* process (non-IRQ) nesting depth */
    long     nmi_nesting;
};
```

`CT_RCU_WATCHING = CT_STATE_MAX = 4`. The upper bits form an integer counter:
- **Odd** value: CPU is being watched by RCU (not idle)
- **Even** value: CPU is in Extended Quiescent State (EQS) — idle or nohz_full userspace

`ct_rcu_watching()` returns the upper bits. `rcu_watching_snap_in_eqs(snap)` returns true when the watching counter is even.

### Entering/Exiting EQS

Transitions happen via `ct_state_inc(CT_RCU_WATCHING)` which atomically adds 4 to `state`. Each enter/exit increments by one unit (4), so two increments advance by 8, making a full EQS cycle detectable by comparing snapshots.

`rcu_momentary_eqs()` (`kernel/rcu/tree.c:370`): atomically adds `2 * CT_RCU_WATCHING` — passes through one even value and back to odd, synthesizing a zero-duration EQS visible to remote watchers.

### How the GP Kthread Detects Idle CPUs

During `rcu_gp_fqs()`, `force_qs_rnp()` calls `rcu_watching_snap_save(rdp)` (`kernel/rcu/tree.c:783-802`):

```c
rdp->watching_snap = ct_rcu_watching_cpu_acquire(rdp->cpu);
if (rcu_watching_snap_in_eqs(rdp->watching_snap)):
    // CPU is currently idle — immediate QS, report it
```

On subsequent FQS, `rcu_watching_snap_recheck(rdp)` checks if `ct_rcu_watching_cpu_acquire(cpu) != snap` — counter changed means CPU passed through an EQS.

`rcu_is_cpu_rrupt_from_idle()` (`kernel/rcu/tree.c:390`): checks `ct_nesting() <= 0` for the scheduler tick to classify a tick from idle as a QS.

---

## Locking Summary

| Lock | Type | Location | Protects |
|------|------|----------|----------|
| `rcu_node.lock` | `raw_spinlock_t` | `tree.h:42` | `gp_seq`, `qsmask`, `qsmaskinit`, `qsmaskinitnext`, `completedqs`, `blkd_tasks`, `gp_tasks` |
| `rcu_node.fqslock` | `raw_spinlock_t` | `tree.h:128` | Serializes `rcu_force_quiescent_state()` callers climbing the tree; trylock-based |
| `rcu_node.exp_lock` | `spinlock_t` | `tree.h:130` | Per-node expedited GP state: `exp_wq[]` |
| `rcu_node.exp_poll_lock` | `raw_spinlock_t` | `tree.h:135` | Polled expedited GP state |
| `rcu_state.ofl_lock` | `arch_spinlock_t` | `tree.h:411` | CPU hotplug vs GP pre-initialization |
| `rcu_state.barrier_mutex` | `struct mutex` | `tree.h:366` | Serializes concurrent `rcu_barrier()` calls |
| `rcu_state.barrier_lock` | `raw_spinlock_t` | `tree.h:373` | `barrier_sequence`, per-CPU `barrier_seq_snap`, CB-list checks during barrier |
| `rcu_state.exp_mutex` | `struct mutex` | `tree.h:375` | Serializes expedited GP initiation |
| `rcu_state.exp_wake_mutex` | `struct mutex` | `tree.h:376` | Serializes expedited GP wakeup |
| `rcu_state.nocb_mutex` | `struct mutex` | `tree.h:424` | NOCB offloading/de-offloading transitions |
| `rcu_data.nocb_lock` | `raw_spinlock_t` | `tree.h:225` | `rdp->cblist` for NOCB CPUs |
| `rcu_data.nocb_bypass_lock` | `raw_spinlock_t` | `tree.h:233` | `rdp->nocb_bypass` list |
| `rcu_data.nocb_gp_lock` | `raw_spinlock_t` | `tree.h:240` | `nocb_gp_sleep`, `nocb_gp_bypass`, `nocb_gp_gp`, `nocb_gp_seq` |

---

## Source File Reference

| File | Purpose | Key Functions |
|------|---------|---------------|
| `kernel/rcu/tree.c` | Core tree-RCU implementation | `rcu_gp_kthread()`, `rcu_gp_init()`, `rcu_gp_fqs_loop()`, `rcu_gp_cleanup()`, `rcu_report_qs_rdp()`, `rcu_report_qs_rnp()`, `rcu_do_batch()`, `note_gp_changes()`, `call_rcu()`, `synchronize_rcu()`, `rcu_barrier()` |
| `kernel/rcu/tree.h` | Internal struct definitions | `struct rcu_node`, `struct rcu_data`, `struct rcu_state`, GP state constants |
| `kernel/rcu/tree_plugin.h` | Preemptible vs non-preemptible variants | `rcu_note_context_switch()`, `rcu_read_unlock_special()`, `rcu_preempt_deferred_qs()` |
| `kernel/rcu/tree_nocb.h` | No-callbacks CPU offloading | `rcu_nocb_gp_kthread()`, `rcu_nocb_cb_kthread()`, `nocb_gp_wait()`, `rcu_nocb_try_bypass()` |
| `kernel/rcu/tree_exp.h` | Expedited grace periods | `synchronize_rcu_expedited()`, `rcu_exp_gp_seq_snap()` |
| `kernel/rcu/tree_stall.h` | CPU stall detection and reporting | `print_cpu_stall()`, `check_cpu_stall()` |
| `kernel/rcu/rcu_segcblist.c` | Segmented callback list operations | `rcu_segcblist_advance()`, `rcu_segcblist_accelerate()`, `rcu_segcblist_extract_done_cbs()` |
| `kernel/rcu/rcu_segcblist.h` | Segmented CB list inline helpers | `rcu_segcblist_ready_cbs()`, `rcu_segcblist_empty()`, `rcu_segcblist_n_cbs()` |
| `kernel/rcu/rcu.h` | Shared RCU internal declarations | `rcu_seq_start/end/snap/current/state()`, GP sequence encoding macros |
| `kernel/rcu/update.c` | Core RCU API and init | `rcu_init()`, exported `synchronize_rcu()` wrapper |
| `kernel/rcu/srcutree.c` | Sleepable RCU (SRCU) | `srcu_read_lock/unlock()`, `synchronize_srcu()` |
| `kernel/rcu/tiny.c` | Uniprocessor (tiny) RCU | Simplified `call_rcu()` and `synchronize_rcu()` for `!SMP` |
| `kernel/rcu/tasks.h` | RCU-tasks variants | `synchronize_rcu_tasks()`, `synchronize_rcu_tasks_rude()`, `synchronize_rcu_tasks_trace()` |
| `kernel/rcu/sync.c` | `rcu_sync` utility | `rcu_sync_enter()`, `rcu_sync_exit()` — used by percpu-rwsem fastpaths |
