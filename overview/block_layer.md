# Linux Kernel Block Layer Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Block Layer Architecture](#block-layer-architecture)
3. [Bio (Block I/O)](#bio-block-io)
4. [Request and Request Queue](#request-and-request-queue)
5. [blk-mq (Multi-Queue Block Layer)](#blk-mq-multi-queue-block-layer)
6. [I/O Schedulers](#io-schedulers)
7. [Gendisk and Block Device](#gendisk-and-block-device)
8. [Device Mapper](#device-mapper)
9. [I/O Path](#io-path)

---

## Introduction

The block layer is the kernel subsystem responsible for managing block devices
(hard drives, SSDs, NVMe, etc.) and handling I/O requests. It provides:

- Abstraction layer between filesystems and device drivers
- Request queuing and scheduling
- I/O merging and optimization
- Plugging for batching requests
- Multi-queue support for modern NVMe devices

### Block vs Character Devices

```
┌─────────────────────────────────────────────────────────────┐
│              Block vs Character Devices                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Block Device                    Character Device           │
│  ────────────                    ────────────────           │
│  - Random access                - Sequential access         │
│  - Fixed-size blocks            - Byte stream               │
│  - Uses block layer             - Direct to driver          │
│  - Buffered I/O (page cache)    - Usually unbuffered        │
│  - Examples: HDD, SSD, NVMe     - Examples: tty, mouse      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  /dev/sda1   /dev/nvme0n1   ← Block devices         │    │
│  │  /dev/tty0   /dev/input/    ← Character devices     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Block Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Block Layer Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Filesystem (ext4, xfs, btrfs)          │    │
│  │   submit_bio(), block_read_full_folio()             │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Page Cache                             │    │
│  │   Caches file data, handles read/write buffering    │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          ▼                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Block Layer                             │   │
│  │                                                      │   │
│  │   ┌─────────────────────────────────────────────┐    │   │
│  │   │   Bio Layer                                 │    │   │
│  │   │   - Describes I/O operation                 │    │   │
│  │   │   - submit_bio() entry point                │    │   │
│  │   └────────────────────┬────────────────────────┘    │   │
│  │                        │                             │   │
│  │   ┌────────────────────▼────────────────────────┐    │   │
│  │   │   blk-mq (Multi-Queue)                      │    │   │
│  │   │   - Software staging queues                 │    │   │
│  │   │   - Hardware dispatch queues                │    │   │
│  │   │   - I/O scheduler integration               │    │   │
│  │   └────────────────────┬────────────────────────┘    │   │
│  │                        │                             │   │
│  │   ┌────────────────────▼────────────────────────┐    │   │
│  │   │   I/O Scheduler                             │    │   │
│  │   │   - mq-deadline, bfq, kyber, none           │    │   │
│  │   └────────────────────┬────────────────────────┘    │   │
│  │                        │                             │   │
│  └────────────────────────┼─────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Device Driver                          │    │
│  │   NVMe, SCSI, virtio-blk, loop, etc.                │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Hardware Device                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Bio (Block I/O)

The `bio` structure is the fundamental unit of I/O in the block layer. It
describes a single I/O operation.

### Core Data Structure

The `bio` structure is the primary container for block I/O requests in the
Linux kernel. It describes a single I/O operation including the target device,
operation type (read/write/discard), and the memory segments involved. Each bio
contains an array of `bio_vec` structures that point to the actual data pages,
along with an iterator (`bvec_iter`) to track progress through those segments.
Bios are typically allocated from a `bio_set` memory pool, submitted via
`submit_bio()`, and completed asynchronously through the `bi_end_io` callback.
The key fields include `bi_bdev` (target device), `bi_opf` (operation and
flags), `bi_iter` (current position), and `bi_io_vec` (memory segments).

```c
/* include/linux/bio.h */
struct bio {
    struct bio              *bi_next;       /* Next bio in chain */
    struct block_device     *bi_bdev;       /* Target device */
    blk_opf_t               bi_opf;         /* Operation and flags */

    unsigned short          bi_flags;       /* BIO_* flags */
    unsigned short          bi_ioprio;      /* I/O priority */
    blk_status_t            bi_status;      /* Completion status */
    atomic_t                __bi_remaining; /* Pending completions */

    struct bvec_iter        bi_iter;        /* Current position */

    bio_end_io_t            *bi_end_io;     /* Completion callback */
    void                    *bi_private;    /* Private data */

    unsigned short          bi_vcnt;        /* Number of bio_vecs */
    unsigned short          bi_max_vecs;    /* Max bio_vecs */

    atomic_t                __bi_cnt;       /* Reference count */

    struct bio_vec          *bi_io_vec;     /* Actual vec list */
    struct bio_set          *bi_pool;       /* Pool this came from */

    /* Inline bio_vecs for small I/O */
    struct bio_vec          bi_inline_vecs[];
};
```

The `bio_vec` structure represents a single contiguous memory segment within a
bio. Each bio_vec points to a specific page in memory and specifies the offset
within that page and the length of data. Multiple bio_vecs allow a bio to
describe scatter-gather I/O operations where data is spread across
non-contiguous memory locations.

```c
/* Bio vector - describes a segment of memory */
struct bio_vec {
    struct page     *bv_page;       /* Page containing data */
    unsigned int    bv_len;         /* Length of segment */
    unsigned int    bv_offset;      /* Offset within page */
};
```

The `bvec_iter` structure tracks the current position when iterating through a
bio's memory segments. It maintains the current sector being processed,
remaining bytes, current index in the bio_vec array, and bytes completed in the
current vector. This iterator allows efficient traversal of bio data during I/O
processing, especially when bios are split or partially completed.

```c
/* Iterator for walking bio segments */
struct bvec_iter {
    sector_t        bi_sector;      /* Current sector */
    unsigned int    bi_size;        /* Remaining bytes */
    unsigned int    bi_idx;         /* Current index in bi_io_vec */
    unsigned int    bi_bvec_done;   /* Bytes done in current vec */
};
```

### Bio Operations

```c
/* Operation types (in bi_opf) */
enum req_op {
    REQ_OP_READ,            /* Read from device */
    REQ_OP_WRITE,           /* Write to device */
    REQ_OP_FLUSH,           /* Flush volatile cache */
    REQ_OP_DISCARD,         /* Trim/discard blocks */
    REQ_OP_SECURE_ERASE,    /* Secure erase */
    REQ_OP_WRITE_ZEROES,    /* Write zeros */
    REQ_OP_ZONE_OPEN,       /* Zoned device operations */
    REQ_OP_ZONE_CLOSE,
    REQ_OP_ZONE_FINISH,
    REQ_OP_ZONE_APPEND,
    REQ_OP_ZONE_RESET,
    REQ_OP_ZONE_RESET_ALL,
    REQ_OP_DRV_IN,          /* Driver-specific input */
    REQ_OP_DRV_OUT,         /* Driver-specific output */
};

/* Flags (combined with op in bi_opf) */
#define REQ_SYNC        (1ULL << __REQ_SYNC)    /* Synchronous I/O */
#define REQ_META        (1ULL << __REQ_META)    /* Metadata */
#define REQ_PRIO        (1ULL << __REQ_PRIO)    /* High priority */
#define REQ_FUA         (1ULL << __REQ_FUA)     /* Force Unit Access */
#define REQ_PREFLUSH    (1ULL << __REQ_PREFLUSH) /* Flush before */
#define REQ_RAHEAD      (1ULL << __REQ_RAHEAD)  /* Read-ahead */
```

### Bio Lifecycle

```c
/* Allocate a bio */
struct bio *bio = bio_alloc(bdev, nr_vecs, opf, GFP_KERNEL);

/* Add pages to bio */
bio_add_page(bio, page, len, offset);

/* Set completion handler */
bio->bi_end_io = my_bio_endio;
bio->bi_private = my_private_data;

/* Submit bio */
submit_bio(bio);

/* Completion callback */
static void my_bio_endio(struct bio *bio)
{
    if (bio->bi_status)
        handle_error(bio->bi_status);
    else
        process_data(bio);

    bio_put(bio);  /* Release reference */
}
```

### Bio Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Bio Lifecycle                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Allocation                                              │
│     bio = bio_alloc(bdev, nvecs, op, gfp)                   │
│         │                                                   │
│         ▼                                                   │
│  2. Setup                                                   │
│     bio_add_page(bio, page, len, off)                       │
│     bio->bi_iter.bi_sector = sector                         │
│     bio->bi_end_io = callback                               │
│         │                                                   │
│         ▼                                                   │
│  3. Submit                                                  │
│     submit_bio(bio)                                         │
│         │                                                   │
│         ├─── bio enters block layer                         │
│         │    - Integrity check                              │
│         │    - Merge with other bios                        │
│         │    - Add to request                               │
│         │                                                   │
│         ▼                                                   │
│  4. Dispatch                                                │
│     Request sent to device driver                           │
│         │                                                   │
│         ▼                                                   │
│  5. Hardware processing                                     │
│     Device performs I/O                                     │
│         │                                                   │
│         ▼                                                   │
│  6. Completion                                              │
│     bio->bi_end_io(bio) called                              │
│     bio_put(bio) to free                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Request and Request Queue

Requests group multiple bios for efficient I/O processing.

### Request Structure

The `request` structure represents a block I/O request that groups one or more
bios together for efficient processing by the device driver. Requests are the
primary unit of work that flows through the blk-mq layer and I/O schedulers.
Each request is associated with a hardware queue context (`mq_hctx`) and
software queue context (`mq_ctx`), and is identified by tags used for
completion tracking. The structure contains linked bios (`bio` and `biotail`),
timing information for latency tracking, and driver-specific completion
callbacks. Requests are allocated from tag pools and recycled after completion.

```c
/* include/linux/blk-mq.h */
struct request {
    struct request_queue    *q;             /* Request queue */
    struct blk_mq_ctx       *mq_ctx;        /* Software queue context */
    struct blk_mq_hw_ctx    *mq_hctx;       /* Hardware queue context */

    blk_opf_t               cmd_flags;      /* Operation and flags */
    req_flags_t             rq_flags;       /* Request flags */

    int                     tag;            /* Tag for hardware queue */
    int                     internal_tag;   /* Internal driver tag */

    sector_t                __sector;       /* Starting sector */
    unsigned int            __data_len;     /* Total data length */

    struct bio              *bio;           /* First bio */
    struct bio              *biotail;       /* Last bio */

    struct list_head        queuelist;      /* Queue linkage */
    union {
        struct hlist_node   hash;
        struct llist_node   ipi_list;
    };

    /* Time tracking */
    u64                     start_time_ns;
    u64                     io_start_time_ns;

    unsigned short          stats_sectors;
    unsigned short          nr_phys_segments;

    /* For driver use */
    void                    *end_io_data;
    rq_end_io_fn            *end_io;

    /* ... more fields ... */
};
```

### Request Queue

The `request_queue` structure is the central control structure for a block
device's I/O processing. It manages the flow of requests from submission to
completion, holding references to the I/O scheduler (`elevator`),
quality-of-service policies (`rq_qos`), and device capabilities (`limits`).
Each block device has an associated request queue that coordinates between the
blk-mq infrastructure and the device driver. The queue maintains configuration
such as the number of hardware queues, device parameters, and various
operational flags.

```c
/* include/linux/blkdev.h */
struct request_queue {
    struct request          *last_merge;    /* Last merged request */
    struct elevator_queue   *elevator;      /* I/O scheduler */

    struct blk_queue_stats  *stats;         /* I/O statistics */
    struct rq_qos           *rq_qos;        /* QoS policies */

    /* Queue limits */
    struct queue_limits     limits;

    /* Device parameters */
    unsigned int            nr_hw_queues;   /* Hardware queue count */
    unsigned int            nr_zones;       /* For zoned devices */

    /* Flags */
    unsigned long           queue_flags;

    /* blk-mq specific */
    struct blk_mq_tag_set   *tag_set;
    struct list_head        tag_set_list;

    /* ... more fields ... */
};
```

The `queue_limits` structure defines the physical and logical constraints of a
block device that the block layer must respect when building I/O requests. It
specifies maximum sector counts, segment sizes, and block sizes that determine
how bios can be merged and split. These limits are typically set by the device
driver during initialization and are used throughout the I/O path to ensure
requests conform to hardware capabilities.

```c
/* Queue limits structure */
struct queue_limits {
    unsigned int            max_hw_sectors;     /* Max sectors per request */
    unsigned int            max_dev_sectors;
    unsigned int            chunk_sectors;
    unsigned int            max_sectors;
    unsigned int            max_segment_size;   /* Max size per segment */
    unsigned int            max_segments;       /* Max segments per request */

    unsigned int            logical_block_size;  /* Usually 512 or 4096 */
    unsigned int            physical_block_size;
    unsigned int            io_min;
    unsigned int            io_opt;

    /* ... more limits ... */
};
```

---

## blk-mq (Multi-Queue Block Layer)

blk-mq is the modern block layer designed for high-performance NVMe and
multi-queue devices.

### blk-mq Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    blk-mq Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPUs                Software Queues         Hardware Queues│
│  ────                ───────────────         ───────────────│
│                                                             │
│  ┌─────┐            ┌─────────────┐                         │
│  │CPU 0│ ─────────▶│  ctx[0]     │─┐                       │
│  └─────┘            └─────────────┘ │                       │
│                                     │      ┌────────────┐   │
│  ┌─────┐            ┌─────────────┐ ├─────▶│  hctx[0]  │───┼─▶ HW Q 0
│  │CPU 1│ ─────────▶│  ctx[1]     │─┤      │            │   │
│  └─────┘            └─────────────┘ │      └────────────┘   │
│                                     │                       │
│  ┌─────┐            ┌─────────────┐ │      ┌────────────┐   │
│  │CPU 2│ ─────────▶│  ctx[2]     │─┼────▶│  hctx[1]   │───┼─▶ HW Q 1
│  └─────┘            └─────────────┘ │      │            │   │
│                                     │      └────────────┘   │
│  ┌─────┐            ┌─────────────┐ │                       │
│  │CPU 3│ ─────────▶│  ctx[3]     │─┘      ┌────────────┐   │
│  └─────┘            └─────────────┘        │  hctx[N]   │───┼─▶ HW Q N
│                                            │            │   │
│                                            └────────────┘   │
│                                                             │
│  - One software queue (ctx) per CPU                         │
│  - Multiple hardware queues (hctx) based on device          │
│  - CPUs mapped to hardware queues                           │
│  - Reduces lock contention                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### blk-mq Key Structures

The `blk_mq_hw_ctx` structure represents a hardware queue context in the blk-mq
subsystem. Each hardware queue corresponds to a submission queue on the
physical device (such as an NVMe submission queue). The structure maintains a
dispatch list of pending requests, pointers to associated software queues
(`ctxs`), request tags for tracking in-flight I/O, and the queue's NUMA node
for memory locality. Hardware queue contexts are the bridge between the
    software queuing layer and the device driver's queue_rq callback.

```c
/* Hardware queue context */
struct blk_mq_hw_ctx {
    struct {
        spinlock_t              lock;
        struct list_head        dispatch;   /* Pending dispatches */
    } ____cacheline_aligned_in_smp;

    struct blk_mq_ctx       **ctxs;         /* Software queue pointers */
    unsigned int            nr_ctx;          /* Number of sw queues */

    struct sbitmap          ctx_map;         /* Bitmap of active ctxs */

    atomic_t                nr_active;       /* Active requests */
    atomic_t                wait_index;

    struct blk_mq_tags      *tags;           /* Request tags */
    struct blk_mq_tags      *sched_tags;     /* Scheduler tags */

    unsigned int            queue_num;       /* Index in array */
    unsigned int            numa_node;       /* NUMA node */
    unsigned int            flags;

    struct request_queue    *queue;
    struct blk_mq_tag_set   *set;

    struct kobject          kobj;

    /* ... more fields ... */
};
```

The `blk_mq_ctx` structure represents a per-CPU software queue context. Each
CPU has its own software queue to minimize lock contention during I/O
submission. The structure contains per-type request lists (for different I/O
priorities), the associated CPU number, and pointers to the hardware queue
contexts it maps to. Software queues serve as staging areas where requests
accumulate before being dispatched to hardware queues.

```c
/* Software queue context */
struct blk_mq_ctx {
    struct {
        spinlock_t          lock;
        struct list_head    rq_lists[HCTX_MAX_TYPES];
    } ____cacheline_aligned_in_smp;

    unsigned int            cpu;            /* Associated CPU */
    unsigned int            index_hw;       /* Index in hctx */

    struct blk_mq_hw_ctx    *hctxs[HCTX_MAX_TYPES];

    struct request_queue    *queue;
    struct kobject          kobj;
};
```

The `blk_mq_tag_set` structure is a shared configuration object that can be
used by multiple request queues. It contains the driver's operation callbacks
(`ops`), hardware queue count, queue depth (number of tags/requests per queue),
and driver-private data. The tag set manages the allocation of request tags
across all queues and is typically initialized once by the driver during probe.
Multiple request queues can share a single tag set, which is common for devices
with multiple namespaces.

```c
/* Tag set - shared by multiple queues */
struct blk_mq_tag_set {
    const struct blk_mq_ops *ops;           /* Driver operations */
    unsigned int            nr_hw_queues;   /* Hardware queue count */
    unsigned int            queue_depth;    /* Requests per queue */
    unsigned int            reserved_tags;
    unsigned int            cmd_size;       /* Driver data size */
    unsigned int            numa_node;
    unsigned int            timeout;
    unsigned int            flags;

    void                    *driver_data;

    struct blk_mq_tags      **tags;

    struct list_head        tag_list;       /* All queues using set */
};
```

### blk-mq Operations

The `blk_mq_ops` structure defines the callback interface that block device
drivers must implement to work with the blk-mq subsystem. The most critical
callback is `queue_rq`, which the block layer invokes to submit a request to
the hardware. Other callbacks handle request completion, timeout handling,
polling for completions, and hardware queue initialization. Drivers implement
these callbacks to translate generic block requests into device-specific
commands.

```c
/* include/linux/blk-mq.h */
struct blk_mq_ops {
    /* Queue request */
    blk_status_t (*queue_rq)(struct blk_mq_hw_ctx *hctx,
                             const struct blk_mq_queue_data *bd);

    /* Commit multiple requests (optional) */
    void (*commit_rqs)(struct blk_mq_hw_ctx *hctx);

    /* Timeout handler */
    enum blk_eh_timer_return (*timeout)(struct request *rq);

    /* Poll for completions */
    int (*poll)(struct blk_mq_hw_ctx *hctx, struct io_comp_batch *iob);

    /* Request completion */
    void (*complete)(struct request *rq);

    /* Initialize request */
    int (*init_request)(struct blk_mq_tag_set *set, struct request *rq,
                        unsigned int hctx_idx, unsigned int numa_node);
    void (*exit_request)(struct blk_mq_tag_set *set, struct request *rq,
                         unsigned int hctx_idx);

    /* Initialize hardware context */
    int (*init_hctx)(struct blk_mq_hw_ctx *hctx, void *data,
                     unsigned int hctx_idx);
    void (*exit_hctx)(struct blk_mq_hw_ctx *hctx, unsigned int hctx_idx);

    /* Map queues to CPUs */
    void (*map_queues)(struct blk_mq_tag_set *set);
};
```

The `blk_mq_queue_data` structure is a simple container passed to the driver's
`queue_rq` callback. It holds the request being submitted and a flag indicating
whether this is the last request in the current batch. The `last` field allows
drivers to optimize by deferring doorbell writes until the final request in a
batch, reducing MMIO overhead.

```c
/* Queue data for queue_rq */
struct blk_mq_queue_data {
    struct request *rq;
    bool last;      /* Last in batch? */
};
```

### blk-mq I/O Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    blk-mq I/O Flow                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Submit bio                                              │
│     submit_bio(bio)                                         │
│         │                                                   │
│         ▼                                                   │
│  2. Get software queue context (per-CPU)                    │
│     ctx = blk_mq_get_ctx(q)                                 │
│         │                                                   │
│         ▼                                                   │
│  3. Get request from tag pool                               │
│     rq = blk_mq_get_request()                               │
│         │                                                   │
│         ▼                                                   │
│  4. Attach bio(s) to request                                │
│     blk_mq_bio_to_request(rq, bio)                          │
│         │                                                   │
│         ▼                                                   │
│  5. Try to merge with existing request                      │
│     blk_mq_sched_try_merge()                                │
│         │                                                   │
│         ├─── Merged? Done                                   │
│         │                                                   │
│         ▼                                                   │
│  6. Insert into scheduler or direct dispatch                │
│     blk_mq_sched_insert_request() or                        │
│     blk_mq_try_issue_directly()                             │
│         │                                                   │
│         ▼                                                   │
│  7. Run hardware queue                                      │
│     blk_mq_run_hw_queue()                                   │
│         │                                                   │
│         ▼                                                   │
│  8. Driver queue_rq callback                                │
│     hctx->ops->queue_rq(hctx, &bd)                          │
│         │                                                   │
│         ▼                                                   │
│  9. Hardware processes request                              │
│         │                                                   │
│         ▼                                                   │
│  10. Completion interrupt                                   │
│      blk_mq_complete_request(rq)                            │
│      rq->end_io(rq, status)                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## I/O Schedulers

I/O schedulers optimize request ordering and dispatching.

### Available Schedulers

```
┌─────────────────────────────────────────────────────────────┐
│                    I/O Schedulers                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  none (noop)                                                │
│  ──────────                                                 │
│  - No reordering                                            │
│  - FIFO dispatch                                            │
│  - Best for NVMe (device has internal scheduler)            │
│  - Lowest CPU overhead                                      │
│                                                             │
│  mq-deadline                                                │
│  ───────────                                                │
│  - Deadline guarantee for requests                          │
│  - Separate read/write queues                               │
│  - Good for HDDs and general use                            │
│  - Default for many devices                                 │
│                                                             │
│  bfq (Budget Fair Queueing)                                 │
│  ──────────────────────────                                 │
│  - Per-process I/O scheduling                               │
│  - Guarantees bandwidth/latency                             │
│  - Good for desktop/interactive                             │
│  - Higher CPU overhead                                      │
│                                                             │
│  kyber                                                      │
│  ─────                                                      │
│  - Token-based scheduling                                   │
│  - Low latency focus                                        │
│  - Good for fast devices (NVMe)                             │
│  - Minimal tuning needed                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Scheduler Structure

The `elevator_type` structure defines an I/O scheduler implementation that can
be registered with the block layer. It specifies the scheduler's name, module
owner, and the `elevator_mq_ops` callbacks that implement the scheduling logic.
Schedulers can be selected at runtime on a per-device basis through sysfs,
allowing administrators to choose the best scheduler for each device's
characteristics.

```c
/* include/linux/elevator.h */
struct elevator_type {
    struct kmem_cache *icq_cache;

    /* Scheduler operations */
    struct elevator_mq_ops ops;

    size_t icq_size;
    size_t icq_align;
    struct elv_fs_entry *elevator_attrs;
    const char *elevator_name;
    const char *elevator_alias;
    struct module *elevator_owner;
};
```

The `elevator_mq_ops` structure contains the operation callbacks that an I/O
scheduler must implement. Key callbacks include `insert_requests` for adding
requests to scheduler queues, `dispatch_request` for selecting the next request
to send to hardware, and merge callbacks for combining adjacent I/O operations.
The scheduler uses these hooks to reorder, merge, and prioritize requests
according to its scheduling policy.

```c
struct elevator_mq_ops {
    /* Per-request init/exit */
    int (*init_sched)(struct request_queue *q, struct elevator_type *e);
    void (*exit_sched)(struct elevator_queue *e);
    int (*init_hctx)(struct blk_mq_hw_ctx *hctx, unsigned int hctx_idx);
    void (*exit_hctx)(struct blk_mq_hw_ctx *hctx, unsigned int hctx_idx);

    /* Request handling */
    void (*insert_requests)(struct blk_mq_hw_ctx *hctx,
                           struct list_head *list, bool at_head);
    struct request *(*dispatch_request)(struct blk_mq_hw_ctx *hctx);
    bool (*has_work)(struct blk_mq_hw_ctx *hctx);
    void (*completed_request)(struct request *rq, u64 now);

    /* Merging */
    bool (*bio_merge)(struct request_queue *q, struct bio *bio,
                      unsigned int nr_segs);
    int (*request_merge)(struct request_queue *q, struct request **rq,
                         struct bio *bio);
    void (*request_merged)(struct request_queue *q, struct request *rq,
                          enum elv_merge type);
    void (*requests_merged)(struct request_queue *q, struct request *rq,
                           struct request *next);

    /* ... more callbacks ... */
};
```

### Scheduler Selection

```bash
# View available schedulers
cat /sys/block/sda/queue/scheduler
# Output: [mq-deadline] kyber bfq none

# Change scheduler
echo "bfq" > /sys/block/sda/queue/scheduler

# View current scheduler
cat /sys/block/nvme0n1/queue/scheduler
# NVMe typically uses: [none] mq-deadline kyber bfq
```

---

## Gendisk and Block Device

### Gendisk Structure

The `gendisk` structure represents a physical or logical disk device in the
Linux kernel. It contains the device's major and minor numbers, device name
(visible in /dev), partition table, and a pointer to the request queue. The
gendisk also holds the block device operations (`fops`) that define how the
device handles opens, ioctls, and other operations. Device drivers allocate a
gendisk with `blk_alloc_disk()`, configure it, and register it with
`add_disk()` to make the device available to the system.

```c
/* include/linux/blkdev.h */
struct gendisk {
    int                     major;          /* Major number */
    int                     first_minor;    /* First minor number */
    int                     minors;         /* Maximum minor numbers */

    char                    disk_name[DISK_NAME_LEN];  /* /dev name */

    unsigned short          events;         /* Supported events */
    unsigned short          event_flags;

    struct xarray           part_tbl;       /* Partition table */
    struct block_device     *part0;         /* Whole disk device */

    const struct block_device_operations *fops;

    struct request_queue    *queue;
    void                    *private_data;

    struct bio_set          bio_split;

    int                     flags;
    unsigned long           state;

    struct mutex            open_mutex;
    unsigned                open_partitions;

    struct kobject          *slave_dir;

    int                     node_id;

    struct badblocks        *bb;

    /* ... more fields ... */
};
```

The `block_device` structure represents either a whole disk or a partition on a
disk. It tracks the device's sector range (`bd_start_sect` and
`bd_nr_sectors`), maintains references to the parent gendisk and request queue,
and manages device opening/holding semantics. For partitions, the `bd_partno`
field identifies the partition number. The block_device is the structure that
filesystems and other consumers interact with when performing I/O to a specific
device or partition.

```c
/* Block device (represents partition or whole disk) */
struct block_device {
    sector_t                bd_start_sect;  /* Start sector */
    sector_t                bd_nr_sectors;  /* Number of sectors */

    struct gendisk          *bd_disk;
    struct request_queue    *bd_queue;

    struct disk_stats __percpu *bd_stats;

    unsigned long           bd_stamp;
    bool                    bd_read_only;

    dev_t                   bd_dev;
    atomic_t                bd_openers;

    struct inode            *bd_inode;

    void                    *bd_holder;
    int                     bd_holders;
    bool                    bd_write_holder;

    struct kobject          *bd_holder_dir;

    u8                      bd_partno;
    spinlock_t              bd_size_lock;

    /* ... more fields ... */
};
```

### Block Device Operations

The `block_device_operations` structure defines the methods that a block device
driver implements to handle device-level operations beyond I/O. Key callbacks
include `open` and `release` for device open/close, `ioctl` for device-specific
commands, `getgeo` for reporting disk geometry, and `submit_bio` for drivers
that handle bio submission directly. For zoned storage devices, the
`report_zones` callback provides zone information. This structure is registered
with the gendisk during device initialization.

```c
/* include/linux/blkdev.h */
struct block_device_operations {
    void (*submit_bio)(struct bio *bio);
    int (*open)(struct block_device *bdev, fmode_t mode);
    void (*release)(struct gendisk *disk, fmode_t mode);
    int (*rw_page)(struct block_device *bdev, sector_t sector,
                   struct page *page, enum req_op op);
    int (*ioctl)(struct block_device *bdev, fmode_t mode,
                 unsigned cmd, unsigned long arg);
    int (*compat_ioctl)(struct block_device *bdev, fmode_t mode,
                        unsigned cmd, unsigned long arg);
    unsigned int (*check_events)(struct gendisk *disk,
                                  unsigned int clearing);
    void (*unlock_native_capacity)(struct gendisk *disk);
    int (*getgeo)(struct block_device *bdev, struct hd_geometry *geo);
    int (*set_read_only)(struct block_device *bdev, bool ro);
    void (*free_disk)(struct gendisk *disk);

    /* Swap */
    int (*swap_slot_free_notify)(struct block_device *bdev,
                                  unsigned long slot);

    /* Report zones for zoned devices */
    int (*report_zones)(struct gendisk *disk, sector_t sector,
                        unsigned int nr_zones,
                        report_zones_cb cb, void *data);

    /* For stackable devices */
    struct module *owner;
    const struct pr_ops *pr_ops;
};
```

### Registering a Block Device

```c
/* Driver registration example */
static int my_blk_driver_init(void)
{
    int ret;

    /* Register major number */
    my_major = register_blkdev(0, "myblk");
    if (my_major < 0)
        return my_major;

    /* Allocate gendisk */
    my_disk = blk_alloc_disk(NUMA_NO_NODE);
    if (!my_disk) {
        ret = -ENOMEM;
        goto out_unregister;
    }

    /* Setup disk */
    my_disk->major = my_major;
    my_disk->first_minor = 0;
    my_disk->minors = 16;  /* Allow 15 partitions */
    my_disk->fops = &my_block_ops;
    my_disk->private_data = my_data;
    snprintf(my_disk->disk_name, DISK_NAME_LEN, "myblk0");

    /* Set capacity in 512-byte sectors */
    set_capacity(my_disk, my_size_bytes >> 9);

    /* Setup request queue */
    blk_queue_logical_block_size(my_disk->queue, 512);
    blk_queue_physical_block_size(my_disk->queue, 4096);

    /* Add disk to system */
    ret = add_disk(my_disk);
    if (ret)
        goto out_put_disk;

    return 0;

out_put_disk:
    put_disk(my_disk);
out_unregister:
    unregister_blkdev(my_major, "myblk");
    return ret;
}

static void my_blk_driver_exit(void)
{
    del_gendisk(my_disk);
    put_disk(my_disk);
    unregister_blkdev(my_major, "myblk");
}
```

---

## Device Mapper

Device mapper provides a flexible framework for creating virtual block devices.

### Device Mapper Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Device Mapper Architecture                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Space                                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  dmsetup, LVM tools, cryptsetup, multipathd         │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │ ioctl                           │
│  ─────────────────────────┼─────────────────────────────    │
│                           ▼                                 │
│  Kernel Space                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              DM Core (/dev/mapper/*)                │    │
│  │  - Manages mapped devices                           │    │
│  │  - Routes I/O to targets                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              DM Targets                             │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │ dm-linear  │ dm-striped │ dm-mirror │ dm-snapshot   │    │
│  │ dm-crypt   │ dm-cache   │ dm-thin   │ dm-raid       │    │
│  │ dm-verity  │ dm-zero    │ dm-error  │ dm-delay      │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                 │
│                           ▼                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Underlying Devices                     │    │
│  │  /dev/sda, /dev/nvme0n1, /dev/md0, etc.             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### DM Target Types

The `target_type` structure defines a device mapper target implementation. Each
target type (such as dm-linear, dm-crypt, or dm-thin) registers a target_type
with the device mapper core. The structure contains the target name, feature
flags, and operation callbacks including the constructor (`ctr`) that parses
target arguments, destructor (`dtr`), and the crucial `map` callback that
transforms bios as they pass through the target. This modular design allows new
storage virtualization features to be added as loadable kernel modules.

```c
/* include/linux/device-mapper.h */
struct target_type {
    const char *name;
    struct module *module;

    /* Feature flags */
    unsigned features;

    /* Target operations */
    int (*ctr)(struct dm_target *ti, unsigned int argc, char **argv);
    void (*dtr)(struct dm_target *ti);
    int (*map)(struct dm_target *ti, struct bio *bio);
    int (*clone_and_map_rq)(struct dm_target *ti, struct request *rq,
                            union map_info *info,
                            struct request **clone);
    void (*release_clone_rq)(struct request *clone, union map_info *info);
    int (*end_io)(struct dm_target *ti, struct bio *bio,
                  blk_status_t *error);
    void (*presuspend)(struct dm_target *ti);
    void (*postsuspend)(struct dm_target *ti);
    int (*preresume)(struct dm_target *ti);
    void (*resume)(struct dm_target *ti);
    void (*status)(struct dm_target *ti, status_type_t status_type,
                   unsigned int status_flags, char *result,
                   unsigned int maxlen);
    int (*message)(struct dm_target *ti, unsigned int argc, char **argv,
                   char *result, unsigned int maxlen);
    int (*prepare_ioctl)(struct dm_target *ti, struct block_device **bdev);
    int (*iterate_devices)(struct dm_target *ti,
                           iterate_devices_callout_fn fn, void *data);
    void (*io_hints)(struct dm_target *ti, struct queue_limits *limits);

    struct list_head list;
};
```

### Common DM Targets

```
┌─────────────────────────────────────────────────────────────┐
│                  Common DM Targets                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  dm-linear                                                  │
│  ─────────                                                  │
│  Maps a linear range from underlying device                 │
│  Used by LVM for simple volume extension                    │
│  Table: "0 1000000 linear /dev/sda1 0"                      │
│                                                             │
│  dm-striped                                                 │
│  ──────────                                                 │
│  RAID-0 striping across multiple devices                    │
│  Table: "0 2000000 striped 2 128 /dev/sda 0 /dev/sdb 0"     │
│                                                             │
│  dm-crypt                                                   │
│  ────────                                                   │
│  Transparent disk encryption                                │
│  Used by LUKS, cryptsetup                                   │
│  Table: "0 1000000 crypt aes-xts-plain64 <key> 0 /dev/sda 0"│
│                                                             │
│  dm-thin                                                    │
│  ───────                                                    │
│  Thin provisioning with copy-on-write                       │
│  Over-provisioning, snapshots                               │
│                                                             │
│  dm-verity                                                  │
│  ─────────                                                  │
│  Integrity verification for read-only devices               │
│  Used by Android dm-verity boot                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## I/O Path

### Complete Read Path

```
┌─────────────────────────────────────────────────────────────┐
│                    Complete Read Path                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User: read(fd, buf, len)                                   │
│         │                                                   │
│         ▼                                                   │
│  VFS: vfs_read()                                            │
│         │                                                   │
│         ▼                                                   │
│  Page Cache: Check if data cached                           │
│         │                                                   │
│         ├── Hit: Copy to user buffer, return                │
│         │                                                   │
│         ▼ Miss                                              │
│  Filesystem: ext4_file_read_iter()                          │
│         │                                                   │
│         ▼                                                   │
│  Create bio: bio_alloc(), set page, sector                  │
│         │                                                   │
│         ▼                                                   │
│  submit_bio(bio)                                            │
│         │                                                   │
│         ▼                                                   │
│  Block layer: blk_mq_submit_bio()                           │
│         │                                                   │
│         ├── Try merge with pending requests                 │
│         │                                                   │
│         ▼                                                   │
│  Get request: blk_mq_get_new_requests()                     │
│         │                                                   │
│         ▼                                                   │
│  I/O Scheduler: insert or direct dispatch                   │
│         │                                                   │
│         ▼                                                   │
│  Hardware queue: blk_mq_run_hw_queue()                      │
│         │                                                   │
│         ▼                                                   │
│  Driver: ops->queue_rq() (e.g., nvme_queue_rq)              │
│         │                                                   │
│         ▼                                                   │
│  DMA: Setup SGL, write doorbell                             │
│         │                                                   │
│         ▼                                                   │
│  Hardware: NVMe controller processes                        │
│         │                                                   │
│         ▼                                                   │
│  Completion IRQ                                             │
│         │                                                   │
│         ▼                                                   │
│  blk_mq_complete_request()                                  │
│         │                                                   │
│         ▼                                                   │
│  bio_endio(bio)                                             │
│         │                                                   │
│         ▼                                                   │
│  Unlock page, wake waiter                                   │
│         │                                                   │
│         ▼                                                   │
│  Copy to user, return to application                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Complete Write Path

```
┌─────────────────────────────────────────────────────────────┐
│                    Complete Write Path                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User: write(fd, buf, len)                                  │
│         │                                                   │
│         ▼                                                   │
│  VFS: vfs_write()                                           │
│         │                                                   │
│         ▼                                                   │
│  Buffered I/O path (default):                               │
│  ────────────────────────────                               │
│         │                                                   │
│         ▼                                                   │
│  Copy to page cache                                         │
│         │                                                   │
│         ▼                                                   │
│  Mark page dirty, return to user                            │
│         │                                                   │
│         │ (async writeback later)                           │
│         ▼                                                   │
│  Writeback: balance_dirty_pages()                           │
│         │                                                   │
│         ▼                                                   │
│  Filesystem: ext4_writepages()                              │
│         │                                                   │
│         ▼                                                   │
│  Create bio(s) for dirty pages                              │
│         │                                                   │
│         ▼                                                   │
│  [Same block layer path as read]                            │
│                                                             │
│  ═══════════════════════════════════════════════════════    │
│                                                             │
│  Direct I/O path (O_DIRECT):                                │
│  ───────────────────────────                                │
│         │                                                   │
│         ▼                                                   │
│  Pin user pages                                             │
│         │                                                   │
│         ▼                                                   │
│  Create bio directly from user pages                        │
│         │                                                   │
│         ▼                                                   │
│  [Same block layer path as read]                            │
│                                                             │
│  ═══════════════════════════════════════════════════════    │
│                                                             │
│  Sync write (O_SYNC or fsync):                              │
│  ─────────────────────────────                              │
│         │                                                   │
│         ▼                                                   │
│  Write data (as above)                                      │
│         │                                                   │
│         ▼                                                   │
│  Issue REQ_PREFLUSH + REQ_FUA                               │
│         │                                                   │
│         ▼                                                   │
│  Wait for completion                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Bio Splitting

Bios are split when they exceed device queue limits. The entry point is
`__bio_split_to_limits()` (`block/blk.h`), which dispatches by operation type
to `bio_split_rw()`, `bio_split_discard()`, or `bio_split_write_zeroes()`.

`bio_split_rw_at()` (`block/blk-merge.c`) walks the bvec array counting
physical segments and returns the sector offset where a split must occur.
Split triggers include:

- `lim->max_sectors` — maximum sectors per request
- `lim->chunk_sectors` — boundary alignment (NVMe namespace, zone granularity)
- `lim->max_segments` — maximum scatter-gather segments
- `lim->max_segment_size` — maximum size of a single DMA segment
- `lim->seg_boundary_mask` — segment crossing restrictions

The actual split uses `bio_submit_split()` (`block/blk-merge.c`):

```c
split = bio_split(bio, split_sectors, GFP_NOIO, &disk->bio_split);
split->bi_opf |= REQ_NOMERGE;
bio_chain(split, bio);       /* parent completes only when both halves finish */
submit_bio_noacct(bio);      /* re-submit the remainder */
return split;                /* return the front portion */
```

---

## Plugging

Plugging batches I/O submissions to reduce per-request overhead and enable
merging. The plug is stored in `current->plug`.

### struct blk_plug (include/linux/blkdev.h)

```c
struct blk_plug {
    struct rq_list mq_list;     /* blk-mq requests pending flush */
    struct rq_list cached_rqs;  /* pre-allocated requests for reuse */
    unsigned short nr_ios;      /* hint: how many IOs will be submitted */
    unsigned short rq_count;    /* current count in mq_list */
    bool multiple_queues;       /* requests span >1 request_queue */
    bool has_elevator;          /* any request uses a scheduler */
    struct list_head cb_list;   /* unplug callbacks */
};
```

### Plug Lifecycle

`blk_start_plug()` (`block/blk-core.c`) initializes the plug and assigns it
to `current->plug`. Nested plugs are silently ignored.

`blk_add_rq_to_plug()` (`block/blk-mq.c`) adds requests to the plug. An
early flush triggers when:
- `rq_count >= BLK_MAX_REQUEST_COUNT` (32, or 64 for multi-queue)
- Last request exceeds `BLK_PLUG_FLUSH_SIZE` (128 KB)

`blk_finish_plug()` (`block/blk-core.c`) flushes the plug via
`blk_mq_flush_plug_list()`, which dispatches to the hardware queue. When no
scheduler is involved, it tries `mq_ops->queue_rqs()` for batch submission.

The plug is also flushed implicitly when the task calls `io_schedule()`.

`__submit_bio()` (`block/blk-core.c`) creates an **implicit plug** around
every `blk_mq_submit_bio()` call, ensuring at minimum a single-request batch.

---

## Tag Allocation

### struct blk_mq_tags (include/linux/blk-mq.h)

```c
struct blk_mq_tags {
    unsigned int nr_tags;
    unsigned int nr_reserved_tags;  /* reserved for flush/internal */
    unsigned int active_queues;

    struct sbitmap_queue bitmap_tags;    /* normal tag bitmap */
    struct sbitmap_queue breserved_tags; /* reserved tag bitmap */

    struct request **rqs;        /* tag -> in-flight request */
    struct request **static_rqs; /* tag -> pre-allocated request */
    struct list_head page_list;

    spinlock_t lock;
};
```

Two sbitmap queues: `bitmap_tags` for normal I/O, `breserved_tags` for flush
operations. When a scheduler is active, requests come from `hctx->sched_tags`
first; the hardware driver tag from `hctx->tags` is assigned at dispatch time
via `blk_mq_get_driver_tag()`.

`blk_mq_get_tag()` (`block/blk-mq-tag.c`) atomically allocates a bit from
the sbitmap. If no tag is available and `BLK_MQ_REQ_NOWAIT` is not set, it
calls `blk_mq_run_hw_queue()` to drain completions, then sleeps on the
sbitmap wait queue via `io_schedule()`.

`blk_mq_put_tag()` clears the bit and wakes waiters.

---

## I/O Accounting

### struct disk_stats (include/linux/part_stat.h)

```c
struct disk_stats {
    u64 nsecs[NR_STAT_GROUPS];             /* time spent (ns) per direction */
    unsigned long sectors[NR_STAT_GROUPS]; /* sectors transferred */
    unsigned long ios[NR_STAT_GROUPS];     /* completed IOs */
    unsigned long merges[NR_STAT_GROUPS];  /* merged IOs */
    unsigned long io_ticks;                /* time device was busy (jiffies) */
    local_t in_flight[2];                  /* in-flight: [0]=read, [1]=write */
};
```

Allocated **per-CPU per block_device** (`bd_stats`). Locking is just
`preempt_disable()` — no spinlock — because per-CPU data is accessed only
from the local CPU. `part_stat_read()` sums across all CPUs for display
(e.g., `/sys/block/sda/stat`).

`blk_account_io_start()` (`block/blk-mq.c`) records `start_time_ns` and
increments `in_flight`. `blk_account_io_done()` records completion time,
increments `ios` and `nsecs`, and decrements `in_flight`.

---

## Writeback Throttling (wbt)

wbt is an rq-qos plugin (`block/blk-wbt.c`) that throttles buffered writes
to prevent them from starving reads.

### rq-qos Framework

rq-qos is a hook chain on `request_queue` (`q->rq_qos`). Each node has a
`struct rq_qos_ops` vtable with hooks called at key points:

| Hook | Called In | Purpose |
|------|-----------|---------|
| `throttle` | `blk_mq_get_new_requests()` | May sleep to throttle bio |
| `track` | `blk_mq_submit_bio()` | Associate bio flags with request |
| `issue` | `blk_mq_start_request()` | Request dispatched to hardware |
| `done` | `__blk_mq_end_request()` | Request completed |
| `cleanup` | alloc failure path | Release throttle token on error |

Three rq-qos plugins exist:
- `RQ_QOS_WBT` — writeback throttling (`block/blk-wbt.c`)
- `RQ_QOS_LATENCY` — I/O latency QoS (`block/blk-iolatency.c`)
- `RQ_QOS_COST` — iocost (`block/blk-iocost.c`)

### wbt Operation

wbt tracks only buffered writes and discards (not reads or sync writes).
`wbt_wait()` sleeps if the in-flight count exceeds the current depth limit.
Limits adapt via a 100ms timer: if latency exceeds `min_lat_nsec`, the depth
shrinks; if latency is good, it grows. Three wait classes: background writes,
swap, and discards.

---

## Locking Summary

| Lock | Type | Protects |
|------|------|----------|
| `hctx->lock` | spinlock (cacheline-aligned) | `hctx->dispatch` list (failed dispatch retry) |
| `ctx->lock` | spinlock (cacheline-aligned) | `ctx->rq_lists[]` per-CPU software queues |
| `tags->lock` | spinlock | `tags->rqs[]` tag-to-request mapping |
| `q->requeue_lock` | spinlock | `q->requeue_list` |
| `q->rq_qos_mutex` | mutex | `q->rq_qos` chain add/remove |
| `q->limits_lock` | mutex | `q->limits` (queue_limits) |
| `q->sysfs_lock` | mutex | sysfs attribute access |
| `q->mq_freeze_lock` | mutex | freeze/quiesce ref counting |
| `q->blkcg_mutex` | mutex | blkcg group list |
| `flush_queue->mq_flush_lock` | spinlock | flush state machine |
| `elv_list_lock` | spinlock (global) | registered scheduler types |

The submission path (`blk_mq_submit_bio()`) is **lockless** for the common
case. Sbitmap uses atomic bit operations. `ctx->lock` is taken only when
inserting into software queues (scheduler path). `hctx->lock` is taken only
when adding to `hctx->dispatch` (failed dispatch retry).

---

## Source Files

```
block/blk-core.c          - submit_bio(), plugging, queue lifecycle
block/blk-mq.c            - blk_mq_submit_bio(), tag alloc, dispatch, completion
block/blk-mq-tag.c        - blk_mq_get_tag(), sbitmap tag operations
block/blk-merge.c          - bio_split_to_limits(), segment merge logic
block/blk-mq-sched.c      - scheduler integration (bio_merge, dispatch)
block/elevator.c           - scheduler framework: registration, switching
block/mq-deadline.c        - mq-deadline scheduler
block/kyber-iosched.c      - kyber scheduler
block/bfq-iosched.c        - BFQ scheduler
block/blk-wbt.c            - writeback throttling
block/blk-rq-qos.c         - rq-qos framework dispatch
block/blk-throttle.c       - cgroup blkio throttling
block/blk-iocost.c         - iocost controller
block/blk-iolatency.c      - IO latency QoS
block/blk-flush.c          - flush sequence state machine
block/blk-stat.c           - request statistics
block/blk-settings.c       - queue_limits setters
block/blk-sysfs.c          - queue sysfs attributes
block/blk-cgroup.c         - blkcg policy framework
block/genhd.c              - gendisk registration, disk_stats
block/bdev.c               - block device open/close
block/bio.c                - bio allocation, bio_split(), bio_chain()
block/bounce.c             - high-memory page bouncing
block/fops.c               - block device file operations

drivers/md/dm.c            - DM core: dm_submit_bio(), bio splitting
drivers/md/dm-table.c      - DM target table management
drivers/md/dm-ioctl.c      - DM userspace control (/dev/mapper/control)
drivers/md/dm-linear.c     - linear target
drivers/md/dm-stripe.c     - striping target
drivers/md/dm-crypt.c      - encryption target
drivers/md/dm-thin.c       - thin provisioning target
drivers/md/dm-verity-target.c - verity target
drivers/md/dm-mpath.c      - multipath target

include/linux/blkdev.h     - request_queue, blk_plug, queue_limits
include/linux/blk-mq.h     - blk_mq_hw_ctx, blk_mq_tags, blk_mq_tag_set, blk_mq_ops
include/linux/blk_types.h  - bio, block_device, blk_opf_t
include/linux/part_stat.h  - disk_stats, part_stat macros
```

---

## Complete submit_bio() Call Chain

```
submit_bio()                         [block/blk-core.c]
  submit_bio_noacct()
    blk_partition_remap()            (if partition)
    blk_throtl_bio()                 (cgroup throttle, may sleep)
    submit_bio_noacct_nocheck()
      __submit_bio()
        blk_start_plug()             (implicit plug)
        blk_mq_submit_bio()          [block/blk-mq.c]
          blk_queue_bounce()         (if highmem pages)
          __bio_split_to_limits()    (split if exceeds limits)
          bio_integrity_prep()
          blk_mq_attempt_bio_merge() (try merge with existing)
          blk_mq_get_new_requests()
            rq_qos_throttle()        -> wbt_wait() (may sleep)
            __blk_mq_alloc_requests()
              blk_mq_get_tag()       (sbitmap atomic alloc)
          rq_qos_track()
          blk_mq_bio_to_request()    (populate request from bio)
          [if plug active]:
            blk_add_rq_to_plug()
          [else direct issue]:
            blk_mq_try_issue_directly()
              q->mq_ops->queue_rq()  [DRIVER]
        blk_finish_plug()            (flush plug -> dispatch)

Completion path (interrupt context):
  blk_mq_complete_request()
    blk_mq_end_request()
      blk_account_io_done()          (update disk_stats)
      rq_qos_done()                  -> wbt_done()
      rq->end_io()
      blk_mq_free_request()
        blk_mq_put_tag()             (sbitmap clear, wake waiters)
```
