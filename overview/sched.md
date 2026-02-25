# Linux Kernel Scheduler Overview

This document provides an in-depth overview of the Linux kernel scheduler
internals, covering task wakeup mechanisms, inter-processor interrupts, IRQ
handling, workqueues, load balancing, and scheduler topology.

---

## Table of Contents

1. [Core Data Structures](#1-core-data-structures)
2. [Try To Wake Up (TTWU)](#2-try-to-wake-up-ttwu)
3. [Inter-Processor Interrupts (IPI)](#3-inter-processor-interrupts-ipi)
4. [IRQ Handling and Scheduler Integration](#4-irq-handling-and-scheduler-integration)
5. [Workqueues and Workers](#5-workqueues-and-workers)
6. [Load Balancing](#6-load-balancing)
7. [Scheduler Topology](#7-scheduler-topology)
8. [PELT (Per-Entity Load Tracking)](#8-pelt-per-entity-load-tracking)

---

## 1. Core Data Structures

### 1.1 struct task_struct (Scheduler-Related Fields)

**Location:** `include/linux/sched.h`

The task_struct is the fundamental data structure representing every process
and thread in the Linux kernel. It serves as the process control block (PCB)
and contains all information the kernel needs to manage and schedule a task.
From a scheduling perspective, the key fields include the task's current state
(running, sleeping, etc.), priority values for different scheduling policies,
CPU affinity masks, and embedded scheduling entities for CFS, real-time, and
deadline schedulers. The sched_class pointer determines which scheduler
implementation handles the task, while PELT-related fields (sched_avg) track
the task's historical CPU usage for load balancing decisions. The task_struct
is allocated via the slab allocator when a new process is created and persists
until the process terminates.

```c
struct task_struct {
    /* Process state */
    volatile long __state;              /* Task state: TASK_RUNNING, etc. */
    void *stack;                        /* Kernel stack pointer */
    unsigned int flags;                 /* PF_* flags */

    /* Scheduling */
    int on_rq;                          /* On runqueue status */
    int prio;                           /* Dynamic priority */
    int static_prio;                    /* Static priority (nice value) */
    int normal_prio;                    /* Normal priority */
    unsigned int rt_priority;           /* Real-time priority */
    unsigned int policy;                /* Scheduling policy */

    const struct sched_class *sched_class;  /* Scheduler class */
    struct sched_entity se;             /* CFS scheduling entity */
    struct sched_rt_entity rt;          /* RT scheduling entity */
    struct sched_dl_entity dl;          /* Deadline scheduling entity */

    /* CPU affinity */
    int nr_cpus_allowed;
    const cpumask_t *cpus_ptr;
    cpumask_t *user_cpus_ptr;
    cpumask_t cpus_mask;

    /* Context switch stats */
    unsigned long nvcsw;                /* Voluntary context switches */
    unsigned long nivcsw;               /* Involuntary context switches */

    /* Load tracking */
    struct sched_avg avg;               /* PELT averages */
    unsigned long wakee_flips;          /* Wakeup pattern tracking */
    unsigned long wakee_flip_decay_ts;
    struct task_struct *last_wakee;

    /* Thread info (includes preempt_count) */
    struct thread_info thread_info;
};
```

**Task States:**
```c
#define TASK_RUNNING            0x00000000
#define TASK_INTERRUPTIBLE      0x00000001
#define TASK_UNINTERRUPTIBLE    0x00000002
#define __TASK_STOPPED          0x00000004
#define __TASK_TRACED           0x00000008
#define EXIT_DEAD               0x00000010
#define EXIT_ZOMBIE             0x00000020
#define TASK_PARKED             0x00000040
#define TASK_DEAD               0x00000080
#define TASK_WAKEKILL           0x00000100
#define TASK_WAKING             0x00000200
#define TASK_NOLOAD             0x00000400
#define TASK_NEW                0x00000800
#define TASK_RTLOCK_WAIT        0x00001000
```

### 1.2 struct rq (Run Queue)

**Location:** `kernel/sched/sched.h`

The rq (run queue) structure is the per-CPU data structure that manages all
schedulable tasks on a single CPU. Every CPU in the system has exactly one run
queue, accessed via the cpu_rq(cpu) macro. It contains embedded sub-runqueues
for each scheduling class (cfs_rq for CFS tasks, rt_rq for real-time tasks, and
    dl_rq for deadline tasks), along with pointers to the currently running
    task, the idle task, and the special stop task used for migration. The run
    queue maintains timing information including various clocks (clock,
    clock_task, clock_pelt) used for scheduling decisions and load tracking.
    The lock field is a critical spinlock that must be held when modifying the
    run queue state. Load balancing fields track when the next balance should
    occur and whether an active migration is in progress. The wake_list enables
    batching of remote wakeups to reduce cross-CPU lock contention.

```c
struct rq {
    raw_spinlock_t      lock;           /* Runqueue lock */

    /* Task counts */
    unsigned int        nr_running;      /* Total runnable tasks */
    unsigned int        nr_numa_running; /* NUMA migrating tasks */
    unsigned int        nr_preferred_running;

    /* Time tracking */
    u64                 clock;           /* Runqueue clock */
    u64                 clock_task;      /* Task clock (excludes IRQ time) */
    u64                 clock_pelt;      /* PELT clock */
    u64                 lost_idle_time;  /* Idle time not accounted */

    /* Current task */
    struct task_struct  *curr;           /* Currently running task */
    struct task_struct  *idle;           /* Idle task for this CPU */
    struct task_struct  *stop;           /* Stop task for migration */

    /* Scheduler classes */
    struct cfs_rq       cfs;             /* CFS runqueue */
    struct rt_rq        rt;              /* Real-time runqueue */
    struct dl_rq        dl;              /* Deadline runqueue */

    /* Load balancing */
    unsigned long       next_balance;    /* Next balance time */
    int                 active_balance;  /* Active balance in progress */
    int                 push_cpu;        /* CPU to push tasks to */
    struct cpu_stop_work active_balance_work;

    /* Scheduling domains */
    struct root_domain  *rd;             /* Root domain */
    struct sched_domain __rcu *sd;       /* Sched domain hierarchy */

    /* CPU capacity */
    unsigned long       cpu_capacity;    /* Capacity of this CPU */
    unsigned long       cpu_capacity_orig;

    /* Wake list for batching */
    struct llist_head   wake_list;
    unsigned long       ttwu_pending;

    /* NOHZ idle tracking */
    atomic_t            nr_iowait;
    int                 nohz_tick_stopped;

    /* Load tracking */
    unsigned long       calc_load_update;
    long                calc_load_active;

    /* Statistics */
    unsigned int        lb_count[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_failed[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_balanced[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_imbalanced[CPU_MAX_IDLE_TYPES];
};
```

### 1.3 struct sched_class

**Location:** `kernel/sched/sched.h`

The sched_class structure defines the interface that each scheduling policy
must implement, following an object-oriented design pattern within the kernel.
Each scheduler class (stop, deadline, real-time, fair/CFS, and idle) provides
its own implementation of these function pointers. The structure defines
operations for enqueueing and dequeueing tasks on run queues, selecting the
next task to run (pick_next_task), handling preemption checks, and managing
load balancing. The enqueue_task and dequeue_task functions add or remove tasks
from the class-specific run queue, while check_preempt_curr determines if a
newly woken task should preempt the current one. The select_task_rq function is
crucial for task placement decisions during wakeup and fork. Scheduler classes
are organized in a strict priority hierarchy, where higher-priority classes
always run before lower-priority ones.

```c
struct sched_class {
    /* Enqueue/dequeue operations */
    void (*enqueue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*dequeue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*yield_task)(struct rq *rq);
    bool (*yield_to_task)(struct rq *rq, struct task_struct *p);

    /* Preemption check */
    void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);

    /* Task selection */
    struct task_struct *(*pick_next_task)(struct rq *rq);
    void (*put_prev_task)(struct rq *rq, struct task_struct *p);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

    /* Load balancing */
    int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
    int (*select_task_rq)(struct task_struct *p, int cpu, int wake_flags);
    void (*migrate_task_rq)(struct task_struct *p, int new_cpu);
    void (*task_woken)(struct rq *this_rq, struct task_struct *task);

    /* Runtime management */
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
    void (*task_fork)(struct task_struct *p);
    void (*task_dead)(struct task_struct *p);

    /* Priority changes */
    void (*prio_changed)(struct rq *this_rq, struct task_struct *task, int oldprio);
    unsigned int (*get_rr_interval)(struct rq *rq, struct task_struct *task);

    /* Accounting */
    void (*update_curr)(struct rq *rq);
};
```

**Scheduler Class Hierarchy (priority order):**
```
stop_sched_class    → Highest (stop machine tasks)
dl_sched_class      → Deadline scheduling
rt_sched_class      → Real-time (SCHED_FIFO, SCHED_RR)
fair_sched_class    → CFS (SCHED_NORMAL, SCHED_BATCH)
idle_sched_class    → Lowest (idle task)
```

### 1.4 struct sched_entity

**Location:** `include/linux/sched.h`

The sched_entity structure represents a schedulable unit within the CFS
(Completely Fair Scheduler). It is embedded within task_struct for regular
tasks, but can also represent a task group in hierarchical scheduling
configurations (cgroups). The central concept is virtual runtime (vruntime),
which tracks how much CPU time the entity has consumed, weighted by its
priority. Entities with lower vruntime are selected to run first, ensuring
fairness. The structure contains an rb_node for placement in the red-black tree
that CFS uses to efficiently find the task with the smallest vruntime. Load
tracking fields (sched_avg) maintain PELT metrics for the entity. For task
groups, the my_q pointer references the owned cfs_rq containing child entities,
while parent and cfs_rq establish the hierarchy. The run_node and deadline
    fields support the EEVDF (Earliest Eligible Virtual Deadline First)
    scheduling algorithm used in modern CFS implementations.

```c
struct sched_entity {
    struct load_weight      load;        /* Task weight for CFS */
    struct rb_node          run_node;    /* Red-black tree node */
    u64                     deadline;    /* Virtual deadline */
    u64                     min_vruntime; /* Minimum vruntime seen */

    struct list_head        group_node;  /* Task group linkage */
    unsigned int            on_rq;       /* On runqueue flag */

    /* Virtual runtime tracking */
    u64                     exec_start;  /* Start of current slice */
    u64                     sum_exec_runtime; /* Total CPU time */
    u64                     prev_sum_exec_runtime;
    u64                     vruntime;    /* Virtual runtime */

    u64                     nr_migrations; /* Migration count */

    /* PELT load tracking */
    struct sched_avg        avg;

    /* Statistics */
    struct sched_statistics statistics;

    /* Hierarchy */
    int                     depth;       /* Depth in task group tree */
    struct sched_entity     *parent;     /* Parent entity */
    struct cfs_rq           *cfs_rq;     /* CFS runqueue we're on */
    struct cfs_rq           *my_q;       /* Owned runqueue (for groups) */
};
```

### 1.5 struct cfs_rq

**Location:** `kernel/sched/sched.h`

The cfs_rq structure is the CFS-specific run queue that manages all
sched_entities at a particular level in the scheduling hierarchy. Each per-CPU
rq contains an embedded cfs_rq for top-level CFS tasks, and each task group
also has its own cfs_rq per CPU. The min_vruntime field is critical as it
tracks the smallest virtual runtime among all runnable entities, ensuring that
newly woken or migrated tasks are placed fairly in the timeline. The
tasks_timeline is a red-black tree (rb_root_cached) containing all runnable
entities, ordered by their virtual deadline. The curr, next, last, and skip
pointers optimize task selection by tracking recently run entities and
scheduling hints. PELT fields (sched_avg, removed) aggregate load metrics for
all entities on this run queue. For cgroup bandwidth control, throttling fields
track whether the group has exceeded its CPU quota. The rq pointer links back
to the parent per-CPU run queue, while tg points to the owning task group.

```c
struct cfs_rq {
    struct load_weight      load;        /* Total load of all entities */
    unsigned int            nr_running;  /* Number of runnable entities */
    unsigned int            h_nr_running; /* Hierarchical count */

    u64                     exec_clock;  /* Execution clock */
    u64                     min_vruntime; /* Minimum vruntime */

    /* Red-black tree of runnable entities */
    struct rb_root_cached   tasks_timeline;

    /* Current/next/last entities */
    struct sched_entity     *curr;
    struct sched_entity     *next;
    struct sched_entity     *last;
    struct sched_entity     *skip;

    /* PELT load tracking */
    struct sched_avg        avg;
    struct {
        raw_spinlock_t  lock;
        int             nr;
        unsigned long   load_avg;
        unsigned long   util_avg;
        unsigned long   runnable_avg;
    } removed;

    /* Throttling (for cgroups) */
    int                     throttled;
    u64                     throttled_clock;
    u64                     throttled_clock_task;
    struct list_head        throttled_list;

    /* Task group */
    struct rq               *rq;         /* Parent runqueue */
    struct task_group       *tg;         /* Task group */
};
```

### 1.6 struct rt_rq

**Location:** `kernel/sched/sched.h`

The rt_rq structure manages real-time tasks using SCHED_FIFO and SCHED_RR
policies. Unlike CFS which uses a single red-black tree, the real-time run
queue uses a priority array (rt_prio_array) with a bitmap for O(1) lookup of
the highest priority runnable task. Real-time priorities range from 0-99, with
higher values indicating higher priority. The highest_prio field tracks the
current and next highest priorities for quick access. The pushable_tasks list
contains tasks that can be migrated to other CPUs when this CPU becomes
overloaded (more RT tasks than can run). For RT bandwidth throttling, rt_time
tracks accumulated runtime against rt_runtime quota to prevent real-time tasks
from starving the system. The overloaded flag indicates when push/pull
balancing should be considered. This structure is embedded in the per-CPU rq
and accessed when scheduling real-time tasks.

```c
struct rt_rq {
    struct rt_prio_array    active;      /* Priority array */
    unsigned int            rt_nr_running; /* RT tasks running */
    unsigned int            rr_nr_running; /* RR tasks running */

    struct {
        int curr;
        int next;
    } highest_prio;                      /* Highest priority tracking */

    unsigned long           rt_nr_migratory; /* Migratable RT tasks */
    unsigned long           rt_nr_total;     /* Total RT tasks */
    int                     overloaded;      /* More tasks than CPUs */

    struct plist_head       pushable_tasks;  /* Tasks that can migrate */

    /* RT bandwidth control */
    int                     rt_throttled;
    u64                     rt_time;
    u64                     rt_runtime;
    raw_spinlock_t          rt_runtime_lock;
};
```

---

## 2. Try To Wake Up (TTWU)

### 2.1 Overview

The `try_to_wake_up()` function is the core mechanism for transitioning a task
from a sleeping state to a runnable state. It's called whenever a task needs to
be woken up, whether from a blocking I/O operation, a wait queue, or a signal.

**Location:** `kernel/sched/core.c`

### 2.2 Main Function

```c
int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    unsigned long flags;
    int cpu, success = 0;

    /* Fast path: check if already running */
    if (READ_ONCE(p->__state) & TASK_RUNNING)
        return 0;

    raw_spin_lock_irqsave(&p->pi_lock, flags);

    /* State check with lock held */
    if (!(READ_ONCE(p->__state) & state))
        goto unlock;

    /* Record wakeup statistics */
    trace_sched_waking(p);

    /* Update state to TASK_WAKING (transitional) */
    WRITE_ONCE(p->__state, TASK_WAKING);

    /* Select CPU for task placement */
    cpu = select_task_rq(p, p->wake_cpu, wake_flags);

    /* Queue the task on the target CPU */
    if (task_cpu(p) != cpu)
        set_task_cpu(p, cpu);

    ttwu_queue(p, cpu, wake_flags);
    success = 1;

unlock:
    raw_spin_unlock_irqrestore(&p->pi_lock, flags);

    if (success)
        ttwu_stat(p, task_cpu(p), wake_flags);

    return success;
}
```

### 2.3 TTWU Queue Function

```c
static void ttwu_queue(struct task_struct *p, int cpu, int wake_flags)
{
    struct rq *rq = cpu_rq(cpu);
    struct rq_flags rf;

    /* Check if we can batch the wakeup via wake list */
    if (ttwu_queue_wakelist(p, cpu, wake_flags))
        return;

    /* Direct wakeup path */
    rq_lock(rq, &rf);
    update_rq_clock(rq);
    ttwu_do_activate(rq, p, wake_flags, &rf);
    rq_unlock(rq, &rf);
}
```

### 2.4 TTWU Activation

```c
static void ttwu_do_activate(struct rq *rq, struct task_struct *p,
                             int wake_flags, struct rq_flags *rf)
{
    int en_flags = ENQUEUE_WAKEUP | ENQUEUE_NOCLOCK;

    if (p->sched_contributes_to_load)
        rq->nr_uninterruptible--;

    if (wake_flags & WF_MIGRATED)
        en_flags |= ENQUEUE_MIGRATED;

    /* Add to runqueue */
    activate_task(rq, p, en_flags);

    /* Set task to TASK_RUNNING */
    ttwu_do_wakeup(p);

    /* Check if we should preempt current */
    check_preempt_curr(rq, p, wake_flags);
}
```

### 2.5 Wake Flags

```c
/* Wake flags for try_to_wake_up */
#define WF_SYNC         0x01    /* Waker going to sleep after wakeup */
#define WF_FORK         0x02    /* Child wakeup after fork */
#define WF_MIGRATED     0x04    /* Task was migrated */
#define WF_ON_RQ        0x08    /* Task already on runqueue */
#define WF_EXEC         0x10    /* Wakeup from exec */
#define WF_TTWU_QUEUE   0x20    /* Use wake list optimization */
```

### 2.6 TTWU Flow Diagram

```
try_to_wake_up(task, state, wake_flags)
    │
    ├─► Check: task already TASK_RUNNING?
    │   └─► Yes: return 0 (already awake)
    │
    ├─► Acquire pi_lock (spinlock with IRQ save)
    │
    ├─► Check: task state matches wake condition?
    │   └─► No: unlock and return 0
    │
    ├─► Set task state to TASK_WAKING
    │
    ├─► select_task_rq()
    │   ├─► Consider wake affinity (WF_SYNC)
    │   ├─► Check CPU affinity mask
    │   ├─► Use scheduler class selection
    │   └─► Return optimal CPU
    │
    ├─► If CPU changed: set_task_cpu()
    │
    ├─► ttwu_queue(task, cpu, wake_flags)
    │   │
    │   ├─► Try wake list optimization
    │   │   └─► Batch wakeups to reduce lock contention
    │   │
    │   └─► Or direct path:
    │       ├─► rq_lock(rq)
    │       ├─► update_rq_clock()
    │       ├─► ttwu_do_activate()
    │       │   ├─► activate_task() - enqueue
    │       │   ├─► ttwu_do_wakeup() - set TASK_RUNNING
    │       │   └─► check_preempt_curr()
    │       └─► rq_unlock(rq)
    │
    ├─► Release pi_lock
    │
    └─► ttwu_stat() - record statistics
```

---

## 3. Inter-Processor Interrupts (IPI)

### 3.1 TIF_NEED_RESCHED Flag

**Location:** `include/linux/sched.h`, arch-specific headers

The TIF_NEED_RESCHED flag signals that a task should be rescheduled:

```c
/* Thread info flags */
#define TIF_NEED_RESCHED        3       /* Reschedule required */
#define TIF_NEED_RESCHED_LAZY   4       /* Lazy reschedule (PREEMPT_RT) */

/* Macros for flag manipulation */
static inline void set_tsk_need_resched(struct task_struct *tsk)
{
    set_tsk_thread_flag(tsk, TIF_NEED_RESCHED);
}

static inline void clear_tsk_need_resched(struct task_struct *tsk)
{
    clear_tsk_thread_flag(tsk, TIF_NEED_RESCHED);
}

static inline int test_tsk_need_resched(struct task_struct *tsk)
{
    return test_tsk_thread_flag(tsk, TIF_NEED_RESCHED);
}
```

### 3.2 resched_curr()

**Location:** `kernel/sched/core.c`

The core function that triggers rescheduling:

```c
void resched_curr(struct rq *rq)
{
    struct task_struct *curr = rq->curr;
    int cpu = cpu_of(rq);

    lockdep_assert_rq_held(rq);

    /* Already flagged for reschedule */
    if (test_tsk_need_resched(curr))
        return;

    /* Set the reschedule flag */
    set_tsk_need_resched(curr);

    /* If this is the current CPU, done */
    if (cpu == smp_processor_id())
        return;

    /* Send IPI to remote CPU */
    smp_send_reschedule(cpu);
}
```

### 3.3 smp_send_reschedule()

**Location:** `arch/x86/kernel/smp.c` (x86 specific)

Sends a reschedule IPI to a remote CPU:

```c
void smp_send_reschedule(int cpu)
{
    if (unlikely(cpu_is_offline(cpu))) {
        WARN_ON(1);
        return;
    }
    apic->send_IPI(cpu, RESCHEDULE_VECTOR);
}

/* APIC-level IPI send */
static inline void __apic_send_IPI_dest(unsigned int dest, int vector)
{
    unsigned long flags;

    local_irq_save(flags);
    __x2apic_send_IPI_dest(dest, vector, APIC_DEST_PHYSICAL);
    local_irq_restore(flags);
}
```

### 3.4 scheduler_ipi()

**Location:** `kernel/sched/core.c`

Handler for the reschedule IPI on the target CPU:

```c
void scheduler_ipi(void)
{
    struct rq *rq = this_rq();

    /*
     * The sender has already set TIF_NEED_RESCHED.
     * This IPI just ensures we process it promptly.
     *
     * The actual reschedule happens at:
     * - Return to userspace
     * - Return from interrupt
     * - Explicit preemption point
     */

    /* Handle wake list if pending */
    if (llist_empty(&rq->wake_list) && !got_nohz_idle_kick())
        return;

    irq_enter();
    sched_ttwu_pending();
    irq_exit();
}
```

### 3.5 x86 Reschedule Interrupt Handler

**Location:** `arch/x86/kernel/smp.c`

```c
/* Vector for reschedule IPI */
#define RESCHEDULE_VECTOR       0xfd

/* Interrupt handler */
DEFINE_IDTENTRY_SYSVEC_SIMPLE(sysvec_reschedule_ipi)
{
    ack_APIC_irq();
    trace_reschedule_entry(RESCHEDULE_VECTOR);
    inc_irq_stat(irq_resched_count);
    scheduler_ipi();
    trace_reschedule_exit(RESCHEDULE_VECTOR);
}
```

### 3.6 IPI Flow Diagram

```
CPU 0: resched_curr(rq1)                    CPU 1: (target)
       │
       ├─► set_tsk_need_resched(curr1)
       │   └─► Sets TIF_NEED_RESCHED
       │
       ├─► cpu_of(rq1) != smp_processor_id()
       │
       └─► smp_send_reschedule(1)
           │
           └─► apic->send_IPI(1, RESCHEDULE_VECTOR)
               │
               └─────────────────────────────────────► IPI arrives
                                                       │
                                                       ├─► sysvec_reschedule_ipi()
                                                       │   ├─► ack_APIC_irq()
                                                       │   └─► scheduler_ipi()
                                                       │       └─► Process wake list
                                                       │
                                                       └─► At preemption point:
                                                           ├─► test_tsk_need_resched()
                                                           └─► schedule()
                                                               └─► Task switch
```

---

## 4. IRQ Handling and Scheduler Integration

### 4.1 Preempt Count Architecture

**Location:** `include/linux/preempt.h`

The preempt_count tracks interrupt and preemption nesting:

```c
/*
 * Preempt count bit layout:
 *
 * Bits 0-7:   Preemption disable count (256 levels)
 * Bits 8-15:  Softirq disable count
 * Bits 16-19: Hardirq count
 * Bits 20-23: NMI count
 * Bit 24:     PREEMPT_NEED_RESCHED
 */

#define PREEMPT_BITS        8
#define SOFTIRQ_BITS        8
#define HARDIRQ_BITS        4
#define NMI_BITS            4

#define PREEMPT_SHIFT       0
#define SOFTIRQ_SHIFT       (PREEMPT_SHIFT + PREEMPT_BITS)
#define HARDIRQ_SHIFT       (SOFTIRQ_SHIFT + SOFTIRQ_BITS)
#define NMI_SHIFT           (HARDIRQ_SHIFT + HARDIRQ_BITS)

#define PREEMPT_OFFSET      (1UL << PREEMPT_SHIFT)
#define SOFTIRQ_OFFSET      (1UL << SOFTIRQ_SHIFT)
#define HARDIRQ_OFFSET      (1UL << HARDIRQ_SHIFT)
#define NMI_OFFSET          (1UL << NMI_SHIFT)
```

### 4.2 Context Check Macros

```c
/* Check current context */
#define in_irq()            (hardirq_count())
#define in_softirq()        (softirq_count())
#define in_interrupt()      (irq_count())       /* hardirq OR softirq */
#define in_task()           (!(in_irq() | in_softirq()))

/* Count accessors */
#define hardirq_count()     (preempt_count() & HARDIRQ_MASK)
#define softirq_count()     (preempt_count() & SOFTIRQ_MASK)
#define irq_count()         (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))
#define preempt_count()     (current_thread_info()->preempt.count)

/* Preemptible check */
#define preemptible()       (preempt_count() == 0 && !irqs_disabled())
```

### 4.3 IRQ Enter/Exit

**Location:** `kernel/softirq.c`

```c
void irq_enter(void)
{
    rcu_irq_enter();
    irqtime_start();
    preempt_count_add(HARDIRQ_OFFSET);
    lockdep_hardirq_enter();
}

void irq_exit(void)
{
    lockdep_hardirq_exit();
    preempt_count_sub(HARDIRQ_OFFSET);

    /* Process softirqs if pending and not in interrupt */
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();

    irqtime_end();
    rcu_irq_exit();
}
```

### 4.4 Softirq Handling

**Location:** `kernel/softirq.c`

```c
/* Softirq types */
enum {
    HI_SOFTIRQ = 0,         /* High priority tasklets */
    TIMER_SOFTIRQ,          /* Timer callbacks */
    NET_TX_SOFTIRQ,         /* Network transmit */
    NET_RX_SOFTIRQ,         /* Network receive */
    BLOCK_SOFTIRQ,          /* Block I/O completion */
    IRQ_POLL_SOFTIRQ,       /* IRQ polling */
    TASKLET_SOFTIRQ,        /* Normal tasklets */
    SCHED_SOFTIRQ,          /* Scheduler load balancing */
    HRTIMER_SOFTIRQ,        /* High-res timer */
    RCU_SOFTIRQ,            /* RCU callbacks */
    NR_SOFTIRQS
};

/* Main softirq execution */
asmlinkage __visible void __do_softirq(void)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
    int max_restart = MAX_SOFTIRQ_RESTART;
    struct softirq_action *h;
    __u32 pending;
    int softirq_bit;

    pending = local_softirq_pending();

    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);

restart:
    set_softirq_pending(0);
    local_irq_enable();

    h = softirq_vec;

    while ((softirq_bit = ffs(pending))) {
        h += softirq_bit - 1;
        h->action(h);
        h++;
        pending >>= softirq_bit;
    }

    local_irq_disable();
    pending = local_softirq_pending();

    if (pending) {
        if (time_before(jiffies, end) && --max_restart)
            goto restart;
        wakeup_softirqd();
    }

    __local_bh_enable(SOFTIRQ_OFFSET);
}
```

### 4.5 raise_softirq()

```c
void raise_softirq(unsigned int nr)
{
    unsigned long flags;

    local_irq_save(flags);
    raise_softirq_irqoff(nr);
    local_irq_restore(flags);
}

inline void raise_softirq_irqoff(unsigned int nr)
{
    __raise_softirq_irqoff(nr);

    if (!in_interrupt())
        wakeup_softirqd();
}

void __raise_softirq_irqoff(unsigned int nr)
{
    or_softirq_pending(1UL << nr);
}
```

### 4.6 Complete IRQ to Reschedule Flow

```
HARDWARE INTERRUPT
    │
    ├─► CPU receives interrupt vector
    │
    ├─► Interrupt handler entry (asm_common_interrupt)
    │
    ├─► irq_enter()
    │   ├─► rcu_irq_enter()
    │   ├─► preempt_count_add(HARDIRQ_OFFSET)
    │   └─► [Now in hardirq context]
    │
    ├─► handle_irq_event()
    │   └─► Call registered ISRs
    │       └─► May: raise_softirq(SCHED_SOFTIRQ)
    │
    ├─► irq_exit()
    │   ├─► preempt_count_sub(HARDIRQ_OFFSET)
    │   ├─► [Exit hardirq context]
    │   │
    │   └─► if (!in_interrupt() && local_softirq_pending())
    │       └─► invoke_softirq()
    │           └─► __do_softirq()
    │               ├─► SCHED_SOFTIRQ handler
    │               │   └─► run_rebalance_domains()
    │               │       └─► rebalance_domains()
    │               │           └─► May: resched_curr()
    │               └─► Other softirq handlers
    │
    └─► Return path (prepare_exit_to_usermode)
        ├─► Check TIF_NEED_RESCHED
        └─► If set: schedule()
            └─► Context switch
```

---

## 5. Workqueues and Workers

### 5.1 Overview

Workqueues provide a deferred execution mechanism in process context, allowing
work to sleep and use the full kernel API.

**Location:** `kernel/workqueue.c`, `include/linux/workqueue.h`

### 5.2 struct worker

The worker structure represents a single worker thread (kworker) that executes
work items from a worker pool. Each worker is backed by a kernel thread stored
in the task field, and belongs to exactly one worker_pool referenced by the
pool field. When idle, workers are linked on the pool's idle_list via the entry
union member; when executing work, they're tracked in the busy_hash using
hentry. The current_work, current_func, and current_pwq fields track what work
item is currently being executed. Workers can be in various states indicated by
flags: idle, preparing to work, bound to a specific CPU, or marked for
termination. The scheduled list holds work items that are linked together and
must be processed as a group. Workers are dynamically created and destroyed by
the pool based on workload, with idle workers being reaped after a timeout.

```c
struct worker {
    /* List membership */
    union {
        struct list_head    entry;      /* In worklist */
        struct hlist_node   hentry;     /* In busy hash */
    };

    /* Current work */
    struct work_struct      *current_work;
    work_func_t             current_func;
    struct pool_workqueue   *current_pwq;

    /* State */
    unsigned long           last_active;
    unsigned int            flags;
    int                     id;

    /* Thread info */
    struct task_struct      *task;
    struct worker_pool      *pool;

    /* Scheduling */
    struct list_head        scheduled;
    struct list_head        node;
};
```

**Worker Flags:**
```c
#define WORKER_DIE              0x0002  /* Worker should exit */
#define WORKER_IDLE             0x0004  /* Worker is idle */
#define WORKER_PREP             0x0008  /* Preparing to run work */
#define WORKER_CPU_INTENSIVE    0x0040  /* CPU intensive work */
#define WORKER_UNBOUND          0x0080  /* Not bound to CPU */
#define WORKER_REBOUND          0x0100  /* Rebinding after CPU hotplug */
```

### 5.3 struct worker_pool

The worker_pool structure manages a collection of worker threads that share
common attributes and a work queue. There are typically two bound pools per CPU
(one normal priority, one high priority) plus additional unbound pools for work
that doesn't need CPU affinity. The worklist is a linked list of pending
work_struct items waiting to be executed. The pool maintains counts of total
workers (nr_workers) and idle workers (nr_idle), dynamically creating new
workers when there's pending work but no idle workers, and destroying excess
idle workers via the idle_timer. The busy_hash provides O(1) lookup of workers
by the work item they're currently executing, which is essential for work item
collision detection. The mayday_timer detects stalled pools where workers are
blocked and signals the rescuer to help. Pool attributes like CPU affinity and
NUMA node are stored in attrs, allowing unbound pools to be matched to specific
system topologies.

```c
struct worker_pool {
    raw_spinlock_t          lock;       /* Pool lock */
    int                     cpu;        /* CPU this pool is for (-1 if unbound) */
    int                     node;       /* NUMA node */
    int                     id;         /* Pool ID */
    unsigned int            flags;      /* Pool flags */

    /* Work queue */
    struct list_head        worklist;   /* Pending work items */

    /* Workers */
    int                     nr_workers; /* Total workers */
    int                     nr_idle;    /* Idle workers */
    struct list_head        idle_list;  /* Idle worker list */
    struct list_head        workers;    /* All workers */

    /* Management */
    struct timer_list       idle_timer; /* Idle worker timeout */
    struct timer_list       mayday_timer; /* Stall detection */

    /* Hash for busy workers */
    DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);

    /* Attributes */
    struct workqueue_attrs  *attrs;
    int                     refcnt;
};
```

### 5.4 worker_thread() - Main Loop

```c
static int worker_thread(void *__worker)
{
    struct worker *worker = __worker;
    struct worker_pool *pool = worker->pool;

    /* Set up thread */
    set_pf_worker(true);
    set_user_nice(current, WORKER_NICE);
    set_cpus_allowed_ptr(current, pool->attrs->cpumask);

    /* Mark ready */
    worker->task->flags |= PF_WQ_WORKER;

woke_up:
    raw_spin_lock_irq(&pool->lock);

    /* Rescuer can't work here if pool busy */
    if (unlikely(worker->flags & WORKER_DIE)) {
        raw_spin_unlock_irq(&pool->lock);
        set_pf_worker(false);
        return 0;
    }

    worker_leave_idle(worker);

recheck:
    /* Need manager? */
    if (!need_more_worker(pool))
        goto sleep;

    /* Still more work to do */
    if (likely(!list_empty(&pool->worklist)))
        goto keep_working;

sleep:
    worker_enter_idle(worker);
    __set_current_state(TASK_IDLE);
    raw_spin_unlock_irq(&pool->lock);
    schedule();
    goto woke_up;

keep_working:
    /* Process work items */
    do {
        struct work_struct *work =
            list_first_entry(&pool->worklist,
                            struct work_struct, entry);

        if (likely(!(*work_data_bits(work) & WORK_STRUCT_LINKED))) {
            /* Non-linked work - simple case */
            process_one_work(worker, work);
        } else {
            /* Linked work - multiple items */
            move_linked_works(work, &worker->scheduled, NULL);
            process_scheduled_works(worker);
        }
    } while (!list_empty(&pool->worklist) &&
             need_more_worker(pool));

    goto recheck;
}
```

### 5.5 process_one_work()

```c
static void process_one_work(struct worker *worker, struct work_struct *work)
{
    struct pool_workqueue *pwq = get_work_pwq(work);
    struct worker_pool *pool = worker->pool;
    bool cpu_intensive = pwq->wq->flags & WQ_CPU_INTENSIVE;
    unsigned long work_data;
    struct worker *collision;

    /* Find the work item in busy hash */
    collision = find_worker_executing_work(pool, work);
    if (unlikely(collision)) {
        move_linked_works(work, &collision->scheduled, NULL);
        return;
    }

    /* Clear linked and pending bits */
    set_work_pool_and_clear_pending(work, pool->id);

    /* Set up execution */
    worker->current_work = work;
    worker->current_func = work->func;
    worker->current_pwq = pwq;

    /* Track in busy hash */
    hash_add(pool->busy_hash, &worker->hentry,
             (unsigned long)work);

    /* CPU intensive: allow other workers */
    if (unlikely(cpu_intensive))
        worker_set_flags(worker, WORKER_CPU_INTENSIVE);

    raw_spin_unlock_irq(&pool->lock);

    /* Execute the work function */
    trace_workqueue_execute_start(work);
    worker->current_func(work);
    trace_workqueue_execute_end(work, worker->current_func);

    raw_spin_lock_irq(&pool->lock);

    /* Cleanup */
    if (unlikely(cpu_intensive))
        worker_clr_flags(worker, WORKER_CPU_INTENSIVE);

    hash_del(&worker->hentry);
    worker->current_work = NULL;
    worker->current_func = NULL;
    worker->current_pwq = NULL;
    pwq_dec_nr_in_flight(pwq, work_data);
}
```

### 5.6 Workqueue Flags

```c
/* Workqueue creation flags */
#define WQ_UNBOUND          0x00000002  /* Not CPU bound */
#define WQ_FREEZABLE        0x00000004  /* Frozen during suspend */
#define WQ_MEM_RECLAIM      0x00000008  /* May be used in memory reclaim */
#define WQ_HIGHPRI          0x00000010  /* High priority work */
#define WQ_CPU_INTENSIVE    0x00000020  /* CPU intensive work */
#define WQ_POWER_EFFICIENT  0x00000080  /* Power efficient scheduling */

/* Standard system workqueues */
extern struct workqueue_struct *system_wq;
extern struct workqueue_struct *system_highpri_wq;
extern struct workqueue_struct *system_long_wq;
extern struct workqueue_struct *system_unbound_wq;
extern struct workqueue_struct *system_freezable_wq;
extern struct workqueue_struct *system_power_efficient_wq;
```

### 5.7 Workqueue-Scheduler Interaction

```
                ┌────────────────────────────────────────┐
                │           User/Kernel Code             │
                └─────────────────┬──────────────────────┘
                                  │ queue_work()
                                  ▼
                ┌────────────────────────────────────────┐
                │            Workqueue                   │
                │  ┌─────────────────────────────────┐   │
                │  │      pool_workqueue (pwq)       │   │
                │  └─────────────────┬───────────────┘   │
                └────────────────────┼───────────────────┘
                                     │
                                     ▼
                ┌────────────────────────────────────────┐
                │           Worker Pool                  │
                │  ┌─────────────────────────────────┐   │
                │  │    worklist (pending work)      │   │
                │  └─────────────────┬───────────────┘   │
                │                    │                   │
                │  ┌─────────────────┴───────────────┐   │
                │  │         Worker Threads          │   │
                │  │  ┌─────────┐  ┌─────────┐       │   │
                │  │  │kworker/0│  │kworker/1│  ...  │   │
                │  │  └────┬────┘  └────┬────┘       │   │
                │  └───────┼───────────┼─────────────┘   │
                └──────────┼───────────┼─────────────────┘
                           │           │
                           ▼           ▼
                ┌────────────────────────────────────────┐
                │              Scheduler                 │
                │  ┌─────────────────────────────────┐   │
                │  │    Per-CPU Runqueues (rq)       │   │
                │  │  Workers scheduled as tasks     │   │
                │  └─────────────────────────────────┘   │
                └────────────────────────────────────────┘
```

---

## 6. Load Balancing

### 6.1 Overview

The scheduler's load balancing mechanism ensures work is evenly distributed
across CPUs for optimal throughput while respecting cache locality and NUMA
topology.

**Location:** `kernel/sched/fair.c`

### 6.2 sched_balance_rq() - Main Balance Function

```c
static int sched_balance_rq(int this_cpu, struct rq *this_rq,
                            struct sched_domain *sd,
                            enum cpu_idle_type idle,
                            int *continue_balancing)
{
    int ld_moved, cur_ld_moved, active_balance = 0;
    struct sched_group *group;
    struct rq *busiest;
    struct lb_env env = {
        .sd             = sd,
        .dst_cpu        = this_cpu,
        .dst_rq         = this_rq,
        .dst_grpmask    = group_balance_mask(sd->groups),
        .idle           = idle,
        .loop_break     = SCHED_NR_MIGRATE_BREAK,
        .cpus           = this_cpu_cpumask_var_ptr(load_balance_mask),
        .fbq_type       = all,
        .tasks          = LIST_HEAD_INIT(env.tasks),
    };

    cpumask_and(cpus, sched_domain_span(sd), cpu_active_mask);

    /* Find busiest group in domain */
    group = sched_balance_find_src_group(&env);
    if (!group)
        goto out_balanced;

    /* Find busiest runqueue in group */
    busiest = sched_balance_find_src_rq(&env, group);
    if (!busiest)
        goto out_balanced;

    /* Compute imbalance */
    env.src_cpu = busiest->cpu;
    env.src_rq = busiest;

    ld_moved = 0;
    if (busiest->nr_running > 1) {
        /* Move tasks from busiest to this_rq */
        env.loop_max = min(sysctl_sched_nr_migrate, busiest->nr_running);

        cur_ld_moved = detach_tasks(&env);
        if (cur_ld_moved) {
            attach_tasks(&env);
            ld_moved += cur_ld_moved;
        }
    }

    return ld_moved;

out_balanced:
    return 0;
}
```

### 6.3 Load Balance Environment

The lb_env structure encapsulates all the context needed for a single load
balancing operation. It is typically allocated on the stack and passed through
the load balancing call chain. The src_rq and dst_rq fields identify the source
(busiest) and destination (this CPU's) run queues for task migration. The
imbalance field specifies how much load should be moved, calculated based on
the difference between source and destination loads and the group type
classification. The cpus mask limits which CPUs are considered during the
balancing operation. Loop control fields (loop, loop_break, loop_max) prevent
spending too much time in a single balance operation by limiting iterations.
The tasks list temporarily holds tasks being migrated between detach_tasks()
and attach_tasks(). The migration_type and fbq_type fields influence which
tasks are eligible for migration, considering factors like NUMA placement and
cache affinity.

```c
struct lb_env {
    struct sched_domain     *sd;        /* Balancing domain */
    struct rq               *src_rq;    /* Source runqueue */
    struct rq               *dst_rq;    /* Destination runqueue */
    int                     src_cpu;
    int                     dst_cpu;

    enum cpu_idle_type      idle;       /* Idle state of dst */
    long                    imbalance;  /* Load to migrate */

    struct cpumask          *cpus;      /* CPUs to consider */
    unsigned int            flags;      /* LBF_* flags */

    unsigned int            loop;       /* Iteration count */
    unsigned int            loop_break; /* Break threshold */
    unsigned int            loop_max;   /* Maximum iterations */

    enum migration_type     migration_type;
    struct list_head        tasks;      /* Tasks being migrated */
    enum fbq_type           fbq_type;   /* NUMA filtering */
};
```

### 6.4 Group Classification

```c
enum group_type {
    group_has_spare = 0,    /* Idle CPUs available */
    group_fully_busy,       /* At or above capacity */
    group_misfit_task,      /* Task too big for CPU */
    group_smt_balance,      /* SMT imbalance */
    group_asym_packing,     /* Asymmetric capacity */
    group_imbalanced,       /* Affinity constrained */
    group_overloaded,       /* Most tasks per CPU */
};

enum group_type group_classify(unsigned int imbalance_pct,
                               struct sched_group *group,
                               struct sg_lb_stats *sgs)
{
    if (group_is_overloaded(imbalance_pct, sgs))
        return group_overloaded;

    if (sg_imbalanced(group))
        return group_imbalanced;

    if (sgs->group_asym_packing)
        return group_asym_packing;

    if (sgs->group_smt_balance)
        return group_smt_balance;

    if (sgs->group_misfit_task_load)
        return group_misfit_task;

    if (!group_has_capacity(imbalance_pct, sgs))
        return group_fully_busy;

    return group_has_spare;
}
```

### 6.5 select_task_rq_fair()

```c
static int select_task_rq_fair(struct task_struct *p, int prev_cpu,
                               int wake_flags)
{
    int sync = (wake_flags & WF_SYNC) && !(current->flags & PF_EXITING);
    struct sched_domain *tmp, *sd = NULL;
    int cpu = smp_processor_id();
    int new_cpu = prev_cpu;
    int want_affine = 0;

    /* Check wake affinity */
    if (wake_flags & WF_TTWU) {
        record_wakee(p);
        if (sched_energy_enabled()) {
            new_cpu = find_energy_efficient_cpu(p, prev_cpu);
            if (new_cpu >= 0)
                return new_cpu;
        }
        want_affine = !wake_wide(p) && cpumask_test_cpu(cpu, p->cpus_ptr);
    }

    /* Walk domain hierarchy */
    for_each_domain(cpu, tmp) {
        if (want_affine && (tmp->flags & SD_WAKE_AFFINE) &&
            cpumask_test_cpu(prev_cpu, sched_domain_span(tmp))) {
            if (cpu == prev_cpu)
                goto pick;

            if (wake_affine(tmp, p, cpu, prev_cpu, sync))
                new_cpu = cpu;
        }

        if (tmp->flags & sd_flag)
            sd = tmp;
    }

    if (sd) {
        new_cpu = sched_balance_find_dst_cpu(sd, p, cpu, prev_cpu, sd_flag);
    } else if (wake_flags & WF_TTWU) {
pick:
        new_cpu = select_idle_sibling(p, prev_cpu, new_cpu);
    }

    return new_cpu;
}
```

### 6.6 Load Balance Flow

```
scheduler_tick() / trigger_load_balance()
    │
    ├─► raise_softirq(SCHED_SOFTIRQ)
    │
    └─► [Later] run_rebalance_domains()
        │
        └─► rebalance_domains(this_rq, idle)
            │
            └─► For each sched_domain (bottom to top):
                │
                ├─► should_we_balance()
                │   └─► Check if this CPU is balance target
                │
                ├─► sched_balance_rq()
                │   │
                │   ├─► sched_balance_find_src_group()
                │   │   ├─► update_sd_lb_stats()
                │   │   │   └─► Gather per-group statistics
                │   │   └─► calculate_imbalance()
                │   │       └─► Determine load to move
                │   │
                │   ├─► sched_balance_find_src_rq()
                │   │   └─► Find busiest rq in group
                │   │
                │   ├─► detach_tasks()
                │   │   └─► Remove tasks from busiest rq
                │   │
                │   └─► attach_tasks()
                │       └─► Add tasks to this rq
                │
                └─► Update balance_interval
```

---

## 7. Scheduler Topology

### 7.1 Overview

Scheduler topology describes the hierarchical structure of CPUs, used to make
intelligent load balancing decisions based on cache sharing, NUMA distances,
and power efficiency.

**Location:** `kernel/sched/topology.c`, `include/linux/sched/topology.h`

### 7.2 struct sched_domain_topology_level

The sched_domain_topology_level structure defines a single level in the CPU
topology hierarchy used to build scheduling domains. The kernel maintains an
array of these structures representing the system topology from smallest
(SMT/hyperthreading) to largest (NUMA nodes or packages). The mask function
pointer returns a cpumask indicating which CPUs share resources at this level
for a given CPU. The sd_flags function returns the appropriate SD_* flags for
    domains at this level, controlling behaviors like load balancing and wake
    affinity. The numa_level field indicates the NUMA distance for NUMA-related
    levels. The data field provides per-CPU storage for domain structures at
    this level. During boot and CPU hotplug, build_sched_domains() iterates
    through these levels to construct the domain hierarchy. System
    administrators and architecture code can customize this array to match
    specific hardware topologies.

```c
struct sched_domain_topology_level {
    sched_domain_mask_f     mask;       /* Function returning CPU mask */
    sched_domain_flags_f    sd_flags;   /* Function returning SD flags */
    int                     numa_level; /* NUMA distance level */
    struct sd_data          data;       /* Per-CPU domain data */
    char                    *name;      /* Level name */
};

/* Default topology levels */
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
    { tl_smt_mask, cpu_smt_flags, "SMT" },
#endif
#ifdef CONFIG_SCHED_CLUSTER
    { tl_cls_mask, cpu_cluster_flags, "CLS" },
#endif
#ifdef CONFIG_SCHED_MC
    { tl_mc_mask, cpu_core_flags, "MC" },
#endif
    { tl_pkg_mask, NULL, "PKG" },
    { NULL, },
};
```

### 7.3 struct sched_domain

The sched_domain structure represents a scheduling domain at a particular level
of the CPU topology hierarchy. Each CPU has a chain of domains from the lowest
level (SMT siblings sharing a core) up to the highest level (all CPUs). The
parent and child pointers form a doubly-linked hierarchy, while groups points
to a circular list of sched_groups at this level. Balance parameters control
load balancing behavior: min_interval and max_interval bound how frequently
balancing occurs, busy_factor reduces balancing when the CPU is busy, and
imbalance_pct sets the threshold for considering groups imbalanced. The flags
field contains SD_* bits indicating domain characteristics like cache sharing
(SD_SHARE_LLC), NUMA boundary (SD_NUMA), or asymmetric packing preference
(SD_ASYM_PACKING). Runtime fields track the last balance time, current
interval, and consecutive failures. The span array (variable length) is a
cpumask of all CPUs in this domain. Scheduling domains are rebuilt during CPU
hotplug and can be examined via /proc/schedstat or debugfs.

```c
struct sched_domain {
    /* Hierarchy */
    struct sched_domain __rcu   *parent;    /* Parent domain */
    struct sched_domain __rcu   *child;     /* Child domain */
    struct sched_group          *groups;    /* Balancing groups */

    /* Balance parameters */
    unsigned long       min_interval;       /* Min balance interval */
    unsigned long       max_interval;       /* Max balance interval */
    unsigned int        busy_factor;        /* Reduce balancing when busy */
    unsigned int        imbalance_pct;      /* Imbalance threshold */
    unsigned int        cache_nice_tries;   /* Cache locality attempts */
    unsigned int        imb_numa_nr;        /* NUMA imbalance allowance */

    int                 nohz_idle;          /* NOHZ idle status */
    int                 flags;              /* SD_* flags */
    int                 level;              /* Domain level */

    /* Runtime state */
    unsigned long       last_balance;       /* Last balance time */
    unsigned int        balance_interval;   /* Current interval */
    unsigned int        nr_balance_failed;  /* Consecutive failures */

    /* Cost tracking */
    u64                 max_newidle_lb_cost;
    unsigned long       last_decay_max_lb_cost;

    /* Statistics */
    unsigned int        lb_count[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_failed[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_balanced[CPU_MAX_IDLE_TYPES];
    unsigned int        lb_imbalanced[CPU_MAX_IDLE_TYPES];

    /* Domain info */
    char                *name;
    unsigned int        span_weight;        /* CPUs in domain */
    unsigned long       span[];             /* CPU mask (variable) */
};
```

### 7.4 struct sched_group

The sched_group structure represents a group of CPUs within a scheduling domain
that are balanced as a unit. Domains contain a circular list of groups (linked
via next), where load balancing first identifies the busiest group, then the
busiest run queue within that group. The group_weight field indicates how many
CPUs are in the group, while cores counts physical cores (relevant for SMT).
The sgc pointer references a shared sched_group_capacity structure containing
aggregate capacity information for the group, including total capacity and
per-CPU min/max capacities (important for asymmetric systems like big.LITTLE).
The asym_prefer_cpu field identifies the preferred CPU in asymmetric packing
scenarios. The cpumask array (variable length) identifies which CPUs belong to
this group. Groups at non-NUMA levels typically don't overlap, while NUMA-level
groups may have overlapping CPU sets to represent the topology more accurately.
The reference count (ref) manages the lifetime of groups that may be shared
across CPUs.

```c
struct sched_group {
    struct sched_group      *next;          /* Circular list */
    atomic_t                ref;            /* Reference count */
    unsigned int            group_weight;   /* CPUs in group */
    unsigned int            cores;          /* Physical cores */
    struct sched_group_capacity *sgc;       /* Capacity info */
    int                     asym_prefer_cpu; /* Preferred CPU */
    unsigned long           cpumask[];      /* CPU mask */
};

struct sched_group_capacity {
    atomic_t                ref;
    unsigned long           capacity;       /* Total capacity */
    unsigned long           min_capacity;   /* Minimum CPU capacity */
    unsigned long           max_capacity;   /* Maximum CPU capacity */
    unsigned long           next_update;    /* Next update time */
    int                     imbalance;      /* Imbalance indicator */
    int                     id;             /* Unique ID */
    unsigned long           cpumask[];      /* CPU mask */
};
```

### 7.5 SD_* Flags

```c
/* Load balancing flags */
#define SD_BALANCE_NEWIDLE      0x0001  /* Balance when going idle */
#define SD_BALANCE_EXEC         0x0002  /* Balance on exec */
#define SD_BALANCE_FORK         0x0004  /* Balance on fork */
#define SD_BALANCE_WAKE         0x0008  /* Balance on wakeup */
#define SD_WAKE_AFFINE          0x0010  /* Wake task on waking CPU */

/* Topology flags */
#define SD_SHARE_CPUCAPACITY    0x0020  /* Members share CPU capacity (SMT) */
#define SD_SHARE_LLC            0x0040  /* Members share last-level cache */
#define SD_CLUSTER              0x0080  /* Members share CPU cluster */
#define SD_SERIALIZE            0x0100  /* Serialize balancing */
#define SD_ASYM_PACKING         0x0200  /* Pack tasks on preferred CPUs */
#define SD_PREFER_SIBLING       0x0400  /* Prefer sibling domains */
#define SD_NUMA                 0x0800  /* Cross-node balancing */

/* Asymmetric capacity */
#define SD_ASYM_CPUCAPACITY         0x1000  /* Asymmetric CPU capacity */
#define SD_ASYM_CPUCAPACITY_FULL    0x2000  /* Full asymmetry visible */
```

### 7.6 Domain Hierarchy

```
                    ┌─────────────────────────────────────┐
                    │           Root Domain               │
                    │  (All CPUs in scheduling scope)     │
                    └──────────────────┬──────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
    ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
    │   NUMA Domain   │      │   NUMA Domain   │      │   NUMA Domain   │
    │   (Node 0)      │      │   (Node 1)      │      │   (Node 2)      │
    │   SD_NUMA       │      │   SD_NUMA       │      │   SD_NUMA       │
    └────────┬────────┘      └────────┬────────┘      └────────┬────────┘
             │                        │                        │
    ┌────────┴────────┐      ┌────────┴────────┐      ┌────────┴────────┐
    ▼                 ▼      ▼                 ▼      ▼                 ▼
┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐   ┌───────────┐
│ MC Domain │   │ MC Domain │   │ MC Domain │   │ MC Domain │   │ MC Domain │
│ (Package) │   │ (Package) │   │ (Package) │   │ (Package) │   │ (Package) │
│ SD_SHARE_ │   │ SD_SHARE_ │   │ SD_SHARE_ │   │ SD_SHARE_ │   │ SD_SHARE_ │
│ LLC       │   │ LLC       │   │ LLC       │   │ LLC       │   │ LLC       │
└─────┬─────┘   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
      │               │               │               │               │
  ┌───┴───┐       ┌───┴───┐       ┌───┴───┐       ┌───┴───┐       ┌───┴───┐
  ▼       ▼       ▼       ▼       ▼       ▼       ▼       ▼       ▼       ▼
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │ │ SMT │
│  0  │ │  1  │ │  2  │ │  3  │ │  4  │ │  5  │ │  6  │ │  7  │ │  8  │ │  9  │
└─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘
  │ │     │ │     │ │     │ │     │ │     │ │     │ │     │ │     │ │     │ │
 T0 T1   T0 T1   T0 T1   T0 T1   T0 T1   T0 T1   T0 T1   T0 T1   T0 T1   T0 T1
 (Threads sharing core - SD_SHARE_CPUCAPACITY)
```

### 7.7 build_sched_domains()

```c
static int build_sched_domains(const struct cpumask *cpu_map,
                               struct sched_domain_attr *attr)
{
    struct sched_domain *sd;
    struct s_data d;
    int i, ret = -ENOMEM;

    /* Allocate domain structures */
    alloc_state = __visit_domain_allocation_hell(&d, cpu_map);
    if (alloc_state != sa_rootdomain)
        goto error;

    /* Build domains for each CPU */
    for_each_cpu(i, cpu_map) {
        struct sched_domain_topology_level *tl;
        sd = NULL;

        for_each_sd_topology(tl) {
            sd = build_sched_domain(tl, cpu_map, attr, sd, i);

            if (tl == sched_domain_topology)
                *per_cpu_ptr(d.sd, i) = sd;

            if (cpumask_equal(cpu_map, sched_domain_span(sd)))
                break;
        }
    }

    /* Build groups for each domain */
    for_each_cpu(i, cpu_map) {
        for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {
            sd->span_weight = cpumask_weight(sched_domain_span(sd));

            if (sd->flags & SD_NUMA)
                build_overlap_sched_groups(sd, i);
            else
                build_sched_groups(sd, i);
        }
    }

    /* Initialize capacities */
    for_each_cpu(i, cpu_map) {
        for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {
            init_sched_groups_capacity(i, sd);
        }
    }

    /* Attach domains to runqueues */
    for_each_cpu(i, cpu_map) {
        sd = *per_cpu_ptr(d.sd, i);
        cpu_attach_domain(sd, d.rd, i);
    }

    return 0;
}
```

---

## 8. PELT (Per-Entity Load Tracking)

### 8.1 Overview

PELT provides accurate load and utilization tracking for scheduling entities,
enabling better load balancing and energy-aware scheduling decisions.

**Location:** `kernel/sched/pelt.c`

### 8.2 struct sched_avg

The sched_avg structure stores Per-Entity Load Tracking (PELT) metrics for
scheduling entities, CFS run queues, and other scheduler components. PELT
provides accurate, time-decayed measurements of CPU demand that are essential
for load balancing and energy-aware scheduling. The core metrics are load_avg
    (weighted by task priority), runnable_avg (time spent runnable including
    waiting), and util_avg (actual CPU utilization). These averages are
    computed from corresponding sum fields (load_sum, runnable_sum, util_sum)
    using exponential decay with a half-life of approximately 32 milliseconds.
    The last_update_time tracks when the metrics were last updated, enabling
    proper decay calculation even when tasks sleep. The period_contrib field
    tracks partial contribution within the current 1024-microsecond PELT
    period. The util_est sub-structure provides utilization estimation that
    persists across sleep periods, helping predict future CPU demand. These
    metrics are propagated up the task group hierarchy and aggregated at the
    run queue level for load balancing decisions.

```c
struct sched_avg {
    u64                     last_update_time;  /* Time of last update */
    u64                     load_sum;          /* Running load sum */
    u64                     runnable_sum;      /* Runnable time sum */
    u32                     util_sum;          /* Utilization sum */
    u32                     period_contrib;    /* Contribution in period */

    unsigned long           load_avg;          /* Decayed load average */
    unsigned long           runnable_avg;      /* Decayed runnable avg */
    unsigned long           util_avg;          /* Decayed utilization */

    struct util_est         util_est;          /* Utilization estimation */
};

struct util_est {
    unsigned int            enqueued;          /* Util when enqueued */
    unsigned int            ewma;              /* Exponential average */
};
```

### 8.3 PELT Decay

```c
/*
 * PELT uses geometric series for load decay:
 *
 *   load(t) = load(t-1) * y + new_contrib
 *
 * Where y^32 ≈ 0.5 (contribution halves every 32ms)
 *
 * Period: 1024 microseconds (~1ms)
 * Maximum value: LOAD_AVG_MAX = 47742
 */

#define PELT_PERIOD         1024    /* microseconds */
#define LOAD_AVG_PERIOD     32      /* periods for half-life */
#define LOAD_AVG_MAX        47742   /* Maximum load_avg value */

/* Decay factor lookup table */
static const u32 runnable_avg_yN_inv[] = {
    [0] = 0xffffffff,   /* y^0 = 1 */
    [1] = 0xfa83b2da,   /* y^1 */
    [2] = 0xf5257d14,   /* y^2 */
    /* ... */
    [32] = 0x7fffffff,  /* y^32 ≈ 0.5 */
};
```

### 8.4 ___update_load_sum()

```c
static __always_inline int
___update_load_sum(u64 now, struct sched_avg *sa,
                   unsigned long load, unsigned long runnable,
                   int running)
{
    u64 delta;

    delta = now - sa->last_update_time;
    delta >>= 10;  /* Convert to PELT periods */

    if (!delta)
        return 0;

    sa->last_update_time += delta << 10;

    /*
     * Three-part sum:
     *   d1: Remainder from last incomplete period
     *   d2: Full periods with exponential decay
     *   d3: Current incomplete period
     */
    if (delta > 1) {
        /* Decay existing sums */
        sa->load_sum = decay_load(sa->load_sum, delta);
        sa->runnable_sum = decay_load(sa->runnable_sum, delta);
        sa->util_sum = decay_load(sa->util_sum, delta);
    }

    /* Add new contribution */
    sa->load_sum += load * delta;
    sa->runnable_sum += runnable * delta;
    if (running)
        sa->util_sum += delta;

    return 1;
}
```

### 8.5 ___update_load_avg()

```c
static __always_inline void
___update_load_avg(struct sched_avg *sa, unsigned long load)
{
    u32 divider = get_pelt_divider(sa);

    /*
     * Convert sums to averages:
     *   avg = sum / divider
     *
     * Where divider accounts for the decay series sum
     */
    sa->load_avg = div_u64(load * sa->load_sum, divider);
    sa->runnable_avg = div_u64(sa->runnable_sum, divider);
    sa->util_avg = sa->util_sum / divider;
}
```

### 8.6 update_load_avg()

```c
static inline void update_load_avg(struct cfs_rq *cfs_rq,
                                   struct sched_entity *se,
                                   int flags)
{
    u64 now = cfs_rq_clock_pelt(cfs_rq);
    int decayed;

    /* Update entity's load average */
    if (se->avg.last_update_time && !(flags & SKIP_AGE_LOAD))
        __update_load_avg_se(now, cfs_rq, se);

    /* Update runqueue's load average */
    decayed = update_cfs_rq_load_avg(now, cfs_rq);
    decayed |= propagate_entity_load_avg(se);

    /* Handle migration */
    if (!se->avg.last_update_time && (flags & DO_ATTACH)) {
        attach_entity_load_avg(cfs_rq, se);
        update_tg_load_avg(cfs_rq);
    } else if (flags & DO_DETACH) {
        detach_entity_load_avg(cfs_rq, se);
        update_tg_load_avg(cfs_rq);
    } else if (decayed) {
        cfs_rq_util_change(cfs_rq, 0);
        if (flags & UPDATE_TG)
            update_tg_load_avg(cfs_rq);
    }
}
```

### 8.7 PELT Visualization

```
Time →

Load Contribution
    ▲
    │  ██
    │  ██ ░░
    │  ██ ░░ ░
    │  ██ ░░ ░ ░
    │  ██ ░░ ░ ░ ░
    │  ██ ░░ ░ ░ ░ ░
    │  ██ ░░ ░ ░ ░ ░ ░
    ├──██─░░─░─░─░─░─░─░───────────────────────────────────────────►
    │   │  │  │  │  │  │  │                                    Time
    │   │  │  │  │  │  │  └─ y^6 × original
    │   │  │  │  │  │  └──── y^5 × original
    │   │  │  │  │  └─────── y^4 × original
    │   │  │  │  └────────── y^3 × original
    │   │  │  └───────────── y^2 × original
    │   │  └──────────────── y^1 × original
    │   └─────────────────── Current period (full weight)
    │
    └─ Contributions decay exponentially over time
       Half-life ≈ 32 periods (32ms)
```

---

## Summary

This document covered the essential components of the Linux kernel scheduler:

1. **Core Structures**: task_struct, rq, sched_class, sched_entity, cfs_rq, rt_rq
2. **TTWU**: Task wakeup mechanism with CPU selection and preemption checks
3. **IPI**: Inter-processor interrupts for cross-CPU scheduling coordination
4. **IRQ Handling**: Hardirq, softirq, and scheduler integration via preempt_count
5. **Workqueues**: Deferred execution in process context with kworker threads
6. **Load Balancing**: Domain-based task migration with imbalance detection
7. **Topology**: Hierarchical CPU organization for cache-aware scheduling
8. **PELT**: Exponentially decayed load tracking for fair scheduling decisions

These components work together to provide efficient, fair, and responsive task
scheduling across complex multi-core and NUMA systems.

---

## References

- Linux kernel source: `kernel/sched/`
- Documentation: `Documentation/scheduler/`
- Key files:
  - `kernel/sched/core.c` - Core scheduler
  - `kernel/sched/fair.c` - CFS implementation
  - `kernel/sched/rt.c` - Real-time scheduler
  - `kernel/sched/topology.c` - Domain building
  - `kernel/sched/pelt.c` - Load tracking
  - `kernel/workqueue.c` - Workqueue system
  - `kernel/softirq.c` - Softirq handling
