# Linux Kernel IPC Subsystem

## Overview

Linux provides multiple inter-process communication mechanisms spanning several kernel subsystems: traditional SysV IPC objects (`ipc/`), POSIX IPC layered on VFS, pipes (`fs/pipe.c`), Unix domain sockets (`net/unix/`), signals (`kernel/signal.c`), and a family of file-descriptor-based primitives (eventfd, signalfd, timerfd, pidfd, futex, memfd, userfaultfd). Each has distinct data structures, locking strategies, and lifetime semantics.

---

## System V IPC

### Core Architecture

All SysV IPC objects (shared memory, semaphores, message queues) share a common header:

#### `struct kern_ipc_perm`

`include/linux/ipc.h:12-29`

```c
struct kern_ipc_perm {
    spinlock_t   lock;         // per-object spinlock
    bool         deleted;      // set during ipc_rmid()
    int          id;           // IPC identifier
    key_t        key;          // user-visible key (IPC_PRIVATE = 0)
    kuid_t       uid, cuid;    // owner and creator UIDs
    kgid_t       gid, cgid;    // owner and creator GIDs
    umode_t      mode;         // permission bits
    unsigned long seq;         // sequence number for public ID construction
    void        *security;     // LSM blob
    struct rhash_head khtnode; // per-namespace key hash table node
    struct rcu_head   rcu;     // deferred kfree
    refcount_t   refcount;     // RCU-safe lifetime
};
```

Embedded as the first member of `shmid_kernel.shm_perm`, `sem_array.sem_perm`, and `msg_queue.q_perm`.

#### `struct ipc_ids` — Per-Namespace ID Management

`include/linux/ipc_namespace.h:18-29`

```c
struct ipc_ids {
    int                   in_use;     // allocated object count
    unsigned short        seq;        // sequence counter for ID generation
    struct rw_semaphore   rwsem;      // protects idr and rhashtable
    struct idr            ipcs_idr;   // numeric ID → kern_ipc_perm *
    int                   max_idx;    // highest index ever used
    struct rhashtable     key_ht;     // key_t → kern_ipc_perm * (O(1) lookup)
};
```

Each `ipc_namespace` has `ids[3]`: index 0 for semaphores, 1 for message queues, 2 for shared memory.

#### Locking Protocol

Documented at `ipc/util.c:22-45`:

1. `rcu_read_lock()` — allows IDR lookups without rwsem
2. `ipc_obtain_object_idr()` — lookup `kern_ipc_perm *` from IDR
3. `ipc_lock_object(ipcp)` — acquires `ipcp->lock` for mutation
4. `ipc_valid_object(ipcp)` — checks `!ipcp->deleted` for race detection
5. For structural changes (create/delete): `down_write(&ids->rwsem)`

### Shared Memory

`ipc/shm.c`

#### `struct shmid_kernel` (line 53)

```c
struct shmid_kernel {
    struct kern_ipc_perm  shm_perm;     // base IPC object
    struct file          *shm_file;     // backing tmpfs/hugetlbfs file
    unsigned long         shm_nattch;   // current attach count
    unsigned long         shm_segsz;    // segment size in bytes
    time64_t              shm_atim;     // last shmat time
    time64_t              shm_dtim;     // last shmdt time
    time64_t              shm_ctim;     // last change time
    struct pid           *shm_cprid;    // creator pid
    struct pid           *shm_lprid;    // last operation pid
    struct task_struct   *shm_creator;  // for shm_clist
    struct list_head      shm_clist;    // creator's task list
    struct ipc_namespace *ns;
};
```

**Key design**: `shm_file` points to a `shmem_fs` (tmpfs) or hugetlbfs file. `shmat()` calls `do_mmap()` on this file — all attachments share the same page cache. The wrapper `shm_vm_ops` tracks attach/detach counts.

**Destruction**: `IPC_RMID` with `shm_nattch > 0` sets `SHM_DEST` flag (mode bit `01000`) and makes the segment invisible to new lookups. Actual destruction happens on last detach via `shm_destroy()`.

#### Syscalls

| Syscall | Entry Point |
|---------|-------------|
| `shmget()` | `ipc/shm.c:842` → `newseg()` |
| `shmat()` | `ipc/shm.c:1688` → `do_shmat()` → `do_mmap()` |
| `shmdt()` | `ipc/shm.c:1829` → iterates VMAs, `do_munmap()` |
| `shmctl()` | `ipc/shm.c:1291` → `IPC_STAT/SET/RMID/INFO` |

### Semaphores

`ipc/sem.c`

#### `struct sem_array` (line 114)

```c
struct sem_array {
    struct kern_ipc_perm  sem_perm;
    time64_t              sem_ctime;
    struct list_head      pending_alter;   // array-level pending ops
    struct list_head      pending_const;   // array-level const ops
    struct list_head      list_id;         // per-array undo list
    int                   sem_nsems;       // number of semaphores
    int                   complex_count;   // multi-semaphore pending ops
    unsigned int          use_global_lock; // >0: must use array lock
    struct sem            sems[];          // flexible array
};
```

#### `struct sem` (line 95)

```c
struct sem {
    int              semval;         // current value
    struct pid      *sempid;         // last modifier pid
    spinlock_t       lock;           // per-semaphore fine-grained lock
    struct list_head pending_alter;  // single-sop alter ops
    struct list_head pending_const;  // single-sop const ops
};
```

**Two-level locking**: Simple single-semaphore operations use per-`sem` spinlocks (avoiding global contention). Complex multi-semaphore or undo operations use the array-level `kern_ipc_perm.lock`.

#### SEM_UNDO

`struct sem_undo` stores per-process `semadj[]` values. At `exit_sem()`, the kernel walks the process's undo list, applies adjustments, and wakes newly satisfiable waiters.

### Message Queues

`ipc/msg.c`

#### `struct msg_queue` (line 49)

```c
struct msg_queue {
    struct kern_ipc_perm  q_perm;
    time64_t              q_stime, q_rtime, q_ctime;
    unsigned long         q_cbytes;    // current bytes in queue
    unsigned long         q_qnum;      // message count
    unsigned long         q_qbytes;    // max bytes allowed
    struct pid           *q_lspid;     // last sender pid
    struct pid           *q_lrpid;     // last receiver pid
    struct list_head      q_messages;  // linked list of msg_msg
    struct list_head      q_receivers; // sleeping receivers
    struct list_head      q_senders;   // sleeping senders
};
```

**Pipelined delivery**: If a matching receiver exists in `q_receivers`, the message is handed directly without being queued — `wake_q_add_safe()` wakes the receiver. Otherwise the message is appended to `q_messages`.

### IPC Namespaces

`ipc/namespace.c`, `include/linux/ipc_namespace.h`

Each `struct ipc_namespace` holds:
- `ids[3]` — per-subsystem `ipc_ids`
- Per-namespace limits (semaphore counts, msg queue sizes, shm limits)
- `mq_mnt` — mqueuefs mount for POSIX MQ

Created with `CLONE_NEWIPC` in `copy_ipcs()`. Teardown via `free_ipc_work` destroys all objects after RCU grace period.

---

## POSIX IPC

### POSIX Message Queues

`ipc/mqueue.c`

Implemented as a pseudo-filesystem (`mqueuefs`). Each queue is a file in the namespace's `mq_mnt`, accessible at `/dev/mqueue`.

#### `struct mqueue_inode_info` (line 134)

```c
struct mqueue_inode_info {
    spinlock_t         lock;
    struct inode       vfs_inode;
    wait_queue_head_t  wait_q;
    struct rb_root     msg_tree;            // priority-sorted red-black tree
    struct rb_node    *msg_tree_rightmost;  // cached highest priority (O(1) receive)
    struct mq_attr     attr;
    struct sigevent    notify;              // async notification config
    struct pid        *notify_owner;
    struct ext_wait_queue e_wait_q[2];      // [SEND] and [RECV] queues
    unsigned long      qsize;
};
```

Messages stored in red-black tree keyed by priority; within each priority, FIFO ordering via `msg_list`.

### POSIX Semaphores

- **Unnamed** (`sem_init`): reside in user memory, implemented via futexes in glibc
- **Named** (`sem_open`): backed by shmem file in `/dev/shm`, use futex syscalls for contention

### POSIX Shared Memory

`shm_open()` is a glibc wrapper around `open("/dev/shm/<name>", ...)` on tmpfs. No kernel-level `shmid_kernel` — lifetime follows file link count and open descriptions. Contrast with SysV which has its own ID space and `IPC_RMID` semantics.

---

## Pipes and FIFOs

`fs/pipe.c`, `include/linux/pipe_fs_i.h`

### `struct pipe_inode_info` (line 58)

```c
struct pipe_inode_info {
    struct mutex          mutex;       // serializes read/write
    wait_queue_head_t     rd_wait;     // readers wait when empty
    wait_queue_head_t     wr_wait;     // writers wait when full
    unsigned int          head;        // production index (unwrapped)
    unsigned int          tail;        // consumption index (unwrapped)
    unsigned int          max_usage;   // current max slots
    unsigned int          ring_size;   // total slots (power of 2)
    unsigned int          readers;     // open read ends
    unsigned int          writers;     // open write ends
    struct page          *tmp_page;    // single-page alloc cache
    struct pipe_buffer   *bufs;        // circular buffer array
    struct user_struct   *user;        // quota accounting
};
```

### `struct pipe_buffer` (line 26)

```c
struct pipe_buffer {
    struct page *page;
    unsigned int offset;   // byte offset within page
    unsigned int len;      // valid data length
    const struct pipe_buf_operations *ops;
    unsigned int flags;    // PIPE_BUF_FLAG_*
};
```

**Ring buffer**: Uses unwrapped counters; slot index = `head & (ring_size - 1)`. Empty when `head == tail`, full when `head - tail >= max_usage`.

**Default capacity**: `PIPE_DEF_BUFFERS = 16` slots × `PAGE_SIZE` = 64KB. Resizable via `fcntl(F_SETPIPE_SZ)` up to `pipe_max_size` (default 1MB).

**Atomicity**: Writes ≤ `PIPE_BUF` (4096 bytes) are atomic — no interleaving from concurrent writers.

### O_NONBLOCK Behavior

- Read on empty pipe: `-EAGAIN`
- Write on full pipe: `-EAGAIN`
- Write with no readers: `EPIPE` + `SIGPIPE`

### splice / vmsplice / tee

- **`vmsplice()`**: maps user pages into pipe ring without copying
- **`splice()`**: moves `pipe_buffer` entries between pipe and file (zero-copy when possible)
- **`tee()`**: duplicates pipe buffers between two pipes without consuming source

### FIFOs (Named Pipes)

Same `pipe_inode_info` as anonymous pipes, allocated via `fifo_open()` in `fs/fifo.c`. The FIFO inode lives in a real filesystem. Read and write ends opened by separate `open()` calls which block (without `O_NONBLOCK`) until both ends are open.

---

## Unix Domain Sockets

`net/unix/af_unix.c`

### Socket Types

All use `AF_UNIX` / `PF_UNIX`:
- `SOCK_STREAM` — connection-oriented, byte stream
- `SOCK_DGRAM` — connectionless, datagram
- `SOCK_SEQPACKET` — connection-oriented, message boundaries preserved

### Addressing

- **Filesystem namespace**: `bind()` creates a socket inode (`S_IFSOCK`) at the path
- **Abstract namespace** (Linux extension): `sun_path[0] == '\0'`, remaining bytes form the name; no filesystem entry, garbage-collected on close

### Credential Passing (SCM_CREDENTIALS)

`include/net/scm.h`

```c
struct scm_cookie {
    struct pid        *pid;
    struct scm_fp_list *fp;    // file descriptor list
    struct scm_creds   creds;  // pid, uid, gid snapshot
    u32                secid;  // LSM security ID
};
```

- **`SO_PASSCRED`**: set on receiving socket to auto-attach sender credentials
- **`SO_PEERCRED`**: `getsockopt()` returns peer's credentials from `connect()`/`accept()` time
- Delivered via `cmsg` with `SCM_CREDENTIALS` level

### File Descriptor Passing (SCM_RIGHTS)

```c
struct scm_fp_list {
    short          count;      // number of files
    short          max;        // SCM_MAX_FD = 253
    struct file   *fp[];       // file pointers
};
```

- Sender: builds `cmsg` with `SCM_RIGHTS`, `__scm_send()` calls `fget()` on each fd
- Receiver: `scm_detach_fds()` installs files into receiver's fd table via `receive_fd()`
- Maximum: 253 file descriptors per message

**Garbage Collection**: `net/unix/garbage.c` implements mark-and-sweep GC to detect cycles of AF_UNIX sockets holding each other's fds.

---

## Signals

`kernel/signal.c`, `include/linux/signal_types.h`, `include/linux/sched/signal.h`

### Key Structures

#### `struct sigpending`

```c
struct sigpending {
    struct list_head list;    // list of sigqueue for RT signals
    sigset_t         signal;  // bitmask for standard signals
};
```

Two pending sets per process:
- `task->pending` — thread-private
- `task->signal->shared_pending` — process-wide (all threads)

#### `struct sighand_struct`

```c
struct sighand_struct {
    spinlock_t        siglock;          // protects disposition + pending sets
    refcount_t        count;            // shared via CLONE_SIGHAND
    wait_queue_head_t signalfd_wqh;    // signalfd pollers
    struct k_sigaction action[_NSIG];  // per-signal disposition
};
```

### Delivery Path

1. **`send_signal_locked()`**: acquires `sighand->siglock`, sets pending bit (standard) or allocates `sigqueue` node (RT), calls `complete_signal()` which sets `TIF_SIGPENDING` and wakes target
2. **`get_signal()`** (`kernel/signal.c:2801`): called on return to userspace, dequeues next signal, handles stop/continue/fatal signals

### Standard vs Real-Time Signals

- **Standard (1-31)**: not queued, only bitmask bit set; multiple sends coalesce
- **Real-time (32-64)**: fully queued via `sigqueue` nodes; carry `si_value` payload; priority ordered (lower number first); limited by `RLIMIT_SIGPENDING`

---

## eventfd

`fs/eventfd.c`

### `struct eventfd_ctx` (line 30)

```c
struct eventfd_ctx {
    struct kref           kref;
    wait_queue_head_t     wqh;     // pollers; wqh.lock guards count
    __u64                 count;   // event counter
    unsigned int          flags;   // EFD_SEMAPHORE | EFD_NONBLOCK | EFD_CLOEXEC
    int                   id;
};
```

### Semantics

- **`write()`**: adds value to `count` (blocks/EAGAIN if would overflow `ULLONG_MAX`)
- **`read()`**: returns and resets `count` to 0 (or decrements by 1 with `EFD_SEMAPHORE`)
- **`poll()`**: `EPOLLIN` if `count > 0`, `EPOLLOUT` if `count < ULLONG_MAX`
- io_uring uses `eventfd_signal_mask()` for kernel-internal signaling

---

## signalfd

`fs/signalfd.c`

### Operation

```c
struct signalfd_ctx {
    sigset_t sigmask;  // signals to monitor
};
```

- Uses the process's standard `sigpending` infrastructure (no separate buffer)
- `signalfd_dequeue()`: acquires `sighand->siglock`, calls `dequeue_signal()`
- Returns `struct signalfd_siginfo` (128 bytes fixed layout) per signal
- Signals must be blocked via `sigprocmask()` to prevent normal delivery
- Supports `poll()`/`epoll()` via `signalfd_poll()` on `sighand->signalfd_wqh`

---

## timerfd

`fs/timerfd.c`

### `struct timerfd_ctx` (line 31)

```c
struct timerfd_ctx {
    union {
        struct hrtimer tmr;    // for CLOCK_MONOTONIC/REALTIME
        struct alarm   alarm;  // for ALARM clocks
    } t;
    ktime_t           tintv;         // interval (0 = one-shot)
    wait_queue_head_t wqh;           // pollers
    u64               ticks;         // expirations since last read
    int               clockid;
    short unsigned    expired;
    short unsigned    settime_flags;
};
```

### Syscalls

- **`timerfd_create(clockid, flags)`**: supports `CLOCK_MONOTONIC`, `CLOCK_REALTIME`, `CLOCK_BOOTTIME`, `CLOCK_REALTIME_ALARM`, `CLOCK_BOOTTIME_ALARM`
- **`timerfd_settime(fd, flags, new, old)`**: `TFD_TIMER_ABSTIME`, `TFD_TIMER_CANCEL_ON_SET`
- **`timerfd_gettime(fd, cur)`**: returns time to next expiration

Timer callback (`timerfd_tmrproc`): sets `expired=1`, increments `ticks`, wakes pollers. Re-arming for periodic timers is done lazily on `read()` via `hrtimer_forward_now()`.

**CLOCK_REALTIME discontinuity**: `timerfd_clock_was_set()` iterates `cancel_list` and wakes `TFD_TIMER_CANCEL_ON_SET` timers.

---

## pidfd

`kernel/pid.c`, `kernel/signal.c`

### `pidfd_open(pid, flags)`

`kernel/pid.c:640`

- Looks up `struct pid *` via `find_get_pid()` (not `task_struct` — survives process exit)
- Holds reference to `struct pid`, preventing PID number reuse while fd is open
- Flags: `PIDFD_NONBLOCK`, `PIDFD_THREAD`
- `pidfd_poll()` returns `EPOLLIN` when process exits (zombie or reaped)

### `pidfd_send_signal(pidfd, sig, info, flags)`

`kernel/signal.c:4067`

Target flags: `PIDFD_SIGNAL_THREAD`, `PIDFD_SIGNAL_THREAD_GROUP`, `PIDFD_SIGNAL_PROCESS_GROUP`

### `pidfd_getfd(pidfd, fd, flags)`

`kernel/pid.c:757`

- Duplicates a file descriptor from another process's fd table
- Requires `ptrace_may_access(PTRACE_MODE_ATTACH_REALCREDS)`
- Installs via `receive_fd()` with `O_CLOEXEC`

---

## Futexes

`kernel/futex/` (core.c, waitwake.c, syscalls.c, pi.c, requeue.c)

### Core Structures

#### `struct futex_hash_bucket`

`kernel/futex/futex.h:116`

```c
struct futex_hash_bucket {
    atomic_t           waiters;  // lock-free wake fast path
    spinlock_t         lock;     // protects chain
    struct plist_head  chain;    // priority-sorted waiters
};
```

Global hash table `futex_queues` — size chosen at boot proportional to memory.

#### `struct futex_q`

`kernel/futex/futex.h:172`

```c
struct futex_q {
    struct plist_node    list;       // in hash bucket chain
    struct task_struct  *task;       // waiting task
    spinlock_t          *lock_ptr;  // bucket lock; NULL when woken
    union futex_key      key;       // identifies this futex
    struct futex_pi_state *pi_state;
    u32                  bitset;    // for FUTEX_WAIT_BITSET
};
```

### Futex Keys

- **Private** (`FUTEX_PRIVATE_FLAG`): key = `(mm, virtual_address, 0)` — no page pin needed
- **Shared**: key = `(inode_sequence, page_index, page_offset)` — resolved via `get_futex_key()` which pins page momentarily

### FUTEX_WAIT / FUTEX_WAKE

**WAIT**:
1. Resolve address → `futex_key`
2. Hash → `futex_hash_bucket`
3. `hb->waiters++` with `smp_mb()` (barrier A)
4. Lock bucket, re-read `*uaddr` (check for change), enqueue, unlock, `schedule()`

**WAKE**:
1. Read `hb->waiters` with `smp_mb()` (barrier B) — if 0, return immediately
2. Lock bucket, iterate chain matching keys/bitsets, remove and wake via `wake_q`

Memory barriers A and B ensure the waker cannot miss a waiter.

### PI Futexes

```c
struct futex_pi_state {
    struct list_head     list;       // in owner->pi_state_list
    struct rt_mutex_base pi_mutex;  // priority inheritance mutex
    struct task_struct  *owner;
    union futex_key      key;
};
```

- Userspace `u32` stores owner TID + `FUTEX_WAITERS`/`FUTEX_OWNER_DIED` flags
- Contended path creates `futex_pi_state` with `rt_mutex`, boosting owner priority
- `FUTEX_UNLOCK_PI` hands lock to highest-priority waiter

### futex2: `futex_waitv`

`kernel/futex/syscalls.c:290` — waits on an array of up to 128 futex addresses simultaneously, returns when any one is woken. Each entry can have different size and shared/private flags.

### Robust Futexes

`task_struct->robust_list` — userspace `robust_list_head`. On `do_exit()`, kernel walks the list, marks each held futex with `FUTEX_OWNER_DIED`, wakes one waiter per lock.

---

## Cross-Memory Attach

`mm/process_vm_access.c`

### `process_vm_readv` / `process_vm_writev`

1. Resolve target via `find_get_task_by_vpid(pid)`
2. Check `ptrace_may_access(PTRACE_MODE_ATTACH_REALCREDS)`
3. Get target's `mm_struct` via `mm_access()`
4. For each remote iovec: `mmap_read_lock`, `pin_user_pages_remote()`, copy via `iov_iter`, `unpin_user_pages_dirty_lock()`

Direct page-level copy — no kernel bounce buffer. Requires same permissions as `ptrace()` attach.

---

## memfd

`mm/memfd.c`

### `memfd_create(name, flags)`

`mm/memfd.c:330`

1. Creates a tmpfs file (or hugetlbfs with `MFD_HUGETLB`), prefixed with `"memfd:"`
2. Returns anonymous fd — no filesystem path

**Flags**:
- `MFD_CLOEXEC` — close-on-exec
- `MFD_ALLOW_SEALING` — enables `F_ADD_SEALS`
- `MFD_HUGETLB` — use hugetlbfs
- `MFD_NOEXEC_SEAL` — clear execute bits, set `F_SEAL_EXEC`

### File Sealing

Controlled via `fcntl(fd, F_ADD_SEALS, seals)` and `fcntl(fd, F_GET_SEALS)`:

| Seal | Effect |
|------|--------|
| `F_SEAL_SEAL` | Prevent further seals |
| `F_SEAL_SHRINK` | Prevent `ftruncate()` to smaller size |
| `F_SEAL_GROW` | Prevent `ftruncate()` to larger size |
| `F_SEAL_WRITE` | Prevent `write()` and `mmap(PROT_WRITE\|MAP_SHARED)` |
| `F_SEAL_FUTURE_WRITE` | Prevent new writable mappings |
| `F_SEAL_EXEC` | Prevent making executable |

`F_SEAL_WRITE` uses `memfd_wait_for_pins()` to wait for elevated page ref counts before sealing.

---

## userfaultfd

`fs/userfaultfd.c`, `include/linux/userfaultfd_k.h`

### `struct userfaultfd_ctx` (line 53)

```c
struct userfaultfd_ctx {
    wait_queue_head_t fault_pending_wqh;  // faults waiting to be read
    wait_queue_head_t fault_wqh;          // read but unresolved faults
    wait_queue_head_t fd_wqh;             // poll/epoll on uffd fd
    wait_queue_head_t event_wqh;          // fork/remap/unmap events
    unsigned int      flags;
    unsigned int      features;           // UFFD_FEATURE_* negotiated
    struct mm_struct *mm;                  // monitored mm
};
```

### Creation

- `userfaultfd(flags)` — requires `CAP_SYS_PTRACE` or `sysctl_unprivileged_userfaultfd`
- Or `open("/dev/userfaultfd")` + `ioctl(USERFAULTFD_IOC_NEW, flags)`
- Must call `UFFDIO_API` ioctl first for feature negotiation

### VMA Registration Modes

- `VM_UFFD_MISSING` — trigger on not-present page faults
- `VM_UFFD_WP` — trigger on write-protect faults
- `VM_UFFD_MINOR` — trigger on minor faults (present but not in page table)

### ioctl Interface

| ioctl | Purpose |
|-------|---------|
| `UFFDIO_API` | Feature negotiation |
| `UFFDIO_REGISTER` | Register VMA range |
| `UFFDIO_UNREGISTER` | Unregister VMA range |
| `UFFDIO_COPY` | Copy data into faulting page, wake |
| `UFFDIO_ZEROPAGE` | Map zero page, wake |
| `UFFDIO_MOVE` | Move page between addresses |
| `UFFDIO_CONTINUE` | Continue minor fault |
| `UFFDIO_WRITEPROTECT` | Toggle write protection |
| `UFFDIO_WAKE` | Wake without resolving |
| `UFFDIO_POISON` | Inject SIGBUS |

### Fault Handling Flow

1. `handle_userfault()` called from `do_fault()` when VMA has uffd flags
2. Constructs `uffd_msg` with `UFFD_EVENT_PAGEFAULT`
3. Enqueues on `fault_pending_wqh` — manager reads via `read()`
4. Faulting task sleeps on `fault_wqh`
5. Manager resolves via `UFFDIO_COPY`/`ZEROPAGE`/`CONTINUE` → `wake_userfault()`
6. Returns `VM_FAULT_RETRY` — fault re-enters with page populated

---

## Key Source Files

| Subsystem | Files |
|-----------|-------|
| SysV IPC base | `include/linux/ipc.h`, `ipc/util.c`, `ipc/namespace.c` |
| SysV shared memory | `ipc/shm.c` |
| SysV semaphores | `ipc/sem.c` |
| SysV message queues | `ipc/msg.c` |
| POSIX message queues | `ipc/mqueue.c` |
| Pipes / FIFOs | `fs/pipe.c`, `include/linux/pipe_fs_i.h`, `fs/fifo.c` |
| splice/tee/vmsplice | `fs/splice.c` |
| Unix domain sockets | `net/unix/af_unix.c` |
| SCM (cmsg) | `include/net/scm.h`, `net/core/scm.c` |
| Signals | `kernel/signal.c`, `include/linux/signal_types.h` |
| eventfd | `fs/eventfd.c` |
| signalfd | `fs/signalfd.c` |
| timerfd | `fs/timerfd.c` |
| pidfd | `kernel/pid.c` |
| Futex | `kernel/futex/core.c`, `kernel/futex/futex.h`, `kernel/futex/waitwake.c`, `kernel/futex/pi.c` |
| Cross-memory attach | `mm/process_vm_access.c` |
| memfd / sealing | `mm/memfd.c` |
| userfaultfd | `fs/userfaultfd.c`, `include/linux/userfaultfd_k.h` |
