# Linux Kernel Cgroups Overview

This document provides an in-depth overview of the Linux kernel control groups
(cgroups) subsystem, covering the core data structures, hierarchy management,
resource controllers, interface files, process association, delegation model,
BPF integration, and key code paths with source references.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Data Structures](#2-core-data-structures)
3. [Hierarchy Management](#3-hierarchy-management)
4. [Resource Controllers](#4-resource-controllers)
5. [Interface Files and the cftype System](#5-interface-files-and-the-cftype-system)
6. [Process Association](#6-process-association)
7. [Delegation and Permission Model](#7-delegation-and-permission-model)
8. [Key Code Paths](#8-key-code-paths)
9. [Integration with Scheduler, Memory, and BPF](#9-integration-with-scheduler-memory-and-bpf)
10. [Cgroup BPF Programs](#10-cgroup-bpf-programs)

---

## 1. Architecture Overview

Control groups (cgroups) provide a mechanism for organizing processes into
hierarchical groups whose resource usage can be monitored, limited, and
controlled. The cgroup subsystem has undergone a major redesign from v1 to v2,
resulting in two co-existing interfaces.

### 1.1 Cgroup v1

Cgroup v1 allows multiple independent hierarchies, each with a different set
of resource controllers (subsystems) attached. A single controller can only be
attached to one hierarchy at a time. This design led to fragmentation: a
process could be in different positions across different hierarchies, making
coordinated resource control difficult.

In v1, each hierarchy is mounted separately (e.g., `mount -t cgroup -o cpu
none /sys/fs/cgroup/cpu`). The controllers available for v1 include legacy-only
subsystems like `net_cls`, `net_prio`, `devices`, and `freezer` that are not
supported on the default (v2) hierarchy.

### 1.2 Cgroup v2 (Unified Hierarchy)

Cgroup v2 enforces a single unified hierarchy, represented by the global
`cgrp_dfl_root` variable declared in `include/linux/cgroup.h:70`. All
controllers attach to this single hierarchy and are enabled or disabled on a
per-subtree basis via the `cgroup.subtree_control` interface file.

Key v2 design principles:

- **Single hierarchy**: One tree to rule them all, eliminating inconsistencies.
- **No internal process constraint**: A cgroup that has controllers enabled
  for its children cannot itself contain processes (with the exception of
  the root cgroup and threaded subtrees).
- **Top-down constraint**: A controller can only be enabled in a child if it
  is enabled in the parent's `subtree_control`.
- **Threaded mode**: Subtrees can be converted to "threaded" mode, relaxing
  the no-internal-process constraint and allowing per-thread resource control
  for controllers that support it.

### 1.3 Subsystem Enumeration

All cgroup subsystems are enumerated in `include/linux/cgroup_subsys.h` using
the `SUBSYS()` macro pattern. This file is included multiple times with
different macro definitions to generate the subsystem ID enum, extern
declarations, and name arrays. The subsystems registered are:

```c
/* include/linux/cgroup_subsys.h */
SUBSYS(cpuset)        /* CONFIG_CPUSETS */
SUBSYS(cpu)           /* CONFIG_CGROUP_SCHED */
SUBSYS(cpuacct)       /* CONFIG_CGROUP_CPUACCT */
SUBSYS(io)            /* CONFIG_BLK_CGROUP */
SUBSYS(memory)        /* CONFIG_MEMCG */
SUBSYS(devices)       /* CONFIG_CGROUP_DEVICE */
SUBSYS(freezer)       /* CONFIG_CGROUP_FREEZER */
SUBSYS(net_cls)       /* CONFIG_CGROUP_NET_CLASSID */
SUBSYS(perf_event)    /* CONFIG_CGROUP_PERF */
SUBSYS(net_prio)      /* CONFIG_CGROUP_NET_PRIO */
SUBSYS(hugetlb)       /* CONFIG_CGROUP_HUGETLB */
SUBSYS(pids)          /* CONFIG_CGROUP_PIDS */
SUBSYS(rdma)          /* CONFIG_CGROUP_RDMA */
SUBSYS(misc)          /* CONFIG_CGROUP_MISC */
SUBSYS(debug)         /* CONFIG_CGROUP_DEBUG -- not on default hierarchy */
```

The corresponding enum `cgroup_subsys_id` is generated in
`include/linux/cgroup-defs.h:43-46`:

```c
#define SUBSYS(_x) _x ## _cgrp_id,
enum cgroup_subsys_id {
#include <linux/cgroup_subsys.h>
    CGROUP_SUBSYS_COUNT,
};
#undef SUBSYS
```

---

## 2. Core Data Structures

### 2.1 struct cgroup

**Location:** `include/linux/cgroup-defs.h:431`

The `cgroup` struct represents a single node in the cgroup hierarchy. It
embeds a `cgroup_subsys_state` named `self` (with a NULL `->ss` pointer) that
serves as the cgroup's own CSS for reference counting and tree linkage. The
struct tracks hierarchy depth, descendant counts, population counters, and
pointers to per-subsystem CSS instances. It also contains the kernfs node for
filesystem representation, PSI tracking, BPF program storage, and freezer
state.

```c
struct cgroup {
    /* self css with NULL ->ss, points back to this cgroup */
    struct cgroup_subsys_state self;

    unsigned long flags;            /* CGRP_NOTIFY_ON_RELEASE, CGRP_FREEZE, etc. */

    int level;                      /* depth in hierarchy (root = 0) */
    int max_depth;                  /* maximum allowed descent tree depth */

    int nr_descendants;             /* total visible descendant cgroups */
    int nr_dying_descendants;       /* deleted but still referenced descendants */
    int max_descendants;            /* maximum allowed descendants */

    /* population tracking */
    int nr_populated_csets;                 /* non-empty css_sets in this cgroup */
    int nr_populated_domain_children;       /* domain children with tasks */
    int nr_populated_threaded_children;     /* threaded children with tasks */
    int nr_threaded_children;               /* live threaded child cgroups */

    struct kernfs_node *kn;                 /* cgroup kernfs entry */
    struct cgroup_file procs_file;          /* handle for "cgroup.procs" */
    struct cgroup_file events_file;         /* handle for "cgroup.events" */

    /* controller enable masks */
    u16 subtree_control;            /* user-configured via cgroup.subtree_control */
    u16 subtree_ss_mask;            /* effective mask (may include dependencies) */

    /* per-subsystem CSS pointers */
    struct cgroup_subsys_state __rcu *subsys[CGROUP_SUBSYS_COUNT];

    struct cgroup_root *root;       /* hierarchy root */
    struct list_head cset_links;    /* list of cgrp_cset_links */

    /* effective css_set lists, one per subsystem */
    struct list_head e_csets[CGROUP_SUBSYS_COUNT];

    struct cgroup *dom_cgrp;        /* domain ancestor (self if not threaded) */

    /* resource statistics */
    struct cgroup_base_stat bstat;
    struct prev_cputime prev_cputime;

    /* PSI tracking */
    struct psi_group *psi;

    /* BPF programs */
    struct cgroup_bpf bpf;

    /* freezer state */
    struct cgroup_freezer_state freezer;

    /* all ancestors including self (flexible array) */
    struct cgroup *ancestors[];
};
```

The `ancestors[]` flexible array member enables O(1) ancestry tests. The
function `cgroup_is_descendant()` in `include/linux/cgroup.h:516-522` simply
checks `cgrp->ancestors[ancestor->level] == ancestor`.

### 2.2 struct cgroup_subsys_state (CSS)

**Location:** `include/linux/cgroup-defs.h:162`

The CSS is the fundamental building block that controllers interact with. Each
cgroup has one CSS per enabled controller, plus a special `self` CSS (with
`ss == NULL`) for the cgroup core. The CSS provides reference counting,
parent-child linkage, and an ID for lookup. Fields marked "PI:" (Public and
Immutable) can be accessed without synchronization.

```c
struct cgroup_subsys_state {
    /* PI: the cgroup that this css is attached to */
    struct cgroup *cgroup;

    /* PI: the cgroup subsystem that this css is attached to */
    struct cgroup_subsys *ss;

    /* reference count - access via css_[try]get() and css_put() */
    struct percpu_ref refcnt;

    /* tree linkage (protected by cgroup_mutex or RCU) */
    struct list_head sibling;
    struct list_head children;

    /* PI: subsys-unique ID (0 unused, root always 1) */
    int id;

    unsigned int flags;             /* CSS_NO_REF, CSS_ONLINE, CSS_DYING, etc. */
    u64 serial_nr;                  /* monotonically increasing order among csses */
    atomic_t online_cnt;            /* online self + children count */

    struct work_struct destroy_work;
    struct rcu_work destroy_rwork;

    /* PI: parent css (placed for cache proximity) */
    struct cgroup_subsys_state *parent;
    int nr_descendants;             /* visible descendant CSS count */
};
```

CSS flags are defined at `include/linux/cgroup-defs.h:50-56`:

- `CSS_NO_REF` -- no reference counting (used for root CSSes)
- `CSS_ONLINE` -- between `css_online()` and `css_offline()` callbacks
- `CSS_RELEASED` -- refcnt reached zero
- `CSS_VISIBLE` -- visible to userland
- `CSS_DYING` -- css is dying

### 2.3 struct css_set

**Location:** `include/linux/cgroup-defs.h:244`

A `css_set` aggregates one CSS pointer per subsystem, representing the
complete cgroup membership of one or more tasks. Multiple tasks that belong
to exactly the same set of cgroups share a single `css_set`, avoiding per-task
overhead. The task_struct's `->cgroups` pointer (at `include/linux/sched.h:1291`)
points to a `css_set`. This design makes `fork()` and `exit()` efficient since
only a refcount increment/decrement and a list operation are needed.

```c
struct css_set {
    /* one CSS per subsystem (immutable after creation) */
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

    refcount_t refcount;

    /* for threaded cgroups: domain cset (points to self if not threaded) */
    struct css_set *dom_cset;

    /* the default-hierarchy cgroup for this css_set */
    struct cgroup *dfl_cgrp;

    /* task count, protected by css_set_lock */
    int nr_tasks;

    /* task lists */
    struct list_head tasks;         /* tasks using this css_set */
    struct list_head mg_tasks;      /* tasks being migrated */
    struct list_head dying_tasks;   /* exiting tasks */

    /* effective cset nodes (per-subsystem) */
    struct list_head e_cset_node[CGROUP_SUBSYS_COUNT];

    /* hash table linkage for deduplication */
    struct hlist_node hlist;

    /* M:N relationship with cgroups */
    struct list_head cgrp_links;    /* cgrp_cset_links for this css_set */

    /* migration bookkeeping */
    struct list_head mg_src_preload_node;
    struct list_head mg_dst_preload_node;
    struct list_head mg_node;
    struct cgroup *mg_src_cgrp;
    struct cgroup *mg_dst_cgrp;
    struct css_set *mg_dst_cset;

    bool dead;                      /* dead and being drained */
    struct rcu_head rcu_head;
};
```

The M:N relationship between cgroups and css_sets is tracked by the
`cgrp_cset_link` structure defined in `kernel/cgroup/cgroup-internal.h:96-106`:

```c
struct cgrp_cset_link {
    struct cgroup       *cgrp;
    struct css_set      *cset;
    struct list_head    cset_link;   /* anchored at cgrp->cset_links */
    struct list_head    cgrp_link;   /* anchored at css_set->cgrp_links */
};
```

### 2.4 struct cgroup_subsys

**Location:** `include/linux/cgroup-defs.h:708`

This structure defines a cgroup controller (subsystem). It contains the
lifecycle callback functions that the core invokes when cgroups are created,
destroyed, or tasks are migrated. Each controller implements a subset of
these callbacks. The struct also holds the controller's cftype arrays for both
the default (v2) and legacy (v1) hierarchies.

```c
struct cgroup_subsys {
    /* CSS lifecycle callbacks */
    struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent_css);
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    void (*css_released)(struct cgroup_subsys_state *css);
    void (*css_free)(struct cgroup_subsys_state *css);
    void (*css_reset)(struct cgroup_subsys_state *css);
    void (*css_rstat_flush)(struct cgroup_subsys_state *css, int cpu);

    /* task migration callbacks */
    int (*can_attach)(struct cgroup_taskset *tset);
    void (*cancel_attach)(struct cgroup_taskset *tset);
    void (*attach)(struct cgroup_taskset *tset);
    void (*post_attach)(void);

    /* fork/exit callbacks */
    int (*can_fork)(struct task_struct *task, struct css_set *cset);
    void (*cancel_fork)(struct task_struct *task, struct css_set *cset);
    void (*fork)(struct task_struct *task);
    void (*exit)(struct task_struct *task);
    void (*release)(struct task_struct *task);

    /* properties */
    bool early_init:1;
    bool implicit_on_dfl:1;         /* always enabled on default hierarchy */
    bool threaded:1;                /* supports threaded mode */

    int id;                         /* CGROUP_SUBSYS_COUNT-bounded */
    const char *name;

    struct cgroup_root *root;       /* current hierarchy root */
    struct idr css_idr;             /* css->id allocator */

    struct list_head cfts;          /* registered cftypes */
    struct cftype *dfl_cftypes;     /* v2 interface files */
    struct cftype *legacy_cftypes;  /* v1 interface files */

    unsigned int depends_on;        /* bitmask of subsystem dependencies */
};
```

### 2.5 struct cftype

**Location:** `include/linux/cgroup-defs.h:621`

The `cftype` (cgroup file type) structure defines a control file that appears
in cgroup directories. Each cftype specifies the file name, read/write
handlers, flags, and optional notification support. Controllers declare arrays
of cftypes that are registered during initialization.

```c
struct cftype {
    char name[MAX_CFTYPE_NAME];     /* file name (e.g., "max", "current") */
    unsigned long private;          /* controller-private data */
    size_t max_write_len;           /* max write buffer size */
    unsigned int flags;             /* CFTYPE_* flags */
    unsigned int file_offset;       /* offset for cgroup_file handle */

    /* internal bookkeeping */
    struct cgroup_subsys *ss;       /* NULL for cgroup core files */
    struct list_head node;          /* anchored at ss->cfts */
    struct kernfs_ops *kf_ops;

    /* callbacks */
    int (*open)(struct kernfs_open_file *of);
    void (*release)(struct kernfs_open_file *of);

    u64 (*read_u64)(struct cgroup_subsys_state *css, struct cftype *cft);
    s64 (*read_s64)(struct cgroup_subsys_state *css, struct cftype *cft);
    int (*seq_show)(struct seq_file *sf, void *v);

    int (*write_u64)(struct cgroup_subsys_state *css, struct cftype *cft, u64 val);
    int (*write_s64)(struct cgroup_subsys_state *css, struct cftype *cft, s64 val);
    ssize_t (*write)(struct kernfs_open_file *of, char *buf, size_t nbytes, loff_t off);

    __poll_t (*poll)(struct kernfs_open_file *of, struct poll_table_struct *pt);
};
```

Key cftype flags (`include/linux/cgroup-defs.h:128-141`):

- `CFTYPE_ONLY_ON_ROOT` -- only create on root cgroup
- `CFTYPE_NOT_ON_ROOT` -- do not create on root cgroup
- `CFTYPE_NS_DELEGATABLE` -- writeable beyond delegation boundaries
- `__CFTYPE_ONLY_ON_DFL` -- only on default (v2) hierarchy
- `__CFTYPE_NOT_ON_DFL` -- not on default hierarchy

### 2.6 struct cgroup_root

**Location:** `include/linux/cgroup-defs.h:578`

A `cgroup_root` represents the root of a cgroup hierarchy. The default (v2)
hierarchy root is the global `cgrp_dfl_root`. Each v1 mount creates a
separate `cgroup_root`. The root embeds a `cgroup` struct that serves as the
root cgroup of the hierarchy.

```c
struct cgroup_root {
    struct kernfs_root *kf_root;            /* kernfs root */
    unsigned int subsys_mask;               /* attached subsystems bitmask */
    int hierarchy_id;                       /* unique hierarchy ID */
    struct list_head root_list;             /* active hierarchies list */

    struct cgroup cgrp;                     /* root cgroup (embedded) */
    struct cgroup *cgrp_ancestor_storage;   /* overflow for ancestors[] */

    atomic_t nr_cgrps;                      /* total cgroups for /proc/cgroups */
    unsigned int flags;                     /* CGRP_ROOT_* flags */
    char release_agent_path[PATH_MAX];      /* v1 release agent */
    char name[MAX_CGROUP_ROOT_NAMELEN];     /* hierarchy name */
};
```

Root flags (`include/linux/cgroup-defs.h:77-125`) include:

- `CGRP_ROOT_NOPREFIX` -- mounted subsystems have no named prefix
- `CGRP_ROOT_NS_DELEGATE` -- namespaces are delegation boundaries
- `CGRP_ROOT_FAVOR_DYNMODS` -- optimize for dynamic cgroup modifications
- `CGRP_ROOT_MEMORY_RECURSIVE_PROT` -- enable recursive subtree protection
- `CGRP_ROOT_PIDS_LOCAL_EVENTS` -- enable legacy local pids.events

---

## 3. Hierarchy Management

### 3.1 Creation (mkdir)

**Location:** `kernel/cgroup/cgroup.c:5809`

When a user creates a new cgroup directory via `mkdir`, the kernfs syscall
operation routes to `cgroup_mkdir()`. This function:

1. Validates the name (rejects newlines to keep `/proc/<pid>/cgroup` parseable)
2. Locks the parent cgroup via `cgroup_kn_lock_live()`
3. Checks hierarchy depth/descendant limits via `cgroup_check_hierarchy_limits()`
4. Calls `cgroup_create()` to allocate and initialize the new cgroup
5. Populates the cgroup directory with control files via `css_populate_dir()`
6. Enables controllers inherited from the parent via `cgroup_apply_control_enable()`
7. Activates the kernfs node

```c
/* kernel/cgroup/cgroup.c:5809 */
int cgroup_mkdir(struct kernfs_node *parent_kn, const char *name, umode_t mode)
{
    struct cgroup *parent, *cgrp;
    int ret;

    if (strchr(name, '\n'))
        return -EINVAL;

    parent = cgroup_kn_lock_live(parent_kn, false);
    if (!parent)
        return -ENODEV;

    if (!cgroup_check_hierarchy_limits(parent)) {
        ret = -EAGAIN;
        goto out_unlock;
    }

    cgrp = cgroup_create(parent, name, mode);
    /* ... */
    ret = css_populate_dir(&cgrp->self);
    /* ... */
    ret = cgroup_apply_control_enable(cgrp);
    /* ... */
    kernfs_activate(cgrp->kn);
    /* ... */
}
```

### 3.2 Destruction (rmdir)

**Location:** `kernel/cgroup/cgroup.c:5963`

Cgroup removal via `rmdir` routes to `cgroup_rmdir()` which calls
`cgroup_destroy_locked()`. Destruction is only permitted when the cgroup is
empty (no tasks) and has no online children. The function:

1. Verifies the cgroup is not populated (`cgroup_is_populated()`)
2. Checks for live children (`css_has_online_children()`)
3. Clears the `CSS_ONLINE` flag and marks associated css_sets as dead
4. Kills all CSSes for the cgroup (`kill_css()` for each subsystem)
5. Removes the kernfs directory
6. Updates ancestor descendant/dying counts
7. For v2, offlines BPF programs (`cgroup_bpf_offline()`)
8. Kills the self refcount via `percpu_ref_kill()`

The actual freeing happens asynchronously when all references are released.
The css_release path (`kernel/cgroup/cgroup.c:5449`) handles final cleanup
via a workqueue, decrementing `nr_dying_descendants` in ancestors and
eventually freeing the cgroup memory.

```c
/* kernel/cgroup/cgroup.c:5963 */
static int cgroup_destroy_locked(struct cgroup *cgrp)
{
    if (cgroup_is_populated(cgrp))
        return -EBUSY;
    if (css_has_online_children(&cgrp->self))
        return -EBUSY;

    cgrp->self.flags &= ~CSS_ONLINE;
    /* mark associated css_sets dead */
    /* kill all CSSes */
    for_each_css(css, ssid, cgrp)
        kill_css(css);
    /* remove kernfs directory */
    css_clear_dir(&cgrp->self);
    kernfs_remove(cgrp->kn);
    /* ... */
    percpu_ref_kill(&cgrp->self.refcnt);
    return 0;
}
```

### 3.3 Migration

**Location:** `kernel/cgroup/cgroup.c:2880`

Task migration moves tasks between cgroups. The process is atomic: either all
tasks in the set move successfully or none do. Migration involves three phases:

**Phase 1 -- Preparation:** `cgroup_attach_task()` iterates all tasks in the
threadgroup, collecting their source css_sets via `cgroup_migrate_add_src()`.
Then `cgroup_migrate_prepare_dst()` computes or creates destination css_sets
using `find_css_set()`.

**Phase 2 -- Execution:** `cgroup_migrate_execute()` at line 2560 runs each
controller's `can_attach()` callback. If all succeed, it commits the migration
by atomically moving tasks from source to destination css_sets under
`css_set_lock`. It then calls each controller's `attach()` callback.

**Phase 3 -- Cleanup:** On failure, `cancel_attach()` callbacks are invoked in
reverse order. Migration context resources are freed by `cgroup_migrate_finish()`.

```c
/* kernel/cgroup/cgroup.c:2887 */
int cgroup_attach_task(struct cgroup *dst_cgrp, struct task_struct *leader,
                       bool threadgroup)
{
    DEFINE_CGROUP_MGCTX(mgctx);

    /* collect source csets */
    spin_lock_irq(&css_set_lock);
    rcu_read_lock();
    task = leader;
    do {
        cgroup_migrate_add_src(task_css_set(task), dst_cgrp, &mgctx);
        if (!threadgroup) break;
    } while_each_thread(leader, task);
    rcu_read_unlock();
    spin_unlock_irq(&css_set_lock);

    /* prepare destinations and commit */
    ret = cgroup_migrate_prepare_dst(&mgctx);
    if (!ret)
        ret = cgroup_migrate(leader, threadgroup, &mgctx);

    cgroup_migrate_finish(&mgctx);
    return ret;
}
```

The migration context structures are defined in
`kernel/cgroup/cgroup-internal.h:109-167`:

```c
struct cgroup_taskset {
    struct list_head    src_csets;
    struct list_head    dst_csets;
    int                 nr_tasks;
    int                 ssid;       /* subsystem being processed */
    /* ... iteration cursors ... */
};

struct cgroup_mgctx {
    struct list_head        preloaded_src_csets;
    struct list_head        preloaded_dst_csets;
    struct cgroup_taskset   tset;
    u16                     ss_mask;  /* affected subsystems */
};
```

---

## 4. Resource Controllers

### 4.1 CPU Controller (cpu)

**Config:** `CONFIG_CGROUP_SCHED`

The CPU controller integrates with the CFS and RT schedulers to provide CPU
bandwidth control. In v2, it offers weight-based proportional sharing via
`cpu.weight` (range 1-10000, default 100, defined as `CGROUP_WEIGHT_DFL` in
`include/linux/cgroup.h:38`) and hard bandwidth limits via `cpu.max`
(quota/period in microseconds). The controller creates per-cgroup task groups
(`struct task_group`) that embed CFS and RT bandwidth structures.

### 4.2 Memory Controller (memory)

**Config:** `CONFIG_MEMCG`

The memory controller tracks and limits memory usage including anonymous pages,
file cache, kernel memory (slab, stack, page tables), and swap. Key interface
files include `memory.max` (hard limit), `memory.high` (throttling threshold),
`memory.low` and `memory.min` (reclaim protection), and `memory.current`
(current usage). The controller deeply integrates with the page allocator, page
cache, and reclaim mechanisms.

### 4.3 I/O Controller (io)

**Config:** `CONFIG_BLK_CGROUP`

The I/O controller provides both weight-based proportional I/O bandwidth
allocation (`io.weight`) and absolute bandwidth/IOPS limits (`io.max`). It
integrates with the block layer's request queuing infrastructure. Each cgroup
gets per-device I/O statistics accessible through `io.stat`.

### 4.4 PIDs Controller (pids)

**Config:** `CONFIG_CGROUP_PIDS`

**Location:** `kernel/cgroup/pids.c`

The PIDs controller limits the number of processes that can be created within
a cgroup hierarchy. It is an excellent example of a minimal controller
implementation. The controller embeds a `cgroup_subsys_state` in its
per-cgroup state:

```c
/* kernel/cgroup/pids.c:49 */
struct pids_cgroup {
    struct cgroup_subsys_state  css;
    atomic64_t                  counter;        /* current PID count */
    atomic64_t                  limit;          /* configured limit */
    int64_t                     watermark;      /* peak usage */
    struct cgroup_file          events_file;
    struct cgroup_file          events_local_file;
    atomic64_t                  events[NR_PIDCG_EVENTS];
    atomic64_t                  events_local[NR_PIDCG_EVENTS];
};
```

The controller hooks into fork via `can_fork()` (line 273) which calls
`pids_try_charge()` to hierarchically check limits from the cgroup up to the
root. If any ancestor's counter would exceed its limit, fork returns `-EAGAIN`.
When a task exits, `pids_release()` (line 294) hierarchically decrements
counters via `pids_uncharge()`.

The controller definition at line 449 shows how a controller registers itself:

```c
/* kernel/cgroup/pids.c:449 */
struct cgroup_subsys pids_cgrp_subsys = {
    .css_alloc      = pids_css_alloc,
    .css_free       = pids_css_free,
    .can_attach     = pids_can_attach,
    .cancel_attach  = pids_cancel_attach,
    .can_fork       = pids_can_fork,
    .cancel_fork    = pids_cancel_fork,
    .release        = pids_release,
    .legacy_cftypes = pids_files_legacy,
    .dfl_cftypes    = pids_files,
    .threaded       = true,
};
```

### 4.5 Cpuset Controller (cpuset)

**Config:** `CONFIG_CPUSETS`

The cpuset controller restricts which CPUs and memory nodes a cgroup's tasks
can use. It provides `cpuset.cpus` and `cpuset.mems` to configure allowed
resources. The cpuset controller interacts closely with the scheduler's CPU
affinity mechanisms and the memory allocator's NUMA policies.

---

## 5. Interface Files and the cftype System

### 5.1 How Interface Files Work

Cgroup interface files are backed by kernfs. Each file in a cgroup directory
corresponds to a `cftype` entry. The cgroup core maintains a list of core
interface files (`cgroup_base_files[]` at `kernel/cgroup/cgroup.c:5250`), and
each controller registers its own cftypes via `dfl_cftypes` (v2) and
`legacy_cftypes` (v1) arrays.

During initialization (`kernel/cgroup/cgroup.c:6163`), the core calls
`cgroup_init_cftypes()` for base files and then registers each controller's
files via `cgroup_add_dfl_cftypes()` and `cgroup_add_legacy_cftypes()`:

```c
/* kernel/cgroup/cgroup.c:6239-6243 */
if (ss->dfl_cftypes == ss->legacy_cftypes) {
    WARN_ON(cgroup_add_cftypes(ss, ss->dfl_cftypes));
} else {
    WARN_ON(cgroup_add_dfl_cftypes(ss, ss->dfl_cftypes));
    WARN_ON(cgroup_add_legacy_cftypes(ss, ss->legacy_cftypes));
}
```

### 5.2 Core Interface Files (v2)

The `cgroup_base_files[]` array at `kernel/cgroup/cgroup.c:5250` defines the
files present in every v2 cgroup directory:

| File | Purpose |
|------|---------|
| `cgroup.type` | Show/set cgroup type (domain, threaded) |
| `cgroup.procs` | List/migrate processes (NS_DELEGATABLE) |
| `cgroup.threads` | List/migrate threads (NS_DELEGATABLE) |
| `cgroup.controllers` | Available controllers |
| `cgroup.subtree_control` | Enable/disable controllers (NS_DELEGATABLE) |
| `cgroup.events` | Populated/frozen events |
| `cgroup.max.descendants` | Max descendant cgroups |
| `cgroup.max.depth` | Max hierarchy depth |
| `cgroup.stat` | Descendant statistics |
| `cgroup.freeze` | Freeze/thaw the cgroup |
| `cgroup.kill` | Kill all processes in the cgroup |
| `cpu.stat` | CPU usage statistics |
| `cpu.stat.local` | Local (non-recursive) CPU stats |

Additionally, PSI (Pressure Stall Information) files are registered separately
in `cgroup_psi_files[]` at line 5328:

| File | Purpose |
|------|---------|
| `io.pressure` | I/O pressure stall information |
| `memory.pressure` | Memory pressure stall information |
| `cpu.pressure` | CPU pressure stall information |

### 5.3 Controller cftype Example (PIDs)

The PIDs controller registers separate file arrays for v2 and v1. The v2
array at `kernel/cgroup/pids.c:390` includes:

```c
static struct cftype pids_files[] = {
    {
        .name = "max",
        .write = pids_max_write,
        .seq_show = pids_max_show,
        .flags = CFTYPE_NOT_ON_ROOT,
    },
    {
        .name = "current",
        .read_s64 = pids_current_read,
        .flags = CFTYPE_NOT_ON_ROOT,
    },
    {
        .name = "peak",
        .flags = CFTYPE_NOT_ON_ROOT,
        .read_s64 = pids_peak_read,
    },
    {
        .name = "events",
        .seq_show = pids_events_show,
        .file_offset = offsetof(struct pids_cgroup, events_file),
        .flags = CFTYPE_NOT_ON_ROOT,
    },
    {
        .name = "events.local",
        .seq_show = pids_events_local_show,
        .file_offset = offsetof(struct pids_cgroup, events_local_file),
        .flags = CFTYPE_NOT_ON_ROOT,
    },
    { }  /* terminate */
};
```

These files appear as `pids.max`, `pids.current`, etc. in cgroup directories
because the controller name is prefixed automatically. The `CFTYPE_NOT_ON_ROOT`
flag prevents these files from appearing in the root cgroup where limits do
not apply. The `file_offset` field allows the core to record a `cgroup_file`
handle for event notification via `cgroup_file_notify()`.

### 5.4 Read/Write Callbacks

The cftype system supports multiple callback styles for convenience:

- **`read_u64` / `write_u64`**: For simple single-integer values
- **`read_s64` / `write_s64`**: For signed single-integer values
- **`seq_show`**: For complex output using seq_file
- **`write`**: For generic string-based input (most flexible)

Within a write callback, the CSS and cftype can be obtained from the
`kernfs_open_file` using `of_css()` and `of_cft()` helpers defined at
`include/linux/cgroup.h:574-585`.

---

## 6. Process Association

### 6.1 Task-to-Cgroup Linkage

Every task in the system is associated with exactly one cgroup per hierarchy
through the `css_set` indirection. The `task_struct` contains:

```c
/* include/linux/sched.h:1291-1293 */
struct css_set __rcu    *cgroups;    /* RCU-protected pointer to css_set */
struct list_head        cg_list;     /* linkage into css_set->tasks */
```

The `cgroups` pointer is RCU-protected, initialized during fork, and can only
be modified while holding both `cgroup_mutex` and `task_lock()`. For read
access, either `rcu_read_lock`, `cgroup_mutex`, or `css_set_lock` suffices
(see `task_css_set_check()` at `include/linux/cgroup.h:395-400`).

### 6.2 css_set Deduplication

Since many tasks share the same cgroup memberships, css_sets are deduplicated
via a hash table (`css_set_table`). The `find_css_set()` function at
`kernel/cgroup/cgroup.c:1172` first checks for an existing matching css_set
via `find_existing_css_set()`. If none exists, it allocates a new one, copies
the subsystem state template, links it to all relevant cgroups, and adds it
to the hash table:

```c
/* kernel/cgroup/cgroup.c:1172 */
static struct css_set *find_css_set(struct css_set *old_cset,
                                    struct cgroup *cgrp)
{
    struct cgroup_subsys_state *template[CGROUP_SUBSYS_COUNT] = { };

    /* check for existing match */
    spin_lock_irq(&css_set_lock);
    cset = find_existing_css_set(old_cset, cgrp, template);
    if (cset) get_css_set(cset);
    spin_unlock_irq(&css_set_lock);
    if (cset) return cset;

    /* allocate new css_set */
    cset = kzalloc(sizeof(*cset), GFP_KERNEL);
    memcpy(cset->subsys, template, sizeof(cset->subsys));

    /* link to cgroups and add to hash table */
    key = css_set_hash(cset->subsys);
    hash_add(css_set_table, &cset->hlist, key);
    /* ... */
}
```

### 6.3 Fork Path

**Location:** `kernel/cgroup/cgroup.c:6426` and `kernel/cgroup/cgroup.c:6683`

During `fork()`, cgroup handling occurs in two phases:

1. **`cgroup_fork()`** (line 6426): Initializes the child's `cgroups` pointer
   to `init_css_set` and initializes `cg_list`. The child is temporarily
   associated with the root cgroup.

2. **`cgroup_post_fork()`** (line 6683): After the child is fully set up, this
   function attaches it to the correct css_set. Under `css_set_lock`, it
   increments `cset->nr_tasks`, moves the child to the css_set's task list,
   and handles frozen/killed cgroup states. It then invokes each controller's
   `fork()` callback.

If `CLONE_INTO_CGROUP` is specified, `cgroup_css_set_fork()` (line 6472)
finds or creates a css_set for the target cgroup, checking permissions via
`cgroup_attach_permissions()`.

### 6.4 Exit Path

**Location:** `kernel/cgroup/cgroup.c:6779`

When a task exits, `cgroup_exit()` removes it from its css_set's task list,
decrements `nr_tasks`, and invokes each controller's `exit()` callback. The
task's `cg_list` is moved to the `dying_tasks` list if it is a thread group
leader. Later, `cgroup_release()` (line 6811) invokes `release()` callbacks
(used by the PIDs controller to uncharge the PID count), and `cgroup_free()`
drops the final css_set reference.

---

## 7. Delegation and Permission Model

### 7.1 Delegation in cgroup v2

Cgroup v2 supports safe delegation of cgroup subtrees to unprivileged users.
A user who has write access to a cgroup directory can manage its children.
The delegation model works through standard filesystem permissions and
namespace boundaries.

The key delegatable files are marked with `CFTYPE_NS_DELEGATABLE` in their
cftype flags. These are:

- `cgroup.procs` -- migrate processes
- `cgroup.threads` -- migrate threads
- `cgroup.subtree_control` -- enable/disable controllers

### 7.2 Permission Checks

**Location:** `kernel/cgroup/cgroup.c:5133`

The `cgroup_procs_write_permission()` function enforces the delegation model.
To migrate a process from one cgroup to another, the caller must have write
permission on the common ancestor of the source and destination cgroups:

```c
/* kernel/cgroup/cgroup.c:5133 */
static int cgroup_procs_write_permission(struct cgroup *src_cgrp,
                                         struct cgroup *dst_cgrp,
                                         struct super_block *sb,
                                         struct cgroup_namespace *ns)
{
    struct cgroup *com_cgrp = src_cgrp;

    /* find the common ancestor */
    while (!cgroup_is_descendant(dst_cgrp, com_cgrp))
        com_cgrp = cgroup_parent(com_cgrp);

    /* must be authorized to migrate to the common ancestor */
    ret = cgroup_may_write(com_cgrp, sb);
    if (ret)
        return ret;

    /* namespace delegation boundary check */
    if ((cgrp_dfl_root.flags & CGRP_ROOT_NS_DELEGATE) &&
        (!cgroup_is_descendant(src_cgrp, ns->root_cset->dfl_cgrp) ||
         !cgroup_is_descendant(dst_cgrp, ns->root_cset->dfl_cgrp)))
        return -ENOENT;

    return 0;
}
```

The full permission check is composed in `cgroup_attach_permissions()` at line
5164, which calls `cgroup_procs_write_permission()` followed by
`cgroup_migrate_vet_dst()` to ensure the destination cgroup can accept tasks.

### 7.3 Namespace Delegation Boundaries

When `CGRP_ROOT_NS_DELEGATE` is set (via the `nsdelegate` mount option), cgroup
namespaces act as delegation boundaries. Controller-specific files in a
namespace root are not writeable from inside the namespace. Both source and
destination cgroups must be visible within the caller's cgroup namespace for
migration to succeed.

### 7.4 Credential-Based Protection

The migration path at `kernel/cgroup/cgroup.c:5214` uses `override_creds()`
to check permissions using the credentials captured at file open time rather
than the current credentials. This prevents inherited file descriptor attacks
where a privileged process opens `cgroup.procs` and passes the fd to an
unprivileged process.

---

## 8. Key Code Paths

### 8.1 Initialization

The cgroup subsystem initializes in two stages:

**Early init** (`kernel/cgroup/cgroup.c:6126`):
- Initializes the default root (`cgrp_dfl_root`)
- Sets up `init_css_set` for the init task
- Initializes subsystems that require early setup (`ss->early_init == true`)

**Full init** (`kernel/cgroup/cgroup.c:6163`):
- Initializes core cftypes
- Sets up the default hierarchy root via `cgroup_setup_root()`
- Initializes remaining subsystems
- Registers cftypes for each subsystem
- Registers the cgroup filesystem type

### 8.2 Locking Summary

The cgroup subsystem uses a layered locking scheme:

| Lock | Type | Protects |
|------|------|----------|
| `cgroup_mutex` | mutex | All hierarchy modifications, css lifecycle |
| `css_set_lock` | spinlock | `task->cgroups`, css_set lists, task lists |
| `cgroup_threadgroup_rwsem` | percpu rwsem | Thread group integrity during migration |
| `cgroup_idr_lock` | spinlock | `cgroup_idr` and `css_idr` |
| `cgroup_file_kn_lock` | spinlock | `cgroup_file->kn` for non-self CSSes |

These are defined at `kernel/cgroup/cgroup.c:90-114`:

```c
DEFINE_MUTEX(cgroup_mutex);
DEFINE_SPINLOCK(css_set_lock);
DEFINE_PERCPU_RWSEM(cgroup_threadgroup_rwsem);
```

Modification of `task->cgroups` requires both `cgroup_mutex` and `task_lock()`.
Reading requires any one of: `rcu_read_lock`, `cgroup_mutex`, or
`css_set_lock`.

### 8.3 Controller Enable/Disable Flow

Enabling or disabling controllers follows a five-step protocol documented at
`kernel/cgroup/cgroup.c:3298`:

1. `cgroup_save_control()` -- stash current state
2. Update `->subtree_control` masks
3. `cgroup_apply_control()` -- propagate changes:
   - `cgroup_propagate_control()` -- compute effective masks
   - `cgroup_apply_control_enable()` (line 3223) -- create/show CSSes
   - `cgroup_update_dfl_csses()` -- migrate existing tasks to new css_sets
4. Perform related operations
5. `cgroup_finalize_control()` -- clean up (disable unused CSSes on success,
   restore old state on failure)

### 8.4 CSS Lifecycle

A CSS goes through these states:

1. **Allocation**: `ss->css_alloc()` creates controller-specific state
2. **Online**: `ss->css_online()` activates the CSS (sets `CSS_ONLINE`)
3. **Active**: CSS is operational, referenced by tasks and cgroup core
4. **Kill**: `percpu_ref_kill()` initiates shutdown
5. **Offline**: `ss->css_offline()` runs when refcount confirms kill (via
   `css_killed_work_fn()` at line 5867)
6. **Released**: `css_release()` (line 5517) triggers when all references drop
7. **Free**: `ss->css_free()` deallocates controller state

### 8.5 Source File Reference

| File | Purpose |
|------|---------|
| `include/linux/cgroup-defs.h` | Core type definitions |
| `include/linux/cgroup.h` | Public API and inline helpers |
| `include/linux/cgroup_subsys.h` | Subsystem enumeration |
| `kernel/cgroup/cgroup.c` | Core implementation (~6800 lines) |
| `kernel/cgroup/cgroup-internal.h` | Internal types and helpers |
| `kernel/cgroup/cgroup-v1.c` | v1-specific code |
| `kernel/cgroup/rstat.c` | Recursive statistics |
| `kernel/cgroup/namespace.c` | Cgroup namespace operations |
| `kernel/cgroup/pids.c` | PIDs controller |
| `kernel/cgroup/cpuset.c` | Cpuset controller |
| `kernel/cgroup/rdma.c` | RDMA controller |
| `kernel/cgroup/misc.c` | Miscellaneous controller |
| `mm/memcontrol.c` | Memory controller |
| `kernel/sched/core.c` | CPU controller (scheduler integration) |
| `block/blk-cgroup.c` | I/O controller |
| `kernel/bpf/cgroup.c` | BPF-cgroup integration |

---

## 9. Integration with Scheduler, Memory, and BPF

### 9.1 Scheduler Integration

The CPU controller creates a `struct task_group` for each cgroup that has the
CPU controller enabled. Task groups contain per-CPU CFS and RT runqueue entries
(`struct sched_entity` and `struct rt_rq`), allowing the scheduler to perform
hierarchical fair scheduling. When a task is placed in a cgroup with a CPU
weight, the scheduler uses the task group's shares to compute proportional CPU
time.

CPU accounting is performed in the scheduler hot path via
`cgroup_account_cputime()` defined at `include/linux/cgroup.h:719`:

```c
static inline void cgroup_account_cputime(struct task_struct *task,
                                          u64 delta_exec)
{
    struct cgroup *cgrp;

    cpuacct_charge(task, delta_exec);

    cgrp = task_dfl_cgroup(task);
    if (cgroup_parent(cgrp))
        __cgroup_account_cputime(cgrp, delta_exec);
}
```

This function is called from `update_curr()` in the scheduler, feeding data
into the per-cgroup `cpu.stat` files and the cgroup rstat infrastructure.

The cpuset controller interacts with the scheduler through CPU affinity
masks. When `cpuset.cpus` is updated, the controller updates the allowed CPU
mask for all tasks in the cgroup, which the scheduler respects when selecting
runqueues for task placement.

### 9.2 Memory Controller Integration

The memory controller (`memcg`) deeply integrates with the kernel's memory
management subsystem. Key integration points include:

- **Page allocator**: Memory charges are checked during page allocation; when
  limits are hit, reclaim is triggered or allocation fails
- **Page cache**: File-backed pages are charged to the cgroup that first faults
  them in
- **Slab allocator**: Kernel objects (dentries, inodes, etc.) are tracked per
  cgroup via `memcg_kmem_charge()`
- **OOM killer**: The OOM killer operates within cgroup boundaries, killing
  tasks in the cgroup that exceeded its limit
- **Reclaim**: The `memory.high` threshold triggers throttled reclaim;
  `memory.max` triggers synchronous reclaim and OOM
- **PSI**: Pressure Stall Information tracking per cgroup integrates with
  the cgroup's `psi_group` struct

### 9.3 BPF Integration

Cgroups embed a `struct cgroup_bpf` (defined in
`include/linux/bpf-cgroup-defs.h:55`) that holds BPF programs attached to the
cgroup. The BPF programs are organized by attach type, with an effective
program array that includes inherited programs from ancestor cgroups.

The cgroup BPF infrastructure provides per-cgroup hooks at various points in
the networking stack and system call processing. See section 10 for detailed
coverage.

---

## 10. Cgroup BPF Programs

### 10.1 Overview

BPF programs of type `BPF_PROG_TYPE_CGROUP_*` can be attached to cgroups to
implement per-cgroup policies for networking, device access, and sysctl
operations. These programs are inherited hierarchically: a program attached
to a parent cgroup affects all tasks in descendant cgroups unless overridden.

### 10.2 The cgroup_bpf Structure

**Location:** `include/linux/bpf-cgroup-defs.h:55`

```c
struct cgroup_bpf {
    /* effective programs (including inherited from ancestors) */
    struct bpf_prog_array __rcu *effective[MAX_CGROUP_BPF_ATTACH_TYPE];

    /* directly attached programs and flags */
    struct hlist_head progs[MAX_CGROUP_BPF_ATTACH_TYPE];
    u8 flags[MAX_CGROUP_BPF_ATTACH_TYPE];
    u64 revisions[MAX_CGROUP_BPF_ATTACH_TYPE];

    /* cgroup shared storages */
    struct list_head storages;

    /* temp storage for program array updates */
    struct bpf_prog_array *inactive;

    /* reference counter for cleanup after cgroup removal */
    struct percpu_ref refcnt;
    struct work_struct release_work;
};
```

The `effective[]` array is the key runtime structure. It contains the flattened
array of BPF programs that apply to this cgroup, including programs inherited
from ancestors. This array is recomputed when programs are attached or detached
anywhere in the hierarchy.

### 10.3 Attach Types

The `enum cgroup_bpf_attach_type` at `include/linux/bpf-cgroup-defs.h:20`
defines all the hook points where BPF programs can be attached to cgroups:

**Networking hooks:**
- `CGROUP_INET_INGRESS` / `CGROUP_INET_EGRESS` -- packet filtering
- `CGROUP_INET_SOCK_CREATE` / `CGROUP_INET_SOCK_RELEASE` -- socket lifecycle
- `CGROUP_INET4_BIND` / `CGROUP_INET6_BIND` -- bind() control
- `CGROUP_INET4_CONNECT` / `CGROUP_INET6_CONNECT` / `CGROUP_UNIX_CONNECT` -- connect() control
- `CGROUP_INET4_POST_BIND` / `CGROUP_INET6_POST_BIND` -- post-bind processing
- `CGROUP_UDP4_SENDMSG` / `CGROUP_UDP6_SENDMSG` / `CGROUP_UNIX_SENDMSG` -- sendmsg control
- `CGROUP_UDP4_RECVMSG` / `CGROUP_UDP6_RECVMSG` / `CGROUP_UNIX_RECVMSG` -- recvmsg control
- `CGROUP_SOCK_OPS` -- TCP/socket operation callbacks
- `CGROUP_GETSOCKOPT` / `CGROUP_SETSOCKOPT` -- socket option interception
- `CGROUP_INET4_GETPEERNAME` / `CGROUP_INET6_GETPEERNAME` -- getpeername interception
- `CGROUP_INET4_GETSOCKNAME` / `CGROUP_INET6_GETSOCKNAME` -- getsockname interception

**System hooks:**
- `CGROUP_SYSCTL` -- sysctl read/write interception
- `CGROUP_DEVICE` -- device access control (replaces v1 devices controller)

**LSM hooks:**
- `CGROUP_LSM_START` through `CGROUP_LSM_END` -- per-cgroup LSM hooks
  (up to `CGROUP_LSM_NUM = 10` concurrent hooks when `CONFIG_BPF_LSM` is enabled)

### 10.4 Attach Flags

Programs can be attached with different flags controlling inheritance behavior:

- **No flags / `BPF_F_ALLOW_OVERRIDE`**: The cgroup can have at most one
  program per attach type. A child cgroup can override the parent's program.
- **`BPF_F_ALLOW_MULTI`**: Multiple programs can be attached to the same type.
  Programs from the cgroup and all its ancestors form the effective array, with
  all programs executed in order from root to leaf.

### 10.5 Runtime Execution

BPF cgroup programs are invoked through `BPF_CGROUP_RUN_PROG_*` macros defined
in `include/linux/bpf-cgroup.h`. These macros first check a static key
(`cgroup_bpf_enabled()`) to avoid overhead when no programs are attached, then
call the corresponding `__cgroup_bpf_run_filter_*()` function from
`kernel/bpf/cgroup.c`.

For example, ingress packet filtering (line 1529 of `kernel/bpf/cgroup.c`):

```c
/* include/linux/bpf-cgroup.h:196 */
#define BPF_CGROUP_RUN_PROG_INET_INGRESS(sk, skb)                          \
({                                                                          \
    int __ret = 0;                                                          \
    if (cgroup_bpf_enabled(CGROUP_INET_INGRESS) &&                         \
        cgroup_bpf_sock_enabled(sk, CGROUP_INET_INGRESS))                  \
        __ret = __cgroup_bpf_run_filter_skb(sk, skb,                        \
                                            CGROUP_INET_INGRESS);           \
    __ret;                                                                  \
})
```

The `__cgroup_bpf_run_filter_skb()` function at `kernel/bpf/cgroup.c:1529`
looks up the task's cgroup, retrieves the effective program array for the
attach type, and runs each program through the BPF interpreter/JIT. The
program can return `1` (allow) or `0` (deny).

### 10.6 Lifecycle

BPF programs attached to a cgroup are reference-counted through
`cgroup_bpf.refcnt`. When a cgroup is destroyed, `cgroup_bpf_offline()` is
called (invoked from `cgroup_destroy_locked()` at line 6028 of
`kernel/cgroup/cgroup.c`). The actual release of BPF resources happens
asynchronously via `release_work` when the percpu refcount reaches zero. The
`cgroup_bpf_get()` and `cgroup_bpf_put()` helpers at
`include/linux/cgroup.h:847-855` manage the lifecycle reference counting.
