# Linux Kernel Namespaces Overview

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [The Eight Namespace Types](#the-eight-namespace-types)
3. [Core Data Structures](#core-data-structures)
4. [Namespace Lifecycle](#namespace-lifecycle)
5. [The nsproxy Mechanism](#the-nsproxy-mechanism)
6. [User Namespaces and the Capability/Permission Model](#user-namespaces-and-the-capabilitypermission-model)
7. [PID Namespace Nesting and PID Translation](#pid-namespace-nesting-and-pid-translation)
8. [Network Namespace Internals](#network-namespace-internals)
9. [Key Syscalls](#key-syscalls)
10. [Integration with Other Subsystems](#integration-with-other-subsystems)

---

## Architecture Overview

Namespaces are a Linux kernel feature that partition global system resources so
that a set of processes sees one view of a resource while another set of
processes sees a different view. They are the fundamental isolation primitive
underlying containers (Docker, LXC, systemd-nspawn) and sandboxing technologies.

### What Namespaces Provide

```
┌─────────────────────────────────────────────────────────────────┐
│               Linux Namespace Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Without namespaces:              With namespaces:              │
│  ──────────────────               ────────────────              │
│                                                                 │
│  ┌───────────────┐               ┌────────┐  ┌────────┐        │
│  │  Global View  │               │ View A │  │ View B │        │
│  │               │               │        │  │        │        │
│  │ PID 1..N      │               │ PID 1  │  │ PID 1  │        │
│  │ eth0, lo      │               │ veth0  │  │ veth1  │        │
│  │ hostname      │               │ host-a │  │ host-b │        │
│  │ /proc /sys    │               │ /proc  │  │ /proc  │        │
│  │ uid 0..65535  │               │ uid 0  │  │ uid 0  │        │
│  └───────────────┘               └────────┘  └────────┘        │
│                                       │           │             │
│                                  ┌────┴───────────┴────┐       │
│                                  │   Actual Hardware    │       │
│                                  │   (shared kernel)    │       │
│                                  └─────────────────────┘       │
│                                                                 │
│  Key insight: namespaces provide isolation WITHOUT duplication  │
│  of the kernel itself (unlike virtual machines).                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Hierarchical where needed** -- PID and user namespaces form parent-child
   trees. Others are flat (a process is simply "in" one namespace instance).

2. **Reference counted** -- every namespace embeds `struct ns_common` with a
   refcount. The namespace is freed when the last reference drops.

3. **Lazy creation** -- most tasks share the init namespace. A new namespace is
   created only when explicitly requested via `clone()`, `unshare()`, or
   `setns()`.

4. **Owner tracking** -- every non-user namespace has an owning
   `user_namespace`. Capabilities are checked against this owner.

---

## The Eight Namespace Types

Linux supports eight distinct namespace types, each isolating a different
global resource.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Type        CLONE Flag       Isolates                         Since        │
├──────────────────────────────────────────────────────────────────────────────┤
│  Mount       CLONE_NEWNS      Mount points, filesystem tree    2.4.19       │
│  UTS         CLONE_NEWUTS     Hostname and NIS domain name     2.6.19       │
│  IPC         CLONE_NEWIPC     System V IPC, POSIX mqueues      2.6.19       │
│  PID         CLONE_NEWPID     Process IDs                      2.6.24       │
│  Network     CLONE_NEWNET     Network stack (devices, etc.)    2.6.29       │
│  User        CLONE_NEWUSER    UIDs/GIDs, capabilities          3.8          │
│  Cgroup      CLONE_NEWCGROUP  Cgroup root directory            4.6          │
│  Time        CLONE_NEWTIME    CLOCK_MONOTONIC, CLOCK_BOOTTIME  5.6          │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Clone Flag Definitions

```c
/* include/uapi/linux/sched.h */

#define CLONE_NEWNS       0x00020000   /* New mount namespace group */
#define CLONE_NEWCGROUP   0x02000000   /* New cgroup namespace */
#define CLONE_NEWUTS      0x04000000   /* New utsname namespace */
#define CLONE_NEWIPC      0x08000000   /* New ipc namespace */
#define CLONE_NEWUSER     0x10000000   /* New user namespace */
#define CLONE_NEWPID      0x20000000   /* New pid namespace */
#define CLONE_NEWNET      0x40000000   /* New network namespace */
#define CLONE_NEWTIME     0x00000080   /* New time namespace */
```

### Brief Description of Each Type

**Mount (mnt)** -- Isolates the set of filesystem mount points. Each mount
namespace has its own mount tree. This was the first namespace type added to
Linux (hence the generic `CLONE_NEWNS` flag name). Defined in `fs/mount.h`.

**UTS** -- Isolates the hostname (`uname -n`) and NIS domain name. Named after
the `utsname` structure from `uname()`. Allows each container to have its own
hostname.

**IPC** -- Isolates System V IPC objects (shared memory segments, semaphore
sets, message queues) and POSIX message queues. Prevents cross-container IPC.

**PID** -- Isolates the PID number space. Processes in different PID namespaces
can have the same PID. The first process in a new PID namespace is PID 1 (the
namespace's init). PID namespaces are hierarchical -- a parent can see children
but not vice versa.

**Network (net)** -- Isolates the entire network stack: network devices,
IP addresses, routing tables, firewall rules, `/proc/net`, sockets, and more.
Each network namespace gets its own loopback device.

**User** -- Isolates UID/GID number spaces and capabilities. A process can have
`uid 0` (root) inside a user namespace while being an unprivileged user in the
parent namespace. This is the foundation for unprivileged containers.

**Cgroup** -- Isolates the view of the cgroup hierarchy. A process in a new
cgroup namespace sees its current cgroup as the root. This prevents container
processes from seeing the host's cgroup topology.

**Time** -- Isolates `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME`. Allows each
container to have its own notion of system uptime. Does not affect
`CLOCK_REALTIME`.

---

## Core Data Structures

### ns_common -- The Base Namespace Type

Every namespace type embeds `struct ns_common`, which provides the common
interface for reference counting, procfs integration, and namespace
filesystem (nsfs) operations.

```c
/* include/linux/ns_common.h:9 */

struct ns_common {
    struct dentry *stashed;                    /* cached nsfs dentry */
    const struct proc_ns_operations *ops;      /* namespace-type-specific ops */
    unsigned int inum;                         /* inode number (unique ID) */
    refcount_t count;                          /* reference count */
};
```

The `inum` field provides a globally unique identifier for each namespace
instance. It is allocated via `proc_alloc_inum()` from an IDA and serves as
the inode number visible in `/proc/[pid]/ns/` and on the nsfs filesystem.

### proc_ns_operations -- The Namespace Operations Table

Each namespace type registers a `proc_ns_operations` structure that defines
how the namespace is accessed, joined, and queried.

```c
/* include/linux/proc_ns.h:16 */

struct proc_ns_operations {
    const char *name;                                    /* "pid", "net", etc. */
    const char *real_ns_name;                            /* alias resolution */
    int type;                                            /* CLONE_NEW* flag */
    struct ns_common *(*get)(struct task_struct *task);   /* get ns from task */
    void (*put)(struct ns_common *ns);                   /* release reference */
    int (*install)(struct nsset *nsset, struct ns_common *ns); /* join ns */
    struct user_namespace *(*owner)(struct ns_common *ns);     /* owning userns */
    struct ns_common *(*get_parent)(struct ns_common *ns);     /* parent ns */
} __randomize_layout;
```

Each namespace type provides a global instance:

```c
/* include/linux/proc_ns.h:27 */
extern const struct proc_ns_operations netns_operations;
extern const struct proc_ns_operations utsns_operations;
extern const struct proc_ns_operations ipcns_operations;
extern const struct proc_ns_operations pidns_operations;
extern const struct proc_ns_operations pidns_for_children_operations;
extern const struct proc_ns_operations userns_operations;
extern const struct proc_ns_operations mntns_operations;
extern const struct proc_ns_operations cgroupns_operations;
extern const struct proc_ns_operations timens_operations;
extern const struct proc_ns_operations timens_for_children_operations;
```

### Init Namespace Inode Numbers

```c
/* include/linux/proc_ns.h:41 */
enum {
    PROC_ROOT_INO        = 1,
    PROC_IPC_INIT_INO    = 0xEFFFFFFFU,
    PROC_UTS_INIT_INO    = 0xEFFFFFFEU,
    PROC_USER_INIT_INO   = 0xEFFFFFFDU,
    PROC_PID_INIT_INO    = 0xEFFFFFFCU,
    PROC_CGROUP_INIT_INO = 0xEFFFFFFBU,
    PROC_TIME_INIT_INO   = 0xEFFFFFFAU,
};
```

The initial (root) namespaces use these well-known inode numbers. Dynamically
created namespaces receive inode numbers from the IDA allocator.

### nsproxy -- The Per-Task Namespace Aggregator

The `nsproxy` structure groups all non-user namespace pointers for a task.
Tasks that share all namespaces share the same `nsproxy` instance to save
memory.

```c
/* include/linux/nsproxy.h:32 */

struct nsproxy {
    refcount_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net           *net_ns;
    struct time_namespace *time_ns;
    struct time_namespace *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;
};
extern struct nsproxy init_nsproxy;
```

**Important subtleties:**

- `pid_ns_for_children` is the PID namespace that *children* will use, not the
  PID namespace of the current task itself. The task's own PID namespace is
  accessed via `task_active_pid_ns()`, which follows the `struct pid`.

- `time_ns` is the task's current time namespace, while `time_ns_for_children`
  is the one that children will inherit. This split allows `unshare(CLONE_NEWTIME)`
  to affect only future children, not the calling process (since changing time
  offsets for a running process mid-flight is problematic).

- User namespaces are NOT stored in `nsproxy`. They are part of the task's
  credentials (`struct cred`), accessed via `current_user_ns()`.

### nsset -- Transactional Namespace Installation

When joining namespaces via `setns()`, all changes are prepared transactionally
using `struct nsset` and then committed atomically:

```c
/* include/linux/nsproxy.h:65 */

struct nsset {
    unsigned flags;
    struct nsproxy *nsproxy;
    struct fs_struct *fs;
    const struct cred *cred;
};
```

### to_ns_common -- Type-Generic Namespace Access

The kernel provides a `_Generic` macro to get the `ns_common` from any
namespace pointer type:

```c
/* include/linux/nsproxy.h:45 */

#define to_ns_common(__ns)                          \
    _Generic((__ns),                                \
        struct cgroup_namespace *: &(__ns->ns),     \
        struct ipc_namespace *:    &(__ns->ns),     \
        struct net *:              &(__ns->ns),     \
        struct pid_namespace *:    &(__ns->ns),     \
        struct mnt_namespace *:    &(__ns->ns),     \
        struct time_namespace *:   &(__ns->ns),     \
        struct user_namespace *:   &(__ns->ns),     \
        struct uts_namespace *:    &(__ns->ns))
```

### Per-Type Namespace Structures

#### pid_namespace

```c
/* include/linux/pid_namespace.h:26 */

struct pid_namespace {
    struct idr idr;                     /* PID-to-struct-pid mapping */
    struct rcu_head rcu;
    unsigned int pid_allocated;         /* count of allocated PIDs */
    struct task_struct *child_reaper;   /* namespace init (PID 1) */
    struct kmem_cache *pid_cachep;      /* slab cache for struct pid */
    unsigned int level;                 /* nesting depth (0 = root) */
    struct pid_namespace *parent;       /* parent PID namespace */
    struct user_namespace *user_ns;     /* owning user namespace */
    struct ucounts *ucounts;
    int reboot;                         /* exit code if ns was rebooted */
    struct ns_common ns;
} __randomize_layout;

#define MAX_PID_NS_LEVEL 32             /* maximum nesting depth */
```

#### user_namespace

```c
/* include/linux/user_namespace.h:74 */

struct user_namespace {
    struct uid_gid_map uid_map;         /* UID mapping table */
    struct uid_gid_map gid_map;         /* GID mapping table */
    struct uid_gid_map projid_map;      /* project ID mapping table */
    struct user_namespace *parent;      /* parent user namespace */
    int level;                          /* nesting depth */
    kuid_t owner;                       /* uid of creator in parent ns */
    kgid_t group;                       /* gid of creator in parent ns */
    struct ns_common ns;
    unsigned long flags;
    bool parent_could_setfcap;          /* creator had CAP_SETFCAP */
    struct ucounts *ucounts;
    long ucount_max[UCOUNT_COUNTS];    /* per-type resource limits */
    long rlimit_max[UCOUNT_RLIMIT_COUNTS];
} __randomize_layout;
```

#### mnt_namespace

```c
/* fs/mount.h:8 */

struct mnt_namespace {
    struct ns_common    ns;
    struct mount *      root;           /* root mount of this namespace */
    struct rb_root      mounts;         /* rbtree of all mounts */
    struct user_namespace *user_ns;     /* owning user namespace */
    struct ucounts      *ucounts;
    u64                 seq;            /* sequence number */
    wait_queue_head_t   poll;           /* poll for mount changes */
    u64                 event;          /* event counter */
    unsigned int        nr_mounts;      /* number of mounts */
    unsigned int        pending_mounts;
    struct rb_node      mnt_ns_tree_node;
    refcount_t          passive;
} __randomize_layout;
```

#### struct net (network namespace)

```c
/* include/net/net_namespace.h:61 */

struct net {
    refcount_t      passive;            /* for deferred freeing */
    spinlock_t      rules_mod_lock;
    unsigned int    dev_base_seq;
    u32             ifindex;
    spinlock_t      nsid_lock;
    atomic_t        fnhe_genid;
    struct list_head list;              /* global net_namespace_list */
    struct list_head exit_list;         /* for pernet exit callbacks */
    struct user_namespace *user_ns;     /* owning user namespace */
    struct ucounts  *ucounts;
    struct idr      netns_ids;          /* peer ns ID mappings */
    struct ns_common ns;
    struct list_head dev_base_head;     /* list of net_devices */
    struct proc_dir_entry *proc_net;    /* /proc/net */
    struct sock     *rtnl;              /* rtnetlink socket */
    struct net_device *loopback_dev;    /* per-ns loopback */
    struct netns_ipv4 ipv4;             /* per-ns IPv4 state */
    struct netns_ipv6 ipv6;             /* per-ns IPv6 state */
    struct net_generic __rcu *gen;      /* extensible per-ns data */
    struct netns_bpf bpf;               /* per-ns BPF state */
    u64             net_cookie;         /* unique cookie */
    /* ... many more subsystem-specific fields ... */
} __randomize_layout;
```

#### struct pid and struct upid

```c
/* include/linux/pid.h:50 */

struct upid {
    int nr;                             /* PID value in this namespace */
    struct pid_namespace *ns;           /* the namespace */
};

/* include/linux/pid.h:55 */

struct pid {
    refcount_t count;
    unsigned int level;                 /* deepest namespace level */
    spinlock_t lock;
    struct dentry *stashed;
    u64 ino;
    struct hlist_head tasks[PIDTYPE_MAX]; /* tasks using this PID */
    struct hlist_head inodes;
    wait_queue_head_t wait_pidfd;       /* pidfd notification queue */
    struct rcu_head rcu;
    struct upid numbers[];              /* per-level PID values (flex array) */
};
```

The `numbers[]` flex array contains one `struct upid` for each namespace level
from level 0 (the root PID namespace) up to level `pid->level`. This is how a
single kernel-internal `struct pid` maps to different numeric PID values in
different PID namespaces.

---

## Namespace Lifecycle

### Creation via clone() and unshare()

Namespaces are created by passing one or more `CLONE_NEW*` flags to either
`clone()` (creates a child in the new namespace) or `unshare()` (moves the
current task into a new namespace).

```
┌─────────────────────────────────────────────────────────────────┐
│                Namespace Creation Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  clone(CLONE_NEWPID | CLONE_NEWNET)                            │
│    │                                                            │
│    ├─► copy_process()                                           │
│    │     ├─► copy_namespaces()      [kernel/nsproxy.c:151]     │
│    │     │     ├─► ns_capable(CAP_SYS_ADMIN) check             │
│    │     │     └─► create_new_namespaces()  [nsproxy.c:67]     │
│    │     │           ├─► copy_mnt_ns()                          │
│    │     │           ├─► copy_utsname()                         │
│    │     │           ├─► copy_ipcs()                            │
│    │     │           ├─► copy_pid_ns()                          │
│    │     │           ├─► copy_cgroup_ns()                       │
│    │     │           ├─► copy_net_ns()                          │
│    │     │           └─► copy_time_ns()                         │
│    │     └─► child->nsproxy = new_nsp                           │
│    │                                                            │
│  unshare(CLONE_NEWNS)                                          │
│    │                                                            │
│    ├─► ksys_unshare()                                           │
│    │     └─► unshare_nsproxy_namespaces()  [nsproxy.c:213]     │
│    │           └─► create_new_namespaces()                      │
│    └─► switch_task_namespaces(current, new_nsp)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

The `copy_namespaces()` function (`kernel/nsproxy.c:151`) is the entry point
during `clone()`. It follows a fast path for the common case:

```c
/* kernel/nsproxy.c:151 */

int copy_namespaces(unsigned long flags, struct task_struct *tsk)
{
    struct nsproxy *old_ns = tsk->nsproxy;
    struct user_namespace *user_ns = task_cred_xxx(tsk, user_ns);
    struct nsproxy *new_ns;

    if (likely(!(flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
                          CLONE_NEWPID | CLONE_NEWNET |
                          CLONE_NEWCGROUP | CLONE_NEWTIME)))) {
        if ((flags & CLONE_VM) ||
            likely(old_ns->time_ns_for_children == old_ns->time_ns)) {
            get_nsproxy(old_ns);   /* fast path: share existing nsproxy */
            return 0;
        }
    } else if (!ns_capable(user_ns, CAP_SYS_ADMIN))
        return -EPERM;

    new_ns = create_new_namespaces(flags, tsk, user_ns, tsk->fs);
    /* ... error handling ... */
    tsk->nsproxy = new_ns;
    return 0;
}
```

**Key observation:** If no `CLONE_NEW*` flag is set, the child simply
increments the refcount on the parent's `nsproxy` (the fast path). As soon as
any single namespace is cloned, a new `nsproxy` is allocated and all namespace
pointers are copied or replaced.

### Each copy_*_ns() Function

Each per-type `copy_*_ns()` function follows the same pattern:

1. If the corresponding `CLONE_NEW*` flag is not set, take a reference on the
   existing namespace and return it.
2. If the flag is set, allocate a new namespace, initialize it as a copy or
   fresh instance, and return it.

Example for PID namespaces:

```c
/* kernel/pid_namespace.c:146 */

struct pid_namespace *copy_pid_ns(unsigned long flags,
    struct user_namespace *user_ns, struct pid_namespace *old_ns)
{
    if (!(flags & CLONE_NEWPID))
        return get_pid_ns(old_ns);         /* share existing */
    if (task_active_pid_ns(current) != old_ns)
        return ERR_PTR(-EINVAL);
    return create_pid_namespace(user_ns, old_ns);  /* create child */
}
```

### Joining via setns()

`setns()` allows a process to join an existing namespace by passing a file
descriptor to a namespace file (from `/proc/[pid]/ns/` or a bind mount of one).
It can also accept a pidfd to join multiple namespaces of another process at
once.

```c
/* kernel/nsproxy.c:546 */

SYSCALL_DEFINE2(setns, int, fd, int, flags)
{
    /* ... */
    if (proc_ns_file(fd_file(f))) {
        ns = get_proc_ns(file_inode(fd_file(f)));
        /* ... validate type matches flags ... */
        flags = ns->ops->type;
    } else if (!IS_ERR(pidfd_pid(fd_file(f)))) {
        err = check_setns_flags(flags);   /* pidfd: join multiple */
    }

    err = prepare_nsset(flags, &nsset);  /* prepare transactional nsset */
    /* ... */
    err = validate_ns(&nsset, ns);       /* call install() for each ns */
    if (!err)
        commit_nsset(&nsset);            /* atomic switch */
    /* ... */
}
```

The process is transactional: `prepare_nsset()` creates a temporary copy,
`validate_ns()` installs each requested namespace into the temporary copy, and
only on success does `commit_nsset()` atomically switch the task's nsproxy
using `switch_task_namespaces()`.

### Destruction

Namespace destruction is reference-count driven. When the last reference to an
`ns_common` drops, the namespace is freed. Different namespace types handle this
differently:

- **Simple namespaces (UTS, IPC, cgroup):** freed immediately via slab cache.
- **PID namespaces:** use `call_rcu()` for deferred freeing, and chain up to
  free parent namespaces if their refcount also reaches zero.
- **User namespaces:** schedule a work item because freeing can trigger
  complex cleanup (keyrings, sysctls, ID map memory).
- **Network namespaces:** use a dedicated `netns_wq` workqueue because
  cleanup requires calling `pernet_operations` exit callbacks, which may
  sleep.

```c
/* kernel/nsproxy.c:254 */
void exit_task_namespaces(struct task_struct *p)
{
    switch_task_namespaces(p, NULL);  /* sets nsproxy to NULL, puts old */
}
```

---

## The nsproxy Mechanism

### How Tasks Reference Namespaces

Each task's `task_struct` has a `nsproxy` pointer that holds references to
all namespaces except the user namespace (which lives in `struct cred`).

```
┌─────────────────────────────────────────────────────────────────┐
│            Task to Namespace Reference Model                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  task_struct                                                    │
│  ├── nsproxy ──────────► struct nsproxy                         │
│  │                       ├── count (refcount)                   │
│  │                       ├── uts_ns ──────► uts_namespace        │
│  │                       ├── ipc_ns ──────► ipc_namespace        │
│  │                       ├── mnt_ns ──────► mnt_namespace        │
│  │                       ├── pid_ns_for_children ─► pid_namespace│
│  │                       ├── net_ns ──────► struct net            │
│  │                       ├── time_ns ─────► time_namespace       │
│  │                       ├── time_ns_for_children ─► time_ns    │
│  │                       └── cgroup_ns ──► cgroup_namespace     │
│  │                                                              │
│  ├── cred ────────────► struct cred                              │
│  │                       └── user_ns ────► user_namespace        │
│  │                                                              │
│  └── thread_pid ──────► struct pid                               │
│                          └── numbers[level].ns ─► pid_namespace  │
│                             (task's ACTIVE pid ns)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Access Rules

The kernel enforces strict rules for accessing `nsproxy`, documented in
`include/linux/nsproxy.h:80`:

1. Only the **current task** may change its own `tsk->nsproxy` pointer or any
   pointer within the nsproxy. It must hold `task_lock` when changing
   `tsk->nsproxy`.

2. Reading the **current task's** namespaces requires no locking -- just
   dereference the pointers.

3. Reading **another task's** namespaces requires `task_lock`:

```c
/* Access pattern for reading another task's namespaces */
task_lock(task);
nsproxy = task->nsproxy;
if (nsproxy != NULL) {
    /* work with the namespaces, e.g. get a reference */
} /* NULL nsproxy means the task is almost dead (zombie) */
task_unlock(task);
```

### nsproxy Sharing

The `nsproxy` is shared by tasks that share **all** namespaces. As soon as a
single namespace is cloned or unshared, the entire nsproxy is copied. This
sharing optimization means the refcount on `nsproxy` reflects the number of
tasks sharing that exact combination of namespaces, not the number of tasks
in any individual namespace.

### The init_nsproxy

```c
/* kernel/nsproxy.c:32 */

struct nsproxy init_nsproxy = {
    .count              = REFCOUNT_INIT(1),
    .uts_ns             = &init_uts_ns,
    .ipc_ns             = &init_ipc_ns,
    .mnt_ns             = NULL,
    .pid_ns_for_children = &init_pid_ns,
    .net_ns             = &init_net,
    .cgroup_ns          = &init_cgroup_ns,
    .time_ns            = &init_time_ns,
    .time_ns_for_children = &init_time_ns,
};
```

Note that `mnt_ns` is `NULL` initially -- the initial mount namespace is set up
later during boot by `mnt_init()`.

### switch_task_namespaces()

This is the core function for atomically switching a task's nsproxy:

```c
/* kernel/nsproxy.c:239 */

void switch_task_namespaces(struct task_struct *p, struct nsproxy *new)
{
    struct nsproxy *ns;

    might_sleep();

    task_lock(p);
    ns = p->nsproxy;
    p->nsproxy = new;
    task_unlock(p);

    if (ns)
        put_nsproxy(ns);
}
```

### exec and Namespaces

On `exec()`, the kernel checks if the time namespace for children differs from
the current time namespace. If so, a new nsproxy is created and the task
transitions into its `time_ns_for_children`:

```c
/* kernel/nsproxy.c:259 */

int exec_task_namespaces(void)
{
    struct task_struct *tsk = current;

    if (tsk->nsproxy->time_ns_for_children == tsk->nsproxy->time_ns)
        return 0;

    /* Create new nsproxy and switch to time_ns_for_children */
    /* ... */
}
```

---

## User Namespaces and the Capability/Permission Model

User namespaces are the most consequential namespace type. They provide the
security boundary that allows unprivileged users to create and use other
namespace types.

### Creation

```c
/* kernel/user_namespace.c:82 */

int create_user_ns(struct cred *new)
{
    struct user_namespace *ns, *parent_ns = new->user_ns;
    kuid_t owner = new->euid;
    kgid_t group = new->egid;

    /* Maximum nesting depth of 32 */
    if (parent_ns->level > 32)
        goto fail;

    /* Cannot create user ns from a chroot */
    if (current_chrooted())
        goto fail_dec;

    /* Creator must have a mapping in the parent */
    if (!kuid_has_mapping(parent_ns, owner) ||
        !kgid_has_mapping(parent_ns, group))
        goto fail_dec;

    /* LSM hook */
    ret = security_create_user_ns(new);

    /* Allocate and initialize */
    ns = kmem_cache_zalloc(user_ns_cachep, GFP_KERNEL);
    ns->parent = parent_ns;
    ns->level = parent_ns->level + 1;
    ns->owner = owner;
    ns->group = group;

    /* Grant full capabilities within the new namespace */
    set_cred_user_ns(new, ns);
    return 0;
}
```

### The Capability Model

When a new user namespace is created, the creator's credentials are updated
to grant full capabilities -- but only within the scope of that namespace:

```c
/* kernel/user_namespace.c:43 */

static void set_cred_user_ns(struct cred *cred, struct user_namespace *user_ns)
{
    cred->securebits = SECUREBITS_DEFAULT;
    cred->cap_inheritable = CAP_EMPTY_SET;
    cred->cap_permitted  = CAP_FULL_SET;
    cred->cap_effective  = CAP_FULL_SET;
    cred->cap_ambient    = CAP_EMPTY_SET;
    cred->cap_bset       = CAP_FULL_SET;
    cred->user_ns = user_ns;
}
```

The `ns_capable()` function checks whether a task has a capability relative to
a specific user namespace. A capability is valid in a namespace only if:

1. The task has that capability in its `cap_effective` set, AND
2. The task's user namespace is the target namespace or an ancestor of it.

This means root in a child user namespace cannot affect resources owned by a
parent or sibling user namespace.

### UID/GID Mapping

User namespaces maintain UID and GID mapping tables that translate between
namespace-local IDs and kernel-internal IDs (which correspond to parent
namespace IDs).

```c
/* include/linux/user_namespace.h:17 */

struct uid_gid_extent {
    u32 first;          /* first ID in the namespace */
    u32 lower_first;    /* first ID in the parent namespace */
    u32 count;          /* number of IDs in this mapping extent */
};

struct uid_gid_map {
    union {
        struct {
            struct uid_gid_extent extent[UID_GID_MAP_MAX_BASE_EXTENTS]; /* 5 */
            u32 nr_extents;
        };
        struct {
            struct uid_gid_extent *forward;   /* sorted by first */
            struct uid_gid_extent *reverse;   /* sorted by lower_first */
        };
    };
};
```

For up to 5 extents, the mapping is stored inline (one cache line). For more
extents (up to 340), dynamically allocated sorted arrays are used with binary
search for O(log n) lookups.

**Translation functions:**

- `map_id_down()` -- translates namespace ID to kernel-global ID (used by
  `make_kuid()`, `make_kgid()`)
- `map_id_up()` -- translates kernel-global ID to namespace ID (used by
  `from_kuid()`, `from_kgid()`)

Mappings are written once to `/proc/[pid]/uid_map` and `/proc/[pid]/gid_map`
(handled by `map_write()` at `kernel/user_namespace.c:922`). They are
immutable after being set.

### setgroups Control

To prevent privilege escalation, the `setgroups` system call can be
permanently disabled in a user namespace by writing "deny" to
`/proc/[pid]/setgroups`. This must be done before writing the GID map.

### Resource Accounting

User namespaces track resource usage via `struct ucounts`:

```c
/* include/linux/user_namespace.h:42 */

enum ucount_type {
    UCOUNT_USER_NAMESPACES,
    UCOUNT_PID_NAMESPACES,
    UCOUNT_UTS_NAMESPACES,
    UCOUNT_IPC_NAMESPACES,
    UCOUNT_NET_NAMESPACES,
    UCOUNT_MNT_NAMESPACES,
    UCOUNT_CGROUP_NAMESPACES,
    UCOUNT_TIME_NAMESPACES,
    /* ... inotify, fanotify ... */
    UCOUNT_COUNTS,
};
```

Each user namespace can set limits on how many of each namespace type (and
other resources like inotify watches) can be created within it.

---

## PID Namespace Nesting and PID Translation

PID namespaces form a strict hierarchy. A process is visible in its own PID
namespace and all ancestor PID namespaces, but has a different PID number in
each.

### Hierarchical PID Allocation

```
┌─────────────────────────────────────────────────────────────────┐
│              PID Namespace Hierarchy                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Root PID NS (level 0)                                         │
│  ├── PID 1 (systemd)                                           │
│  ├── PID 1234 ─────────────────────────────────┐               │
│  │                                              │               │
│  │   Child PID NS (level 1)                     │               │
│  │   ├── PID 1 (container init) ←── PID 1234   │               │
│  │   ├── PID 42 ←──────────────── PID 5678      │               │
│  │   │                                          │               │
│  │   │   Grandchild PID NS (level 2)            │               │
│  │   │   ├── PID 1 ←── PID 42 ←── PID 5678     │               │
│  │   │   └── PID 7 ←── PID 43 ←── PID 5679     │               │
│  │   │                                          │               │
│  │   └── PID 43 ←──────────────── PID 5679      │               │
│  │                                              │               │
│  └──────────────────────────────────────────────┘               │
│                                                                 │
│  A process at level 2 has THREE PID values:                    │
│    numbers[0].nr = 5678  (root namespace value)                │
│    numbers[1].nr = 42    (level 1 namespace value)             │
│    numbers[2].nr = 1     (level 2 namespace value)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### struct pid and the upid Array

When a process is created in a PID namespace at level N, `alloc_pid()`
allocates a `struct pid` with N+1 `struct upid` entries:

```c
/* include/linux/pid.h:55 */

struct pid {
    refcount_t count;
    unsigned int level;         /* deepest namespace level */
    /* ... */
    struct upid numbers[];      /* one entry per namespace level */
};
```

Each `struct upid` records the PID value within a specific namespace:

```c
struct upid {
    int nr;                     /* PID value in this namespace */
    struct pid_namespace *ns;   /* the namespace */
};
```

### PID Translation Functions

```c
/* include/linux/pid.h:175 */

/* Global PID (as seen from init_pid_ns) */
static inline pid_t pid_nr(struct pid *pid)
{
    pid_t nr = 0;
    if (pid)
        nr = pid->numbers[0].nr;
    return nr;
}

/* PID as seen from a specific namespace */
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns);

/* PID as seen from current's namespace */
pid_t pid_vnr(struct pid *pid);

/* Get the namespace a pid was allocated in */
static inline struct pid_namespace *ns_of_pid(struct pid *pid)
{
    struct pid_namespace *ns = NULL;
    if (pid)
        ns = pid->numbers[pid->level].ns;
    return ns;
}
```

The `pid_nr_ns()` function returns 0 if the target namespace is deeper (higher
level) than the process's namespace, enforcing the rule that a parent namespace
cannot be seen from a child.

### The child_reaper

Each PID namespace has a `child_reaper` -- the PID 1 process within that
namespace. This process inherits orphaned children within the namespace (just
as the global init inherits orphans system-wide). If the child_reaper exits,
all processes in the PID namespace are killed.

```c
/* kernel/pid_namespace.c:170 */

void zap_pid_ns_processes(struct pid_namespace *pid_ns)
{
    /* Don't allow any more processes into the pid namespace */
    disable_pid_allocation(pid_ns);

    /* Signal SIGKILL to every remaining process */
    idr_for_each_entry_continue(&pid_ns->idr, pid, nr) {
        task = pid_task(pid, PIDTYPE_PID);
        if (task && !__fatal_signal_pending(task))
            group_send_sig_info(SIGKILL, SEND_SIG_PRIV, task, PIDTYPE_MAX);
    }

    /* Wait for all processes to exit */
    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE);
        if (pid_ns->pid_allocated == init_pids)
            break;
        schedule();
    }
}
```

### reboot() in PID Namespaces

`reboot()` from within a non-init PID namespace sends `SIGKILL` to the
namespace's init process rather than rebooting the machine:

```c
/* kernel/pid_namespace.c:297 */

int reboot_pid_ns(struct pid_namespace *pid_ns, int cmd)
{
    if (pid_ns == &init_pid_ns)
        return 0;   /* let normal reboot handle it */

    switch (cmd) {
    case LINUX_REBOOT_CMD_RESTART:
        pid_ns->reboot = SIGHUP;
        break;
    case LINUX_REBOOT_CMD_HALT:
        pid_ns->reboot = SIGINT;
        break;
    }

    send_sig(SIGKILL, pid_ns->child_reaper, 1);
    do_exit(0);
}
```

---

## Network Namespace Internals

Network namespaces are the most complex namespace type due to the vast amount
of state in the networking stack.

### Structure Overview

Each network namespace is represented by `struct net` (`include/net/net_namespace.h:61`),
which contains per-namespace state for every network subsystem: devices,
routing, netfilter, sockets, and more.

### pernet_operations -- The Subsystem Registration Framework

Network subsystems register init/exit callbacks to be called whenever a
network namespace is created or destroyed:

```c
/* include/net/net_namespace.h:433 */

struct pernet_operations {
    struct list_head list;
    int (*init)(struct net *net);            /* called on ns creation */
    void (*pre_exit)(struct net *net);       /* called before exit */
    void (*exit)(struct net *net);           /* called on ns destruction */
    void (*exit_batch)(struct list_head *net_exit_list);
    void (*exit_batch_rtnl)(struct list_head *net_exit_list,
                            struct list_head *dev_kill_list);
    unsigned int * const id;                /* for net_generic storage */
    const size_t size;                      /* size of private data */
};
```

There are two categories of registration:

```c
int register_pernet_subsys(struct pernet_operations *);   /* subsystem data */
int register_pernet_device(struct pernet_operations *);   /* device data */
```

**Device vs subsystem distinction:** Network devices must be cleaned up before
subsystem notifiers run. The `first_device` pointer in the `pernet_list`
separates subsystem operations (before the marker) from device operations
(after the marker). During namespace teardown, device exit callbacks run first.

### copy_net_ns

```c
/* net/core/net_namespace.c:488 */

struct net *copy_net_ns(unsigned long flags,
                        struct user_namespace *user_ns, struct net *old_net)
{
    if (!(flags & CLONE_NEWNET))
        return get_net(old_net);         /* share existing */

    ucounts = inc_net_namespaces(user_ns);
    net = net_alloc();
    preinit_net(net, user_ns);

    rv = setup_net(net);                 /* calls all pernet init() callbacks */
    /* ... */
}
```

### setup_net -- The Namespace Initialization Chain

```c
/* net/core/net_namespace.c:349 */

static int setup_net(struct net *net)
{
    list_for_each_entry(ops, &pernet_list, list) {
        error = ops_init(ops, net);     /* call each subsystem's init() */
        if (error < 0)
            goto out_undo;
    }
    /* Add to global namespace list */
    list_add_tail_rcu(&net->list, &net_namespace_list);
}
```

This iterates through all registered `pernet_operations` and calls their
`init()` callback, which sets up per-namespace state (creating a loopback
device, initializing routing tables, setting up netfilter, etc.).

### net_generic -- Extensible Per-Namespace Storage

Subsystems that need per-namespace private data use the `net_generic` mechanism.
They declare a static `unsigned int` ID and register with a `size`:

```c
static unsigned int my_subsys_net_id;

static struct pernet_operations my_ops = {
    .init = my_init,
    .exit = my_exit,
    .id   = &my_subsys_net_id,
    .size = sizeof(struct my_per_ns_data),
};
```

The framework allocates the data during `ops_init()` and stores a pointer in
`net->gen->ptr[id]`. Access is via `net_generic(net, my_subsys_net_id)`.

### Namespace ID Tracking

Network namespaces can assign integer IDs to peer namespaces for use in
netlink messages (e.g., to refer to a target namespace in `RTM_NEWLINK`):

```c
int peernet2id_alloc(struct net *net, struct net *peer, gfp_t gfp);
int peernet2id(const struct net *net, struct net *peer);
struct net *get_net_ns_by_id(const struct net *net, int id);
```

### Global Namespace List

All network namespaces are linked in `net_namespace_list`, protected by
`net_rwsem`:

```c
/* net/core/net_namespace.c:37 */
LIST_HEAD(net_namespace_list);

#define for_each_net(VAR)           \
    list_for_each_entry(VAR, &net_namespace_list, list)
```

---

## Key Syscalls

### clone() / clone3()

```
clone(flags)
  └─► copy_process()
        └─► copy_namespaces(flags, child)
              └─► create_new_namespaces() if any CLONE_NEW* set
```

When `CLONE_NEWUSER` is set, the user namespace is created during
`copy_creds()` (before `copy_namespaces()`), because the new user namespace
is needed as the owner for the other namespaces being created.

### unshare()

```
unshare(flags)
  ├─► unshare_userns()               if CLONE_NEWUSER
  ├─► unshare_nsproxy_namespaces()   for all other ns types
  └─► switch_task_namespaces()       commit the new nsproxy
```

Unlike `clone()`, `unshare()` moves the calling task into a new namespace
rather than creating a child. PID namespace unshare is special: it only
affects `pid_ns_for_children`, meaning the calling process stays in its
current PID namespace, but children will be in the new one.

### setns()

```c
/* kernel/nsproxy.c:546 */

SYSCALL_DEFINE2(setns, int, fd, int, flags)
```

`setns()` joins an existing namespace. It accepts two forms of file descriptor:

1. **Namespace file descriptor** (from `/proc/[pid]/ns/*` or a bind mount):
   joins that specific namespace. The `flags` parameter must be 0 or match
   the namespace type.

2. **pidfd** (a file descriptor referring to a process): joins one or more of
   that process's namespaces. The `flags` parameter specifies which namespace
   types to join (a bitmask of `CLONE_NEW*` flags).

The implementation uses a two-phase commit:

```
setns(fd, flags)
  ├─► prepare_nsset()      allocate temporary nsproxy
  ├─► validate_ns()        install requested namespaces (in order:
  │     ├── user            user, mount, uts, ipc, pid, cgroup, net, time)
  │     ├── mount
  │     ├── uts
  │     ├── ipc
  │     ├── pid
  │     ├── cgroup
  │     ├── net
  │     └── time
  ├─► commit_nsset()       atomically switch if all validations pass
  │     ├── commit_creds()         if CLONE_NEWUSER
  │     ├── set_fs_root/pwd()      if CLONE_NEWNS
  │     ├── exit_sem()             if CLONE_NEWIPC
  │     ├── timens_commit()        if CLONE_NEWTIME
  │     └── switch_task_namespaces()
  └─► put_nsset()          cleanup
```

### ioctl_ns (NS_GET_* ioctls)

The nsfs filesystem supports ioctl operations on namespace file descriptors
(`fs/nsfs.c:155`):

```
NS_GET_USERNS          Get the owning user namespace (returns fd)
NS_GET_PARENT          Get the parent namespace (returns fd)
NS_GET_NSTYPE          Get the namespace type (returns CLONE_NEW* value)
NS_GET_OWNER_UID       Get the UID of the user namespace owner
NS_GET_MNTNS_ID        Get the mount namespace sequence ID
NS_GET_PID_FROM_PIDNS  Translate PID from the target PID namespace
NS_GET_TGID_FROM_PIDNS Translate TGID from the target PID namespace
NS_GET_PID_IN_PIDNS    Translate PID into the target PID namespace
NS_GET_TGID_IN_PIDNS   Translate TGID into the target PID namespace
NS_MNT_GET_INFO        Get mount namespace info (struct mnt_ns_info)
NS_MNT_GET_NEXT        Get next mount namespace (returns fd)
NS_MNT_GET_PREV        Get previous mount namespace (returns fd)
```

These ioctls enable namespace introspection and navigation from userspace.

---

## Integration with Other Subsystems

### procfs Integration

Each process exposes its namespace memberships via `/proc/[pid]/ns/`:

```
/proc/[pid]/ns/
├── cgroup    -> cgroup:[inum]
├── ipc       -> ipc:[inum]
├── mnt       -> mnt:[inum]
├── net       -> net:[inum]
├── pid       -> pid:[inum]
├── pid_for_children -> pid:[inum]
├── time      -> time:[inum]
├── time_for_children -> time:[inum]
├── user      -> user:[inum]
└── uts       -> uts:[inum]
```

Each entry is a symlink whose target encodes the namespace type and inode
number. Opening these files returns a file descriptor on the nsfs filesystem,
which can be used with `setns()` or the `NS_GET_*` ioctls.

Bind-mounting these files (e.g., `mount --bind /proc/1/ns/net /var/run/netns/foo`)
keeps the namespace alive even after all processes have exited, because the
mount holds a reference on the nsfs inode which holds a reference on the
`ns_common`.

### The nsfs Filesystem

The namespace filesystem (nsfs) is an internal pseudo-filesystem that provides
file-based handles to namespaces:

```c
/* fs/nsfs.c:403 */

static struct file_system_type nsfs = {
    .name = "nsfs",
    .init_fs_context = nsfs_init_fs_context,
    .kill_sb = kill_anon_super,
};
```

Each namespace instance can have a stashed dentry (`ns_common.stashed`) that
is created on first access and cached for subsequent opens. The nsfs inode's
`i_private` points to the `ns_common`, and its `i_ino` matches `ns_common.inum`.

When the dentry is pruned (no more references), `nsfs_evict()` calls the
namespace type's `put()` operation to release the namespace reference.

### Cgroups Integration

Namespaces interact with cgroups in several ways:

- The **cgroup namespace** (`CLONE_NEWCGROUP`) virtualizes the cgroup
  filesystem view. A process in a cgroup namespace sees its own cgroup as the
  root. This prevents container processes from seeing the host cgroup hierarchy.

- **Resource accounting** for namespace creation uses `ucounts` which are
  tracked per user namespace, allowing limits on how many namespaces a user can
  create.

- The **PID namespace** uses the cgroup-aware `child_reaper` to manage process
  reaping within containers.

### Security/LSM Integration

Linux Security Modules have hooks at key namespace operations:

- **`security_create_user_ns()`** -- called during user namespace creation.
  This allows LSMs (AppArmor, SELinux) to restrict which processes can create
  user namespaces.

- **`ns_capable()`** -- the capability check that gates most namespace
  operations. It verifies that a task has a capability in the context of a
  specific user namespace.

- The `ptrace_may_access()` check in `validate_nsset()` ensures that joining
  another process's namespaces via a pidfd requires appropriate access rights.

- SELinux provides `SECCLASS_USER_NAMESPACE` for controlling user namespace
  creation policy.

### BPF Integration

BPF interacts with namespaces at multiple levels:

- **Network namespace awareness:** BPF programs attached to network events
  (XDP, TC, socket filters) execute in the context of a specific network
  namespace. The `struct net` contains a `struct netns_bpf` for per-namespace
  BPF state.

- **BPF namespace-aware helpers:** Functions like `bpf_get_netns_cookie()`
  return the network namespace cookie, allowing BPF programs to distinguish
  traffic from different namespaces.

- **Cgroup-BPF programs:** These can be associated with cgroups that span
  namespace boundaries, providing policy enforcement across containers.

---

## Source File Reference

```
Core Infrastructure:
  include/linux/nsproxy.h          nsproxy structure definition, access rules
  include/linux/ns_common.h        ns_common base structure
  include/linux/proc_ns.h          proc_ns_operations, inode numbers
  kernel/nsproxy.c                 nsproxy management, setns(), copy_namespaces()
  fs/nsfs.c                        namespace filesystem, NS_GET_* ioctls

PID Namespace:
  include/linux/pid_namespace.h    pid_namespace structure
  include/linux/pid.h              struct pid, struct upid, PID translation
  kernel/pid_namespace.c           PID namespace creation/destruction, reboot

User Namespace:
  include/linux/user_namespace.h   user_namespace, uid_gid_map, ucounts
  kernel/user_namespace.c          UID/GID mapping, capability model

Network Namespace:
  include/net/net_namespace.h      struct net, pernet_operations
  net/core/net_namespace.c         copy_net_ns(), setup_net(), pernet framework

Mount Namespace:
  fs/mount.h                       mnt_namespace, struct mount
  fs/namespace.c                   mount namespace operations

Clone Flags:
  include/uapi/linux/sched.h       CLONE_NEW* flag definitions
```
