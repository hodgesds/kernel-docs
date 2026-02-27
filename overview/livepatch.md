# Linux Kernel Live Patching (livepatch) Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Data Structures](#core-data-structures)
4. [Patch Lifecycle](#patch-lifecycle)
5. [Consistency Model](#consistency-model)
6. [ftrace-Based Function Redirection](#ftrace-based-function-redirection)
7. [Shadow Variables](#shadow-variables)
8. [Patch Stacking and Atomic Replace](#patch-stacking-and-atomic-replace)
9. [Integration with Modules, ftrace, and kallsyms](#integration-with-modules-ftrace-and-kallsyms)
10. [Key Code Paths](#key-code-paths)
11. [Limitations and Constraints](#limitations-and-constraints)

---

## Introduction

Linux kernel live patching (livepatch) enables runtime replacement of kernel
functions without rebooting the system. It allows critical security fixes and
bug patches to be applied to a running kernel, maintaining system uptime for
production servers and infrastructure.

Key capabilities:

- Runtime function-level replacement in vmlinux and loadable modules
- Per-task consistency model ensuring safe transitions
- Patch stacking for incremental fixes
- Atomic replace mode for cumulative patches
- Shadow variables for extending existing data structures
- Integration with ftrace for transparent function redirection

### History and Dependencies

```
Livepatch Evolution
--------------------
  2014  - Initial livepatch infrastructure (Seth Jennings, SUSE)
          Merged kGraft (SUSE) and kpatch (Red Hat) approaches
  2015  - Consistency model added (Josh Poimboeuf)
  2017  - Shadow variables API
  2019  - Atomic replace ("replace") mode
  2019  - System state tracking (klp_state)

Kernel Configuration Dependencies (kernel/livepatch/Kconfig):
  CONFIG_LIVEPATCH depends on:
    - DYNAMIC_FTRACE_WITH_REGS || DYNAMIC_FTRACE_WITH_ARGS
    - MODULES
    - SYSFS
    - KALLSYMS_ALL
    - HAVE_LIVEPATCH (arch support)
    - !TRIM_UNUSED_KSYMS
```

---

## Architecture Overview

Livepatch intercepts calls to original kernel functions and redirects them to
replacement functions provided by a patch module. The mechanism uses ftrace
to hook into function prologues, and a per-task consistency model to ensure
all tasks transition safely from old code to new code.

```
+------------------------------------------------------------------+
|                   Livepatch Architecture                          |
+------------------------------------------------------------------+
|                                                                   |
|  Patch Module (.ko)          Kernel (vmlinux / modules)           |
|  +-----------------------+   +------------------------------+     |
|  | klp_patch             |   |                              |     |
|  |  +- klp_object[]      |   |  original_func:              |     |
|  |  |   +- klp_func[]    |   |    call __fentry__  -------+  |     |
|  |  |   |  .old_name     |   |    <original code>     |   |  |     |
|  |  |   |  .new_func ----+---+->  patched_func()      |   |  |     |
|  |  |   +- klp_func[]    |   |                        |   |  |     |
|  |  +- klp_object[]      |   +------------------------------+     |
|  +-----------------------+                            |            |
|                                                       |            |
|  ftrace layer                                         |            |
|  +------------------------------------------------+   |            |
|  | klp_ops                                        |   |            |
|  |   .fops -> klp_ftrace_handler()               <----+            |
|  |   .func_stack -> [newest_func | older_func]    |                |
|  +------------------------------------------------+               |
|                                                                   |
|  Per-task transition                                               |
|  +------------------------------------------------+               |
|  | task_struct.patch_state                         |               |
|  |   KLP_TRANSITION_IDLE      = -1                 |               |
|  |   KLP_TRANSITION_UNPATCHED =  0                 |               |
|  |   KLP_TRANSITION_PATCHED   =  1                 |               |
|  | TIF_PATCH_PENDING thread flag                   |               |
|  +------------------------------------------------+               |
|                                                                   |
+------------------------------------------------------------------+
```

### Source File Organization

| File | Purpose |
|------|---------|
| `include/linux/livepatch.h` | Public API, core struct definitions |
| `include/linux/livepatch_sched.h` | Scheduler integration hooks |
| `kernel/livepatch/core.c` | Patch initialization, sysfs, module hooks |
| `kernel/livepatch/core.h` | Internal core helpers, callback wrappers |
| `kernel/livepatch/patch.c` | ftrace handler, function patching/unpatching |
| `kernel/livepatch/patch.h` | `klp_ops` struct, patch function prototypes |
| `kernel/livepatch/transition.c` | Per-task transition, stack checking |
| `kernel/livepatch/transition.h` | Transition function prototypes |
| `kernel/livepatch/shadow.c` | Shadow variable implementation |
| `kernel/livepatch/state.c` | System state tracking and compatibility |
| `kernel/livepatch/state.h` | State compatibility check prototype |

---

## Core Data Structures

The livepatch subsystem uses a hierarchy of structures: a `klp_patch` contains
one or more `klp_object` entries, each of which contains one or more `klp_func`
entries. Optionally, a patch may declare `klp_state` entries for system state
tracking.

### klp_patch - Top-level Patch Container

Defined in `include/linux/livepatch.h:158`:

```c
struct klp_patch {
    /* external - set by the patch module author */
    struct module *mod;          /* reference to the live patch module */
    struct klp_object *objs;     /* object entries for kernel objects to be patched */
    struct klp_state *states;    /* system states that can get modified */
    bool replace;                /* replace all actively used patches */

    /* internal - managed by livepatch core */
    struct list_head list;       /* node for global list of actively used patches */
    struct kobject kobj;         /* kobject for sysfs resources */
    struct list_head obj_list;   /* dynamic list of the object entries */
    bool enabled;                /* the patch is enabled (but operation may be incomplete) */
    bool forced;                 /* was involved in a forced transition */
    struct work_struct free_work;/* patch cleanup from workqueue-context */
    struct completion finish;    /* for waiting till it is safe to remove the patch module */
};
```

The global list of active patches is maintained at `core.c:45`:

```c
LIST_HEAD(klp_patches);
```

### klp_object - Kernel Object to Patch

Represents vmlinux or a loadable kernel module containing functions to be
patched. Defined in `include/linux/livepatch.h:117`:

```c
struct klp_object {
    /* external */
    const char *name;             /* module name (or NULL for vmlinux) */
    struct klp_func *funcs;       /* function entries for functions to be patched */
    struct klp_callbacks callbacks;/* pre/post (un)patching callbacks */

    /* internal */
    struct kobject kobj;
    struct list_head func_list;   /* dynamic list of function entries */
    struct list_head node;        /* list node for klp_patch obj_list */
    struct module *mod;           /* kernel module (NULL for vmlinux) */
    bool dynamic;                 /* temporary object for nop functions */
    bool patched;                 /* the object's funcs have been added to klp_ops */
};
```

When `name` is NULL, the object refers to vmlinux. This is checked by
`klp_is_module()` at `core.c:49`:

```c
static bool klp_is_module(struct klp_object *obj)
{
    return obj->name;
}
```

### klp_func - Individual Function Replacement

Defined in `include/linux/livepatch.h:56`:

```c
struct klp_func {
    /* external */
    const char *old_name;        /* name of the function to be patched */
    void *new_func;              /* pointer to the patched function code */
    unsigned long old_sympos;    /* symbol position hint for duplicates (0 = unique) */

    /* internal */
    void *old_func;              /* pointer to the function being patched */
    struct kobject kobj;
    struct list_head node;       /* list node for klp_object func_list */
    struct list_head stack_node; /* list node for klp_ops func_stack */
    unsigned long old_size, new_size;
    bool nop;                    /* temporary patch to restore original code */
    bool patched;                /* func has been added to the klp_ops list */
    bool transition;             /* func is currently being applied or reverted */
};
```

The patching state machine for each function is documented in the header
at `include/linux/livepatch.h:42-54`:

```
When patching:
  patched=0 transition=0: unpatched
  patched=0 transition=1: unpatched, temporary starting state
  patched=1 transition=1: patched, may be visible to some tasks
  patched=1 transition=0: patched, visible to all tasks

When unpatching (reverse order):
  patched=1 transition=0: patched, visible to all tasks
  patched=1 transition=1: patched, may be visible to some tasks
  patched=0 transition=1: unpatched, temporary ending state
  patched=0 transition=0: unpatched
```

### klp_callbacks - Pre/Post Patch Hooks

Defined in `include/linux/livepatch.h:96`:

```c
struct klp_callbacks {
    int (*pre_patch)(struct klp_object *obj);
    void (*post_patch)(struct klp_object *obj);
    void (*pre_unpatch)(struct klp_object *obj);
    void (*post_unpatch)(struct klp_object *obj);
    bool post_unpatch_enabled;
};
```

All callbacks are optional. Only `pre_patch` can return an error to abort
patching. The `post_unpatch_enabled` flag is set internally by
`klp_pre_patch_callback()` in `core.h:26-35` and gates whether
`post_unpatch` runs.

### klp_state - System State Tracking

Defined in `include/linux/livepatch.h:138`:

```c
struct klp_state {
    unsigned long id;     /* system state identifier (non-zero) */
    unsigned int version; /* version of the change */
    void *data;           /* custom data */
};
```

Used for cumulative patches to track what system modifications have been
applied. The state API is implemented in `state.c` with two key functions:

- `klp_get_state()` (`state.c:31`) - check if a patch modifies a given state
- `klp_get_prev_state()` (`state.c:64`) - find state from previously installed patches

### klp_ops - ftrace Operations Tracking

Defined internally in `kernel/livepatch/patch.h:22`:

```c
struct klp_ops {
    struct list_head node;        /* node for the global klp_ops list */
    struct list_head func_stack;  /* stack of klp_func's (active func on top) */
    struct ftrace_ops fops;       /* registered ftrace ops struct */
};
```

A single `ftrace_ops` is shared between all enabled replacement functions
that target the same original function. The `func_stack` allows instantaneous
switching between function versions.

### Iteration Macros

The header provides macros for iterating over patch hierarchies
(`include/linux/livepatch.h:175-193`):

```c
#define klp_for_each_object_static(patch, obj)   /* iterate static obj array */
#define klp_for_each_object(patch, obj)           /* iterate dynamic obj_list */
#define klp_for_each_object_safe(patch, obj, tmp) /* safe iteration for removal */
#define klp_for_each_func_static(obj, func)       /* iterate static func array */
#define klp_for_each_func(obj, func)              /* iterate dynamic func_list */
#define klp_for_each_func_safe(obj, func, tmp)    /* safe iteration for removal */
```

---

## Patch Lifecycle

### Registration and Enabling

The entry point for all livepatches is `klp_enable_patch()`, defined at
`core.c:1077`. It is called from the patch module's `module_init()` callback.

```
klp_enable_patch(patch)                           [core.c:1077]
  |
  +-- Validate patch, objs, funcs are non-NULL
  +-- Verify module is marked as livepatch (MODULE_INFO(livepatch, "Y"))
  +-- Check klp_initialized() (sysfs root exists)
  +-- Check klp_have_reliable_stack()
  |
  +-- mutex_lock(&klp_mutex)
  |
  +-- klp_is_patch_compatible(patch)              [state.c:106]
  |     Verify state compatibility with existing patches
  |
  +-- try_module_get(patch->mod)                  Take module reference
  |
  +-- klp_init_patch_early(patch)                 [core.c:929]
  |     Initialize lists, kobjects, work, completion
  |     Walk static arrays, call klp_init_object_early/klp_init_func_early
  |
  +-- klp_init_patch(patch)                       [core.c:951]
  |     kobject_add to sysfs under /sys/kernel/livepatch/<name>
  |     If patch->replace: klp_add_nops(patch)    [core.c:584]
  |     For each object: klp_init_object()        [core.c:883]
  |       Find module via klp_find_object_module() [core.c:55]
  |       For each func: klp_init_func()           [core.c:769]
  |       If object loaded: klp_init_object_loaded() [core.c:835]
  |         Apply relocations, resolve symbols via kallsyms
  |     Add patch to klp_patches list
  |
  +-- __klp_enable_patch(patch)                   [core.c:1009]
  |     klp_init_transition(patch, KLP_TRANSITION_PATCHED)
  |     smp_wmb()
  |     For each loaded object:
  |       klp_pre_patch_callback(obj)
  |       klp_patch_object(obj) -> klp_patch_func() for each func
  |     klp_start_transition()
  |     patch->enabled = true
  |     klp_try_complete_transition()
  |
  +-- mutex_unlock(&klp_mutex)
```

### Disabling

Patches are disabled via the sysfs `enabled` attribute or when replaced by
a cumulative patch. The internal path is `__klp_disable_patch()` at
`core.c:977`:

```
__klp_disable_patch(patch)                        [core.c:977]
  |
  +-- Verify patch->enabled == true
  +-- Verify no transition in progress (klp_transition_patch == NULL)
  |
  +-- klp_init_transition(patch, KLP_TRANSITION_UNPATCHED)
  +-- klp_pre_unpatch_callback(obj) for each patched object
  +-- smp_wmb()
  +-- klp_start_transition()
  +-- patch->enabled = false
  +-- klp_try_complete_transition()
```

### Transition Completion and Cleanup

After a transition completes, cleanup differs based on the operation:

- **Disable**: `klp_free_patch_async(patch)` removes the patch from the
  global list and schedules sysfs/kobject cleanup via a workqueue
  (`core.c:752-756`)
- **Replace**: `klp_free_replaced_patches_async(new_patch)` frees all
  patches older than the new one (`core.c:758-767`)
- **Forced transitions**: The `forced` flag prevents `module_put()` on
  the patch module, since forced transitions may leave stale function
  references (`core.c:735-736`)

### Sysfs Interface

The livepatch sysfs hierarchy is created at `core.c:341-353`:

```
/sys/kernel/livepatch/
/sys/kernel/livepatch/<patch>/
/sys/kernel/livepatch/<patch>/enabled       (rw - enable/disable/reverse)
/sys/kernel/livepatch/<patch>/transition    (ro - transition in progress?)
/sys/kernel/livepatch/<patch>/force         (wo - force complete transition)
/sys/kernel/livepatch/<patch>/replace       (ro - is this a replace patch?)
/sys/kernel/livepatch/<patch>/<object>/
/sys/kernel/livepatch/<patch>/<object>/patched
/sys/kernel/livepatch/<patch>/<object>/<function,sympos>
```

Writing `0` to `enabled` disables a patch. Writing `0` during a patching
transition reverses it (`enabled_store()` at `core.c:356`). Writing `1`
to `force` forces all remaining tasks to the target state
(`force_store()` at `core.c:417`).

---

## Consistency Model

The livepatch consistency model ensures that every task sees a consistent
set of function implementations -- either all old or all new. Tasks are
never allowed to execute a mix of old and new versions of patched functions.

### Per-Task Transition

Each task in the system has a `patch_state` field in its `task_struct` and
a `TIF_PATCH_PENDING` thread flag. The three states are defined in
`include/linux/livepatch.h:21-23`:

```c
#define KLP_TRANSITION_IDLE       -1
#define KLP_TRANSITION_UNPATCHED   0
#define KLP_TRANSITION_PATCHED     1
```

### Transition Flow

```
Transition Lifecycle
---------------------

1. INIT: klp_init_transition()                     [transition.c:573]
   - Set klp_target_state to desired state
   - Set all tasks to initial_state (opposite of target)
   - smp_wmb()
   - Mark all funcs as func->transition = true

2. START: klp_start_transition()                   [transition.c:530]
   - Set TIF_PATCH_PENDING on all tasks not already at target state
   - Set TIF_PATCH_PENDING on all idle tasks
   - Enable klp_cond_resched (scheduler hook)

3. MIGRATE: Tasks switch to target state via:
   a) klp_try_complete_transition()                [transition.c:451]
      - Walk all process threads, call klp_try_switch_task()
      - Walk all idle tasks on each CPU
      - If incomplete, schedule retry via delayed_work (1 second)
      - After SIGNALS_TIMEOUT (15) retries, send fake signals

   b) klp_update_patch_state()                     [transition.c:184]
      - Called on kernel exit (return to userspace)
      - test_and_clear TIF_PATCH_PENDING
      - Set task->patch_state = klp_target_state

   c) __klp_sched_try_switch()                     [transition.c:366]
      - Called from cond_resched() for CPU-bound kthreads
      - Checks stack and switches if safe

4. COMPLETE: klp_complete_transition()             [transition.c:90]
   - If replace: unpatch replaced patches, discard nops
   - If unpatching: unpatch objects, synchronize
   - Clear func->transition for all funcs
   - Synchronize transition (schedule_on_each_cpu)
   - Reset all task->patch_state to KLP_TRANSITION_IDLE
   - Call post_patch or post_unpatch callbacks
```

### Stack Checking

Before a task can be switched to the target patch state, its stack must be
verified to not contain any to-be-patched or to-be-unpatched functions.
This is performed by `klp_check_stack()` at `transition.c:263`:

```c
static int klp_check_stack(struct task_struct *task, const char **oldname)
{
    unsigned long *entries = this_cpu_ptr(klp_stack_entries);
    /* ... */
    ret = stack_trace_save_tsk_reliable(task, entries, MAX_STACK_ENTRIES);
    /* ... */
    klp_for_each_object(klp_transition_patch, obj) {
        if (!obj->patched)
            continue;
        klp_for_each_func(obj, func) {
            ret = klp_check_stack_func(func, entries, nr_entries);
            /* ... */
        }
    }
    return 0;
}
```

The function `klp_check_stack_func()` at `transition.c:214` checks whether
any stack frame address falls within the address range of the function being
replaced:

- When **patching**: checks for the previous function (original or prior patch)
- When **unpatching**: checks for the to-be-removed new function

```c
for (i = 0; i < nr_entries; i++) {
    address = entries[i];
    if (address >= func_addr && address < func_addr + func_size)
        return -EAGAIN;
}
```

A per-CPU buffer of `MAX_STACK_ENTRIES` (100) entries is used for stack traces
(`transition.c:17-18`).

### Task Switching Strategies

Tasks that cannot be immediately switched are retried through several
mechanisms:

1. **Periodic retry**: `klp_transition_work` delayed workqueue retries every
   second (`transition.c:62`)
2. **Kernel exit**: `klp_update_patch_state()` is called when tasks return to
   userspace
3. **cond_resched()**: `__klp_sched_try_switch()` hooks into the scheduler
   for CPU-bound kthreads via a static key or dynamic preempt call
4. **Fake signals**: After 15 attempts (`SIGNALS_TIMEOUT`), `klp_send_signals()`
   at `transition.c:408` sends fake signals to userspace tasks and wakes kthreads
5. **Force**: Administrator can write to the `force` sysfs attribute, which
   calls `klp_force_transition()` at `transition.c:727` to unconditionally
   switch all tasks (breaks consistency guarantees)

### Transition Reversal

A transition in progress can be reversed via the `enabled` sysfs attribute.
`klp_reverse_transition()` at `transition.c:648`:

1. Clears all `TIF_PATCH_PENDING` flags
2. Calls `klp_synchronize_transition()` (schedule on each CPU)
3. Flips `klp_target_state` and `patch->enabled`
4. Issues `smp_wmb()` and restarts transition via `klp_start_transition()`

### Fork Handling

When a process forks, `klp_copy_process()` at `transition.c:697` copies the
parent's patch state to the child:

```c
void klp_copy_process(struct task_struct *child)
{
    if (test_tsk_thread_flag(current, TIF_PATCH_PENDING))
        set_tsk_thread_flag(child, TIF_PATCH_PENDING);
    else
        clear_tsk_thread_flag(child, TIF_PATCH_PENDING);
    child->patch_state = current->patch_state;
}
```

---

## ftrace-Based Function Redirection

Livepatch uses ftrace to intercept calls to original functions and redirect
them to replacement functions. Each patchable function must have an
`__fentry__` or `mcount` call at its prologue (ensured by
`CONFIG_DYNAMIC_FTRACE_WITH_REGS` or `CONFIG_DYNAMIC_FTRACE_WITH_ARGS`).

### The ftrace Handler

The core redirection logic is in `klp_ftrace_handler()` at `patch.c:40`:

```c
static void notrace klp_ftrace_handler(unsigned long ip,
                                       unsigned long parent_ip,
                                       struct ftrace_ops *fops,
                                       struct ftrace_regs *fregs)
{
    struct klp_ops *ops;
    struct klp_func *func;
    int patch_state;

    ops = container_of(fops, struct klp_ops, fops);

    bit = ftrace_test_recursion_trylock(ip, parent_ip);

    func = list_first_or_null_rcu(&ops->func_stack, struct klp_func,
                                  stack_node);

    smp_rmb();  /* pairs with smp_wmb in __klp_enable_patch */

    if (unlikely(func->transition)) {
        smp_rmb();  /* pairs with smp_wmb in klp_init_transition */
        patch_state = current->patch_state;

        if (patch_state == KLP_TRANSITION_UNPATCHED) {
            /* Use previously patched version or original */
            func = list_entry_rcu(func->stack_node.next,
                                  struct klp_func, stack_node);
            if (&func->stack_node == &ops->func_stack)
                goto unlock;  /* no previous patch, use original */
        }
    }

    if (func->nop)
        goto unlock;  /* NOP: run original code */

    ftrace_regs_set_instruction_pointer(fregs, (unsigned long)func->new_func);

unlock:
    ftrace_test_recursion_unlock(bit);
}
```

The handler's key behavior:

1. Gets the top function from `ops->func_stack` (the newest patch)
2. During transitions, checks `current->patch_state`:
   - If `UNPATCHED`: uses the previous function version (or original)
   - If `PATCHED`: uses the newest function version
3. NOP functions cause no redirection (original code runs)
4. Otherwise, sets the instruction pointer to `func->new_func`

### Patching a Function

`klp_patch_func()` at `patch.c:160` registers the ftrace hook:

```c
static int klp_patch_func(struct klp_func *func)
{
    ops = klp_find_ops(func->old_func);
    if (!ops) {
        /* First patch for this function - create new klp_ops */
        ftrace_loc = ftrace_location((unsigned long)func->old_func);

        ops = kzalloc(sizeof(*ops), GFP_KERNEL);
        ops->fops.func = klp_ftrace_handler;
        ops->fops.flags = FTRACE_OPS_FL_DYNAMIC |
                          FTRACE_OPS_FL_IPMODIFY |
                          FTRACE_OPS_FL_PERMANENT;

        INIT_LIST_HEAD(&ops->func_stack);
        list_add_rcu(&func->stack_node, &ops->func_stack);

        ftrace_set_filter_ip(&ops->fops, ftrace_loc, 0, 0);
        register_ftrace_function(&ops->fops);
    } else {
        /* Additional patch stacked on existing one */
        list_add_rcu(&func->stack_node, &ops->func_stack);
    }
    func->patched = true;
}
```

Key ftrace flags used (`patch.c:187-192`):

| Flag | Purpose |
|------|---------|
| `FTRACE_OPS_FL_DYNAMIC` | Dynamically allocated ops |
| `FTRACE_OPS_FL_SAVE_REGS` | Save registers (when no DYNAMIC_FTRACE_WITH_ARGS) |
| `FTRACE_OPS_FL_IPMODIFY` | Handler modifies the instruction pointer |
| `FTRACE_OPS_FL_PERMANENT` | Cannot be unregistered while module is loaded |

### Unpatching a Function

`klp_unpatch_func()` at `patch.c:127`:

- If this is the last function on the `func_stack` (singleton): unregister
  the ftrace function, remove the filter, free the `klp_ops`
- If other patches remain on the stack: simply remove this function from
  the `func_stack` via `list_del_rcu()`

RCU is used for safe removal: `list_del_rcu()` ensures the handler can
still read the list while removal is in progress, and
`unregister_ftrace_function()` performs an RCU synchronization internally.

---

## Shadow Variables

Shadow variables allow livepatch modules to associate new data with existing
kernel objects without modifying their structure definitions. They are stored
in a global RCU-protected hash table.

### Implementation

Defined in `kernel/livepatch/shadow.c`:

```c
/* shadow.c:38 */
static DEFINE_HASHTABLE(klp_shadow_hash, 12);   /* 4096 buckets */
static DEFINE_SPINLOCK(klp_shadow_lock);

/* shadow.c:54 */
struct klp_shadow {
    struct hlist_node node;     /* hash table node */
    struct rcu_head rcu_head;   /* RCU-safe freeing */
    void *obj;                  /* pointer to parent object */
    unsigned long id;           /* data identifier */
    char data[];                /* flexible array for shadow data */
};
```

Shadow variables are keyed by an `<obj, id>` pair where `obj` is a pointer
to the parent kernel object and `id` is a caller-defined identifier.

### API

| Function | Location | Purpose |
|----------|----------|---------|
| `klp_shadow_get()` | `shadow.c:83` | Retrieve existing shadow data |
| `klp_shadow_alloc()` | `shadow.c:196` | Allocate new shadow (WARN on duplicate) |
| `klp_shadow_get_or_alloc()` | `shadow.c:225` | Get existing or allocate new |
| `klp_shadow_free()` | `shadow.c:253` | Free a specific `<obj, id>` shadow |
| `klp_shadow_free_all()` | `shadow.c:283` | Free all shadows with a given `id` |

Constructor and destructor callbacks (`include/linux/livepatch.h:215-218`):

```c
typedef int (*klp_shadow_ctor_t)(void *obj, void *shadow_data, void *ctor_data);
typedef void (*klp_shadow_dtor_t)(void *obj, void *shadow_data);
```

### Concurrency Model

- **klp_shadow_get()**: RCU read-side lock (`shadow.c:87-98`), lock-free lookup
- **klp_shadow_alloc()**: Optimistic allocation outside lock, then recheck
  under `klp_shadow_lock` spinlock (`shadow.c:104-170`)
- **klp_shadow_free()**: `klp_shadow_lock` + `hash_del_rcu()` + `kfree_rcu()`
  (`shadow.c:234-241`)

The double-check pattern in `__klp_shadow_get_or_alloc()` (`shadow.c:104`)
first does a lock-free lookup, then allocates memory outside the spinlock,
and finally rechecks under the lock to handle races.

### Usage Example

```c
/* In a livepatch function, attach extra data to an existing struct: */
struct my_extra_data *data;

/* First time: allocate shadow variable */
data = klp_shadow_alloc(existing_obj, MY_SHADOW_ID,
                        sizeof(*data), GFP_KERNEL, NULL, NULL);

/* Later: retrieve shadow variable */
data = klp_shadow_get(existing_obj, MY_SHADOW_ID);

/* Cleanup: free shadow variable */
klp_shadow_free(existing_obj, MY_SHADOW_ID, NULL);

/* Module exit: free all shadows of this type */
klp_shadow_free_all(MY_SHADOW_ID, my_dtor);
```

---

## Patch Stacking and Atomic Replace

### Patch Stacking

Multiple patches can coexist on the system. When two patches modify the same
function, they are stacked in the `klp_ops.func_stack` list. The most recently
enabled patch's function sits at the top (front) of the list and is the one
called by `klp_ftrace_handler()`.

```
func_stack (list_head in klp_ops)
  |
  +-- [patch_C func]  <-- top: called by handler (newest)
  +-- [patch_B func]  <-- called if C is being unpatched and task is UNPATCHED
  +-- [patch_A func]  <-- called if B is being unpatched and task is UNPATCHED
  |
  (if list exhausted, original function runs)
```

The stacking is achieved through RCU list operations:
- `list_add_rcu()` pushes new functions to the front (`patch.c:197` or `216`)
- `list_del_rcu()` removes them during unpatching (`patch.c:150` or `154`)

### Atomic Replace Mode

When `patch->replace = true`, the new patch atomically replaces all existing
patches. This is designed for cumulative patches that include all prior fixes.

The replace mechanism works by creating "NOP" (no-operation) functions:

1. **NOP creation** (`core.c:584-600`): `klp_add_nops()` iterates over all
   existing patches and, for any function not already covered by the new
   patch, creates a NOP function entry:

```c
static int klp_add_nops(struct klp_patch *patch)
{
    klp_for_each_patch(old_patch) {
        klp_for_each_object(old_patch, old_obj) {
            err = klp_add_object_nops(patch, old_obj);
        }
    }
    return 0;
}
```

2. **NOP function allocation** (`core.c:524-550`): Each NOP function has
   `func->nop = true` and `func->new_func = func->old_func` (set later
   during `klp_init_object_loaded()` at `core.c:868-869`).

3. **During transition**: The `klp_ftrace_handler()` detects NOP functions
   and skips the instruction pointer modification (`patch.c:118-119`):

```c
if (func->nop)
    goto unlock;
```

4. **After transition completes** (`transition.c:101-103`):
   - `klp_unpatch_replaced_patches()` (`core.c:1159`) disables and unpatches
     all older patches
   - `klp_discard_nops()` (`core.c:1188`) removes the NOP functions from the
     new patch since they are no longer needed

### System State Compatibility

For cumulative patches, the state system ensures compatibility. A cumulative
patch (`replace=true`) **must** handle all system states modified by
existing patches. The check is in `klp_is_state_compatible()` at
`state.c:87`:

```c
static bool klp_is_state_compatible(struct klp_patch *patch,
                                    struct klp_state *old_state)
{
    state = klp_get_state(patch, old_state->id);
    /* A cumulative livepatch must handle all already modified states */
    if (!state)
        return !patch->replace;
    return state->version >= old_state->version;
}
```

---

## Integration with Modules, ftrace, and kallsyms

### Module Integration

Livepatch hooks into the module loader to handle patching of modules that
load or unload after a patch is applied.

**Module coming** (`core.c:1228`): `klp_module_coming()` is called when a
module enters `MODULE_STATE_COMING`. For each active patch with an object
matching the module name:

1. Sets `obj->mod` to the new module
2. Calls `klp_init_object_loaded()` to resolve symbols and apply relocations
3. Calls `klp_pre_patch_callback()` and `klp_patch_object()`
4. On failure, refuses the module load entirely

**Module going** (`core.c:1309`): `klp_module_going()` is called when a
module enters `MODULE_STATE_GOING`:

1. Sets `mod->klp_alive = false`
2. Calls `klp_cleanup_module_patches_limited()` to unpatch and clean up
   all patch objects targeting this module

The `mod->klp_alive` flag prevents races between module unloading and patch
operations (`core.c:76`):

```c
if (mod && mod->klp_alive)
    obj->mod = mod;
```

### kallsyms Integration

Livepatch uses kallsyms to resolve function addresses. The function
`klp_find_object_symbol()` at `core.c:156` handles symbol lookup:

- For vmlinux symbols: `kallsyms_on_each_match_symbol()`
- For module symbols: `module_kallsyms_on_each_symbol()`

The `old_sympos` field in `klp_func` resolves ambiguous symbols. When
`old_sympos = 0`, the symbol must be unique; otherwise, `old_sympos`
specifies the Nth occurrence in kallsyms.

During `klp_init_object_loaded()` (`core.c:835`), symbol sizes are also
resolved via `kallsyms_lookup_size_offset()` to populate `func->old_size`
and `func->new_size`, which are used by the stack checker.

### Livepatch Relocations

Livepatch modules use special ELF sections for relocations that reference
unexported kernel symbols. These are identified by section names with the
format `.klp.rela.<objname>.<section>` and symbols with the format
`.klp.sym.<objname>.<symname>,<sympos>`.

The relocation resolution is done by `klp_resolve_symbols()` at
`core.c:192`, which parses symbol names using `sscanf()`:

```c
cnt = sscanf(strtab + sym->st_name,
             ".klp.sym.%55[^.].%511[^,],%lu",
             sym_objname, sym_name, &sympos);
```

Two types of relocations are applied at different times:

1. **vmlinux relocations**: Applied when the klp module itself loads,
   via `klp_apply_section_relocs()` (`core.c:332`)
2. **Module relocations**: Applied when the target module loads (or is
   already loaded), via `klp_apply_object_relocs()` (`core.c:822`)

### ftrace Integration

The `FTRACE_OPS_FL_IPMODIFY` flag is critical -- it tells ftrace that this
handler modifies the instruction pointer. Only one `IPMODIFY` handler can
be registered per function at a time. The `FTRACE_OPS_FL_PERMANENT` flag
ensures the handler stays registered as long as the module is loaded.

---

## Key Code Paths

### Enabling a Patch (Happy Path)

```
module_init()
  klp_enable_patch()                               core.c:1077
    klp_init_patch_early()                          core.c:929
      Initialize lists, kobjects, work structs
      klp_init_object_early() for each obj          core.c:921
        klp_init_func_early() for each func         core.c:914
    klp_init_patch()                                core.c:951
      kobject_add (creates sysfs entries)
      klp_add_nops() if replace mode                core.c:584
      klp_init_object() for each obj                core.c:883
        klp_find_object_module()                    core.c:55
        klp_init_object_loaded() if loaded          core.c:835
          klp_apply_object_relocs()                 core.c:822
          klp_find_object_symbol() per func         core.c:156
    __klp_enable_patch()                            core.c:1009
      klp_init_transition(PATCHED)                  transition.c:573
      smp_wmb()
      klp_pre_patch_callback() per obj              core.h:26
      klp_patch_object() per obj                    patch.c:252
        klp_patch_func() per func                   patch.c:160
          ftrace_set_filter_ip()
          register_ftrace_function()
      klp_start_transition()                        transition.c:530
        Set TIF_PATCH_PENDING on all tasks
        klp_cond_resched_enable()
      klp_try_complete_transition()                 transition.c:451
        klp_try_switch_task() per task
          klp_check_stack()                         transition.c:263
            stack_trace_save_tsk_reliable()
            klp_check_stack_func() per func         transition.c:214
```

### The ftrace Handler (Hot Path)

```
original_func() entry:
  __fentry__
    ftrace dispatcher
      klp_ftrace_handler()                          patch.c:40
        ftrace_test_recursion_trylock()
        func = list_first_or_null_rcu(func_stack)
        smp_rmb()
        if func->transition:
          smp_rmb()
          patch_state = current->patch_state
          if UNPATCHED: walk to previous func
        if func->nop: goto unlock (run original)
        ftrace_regs_set_instruction_pointer(new_func)
        ftrace_test_recursion_unlock()
```

### Module Notification Path

```
module load notification:
  klp_module_coming(mod)                            core.c:1228
    For each active patch:
      For each obj matching mod->name:
        obj->mod = mod
        klp_init_object_loaded()                    core.c:835
        klp_pre_patch_callback()
        klp_patch_object()
        klp_post_patch_callback() if not transition patch

module unload notification:
  klp_module_going(mod)                             core.c:1309
    mod->klp_alive = false
    klp_cleanup_module_patches_limited(mod, NULL)   core.c:1199
      For each patch, for each matching obj:
        klp_pre_unpatch_callback()
        klp_unpatch_object()
        klp_post_unpatch_callback()
        klp_clear_object_relocs()
        klp_free_object_loaded()
```

---

## Limitations and Constraints

### Architecture Requirements

- Requires `CONFIG_HAVE_LIVEPATCH` arch support (x86_64, s390, arm64, ppc64)
- Requires `CONFIG_DYNAMIC_FTRACE_WITH_REGS` or `CONFIG_DYNAMIC_FTRACE_WITH_ARGS`
  for function entry hooking
- Reliable stack traces (`CONFIG_HAVE_RELIABLE_STACKTRACE`) are needed for
  the consistency model; without them, transitions may never complete
  (`core.c:1100-1103`)

### Function-Level Constraints

- Only functions with ftrace hooks (those compiled with `-pg` /
  `__fentry__`) can be patched
- Functions must be large enough to contain the ftrace call site
- Inline functions cannot be patched (they are expanded at compile time)
- Functions called before ftrace is initialized cannot be patched
- `__init` functions are discarded after boot and cannot be patched
- Static local variables in patched functions retain their original
  addresses (shared between old and new versions)

### Structural Constraints

- Data structure layout changes are not supported directly; shadow
  variables must be used to extend structures
- Semantic changes to functions called during the transition period require
  careful design (functions on sleeping task stacks delay the transition)
- Only one transition can be in progress at a time
  (`core.c:1014` and `core.c:984`)
- Symbol names must be unique within an object unless `old_sympos` is
  specified (`core.c:177-179`)

### Concurrency and Serialization

- The coarse `klp_mutex` serializes all patch operations (`core.c:38`),
  but is not held by the ftrace handler, `klp_update_patch_state()`, or
  `__klp_sched_try_switch()`
- Multiple memory barriers (`smp_wmb()` / `smp_rmb()`) are required to
  ensure correct ordering between transition state writes and reads in
  the handler
- RCU is used for `func_stack` manipulation, but a special
  `klp_synchronize_transition()` (`transition.c:81-84`) uses
  `schedule_on_each_cpu()` instead of standard `synchronize_rcu()` because
  patched functions may be called where RCU is not watching

### Forced Transitions

- Writing to the `force` sysfs attribute bypasses the consistency model
  (`transition.c:727`)
- Forced patches set the `forced` flag, which prevents `module_put()` on
  the patch module to avoid use-after-free
- Once forced, a patch module can never be safely unloaded
- Forced transitions should only be used as a last resort when tasks
  are permanently stuck in the old state

### Module Interaction

- Patching a module that is not yet loaded works: the patch is applied
  when the module loads via `klp_module_coming()`
- If a patch fails to apply to a loading module, the module load itself
  is refused (`core.c:1299-1300`)
- `CONFIG_TRIM_UNUSED_KSYMS` must be disabled because livepatch needs
  access to unexported symbols via kallsyms
- Module names in patch objects are limited to `MODULE_NAME_LEN`
  (`core.c:889`)
