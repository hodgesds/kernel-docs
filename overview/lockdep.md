# Linux Kernel Lockdep (Lock Dependency Validator) Overview

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Core Data Structures](#core-data-structures)
3. [Lock Class Registration](#lock-class-registration)
4. [Dependency Graph Construction](#dependency-graph-construction)
5. [Deadlock Detection Algorithm](#deadlock-detection-algorithm)
6. [Lock Usage State Tracking](#lock-usage-state-tracking)
7. [Lockdep Annotations](#lockdep-annotations)
8. [Lock Statistics](#lock-statistics)
9. [Common Lockdep Warnings](#common-lockdep-warnings)
10. [Performance Overhead and Configuration](#performance-overhead-and-configuration)

---

## Architecture Overview

Lockdep is the kernel's runtime locking correctness validator. It was created
by Ingo Molnar and Peter Zijlstra (Copyright 2006-2007, Red Hat) and is
implemented primarily in `kernel/locking/lockdep.c`. It proves that all
observed locking sequences are mathematically correct and that no possible
combination of CPUs, tasks, and interrupt contexts could produce a deadlock.

The key insight behind lockdep is that it operates on **lock classes**, not
individual lock instances. Two spinlocks protecting different instances of
the same data structure (e.g., two different inodes) belong to the same lock
class. This allows lockdep to detect potential deadlocks even if the specific
combination of locks that would trigger the deadlock has never been observed
in practice.

```
Lockdep detects three classes of locking bugs
(kernel/locking/lockdep.c:13-17):

  1. Lock inversion scenarios
  2. Circular lock dependencies
  3. Hardirq/softirq safe/unsafe locking bugs

Bugs are reported even if the current locking scenario does not
cause any deadlock at this point.
```

### High-Level Architecture

```
                        Lock Acquire/Release Events
                                   |
                                   v
         +--------------------------------------------------+
         |               lock_acquire() / lock_release()     |
         |           (kernel/locking/lockdep.c:5814,5856)    |
         +--------------------------------------------------+
                    |                          |
                    v                          v
    +----------------------------+   +----------------------+
    |   register_lock_class()    |   |   __lock_acquire()   |
    |   (lockdep.c:1281)         |   |   (lockdep.c:5073)   |
    +----------------------------+   +----------------------+
                                       |       |        |
                           +-----------+       |        +----------+
                           v                   v                   v
                  +----------------+  +------------------+  +-------------+
                  | mark_usage()   |  | validate_chain() |  | chain key   |
                  | (lockdep.c:    |  | (lockdep.c:3857) |  | computation |
                  |  4614)         |  +------------------+  +-------------+
                  +----------------+     |            |
                           |             v            v
                           |    +---------------+  +------------------+
                           |    |check_deadlock |  |check_prevs_add() |
                           |    |(lockdep.c:    |  |(lockdep.c:3254)  |
                           |    | 3053)         |  +------------------+
                           |    +---------------+    |           |
                           v                         v           v
                  +------------------+    +----------------+ +-----------+
                  | IRQ safety check |    | check_prev_add | | check_irq |
                  | (mark_lock_irq)  |    | (lockdep.c:    | | _usage()  |
                  +------------------+    |  3118)         | +-----------+
                                          +----------------+
                                               |
                                               v
                                     +-------------------+
                                     |check_noncircular()|
                                     | BFS cycle detect  |
                                     |(lockdep.c:2180)   |
                                     +-------------------+
```

The entire system is protected by its own internal `arch_spinlock_t`
(`kernel/locking/lockdep.c:136`) which avoids recursion into lockdep itself.
Lockdep guards against re-entrancy with a per-CPU recursion counter:

```c
/* kernel/locking/lockdep.c:111 */
DEFINE_PER_CPU(unsigned int, lockdep_recursion);

/* kernel/locking/lockdep.c:114-126 */
static __always_inline bool lockdep_enabled(void)
{
    if (!debug_locks)
        return false;
    if (this_cpu_read(lockdep_recursion))
        return false;
    if (current->lockdep_recursion)
        return false;
    return true;
}
```

---

## Core Data Structures

Lockdep revolves around four fundamental data structures: `lock_class`,
`lock_chain`, `held_lock`, and `lock_class_key`. These are defined across
`include/linux/lockdep_types.h` and `include/linux/lockdep.h`.

### lock_class_key -- Lock Identity

A `lock_class_key` identifies a lock class. For static locks, the address of
the key (which is typically embedded in the lock itself) serves as the unique
identifier. For dynamically allocated locks, the key must be explicitly
registered with `lockdep_register_key()`.

```c
/* include/linux/lockdep_types.h:70-80 */
struct lockdep_subclass_key {
    char __one_byte;
} __attribute__ ((__packed__));

struct lock_class_key {
    union {
        struct hlist_node               hash_entry;
        struct lockdep_subclass_key     subkeys[MAX_LOCKDEP_SUBCLASSES];
    };
};
```

The `subkeys` array supports up to `MAX_LOCKDEP_SUBCLASSES` (8) subclasses
per key. Each subclass gets its own `lock_class` entry, which is how lockdep
handles nested locking of the same lock type (e.g., locking a parent inode
then a child inode).

### lock_class -- The Central Node

Each unique lock class (key + subclass combination) maps to one `lock_class`
instance. These form the nodes of lockdep's dependency graph. Lock classes
are stored in a statically allocated array.

```c
/* include/linux/lockdep_types.h:98-147 */
struct lock_class {
    /*
     * class-hash:
     */
    struct hlist_node               hash_entry;

    /*
     * Entry in all_lock_classes when in use. Entry in free_lock_classes
     * when not in use.
     */
    struct list_head                lock_entry;

    /*
     * These fields represent a directed graph of lock dependencies,
     * to every node we attach a list of "forward" and a list of
     * "backward" graph nodes.
     */
    struct list_head                locks_after, locks_before;

    const struct lockdep_subclass_key *key;
    lock_cmp_fn                     cmp_fn;
    lock_print_fn                   print_fn;

    unsigned int                    subclass;
    unsigned int                    dep_gen_id;

    /*
     * IRQ/softirq usage tracking bits:
     */
    unsigned long                   usage_mask;
    const struct lock_trace         *usage_traces[LOCK_TRACE_STATES];

    const char                      *name;
    int                             name_version;

    u8                              wait_type_inner;
    u8                              wait_type_outer;
    u8                              lock_type;

#ifdef CONFIG_LOCK_STAT
    unsigned long                   contention_point[LOCKSTAT_POINTS];
    unsigned long                   contending_point[LOCKSTAT_POINTS];
#endif
} __no_randomize_layout;
```

The `locks_after` and `locks_before` lists are the adjacency lists of the
dependency graph. Each entry in these lists is a `lock_list` that points to
another `lock_class`, forming a directed edge in the graph.

The global storage is a fixed-size array:

```c
/* kernel/locking/lockdep.c:218 */
struct lock_class lock_classes[MAX_LOCKDEP_KEYS];
DECLARE_BITMAP(lock_classes_in_use, MAX_LOCKDEP_KEYS);

/* include/linux/lockdep_types.h:202-203 */
#define MAX_LOCKDEP_KEYS_BITS       13
#define MAX_LOCKDEP_KEYS            (1UL << MAX_LOCKDEP_KEYS_BITS)  /* 8192 */
```

### lockdep_map -- Per-Instance Mapping

Every lock instance (every `spinlock_t`, `mutex`, etc.) embeds a
`lockdep_map` that maps the instance to its `lock_class`.

```c
/* include/linux/lockdep_types.h:186-198 */
struct lockdep_map {
    struct lock_class_key           *key;
    struct lock_class               *class_cache[NR_LOCKDEP_CACHING_CLASSES];
    const char                      *name;
    u8                              wait_type_outer;
    u8                              wait_type_inner;
    u8                              lock_type;
#ifdef CONFIG_LOCK_STAT
    int                             cpu;
    unsigned long                   ip;
#endif
};
```

The `class_cache` array (size 2) caches the most recently looked-up lock
class for subclass 0 and subclass 1, avoiding repeated hash table lookups.

### held_lock -- Per-Task Lock Stack

When a task acquires a lock, a `held_lock` entry is pushed onto the task's
lock stack (`current->held_locks[]`). This stack has a maximum depth of 48.

```c
/* include/linux/lockdep_types.h:206-257 */
struct held_lock {
    /*
     * One-way hash of the dependency chain up to this point.
     * Used for dependency-caching to skip redundant checks.
     */
    u64                             prev_chain_key;
    unsigned long                   acquire_ip;
    struct lockdep_map              *instance;
    struct lockdep_map              *nest_lock;
#ifdef CONFIG_LOCK_STAT
    u64                             waittime_stamp;
    u64                             holdtime_stamp;
#endif
    unsigned int                    class_idx:MAX_LOCKDEP_KEYS_BITS;
    unsigned int irq_context:2;   /* bit 0 - soft, bit 1 - hard */
    unsigned int trylock:1;
    unsigned int read:2;          /* 0=exclusive, 1=read, 2=read-recursive */
    unsigned int check:1;         /* full validation or simple checks */
    unsigned int hardirqs_off:1;
    unsigned int sync:1;
    unsigned int references:11;
    unsigned int pin_count;
};
```

The `prev_chain_key` is critical for performance: it stores a rolling 64-bit
hash of all locks held up to this point. When a new lock is acquired, lockdep
checks if this exact chain has been validated before (cache hit), avoiding
the expensive O(N^2) dependency checks on subsequent occurrences.

```c
/* include/linux/sched.h:1237,1241 */
# define MAX_LOCK_DEPTH                 48UL
    struct held_lock            held_locks[MAX_LOCK_DEPTH];
```

### lock_list -- Dependency Graph Edges

Each edge in the dependency graph is represented by a `lock_list` entry,
stored in the `locks_after` and `locks_before` lists of `lock_class`.

```c
/* include/linux/lockdep.h:48-64 */
struct lock_list {
    struct list_head            entry;
    struct lock_class           *class;
    struct lock_class           *links_to;
    const struct lock_trace     *trace;
    u16                         distance;
    u8                          dep;     /* dependency type bitmap */
    u8                          only_xr; /* used by BFS */
    struct lock_list            *parent; /* used by BFS */
};
```

The `dep` field encodes the type of dependency using four bits representing
the combinations of exclusive/shared and non-recursive/recursive (EN, ER,
SN, SR).

### lock_chain -- Chain Caching

To avoid re-validating the same sequence of locks, lockdep caches validated
lock chains using a 64-bit hash.

```c
/* include/linux/lockdep.h:75-83 */
struct lock_chain {
    unsigned int                irq_context :  2,
                                depth       :  6,
                                base        : 24;
    struct hlist_node           entry;
    u64                         chain_key;
};
```

---

## Lock Class Registration

When a lock is first acquired, lockdep must register its class if not already
known. The `register_lock_class()` function at `kernel/locking/lockdep.c:1281`
handles this.

### Registration Flow

```
lock_acquire()
  -> __lock_acquire()                   (lockdep.c:5073)
       -> class = lock->class_cache[]   (fast path: cached)
       -> register_lock_class()         (slow path: lockdep.c:1281)
            -> look_up_lock_class()     (hash table lookup)
            -> assign_lock_key()        (if no key set)
            -> allocate from free_lock_classes list
            -> hash and insert into classhash_table[]
            -> move to all_lock_classes list
```

The implementation first checks the class cache on the `lockdep_map` for a
fast path hit. If the class is not cached, it looks up the hash table. If
still not found, it allocates a new `lock_class` from the free list:

```c
/* kernel/locking/lockdep.c:1280-1390 (abbreviated) */
static struct lock_class *
register_lock_class(struct lockdep_map *lock, unsigned int subclass, int force)
{
    struct lockdep_subclass_key *key;
    struct hlist_head *hash_head;
    struct lock_class *class;

    class = look_up_lock_class(lock, subclass);
    if (likely(class))
        goto out_set_class_cache;

    if (!lock->key) {
        if (!assign_lock_key(lock))
            return NULL;
    } else if (!static_obj(lock->key) && !is_dynamic_key(lock->key)) {
        return NULL;
    }

    key = lock->key->subkeys + subclass;
    hash_head = classhashentry(key);

    if (!graph_lock())
        return NULL;

    /* ... double-check under lock ... */

    /* Allocate a new lock class and add it to the hash */
    class = list_first_entry_or_null(&free_lock_classes, typeof(*class),
                                     lock_entry);
    if (!class) {
        print_lockdep_off("BUG: MAX_LOCKDEP_KEYS too low!");
        return NULL;
    }
    nr_lock_classes++;
    class->key = key;
    class->name = lock->name;
    class->subclass = subclass;
    /* ... */
    hlist_add_head_rcu(&class->hash_entry, hash_head);
    list_move_tail(&class->lock_entry, &all_lock_classes);

    /* ... */
out_set_class_cache:
    if (!subclass || force)
        lock->class_cache[0] = class;
    else if (subclass < NR_LOCKDEP_CACHING_CLASSES)
        lock->class_cache[subclass] = class;

    return class;
}
```

### Key Types

Lock class keys can be either static or dynamic:

- **Static keys**: The key is a static variable whose address is used as the
  identifier. `static_obj()` checks whether the key resides in the kernel's
  static data sections (`.data`, `.rodata`, `.bss`, etc.).

- **Dynamic keys**: Must be registered with `lockdep_register_key()` before
  use and unregistered with `lockdep_unregister_key()` before the key memory
  is freed (`kernel/locking/lockdep.c:1218`).

---

## Dependency Graph Construction

Every time a task acquires a new lock while already holding other locks,
lockdep records the ordering dependency. If task T holds lock A and then
acquires lock B, lockdep adds the directed edge A -> B to the dependency
graph (meaning "A was held when B was acquired").

### The validate_chain() Entry Point

When `__lock_acquire()` processes a new lock acquisition, it calls
`validate_chain()` (`kernel/locking/lockdep.c:3857`):

```c
/* kernel/locking/lockdep.c:3857-3916 (abbreviated) */
static int validate_chain(struct task_struct *curr,
                          struct held_lock *hlock,
                          int chain_head, u64 chain_key)
{
    /*
     * Trylock does not add new dependencies, because trylock
     * can be done in any order.
     */
    if (!hlock->trylock && hlock->check &&
        lookup_chain_cache_add(curr, hlock, chain_key)) {
        /*
         * New chain -- perform full validation:
         * 1. check_deadlock() -- same class already held?
         * 2. check_prevs_add() -- add dependency to all prev locks
         */
        int ret = check_deadlock(curr, hlock);
        if (!ret)
            return 0;

        if (!chain_head && ret != 2) {
            if (!check_prevs_add(curr, hlock))
                return 0;
        }
        graph_unlock();
    }
    return 1;
}
```

### Chain Key Hashing

The chain key is a 64-bit rolling hash computed incrementally as locks are
acquired. It uniquely identifies the exact sequence of lock classes held by
the current task:

```c
/* kernel/locking/lockdep.c:444-451 */
static inline u64 iterate_chain_key(u64 key, u32 idx)
{
    u32 k0 = key, k1 = key >> 32;
    __jhash_mix(idx, k0, k1);   /* Jenkins hash mixing */
    return k0 | (u64)k1 << 32;
}
```

When a new chain key matches one in the chain hash table, the entire
validation is skipped (chain cache hit). This is the primary optimization
that makes lockdep practical: most lock acquisitions match a previously
validated chain and cost only a hash lookup.

### Adding Dependencies via check_prev_add()

For each previously held lock (the "prev"), `check_prev_add()` at
`kernel/locking/lockdep.c:3118` performs three validations before adding a
new prev -> next edge:

```c
/* kernel/locking/lockdep.c:3118-3244 (abbreviated) */
static int
check_prev_add(struct task_struct *curr, struct held_lock *prev,
               struct held_lock *next, u16 distance,
               struct lock_trace **const trace)
{
    /* 1. Prove that next -> ... -> prev path does NOT exist
     *    (would create a cycle = deadlock) */
    ret = check_noncircular(next, prev, trace);
    if (unlikely(bfs_error(ret) || ret == BFS_RMATCH))
        return 0;

    /* 2. Check IRQ safety: no hardirq-safe -> hardirq-unsafe path */
    if (!check_irq_usage(curr, prev, next))
        return 0;

    /* 3. Check if dependency already exists */
    list_for_each_entry(entry, &hlock_class(prev)->locks_after, entry) {
        if (entry->class == hlock_class(next)) {
            /* Already present, update dep type */
            entry->dep |= calc_dep(prev, next);
            return 1;
        }
    }

    /* 4. Check if dependency is redundant */
    ret = check_redundant(prev, next);
    if (ret == BFS_RMATCH)
        return 2;  /* redundant, skip adding */

    /* 5. Add forward edge: prev->locks_after += next */
    add_lock_to_list(hlock_class(next), hlock_class(prev),
                     &hlock_class(prev)->locks_after, ...);

    /* 6. Add backward edge: next->locks_before += prev */
    add_lock_to_list(hlock_class(prev), hlock_class(next),
                     &hlock_class(next)->locks_before, ...);

    return 2;
}
```

### Dependency Types

Each edge carries a dependency type bitmap encoding how the locks were
acquired (exclusive vs. shared, non-recursive vs. recursive):

```
    -(EN)->   Exclusive lock, Non-recursive (strongest)
    -(ER)->   Exclusive lock, Recursive
    -(SN)->   Shared lock, Non-recursive
    -(SR)->   Shared lock, Recursive (weakest)
```

Only "strong" dependency paths can cause deadlocks. A strong path is one
where no two adjacent edges form a `-(*R)-> -(S*)->` pattern, because a
recursive reader and a shared acquirer cannot actually block each other.

---

## Deadlock Detection Algorithm

Lockdep detects deadlocks by proving that no cycle exists in the dependency
graph. It uses Breadth-First Search (BFS) rather than DFS to find the
shortest cycle path.

### BFS Implementation

The core BFS engine is `__bfs()` at `kernel/locking/lockdep.c:1729`. It
uses a statically allocated circular queue:

```c
/* kernel/locking/lockdep.c:1452-1470 */
#define MAX_CIRCULAR_QUEUE_SIZE   (1UL << CONFIG_LOCKDEP_CIRCULAR_QUEUE_BITS)
#define CQ_MASK                   (MAX_CIRCULAR_QUEUE_SIZE-1)

struct circular_queue {
    struct lock_list *element[MAX_CIRCULAR_QUEUE_SIZE];
    unsigned int  front, rear;
};

static struct circular_queue lock_cq;
```

The BFS traversal with strong-path filtering:

```c
/* kernel/locking/lockdep.c:1729-1835 (abbreviated) */
static enum bfs_result __bfs(struct lock_list *source_entry,
                             void *data,
                             bool (*match)(struct lock_list *entry, void *data),
                             bool (*skip)(struct lock_list *entry, void *data),
                             struct lock_list **target_entry,
                             int offset)
{
    struct circular_queue *cq = &lock_cq;

    __cq_init(cq);
    __cq_enqueue(cq, source_entry);

    while ((lock = __bfs_next(lock, offset)) ||
           (lock = __cq_dequeue(cq))) {

        if (lock_accessed(lock))
            continue;
        mark_lock_accessed(lock);

        /* Filter out weak paths: -(*R)-> -(S*)-> */
        if (lock->parent) {
            u8 dep = lock->dep;
            bool prev_only_xr = lock->parent->only_xr;

            if (prev_only_xr)
                dep &= ~(DEP_SR_MASK | DEP_SN_MASK);
            if (!dep)
                continue;
            lock->only_xr = !(dep & (DEP_SN_MASK | DEP_EN_MASK));
        }

        if (match(lock, data)) {
            *target_entry = lock;
            return BFS_RMATCH;
        }

        /* Expand: enqueue children */
        head = get_dep_list(lock, offset);
        list_for_each_entry_rcu(entry, head, entry) {
            visit_lock_entry(entry, lock);
            if (first) {
                first = false;
                if (__cq_enqueue(cq, entry))
                    return BFS_EQUEUEFULL;
            }
        }
    }
    return BFS_RNOMATCH;
}
```

### Cycle Detection via check_noncircular()

When adding edge B -> A (task holds B, acquiring A), lockdep must prove
that no path A -> ... -> B already exists in the graph. If such a path
exists, adding B -> A would create a cycle:

```c
/* kernel/locking/lockdep.c:2179-2210 */
static noinline enum bfs_result
check_noncircular(struct held_lock *src, struct held_lock *target,
                  struct lock_trace **const trace)
{
    struct lock_list src_entry;

    bfs_init_root(&src_entry, src);
    debug_atomic_inc(nr_cyclic_checks);

    /* Search forward from 'next' (src) looking for 'prev' (target) */
    ret = check_path(target, &src_entry, hlock_conflict, NULL, &target_entry);

    if (unlikely(ret == BFS_RMATCH)) {
        /* Cycle found! */
        if (src->class_idx == target->class_idx)
            print_deadlock_bug(current, src, target);
        else
            print_circular_bug(&src_entry, target_entry, src, target);
    }
    return ret;
}
```

### Self-Deadlock Detection via check_deadlock()

Before checking the graph, lockdep first checks the simpler case: is the
current task trying to acquire a lock of a class it already holds?

```c
/* kernel/locking/lockdep.c:3052-3093 */
static int
check_deadlock(struct task_struct *curr, struct held_lock *next)
{
    struct held_lock *prev;
    struct held_lock *nest = NULL;

    for (i = 0; i < curr->lockdep_depth; i++) {
        prev = curr->held_locks + i;

        if (prev->instance == next->nest_lock)
            nest = prev;

        if (hlock_class(prev) != hlock_class(next))
            continue;

        /* Allow read-after-read recursion */
        if ((next->read == 2) && prev->read)
            continue;

        /* Allow if cmp_fn says this ordering is valid */
        if (class->cmp_fn &&
            class->cmp_fn(prev->instance, next->instance) < 0)
            continue;

        /* Allow if nest_lock serializes the nesting */
        if (nest)
            return 2;

        print_deadlock_bug(curr, prev, next);
        return 0;
    }
    return 1;
}
```

### Forward and Backward BFS

The BFS can traverse in two directions:

- **`__bfs_forwards()`**: Follows `locks_after` lists -- finds what locks
  might be acquired after the source lock.
- **`__bfs_backwards()`**: Follows `locks_before` lists -- finds what locks
  were held before the source lock.

```c
/* kernel/locking/lockdep.c:1837-1858 */
static inline enum bfs_result
__bfs_forwards(struct lock_list *src_entry, ...)
{
    return __bfs(src_entry, ...,
                 offsetof(struct lock_class, locks_after));
}

static inline enum bfs_result
__bfs_backwards(struct lock_list *src_entry, ...)
{
    return __bfs(src_entry, ...,
                 offsetof(struct lock_class, locks_before));
}
```

---

## Lock Usage State Tracking

Lockdep tracks how each lock class is used with respect to interrupt contexts.
This allows it to detect a critical class of bugs: locks used in both IRQ and
non-IRQ contexts without proper IRQ disabling.

### Usage States

The states are generated from `kernel/locking/lockdep_states.h`:

```c
/* kernel/locking/lockdep_states.h */
LOCKDEP_STATE(HARDIRQ)
LOCKDEP_STATE(SOFTIRQ)
```

These expand (via `kernel/locking/lockdep_internals.h:13-24`) into:

```c
enum lock_usage_bit {
    LOCK_USED_IN_HARDIRQ,           /* Lock taken in hardirq context */
    LOCK_USED_IN_HARDIRQ_READ,      /* Lock read-acquired in hardirq */
    LOCK_ENABLED_HARDIRQ,           /* Lock taken with hardirqs enabled */
    LOCK_ENABLED_HARDIRQ_READ,      /* Lock read-acquired, hardirqs on */
    LOCK_USED_IN_SOFTIRQ,           /* Lock taken in softirq context */
    LOCK_USED_IN_SOFTIRQ_READ,      /* Lock read-acquired in softirq */
    LOCK_ENABLED_SOFTIRQ,           /* Lock taken with softirqs enabled */
    LOCK_ENABLED_SOFTIRQ_READ,      /* Lock read-acquired, softirqs on */
    LOCK_USED,                      /* Lock has been acquired at all */
    LOCK_USED_READ,                 /* Lock has been read-acquired */
    LOCK_USAGE_STATES,
};
```

Each bit is stored in `lock_class::usage_mask` along with a saved stack
trace in `usage_traces[]` for diagnostic output.

### The Usage Character Display

Lockdep uses a compact notation for usage state, displayed as a string like
`{....}` or `{+.+.}` where each position represents a hardirq/softirq state:

```c
/* kernel/locking/lockdep.c:675-698 */
static char get_usage_char(struct lock_class *class, enum lock_usage_bit bit)
{
    char c = '.';   /* irqs disabled, not in irq context (safest) */

    if (class->usage_mask & lock_flag(bit + LOCK_USAGE_DIR_MASK)) {
        c = '+';    /* irq is enabled and not in irq context */
        if (class->usage_mask & lock_flag(bit))
            c = '?'; /* in irq context AND irq is enabled (UNSAFE!) */
    } else if (class->usage_mask & lock_flag(bit))
        c = '-';    /* in irq context and irq is disabled */

    return c;
}
```

The four characters represent: hardirq-write, hardirq-read, softirq-write,
softirq-read:

```
  '.'  = irqs disabled and not in irq context (safe)
  '-'  = in irq context and irqs disabled (safe)
  '+'  = irqs enabled and not in irq context (safe if consistent)
  '?'  = in irq context with irqs enabled (potentially unsafe)
```

### mark_usage() -- Setting Usage Bits

When a lock is acquired, `mark_usage()` (`kernel/locking/lockdep.c:4614`)
sets the appropriate usage bits based on the current execution context:

```c
/* kernel/locking/lockdep.c:4614-4674 (abbreviated) */
static int
mark_usage(struct task_struct *curr, struct held_lock *hlock, int check)
{
    if (!check)
        goto lock_used;

    /* Mark USED_IN bits based on current IRQ context */
    if (!hlock->trylock) {
        if (hlock->read) {
            if (lockdep_hardirq_context())
                mark_lock(curr, hlock, LOCK_USED_IN_HARDIRQ_READ);
            if (curr->softirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_SOFTIRQ_READ);
        } else {
            if (lockdep_hardirq_context())
                mark_lock(curr, hlock, LOCK_USED_IN_HARDIRQ);
            if (curr->softirq_context)
                mark_lock(curr, hlock, LOCK_USED_IN_SOFTIRQ);
        }
    }

    /* Mark ENABLED bits based on IRQ enable state */
    if (!hlock->hardirqs_off && !hlock->sync) {
        if (hlock->read) {
            mark_lock(curr, hlock, LOCK_ENABLED_HARDIRQ_READ);
            if (curr->softirqs_enabled)
                mark_lock(curr, hlock, LOCK_ENABLED_SOFTIRQ_READ);
        } else {
            mark_lock(curr, hlock, LOCK_ENABLED_HARDIRQ);
            if (curr->softirqs_enabled)
                mark_lock(curr, hlock, LOCK_ENABLED_SOFTIRQ);
        }
    }

lock_used:
    mark_lock(curr, hlock, LOCK_USED);
    return 1;
}
```

### IRQ Safety Checking via check_irq_usage()

When adding a new dependency prev -> next, lockdep checks whether this
creates an IRQ-safety inversion. The algorithm
(`kernel/locking/lockdep.c:2811-2895`) works in four steps:

```
Step 1: Walk backwards from <prev>, accumulating USED_IN_IRQ usage.
Step 2: Compute the exclusive mask; walk forwards from <next> looking
        for ENABLED_IRQ usage that conflicts.
Step 3: If a conflict is found, narrow down to the specific lock that
        causes the problem by walking backwards again.
Step 4: Report the pair of incompatible usage bits.
```

For example, if lock A is ever used in hardirq context (USED_IN_HARDIRQ) and
lock B is ever acquired with hardirqs enabled (ENABLED_HARDIRQ), then a
dependency path A -> ... -> B would be an IRQ-safety violation: an interrupt
could try to acquire A while B is held, but B requires hardirqs to be
enabled, meaning A's hardirq handler could deadlock.

---

## Lockdep Annotations

Lockdep provides various annotations that subsystems use to give lockdep
additional information about their locking semantics.

### lockdep_set_class() -- Reclassifying Locks

When multiple lock instances share the same static key but represent
logically different lock classes, use `lockdep_set_class()` to assign a
new class key:

```c
/* include/linux/lockdep.h:157-161 */
#define lockdep_set_class(lock, key)                            \
    lockdep_init_map_type(&(lock)->dep_map, #key, key, 0,       \
                          (lock)->dep_map.wait_type_inner,       \
                          (lock)->dep_map.wait_type_outer,       \
                          (lock)->dep_map.lock_type)
```

Example usage: filesystem code might define separate lock classes for inode
locks vs. dentry locks even though both are mutexes.

### lockdep_set_class_and_subclass() -- Nesting Support

For locks that are legitimately nested (e.g., locking parent and child
directory inodes), subclasses tell lockdep the expected nesting order:

```c
/* include/linux/lockdep.h:169-173 */
#define lockdep_set_class_and_subclass(lock, key, sub)          \
    lockdep_init_map_type(&(lock)->dep_map, #key, key, sub,     \
                          (lock)->dep_map.wait_type_inner,       \
                          (lock)->dep_map.wait_type_outer,       \
                          (lock)->dep_map.lock_type)
```

The `SINGLE_DEPTH_NESTING` constant (value 1) is provided for the common
case of single-level nesting:

```c
/* include/linux/lockdep.h:502 */
#define SINGLE_DEPTH_NESTING    1
```

Usage:
```c
/* Acquiring a child lock while holding the parent */
spin_lock_nested(&child->lock, SINGLE_DEPTH_NESTING);
```

### lockdep_assert_held() -- Runtime Assertions

These macros verify at runtime that a lock is actually held:

```c
/* include/linux/lockdep.h:284-300 */
#define lockdep_assert_held(l)                      \
    lockdep_assert(lockdep_is_held(l) != LOCK_STATE_NOT_HELD)

#define lockdep_assert_not_held(l)                  \
    lockdep_assert(lockdep_is_held(l) != LOCK_STATE_HELD)

#define lockdep_assert_held_write(l)                \
    lockdep_assert(lockdep_is_held_type(l, 0))

#define lockdep_assert_held_read(l)                 \
    lockdep_assert(lockdep_is_held_type(l, 1))

#define lockdep_assert_held_once(l)                 \
    lockdep_assert_once(lockdep_is_held(l) != LOCK_STATE_NOT_HELD)

#define lockdep_assert_none_held_once()             \
    lockdep_assert_once(!current->lockdep_depth)
```

### lockdep_set_novalidate_class() -- Disabling Validation

For locks where ordering validation is not meaningful (e.g., locks that are
never nested or have complex dynamic ordering):

```c
/* include/linux/lockdep.h:189-190 */
#define lockdep_set_novalidate_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_validate__, #lock)
```

### lockdep_set_notrack_class() -- Disabling Tracking Entirely

For subsystems that exceed lockdep's 48-lock depth limit (notably bcachefs):

```c
/* include/linux/lockdep.h:199-200 */
#define lockdep_set_notrack_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_track__, #lock)
```

### lock_set_cmp_fn() -- Custom Lock Ordering

For same-class locks that have a natural ordering (e.g., by inode number),
a comparison function tells lockdep which nesting order is valid:

```c
/* kernel/locking/lockdep.c:5002-5025 */
void lockdep_set_lock_cmp_fn(struct lockdep_map *lock, lock_cmp_fn cmp_fn,
                             lock_print_fn print_fn)
{
    struct lock_class *class = lock->class_cache[0];
    /* ... */
    if (class) {
        class->cmp_fn   = cmp_fn;
        class->print_fn = print_fn;
    }
}
```

When `check_deadlock()` finds two held locks of the same class, it calls
`class->cmp_fn()`: if the result is negative (first lock < second lock),
the nesting is considered valid and no warning is issued
(`kernel/locking/lockdep.c:3078-3080`).

### IRQ Context Assertions

```c
/* include/linux/lockdep.h:575-620 */
#define lockdep_assert_irqs_enabled()       /* WARN if hardirqs are off */
#define lockdep_assert_irqs_disabled()      /* WARN if hardirqs are on */
#define lockdep_assert_in_irq()             /* WARN if not in hardirq */
#define lockdep_assert_no_hardirq()         /* WARN if in hardirq or irqs off */
#define lockdep_assert_preemption_enabled()
#define lockdep_assert_preemption_disabled()
#define lockdep_assert_in_softirq()
```

### might_lock() -- Hypothetical Acquisitions

These macros perform a "dry run" acquire-then-release to let lockdep check
whether acquiring the lock would create an ordering problem:

```c
/* include/linux/lockdep.h:549-554 */
# define might_lock(lock)                                       \
do {                                                            \
    typecheck(struct lockdep_map *, &(lock)->dep_map);          \
    lock_acquire(&(lock)->dep_map, 0, 0, 0, 1, NULL, _THIS_IP_); \
    lock_release(&(lock)->dep_map, _THIS_IP_);                  \
} while (0)
```

---

## Lock Statistics

When `CONFIG_LOCK_STAT` is enabled, lockdep collects detailed contention
and timing statistics for every lock class. Statistics are stored per-CPU
to minimize cache bouncing.

### Per-CPU Statistics Storage

```c
/* kernel/locking/lockdep.c:244 */
static DEFINE_PER_CPU(struct lock_class_stats[MAX_LOCKDEP_KEYS], cpu_lock_stats);
```

### lock_class_stats Structure

```c
/* include/linux/lockdep_types.h:168-176 */
struct lock_class_stats {
    unsigned long   contention_point[LOCKSTAT_POINTS];   /* where contention happened */
    unsigned long   contending_point[LOCKSTAT_POINTS];   /* who held the lock */
    struct lock_time    read_waittime;       /* time waiting for read lock */
    struct lock_time    write_waittime;      /* time waiting for write lock */
    struct lock_time    read_holdtime;       /* time holding read lock */
    struct lock_time    write_holdtime;      /* time holding write lock */
    unsigned long   bounces[nr_bounce_types]; /* cross-CPU lock migrations */
};
```

The `lock_time` structure tracks min, max, total, and count:

```c
/* include/linux/lockdep_types.h:150-155 */
struct lock_time {
    s64             min;
    s64             max;
    s64             total;
    unsigned long   nr;
};
```

### Contention Tracking

When a lock acquisition blocks (contention), the `LOCK_CONTENDED` macro
records it:

```c
/* include/linux/lockdep.h:441-448 */
#define LOCK_CONTENDED(_lock, try, lock)                \
do {                                                    \
    if (!try(_lock)) {                                  \
        lock_contended(&(_lock)->dep_map, _RET_IP_);    \
        lock(_lock);                                    \
    }                                                   \
    lock_acquired(&(_lock)->dep_map, _RET_IP_);         \
} while (0)
```

The internal `__lock_contended()` function (`kernel/locking/lockdep.c:6017`)
records the contention point (where the waiter is) and the contending point
(where the current holder acquired the lock), timestamps the wait start, and
tracks cross-CPU bounces.

The `__lock_acquired()` function (`kernel/locking/lockdep.c:6061`) records
wait times and hold time start.

### Accessing Lock Statistics

Statistics are exposed through `/proc/lock_stat`:

```
Format:
  class name:   contention points   wait time   hold time   bounces

Controllable via sysctl:
  /proc/sys/kernel/lock_stat     (0 = off, 1 = on)
```

The `lock_stat` module parameter at `kernel/locking/lockdep.c:76` controls
whether statistics collection is active:

```c
/* kernel/locking/lockdep.c:74-76 */
#ifdef CONFIG_LOCK_STAT
static int lock_stat = 1;
module_param(lock_stat, int, 0644);
#endif
```

---

## Common Lockdep Warnings

### 1. Circular Dependency Detected

```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
task/PID is trying to acquire lock:
 <lock B info>

but task is already holding lock:
 <lock A info>

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:
-> #1 (<lock B>): ...
-> #0 (<lock A>): ...

 Possible unsafe locking scenario:

       CPU0                    CPU1
       ----                    ----
  lock(A);
                               lock(B);
                               lock(A);
  lock(B);

 *** DEADLOCK ***
```

**Meaning**: Lockdep found a path B -> ... -> A in the dependency graph, and
now you are trying to acquire B while holding A. This would create a cycle
A -> B -> ... -> A. Two CPUs following different orderings would deadlock.

**Output from**: `print_circular_bug()` at `kernel/locking/lockdep.c:2036`.

### 2. Inconsistent Lock State

```
================================
WARNING: inconsistent lock state
--------------------------------
inconsistent {IN-HARDIRQ-W} -> {HARDIRQ-ON-W} usage.
task/PID takes:
 <lock info>

{IN-HARDIRQ-W} state was registered at:
 <stack trace>
```

**Meaning**: A lock was previously used inside a hardirq handler (making it
"hardirq-safe"), but is now being acquired with hardirqs enabled (making it
"hardirq-unsafe"). An interrupt could fire while this lock is held, and the
interrupt handler would try to acquire the same lock, causing a deadlock.

**Output from**: `print_usage_bug()` at `kernel/locking/lockdep.c:4003`.

### 3. Possible IRQ Lock Inversion

```
=========================================================
WARNING: possible irq lock inversion dependency detected
---------------------------------------------------------
task/PID just changed the state of lock:
 <lock B info>

but this lock took another, HARDIRQ-unsafe lock in the past:
 <lock A info>

and interrupts could create inverse lock ordering between them.
```

**Meaning**: Lock B is hardirq-safe (taken in hardirq context), and lock A
is hardirq-unsafe (taken with hardirqs enabled). A dependency path exists
between them, meaning an interrupt could create a deadlock through lock
inversion.

**Output from**: `print_bad_irq_dependency()`.

### 4. Bad Unlock Balance

```
=====================================
WARNING: bad unlock balance detected!
-------------------------------------
task/PID is trying to release lock (lockname) at:
 <address>
but there are no more locks to release!
```

**Meaning**: A `lock_release()` was called but the lock is not on the current
task's held_locks stack. This usually indicates a mismatched lock/unlock pair.

**Output from**: `print_unlock_imbalance_bug()` at
`kernel/locking/lockdep.c:5261`.

### 5. Held Lock at Task Exit

```
================================================
WARNING: lock held when returning to user space!
------------------------------------------------
```

**Meaning**: A task returned to user space (or exited) while still holding
a lock. This is detected by `lockdep_sys_exit()`.

### 6. BUG Messages (Resource Exhaustion)

```
BUG: MAX_LOCKDEP_KEYS too low!
BUG: MAX_LOCKDEP_ENTRIES too low!
BUG: MAX_LOCKDEP_CHAINS too low!
BUG: MAX_STACK_TRACE_ENTRIES too low!
BUG: MAX_LOCK_DEPTH too low!
```

**Meaning**: Lockdep's statically allocated arrays have been exhausted.
Lockdep turns itself off when this happens. Increase the corresponding
`CONFIG_LOCKDEP_*` Kconfig values.

### 7. Self-Deadlock (AA Deadlock)

```
=============================================
WARNING: possible recursive locking detected
---------------------------------------------
task/PID is trying to acquire lock:
 <lock A info>

but task is already holding lock:
 <lock A (same class) info>
```

**Meaning**: The task is trying to acquire a lock of the same class it already
holds (without proper nesting annotations). This is a direct self-deadlock.

**Output from**: `print_deadlock_bug()`.

---

## Performance Overhead and Configuration

### Kconfig Options

Lockdep is controlled by several Kconfig options defined in
`lib/Kconfig.debug`:

| Option | Default | Description |
|--------|---------|-------------|
| `CONFIG_PROVE_LOCKING` | n | Main switch: enables full deadlock detection |
| `CONFIG_LOCK_STAT` | n | Enables contention statistics |
| `CONFIG_DEBUG_LOCKDEP` | n | Extra self-consistency checks for lockdep |
| `CONFIG_LOCKDEP_BITS` | 15 | log2(MAX_LOCKDEP_ENTRIES), default 32768 |
| `CONFIG_LOCKDEP_CHAINS_BITS` | 16 | log2(MAX_LOCKDEP_CHAINS), default 65536 |
| `CONFIG_LOCKDEP_STACK_TRACE_BITS` | 19 | log2(MAX_STACK_TRACE_ENTRIES) |
| `CONFIG_LOCKDEP_STACK_TRACE_HASH_BITS` | 14 | log2(STACK_TRACE_HASH_SIZE) |
| `CONFIG_LOCKDEP_CIRCULAR_QUEUE_BITS` | 12 | BFS queue size, default 4096 |
| `CONFIG_MAX_LOCKDEP_SUBCLASSES` | 8 | Max nesting subclasses per key |

`CONFIG_PROVE_LOCKING` selects `CONFIG_LOCKDEP`, `CONFIG_DEBUG_LOCK_ALLOC`,
and `CONFIG_TRACE_IRQFLAGS`. The Kconfig help text
(`lib/Kconfig.debug:1366-1385`) emphasizes that the proof is independent of
timing and the number of CPUs: if a deadlock is theoretically possible based
on observed locking patterns, it will be reported.

### Static Memory Consumption

Lockdep allocates several large static arrays at compile time:

```c
/* Key structures and their default sizes */
struct lock_class   lock_classes[MAX_LOCKDEP_KEYS];      /* 8192 entries */
struct lock_list    list_entries[MAX_LOCKDEP_ENTRIES];    /* 32768 entries */
struct lock_chain   lock_chains[MAX_LOCKDEP_CHAINS];     /* 65536 entries */
u16                 chain_hlocks[MAX_LOCKDEP_CHAIN_HLOCKS]; /* 5 * chains */
unsigned long       stack_trace[MAX_STACK_TRACE_ENTRIES]; /* 524288 entries */

/* Plus per-CPU stats when LOCK_STAT is on */
DEFINE_PER_CPU(struct lock_class_stats[MAX_LOCKDEP_KEYS], cpu_lock_stats);
```

For architectures with tight memory constraints (Sparc), `CONFIG_LOCKDEP_SMALL`
reduces these limits (`kernel/locking/lockdep_internals.h:87-100`).

### Runtime Overhead

The primary overhead mechanisms:

1. **Every lock acquisition** calls `lock_acquire()` which disables IRQs,
   increments the recursion counter, and calls `__lock_acquire()`.

2. **Chain cache hits** (the common case after warm-up): only a hash
   computation and hash table lookup are needed. The chain key is computed
   incrementally via `iterate_chain_key()`.

3. **Chain cache misses** trigger the expensive O(N^2) `check_prevs_add()`
   path, which runs BFS on the dependency graph.

4. **Stack trace capture** via `save_trace()` on each new dependency edge,
   with deduplication through a hash table.

5. **Usage marking** on each acquisition checks and potentially updates
   the `usage_mask` bitmap.

### Sysctl Interface

Two sysctl parameters control lockdep at runtime
(`kernel/locking/lockdep.c:82-108`):

```
/proc/sys/kernel/prove_locking   (0 = off, 1 = on)
/proc/sys/kernel/lock_stat       (0 = off, 1 = on)
```

### Lockdep Self-Protection

Lockdep protects itself from recursion using multiple mechanisms:

- **Per-CPU recursion counter**: `lockdep_recursion` is incremented before
  any lockdep operation and checked at entry.
- **Per-task recursion field**: `current->lockdep_recursion` provides
  coarser-grained off/on control via `lockdep_off()`/`lockdep_on()`.
- **Internal arch_spinlock_t**: The lockdep graph lock at
  `kernel/locking/lockdep.c:136` uses a raw architecture lock to avoid
  recursing into the lockdep code.
- **debug_locks global**: Once lockdep encounters a fatal error, it sets
  `debug_locks = 0` and disables itself permanently for the remainder of
  the boot.

### Proc Interfaces

Lockdep exposes its state via `/proc`:

- `/proc/lockdep` -- Lists all registered lock classes with usage state
- `/proc/lockdep_chains` -- Lists all validated lock chains
- `/proc/lockdep_stats` -- Shows internal statistics (chain hits/misses,
  dependency counts, etc.)
- `/proc/lock_stat` -- Lock contention statistics (requires CONFIG_LOCK_STAT)

### Best Practices

- Always run development and testing kernels with `CONFIG_PROVE_LOCKING=y`.
- Lockdep reports the first potential deadlock and then typically disables
  itself. Fix the reported issue and re-test to find additional problems.
- Use `lockdep_set_class()` and subclasses to eliminate false positives
  when your subsystem has legitimate same-class nesting.
- Use `lockdep_assert_held()` liberally in functions that require a lock to
  be held -- this serves as executable documentation and catches callers that
  forget to take the lock.
- Do not use `lockdep_set_novalidate_class()` as a shortcut to silence
  warnings without understanding the root cause.
