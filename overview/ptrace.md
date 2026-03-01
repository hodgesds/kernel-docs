# Linux Kernel ptrace Overview

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Core Data Structures](#core-data-structures)
3. [ptrace Operations](#ptrace-operations)
4. [Stop States and Lifecycle](#stop-states-and-lifecycle)
5. [Syscall Tracing](#syscall-tracing)
6. [Register Access](#register-access)
7. [Memory Access](#memory-access)
8. [Single-Stepping](#single-stepping)
9. [Security Model](#security-model)
10. [Integration with Signals, Seccomp, and BPF Uprobes](#integration-with-signals-seccomp-and-bpf-uprobes)
11. [Source File Reference](#source-file-reference)

---

## Architecture Overview

ptrace is the Linux kernel's primary mechanism for process debugging and tracing.
It allows one process (the **tracer**, typically a debugger like GDB or strace) to
observe and control the execution of another process (the **tracee**), including
examining and modifying its memory and registers, intercepting system calls, and
single-stepping through instructions.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    ptrace Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tracer (debugger)                Tracee (debugged process) │
│  ──────────────────               ─────────────────────     │
│                                                             │
│  ptrace(ATTACH, pid) ───────────► task->ptrace = PT_PTRACED │
│                                   task->parent = tracer     │
│                                                             │
│  waitpid(pid) ◄─────────────────  ptrace_stop()             │
│    (tracee stopped)               (enters __TASK_TRACED)    │
│                                                             │
│  ptrace(PEEKDATA, addr) ────────► read tracee memory        │
│  ptrace(GETREGS) ──────────────► read tracee registers      │
│  ptrace(SINGLESTEP) ──────────── set TF flag, resume        │
│  ptrace(SYSCALL) ────────────── resume, stop at next syscall│
│  ptrace(CONT, sig) ─────────── resume with optional signal  │
│  ptrace(DETACH) ───────────────► restore original parent    │
│                                                             │
│  Flow:                                                      │
│   1. Tracer attaches (ATTACH/SEIZE or tracee uses TRACEME)  │
│   2. Tracee stops at events (signals, syscalls, steps)      │
│   3. Kernel notifies tracer via waitpid()                   │
│   4. Tracer inspects/modifies state                         │
│   5. Tracer resumes tracee (CONT/SYSCALL/SINGLESTEP)        │
│   6. Repeat until DETACH or tracee exits                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### System Call Entry Point

The `ptrace()` system call is defined in `kernel/ptrace.c`:

```c
/* kernel/ptrace.c:1258 */
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
                unsigned long, data)
{
    struct task_struct *child;
    long ret;

    if (request == PTRACE_TRACEME) {
        ret = ptrace_traceme();
        goto out;
    }

    child = find_get_task_by_vpid(pid);
    if (!child) {
        ret = -ESRCH;
        goto out;
    }

    if (request == PTRACE_ATTACH || request == PTRACE_SEIZE) {
        ret = ptrace_attach(child, request, addr, data);
        goto out_put_task_struct;
    }

    ret = ptrace_check_attach(child, request == PTRACE_KILL ||
                              request == PTRACE_INTERRUPT);
    if (ret < 0)
        goto out_put_task_struct;

    ret = arch_ptrace(child, request, addr, data);
    if (ret || request != PTRACE_DETACH)
        ptrace_unfreeze_traced(child);
    ...
}
```

The dispatch follows a layered model: generic requests are handled in
`ptrace_request()` (`kernel/ptrace.c`), while architecture-specific requests are
dispatched through `arch_ptrace()` (e.g., `arch/x86/kernel/ptrace.c:730`).

---

## Core Data Structures

### ptrace Fields in task_struct

The `task_struct` contains several fields that are fundamental to ptrace operation:

```c
/* include/linux/sched.h */

struct task_struct {
    ...
    unsigned int            ptrace;           /* :808 - PT_PTRACED, PT_SEIZED, event flags */

    /* Real parent process (biological): */
    struct task_struct __rcu *real_parent;     /* :1046 */

    /* Current parent (may be ptracer): */
    struct task_struct __rcu *parent;          /* :1049 */

    /*
     * 'ptraced' is the list of tasks this task is using ptrace() on.
     * 'ptrace_entry' is this task's link on p->parent->ptraced list.
     */
    struct list_head        ptraced;          /* :1064 */
    struct list_head        ptrace_entry;     /* :1065 */

    unsigned long           jobctl;           /* :938 - JOBCTL_TRACED, JOBCTL_TRAP_*, etc. */
    int                     exit_code;        /* :933 - stop signal code for tracer */

    const struct cred __rcu *ptracer_cred;    /* :1122 - credentials of the ptracer */
    unsigned long           ptrace_message;   /* :1266 - PTRACE_GETEVENTMSG value */
    kernel_siginfo_t        *last_siginfo;    /* :1267 - siginfo for current ptrace stop */
    ...
};
```

### ptrace Flags (task->ptrace)

```c
/* include/linux/ptrace.h:31-47 */

#define PT_PTRACED      0x00000001      /* Task is being ptraced */
#define PT_SEIZED       0x00010000      /* Attached via PTRACE_SEIZE (new behavior) */

#define PT_OPT_FLAG_SHIFT   3
#define PT_EVENT_FLAG(event)    (1 << (PT_OPT_FLAG_SHIFT + (event)))
#define PT_TRACESYSGOOD     PT_EVENT_FLAG(0)    /* Set bit 7 of syscall trap signal */
#define PT_TRACE_FORK       PT_EVENT_FLAG(PTRACE_EVENT_FORK)
#define PT_TRACE_VFORK      PT_EVENT_FLAG(PTRACE_EVENT_VFORK)
#define PT_TRACE_CLONE      PT_EVENT_FLAG(PTRACE_EVENT_CLONE)
#define PT_TRACE_EXEC       PT_EVENT_FLAG(PTRACE_EVENT_EXEC)
#define PT_TRACE_VFORK_DONE PT_EVENT_FLAG(PTRACE_EVENT_VFORK_DONE)
#define PT_TRACE_EXIT       PT_EVENT_FLAG(PTRACE_EVENT_EXIT)
#define PT_TRACE_SECCOMP    PT_EVENT_FLAG(PTRACE_EVENT_SECCOMP)

#define PT_EXITKILL         (PTRACE_O_EXITKILL << PT_OPT_FLAG_SHIFT)
#define PT_SUSPEND_SECCOMP  (PTRACE_O_SUSPEND_SECCOMP << PT_OPT_FLAG_SHIFT)
```

### Job Control Flags (task->jobctl)

The `jobctl` field tracks stop/trap state for both ptrace and job control:

```c
/* include/linux/sched/jobctl.h:17-41 */

#define JOBCTL_TRAP_STOP_BIT      19  /* trap for STOP */
#define JOBCTL_TRAP_NOTIFY_BIT    20  /* trap for NOTIFY */
#define JOBCTL_TRAPPING_BIT       21  /* switching to TRACED */
#define JOBCTL_LISTENING_BIT      22  /* ptracer is listening for events */
#define JOBCTL_PTRACE_FROZEN_BIT  24  /* frozen for ptrace */
#define JOBCTL_TRACED_BIT         27  /* ptrace_stop() */

#define JOBCTL_TRAP_STOP     (1UL << JOBCTL_TRAP_STOP_BIT)
#define JOBCTL_TRAP_NOTIFY   (1UL << JOBCTL_TRAP_NOTIFY_BIT)
#define JOBCTL_TRAPPING      (1UL << JOBCTL_TRAPPING_BIT)
#define JOBCTL_LISTENING     (1UL << JOBCTL_LISTENING_BIT)
#define JOBCTL_PTRACE_FROZEN (1UL << JOBCTL_PTRACE_FROZEN_BIT)
#define JOBCTL_TRACED        (1UL << JOBCTL_TRACED_BIT)
#define JOBCTL_TRAP_MASK     (JOBCTL_TRAP_STOP | JOBCTL_TRAP_NOTIFY)

/* Macros for checking state (include/linux/sched/jobctl.h:146-148) */
#define task_is_traced(task)              ((READ_ONCE(task->jobctl) & JOBCTL_TRACED) != 0)
#define task_is_stopped(task)             ((READ_ONCE(task->jobctl) & JOBCTL_STOPPED) != 0)
#define task_is_stopped_or_traced(task)   ((READ_ONCE(task->jobctl) & (JOBCTL_STOPPED | JOBCTL_TRACED)) != 0)
```

### Parent Reparenting on Attach

When a tracer attaches, the tracee's `parent` pointer is redirected to the tracer,
while `real_parent` remains unchanged. This is managed by `__ptrace_link()` and
`__ptrace_unlink()`:

```c
/* kernel/ptrace.c:69 */
void __ptrace_link(struct task_struct *child, struct task_struct *new_parent,
                   const struct cred *ptracer_cred)
{
    BUG_ON(!list_empty(&child->ptrace_entry));
    list_add(&child->ptrace_entry, &new_parent->ptraced);
    child->parent = new_parent;
    child->ptracer_cred = get_cred(ptracer_cred);
}
```

```
┌───────────────────────────────────────────────────────────┐
│               Parent Pointer Reparenting                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  Before attach:                                           │
│    task->real_parent ──► original parent                  │
│    task->parent ─────── ► original parent (same)          │
│                                                           │
│  After ptrace_attach():                                   │
│    task->real_parent ──► original parent (unchanged)      │
│    task->parent ─────── ► tracer process                  │
│    task->ptrace_entry ──► on tracer->ptraced list         │
│    task->ptracer_cred ──► tracer's credentials            │
│                                                           │
│  After ptrace_detach() / __ptrace_unlink():               │
│    task->parent ─────── ► real_parent (restored)          │
│    task->ptrace_entry ──► removed from list               │
│    task->ptracer_cred ──► NULL                            │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## ptrace Operations

### Request Constants

```c
/* include/uapi/linux/ptrace.h:11-55 */

/* Tracing requests */
#define PTRACE_TRACEME       0    /* Allow parent to trace this task */
#define PTRACE_PEEKTEXT      1    /* Read word at addr in tracee's text */
#define PTRACE_PEEKDATA      2    /* Read word at addr in tracee's data */
#define PTRACE_PEEKUSR       3    /* Read word in tracee's USER area */
#define PTRACE_POKETEXT      4    /* Write word at addr in tracee's text */
#define PTRACE_POKEDATA      5    /* Write word at addr in tracee's data */
#define PTRACE_POKEUSR       6    /* Write word in tracee's USER area */
#define PTRACE_CONT          7    /* Continue stopped tracee */
#define PTRACE_KILL          8    /* Kill tracee (deprecated, use SIGKILL) */
#define PTRACE_SINGLESTEP    9    /* Single-step tracee, deliver signal */

#define PTRACE_ATTACH       16    /* Attach to tracee (sends SIGSTOP) */
#define PTRACE_DETACH       17    /* Detach from tracee */

#define PTRACE_SYSCALL      24    /* Continue, stop at next syscall entry/exit */

/* Extended operations */
#define PTRACE_SETOPTIONS   0x4200  /* Set ptrace options */
#define PTRACE_GETEVENTMSG  0x4201  /* Get event message */
#define PTRACE_GETSIGINFO   0x4202  /* Get siginfo for current stop */
#define PTRACE_SETSIGINFO   0x4203  /* Set siginfo */
#define PTRACE_GETREGSET    0x4204  /* Get register set (NT_* type) */
#define PTRACE_SETREGSET    0x4205  /* Set register set (NT_* type) */
#define PTRACE_SEIZE        0x4206  /* Attach without stopping (new API) */
#define PTRACE_INTERRUPT    0x4207  /* Interrupt seized tracee */
#define PTRACE_LISTEN       0x4208  /* Listen for events (group stop) */
#define PTRACE_GETSIGMASK   0x420a  /* Get tracee's signal mask */
#define PTRACE_SETSIGMASK   0x420b  /* Set tracee's signal mask */
#define PTRACE_GET_SYSCALL_INFO  0x420e  /* Get syscall info struct */
```

### PTRACE_ATTACH vs PTRACE_SEIZE

PTRACE_SEIZE is the modern attach mechanism introduced to fix issues with the
legacy PTRACE_ATTACH. The key differences:

```
┌─────────────────────────────────────────────────────────────┐
│             PTRACE_ATTACH vs PTRACE_SEIZE                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PTRACE_ATTACH (legacy)              PTRACE_SEIZE (new)     │
│  ──────────────────────              ──────────────────     │
│  - Sends SIGSTOP to tracee           - Does NOT send signal │
│  - Sets PT_PTRACED only              - Sets PT_PTRACED |    │
│                                        PT_SEIZED            │
│  - Signal-delivery-stop model        - Enables new STOP     │
│                                        event model          │
│  - No PTRACE_INTERRUPT support       - Supports INTERRUPT   │
│  - No PTRACE_LISTEN support          - Supports LISTEN      │
│  - Options must be set later         - Can set options in   │
│    with PTRACE_SETOPTIONS              flags argument       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The attach implementation (`kernel/ptrace.c:409`):

```c
static int ptrace_attach(struct task_struct *task, long request,
                         unsigned long addr, unsigned long flags)
{
    bool seize = (request == PTRACE_SEIZE);
    int retval;

    if (seize) {
        if (addr != 0)
            return -EIO;
        if (flags & ~(unsigned long)PTRACE_O_MASK)
            return -EIO;
        retval = check_ptrace_options(flags);
        if (retval)
            return retval;
        flags = PT_PTRACED | PT_SEIZED | (flags << PT_OPT_FLAG_SHIFT);
    } else {
        flags = PT_PTRACED;
    }

    /* Must not be kernel thread, must not be self */
    if (unlikely(task->flags & PF_KTHREAD))
        return -EPERM;
    if (same_thread_group(task, current))
        return -EPERM;

    /* Credential guard and access checks */
    scoped_cond_guard(mutex_intr, ..., &task->signal->cred_guard_mutex) {
        scoped_guard(task_lock, task) {
            retval = __ptrace_may_access(task, PTRACE_MODE_ATTACH_REALCREDS);
            if (retval)
                return retval;
        }
        scoped_guard(write_lock_irq, &tasklist_lock) {
            if (unlikely(task->exit_state))
                return -EPERM;
            if (task->ptrace)       /* Already being traced */
                return -EPERM;
            task->ptrace = flags;
            ptrace_link(task, current);
            ptrace_set_stopped(task, seize);
        }
    }

    wait_on_bit(&task->jobctl, JOBCTL_TRAPPING_BIT, TASK_KILLABLE);
    return 0;
}
```

### PTRACE_TRACEME

Allows a child process to request tracing by its parent:

```c
/* kernel/ptrace.c:487 */
static int ptrace_traceme(void)
{
    int ret = -EPERM;

    write_lock_irq(&tasklist_lock);
    if (!current->ptrace) {
        ret = security_ptrace_traceme(current->parent);
        if (!ret && !(current->real_parent->flags & PF_EXITING)) {
            current->ptrace = PT_PTRACED;
            ptrace_link(current, current->real_parent);
        }
    }
    write_unlock_irq(&tasklist_lock);
    return ret;
}
```

### Event Options (PTRACE_SETOPTIONS)

```c
/* include/uapi/linux/ptrace.h:156-178 */

#define PTRACE_EVENT_FORK       1
#define PTRACE_EVENT_VFORK      2
#define PTRACE_EVENT_CLONE      3
#define PTRACE_EVENT_EXEC       4
#define PTRACE_EVENT_VFORK_DONE 5
#define PTRACE_EVENT_EXIT       6
#define PTRACE_EVENT_SECCOMP    7
#define PTRACE_EVENT_STOP       128    /* Group-stop related (SEIZE only) */

/* Options set using PTRACE_SETOPTIONS or PTRACE_SEIZE @data */
#define PTRACE_O_TRACESYSGOOD      1   /* Set bit 7 in si_code for syscall stops */
#define PTRACE_O_TRACEFORK         (1 << PTRACE_EVENT_FORK)
#define PTRACE_O_TRACEVFORK        (1 << PTRACE_EVENT_VFORK)
#define PTRACE_O_TRACECLONE        (1 << PTRACE_EVENT_CLONE)
#define PTRACE_O_TRACEEXEC         (1 << PTRACE_EVENT_EXEC)
#define PTRACE_O_TRACEVFORKDONE    (1 << PTRACE_EVENT_VFORK_DONE)
#define PTRACE_O_TRACEEXIT         (1 << PTRACE_EVENT_EXIT)
#define PTRACE_O_TRACESECCOMP      (1 << PTRACE_EVENT_SECCOMP)
#define PTRACE_O_EXITKILL          (1 << 20)   /* Kill tracee if tracer exits */
#define PTRACE_O_SUSPEND_SECCOMP   (1 << 21)   /* Suspend seccomp (CAP_SYS_ADMIN) */
```

Events are fired from key kernel code paths using `ptrace_event()`:

```c
/* include/linux/ptrace.h:148 */
static inline void ptrace_event(int event, unsigned long message)
{
    if (unlikely(ptrace_event_enabled(current, event))) {
        ptrace_notify((event << 8) | SIGTRAP, message);
    } else if (event == PTRACE_EVENT_EXEC) {
        /* legacy EXEC report via SIGTRAP */
        if ((current->ptrace & (PT_PTRACED|PT_SEIZED)) == PT_PTRACED)
            send_sig(SIGTRAP, current, 0);
    }
}
```

---

## Stop States and Lifecycle

### Task State Transitions

Traced tasks cycle through several states managed by `task->jobctl` and
the task scheduler state (`__TASK_TRACED`):

```
┌───────────────────────────────────────────────────────────────────┐
│                   ptrace Stop/Resume Lifecycle                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  RUNNING ───────────────────────────────► TRACED                  │
│    │         ptrace_stop() sets:          │                       │
│    │           set_special_state(         │  Tracee is stopped    │
│    │             TASK_TRACED)             │  in schedule()        │
│    │           JOBCTL_TRACED              │                       │
│    │           current->exit_code = sig   │                       │
│    │           notify tracer via waitpid()│                       │
│    │                                      │                       │
│    │                                      ▼                       │
│    │                              Tracer calls wait()             │
│    │                              gets stop status                │
│    │                                       │                      │
│    │         ptrace_resume():              │                      │
│    │           clear JOBCTL_TRACED         │                      │
│    │           wake_up_state(__TASK_TRACED)│                      │
│    ◄───────────────────────────────────────┘                      │
│                                                                   │
│  Special: ptrace_freeze_traced() / ptrace_unfreeze_traced()       │
│  ───────────────────────────────────────────────────────          │
│  While tracer inspects/modifies tracee state, the tracee is       │
│  "frozen" (JOBCTL_PTRACE_FROZEN) to prevent even SIGKILL from     │
│  waking it. This makes ptrace operations uninterruptible.         │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### ptrace_stop()

The core function where a tracee enters the traced state. Defined in
`kernel/signal.c:2351`:

```c
static int ptrace_stop(int exit_code, int why, unsigned long message,
                       kernel_siginfo_t *info)
    __releases(&current->sighand->siglock)
    __acquires(&current->sighand->siglock)
{
    bool gstop_done = false;

    if (arch_ptrace_stop_needed()) {
        spin_unlock_irq(&current->sighand->siglock);
        arch_ptrace_stop();
        spin_lock_irq(&current->sighand->siglock);
    }

    if (!current->ptrace || __fatal_signal_pending(current))
        return exit_code;

    set_special_state(TASK_TRACED);
    current->jobctl |= JOBCTL_TRACED;

    /* Memory barrier for synchronization with tracer */
    smp_wmb();

    current->ptrace_message = message;
    current->last_siginfo = info;
    current->exit_code = exit_code;

    /* Bookkeeping for group stop participation */
    if (why == CLD_STOPPED && (current->jobctl & JOBCTL_STOP_PENDING))
        gstop_done = task_participate_group_stop(current);

    task_clear_jobctl_pending(current, JOBCTL_TRAP_STOP);
    task_clear_jobctl_trapping(current);

    spin_unlock_irq(&current->sighand->siglock);
    read_lock(&tasklist_lock);

    /* Notify ptracer, and optionally real parent for group stop */
    if (current->ptrace)
        do_notify_parent_cldstop(current, true, why);
    if (gstop_done && (!current->ptrace || ptrace_reparented(current)))
        do_notify_parent_cldstop(current, false, why);

    read_unlock(&tasklist_lock);
    cgroup_enter_frozen();
    schedule();                /* <<< tracee sleeps here */
    cgroup_leave_frozen(true);

    /* Woken up by tracer - reacquire siglock, clear state */
    spin_lock_irq(&current->sighand->siglock);
    exit_code = current->exit_code;
    current->last_siginfo = NULL;
    current->ptrace_message = 0;
    current->exit_code = 0;
    current->jobctl &= ~(JOBCTL_LISTENING | JOBCTL_PTRACE_FROZEN);

    recalc_sigpending_tsk(current);
    return exit_code;
}
```

### ptrace_resume()

Resumes a stopped tracee. Handles enabling syscall tracing, single-stepping,
and waking the task (`kernel/ptrace.c:823`):

```c
static int ptrace_resume(struct task_struct *child, long request,
                         unsigned long data)
{
    if (!valid_signal(data))
        return -EIO;

    if (request == PTRACE_SYSCALL)
        set_task_syscall_work(child, SYSCALL_TRACE);
    else
        clear_task_syscall_work(child, SYSCALL_TRACE);

    if (request == PTRACE_SYSEMU || request == PTRACE_SYSEMU_SINGLESTEP)
        set_task_syscall_work(child, SYSCALL_EMU);
    else
        clear_task_syscall_work(child, SYSCALL_EMU);

    if (is_singleblock(request)) {
        user_enable_block_step(child);
    } else if (is_singlestep(request) || is_sysemu_singlestep(request)) {
        user_enable_single_step(child);
    } else {
        user_disable_single_step(child);
    }

    spin_lock_irq(&child->sighand->siglock);
    child->exit_code = data;        /* Signal to deliver (0 = none) */
    child->jobctl &= ~JOBCTL_TRACED;
    wake_up_state(child, __TASK_TRACED);
    spin_unlock_irq(&child->sighand->siglock);

    return 0;
}
```

### ptrace_check_attach() and Freezing

Before a tracer can operate on a stopped tracee, the kernel verifies the
relationship and "freezes" the tracee to prevent interference:

```c
/* kernel/ptrace.c:239 */
static int ptrace_check_attach(struct task_struct *child, bool ignore_state)
{
    int ret = -ESRCH;

    read_lock(&tasklist_lock);
    if (child->ptrace && child->parent == current) {
        if (ignore_state || ptrace_freeze_traced(child))
            ret = 0;
    }
    read_unlock(&tasklist_lock);

    if (!ret && !ignore_state &&
        WARN_ON_ONCE(!wait_task_inactive(child, __TASK_TRACED|TASK_FROZEN)))
        ret = -ESRCH;

    return ret;
}
```

```c
/* kernel/ptrace.c:184 */
static bool ptrace_freeze_traced(struct task_struct *task)
{
    bool ret = false;

    if (task->jobctl & JOBCTL_LISTENING)
        return ret;

    spin_lock_irq(&task->sighand->siglock);
    if (task_is_traced(task) && !looks_like_a_spurious_pid(task) &&
        !__fatal_signal_pending(task)) {
        task->jobctl |= JOBCTL_PTRACE_FROZEN;
        ret = true;
    }
    spin_unlock_irq(&task->sighand->siglock);
    return ret;
}
```

### PTRACE_INTERRUPT and PTRACE_LISTEN (SEIZE-only)

PTRACE_INTERRUPT forces a seized tracee to stop without sending a signal.
PTRACE_LISTEN allows a tracee in group-stop to remain stopped but will
re-trap when an asynchronous event occurs (e.g., group stop state change):

```c
/* kernel/ptrace.c:1106-1157 */
case PTRACE_INTERRUPT:
    if (unlikely(!seized || !lock_task_sighand(child, &flags)))
        break;
    if (likely(task_set_jobctl_pending(child, JOBCTL_TRAP_STOP)))
        ptrace_signal_wake_up(child, child->jobctl & JOBCTL_LISTENING);
    unlock_task_sighand(child, &flags);
    ret = 0;
    break;

case PTRACE_LISTEN:
    if (unlikely(!seized || !lock_task_sighand(child, &flags)))
        break;
    si = child->last_siginfo;
    if (likely(si && (si->si_code >> 8) == PTRACE_EVENT_STOP)) {
        child->jobctl |= JOBCTL_LISTENING;
        if (child->jobctl & JOBCTL_TRAP_NOTIFY)
            ptrace_signal_wake_up(child, true);
        ret = 0;
    }
    unlock_task_sighand(child, &flags);
    break;
```

---

## Syscall Tracing

### PTRACE_SYSCALL

When resumed with `PTRACE_SYSCALL`, the tracee will stop at the next system call
entry or exit. The mechanism sets `SYSCALL_WORK_SYSCALL_TRACE` on the tracee's
thread info, which is checked by the syscall entry/exit paths:

```c
/* include/linux/ptrace.h:407 */
static inline int ptrace_report_syscall(unsigned long message)
{
    int ptrace = current->ptrace;
    int signr;

    if (!(ptrace & PT_PTRACED))
        return 0;

    signr = ptrace_notify(SIGTRAP | ((ptrace & PT_TRACESYSGOOD) ? 0x80 : 0),
                          message);

    if (signr)
        send_sig(signr, current, 1);

    return fatal_signal_pending(current);
}
```

The entry and exit report functions:

```c
/* include/linux/ptrace.h:449 */
static inline __must_check int ptrace_report_syscall_entry(struct pt_regs *regs)
{
    return ptrace_report_syscall(PTRACE_EVENTMSG_SYSCALL_ENTRY);
}

/* include/linux/ptrace.h:472 */
static inline void ptrace_report_syscall_exit(struct pt_regs *regs, int step)
{
    if (step)
        user_single_step_report(regs);
    else
        ptrace_report_syscall(PTRACE_EVENTMSG_SYSCALL_EXIT);
}
```

### PTRACE_SYSEMU

PTRACE_SYSEMU is used primarily by User-Mode Linux (UML). It stops the tracee at
syscall entry but **skips the actual system call execution**, allowing the tracer
to emulate it:

```c
/* kernel/ptrace.c:834-838 */
if (request == PTRACE_SYSEMU || request == PTRACE_SYSEMU_SINGLESTEP)
    set_task_syscall_work(child, SYSCALL_EMU);
else
    clear_task_syscall_work(child, SYSCALL_EMU);
```

### PTRACE_GET_SYSCALL_INFO

A modern interface that provides structured information about the current
syscall stop:

```c
/* include/uapi/linux/ptrace.h:82 */
struct ptrace_syscall_info {
    __u8 op;        /* PTRACE_SYSCALL_INFO_* */
    __u8 pad[3];
    __u32 arch;
    __u64 instruction_pointer;
    __u64 stack_pointer;
    union {
        struct {
            __u64 nr;           /* Syscall number */
            __u64 args[6];      /* Syscall arguments */
        } entry;
        struct {
            __s64 rval;         /* Return value */
            __u8 is_error;      /* Whether rval is an error */
        } exit;
        struct {
            __u64 nr;           /* Syscall number */
            __u64 args[6];      /* Syscall arguments */
            __u32 ret_data;     /* Seccomp return data */
        } seccomp;
    };
};

/* Op codes */
#define PTRACE_SYSCALL_INFO_NONE    0
#define PTRACE_SYSCALL_INFO_ENTRY   1
#define PTRACE_SYSCALL_INFO_EXIT    2
#define PTRACE_SYSCALL_INFO_SECCOMP 3
```

The implementation dispatches based on `child->last_siginfo->si_code` and
`child->ptrace_message` to determine whether we are at entry, exit, or
a seccomp stop (`kernel/ptrace.c:967`):

```c
static int ptrace_get_syscall_info(struct task_struct *child,
                                   unsigned long user_size,
                                   void __user *datavp)
{
    struct pt_regs *regs = task_pt_regs(child);
    struct ptrace_syscall_info info = {
        .op = PTRACE_SYSCALL_INFO_NONE,
        .arch = syscall_get_arch(child),
        .instruction_pointer = instruction_pointer(regs),
        .stack_pointer = user_stack_pointer(regs),
    };

    switch (child->last_siginfo ? child->last_siginfo->si_code : 0) {
    case SIGTRAP | 0x80:
        switch (child->ptrace_message) {
        case PTRACE_EVENTMSG_SYSCALL_ENTRY:
            actual_size = ptrace_get_syscall_info_entry(child, regs, &info);
            break;
        case PTRACE_EVENTMSG_SYSCALL_EXIT:
            actual_size = ptrace_get_syscall_info_exit(child, regs, &info);
            break;
        }
        break;
    case SIGTRAP | (PTRACE_EVENT_SECCOMP << 8):
        actual_size = ptrace_get_syscall_info_seccomp(child, regs, &info);
        break;
    }
    ...
}
```

### Syscall Tracing Flow

```
┌────────────────────────────────────────────────────────────────┐
│                   Syscall Tracing Flow                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Tracee enters kernel for syscall                              │
│       │                                                        │
│       ▼                                                        │
│  Check SYSCALL_WORK_SYSCALL_TRACE?                             │
│       │ yes                                                    │
│       ▼                                                        │
│  ptrace_report_syscall_entry()                                 │
│       │                                                        │
│       ▼                                                        │
│  ptrace_stop() ──► tracee sleeps, tracer woken via waitpid()   │
│       │             tracer may inspect/modify registers        │
│       │             tracer may change syscall number           │
│       ▼                                                        │
│  Check SYSCALL_WORK_SYSCALL_EMU?                               │
│       │ yes: skip syscall entirely                             │
│       │ no:  execute the system call                           │
│       ▼                                                        │
│  (syscall executes)                                            │
│       │                                                        │
│       ▼                                                        │
│  Check SYSCALL_WORK_SYSCALL_TRACE?                             │
│       │ yes                                                    │
│       ▼                                                        │
│  ptrace_report_syscall_exit()                                  │
│       │                                                        │
│       ▼                                                        │
│  ptrace_stop() ──► tracee sleeps, tracer woken                 │
│                     tracer may inspect return value            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Register Access

### PTRACE_GETREGS / PTRACE_SETREGS (x86-specific)

The legacy register access interface provides bulk read/write of the general
purpose register set. On x86, this is handled in `arch/x86/kernel/ptrace.c:780`:

```c
/* arch/x86/kernel/ptrace.c:780 */
case PTRACE_GETREGS:
    return copy_regset_to_user(child,
                               regset_view,
                               REGSET_GENERAL,
                               0, sizeof(struct user_regs_struct),
                               datap);

case PTRACE_SETREGS:
    return copy_regset_from_user(child,
                                 regset_view,
                                 REGSET_GENERAL,
                                 0, sizeof(struct user_regs_struct),
                                 datap);
```

### PTRACE_GETREGSET / PTRACE_SETREGSET (Generic)

The modern, architecture-independent register access interface uses NT_* types
(the same types used for core dumps). This provides access to general registers,
floating-point state, extended state (AVX/AVX-512), and more:

```c
/* kernel/ptrace.c:888 */
static int ptrace_regset(struct task_struct *task, int req, unsigned int type,
                         struct iovec *kiov)
{
    const struct user_regset_view *view = task_user_regset_view(task);
    const struct user_regset *regset = find_regset(view, type);
    int regset_no;

    if (!regset || (kiov->iov_len % regset->size) != 0)
        return -EINVAL;

    regset_no = regset - view->regsets;
    kiov->iov_len = min(kiov->iov_len,
                        (__kernel_size_t)(regset->n * regset->size));

    if (req == PTRACE_GETREGSET)
        return copy_regset_to_user(task, view, regset_no, 0,
                                   kiov->iov_len, kiov->iov_base);
    else
        return copy_regset_from_user(task, view, regset_no, 0,
                                     kiov->iov_len, kiov->iov_base);
}
```

### user_regset and user_regset_view

The regset abstraction (`include/linux/regset.h:200`):

```c
struct user_regset {
    user_regset_get2_fn     *regset_get;    /* Function to fetch values */
    user_regset_set_fn      *set;           /* Function to store values */
    user_regset_active_fn   *active;        /* Reports if regset is active */
    user_regset_writeback_fn *writeback;    /* Write data back to user memory */
    unsigned int            n;              /* Number of slots (registers) */
    unsigned int            size;           /* Size in bytes of a slot */
    unsigned int            align;          /* Required alignment */
    unsigned int            bias;           /* Bias from natural indexing */
    unsigned int            core_note_type; /* ELF note NT_* value */
};

struct user_regset_view {
    const char *name;                       /* e.g., "x86_64" */
    const struct user_regset *regsets;       /* Array of regsets */
    unsigned int n;                          /* Number of regsets */
    u32 e_flags;
    u16 e_machine;                           /* ELF e_machine (EM_X86_64, etc.) */
    u8 ei_osabi;
};
```

### x86-64 Regset Definitions

```c
/* arch/x86/kernel/ptrace.c:47 */
enum x86_regset_64 {
    REGSET64_GENERAL,   /* NT_PRSTATUS - general purpose registers */
    REGSET64_FP,        /* NT_PRFPREG  - floating point (FXSAVE) */
    REGSET64_IOPERM,    /* NT_386_IOPERM - I/O permissions */
    REGSET64_XSTATE,    /* NT_X86_XSTATE - extended state (XSAVE) */
    REGSET64_SSP,       /* NT_X86_SHSTK - shadow stack pointer */
};

/* arch/x86/kernel/ptrace.c:1237 */
static struct user_regset x86_64_regsets[] __ro_after_init = {
    [REGSET64_GENERAL] = {
        .core_note_type = NT_PRSTATUS,
        .n       = sizeof(struct user_regs_struct) / sizeof(long),
        .size    = sizeof(long),
        .align   = sizeof(long),
        .regset_get = genregs_get,
        .set     = genregs_set
    },
    [REGSET64_FP] = {
        .core_note_type = NT_PRFPREG,
        .n       = sizeof(struct fxregs_state) / sizeof(long),
        .size    = sizeof(long),
        .align   = sizeof(long),
        .active  = regset_xregset_fpregs_active,
        .regset_get = xfpregs_get,
        .set     = xfpregs_set
    },
    [REGSET64_XSTATE] = {
        .core_note_type = NT_X86_XSTATE,
        .size    = sizeof(u64),
        .align   = sizeof(u64),
        .active  = xstateregs_active,
        .regset_get = xstateregs_get,
        .set     = xstateregs_set
    },
    ...
};
```

### Individual Register Access (x86)

On x86, `getreg()` and `putreg()` handle reading and writing individual
registers, with special handling for segment registers, flags, and FS/GS
base addresses (`arch/x86/kernel/ptrace.c:374-430`):

```c
static int putreg(struct task_struct *child,
                  unsigned long offset, unsigned long value)
{
    switch (offset) {
    case offsetof(struct user_regs_struct, cs):
    case offsetof(struct user_regs_struct, ds):
    case offsetof(struct user_regs_struct, es):
    case offsetof(struct user_regs_struct, fs):
    case offsetof(struct user_regs_struct, gs):
    case offsetof(struct user_regs_struct, ss):
        return set_segment_reg(child, offset, value);

    case offsetof(struct user_regs_struct, flags):
        return set_flags(child, value);

    case offsetof(struct user_regs_struct, fs_base):
        if (value >= TASK_SIZE_MAX)
            return -EIO;
        x86_fsbase_write_task(child, value);
        return 0;
    case offsetof(struct user_regs_struct, gs_base):
        if (value >= TASK_SIZE_MAX)
            return -EIO;
        x86_gsbase_write_task(child, value);
        return 0;
    }

    *pt_regs_access(task_pt_regs(child), offset) = value;
    return 0;
}
```

### x86 Register Offset Table

```c
/* arch/x86/kernel/ptrace.c:85 */
static const struct pt_regs_offset regoffset_table[] = {
#ifdef CONFIG_X86_64
    REG_OFFSET_NAME(r15),
    REG_OFFSET_NAME(r14),
    REG_OFFSET_NAME(r13),
    REG_OFFSET_NAME(r12),
    REG_OFFSET_NAME(r11),
    REG_OFFSET_NAME(r10),
    REG_OFFSET_NAME(r9),
    REG_OFFSET_NAME(r8),
#endif
    REG_OFFSET_NAME(bx),
    REG_OFFSET_NAME(cx),
    REG_OFFSET_NAME(dx),
    REG_OFFSET_NAME(si),
    REG_OFFSET_NAME(di),
    REG_OFFSET_NAME(bp),
    REG_OFFSET_NAME(ax),
    REG_OFFSET_NAME(orig_ax),
    REG_OFFSET_NAME(ip),
    REG_OFFSET_NAME(cs),
    REG_OFFSET_NAME(flags),
    REG_OFFSET_NAME(sp),
    REG_OFFSET_NAME(ss),
    REG_OFFSET_END,
};
```

---

## Memory Access

### PTRACE_PEEKDATA / PTRACE_POKEDATA

The simplest memory access operations read or write a single machine word
at a time. Implemented via `ptrace_access_vm()`:

```c
/* kernel/ptrace.c:1295 */
int generic_ptrace_peekdata(struct task_struct *tsk, unsigned long addr,
                            unsigned long data)
{
    unsigned long tmp;
    int copied;

    copied = ptrace_access_vm(tsk, addr, &tmp, sizeof(tmp), FOLL_FORCE);
    if (copied != sizeof(tmp))
        return -EIO;
    return put_user(tmp, (unsigned long __user *)data);
}

/* kernel/ptrace.c:1307 */
int generic_ptrace_pokedata(struct task_struct *tsk, unsigned long addr,
                            unsigned long data)
{
    int copied;

    copied = ptrace_access_vm(tsk, addr, &data, sizeof(data),
                              FOLL_FORCE | FOLL_WRITE);
    return (copied == sizeof(data)) ? 0 : -EIO;
}
```

### ptrace_access_vm()

The core function for accessing a tracee's address space. It validates the
ptrace relationship and uses `access_remote_vm()` to perform the actual
memory access through the tracee's page tables:

```c
/* kernel/ptrace.c:44 */
int ptrace_access_vm(struct task_struct *tsk, unsigned long addr,
                     void *buf, int len, unsigned int gup_flags)
{
    struct mm_struct *mm;
    int ret;

    mm = get_task_mm(tsk);
    if (!mm)
        return 0;

    if (!tsk->ptrace ||
        (current != tsk->parent) ||
        ((get_dumpable(mm) != SUID_DUMP_USER) &&
         !ptracer_capable(tsk, mm->user_ns))) {
        mmput(mm);
        return 0;
    }

    ret = access_remote_vm(mm, addr, buf, len, gup_flags);
    mmput(mm);
    return ret;
}
```

Note the use of `FOLL_FORCE`, which allows writing to read-only mappings (e.g.,
setting breakpoints in text segments).

### Bulk Data Transfer: ptrace_readdata / ptrace_writedata

For reading/writing larger blocks of memory, these functions transfer data
in 128-byte chunks (`kernel/ptrace.c:607`):

```c
int ptrace_readdata(struct task_struct *tsk, unsigned long src,
                    char __user *dst, int len)
{
    int copied = 0;

    while (len > 0) {
        char buf[128];
        int this_len, retval;

        this_len = (len > sizeof(buf)) ? sizeof(buf) : len;
        retval = ptrace_access_vm(tsk, src, buf, this_len, FOLL_FORCE);

        if (!retval) {
            if (copied) break;
            return -EIO;
        }
        if (copy_to_user(dst, buf, retval))
            return -EFAULT;
        copied += retval;
        src += retval;
        dst += retval;
        len -= retval;
    }
    return copied;
}
```

### process_vm_readv / process_vm_writev

These system calls provide a more efficient alternative to ptrace for bulk
memory transfer. They do not require the target to be stopped, but do require
the same `PTRACE_MODE_ATTACH_REALCREDS` access check:

```c
/* mm/process_vm_access.c:151 */
static ssize_t process_vm_rw_core(pid_t pid, struct iov_iter *iter,
                                  const struct iovec *rvec,
                                  unsigned long riovcnt,
                                  unsigned long flags, int vm_write)
{
    struct task_struct *task;
    struct mm_struct *mm;
    ...
    task = find_get_task_by_vpid(pid);
    if (!task) {
        rc = -ESRCH;
        goto free_proc_pages;
    }

    mm = mm_access(task, PTRACE_MODE_ATTACH_REALCREDS);
    if (IS_ERR(mm)) {
        rc = PTR_ERR(mm);
        if (rc == -EACCES)
            rc = -EPERM;
        goto put_task_struct;
    }
    ...
}
```

### /proc/pid/mem

The proc filesystem provides `/proc/<pid>/mem` for reading and writing a
process's memory. This interface also uses `ptrace_may_access()` for
permission checks and allows seeking to arbitrary virtual addresses. Unlike
PTRACE_PEEKDATA, it can transfer arbitrary amounts of data in a single
read/write call.

### Memory Access Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│              Memory Access Methods Comparison                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Method               Requires    Granularity  Target Must       │
│                       Attach?                  Be Stopped?       │
│  ────────────────────────────────────────────────────────────    │
│  PTRACE_PEEKDATA      Yes         1 word       Yes               │
│  PTRACE_POKEDATA      Yes         1 word       Yes               │
│  ptrace_readdata      Yes         bulk         Yes               │
│  process_vm_readv     No*         bulk (iov)   No                │
│  process_vm_writev    No*         bulk (iov)   No                │
│  /proc/pid/mem        No*         arbitrary    No                │
│                                                                  │
│  *These use PTRACE_MODE_ATTACH_REALCREDS access check            │
│   (same permission model as ptrace attach)                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Single-Stepping

### How Single-Stepping Works on x86

On x86, single-stepping is implemented using the CPU's Trap Flag (TF) in the
EFLAGS register. When TF is set, the CPU generates a `#DB` (debug exception)
after executing each instruction.

The implementation is in `arch/x86/kernel/step.c`:

```c
/* arch/x86/kernel/step.c:113 */
static int enable_single_step(struct task_struct *child)
{
    struct pt_regs *regs = task_pt_regs(child);
    unsigned long oflags;

    /*
     * If we stepped into a sysenter/syscall insn, it trapped in
     * kernel mode; do_debug() cleared TF and set TIF_SINGLESTEP.
     */
    if (unlikely(test_tsk_thread_flag(child, TIF_SINGLESTEP)))
        regs->flags |= X86_EFLAGS_TF;

    /* Always set TIF_SINGLESTEP - also causes TF to be set on return */
    set_tsk_thread_flag(child, TIF_SINGLESTEP);

    /* Ensure trap is triggered even when stepping out of a syscall */
    set_task_syscall_work(child, SYSCALL_EXIT_TRAP);

    oflags = regs->flags;

    /* Set TF on the kernel stack */
    regs->flags |= X86_EFLAGS_TF;

    /*
     * Track whether the debugger set TF or the user program did.
     * If the next instruction changes TF (e.g., popf), don't
     * mark it as debugger-set.
     */
    if (is_setting_trap_flag(child, regs)) {
        clear_tsk_thread_flag(child, TIF_FORCED_TF);
        return 0;
    }

    if (oflags & X86_EFLAGS_TF)
        return test_tsk_thread_flag(child, TIF_FORCED_TF);

    set_tsk_thread_flag(child, TIF_FORCED_TF);
    return 1;
}
```

### user_enable_single_step / user_disable_single_step

These are the architecture-independent API functions:

```c
/* arch/x86/kernel/step.c:219 */
void user_enable_single_step(struct task_struct *child)
{
    enable_step(child, 0);  /* single-step (not block-step) */
}

void user_enable_block_step(struct task_struct *child)
{
    enable_step(child, 1);  /* step until next branch */
}

/* arch/x86/kernel/step.c:229 */
void user_disable_single_step(struct task_struct *child)
{
    /* Disable block stepping (BTF) */
    if (test_tsk_thread_flag(child, TIF_BLOCKSTEP))
        set_task_blockstep(child, false);

    /* Always clear TIF_SINGLESTEP */
    clear_tsk_thread_flag(child, TIF_SINGLESTEP);
    clear_task_syscall_work(child, SYSCALL_EXIT_TRAP);

    /* Only clear TF if we set it (TIF_FORCED_TF) */
    if (test_and_clear_tsk_thread_flag(child, TIF_FORCED_TF))
        task_pt_regs(child)->flags &= ~X86_EFLAGS_TF;
}
```

### Block Stepping

x86 supports "block stepping" via the Branch Trap Flag (BTF) in the
`DEBUGCTLMSR`. When BTF is set, the CPU generates a debug exception only on
branch instructions (call, jmp, ret, etc.) rather than on every instruction:

```c
/* arch/x86/kernel/step.c:174 */
void set_task_blockstep(struct task_struct *task, bool on)
{
    unsigned long debugctl;

    local_irq_disable();
    debugctl = get_debugctlmsr();
    if (on) {
        debugctl |= DEBUGCTLMSR_BTF;
        set_tsk_thread_flag(task, TIF_BLOCKSTEP);
    } else {
        debugctl &= ~DEBUGCTLMSR_BTF;
        clear_tsk_thread_flag(task, TIF_BLOCKSTEP);
    }
    if (task == current)
        update_debugctlmsr(debugctl);
    local_irq_enable();
}
```

### TF Flag Management

The kernel must carefully track who set the Trap Flag:

```
┌───────────────────────────────────────────────────────────────┐
│                TF Flag Ownership Tracking                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  TIF_SINGLESTEP   - Set when single-stepping is active        │
│  TIF_FORCED_TF    - Set when the debugger (not user code)     │
│                     is responsible for TF being set           │
│  TIF_BLOCKSTEP    - Set when block-stepping (BTF) is active   │
│                                                               │
│  When reading flags (getreg):                                 │
│    If TIF_FORCED_TF is set, hide TF from the readout          │
│    (so the user program doesn't see the debugger's TF)        │
│                                                               │
│  When disabling single-step:                                  │
│    Only clear TF if TIF_FORCED_TF is set (debugger set it)    │
│    If user code set TF, leave it alone                        │
│                                                               │
│  arch/x86/kernel/ptrace.c:342:                                │
│    static unsigned long get_flags(struct task_struct *task) { │
│        unsigned long retval = task_pt_regs(task)->flags;      │
│        if (test_tsk_thread_flag(task, TIF_FORCED_TF))         │
│            retval &= ~X86_EFLAGS_TF;                          │
│        return retval;                                         │
│    }                                                          │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### ptrace_disable() on Detach

When detaching, the architecture ensures single-stepping is turned off:

```c
/* arch/x86/kernel/ptrace.c:718 */
void ptrace_disable(struct task_struct *child)
{
    user_disable_single_step(child);
}
```

---

## Security Model

### Access Check: __ptrace_may_access()

The core access check is `__ptrace_may_access()` (`kernel/ptrace.c:276`). It
enforces a multi-layered security model:

```c
static int __ptrace_may_access(struct task_struct *task, unsigned int mode)
{
    const struct cred *cred = current_cred(), *tcred;
    struct mm_struct *mm;
    kuid_t caller_uid;
    kgid_t caller_gid;

    /* Validate that exactly one of FSCREDS/REALCREDS is set */
    if (!(mode & PTRACE_MODE_FSCREDS) == !(mode & PTRACE_MODE_REALCREDS)) {
        WARN(1, "denying ptrace access check without PTRACE_MODE_*CREDS\n");
        return -EPERM;
    }

    /* Self-tracing is always allowed */
    if (same_thread_group(task, current))
        return 0;

    /* UID/GID check: caller must match all of target's UIDs/GIDs */
    rcu_read_lock();
    if (mode & PTRACE_MODE_FSCREDS) {
        caller_uid = cred->fsuid;
        caller_gid = cred->fsgid;
    } else {
        caller_uid = cred->uid;
        caller_gid = cred->gid;
    }
    tcred = __task_cred(task);
    if (uid_eq(caller_uid, tcred->euid) &&
        uid_eq(caller_uid, tcred->suid) &&
        uid_eq(caller_uid, tcred->uid)  &&
        gid_eq(caller_gid, tcred->egid) &&
        gid_eq(caller_gid, tcred->sgid) &&
        gid_eq(caller_gid, tcred->gid))
        goto ok;

    /* Or caller has CAP_SYS_PTRACE */
    if (ptrace_has_cap(tcred->user_ns, mode))
        goto ok;

    rcu_read_unlock();
    return -EPERM;
ok:
    rcu_read_unlock();

    /* Check dumpability: non-dumpable processes require CAP_SYS_PTRACE */
    smp_rmb();
    mm = task->mm;
    if (mm &&
        ((get_dumpable(mm) != SUID_DUMP_USER) &&
         !ptrace_has_cap(mm->user_ns, mode)))
        return -EPERM;

    /* Finally, consult LSM (YAMA, SELinux, etc.) */
    return security_ptrace_access_check(task, mode);
}
```

### Access Mode Flags

```c
/* include/linux/ptrace.h:62-72 */
#define PTRACE_MODE_READ        0x01    /* Read-only access (e.g., /proc) */
#define PTRACE_MODE_ATTACH      0x02    /* Full attach access */
#define PTRACE_MODE_NOAUDIT     0x04    /* Don't generate audit events */
#define PTRACE_MODE_FSCREDS     0x08    /* Use fsuid/fsgid (filesystem ops) */
#define PTRACE_MODE_REALCREDS   0x10    /* Use real uid/gid (explicit syscalls) */

/* Common combinations */
#define PTRACE_MODE_READ_FSCREDS    (PTRACE_MODE_READ | PTRACE_MODE_FSCREDS)
#define PTRACE_MODE_READ_REALCREDS  (PTRACE_MODE_READ | PTRACE_MODE_REALCREDS)
#define PTRACE_MODE_ATTACH_FSCREDS  (PTRACE_MODE_ATTACH | PTRACE_MODE_FSCREDS)
#define PTRACE_MODE_ATTACH_REALCREDS (PTRACE_MODE_ATTACH | PTRACE_MODE_REALCREDS)
```

### CAP_SYS_PTRACE

The `CAP_SYS_PTRACE` capability grants elevated ptrace privileges:
- Bypass UID/GID matching checks
- Trace non-dumpable processes (e.g., setuid binaries)
- Override some YAMA restrictions
- Access `/proc/<pid>/mem` for otherwise inaccessible processes

### YAMA Linux Security Module

YAMA provides additional ptrace restrictions via `/proc/sys/kernel/yama/ptrace_scope`.
Implemented in `security/yama/yama_lsm.c:356`:

```c
static int yama_ptrace_access_check(struct task_struct *child,
                                    unsigned int mode)
{
    int rc = 0;

    if (mode & PTRACE_MODE_ATTACH) {
        switch (ptrace_scope) {
        case YAMA_SCOPE_DISABLED:       /* 0 - No additional restrictions */
            break;
        case YAMA_SCOPE_RELATIONAL:     /* 1 - Only descendants (default on many distros) */
            rcu_read_lock();
            if (!pid_alive(child))
                rc = -EPERM;
            if (!rc && !task_is_descendant(current, child) &&
                !ptracer_exception_found(current, child) &&
                !ns_capable(__task_cred(child)->user_ns, CAP_SYS_PTRACE))
                rc = -EPERM;
            rcu_read_unlock();
            break;
        case YAMA_SCOPE_CAPABILITY:     /* 2 - Only CAP_SYS_PTRACE holders */
            rcu_read_lock();
            if (!ns_capable(__task_cred(child)->user_ns, CAP_SYS_PTRACE))
                rc = -EPERM;
            rcu_read_unlock();
            break;
        case YAMA_SCOPE_NO_ATTACH:      /* 3 - No attach at all */
        default:
            rc = -EPERM;
            break;
        }
    }
    return rc;
}
```

### YAMA Scope Levels

```
┌───────────────────────────────────────────────────────────────┐
│             YAMA ptrace_scope Levels                          │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Level 0 - YAMA_SCOPE_DISABLED                                │
│    Classic behavior. Any process with matching UIDs can trace.│
│                                                               │
│  Level 1 - YAMA_SCOPE_RELATIONAL (common default)             │
│    Only descendants of the tracer can be traced.              │
│    Exceptions: prctl(PR_SET_PTRACER, pid) allows specific     │
│    processes. CAP_SYS_PTRACE overrides.                       │
│                                                               │
│  Level 2 - YAMA_SCOPE_CAPABILITY                              │
│    Only processes with CAP_SYS_PTRACE can attach.             │
│    PTRACE_TRACEME is also restricted.                         │
│                                                               │
│  Level 3 - YAMA_SCOPE_NO_ATTACH                               │
│    No process can attach to another. Only PTRACE_TRACEME      │
│    by the tracee itself is possible (for debuggers that       │
│    fork-and-exec under ptrace).                               │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Security Check Flow

```
┌──────────────────────────────────────────────────────────┐
│           ptrace Security Check Chain                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ptrace_attach()                                         │
│       │                                                  │
│       ▼                                                  │
│  1. Not a kernel thread?   (PF_KTHREAD check)            │
│       │                                                  │
│       ▼                                                  │
│  2. Not same thread group?                               │
│       │                                                  │
│       ▼                                                  │
│  3. cred_guard_mutex (protect against exec race)         │
│       │                                                  │
│       ▼                                                  │
│  4. __ptrace_may_access(PTRACE_MODE_ATTACH_REALCREDS)    │
│       │                                                  │
│       ├── Same thread group? → allow                     │
│       ├── UID/GID match all creds? → ok                  │
│       ├── CAP_SYS_PTRACE? → ok                           │
│       ├── Otherwise → deny                               │
│       │                                                  │
│       ▼  (if ok)                                         │
│  5. Dumpability check                                    │
│       │  SUID_DUMP_USER → ok                             │
│       │  Otherwise → need CAP_SYS_PTRACE                 │
│       │                                                  │
│       ▼                                                  │
│  6. security_ptrace_access_check()                       │
│       ├── YAMA check                                     │
│       ├── SELinux check (if enabled)                     │
│       └── AppArmor check (if enabled)                    │
│                                                          │
│  7. Not already traced? Not exiting?                     │
│       │                                                  │
│       ▼                                                  │
│  Attach succeeds                                         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### PTRACE_O_SUSPEND_SECCOMP

A special option that allows a tracer with `CAP_SYS_ADMIN` to suspend the
tracee's seccomp filters. This is restricted to checkpoint/restore use cases:

```c
/* kernel/ptrace.c:358 */
static int check_ptrace_options(unsigned long data)
{
    if (data & ~(unsigned long)PTRACE_O_MASK)
        return -EINVAL;

    if (unlikely(data & PTRACE_O_SUSPEND_SECCOMP)) {
        if (!IS_ENABLED(CONFIG_CHECKPOINT_RESTORE) ||
            !IS_ENABLED(CONFIG_SECCOMP))
            return -EINVAL;

        if (!capable(CAP_SYS_ADMIN))
            return -EPERM;

        if (seccomp_mode(&current->seccomp) != SECCOMP_MODE_DISABLED ||
            current->ptrace & PT_SUSPEND_SECCOMP)
            return -EPERM;
    }
    return 0;
}
```

---

## Integration with Signals, Seccomp, and BPF Uprobes

### Signals and ptrace

Signals are the primary mechanism by which ptrace stops are communicated.
When a signal is delivered to a traced process, the kernel intercepts it
in `get_signal()` and calls `ptrace_stop()` instead of delivering the signal
directly. The tracer observes the stop via `waitpid()` and can:

- Inspect the signal via `PTRACE_GETSIGINFO`
- Modify the signal via `PTRACE_SETSIGINFO`
- Suppress the signal by resuming with `data = 0`
- Inject a different signal by resuming with `data = signum`

```c
/* Signal mask manipulation via ptrace (kernel/ptrace.c:1076) */
case PTRACE_SETSIGMASK: {
    sigset_t new_set;
    ...
    if (copy_from_user(&new_set, datavp, sizeof(sigset_t))) {
        ret = -EFAULT;
        break;
    }
    /* SIGKILL and SIGSTOP can never be masked */
    sigdelsetmask(&new_set, sigmask(SIGKILL)|sigmask(SIGSTOP));

    spin_lock_irq(&child->sighand->siglock);
    child->blocked = new_set;
    spin_unlock_irq(&child->sighand->siglock);
    ...
}
```

### PTRACE_O_TRACESYSGOOD

To distinguish syscall stops from signal-delivery stops, tracers can set
`PTRACE_O_TRACESYSGOOD`. This causes syscall stops to report `SIGTRAP | 0x80`
instead of plain `SIGTRAP`:

```c
/* include/linux/ptrace.h:415 */
signr = ptrace_notify(SIGTRAP | ((ptrace & PT_TRACESYSGOOD) ? 0x80 : 0),
                      message);
```

### Seccomp Integration

When a seccomp filter returns `SECCOMP_RET_TRACE`, the kernel generates a
`PTRACE_EVENT_SECCOMP` stop, allowing the tracer to inspect and potentially
modify the system call:

```c
/* kernel/seccomp.c:1255 */
case SECCOMP_RET_TRACE:
    if (recheck_after_trace)
        return 0;

    /* If no tracer, return ENOSYS */
    if (!ptrace_event_enabled(current, PTRACE_EVENT_SECCOMP)) {
        syscall_set_return_value(current, current_pt_regs(), -ENOSYS, 0);
        goto skip;
    }

    /* Fire the ptrace event */
    ptrace_event(PTRACE_EVENT_SECCOMP, data);

    if (fatal_signal_pending(current))
        goto skip;

    /* Check if tracer changed the syscall */
    this_syscall = syscall_get_nr(current, current_pt_regs());
    if (this_syscall < 0)
        goto skip;
    ...
```

The tracer can use `PTRACE_GET_SYSCALL_INFO` with `PTRACE_SYSCALL_INFO_SECCOMP`
to get full details about the seccomp stop, including the filter's return data.

Seccomp filter inspection is also available:

```c
/* kernel/ptrace.c:1229 */
case PTRACE_SECCOMP_GET_FILTER:
    ret = seccomp_get_filter(child, addr, datavp);
    break;

case PTRACE_SECCOMP_GET_METADATA:
    ret = seccomp_get_metadata(child, addr, datavp);
    break;
```

### BPF Uprobes

Uprobes (user-space probes) use the same underlying single-step and
breakpoint mechanisms as ptrace. The uprobe subsystem
(`kernel/events/uprobes.c`) uses `user_enable_single_step()` to single-step
the original instruction after executing it out-of-line:

```c
/* kernel/events/uprobes.c - includes linux/ptrace.h for single-step API */
#include <linux/ptrace.h>   /* user_enable_single_step */
```

When a uprobe fires and no matching uprobe handler is found, it signals
SIGTRAP to the task, which a ptrace tracer will observe:

```c
/* kernel/events/uprobes.c:2666 */
/* No matching uprobe; signal SIGTRAP. */
force_sig(SIGTRAP);
```

### Event Integration Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│              ptrace Event Sources Integration                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Signal Delivery                                                 │
│    └──► get_signal() ──► ptrace_stop(signo)                      │
│                           └──► tracer: waitpid() → WIFSTOPPED    │
│                                                                  │
│  Syscall Entry/Exit                                              │
│    └──► syscall_trace_enter/exit()                               │
│           └──► ptrace_report_syscall_entry/exit()                │
│                 └──► ptrace_stop(SIGTRAP|0x80)                   │
│                       └──► tracer: waitpid() → WIFSTOPPED        │
│                                                                  │
│  Seccomp Filter (SECCOMP_RET_TRACE)                              │
│    └──► __seccomp_filter()                                       │
│           └──► ptrace_event(PTRACE_EVENT_SECCOMP, data)          │
│                 └──► ptrace_stop((PTRACE_EVENT_SECCOMP<<8)|TRAP) │
│                       └──► tracer: waitpid() → WIFSTOPPED        │
│                            status >> 16 == PTRACE_EVENT_SECCOMP  │
│                                                                  │
│  Process Events (fork/exec/exit)                                 │
│    └──► ptrace_event(PTRACE_EVENT_FORK/EXEC/EXIT, msg)           │
│           └──► ptrace_stop((event<<8)|SIGTRAP)                   │
│                 └──► tracer: waitpid() → WIFSTOPPED              │
│                      PTRACE_GETEVENTMSG → child pid / exit code  │
│                                                                  │
│  Single-Step Trap                                                │
│    └──► do_debug() ──► SIGTRAP                                   │
│           └──► ptrace_stop(SIGTRAP)                              │
│                                                                  │
│  Hardware Breakpoint / Watchpoint                                │
│    └──► do_debug() ──► SIGTRAP                                   │
│           └──► ptrace_stop(SIGTRAP)                              │
│                                                                  │
│  Uprobe (BPF)                                                    │
│    └──► uprobe_notify_resume()                                   │
│           └──► user_enable_single_step() for out-of-line exec    │
│           └──► force_sig(SIGTRAP) if no handler                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### ptrace_init_task() - New Task Initialization

When a traced process forks, the kernel initializes ptrace state in the
new child:

```c
/* include/linux/ptrace.h:200 */
static inline void ptrace_init_task(struct task_struct *child, bool ptrace)
{
    INIT_LIST_HEAD(&child->ptrace_entry);
    INIT_LIST_HEAD(&child->ptraced);
    child->jobctl = 0;
    child->ptrace = 0;
    child->parent = child->real_parent;

    if (unlikely(ptrace) && current->ptrace) {
        child->ptrace = current->ptrace;
        __ptrace_link(child, current->parent, current->ptracer_cred);

        if (child->ptrace & PT_SEIZED)
            task_set_jobctl_pending(child, JOBCTL_TRAP_STOP);
        else
            sigaddset(&child->pending.signal, SIGSTOP);
    }
    else
        child->ptracer_cred = NULL;
}
```

### Detach and Tracer Exit

When a tracer detaches or exits, the tracee must be returned to a consistent
state:

```c
/* kernel/ptrace.c:117 - __ptrace_unlink() */
void __ptrace_unlink(struct task_struct *child)
{
    BUG_ON(!child->ptrace);

    clear_task_syscall_work(child, SYSCALL_TRACE);
    clear_task_syscall_work(child, SYSCALL_EMU);

    child->parent = child->real_parent;
    list_del_init(&child->ptrace_entry);
    child->ptracer_cred = NULL;

    spin_lock(&child->sighand->siglock);
    child->ptrace = 0;

    /* Clear all pending traps */
    task_clear_jobctl_pending(child, JOBCTL_TRAP_MASK);
    task_clear_jobctl_trapping(child);

    /* Reinstate group stop if needed */
    if (!(child->flags & PF_EXITING) &&
        (child->signal->flags & SIGNAL_STOP_STOPPED ||
         child->signal->group_stop_count))
        child->jobctl |= JOBCTL_STOP_PENDING;

    /* Wake up the tracee if it was traced or needs to stop */
    if (child->jobctl & JOBCTL_STOP_PENDING || task_is_traced(child))
        ptrace_signal_wake_up(child, true);

    spin_unlock(&child->sighand->siglock);
}
```

When the tracer exits, `exit_ptrace()` detaches all tracees, optionally
sending SIGKILL to those with `PT_EXITKILL`:

```c
/* kernel/ptrace.c:594 */
void exit_ptrace(struct task_struct *tracer, struct list_head *dead)
{
    struct task_struct *p, *n;

    list_for_each_entry_safe(p, n, &tracer->ptraced, ptrace_entry) {
        if (unlikely(p->ptrace & PT_EXITKILL))
            send_sig_info(SIGKILL, SEND_SIG_PRIV, p);

        if (__ptrace_detach(tracer, p))
            list_add(&p->ptrace_entry, dead);
    }
}
```

---

## Source File Reference

| File | Description |
|------|-------------|
| `kernel/ptrace.c` | Core ptrace implementation: attach/detach, memory access, resume, regset dispatch, syscall info |
| `include/linux/ptrace.h` | Kernel-internal ptrace API: flags (PT_*), inline helpers, single-step interface, regset declarations |
| `include/uapi/linux/ptrace.h` | User-space API: request constants, option flags, event types, struct definitions |
| `include/linux/sched.h` | `task_struct` definition including ptrace fields (ptrace, parent, real_parent, ptraced, etc.) |
| `include/linux/sched/jobctl.h` | Job control flags: JOBCTL_TRACED, JOBCTL_TRAP_*, JOBCTL_LISTENING |
| `include/linux/regset.h` | `user_regset` and `user_regset_view` structures for register access abstraction |
| `arch/x86/kernel/ptrace.c` | x86-specific ptrace: arch_ptrace(), register access (getreg/putreg), regset definitions, debug registers, hardware breakpoints |
| `arch/x86/kernel/step.c` | x86 single-step implementation: TF flag management, block stepping via BTF |
| `kernel/signal.c` | `ptrace_stop()` implementation where tracees enter the TRACED state |
| `security/yama/yama_lsm.c` | YAMA LSM: ptrace_scope levels, ptracer exceptions |
| `mm/process_vm_access.c` | `process_vm_readv()`/`process_vm_writev()` system calls |
| `kernel/seccomp.c` | Seccomp filter interaction with ptrace via SECCOMP_RET_TRACE |
| `kernel/events/uprobes.c` | Uprobe infrastructure using ptrace single-step mechanism |
