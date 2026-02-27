# Linux Kernel Audit Subsystem

## Overview

The Linux Audit subsystem provides a framework for recording security-relevant
events in the kernel and delivering them to user-space. It acts as a gateway
between kernel components (syscall paths, LSMs, filesystem layer) and the
user-space audit daemon (`auditd`). The subsystem supports configurable
filtering rules so that administrators can selectively record events based on
syscall number, user identity, file path, and other criteria. Communication
between kernel and user space uses the Netlink protocol (`NETLINK_AUDIT`),
with a dedicated kernel thread (`kauditd`) managing message delivery.

The implementation is split across several files:

- `kernel/audit.c` -- core infrastructure, netlink interface, kauditd thread
- `kernel/auditsc.c` -- syscall-specific auditing (entry/exit hooks, filtering)
- `kernel/auditfilter.c` -- filter rule management (add/delete/list)
- `kernel/audit_watch.c` -- inode-based file watches via fsnotify
- `kernel/audit_tree.c` -- directory tree watches via fsnotify
- `kernel/audit.h` -- private header with `audit_context` and internal types
- `include/linux/audit.h` -- public kernel API
- `include/uapi/linux/audit.h` -- user-kernel ABI (message types, rule format)

---

## 1. Architecture Overview -- Security Event Recording

The audit subsystem follows a deferred-work design to minimize runtime
overhead. The key design goals are stated in `kernel/audit.c:11-26`:

1. Full integration with Linux Security Modules (SELinux, AppArmor, etc.)
2. Minimal overhead when auditing is disabled (`audit_enable=0`)
3. Deferred work: context is allocated early but record generation is postponed
   to syscall exit time
4. Kernel-based filtering to reduce data sent to user space
5. Netlink interface to user space

The high-level data flow is:

```
  Task enters syscall
       |
       v
  __audit_syscall_entry()  -- save syscall number, args, timestamp
       |
       v
  VFS/security hooks collect names, inodes, security contexts
       |
       v
  __audit_syscall_exit()   -- apply exit filters, generate records
       |
       v
  audit_log_start() / audit_log_format() / audit_log_end()
       |                              -- format records into skbs
       v
  audit_queue  (sk_buff_head)
       |
       v
  kauditd_thread()         -- deliver via netlink unicast + multicast
       |
       v
  auditd (user space)      -- writes to log files
```

Three global variables control the subsystem state:

```c
// kernel/audit.c:71-72
u32    audit_enabled = AUDIT_OFF;    // AUDIT_OFF(0), AUDIT_ON(1), AUDIT_LOCKED(2)
bool   audit_ever_enabled = !!AUDIT_OFF;
```

When `audit_enabled` is `AUDIT_LOCKED` (2), configuration changes are
forbidden until the next reboot.

---

## 2. Core Data Structures

### `struct audit_context`

`kernel/audit.h:109`

The per-task audit context is the central data structure. It accumulates all
information for a single auditable event (syscall or io_uring operation):

```c
struct audit_context {
    int                 dummy;          /* must be first element */
    enum {
        AUDIT_CTX_UNUSED,               /* currently unused */
        AUDIT_CTX_SYSCALL,              /* in use by syscall */
        AUDIT_CTX_URING,                /* in use by io_uring */
    } context;
    enum audit_state    state, current_state;
    struct audit_stamp  stamp;          /* event identifier (time + serial) */
    int                 major;          /* syscall number */
    int                 uring_op;       /* uring operation */
    unsigned long       argv[4];        /* syscall arguments */
    long                return_code;    /* syscall return code */
    u64                 prio;
    int                 return_valid;   /* return code is valid */

    struct audit_names  preallocated_names[AUDIT_NAMES]; /* 5 preallocated */
    int                 name_count;
    struct list_head    names_list;     /* all audit_names for this event */
    char                *filterkey;     /* key for rule that triggered record */
    struct path         pwd;
    struct audit_aux_data *aux;
    struct audit_aux_data *aux_pids;
    struct sockaddr_storage *sockaddr;
    size_t              sockaddr_len;

    /* Task identity snapshot */
    pid_t               ppid;
    kuid_t              uid, euid, suid, fsuid;
    kgid_t              gid, egid, sgid, fsgid;
    unsigned long       personality;
    int                 arch;

    /* Target process info (for ptrace, signals) */
    pid_t               target_pid;
    kuid_t              target_auid, target_uid;
    unsigned int        target_sessionid;
    struct lsm_prop     target_ref;
    char                target_comm[TASK_COMM_LEN];

    /* Directory tree watch references */
    struct audit_tree_refs *trees, *first_trees;
    struct list_head    killed_trees;
    int                 tree_count;

    /* Type-specific data (union for different syscall families) */
    int type;
    union {
        struct { int nargs; long args[6]; } socketcall;
        struct { kuid_t uid; kgid_t gid; umode_t mode; ... } ipc;
        struct { mqd_t mqdes; struct mq_attr mqstat; } mq_getsetattr;
        struct { pid_t pid; struct audit_cap_data cap; } capset;
        struct { int fd; int flags; } mmap;
        struct open_how openat2;
        struct { int argc; } execve;
        struct { const char *name; } module;
        ...
    };
    int fds[2];
    struct audit_proctitle proctitle;
};
```

The `dummy` field is required to be the first element. `audit_dummy_context()`
checks this field to determine if the context is active:

```c
// include/linux/audit.h:342
static inline bool audit_dummy_context(void)
{
    void *p = audit_context();
    return !p || *(int *)p;
}
```

### `struct audit_buffer`

`kernel/audit.c:205`

Used when formatting an audit record. Multiple buffers can be in use
simultaneously:

```c
struct audit_buffer {
    struct sk_buff       *skb;      /* the skb for audit_log functions */
    struct sk_buff_head  skb_list;  /* formatted skbs, ready to send */
    struct audit_context *ctx;      /* NULL or associated context */
    struct audit_stamp   stamp;     /* audit stamp for these records */
    gfp_t                gfp_mask;
};
```

Buffers are allocated from a dedicated slab cache (`audit_buffer_cache`) and
use `AUDIT_BUFSIZ` (1024 bytes) as the initial formatting buffer size
(`kernel/audit.c:198`).

### `struct audit_rule_data`

`include/uapi/linux/audit.h:518`

The user-kernel ABI structure for passing filter rules:

```c
struct audit_rule_data {
    __u32   flags;                          /* AUDIT_FILTER_* */
    __u32   action;                         /* AUDIT_NEVER, AUDIT_POSSIBLE, AUDIT_ALWAYS */
    __u32   field_count;
    __u32   mask[AUDIT_BITMASK_SIZE];       /* syscall(s) affected (64 u32s) */
    __u32   fields[AUDIT_MAX_FIELDS];       /* field types (up to 64) */
    __u32   values[AUDIT_MAX_FIELDS];       /* field values */
    __u32   fieldflags[AUDIT_MAX_FIELDS];   /* comparison operators */
    __u32   buflen;                         /* total length of string fields */
    char    buf[];                          /* string fields buffer */
};
```

### `struct audit_krule`

`include/linux/audit.h:43`

The kernel-internal representation of a filter rule:

```c
struct audit_krule {
    u32                 pflags;
    u32                 flags;
    u32                 listnr;
    u32                 action;
    u32                 mask[AUDIT_BITMASK_SIZE];
    u32                 buflen;
    u32                 field_count;
    char                *filterkey;     /* ties events to rules */
    struct audit_field  *fields;
    struct audit_field  *arch_f;        /* quick access to arch field */
    struct audit_field  *inode_f;       /* quick access to inode field */
    struct audit_watch  *watch;         /* associated watch */
    struct audit_tree   *tree;          /* associated watched tree */
    struct audit_fsnotify_mark *exe;
    struct list_head    rlist;          /* entry in watch/tree rules list */
    struct list_head    list;           /* for AUDIT_LIST* purposes */
    u64                 prio;
};
```

### `struct audit_entry`

`kernel/audit.h:50`

Wraps `audit_krule` for list management and RCU-safe freeing:

```c
struct audit_entry {
    struct list_head    list;
    struct rcu_head     rcu;
    struct audit_krule  rule;
};
```

### `struct audit_names`

`kernel/audit.h:72`

Records file path and inode information collected during syscall processing:

```c
struct audit_names {
    struct list_head    list;           /* audit_context->names_list */
    struct filename     *name;
    int                 name_len;
    bool                hidden;
    unsigned long       ino;
    dev_t               dev;
    umode_t             mode;
    kuid_t              uid;
    kgid_t              gid;
    dev_t               rdev;
    struct lsm_prop     oprop;         /* object security properties */
    struct audit_cap_data fcap;
    unsigned int        fcap_ver;
    unsigned char       type;          /* AUDIT_TYPE_NORMAL, _PARENT, etc. */
    bool                should_free;
};
```

### `struct audit_stamp`

`kernel/audit.h:103`

A timestamp/serial pair that uniquely identifies an event:

```c
struct audit_stamp {
    struct timespec64   ctime;          /* time of syscall entry */
    unsigned int        serial;         /* serial number for record */
};
```

---

## 3. Audit Message Types and Record Format

### Message Type Ranges

Defined in `include/uapi/linux/audit.h:31-52`:

| Range       | Purpose                                    |
|-------------|--------------------------------------------|
| 1000 - 1099 | Command/control (bidirectional)             |
| 1100 - 1199 | User-space trusted application messages     |
| 1200 - 1299 | Audit daemon internal messages              |
| 1300 - 1399 | Kernel audit event messages                 |
| 1400 - 1499 | LSM access control messages (AVC, etc.)     |
| 1500 - 1599 | Kernel LSPP events                          |
| 1600 - 1699 | Kernel crypto events                        |
| 1700 - 1799 | Kernel anomaly records                      |
| 1800 - 1899 | Kernel integrity events (IMA/EVM)           |
| 2000        | Generic kernel audit message (legacy)       |
| 2100 - 2999 | User-space anomaly/response/LSPP/crypto     |

### Key Command Messages (1000-series)

```c
#define AUDIT_GET           1000    /* Get audit status */
#define AUDIT_SET           1001    /* Set status (enable/disable/auditd) */
#define AUDIT_ADD_RULE      1011    /* Add syscall filtering rule */
#define AUDIT_DEL_RULE      1012    /* Delete syscall filtering rule */
#define AUDIT_LIST_RULES    1013    /* List syscall filtering rules */
#define AUDIT_SET_FEATURE   1018    /* Turn an audit feature on or off */
#define AUDIT_GET_FEATURE   1019    /* Get which features are enabled */
#define AUDIT_SIGNAL_INFO   1010    /* Get info about signal sender */
```

### Key Event Record Types (1300-series)

```c
#define AUDIT_SYSCALL       1300    /* Syscall event */
#define AUDIT_PATH          1302    /* Filename path information */
#define AUDIT_IPC           1303    /* IPC record */
#define AUDIT_SOCKETCALL    1304    /* sys_socketcall arguments */
#define AUDIT_CONFIG_CHANGE 1305    /* Audit system configuration change */
#define AUDIT_SOCKADDR      1306    /* sockaddr copied as syscall arg */
#define AUDIT_CWD           1307    /* Current working directory */
#define AUDIT_EXECVE        1309    /* execve arguments */
#define AUDIT_PROCTITLE     1327    /* Process title (cmdline) */
#define AUDIT_OPENAT2       1337    /* openat2 how args */
#define AUDIT_EOE           1320    /* End of multi-record event */
```

### LSM-specific Types (1400-series)

```c
#define AUDIT_AVC               1400    /* SELinux AVC denial or grant */
#define AUDIT_SELINUX_ERR       1401    /* Internal SELinux errors */
#define AUDIT_MAC_POLICY_LOAD   1403    /* Policy file load */
#define AUDIT_MAC_STATUS        1404    /* Enforcing/permissive/off change */
#define AUDIT_LANDLOCK_ACCESS   1423    /* Landlock denial */
#define AUDIT_MAC_TASK_CONTEXTS 1425    /* Multiple LSM task contexts */
#define AUDIT_MAC_OBJ_CONTEXTS  1426    /* Multiple LSM object contexts */
```

### Record Format

Each audit record begins with a common header generated by `audit_log_start()`
at `kernel/audit.c:1965`:

```c
audit_log_format(ab, "audit(%llu.%03lu:%u): ",
                 (unsigned long long)ab->stamp.ctime.tv_sec,
                 ab->stamp.ctime.tv_nsec/1000000,
                 ab->stamp.serial);
```

This produces: `audit(EPOCH.MILLISECONDS:SERIAL):` followed by type-specific
key=value pairs. For a syscall record (`AUDIT_SYSCALL`), the format is
generated at `kernel/auditsc.c:1664-1684`:

```
audit(1234567890.123:456): arch=c000003e syscall=59 success=yes exit=0
  a0=... a1=... a2=... a3=... items=2 ppid=1234 pid=5678 auid=1000
  uid=0 gid=0 euid=0 ... tty=pts0 comm="bash" exe="/bin/bash" key="exec_log"
```

A single auditable event (identified by timestamp:serial) may produce
multiple records (SYSCALL, PATH, CWD, EXECVE, PROCTITLE, etc.), all sharing
the same stamp.

---

## 4. Syscall Auditing -- Entry/Exit Hooks

### Task Allocation

When a task is forked, `audit_alloc()` (`kernel/auditsc.c:1057`) determines
whether the new task requires an audit context:

```c
int audit_alloc(struct task_struct *tsk)
{
    struct audit_context *context;
    enum audit_state     state;
    char *key = NULL;

    if (likely(!audit_ever_enabled))
        return 0;

    state = audit_filter_task(tsk, &key);
    if (state == AUDIT_STATE_DISABLED) {
        clear_task_syscall_work(tsk, SYSCALL_AUDIT);
        return 0;
    }

    context = audit_alloc_context(state);
    ...
    audit_set_context(tsk, context);
    set_task_syscall_work(tsk, SYSCALL_AUDIT);
    return 0;
}
```

The `SYSCALL_AUDIT` flag in the task's syscall work bits causes the
architecture-specific entry/exit code to invoke the audit hooks.

### Audit State Machine

`kernel/audit.h:28`

```c
enum audit_state {
    AUDIT_STATE_DISABLED,   /* No audit context; no records possible */
    AUDIT_STATE_BUILD,      /* Context exists; fill at entry; record only
                               if something else triggers it */
    AUDIT_STATE_RECORD      /* Always fill context and write record */
};
```

### Syscall Entry

`kernel/auditsc.c:1986`

Called from architecture-specific syscall entry paths via `audit_syscall_entry()`:

```c
void __audit_syscall_entry(int major, unsigned long a1, unsigned long a2,
                           unsigned long a3, unsigned long a4)
{
    struct audit_context *context = audit_context();
    enum audit_state     state;

    if (!audit_enabled || !context)
        return;

    state = context->state;
    if (state == AUDIT_STATE_DISABLED)
        return;

    context->dummy = !audit_n_rules;
    if (!context->dummy && state == AUDIT_STATE_BUILD) {
        context->prio = 0;
        if (auditd_test_task(current))
            return;
    }

    context->arch       = syscall_get_arch(current);
    context->major      = major;
    context->argv[0]    = a1;
    context->argv[1]    = a2;
    context->argv[2]    = a3;
    context->argv[3]    = a4;
    context->context    = AUDIT_CTX_SYSCALL;
    context->current_state = state;
    ktime_get_coarse_real_ts64(&context->stamp.ctime);
}
```

The `context->dummy` flag is set when there are no audit rules loaded
(`audit_n_rules == 0`). This enables a fast path where name/inode collection
is skipped via `audit_dummy_context()`.

### Syscall Exit

`kernel/auditsc.c:2035`

```c
void __audit_syscall_exit(int success, long return_code)
{
    struct audit_context *context = audit_context();

    if (!context || context->dummy ||
        context->context != AUDIT_CTX_SYSCALL)
        goto out;

    if (!list_empty(&context->killed_trees))
        audit_kill_trees(context);

    audit_return_fixup(context, success, return_code);
    /* run through both filters to ensure we set the filterkey properly */
    audit_filter_syscall(current, context);
    audit_filter_inodes(current, context);
    if (context->current_state != AUDIT_STATE_RECORD)
        goto out;

    audit_log_exit();

out:
    audit_reset_context(context);
}
```

Exit filtering runs against the `AUDIT_FILTER_EXIT` list
(`kernel/auditsc.c:869-878`). Then inode-specific filtering runs against the
inode hash table (`kernel/auditsc.c:885`). Only if `current_state` reaches
`AUDIT_STATE_RECORD` does `audit_log_exit()` emit the audit records.

### Name and Inode Collection

During syscall execution, VFS operations call:

- `audit_getname()` / `__audit_getname()` -- records filename pointers
  (zero-copy from `getname()`)
- `audit_inode()` / `__audit_inode()` -- records inode number, device,
  mode, ownership, security context
- `audit_inode_child()` / `__audit_inode_child()` -- records child
  inode info (for creates/deletes in a watched directory)

These are called inline from `include/linux/audit.h:389-399` and are
no-ops when `audit_dummy_context()` returns true.

---

## 5. Filter Rules -- Definition and Matching

### Filter Lists

`kernel/auditfilter.c:39`

There are 8 filter lists, each corresponding to a different point in the
audit pipeline:

```c
struct list_head audit_filter_list[AUDIT_NR_FILTERS];
```

```c
// include/uapi/linux/audit.h:173-183
#define AUDIT_FILTER_USER       0x00    /* user-generated messages */
#define AUDIT_FILTER_TASK       0x01    /* task creation (fork) */
#define AUDIT_FILTER_ENTRY      0x02    /* syscall entry (deprecated) */
#define AUDIT_FILTER_WATCH      0x03    /* file system watches */
#define AUDIT_FILTER_EXIT       0x04    /* syscall exit */
#define AUDIT_FILTER_EXCLUDE    0x05    /* before record creation */
#define AUDIT_FILTER_FS         0x06    /* at __audit_inode_child */
#define AUDIT_FILTER_URING_EXIT 0x07    /* io_uring op exit */
```

Rules are protected by `audit_filter_mutex` for writes and RCU for reads
(`kernel/auditfilter.c:27-36`).

### Rule Actions

```c
// include/uapi/linux/audit.h:188-190
#define AUDIT_NEVER     0    /* Do not build context if rule matches */
#define AUDIT_POSSIBLE  1    /* Build context if rule matches */
#define AUDIT_ALWAYS    2    /* Generate audit record if rule matches */
```

### Field Types

Rules can match on many fields (`include/uapi/linux/audit.h:253-307`):

| Field          | Value | Description                       |
|----------------|-------|-----------------------------------|
| `AUDIT_PID`    |   0   | Process ID                        |
| `AUDIT_UID`    |   1   | Real user ID                      |
| `AUDIT_LOGINUID` | 9   | Login UID (auid)                  |
| `AUDIT_ARCH`   |  11   | System call architecture          |
| `AUDIT_MSGTYPE`|  12   | Audit message type                |
| `AUDIT_SUBJ_*` | 13-17 | SELinux subject labels            |
| `AUDIT_OBJ_*`  | 19-23 | SELinux object labels             |
| `AUDIT_EXIT`   | 103   | Syscall exit code                 |
| `AUDIT_SUCCESS` | 104  | Syscall success/failure           |
| `AUDIT_WATCH`  | 105   | File watch path                   |
| `AUDIT_PERM`   | 106   | Permission filter (r/w/x/a)       |
| `AUDIT_DIR`    | 107   | Directory watch path              |
| `AUDIT_EXE`    | 112   | Executable path of the process    |
| `AUDIT_ARG0-3` |200-203| Syscall arguments                 |

### Comparison Operators

`include/uapi/linux/audit.h:311-334`

```c
#define AUDIT_BIT_MASK      0x08000000  /* &  bit mask */
#define AUDIT_LESS_THAN     0x10000000  /* <  */
#define AUDIT_GREATER_THAN  0x20000000  /* >  */
#define AUDIT_NOT_EQUAL     0x30000000  /* != */
#define AUDIT_EQUAL         0x40000000  /* =  */
#define AUDIT_BIT_TEST      (AUDIT_BIT_MASK|AUDIT_EQUAL)  /* &= */
#define AUDIT_LESS_THAN_OR_EQUAL    (AUDIT_LESS_THAN|AUDIT_EQUAL)
#define AUDIT_GREATER_THAN_OR_EQUAL (AUDIT_GREATER_THAN|AUDIT_EQUAL)
```

### The `audit_field` Structure

`include/linux/audit.h:66`

```c
struct audit_field {
    u32             type;
    union {
        u32         val;
        kuid_t      uid;
        kgid_t      gid;
        struct {
            char    *lsm_str;
            void    *lsm_rule;
        };
    };
    u32             op;
};
```

For LSM-related fields (`AUDIT_SUBJ_*`, `AUDIT_OBJ_*`), the `lsm_str`
stores the string representation and `lsm_rule` stores the opaque
LSM-compiled rule.

### The Filter Matching Engine

`kernel/auditsc.c:464`

```c
static int audit_filter_rules(struct task_struct *tsk,
                              struct audit_krule *rule,
                              struct audit_context *ctx,
                              struct audit_names *name,
                              enum audit_state *state,
                              bool task_creation)
```

This function iterates over all fields in a rule, checking each against
the current task's credentials, the syscall context, and any collected
inode names. Every field must match for the rule to fire (AND logic).
The function handles:

- PID, UID, GID, session ID comparisons (lines 488-548)
- Architecture and syscall exit value matching (lines 549-565)
- Inode number, device major/minor matching (lines 566-607)
- LSM subject/object label matching via `security_audit_rule_match()` (SELinux
  type/role/user comparisons)
- Watch and tree matching (inode and directory path comparisons)
- Executable path matching via `audit_exe_compare()` (line 499-503)
- Permission matching via `audit_match_perm()` (read/write/exec/attr)

---

## 6. File Watches and Directory Trees

### File Watches (`audit_watch.c`)

File watches monitor a specific file path by tracking the parent directory's
inode through fsnotify.

#### `struct audit_watch`

`kernel/audit_watch.c:36`

```c
struct audit_watch {
    refcount_t          count;      /* reference count */
    dev_t               dev;        /* associated superblock device */
    char                *path;      /* insertion path */
    unsigned long       ino;        /* associated inode number */
    struct audit_parent *parent;    /* associated parent */
    struct list_head    wlist;      /* entry in parent->watches list */
    struct list_head    rules;      /* anchor for krule->rlist */
};
```

#### `struct audit_parent`

`kernel/audit_watch.c:46`

```c
struct audit_parent {
    struct list_head    watches;    /* anchor for audit_watch->wlist */
    struct fsnotify_mark mark;     /* fsnotify mark on the inode */
};
```

The watch system uses `fsnotify` to receive notifications about file events
on the parent directory. The events monitored are
(`kernel/audit_watch.c:55`):

```c
#define AUDIT_FS_WATCH (FS_MOVE | FS_CREATE | FS_DELETE | FS_DELETE_SELF | \
                        FS_MOVE_SELF | FS_UNMOUNT)
```

When these events occur, the watch updates its tracked inode number or
invalidates the watch. Watch lifetime extends from `audit_init_parent()` to
receipt of an `FS_IGNORED` event.

### Directory Trees (`audit_tree.c`)

Directory tree watches monitor an entire subtree, tagging every inode in
the tree for audit purposes.

#### `struct audit_tree`

`kernel/audit_tree.c:13`

```c
struct audit_tree {
    refcount_t  count;
    int         goner;
    struct audit_chunk *root;
    struct list_head chunks;
    struct list_head rules;
    struct list_head list;
    struct list_head same_root;
    struct rcu_head head;
    char        pathname[];
};
```

#### `struct audit_chunk`

`kernel/audit_tree.c:25`

```c
struct audit_chunk {
    struct list_head    hash;
    unsigned long       key;
    struct fsnotify_mark *mark;
    struct list_head    trees;      /* trees with root here */
    int                 count;
    atomic_long_t       refs;
    struct rcu_head     head;
    struct audit_node {
        struct list_head list;
        struct audit_tree *owner;
        unsigned index;             /* upper bit indicates 'will prune' */
    } owners[] __counted_by(count);
};
```

Each inode of interest has an `audit_tree_mark` (fsnotify mark) with an
attached `audit_chunk`. The chunk-to-mark association is protected by
`hash_lock` and `audit_tree_group->mark_mutex`
(`kernel/audit_tree.c:49-88`).

During syscall processing, `handle_one()` and `handle_path()`
(`kernel/auditsc.c:2060-2087`) look up chunks for inodes encountered in
path resolution, storing references in the context's `trees` list for later
matching in `match_tree_refs()`.

### Inode Hash Table

`kernel/audit.h:225`

Inode-based rules (watches) are also tracked in a hash table for fast lookup
at syscall exit time:

```c
#define AUDIT_INODE_BUCKETS  32
struct list_head audit_inode_hash[AUDIT_INODE_BUCKETS];

static inline int audit_hash_ino(u32 ino)
{
    return (ino & (AUDIT_INODE_BUCKETS-1));
}
```

---

## 7. The Netlink Interface

### Socket Creation

`kernel/audit.c:1682`

The audit netlink socket is created per network namespace:

```c
static int __net_init audit_net_init(struct net *net)
{
    struct netlink_kernel_cfg cfg = {
        .input  = audit_receive,
        .bind   = audit_multicast_bind,
        .unbind = audit_multicast_unbind,
        .flags  = NL_CFG_F_NONROOT_RECV,
        .groups = AUDIT_NLGRP_MAX,
    };

    struct audit_net *aunet = net_generic(net, audit_net_id);
    aunet->sk = netlink_kernel_create(net, NETLINK_AUDIT, &cfg);
    ...
}
```

### Message Reception

`kernel/audit.c:1591`

Incoming netlink messages are dispatched through `audit_receive()` which
calls `audit_receive_msg()` (`kernel/audit.c:1250`). The handler processes
a `switch` on `msg_type`:

- `AUDIT_GET` (1000) -- returns `struct audit_status` with current configuration
- `AUDIT_SET` (1001) -- modifies enabled/failure/pid/rate_limit/backlog_limit
- `AUDIT_ADD_RULE` (1011) -- adds a filter rule via `audit_rule_change()`
- `AUDIT_DEL_RULE` (1012) -- deletes a filter rule
- `AUDIT_LIST_RULES` (1013) -- lists all rules
- `AUDIT_SET_FEATURE` (1018) -- toggles features like `loginuid_immutable`
- `AUDIT_SIGNAL_INFO` (1010) -- returns information about the last signal
  sender
- `AUDIT_FIRST_USER_MSG..AUDIT_LAST_USER_MSG` -- relays user-space messages
  into the audit trail

### The `struct audit_status` ABI

`include/uapi/linux/audit.h:472`

```c
struct audit_status {
    __u32   mask;                   /* Bit mask for valid entries */
    __u32   enabled;                /* 1 = enabled, 0 = disabled */
    __u32   failure;                /* Failure-to-log action */
    __u32   pid;                    /* pid of auditd process */
    __u32   rate_limit;             /* messages rate limit (per second) */
    __u32   backlog_limit;          /* waiting messages limit */
    __u32   lost;                   /* messages lost */
    __u32   backlog;                /* messages waiting in queue */
    union {
        __u32 version;              /* deprecated */
        __u32 feature_bitmap;       /* bitmap of kernel audit features */
    };
    __u32   backlog_wait_time;      /* message queue wait timeout */
    __u32   backlog_wait_time_actual; /* actual time spent waiting */
};
```

### Auditd Connection Tracking

`kernel/audit.c:111`

```c
struct auditd_connection {
    struct pid  *pid;
    u32         portid;
    struct net  *net;
    struct rcu_head rcu;
};
static struct auditd_connection __rcu *auditd_conn;
```

This RCU-protected structure tracks the currently registered audit daemon.
Only one auditd may be registered at a time.

### Multicast Support

Audit events are also sent via netlink multicast to group
`AUDIT_NLGRP_READLOG` (`include/uapi/linux/audit.h:467`), allowing
"best effort" read-only listeners.

### The kauditd Thread

`kernel/audit.c:878`

A dedicated kernel thread delivers audit records through three queues:

```c
static struct sk_buff_head audit_queue;       /* primary queue */
static struct sk_buff_head audit_retry_queue;  /* temporary unicast failures */
static struct sk_buff_head audit_hold_queue;   /* awaiting new auditd */
```

The thread's main loop:
1. Flush the `audit_hold_queue` (records waiting for auditd reconnect)
2. Flush the `audit_retry_queue` (records that failed unicast delivery)
3. Process the `audit_queue` -- multicast to listeners, then attempt
   unicast to auditd; on failure, move to retry or hold queue
4. Wake any threads blocked on `audit_backlog_wait`
5. Sleep until new records arrive

---

## 8. Integration with SELinux/AppArmor

### LSM Security Context Logging

The audit subsystem records LSM security contexts on both subjects (processes)
and objects (files, IPC, etc.) through dedicated functions.

#### `audit_log_subj_ctx()`

`kernel/audit.c:2282`

```c
int audit_log_subj_ctx(struct audit_buffer *ab, struct lsm_prop *prop)
{
    struct lsm_context ctx;
    ...
    security_current_getlsmprop_subj(prop);
    if (!lsmprop_is_set(prop))
        return 0;

    if (audit_subj_secctx_cnt < 2) {
        error = security_lsmprop_to_secctx(prop, &ctx, LSM_ID_UNDEF);
        ...
    }
    ...
}
```

This calls into the LSM framework to convert the process's security property
into a human-readable security context string (e.g., SELinux's
`unconfined_u:unconfined_r:unconfined_t:s0`). When multiple LSMs provide
security contexts (`audit_subj_secctx_cnt >= 2`), a separate
`AUDIT_MAC_TASK_CONTEXTS` record is emitted.

#### LSM Tracking

`kernel/audit.c:87-90, 300`

The audit subsystem maintains lists of LSMs that provide security contexts:

```c
static u32 audit_subj_secctx_cnt;
static u32 audit_obj_secctx_cnt;
static const struct lsm_id *audit_subj_lsms[MAX_LSM_COUNT];
static const struct lsm_id *audit_obj_lsms[MAX_LSM_COUNT];
```

LSMs register themselves via `audit_cfg_lsm()` during initialization.

#### `audit_log_task_context()`

`kernel/audit.c:2339`

A convenience wrapper used throughout the audit code to log the current
task's subject context. It is called from `audit_log_exit()` and many
other logging functions.

### LSM Rule Matching

Filter rules with `AUDIT_SUBJ_*` or `AUDIT_OBJ_*` fields contain
LSM-compiled rules (`audit_field.lsm_rule`). During filtering at
`kernel/auditsc.c:464`, these are evaluated via:

```c
security_audit_rule_match(prop, f->type, f->op, f->lsm_rule)
```

This delegates to the active LSM (SELinux, AppArmor, etc.) to determine
if the task or object matches the rule's security label criteria.

When LSM policy is reloaded, `audit_update_lsm_rules()` is called to
recompile all LSM-based filter rules against the new policy.

### AVC Audit Messages

SELinux and other LSMs call `audit_log()` directly to emit access vector
cache (AVC) decisions as `AUDIT_AVC` (1400) records. AppArmor similarly
generates its own audit records for policy decisions. These records share
the same timestamp/serial as the associated syscall record when they occur
within a syscall context.

---

## 9. Performance Considerations

### Minimal Overhead When Disabled

The audit subsystem is designed for near-zero overhead when disabled. The
primary fast-path check is `audit_ever_enabled` (`kernel/auditsc.c:1063`):

```c
if (likely(!audit_ever_enabled))
    return 0;
```

Once auditing has ever been enabled, this flag remains true. Per-syscall
overhead is then governed by `audit_dummy_context()`, which checks the
`dummy` field at the start of `audit_context`.

### The Dummy Context Optimization

`include/linux/audit.h:342`

When no rules are loaded (`audit_n_rules == 0`), the context's `dummy`
field is set to 1 at syscall entry (`kernel/auditsc.c:2006`). This causes
all `audit_getname()`, `audit_inode()`, and similar collection calls to
be skipped entirely (they check `audit_dummy_context()` first).

### Preallocated Name Slots

`kernel/audit.h:132`

```c
struct audit_names  preallocated_names[AUDIT_NAMES]; /* AUDIT_NAMES = 5 */
```

The first 5 file names per syscall use preallocated slots in the
`audit_context` rather than dynamic allocation, avoiding `kmalloc` overhead
for the common case.

### Rate Limiting

`kernel/audit.c:122-128`

```c
static u32  audit_rate_limit;           /* messages per second, 0 = unlimited */
static u32  audit_backlog_limit = 64;   /* max queued messages */
#define AUDIT_BACKLOG_WAIT_TIME (60 * HZ)
static u32  audit_backlog_wait_time = AUDIT_BACKLOG_WAIT_TIME;
```

When the backlog limit is exceeded, `audit_log_start()` blocks the calling
task for up to `audit_backlog_wait_time` jiffies (default 60 seconds),
applying back-pressure to producers (`kernel/audit.c:1926-1952`). The
auditd process and the audit control mutex holder are exempt from blocking
to avoid deadlocks.

### Failure Modes

`include/uapi/linux/audit.h:381-383`

```c
#define AUDIT_FAIL_SILENT   0   /* silently drop records */
#define AUDIT_FAIL_PRINTK   1   /* log to printk (default) */
#define AUDIT_FAIL_PANIC    2   /* kernel panic */
```

The `audit_failure` variable controls behavior when records cannot be
delivered. `AUDIT_FAIL_PANIC` is used in high-security environments where
losing audit records is unacceptable.

### Lost Record Tracking

`kernel/audit.c:143`

```c
static atomic_t audit_lost = ATOMIC_INIT(0);
```

Records can be lost due to: suppression in `audit_alloc`, out-of-memory in
`audit_log_start` or buffer allocation, rate limiting, or backlog limit
exceeded. The lost count is reported through `audit_status.lost`.

### RCU for Filter Traversal

Filter rules are read under RCU protection (`rcu_read_lock()` in
`audit_filter_syscall()` at `kernel/auditsc.c:875`), allowing lock-free
traversal on the hot path. Modifications use `audit_filter_mutex` and
`call_rcu()` for safe deferred freeing (`kernel/auditfilter.c:63, 99`).

---

## 10. Key Code Paths

### Syscall Audit Path (Hot Path)

```
syscall_enter_from_user_mode()           [arch/*/entry.S or C]
  -> audit_syscall_entry()               [include/linux/audit.h:367]
    -> __audit_syscall_entry()           [kernel/auditsc.c:1986]
         records: arch, major, argv[0-3], timestamp

    ... syscall body executes ...
    ... VFS calls audit_getname(), audit_inode() ...

syscall_exit_to_user_mode()
  -> audit_syscall_exit()                [include/linux/audit.h:374]
    -> __audit_syscall_exit()            [kernel/auditsc.c:2035]
      -> audit_return_fixup()            -- set return_valid, return_code
      -> audit_filter_syscall()          [kernel/auditsc.c:869]
        -> __audit_filter_op()           -- traverse AUDIT_FILTER_EXIT list
          -> audit_filter_rules()        [kernel/auditsc.c:464]
      -> audit_filter_inodes()           -- check inode hash for watch matches
      -> audit_log_exit()                [kernel/auditsc.c:1652]
        -> audit_log_start()             [kernel/audit.c:1903]
          -> audit_buffer_alloc()        -- allocate from slab cache
          -> audit_get_stamp()           -- get/assign serial number
        -> audit_log_format()            -- format key=value pairs into skb
        -> audit_log_end()               -- enqueue skb to audit_queue
          -> wake_up_interruptible(&kauditd_wait)
```

### Record Delivery Path

```
kauditd_thread()                         [kernel/audit.c:878]
  -> kauditd_send_queue(&audit_hold_queue)   -- flush held records
  -> kauditd_send_queue(&audit_retry_queue)  -- retry failed unicasts
  -> kauditd_send_queue(&audit_queue)        -- process main queue
    -> kauditd_send_multicast_skb()          -- multicast to READLOG group
    -> netlink_unicast()                     -- unicast to auditd
  -> wake_up(&audit_backlog_wait)            -- unblock producers
  -> wait_event_freezable(kauditd_wait, ...) -- sleep until more records
```

### Rule Management Path

```
audit_receive()                          [kernel/audit.c:1591]
  -> audit_receive_msg()                 [kernel/audit.c:1250]
    case AUDIT_ADD_RULE:
      -> audit_rule_change()             [kernel/auditfilter.c]
        -> audit_data_to_entry()         -- parse audit_rule_data into audit_entry
        -> audit_add_rule()              -- add to filter list under mutex
          -> audit_add_watch()           -- set up fsnotify watch if needed
          -> audit_add_tree_rule()       -- set up tree watch if needed
          -> list_add_rcu()              -- add to appropriate filter list
    case AUDIT_DEL_RULE:
      -> audit_rule_change()
        -> audit_del_rule()
          -> list_del_rcu()
          -> call_rcu(&e->rcu, audit_free_rule_rcu)
```

### Netlink Command Path

```
userspace (auditctl, auditd)
  -> sendmsg(NETLINK_AUDIT, ...)
    -> audit_receive()                   [kernel/audit.c:1591]
      -> audit_receive_msg()             [kernel/audit.c:1250]
        case AUDIT_GET:  -> audit_send_reply() with audit_status
        case AUDIT_SET:  -> audit_set_enabled/failure/rate_limit/...
        case AUDIT_ADD_RULE / AUDIT_DEL_RULE -> audit_rule_change()
        case AUDIT_LIST_RULES -> audit_list_rules_send()
        case AUDIT_USER..AUDIT_LAST_USER_MSG -> relay to audit log
```
