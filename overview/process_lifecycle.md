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
│  │  Parent  │────▶│     Physical Pages       │              │
│  │   mm     │     │  [Page 1][Page 2][...]   │              │
│  └──────────┘     └──────────────────────────┘              │
│                                                             │
│  After fork (no copy yet):                                  │
│  ┌──────────┐                                               │
│  │  Parent  │──┐                                            │
│  │   mm     │  │  ┌──────────────────────────┐              │
│  └──────────┘  └─▶│     Physical Pages       │              │
│  ┌──────────┐  ┌─▶│  [Page 1][Page 2][...]   │              │
│  │  Child   │──┘  │    (marked read-only)    │              │
│  │   mm     │     └──────────────────────────┘              │
│  └──────────┘                                               │
│                                                             │
│  After child writes to Page 2:                              │
│  ┌──────────┐     ┌──────────────────────────┐              │
│  │  Parent  │────▶│  [Page 1]     [Page 2]   │              │
│  │   mm     │     └──────────────────────────┘              │
│  └──────────┘                                               │
│  ┌──────────┐     ┌──────────────────────────┐              │
│  │  Child   │────▶│  [Page 1]     [Page 2']  │ ← New copy   │
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
│                     mmap() Flow                              │
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

## Summary

The Linux process lifecycle involves:

| Stage | Key Functions | Structures |
|-------|---------------|------------|
| Creation | fork(), clone(), copy_process() | task_struct, mm_struct |
| Memory Mapping | mmap(), do_mmap() | vm_area_struct |
| Execution | execve(), load_elf_binary() | linux_binprm |
| Dynamic Linking | _dl_runtime_resolve() | PLT, GOT, Elf64_Dyn |
| Running | schedule(), wake_up() | task states |
| Termination | do_exit(), exit_notify() | exit codes |
| Reaping | wait(), release_task() | zombie state |
| Signals | send_signal(), do_signal() | sigpending, sigqueue |

Key mechanisms:
- **clone()**: Fundamental syscall for creating processes and threads
- **mmap()**: Memory mapping for loading code, allocating memory, IPC
- **PLT/GOT**: Lazy symbol resolution for dynamic linking
- **Namespaces**: Resource view isolation (PID, network, mount, etc.)
- **Cgroups**: Resource limits and accounting
- **Credentials/Capabilities**: Permission control
- **Seccomp**: System call filtering

