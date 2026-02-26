# Linux Kernel Locking and Synchronization Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Spinlocks](#spinlocks)
3. [Mutexes](#mutexes)
4. [Read-Write Locks](#read-write-locks)
5. [Seqlocks](#seqlocks)
6. [Atomic Operations](#atomic-operations)
7. [Memory Barriers](#memory-barriers)
8. [Per-CPU Variables](#per-cpu-variables)
9. [Wait Queues](#wait-queues)
10. [Completions](#completions)
11. [Guard/Cleanup Macros (RAII Locking)](#guardcleanup-macros-raii-locking)
12. [Lockdep](#lockdep)
13. [Lock Selection Guide](#lock-selection-guide)

---

## Introduction

The Linux kernel is a massively concurrent system where multiple CPUs execute
kernel code simultaneously, interrupts can preempt execution at almost any
point, and multiple processes compete for shared resources. Proper
synchronization is critical to prevent:

- **Race conditions**: Two threads accessing shared data without coordination
- **Data corruption**: Partial updates visible to other CPUs
- **Deadlocks**: Circular dependencies where threads wait forever
- **Priority inversion**: High-priority tasks blocked by low-priority ones

### Concurrency Sources in the Kernel

```
┌─────────────────────────────────────────────────────────────┐
│                    Concurrency Sources                      │
├─────────────────────────────────────────────────────────────┤
│  1. True Parallelism (SMP)                                  │
│     - Multiple CPUs executing simultaneously                │
│                                                             │
│  2. Preemption                                              │
│     - Kernel can be preempted (CONFIG_PREEMPT)              │
│     - Higher priority tasks can interrupt                   │
│                                                             │
│  3. Interrupts                                              │
│     - Hardware interrupts (IRQ)                             │
│     - Software interrupts (softirq)                         │
│     - Tasklets                                              │
│                                                             │
│  4. Workqueues                                              │
│     - Deferred work in process context                      │
│                                                             │
│  5. User-space requests                                     │
│     - Multiple processes making syscalls                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Spinlocks

Spinlocks are the most fundamental locking primitive in the kernel. They
provide mutual exclusion by "spinning" (busy-waiting) until the lock becomes
available.

### Core Data Structure

The spinlock_t structure is the primary spinlock type used throughout the
kernel for mutual exclusion in contexts where sleeping is not allowed. It wraps
a raw_spinlock and optionally includes lockdep debugging information. The
raw_spinlock_t contains the architecture-specific lock implementation along
with debugging fields like owner tracking. On x86, the underlying
arch_spinlock_t uses a ticket-based or queued design with head/tail counters to
ensure fair FIFO ordering among contending CPUs.

```c
/* include/linux/spinlock_types.h */
typedef struct spinlock {
    union {
        struct raw_spinlock rlock;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct lockdep_map dep_map;
#endif
    };
} spinlock_t;

typedef struct raw_spinlock {
    arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic, owner_cpu;
    void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
} raw_spinlock_t;

/* Architecture-specific (x86) - Queued Spinlock */
typedef struct qspinlock {
    union {
        atomic_t val;
        struct {
            u8 locked;      /* 0=unlocked, 1=locked */
            u8 pending;     /* one waiter bypasses MCS queue */
        };
        struct {
            u16 locked_pending;
            u16 tail;       /* MCS queue tail (CPU+1, index) */
        };
    };
} arch_spinlock_t;

/*
 * Qspinlock bit layout (NR_CPUS < 16K):
 *   Bits 0-7:   locked byte
 *   Bit 8:      pending (one waiter bypass)
 *   Bits 16-17: tail index (which per-CPU MCS node)
 *   Bits 18-31: tail CPU (+1)
 *
 * Note: Under CONFIG_PREEMPT_RT, spinlock_t maps to
 * rt_mutex_base, not raw_spinlock. Only raw_spinlock_t
 * remains a true spinning lock on PREEMPT_RT.
 */
```

### Spinlock API

```c
/* Static initialization */
DEFINE_SPINLOCK(my_lock);

/* Dynamic initialization */
spinlock_t my_lock;
spin_lock_init(&my_lock);

/* Basic locking */
spin_lock(&my_lock);           /* Acquire lock, disable preemption */
spin_unlock(&my_lock);         /* Release lock, enable preemption */

/* Try lock (non-blocking) */
if (spin_trylock(&my_lock)) {
    /* Got the lock */
    spin_unlock(&my_lock);
}

/* IRQ-safe variants */
spin_lock_irq(&my_lock);       /* Disable IRQs + acquire lock */
spin_unlock_irq(&my_lock);     /* Release lock + enable IRQs */

/* IRQ-safe with flags save/restore */
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);    /* Save IRQ state + disable + lock */
spin_unlock_irqrestore(&my_lock, flags); /* Unlock + restore IRQ state */

/* Softirq-safe variants */
spin_lock_bh(&my_lock);        /* Disable bottom halves + lock */
spin_unlock_bh(&my_lock);      /* Unlock + enable bottom halves */
```

### Ticket Spinlock (Historical)

The ticket spinlock was the traditional x86 spinlock implementation, now replaced
by the queued spinlock (qspinlock). Shown here for historical context:

```
┌─────────────────────────────────────────────────────────────┐
│                    Ticket Spinlock                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Lock State: [head=5] [tail=8]                              │
│                                                             │
│  Waiting Queue:                                             │
│  ┌────┐   ┌────┐   ┌────┐                                   │
│  │ 5  │ → │ 6  │ → │ 7  │   (ticket numbers)                │
│  │CURR│   │WAIT│   │WAIT│                                   │
│  └────┘   └────┘   └────┘                                   │
│    ↑                                                        │
│  Currently holding lock (head == my_ticket)                 │
│                                                             │
│  Acquire: my_ticket = atomic_fetch_add(&tail, 1)            │
│           while (head != my_ticket) cpu_relax();            │
│                                                             │
│  Release: atomic_inc(&head);                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Queued Spinlock (MCS-based)

Modern Linux uses queued spinlocks to reduce cache-line bouncing on NUMA
systems:

```
┌─────────────────────────────────────────────────────────────┐
│                   Queued Spinlock                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Each CPU spins on its own local variable (MCS node)        │
│                                                             │
│  Lock: [locked] [pending] [tail→]                           │
│                     │                                       │
│                     ▼                                       │
│         ┌──────┐   ┌──────┐   ┌──────┐                      │
│         │CPU 0 │ → │CPU 2 │ → │CPU 5 │                      │
│         │locked│   │ next │   │ next │                      │
│         └──────┘   └──────┘   └──────┘                      │
│                                                             │
│  Benefits:                                                  │
│  - Each waiter spins on local cache line                    │
│  - No cache-line bouncing on contention                     │
│  - FIFO ordering preserved                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Spinlocks

```
┌─────────────────────────────────────────────────────────────┐
│                 Spinlock Decision Tree                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Can you sleep while holding the lock?                      │
│     │                                                       │
│     ├── YES → Use mutex (not spinlock)                      │
│     │                                                       │
│     └── NO → Is the critical section very short?            │
│              │                                              │
│              ├── YES → spinlock is appropriate              │
│              │                                              │
│              └── NO → Consider restructuring code           │
│                                                             │
│  Additional considerations:                                 │
│  - Interrupt context? → Must use spinlock (can't sleep)     │
│  - Protecting from IRQ? → Use spin_lock_irq*                │
│  - Protecting from softirq? → Use spin_lock_bh              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Mutexes

Mutexes (mutual exclusion locks) are sleeping locks. When a mutex is contended,
the waiting task is put to sleep rather than spinning, making them more
efficient for longer critical sections.

### Core Data Structure

The mutex structure is the kernel's primary sleeping lock for mutual exclusion
in process context. The owner field stores both the owning task pointer and
status flags (WAITERS, HANDOFF) using the low bits. The wait_lock spinlock
protects the wait_list, which is a linked list of tasks blocked waiting for the
mutex. When a task cannot acquire the mutex, it adds itself to wait_list and
sleeps. The mutex supports adaptive spinning (optimistic spinning) where
waiters briefly spin if the owner is running on another CPU, avoiding the
overhead of context switches for short critical sections.

```c
/* include/linux/mutex_types.h */
struct mutex {
    atomic_long_t       owner;      /* Owner task + flags */
    raw_spinlock_t      wait_lock;  /* Protects wait_list */
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* MCS spinner queue for adaptive spinning */
#endif
    struct list_head    wait_list;  /* List of waiting tasks */
#ifdef CONFIG_DEBUG_MUTEXES
    void                *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};

/* Owner field encoding (kernel/locking/mutex.h) */
#define MUTEX_FLAG_WAITERS    0x01  /* Non-empty waiter list; wakeup needed */
#define MUTEX_FLAG_HANDOFF    0x02  /* Unlock must hand lock to top waiter */
#define MUTEX_FLAG_PICKUP     0x04  /* Handoff done, waiting for pickup */
#define MUTEX_FLAGS           0x07  /* All three low bits */
```

### Mutex API

```c
/* Static initialization */
DEFINE_MUTEX(my_mutex);

/* Dynamic initialization */
struct mutex my_mutex;
mutex_init(&my_mutex);

/* Basic operations */
mutex_lock(&my_mutex);         /* Acquire (sleeps if contended) */
mutex_unlock(&my_mutex);       /* Release */

/* Interruptible (can be interrupted by signals) */
if (mutex_lock_interruptible(&my_mutex)) {
    /* Interrupted by signal, didn't get lock */
    return -ERESTARTSYS;
}

/* Killable (only fatal signals interrupt) */
if (mutex_lock_killable(&my_mutex)) {
    return -EINTR;
}

/* Try lock (non-blocking) */
if (mutex_trylock(&my_mutex)) {
    /* Got the lock */
    mutex_unlock(&my_mutex);
}

/* Check if locked */
bool is_locked = mutex_is_locked(&my_mutex);
```

### Mutex Internals

```
┌─────────────────────────────────────────────────────────────┐
│                   Mutex State Machine                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Unlocked State:                                            │
│  owner = NULL (0)                                           │
│                                                             │
│  Locked (no waiters):                                       │
│  owner = current_task                                       │
│                                                             │
│  Locked (with waiters):                                     │
│  owner = current_task | MUTEX_FLAG_WAITERS                  │
│  wait_list = [task1] → [task2] → [task3]                    │
│                                                             │
│  Acquisition Flow:                                          │
│  1. Fast path: atomic cmpxchg(owner, 0, current)            │
│  2. Midpath: Brief spin if owner is running                 │
│  3. Slow path: Add to wait_list, sleep                      │
│                                                             │
│  Release Flow:                                              │
│  1. Fast path: atomic cmpxchg(owner, current, 0)            │
│  2. Slow path: Wake first waiter if WAITERS flag set        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Adaptive Spinning (Optimistic Spinning)

```c
/*
 * Mutexes implement adaptive spinning:
 * If the lock owner is running on another CPU, spin briefly
 * rather than immediately sleeping.
 *
 * This avoids the overhead of:
 * 1. Context switch to sleep
 * 2. Context switch to wake up
 * 3. Cache line migrations
 *
 * Only spin while:
 * - Lock owner is actively running
 * - No higher priority tasks need to run
 * - We haven't spun too long
 */
```

### Mutex Rules

1. **Only the owner can unlock**: Unlike semaphores, only the task that
   acquired the mutex can release it
2. **No recursive locking**: A task cannot acquire the same mutex twice
3. **Cannot be used in interrupt context**: Mutexes can sleep
4. **Must be released before returning to userspace**

---

## Read-Write Locks

Read-write locks allow multiple concurrent readers OR a single exclusive
writer.

### rwlock_t (Spinlock-based)

The rwlock_t structure provides a non-sleeping read-write spinlock that allows
multiple concurrent readers or a single exclusive writer. It wraps an
architecture-specific arch_rwlock_t and includes optional lockdep debugging
support. This lock is appropriate for short critical sections where sleeping is
not permitted, such as interrupt handlers. Writers must wait for all readers to
release before acquiring, and readers must wait if a writer holds the lock.

```c
/* include/linux/rwlock_types.h */
typedef struct {
    arch_rwlock_t raw_lock;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
} rwlock_t;

/* API */
DEFINE_RWLOCK(my_rwlock);

read_lock(&my_rwlock);         /* Shared access */
read_unlock(&my_rwlock);

write_lock(&my_rwlock);        /* Exclusive access */
write_unlock(&my_rwlock);

/* IRQ-safe variants */
read_lock_irq(&my_rwlock);
read_lock_irqsave(&my_rwlock, flags);
read_lock_bh(&my_rwlock);

write_lock_irq(&my_rwlock);
write_lock_irqsave(&my_rwlock, flags);
write_lock_bh(&my_rwlock);
```

### rw_semaphore (Sleeping)

The rw_semaphore structure is a sleeping read-write lock that allows multiple
concurrent readers or a single exclusive writer. The count field tracks the
reader count or indicates writer ownership. The owner field identifies the
current writer task for debugging and optimistic spinning. The wait_list holds
tasks blocked waiting to acquire the lock, protected by wait_lock. Unlike
rwlock_t, rw_semaphore can sleep when contended, making it suitable for longer
critical sections in process context. It also implements writer preference to
prevent writer starvation.

```c
/* include/linux/rwsem.h */
struct rw_semaphore {
    atomic_long_t count;        /* Reader count or writer flag */
    atomic_long_t owner;        /* Writer task + state flags */
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* MCS spinner queue */
#endif
    raw_spinlock_t wait_lock;
    struct list_head wait_list; /* Waiting readers/writers */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};

/* count field bit 0 is the write-lock bit */
#define RWSEM_WRITER_LOCKED     (1UL << 0)

/* API */
DECLARE_RWSEM(my_rwsem);

down_read(&my_rwsem);          /* Acquire read lock (sleeps) */
up_read(&my_rwsem);            /* Release read lock */

down_write(&my_rwsem);         /* Acquire write lock (sleeps) */
up_write(&my_rwsem);           /* Release write lock */

/* Interruptible variants */
down_read_interruptible(&my_rwsem);
down_write_killable(&my_rwsem);   /* Note: no down_write_interruptible */

/* Try variants */
down_read_trylock(&my_rwsem);
down_write_trylock(&my_rwsem);

/* Downgrade write to read */
downgrade_write(&my_rwsem);    /* Atomic: release write, acquire read */
```

### Read-Write Lock State

```
┌─────────────────────────────────────────────────────────────┐
│                  RW Lock States                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Unlocked:                                                  │
│  count = 0                                                  │
│                                                             │
│  Read-locked (N readers):                                   │
│  count = N                                                  │
│  ┌────────────────────────────────────┐                     │
│  │ Reader 1 │ Reader 2 │ ... │ Reader N │  (concurrent)     │
│  └────────────────────────────────────┘                     │
│                                                             │
│  Write-locked:                                              │
│  count = -1 (or special WRITER flag)                        │
│  ┌──────────┐                                               │
│  │ Writer   │  (exclusive)                                  │
│  └──────────┘                                               │
│                                                             │
│  Waiting:                                                   │
│  wait_list: [W1] → [R2] → [R3] → [W4]                       │
│             writer  readers    writer                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Reader-Writer Fairness

```
┌─────────────────────────────────────────────────────────────┐
│              RW Semaphore Fairness Policy                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Problem: Writer starvation                                 │
│  - Continuous stream of readers can starve writers          │
│                                                             │
│  Solution: Writer preference after waiter appears           │
│  - Once a writer is waiting, new readers must wait          │
│  - When writer releases, batch of waiting readers run       │
│  - FIFO ordering at phase boundaries                        │
│                                                             │
│  Timeline example:                                          │
│  R1 R2 R3 [W1 waits] R4 R5 blocked                          │
│       │                    │                                │
│       ▼                    ▼                                │
│  R1,R2,R3 complete → W1 runs → R4,R5 run                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Seqlocks

Seqlocks provide a mechanism for very fast reads of shared data where reads are
much more common than writes. Readers never block writers; instead, readers
detect concurrent writes and retry.

### Core Data Structure

The seqlock_t structure combines a sequence counter with a spinlock to provide
fast, lockless reads with exclusive writer access. The sequence counter is
incremented at the start and end of each write, so readers can detect
concurrent modifications by checking if the sequence changed. The embedded
spinlock serializes writers. The seqcount_t variant provides just the sequence
counter without a lock, allowing the caller to manage write serialization
separately. Seqlocks are ideal for read-mostly data like system time where
readers should never block writers.

```c
/* include/linux/seqlock_types.h */
typedef struct {
    seqcount_spinlock_t seqcount; /* Counter WITH associated spinlock */
    spinlock_t lock;
} seqlock_t;

/* Sequence counter only (no embedded lock) */
typedef struct seqcount {
    unsigned sequence;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
} seqcount_t;
```

### Seqlock API

```c
/* Static initialization */
DEFINE_SEQLOCK(my_seqlock);

/* Dynamic initialization */
seqlock_t my_seqlock;
seqlock_init(&my_seqlock);

/* Writer side (exclusive) */
write_seqlock(&my_seqlock);
/* ... modify data ... */
write_sequnlock(&my_seqlock);

/* Reader side (lockless, may retry) */
unsigned seq;
do {
    seq = read_seqbegin(&my_seqlock);
    /* ... read data into local variables ... */
} while (read_seqretry(&my_seqlock, seq));

/* IRQ-safe variants */
write_seqlock_irq(&my_seqlock);
write_seqlock_irqsave(&my_seqlock, flags);
write_seqlock_bh(&my_seqlock);
```

### How Seqlocks Work

```
┌─────────────────────────────────────────────────────────────┐
│                  Seqlock Operation                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Sequence counter: starts at 0 (even)                       │
│                                                             │
│  Writer:                                                    │
│  ┌─────────────────────────────────────────────┐            │
│  │ 1. sequence++ (now odd - write in progress) │            │
│  │ 2. smp_wmb() (memory barrier)               │            │
│  │ 3. <modify protected data>                  │            │
│  │ 4. smp_wmb() (memory barrier)               │            │
│  │ 5. sequence++ (now even - write complete)   │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Reader:                                                    │
│  ┌─────────────────────────────────────────────┐            │
│  │ 1. seq = sequence (read into local)         │            │
│  │ 2. if (seq & 1) retry (write in progress)   │            │
│  │ 3. smp_rmb() (memory barrier)               │            │
│  │ 4. <read protected data into locals>        │            │
│  │ 5. smp_rmb() (memory barrier)               │            │
│  │ 6. if (sequence != seq) retry (was updated) │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Key: Readers never block, they just detect and retry       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Seqcount Variants

```c
/* Raw seqcount (no protection, caller manages) */
seqcount_t seq;
seqcount_init(&seq);

/* Write side */
write_seqcount_begin(&seq);
/* ... modify data ... */
write_seqcount_end(&seq);

/* Seqcount with associated lock (stores POINTER to lock) */
seqcount_spinlock_t seq;       /* validates writer holds spinlock */
seqcount_raw_spinlock_t seq;   /* raw spinlock variant */
seqcount_mutex_t seq;          /* mutex variant */
seqcount_rwlock_t seq;         /* rwlock variant */

/* These validate that the associated lock is held on write */
```

### Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                Seqlock Use Cases                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✓ Good for:                                                │
│  - Read-mostly data (jiffies, xtime, vDSO time)             │
│  - Small data that's quickly read                           │
│  - Data where reads vastly outnumber writes                 │
│  - No pointers to dynamically allocated memory              │
│                                                             │
│  ✗ Bad for:                                                 │
│  - Data with pointers (may read freed memory!)              │
│  - Large data structures                                    │
│  - Write-heavy workloads                                    │
│                                                             │
│  Example: System time (jiffies_64)                          │
│  - Read by every timer interrupt handler                    │
│  - Updated once per tick                                    │
│  - Perfect for seqlock                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Atomic Operations

Atomic operations provide indivisible read-modify-write operations without full
locks.

### atomic_t and atomic64_t

The atomic_t and atomic64_t structures wrap integer values to provide atomic
read-modify-write operations without requiring locks. The counter field holds
the actual value, which can only be accessed through atomic API functions that
map to architecture-specific atomic instructions (like x86 LOCK prefix or ARM
LDREX/STREX). These types are used for reference counts, simple counters, and
flags where only the value itself needs protection. They support arithmetic
operations (add, sub, inc, dec), exchange operations (xchg, cmpxchg), and
bitwise operations, all with configurable memory ordering semantics.

```c
/* include/linux/atomic.h */
typedef struct {
    int counter;
} atomic_t;

typedef struct {
    s64 counter;
} atomic64_t;

/* Initialization */
atomic_t v = ATOMIC_INIT(0);
atomic64_t v64 = ATOMIC64_INIT(0);

/* Basic operations */
atomic_set(&v, 5);              /* v = 5 */
int x = atomic_read(&v);        /* x = v */

/* Arithmetic */
atomic_add(3, &v);              /* v += 3 */
atomic_sub(2, &v);              /* v -= 2 */
atomic_inc(&v);                 /* v++ */
atomic_dec(&v);                 /* v-- */

/* Return new value */
int new = atomic_add_return(3, &v);
int new = atomic_sub_return(2, &v);
int new = atomic_inc_return(&v);
int new = atomic_dec_return(&v);

/* Test and modify */
bool was_zero = atomic_dec_and_test(&v);  /* Returns true if result is 0 */
bool was_neg = atomic_inc_and_test(&v);   /* Returns true if result is 0 */
bool was_one = atomic_sub_and_test(1, &v);

/* Exchange */
int old = atomic_xchg(&v, 10);            /* old = v; v = 10 */

/* Compare and exchange */
int old = atomic_cmpxchg(&v, expected, new);
/* if (v == expected) { v = new; } return old_v; */

/* Try compare and exchange (returns bool) */
bool success = atomic_try_cmpxchg(&v, &expected, new);

/* Fetch and op (return old value) */
int old = atomic_fetch_add(3, &v);
int old = atomic_fetch_sub(2, &v);
int old = atomic_fetch_and(mask, &v);
int old = atomic_fetch_or(bits, &v);
int old = atomic_fetch_xor(bits, &v);
```

### Bitwise Atomics

```c
/* include/linux/bitops.h */
unsigned long flags;

/* Set/clear/toggle bits atomically */
set_bit(3, &flags);             /* flags |= (1 << 3) */
clear_bit(3, &flags);           /* flags &= ~(1 << 3) */
change_bit(3, &flags);          /* flags ^= (1 << 3) */

/* Test and modify (returns old value) */
bool was_set = test_and_set_bit(3, &flags);
bool was_set = test_and_clear_bit(3, &flags);
bool was_set = test_and_change_bit(3, &flags);

/* Non-atomic versions (faster, use when already protected) */
__set_bit(3, &flags);
__clear_bit(3, &flags);
__change_bit(3, &flags);

/* Test bit */
bool is_set = test_bit(3, &flags);
```

### Memory Ordering in Atomics

```c
/* Full barrier versions (default) */
atomic_add_return(1, &v);       /* Full memory barrier */

/* Relaxed versions (no barrier) */
atomic_add_return_relaxed(1, &v);

/* Acquire semantics (barrier after) */
atomic_add_return_acquire(1, &v);

/* Release semantics (barrier before) */
atomic_add_return_release(1, &v);

/*
 * Ordering:
 *
 * _relaxed: No ordering guarantees
 * _acquire: Subsequent loads/stores won't be reordered before
 * _release: Previous loads/stores won't be reordered after
 * (default): Full barrier both directions
 */
```

### Reference Counting

The refcount_t structure provides a safer alternative to raw atomic_t for
reference counting. It wraps an atomic_t but adds runtime overflow and
underflow detection to catch use-after-free and double-free bugs. The API
prevents incrementing a zero refcount (which would indicate a freed object) and
warns on overflow attempts. Unlike atomic_t, refcount_t is specifically
designed for object lifetime management and includes saturation semantics that
prevent wrapping. This makes it the preferred type for reference counting in
new kernel code.

```c
/* include/linux/refcount.h */
typedef struct refcount_struct {
    atomic_t refs;
} refcount_t;

#define REFCOUNT_INIT(n)    { .refs = ATOMIC_INIT(n) }

refcount_t ref = REFCOUNT_INIT(1);

/* Safe operations (with overflow/underflow checking) */
refcount_set(&ref, 1);
unsigned int val = refcount_read(&ref);

refcount_inc(&ref);                    /* Increment (warns if 0) */
bool not_zero = refcount_inc_not_zero(&ref);  /* Inc if not 0 */

bool is_zero = refcount_dec_and_test(&ref);   /* Dec and test for 0 */
bool is_zero = refcount_sub_and_test(n, &ref);

/* These are safer than raw atomic_t for reference counting */
/* They include checks for use-after-free and double-free */
```

---

## Memory Barriers

Memory barriers ensure ordering of memory operations across CPUs. Modern CPUs
and compilers reorder memory operations for performance; barriers prevent
unwanted reordering.

### Barrier Types

```c
/* include/asm/barrier.h */

/* Compiler barrier - prevents compiler reordering */
barrier();

/* Full memory barrier - orders all memory operations */
smp_mb();

/* Read memory barrier - orders reads only */
smp_rmb();

/* Write memory barrier - orders writes only */
smp_wmb();

/* Data dependency barrier (for pointer chasing) */
smp_read_barrier_depends();  /* Often no-op on modern CPUs */

/* Acquire/Release barriers */
smp_load_acquire(p);         /* Load with acquire semantics */
smp_store_release(p, v);     /* Store with release semantics */

/* Combined barrier with atomic */
smp_mb__before_atomic();     /* Barrier before following atomic */
smp_mb__after_atomic();      /* Barrier after preceding atomic */
```

### Understanding Memory Ordering

```
┌─────────────────────────────────────────────────────────────┐
│              Memory Ordering Problem                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU 0                          CPU 1                       │
│  ─────                          ─────                       │
│  A = 1;                         while (B == 0);             │
│  B = 1;                         print(A);                   │
│                                                             │
│  Expected: prints 1                                         │
│  Possible: prints 0!                                        │
│                                                             │
│  Why? CPU 0 might reorder stores, or                        │
│       CPU 1 might see B=1 before A=1 propagates             │
│                                                             │
│  Fix:                                                       │
│  CPU 0                          CPU 1                       │
│  ─────                          ─────                       │
│  A = 1;                         while (B == 0);             │
│  smp_wmb();                     smp_rmb();                  │
│  B = 1;                         print(A);                   │
│                                                             │
│  Now: Always prints 1                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Store-Load Barrier Pattern

```
┌─────────────────────────────────────────────────────────────┐
│              Acquire-Release Pattern                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Producer (writer)              Consumer (reader)           │
│  ────────────────               ───────────────             │
│  data.x = 1;                                                │
│  data.y = 2;                                                │
│  data.z = 3;                                                │
│  smp_store_release(&ready, 1);  while (!smp_load_acquire    │
│      ▲                               (&ready));             │
│      │                                  │                   │
│      │                                  ▼                   │
│  Release ensures all              Acquire ensures all       │
│  prior writes complete            subsequent reads see      │
│  before ready=1 visible           writes before ready=1     │
│                                                             │
│                                   use(data.x);  /* sees 1 */│
│                                   use(data.y);  /* sees 2 */│
│                                   use(data.z);  /* sees 3 */│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Barrier Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│              When to Use Which Barrier                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  barrier()                                                  │
│  - Prevent compiler reordering only                         │
│  - No CPU barrier, just optimizer fence                     │
│                                                             │
│  smp_wmb()                                                  │
│  - Publishing data: ensure stores visible in order          │
│  - Before setting "data ready" flag                         │
│                                                             │
│  smp_rmb()                                                  │
│  - Consuming data: ensure reads happen in order             │
│  - After seeing "data ready" flag                           │
│                                                             │
│  smp_mb()                                                   │
│  - Need full ordering in both directions                    │
│  - Expensive, avoid if possible                             │
│                                                             │
│  smp_load_acquire() / smp_store_release()                   │
│  - Preferred for most producer-consumer patterns            │
│  - More efficient than explicit barriers                    │
│  - Self-documenting intent                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Per-CPU Variables

Per-CPU variables provide each CPU with its own copy of data, eliminating
cache-line bouncing and most synchronization needs.

### Declaration and Access

```c
/* include/linux/percpu.h */

/* Static per-CPU variable */
DEFINE_PER_CPU(int, my_counter);
DEFINE_PER_CPU(struct my_struct, my_data);

/* With initial value */
DEFINE_PER_CPU(int, my_counter) = 0;

/* External declaration */
DECLARE_PER_CPU(int, my_counter);

/* Access (must disable preemption first) */
int val = this_cpu_read(my_counter);      /* Read current CPU's copy */
this_cpu_write(my_counter, 5);            /* Write current CPU's copy */
this_cpu_add(my_counter, 3);              /* Atomic add on current CPU */
this_cpu_inc(my_counter);
this_cpu_dec(my_counter);

/* Access with explicit CPU number */
int val = per_cpu(my_counter, cpu_id);

/* Raw access (requires preemption already disabled) */
int val = __this_cpu_read(my_counter);
__this_cpu_write(my_counter, 5);

/* Pointer to per-CPU variable */
int *ptr = this_cpu_ptr(&my_counter);
int *ptr = per_cpu_ptr(&my_counter, cpu_id);

/* Preemption-safe access block */
int cpu = get_cpu();        /* Disable preemption, get CPU id */
this_cpu_write(my_counter, 10);
/* ... access per-CPU data ... */
put_cpu();                  /* Re-enable preemption */
```

### Dynamic Per-CPU Allocation

```c
/* Allocate per-CPU memory at runtime */
void __percpu *ptr;

ptr = alloc_percpu(struct my_struct);      /* GFP_KERNEL */
ptr = alloc_percpu_gfp(struct my_struct, GFP_ATOMIC);
ptr = __alloc_percpu(size, align);

if (!ptr)
    return -ENOMEM;

/* Access */
struct my_struct *p = per_cpu_ptr(ptr, cpu);
struct my_struct *p = this_cpu_ptr(ptr);

/* Free */
free_percpu(ptr);
```

### Per-CPU Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                Per-CPU Variable Use Cases                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Statistics counters                                     │
│     - Each CPU increments its own counter                   │
│     - Sum all CPUs when reading                             │
│     - Example: network packet counts                        │
│                                                             │
│  2. Per-CPU caches/pools                                    │
│     - Each CPU has local object cache                       │
│     - Reduces contention on central pool                    │
│     - Example: SLUB per-CPU slabs                           │
│                                                             │
│  3. Current state                                           │
│     - Currently running task (current macro)                │
│     - Current interrupt state                               │
│     - Per-CPU runqueue in scheduler                         │
│                                                             │
│  4. Deferred work                                           │
│     - Per-CPU work items                                    │
│     - RCU callbacks                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Wait Queues

Wait queues allow tasks to sleep until a condition becomes true.

### Core Data Structures

The wait_queue_head structure is the anchor for a wait queue, containing a
spinlock to protect the list and the list head itself. Tasks waiting on a
condition add wait_queue_entry structures to this list and sleep. The
wait_queue_entry represents a single waiting task, with private typically
pointing to the current task, func being the wakeup callback function, and
flags controlling behavior like exclusive wakeups. When the condition becomes
true, wake_up() iterates through the list calling each entry's func to wake the
associated task.

```c
/* include/linux/wait.h */
struct wait_queue_head {
    spinlock_t              lock;
    struct list_head        head;
};
typedef struct wait_queue_head wait_queue_head_t;

struct wait_queue_entry {
    unsigned int            flags;
    void                    *private;      /* Usually current task */
    wait_queue_func_t       func;          /* Wakeup function */
    struct list_head        entry;
};
typedef struct wait_queue_entry wait_queue_entry_t;

/* Flags */
#define WQ_FLAG_EXCLUSIVE   0x01  /* Wake only one waiter */
#define WQ_FLAG_WOKEN       0x02  /* Was woken */
#define WQ_FLAG_CUSTOM      0x04  /* Custom wakeup function */
#define WQ_FLAG_DONE        0x08  /* Wait completed */
#define WQ_FLAG_PRIORITY    0x10  /* Priority waiter */
```

### Wait Queue API

```c
/* Static initialization */
DECLARE_WAIT_QUEUE_HEAD(my_waitqueue);

/* Dynamic initialization */
wait_queue_head_t my_waitqueue;
init_waitqueue_head(&my_waitqueue);

/* Simple wait (sleeps until condition is true) */
wait_event(my_waitqueue, condition);

/* Interruptible (returns -ERESTARTSYS if interrupted) */
wait_event_interruptible(my_waitqueue, condition);

/* With timeout */
long remaining = wait_event_timeout(my_waitqueue, condition, timeout_jiffies);
long remaining = wait_event_interruptible_timeout(wq, cond, timeout);

/* Wakeup */
wake_up(&my_waitqueue);            /* Wake one exclusive + all non-exclusive */
wake_up_all(&my_waitqueue);        /* Wake all waiters */
wake_up_interruptible(&my_waitqueue);
wake_up_interruptible_all(&my_waitqueue);

/* Check if anyone waiting */
bool has_waiters = waitqueue_active(&my_waitqueue);
```

### Wait Queue Internals

```
┌─────────────────────────────────────────────────────────────┐
│                 Wait Queue Operation                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  wait_event(wq, condition):                                 │
│  ┌────────────────────────────────────────────────┐         │
│  │ 1. Check condition - if true, return           │         │
│  │ 2. Create wait_queue_entry for current task    │         │
│  │ 3. Add entry to wq->head list                  │         │
│  │ 4. Loop:                                       │         │
│  │    a. set_current_state(TASK_UNINTERRUPTIBLE)  │         │
│  │    b. if (condition) break                     │         │
│  │    c. schedule() - sleep                       │         │
│  │ 5. Remove entry from list                      │         │
│  │ 6. set_current_state(TASK_RUNNING)             │         │
│  └────────────────────────────────────────────────┘         │
│                                                             │
│  wake_up(wq):                                               │
│  ┌────────────────────────────────────────────────┐         │
│  │ 1. For each entry in wq->head:                 │         │
│  │    a. Call entry->func (usually default_wake)  │         │
│  │    b. This sets task RUNNING and wakes it      │         │
│  │    c. If WQ_FLAG_EXCLUSIVE, stop after one     │         │
│  └────────────────────────────────────────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Exclusive Waiters

```c
/*
 * Exclusive waiters prevent thundering herd problem.
 * Only one exclusive waiter is woken per wake_up() call.
 */

/* Add as exclusive waiter */
DEFINE_WAIT(wait);
add_wait_queue_exclusive(&my_waitqueue, &wait);

/* Or use prepare_to_wait_exclusive */
DEFINE_WAIT(wait);
prepare_to_wait_exclusive(&wq, &wait, TASK_INTERRUPTIBLE);
schedule();
finish_wait(&wq, &wait);
```

---

## Completions

Completions are a simple synchronization mechanism for waiting until something
completes.

### Core Data Structure

The completion structure provides a simple mechanism for one task to signal
another that an event has occurred. The done counter tracks the number of
pending completions, while the wait field is a simple wait queue where waiting
tasks sleep. When a task calls wait_for_completion(), it sleeps until done
becomes non-zero. Calling complete() increments done and wakes one waiter,
while complete_all() sets done to a maximum value and wakes all waiters.
    Completions are commonly used to synchronize thread startup, I/O
    completion, and module unloading.

```c
/* include/linux/completion.h */
struct completion {
    unsigned int done;           /* Completion count */
    struct swait_queue_head wait; /* Simple wait queue */
};
```

### Completion API

```c
/* Static initialization */
DECLARE_COMPLETION(my_completion);

/* Dynamic initialization */
struct completion my_completion;
init_completion(&my_completion);

/* Wait for completion */
wait_for_completion(&my_completion);
wait_for_completion_interruptible(&my_completion);
wait_for_completion_killable(&my_completion);
wait_for_completion_timeout(&my_completion, timeout);
wait_for_completion_interruptible_timeout(&my_completion, timeout);

/* Signal completion */
complete(&my_completion);        /* Wake one waiter, increment done */
complete_all(&my_completion);    /* Wake all waiters, set done = UINT_MAX */
complete_on_current_cpu(&my_completion); /* Wake waiter on current CPU */

/* Reinitialize (for reuse) */
reinit_completion(&my_completion);

/* Check without waiting */
bool completed = try_wait_for_completion(&my_completion);
bool done = completion_done(&my_completion);
```

### Completion Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│               Completion Use Cases                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Thread synchronization                                  │
│     ┌─────────────────┐    ┌─────────────────┐              │
│     │ Parent thread   │    │ Child thread    │              │
│     ├─────────────────┤    ├─────────────────┤              │
│     │ kthread_create()│───▶│ start running   │              │
│     │ wait_for_       │    │ <initialize>    │              │
│     │   completion()  │◀───│ complete()      │              │
│     │ continue...     │    │ do_work()       │              │
│     └─────────────────┘    └─────────────────┘              │
│                                                             │
│  2. Request completion                                      │
│     - Submit I/O request                                    │
│     - wait_for_completion()                                 │
│     - IRQ handler calls complete()                          │
│                                                             │
│  3. Module unload synchronization                           │
│     - Module sets completion when safe to unload            │
│     - Unload waits for completion                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Guard/Cleanup Macros (RAII Locking)

The guard/cleanup infrastructure (`include/linux/cleanup.h`) provides RAII-style
scope-based resource management using GCC `__attribute__((cleanup))`. This is a
major modern addition that eliminates the "goto error" pattern and is now used
pervasively in new kernel code.

### Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│              Guard/Cleanup Infrastructure                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Problem: Manual lock/unlock pairs are error-prone          │
│  - Forget to unlock on error paths                          │
│  - Complex goto chains for cleanup                          │
│  - Duplicated unlock code                                   │
│                                                             │
│  Solution: RAII-style automatic cleanup                     │
│  - guard(mutex)(&my_mutex) auto-unlocks at scope end        │
│  - __free(kfree) auto-frees at scope end                    │
│  - no_free_ptr(p) transfers ownership out                   │
│                                                             │
│  Three main mechanisms:                                     │
│  1. __free(name) - automatic resource cleanup               │
│  2. guard(name)  - block-scoped lock acquisition            │
│  3. scoped_guard(name) - statement-scoped lock              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Resource Cleanup with __free()

```c
/* include/linux/cleanup.h */

/* Define a cleanup function */
DEFINE_FREE(kfree, void *, if (_T) kfree(_T));

/* Usage: variable auto-freed when it goes out of scope */
void *p __free(kfree) = kmalloc(size, GFP_KERNEL);
if (!p)
    return -ENOMEM;

/* Transfer ownership out (prevents auto-free) */
return no_free_ptr(p);

/* Or in a function returning a pointer */
return_ptr(p);  /* equivalent to return no_free_ptr(p) */
```

### Lock Guards

```c
/* guard(name) - held for rest of enclosing scope */
guard(mutex)(&my_mutex);
/* ... critical section ... */
/* mutex_unlock() called automatically at scope end */

/* scoped_guard(name) - held only for the compound statement */
scoped_guard(spinlock_irqsave, &my_lock) {
    /* ... critical section ... */
}
/* spin_unlock_irqrestore() called automatically */

/* scoped_guard with conditional lock - body SKIPPED if lock fails */
scoped_guard(mutex_try, &my_mutex) {
    /* only reached if mutex_trylock succeeded */
    do_work();
}

/* scoped_cond_guard - conditional with explicit failure action */
scoped_cond_guard(mutex_intr, return -ERESTARTSYS, &my_mutex) {
    do_work();
}

/* Zero-argument guards (no lock pointer needed) */
guard(rcu)();           /* rcu_read_lock/unlock */
guard(irqsave)();       /* local_irq_save/restore (flags in struct) */
guard(preempt)();       /* preempt_disable/enable */
```

### Available Guards

```
┌─────────────────────────────────────────────────────────────┐
│               Registered Guard Names                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Spinlock guards (include/linux/spinlock.h):                │
│  ──────────────────────────────────────────                 │
│  guard(raw_spinlock)         raw_spin_lock/unlock           │
│  guard(raw_spinlock_irq)     + IRQ disable/enable           │
│  guard(raw_spinlock_irqsave) + IRQ save/restore             │
│  guard(spinlock)             spin_lock/unlock               │
│  guard(spinlock_irq)         + IRQ disable/enable           │
│  guard(spinlock_irqsave)     + IRQ save/restore             │
│  guard(read_lock)            read_lock/unlock               │
│  guard(read_lock_irq)        + IRQ disable/enable           │
│  guard(read_lock_irqsave)    + IRQ save/restore             │
│  guard(write_lock)           write_lock/unlock              │
│  guard(write_lock_irq)       + IRQ disable/enable           │
│  guard(write_lock_irqsave)   + IRQ save/restore             │
│                                                             │
│  All spinlock guards also have _try conditional variants    │
│                                                             │
│  Mutex guards (include/linux/mutex.h):                      │
│  ─────────────────────────────────────                      │
│  guard(mutex)                mutex_lock/unlock              │
│  guard(mutex_try)            mutex_trylock (conditional)    │
│  guard(mutex_intr)           mutex_lock_interruptible       │
│                                                             │
│  RW semaphore guards (include/linux/rwsem.h):               │
│  ────────────────────────────────────────────                │
│  guard(rwsem_read)           down_read/up_read              │
│  guard(rwsem_read_try)       trylock (conditional)          │
│  guard(rwsem_read_intr)      interruptible (conditional)    │
│  guard(rwsem_write)          down_write/up_write            │
│  guard(rwsem_write_try)      trylock (conditional)          │
│                                                             │
│  Zero-argument guards (no pointer parameter):               │
│  ────────────────────────────────────────────                │
│  guard(rcu)                  rcu_read_lock/unlock           │
│  guard(irq)                  local_irq_disable/enable       │
│  guard(irqsave)              local_irq_save/restore         │
│  guard(preempt)              preempt_disable/enable         │
│  guard(preempt_notrace)      notrace variant                │
│  guard(migrate)              migrate_disable/enable         │
│  guard(cpus_read_lock)       CPU hotplug read lock          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Defining Custom Guards

```c
/* include/linux/cleanup.h */

/* DEFINE_GUARD(name, type, lock_expr, unlock_expr) */
DEFINE_GUARD(my_lock, struct my_lock_type *,
             my_lock_acquire(_T), my_lock_release(_T));

/* Conditional guard (trylock variant) */
DEFINE_GUARD_COND(my_lock, _try, my_trylock(_T));

/* Zero-argument guard */
DEFINE_LOCK_GUARD_0(my_context,
    my_context_enter(),
    my_context_exit());

/* One-argument guard with extra state */
DEFINE_LOCK_GUARD_1_COND(my_lock, _intr,
    my_lock_interruptible(_T) == 0);
```

### LIFO Ordering Rule

```c
/*
 * CRITICAL: Multiple cleanup variables in the same scope are
 * cleaned up in REVERSE definition order (LIFO).
 *
 * This means: define the lock guard BEFORE the __free variable
 * so the lock is still held when the resource cleanup runs.
 */

/* CORRECT */
guard(mutex)(&lock);                    /* defined first, cleaned up last */
void *p __free(kfree) = kmalloc(...);   /* defined second, freed first */
/* p is freed while lock is still held */

/* WRONG - lock released before free */
void *p __free(kfree) = kmalloc(...);   /* defined first, freed LAST */
guard(mutex)(&lock);                    /* defined second, unlocked FIRST */
/* lock released before p is freed - potential use-after-free! */
```

---

## Lockdep

Lockdep is the kernel's runtime locking correctness validator. It detects
potential deadlocks, improper lock usage, and lock ordering violations.

### What Lockdep Detects

```
┌─────────────────────────────────────────────────────────────┐
│                 Lockdep Detections                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Lock ordering violations (potential deadlocks)          │
│     CPU 0: lock(A); lock(B);                                │
│     CPU 1: lock(B); lock(A);  ← DEADLOCK!                   │
│                                                             │
│  2. Recursive locking (without annotation)                  │
│     lock(A); lock(A);  ← Usually a bug                      │
│                                                             │
│  3. Lock-under-lock inconsistencies                         │
│     IRQ context: lock(A)                                    │
│     Process: lock(A) without disabling IRQs ← BUG!          │
│                                                             │
│  4. Held lock at improper times                             │
│     Returning to userspace with locks held                  │
│     Sleeping with spinlock held                             │
│                                                             │
│  5. Incorrect nesting                                       │
│     lock(A); lock(B); unlock(A); ← Wrong order              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Lock Classes

```c
/* Lockdep groups locks into "classes" based on initialization */
static DEFINE_SPINLOCK(my_lock);  /* This is one class */

struct my_struct {
    spinlock_t lock;
};

/* All instances share one class by default */
struct my_struct a, b;
spin_lock_init(&a.lock);  /* Same class */
spin_lock_init(&b.lock);  /* Same class */

/* Create separate classes for same-type locks */
static struct lock_class_key my_key;
spin_lock_init(&a.lock);
lockdep_set_class(&a.lock, &my_key);  /* Different class */
```

### Lockdep Annotations

```c
/* Nested locking (same class, different instances) */
spin_lock(&parent->lock);
spin_lock_nested(&child->lock, SINGLE_DEPTH_NESTING);

/* Subclass for complex hierarchies */
spin_lock_nested(&lock, subclass);

/* Mark lock as intentionally recursive */
spin_lock_irqsave_nested(&lock, flags, SINGLE_DEPTH_NESTING);

/* Assert lock is held */
lockdep_assert_held(&lock);
lockdep_assert_held_write(&rwsem);
lockdep_assert_held_read(&rwsem);

/* Mark as intentionally not held */
lockdep_assert_not_held(&lock);

/* For locks acquired in special contexts */
lock_acquire_exclusive/shared(...);
lock_release(...);
```

### Common Lockdep Messages

```
┌─────────────────────────────────────────────────────────────┐
│              Lockdep Warning Examples                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  "possible circular locking dependency detected"            │
│  ─────────────────────────────────────────────              │
│  Shows the lock chain that could deadlock:                  │
│  -> #0 (lock_A){....}:                                      │
│       lock_acquire+0x...                                    │
│  -> #1 (lock_B){....}:                                      │
│       lock_acquire+0x...                                    │
│                                                             │
│  "inconsistent lock state"                                  │
│  ────────────────────────                                   │
│  Lock used in IRQ but not marked IRQ-safe:                  │
│  irq event stamp: 12345                                     │
│  hardirqs last enabled at: ...                              │
│                                                             │
│  "BUG: sleeping function called from invalid context"       │
│  ──────────────────────────────────────────────────         │
│  Tried to sleep while holding spinlock or in atomic         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Lock Selection Guide

### Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│                  Lock Selection Matrix                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Need to sleep in critical section?                         │
│  ├── Yes: Must use sleeping lock                            │
│  │   ├── Exclusive access? → mutex                          │
│  │   └── Reader/writer? → rw_semaphore                      │
│  │                                                          │
│  └── No: Can use spinning lock                              │
│      ├── Very short critical section? → spinlock            │
│      ├── Reader/writer, short? → rwlock_t                   │
│      ├── Read-mostly, can retry? → seqlock                  │
│      └── No sharing needed? → per-CPU variable              │
│                                                             │
│  Protecting from interrupts?                                │
│  ├── Hardware IRQ: *_irq or *_irqsave variants              │
│  └── Softirq/BH: *_bh variants                              │
│                                                             │
│  Quick reference counts? → atomic_t / refcount_t            │
│  Only need to order operations? → memory barriers           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cost Comparison

```
┌─────────────────────────────────────────────────────────────┐
│               Synchronization Costs                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Cheapest                                                   │
│  ────────                                                   │
│  1. Per-CPU variables (no sync needed)                      │
│  2. RCU read-side (no atomic ops)                           │
│  3. Atomic operations                                       │
│  4. Uncontended spinlock                                    │
│  5. Uncontended mutex                                       │
│  6. Contended spinlock (spinning)                           │
│  7. Contended mutex (context switch)                        │
│  ────────                                                   │
│  Most expensive                                             │
│                                                             │
│  Notes:                                                     │
│  - Contention is the killer                                 │
│  - Context switches: ~1000s of cycles                       │
│  - Cache-line bouncing on NUMA: can be worse                │
│  - Choose primitive based on critical section length        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common Patterns

```c
/* Pattern 1: Simple exclusive access */
static DEFINE_MUTEX(my_mutex);

mutex_lock(&my_mutex);
/* ... critical section (may sleep) ... */
mutex_unlock(&my_mutex);

/* Pattern 2: Interrupt-safe spinlock */
static DEFINE_SPINLOCK(my_lock);
unsigned long flags;

spin_lock_irqsave(&my_lock, flags);
/* ... critical section (cannot sleep) ... */
spin_unlock_irqrestore(&my_lock, flags);

/* Pattern 3: Reader-writer with sleeping */
static DECLARE_RWSEM(my_rwsem);

/* Reader */
down_read(&my_rwsem);
/* ... read data ... */
up_read(&my_rwsem);

/* Writer */
down_write(&my_rwsem);
/* ... modify data ... */
up_write(&my_rwsem);

/* Pattern 4: Statistics counter */
static DEFINE_PER_CPU(unsigned long, stats);

this_cpu_inc(stats);  /* Fast, lock-free */

/* To read total: */
unsigned long total = 0;
int cpu;
for_each_possible_cpu(cpu)
    total += per_cpu(stats, cpu);

/* Pattern 5: Producer-consumer with completion */
static DECLARE_COMPLETION(work_done);

/* Producer */
do_work();
complete(&work_done);

/* Consumer */
wait_for_completion(&work_done);
use_result();
```

---

## Summary

| Primitive | Sleeps? | IRQ-safe? | Best For |
|-----------|---------|-----------|----------|
| spinlock_t | No | Yes | Short critical sections, IRQ context |
| mutex | Yes | No | Longer critical sections, process context |
| rwlock_t | No | Yes | Read-heavy, short operations |
| rw_semaphore | Yes | No | Read-heavy, longer operations |
| seqlock | No (readers) | N/A | Read-mostly, small data |
| atomic_t | N/A | Yes | Simple counters, flags |
| per-CPU | N/A | N/A | Per-CPU statistics, caches |
| completion | Yes | No | Thread synchronization |
| wait_queue | Yes | No | Complex wait conditions |
| RCU | No (read) | Yes (read) | Read-mostly, pointer-based data |
| guard() | N/A | N/A | RAII lock management (any lock) |

Understanding when to use each primitive is key to writing correct, efficient
kernel code. Always consider the critical section length, whether you might
sleep, and whether interrupts need protection. For new code, prefer guard()
and scoped_guard() over manual lock/unlock pairs to eliminate error-path bugs.
