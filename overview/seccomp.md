# Linux Kernel Seccomp Subsystem

## Table of Contents
1. [Introduction](#1-introduction)
2. [Seccomp Modes](#2-seccomp-modes)
3. [Core Data Structures](#3-core-data-structures)
4. [Filter Input: seccomp_data](#4-filter-input-seccomp_data)
5. [BPF Filter Programs](#5-bpf-filter-programs)
6. [SECCOMP_RET Actions](#6-seccomp_ret-actions)
7. [Filter Installation](#7-filter-installation)
8. [Filter Evaluation](#8-filter-evaluation)
9. [Filter Inheritance (fork/exec)](#9-filter-inheritance-forkexec)
10. [Thread Synchronization (TSYNC)](#10-thread-synchronization-tsync)
11. [User Notification Mechanism](#11-user-notification-mechanism)
12. [Interaction with ptrace](#12-interaction-with-ptrace)
13. [Logging and Auditing](#13-logging-and-auditing)
14. [Action Cache (Performance Optimization)](#14-action-cache-performance-optimization)
15. [Sysctl and Proc Interfaces](#15-sysctl-and-proc-interfaces)
16. [Key Source Files](#16-key-source-files)

---

## 1. Introduction

Seccomp (Secure Computing) is a Linux kernel facility that restricts the system
calls a process can make. Originally introduced in 2005 by Andrea Arcangeli as
a strict allow-list of four syscalls, it was extended in 2012 by Will Drewry at
Google to support arbitrary BPF filter programs. Seccomp is widely used by
container runtimes (Docker, Podman), web browsers (Chrome, Firefox), systemd
services, and sandboxing tools (Firejail, bubblewrap) to reduce the kernel
attack surface available to potentially compromised processes.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Seccomp Overview                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User Space                                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Application                                             │   │
│  │    │                                                     │   │
│  │    │  1. Install filter:                                 │   │
│  │    │     prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &fp) │   │
│  │    │     - or -                                          │   │
│  │    │     seccomp(SECCOMP_SET_MODE_FILTER, flags, &fp)    │   │
│  │    │                                                     │   │
│  │    │  2. Make syscall (e.g., open())                      │   │
│  │    ▼                                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│         │                                                       │
│  ───────┼───────────────────────────────────── syscall entry ─  │
│         ▼                                                       │
│  Kernel Space                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  __secure_computing()                                    │   │
│  │    │                                                     │   │
│  │    ▼                                                     │   │
│  │  seccomp_run_filters(sd, &match)                         │   │
│  │    │                                                     │   │
│  │    │  Walk filter chain (newest → oldest)                │   │
│  │    │  Run each cBPF program against seccomp_data         │   │
│  │    │  Keep lowest (most restrictive) return value        │   │
│  │    │                                                     │   │
│  │    ▼                                                     │   │
│  │  __seccomp_filter() — act on result                      │   │
│  │    ├── ALLOW  → proceed with syscall                     │   │
│  │    ├── ERRNO  → return -errno to user                    │   │
│  │    ├── TRAP   → deliver SIGSYS                           │   │
│  │    ├── TRACE  → notify ptrace tracer                     │   │
│  │    ├── LOG    → allow + audit log                        │   │
│  │    ├── USER_NOTIF → notify userspace supervisor          │   │
│  │    ├── KILL_THREAD → SIGSYS, kill thread                 │   │
│  │    └── KILL_PROCESS → SIGSYS, kill process               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Design Principles

- **One-way transition**: Once seccomp is enabled, it cannot be disabled. The
  mode can only become more restrictive, never less.
- **No privilege escalation**: Installing a filter requires either
  `no_new_privs` or `CAP_SYS_ADMIN`, preventing unprivileged tasks from
  affecting privileged children.
- **All filters evaluated**: Every filter in the chain runs on every syscall;
  the most restrictive result wins via `min()` semantics.
- **Immutable filters**: Filters are never modified after attachment (only
  reference counts change). This allows safe lock-free reads at syscall entry.

---

## 2. Seccomp Modes

Defined in `include/uapi/linux/seccomp.h:10-12`:

```c
#define SECCOMP_MODE_DISABLED   0  /* seccomp is not in use */
#define SECCOMP_MODE_STRICT     1  /* uses hard-coded filter */
#define SECCOMP_MODE_FILTER     2  /* uses user-supplied filter */
```

An internal-only mode is also defined in `kernel/seccomp.c:33`:

```c
#define SECCOMP_MODE_DEAD       (SECCOMP_MODE_FILTER + 1)
```

### Mode 1: Strict

The original seccomp mode. Only four syscalls are permitted:

`kernel/seccomp.c:1024-1027`:
```c
static const int mode1_syscalls[] = {
    __NR_seccomp_read, __NR_seccomp_write,
    __NR_seccomp_exit, __NR_seccomp_sigreturn,
    -1, /* negative terminated */
};
```

Any other syscall causes immediate `SIGKILL`. Strict mode is useful for
pure computation tasks that only need to read input and write output (e.g.,
the original cpushare use case).

### Mode 2: Filter

Allows arbitrary syscall filtering using classic BPF (cBPF) programs. Each
filter program examines `struct seccomp_data` and returns a `SECCOMP_RET_*`
action. Multiple filters can be stacked, and the most restrictive result
across all filters takes effect.

### Mode DEAD

Set internally when a process survives `SECCOMP_RET_KILL_*` (e.g., the
signal was somehow caught). Any subsequent syscall in `SECCOMP_MODE_DEAD`
triggers an unconditional `SIGKILL` via `do_exit()`.

---

## 3. Core Data Structures

### struct seccomp (task state)

`include/linux/seccomp_types.h:22-26`:

```c
struct seccomp {
    int mode;                          /* SECCOMP_MODE_* */
    atomic_t filter_count;             /* number of installed filters */
    struct seccomp_filter *filter;     /* most recently installed filter */
};
```

Embedded directly in `task_struct`. The `filter` pointer is accessed without
locking at syscall entry (via `READ_ONCE`), so it must only be modified from
the context of the current task while holding `sighand->siglock`.

### struct seccomp_filter

`kernel/seccomp.c:226-237`:

```c
struct seccomp_filter {
    refcount_t refs;                   /* lifetime reference count */
    refcount_t users;                  /* directly attached task count */
    bool log;                          /* log all non-ALLOW actions */
    bool wait_killable_recv;           /* killable state after notif recv */
    struct action_cache cache;         /* per-syscall allow bitmap */
    struct seccomp_filter *prev;       /* parent filter (linked list) */
    struct bpf_prog *prog;             /* the cBPF program */
    struct notification *notif;        /* for SECCOMP_RET_USER_NOTIF */
    struct mutex notify_lock;          /* protects notification state */
    wait_queue_head_t wqh;             /* poll wait queue for notifier */
};
```

Filters form a singly-linked list via `->prev`, with `task->seccomp.filter`
pointing to the most recently installed filter. Due to `fork()`, this list
forms a tree in memory -- multiple tasks may share the same `->prev` chain,
similar to how namespaces work.

```
  Task A installs Filter 1, then Filter 2
  Task A forks Task B
  Task B installs Filter 3

  Task A:  filter → [F2] → [F1] → NULL
  Task B:  filter → [F3] → [F2] → [F1] → NULL
                             ↑
                       shared node (refcount > 1)
```

**Reference counting**: `refs` is incremented for each directly attached task,
once for the dependent filter (`->prev`), and once if a user notification
listener FD exists. `users` tracks only direct task attachments and the
dependent filter. When `users` reaches zero, waiting notification listeners
are woken with `EPOLLHUP`.

### struct notification

`kernel/seccomp.c:151-156`:

```c
struct notification {
    atomic_t requests;                 /* pending request semaphore */
    u32 flags;                         /* SECCOMP_USER_NOTIF_FD_* */
    u64 next_id;                       /* monotonic notification ID */
    struct list_head notifications;    /* list of seccomp_knotif */
};
```

Allocated lazily only when `SECCOMP_FILTER_FLAG_NEW_LISTENER` is used.

### struct seccomp_knotif (kernel notification)

`kernel/seccomp.c:63-102`:

```c
struct seccomp_knotif {
    struct task_struct *task;           /* task whose syscall was caught */
    u64 id;                            /* unique cookie for this request */
    const struct seccomp_data *data;   /* syscall data (valid while active) */
    enum notify_state state;           /* INIT → SENT → REPLIED */
    int error;                         /* reply errno (valid when REPLIED) */
    long val;                          /* reply return value */
    u32 flags;                         /* reply flags */
    struct completion ready;           /* state-change signal */
    struct list_head list;             /* linked into notification->notifications */
    struct list_head addfd;            /* outstanding addfd requests */
};
```

The `notify_state` enum (`kernel/seccomp.c:57-61`):

```c
enum notify_state {
    SECCOMP_NOTIFY_INIT,               /* created, not yet read by listener */
    SECCOMP_NOTIFY_SENT,               /* read by listener, awaiting reply */
    SECCOMP_NOTIFY_REPLIED,            /* listener has sent a response */
};
```

### struct seccomp_kaddfd

`kernel/seccomp.c:122-135`:

```c
struct seccomp_kaddfd {
    struct file *file;                 /* file to install in target task */
    int fd;                            /* target fd number (-1 = allocate) */
    unsigned int flags;                /* O_CLOEXEC etc. */
    __u32 ioctl_flags;                 /* SECCOMP_ADDFD_FLAG_* */
    union {
        bool setfd;                    /* whether SETFD was requested */
        int ret;                       /* installed fd or error (on reply) */
    };
    struct completion completion;      /* signals when fd installation done */
    struct list_head list;             /* chained onto knotif->addfd */
};
```

---

## 4. Filter Input: seccomp_data

`include/uapi/linux/seccomp.h:62-67`:

```c
struct seccomp_data {
    int nr;                    /* syscall number */
    __u32 arch;                /* AUDIT_ARCH_* value (syscall convention) */
    __u64 instruction_pointer; /* IP at time of syscall */
    __u64 args[6];             /* syscall arguments (always 64-bit) */
};
```

This structure is populated at syscall entry by `populate_seccomp_data()`
(`kernel/seccomp.c:246-266`), which reads register state via the
architecture-specific `syscall_get_nr()`, `syscall_get_arch()`, and
`syscall_get_arguments()` helpers. Arguments are always stored as 64-bit
values regardless of the architecture.

**Important**: The BPF filter can only examine the `seccomp_data` fields.
It cannot dereference pointers in syscall arguments -- it sees only the
raw pointer values. This is by design: the filter operates in a context
where dereferencing user pointers would be unsafe and could be subject to
TOCTOU attacks.

```
┌─────────────────────────────────────────────────────────────┐
│                struct seccomp_data layout                    │
├──────────┬──────┬───────────────────────────────────────────┤
│ Offset   │ Size │ Field                                     │
├──────────┼──────┼───────────────────────────────────────────┤
│ 0x00     │  4   │ nr (syscall number)                       │
│ 0x04     │  4   │ arch (AUDIT_ARCH_*)                       │
│ 0x08     │  8   │ instruction_pointer                       │
│ 0x10     │  8   │ args[0]                                   │
│ 0x18     │  8   │ args[1]                                   │
│ 0x20     │  8   │ args[2]                                   │
│ 0x28     │  8   │ args[3]                                   │
│ 0x30     │  8   │ args[4]                                   │
│ 0x38     │  8   │ args[5]                                   │
├──────────┼──────┼───────────────────────────────────────────┤
│ Total    │ 64   │ bytes                                     │
└──────────┴──────┴───────────────────────────────────────────┘
```

---

## 5. BPF Filter Programs

Seccomp uses **classic BPF (cBPF)**, not eBPF. The filter is provided as a
`struct sock_fprog` containing an array of `struct sock_filter` instructions.
Internally, the kernel converts cBPF to eBPF for execution via JIT compilation,
but the user-facing API remains cBPF only.

### Allowed Instructions

`seccomp_check_filter()` at `kernel/seccomp.c:280-348` validates each
instruction. Only a subset of cBPF opcodes is permitted:

- **Load**: `BPF_LD|BPF_W|BPF_ABS` (load from seccomp_data), `BPF_LD|BPF_IMM`,
  `BPF_LD|BPF_MEM`, `BPF_LDX|BPF_IMM`, `BPF_LDX|BPF_MEM`, `BPF_LD|BPF_W|BPF_LEN`
- **Store**: `BPF_ST`, `BPF_STX`
- **ALU**: ADD, SUB, MUL, DIV, AND, OR, XOR, LSH, RSH, NEG (with K or X)
- **Jump**: JA, JEQ, JGE, JGT, JSET (with K or X)
- **Return**: `BPF_RET|BPF_K`, `BPF_RET|BPF_A`
- **Misc**: TAX, TXA

Notably, **no packet data loads** (`BPF_LD|BPF_W|BPF_IND` etc.) are allowed.
The `BPF_LD|BPF_W|BPF_ABS` instruction is rewritten to load from `seccomp_data`
fields instead of socket buffer data.

### Filter Size Limits

`kernel/seccomp.c:239`:

```c
#define MAX_INSNS_PER_PATH ((1 << 18) / sizeof(struct sock_filter))
```

This limits the total instructions across all stacked filters to 256KB worth
(approximately 32,768 instructions). Each existing filter in the chain incurs
a 4-instruction penalty (`kernel/seccomp.c:900`).

Individual filters are also limited to `BPF_MAXINSNS` (4096) instructions.

### Example Filter (Pseudocode)

A filter that allows `read`, `write`, `exit`, and `exit_group`, returning
`SECCOMP_RET_KILL` for everything else:

```c
struct sock_filter filter[] = {
    /* Load architecture */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, arch)),
    /* Verify architecture is x86_64 */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, AUDIT_ARCH_X86_64, 1, 0),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),

    /* Load syscall number */
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
             offsetof(struct seccomp_data, nr)),

    /* Allow read (0), write (1), exit (60), exit_group (231) */
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 3, 0),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 2, 0),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 1, 0),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),

    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};

struct sock_fprog prog = {
    .len = (unsigned short)(sizeof(filter) / sizeof(filter[0])),
    .filter = filter,
};
```

**Architecture check**: Filters should always verify `seccomp_data.arch`
before checking `seccomp_data.nr`. On architectures that support multiple
ABIs (e.g., x86_64 running 32-bit binaries), syscall numbers differ between
ABIs. Without an architecture check, a filter could be bypassed by switching
to a different ABI.

---

## 6. SECCOMP_RET Actions

`include/uapi/linux/seccomp.h:38-46`:

```c
#define SECCOMP_RET_KILL_PROCESS 0x80000000U  /* kill the process */
#define SECCOMP_RET_KILL_THREAD  0x00000000U  /* kill the thread */
#define SECCOMP_RET_KILL         SECCOMP_RET_KILL_THREAD
#define SECCOMP_RET_TRAP         0x00030000U  /* disallow and force a SIGSYS */
#define SECCOMP_RET_ERRNO        0x00050000U  /* returns an errno */
#define SECCOMP_RET_USER_NOTIF   0x7fc00000U  /* notifies userspace */
#define SECCOMP_RET_TRACE        0x7ff00000U  /* pass to a tracer or disallow */
#define SECCOMP_RET_LOG          0x7ffc0000U  /* allow after logging */
#define SECCOMP_RET_ALLOW        0x7fff0000U  /* allow */
```

The upper 16 bits encode the action; the lower 16 bits carry optional data
(`SECCOMP_RET_DATA = 0x0000ffffU`). Actions are ordered from most restrictive
(lowest numeric value, since `KILL_PROCESS` is `0x80000000` -- negative as
signed) to least restrictive (`ALLOW` is `0x7fff0000`).

```
┌─────────────────────────────────────────────────────────────┐
│           SECCOMP_RET_* Action Priority                     │
│           (most restrictive to least)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  KILL_PROCESS (0x80000000)  ◄── highest priority (lowest    │
│  KILL_THREAD  (0x00000000)      numeric as signed int)      │
│  TRAP         (0x00030000)                                  │
│  ERRNO        (0x00050000)                                  │
│  USER_NOTIF   (0x7fc00000)                                  │
│  TRACE        (0x7ff00000)                                  │
│  LOG          (0x7ffc0000)                                  │
│  ALLOW        (0x7fff0000)  ◄── lowest priority             │
│                                                             │
│  min_t() over all filter results selects the most           │
│  restrictive outcome.                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Action Details

| Action | Behavior |
|--------|----------|
| `KILL_PROCESS` | Kills entire thread group with `SIGSYS`. Core dump if last thread. |
| `KILL_THREAD` | Kills only the offending thread with `SIGSYS`. Core dump if last thread. |
| `TRAP` | Delivers `SIGSYS` to the thread. The `si_call_addr` is set to the instruction pointer, `si_syscall` to the syscall number, `si_arch` to the architecture. The low 16 bits of the return value are placed in `si_errno`. |
| `ERRNO` | Skips the syscall and returns `-errno` to userspace. The errno value is the low 16 bits of the return value, capped at `MAX_ERRNO`. |
| `USER_NOTIF` | Suspends the thread and sends a notification to a listening userspace supervisor process. See [Section 11](#11-user-notification-mechanism). |
| `TRACE` | If a ptrace tracer is attached with `PTRACE_EVENT_SECCOMP` enabled, notifies it and allows the tracer to modify or skip the syscall. If no tracer is attached, the syscall returns `-ENOSYS`. The low 16 bits are delivered as the ptrace event message. |
| `LOG` | Allows the syscall but emits an audit log entry. |
| `ALLOW` | Allows the syscall to proceed normally. |

`KILL_PROCESS` and `KILL_THREAD` both set the mode to `SECCOMP_MODE_DEAD`
before killing, ensuring no further syscalls can execute
(`kernel/seccomp.c:1319`).

---

## 7. Filter Installation

Filters can be installed via two interfaces:

### prctl(2) Interface

```c
prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);           /* strict mode */
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &fprog);   /* filter mode */
```

The prctl interface does not support flags -- flags are always zero.
Implemented at `kernel/seccomp.c:2102-2127`, `prctl_set_seccomp()` dispatches
to `do_seccomp()`.

### seccomp(2) Syscall

```c
seccomp(SECCOMP_SET_MODE_STRICT, 0, NULL);
seccomp(SECCOMP_SET_MODE_FILTER, flags, &fprog);
seccomp(SECCOMP_GET_ACTION_AVAIL, 0, &action);
seccomp(SECCOMP_GET_NOTIF_SIZES, 0, &sizes);
```

The syscall interface supports flags and additional operations. Defined as
`SYSCALL_DEFINE3` at `kernel/seccomp.c:2089-2093`.

### Syscall Operations

`include/uapi/linux/seccomp.h:15-18`:

```c
#define SECCOMP_SET_MODE_STRICT     0  /* enter strict mode */
#define SECCOMP_SET_MODE_FILTER     1  /* install filter */
#define SECCOMP_GET_ACTION_AVAIL    2  /* check if action is supported */
#define SECCOMP_GET_NOTIF_SIZES     3  /* get notification struct sizes */
```

### Filter Flags

`include/uapi/linux/seccomp.h:21-27`:

```c
#define SECCOMP_FILTER_FLAG_TSYNC              (1UL << 0)
#define SECCOMP_FILTER_FLAG_LOG                (1UL << 1)
#define SECCOMP_FILTER_FLAG_SPEC_ALLOW         (1UL << 2)
#define SECCOMP_FILTER_FLAG_NEW_LISTENER       (1UL << 3)
#define SECCOMP_FILTER_FLAG_TSYNC_ESRCH        (1UL << 4)
#define SECCOMP_FILTER_FLAG_WAIT_KILLABLE_RECV (1UL << 5)
```

| Flag | Purpose |
|------|---------|
| `TSYNC` | Synchronize the filter to all threads in the thread group. |
| `LOG` | Log all actions except `SECCOMP_RET_ALLOW`. |
| `SPEC_ALLOW` | Disable Spectre mitigation for the task (normally enabled when seccomp is active). |
| `NEW_LISTENER` | Create a notification listener FD and return it. Enables `SECCOMP_RET_USER_NOTIF`. |
| `TSYNC_ESRCH` | Return `-ESRCH` instead of the failing thread's PID when TSYNC fails. Allows combining TSYNC + NEW_LISTENER. |
| `WAIT_KILLABLE_RECV` | Once a notification is received by the listener, put the notifying process in killable (not interruptible) sleep. Requires `NEW_LISTENER`. |

### Privilege Requirements

`kernel/seccomp.c:683-685`:

```c
if (!task_no_new_privs(current) &&
        !ns_capable_noaudit(current_user_ns(), CAP_SYS_ADMIN))
    return ERR_PTR(-EACCES);
```

Installing a filter requires either:
1. `no_new_privs` is set (`prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)`), or
2. The task has `CAP_SYS_ADMIN` in its user namespace.

The `no_new_privs` requirement prevents an unprivileged process from installing
a filter that would affect a subsequently exec'd setuid binary, which could be
tricked into doing something unintended if certain syscalls fail unexpectedly.

### Installation Call Flow

```
seccomp(SECCOMP_SET_MODE_FILTER, flags, &fprog)
  └─ do_seccomp()                                   [seccomp.c:2064]
       └─ seccomp_set_mode_filter(flags, filter)     [seccomp.c:1919]
            ├─ seccomp_prepare_user_filter()          [seccomp.c:713]
            │    └─ seccomp_prepare_filter(&fprog)    [seccomp.c:661]
            │         ├─ check no_new_privs or CAP_SYS_ADMIN
            │         ├─ kzalloc(seccomp_filter)
            │         ├─ bpf_prog_create_from_user()  [cBPF → internal BPF]
            │         │    └─ seccomp_check_filter()  [validate instructions]
            │         └─ refcount_set(&refs, 1)
            ├─ [if NEW_LISTENER] init_listener()      [seccomp.c:1854]
            ├─ spin_lock_irq(sighand->siglock)
            ├─ seccomp_attach_filter(flags, filter)   [seccomp.c:889]
            │    ├─ validate total instruction count ≤ MAX_INSNS_PER_PATH
            │    ├─ [if TSYNC] seccomp_can_sync_threads()
            │    ├─ filter->prev = current->seccomp.filter
            │    ├─ seccomp_cache_prepare(filter)
            │    ├─ current->seccomp.filter = filter
            │    ├─ atomic_inc(filter_count)
            │    └─ [if TSYNC] seccomp_sync_threads()
            ├─ seccomp_assign_mode(current, FILTER, flags)
            └─ spin_unlock_irq(sighand->siglock)
```

---

## 8. Filter Evaluation

### Entry Point

At every syscall entry, the architecture-specific code checks if the
`SYSCALL_WORK_SECCOMP` flag is set on the task, then calls:

`include/linux/seccomp.h:27-32`:

```c
static inline int secure_computing(void)
{
    if (unlikely(test_syscall_work(SECCOMP)))
        return __secure_computing(NULL);
    return 0;
}
```

### __secure_computing()

`kernel/seccomp.c:1350-1376`:

```c
int __secure_computing(const struct seccomp_data *sd)
{
    int mode = current->seccomp.mode;

    /* CHECKPOINT_RESTORE: ptrace can suspend seccomp */
    if (IS_ENABLED(CONFIG_CHECKPOINT_RESTORE) &&
        unlikely(current->ptrace & PT_SUSPEND_SECCOMP))
        return 0;

    switch (mode) {
    case SECCOMP_MODE_STRICT:
        __secure_computing_strict(this_syscall);
        return 0;
    case SECCOMP_MODE_FILTER:
        return __seccomp_filter(this_syscall, sd, false);
    case SECCOMP_MODE_DEAD:
        WARN_ON_ONCE(1);
        do_exit(SIGKILL);
        return -1;
    }
}
```

### seccomp_run_filters()

`kernel/seccomp.c:406-434`:

```c
static u32 seccomp_run_filters(const struct seccomp_data *sd,
                               struct seccomp_filter **match)
{
    u32 ret = SECCOMP_RET_ALLOW;
    struct seccomp_filter *f =
            READ_ONCE(current->seccomp.filter);

    if (WARN_ON(f == NULL))
        return SECCOMP_RET_KILL_PROCESS;

    /* Fast path: check action_cache bitmap */
    if (seccomp_cache_check_allow(f, sd))
        return SECCOMP_RET_ALLOW;

    /* Evaluate all filters, keep most restrictive result */
    for (; f; f = f->prev) {
        u32 cur_ret = bpf_prog_run_pin_on_cpu(f->prog, sd);

        if (ACTION_ONLY(cur_ret) < ACTION_ONLY(ret)) {
            ret = cur_ret;
            *match = f;
        }
    }
    return ret;
}
```

Key points:
- Starts with `SECCOMP_RET_ALLOW` as the default.
- Checks the action cache first (fast path for always-allowed syscalls).
- Runs each filter program in the chain from newest to oldest.
- Uses `ACTION_ONLY()` macro (`(s32)((ret) & SECCOMP_RET_ACTION_FULL)`) to
  compare only the action portion as a signed 32-bit value.
- Keeps the lowest (most restrictive) result.

### __seccomp_filter() Action Dispatch

`kernel/seccomp.c:1216-1339` implements the action dispatch:

```
__seccomp_filter(this_syscall, sd, recheck_after_trace)
  ├─ populate_seccomp_data() [if sd == NULL]
  ├─ seccomp_run_filters(sd, &match)
  │
  └─ switch (action):
       ├─ SECCOMP_RET_ERRNO:
       │    syscall_set_return_value(current, regs, -data, 0)
       │    skip syscall
       │
       ├─ SECCOMP_RET_TRAP:
       │    syscall_rollback(current, regs)
       │    force_sig_seccomp(this_syscall, data, false)
       │    skip syscall
       │
       ├─ SECCOMP_RET_TRACE:
       │    [if no tracer] return -ENOSYS
       │    ptrace_event(PTRACE_EVENT_SECCOMP, data)
       │    [recheck filter with modified regs]
       │
       ├─ SECCOMP_RET_USER_NOTIF:
       │    seccomp_do_user_notification(this_syscall, match, sd)
       │
       ├─ SECCOMP_RET_LOG:
       │    seccomp_log(...)
       │    allow syscall
       │
       ├─ SECCOMP_RET_ALLOW:
       │    allow syscall
       │
       └─ SECCOMP_RET_KILL_*:
            mode = SECCOMP_MODE_DEAD
            seccomp_log(...)
            force_sig_seccomp() or do_exit(SIGSYS)
```

---

## 9. Filter Inheritance (fork/exec)

### fork() Inheritance

When a task forks, seccomp state is copied in `copy_seccomp()` at
`kernel/fork.c:1915-1944`:

```c
static void copy_seccomp(struct task_struct *p)
{
    assert_spin_locked(&current->sighand->siglock);

    /* Ref-count the new filter user, and assign it. */
    get_seccomp_filter(current);
    p->seccomp = current->seccomp;

    /* Ensure no_new_privs is in sync */
    if (task_no_new_privs(current))
        task_set_no_new_privs(p);

    if (p->seccomp.mode != SECCOMP_MODE_DISABLED)
        set_task_syscall_work(p, SECCOMP);
}
```

The child gets a reference to the **same** filter chain as the parent. Both
`refs` and `users` reference counts on the filter are incremented. The child
then shares the entire filter tree with the parent. If the child subsequently
installs new filters, they are prepended to the chain, creating a divergence
point.

### exec() Behavior

Seccomp filters **persist across exec()**. The `execve` path does not clear
or modify seccomp state. This is by design: a sandboxed process cannot escape
its sandbox by executing a new binary.

Combined with `no_new_privs`, this means:
- A seccomp-filtered process cannot exec a setuid binary to gain privileges.
- The filter continues to apply to the new program image.
- Since filters cannot dereference pointers, the new program's arguments are
  opaque to the filter (only raw pointer values are visible).

```
┌─────────────────────────────────────────────────────────────┐
│              Filter Inheritance                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Parent Process                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  seccomp.filter → [F2] → [F1] → NULL               │    │
│  └────────┬────────────────────────────────────────────┘    │
│           │ fork()                                          │
│           ▼                                                 │
│  Child Process                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  seccomp.filter → [F2] → [F1] → NULL               │    │
│  │                    ↑ shared (refcount incremented)   │    │
│  └────────┬────────────────────────────────────────────┘    │
│           │ seccomp(SET_MODE_FILTER, 0, &new_filter)        │
│           ▼                                                 │
│  Child (after new filter)                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  seccomp.filter → [F3] → [F2] → [F1] → NULL        │    │
│  │                           ↑ still shared with parent │    │
│  └────────┬────────────────────────────────────────────┘    │
│           │ exec()                                          │
│           ▼                                                 │
│  Child (after exec)                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  seccomp.filter → [F3] → [F2] → [F1] → NULL        │    │
│  │  (filters persist — no escape via exec)              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Thread Synchronization (TSYNC)

The `SECCOMP_FILTER_FLAG_TSYNC` flag causes a newly installed filter to be
propagated to all threads in the thread group. This is critical for
multi-threaded applications where you want consistent syscall filtering
across all threads.

### Validation

`seccomp_can_sync_threads()` at `kernel/seccomp.c:490-524` checks that every
thread either:
- Has `SECCOMP_MODE_DISABLED`, or
- Is in `SECCOMP_MODE_FILTER` with an ancestral filter chain (the thread's
  current filter must be an ancestor of the caller's filter).

Threads that are exiting (`PF_EXITING`) are skipped. If any thread fails
validation, its PID is returned (or `-ESRCH` if `TSYNC_ESRCH` is set).

### Synchronization

`seccomp_sync_threads()` at `kernel/seccomp.c:597-653` iterates all threads
via `for_each_thread()` and:

1. Increments the reference count on the caller's filter for each synced thread.
2. Releases the thread's old filter tree.
3. Replaces the thread's filter pointer with the caller's filter (via
   `smp_store_release()`).
4. Propagates `no_new_privs` if the caller has it set.
5. If the thread was in `SECCOMP_MODE_DISABLED`, transitions it to
   `SECCOMP_MODE_FILTER`.

Both `sighand->siglock` and `cred_guard_mutex` must be held during
synchronization to prevent races with `exec()`.

### TSYNC + NEW_LISTENER Interaction

`TSYNC` and `NEW_LISTENER` can be combined, but only if `TSYNC_ESRCH` is
also set. Without `TSYNC_ESRCH`, the return value is ambiguous: on success
`NEW_LISTENER` returns a file descriptor, but on failure `TSYNC` returns a
PID -- both are positive integers (`kernel/seccomp.c:1939-1942`).

---

## 11. User Notification Mechanism

The user notification mechanism (`SECCOMP_RET_USER_NOTIF`) allows a supervisor
process to intercept and handle syscalls on behalf of a sandboxed process.
This is used by container runtimes to emulate syscalls that would otherwise
be blocked (e.g., `mount`, `mknod`).

### Setup

1. Install a filter with `SECCOMP_FILTER_FLAG_NEW_LISTENER`:

```c
int listener_fd = seccomp(SECCOMP_SET_MODE_FILTER,
                          SECCOMP_FILTER_FLAG_NEW_LISTENER,
                          &fprog);
```

2. The returned `listener_fd` is an anonymous inode backed by
   `seccomp_notify_ops` (`kernel/seccomp.c:1847-1852`):

```c
static const struct file_operations seccomp_notify_ops = {
    .poll = seccomp_notify_poll,
    .release = seccomp_notify_release,
    .unlocked_ioctl = seccomp_notify_ioctl,
    .compat_ioctl = seccomp_notify_ioctl,
};
```

### Notification Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                 User Notification Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sandboxed Process              Supervisor Process              │
│  ─────────────────              ────────────────────            │
│                                                                 │
│  1. syscall() ──────────────►                                   │
│                                                                 │
│  2. Filter returns                                              │
│     SECCOMP_RET_USER_NOTIF                                      │
│                                                                 │
│  3. seccomp_do_user_notification()                              │
│     - creates seccomp_knotif                                    │
│     - adds to notifications list                                │
│     - wake_up_poll(wqh)         4. poll()/epoll() wakes up      │
│     - wait_for_completion()  ◄─────                             │
│       (TASK_INTERRUPTIBLE)                                      │
│                                 5. ioctl(SECCOMP_IOCTL_NOTIF_RECV)
│                                    - reads seccomp_notif:       │
│                                      { id, pid, data }         │
│                                    - state: INIT → SENT         │
│                                                                 │
│                                 6. Supervisor examines syscall, │
│                                    performs action on behalf     │
│                                    of sandboxed process.        │
│                                                                 │
│                                 7. [Optional] ioctl(NOTIF_ADDFD)│
│                                    - inject fd into target      │
│                                                                 │
│                                 8. ioctl(SECCOMP_IOCTL_NOTIF_SEND)
│                                    - sends seccomp_notif_resp:  │
│                                      { id, val, error, flags }  │
│                                    - state: SENT → REPLIED      │
│                                                                 │
│  9. complete(&knotif->ready) ◄─────                             │
│     Thread wakes up, uses                                       │
│     reply values as syscall                                     │
│     return.                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### ioctl Commands

`include/uapi/linux/seccomp.h:147-155`:

| ioctl | Direction | Purpose |
|-------|-----------|---------|
| `SECCOMP_IOCTL_NOTIF_RECV` | Read | Receive a pending notification. Blocks if none available. |
| `SECCOMP_IOCTL_NOTIF_SEND` | Write | Send a response to a notification. |
| `SECCOMP_IOCTL_NOTIF_ID_VALID` | Write | Check if a notification ID is still valid and pending. |
| `SECCOMP_IOCTL_NOTIF_ADDFD` | Write | Inject a file descriptor into the notifying process. |
| `SECCOMP_IOCTL_NOTIF_SET_FLAGS` | Write | Set notification FD flags (e.g., `SYNC_WAKE_UP`). |

### Notification Structures (UAPI)

```c
/* include/uapi/linux/seccomp.h:75-80 */
struct seccomp_notif {
    __u64 id;                  /* unique notification cookie */
    __u32 pid;                 /* PID of notifying process */
    __u32 flags;
    struct seccomp_data data;  /* syscall info */
};

/* include/uapi/linux/seccomp.h:111-116 */
struct seccomp_notif_resp {
    __u64 id;                  /* must match the notification id */
    __s64 val;                 /* syscall return value */
    __s32 error;               /* errno (negative) or 0 */
    __u32 flags;               /* SECCOMP_USER_NOTIF_FLAG_* */
};

/* include/uapi/linux/seccomp.h:132-138 */
struct seccomp_notif_addfd {
    __u64 id;                  /* notification ID */
    __u32 flags;               /* SECCOMP_ADDFD_FLAG_* */
    __u32 srcfd;               /* fd in supervisor's table */
    __u32 newfd;               /* target fd number (if SETFD) */
    __u32 newfd_flags;         /* O_CLOEXEC etc. */
};
```

### SECCOMP_USER_NOTIF_FLAG_CONTINUE

`include/uapi/linux/seccomp.h:109`:

```c
#define SECCOMP_USER_NOTIF_FLAG_CONTINUE (1UL << 0)
```

When set in the response flags, the syscall is allowed to proceed as if the
filter returned `SECCOMP_RET_ALLOW`. **Warning**: This introduces a TOCTOU
vulnerability -- the notified process could have modified its memory (e.g.,
syscall arguments that are pointers) between when the notification was created
and when the response is sent. This flag should only be used when another
security mechanism will validate the actual syscall arguments.

### Duplicate Listener Prevention

`has_duplicate_listener()` at `kernel/seccomp.c:1889-1903` prevents installing
a filter with `NEW_LISTENER` when an ancestor filter already has a listener.
This avoids ambiguity about which listener handles a notification.

### Wait Killable

When `SECCOMP_FILTER_FLAG_WAIT_KILLABLE_RECV` is set and the notification has
been received by the listener (state `SECCOMP_NOTIFY_SENT`), the notifying
process sleeps in `TASK_KILLABLE` instead of `TASK_INTERRUPTIBLE`. This means
only fatal signals can interrupt the wait, preventing spurious `EINTR` returns
that would require the notification to be re-sent.

---

## 12. Interaction with ptrace

### SECCOMP_RET_TRACE

When a filter returns `SECCOMP_RET_TRACE`, seccomp cooperates with ptrace to
allow a tracer to inspect and modify the syscall. At `kernel/seccomp.c:1255-1296`:

1. If no tracer is attached (with `PTRACE_EVENT_SECCOMP` enabled), the syscall
   returns `-ENOSYS`.
2. Otherwise, `ptrace_event(PTRACE_EVENT_SECCOMP, data)` is called, stopping
   the tracee and notifying the tracer. The low 16 bits of the filter's return
   value are delivered as the ptrace event message.
3. The tracer can inspect/modify registers (including changing the syscall
   number or arguments, or setting the syscall number to -1 to skip it).
4. After the tracer resumes the tracee, the filters are **re-evaluated** with
   the potentially modified syscall data. This prevents the tracer from
   undermining filter restrictions.

### PT_SUSPEND_SECCOMP (Checkpoint/Restore)

When `CONFIG_CHECKPOINT_RESTORE` is enabled, a privileged ptrace tracer can
set `PT_SUSPEND_SECCOMP` on a traced process, completely bypassing seccomp
checks. This is used by CRIU (Checkpoint/Restore In Userspace) to restore
processes that have seccomp filters installed -- the restore process needs
to make syscalls that would be blocked by the restored filters.

`kernel/seccomp.c:1355-1357`:

```c
if (IS_ENABLED(CONFIG_CHECKPOINT_RESTORE) &&
    unlikely(current->ptrace & PT_SUSPEND_SECCOMP))
    return 0;
```

---

## 13. Logging and Auditing

### seccomp_log()

`kernel/seccomp.c:976-1017`:

The `seccomp_log()` function decides whether to emit an audit record for a
given action. Logging is controlled by two factors:

1. **Per-filter flag**: If the filter was installed with
   `SECCOMP_FILTER_FLAG_LOG`, all actions except `ALLOW` are logged (the
   `requested` parameter is `true` for the matching filter's `log` field).

2. **Global sysctl**: The `seccomp_actions_logged` bitmask
   (`kernel/seccomp.c:968-974`) controls which actions are eligible for
   logging system-wide.

`SECCOMP_RET_KILL_PROCESS` and `SECCOMP_RET_KILL_THREAD` are always logged
(regardless of the per-filter flag) as long as the sysctl permits.
`SECCOMP_RET_ALLOW` is never logged by default (and the sysctl rejects
attempts to enable it via `kernel/seccomp.c:2395`).

The actual log emission uses `audit_seccomp()` from the kernel audit
subsystem.

### Default Logged Actions

```c
static u32 seccomp_actions_logged = SECCOMP_LOG_KILL_PROCESS |
                                    SECCOMP_LOG_KILL_THREAD  |
                                    SECCOMP_LOG_TRAP  |
                                    SECCOMP_LOG_ERRNO |
                                    SECCOMP_LOG_USER_NOTIF |
                                    SECCOMP_LOG_TRACE |
                                    SECCOMP_LOG_LOG;
```

Note that `SECCOMP_LOG_ALLOW` is intentionally excluded.

---

## 14. Action Cache (Performance Optimization)

The action cache is a per-filter bitmap that records which syscalls will
**always** result in `SECCOMP_RET_ALLOW` for a given architecture. If a
syscall is marked as always-allow in the cache, the BPF program does not
need to run at all.

### struct action_cache

`kernel/seccomp.c:170-175`:

```c
struct action_cache {
    DECLARE_BITMAP(allow_native, SECCOMP_ARCH_NATIVE_NR);
#ifdef SECCOMP_ARCH_COMPAT
    DECLARE_BITMAP(allow_compat, SECCOMP_ARCH_COMPAT_NR);
#endif
};
```

### Cache Construction

`seccomp_cache_prepare()` at `kernel/seccomp.c:857-874` is called when a
filter is attached. For each syscall number:

1. If the parent filter's cache already marks the syscall as not-always-allow,
   the new cache inherits that (the cache can only become more restrictive).
2. Otherwise, `seccomp_is_const_allow()` (`kernel/seccomp.c:742-813`)
   symbolically executes the BPF program with only the syscall number and
   architecture as known values. If the program always returns
   `SECCOMP_RET_ALLOW` regardless of other fields, the syscall is marked
   as cacheable.

### Cache Check

`seccomp_cache_check_allow()` at `kernel/seccomp.c:369-393` is the fast path
in `seccomp_run_filters()`. It checks the bitmap for the current architecture
and syscall number. Only the **newest** filter's cache needs to be checked,
since it already incorporates the constraints of all ancestor filters.

```
┌─────────────────────────────────────────────────────────────┐
│              Filter Evaluation Fast Path                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  seccomp_run_filters(sd, &match)                            │
│    │                                                        │
│    ├─ seccomp_cache_check_allow(f, sd)                      │
│    │    └─ test_bit(syscall_nr, cache->allow_native)        │
│    │         │                                              │
│    │         ├─ HIT (bit set):                              │
│    │         │    return SECCOMP_RET_ALLOW immediately       │
│    │         │    (no BPF programs run)                      │
│    │         │                                              │
│    │         └─ MISS (bit clear):                           │
│    │              fall through to BPF evaluation            │
│    │                                                        │
│    └─ for each filter f:                                    │
│         bpf_prog_run_pin_on_cpu(f->prog, sd)               │
│         keep most restrictive result                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

This optimization is particularly effective for common syscalls like `read`,
`write`, `close`, etc., which most filters unconditionally allow. It avoids
the overhead of running BPF programs on the hot path for these syscalls.

---

## 15. Sysctl and Proc Interfaces

### Sysctl: /proc/sys/kernel/seccomp/

Registered at `kernel/seccomp.c:2453-2474`:

| Sysctl Path | Mode | Description |
|-------------|------|-------------|
| `kernel/seccomp/actions_avail` | 0444 | Read-only list of supported actions (space-separated). |
| `kernel/seccomp/actions_logged` | 0644 | Read/write list of actions that produce audit log entries. |

`actions_avail` is a static string listing all supported return actions:

```
kill_process kill_thread trap errno user_notif trace log allow
```

`actions_logged` can be modified by the administrator to control which actions
produce log entries. Setting `allow` in `actions_logged` is rejected
(`kernel/seccomp.c:2395`).

### Proc: /proc/<pid>/seccomp_cache

Available when `CONFIG_SECCOMP_CACHE_DEBUG` is enabled.
`proc_pid_seccomp_cache()` at `kernel/seccomp.c:2493-2531` outputs the
action cache bitmap for a process, showing which syscalls are cached as
always-allow:

```
NativeArch  0  ALLOW
NativeArch  1  ALLOW
NativeArch  2  FILTER
...
```

This is useful for debugging filter performance and understanding which
syscalls benefit from the cache fast path.

---

## 16. Key Source Files

| File | Purpose |
|------|---------|
| `kernel/seccomp.c` | Core implementation: filter management, evaluation, notification, sysctl (~2,500 lines) |
| `include/linux/seccomp.h` | Internal kernel API: `__secure_computing()`, `secure_computing()`, `prctl_get/set_seccomp()`, flag mask |
| `include/linux/seccomp_types.h` | `struct seccomp` definition (embedded in `task_struct`) |
| `include/uapi/linux/seccomp.h` | User-facing API: modes, operations, flags, `SECCOMP_RET_*`, `seccomp_data`, notification structs, ioctl commands |
| `kernel/fork.c:1915` | `copy_seccomp()` -- filter inheritance across `fork()` |
| `net/core/filter.c` | `bpf_prog_create_from_user()` -- cBPF-to-internal-BPF conversion |
| `tools/testing/selftests/seccomp/seccomp_bpf.c` | Comprehensive selftest suite (~4,900 lines) |

### Key Functions Reference

| Function | Location | Purpose |
|----------|----------|---------|
| `__secure_computing()` | `kernel/seccomp.c:1350` | Main entry point from syscall path |
| `__seccomp_filter()` | `kernel/seccomp.c:1216` | Filter mode dispatch: run filters, act on result |
| `seccomp_run_filters()` | `kernel/seccomp.c:406` | Run all filters, return most restrictive result |
| `seccomp_check_filter()` | `kernel/seccomp.c:280` | Validate cBPF instructions are seccomp-safe |
| `seccomp_prepare_filter()` | `kernel/seccomp.c:661` | Allocate and initialize a filter |
| `seccomp_attach_filter()` | `kernel/seccomp.c:889` | Attach filter to task, validate total size, sync threads |
| `seccomp_set_mode_strict()` | `kernel/seccomp.c:1391` | Enter strict mode |
| `seccomp_set_mode_filter()` | `kernel/seccomp.c:1919` | Install a new filter (with flag handling) |
| `do_seccomp()` | `kernel/seccomp.c:2064` | Common entry for both prctl and syscall |
| `prctl_set_seccomp()` | `kernel/seccomp.c:2102` | prctl(PR_SET_SECCOMP) handler |
| `seccomp_do_user_notification()` | `kernel/seccomp.c:1118` | Handle SECCOMP_RET_USER_NOTIF: suspend thread, wait for reply |
| `seccomp_notify_recv()` | `kernel/seccomp.c:1515` | NOTIF_RECV ioctl: dequeue notification |
| `seccomp_notify_send()` | `kernel/seccomp.c:1588` | NOTIF_SEND ioctl: send response |
| `seccomp_notify_addfd()` | `kernel/seccomp.c:1675` | NOTIF_ADDFD ioctl: inject fd into target |
| `seccomp_can_sync_threads()` | `kernel/seccomp.c:490` | Validate all threads are eligible for TSYNC |
| `seccomp_sync_threads()` | `kernel/seccomp.c:597` | Propagate filter to all threads |
| `seccomp_cache_prepare()` | `kernel/seccomp.c:857` | Build action cache bitmap |
| `seccomp_cache_check_allow()` | `kernel/seccomp.c:369` | Fast-path cache check |
| `seccomp_is_const_allow()` | `kernel/seccomp.c:742` | Symbolic execution to determine cacheable syscalls |
| `populate_seccomp_data()` | `kernel/seccomp.c:246` | Fill seccomp_data from current register state |
| `copy_seccomp()` | `kernel/fork.c:1915` | Copy seccomp state to child on fork |
| `seccomp_filter_release()` | `kernel/seccomp.c:572` | Release filter tree on task exit |
| `seccomp_log()` | `kernel/seccomp.c:976` | Conditionally emit audit log for an action |
| `init_listener()` | `kernel/seccomp.c:1854` | Create notification listener anonymous inode |
| `seccomp_notify_ioctl()` | `kernel/seccomp.c:1789` | Dispatch notification ioctl commands |

### Kconfig Options

| Config | Description |
|--------|-------------|
| `CONFIG_SECCOMP` | Enable seccomp (strict mode) |
| `CONFIG_SECCOMP_FILTER` | Enable seccomp filter mode (requires `CONFIG_HAVE_ARCH_SECCOMP_FILTER`) |
| `CONFIG_SECCOMP_CACHE_DEBUG` | Expose `/proc/<pid>/seccomp_cache` for debugging the action cache |
| `CONFIG_CHECKPOINT_RESTORE` | Enable `PT_SUSPEND_SECCOMP` for CRIU support |

---

## Summary

Seccomp provides a robust, efficient mechanism for syscall sandboxing:

| Component | Purpose |
|-----------|---------|
| Strict mode | Minimal syscall allow-list (read/write/exit/sigreturn) |
| Filter mode | Arbitrary cBPF programs evaluate `seccomp_data` |
| `SECCOMP_RET_*` actions | Graduated responses from ALLOW to KILL_PROCESS |
| Filter chain | Stacked filters evaluated via min() semantics |
| Action cache | Bitmap fast-path avoids BPF execution for always-allowed syscalls |
| User notification | Supervisor process can intercept and emulate syscalls |
| TSYNC | Atomic filter propagation to all threads |
| fork inheritance | Child inherits parent's filter chain (shared via refcount) |
| exec persistence | Filters survive exec, cannot be removed |
| ptrace integration | `RET_TRACE` cooperates with debuggers, `PT_SUSPEND_SECCOMP` for CRIU |
| Logging | Per-filter and global sysctl control of audit log emission |
