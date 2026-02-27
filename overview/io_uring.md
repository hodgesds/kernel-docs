# Linux Kernel io_uring Subsystem

## Overview

io_uring is a high-performance asynchronous I/O interface using shared-memory
ring buffers between kernel and userspace. It eliminates per-operation syscall
overhead by batching submissions and completions through lock-free rings, with
optional kernel-side polling to avoid syscalls entirely.

---

## Architecture

### Ring Buffers and Shared Memory

After `io_uring_setup(2)`, userspace mmaps three regions:

| mmap offset | Constant | Contents |
|-------------|----------|----------|
| `0x0` | `IORING_OFF_SQ_RING` | `struct io_rings` (SQ/CQ metadata + CQE array) |
| `0x8000000` | `IORING_OFF_CQ_RING` | Same mapping (single-mmap) |
| `0x10000000` | `IORING_OFF_SQES` | Array of `struct io_uring_sqe` |

With `IORING_FEAT_SINGLE_MMAP` (default), SQ and CQ ring offsets map the same physical memory.

### `struct io_rings` — Shared Control Block

`include/linux/io_uring_types.h:135`

```c
struct io_rings {
    struct io_uring  sq, cq;       // head/tail pairs
    u32  sq_ring_mask;             // sq_entries - 1
    u32  cq_ring_mask;             // cq_entries - 1
    u32  sq_ring_entries;          // power-of-2 SQ capacity
    u32  cq_ring_entries;          // power-of-2 CQ capacity
    u32  sq_dropped;               // kernel: dropped SQEs
    atomic_t sq_flags;             // IORING_SQ_NEED_WAKEUP etc.
    u32  cq_flags;                 // IORING_CQ_EVENTFD_DISABLED
    u32  cq_overflow;              // overflow counter
    struct io_uring_cqe cqes[];    // CQE ring buffer
    // sq_array follows after cqes (unless NO_SQARRAY)
};
```

**Ownership rules**:
- Kernel writes `sq.head` (consumption), userspace writes `sq.tail` (production)
- Kernel writes `cq.tail` (production), userspace writes `cq.head` (consumption)

**SQ array indirection**: By default, `sq_array[sq.head & mask]` indexes into
the SQE array, allowing userspace to reorder submission.
`IORING_SETUP_NO_SQARRAY` eliminates this indirection for sequential access.

`sq_flags` bits (written by kernel):
- `IORING_SQ_NEED_WAKEUP` — SQPOLL thread needs waking
- `IORING_SQ_CQ_OVERFLOW` — CQ ring overflowed
- `IORING_SQ_TASKRUN` — task should enter kernel for task_work

### `struct io_ring_ctx` — The Ring Context

`include/linux/io_uring_types.h:241` — main kernel-side object, cacheline-aligned:

**Submission state**: `uring_lock` (mutex), `sq_array`, `sq_sqes`, `cached_sq_head`, `file_table`, `buf_table`, `submit_state`, `cancel_table`

**Completion state**: `cqe_cached`/`cqe_sentinel` (CQE slot pointers), `cached_cq_tail`, `io_ev_fd` (eventfd)

**Task-work**: `work_llist` (DEFER_TASKRUN local list), `cq_wait_nr`, `cq_wait` (waiter queue)

**Resources**: `sq_data` (SQPOLL), `hash_map` (hashed writes), `personalities` (registered creds)

---

## Syscalls

### `io_uring_setup(entries, params)`

`io_uring/io_uring.c:3872`

- Allocates `struct io_ring_ctx`
- Allocates shared memory for rings + CQE array + sq_array
- Allocates SQE array separately
- Optionally spawns SQPOLL thread
- Returns anonymous fd; writes back `io_uring_params` with ring offsets

**Setup flags**:

| Flag | Effect |
|------|--------|
| `IORING_SETUP_IOPOLL` | Polling completion, no IRQ (requires O_DIRECT) |
| `IORING_SETUP_SQPOLL` | Kernel thread polls SQ |
| `IORING_SETUP_SQ_AFF` | Pin SQPOLL to specified CPU |
| `IORING_SETUP_CQSIZE` | App specifies CQ size |
| `IORING_SETUP_CLAMP` | Clamp ring sizes rather than error |
| `IORING_SETUP_ATTACH_WQ` | Share io-wq with existing ring |
| `IORING_SETUP_SUBMIT_ALL` | Continue submitting after per-SQE errors |
| `IORING_SETUP_COOP_TASKRUN` | Defer task_work to natural kernel transitions |
| `IORING_SETUP_TASKRUN_FLAG` | Set `IORING_SQ_TASKRUN` when tw pending |
| `IORING_SETUP_SQE128` | 128-byte SQEs |
| `IORING_SETUP_CQE32` | 32-byte CQEs |
| `IORING_SETUP_SINGLE_ISSUER` | Only one task submits |
| `IORING_SETUP_DEFER_TASKRUN` | Defer tw to io_uring_enter with GETEVENTS |
| `IORING_SETUP_NO_MMAP` | Userspace provides ring memory |
| `IORING_SETUP_REGISTERED_FD_ONLY` | Return registered fd index |
| `IORING_SETUP_NO_SQARRAY` | Skip SQ indirection array |
| `IORING_SETUP_HYBRID_IOPOLL` | Hybrid spin+sleep iopoll |

### `io_uring_enter(fd, to_submit, min_complete, flags, argp, argsz)`

`io_uring/io_uring.c:3359`

1. Resolve fd (registered ring fast path or `fget()`)
2. If SQPOLL: wake SQ thread or wait for SQ room
3. If `to_submit > 0`: take `uring_lock`, call `io_submit_sqes()`
4. If `IORING_ENTER_GETEVENTS`: wait for completions via `io_cqring_wait()` or `io_iopoll_check()`

### `io_uring_register(fd, opcode, arg, nr_args)`

`io_uring/register.c` — resource management under `uring_lock`:

| Opcode | Purpose |
|--------|---------|
| `REGISTER_BUFFERS` | Pre-register fixed buffers |
| `REGISTER_FILES` | Pre-register file descriptors |
| `REGISTER_FILES_UPDATE` | Update file slots |
| `REGISTER_EVENTFD` | Register eventfd for notifications |
| `REGISTER_PERSONALITY` | Save creds for later use |
| `REGISTER_IOWQ_AFF` | Set io-wq CPU affinity |
| `REGISTER_IOWQ_MAX_WORKERS` | Set max worker counts |
| `REGISTER_RING_FDS` | Register ring fds |
| `REGISTER_PBUF_RING` | Register provided buffer ring |
| `REGISTER_SYNC_CANCEL` | Synchronous cancellation |
| `REGISTER_FILE_ALLOC_RANGE` | Set direct fd alloc range |
| `REGISTER_CLOCK` | Override clock source |
| `REGISTER_RESIZE_RINGS` | Resize CQ ring |

---

## SQE Format

`struct io_uring_sqe` — 64 bytes (128 with `IORING_SETUP_SQE128`):

`include/uapi/linux/io_uring.h:30`

```c
struct io_uring_sqe {
    __u8   opcode;         // IORING_OP_*
    __u8   flags;          // IOSQE_*
    __u16  ioprio;         // I/O priority
    __s32  fd;             // file descriptor (or fixed index)
    __u64  off;            // file offset (or addr2/cmd_op)
    __u64  addr;           // buffer pointer (or splice_off_in)
    __u32  len;            // buffer length / iovec count
    __u32  op_flags;       // operation-specific flags union
    __u64  user_data;      // copied verbatim to CQE
    __u16  buf_index;      // fixed buffer index / buf_group
    __u16  personality;    // registered personality index
    __s32  splice_fd_in;   // SPLICE input fd / file_index
    __u64  addr3;          // tertiary address / cmd[] for SQE128
};
```

### SQE Flags (`IOSQE_*`)

| Flag | Bit | Meaning |
|------|-----|---------|
| `IOSQE_FIXED_FILE` | 0 | `fd` is a fixed file index |
| `IOSQE_IO_DRAIN` | 1 | Wait for all prior requests |
| `IOSQE_IO_LINK` | 2 | Soft link: next runs only if this succeeds |
| `IOSQE_IO_HARDLINK` | 3 | Hard link: next runs regardless |
| `IOSQE_ASYNC` | 4 | Always issue via io-wq |
| `IOSQE_BUFFER_SELECT` | 5 | Select buffer from provided group |
| `IOSQE_CQE_SKIP_SUCCESS` | 6 | Suppress CQE on success |

---

## CQE Format

`struct io_uring_cqe` — 16 bytes (32 with `CQE32`):

```c
struct io_uring_cqe {
    __u64  user_data;   // from sqe->user_data
    __s32  res;         // result: >=0 success, <0 -errno
    __u32  flags;       // IORING_CQE_F_*
    __u64  big_cqe[];   // CQE32: 16 extra bytes
};
```

CQE flags:

| Flag | Meaning |
|------|---------|
| `IORING_CQE_F_BUFFER` | Buffer selected; upper 16 bits = buffer ID |
| `IORING_CQE_F_MORE` | More CQEs from this SQE (multishot) |
| `IORING_CQE_F_SOCK_NONEMPTY` | More data on socket after recv |
| `IORING_CQE_F_NOTIF` | Zerocopy send notification |
| `IORING_CQE_F_BUF_MORE` | Buffer partially consumed |

---

## Submission Path

### `io_submit_sqes(ctx, nr)` — Batch Submission

`io_uring/io_uring.c:2336` — under `uring_lock`:

```
io_submit_sqes(ctx, nr)
  io_submit_state_start()           // set up blk_plug
  loop:
    io_alloc_req(ctx, &req)         // from slab cache, batch alloc of 8
    io_get_sqe(ctx, &sqe)           // read SQE from shared memory
    io_submit_sqe(ctx, req, sqe)    // prep + issue
  io_submit_state_end()             // flush completions, blk_plug
  io_commit_sqring(ctx)             // smp_store_release(sq.head)
```

### Single SQE: `io_submit_sqe()`

```
io_init_req(ctx, req, sqe)          // populate from sqe fields
  → validate opcode, flags
  → def->prep(req, sqe)            // opcode-specific prep
io_queue_sqe(req)                   // attempt inline issue
  → io_issue_sqe(req, NONBLOCK)    // dispatch to opcode handler
    → def->issue(req, flags)       // THE OPCODE HANDLER
    → on success: io_req_complete_defer()
    → on -EAGAIN: io_queue_async()
```

### Async Punt: `io_queue_async()`

When inline issue returns `-EAGAIN`:

1. Copy SQE data for async retention
2. Try `io_arm_poll_handler()` — fast-poll on file's waitqueue
   - `IO_APOLL_OK`: poll armed, wait for readiness
   - `IO_APOLL_READY`: already ready, re-queue
   - `IO_APOLL_ABORTED`: not pollable, fall through
3. If poll fails: `io_queue_iowq()` — dispatch to io-wq worker

### Issue Return Values

- `IOU_COMPLETE` (0) — done, post CQE
- `IOU_ISSUE_SKIP_COMPLETE` — submitted async, completion via callback
- `IOU_RETRY` (-EAGAIN) — would block, retry or poll
- `IOU_REQUEUE` — requeue via task_work

---

## Completion Path

### Deferred Completion (Normal)

Completions accumulate in `submit_state.compl_reqs`, flushed by `__io_submit_flush_completions()`:

1. Acquire `completion_lock` (if needed)
2. For each request: `io_fill_cqe_req()` — write to CQE ring
3. `io_commit_cqring()` — `smp_store_release(&rings->cq.tail, ...)`
4. `io_cqring_wake()` — wake pollers
5. `io_free_batch_list()` — release resources

### CQE Overflow

When CQ ring is full, `io_cqring_event_overflow()` allocates `io_overflow_cqe`
on heap, appends to `ctx->cq_overflow_list`, sets `IORING_SQ_CQ_OVERFLOW`.
Drained when space becomes available.

### Task Work

For completions from non-submitter contexts (IRQ, foreign task):

**Without `DEFER_TASKRUN`**: `task_work_add()` → `TIF_NOTIFY_SIGNAL` → runs on return to userspace

**With `DEFER_TASKRUN`**: lock-free push to `ctx->work_llist` → processed inside `io_uring_enter()` with `GETEVENTS`

### IOPOLL Completion

No interrupts. Requests placed on `ctx->iopoll_list`. Kernel polls via
`blk_poll()` / `io_do_iopoll()` in the enter syscall or SQPOLL loop.

---

## Key Opcodes

### I/O Operations

| Opcode | Prep/Issue Files | Description |
|--------|-----------------|-------------|
| `NOP` | `nop.c` | No-op, returns 0 |
| `READ` / `WRITE` | `rw.c` | Single buffer read/write |
| `READV` / `WRITEV` | `rw.c` | Vectored read/write |
| `READ_FIXED` / `WRITE_FIXED` | `rw.c` | Fixed (pre-registered) buffer I/O |
| `READ_MULTISHOT` | `rw.c` | Repeated reads with buffer selection |

### File Operations

| Opcode | File | Description |
|--------|------|-------------|
| `OPENAT` / `OPENAT2` | `openclose.c` | Open file, optional direct fd |
| `CLOSE` | `openclose.c` | Close fd or fixed slot |
| `FIXED_FD_INSTALL` | `openclose.c` | Convert fixed fd to real fd |
| `SPLICE` / `TEE` | `splice.c` | Pipe splice/tee |

### Network Operations

| Opcode | File | Description |
|--------|------|-------------|
| `ACCEPT` | `net.c` | Accept connection (multishot capable) |
| `CONNECT` | `net.c` | Connect socket |
| `SEND` / `RECV` | `net.c` | Send/receive data (multishot recv) |
| `SENDMSG` / `RECVMSG` | `net.c` | Send/receive with msghdr |
| `SEND_ZC` / `SENDMSG_ZC` | `net.c` | Zero-copy send (two CQEs: completion + notification) |
| `SOCKET` | `net.c` | Create socket |
| `SHUTDOWN` | `net.c` | Shutdown socket |

### Polling and Timing

| Opcode | File | Description |
|--------|------|-------------|
| `POLL_ADD` | `poll.c` | Add poll monitor (multishot capable) |
| `POLL_REMOVE` | `poll.c` | Remove/update poll |
| `TIMEOUT` | `timeout.c` | Timer (multishot capable) |
| `LINK_TIMEOUT` | `timeout.c` | Timeout for preceding linked SQE |

### Synchronization

| Opcode | File | Description |
|--------|------|-------------|
| `FUTEX_WAIT` / `FUTEX_WAKE` | `futex.c` | Futex operations |
| `FUTEX_WAITV` | `futex.c` | Wait on multiple futexes |
| `WAITID` | `waitid.c` | Wait for process state change |

### Management

| Opcode | File | Description |
|--------|------|-------------|
| `ASYNC_CANCEL` | `cancel.c` | Cancel pending request |
| `PROVIDE_BUFFERS` | `kbuf.c` | Legacy buffer provision |
| `REMOVE_BUFFERS` | `kbuf.c` | Remove provided buffers |
| `MSG_RING` | `msg_ring.c` | Cross-ring messaging |

---

## Fixed Files and Buffers

### Fixed Files

`io_uring_types.h:68` — `struct io_file_table`:

```c
struct io_file_table {
    struct io_rsrc_data data;   // .nr, .nodes array
    unsigned long *bitmap;      // allocation bitmap
    unsigned int alloc_hint;
};
```

Each slot is an `io_rsrc_node` containing a tagged file pointer with `FFS_NOWAIT`/`FFS_ISREG` flags in low bits.

**Key benefit**: Avoids `fget()`/`fput()` per request. The reference is held
for the table entry's lifetime. Fast-path: `io_file_get_fixed()` does direct
    array lookup + reference count increment on the rsrc node.

Direct fd auto-allocation: `IORING_FILE_INDEX_ALLOC` in `sqe->file_index` scans bitmap for free slot.

### Fixed Buffers

Each registered buffer becomes an `io_rsrc_node` with `io_mapped_ubuf`:

```c
struct io_mapped_ubuf {
    u64          ubuf;       // user buffer address
    unsigned int len;
    unsigned int nr_bvecs;
    struct bio_vec bvec[];   // pinned page array for DMA
};
```

Pages pinned via `pin_user_pages_fast()`. Enables zero-copy I/O to pre-registered memory.

---

## io-wq: Async Worker Thread Pool

`io_uring/io-wq.c`

### Structure

```c
struct io_wq {
    io_wq_work_fn *do_work;     // = io_wq_submit_work()
    struct io_wq_acct acct[2];  // [BOUND, UNBOUND]
    cpumask_var_t cpu_mask;
};

struct io_wq_acct {
    unsigned nr_workers, max_workers;
    atomic_t nr_running;
    struct hlist_nulls_head free_list;  // idle workers
    struct io_wq_work_list work_list;   // pending work
};
```

### Bounded vs. Unbound Workers

- **Bounded**: CPU-affine, for regular file I/O. Hashed work on same inode serialized.
- **Unbound**: For non-regular files (sockets, pipes), no CPU affinity.

Selection via `IO_WQ_WORK_UNBOUND` flag on the work item.

### Worker Lifecycle

Workers created via `create_io_thread()`, run `io_wqe_worker()` loop. Idle for
5 seconds (`WORKER_IDLE_TIMEOUT`), then exit. Identified by `PF_IO_WORKER` in
`task->flags`.

Work dispatch: `io_wq_enqueue()` → wake idle worker or spawn new (up to `max_workers`).

---

## Linked SQEs

### Chain Assembly

In `io_submit_sqe()`, `IOSQE_IO_LINK` or `IOSQE_IO_HARDLINK` builds a chain via
`req->link` pointers. Chain is flushed when a non-link SQE is encountered.

### Execution

Head issued first. On completion, `io_req_find_next()` retrieves `req->link` and queues it.

**Soft link** (`IOSQE_IO_LINK`): if head fails, next receives `-ECANCELED`, rest of chain cancelled.

**Hard link** (`IOSQE_IO_HARDLINK`): next runs regardless of head's result.

### Link Timeout

`IORING_OP_LINK_TIMEOUT` must follow target SQE in chain. Arms an hrtimer that
cancels the preceding request if it doesn't complete in time. Disarmed on
normal completion.

---

## Multishot Operations

Multishot requests generate multiple CQEs from a single SQE submission. Each
CQE has `IORING_CQE_F_MORE` until the operation terminates.

| Operation | Trigger | Description |
|-----------|---------|-------------|
| `ACCEPT` + `IORING_ACCEPT_MULTISHOT` | `sqe->ioprio` | CQE per accepted connection |
| `RECV` + `IORING_RECV_MULTISHOT` | `sqe->ioprio` | CQE per received message |
| `POLL_ADD` + `IORING_POLL_ADD_MULTI` | `sqe->len` | CQE per poll event |
| `READ_MULTISHOT` | opcode | CQE per read (requires buffer select) |
| `TIMEOUT` + `IORING_TIMEOUT_MULTISHOT` | `sqe->timeout_flags` | CQE per timer expiry |

Multishot disallowed from io-wq workers — re-arms via poll or strips flag.

---

## SQPOLL Mode

### `struct io_sq_data`

`io_uring/sqpoll.h`

```c
struct io_sq_data {
    struct list_head ctx_list;      // rings using this sqd
    struct task_struct *thread;     // iou-sqp-<pid> kthread
    unsigned sq_thread_idle;        // idle timeout (jiffies)
    int sq_cpu;                     // pinned CPU (-1 = any)
};
```

Multiple rings share a single `io_sq_data` via `IORING_SETUP_ATTACH_WQ`.

### SQPOLL Loop

`io_sq_thread()` at `sqpoll.c:260`:

1. For each ring: `io_submit_sqes()` (capped at 8 per ring for fairness)
2. Process task_work (up to 32 items)
3. NAPI busy poll if configured
4. If no work: set `IORING_SQ_NEED_WAKEUP`, sleep
5. On wake or new entries: clear flag, resume

Userspace must check `IORING_SQ_NEED_WAKEUP` after writing SQ tail (with
`smp_mb()` barrier) and call `io_uring_enter(IORING_ENTER_SQ_WAKEUP)` if set.

---

## Buffer Selection

### Legacy: `IORING_OP_PROVIDE_BUFFERS`

Buffers registered per group. Each `struct io_buffer` has a buffer ID. At issue
time with `IOSQE_BUFFER_SELECT`, a buffer is popped from the group. Buffer ID
reported in `cqe->flags >> 16`.

### Buffer Rings: `IORING_REGISTER_PBUF_RING`

Shared-memory ring of `struct io_uring_buf`:

```c
struct io_uring_buf {
    __u64  addr;   // buffer address
    __u32  len;    // buffer length
    __u16  bid;    // buffer ID
};

struct io_uring_buf_ring {
    union {
        struct { ...; __u16 tail; };
        struct io_uring_buf bufs[];
    };
};
```

Userspace advances `tail` to add buffers; kernel reads from `head`. The `tail`
field overlaps `bufs[0].resv` — zero extra space for the ring header.

`IOBL_INC` flag: incremental consumption — kernel tracks sub-offset, partially
consuming large buffers. `IORING_CQE_F_BUF_MORE` set when buffer not fully
consumed.

---

## Cancellation

### `IORING_OP_ASYNC_CANCEL`

`io_uring/cancel.c`

`io_try_cancel()` tries in order:
1. io-wq cancel via `io_wq_cancel_cb()`
2. Poll cancel
3. Waitid cancel
4. Futex cancel
5. Timeout cancel

Cancel flags:
- `IORING_ASYNC_CANCEL_ALL` — cancel all matching
- `IORING_ASYNC_CANCEL_FD` — match by fd
- `IORING_ASYNC_CANCEL_ANY` — match any request
- `IORING_ASYNC_CANCEL_OP` — match by opcode
- `IORING_ASYNC_CANCEL_FD_FIXED` — fd is fixed index

### `IORING_REGISTER_SYNC_CANCEL`

Synchronous cancel via register path with timeout.

---

## Registered Ring FDs

`IORING_REGISTER_RING_FDS` stores ring's `struct file *` in
`tctx->registered_rings[]` (16 slots). `IORING_ENTER_REGISTERED_RING` skips
`fget()` — direct array lookup.

`IORING_SETUP_REGISTERED_FD_ONLY`: setup returns registered index (not fd). Combined with `NO_MMAP` + `NO_SQARRAY` for minimum overhead.

---

## Memory Barriers

- **Userspace writing SQ tail**: `smp_store_release()` to order SQE stores before tail update
- **Userspace reading CQ tail**: `smp_load_acquire()` to order tail read before CQE reads
- **Kernel committing CQ**: `smp_store_release(&rings->cq.tail, ...)` in `io_commit_cqring()`
- **Kernel reading SQ**: `smp_load_acquire(&rings->sq.tail)` in `io_sqring_entries()`
- **SQPOLL users**: `smp_mb()` between writing SQ tail and checking `IORING_SQ_NEED_WAKEUP`

---

## `struct io_kiocb` — The Request Object

`include/linux/io_uring_types.h:644` — the per-request structure:

```c
struct io_kiocb {
    struct file       *file;
    u8                 opcode;
    io_req_flags_t     flags;        // REQ_F_*
    struct io_cqe      cqe;          // user_data, res, flags
    struct io_ring_ctx *ctx;
    struct io_uring_task *tctx;
    struct io_rsrc_node  *file_node; // fixed file rsrc node
    atomic_t           refs;
    struct io_kiocb   *link;         // next in link chain
    struct async_poll *apoll;        // fast-poll state
    void              *async_data;   // per-opcode async data
    struct io_wq_work  work;         // io-wq linkage
    struct io_cmd_data cmd;          // 56 bytes opcode-private overlay
};
```

---

## Opcode Dispatch Tables

`io_uring/opdef.c` — `io_issue_defs[]` (hot path) and `io_cold_defs[]` (cleanup/failure):

```c
struct io_issue_def {
    unsigned needs_file : 1;
    unsigned plug : 1;           // eligible for blk_plug
    unsigned iopoll : 1;         // supports IOPOLL
    unsigned buffer_select : 1;  // supports buffer selection
    unsigned hash_reg_file : 1;  // serialize regular file writes
    unsigned short async_size;   // size of async_data
    int (*issue)(struct io_kiocb *, unsigned int);
    int (*prep)(struct io_kiocb *, const struct io_uring_sqe *);
};
```

---

## Locking Summary

| Lock | Type | Protects |
|------|------|----------|
| `ctx->uring_lock` | mutex | Submission, registration, deferred tw, iopoll |
| `ctx->completion_lock` | spinlock | CQ writes, overflow list |
| `ctx->timeout_lock` | raw_spinlock | Timeout lists |
| `sqd->lock` | mutex | SQPOLL park/stop |

"Lockless CQ" with `DEFER_TASKRUN`: all CQE posts from submitter task under
`uring_lock`, no `completion_lock` needed.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `io_uring/io_uring.c` | Core: setup, enter, submission/completion |
| `io_uring/io_uring.h` | Internal declarations, CQE helpers |
| `include/linux/io_uring_types.h` | Major structs: io_ring_ctx, io_kiocb, io_rings |
| `include/uapi/linux/io_uring.h` | UAPI: sqe, cqe, opcodes, flags |
| `io_uring/io-wq.c` | Async worker thread pool |
| `io_uring/sqpoll.c` | SQPOLL kernel thread |
| `io_uring/opdef.c` | Opcode dispatch tables |
| `io_uring/rsrc.c` | Fixed file/buffer management |
| `io_uring/kbuf.c` | Provided buffers and buffer rings |
| `io_uring/poll.c` | POLL_ADD/REMOVE, fast-poll |
| `io_uring/rw.c` | Read/write operations |
| `io_uring/net.c` | Network operations |
| `io_uring/timeout.c` | Timeout operations |
| `io_uring/cancel.c` | Cancellation |
| `io_uring/openclose.c` | Open/close/fixed_fd_install |
| `io_uring/tctx.c` | Per-task context, registered ring fds |
| `io_uring/register.c` | io_uring_register dispatch |
| `io_uring/filetable.c` | Fixed file table management |
| `io_uring/memmap.c` | Ring mmap, page pinning |
| `io_uring/futex.c` | Futex operations |
| `io_uring/splice.c` | Splice/tee operations |
| `io_uring/msg_ring.c` | Cross-ring messaging |
| `io_uring/notif.c` | Zero-copy send notifications |
| `io_uring/napi.c` | NAPI busy-poll integration |
