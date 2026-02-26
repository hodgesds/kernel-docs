# Linux Kernel Memory Management Overview

A comprehensive guide to Linux kernel memory management internals, covering the
full stack from physical page allocation through virtual memory translation.

---

## Table of Contents

1. [Physical Memory Organization](#1-physical-memory-organization)
2. [The Buddy Allocator](#2-the-buddy-allocator)
3. [GFP Flags System](#3-gfp-flags-system)
4. [NUMA Memory Management](#4-numa-memory-management)
5. [The Slab Allocator (SLUB)](#5-the-slab-allocator-slub)
6. [Folio Abstraction](#6-folio-abstraction)
7. [Virtual Memory Areas (VMAs)](#7-virtual-memory-areas-vmas)
8. [Page Tables and Address Translation](#8-page-tables-and-address-translation)
9. [Page Table Walking](#9-page-table-walking)
10. [TLB Management](#10-tlb-management)
11. [Page Fault Handling](#11-page-fault-handling)
12. [Memory Reclaim and LRU](#12-memory-reclaim-and-lru)
13. [Swap Subsystem](#13-swap-subsystem)
14. [Key Data Structures Reference](#14-key-data-structures-reference)
15. [Memory Compaction](#15-memory-compaction)
16. [OOM Killer](#16-oom-killer)
17. [vmalloc](#17-vmalloc)
18. [Memory Control Groups (memcg)](#18-memory-control-groups-memcg)
19. [Locking Summary](#19-locking-summary)
20. [Source Files Reference](#20-source-files-reference)

---

## 1. Physical Memory Organization

### Memory Zones

Linux organizes physical memory into zones based on their addressing
capabilities and usage characteristics.

**Zone Types** (defined in `include/linux/mmzone.h`):

```c
enum zone_type {
    ZONE_DMA,      /* Memory for legacy DMA (< 16MB on x86) */
    ZONE_DMA32,    /* Memory addressable by 32-bit DMA (< 4GB) */
    ZONE_NORMAL,   /* Normal kernel-addressable memory */
    ZONE_HIGHMEM,  /* Memory requiring kmap() (32-bit only) */
    ZONE_MOVABLE,  /* Movable pages for memory hotplug */
    ZONE_DEVICE,   /* Device memory (DAX, GPU, etc.) */
    __MAX_NR_ZONES
};
```

### struct zone

The zone structure is the fundamental unit of physical memory organization in
Linux. Each zone represents a contiguous range of physical memory with specific
addressing characteristics and usage constraints. The kernel uses zones to
segregate memory based on DMA capabilities, ensuring that device drivers can
always find suitable memory for their hardware requirements.

The structure contains watermarks that control memory reclaim behavior, free
lists organized by the buddy allocator for efficient page allocation, and
per-CPU page caches for fast allocation without lock contention. Key fields
include the watermark array for triggering reclaim, free_area containing the
buddy allocator's free lists, and per_cpu_pageset for lockless order-0
allocations.

```c
struct zone {
    /* Watermarks for reclaim */
    unsigned long _watermark[NR_WMARK];  /* MIN, LOW, HIGH, PROMO */
    unsigned long watermark_boost;

    /* Reserved memory for lower zones */
    long lowmem_reserve[MAX_NR_ZONES];

    /* Zone boundaries */
    unsigned long zone_start_pfn;
    atomic_long_t managed_pages;     /* Managed by buddy allocator */
    unsigned long spanned_pages;     /* Total including holes */
    unsigned long present_pages;     /* Physical pages present */

    /* Buddy allocator free lists */
    struct free_area free_area[NR_PAGE_ORDERS];

    /* Per-CPU page lists for fast allocation */
    struct per_cpu_pages __percpu *per_cpu_pageset;

    spinlock_t lock;                 /* Protects free_area */
    const char *name;                /* "DMA", "Normal", etc. */
};
```

### Watermarks

Watermarks control when memory reclaim activates:

| Watermark | Purpose |
|-----------|---------|
| **WMARK_MIN** | Emergency reserves - only PF_MEMALLOC processes |
| **WMARK_LOW** | Triggers kswapd background reclaim |
| **WMARK_HIGH** | kswapd stops reclaiming |
| **WMARK_PROMO** | Memory tiering promotion threshold |

```c
/* Access watermarks */
static inline unsigned long min_wmark_pages(const struct zone *z);
static inline unsigned long low_wmark_pages(const struct zone *z);
static inline unsigned long high_wmark_pages(const struct zone *z);
```

---

## 2. The Buddy Allocator

The buddy allocator manages physical pages in power-of-two sized blocks for
efficient allocation and coalescing.

### Core Concepts

- **Order**: Block size as power of 2 (order 0 = 1 page, order 10 = 1024 pages = 4MB)
- **Buddy**: Adjacent block that can be merged (found via XOR operation)
- **MAX_PAGE_ORDER**: Maximum order (default 10, configurable)

### struct free_area

The free_area structure represents a single order level in the buddy
allocator's free list hierarchy. Each zone maintains an array of free_area
structures, one for each possible allocation order from 0 to MAX_PAGE_ORDER.
This organization enables the buddy allocator to quickly locate free blocks of
any size.

Within each free_area, pages are further segregated by migration type to reduce
memory fragmentation. The free_list array contains separate linked lists for
unmovable, movable, and reclaimable pages, allowing the allocator to group
pages by their mobility characteristics. The nr_free field tracks the total
number of free blocks at this order level across all migration types.

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  /* Per-migratetype lists */
    unsigned long nr_free;                       /* Total free blocks */
};
```

**Migration Types** (anti-fragmentation):
- `MIGRATE_UNMOVABLE` - Kernel allocations that cannot move
- `MIGRATE_MOVABLE` - User pages that can be migrated/reclaimed
- `MIGRATE_RECLAIMABLE` - Kernel caches that can be reclaimed
- `MIGRATE_HIGHATOMIC` - Reserved for high-priority atomic allocations
- `MIGRATE_CMA` - Contiguous Memory Allocator regions

### struct page (Buddy Allocator Fields)

The page structure is the most fundamental memory descriptor in the Linux
kernel, representing a single physical page frame. Due to its ubiquity (one
struct page exists for every physical page), the structure uses extensive
unions to minimize memory overhead while supporting multiple use cases
including buddy allocation, slab allocation, page cache, and anonymous memory.

When a page is free in the buddy allocator, the buddy_list field links it into
the appropriate free_area list, and the private field stores the block's order.
The flags field contains page state bits including PageBuddy which marks free
pages. The refcount tracks references to prevent premature freeing. This
union-based design allows the same memory to serve different roles depending on
the page's current state.

```c
struct page {
    unsigned long flags;           /* Page flags including PageBuddy */

    union {
        struct {
            union {
                struct list_head lru;
                struct list_head buddy_list;  /* Link in free_area */
                struct list_head pcp_list;    /* Per-CPU page list */
            };
            struct address_space *mapping;
            pgoff_t index;
            unsigned long private;  /* Stores order when free */
        };
        /* ... other uses ... */
    };

    atomic_t _refcount;            /* Reference count */
};
```

### Buddy Finding Algorithm

```c
/* Find buddy's PFN using XOR - buddies differ only in bit 'order' */
static inline unsigned long __find_buddy_pfn(unsigned long page_pfn,
                                              unsigned int order)
{
    return page_pfn ^ (1 << order);
}

/* Example: PFN 8, order 1 → buddy = 8 ^ 2 = 10 */
```

### Allocation: expand() - Splitting Blocks

When allocating order N from a larger block of order M:

```c
static inline unsigned int expand(struct zone *zone, struct page *page,
                                  int low, int high, int migratetype)
{
    unsigned int size = 1 << high;

    while (high > low) {
        high--;
        size >>= 1;

        /* Add second half to free list at current order */
        __add_to_free_list(&page[size], zone, high, migratetype, false);
        set_buddy_order(&page[size], high);
    }

    return nr_added;  /* Pages added to free lists */
}
```

**Example**: Splitting order 3 to allocate order 1:
```
Initial: [0-7] order 3 (8 pages)
Split 1: [0-3] order 2, [4-7] → free list order 2
Split 2: [0-1] order 1, [2-3] → free list order 1
Result:  [0-1] returned, [2-3] free@order1, [4-7] free@order2
```

### Freeing: __free_one_page() - Coalescing Blocks

```c
static inline void __free_one_page(struct page *page, unsigned long pfn,
                                   struct zone *zone, unsigned int order,
                                   int migratetype, fpi_t fpi_flags)
{
    while (order < MAX_PAGE_ORDER) {
        /* Find buddy */
        buddy = find_buddy_page_pfn(page, pfn, order, &buddy_pfn);
        if (!buddy)
            goto done_merging;  /* Buddy not free */

        /* Remove buddy from its free list */
        __del_page_from_free_list(buddy, zone, order, buddy_mt);

        /* Combine: use lower PFN as new block base */
        combined_pfn = buddy_pfn & pfn;
        page = page + (combined_pfn - pfn);
        pfn = combined_pfn;
        order++;  /* Move to next order */
    }

done_merging:
    set_buddy_order(page, order);
    __add_to_free_list(page, zone, order, migratetype, to_tail);
}
```

### Per-CPU Page Lists (PCP)

The per_cpu_pages structure provides a fast-path allocation mechanism for
single pages (order-0) by maintaining per-CPU caches of free pages. This design
eliminates zone lock contention for the most common allocation pattern,
significantly improving performance on multi-core systems.

Each CPU maintains its own pool of cached pages with high and low watermarks.
When the cache is depleted, it refills from the buddy allocator in batches.
When it exceeds the high watermark, excess pages are returned to the buddy
allocator. The lists array contains multiple lists organized by migration type,
enabling the PCP cache to serve allocations with specific mobility
requirements.

```c
struct per_cpu_pages {
    spinlock_t lock;
    int count;               /* Number of cached pages */
    int high;                /* High watermark */
    int high_min;
    int high_max;
    int batch;               /* Refill/drain batch size */
    struct list_head lists[NR_PCP_LISTS];
};
```

---

## 3. GFP Flags System

GFP (Get Free Pages) flags control allocation behavior, zone selection, and
reclaim policies.

### Flag Categories

#### Zone Modifiers (bits 0-3)

```c
#define __GFP_DMA       (1 << 0)   /* Allocate from ZONE_DMA */
#define __GFP_HIGHMEM   (1 << 1)   /* Allocate from ZONE_HIGHMEM */
#define __GFP_DMA32     (1 << 2)   /* Allocate from ZONE_DMA32 */
#define __GFP_MOVABLE   (1 << 3)   /* Allow ZONE_MOVABLE, page can migrate */

#define GFP_ZONEMASK    (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)
```

#### Watermark Modifiers

```c
#define __GFP_HIGH       (1 << 5)   /* High priority, access 50% of min reserves */
#define __GFP_MEMALLOC   (1 << 17)  /* Access ALL reserves (dangerous!) */
#define __GFP_NOMEMALLOC (1 << 19)  /* Forbid reserve access */
```

#### Reclaim Modifiers

```c
#define __GFP_IO              (1 << 6)   /* Can start physical I/O */
#define __GFP_FS              (1 << 7)   /* Can call into filesystem */
#define __GFP_DIRECT_RECLAIM  (1 << 10)  /* Can enter direct reclaim (blocking) */
#define __GFP_KSWAPD_RECLAIM  (1 << 11)  /* Wake kswapd */
#define __GFP_RECLAIM         (__GFP_DIRECT_RECLAIM | __GFP_KSWAPD_RECLAIM)

/* Retry behavior */
#define __GFP_NORETRY       (1 << 16)  /* Try lightly, may fail */
#define __GFP_RETRY_MAYFAIL (1 << 14)  /* Retry harder, finite limit */
#define __GFP_NOFAIL        (1 << 15)  /* MUST succeed, infinite retry */
```

#### Action Modifiers

```c
#define __GFP_ZERO     (1 << 8)   /* Zero the allocation */
#define __GFP_COMP     (1 << 18)  /* Create compound page */
#define __GFP_NOWARN   (1 << 13)  /* Suppress allocation failure warnings */
#define __GFP_ACCOUNT  (1 << 22)  /* Account to memory cgroup */
```

### Common Compound Flags

| Flag | Definition | Use Case |
|------|------------|----------|
| `GFP_ATOMIC` | `__GFP_HIGH \| __GFP_KSWAPD_RECLAIM` | Interrupt context, cannot sleep |
| `GFP_KERNEL` | `__GFP_RECLAIM \| __GFP_IO \| __GFP_FS` | Normal kernel allocations |
| `GFP_NOWAIT` | `__GFP_KSWAPD_RECLAIM \| __GFP_NOWARN` | Non-blocking, may fail |
| `GFP_NOIO` | `__GFP_RECLAIM` | Block layer, no I/O during reclaim |
| `GFP_NOFS` | `__GFP_RECLAIM \| __GFP_IO` | Filesystem, no FS callbacks |
| `GFP_USER` | `GFP_KERNEL \| __GFP_HARDWALL` | Userspace allocations |
| `GFP_HIGHUSER_MOVABLE` | `GFP_USER \| __GFP_HIGHMEM \| __GFP_MOVABLE` | Typical user pages |

### GFP to Allocation Flags Conversion

```c
static inline unsigned int gfp_to_alloc_flags(gfp_t gfp_mask, unsigned int order)
{
    unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;

    /* __GFP_HIGH grants access to 50% of min watermark */
    alloc_flags |= (gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM));

    if (!(gfp_mask & __GFP_DIRECT_RECLAIM)) {
        /* Non-blocking: access 25% of min watermark */
        if (!(gfp_mask & __GFP_NOMEMALLOC)) {
            alloc_flags |= ALLOC_NON_BLOCK;
            if (order > 0)
                alloc_flags |= ALLOC_HIGHATOMIC;
        }
    }

    return alloc_flags;
}
```

---

## 4. NUMA Memory Management

### struct pglist_data (pg_data_t)

The pglist_data structure (commonly accessed via the pg_data_t typedef)
represents a NUMA node's complete memory description. On NUMA systems, each
node has its own pg_data_t containing all zones, memory statistics, and reclaim
state for that node. Even on UMA systems, a single pg_data_t describes all
physical memory.

This structure contains the node's zone array, zonelists for allocation
fallback ordering, and per-node LRU lists via the embedded lruvec. Each node
runs its own kswapd daemon for background reclaim, tracked in the kswapd field.
The node boundaries are defined by node_start_pfn and the various page count
fields, while vm_stat provides per-node memory statistics for monitoring and
tuning.

```c
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];       /* All zones in node */
    struct zonelist node_zonelists[MAX_ZONELISTS]; /* Allocation fallback */
    int nr_zones;                               /* Populated zones count */

    unsigned long node_start_pfn;               /* First PFN in node */
    unsigned long node_present_pages;           /* Physical pages */
    unsigned long node_spanned_pages;           /* Including holes */
    int node_id;

    /* Reclaim */
    wait_queue_head_t kswapd_wait;
    struct task_struct *kswapd;                 /* Per-node reclaim daemon */

    /* LRU lists */
    struct lruvec __lruvec;

    /* Statistics */
    atomic_long_t vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```

### Zonelist and Fallback

The zoneref and zonelist structures implement the allocation fallback mechanism
that determines where memory comes from when the preferred zone is exhausted. A
zonelist contains an ordered array of zone references, allowing the allocator
to try zones in preference order until one satisfies the allocation request.

Each NUMA node maintains two zonelists: ZONELIST_FALLBACK for normal
allocations that can fall back to remote nodes, and ZONELIST_NOFALLBACK for
allocations restricted to the local node. The zoneref structure pairs a zone
pointer with its index for efficient iteration. The fallback order is built at
boot time based on NUMA distances, preferring local zones before remote ones.

```c
struct zoneref {
    struct zone *zone;    /* Pointer to zone */
    int zone_idx;         /* Zone index */
};

struct zonelist {
    struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};
```

**Fallback Order** (built at boot):
```
Node 0: Normal → DMA32 → DMA →
Node 1: Normal → DMA32 → DMA →  (ordered by distance)
Node 2: ...
```

Two zonelists per node:
- `ZONELIST_FALLBACK` - Normal allocation with cross-node fallback
- `ZONELIST_NOFALLBACK` - For `__GFP_THISNODE`, no cross-node

### Memory Policies

The mempolicy structure defines NUMA memory placement policies for processes or
memory regions. These policies control which NUMA nodes are preferred or
allowed for memory allocations, enabling applications to optimize memory
locality for their access patterns.

The mode field specifies the placement strategy (bind, interleave, preferred,
etc.), while the nodes bitmask identifies the target NUMA nodes. Policies can
be set at the process level via set_mempolicy() or per-VMA via mbind(). The
kernel consults the active policy during page allocation to select the
appropriate node, with home_node providing the primary preference for policies
that support fallback behavior.

```c
struct mempolicy {
    atomic_t refcnt;
    unsigned short mode;      /* MPOL_* mode */
    unsigned short flags;     /* MPOL_F_* flags */
    nodemask_t nodes;         /* Target nodes */
    int home_node;            /* Preferred node for BIND/PREFERRED_MANY */
};
```

**Policy Modes**:

| Mode | Behavior |
|------|----------|
| `MPOL_DEFAULT` | Local node first, fallback via zonelist |
| `MPOL_BIND` | Restrict to specified nodes only |
| `MPOL_PREFERRED` | Try first node, fallback normally |
| `MPOL_PREFERRED_MANY` | Try node set, fallback normally |
| `MPOL_INTERLEAVE` | Round-robin across nodes |
| `MPOL_WEIGHTED_INTERLEAVE` | Weighted round-robin |
| `MPOL_LOCAL` | Force local allocation |

### NUMA Balancing

Automatic page migration based on access patterns:

```c
/* Record NUMA fault for balancing decisions */
void task_numa_fault(int last_node, int node, int pages, int flags);

/* Decide if migration is beneficial */
bool should_numa_migrate_memory(struct task_struct *p, struct folio *folio,
                                int src_nid, int dst_cpu);
```

**NUMA Statistics** (per zone):
- `NUMA_HIT` - Allocated from intended node
- `NUMA_MISS` - Allocated from different node
- `NUMA_LOCAL` - Allocated from local node
- `NUMA_OTHER` - Allocated from remote node

---

## 5. The Slab Allocator (SLUB)

SLUB is the default slab allocator for fixed-size object caches.

### struct kmem_cache

The kmem_cache structure is the central descriptor for a SLUB slab cache,
managing allocations of fixed-size objects. Each cache is optimized for a
specific object size and maintains per-CPU and per-node data structures to
minimize lock contention and improve cache locality.

The cpu_slab field points to per-CPU structures enabling lockless fast-path
allocation. The size and object_size fields track the actual allocation size
(with alignment and metadata) and the requested object size. The oo field
encodes both the page order and objects-per-slab for optimal memory
utilization. The optional constructor (ctor) initializes objects when slabs are
created, while the node array provides per-NUMA-node partial slab lists.

```c
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  /* Per-CPU fast path */

    slab_flags_t flags;
    unsigned long min_partial;     /* Min partial slabs to keep */
    unsigned int size;             /* Object size with metadata */
    unsigned int object_size;      /* Actual object size */
    unsigned int offset;           /* Free pointer offset in object */

    struct kmem_cache_order_objects oo;  /* Order and objects/slab */
    gfp_t allocflags;

    void (*ctor)(void *object);    /* Constructor */
    const char *name;

    struct kmem_cache_node *node[MAX_NUMNODES];  /* Per-node lists */
};
```

### struct kmem_cache_cpu (Per-CPU Fast Path)

The kmem_cache_cpu structure provides the per-CPU fast path for SLUB
allocations, enabling most allocations to complete without any locking. Each
CPU maintains its own instance containing a freelist pointer, the current
active slab, and a list of partial slabs.

The freelist field points to the next available free object, while the tid
(transaction ID) enables lockless cmpxchg-based allocation that detects
preemption or CPU migration. The slab field points to the currently active slab
(marked as "frozen" to prevent other CPUs from modifying it). The partial list
provides additional slabs when the current one is exhausted, reducing the
frequency of slower node-level operations.

```c
struct kmem_cache_cpu {
    union {
        struct {
            void **freelist;       /* Next free object */
            unsigned long tid;     /* Transaction ID for cmpxchg */
        };
        freelist_aba_t freelist_tid;  /* Atomic combined field */
    };
    struct slab *slab;             /* Current slab (frozen) */
    struct slab *partial;          /* Per-CPU partial list */
    local_lock_t lock;
};
```

### struct slab

The slab structure describes a single slab page (or compound page) containing
multiple objects of the same type. It is embedded within the struct page/folio
of the underlying memory, reusing page structure fields when the page serves as
a slab.

Key fields include slab_cache linking back to the owning kmem_cache, freelist
pointing to the first free object within the slab, and counters tracking the
number of allocated (inuse) and total objects. The frozen bit indicates whether
the slab is currently owned by a CPU's fast path (preventing concurrent
modification). Slabs cycle through states: full (on node full list), partial
(on node or CPU partial list), or frozen (active CPU slab).

```c
struct slab {
    unsigned long __page_flags;
    struct kmem_cache *slab_cache;

    union {
        struct {
            struct list_head slab_list;  /* Node partial/full list */
            void *freelist;              /* First free object */
            union {
                unsigned long counters;
                struct {
                    unsigned inuse:16;   /* Allocated objects */
                    unsigned objects:15; /* Total objects */
                    unsigned frozen:1;   /* CPU slab flag */
                };
            };
        };
        struct slab *next;              /* CPU partial list link */
    };
};
```

### Allocation Fast Path

```c
static __always_inline void *slab_alloc_node(struct kmem_cache *s, ...)
{
    struct kmem_cache_cpu *c;
    void *object;
    unsigned long tid;

redo:
    /* Read CPU slab and transaction ID */
    c = raw_cpu_ptr(s->cpu_slab);
    tid = READ_ONCE(c->tid);

    barrier();

    object = c->freelist;
    slab = c->slab;

    if (unlikely(!object || !slab || /* other checks */)) {
        return __slab_alloc(s, gfpflags, node, addr, c);  /* Slow path */
    }

    next_object = get_freepointer_safe(s, object);

    /* Atomic update - detects preemption/migration */
    if (!__update_cpu_freelist_fast(s, object, next_object, tid))
        goto redo;

    return object;
}
```

### kmalloc Size Classes

```c
/* Pre-created caches for common sizes */
const struct kmalloc_info_struct kmalloc_info[] = {
    {0, 0},        /* Index 0: special */
    {96, "96"},    /* Odd size for efficiency */
    {192, "192"},
    {8, "8"},      {16, "16"},    {32, "32"},    {64, "64"},
    {128, "128"},  {256, "256"},  {512, "512"},  {1024, "1k"},
    {2048, "2k"},  {4096, "4k"},  {8192, "8k"},  {16384, "16k"},
    /* ... up to 2MB */
};

/* Fast kmalloc with compile-time size optimization */
static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
    if (__builtin_constant_p(size) && size <= KMALLOC_MAX_CACHE_SIZE) {
        unsigned int index = kmalloc_index(size);
        return kmem_cache_alloc(kmalloc_caches[kmalloc_type(flags)][index],
                               flags);
    }
    return __kmalloc(size, flags);
}
```

### Freelist Encoding (Security Hardening)

```c
/* Encode freelist pointer to prevent exploitation */
static inline freeptr_t freelist_ptr_encode(const struct kmem_cache *s,
                                            void *ptr, unsigned long ptr_addr)
{
    return (freeptr_t){.v = (unsigned long)ptr ^ s->random ^ swab(ptr_addr)};
}
```

---

## 6. Folio Abstraction

A folio represents a physically contiguous set of pages (power-of-two size, at
least PAGE_SIZE).

### struct folio

The folio structure is a modern abstraction representing one or more physically
contiguous pages managed as a single unit. Introduced to replace ambiguous
struct page pointers, a folio is always the head of a potentially compound page
and provides type safety by ensuring code operates on complete memory units
rather than arbitrary tail pages.

The structure is designed to be binary-compatible with struct page for
single-page folios, sharing the same initial layout including flags, LRU list
linkage, mapping, and reference counts. For large folios (multiple pages),
additional fields in tail page locations track the large mapcount, entire
mapcount (for huge page mappings), and pin count for DMA operations. Folios are
the primary unit for page cache operations, LRU management, and memory reclaim.

```c
struct folio {
    union {
        struct {
            unsigned long flags;
            struct list_head lru;
            struct address_space *mapping;
            pgoff_t index;
            union {
                void *private;
                swp_entry_t swap;
            };
            atomic_t _mapcount;
            atomic_t _refcount;
        };
        struct page page;  /* Binary compatible with struct page */
    };

    /* Large folio fields (in tail pages) */
    union {
        struct {
            unsigned long _flags_1;
            unsigned long _head_1;
            atomic_t _large_mapcount;   /* Large folio mappings */
            atomic_t _entire_mapcount;  /* PMD/PUD mappings */
            atomic_t _nr_pages_mapped;
            atomic_t _pincount;         /* DMA pin count */
            unsigned int _folio_nr_pages; /* Page count (64-bit) */
        };
        struct page __page_1;
    };
};
```

### Key Folio Operations

```c
/* Allocation */
struct folio *folio_alloc(gfp_t gfp, unsigned int order);

/* Reference counting */
void folio_get(struct folio *folio);    /* Increment refcount */
void folio_put(struct folio *folio);    /* Decrement, free if zero */

/* Size queries */
unsigned int folio_order(const struct folio *folio);  /* log2(pages) */
long folio_nr_pages(const struct folio *folio);       /* Number of pages */
size_t folio_size(const struct folio *folio);         /* Size in bytes */

/* Conversions */
struct folio *page_folio(struct page *page);  /* Get containing folio */
struct page *folio_page(struct folio *folio, int n);  /* Get nth page */

/* Flag testing */
bool folio_test_large(const struct folio *folio);  /* Multi-page? */
bool folio_test_locked(const struct folio *folio);
bool folio_test_dirty(const struct folio *folio);
```

### Why Folios?

1. **Type Safety**: `struct folio *` is always the head of a compound page
2. **Unified API**: Single abstraction for single and multi-page allocations
3. **Page Cache**: Primary unit for filesystem operations
4. **Reduced Overhead**: One metadata structure for multiple pages
5. **Large Page Support**: First-class support for THP

---

## 7. Virtual Memory Areas (VMAs)

### struct vm_area_struct

The vm_area_struct (VMA) describes a contiguous region of a process's virtual
address space with uniform properties. Each VMA defines a range from vm_start
to vm_end with consistent permissions, backing (file or anonymous), and
behavior. A process's address space consists of multiple non-overlapping VMAs
organized in a maple tree for efficient lookup.

Key fields include vm_mm linking to the owning address space, vm_flags
containing protection and behavior flags, and vm_ops providing file-system or
device-specific fault handlers. For file-backed mappings, vm_file and vm_pgoff
identify the backing file and offset. For anonymous memory, anon_vma enables
reverse mapping to find all PTEs referencing a page. The vm_page_prot field
holds architecture-specific page table protection bits derived from vm_flags.

```c
struct vm_area_struct {
    /* Address range: [vm_start, vm_end) */
    unsigned long vm_start;
    unsigned long vm_end;

    struct mm_struct *vm_mm;           /* Owning address space */
    pgprot_t vm_page_prot;             /* Access permissions */
    const vm_flags_t vm_flags;         /* VMA flags */

    /* Anonymous memory tracking */
    struct list_head anon_vma_chain;
    struct anon_vma *anon_vma;

    /* Operations */
    const struct vm_operations_struct *vm_ops;

    /* File mapping */
    unsigned long vm_pgoff;            /* Offset in PAGE_SIZE units */
    struct file *vm_file;              /* Mapped file (NULL for anon) */
    void *vm_private_data;

    /* NUMA policy */
    struct mempolicy *vm_policy;
};
```

### VMA Flags

```c
#define VM_READ         0x00000001     /* Readable */
#define VM_WRITE        0x00000002     /* Writable */
#define VM_EXEC         0x00000004     /* Executable */
#define VM_SHARED       0x00000008     /* Shared mapping */

#define VM_MAYREAD      0x00000010     /* Can be made readable */
#define VM_MAYWRITE     0x00000020     /* Can be made writable */
#define VM_MAYEXEC      0x00000040     /* Can be made executable */
#define VM_MAYSHARE     0x00000080     /* Can be made shared */

#define VM_GROWSDOWN    0x00000100     /* Stack grows down */
#define VM_PFNMAP       0x00000400     /* Page-frame-number mapping */
#define VM_LOCKED       0x00002000     /* Pages mlocked */
#define VM_HUGETLB      0x00400000     /* Huge TLB page */
```

### struct vm_operations_struct

The vm_operations_struct provides a set of callbacks that customize VMA
behavior for different memory types. File systems, device drivers, and special
memory regions implement these operations to handle mapping lifecycle events,
page faults, and permission changes specific to their needs.

The most critical callback is fault(), which handles page faults by allocating
or locating the appropriate page for a virtual address. The huge_fault()
variant handles transparent huge page faults. The map_pages() callback enables
readahead by mapping multiple pages in a single fault. Other callbacks handle
VMA lifecycle events (open, close), permission changes (mprotect), and
copy-on-write notifications (page_mkwrite for file-backed pages becoming
writable).

```c
struct vm_operations_struct {
    void (*open)(struct vm_area_struct *area);
    void (*close)(struct vm_area_struct *area);

    int (*may_split)(struct vm_area_struct *area, unsigned long addr);
    int (*mremap)(struct vm_area_struct *area);
    int (*mprotect)(struct vm_area_struct *vma, unsigned long start,
                    unsigned long end, unsigned long newflags);

    /* Page fault handler */
    vm_fault_t (*fault)(struct vm_fault *vmf);
    vm_fault_t (*huge_fault)(struct vm_fault *vmf, unsigned int order);

    /* Readahead optimization */
    vm_fault_t (*map_pages)(struct vm_fault *vmf,
                           pgoff_t start_pgoff, pgoff_t end_pgoff);

    /* Copy-on-write notification */
    vm_fault_t (*page_mkwrite)(struct vm_fault *vmf);
    vm_fault_t (*pfn_mkwrite)(struct vm_fault *vmf);
};
```

### struct mm_struct

The mm_struct represents a complete process address space, containing all
virtual memory state for a process or group of threads sharing memory. Every
user process has an associated mm_struct that describes its virtual address
layout, page tables, and memory statistics.

The mm_mt maple tree stores all VMAs for efficient address-to-VMA lookup. The
pgd field points to the top-level page table (Page Global Directory). Memory
accounting fields track various categories: total_vm for total mapped pages,
locked_vm for mlocked pages, and RSS counters for resident set size by type.
The mmap_lock semaphore protects VMA operations, while page_table_lock protects
page table modifications. Code and data segment boundaries (start_code,
end_code, etc.) define the process memory layout.

```c
struct mm_struct {
    struct maple_tree mm_mt;           /* VMA tree */

    unsigned long mmap_base;           /* Base of mmap area */
    unsigned long task_size;           /* Size of task vm space */
    pgd_t *pgd;                        /* Page Global Directory */

    atomic_t mm_users;                 /* Process count */
    atomic_t mm_count;                 /* Reference count */

    spinlock_t page_table_lock;        /* Page table lock */
    struct rw_semaphore mmap_lock;     /* VMA tree lock */

    /* Memory statistics */
    unsigned long total_vm;            /* Total mapped pages */
    unsigned long locked_vm;           /* Mlocked pages */
    atomic64_t pinned_vm;              /* Pinned pages */
    unsigned long data_vm;             /* Data segment pages */
    unsigned long exec_vm;             /* Executable pages */
    unsigned long stack_vm;            /* Stack pages */

    /* Code/data boundaries */
    unsigned long start_code, end_code;
    unsigned long start_data, end_data;
    unsigned long start_brk, brk;
    unsigned long start_stack;

    /* RSS counters */
    struct percpu_counter rss_stat[NR_MM_COUNTERS];

    mm_context_t context;              /* Arch-specific (TLB gen, etc.) */
};
```

---

## 8. Page Tables and Address Translation

### Page Table Hierarchy

Linux supports up to 5 levels of page tables:

```
Virtual Address (48-bit, 4-level example):
┌─────────┬─────────┬─────────┬─────────┬─────────────┐
│ PGD[9]  │ PUD[9]  │ PMD[9]  │ PTE[9]  │ Offset[12]  │
│ 47...39 │ 38...30 │ 29...21 │ 20...12 │ 11...0      │
└─────────┴─────────┴─────────┴─────────┴─────────────┘
     ↓          ↓          ↓          ↓
   pgd_t     pud_t      pmd_t      pte_t → Physical Page
```

### Page Table Entry Types (x86-64)

```c
typedef struct { unsigned long pte; } pte_t;
typedef struct { unsigned long pmd; } pmd_t;
typedef struct { unsigned long pud; } pud_t;
typedef struct { unsigned long p4d; } p4d_t;
typedef struct { unsigned long pgd; } pgd_t;
```

### PTE Bit Layout (x86-64)

```c
/* Bit positions */
#define _PAGE_BIT_PRESENT     0    /* Page in memory */
#define _PAGE_BIT_RW          1    /* Writable */
#define _PAGE_BIT_USER        2    /* User-accessible */
#define _PAGE_BIT_PWT         3    /* Page write-through */
#define _PAGE_BIT_PCD         4    /* Page cache disabled */
#define _PAGE_BIT_ACCESSED    5    /* CPU has accessed */
#define _PAGE_BIT_DIRTY       6    /* CPU has written */
#define _PAGE_BIT_PSE         7    /* Page Size Extension (huge page) */
#define _PAGE_BIT_GLOBAL      8    /* Global TLB entry */
#define _PAGE_BIT_NX          63   /* No execute */

/* Flag macros */
#define _PAGE_PRESENT   (1UL << 0)
#define _PAGE_RW        (1UL << 1)
#define _PAGE_USER      (1UL << 2)
#define _PAGE_ACCESSED  (1UL << 5)
#define _PAGE_DIRTY     (1UL << 6)
#define _PAGE_PSE       (1UL << 7)   /* 2MB/1GB page */
#define _PAGE_GLOBAL    (1UL << 8)
#define _PAGE_NX        (1UL << 63)

/* PTE format:
 * Bits [11:0]:  Flags
 * Bits [51:12]: Physical Page Frame Number (PFN)
 * Bits [62:59]: Protection keys (if supported)
 * Bit  [63]:    NX (No Execute)
 */
```

### Page Table Operations

```c
/* Index calculation */
static inline unsigned long pgd_index(unsigned long addr) {
    return (addr >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1);
}
static inline unsigned long pud_index(unsigned long addr) {
    return (addr >> PUD_SHIFT) & (PTRS_PER_PUD - 1);
}
static inline unsigned long pmd_index(unsigned long addr) {
    return (addr >> PMD_SHIFT) & (PTRS_PER_PMD - 1);
}
static inline unsigned long pte_index(unsigned long addr) {
    return (addr >> PAGE_SHIFT) & (PTRS_PER_PTE - 1);
}

/* Entry pointer calculation */
pgd_t *pgd_offset(struct mm_struct *mm, unsigned long addr);
p4d_t *p4d_offset(pgd_t *pgd, unsigned long addr);
pud_t *pud_offset(p4d_t *p4d, unsigned long addr);
pmd_t *pmd_offset(pud_t *pud, unsigned long addr);
pte_t *pte_offset_kernel(pmd_t *pmd, unsigned long addr);

/* Entry manipulation */
static inline pte_t mk_pte(struct page *page, pgprot_t prot);
static inline pte_t pte_mkwrite(pte_t pte);
static inline pte_t pte_mkdirty(pte_t pte);
static inline pte_t pte_mkyoung(pte_t pte);

/* Entry testing */
static inline int pte_present(pte_t pte);
static inline int pte_write(pte_t pte);
static inline int pte_dirty(pte_t pte);
static inline int pte_young(pte_t pte);
```

### Page Table Allocation

```c
/* Allocate and install page table levels */
int __pte_alloc(struct mm_struct *mm, pmd_t *pmd);
int __pmd_alloc(struct mm_struct *mm, pud_t *pud, unsigned long addr);
int __pud_alloc(struct mm_struct *mm, p4d_t *p4d, unsigned long addr);
int __p4d_alloc(struct mm_struct *mm, pgd_t *pgd, unsigned long addr);

/* Macros that check before allocating */
#define pte_alloc(mm, pmd) \
    (unlikely(pmd_none(*(pmd))) && __pte_alloc(mm, pmd))

#define pmd_alloc(mm, pud, addr) \
    (unlikely(pud_none(*(pud))) && __pmd_alloc(mm, pud, addr))
```

### Memory Ordering for Page Tables

```c
static void pmd_install(struct mm_struct *mm, pmd_t *pmd, pgtable_t *pte)
{
    spinlock_t *ptl = pmd_lock(mm, pmd);

    if (pmd_none(*pmd)) {
        mm_inc_nr_ptes(mm);

        /* Ensure PTE table setup visible before PMD update */
        smp_wmb();

        pmd_populate(mm, pmd, *pte);
        *pte = NULL;
    }

    spin_unlock(ptl);
}
```

---

## 9. Page Table Walking

### Generic Walk Framework

The mm_walk_ops and mm_walk structures provide a generic framework for
traversing page tables. This abstraction allows various kernel subsystems
(memory reclaim, migration, debugging) to walk page tables using custom
callbacks without duplicating traversal logic.

The mm_walk_ops structure defines callbacks for each page table level
(pgd_entry through pte_entry), holes in the address space (pte_hole), and huge
page entries (hugetlb_entry). The mm_walk structure carries walk state
including the target mm, current VMA, and private data for the callbacks. The
walk_page_range() function drives the traversal, calling appropriate callbacks
as it descends through page table levels.

```c
struct mm_walk_ops {
    int (*pgd_entry)(pgd_t *pgd, unsigned long addr,
                     unsigned long next, struct mm_walk *walk);
    int (*p4d_entry)(p4d_t *p4d, ...);
    int (*pud_entry)(pud_t *pud, ...);
    int (*pmd_entry)(pmd_t *pmd, ...);
    int (*pte_entry)(pte_t *pte, unsigned long addr,
                     unsigned long next, struct mm_walk *walk);

    /* Called for missing page table entries */
    int (*pte_hole)(unsigned long addr, unsigned long next,
                    int depth, struct mm_walk *walk);

    /* Hugetlb handling */
    int (*hugetlb_entry)(pte_t *pte, unsigned long hmask,
                        unsigned long addr, unsigned long next,
                        struct mm_walk *walk);

    /* VMA filtering */
    int (*test_walk)(unsigned long addr, unsigned long next,
                     struct mm_walk *walk);

    enum page_walk_lock walk_lock;  /* Locking requirement */
};

struct mm_walk {
    const struct mm_walk_ops *ops;
    struct mm_struct *mm;
    pgd_t *pgd;
    struct vm_area_struct *vma;
    enum page_walk_action action;
    bool no_vma;
    void *private;
};

/* Main walk function */
int walk_page_range(struct mm_struct *mm, unsigned long start,
                    unsigned long end, const struct mm_walk_ops *ops,
                    void *private);
```

### PTE Locking

```c
/* Map PTE and acquire lock */
pte_t *pte_offset_map_lock(struct mm_struct *mm, pmd_t *pmd,
                           unsigned long addr, spinlock_t **ptlp)
{
    pte_t *pte;
    pmdval_t pmdval;

again:
    pte = __pte_offset_map(pmd, addr, &pmdval);
    if (!pte)
        return NULL;

    *ptlp = pte_lockptr(mm, &pmdval);
    spin_lock(*ptlp);

    /* Verify PMD unchanged after acquiring lock */
    if (unlikely(!pmd_same(pmdval, pmdp_get_lockless(pmd)))) {
        spin_unlock(*ptlp);
        pte_unmap(pte);
        goto again;
    }

    return pte;
}

/* Release lock and unmap */
#define pte_unmap_unlock(pte, ptl) do { \
    spin_unlock(ptl);                   \
    pte_unmap(pte);                     \
} while (0)
```

### Get User Pages (GUP)

```c
/* Slow path with faulting */
long get_user_pages(unsigned long start, unsigned long nr_pages,
                    unsigned int gup_flags, struct page **pages);

/* Fast lockless path */
int get_user_pages_fast(unsigned long start, int nr_pages,
                        unsigned int gup_flags, struct page **pages);

/* Pin pages for DMA */
long pin_user_pages(unsigned long start, unsigned long nr_pages,
                    unsigned int gup_flags, struct page **pages);

/* GUP Flags */
#define FOLL_WRITE      0x01   /* Need write access */
#define FOLL_GET        0x04   /* Increment page refcount */
#define FOLL_FORCE      0x10   /* Override permissions (ptrace) */
#define FOLL_PIN        0x800  /* Pin for DMA (tracked separately) */
#define FOLL_LONGTERM   0x1000 /* Long-term pin */
```

### GUP Fast Path (Lockless)

```c
static int gup_fast_pte_range(pmd_t pmd, pmd_t *pmdp, unsigned long addr,
                              unsigned long end, unsigned int flags,
                              struct page **pages, int *nr)
{
    pte_t *ptep, pte;

    ptep = pte_offset_map(&pmd, addr);

    do {
        pte = ptep_get_lockless(ptep);

        if (!pte_access_permitted(pte, flags & FOLL_WRITE))
            return 0;  /* Fallback to slow path */

        page = pte_page(pte);

        if (!try_grab_folio_fast(page, 1, flags))
            return 0;

        /* Verify PTE unchanged after grabbing reference */
        if (unlikely(pte_val(pte) != pte_val(ptep_get(ptep)))) {
            gup_put_folio(folio, 1, flags);
            return 0;  /* Race detected, fallback */
        }

        pages[*nr] = page;
        (*nr)++;
    } while (ptep++, addr += PAGE_SIZE, addr != end);

    pte_unmap(ptep - 1);
    return 1;
}
```

### follow_page() for Page Lookup

```c
/* Look up page without faulting */
struct page *follow_page_mask(struct vm_area_struct *vma,
                              unsigned long address,
                              unsigned int flags,
                              struct follow_page_context *ctx)
{
    pgd_t *pgd = pgd_offset(mm, address);

    if (pgd_none(*pgd) || pgd_bad(*pgd))
        return no_page_table(vma, flags, address);

    return follow_p4d_mask(vma, address, pgd, flags, ctx);
    /* → follow_pud_mask → follow_pmd_mask → follow_page_pte */
}
```

---

## 10. TLB Management

### TLB Flush Functions

```c
/* Flush entire TLB on all CPUs */
void flush_tlb_all(void);

/* Flush entire mm on all relevant CPUs */
void flush_tlb_mm(struct mm_struct *mm);

/* Flush address range */
void flush_tlb_range(struct vm_area_struct *vma,
                     unsigned long start, unsigned long end);

/* Flush single page */
void flush_tlb_page(struct vm_area_struct *vma, unsigned long addr);

/* Flush kernel address range */
void flush_tlb_kernel_range(unsigned long start, unsigned long end);
```

### struct mmu_gather (Batch TLB Invalidation)

The mmu_gather structure collects page table entries and pages for batched TLB
invalidation and freeing. When unmapping memory ranges, individual TLB flushes
for each page would be prohibitively expensive, so the kernel accumulates
    changes and performs a single flush covering the entire range.

The structure tracks the address range being unmapped (start, end), flags
indicating which page table levels were modified, and batched lists of pages to
free after the TLB flush. The fullmm flag indicates an entire address space is
being torn down (allowing optimization). The freed_tables flag indicates page
table pages were freed, requiring more aggressive TLB invalidation. After the
TLB flush completes, accumulated pages are returned to the allocator.

```c
struct mmu_gather {
    struct mm_struct *mm;

    unsigned long start;       /* Range start */
    unsigned long end;         /* Range end */

    /* Flags */
    unsigned int fullmm : 1;        /* Full mm flush */
    unsigned int need_flush_all : 1;
    unsigned int freed_tables : 1;  /* Page tables freed */
    unsigned int delayed_rmap : 1;

    /* Page table level tracking */
    unsigned int cleared_ptes : 1;
    unsigned int cleared_pmds : 1;
    unsigned int cleared_puds : 1;
    unsigned int cleared_p4ds : 1;

    /* Batched pages for freeing */
    struct mmu_gather_batch *active;
    struct mmu_gather_batch local;
    struct page *__pages[MMU_GATHER_BUNDLE];
};

/* Usage pattern */
void tlb_gather_mmu(struct mmu_gather *tlb, struct mm_struct *mm);
void tlb_remove_page(struct mmu_gather *tlb, struct page *page);
void tlb_flush_mmu(struct mmu_gather *tlb);  /* Flush and free */
void tlb_finish_mmu(struct mmu_gather *tlb);
```

### TLB Shootdown (x86)

The flush_tlb_info structure carries TLB invalidation parameters for
inter-processor interrupts (IPIs) during TLB shootdown operations. When one CPU
modifies page tables, it must notify other CPUs that may have cached the old
translations to invalidate their TLB entries.

This structure specifies the target mm, address range, and the new TLB
generation number for synchronization. The stride_shift field indicates the
page size for strided flushes. The initiating_cpu field identifies the sender
to avoid self-IPI. The kernel uses TLB generation numbers to track whether a
CPU's cached entries are stale, allowing lazy TLB mode CPUs to skip immediate
flushes and invalidate on their next context switch instead.

```c
/* IPI-based remote TLB flush */
void flush_tlb_multi(const struct cpumask *cpumask,
                     const struct flush_tlb_info *info);

struct flush_tlb_info {
    struct mm_struct *mm;
    unsigned long start;
    unsigned long end;
    u64 new_tlb_gen;           /* TLB generation for sync */
    unsigned int initiating_cpu;
    u8 stride_shift;           /* Page size shift */
    u8 freed_tables;           /* Page tables were freed */
};

/* IPI handler on each CPU */
static void flush_tlb_func(void *info)
{
    const struct flush_tlb_info *f = info;
    u64 local_tlb_gen = this_cpu_read(cpu_tlbstate.ctxs[asid].tlb_gen);

    /* Check if already up to date */
    if (f->new_tlb_gen <= local_tlb_gen)
        return;

    /* Partial or full flush */
    if (/* conditions for partial */) {
        for (addr = f->start; addr < f->end; addr += stride)
            flush_tlb_one_user(addr);
    } else {
        flush_tlb_local();  /* Full flush */
    }

    /* Update local TLB generation */
    this_cpu_write(cpu_tlbstate.ctxs[asid].tlb_gen, mm_tlb_gen);
}
```

### PCID/ASID Support (x86)

The tlb_state and tlb_context structures manage per-CPU TLB state for PCID
(Process Context Identifier) support. PCIDs allow the CPU to tag TLB entries
with an address space identifier, avoiding full TLB flushes on context switches
by preserving entries from other processes.

Each CPU maintains a small cache of recently used ASIDs in the ctxs array,
mapping context IDs to TLB generation numbers. The loaded_mm and loaded_mm_asid
fields track the currently active address space. When switching contexts,
choose_new_asid() searches for an existing ASID for the target mm or allocates
a new slot using round-robin replacement. The TLB generation tracking enables
deferred invalidation, where stale entries are flushed only when actually
needed.

```c
struct tlb_state {
    struct mm_struct *loaded_mm;     /* Current mm */
    u16 loaded_mm_asid;              /* Current ASID/PCID */
    u16 next_asid;                   /* Next ASID to allocate */
    bool invalidate_other;           /* Need to invalidate other ASIDs */

    /* ASID cache */
    struct tlb_context ctxs[TLB_NR_DYN_ASIDS];
};

struct tlb_context {
    u64 ctx_id;    /* mm->context.ctx_id */
    u64 tlb_gen;   /* mm->context.tlb_gen */
};

/* ASID allocation */
static void choose_new_asid(struct mm_struct *next, u64 next_tlb_gen,
                            u16 *new_asid, bool *need_flush)
{
    /* Search for existing ASID for this mm */
    for (asid = 0; asid < TLB_NR_DYN_ASIDS; asid++) {
        if (ctxs[asid].ctx_id == next->context.ctx_id) {
            *new_asid = asid;
            *need_flush = (ctxs[asid].tlb_gen < next_tlb_gen);
            return;
        }
    }

    /* Allocate new ASID slot (round-robin) */
    *new_asid = next_asid++;
    *need_flush = true;
}
```

### Lazy TLB Mode

```c
struct tlb_state_shared {
    bool is_lazy;  /* In lazy TLB mode */
};

/* Enter lazy TLB when switching to kernel thread */
void enter_lazy_tlb(struct mm_struct *mm, struct task_struct *tsk)
{
    if (this_cpu_read(cpu_tlbstate.loaded_mm) == &init_mm)
        return;

    /* Keep using previous mm's page tables, mark as lazy */
    this_cpu_write(cpu_tlbstate_shared.is_lazy, true);
}

/* Skip TLB flush IPIs to lazy CPUs */
static bool should_flush_tlb(int cpu, void *data)
{
    if (per_cpu(cpu_tlbstate_shared.is_lazy, cpu))
        return false;  /* Will flush on next context switch */

    return per_cpu(cpu_tlbstate.loaded_mm, cpu) == info->mm;
}
```

### Hardware TLB Instructions (x86)

```c
/* INVLPG - Invalidate single page */
static inline void invlpg(unsigned long addr) {
    asm volatile("invlpg (%0)" :: "r" (addr) : "memory");
}

/* INVPCID - Invalidate with PCID control */
#define INVPCID_TYPE_INDIV_ADDR     0  /* Single address in PCID */
#define INVPCID_TYPE_SINGLE_CTXT    1  /* All entries for PCID */
#define INVPCID_TYPE_ALL_INCL_GLOBAL 2 /* All entries, all PCIDs */
#define INVPCID_TYPE_ALL_NON_GLOBAL 3  /* All non-global entries */

static inline void invpcid_flush_one(unsigned long pcid, unsigned long addr);
static inline void invpcid_flush_single_context(unsigned long pcid);
static inline void invpcid_flush_all(void);

/* CR3 write - reloads page table base */
static inline void write_cr3(unsigned long val);

/* CR3_NOFLUSH - preserve TLB entries (PCID feature) */
#define CR3_NOFLUSH  (1UL << 63)

/* INVLPGB - Broadcast TLB invalidation (AMD) */
static inline void invlpgb_flush_all(void) {
    __invlpgb(0, 0, 0, 1, 0, INVLPGB_INCLUDE_GLOBAL);
    __tlbsync();  /* Wait for completion */
}
```

---

## 11. Page Fault Handling

### Fault Entry Point

```c
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                           unsigned int flags, struct pt_regs *regs)
{
    /* Validate VMA permissions */
    /* ... */

    return __handle_mm_fault(vma, address, flags);
}
```

### __handle_mm_fault - Page Table Walk

```c
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
                                    unsigned long address, unsigned int flags)
{
    struct vm_fault vmf = {
        .vma = vma,
        .address = address & PAGE_MASK,
        .flags = flags,
        .pgoff = linear_page_index(vma, address),
    };

    /* PGD */
    pgd = pgd_offset(mm, address);

    /* P4D - allocate if needed */
    p4d = p4d_alloc(mm, pgd, address);
    if (!p4d)
        return VM_FAULT_OOM;

    /* PUD - allocate if needed */
    vmf.pud = pud_alloc(mm, p4d, address);
    if (!vmf.pud)
        return VM_FAULT_OOM;

    /* Check for huge PUD (1GB pages) */
    if (pud_trans_huge(*vmf.pud) || pud_devmap(*vmf.pud))
        /* Handle huge page fault */

    /* PMD - allocate if needed */
    vmf.pmd = pmd_alloc(mm, vmf.pud, address);
    if (!vmf.pmd)
        return VM_FAULT_OOM;

    /* Check for huge PMD (2MB pages) */
    if (pmd_trans_huge(*vmf.pmd) || pmd_devmap(*vmf.pmd))
        /* Handle huge page fault */

    /* PTE level */
    return handle_pte_fault(&vmf);
}
```

### handle_pte_fault

```c
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;

    /* Map and read PTE */
    vmf->pte = pte_offset_map(vmf->pmd, vmf->address);
    entry = ptep_get(vmf->pte);

    if (!pte_present(entry)) {
        if (pte_none(entry)) {
            /* No mapping - create new page */
            if (vma->vm_ops && vma->vm_ops->fault)
                return do_fault(vmf);    /* File-backed */
            else
                return do_anonymous_page(vmf);  /* Anonymous */
        }
        /* Swap entry */
        return do_swap_page(vmf);
    }

    /* NUMA balancing */
    if (pte_protnone(entry))
        return do_numa_page(vmf);

    /* Write fault to read-only page */
    if ((vmf->flags & FAULT_FLAG_WRITE) && !pte_write(entry))
        return do_wp_page(vmf);  /* Copy-on-write */

    /* Update accessed bit */
    entry = pte_mkyoung(entry);
    if (vmf->flags & FAULT_FLAG_WRITE)
        entry = pte_mkdirty(entry);

    ptep_set_access_flags(vma, vmf->address, vmf->pte, entry, write);
    update_mmu_cache(vma, vmf->address, vmf->pte);

    pte_unmap_unlock(vmf->pte, vmf->ptl);
    return 0;
}
```

### Anonymous Page Allocation

```c
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct folio *folio;
    pte_t entry;

    /* Allocate PTE table if needed */
    if (pte_alloc(vma->vm_mm, vmf->pmd))
        return VM_FAULT_OOM;

    /* For read faults, use zero page */
    if (!(vmf->flags & FAULT_FLAG_WRITE)) {
        entry = pte_mkspecial(pfn_pte(my_zero_pfn(vmf->address),
                                      vma->vm_page_prot));
        goto setpte;
    }

    /* Allocate anonymous page */
    folio = alloc_anon_folio(vmf);
    if (!folio)
        return VM_FAULT_OOM;

    __folio_mark_uptodate(folio);

    /* Create PTE entry */
    entry = mk_pte(&folio->page, vma->vm_page_prot);
    entry = pte_sw_mkyoung(entry);
    if (vma->vm_flags & VM_WRITE)
        entry = pte_mkwrite(pte_mkdirty(entry), vma);

    /* Lock page table */
    vmf->pte = pte_offset_map_lock(vma->vm_mm, vmf->pmd,
                                    vmf->address, &vmf->ptl);

    /* Add to reverse mapping */
    folio_add_new_anon_rmap(folio, vma, vmf->address, RMAP_EXCLUSIVE);
    folio_add_lru_vma(folio, vma);

setpte:
    /* Install PTE */
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
    update_mmu_cache(vma, vmf->address, vmf->pte);

    pte_unmap_unlock(vmf->pte, vmf->ptl);
    return 0;
}
```

### Copy-On-Write (COW)

```c
static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct folio *folio = page_folio(vmf->page);

    /* Check if we can reuse the page (exclusive owner) */
    if (folio_reuse_one_vma(folio, vma)) {
        /* Make writable in place */
        wp_page_reuse(vmf, folio);
        return 0;
    }

    /* Must copy the page */
    return wp_page_copy(vmf);
}

static vm_fault_t wp_page_copy(struct vm_fault *vmf)
{
    struct folio *old_folio = page_folio(vmf->page);
    struct folio *new_folio;

    /* Allocate new page */
    new_folio = folio_alloc(GFP_HIGHUSER_MOVABLE, 0);
    if (!new_folio)
        return VM_FAULT_OOM;

    /* Copy contents */
    copy_user_highpage(&new_folio->page, vmf->page, vmf->address, vma);

    /* Add to rmap */
    folio_add_new_anon_rmap(new_folio, vma, vmf->address, RMAP_EXCLUSIVE);
    folio_add_lru_vma(new_folio, vma);

    /* Install new PTE */
    entry = mk_pte(&new_folio->page, vma->vm_page_prot);
    entry = pte_mkwrite(pte_mkdirty(entry), vma);

    ptep_clear_flush(vma, vmf->address, vmf->pte);
    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
    update_mmu_cache(vma, vmf->address, vmf->pte);

    /* Release old page */
    folio_put(old_folio);

    return 0;
}
```

---

## 12. Memory Reclaim and LRU

### LRU Lists

```c
enum lru_list {
    LRU_INACTIVE_ANON = 0,  /* Anonymous pages not recently accessed */
    LRU_ACTIVE_ANON   = 1,  /* Anonymous pages recently accessed */
    LRU_INACTIVE_FILE = 2,  /* File pages not recently accessed */
    LRU_ACTIVE_FILE   = 3,  /* File pages recently accessed */
    LRU_UNEVICTABLE   = 4,  /* Pages that cannot be evicted */
    NR_LRU_LISTS
};
```

### struct lruvec

The lruvec structure is the container for LRU (Least Recently Used) lists used
by the memory reclaim subsystem. It holds pages organized by their recency of
access, enabling the kernel to identify and reclaim the least recently used
pages when memory pressure occurs.

The lists array contains the five LRU lists: inactive anonymous, active
anonymous, inactive file, active file, and unevictable. The cost tracking
fields (anon_cost, file_cost) help balance reclaim pressure between anonymous
and file-backed pages. The refaults array tracks page cache refaults for
working set detection. For systems with MGLRU enabled, the lrugen field
provides the multi-generational LRU data structures that offer improved reclaim
efficiency.

```c
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];  /* LRU lists */
    spinlock_t lru_lock;

    /* Reclaim cost tracking */
    unsigned long anon_cost;
    unsigned long file_cost;

    /* Refault tracking */
    atomic_long_t nonresident_age;
    unsigned long refaults[ANON_AND_FILE];

    unsigned long flags;

    /* Multi-generational LRU */
    struct lru_gen_folio lrugen;
};
```

### Multi-Generational LRU (MGLRU)

The lru_gen_folio structure implements the multi-generational LRU algorithm,
which provides more accurate page aging than the traditional active/inactive
LRU by using multiple generations to track page access patterns over time.

Pages are organized into generations (up to MAX_NR_GENS) based on when they
were last accessed, with max_seq tracking the youngest generation and min_seq
tracking the oldest. The folios array is a three-dimensional structure
organizing pages by generation, type (anonymous vs file), and zone. Aging
increments max_seq and moves recently accessed pages to younger generations,
while eviction reclaims pages from the oldest generation. The tier-based
    refault tracking (avg_refaulted, avg_total) helps the algorithm adapt to
    workload characteristics.

```c
struct lru_gen_folio {
    /* Generation tracking */
    unsigned long max_seq;                 /* Youngest generation */
    unsigned long min_seq[ANON_AND_FILE];  /* Oldest generations */
    unsigned long timestamps[MAX_NR_GENS]; /* Birth times */

    /* Page lists by generation/type/zone */
    struct list_head folios[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
    long nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];

    /* Refault statistics */
    unsigned long avg_refaulted[ANON_AND_FILE][MAX_NR_TIERS];
    unsigned long avg_total[ANON_AND_FILE][MAX_NR_TIERS];

    bool enabled;
};

/* Constants */
#define MIN_NR_GENS  2  /* Minimum generations for second-chance */
#define MAX_NR_GENS  4  /* Maximum generations */
#define MAX_NR_TIERS 4  /* Tiers based on access frequency */
```

### struct scan_control

The scan_control structure carries parameters and state for a memory reclaim
operation. It is passed through the reclaim call stack, controlling which pages
can be reclaimed, how aggressively to scan, and tracking progress statistics.

Key control fields include nr_to_reclaim (target page count), priority (scan
intensity from 0-12), and flags like may_writepage, may_unmap, and may_swap
that constrain reclaim behavior based on context. The gfp_mask reflects the
original allocation's requirements. Statistics fields (nr_scanned,
nr_reclaimed) track progress, while the nr substructure counts pages in various
states (dirty, writeback, congested) that may affect reclaim decisions.

```c
struct scan_control {
    unsigned long nr_to_reclaim;   /* Target pages to reclaim */
    nodemask_t *nodemask;          /* Allowed nodes */
    struct mem_cgroup *target_mem_cgroup;

    /* Cost tracking */
    unsigned long anon_cost;
    unsigned long file_cost;

    /* Flags */
    unsigned int may_writepage:1;      /* Can write dirty pages */
    unsigned int may_unmap:1;          /* Can unmap pages */
    unsigned int may_swap:1;           /* Can swap anonymous pages */
    unsigned int proactive:1;          /* Proactive reclaim */
    unsigned int memcg_low_reclaim:1;
    unsigned int cache_trim_mode:1;

    s8 order;                          /* Allocation order */
    s8 priority;                       /* Scan priority (0-12) */
    s8 reclaim_idx;                    /* Highest zone to reclaim */

    gfp_t gfp_mask;

    /* Statistics */
    unsigned long nr_scanned;
    unsigned long nr_reclaimed;

    struct {
        unsigned int dirty;
        unsigned int unqueued_dirty;
        unsigned int congested;
        unsigned int writeback;
    } nr;
};
```

### kswapd - Background Reclaim

```c
static int kswapd(void *p)
{
    pg_data_t *pgdat = (pg_data_t *)p;

    current->flags |= PF_MEMALLOC | PF_KSWAPD;

    for (;;) {
        /* Sleep until woken */
        prepare_kswapd_sleep(pgdat, order, highest_zoneidx);
        schedule();

        /* Read allocation request */
        alloc_order = READ_ONCE(pgdat->kswapd_order);
        highest_zoneidx = READ_ONCE(pgdat->kswapd_highest_zoneidx);

        /* Reclaim until balanced */
        reclaim_order = balance_pgdat(pgdat, alloc_order, highest_zoneidx);
    }
}

/* Wake kswapd when zone drops below low watermark */
void wakeup_kswapd(struct zone *zone, gfp_t gfp_flags, int order,
                   enum zone_type highest_zoneidx)
{
    if (!managed_zone(zone))
        return;

    if (pgdat_balanced(pgdat, order, highest_zoneidx))
        return;

    wake_up_interruptible(&pgdat->kswapd_wait);
}
```

### Direct Reclaim

```c
unsigned long try_to_free_pages(struct zonelist *zonelist, int order,
                                gfp_t gfp_mask, nodemask_t *nodemask)
{
    struct scan_control sc = {
        .nr_to_reclaim = SWAP_CLUSTER_MAX,
        .gfp_mask = current_gfp_context(gfp_mask),
        .order = order,
        .nodemask = nodemask,
        .priority = DEF_PRIORITY,
        .may_writepage = !laptop_mode,
        .may_unmap = 1,
        .may_swap = 1,
    };

    /* Throttle if system under heavy I/O */
    throttle_direct_reclaim(gfp_mask);

    return do_try_to_free_pages(zonelist, &sc);
}
```

---

## 13. Swap Subsystem

### struct swap_info_struct

The swap_info_struct describes a single swap device or file, containing all
metadata needed to manage swap slot allocation and I/O. The kernel maintains an
array of these structures, one for each configured swap area.

The swap_map array tracks the reference count for each swap slot (how many PTEs
reference it). The cluster_info and related fields optimize allocation for SSDs
by grouping slots into clusters. The prio field determines the order in which
swap areas are used. For file-backed swap, swap_file points to the backing
file; for partition swap, bdev points to the block device. The swap_extent_root
maps logical swap offsets to physical disk blocks.

```c
struct swap_info_struct {
    struct percpu_ref users;
    unsigned long flags;              /* SWP_USED, SWP_WRITEOK, etc. */
    signed short prio;                /* Priority */
    signed char type;                 /* Index in swap_info[] */

    unsigned int max;                 /* Size of swap_map */
    unsigned char *swap_map;          /* Usage count per slot */
    unsigned long *zeromap;           /* Bitmap of zero-filled pages */

    struct swap_cluster_info *cluster_info;  /* For SSD optimization */
    struct list_head free_clusters;
    struct list_head full_clusters;

    unsigned int lowest_bit;          /* First free entry */
    unsigned int highest_bit;         /* Last free entry */
    unsigned int pages;               /* Usable pages */
    unsigned int inuse_pages;         /* Allocated pages */

    struct rb_root swap_extent_root;  /* Disk block mapping */
    struct block_device *bdev;        /* Block device (partition) */
    struct file *swap_file;           /* File (swapfile) */

    spinlock_t lock;
};
```

### Swap Entry Encoding

The swp_entry_t type encodes a reference to a swap slot in a format that can be
stored in page table entries when a page is swapped out. This compact encoding
combines the swap device type (index into swap_info array) and the offset
within that device into a single unsigned long value.

When a page is swapped out, its PTE is replaced with a swap entry that
identifies where the page contents reside. The encoding uses bit shifting to
pack both fields efficiently, with helper functions swp_type() and swp_offset()
to extract the components. Conversion functions pte_to_swp_entry() and
swp_entry_to_pte() handle the transformation between PTE format and swap entry
format.

```c
/* Swap entry encodes type and offset */
typedef struct {
    unsigned long val;
} swp_entry_t;

/* Create swap entry */
static inline swp_entry_t swp_entry(unsigned long type, pgoff_t offset)
{
    return (swp_entry_t){(type << SWP_TYPE_SHIFT) | offset};
}

/* Extract type and offset */
static inline unsigned swp_type(swp_entry_t entry) {
    return entry.val >> SWP_TYPE_SHIFT;
}
static inline pgoff_t swp_offset(swp_entry_t entry) {
    return entry.val & SWP_OFFSET_MASK;
}

/* Convert between PTE and swap entry */
static inline swp_entry_t pte_to_swp_entry(pte_t pte);
static inline pte_t swp_entry_to_pte(swp_entry_t entry);
```

### Swap Cache

```c
/* Partitioned address spaces for reduced lock contention */
#define SWAP_ADDRESS_SPACE_SHIFT 14
#define SWAP_ADDRESS_SPACE_PAGES (1 << SWAP_ADDRESS_SPACE_SHIFT)

struct address_space *swapper_spaces[MAX_SWAPFILES];

#define swap_address_space(entry) \
    (&swapper_spaces[swp_type(entry)][swp_offset(entry) >> SWAP_ADDRESS_SPACE_SHIFT])

/* Add folio to swap cache */
int add_to_swap_cache(struct folio *folio, swp_entry_t entry,
                      gfp_t gfp, void **shadowp);

/* Remove from swap cache */
void delete_from_swap_cache(struct folio *folio);

/* Look up in swap cache */
struct folio *swap_cache_get_folio(swp_entry_t entry,
                                   struct vm_area_struct *vma,
                                   unsigned long addr);
```

### Swap I/O

```c
/* Write page to swap */
int swap_writepage(struct page *page, struct writeback_control *wbc)
{
    struct folio *folio = page_folio(page);

    /* Check if can be freed without writing */
    if (folio_free_swap(folio)) {
        folio_unlock(folio);
        return 0;
    }

    /* Try zswap compression first */
    if (zswap_store(folio)) {
        folio_unlock(folio);
        return 0;
    }

    /* Check for zero page optimization */
    if (folio_is_zero_filled(folio)) {
        swap_zeromap_folio_set(folio);
        folio_unlock(folio);
        return 0;
    }

    __swap_writepage(folio, wbc);
    return 0;
}

/* Read page from swap */
void swap_read_folio(struct folio *folio, struct swap_iocb **plug)
{
    /* Check zeromap */
    if (swap_read_folio_zeromap(folio))
        return;

    /* Try zswap */
    if (zswap_load(folio))
        return;

    /* Read from disk */
    if (si->flags & SWP_FS_OPS)
        swap_read_folio_fs(folio, plug);
    else if (si->flags & SWP_SYNCHRONOUS_IO)
        swap_read_folio_bdev_sync(folio);
    else
        swap_read_folio_bdev_async(folio);
}
```

### Swap Page Fault

```c
vm_fault_t do_swap_page(struct vm_fault *vmf)
{
    swp_entry_t entry = pte_to_swp_entry(vmf->orig_pte);

    /* Handle special entries */
    if (is_migration_entry(entry))
        /* Wait for migration to complete */
    if (is_device_private_entry(entry))
        /* Call device migrate_to_ram() */
    if (is_hwpoison_entry(entry))
        return VM_FAULT_HWPOISON;

    /* Try swap cache first */
    folio = swap_cache_get_folio(entry, vma, vmf->address);

    if (!folio) {
        /* Synchronous path for SSDs */
        if ((si->flags & SWP_SYNCHRONOUS_IO) &&
            __swap_count(entry) == 1) {
            folio = alloc_swap_folio(vmf);
            swap_read_folio(folio, NULL);
        } else {
            /* Use readahead */
            folio = swapin_readahead(entry, gfp, vmf);
        }
    }

    /* Wait for I/O */
    folio_lock(folio);

    /* Map into page table */
    set_pte_at(mm, vmf->address, vmf->pte,
               mk_pte(&folio->page, vma->vm_page_prot));

    /* Update statistics */
    count_vm_event(PSWPIN);

    return 0;
}
```

---

## 14. Key Data Structures Reference

### Quick Reference Table

| Structure | Purpose | Key File |
|-----------|---------|----------|
| `struct page` | Physical page descriptor | `include/linux/mm_types.h` |
| `struct folio` | Multi-page container | `include/linux/mm_types.h` |
| `struct zone` | Memory zone descriptor | `include/linux/mmzone.h` |
| `pg_data_t` | NUMA node descriptor | `include/linux/mmzone.h` |
| `struct vm_area_struct` | Virtual memory region | `include/linux/mm_types.h` |
| `struct mm_struct` | Process address space | `include/linux/mm_types.h` |
| `struct kmem_cache` | Slab cache descriptor | `mm/slab.h` |
| `struct lruvec` | LRU list container | `include/linux/mmzone.h` |
| `struct mmu_gather` | Batch TLB invalidation | `include/asm-generic/tlb.h` |
| `struct swap_info_struct` | Swap device descriptor | `include/linux/swap.h` |

### Memory Allocation APIs

| Function | Purpose | Context |
|----------|---------|---------|
| `alloc_pages(gfp, order)` | Allocate 2^order pages | Any |
| `__get_free_pages(gfp, order)` | Allocate and return virtual address | Any |
| `kmalloc(size, gfp)` | Allocate size bytes | Any |
| `kmem_cache_alloc(cache, gfp)` | Allocate from slab cache | Any |
| `vmalloc(size)` | Allocate virtually contiguous | Process |
| `kvmalloc(size, gfp)` | Try kmalloc, fallback to vmalloc | Process |

### Key Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `PAGE_SIZE` | 4096 (typical) | System page size |
| `PAGE_SHIFT` | 12 (typical) | log2(PAGE_SIZE) |
| `MAX_PAGE_ORDER` | 10 | Maximum buddy order (4MB blocks) |
| `SWAP_CLUSTER_MAX` | 32 | Swap I/O batch size |
| `TLB_NR_DYN_ASIDS` | 6 | Per-CPU ASID cache size |
| `DEF_PRIORITY` | 12 | Default scan priority |

---

## 15. Memory Compaction

**Source:** `mm/compaction.c`

### Overview

Memory compaction reduces external fragmentation by scanning a memory zone in two directions: a migration scanner moving upward from the zone start identifies movable pages, and a free-page scanner moving downward from the zone end collects free pages to serve as migration targets. When the two scanners meet, the zone has been fully compacted. The process relies on `migrate_pages()` from `mm/migrate.c` to physically relocate pages.

### Compaction Triggers

Three distinct trigger paths exist:

**1. Direct Compaction (`try_to_compact_pages`)**

Called from `__alloc_pages_slowpath()` in `mm/page_alloc.c` when a high-order allocation fails after reclaim.

```c
enum compact_result try_to_compact_pages(gfp_t gfp_mask, unsigned int order,
        unsigned int alloc_flags, const struct alloc_context *ac,
        enum compact_priority prio, struct page **capture)
```

It iterates each zone in the zonelist and calls `compact_zone_order()` → `compact_zone()`.

**2. kcompactd (Background Compaction)**

One `kcompactd` kernel thread per NUMA node, created via `kcompactd_run()`. It sleeps on `pgdat->kcompactd_wait` and is woken by `wakeup_kcompactd()` when the allocator fails to satisfy a high-order request. The thread function `kcompactd()` calls `kcompactd_do_work()` → `compact_node()` → per-zone `compact_zone()`.

```c
static int kcompactd(void *p)           /* mm/compaction.c */
void wakeup_kcompactd(pg_data_t *pgdat, int order, int highest_zoneidx)
```

**3. Proactive Compaction**

kcompactd also performs proactive (background) compaction based on a per-node **fragmentation score**. Every `HPAGE_FRAG_CHECK_INTERVAL_MSEC` (500 ms), the kcompactd loop evaluates `should_proactive_compact_node()`. The fragmentation score is the external fragmentation percentage at `COMPACTION_HPAGE_ORDER` (PMD order for THP systems), weighted by zone size. If the score exceeds the high watermark (`100 - sysctl_compaction_proactiveness + 10`), proactive compaction runs until the score drops below the low watermark (`100 - sysctl_compaction_proactiveness`).

Manual triggers (order == -1, flagged by `is_via_compact_memory()`):
- Write to `/proc/sys/vm/compact_memory`
- Write to `/sys/devices/system/node/nodeX/compact`

### struct compact_control

```c
struct compact_control {
    struct list_head freepages[NR_PAGE_ORDERS]; /* Isolated free pages to migrate to */
    struct list_head migratepages;      /* Pages queued for migration */
    unsigned int nr_freepages;          /* Count of isolated free pages */
    unsigned int nr_migratepages;       /* Count of pages to migrate */
    unsigned long free_pfn;             /* Free scanner current position (scans downward) */
    unsigned long migrate_pfn;          /* Migration scanner current position (scans upward) */
    unsigned long fast_start_pfn;       /* Starting PFN for fast linear scan */
    struct zone *zone;                  /* Zone being compacted */
    unsigned long total_migrate_scanned;
    unsigned long total_free_scanned;
    unsigned short fast_search_fail;    /* Count of failures using free-list fast search */
    short search_order;                 /* Order at which to start fast search */
    const gfp_t gfp_mask;              /* GFP flags of the direct compactor */
    int order;                          /* Target allocation order */
    int migratetype;                    /* Migratetype of direct compactor */
    const unsigned int alloc_flags;     /* Allocation flags of direct compactor */
    const int highest_zoneidx;          /* Zone index of direct compactor */
    enum migrate_mode mode;             /* Async or sync migration mode */
    bool ignore_skip_hint;              /* Scan blocks even if marked PG_migrate_skip */
    bool no_set_skip_hint;              /* Don't mark blocks for skipping */
    bool ignore_block_suitable;         /* Scan blocks considered unsuitable */
    bool direct_compaction;             /* false from kcompactd or /proc/... */
    bool proactive_compaction;          /* kcompactd proactive compaction flag */
    bool whole_zone;                    /* Whole zone should/has been scanned */
    bool contended;                     /* Signal lock contention */
    bool finish_pageblock;              /* Scan remainder of pageblock on transient failure */
    bool alloc_contig;                  /* alloc_contig_range() allocation */
};
```

### Key Functions

**`compact_zone()`** — Core compaction loop for a single zone. Initializes scanner PFNs from cached positions in `zone->compact_cached_migrate_pfn[]` and `zone->compact_cached_free_pfn`. Loops calling `isolate_migratepages()` and `migrate_pages()` until `compact_finished()` returns non-`COMPACT_CONTINUE`.

**`compaction_suitable()` / `__compaction_suitable()`** — Checks whether compaction is worth running. Verifies that order-0 watermarks are met and checks the fragmentation index (extfrag) to distinguish low-memory failures from fragmentation failures.

**`isolate_migratepages_block()`** — Scans a single pageblock for movable pages. In `MIGRATE_ASYNC` mode, aborts immediately if too many pages are already isolated.

**`migrate_pages()`** (`mm/migrate.c`):

```c
int migrate_pages(struct list_head *from, new_folio_t get_new_folio,
                  free_folio_t put_new_folio, unsigned long private,
                  enum migrate_mode mode, int reason, unsigned int *ret_succeeded)
```

During compaction, `compaction_alloc()` serves as the `get_new_folio` callback, drawing pages from `cc->freepages[]`.

### Migration Modes

```c
enum migrate_mode {
    MIGRATE_ASYNC,       /* Never block; used for initial direct compaction attempts */
    MIGRATE_SYNC_LIGHT,  /* Block on most ops but not ->writepage; kcompactd and retries */
    MIGRATE_SYNC,        /* Block unconditionally; used for CMA and alloc_contig_range */
};
```

Direct compaction at `COMPACT_PRIO_ASYNC` uses `MIGRATE_ASYNC`; all other priorities use `MIGRATE_SYNC_LIGHT`.

### Interaction with the Buddy Allocator

During compaction, free pages are removed from the buddy allocator's free lists via `isolate_freepages_block()` and placed in `cc->freepages[]`. After migration, unused free pages are returned to the buddy via `release_free_list()` → `__free_pages()`. A **direct capture** optimization (`struct capture_control`) allows a page freed via the buddy's free path to be captured directly into the compaction free list, avoiding round-trips (`current->capture_control`).

### Compaction Deferral

To avoid repeatedly running failed compaction, the kernel tracks per-zone state:

- `zone->compact_considered` — attempts since last failure
- `zone->compact_defer_shift` — current backoff exponent (max `COMPACT_MAX_DEFER_SHIFT` = 6)
- `zone->compact_order_failed` — minimum order at which compaction last failed

`compaction_deferred()` returns true if `compact_considered < (1 << compact_defer_shift)`, skipping compaction. On success, `compaction_defer_reset()` resets these fields.

### `/proc/sys/vm/compact_*` Tunables

| Sysctl | Mode | Default | Description |
|--------|------|---------|-------------|
| `compact_memory` | write-only | — | Write any value to trigger full memory compaction immediately |
| `compaction_proactiveness` | 0644 | 20 | Controls aggressiveness of background proactive compaction (0–100); 0 disables |
| `extfrag_threshold` | 0644 | 500 | Fragmentation score threshold (0–1000) below which compaction is skipped |
| `compact_unevictable_allowed` | 0644 | CONFIG-dependent | Allow compaction to scan unevictable LRU lists |

---

## 16. OOM Killer

**Source:** `mm/oom_kill.c`

### Trigger Path

```
__alloc_pages_noprof()                      [mm/page_alloc.c]
  └─ __alloc_pages_slowpath()
       └─ __alloc_pages_may_oom()
            ├─ mutex_trylock(&oom_lock)
            └─ out_of_memory()              [mm/oom_kill.c]
                 ├─ blocking_notifier_call_chain(&oom_notify_list, ...)
                 ├─ select_bad_process()
                 │    └─ oom_evaluate_task() → oom_badness()
                 └─ oom_kill_process()
                      └─ mark_oom_victim()
                      └─ queue_oom_reaper()
```

### struct oom_control

```c
struct oom_control {
    struct zonelist *zonelist;           /* Used to determine cpuset */
    nodemask_t *nodemask;               /* Used to determine mempolicy */
    struct mem_cgroup *memcg;           /* memcg for cgroup-OOM, or NULL for global */
    const gfp_t gfp_mask;              /* GFP flags of triggering allocation */
    const int order;                    /* -1 means sysrq-triggered OOM */
    unsigned long totalpages;           /* total pages available in domain */
    struct task_struct *chosen;         /* task selected for killing */
    long chosen_points;                 /* badness score of chosen task */
    enum oom_constraint constraint;
};
```

### oom_badness() Scoring Algorithm

```c
long oom_badness(struct task_struct *p, unsigned long totalpages)
```

1. **Base score** = RSS pages + swap pages + page table bytes / PAGE_SIZE
   ```c
   points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +
            mm_pgtables_bytes(p->mm) / PAGE_SIZE;
   ```

2. **Adjustment** = `oom_score_adj * totalpages / 1000`
   ```c
   adj = (long)p->signal->oom_score_adj;
   adj *= totalpages / 1000;
   points += adj;
   ```

`oom_score_adj` ranges from -1000 (`OOM_SCORE_ADJ_MIN`, process is unkillable) to +1000. Setting `oom_score_adj = 1000` adds `totalpages` to the score (equivalent to claiming to use all memory). Tasks with `oom_score_adj == OOM_SCORE_ADJ_MIN`, with `MMF_OOM_SKIP` set, or in the middle of `vfork()` return `LONG_MIN` (excluded). `PF_KTHREAD` and init return `LONG_MIN` via `oom_unkillable_task()`.

### OOM Reaper

The OOM reaper is a dedicated kernel thread (`oom_reaper_th`) created at boot by `oom_init()`. Its purpose is to asynchronously reclaim the address space of the OOM victim without holding mm locks that might be held by the dying process.

**Thread function:** `oom_reaper()` — waits on `oom_reaper_wait`, dequeues tasks from `oom_reaper_list` (protected by `oom_reaper_lock` spinlock), and calls `oom_reap_task()`.

**`oom_reap_task()`:** Retries up to `MAX_OOM_REAP_RETRIES` (10) times calling `oom_reap_task_mm()`. Each attempt tries `mmap_read_trylock(mm)` (non-blocking). On success, calls `__oom_reap_task_mm()` which sets `MMF_UNSTABLE` on the mm and calls `unmap_page_range()` for each anonymous or private VMA. Shared file-backed VMAs and `VM_HUGETLB`/`VM_PFNMAP` VMAs are skipped.

**Queuing** (`queue_oom_reaper()`): After `mark_oom_victim()`, a timer is set for `OOM_REAPER_DELAY` (2 * HZ = 2 seconds). This gives the victim time to exit naturally before the reaper intervenes.

### Per-cgroup OOM (memcg OOM)

When `oom_control.memcg != NULL`, the OOM killer operates within a single memory cgroup. `select_bad_process()` uses `mem_cgroup_scan_tasks()` instead of `for_each_process()` to scan only tasks belonging to the memcg.

The memcg OOM path enters from `try_charge_memcg()` → `mem_cgroup_oom()` → `mem_cgroup_out_of_memory()` → `out_of_memory()` with `oc.memcg` set.

**Group kill (`memory.oom.group`):** `mem_cgroup_get_oom_group()` traverses ancestors of the victim's memcg looking for any with `oom_group = true`. If found, all tasks in that cgroup are killed via `mem_cgroup_kill_processes()`.

### /proc Interfaces

- `/proc/pid/oom_score` — read-only; computed by `oom_badness(task, totalpages)` scaled to 0–2000
- `/proc/pid/oom_score_adj` — read/write (range -1000 to 1000); stored in `task->signal->oom_score_adj`; writes serialized by `oom_adj_mutex`

---

## 17. vmalloc

**Source:** `mm/vmalloc.c`

### Overview

vmalloc provides virtually contiguous kernel memory backed by physically discontiguous pages. All vmalloc allocations reside in the `[VMALLOC_START, VMALLOC_END)` virtual address range. Virtual address regions are tracked in a red-black tree of `struct vmap_area`, with each allocation additionally described by a `struct vm_struct`.

### struct vm_struct

```c
struct vm_struct {
    struct vm_struct    *next;      /* linked list of vmalloc areas */
    void                *addr;      /* virtual address of the allocation */
    unsigned long        size;      /* size including guard page (if any) */
    unsigned long        flags;     /* VM_ALLOC, VM_MAP, VM_IOREMAP, etc. */
    struct page        **pages;     /* array of backing physical pages */
    unsigned int         nr_pages;  /* number of pages backing this area */
    phys_addr_t          phys_addr; /* physical address (ioremap only) */
    const void          *caller;    /* __builtin_return_address(0) for debugging */
};
```

Key `flags` values:

| Flag | Value | Meaning |
|------|-------|---------|
| `VM_IOREMAP` | 0x00000001 | Created by `ioremap()` and friends |
| `VM_ALLOC` | 0x00000002 | Created by `vmalloc()` |
| `VM_MAP` | 0x00000004 | Created by `vmap()` |
| `VM_USERMAP` | 0x00000008 | Suitable for `remap_vmalloc_range()` |
| `VM_NO_GUARD` | 0x00000040 | No guard page appended |
| `VM_ALLOW_HUGE_VMAP` | 0x00000400 | Allow huge page backing |

### struct vmap_area

```c
struct vmap_area {
    unsigned long va_start;         /* start of virtual address range */
    unsigned long va_end;           /* end of virtual address range */
    struct rb_node rb_node;         /* node in address-sorted RB tree */
    struct list_head list;          /* node in address-sorted linked list */
    union {
        unsigned long subtree_max_size; /* in "free" tree: max free gap in subtree */
        struct vm_struct *vm;           /* in "busy" tree: back-pointer to vm_struct */
    };
    unsigned long flags;
};
```

Two separate RB trees exist: `free_vmap_area_root` (free regions, augmented with `subtree_max_size` for O(log n) gap search) and `vmap_area_root` (allocated regions).

### __vmalloc_node_range() Internals

```c
void *__vmalloc_node_range_noprof(unsigned long size, unsigned long align,
        unsigned long start, unsigned long end, gfp_t gfp_mask,
        pgprot_t prot, unsigned long vm_flags, int node,
        const void *caller)
```

1. **Huge page probe** (if `VM_ALLOW_HUGE_VMAP`): Attempts PMD-size alignment for huge page PTEs
2. **VA area reservation** via `__get_vm_area_node()` → `alloc_vmap_area()`: Searches `free_vmap_area_root` for a gap ≥ `size + PAGE_SIZE` (guard page)
3. **Physical page allocation** via `__vmalloc_area_node()`: Allocates `nr_pages` pages from the buddy allocator
4. **Page table population** via `vmap_pages_range()` → `vmap_pte_range()`: Creates kernel page table mappings
5. **KASAN tagging** and TLB management

### Lazy TLB Flushing

vmalloc uses **lazy TLB flushing** to amortize the cost of TLB shootdowns. When a vmalloc region is freed, its mappings are not immediately invalidated — they are added to a purge list. `vm_unmap_aliases()` forces a global flush. This is required by callers that need stable direct-map access to pages that may have been vmap'd.

### vmalloc vs vmap vs ioremap

| Function | Flag | Pages | Purpose |
|----------|------|-------|---------|
| `vmalloc()` | `VM_ALLOC` | Allocated by kernel | General kernel memory; physically discontiguous |
| `vmap()` | `VM_MAP` | Provided by caller | Map caller-supplied pages into contiguous VA |
| `ioremap()` | `VM_IOREMAP` | None (phys addr) | Map device MMIO regions |
| `vm_map_ram()` | — | Provided by caller | Short-lived maps; vmap block allocator |

---

## 18. Memory Control Groups (memcg)

**Source:** `mm/memcontrol.c`

### struct mem_cgroup

```c
struct mem_cgroup {
    struct cgroup_subsys_state css;         /* cgroup subsystem state; must be first */
    struct mem_cgroup_id id;                /* Private memcg ID */

    /* Accounted resources — struct page_counter */
    struct page_counter memory;             /* v1 & v2: tracks RSS + page cache */

    union {
        struct page_counter swap;           /* v2 only: tracks swap usage */
        struct page_counter memsw;          /* v1 only: tracks memory + swap combined */
    };

    struct work_struct high_work;           /* async memory.high enforcement */
    struct vmpressure vmpressure;           /* vmpressure notification state */
    bool oom_group;                         /* kill all tasks in group on OOM */
    int swappiness;                         /* per-memcg swappiness override */
    struct memcg_vmstats *vmstats;          /* memory.stat counters */
    atomic_long_t memory_events[MEMCG_NR_MEMORY_EVENTS];
    int kmemcg_id;                          /* kernel memory accounting ID */
    struct obj_cgroup __rcu *objcg;         /* object cgroup for slab/percpu charging */
    struct memcg_vmstats_percpu __percpu *vmstats_percpu;
    /* per-node LRU state embedded via mem_cgroup_per_node */
};
```

### struct page_counter

```c
struct page_counter {
    atomic_long_t usage;        /* current usage in pages */
    unsigned long failcnt;      /* v1: number of times limit was hit */
    unsigned long emin;         /* effective memory.min */
    unsigned long elow;         /* effective memory.low */
    unsigned long watermark;    /* peak usage high-watermark */
    unsigned long min;          /* memory.min: hard protection */
    unsigned long low;          /* memory.low: soft protection */
    unsigned long high;         /* memory.high: throttle threshold */
    unsigned long max;          /* memory.max: hard limit */
    struct page_counter *parent;
};
```

### memory.min / memory.low / memory.high / memory.max Semantics

| Interface | Semantics |
|-----------|-----------|
| `memory.min` | Hard protection: pages will not be reclaimed when `usage <= min`. Protects latency-sensitive workloads. |
| `memory.low` | Soft protection: reclaim scanner deprioritizes this cgroup's pages when `usage <= low`. May still be reclaimed under severe pressure. |
| `memory.high` | Throttle threshold: when usage exceeds `high`, `try_charge_memcg()` invokes reclaim and may sleep the allocating task. |
| `memory.max` | Hard limit: charge fails (→ OOM) when `usage >= max`. Enforced by `page_counter_try_charge()`. |

### Charge Path

```c
int __mem_cgroup_charge(struct folio *folio, struct mm_struct *mm, gfp_t gfp)
```

**`try_charge_memcg()`** — the core charging logic:

1. **Stock fast path**: `consume_stock()` deducts from per-CPU cached charge (`MEMCG_CHARGE_BATCH` pages). No atomic ops on `page_counter`.
2. **Counter check**: `page_counter_try_charge(&memcg->memory, batch, &counter)`. Failure identifies which ancestor exceeded its `max`.
3. **Reclaim loop**: `try_to_free_mem_cgroup_pages()` → `do_try_to_free_pages()`. Retries up to `MAX_RECLAIM_RETRIES`.
4. **Drain stock**: `drain_all_stock()` forces all CPUs to flush cached charges.
5. **OOM**: If reclaim cannot make progress, calls `mem_cgroup_oom()` → `out_of_memory()` with `oc.memcg` set.

### Uncharge Path

```c
void __mem_cgroup_uncharge(struct folio *folio)
```

Calls `uncharge_folio()` which extracts the memcg from `folio->memcg_data`, updates the uncharge gather struct, clears `folio->memcg_data`, and releases the css reference. `uncharge_batch()` calls `page_counter_uncharge()` in one shot.

### Per-memcg LRU Lists

Each memcg embeds an `lruvec` via `struct mem_cgroup_per_node` (one per NUMA node):

```c
struct lruvec {
    struct list_head    lists[NR_LRU_LISTS];  /* INACTIVE_ANON, ACTIVE_ANON,
                                               * INACTIVE_FILE, ACTIVE_FILE,
                                               * UNEVICTABLE */
    spinlock_t          lru_lock;             /* per-lruvec lock (since v5.11) */
    unsigned long       anon_cost;            /* reclaim cost balance tracking */
    unsigned long       file_cost;
    atomic_long_t       nonresident_age;      /* drives refault detection */
    unsigned long       refaults[ANON_AND_FILE];
    unsigned long       flags;
#ifdef CONFIG_LRU_GEN
    struct lru_gen_folio lrugen;              /* MGLRU generation state */
#endif
};
```

---

## 19. Locking Summary

| Lock Name | Type | Protects | Key File |
|-----------|------|----------|----------|
| `zone->lock` | `spinlock_t` | `zone->free_area[]` free lists; buddy allocator page state | `include/linux/mmzone.h` |
| `lruvec->lru_lock` | `spinlock_t` | Per-lruvec (per-memcg-per-node) LRU lists; page isolation | `include/linux/mmzone.h` |
| `mm->mmap_lock` | `struct rw_semaphore` | VMA tree (`mm->mm_mt`); VMA creation/deletion/modification | `include/linux/mm_types.h` |
| Page lock (`PG_locked`) | bit-spinlock in `page->flags` | Individual page/folio state during I/O, writeback, truncation | `include/linux/page-flags.h` |
| `pte_lock` | `spinlock_t` (per page table page) | Hardware page table entries (PTEs, PMDs) | `include/linux/mm_types.h` |
| `anon_vma->rwsem` | `struct rw_semaphore` | anon_vma chain list; reverse mapping of anonymous pages | `include/linux/rmap.h` |
| `mapping->i_mmap_rwsem` | `struct rw_semaphore` | `address_space->i_mmap` interval tree of file-backed VMAs | `include/linux/fs.h` |
| `hugetlb_lock` | `spinlock_t` | Hugepage free pool lists; hugepage allocation/freeing | `include/linux/hugetlb.h` |
| `swap_lock` | `spinlock_t` (static) | `swap_active_head`; `nr_swapfiles`; `total_swap_pages` | `mm/swapfile.c` |
| `pgd_lock` / `init_mm.page_table_lock` | `spinlock_t` | Kernel page global directory modifications | `mm/init-mm.c` |
| `shmem_swaplist_mutex` | `struct mutex` | `shmem_swaplist` during shmem swap reclaim | `mm/shmem.c` |
| `oom_lock` | `struct mutex` | Serializes concurrent `out_of_memory()` invocations | `mm/oom_kill.c` |
| `oom_adj_mutex` | `struct mutex` | Serializes writes to `oom_score_adj` | `mm/oom_kill.c` |
| `vmap_area_lock` | `spinlock_t` (static) | `vmap_area_root` and `free_vmap_area_root` RB trees | `mm/vmalloc.c` |
| `objcg_lock` | `spinlock_t` | `obj_cgroup` list linkage during kmem accounting | `mm/memcontrol.c` |
| `oom_reaper_lock` | `spinlock_t` (static) | `oom_reaper_list` of tasks pending async reaping | `mm/oom_kill.c` |

---

## 20. Source Files Reference

### Core mm/ Implementation Files

| File | Purpose | Key Functions |
|------|---------|---------------|
| `mm/page_alloc.c` | Buddy allocator; zone/node management; watermark enforcement; PCP lists | `__alloc_pages_noprof()`, `get_page_from_freelist()`, `__alloc_pages_slowpath()`, `free_pages()` |
| `mm/vmscan.c` | Page reclaim (kswapd and direct); LRU scanning; refault detection | `kswapd()`, `do_try_to_free_pages()`, `shrink_lruvec()`, `shrink_inactive_list()` |
| `mm/memory.c` | Page fault handling; COW; demand paging; page table manipulation | `handle_mm_fault()`, `do_anonymous_page()`, `do_cow_fault()`, `do_wp_page()` |
| `mm/mmap.c` | VMA management; mmap/munmap/mremap; VMA merging/splitting | `do_mmap()`, `do_munmap()`, `vma_merge()`, `find_vma()`, `mmap_region()` |
| `mm/compaction.c` | Memory compaction; kcompactd thread; migration/free scanners | `compact_zone()`, `try_to_compact_pages()`, `isolate_migratepages_block()`, `kcompactd()` |
| `mm/oom_kill.c` | OOM killer; victim selection; OOM reaper | `out_of_memory()`, `oom_badness()`, `select_bad_process()`, `oom_kill_process()`, `oom_reaper()` |
| `mm/vmalloc.c` | vmalloc/vmap/ioremap VA space management; vmap_area RB tree | `__vmalloc_node_range_noprof()`, `vfree()`, `vmap()`, `alloc_vmap_area()`, `vm_unmap_aliases()` |
| `mm/memcontrol.c` | cgroup v2 memory controller; charge/uncharge; memcg OOM | `__mem_cgroup_charge()`, `__mem_cgroup_uncharge()`, `try_charge_memcg()`, `mem_cgroup_oom()` |
| `mm/memcontrol-v1.c` | cgroup v1 memory controller; soft limit reclaim | `memcg1_soft_limit_reclaim()` |
| `mm/slub.c` | SLUB allocator (default); per-CPU slab; lockless fast path | `kmem_cache_alloc_noprof()`, `kmem_cache_free()`, `__slab_alloc()`, `new_slab()` |
| `mm/filemap.c` | Page cache management; folio locking; readahead | `filemap_fault()`, `filemap_get_folio()`, `filemap_add_folio()`, `folio_lock()` |
| `mm/swap.c` | LRU list management; pagevec/folio_batch processing | `folio_mark_accessed()`, `lru_add_drain()`, `folio_activate()`, `folio_add_lru()` |
| `mm/swap_state.c` | Swap cache; swap readahead; swap-in/swap-out | `add_to_swap()`, `delete_from_swap_cache()`, `lookup_swap_cache()` |
| `mm/huge_memory.c` | Transparent Huge Pages (THP); THP fault handling; splitting | `do_huge_pmd_anonymous_page()`, `__split_huge_page()`, `collapse_huge_page()` |
| `mm/hugetlb.c` | HugeTLB (explicit huge pages); reservation management | `alloc_huge_page()`, `free_huge_page()`, `hugetlb_reserve_pages()`, `hugetlb_fault()` |
| `mm/migrate.c` | Page/folio migration engine; NUMA migration; CMA | `migrate_pages()`, `migrate_folio()`, `move_to_new_folio()`, `putback_movable_pages()` |
| `mm/rmap.c` | Reverse mapping; try_to_unmap; anon_vma management | `try_to_unmap()`, `try_to_unmap_one()`, `folio_referenced()`, `anon_vma_fork()` |
| `mm/madvise.c` | madvise() syscall | `do_madvise()`, `madvise_dontneed_single_vma()`, `madvise_free_single_vma()` |
| `mm/mprotect.c` | mprotect() syscall; permission changes | `sys_mprotect()`, `mprotect_fixup()`, `change_pte_range()` |
| `mm/mlock.c` | mlock/munlock; unevictable LRU management | `do_mlock()`, `mlock_vma_pages_range()`, `mlock_folio()` |
| `mm/khugepaged.c` | khugepaged thread; THP collapse scanning | `khugepaged()`, `khugepaged_scan_mm_slot()`, `collapse_huge_page()` |
| `mm/page-writeback.c` | Dirty page accounting; writeback throttling | `balance_dirty_pages()`, `set_page_dirty()`, `folio_account_dirtied()` |
| `mm/truncate.c` | Page cache truncation; hole punching; invalidation | `truncate_inode_pages_range()`, `invalidate_inode_pages2()` |
| `mm/swapfile.c` | Swap device management; swapon/swapoff; swap slot allocation | `sys_swapon()`, `sys_swapoff()`, `folio_alloc_swap()`, `swap_duplicate()` |
| `mm/ksm.c` | Kernel Samepage Merging; anonymous page deduplication | `ksmd()`, `ksm_scan_thread()`, `cmp_and_merge_page()` |

### Key Header Files

| File | Purpose | Key Definitions |
|------|---------|-----------------|
| `include/linux/mm.h` | Main MM header; page/folio helpers; VMA interfaces | `folio_get()`, `folio_put()`, `get_user_pages()`, `vm_fault_t` |
| `include/linux/mm_types.h` | Core MM struct definitions | `struct mm_struct`, `struct vm_area_struct`, `struct page`, `struct folio` |
| `include/linux/mmzone.h` | Zone and node structures; watermarks; LRU lists | `struct zone`, `struct pglist_data`, `struct lruvec`, `enum zone_type` |
| `include/linux/gfp.h` | GFP flag definitions and zone modifiers | `GFP_KERNEL`, `GFP_ATOMIC`, `__GFP_*` flags |
| `include/linux/swap.h` | Swap interfaces; swap_info_struct; swap entry types | `struct swap_info_struct`, `swp_entry_t`, `folio_alloc_swap()` |
| `include/linux/memcontrol.h` | memcg public API; folio charging macros | `mem_cgroup_charge()`, `mem_cgroup_uncharge()`, `folio_memcg()` |
| `include/linux/vmalloc.h` | vmalloc public API; struct vm_struct; VM_* flags | `vmalloc()`, `vfree()`, `vmap()`, `struct vm_struct`, `struct vmap_area` |
| `include/linux/migrate_mode.h` | Migration mode and reason enums | `enum migrate_mode`, `enum migrate_reason` |
| `include/linux/compaction.h` | Compaction result codes and public API | `enum compact_result`, `enum compact_priority`, `try_to_compact_pages()` |
| `include/linux/rmap.h` | Reverse mapping structures and interfaces | `struct anon_vma`, `try_to_unmap()`, `folio_referenced()` |

---

*This document covers Linux kernel memory management as of kernel version 6.x.
Implementation details may vary between versions.*
