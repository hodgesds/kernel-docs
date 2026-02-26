# Linux Kernel Process Lifecycle Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Task States](#task-states)
3. [Process Creation (fork)](#process-creation-fork)
4. [clone() System Call](#clone-system-call)
5. [Memory Mapping (mmap)](#memory-mapping-mmap)
6. [Program Execution (exec)](#program-execution-exec)
7. [Dynamic Linking](#dynamic-linking)
8. [Process Termination (exit)](#process-termination-exit)
9. [Wait and Reaping](#wait-and-reaping)
10. [Signals](#signals)
11. [Credentials and Capabilities](#credentials-and-capabilities)
12. [Namespaces](#namespaces)
13. [Control Groups (cgroups)](#control-groups-cgroups)
14. [copy_process() Internals](#copy_process-internals)
15. [exec Internals](#exec-internals)
16. [Signal Delivery Internals](#signal-delivery-internals)
17. [Process Exit Path](#process-exit-path)
18. [Locking Summary](#locking-summary)
19. [Source File Reference](#source-file-reference)

---

## Introduction

A process in Linux is an instance of a running program. The kernel maintains
extensive state for each process and provides mechanisms for process creation,
execution, and termination.

### Process vs Thread

```
┌─────────────────────────────────────────────────────────────┐
│                 Process vs Thread                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Process                                                    │
│  ───────                                                    │
│  - Independent address space                                │
│  - Own file descriptors, credentials                        │
│  - Created with fork()                                      │
│  - Heavier to create/switch                                 │
│                                                             │
│  Thread (Linux: "lightweight process")                      │
│  ──────────────────────────────────                         │
│  - Shared address space                                     │
│  - Shared file descriptors (mostly)                         │
│  - Created with clone() with CLONE_VM                       │
│  - Lighter to create/switch                                 │
│                                                             │
│  In Linux kernel:                                           │
│  - Both represented by task_struct                          │
│  - "Process" = thread group                                 │
│  - "Thread" = individual task                               │
│  - tgid = thread group ID (what userspace calls PID)        │
│  - pid = task ID (unique per task)                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Task States

Every task has a state indicating what it's currently doing.

### Task State Definitions

```c
/* include/linux/sched.h */

/* Task state flags */
#define TASK_RUNNING            0x00000000  /* On runqueue or running */
#define TASK_INTERRUPTIBLE      0x00000001  /* Sleeping, wake on signal */
#define TASK_UNINTERRUPTIBLE    0x00000002  /* Sleeping, ignore signals */
#define __TASK_STOPPED          0x00000004  /* Stopped by signal */
#define __TASK_TRACED           0x00000008  /* Stopped by ptrace */

/* Exit states */
#define EXIT_DEAD               0x00000010  /* Final state */
#define EXIT_ZOMBIE             0x00000020  /* Terminated, awaiting wait() */

/* Special states */
#define TASK_PARKED             0x00000040  /* Kthread is parked */
#define TASK_DEAD               0x00000080  /* Exiting */
#define TASK_WAKEKILL           0x00000100  /* Wake on fatal signal */
#define TASK_WAKING             0x00000200  /* Being woken */
#define TASK_NOLOAD             0x00000400  /* Don't count for load */
#define TASK_NEW                0x00000800  /* Just created, not yet run */
#define TASK_RTLOCK_WAIT        0x00001000  /* RT mutex wait */
#define TASK_FREEZABLE          0x00002000  /* Can be frozen */

/* Combinations */
#define TASK_KILLABLE           (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED            (__TASK_STOPPED | TASK_WAKEKILL)
#define TASK_TRACED             (__TASK_TRACED | TASK_WAKEKILL)
```

### State Transitions

```
┌─────────────────────────────────────────────────────────────┐
│                   Task State Machine                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌──────────────┐                       │
│                      │  TASK_NEW    │                       │
│                      │ (just forked)│                       │
│                      └──────┬───────┘                       │
│                             │ wake_up_new_task()            │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   TASK_RUNNING                      │    │
│  │          (on runqueue or actively running)          │    │
│  └────────────┬─────────────────────────┬──────────────┘    │
│               │                         │                   │
│   schedule()  │                         │ do_exit()         │
│   (voluntary  │                         │                   │
│   or preempt) │                         ▼                   │
│               ▼                   ┌──────────────┐          │
│  ┌────────────────────────┐       │ EXIT_ZOMBIE  │          │
│  │   TASK_INTERRUPTIBLE   │       │(waiting for  │          │
│  │   (sleeping, can be    │       │ parent wait) │          │
│  │    woken by signal)    │       └──────┬───────┘          │
│  └────────────┬───────────┘              │                  │
│               │                          │ wait()           │
│   wake_up()   │                          ▼                  │
│   or signal   │                   ┌──────────────┐          │
│               │                   │  EXIT_DEAD   │          │
│               ▼                   │  (released)  │          │
│  ┌────────────────────────┐       └──────────────┘          │
│  │  TASK_UNINTERRUPTIBLE  │                                 │
│  │  (sleeping, ignores    │                                 │
│  │   most signals)        │                                 │
│  └────────────────────────┘                                 │
│               │                                             │
│   wake_up()   │                                             │
│               ▼                                             │
│        [back to RUNNING]                                    │
│                                                             │
│  Other states:                                              │
│  ─────────────                                              │
│  TASK_STOPPED  ← SIGSTOP, SIGTSTP                           │
│              → SIGCONT                                      │
│  TASK_TRACED  ← ptrace attach                               │
│              → ptrace detach                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Setting Task State

```c
/* Set current task state */
set_current_state(TASK_INTERRUPTIBLE);

/* Optimized version (no memory barrier) */
__set_current_state(TASK_RUNNING);

/* Common pattern for sleeping */
set_current_state(TASK_INTERRUPTIBLE);
if (condition)
    schedule();  /* Actually sleep */
__set_current_state(TASK_RUNNING);
```

---

## Process Creation (fork)

Process creation in Linux happens through the clone() system call, with fork()
and vfork() as wrappers.

### Clone Flags

```c
/* include/uapi/linux/sched.h */

/* Resource sharing flags */
#define CLONE_VM             0x00000100  /* Share virtual memory */
#define CLONE_FS             0x00000200  /* Share fs info (cwd, umask) */
#define CLONE_FILES          0x00000400  /* Share file descriptors */
#define CLONE_SIGHAND        0x00000800  /* Share signal handlers */
#define CLONE_PIDFD          0x00001000  /* Create pidfd */
#define CLONE_PTRACE         0x00002000  /* Trace child too */
#define CLONE_VFORK          0x00004000  /* Parent waits for exec */
#define CLONE_PARENT         0x00008000  /* Same parent as caller */
#define CLONE_THREAD         0x00010000  /* Same thread group */
#define CLONE_NEWNS          0x00020000  /* New mount namespace */
#define CLONE_SYSVSEM        0x00040000  /* Share SysV semaphore undo */
#define CLONE_SETTLS         0x00080000  /* Set TLS for child */
#define CLONE_PARENT_SETTID  0x00100000  /* Set TID in parent */
#define CLONE_CHILD_CLEARTID 0x00200000  /* Clear TID on exit */
#define CLONE_DETACHED       0x00400000  /* Unused */
#define CLONE_UNTRACED       0x00800000  /* No ptrace inherit */
#define CLONE_CHILD_SETTID   0x01000000  /* Set TID in child */
#define CLONE_NEWCGROUP      0x02000000  /* New cgroup namespace */
#define CLONE_NEWUTS         0x04000000  /* New UTS namespace */
#define CLONE_NEWIPC         0x08000000  /* New IPC namespace */
#define CLONE_NEWUSER        0x10000000  /* New user namespace */
#define CLONE_NEWPID         0x20000000  /* New PID namespace */
#define CLONE_NEWNET         0x40000000  /* New network namespace */
#define CLONE_IO             0x80000000  /* Share I/O context */
```

### Fork Implementations

```
┌─────────────────────────────────────────────────────────────┐
│                  Fork Variants                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  fork()                                                     │
│  ───────                                                    │
│  clone(SIGCHLD, 0)                                          │
│  - Copy all resources                                       │
│  - Child gets copy of address space (COW)                   │
│  - Most common process creation                             │
│                                                             │
│  vfork()                                                    │
│  ────────                                                   │
│  clone(CLONE_VFORK | CLONE_VM | SIGCHLD, 0)                 │
│  - Parent suspended until child exec/exit                   │
│  - Child borrows parent's address space                     │
│  - Dangerous: child must not modify stack                   │
│  - Rarely used (fork+COW is efficient)                      │
│                                                             │
│  pthread_create() [glibc]                                   │
│  ────────────────────────                                   │
│  clone(CLONE_VM | CLONE_FS | CLONE_FILES |                  │
│        CLONE_SIGHAND | CLONE_THREAD | CLONE_SYSVSEM |       │
│        CLONE_SETTLS | CLONE_PARENT_SETTID |                 │
│        CLONE_CHILD_CLEARTID, ...)                           │
│  - Share everything, same thread group                      │
│                                                             │
│  clone3() [Modern]                                          │
│  ─────────────────                                          │
│  - Extended clone with struct clone_args                    │
│  - Supports pidfd creation                                  │
│  - More flags, better extensibility                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Fork Implementation (kernel_clone)

```c
/* kernel/fork.c - simplified */
pid_t kernel_clone(struct kernel_clone_args *args)
{
    struct task_struct *p;
    pid_t pid;

    /* Copy the process */
    p = copy_process(NULL, args);
    if (IS_ERR(p))
        return PTR_ERR(p);

    /* Get PID */
    pid = get_task_pid(p, PIDTYPE_PID);

    /* Wake up new task */
    wake_up_new_task(p);

    /* For vfork, wait for child to exec/exit */
    if (args->flags & CLONE_VFORK) {
        wait_for_vfork_done(p, &vfork_done);
    }

    return pid;
}
```

### copy_process Flow

```
┌─────────────────────────────────────────────────────────────┐
│                  copy_process() Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. dup_task_struct()                                       │
│     - Allocate task_struct and kernel stack                 │
│     - Copy parent's task_struct                             │
│                                                             │
│  2. copy_creds()                                            │
│     - Duplicate credentials                                 │
│     - Apply any requested credential changes                │
│                                                             │
│  3. sched_fork()                                            │
│     - Initialize scheduler state                            │
│     - Set up fair share of CPU                              │
│                                                             │
│  4. copy_files() / copy_fs() / copy_mm() ...                │
│     - Based on CLONE_* flags:                               │
│       - CLONE_FILES: share file table                       │
│       - !CLONE_FILES: duplicate file table                  │
│       - CLONE_VM: share mm_struct                           │
│       - !CLONE_VM: dup_mm() (COW pages)                     │
│                                                             │
│  5. copy_namespaces()                                       │
│     - Create new namespaces if requested                    │
│                                                             │
│  6. copy_thread()                                           │
│     - Architecture-specific                                 │
│     - Set up return value (0 for child)                     │
│     - Configure child's registers                           │
│                                                             │
│  7. alloc_pid()                                             │
│     - Allocate PID in namespace                             │
│                                                             │
│  8. Various initializations                                 │
│     - Signals, timers, cgroups, etc.                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Copy-on-Write (COW)

```
┌─────────────────────────────────────────────────────────────┐
│                 Copy-on-Write (COW)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Before fork:                                               │
│  ┌──────────┐     ┌──────────────────────────┐              │
│  │  Parent  │───▶│     Physical Pages       │              │
│  │   mm     │     │  [Page 1][Page 2][...]   │              │
│  └──────────┘     └──────────────────────────┘              │
│                                                             │
│  After fork (no copy yet):                                  │
│  ┌──────────┐                                               │
│  │  Parent  │──┐                                            │
│  │   mm     │  │  ┌──────────────────────────┐              │
│  └──────────┘  └▶│     Physical Pages       │              │
│  ┌──────────┐  ┌▶│  [Page 1][Page 2][...]   │              │
│  │  Child   │──┘  │    (marked read-only)    │              │
│  │   mm     │     └──────────────────────────┘              │
│  └──────────┘                                               │
│                                                             │
│  After child writes to Page 2:                              │
│  ┌──────────┐     ┌──────────────────────────┐              │
│  │  Parent  │───▶│  [Page 1]     [Page 2]   │              │
│  │   mm     │     └──────────────────────────┘              │
│  └──────────┘                                               │
│  ┌──────────┐     ┌──────────────────────────┐              │
│  │  Child   │───▶│  [Page 1]     [Page 2']  │ ← New copy   │
│  │   mm     │     └──────────────────────────┘              │
│  └──────────┘                                               │
│                                                             │
│  Benefits:                                                  │
│  - Fast fork (no immediate copy)                            │
│  - Memory efficient (shared read-only pages)                │
│  - Amortized copy cost (only on write)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## clone() System Call

The clone() system call is the fundamental building block for creating new execution
contexts in Linux. Both fork() and pthread_create() are implemented on top of clone().

### clone() vs clone3()

```c
/* Traditional clone() */
long clone(unsigned long flags,
           void *stack,
           int *parent_tid,
           int *child_tid,
           unsigned long tls);

/* Modern clone3() with extensible struct */
long clone3(struct clone_args *args, size_t size);

struct clone_args {
    __u64 flags;           /* Clone flags */
    __u64 pidfd;           /* Pointer to store pidfd */
    __u64 child_tid;       /* Pointer to child TID */
    __u64 parent_tid;      /* Pointer to parent TID */
    __u64 exit_signal;     /* Signal on child exit */
    __u64 stack;           /* Stack for child */
    __u64 stack_size;      /* Size of stack */
    __u64 tls;             /* Thread-local storage */
    __u64 set_tid;         /* Array of TIDs to use */
    __u64 set_tid_size;    /* Number of elements in set_tid */
    __u64 cgroup;          /* Cgroup to place child in */
};
```

### Clone Flag Categories

```
┌─────────────────────────────────────────────────────────────┐
│                   Clone Flag Categories                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  RESOURCE SHARING                                           │
│  ─────────────────                                          │
│  CLONE_VM         Share virtual memory (address space)      │
│  CLONE_FILES      Share file descriptor table               │
│  CLONE_FS         Share filesystem info (cwd, root, umask)  │
│  CLONE_SIGHAND    Share signal handlers                     │
│  CLONE_SYSVSEM    Share SysV semaphore undo values          │
│  CLONE_IO         Share I/O context                         │
│                                                             │
│  THREAD CREATION                                            │
│  ───────────────                                            │
│  CLONE_THREAD     Same thread group (share PID)             │
│  CLONE_SETTLS     Set thread-local storage                  │
│  CLONE_PARENT_SETTID  Write TID to parent location          │
│  CLONE_CHILD_SETTID   Write TID to child location           │
│  CLONE_CHILD_CLEARTID Clear TID location on exit            │
│                                                             │
│  NAMESPACE CREATION                                         │
│  ──────────────────                                         │
│  CLONE_NEWNS      New mount namespace                       │
│  CLONE_NEWPID     New PID namespace                         │
│  CLONE_NEWNET     New network namespace                     │
│  CLONE_NEWUSER    New user namespace                        │
│  CLONE_NEWIPC     New IPC namespace                         │
│  CLONE_NEWUTS     New UTS namespace                         │
│  CLONE_NEWCGROUP  New cgroup namespace                      │
│  CLONE_NEWTIME    New time namespace                        │
│                                                             │
│  SPECIAL BEHAVIOR                                           │
│  ────────────────                                           │
│  CLONE_VFORK      Parent waits until child execs/exits      │
│  CLONE_PARENT     Child has same parent as caller           │
│  CLONE_PIDFD      Return pidfd for child                    │
│  CLONE_PTRACE     Child is ptrace'd if parent is            │
│  CLONE_UNTRACED   Child cannot be ptrace'd                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common Clone Patterns

```c
/* fork() equivalent */
clone(SIGCHLD, 0);

/* vfork() equivalent */
clone(CLONE_VFORK | CLONE_VM | SIGCHLD, 0);

/* pthread_create() equivalent (glibc) */
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS |
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID,
      stack, &parent_tid, &child_tid, tls);

/* Container creation (unshare all namespaces) */
clone(CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET |
      CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWUSER | SIGCHLD, 0);
```

### Thread vs Process: Flag Effects

```
┌─────────────────────────────────────────────────────────────┐
│           Thread vs Process: What Gets Shared               │
├───────────────────┬─────────────────┬───────────────────────┤
│     Resource      │    Process      │       Thread          │
│                   │   (fork)        │   (pthread_create)    │
├───────────────────┼─────────────────┼───────────────────────┤
│ Address space     │     Copy        │       Shared          │
│ File descriptors  │     Copy        │       Shared          │
│ Signal handlers   │     Copy        │       Shared          │
│ Filesystem info   │     Copy        │       Shared          │
│ PID (getpid)      │     New         │       Same            │
│ TID (gettid)      │     New         │       New             │
│ Credentials       │     Copy        │       Shared          │
│ Stack             │     Copy (COW)  │       New (allocated) │
│ Signal mask       │     Copy        │       Copy            │
│ Pending signals   │     Clear       │       Clear           │
└───────────────────┴─────────────────┴───────────────────────┘
```

### TID Management

```c
/* How CLONE_CHILD_CLEARTID works for thread exit synchronization */

/* pthread_create() sets up: */
clone(...| CLONE_CHILD_CLEARTID, ..., &thread->tid);
/*         Thread's tid field points to futex location         */

/* When thread exits: */
void do_exit(long code) {
    /* ... other cleanup ... */

    /* If CLONE_CHILD_CLEARTID was set: */
    if (tsk->clear_child_tid) {
        /* Clear the TID location */
        put_user(0, tsk->clear_child_tid);
        /* Wake any waiters (pthread_join) */
        futex_wake(tsk->clear_child_tid, 1, FUTEX_PRIVATE_FLAG);
    }
}

/* pthread_join() waits via futex: */
while (thread->tid != 0) {
    futex_wait(&thread->tid, tid_value, ...);
}
```

---

## Memory Mapping (mmap)

mmap() maps files or devices into memory, or creates anonymous memory regions.
It's fundamental to how programs are loaded and how they allocate memory.

### mmap System Call

```c
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);

/* Protection flags */
#define PROT_NONE   0x0   /* No access */
#define PROT_READ   0x1   /* Readable */
#define PROT_WRITE  0x2   /* Writable */
#define PROT_EXEC   0x4   /* Executable */

/* Map type flags */
#define MAP_SHARED      0x01  /* Share changes with file/others */
#define MAP_PRIVATE     0x02  /* Copy-on-write private mapping */
#define MAP_FIXED       0x10  /* Place at exact address */
#define MAP_ANONYMOUS   0x20  /* No file backing (fd ignored) */
#define MAP_GROWSDOWN   0x0100 /* Stack-like (grows downward) */
#define MAP_DENYWRITE   0x0800 /* Deprecated */
#define MAP_LOCKED      0x2000 /* Lock pages in memory */
#define MAP_POPULATE    0x8000 /* Prefault pages */
#define MAP_HUGETLB     0x40000 /* Use huge pages */
```

### Virtual Memory Areas (VMAs)

```c
/* include/linux/mm_types.h */
struct vm_area_struct {
    /* VMA range in virtual address space */
    unsigned long vm_start;
    unsigned long vm_end;

    /* Linked list of VMAs for this mm */
    struct vm_area_struct *vm_next, *vm_prev;

    /* Red-black tree for fast lookup */
    struct rb_node vm_rb;

    struct mm_struct *vm_mm;      /* Owning mm_struct */

    pgprot_t vm_page_prot;        /* Page protection bits */
    unsigned long vm_flags;        /* VMA flags */

    /* For file-backed mappings */
    struct file *vm_file;
    unsigned long vm_pgoff;        /* Offset in file (pages) */

    /* Operations for this VMA */
    const struct vm_operations_struct *vm_ops;

    void *vm_private_data;
};
```

### Process Address Space Layout

```
┌─────────────────────────────────────────────────────────────┐
│              Virtual Address Space (x86-64)                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  0xFFFFFFFFFFFFFFFF ┌────────────────────────────────┐      │
│                     │        Kernel Space            │      │
│  0xFFFF800000000000 ├────────────────────────────────┤      │
│                     │    (non-canonical hole)        │      │
│  0x00007FFFFFFFFFFF ├────────────────────────────────┤      │
│                     │                                │      │
│                     │          Stack                 │      │
│                     │            ↓                   │ ←RLIMIT_STACK
│                     │         (grows down)           │      │
│                     ├────────────────────────────────┤      │
│                     │                                │      │
│                     │     Memory mappings            │      │
│                     │  (mmap, shared libs, etc.)     │      │
│                     │                                │      │
│                     ├────────────────────────────────┤      │
│                     │            ↑                   │      │
│                     │          Heap                  │      │
│                     │       (grows up via brk)       │      │
│                     ├────────────────────────────────┤      │
│                     │          .bss                  │ Zero-init
│                     ├────────────────────────────────┤      │
│                     │         .data                  │ Initialized
│                     ├────────────────────────────────┤      │
│                     │        .rodata                 │ Read-only
│                     ├────────────────────────────────┤      │
│                     │         .text                  │ Code
│  0x0000000000400000 ├────────────────────────────────┤      │
│                     │   (unmapped, catches NULLs)    │      │
│  0x0000000000000000 └────────────────────────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### mmap Kernel Implementation

```
┌─────────────────────────────────────────────────────────────┐
│                     mmap() Flow                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sys_mmap() / sys_mmap2()                                   │
│      │                                                      │
│      ▼                                                      │
│  ksys_mmap_pgoff()                                          │
│      │                                                      │
│      ▼                                                      │
│  vm_mmap_pgoff()                                            │
│      │                                                      │
│      ├── Get file reference (if file-backed)                │
│      │                                                      │
│      ▼                                                      │
│  do_mmap()                                                  │
│      │                                                      │
│      ├── get_unmapped_area() - find free address range      │
│      │       │                                              │
│      │       ├── Check vm_flags against file                │
│      │       └── Use arch_get_unmapped_area()               │
│      │                                                      │
│      ├── mmap_region()                                      │
│      │       │                                              │
│      │       ├── Unmap any existing overlapping VMAs        │
│      │       ├── Allocate new vm_area_struct                │
│      │       ├── Set up VMA fields                          │
│      │       ├── call_mmap() if file-backed                 │
│      │       │       └── Filesystem's mmap handler          │
│      │       └── Insert VMA into mm's VMA tree              │
│      │                                                      │
│      └── Return mapped address                              │
│                                                             │
│  Note: Pages are NOT allocated yet (demand paging)          │
│        Page fault handler allocates pages on first access   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Page Fault Handling

```
┌─────────────────────────────────────────────────────────────┐
│                  Page Fault Flow                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU accesses unmapped page                                 │
│      │                                                      │
│      ▼                                                      │
│  page_fault exception (#PF)                                 │
│      │                                                      │
│      ▼                                                      │
│  do_page_fault() / handle_page_fault()                      │
│      │                                                      │
│      ├── Find VMA for faulting address                      │
│      │   └── No VMA? → SIGSEGV                              │
│      │                                                      │
│      ├── Check permissions (read/write/exec)                │
│      │   └── Permission denied? → SIGSEGV                   │
│      │                                                      │
│      ▼                                                      │
│  handle_mm_fault()                                          │
│      │                                                      │
│      ├── Walk page tables, create if needed                 │
│      │                                                      │
│      ▼                                                      │
│  handle_pte_fault()                                         │
│      │                                                      │
│      ├── !pte_present:                                      │
│      │   ├── Anonymous? → do_anonymous_page()               │
│      │   │                 Allocate zero page               │
│      │   └── File-backed? → do_fault()                      │
│      │                       Read page from file            │
│      │                                                      │
│      └── pte_present but write fault:                       │
│          └── COW? → do_wp_page()                            │
│                     Copy page and mark writable             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common mmap Uses

```c
/* Anonymous mapping (memory allocation like malloc) */
void *mem = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

/* File mapping (read file contents) */
int fd = open("file.txt", O_RDONLY);
void *data = mmap(NULL, file_size, PROT_READ,
                  MAP_PRIVATE, fd, 0);

/* Shared memory (IPC between processes) */
int shm_fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(shm_fd, 4096);
void *shared = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                    MAP_SHARED, shm_fd, 0);

/* Executable mapping (loading code) */
void *code = mmap(NULL, code_size, PROT_READ | PROT_EXEC,
                  MAP_PRIVATE, elf_fd, code_offset);
```

---

## Program Execution (exec)

The exec family of system calls replaces the current process image with a new
program.

### execve System Call

```c
/* System call: execute a new program */
int execve(const char *filename,
           char *const argv[],
           char *const envp[]);

/* Kernel implementation entry point */
SYSCALL_DEFINE3(execve, const char __user *, filename,
                const char __user *const __user *, argv,
                const char __user *const __user *, envp)
{
    return do_execve(getname(filename), argv, envp);
}
```

### Exec Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      execve() Flow                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. do_execve() / do_execveat()                             │
│     │                                                       │
│     ▼                                                       │
│  2. bprm_execve()                                           │
│     - Allocate linux_binprm structure                       │
│     │                                                       │
│     ▼                                                       │
│  3. prepare_binprm()                                        │
│     - Open executable file                                  │
│     - Read first 128 bytes (for format detection)           │
│     - Check permissions                                     │
│     │                                                       │
│     ▼                                                       │
│  4. search_binary_handler()                                 │
│     - Try each registered format handler:                   │
│       - ELF (load_elf_binary)                               │
│       - Script (#! shebang)                                 │
│       - Misc (binfmt_misc)                                  │
│     │                                                       │
│     ▼                                                       │
│  5. load_elf_binary() [for ELF]                             │
│     a. Parse ELF header                                     │
│     b. Load program interpreter (ld-linux.so) if needed     │
│     c. flush_old_exec() - destroy old address space         │
│     d. setup_new_exec() - set up new credentials            │
│     e. Map ELF segments into memory                         │
│     f. Set up stack with argv, envp, auxv                   │
│     g. start_thread() - set entry point                     │
│     │                                                       │
│     ▼                                                       │
│  6. Return to user space                                    │
│     - Execution begins at entry point                       │
│     - Dynamic linker runs first (if dynamic executable)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### linux_binprm Structure

```c
/* include/linux/binfmts.h */
struct linux_binprm {
    char buf[BINPRM_BUF_SIZE];      /* First 256 bytes of file */

    struct vm_area_struct *vma;      /* For stack */
    unsigned long vma_pages;

    struct mm_struct *mm;

    unsigned long p;                 /* Top of stack */
    unsigned long argmin;            /* Limit for arguments */

    unsigned int recursion_depth;    /* For script interpreters */
    struct file *file;               /* Executable file */
    struct cred *cred;               /* New credentials */

    int argc, envc;                  /* Argument and env counts */

    const char *filename;            /* Filename to execute */
    const char *interp;              /* Interpreter name */

    unsigned long loader, exec;
    unsigned long min_brk;           /* Start of heap */
    unsigned long brk;               /* Current heap end */

    /* ... */
};
```

### Binary Formats

```c
/* include/linux/binfmts.h */
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *);
    unsigned long min_coredump;
};

/* Common formats */
static struct linux_binfmt elf_format = {
    .module         = THIS_MODULE,
    .load_binary    = load_elf_binary,
    .load_shlib     = load_elf_library,
    .core_dump      = elf_core_dump,
    .min_coredump   = ELF_EXEC_PAGESIZE,
};

/* Script format (handles #! interpreter) */
static struct linux_binfmt script_format = {
    .module         = THIS_MODULE,
    .load_binary    = load_script,
};
```

---

## Dynamic Linking

Dynamic linking defers symbol resolution until runtime, allowing shared
libraries to be loaded and code to be shared across processes.

### The Dynamic Linker (ld-linux.so)

```
┌─────────────────────────────────────────────────────────────┐
│                Dynamic Linker Overview                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  What is ld-linux.so?                                       │
│  ─────────────────────                                      │
│  - The runtime dynamic linker/loader                        │
│  - Typically: /lib64/ld-linux-x86-64.so.2                   │
│  - Specified in ELF's PT_INTERP segment                     │
│  - First code to run for dynamically linked executables     │
│                                                             │
│  Responsibilities:                                          │
│  ─────────────────                                          │
│  1. Load shared libraries (DT_NEEDED entries)               │
│  2. Relocate code and data (fix addresses)                  │
│  3. Resolve symbols (connect function calls)                │
│  4. Initialize libraries (_init, constructors)              │
│  5. Transfer control to program's entry point               │
│                                                             │
│  Key environment variables:                                 │
│  ──────────────────────────                                 │
│  LD_LIBRARY_PATH    Additional library search paths         │
│  LD_PRELOAD         Libraries to load before all others     │
│  LD_BIND_NOW        Resolve all symbols at load time        │
│  LD_DEBUG           Debug output (libs, bindings, etc.)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### ELF Dynamic Section

```c
/* ELF dynamic section entries */
typedef struct {
    Elf64_Sxword d_tag;    /* Entry type */
    union {
        Elf64_Xword d_val; /* Integer value */
        Elf64_Addr  d_ptr; /* Address value */
    } d_un;
} Elf64_Dyn;

/* Common dynamic tags */
#define DT_NULL     0   /* End of dynamic section */
#define DT_NEEDED   1   /* Name of needed library */
#define DT_PLTRELSZ 2   /* Size of PLT relocs */
#define DT_PLTGOT   3   /* Address of PLT/GOT */
#define DT_HASH     4   /* Symbol hash table */
#define DT_STRTAB   5   /* String table */
#define DT_SYMTAB   6   /* Symbol table */
#define DT_RELA     7   /* Relocation table (with addend) */
#define DT_INIT     12  /* Initialization function */
#define DT_FINI     13  /* Termination function */
#define DT_SONAME   14  /* Shared object name */
#define DT_RPATH    15  /* Library search path (deprecated) */
#define DT_RUNPATH  29  /* Library search path */
#define DT_FLAGS    30  /* Flags */
#define DT_JMPREL   23  /* PLT relocations */

/* Example: reading needed libraries */
/* $ readelf -d /bin/ls | grep NEEDED
   0x0000000000000001 (NEEDED)  Shared library: [libc.so.6]
   0x0000000000000001 (NEEDED)  Shared library: [libselinux.so.1]
*/
```

### GOT (Global Offset Table)

```
┌─────────────────────────────────────────────────────────────┐
│                Global Offset Table (GOT)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Purpose: Indirection table for position-independent code   │
│                                                             │
│  Why needed?                                                │
│  ───────────                                                │
│  - Shared libraries loaded at varying addresses (ASLR)      │
│  - Code can't have hardcoded addresses                      │
│  - GOT provides fixed offsets from code to data/functions   │
│                                                             │
│  Structure:                                                 │
│  ──────────                                                 │
│  ┌─────────────────┐                                        │
│  │ GOT[0]          │ → Address of _DYNAMIC section          │
│  ├─────────────────┤                                        │
│  │ GOT[1]          │ → Pointer to link_map (for resolver)   │
│  ├─────────────────┤                                        │
│  │ GOT[2]          │ → Address of _dl_runtime_resolve       │
│  ├─────────────────┤                                        │
│  │ GOT[3]          │ → Resolved address of function 1       │
│  ├─────────────────┤                                        │
│  │ GOT[4]          │ → Resolved address of function 2       │
│  ├─────────────────┤                                        │
│  │ ...             │                                        │
│  └─────────────────┘                                        │
│                                                             │
│  Two GOT sections:                                          │
│  ─────────────────                                          │
│  .got      - Global variables from shared libraries         │
│  .got.plt  - Function addresses (used with PLT)             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### PLT (Procedure Linkage Table)

```
┌─────────────────────────────────────────────────────────────┐
│              Procedure Linkage Table (PLT)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Purpose: Stub code for calling external functions          │
│                                                             │
│  Why needed?                                                │
│  ───────────                                                │
│  - Enables lazy binding (resolve on first call)             │
│  - Provides consistent calling convention                   │
│  - Allows LD_PRELOAD and symbol interposition               │
│                                                             │
│  Structure (x86-64):                                        │
│  ───────────────────                                        │
│                                                             │
│  PLT[0] (resolver stub):                                    │
│  ┌────────────────────────────────────────────┐             │
│  │ push   [GOT+8]        # Push link_map ptr  │             │
│  │ jmp    [GOT+16]       # Jump to resolver   │             │
│  └────────────────────────────────────────────┘             │
│                                                             │
│  PLT[n] (function stub, e.g., printf):                      │
│  ┌────────────────────────────────────────────┐             │
│  │ jmp    [GOT+offset]   # Jump to resolved   │             │
│  │ push   n              # Push reloc index   │             │
│  │ jmp    PLT[0]         # Go to resolver     │             │
│  └────────────────────────────────────────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Lazy Binding in Action

```
┌─────────────────────────────────────────────────────────────┐
│                  Lazy Binding Flow                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FIRST CALL to printf():                                    │
│  ───────────────────────                                    │
│                                                             │
│  1. Code calls printf@PLT                                   │
│         │                                                   │
│         ▼                                                   │
│  2. PLT stub: jmp [GOT entry]                               │
│     GOT entry initially points to next PLT instruction      │
│         │                                                   │
│         ▼                                                   │
│  3. PLT stub: push relocation_index                         │
│         │                                                   │
│         ▼                                                   │
│  4. PLT stub: jmp PLT[0] (resolver stub)                    │
│         │                                                   │
│         ▼                                                   │
│  5. _dl_runtime_resolve():                                  │
│     - Look up "printf" in libc.so symbol table              │
│     - Write resolved address to GOT entry                   │
│     - Jump to printf                                        │
│                                                             │
│  SUBSEQUENT CALLS to printf():                              │
│  ─────────────────────────────                              │
│                                                             │
│  1. Code calls printf@PLT                                   │
│         │                                                   │
│         ▼                                                   │
│  2. PLT stub: jmp [GOT entry]                               │
│     GOT now contains actual printf address                  │
│         │                                                   │
│         ▼                                                   │
│  3. Directly in printf() - no resolver overhead!            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Symbol Resolution

```c
/* Symbol table entry */
typedef struct {
    Elf64_Word    st_name;   /* Symbol name (string table index) */
    unsigned char st_info;   /* Type and binding */
    unsigned char st_other;  /* Visibility */
    Elf64_Half    st_shndx;  /* Section index */
    Elf64_Addr    st_value;  /* Symbol value (address) */
    Elf64_Xword   st_size;   /* Symbol size */
} Elf64_Sym;

/* Symbol binding (upper 4 bits of st_info) */
#define STB_LOCAL   0   /* Local symbol */
#define STB_GLOBAL  1   /* Global symbol */
#define STB_WEAK    2   /* Weak symbol (can be overridden) */

/* Symbol type (lower 4 bits of st_info) */
#define STT_NOTYPE  0   /* Unspecified type */
#define STT_OBJECT  1   /* Data object (variable) */
#define STT_FUNC    2   /* Function */
#define STT_SECTION 3   /* Section */
#define STT_FILE    4   /* Source file name */
#define STT_TLS     6   /* Thread-local storage */

/* Symbol visibility (st_other) */
#define STV_DEFAULT   0   /* Use binding type rules */
#define STV_INTERNAL  1   /* Not externally visible */
#define STV_HIDDEN    2   /* Not visible to other components */
#define STV_PROTECTED 3   /* Visible but not preemptible */
```

### Relocation Types

```c
/* Common x86-64 relocation types */
#define R_X86_64_NONE       0   /* No relocation */
#define R_X86_64_64         1   /* Direct 64-bit */
#define R_X86_64_PC32       2   /* PC-relative 32-bit */
#define R_X86_64_GOT32      3   /* 32-bit GOT entry */
#define R_X86_64_PLT32      4   /* 32-bit PLT address */
#define R_X86_64_COPY       5   /* Copy symbol at runtime */
#define R_X86_64_GLOB_DAT   6   /* Create GOT entry */
#define R_X86_64_JUMP_SLOT  7   /* Create PLT entry */
#define R_X86_64_RELATIVE   8   /* Adjust by base address */
#define R_X86_64_GOTPCREL   9   /* 32-bit PC-rel GOT offset */
#define R_X86_64_TPOFF32    23  /* TLS offset in initial exec */

/* Relocation table entry */
typedef struct {
    Elf64_Addr    r_offset;  /* Address to relocate */
    Elf64_Xword   r_info;    /* Relocation type and symbol */
    Elf64_Sxword  r_addend;  /* Addend */
} Elf64_Rela;

/* Extract type and symbol from r_info */
#define ELF64_R_TYPE(i)    ((i) & 0xffffffff)
#define ELF64_R_SYM(i)     ((i) >> 32)
```

### Dynamic Linking vs Static Linking

```
┌─────────────────────────────────────────────────────────────┐
│           Dynamic vs Static Linking Comparison              │
├───────────────────┬─────────────────┬───────────────────────┤
│     Aspect        │     Static      │       Dynamic         │
├───────────────────┼─────────────────┼───────────────────────┤
│ Binary size       │ Larger          │ Smaller               │
│ Memory usage      │ No sharing      │ Shared pages          │
│ Startup time      │ Faster          │ Slower (linking)      │
│ Updates           │ Recompile       │ Replace .so file      │
│ Dependencies      │ Self-contained  │ Requires libs         │
│ Security patches  │ Must rebuild    │ Update lib only       │
│ Symbol interpose  │ Not possible    │ LD_PRELOAD works      │
│ ASLR              │ Limited         │ Full support          │
│ Portability       │ Better          │ Needs matching libs   │
└───────────────────┴─────────────────┴───────────────────────┘
```

### Viewing Dynamic Linking Information

```bash
# Show dynamic section
$ readelf -d /bin/ls

# Show PLT relocations
$ objdump -R /bin/ls

# Show symbol table
$ nm -D /bin/ls

# Show needed libraries
$ ldd /bin/ls

# Trace dynamic linker operations
$ LD_DEBUG=libs /bin/ls
$ LD_DEBUG=bindings /bin/ls
$ LD_DEBUG=symbols /bin/ls

# Show GOT/PLT addresses
$ objdump -d -j .plt /bin/ls
$ readelf -x .got.plt /bin/ls
```

---

## Process Termination (exit)

Process termination happens through do_exit(), which can be triggered by exit()
syscall, fatal signal, or kernel error.

### Exit Path

```c
/* kernel/exit.c - simplified */
void __noreturn do_exit(long code)
{
    struct task_struct *tsk = current;

    /* Set exit code */
    tsk->exit_code = code;

    /* Exit mm (release address space) */
    exit_mm();

    /* Exit files and fs */
    exit_files(tsk);
    exit_fs(tsk);

    /* Exit signals */
    exit_signals(tsk);

    /* Exit namespaces */
    exit_task_namespaces(tsk);

    /* Exit thread group */
    exit_notify(tsk, group_dead);

    /* Change state to TASK_DEAD */
    tsk->__state = TASK_DEAD;

    /* Final schedule - never returns */
    do_task_dead();
}
```

### Exit Cleanup Sequence

```
┌─────────────────────────────────────────────────────────────┐
│                   Exit Cleanup Sequence                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. exit_signals()                                          │
│     - Dequeue pending signals                               │
│     - Notify traced children                                │
│                                                             │
│  2. exit_mm()                                               │
│     - Release user address space                            │
│     - If last user, free all VMAs                           │
│                                                             │
│  3. exit_sem()                                              │
│     - Undo SysV semaphore operations                        │
│                                                             │
│  4. exit_files()                                            │
│     - Close all file descriptors                            │
│     - Release file table reference                          │
│                                                             │
│  5. exit_fs()                                               │
│     - Release filesystem info                               │
│                                                             │
│  6. exit_task_work()                                        │
│     - Run queued task_work callbacks                        │
│                                                             │
│  7. cgroup_exit()                                           │
│     - Release cgroup references                             │
│                                                             │
│  8. exit_notify()                                           │
│     - Reparent children to init                             │
│     - Send SIGCHLD to parent                                │
│     - Become zombie (EXIT_ZOMBIE)                           │
│                                                             │
│  9. do_task_dead()                                          │
│     - TASK_DEAD state                                       │
│     - Final context switch                                  │
│     - Task freed by other CPU                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Zombie State

```
┌─────────────────────────────────────────────────────────────┐
│                    Zombie Processes                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  A zombie is a process that has exited but hasn't been      │
│  reaped by its parent via wait().                           │
│                                                             │
│  What remains:                                              │
│  - task_struct (minimal)                                    │
│  - Exit status                                              │
│  - Resource usage statistics                                │
│                                                             │
│  What's released:                                           │
│  - Address space (mm_struct)                                │
│  - File descriptors                                         │
│  - Kernel stack                                             │
│                                                             │
│  Parent must wait() to:                                     │
│  - Retrieve exit status                                     │
│  - Free remaining task_struct                               │
│  - Remove from process table                                │
│                                                             │
│  Orphan handling:                                           │
│  - If parent exits first, children reparented to init       │
│  - init reaps orphaned zombies                              │
│                                                             │
│  Viewing zombies:                                           │
│  $ ps aux | grep Z                                          │
│  USER  PID %CPU ... STAT ... COMMAND                        │
│  user  123  0.0 ... Z    ... [defunct]                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Wait and Reaping

The wait() family of system calls allows a parent to collect exit status from
children.

### Wait System Calls

```c
/* Wait for any child */
pid_t wait(int *wstatus);

/* Wait for specific child */
pid_t waitpid(pid_t pid, int *wstatus, int options);

/* Extended wait with rusage */
pid_t wait4(pid_t pid, int *wstatus, int options,
            struct rusage *rusage);

/* Wait with more control (modern) */
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);

/* Options */
#define WNOHANG     0x00000001  /* Don't block */
#define WUNTRACED   0x00000002  /* Report stopped children */
#define WSTOPPED    WUNTRACED
#define WEXITED     0x00000004  /* Report exited (waitid) */
#define WCONTINUED  0x00000008  /* Report continued */
#define WNOWAIT     0x01000000  /* Don't reap, just peek */
#define __WALL      0x40000000  /* Wait for any child */
#define __WCLONE    0x80000000  /* Wait for clone children */

/* Status macros */
WIFEXITED(status)     /* True if normal exit */
WEXITSTATUS(status)   /* Exit code (if WIFEXITED) */
WIFSIGNALED(status)   /* True if killed by signal */
WTERMSIG(status)      /* Signal number (if WIFSIGNALED) */
WCOREDUMP(status)     /* True if core dumped */
WIFSTOPPED(status)    /* True if stopped */
WSTOPSIG(status)      /* Stop signal (if WIFSTOPPED) */
WIFCONTINUED(status)  /* True if continued */
```

### Wait Kernel Implementation

```
┌─────────────────────────────────────────────────────────────┐
│                     Wait Implementation                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  do_wait()                                                  │
│      │                                                      │
│      ▼                                                      │
│  child_wait_callback()                                      │
│      │                                                      │
│      ├── Check each child for match                         │
│      │                                                      │
│      ├── If zombie found:                                   │
│      │   └── wait_task_zombie()                             │
│      │       ├── Collect exit status                        │
│      │       ├── Accumulate rusage                          │
│      │       └── release_task() → free task_struct          │
│      │                                                      │
│      ├── If stopped/continued and flags match:              │
│      │   └── wait_task_stopped/continued()                  │
│      │       └── Collect status without reaping             │
│      │                                                      │
│      ├── If WNOHANG and no child ready:                     │
│      │   └── Return 0                                       │
│      │                                                      │
│      └── Otherwise:                                         │
│          └── schedule() until child state changes           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Signals

Signals provide asynchronous notifications to processes.

### Signal Numbers

```c
/* arch/x86/include/uapi/asm/signal.h */
#define SIGHUP       1    /* Hangup */
#define SIGINT       2    /* Interrupt (Ctrl+C) */
#define SIGQUIT      3    /* Quit (Ctrl+\) */
#define SIGILL       4    /* Illegal instruction */
#define SIGTRAP      5    /* Trace trap */
#define SIGABRT      6    /* Abort */
#define SIGBUS       7    /* Bus error */
#define SIGFPE       8    /* Floating point exception */
#define SIGKILL      9    /* Kill (cannot be caught) */
#define SIGUSR1     10    /* User-defined 1 */
#define SIGSEGV     11    /* Segmentation violation */
#define SIGUSR2     12    /* User-defined 2 */
#define SIGPIPE     13    /* Broken pipe */
#define SIGALRM     14    /* Alarm clock */
#define SIGTERM     15    /* Termination */
#define SIGSTKFLT   16    /* Stack fault */
#define SIGCHLD     17    /* Child status changed */
#define SIGCONT     18    /* Continue if stopped */
#define SIGSTOP     19    /* Stop (cannot be caught) */
#define SIGTSTP     20    /* Terminal stop (Ctrl+Z) */
#define SIGTTIN     21    /* Background read from tty */
#define SIGTTOU     22    /* Background write to tty */
#define SIGURG      23    /* Urgent data on socket */
#define SIGXCPU     24    /* CPU limit exceeded */
#define SIGXFSZ     25    /* File size limit exceeded */
#define SIGVTALRM   26    /* Virtual alarm clock */
#define SIGPROF     27    /* Profiling alarm clock */
#define SIGWINCH    28    /* Window size change */
#define SIGIO       29    /* I/O possible */
#define SIGPWR      30    /* Power failure */
#define SIGSYS      31    /* Bad system call */
/* Real-time signals: SIGRTMIN to SIGRTMAX (typically 32-64) */
```

### Signal Data Structures

```c
/* include/linux/sched/signal.h */
struct signal_struct {
    refcount_t              sigcnt;
    atomic_t                live;           /* Num live threads */
    int                     nr_threads;
    struct list_head        thread_head;

    wait_queue_head_t       wait_chldexit;

    struct task_struct      *curr_target;

    struct sigpending       shared_pending; /* Thread-group signals */

    struct hlist_head       multiprocess;

    int                     group_exit_code;
    int                     notify_count;
    struct task_struct      *group_exec_task;

    int                     group_stop_count;
    unsigned int            flags;

    /* POSIX.1b signal handling */
    struct posix_cputimers  posix_cputimers;

    /* Resource limits (rlimits) */
    struct rlimit           rlim[RLIM_NLIMITS];

    /* ... */
};

/* Pending signals for a task */
struct sigpending {
    struct list_head        list;           /* List of sigqueue */
    sigset_t                signal;         /* Pending mask */
};

/* Signal queue entry */
struct sigqueue {
    struct list_head        list;
    int                     flags;
    kernel_siginfo_t        info;
    struct ucounts          *ucounts;
};
```

### Signal Delivery

```
┌─────────────────────────────────────────────────────────────┐
│                   Signal Delivery Flow                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Signal Generation                                       │
│     kill(), tgkill(), raise(), kernel event                 │
│         │                                                   │
│         ▼                                                   │
│  2. send_signal()                                           │
│     - Add to task->pending or signal->shared_pending        │
│     - Set bit in pending mask                               │
│         │                                                   │
│         ▼                                                   │
│  3. signal_wake_up()                                        │
│     - Set TIF_SIGPENDING                                    │
│     - Wake task if sleeping                                 │
│         │                                                   │
│         ▼                                                   │
│  4. At return to userspace:                                 │
│     exit_to_user_mode_loop() checks TIF_SIGPENDING          │
│         │                                                   │
│         ▼                                                   │
│  5. do_signal()                                             │
│     - Dequeue signal                                        │
│     - Look up handler                                       │
│         │                                                   │
│         ├── SIG_IGN: discard                                │
│         ├── SIG_DFL: default action (may terminate)         │
│         └── handler: setup_signal_frame()                   │
│                     - Save registers on user stack          │
│                     - Set up fake return to handler         │
│                                                             │
│  6. Return to userspace at signal handler                   │
│         │                                                   │
│         ▼                                                   │
│  7. Handler runs in userspace                               │
│         │                                                   │
│         ▼                                                   │
│  8. sigreturn() system call                                 │
│     - Restore saved context                                 │
│     - Resume original execution                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Signal Handler Setup

```c
/* User-space signal handler setup */
struct sigaction {
    void (*sa_handler)(int);
    void (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_restorer)(void);
};

/* Flags */
#define SA_NOCLDSTOP  0x00000001  /* Don't SIGCHLD on stop */
#define SA_NOCLDWAIT  0x00000002  /* Don't create zombies */
#define SA_SIGINFO    0x00000004  /* Use sa_sigaction */
#define SA_ONSTACK    0x08000000  /* Use alternate stack */
#define SA_RESTART    0x10000000  /* Restart syscalls */
#define SA_NODEFER    0x40000000  /* Don't block signal */
#define SA_RESETHAND  0x80000000  /* Reset to SIG_DFL */
```

---

## Credentials and Capabilities

Linux uses credentials to determine what operations a process can perform.

### Credential Structure

```c
/* include/linux/cred.h */
struct cred {
    atomic_t        usage;

    kuid_t          uid;            /* Real user ID */
    kgid_t          gid;            /* Real group ID */
    kuid_t          suid;           /* Saved user ID */
    kgid_t          sgid;           /* Saved group ID */
    kuid_t          euid;           /* Effective user ID */
    kgid_t          egid;           /* Effective group ID */
    kuid_t          fsuid;          /* Filesystem user ID */
    kgid_t          fsgid;          /* Filesystem group ID */

    unsigned        securebits;      /* SECBIT_* flags */
    kernel_cap_t    cap_inheritable; /* Caps preserved across exec */
    kernel_cap_t    cap_permitted;   /* Caps we can use */
    kernel_cap_t    cap_effective;   /* Caps in effect now */
    kernel_cap_t    cap_bset;        /* Capability bounding set */
    kernel_cap_t    cap_ambient;     /* Ambient capabilities */

    struct group_info *group_info;   /* Supplementary groups */

    struct rcu_head rcu;

    /* Keyrings */
    struct key      *session_keyring;
    struct key      *process_keyring;
    struct key      *thread_keyring;
    struct key      *request_key_auth;

    /* LSM security */
    void            *security;

    /* User namespace */
    struct user_namespace *user_ns;

    /* ... */
};
```

### Capabilities

```c
/* include/uapi/linux/capability.h */

/* Common capabilities */
#define CAP_CHOWN            0   /* Change file ownership */
#define CAP_DAC_OVERRIDE     1   /* Bypass file permissions */
#define CAP_DAC_READ_SEARCH  2   /* Bypass read/search permissions */
#define CAP_FOWNER           3   /* Bypass permission for file owner */
#define CAP_FSETID           4   /* Don't clear setuid/setgid */
#define CAP_KILL             5   /* Kill any process */
#define CAP_SETGID           6   /* Set GID */
#define CAP_SETUID           7   /* Set UID */
#define CAP_SETPCAP          8   /* Transfer capabilities */
#define CAP_LINUX_IMMUTABLE  9   /* Modify immutable files */
#define CAP_NET_BIND_SERVICE 10  /* Bind to ports < 1024 */
#define CAP_NET_BROADCAST    11  /* Network broadcast */
#define CAP_NET_ADMIN        12  /* Network configuration */
#define CAP_NET_RAW          13  /* Raw sockets */
#define CAP_IPC_LOCK         14  /* Lock memory */
#define CAP_IPC_OWNER        15  /* IPC permissions override */
#define CAP_SYS_MODULE       16  /* Load kernel modules */
#define CAP_SYS_RAWIO        17  /* Raw I/O (iopl, ioperm) */
#define CAP_SYS_CHROOT       18  /* chroot() */
#define CAP_SYS_PTRACE       19  /* ptrace any process */
#define CAP_SYS_PACCT        20  /* Process accounting */
#define CAP_SYS_ADMIN        21  /* Catch-all admin */
#define CAP_SYS_BOOT         22  /* Reboot */
#define CAP_SYS_NICE         23  /* Set priority/scheduling */
#define CAP_SYS_RESOURCE     24  /* Override resource limits */
#define CAP_SYS_TIME         25  /* Set system time */
#define CAP_SYS_TTY_CONFIG   26  /* TTY configuration */
#define CAP_MKNOD            27  /* mknod() */
#define CAP_LEASE            28  /* File leases */
#define CAP_AUDIT_WRITE      29  /* Write to audit log */
#define CAP_AUDIT_CONTROL    30  /* Configure audit */
#define CAP_SETFCAP          31  /* Set file capabilities */
/* ... more capabilities ... */
```

### Capability Checking

```c
/* Check if current has capability */
bool capable(int cap);
bool ns_capable(struct user_namespace *ns, int cap);
bool file_ns_capable(const struct file *file,
                     struct user_namespace *ns, int cap);

/* Example usage in kernel */
if (!capable(CAP_SYS_ADMIN)) {
    return -EPERM;
}
```

---

## Namespaces

Namespaces provide isolation of system resources.

### Namespace Types

```c
/* include/linux/nsproxy.h */
struct nsproxy {
    atomic_t count;
    struct uts_namespace    *uts_ns;        /* Hostname/domain */
    struct ipc_namespace    *ipc_ns;        /* IPC resources */
    struct mnt_namespace    *mnt_ns;        /* Mount points */
    struct pid_namespace    *pid_ns_for_children;  /* PID space */
    struct net              *net_ns;        /* Network stack */
    struct time_namespace   *time_ns;       /* Time offsets */
    struct time_namespace   *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns;     /* Cgroup root */
};

/* Clone flags for new namespaces */
#define CLONE_NEWNS      0x00020000  /* New mount namespace */
#define CLONE_NEWUTS     0x04000000  /* New UTS namespace */
#define CLONE_NEWIPC     0x08000000  /* New IPC namespace */
#define CLONE_NEWUSER    0x10000000  /* New user namespace */
#define CLONE_NEWPID     0x20000000  /* New PID namespace */
#define CLONE_NEWNET     0x40000000  /* New network namespace */
#define CLONE_NEWCGROUP  0x02000000  /* New cgroup namespace */
#define CLONE_NEWTIME    0x00000080  /* New time namespace */
```

### Namespace Visualization

```
┌─────────────────────────────────────────────────────────────┐
│                  Namespace Hierarchy                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Host (Init Namespace)               │   │
│  │  PID 1 (systemd)                                     │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │              Container A                        │ │   │
│  │  │  ┌───────────────────────────────────────────┐  │ │   │
│  │  │  │ PID namespace: PID 1 = container's init   │  │ │   │
│  │  │  │ MNT namespace: own root filesystem        │  │ │   │
│  │  │  │ NET namespace: own network stack          │  │ │   │
│  │  │  │ UTS namespace: own hostname               │  │ │   │
│  │  │  │ IPC namespace: own IPC resources          │  │ │   │
│  │  │  │ USER namespace: own UID mapping           │  │ │   │
│  │  │  └───────────────────────────────────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  │                                                      │   │
│  │  ┌─────────────────────────────────────────────────┐ │   │
│  │  │              Container B                        │ │   │
│  │  │  (Similar isolated namespaces)                  │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  From host perspective:                                     │
│  Container A's PID 1 = Host PID 12345                       │
│  Container B's PID 1 = Host PID 12400                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Control Groups (cgroups)

Cgroups provide resource limiting, prioritization, accounting, and control.

### Cgroup Structure (v2)

```c
/* include/linux/cgroup.h */
struct cgroup {
    struct cgroup_subsys_state self;

    unsigned long flags;

    int id;
    int level;
    int max_depth;
    int nr_descendants;
    int nr_dying_descendants;

    struct cgroup *parent;
    struct kernfs_node *kn;             /* sysfs representation */

    struct list_head cset_links;

    /* Controllers */
    u16 subtree_control;
    u16 subtree_ss_mask;
    u16 old_subtree_control;
    u16 old_subtree_ss_mask;

    struct cgroup_rstat_cpu __percpu *rstat_cpu;

    /* ... */
};

/* Per-subsystem state */
struct cgroup_subsys_state {
    struct cgroup *cgroup;
    struct cgroup_subsys *ss;

    struct percpu_ref refcnt;
    struct list_head sibling;
    struct list_head children;

    int id;
    unsigned int flags;

    struct cgroup_subsys_state *parent;
};
```

### Cgroup Controllers

```
┌─────────────────────────────────────────────────────────────┐
│                  Cgroup Controllers                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  cpu                                                        │
│  ───                                                        │
│  - cpu.max: Bandwidth limit (quota period)                  │
│  - cpu.weight: Relative CPU shares                          │
│                                                             │
│  memory                                                     │
│  ──────                                                     │
│  - memory.max: Hard memory limit                            │
│  - memory.high: Soft limit (throttling)                     │
│  - memory.current: Current usage                            │
│                                                             │
│  io                                                         │
│  ──                                                         │
│  - io.max: Bandwidth/IOPS limits                            │
│  - io.weight: I/O priority                                  │
│                                                             │
│  pids                                                       │
│  ────                                                       │
│  - pids.max: Maximum number of processes                    │
│  - pids.current: Current count                              │
│                                                             │
│  cpuset                                                     │
│  ──────                                                     │
│  - cpuset.cpus: Allowed CPUs                                │
│  - cpuset.mems: Allowed memory nodes                        │
│                                                             │
│  freezer                                                    │
│  ───────                                                    │
│  - cgroup.freeze: Freeze/thaw processes                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cgroup Filesystem

```bash
# Cgroup v2 (unified hierarchy)
mount -t cgroup2 none /sys/fs/cgroup

# Structure
/sys/fs/cgroup/
├── cgroup.controllers      # Available controllers
├── cgroup.subtree_control  # Enabled for children
├── cgroup.procs           # PIDs in this cgroup
├── cpu.max                 # CPU limit
├── memory.max              # Memory limit
├── mygroup/                # Child cgroup
│   ├── cgroup.procs
│   ├── cpu.max
│   └── ...
└── ...
```

---

## copy_process() Internals

### `struct kernel_clone_args` (`include/linux/sched/task.h:23–47`)

```c
struct kernel_clone_args {
    u64 flags;              /* CLONE_* flags bitmask */
    int __user *pidfd;      /* where to store pidfd (CLONE_PIDFD) */
    int __user *child_tid;  /* for CLONE_CHILD_SETTID / CLONE_CHILD_CLEARTID */
    int __user *parent_tid; /* for CLONE_PARENT_SETTID */
    const char *name;       /* task name (kthreads) */
    int exit_signal;        /* signal to send parent on exit */
    u32 kthread:1;          /* is this a kthread? */
    u32 io_thread:1;        /* is this an IO thread? */
    u32 user_worker:1;      /* is this a user worker? */
    u32 no_files:1;         /* skip file descriptor copy */
    unsigned long stack;    /* stack pointer for new task */
    unsigned long stack_size;
    unsigned long tls;      /* TLS value (CLONE_SETTLS) */
    pid_t *set_tid;         /* explicit TID list for pid namespaces */
    size_t set_tid_size;    /* number of elements in set_tid */
    int cgroup;             /* cgroup fd (clone3) */
    int idle;               /* idle task flag */
    int (*fn)(void *);      /* kthread function */
    void *fn_arg;
    struct cgroup *cgrp;
    struct css_set *cset;
    unsigned int kill_seq;
};
```

### `copy_process()` Signature (`kernel/fork.c:2146–2150`)

```c
__latent_entropy struct task_struct *copy_process(
    struct pid *pid,
    int trace,
    int node,
    struct kernel_clone_args *args)
```

### Clone Flag Validation (early in `copy_process`, lines 2163–2211)

Before any allocation, `copy_process()` enforces these invariants:

| Condition | Error | Reason |
|---|---|---|
| `CLONE_NEWNS \| CLONE_FS` together | `-EINVAL` | Cannot share FS root across mount namespaces |
| `CLONE_NEWUSER \| CLONE_FS` together | `-EINVAL` | User namespace cannot share FS |
| `CLONE_THREAD` without `CLONE_SIGHAND` | `-EINVAL` | Threads must share signal handlers |
| `CLONE_SIGHAND` without `CLONE_VM` | `-EINVAL` | Shared sighand implies shared VM |
| `CLONE_PARENT` on global/container init | `-EINVAL` | Would create multi-rooted trees |
| `CLONE_THREAD` with `CLONE_NEWUSER`/`CLONE_NEWPID` | `-EINVAL` | Thread cannot be in different PID/user ns |
| `CLONE_PIDFD \| CLONE_DETACHED` | `-EINVAL` | Conflicting flags |

### Detailed Flow (`kernel/fork.c:2219–2637`)

**Phase 1: Signal handling setup (lines 2219–2229)**
```
spin_lock_irq(&current->sighand->siglock)
  if !CLONE_THREAD: hlist_add_head(&delayed.node, &signal->multiprocess)
  recalc_sigpending()
spin_unlock_irq(...)
→ if task_sigpending(current): goto fork_out (-ERESTARTNOINTR)
```

**Phase 2: Allocate task struct (lines 2232–2250)**
```
dup_task_struct(current, node)
  alloc_task_struct_node(node)         ← slab alloc from task_struct cache
  arch_dup_task_struct(tsk, orig)      ← struct copy (*dst = *src)
  alloc_thread_stack_node(tsk, node)   ← allocate kernel stack
  setup_thread_stack(tsk, orig)        ← copy thread_info
  set_task_stack_end_magic(tsk)        ← STACK_END_MAGIC canary
  refcount_set(&tsk->rcu_users, 2)     ← 1 for userspace, 1 for scheduler
  refcount_set(&tsk->usage, 1)
```

**Phase 3: Credentials and limits (lines 2266–2285)**
```
copy_creds(p, clone_flags)             ← fork credential COW
is_rlimit_overlimit(...NPROC...)       ← RLIMIT_NPROC check (-EAGAIN)
nr_threads >= max_threads check        ← system-wide thread limit
```

**Phase 4: Scheduler and cgroup init (lines 2287–2383)**
```
delayacct_tsk_init(p)
cgroup_fork(p)                         ← pin CSS set
sched_fork(clone_flags, p)             ← assign CPU, set state=TASK_NEW
perf_event_init_task(p, clone_flags)
audit_alloc(p)
shm_init_task(p)
```

**Phase 5: Security hook (line 2384–2386)**
```
security_task_alloc(p, clone_flags)    ← LSM hook; goto bad_fork_cleanup_audit on error
```

**Phase 6: Resource copying (lines 2387–2413) — the canonical sequence**
```
copy_semundo(clone_flags, p)
copy_files(clone_flags, p, args->no_files)   ← CLONE_FILES: share; else dup fdtable
copy_fs(clone_flags, p)                       ← CLONE_FS: share; else copy fs_struct
copy_sighand(clone_flags, p)                  ← CLONE_SIGHAND: share; else dup sighand
copy_signal(clone_flags, p)                   ← CLONE_THREAD: share; else alloc new
copy_mm(clone_flags, p)                       ← CLONE_VM: share; else dup_mm() → COW
copy_namespaces(clone_flags, p)               ← various CLONE_NEW* flags
copy_io(clone_flags, p)                       ← io context
copy_thread(p, args)                          ← arch-specific: sets up stack frame/entry point
```

**Phase 7: PID allocation (lines 2417–2424)**
```
alloc_pid(p->nsproxy->pid_ns_for_children, args->set_tid, args->set_tid_size)
```
PID is allocated in all visible namespaces. A `struct upid` is created for each level.

**Phase 8: PIDFD setup (lines 2431–2442)**
If `CLONE_PIDFD`:
```
__pidfd_prepare(pid, flags, &pidfile)
put_user(pidfd, args->pidfd)
```

**Phase 9: Make task visible under tasklist_lock (lines 2530–2621)**
```
write_lock_irq(&tasklist_lock)
  p->real_parent = current (or current->real_parent if CLONE_PARENT/CLONE_THREAD)
  p->start_time = ktime_get_ns()
  p->start_boottime = ktime_get_boottime_ns()
  if thread_group_leader(p):
    list_add_tail(&p->sibling, &p->real_parent->children)
    list_add_tail_rcu(&p->tasks, &init_task.tasks)
    attach_pid(p, PIDTYPE_TGID/PGID/SID)
    __this_cpu_inc(process_counts)
  else:
    signal->nr_threads++
    signal->quick_threads++
    list_add_tail_rcu(&p->thread_node, &signal->thread_head)
  attach_pid(p, PIDTYPE_PID)
  nr_threads++
write_unlock_irq(&tasklist_lock)
```

**Phase 10: Post-visible setup (lines 2623–2636)**
```
fd_install(pidfd, pidfile)             ← install pidfd into caller's fdtable
proc_fork_connector(p)                ← netlink proc event
sched_post_fork(p)                     ← complete scheduler setup
cgroup_post_fork(p, args)
perf_event_fork(p)
trace_task_newtask(p, clone_flags)
uprobe_copy_process(p, clone_flags)
copy_oom_score_adj(clone_flags, p)
```

### Clone Flags Behavior Summary

| Flag | Effect in `copy_process()` |
|---|---|
| `CLONE_VM` | `copy_mm()` increments mm refcount (shares); otherwise `dup_mm()` with COW |
| `CLONE_FS` | `copy_fs()` increments `fs->users` (shares `fs_struct`); otherwise `copy_fs_struct()` |
| `CLONE_FILES` | `copy_files()` increments `files->count` (shares fdtable); otherwise duplicates |
| `CLONE_SIGHAND` | `copy_sighand()` increments refcount (shares `sighand_struct`); otherwise duplicates |
| `CLONE_THREAD` | shares `signal_struct`; sets `p->tgid = current->tgid`, `p->group_leader` |
| `CLONE_NEWNS` | `copy_namespaces()` creates new `mnt_namespace` |
| `CLONE_NEWPID` | `copy_namespaces()` creates new `pid_namespace`; child is PID 1 in it |
| `CLONE_PIDFD` | allocates pidfd via `__pidfd_prepare()`, writes fd to `args->pidfd` |
| `CLONE_PARENT_SETTID` | `put_user(pid_nr, args->parent_tid)` done in `kernel_clone()` |
| `CLONE_CHILD_SETTID` | sets `p->set_child_tid`; written in `mm_release()` |
| `CLONE_CHILD_CLEARTID` | sets `p->clear_child_tid`; cleared + futex wake in `mm_release()` |

### Error Unwind Labels (`kernel/fork.c:2639–2703`)

The unwind chain is label-based, releasing each resource in reverse order of acquisition:

```
bad_fork_core_free      → sched_core_free(p); unlock sighand+tasklist
bad_fork_cancel_cgroup  → cgroup_cancel_fork(p, args)
bad_fork_put_pidfd      → fput(pidfile); put_unused_fd(pidfd)
bad_fork_free_pid       → free_pid(pid)
bad_fork_cleanup_thread → exit_thread(p)
bad_fork_cleanup_io     → exit_io_context(p)
bad_fork_cleanup_namespaces → exit_task_namespaces(p)
bad_fork_cleanup_mm     → mm_clear_owner(p->mm); mmput(p->mm)
bad_fork_cleanup_signal → free_signal_struct(p->signal)
bad_fork_cleanup_sighand → __cleanup_sighand(p->sighand)
bad_fork_cleanup_fs     → exit_fs(p)
bad_fork_cleanup_files  → exit_files(p)
bad_fork_cleanup_semundo → exit_sem(p)
bad_fork_cleanup_security → security_task_free(p)
bad_fork_cleanup_audit  → audit_free(p)
bad_fork_cleanup_perf   → perf_event_free_task(p)
bad_fork_sched_cancel_fork → sched_cancel_fork(p)
bad_fork_cleanup_policy → lockdep_free_task(p); mpol_put(p->mempolicy)
bad_fork_cleanup_delayacct → delayacct_tsk_free(p)
bad_fork_cleanup_count  → dec_rlimit_ucounts(...); exit_creds(p)
bad_fork_free           → WRITE_ONCE(p->__state, TASK_DEAD); put_task_stack(p); delayed_free_task(p)
fork_out                → remove delayed.node from multiprocess list
```

---

## exec Internals

### `do_execveat_common()` Flow (`fs/exec.c:1873–1956`)

Common entry point for all exec variants:

```c
static int do_execveat_common(int fd, struct filename *filename,
                              struct user_arg_ptr argv,
                              struct user_arg_ptr envp,
                              int flags)
```

**Full sequence:**

```
1. RLIMIT_NPROC re-check if PF_NPROC_EXCEEDED set
2. alloc_bprm(fd, filename, flags)
     do_open_execat(fd, filename, flags)    ← opens file, denies write access
     kzalloc(sizeof(*bprm), GFP_KERNEL)
     bprm->file = file
     bprm->filename = filename->name (or /dev/fd/N path for execveat)
     bprm->interp = bprm->filename
     bprm_mm_init(bprm)
       mm_alloc()                           ← fresh mm_struct
       __bprm_mm_init(bprm)
         vm_area_alloc(mm)                  ← temporary stack VMA at STACK_TOP_MAX
         insert_vm_struct(mm, vma)
         bprm->p = vma->vm_end - sizeof(void *)
3. count(argv, MAX_ARG_STRINGS)  → bprm->argc
4. count(envp, MAX_ARG_STRINGS)  → bprm->envc
5. bprm_stack_limits(bprm)       ← compute bprm->argmin from RLIMIT_STACK
6. copy_string_kernel(bprm->filename, bprm)  ← push filename onto bprm stack
   bprm->exec = bprm->p
7. copy_strings(bprm->envc, envp, bprm)     ← push env strings
8. copy_strings(bprm->argc, argv, bprm)     ← push arg strings
9. bprm_execve(bprm)
```

### `bprm_execve()` (`fs/exec.c:1818–1871`)

```
prepare_bprm_creds(bprm)          ← allocate bprm->cred via prepare_exec_creds()
check_unsafe_exec(bprm)           ← set bprm->unsafe (ptrace, shared fs, NO_NEW_PRIVS)
current->in_execve = 1
sched_exec()                      ← migrate to best CPU for exec
security_bprm_creds_for_exec(bprm) ← LSM hook (sets secureexec)
exec_binprm(bprm)                 ← loop up to 5 levels deep for interpreter chains
  search_binary_handler(bprm)     ← walks formats list, calls fmt->load_binary(bprm)
sched_mm_cid_after_execve(current)
current->in_execve = 0
```

On `exec_binprm()` error with `bprm->point_of_no_return` set, forces `SIGSEGV` via `force_fatal_sig(SIGSEGV)`.

### `search_binary_handler()` (`fs/exec.c:1726–1769`)

```
prepare_binprm(bprm)                         ← reads first BINPRM_BUF_SIZE bytes into bprm->buf
security_bprm_check(bprm)                    ← LSM check hook
read_lock(&binfmt_lock)
list_for_each_entry(fmt, &formats, lh):
    fmt->load_binary(bprm)                   ← tries each registered handler
    if bprm->point_of_no_return || retval != -ENOEXEC: return retval
read_unlock(&binfmt_lock)
[if CONFIG_MODULES and -ENOEXEC: request_module("binfmt-%04x", magic) and retry once]
```

### `begin_new_exec()` — Point of No Return (`fs/exec.c:1216–1396`)

Where `bprm->point_of_no_return = true` is set (line 1237). After this, errors cannot be returned to the calling userspace process:

```
bprm_creds_from_file(bprm)            ← compute suid/sgid from inode mode
trace_sched_prepare_exec(current, bprm)
bprm->point_of_no_return = true       ← *** POINT OF NO RETURN SET ***
de_thread(me)                          ← kill all other threads in group; wait for them
io_uring_task_cancel()
unshare_files()                        ← ensure private fdtable
set_mm_exe_file(bprm->mm, bprm->file) ← attach exe file to new mm
exec_mmap(bprm->mm)                   ← install new mm (see below)
bprm->mm = NULL
exec_task_namespaces()
posix_cpu_timers_exit(me); exit_itimers(me)
unshare_sighand(me)                    ← get private sighand_struct
me->flags &= ~(PF_RANDOMIZE | PF_FORKNOEXEC | PF_NOFREEZE | PF_NO_SETAFFINITY)
flush_thread()                         ← arch-specific register state flush
do_close_on_exec(me->files)           ← close FD_CLOEXEC descriptors
__set_task_comm(me, kbasename(bprm->filename), true)
flush_signal_handlers(me, 0)          ← reset all signal handlers to SIG_DFL
commit_creds(bprm->cred)              ← install new credentials
```

### `exec_mmap()` — Replacing the Old MM (`fs/exec.c:962–1022`)

```
exec_mm_release(tsk, old_mm)                   ← notify futex, etc.
down_write_killable(&tsk->signal->exec_update_lock)
mmap_read_lock_killable(old_mm)                 ← if old_mm exists
task_lock(tsk)
  local_irq_disable()
  tsk->active_mm = mm
  tsk->mm = mm                                  ← install new mm atomically
  mm_init_cid(mm, tsk)
  local_irq_enable()
  activate_mm(active_mm, mm)                    ← arch TLB switch
task_unlock(tsk)
mmap_read_unlock(old_mm)
mmput(old_mm)                                   ← drop reference to old mm
```

The `exec_update_lock` (`signal->exec_update_lock`, a `rw_semaphore`) is held write-locked across the mm swap so ptrace and credential readers see a consistent view.

### `struct linux_binprm` Key Fields (`include/linux/binfmts.h:18–65`)

```c
struct linux_binprm {
    struct vm_area_struct *vma;      /* temporary argument stack VMA */
    unsigned long vma_pages;
    unsigned long argmin;            /* rlimit marker for copy_strings() */
    struct mm_struct *mm;            /* new mm being constructed */
    unsigned long p;                 /* current top of argument stack */
    unsigned int have_execfd:1;      /* execveat with fd to pass to interpreter */
    unsigned int execfd_creds:1;     /* use creds of script (binfmt_misc) */
    unsigned int secureexec:1;       /* privilege-gaining exec (sets AT_SECURE) */
    unsigned int point_of_no_return:1; /* errors can no longer return to userspace */
    struct file *executable;         /* executable file for interpreter pass-through */
    struct file *interpreter;        /* interpreter file (scripts, binfmt_misc) */
    struct file *file;               /* file being executed */
    struct cred *cred;               /* new credentials under construction */
    int unsafe;                      /* LSM_UNSAFE_* mask */
    unsigned int per_clear;          /* personality bits to clear */
    int argc, envc;                  /* argument and environment counts */
    const char *filename;            /* name as seen by /proc */
    const char *interp;              /* name of binary actually executed */
    const char *fdpath;              /* generated name for execveat fd case */
    unsigned interp_flags;           /* BINPRM_FLAGS_* */
    int execfd;                      /* fd for AT_EXECFD aux vector entry */
    unsigned long loader, exec;      /* stack positions of loader/executable paths */
    struct rlimit rlim_stack;        /* saved RLIMIT_STACK for exec */
    char buf[BINPRM_BUF_SIZE];       /* first bytes of file (magic number buffer) */
};
```

### `struct linux_binfmt` (`include/linux/binfmts.h:82–91`)

```c
struct linux_binfmt {
    struct list_head lh;                      /* linked into global formats list */
    struct module *module;
    int (*load_binary)(struct linux_binprm *); /* load and execute a binary */
    int (*load_shlib)(struct file *);          /* load a shared library */
    int (*core_dump)(struct coredump_params *cprm); /* generate core dump */
    unsigned long min_coredump;               /* minimum dump size */
};
```

Handlers are registered via `register_binfmt()` / `insert_binfmt()` and protected by the `binfmt_lock` rwlock.

### ELF Loading: `load_elf_binary()` Key Steps (`fs/binfmt_elf.c:825`)

```
1. Verify ELF magic: memcmp(elf_ex->e_ident, ELFMAG, SELFMAG)
2. Check e_type == ET_EXEC or ET_DYN; check arch
3. load_elf_phdrs()                     ← read all program headers
4. Scan PT_INTERP segment → open_exec(interpreter_path)
5. Check PT_GNU_STACK → set executable_stack flag
6. arch_check_elf() → last chance to reject before point of no return
7. begin_new_exec(bprm)                 ← POINT OF NO RETURN
8. SET_PERSONALITY2(*elf_ex, &arch_state)
9. setup_new_exec(bprm)                 ← set task name, update mm flags
10. setup_arg_pages(bprm, STACK_TOP, executable_stack) ← finalize stack VMA
11. Loop over PT_LOAD segments:
    elf_load(bprm->file, load_bias + vaddr, elf_ppnt, prot, flags, total_size)
    → elf_map() → vm_mmap() with file-backed mapping
    Track start_code, end_code, start_data, end_data, elf_brk
12. current->mm->start_brk = current->mm->brk = ELF_PAGEALIGN(elf_brk)
13. If interpreter:
    load_elf_interp()                   ← map ld.so segments; elf_entry = interp entry
    Else: elf_entry = e_entry + load_bias
14. create_elf_tables()                 ← push aux vector, argv ptrs, envp ptrs to stack
15. START_THREAD(regs, elf_entry, bprm->p)  ← set PC and SP in pt_regs → return to user
```

---

## Signal Delivery Internals

### Data Structures (`include/linux/signal_types.h`)

```c
/* A queued signal instance — SLAB allocated from sigqueue_cachep */
struct sigqueue {
    struct list_head list;      /* links into sigpending.list */
    int flags;                  /* SIGQUEUE_PREALLOC */
    kernel_siginfo_t info;      /* full siginfo payload */
    struct ucounts *ucounts;    /* for RLIMIT_SIGPENDING accounting */
};

/* Per-task or per-group pending signal set */
struct sigpending {
    struct list_head list;   /* list of struct sigqueue nodes */
    sigset_t signal;         /* bitmap: which signals are pending (any count) */
};
```

Each `task_struct` has `task->pending` (thread-private) and the shared `task->signal->shared_pending`. The `sigset_t` bitmap is the fast-path "any pending?" check; the linked `list` holds full `siginfo` payloads.

### `struct sighand_struct` (`include/linux/sched/signal.h:21–26`)

```c
struct sighand_struct {
    spinlock_t          siglock;           /* protects action[], pending queues */
    refcount_t          count;             /* shared among CLONE_SIGHAND tasks */
    wait_queue_head_t   signalfd_wqh;      /* signalfd waiters */
    struct k_sigaction  action[_NSIG];     /* per-signal disposition table */
};
```

### Standard vs RT Signal Queuing

| Aspect | Standard signals (1–31) | Real-time signals (32–64) |
|---|---|---|
| Bitmap tracking | `sigset_t signal` bit set | `sigset_t signal` bit set |
| Queue semantics | At most one queued (coalesced) via `legacy_queue()` check | Multiple instances queued in `sigqueue.list` |
| `legacy_queue()` check | If bit already set in bitmap, skip `sigqueue_alloc` | No coalescing, always allocate `sigqueue` |
| `override_rlimit` | Kernel-generated signals (si_code >= 0) bypass limit | Never bypass limit |
| SIGKILL / kthread | Skip `sigqueue_alloc` entirely (`goto out_set`) | N/A |

### `__send_signal_locked()` Flow (`kernel/signal.c:1041–1156`)

Requires `t->sighand->siglock` held.

```
1. prepare_signal(sig, t, force)
     → checks ignored, SIGNAL_UNKILLABLE, etc.
     → if false: goto ret (TRACE_SIGNAL_IGNORED)

2. pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending
                                    : &t->pending

3. if legacy_queue(pending, sig):     ← sig < SIGRTMIN && bit already set
     goto ret (TRACE_SIGNAL_ALREADY_PENDING)

4. if (sig == SIGKILL) || PF_KTHREAD: goto out_set  ← skip sigqueue alloc

5. override_rlimit = (sig < SIGRTMIN) && (is_si_special(info) || si_code >= 0)
   sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit)
   if q:
     list_add_tail(&q->list, &pending->list)
     fill q->info from info parameter
   elif RT signal from user and queue overflow: return -EAGAIN

out_set:
6. signalfd_notify(t, sig)
7. sigaddset(&pending->signal, sig)    ← set the bitmap bit
8. complete_signal(sig, t, type)       ← find a thread to wake
```

### `complete_signal()` (`kernel/signal.c:962–1034`)

Finds a thread that can take the signal and wakes it:

```
1. if wants_signal(sig, p):  t = p   ← suggested thread can handle it
2. elif type == PIDTYPE_PID || thread_group_empty: return
3. else: walk thread group starting at signal->curr_target:
     while !wants_signal(sig, t): t = next_thread(t)
     signal->curr_target = t          ← remember last target for next time

4. if sig_fatal(p, sig) && !core_dump_pending && !GROUP_EXIT && !blocked && !ptrace:
     if !sig_kernel_coredump(sig):
       /* Fast-path whole-group kill */
       signal->flags = SIGNAL_GROUP_EXIT
       signal->group_exit_code = sig
       for_each_thread:
         sigaddset(&t->pending.signal, SIGKILL)
         signal_wake_up(t, 1)         ← wake all with TIF_SIGPENDING
       return
5. signal_wake_up(t, sig == SIGKILL)  ← set TIF_SIGPENDING, kick scheduler
```

### `get_signal()` — Dequeuing on Return to Userspace (`kernel/signal.c:2801–3047`)

Called from `exit_to_user_mode_loop()` on each return to userspace when `TIF_SIGPENDING` is set.

```
spin_lock_irq(&sighand->siglock)
relock:
  if SIGNAL_CLD_MASK: handle stop/continue notification → goto relock
  loop:
    if SIGNAL_GROUP_EXIT || group_exec_task: signr = SIGKILL → goto fatal
    if JOBCTL_STOP_PENDING: do_signal_stop() → goto relock
    signr = dequeue_synchronous_signal()  ← fault-generated first
    if !signr: signr = dequeue_signal(&current->blocked, &ksig->info, &type)
    if !signr: break (return false)
    if ptrace && !SA_IMMUTABLE: ptrace_signal()  ← ptrace interception
    ka = &sighand->action[signr-1]
    if ka->sa_handler == SIG_IGN: continue
    if ka->sa_handler != SIG_DFL:
      ksig->ka = *ka
      if SA_ONESHOT: ka->sa_handler = SIG_DFL
      break (return true → handle in userspace)
    /* Default action */
    if sig_kernel_ignore(signr): continue
    if sig_kernel_stop(signr): do_signal_stop() → goto relock
  fatal:
    spin_unlock_irq(...)
    current->flags |= PF_SIGNALED
    if sig_kernel_coredump(signr): do_coredump(&ksig->info)
    do_group_exit(signr)              ← calls do_exit(), never returns
```

### Signal Frame Setup (x86)

After `get_signal()` returns true with a user handler, the arch calls `handle_signal()` → `setup_rt_frame()` (`arch/x86/kernel/signal.c`):
1. Builds a `struct rt_sigframe` on the user stack containing `siginfo`, `ucontext` (saved registers), and restorer trampoline address
2. Sets the task's PC to `ka->sa.sa_handler`
3. Sets the task's SP to the new frame

On handler return, the restorer calls `rt_sigreturn` syscall → `sys_rt_sigreturn()` → restores `ucontext` → `RESTORE_SAVED_SIGMASK`.

### `sigaction` Flag Semantics

| Flag | Effect |
|---|---|
| `SA_RESTART` | Interrupted syscalls restart automatically instead of returning `-EINTR` |
| `SA_SIGINFO` | Handler receives `siginfo_t *` and `ucontext_t *` (three-argument form) |
| `SA_NODEFER` | Signal not blocked during its own handler (no automatic self-masking) |
| `SA_RESETHAND` (`SA_ONESHOT`) | Reset to `SIG_DFL` after first delivery |
| `SA_NOCLDWAIT` | No zombie children; `do_notify_parent()` auto-reaps |
| `SA_NOCLDSTOP` | `SIGCHLD` not sent for child stop/continue events |
| `SA_ONSTACK` | Deliver on alternate signal stack (`task->sas_ss_sp`) |
| `SA_IMMUTABLE` | Handler cannot be intercepted by ptrace |

---

## Process Exit Path

### `do_exit()` (`kernel/exit.c:876–990`)

```c
void __noreturn do_exit(long code)
```

Full sequence:

```
synchronize_group_exit(tsk, code)
  spin_lock_irq(&sighand->siglock)
    signal->quick_threads--
    if (quick_threads == 0) && !SIGNAL_GROUP_EXIT:
      signal->flags = SIGNAL_GROUP_EXIT
      signal->group_exit_code = code
  spin_unlock_irq(...)

kcov_task_exit(tsk)
coredump_task_exit(tsk)            ← sync with ongoing coredump
ptrace_event(PTRACE_EVENT_EXIT, code)
io_uring_files_cancel()
exit_signals(tsk)                  ← sets PF_EXITING; wakes threads blocked on signal
acct_update_integrals(tsk)

group_dead = atomic_dec_and_test(&tsk->signal->live)
  if group_dead:
    hrtimer_cancel(&signal->real_timer)
    exit_itimers(tsk)

tsk->exit_code = code
taskstats_exit(tsk, group_dead)

exit_mm()                          ← mmput(tsk->mm); sets tsk->mm = NULL
if group_dead: acct_process()

exit_sem(tsk)                      ← SysV semaphore undo
exit_shm(tsk)                      ← SysV shared memory
exit_files(tsk)                    ← drop reference to files_struct
exit_fs(tsk)                       ← drop reference to fs_struct
if group_dead: disassociate_ctty(1) ← release controlling terminal
exit_task_namespaces(tsk)          ← put_nsproxy()
exit_task_work(tsk)                ← flush pending task_work
exit_thread(tsk)                   ← arch-specific (e.g., FPU state)
perf_event_exit_task(tsk)
cgroup_exit(tsk)

exit_tasks_rcu_start()
exit_notify(tsk, group_dead)       ← zombie transition, notify parent
proc_exit_connector(tsk)

exit_rcu()
exit_tasks_rcu_finish()
do_task_dead()                     ← set TASK_DEAD, schedule away forever
```

### `exit_notify()` — Zombie Transition (`kernel/exit.c:730–777`)

```
write_lock_irq(&tasklist_lock)
  forget_original_parent(tsk, &dead)   ← reparent children to subreaper/init
  if group_dead: kill_orphaned_pgrp()  ← SIGHUP+SIGCONT to orphaned stopped pgrps

  tsk->exit_state = EXIT_ZOMBIE        ← visible as zombie from this point

  if tsk->ptrace:
    autoreap = do_notify_parent(tsk, SIGCHLD or exit_signal)
  elif thread_group_leader(tsk):
    autoreap = thread_group_empty(tsk) && do_notify_parent(tsk, tsk->exit_signal)
  else:
    autoreap = true                     ← non-leader thread: always autoreap

  if autoreap:
    tsk->exit_state = EXIT_DEAD         ← skip zombie state; straight to dead
    list_add(&tsk->ptrace_entry, &dead)

write_unlock_irq(&tasklist_lock)

list_for_each_entry_safe(p, n, &dead, ptrace_entry):
  release_task(p)                       ← for autoreap cases
```

### `release_task()` (`kernel/exit.c:240–287`)

Called when the task can be fully freed (from `exit_notify()` auto-reap or `wait4()` by parent):

```
write_lock_irq(&tasklist_lock)
  ptrace_release_task(p)
  __exit_signal(p):
    spin_lock(&sighand->siglock)
      posix_cpu_timers_exit(tsk)
      task_cputime() → accumulate into sig->utime/stime/gtime
      __unhash_process(p, group_dead):
        nr_threads--
        detach_pid(p, PIDTYPE_PID)
        if group_dead: detach_pid(TGID/PGID/SID); list_del_rcu(&p->tasks)
        list_del_rcu(&p->thread_node)
      flush_sigqueue(&tsk->pending)
      tsk->sighand = NULL
    spin_unlock(&sighand->siglock)
    __cleanup_sighand(sighand)    ← decref; free if last
write_unlock_irq(&tasklist_lock)
proc_flush_pid(thread_pid)
put_pid(thread_pid)               ← free struct pid when refcount hits 0
put_task_struct_rcu_user(p)       ← dec rcu_users; call_rcu(delayed_put_task_struct)
```

`delayed_put_task_struct()` runs in RCU callback context: calls `put_task_struct()` → `__put_task_struct()` → `security_task_free()`, `exit_creds()`, `free_task()`.

### `do_task_dead()` (`kernel/sched/core.c:6772–6786`)

```c
void __noreturn do_task_dead(void)
{
    set_special_state(TASK_DEAD);       /* sets current->__state atomically */
    current->flags |= PF_NOFREEZE;
    __schedule(SM_NONE);                /* context switch; never returns */
    BUG();
}
```

The task is removed from the run queue in `__schedule()`. `finish_task_switch()` calls `put_task_struct()` on the dead task, dropping the scheduler's reference.

### Thread Group Exit: `zap_other_threads()` (`kernel/signal.c:1336–1355`)

Called from `do_group_exit()` (under `sighand->siglock`) and from `de_thread()` during exec:

```
p->signal->group_stop_count = 0
for_other_threads(p, t):
  task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK)
  if t->exit_state: continue          ← already dead
  sigaddset(&t->pending.signal, SIGKILL)
  signal_wake_up(t, 1)               ← set TIF_SIGPENDING, kick off CPU
```

---

## Locking Summary

| Lock Name | Type | Protects | Key File(s) |
|---|---|---|---|
| `task->alloc_lock` (`task_lock()`) | `spinlock_t` | `task->mm`, `task->files`, `task->fs`, `task->nsproxy`, `task->signal->rlim` (group leader) | `include/linux/sched.h` |
| `sighand->siglock` | `spinlock_t` (irq-disabling) | `sighand->action[]`, all `sigpending` queues, `signal->flags`, thread-group stop state | `include/linux/sched/signal.h:22` |
| `tasklist_lock` | `rwlock_t` | Process/thread list linkage (`task->tasks`, `children`, `sibling`, `thread_node`), PID hash tables, parent/child relationships, `exit_state` transitions | `include/linux/sched/task.h:55` |
| `signal->cred_guard_mutex` | `mutex` | Credential calculation consistency during exec and ptrace attach | `include/linux/sched/signal.h:239` |
| `signal->exec_update_lock` | `rw_semaphore` | Task credential/mm consistency during exec; write-locked in `exec_mmap()` | `include/linux/sched/signal.h:245` |
| `mm->mmap_lock` | `rw_semaphore` | All VMA tree modifications and most mm field writes; readers for page faults | `include/linux/mm_types.h` |
| `files->file_lock` | `spinlock_t` | `files->fdt`, `files->next_fd`, open/close-on-exec bitmaps, `fdtable->fd[]` resizing | `include/linux/fdtable.h:51` |
| `fs->lock` | `spinlock_t` + `seqcount_spinlock_t seq` | `fs->root`, `fs->pwd`, `fs->umask`, `fs->in_exec` | `include/linux/fs_struct.h:9–15` |
| `pid->lock` | `spinlock_t` | `pid->tasks[]` hash lists, `pid->inodes`; used during `attach_pid()` / `detach_pid()` | `include/linux/pid.h:59` |
| `cgroup_threadgroup_rwsem` | `percpu_rw_semaphore` | Thread group membership changes vs cgroup migration | `kernel/cgroup/cgroup.c:114` |
| `cgroup_mutex` | `mutex` | cgroup hierarchy structure, css attachment, task migration | `kernel/cgroup/cgroup.c` |
| `binfmt_lock` | `rwlock_t` | `formats` list of registered `linux_binfmt` handlers | `fs/exec.c:86` |
| `signal->stats_lock` | `seqlock_t` | Thread group CPU time accumulators (`sig->utime`, `sig->stime`) | `include/linux/sched/signal.h:187` |
| RCU (read-side) | RCU | `task->cred` (via `rcu_assign_pointer` / `rcu_dereference`), `task->sighand`, `task->files` for `fget_light` | various |

---

## Source File Reference

| File | Purpose | Key Functions |
|---|---|---|
| `kernel/fork.c` | Process/thread creation; mm/fd/cred duplication | `copy_process()`, `dup_task_struct()`, `kernel_clone()`, `copy_mm()`, `copy_files()`, `copy_fs()`, `copy_sighand()`, `copy_signal()`, `copy_namespaces()`, `copy_thread()` |
| `kernel/exit.c` | Process exit, zombie management, wait | `do_exit()`, `exit_notify()`, `release_task()`, `do_group_exit()`, `__exit_signal()`, `__unhash_process()` |
| `kernel/signal.c` | Signal sending, delivery, queuing, job control | `__send_signal_locked()`, `complete_signal()`, `get_signal()`, `do_notify_parent()`, `zap_other_threads()` |
| `kernel/pid.c` | PID allocation, struct pid lifecycle, pidfd | `alloc_pid()`, `free_pid()`, `attach_pid()`, `detach_pid()`, `find_pid_ns()`, `pid_task()` |
| `kernel/cred.c` | Credential lifecycle, COW credentials | `copy_creds()`, `prepare_creds()`, `prepare_exec_creds()`, `commit_creds()`, `abort_creds()` |
| `kernel/nsproxy.c` | Namespace proxy lifecycle | `copy_namespaces()`, `unshare_nsproxy_namespaces()`, `put_nsproxy()` |
| `kernel/sched/core.c` | Scheduler core; task dead path | `do_task_dead()`, `sched_fork()`, `finish_task_switch()`, `schedule()` |
| `fs/exec.c` | Binary execution, bprm lifecycle, mm replacement | `do_execveat_common()`, `bprm_execve()`, `search_binary_handler()`, `begin_new_exec()`, `exec_mmap()`, `de_thread()` |
| `fs/binfmt_elf.c` | ELF binary loader | `load_elf_binary()`, `load_elf_interp()`, `load_elf_phdrs()`, `create_elf_tables()` |
| `fs/file.c` | File descriptor table management | `alloc_fd()`, `__close_fd()`, `copy_files()`, `unshare_files()`, `do_close_on_exec()` |
| `include/linux/sched.h` | `struct task_struct` definition | All `task_struct` fields, task state constants |
| `include/linux/sched/task.h` | Fork/exit interfaces; `struct kernel_clone_args` | `kernel_clone_args`, `tasklist_lock`, `copy_thread()`, `release_task()` |
| `include/linux/sched/signal.h` | `struct signal_struct`, `struct sighand_struct` | `signal_struct`, `sighand_struct`, signal delivery helpers |
| `include/linux/pid.h` | `struct pid`, `struct upid`, PID API | `struct pid`, `alloc_pid()`, `find_pid_ns()` |
| `include/linux/cred.h` | `struct cred`, credential API | `struct cred`, `prepare_creds()`, `commit_creds()` |
| `include/linux/binfmts.h` | `struct linux_binprm`, `struct linux_binfmt` | `linux_binprm`, `linux_binfmt`, `register_binfmt()` |
| `include/linux/signal_types.h` | `struct sigqueue`, `struct sigpending` | `sigqueue`, `sigpending`, `k_sigaction` |
| `include/linux/fdtable.h` | `struct files_struct`, `struct fdtable` | `files_struct`, `fdtable`, `files_lookup_fd_raw()` |

