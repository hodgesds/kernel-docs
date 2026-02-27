# Linux Kernel Signals Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Signal Types](#signal-types)
3. [Core Data Structures](#core-data-structures)
4. [Signal Masks and Pending Sets](#signal-masks-and-pending-sets)
5. [Signal Generation and Sending](#signal-generation-and-sending)
6. [Signal Queuing](#signal-queuing)
7. [Signal Delivery and Dequeuing](#signal-delivery-and-dequeuing)
8. [Signal Handling (sigaction)](#signal-handling-sigaction)
9. [Architecture-Specific Signal Frame Setup](#architecture-specific-signal-frame-setup)
10. [Process Groups and Job Control Signals](#process-groups-and-job-control-signals)
11. [Core Dump Generation](#core-dump-generation)
12. [Ptrace Interaction](#ptrace-interaction)
13. [signalfd](#signalfd)
14. [System Call Restart](#system-call-restart)
15. [Locking Summary](#locking-summary)
16. [Source File Reference](#source-file-reference)

---

## Introduction

Signals are the oldest inter-process communication mechanism in Unix and remain
a fundamental part of the Linux kernel. They provide asynchronous notification
to processes about events such as hardware exceptions, timer expiration, terminal
input, and explicit requests from other processes or the kernel itself.

The Linux signal subsystem must handle:
- Generating and queuing signals to individual threads or thread groups
- Delivering signals when a process returns to user space
- Setting up architecture-specific stack frames for user-space handlers
- Job control (stop/continue) via signal groups
- Core dump generation for fatal signals
- Interaction with ptrace for debuggers
- Real-time signal semantics (queuing, ordering, priorities)

### High-Level Signal Flow

```
+-------------------------------------------------------------------+
|                    Signal Lifecycle                                |
+-------------------------------------------------------------------+
|                                                                   |
|  1. GENERATION                                                    |
|     kill()/tkill()/tgkill()/raise()/kernel fault                  |
|          |                                                        |
|          v                                                        |
|  2. SENDING  (send_signal_locked)                                 |
|     - Permission checks                                           |
|     - Allocate sigqueue (for RT signals)                           |
|     - Add to pending set (per-thread or shared)                    |
|     - Set TIF_SIGPENDING                                          |
|     - Wake up target thread                                       |
|          |                                                        |
|          v                                                        |
|  3. DELIVERY  (get_signal, on return to user space)               |
|     - Dequeue from pending sets                                    |
|     - Ptrace interception                                         |
|     - Check handler (SIG_IGN, SIG_DFL, user handler)              |
|          |                                                        |
|          +-----> SIG_IGN: discard                                  |
|          +-----> SIG_DFL: default action (ignore/term/core/stop)   |
|          +-----> User handler:                                     |
|                    |                                               |
|                    v                                               |
|  4. HANDLER SETUP  (arch-specific: setup_rt_frame)                |
|     - Save registers + signal mask on user stack                   |
|     - Set up return trampoline (sigreturn)                         |
|     - Redirect execution to handler                                |
|          |                                                        |
|          v                                                        |
|  5. HANDLER RETURN  (sys_rt_sigreturn)                            |
|     - Restore registers + signal mask                              |
|     - Resume interrupted execution                                 |
|                                                                   |
+-------------------------------------------------------------------+
```

---

## Signal Types

### Standard Signals (1-31)

Standard (also called "regular" or "legacy") signals are defined in
`include/uapi/asm-generic/signal.h`. Each signal number can have at most one
instance pending at a time -- sending a signal that is already pending is
silently dropped.

```c
/* include/uapi/asm-generic/signal.h:11-47 */
#define SIGHUP       1
#define SIGINT       2
#define SIGQUIT      3
#define SIGILL       4
#define SIGTRAP      5
#define SIGABRT      6
#define SIGIOT       6
#define SIGBUS       7
#define SIGFPE       8
#define SIGKILL      9
#define SIGUSR1     10
#define SIGSEGV     11
#define SIGUSR2     12
#define SIGPIPE     13
#define SIGALRM     14
#define SIGTERM     15
#define SIGSTKFLT   16
#define SIGCHLD     17
#define SIGCONT     18
#define SIGSTOP     19
#define SIGTSTP     20
#define SIGTTIN     21
#define SIGTTOU     22
#define SIGURG      23
#define SIGXCPU     24
#define SIGXFSZ     25
#define SIGVTALRM   26
#define SIGPROF     27
#define SIGWINCH    28
#define SIGIO       29
#define SIGPWR      30
#define SIGSYS      31
```

### Real-Time Signals (32-64)

```c
/* include/uapi/asm-generic/signal.h:50-53 */
#define SIGRTMIN    32
#define SIGRTMAX    _NSIG    /* 64 */
```

Real-time signals differ from standard signals in important ways:
- **Multiple instances can be queued** (not just a single pending bit)
- **Delivery is ordered**: lower-numbered RT signals are delivered first
- **Carry data**: `sigqueue()` can attach an `si_value` (int or pointer)
- **Queue overflow returns EAGAIN** rather than silently dropping

### Default Actions Table

The kernel classifies signal default actions using bitmasks defined in
`include/linux/signal.h:419-448`:

```
+--------------------+------------------+
|  POSIX signal      |  default action  |
+--------------------+------------------+
|  SIGHUP            |  terminate       |
|  SIGINT            |  terminate       |
|  SIGQUIT           |  coredump        |
|  SIGILL            |  coredump        |
|  SIGTRAP           |  coredump        |
|  SIGABRT/SIGIOT    |  coredump        |
|  SIGBUS            |  coredump        |
|  SIGFPE            |  coredump        |
|  SIGKILL           |  terminate(+)    |
|  SIGUSR1           |  terminate       |
|  SIGSEGV           |  coredump        |
|  SIGUSR2           |  terminate       |
|  SIGPIPE           |  terminate       |
|  SIGALRM           |  terminate       |
|  SIGTERM           |  terminate       |
|  SIGCHLD           |  ignore          |
|  SIGCONT           |  ignore(*)       |
|  SIGSTOP           |  stop(*)(+)      |
|  SIGTSTP           |  stop(*)         |
|  SIGTTIN           |  stop(*)         |
|  SIGTTOU           |  stop(*)         |
|  SIGURG            |  ignore          |
|  SIGXCPU           |  coredump        |
|  SIGXFSZ           |  coredump        |
|  SIGVTALRM         |  terminate       |
|  SIGPROF           |  terminate       |
|  SIGPOLL/SIGIO     |  terminate       |
|  SIGSYS/SIGUNUSED  |  coredump        |
|  SIGRTMIN-SIGRTMAX |  terminate       |
+--------------------+------------------+
(+) Cannot be caught, blocked, or ignored
(*) Special job control effects
```

### Classification Bitmasks

```c
/* include/linux/signal.h:419-448 */
#define SIG_KERNEL_ONLY_MASK    (rt_sigmask(SIGKILL) | rt_sigmask(SIGSTOP))

#define SIG_KERNEL_STOP_MASK    (rt_sigmask(SIGSTOP) | rt_sigmask(SIGTSTP) | \
                                 rt_sigmask(SIGTTIN) | rt_sigmask(SIGTTOU))

#define SIG_KERNEL_COREDUMP_MASK (rt_sigmask(SIGQUIT) | rt_sigmask(SIGILL)  | \
                                  rt_sigmask(SIGTRAP) | rt_sigmask(SIGABRT) | \
                                  rt_sigmask(SIGFPE)  | rt_sigmask(SIGSEGV) | \
                                  rt_sigmask(SIGBUS)  | rt_sigmask(SIGSYS)  | \
                                  rt_sigmask(SIGXCPU) | rt_sigmask(SIGXFSZ) | \
                                  SIGEMT_MASK)

#define SIG_KERNEL_IGNORE_MASK  (rt_sigmask(SIGCONT) | rt_sigmask(SIGCHLD) | \
                                 rt_sigmask(SIGWINCH)| rt_sigmask(SIGURG))
```

Helper macros test a signal against these masks:

```c
#define sig_kernel_only(sig)     siginmask(sig, SIG_KERNEL_ONLY_MASK)
#define sig_kernel_coredump(sig) siginmask(sig, SIG_KERNEL_COREDUMP_MASK)
#define sig_kernel_ignore(sig)   siginmask(sig, SIG_KERNEL_IGNORE_MASK)
#define sig_kernel_stop(sig)     siginmask(sig, SIG_KERNEL_STOP_MASK)
```

### Synchronous Signals

Certain signals are considered synchronous because they are generated by the
CPU in response to the currently executing instruction:

```c
/* kernel/signal.c:198-200 */
#define SYNCHRONOUS_MASK \
    (sigmask(SIGSEGV) | sigmask(SIGBUS) | sigmask(SIGILL) | \
     sigmask(SIGTRAP) | sigmask(SIGFPE) | sigmask(SIGSYS))
```

These are prioritized during dequeuing so that the instruction pointer in the
signal frame points to the faulting instruction.

---

## Core Data Structures

### sigset_t -- Signal Set (Bitmask)

The fundamental type for representing a set of signals is a bitmask:

```c
/* include/uapi/asm-generic/signal.h:7-9,61-63 */
#define _NSIG       64
#define _NSIG_BPW   __BITS_PER_LONG
#define _NSIG_WORDS (_NSIG / _NSIG_BPW)

typedef struct {
    unsigned long sig[_NSIG_WORDS];
} sigset_t;
```

On 64-bit systems, `_NSIG_WORDS` is 1 (a single `unsigned long` holds all 64
bits). On 32-bit systems, `_NSIG_WORDS` is 2.

### sigpending -- Pending Signal Set with Queue

```c
/* include/linux/signal_types.h:32-35 */
struct sigpending {
    struct list_head list;      /* linked list of sigqueue entries */
    sigset_t signal;            /* bitmask of pending signals */
};
```

The `signal` bitmask provides O(1) lookup of whether a signal is pending. The
`list` contains `sigqueue` entries that carry the `siginfo` data for each
queued signal instance.

### sigqueue -- Queued Signal Entry

```c
/* include/linux/signal_types.h:22-27 */
struct sigqueue {
    struct list_head list;
    int flags;
    kernel_siginfo_t info;
    struct ucounts *ucounts;
};

#define SIGQUEUE_PREALLOC   1
```

Each `sigqueue` carries the full `kernel_siginfo_t` with signal metadata
(sender PID, UID, si_code, etc.). The `ucounts` pointer tracks per-user
resource limits to enforce `RLIMIT_SIGPENDING`.

### kernel_siginfo_t -- Signal Information

```c
/* include/linux/signal_types.h:12-14 */
typedef struct kernel_siginfo {
    __SIGINFO;
} kernel_siginfo_t;
```

The `__SIGINFO` macro expands to fields including `si_signo`, `si_errno`,
`si_code`, and a union of type-specific data (kill, timer, rt, chld, fault,
poll, sys).

### sigaction / k_sigaction -- Signal Action

```c
/* include/linux/signal_types.h:37-56 */
struct sigaction {
    __sighandler_t  sa_handler;
    unsigned long   sa_flags;
#ifdef __ARCH_HAS_SA_RESTORER
    __sigrestore_t sa_restorer;
#endif
    sigset_t        sa_mask;    /* mask last for extensibility */
};

struct k_sigaction {
    struct sigaction sa;
#ifdef __ARCH_HAS_KA_RESTORER
    __sigrestore_t ka_restorer;
#endif
};
```

The kernel wraps `sigaction` in `k_sigaction` to allow arch-specific extensions.
Key `sa_flags` values include:

| Flag | Meaning |
|------|---------|
| `SA_SIGINFO` | Handler receives `siginfo_t` and `ucontext_t` |
| `SA_RESTART` | Automatically restart interrupted syscalls |
| `SA_NODEFER` | Do not block the signal inside its own handler |
| `SA_RESETHAND` / `SA_ONESHOT` | Reset handler to SIG_DFL after delivery |
| `SA_ONSTACK` | Use alternate signal stack (`sigaltstack`) |
| `SA_NOCLDSTOP` | Do not generate SIGCHLD for child stops |
| `SA_NOCLDWAIT` | Do not create zombie on child exit |
| `SA_RESTORER` | `sa_restorer` points to signal return trampoline |
| `SA_IMMUTABLE` | Internal: handler cannot be changed (forced signals) |

### ksignal -- Signal Being Delivered

```c
/* include/linux/signal_types.h:67-71 */
struct ksignal {
    struct k_sigaction ka;
    kernel_siginfo_t info;
    int sig;
};
```

This bundles the signal number, its `siginfo`, and the action to take. It is
populated by `get_signal()` and consumed by the arch-specific signal delivery
code.

### sighand_struct -- Per-Process Signal Handlers

```c
/* include/linux/sched/signal.h:21-26 */
struct sighand_struct {
    spinlock_t          siglock;
    refcount_t          count;
    wait_queue_head_t   signalfd_wqh;
    struct k_sigaction  action[_NSIG];
};
```

The `action[]` array holds the handler configuration for all 64 signals.
The `siglock` spinlock protects all signal state for the thread group.
`signalfd_wqh` is the wait queue for `signalfd` poll notifications.

Threads created with `CLONE_SIGHAND` share the same `sighand_struct`.

### signal_struct -- Per-Thread-Group Signal State

```c
/* include/linux/sched/signal.h:94-122 */
struct signal_struct {
    refcount_t          sigcnt;
    atomic_t            live;
    int                 nr_threads;
    int                 quick_threads;
    struct list_head    thread_head;

    wait_queue_head_t   wait_chldexit;      /* for wait4() */

    struct task_struct  *curr_target;        /* signal load-balancing */

    struct sigpending   shared_pending;      /* thread-group-wide pending */

    int                 group_exit_code;
    int                 notify_count;
    struct task_struct  *group_exec_task;

    int                 group_stop_count;    /* job control */
    unsigned int        flags;               /* SIGNAL_* flags */

    struct core_state   *core_state;         /* coredumping support */
    /* ... */
};
```

### Signal-Related Fields in task_struct

```c
/* include/linux/sched.h:1171-1180 */
struct task_struct {
    /* ... */
    struct signal_struct     *signal;        /* shared thread group state */
    struct sighand_struct    *sighand;       /* shared signal handlers */
    sigset_t                 blocked;        /* currently blocked signals */
    sigset_t                 real_blocked;   /* saved blocked during sigsuspend */
    struct sigpending        pending;        /* per-thread pending signals */
    unsigned long            sas_ss_sp;      /* alternate signal stack addr */
    size_t                   sas_ss_size;    /* alternate signal stack size */
    unsigned int             sas_ss_flags;   /* alternate stack flags */
    /* ... */
};
```

### Relationship Diagram

```
task_struct (per thread)
+-----------------------------+
| signal ─────────────────────────> signal_struct (shared by thread group)
|                             |     +----------------------------+
| sighand ────────────────────────> | shared_pending             |
|                             |     |   .signal (sigset_t)       |
| blocked    (sigset_t)       |     |   .list (sigqueue chain)   |
| real_blocked (sigset_t)     |     | group_stop_count           |
|                             |     | group_exit_code            |
| pending (per-thread)        |     | flags (SIGNAL_*)           |
|   .signal (sigset_t)        |     | core_state                 |
|   .list (sigqueue chain)    |     | curr_target                |
|                             |     +----------------------------+
| sas_ss_sp                   |
| sas_ss_size                 |     sighand_struct (shared by threads)
+-----------------------------+     +----------------------------+
                                    | siglock (spinlock)         |
                                    | action[_NSIG]              |
                                    |   (k_sigaction array)      |
                                    | signalfd_wqh               |
                                    +----------------------------+
```

### SIGNAL_* Flags

```c
/* include/linux/sched/signal.h:255-267 */
#define SIGNAL_STOP_STOPPED     0x00000001  /* job control stop in effect */
#define SIGNAL_STOP_CONTINUED   0x00000002  /* SIGCONT since WCONTINUED reap */
#define SIGNAL_GROUP_EXIT       0x00000004  /* group exit in progress */
#define SIGNAL_CLD_STOPPED      0x00000010
#define SIGNAL_CLD_CONTINUED    0x00000020
#define SIGNAL_CLD_MASK         (SIGNAL_CLD_STOPPED|SIGNAL_CLD_CONTINUED)
#define SIGNAL_UNKILLABLE       0x00000040  /* for init: ignore fatal signals */
```

---

## Signal Masks and Pending Sets

### Per-Thread vs Shared Pending

Linux maintains two levels of pending signals:

```
+-------------------------------------------------+
|              Signal Pending Model               |
+-------------------------------------------------+
|                                                 |
|  Thread Group (signal_struct)                   |
|  ┌───────────────────────────────────────────┐  |
|  │  shared_pending                           │  |
|  │    - Signals sent to the process (kill())  │  |
|  │    - Any unblocking thread can dequeue     │  |
|  └───────────────────────────────────────────┘  |
|                                                 |
|  Thread 1 (task_struct)     Thread 2            |
|  ┌─────────────────────┐  ┌───────────────────┐ |
|  │  pending             │  │  pending           │|
|  │    - Signals sent to │  │    - Signals sent  │|
|  │      this specific   │  │      to this       │|
|  │      thread (tkill)  │  │      specific      │|
|  │    - Synchronous     │  │      thread        │|
|  │      signals (fault) │  │                    │|
|  ├─────────────────────┤  ├───────────────────┤ |
|  │  blocked (sigset_t)  │  │  blocked           │|
|  │    - Per-thread mask │  │    - Per-thread    │|
|  └─────────────────────┘  └───────────────────┘ |
|                                                 |
+-------------------------------------------------+
```

**Per-thread pending** (`task_struct->pending`): Used for signals directed at
a specific thread via `tkill()` / `tgkill()` and for synchronous signals
generated by CPU faults (SIGSEGV, SIGFPE, etc.).

**Shared pending** (`signal_struct->shared_pending`): Used for signals sent to
the process as a whole via `kill()`. Any thread that does not have the signal
blocked can dequeue and handle it.

### Blocked Signals

Each thread has its own `blocked` mask. A signal present in `blocked` will
not be delivered (it remains pending). Two signals cannot be blocked:
`SIGKILL` and `SIGSTOP`.

The `real_blocked` field saves the original mask during `sigsuspend()` so it
can be restored after the signal handler returns.

### Recalculating Pending State

The `TIF_SIGPENDING` thread flag is set when a thread has deliverable signals.
It is checked on return from syscalls and interrupts.

```c
/* kernel/signal.c:159-175 */
static bool recalc_sigpending_tsk(struct task_struct *t)
{
    if ((t->jobctl & (JOBCTL_PENDING_MASK | JOBCTL_TRAP_FREEZE)) ||
        PENDING(&t->pending, &t->blocked) ||
        PENDING(&t->signal->shared_pending, &t->blocked) ||
        cgroup_task_frozen(t)) {
        set_tsk_thread_flag(t, TIF_SIGPENDING);
        return true;
    }
    return false;
}
```

The `PENDING(p, b)` macro checks whether any signal in pending set `p` is not
blocked by mask `b`:

```c
/* kernel/signal.c:131-157 */
static inline bool has_pending_signals(sigset_t *signal, sigset_t *blocked)
{
    unsigned long ready;
    long i;

    switch (_NSIG_WORDS) {
    default:
        for (i = _NSIG_WORDS, ready = 0; --i >= 0 ;)
            ready |= signal->sig[i] &~ blocked->sig[i];
        break;
    case 4: ready  = signal->sig[3] &~ blocked->sig[3];
            ready |= signal->sig[2] &~ blocked->sig[2];
            ready |= signal->sig[1] &~ blocked->sig[1];
            ready |= signal->sig[0] &~ blocked->sig[0];
            break;
    case 2: ready  = signal->sig[1] &~ blocked->sig[1];
            ready |= signal->sig[0] &~ blocked->sig[0];
            break;
    case 1: ready  = signal->sig[0] &~ blocked->sig[0];
    }
    return ready != 0;
}
```

---

## Signal Generation and Sending

### Key Sending Functions

```
+--------------------------------------------------------------+
|                   Signal Sending Call Chain                   |
+--------------------------------------------------------------+
|                                                              |
|  User space syscalls:                                        |
|    kill()  ──> kill_something_info()                          |
|    tkill() ──> do_tkill()                                    |
|    tgkill()──> do_tkill()                                    |
|    rt_sigqueueinfo() ──> do_rt_sigqueueinfo()                |
|         |                                                    |
|         v                                                    |
|  group_send_sig_info()     or     do_send_sig_info()         |
|         |                              |                     |
|         v                              v                     |
|  send_signal_locked(sig, info, t, type)                      |
|         |                                                    |
|         v                                                    |
|  __send_signal_locked(sig, info, t, type, force)             |
|         |                                                    |
|         +-- prepare_signal()  : pre-delivery checks          |
|         +-- legacy_queue()    : drop duplicate std signals    |
|         +-- sigqueue_alloc()  : allocate sigqueue entry       |
|         +-- signalfd_notify() : wake signalfd waiters         |
|         +-- complete_signal() : select thread + wake up       |
|                                                              |
+--------------------------------------------------------------+
```

### __send_signal_locked() -- Core Send Path

This is the heart of signal generation (`kernel/signal.c:1041`):

```c
/* kernel/signal.c:1041-1156 */
static int __send_signal_locked(int sig, struct kernel_siginfo *info,
                struct task_struct *t, enum pid_type type, bool force)
{
    struct sigpending *pending;
    struct sigqueue *q;
    int override_rlimit;
    int ret = 0, result;

    lockdep_assert_held(&t->sighand->siglock);

    result = TRACE_SIGNAL_IGNORED;
    if (!prepare_signal(sig, t, force))
        goto ret;

    /* Per-thread or shared pending? */
    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;

    /* Standard signals: only one instance pending */
    result = TRACE_SIGNAL_ALREADY_PENDING;
    if (legacy_queue(pending, sig))
        goto ret;

    /* Allocate sigqueue for non-SIGKILL, non-kthread */
    if ((sig == SIGKILL) || (t->flags & PF_KTHREAD))
        goto out_set;

    /* ... allocate q, fill in siginfo ... */

    q = sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);
    if (q) {
        list_add_tail(&q->list, &pending->list);
        /* fill q->info based on info type */
    } else if (sig >= SIGRTMIN && ...) {
        /* RT signal queue overflow: return -EAGAIN */
        ret = -EAGAIN;
        goto ret;
    }

out_set:
    signalfd_notify(t, sig);
    sigaddset(&pending->signal, sig);
    complete_signal(sig, t, type);
ret:
    trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
    return ret;
}
```

Key design points:
1. **Standard signals** are deduplicated via `legacy_queue()` which checks
   `sig < SIGRTMIN && sigismember(&signals->signal, sig)`.
2. **RT signals** are always queued; queue overflow returns `-EAGAIN`.
3. **SIGKILL** skips sigqueue allocation entirely for speed.
4. The `force` flag bypasses ignore/block checks for kernel-generated signals
   and signals from ancestor pid namespaces.

### complete_signal() -- Thread Selection

After queuing the signal, `complete_signal()` selects a thread to wake up
(`kernel/signal.c:962`):

```c
/* kernel/signal.c:962-1034 */
static void complete_signal(int sig, struct task_struct *p, enum pid_type type)
{
    struct signal_struct *signal = p->signal;
    struct task_struct *t;

    /* Try the suggested task first */
    if (wants_signal(sig, p))
        t = p;
    else if ((type == PIDTYPE_PID) || thread_group_empty(p))
        return;     /* single thread, will see it on return to userspace */
    else {
        /* Round-robin search through thread group */
        t = signal->curr_target;
        while (!wants_signal(sig, t)) {
            t = next_thread(t);
            if (t == signal->curr_target)
                return; /* no eligible thread */
        }
        signal->curr_target = t;
    }

    /* If fatal, kill entire thread group immediately */
    if (sig_fatal(p, sig) && ...) {
        if (!sig_kernel_coredump(sig)) {
            signal->flags = SIGNAL_GROUP_EXIT;
            signal->group_exit_code = sig;
            signal->group_stop_count = 0;
            __for_each_thread(signal, t) {
                task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK);
                sigaddset(&t->pending.signal, SIGKILL);
                signal_wake_up(t, 1);
            }
            return;
        }
    }

    signal_wake_up(t, sig == SIGKILL);
}
```

The `curr_target` field provides simple load-balancing: each group-wide signal
delivery starts from where the last one left off, rotating through threads.

### send_signal_locked() -- Wrapper with Namespace Logic

```c
/* kernel/signal.c:1182-1216 */
int send_signal_locked(int sig, struct kernel_siginfo *info,
                       struct task_struct *t, enum pid_type type)
```

This wrapper determines the `force` flag based on:
- `SEND_SIG_NOINFO`: Force if sent from an ancestor pid namespace
- `SEND_SIG_PRIV`: Always force (kernel-generated)
- Regular info: Force if `si_code == SI_KERNEL` or from ancestor namespace

### force_sig_info_to_task() -- Unblockable Signal Delivery

```c
/* kernel/signal.c:1292-1326 */
static int force_sig_info_to_task(struct kernel_siginfo *info,
    struct task_struct *t, enum sig_handler handler)
```

Used for synchronous signals that must be delivered (e.g., SIGSEGV from a
page fault). It:
1. Resets the handler to `SIG_DFL` if blocked or ignored
2. Unblocks the signal from `t->blocked`
3. Clears `SIGNAL_UNKILLABLE` (so even init can be killed by forced signals)
4. Calls `send_signal_locked()`

---

## Signal Queuing

### Standard Signal Queuing (Non-RT)

Standard signals (1-31) use **bitmap-only** semantics: at most one instance
of each signal is pending. The `legacy_queue()` function enforces this:

```c
/* kernel/signal.c:1036-1038 */
static inline bool legacy_queue(struct sigpending *signals, int sig)
{
    return (sig < SIGRTMIN) && sigismember(&signals->signal, sig);
}
```

If the bit is already set, the new signal is dropped (not queued). A `sigqueue`
entry is still allocated to carry the `siginfo_t` data, but only the first
pending instance survives.

### Real-Time Signal Queuing

RT signals (SIGRTMIN through SIGRTMAX) queue every instance. Each call to
`sigqueue()` or `rt_sigqueueinfo()` creates a separate `sigqueue` entry in
the pending list. The total number of queued signals per user is limited by
`RLIMIT_SIGPENDING`:

```c
/* kernel/signal.c:248-259 (print_dropped_signal) */
static inline void print_dropped_signal(int sig)
{
    /* ... */
    pr_info("%s/%d: reached RLIMIT_SIGPENDING, dropped signal %d\n",
            current->comm, current->pid, sig);
}
```

When the limit is exceeded:
- For standard signals: the signal is still set in the bitmap (without siginfo)
- For RT signals from userspace (`si_code < 0`): `-EAGAIN` is returned

### Queue Memory Management

Signal queue entries are allocated from a dedicated SLAB cache:

```c
/* kernel/signal.c:68 */
static struct kmem_cache *sigqueue_cachep;
```

---

## Signal Delivery and Dequeuing

### get_signal() -- Main Delivery Entry Point

`get_signal()` is called on return to user space when `TIF_SIGPENDING` is set.
It is the central function that dequeues signals and determines the action to
take (`kernel/signal.c:2801`):

```c
/* kernel/signal.c:2801-3047 */
bool get_signal(struct ksignal *ksig)
{
    struct sighand_struct *sighand = current->sighand;
    struct signal_struct *signal = current->signal;
    int signr;

    /* Handle task_work and check for pending signals */
    clear_notify_signal();
    if (unlikely(task_work_pending(current)))
        task_work_run();
    if (!task_sigpending(current))
        return false;

    /* ... try_to_freeze(), check group exit ... */

relock:
    spin_lock_irq(&sighand->siglock);

    for (;;) {
        /* Check for group exit/exec in progress */
        if ((signal->flags & SIGNAL_GROUP_EXIT) || signal->group_exec_task) {
            signr = SIGKILL;
            goto fatal;
        }

        /* Handle pending job control stop */
        if (unlikely(current->jobctl & JOBCTL_STOP_PENDING) &&
            do_signal_stop(0))
            goto relock;

        /* Dequeue: synchronous signals first, then regular */
        signr = dequeue_synchronous_signal(&ksig->info);
        if (!signr)
            signr = dequeue_signal(&current->blocked, &ksig->info, &type);

        if (!signr)
            break;  /* no more signals */

        /* Ptrace interception */
        if (unlikely(current->ptrace) && (signr != SIGKILL) &&
            !(sighand->action[signr-1].sa.sa_flags & SA_IMMUTABLE)) {
            signr = ptrace_signal(signr, &ksig->info, type);
            if (!signr)
                continue;
        }

        ka = &sighand->action[signr-1];

        if (ka->sa.sa_handler == SIG_IGN)
            continue;               /* ignored */
        if (ka->sa.sa_handler != SIG_DFL) {
            ksig->ka = *ka;         /* user handler */
            if (ka->sa.sa_flags & SA_ONESHOT)
                ka->sa.sa_handler = SIG_DFL;
            break;                  /* deliver to handler */
        }

        /* Default action logic: ignore / stop / coredump / terminate */
        if (sig_kernel_ignore(signr))   continue;
        if (sig_kernel_stop(signr))     { do_signal_stop(signr); goto relock; }

    fatal:
        /* Fatal: coredump or group exit */
        current->flags |= PF_SIGNALED;
        if (sig_kernel_coredump(signr))
            do_coredump(&ksig->info);
        do_group_exit(signr);
    }

    ksig->sig = signr;
    return signr > 0;
}
```

### dequeue_signal() -- Remove Next Deliverable Signal

```c
/* kernel/signal.c:617-664 */
int dequeue_signal(sigset_t *mask, kernel_siginfo_t *info, enum pid_type *type)
{
    struct task_struct *tsk = current;
    int signr;

again:
    *type = PIDTYPE_PID;
    signr = __dequeue_signal(&tsk->pending, mask, info, &timer_sigq);
    if (!signr) {
        *type = PIDTYPE_TGID;
        signr = __dequeue_signal(&tsk->signal->shared_pending,
                                 mask, info, &timer_sigq);
    }
    recalc_sigpending();
    /* ... handle stop dequeue marker and POSIX timers ... */
    return signr;
}
```

Per-thread pending signals are dequeued before shared (thread-group) pending
signals.

### next_signal() -- Find First Deliverable Signal

```c
/* kernel/signal.c:202-246 */
int next_signal(struct sigpending *pending, sigset_t *mask)
```

Scans the pending bitmask for the first set bit not in the blocked mask.
Synchronous signals (word 0, `SYNCHRONOUS_MASK`) are checked first to ensure
fault-generated signals are prioritized.

### Signal Delivery Priority

```
+------------------------------------------------------------+
|             Signal Dequeue Priority Order                   |
+------------------------------------------------------------+
|                                                            |
|  1. Synchronous signals (SIGSEGV, SIGBUS, SIGILL, etc.)    |
|     - dequeue_synchronous_signal() checked first           |
|     - Must have positive si_code (kernel-generated)        |
|                                                            |
|  2. Per-thread pending (task->pending)                      |
|     - Within standard signals: lowest number first         |
|     - Standard signals before RT signals                    |
|                                                            |
|  3. Shared pending (signal->shared_pending)                 |
|     - Same ordering as per-thread                           |
|                                                            |
|  Within RT signals: lowest-numbered first (SIGRTMIN has     |
|  highest priority among RT signals)                         |
|                                                            |
+------------------------------------------------------------+
```

---

## Signal Handling (sigaction)

### Installing a Handler

User space installs signal handlers via the `rt_sigaction()` system call,
which updates `sighand->action[sig-1]`. The handler can be:

- `SIG_DFL` (0): Use the default action for this signal
- `SIG_IGN` (1): Ignore the signal
- A function pointer: Invoke the user-space handler

### Handler Execution Flow

When `get_signal()` finds a user handler, the arch-specific code takes over:

```
+-------------------------------------------------------------------+
|              Handler Execution (x86-64 example)                   |
+-------------------------------------------------------------------+
|                                                                   |
|  get_signal() returns true with ksig populated                    |
|       |                                                           |
|       v                                                           |
|  arch_do_signal_or_restart()      (arch/x86/kernel/signal.c:333)  |
|       |                                                           |
|       v                                                           |
|  handle_signal()                  (arch/x86/kernel/signal.c:254)  |
|       |                                                           |
|       +-- Handle syscall restart (ERESTARTSYS, etc.)              |
|       +-- setup_rt_frame()        (arch/x86/kernel/signal.c:236)  |
|       |       |                                                   |
|       |       +-- get_sigframe(): compute stack pointer            |
|       |       +-- Build rt_sigframe on user stack:                 |
|       |       |     - siginfo_t                                   |
|       |       |     - ucontext (saved registers + sigmask)        |
|       |       |     - FPU/XSAVE state                             |
|       |       +-- Set RIP = handler, RSP = frame                  |
|       |       +-- Set RDI = sig number (arg 1)                    |
|       |       +-- Set return address = sa_restorer (sigreturn)    |
|       |                                                           |
|       +-- signal_setup_done()                                     |
|       |       +-- signal_delivered(): update blocked mask          |
|       |                                                           |
|       v                                                           |
|  Return to user space -> handler executes                         |
|       |                                                           |
|       v                                                           |
|  Handler returns -> sa_restorer trampoline -> rt_sigreturn()      |
|       |                                                           |
|       v                                                           |
|  sys_rt_sigreturn(): restore registers + sigmask from frame       |
|                                                                   |
+-------------------------------------------------------------------+
```

### signal_delivered() -- Post-Delivery Mask Update

```c
/* kernel/signal.c:3059-3077 */
static void signal_delivered(struct ksignal *ksig, int stepping)
{
    sigset_t blocked;

    clear_restore_sigmask();

    /* Block sa_mask signals plus the signal itself (unless SA_NODEFER) */
    sigorsets(&blocked, &current->blocked, &ksig->ka.sa.sa_mask);
    if (!(ksig->ka.sa.sa_flags & SA_NODEFER))
        sigaddset(&blocked, ksig->sig);
    set_current_blocked(&blocked);

    if (current->sas_ss_flags & SS_AUTODISARM)
        sas_ss_reset(current);
    if (stepping)
        ptrace_notify(SIGTRAP, 0);
}
```

---

## Architecture-Specific Signal Frame Setup

### Signal Frame Structure (x86-64)

The signal frame is pushed onto the user-space stack (or alternate signal stack)
before the handler is invoked:

```c
/* arch/x86/include/asm/sigframe.h:59-64 */
struct rt_sigframe {
    char __user *pretcode;      /* return address (sigreturn trampoline) */
    struct ucontext uc;         /* saved context (regs, sigmask, stack) */
    struct siginfo info;        /* signal information */
    /* fp state follows here */
};
```

The `ucontext` contains:
- `uc_mcontext` (sigcontext): all saved general-purpose registers
- `uc_sigmask`: the signal mask to restore on return
- `uc_stack`: the alternate signal stack info

### Stack Frame Layout (x86-64)

```
High addresses
+---------------------------+
| (previous stack contents) |
+---------------------------+
| FPU/XSAVE state          |  <-- aligned to 64 bytes
+---------------------------+
| alignment padding         |
+---------------------------+
| struct rt_sigframe         |
|   +-- pretcode (-> vDSO   |
|   |    __vdso_rt_sigreturn)|
|   +-- ucontext             |
|   |    +-- uc_flags        |
|   |    +-- uc_link         |
|   |    +-- uc_stack        |
|   |    +-- uc_mcontext     |
|   |    |    (all regs)     |
|   |    +-- uc_sigmask      |
|   +-- siginfo              |
+---------------------------+  <-- RSP on handler entry
| return address (pretcode) |
+---------------------------+
Low addresses
```

### get_sigframe() -- Computing Stack Position

```c
/* arch/x86/kernel/signal.c:93-173 */
void __user *get_sigframe(struct ksignal *ksig, struct pt_regs *regs,
                          size_t frame_size, void __user **fpstate)
{
    unsigned long sp = regs->sp;

    /* x86-64 redzone: 128 bytes below RSP */
    if (!ia32_frame)
        sp -= 128;

    /* Signal stack switching (SA_ONSTACK) */
    if (ka->sa.sa_flags & SA_ONSTACK) {
        if (sas_ss_flags(sp) == 0) {
            sp = current->sas_ss_sp + current->sas_ss_size;
        }
    }

    /* Allocate space for FPU state */
    sp = fpu__alloc_mathframe(sp, ia32_frame, &buf_fx, &math_size);
    *fpstate = (void __user *)sp;

    /* Allocate space for sigframe */
    sp -= frame_size;

    /* Alignment: 16-byte aligned with 8-byte offset for x86-64 ABI */
    sp = round_down(sp, FRAME_ALIGNMENT) - 8;

    /* Enable all pkeys for signal stack access */
    pkru = sig_prepare_pkru();

    /* Save FPU state to user stack */
    copy_fpstate_to_sigframe(*fpstate, ...);

    return (void __user *)sp;
}
```

### handle_signal() -- x86 Arch Entry Point

```c
/* arch/x86/kernel/signal.c:254-313 */
static void handle_signal(struct ksignal *ksig, struct pt_regs *regs)
{
    /* Handle system call restart */
    if (syscall_get_nr(current, regs) != -1) {
        switch (syscall_get_error(current, regs)) {
        case -ERESTART_RESTARTBLOCK:
        case -ERESTARTNOHAND:
            regs->ax = -EINTR;
            break;
        case -ERESTARTSYS:
            if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
                regs->ax = -EINTR;
                break;
            }
            fallthrough;
        case -ERESTARTNOINTR:
            regs->ax = regs->orig_ax;
            regs->ip -= 2;
            break;
        }
    }

    /* Set up signal frame and redirect to handler */
    failed = (setup_rt_frame(ksig, regs) < 0);
    if (!failed) {
        regs->flags &= ~(X86_EFLAGS_DF|X86_EFLAGS_RF|X86_EFLAGS_TF);
        fpu__clear_user_states(fpu);
    }
    signal_setup_done(failed, ksig, stepping);
}
```

---

## Process Groups and Job Control Signals

### Job Control Overview

Job control allows a shell to manage foreground and background process groups.
The four stop signals (`SIGSTOP`, `SIGTSTP`, `SIGTTIN`, `SIGTTOU`) stop all
threads in the group, and `SIGCONT` resumes them.

```
+---------------------------------------------------------------+
|                    Job Control Flow                            |
+---------------------------------------------------------------+
|                                                               |
|  Terminal (tty)                                                |
|    |                                                          |
|    +-- Ctrl+Z generates SIGTSTP to foreground pgrp            |
|    +-- Ctrl+C generates SIGINT to foreground pgrp             |
|    +-- Ctrl+\ generates SIGQUIT to foreground pgrp            |
|                                                               |
|  Shell                                                        |
|    |                                                          |
|    +-- "fg" sends SIGCONT to stopped pgrp                     |
|    +-- "bg" sends SIGCONT to stopped pgrp                     |
|    +-- "kill %N" sends signal to pgrp                         |
|                                                               |
|  SIGCONT vs Stop signals:                                     |
|    - Sending SIGCONT clears all pending stop signals           |
|    - Sending any stop signal clears pending SIGCONT            |
|    - These rules apply regardless of blocking/catching         |
|                                                               |
|  Orphaned process groups:                                     |
|    - SIGTSTP, SIGTTIN, SIGTTOU are silently discarded         |
|    - Only SIGSTOP can stop an orphaned pgrp                    |
|                                                               |
+---------------------------------------------------------------+
```

### do_signal_stop() -- Group Stop Implementation

```c
/* kernel/signal.c:2552-2651 */
static bool do_signal_stop(int signr)
{
    struct signal_struct *sig = current->signal;

    if (!(current->jobctl & JOBCTL_STOP_PENDING)) {
        /* Initiate group stop */
        if (!(sig->flags & SIGNAL_STOP_STOPPED))
            sig->group_exit_code = signr;

        sig->group_stop_count = 0;
        if (task_set_jobctl_pending(current, signr | gstop))
            sig->group_stop_count++;

        /* Notify all other threads to stop */
        for_other_threads(current, t) {
            if (!task_is_stopped(t) &&
                task_set_jobctl_pending(t, signr | gstop)) {
                sig->group_stop_count++;
                if (likely(!(t->ptrace & PT_SEIZED)))
                    signal_wake_up(t, 0);
                else
                    ptrace_trap_notify(t);
            }
        }
    }

    if (likely(!current->ptrace)) {
        /* Last thread to stop notifies parent */
        if (task_participate_group_stop(current))
            notify = CLD_STOPPED;

        current->jobctl |= JOBCTL_STOPPED;
        set_special_state(TASK_STOPPED);
        spin_unlock_irq(&current->sighand->siglock);

        if (notify) {
            read_lock(&tasklist_lock);
            do_notify_parent_cldstop(current, false, notify);
            read_unlock(&tasklist_lock);
        }

        cgroup_enter_frozen();
        schedule();     /* sleep until SIGCONT or SIGKILL */
        return true;
    }
    /* ... ptrace handling ... */
}
```

### SIGCONT Handling

When `SIGCONT` is sent to a process, `prepare_signal()` (called from
`__send_signal_locked()`) performs special actions regardless of whether
`SIGCONT` is blocked, ignored, or caught:

1. Removes all pending stop signals from both per-thread and shared pending sets
2. Wakes all threads from `TASK_STOPPED` state
3. Sets `SIGNAL_CLD_CONTINUED` for parent notification
4. Clears `SIGNAL_STOP_STOPPED`

---

## Core Dump Generation

When a signal with the coredump default action is delivered and the handler is
`SIG_DFL`, the kernel generates a core dump:

```c
/* kernel/signal.c:3009-3022 (inside get_signal fatal path) */
if (sig_kernel_coredump(signr)) {
    if (print_fatal_signals)
        print_fatal_signal(signr);
    proc_coredump_connector(current);
    /*
     * If it was able to dump core, this kills all
     * other threads in the group and synchronizes with
     * their demise.
     */
    do_coredump(&ksig->info);
}

/* Death signals, no core dump */
do_group_exit(signr);
```

The coredump process:
1. `do_coredump()` is called with the signal info
2. All other threads in the group are killed via `zap_other_threads()`
3. The core state is set up in `signal->core_state`
4. A core file is written (ELF format by default)
5. The process exits via `do_group_exit()`

Signals that generate core dumps:
`SIGQUIT`, `SIGILL`, `SIGTRAP`, `SIGABRT`, `SIGFPE`, `SIGSEGV`, `SIGBUS`,
`SIGSYS`, `SIGXCPU`, `SIGXFSZ` (and `SIGEMT` on architectures that support it).

### zap_other_threads() -- Kill Thread Group

```c
/* kernel/signal.c:1336-1355 */
int zap_other_threads(struct task_struct *p)
{
    struct task_struct *t;
    int count = 0;

    p->signal->group_stop_count = 0;

    for_other_threads(p, t) {
        task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK);
        count++;
        if (t->exit_state)
            continue;
        sigaddset(&t->pending.signal, SIGKILL);
        signal_wake_up(t, 1);
    }
    return count;
}
```

---

## Ptrace Interaction

Debuggers (e.g., gdb) use `ptrace()` to intercept signals before they are
delivered to the tracee. This interaction is central to the signal delivery
path.

### Signal Interception

When a ptraced task dequeues a signal in `get_signal()`, the signal passes
through `ptrace_signal()` before delivery:

```c
/* kernel/signal.c:2733-2777 */
static int ptrace_signal(int signr, kernel_siginfo_t *info, enum pid_type type)
{
    current->jobctl |= JOBCTL_STOP_DEQUEUED;
    signr = ptrace_stop(signr, CLD_TRAPPED, 0, info);

    /* Debugger may have cancelled or changed the signal */
    if (signr == 0)
        return signr;

    /* If debugger changed the signal number, rebuild siginfo */
    if (signr != info->si_signo) {
        clear_siginfo(info);
        info->si_signo = signr;
        info->si_errno = 0;
        info->si_code = SI_USER;
        info->si_pid = task_pid_vnr(current->parent);
        info->si_uid = from_kuid_munged(...);
    }

    /* If the (new) signal is now blocked, requeue it */
    if (sigismember(&current->blocked, signr) ||
        fatal_signal_pending(current)) {
        send_signal_locked(signr, info, current, type);
        signr = 0;
    }

    return signr;
}
```

### Ptrace Signal Flow

```
+-------------------------------------------------------------------+
|                  Ptrace Signal Interception                        |
+-------------------------------------------------------------------+
|                                                                   |
|  Signal arrives at ptraced task                                   |
|       |                                                           |
|       v                                                           |
|  get_signal() dequeues signal                                     |
|       |                                                           |
|       v                                                           |
|  ptrace_signal()                                                  |
|       |                                                           |
|       v                                                           |
|  ptrace_stop() -- task enters TASK_TRACED                         |
|       |                                                           |
|       +-- Parent (debugger) is notified via waitpid()             |
|       |   with WIFSTOPPED(status) and WSTOPSIG(status) == sig     |
|       |                                                           |
|       +-- Debugger can:                                           |
|           - PTRACE_CONT: resume with signal (or signal 0)         |
|           - PTRACE_SETSIGINFO: modify the siginfo                 |
|           - PTRACE_GETSIGINFO: read the siginfo                   |
|           - Change signal number via PTRACE_CONT data arg         |
|                                                                   |
|  ptrace_stop() returns with potentially modified signal           |
|       |                                                           |
|       v                                                           |
|  If signr == 0: signal suppressed, continue to next signal        |
|  If signr changed: rebuild siginfo with SI_USER                   |
|  If signr blocked: requeue and suppress                           |
|  Otherwise: proceed with normal delivery                          |
|                                                                   |
+-------------------------------------------------------------------+
```

### SA_IMMUTABLE Protection

Forced signals with the `SA_IMMUTABLE` flag skip ptrace interception entirely
to prevent debuggers from suppressing critical signals:

```c
/* kernel/signal.c:2921-2926 (inside get_signal) */
if (unlikely(current->ptrace) && (signr != SIGKILL) &&
    !(sighand->action[signr-1].sa.sa_flags & SA_IMMUTABLE)) {
    signr = ptrace_signal(signr, &ksig->info, type);
    /* ... */
}
```

### SIGKILL and Ptrace

`SIGKILL` is never intercepted by ptrace. It is also never blocked, ignored,
or caught. Even a `TASK_TRACED` task will be killed by `SIGKILL`.

---

## signalfd

`signalfd` provides a file-descriptor-based interface for receiving signals,
enabling integration with `poll()`/`epoll()`/`select()` event loops.

### signalfd Context

```c
/* fs/signalfd.c:41-43 */
struct signalfd_ctx {
    sigset_t sigmask;
};
```

### Creating a signalfd

```c
/* fs/signalfd.c:251-305 */
static int do_signalfd4(int ufd, sigset_t *mask, int flags)
{
    /* SIGKILL and SIGSTOP cannot be received via signalfd */
    sigdelsetmask(mask, sigmask(SIGKILL) | sigmask(SIGSTOP));
    signotset(mask);    /* invert: sigmask stores which signals to accept */

    if (ufd == -1) {
        /* Create new signalfd */
        ctx = kmalloc(sizeof(*ctx), GFP_KERNEL);
        ctx->sigmask = *mask;
        file = anon_inode_getfile("[signalfd]", &signalfd_fops, ctx,
                                  O_RDWR | (flags & O_NONBLOCK));
        fd_install(ufd, file);
    } else {
        /* Update existing signalfd mask */
        ctx = fd_file(f)->private_data;
        spin_lock_irq(&current->sighand->siglock);
        ctx->sigmask = *mask;
        spin_unlock_irq(&current->sighand->siglock);
    }
    return ufd;
}
```

### Reading Signals via signalfd

Reading from a signalfd dequeues signals just like the normal delivery path,
but returns them as `struct signalfd_siginfo` (128 bytes) instead of invoking
a handler:

```c
/* fs/signalfd.c:154-194 */
static ssize_t signalfd_dequeue(struct signalfd_ctx *ctx,
                                kernel_siginfo_t *info, int nonblock)
{
    spin_lock_irq(&current->sighand->siglock);
    ret = dequeue_signal(&ctx->sigmask, info, &type);
    /* ... wait if blocking and no signal ... */
    spin_unlock_irq(&current->sighand->siglock);
    return ret;
}
```

### signalfd Poll Support

```c
/* fs/signalfd.c:51-66 */
static __poll_t signalfd_poll(struct file *file, poll_table *wait)
{
    struct signalfd_ctx *ctx = file->private_data;
    __poll_t events = 0;

    poll_wait(file, &current->sighand->signalfd_wqh, wait);

    spin_lock_irq(&current->sighand->siglock);
    if (next_signal(&current->pending, &ctx->sigmask) ||
        next_signal(&current->signal->shared_pending, &ctx->sigmask))
        events |= EPOLLIN;
    spin_unlock_irq(&current->sighand->siglock);

    return events;
}
```

### signalfd Notification

When a signal is queued via `__send_signal_locked()`, `signalfd_notify()` wakes
up any signalfd waiters on the `sighand->signalfd_wqh` wait queue.

### signalfd Usage Pattern

```
+-------------------------------------------------------------------+
|                   signalfd Usage Pattern                          |
+-------------------------------------------------------------------+
|                                                                   |
|  1. Block signals with sigprocmask() (so they are not delivered   |
|     via the normal path)                                          |
|                                                                   |
|  2. Create signalfd with those signals:                           |
|       int sfd = signalfd(-1, &mask, SFD_NONBLOCK | SFD_CLOEXEC)  |
|                                                                   |
|  3. Add sfd to epoll / poll loop                                  |
|                                                                   |
|  4. When sfd is readable:                                         |
|       struct signalfd_siginfo si;                                  |
|       read(sfd, &si, sizeof(si));                                 |
|       // si.ssi_signo contains the signal number                  |
|       // si.ssi_pid, si.ssi_uid, etc. available                   |
|                                                                   |
|  Key constraint: the calling thread must have the signals          |
|  blocked, otherwise they are delivered normally.                   |
|                                                                   |
+-------------------------------------------------------------------+
```

---

## System Call Restart

When a signal interrupts a blocking system call, the kernel must decide whether
to restart it or return an error to user space. This is determined by both the
syscall's restart behavior and the signal handler's `SA_RESTART` flag.

### Restart Error Codes

| Error Code | Behavior |
|-----------|----------|
| `-ERESTARTNOHAND` | Restart only if no handler; otherwise return `-EINTR` |
| `-ERESTARTSYS` | Restart if handler has `SA_RESTART`; otherwise `-EINTR` |
| `-ERESTARTNOINTR` | Always restart (transparent to user space) |
| `-ERESTART_RESTARTBLOCK` | Restart via `restart_syscall` (for syscalls needing modified args) |

### Restart Logic in handle_signal() (x86)

```c
/* arch/x86/kernel/signal.c:264-283 */
if (syscall_get_nr(current, regs) != -1) {
    switch (syscall_get_error(current, regs)) {
    case -ERESTART_RESTARTBLOCK:
    case -ERESTARTNOHAND:
        regs->ax = -EINTR;
        break;
    case -ERESTARTSYS:
        if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
            regs->ax = -EINTR;
            break;
        }
        fallthrough;
    case -ERESTARTNOINTR:
        regs->ax = regs->orig_ax;
        regs->ip -= 2;     /* re-execute the syscall instruction */
        break;
    }
}
```

### No-Handler Restart (arch_do_signal_or_restart)

```c
/* arch/x86/kernel/signal.c:333-366 */
void arch_do_signal_or_restart(struct pt_regs *regs)
{
    struct ksignal ksig;

    if (get_signal(&ksig)) {
        handle_signal(&ksig, regs);
        return;
    }

    /* No signal to deliver -- restart syscall if applicable */
    if (syscall_get_nr(current, regs) != -1) {
        switch (syscall_get_error(current, regs)) {
        case -ERESTARTNOHAND:
        case -ERESTARTSYS:
        case -ERESTARTNOINTR:
            regs->ax = regs->orig_ax;
            regs->ip -= 2;
            break;
        case -ERESTART_RESTARTBLOCK:
            regs->ax = get_nr_restart_syscall(regs);
            regs->ip -= 2;
            break;
        }
    }

    restore_saved_sigmask();
}
```

---

## Locking Summary

### siglock (sighand->siglock)

The primary lock protecting all signal state. It is a spinlock that must be
taken with interrupts disabled (`spin_lock_irq` / `spin_lock_irqsave`):

```
Protected by siglock:
  - task->pending (per-thread pending signals)
  - signal->shared_pending (thread-group pending signals)
  - task->blocked, task->real_blocked
  - sighand->action[] (signal handler table)
  - signal->flags (SIGNAL_* flags)
  - signal->group_stop_count
  - signal->group_exit_code
  - signal->curr_target
  - task->jobctl (job control flags)
```

### tasklist_lock

Read-locked when traversing the task tree for operations like:
- `do_notify_parent_cldstop()` -- notifying parent of child stop/continue
- `is_current_pgrp_orphaned()` -- checking orphaned process group
- Process group signal delivery

### Locking Order

```
tasklist_lock (read)
    └── sighand->siglock
```

The `tasklist_lock` must be acquired before `siglock` when both are needed.

---

## Source File Reference

| File | Description |
|------|-------------|
| `kernel/signal.c` | Core signal implementation: send, queue, dequeue, get_signal |
| `include/linux/signal.h` | Signal set operations, classification macros, kernel API |
| `include/linux/signal_types.h` | Core structs: sigqueue, sigpending, sigaction, k_sigaction, ksignal |
| `include/linux/sched/signal.h` | signal_struct, sighand_struct, SIGNAL_* flags |
| `include/linux/sched.h` | task_struct signal-related fields |
| `include/uapi/asm-generic/signal.h` | Signal numbers, sigset_t, sigaltstack, SIGRTMIN/MAX |
| `include/uapi/linux/signal.h` | User-space signal types and __SIGINFO macro |
| `arch/x86/kernel/signal.c` | x86 signal frame setup, handle_signal, sigreturn |
| `arch/x86/include/asm/sigframe.h` | x86 rt_sigframe, sigframe_ia32 structures |
| `fs/signalfd.c` | signalfd implementation |
| `kernel/coredump.c` | Core dump generation (do_coredump) |
| `kernel/ptrace.c` | ptrace_stop, ptrace signal interception |
| `include/linux/sched/jobctl.h` | JOBCTL_* flags for job control |

### Key Functions Quick Reference

| Function | Location | Purpose |
|----------|----------|---------|
| `__send_signal_locked()` | `kernel/signal.c:1041` | Core signal sending |
| `send_signal_locked()` | `kernel/signal.c:1182` | Wrapper with namespace/force logic |
| `do_send_sig_info()` | `kernel/signal.c:1261` | Lock-acquiring send wrapper |
| `group_send_sig_info()` | `kernel/signal.c:1408` | Permission-checking group send |
| `complete_signal()` | `kernel/signal.c:962` | Thread selection and wakeup |
| `get_signal()` | `kernel/signal.c:2801` | Main signal delivery loop |
| `dequeue_signal()` | `kernel/signal.c:617` | Dequeue next deliverable signal |
| `next_signal()` | `kernel/signal.c:202` | Find first pending unblocked signal |
| `force_sig_info_to_task()` | `kernel/signal.c:1292` | Force unblockable signal delivery |
| `do_signal_stop()` | `kernel/signal.c:2552` | Group stop implementation |
| `zap_other_threads()` | `kernel/signal.c:1336` | Kill all threads in group |
| `recalc_sigpending()` | `kernel/signal.c:177` | Recalculate TIF_SIGPENDING |
| `signal_delivered()` | `kernel/signal.c:3059` | Post-delivery mask update |
| `ptrace_signal()` | `kernel/signal.c:2733` | Ptrace signal interception |
| `handle_signal()` | `arch/x86/kernel/signal.c:254` | x86 handler dispatch |
| `get_sigframe()` | `arch/x86/kernel/signal.c:93` | Compute signal stack frame |
| `setup_rt_frame()` | `arch/x86/kernel/signal.c:236` | Build signal frame on stack |
| `arch_do_signal_or_restart()` | `arch/x86/kernel/signal.c:333` | x86 signal entry point |
| `do_signalfd4()` | `fs/signalfd.c:251` | signalfd creation/update |
| `signalfd_dequeue()` | `fs/signalfd.c:154` | Dequeue signal for signalfd |
| `signalfd_poll()` | `fs/signalfd.c:51` | Poll support for signalfd |
