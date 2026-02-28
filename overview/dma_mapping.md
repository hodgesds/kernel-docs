# Linux Kernel DMA Mapping Subsystem Overview

## Table of Contents
1. [Introduction](#introduction)
2. [The DMA Addressing Problem](#the-dma-addressing-problem)
3. [DMA Directions](#dma-directions)
4. [DMA Masks](#dma-masks)
5. [Streaming DMA Mappings](#streaming-dma-mappings)
6. [Coherent DMA Allocations](#coherent-dma-allocations)
7. [Scatter-Gather Lists](#scatter-gather-lists)
8. [The dma_map_ops Abstraction](#the-dma_map_ops-abstraction)
9. [Direct DMA (No IOMMU)](#direct-dma-no-iommu)
10. [IOMMU-Backed DMA](#iommu-backed-dma)
11. [Bounce Buffers (SWIOTLB)](#bounce-buffers-swiotlb)
12. [DMA Pools](#dma-pools)
13. [Coherent Memory Pools](#coherent-memory-pools)
14. [Atomic DMA Pools](#atomic-dma-pools)
15. [Cache Coherency](#cache-coherency)
16. [DMA Fences](#dma-fences)
17. [DMA Debugging](#dma-debugging)
18. [Source Files](#source-files)

---

## Introduction

The DMA (Direct Memory Access) mapping subsystem provides a unified API for
device drivers to set up memory regions that hardware devices can access
directly, without CPU involvement in data transfers. The subsystem handles the
fundamental mismatch between CPU virtual/physical addresses and the bus
addresses that devices use, while also managing cache coherency, address
translation, and bounce buffering for devices with limited addressing
capabilities.

### Why DMA Mapping Exists

```
+------------------------------------------------------------------+
|                   The DMA Problem                                 |
+------------------------------------------------------------------+
|                                                                   |
|  CPU                          Device                              |
|  +-----------+                +----------+                        |
|  | Virtual   |                | DMA      |                        |
|  | Address   |---MMU--->+     | Engine   |                        |
|  | 0xffff... |          |     |          |                        |
|  +-----------+          |     +----+-----+                        |
|                         |          |                              |
|                    Physical     Bus Address                       |
|                    Address     (dma_addr_t)                       |
|                         |          |                              |
|                         v          v                              |
|                    +----+----------+----+                         |
|                    |                    |                         |
|                    |   Physical RAM     |                         |
|                    |                    |                         |
|                    +--------------------+                         |
|                                                                   |
|  Problems:                                                        |
|  1. Device bus address != CPU physical address (offset, IOMMU)    |
|  2. Device may only address lower 32 bits (or less)               |
|  3. CPU caches may hold stale data vs. what device wrote          |
|  4. Memory may not be physically contiguous                       |
|                                                                   |
+------------------------------------------------------------------+
```

The DMA mapping API solves these problems by:
- Translating between CPU and device address spaces
- Providing bounce buffers when devices cannot address all memory
- Managing cache coherency between CPU and device
- Supporting scatter-gather for non-contiguous memory regions

---

## The DMA Addressing Problem

On many systems, the address a device uses to access memory (the bus address
or DMA address, represented as `dma_addr_t`) differs from the CPU physical
address. This can happen for several reasons:

1. **Fixed offset**: The bus address space has a constant offset from CPU
   physical addresses (common on ARM SoCs with `dma-ranges` in device tree).

2. **IOMMU translation**: An I/O Memory Management Unit remaps device addresses
   through page tables, similar to the CPU MMU.

3. **Limited address width**: Older PCI devices may only generate 32-bit or
   even 24-bit addresses, but system RAM may reside above those limits.

4. **Memory encryption**: With AMD SEV or Intel TDX, DMA memory may need
   to be marked as decrypted (unencrypted) for device access.

The DMA mapping subsystem provides the `dma_addr_t` type to represent
device-visible addresses, which drivers must use instead of raw physical
addresses.

```c
/* include/linux/dma-mapping.h:71 */
#define DMA_MAPPING_ERROR               (~(dma_addr_t)0)
```

---

## DMA Directions

Every DMA mapping operation requires a direction, which tells the subsystem
which cache operations are needed.

```c
/* include/linux/dma-direction.h:5-10 */
enum dma_data_direction {
    DMA_BIDIRECTIONAL = 0,
    DMA_TO_DEVICE = 1,
    DMA_FROM_DEVICE = 2,
    DMA_NONE = 3,
};
```

```
+------------------------------------------------------------------+
|                  DMA Direction Semantics                          |
+------------------------------------------------------------------+
|                                                                   |
|  DMA_TO_DEVICE (CPU -> Device)                                    |
|  ============================                                     |
|  - CPU writes data to buffer, then maps it                        |
|  - Kernel flushes CPU caches so device sees current data          |
|  - Device reads from the buffer                                   |
|  - Example: transmitting a network packet                         |
|                                                                   |
|  DMA_FROM_DEVICE (Device -> CPU)                                  |
|  ===============================                                  |
|  - Device writes data into the buffer                             |
|  - CPU invalidates caches before reading                          |
|  - CPU reads the data written by device                           |
|  - Example: receiving a network packet                            |
|                                                                   |
|  DMA_BIDIRECTIONAL (both directions)                              |
|  ====================================                             |
|  - Both CPU and device may read and write                         |
|  - Most expensive: requires both flush and invalidate             |
|  - Example: command buffer shared with device                     |
|                                                                   |
|  DMA_NONE                                                         |
|  =========                                                        |
|  - No DMA transfer, used as a sentinel / for debug                |
|                                                                   |
+------------------------------------------------------------------+
```

---

## DMA Masks

DMA masks inform the kernel about the addressing capabilities of a device.
There are two separate masks:

1. **Streaming DMA mask** (`dev->dma_mask`): Used for `dma_map_*()` operations
2. **Coherent DMA mask** (`dev->coherent_dma_mask`): Used for `dma_alloc_coherent()`

### Device DMA Parameters

```c
/* include/linux/device.h:442-450 */
struct device_dma_parameters {
    /*
     * a low level driver may set these to teach IOMMU code about
     * sg limitations.
     */
    unsigned int max_segment_size;
    unsigned int min_align_mask;
    unsigned long segment_boundary_mask;
};
```

### Key DMA Fields in struct device

```c
/* include/linux/device.h (selected DMA fields) */
struct device {
    /* ... */
    u64                     *dma_mask;          /* Streaming DMA mask */
    u64                     coherent_dma_mask;  /* Coherent DMA mask */
    u64                     bus_dma_limit;      /* Bus-imposed limit */

    const struct dma_map_ops *dma_ops;          /* DMA operations */
    struct dma_coherent_mem *dma_mem;           /* Per-device coherent pool */

    struct device_dma_parameters *dma_parms;   /* SG constraints */
    struct list_head        dma_pools;          /* DMA pools list */

    const struct bus_dma_region *dma_range_map; /* CPU<->DMA offset map */
    struct io_tlb_mem       *dma_io_tlb_mem;   /* SWIOTLB instance */

    bool                    dma_coherent;       /* HW coherent? */
    bool                    dma_skip_sync;      /* Skip sync ops? */
    bool                    dma_ops_bypass;     /* Bypass ops? */
    /* ... */
};
```

### Mask Functions

```c
/* include/linux/dma-mapping.h:73 */
#define DMA_BIT_MASK(n)  (((n) == 64) ? ~0ULL : ((1ULL<<(n))-1))

/* kernel/dma/mapping.c:878 */
int dma_set_mask(struct device *dev, u64 mask);

/* kernel/dma/mapping.c:897 */
int dma_set_coherent_mask(struct device *dev, u64 mask);

/* include/linux/dma-mapping.h:498-504 */
static inline int dma_set_mask_and_coherent(struct device *dev, u64 mask)
{
    int rc = dma_set_mask(dev, mask);
    if (rc == 0)
        dma_set_coherent_mask(dev, mask);
    return rc;
}

/* include/linux/dma-mapping.h:485-490 */
static inline u64 dma_get_mask(struct device *dev)
{
    if (dev->dma_mask && *dev->dma_mask)
        return *dev->dma_mask;
    return DMA_BIT_MASK(32);
}
```

### DMA Mask Usage in Drivers

```c
/* Typical driver probe function */
static int my_driver_probe(struct pci_dev *pdev, ...)
{
    /* Try 64-bit DMA first */
    if (dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64))) {
        /* Fall back to 32-bit */
        if (dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32)))
            return -EIO;
    }
    /* ... */
}
```

### Addressing Limitations

```c
/* kernel/dma/mapping.c:921 */
bool dma_addressing_limited(struct device *dev)
{
    const struct dma_map_ops *ops = get_dma_ops(dev);

    if (min_not_zero(dma_get_mask(dev), dev->bus_dma_limit) <
                     dma_get_required_mask(dev))
        return true;

    if (unlikely(ops) || use_dma_iommu(dev))
        return false;
    return !dma_direct_all_ram_mapped(dev);
}
```

---

## Streaming DMA Mappings

Streaming mappings are used for one-time DMA transfers where the CPU sets
up a buffer, maps it for device access, the device performs DMA, and then
the buffer is unmapped. The buffer has a clear ownership model: it belongs
to either the CPU or the device at any given time.

### Mapping a Single Buffer

```c
/* include/linux/dma-mapping.h:376-386 */
static inline dma_addr_t dma_map_single_attrs(struct device *dev, void *ptr,
        size_t size, enum dma_data_direction dir, unsigned long attrs)
{
    /* DMA must never operate on areas that might be remapped. */
    if (dev_WARN_ONCE(dev, is_vmalloc_addr(ptr),
                      "rejecting DMA map of vmalloc memory\n"))
        return DMA_MAPPING_ERROR;
    debug_dma_map_single(dev, ptr, size);
    return dma_map_page_attrs(dev, virt_to_page(ptr), offset_in_page(ptr),
            size, dir, attrs);
}

/* Convenience macros (include/linux/dma-mapping.h:460-465) */
#define dma_map_single(d, a, s, r) dma_map_single_attrs(d, a, s, r, 0)
#define dma_unmap_single(d, a, s, r) dma_unmap_single_attrs(d, a, s, r, 0)
#define dma_map_page(d, p, o, s, r) dma_map_page_attrs(d, p, o, s, r, 0)
#define dma_unmap_page(d, a, s, r) dma_unmap_page_attrs(d, a, s, r, 0)
```

### dma_map_page_attrs -- The Core Mapping Function

This is the central function that all single-buffer streaming mappings
pass through. It dispatches to direct DMA, IOMMU, or custom ops.

```c
/* kernel/dma/mapping.c:155-181 */
dma_addr_t dma_map_page_attrs(struct device *dev, struct page *page,
        size_t offset, size_t size, enum dma_data_direction dir,
        unsigned long attrs)
{
    const struct dma_map_ops *ops = get_dma_ops(dev);
    dma_addr_t addr;

    BUG_ON(!valid_dma_direction(dir));

    if (WARN_ON_ONCE(!dev->dma_mask))
        return DMA_MAPPING_ERROR;

    if (dma_map_direct(dev, ops) ||
        arch_dma_map_page_direct(dev, page_to_phys(page) + offset + size))
        addr = dma_direct_map_page(dev, page, offset, size, dir, attrs);
    else if (use_dma_iommu(dev))
        addr = iommu_dma_map_page(dev, page, offset, size, dir, attrs);
    else
        addr = ops->map_page(dev, page, offset, size, dir, attrs);
    kmsan_handle_dma(page, offset, size, dir);
    trace_dma_map_page(dev, page_to_phys(page) + offset, addr, size, dir,
                       attrs);
    debug_dma_map_page(dev, page, offset, size, dir, addr, attrs);

    return addr;
}
```

### Checking for Mapping Errors

```c
/* include/linux/dma-mapping.h:91-98 */
static inline int dma_mapping_error(struct device *dev, dma_addr_t dma_addr)
{
    debug_dma_mapping_error(dev, dma_addr);

    if (unlikely(dma_addr == DMA_MAPPING_ERROR))
        return -ENOMEM;
    return 0;
}
```

### Streaming Mapping Lifecycle

```
+------------------------------------------------------------------+
|              Streaming DMA Mapping Lifecycle                      |
+------------------------------------------------------------------+
|                                                                   |
|  1. CPU prepares data in buffer                                   |
|                                                                   |
|  2. dma_map_single(dev, buf, size, DMA_TO_DEVICE)                 |
|     - Flushes CPU caches (write-back)                             |
|     - Returns dma_addr_t for device                               |
|     - Buffer now "owned" by device                                |
|     +---> CPU must NOT touch the buffer                           |
|                                                                   |
|  3. Program device with dma_addr_t                                |
|                                                                   |
|  4. Device performs DMA transfer                                  |
|                                                                   |
|  5. Device signals completion (interrupt, polling)                |
|                                                                   |
|  6. dma_unmap_single(dev, dma_addr, size, DMA_TO_DEVICE)          |
|     - Invalidates CPU caches if needed                            |
|     - Frees SWIOTLB bounce buffer if used                         |
|     - Buffer now "owned" by CPU again                             |
|     +---> CPU can safely access the buffer                        |
|                                                                   |
+------------------------------------------------------------------+
```

### Synchronization Without Unmapping

For long-lived mappings where the buffer is reused, drivers can transfer
ownership back and forth without unmapping:

```c
/* include/linux/dma-mapping.h:297-323 */
static inline void dma_sync_single_for_cpu(struct device *dev,
        dma_addr_t addr, size_t size, enum dma_data_direction dir)
{
    if (dma_dev_need_sync(dev))
        __dma_sync_single_for_cpu(dev, addr, size, dir);
}

static inline void dma_sync_single_for_device(struct device *dev,
        dma_addr_t addr, size_t size, enum dma_data_direction dir)
{
    if (dma_dev_need_sync(dev))
        __dma_sync_single_for_device(dev, addr, size, dir);
}
```

### DMA Attributes

Attributes modify the behavior of DMA mapping operations:

```c
/* include/linux/dma-mapping.h:20-59 */
#define DMA_ATTR_WEAK_ORDERING          (1UL << 1)  /* Relaxed ordering */
#define DMA_ATTR_WRITE_COMBINE          (1UL << 2)  /* Write-combining */
#define DMA_ATTR_NO_KERNEL_MAPPING      (1UL << 4)  /* No kernel vaddr */
#define DMA_ATTR_SKIP_CPU_SYNC          (1UL << 5)  /* Skip cache sync */
#define DMA_ATTR_FORCE_CONTIGUOUS       (1UL << 6)  /* Physically contiguous */
#define DMA_ATTR_ALLOC_SINGLE_PAGES     (1UL << 7)  /* No TLB optimization */
#define DMA_ATTR_NO_WARN                (1UL << 8)  /* Suppress alloc warns */
#define DMA_ATTR_PRIVILEGED             (1UL << 9)  /* Elevated privilege */
```

---

## Coherent DMA Allocations

Coherent (or consistent) DMA allocations provide memory that is simultaneously
accessible by both the CPU and the device without any explicit synchronization.
The kernel ensures hardware cache coherency, typically by mapping the memory
as uncached or write-combining on architectures without hardware coherence.

### dma_alloc_coherent

```c
/* include/linux/dma-mapping.h:471-476 */
static inline void *dma_alloc_coherent(struct device *dev, size_t size,
        dma_addr_t *dma_handle, gfp_t gfp)
{
    return dma_alloc_attrs(dev, size, dma_handle, gfp,
            (gfp & __GFP_NOWARN) ? DMA_ATTR_NO_WARN : 0);
}
```

The core implementation dispatches through the same three-way path as
streaming mappings:

```c
/* kernel/dma/mapping.c:592-634 */
void *dma_alloc_attrs(struct device *dev, size_t size, dma_addr_t *dma_handle,
        gfp_t flag, unsigned long attrs)
{
    const struct dma_map_ops *ops = get_dma_ops(dev);
    void *cpu_addr;

    WARN_ON_ONCE(!dev->coherent_dma_mask);

    if (WARN_ON_ONCE(flag & __GFP_COMP))
        return NULL;

    if (dma_alloc_from_dev_coherent(dev, size, dma_handle, &cpu_addr))
        return cpu_addr;         /* Per-device coherent pool */

    flag &= ~(__GFP_DMA | __GFP_DMA32 | __GFP_HIGHMEM);

    if (dma_alloc_direct(dev, ops)) {
        cpu_addr = dma_direct_alloc(dev, size, dma_handle, flag, attrs);
    } else if (use_dma_iommu(dev)) {
        cpu_addr = iommu_dma_alloc(dev, size, dma_handle, flag, attrs);
    } else if (ops->alloc) {
        cpu_addr = ops->alloc(dev, size, dma_handle, flag, attrs);
    } else {
        return NULL;
    }

    return cpu_addr;
}
```

### Streaming vs. Coherent Comparison

```
+------------------------------------------------------------------+
|         Streaming vs. Coherent DMA Mappings                       |
+------------------------------------------------------------------+
|                                                                   |
|  Streaming (dma_map_single, dma_map_sg)                           |
|  ======================================                           |
|  - Maps existing CPU memory for device access                     |
|  - Explicit ownership transfer (map/sync/unmap)                   |
|  - Better performance (uses cached memory)                        |
|  - Use for: data buffers, network packets, disk I/O               |
|                                                                   |
|  Coherent (dma_alloc_coherent)                                    |
|  =============================                                    |
|  - Allocates new memory accessible by both CPU and device         |
|  - No synchronization needed (always coherent)                    |
|  - Lower CPU performance (uncached/write-combining memory)        |
|  - Returns both CPU virtual address and DMA address               |
|  - Use for: descriptor rings, command queues, shared state        |
|                                                                   |
|  Non-coherent pages (dma_alloc_pages / dma_alloc_noncoherent)     |
|  =============================================================    |
|  - Allocates pages with a DMA mapping but no coherency guarantee  |
|  - Requires explicit sync like streaming mappings                 |
|  - Returns struct page * (not a kernel virtual address)           |
|  - Use for: large buffers where sync cost < uncached cost         |
|                                                                   |
+------------------------------------------------------------------+
```

### Managed Coherent DMA

Devres-managed variants automatically free on driver unbind:

```c
/* kernel/dma/mapping.c:33-38 */
struct dma_devres {
    size_t          size;
    void            *vaddr;
    dma_addr_t      dma_handle;
    unsigned long   attrs;
};

/* kernel/dma/mapping.c:93-118 */
void *dmam_alloc_attrs(struct device *dev, size_t size,
        dma_addr_t *dma_handle, gfp_t gfp, unsigned long attrs);

/* include/linux/dma-mapping.h:588-593 */
static inline void *dmam_alloc_coherent(struct device *dev, size_t size,
        dma_addr_t *dma_handle, gfp_t gfp)
{
    return dmam_alloc_attrs(dev, size, dma_handle, gfp,
            (gfp & __GFP_NOWARN) ? DMA_ATTR_NO_WARN : 0);
}
```

### Write-Combining Allocations

```c
/* include/linux/dma-mapping.h:595-604 */
static inline void *dma_alloc_wc(struct device *dev, size_t size,
                                 dma_addr_t *dma_addr, gfp_t gfp)
{
    unsigned long attrs = DMA_ATTR_WRITE_COMBINE;
    if (gfp & __GFP_NOWARN)
        attrs |= DMA_ATTR_NO_WARN;
    return dma_alloc_attrs(dev, size, dma_addr, gfp, attrs);
}
```

---

## Scatter-Gather Lists

Scatter-gather (SG) lists describe buffers that are composed of multiple
non-contiguous physical memory segments. They are used extensively for disk
I/O, network packets with fragmented data, and GPU buffer management.

### Core Data Structures

```c
/* include/linux/scatterlist.h:11-22 */
struct scatterlist {
    unsigned long   page_link;      /* Page ptr + chain/end flags */
    unsigned int    offset;         /* Offset within page */
    unsigned int    length;         /* Length of this segment */
    dma_addr_t      dma_address;    /* DMA address after mapping */
#ifdef CONFIG_NEED_SG_DMA_LENGTH
    unsigned int    dma_length;     /* DMA length (may differ) */
#endif
#ifdef CONFIG_NEED_SG_DMA_FLAGS
    unsigned int    dma_flags;      /* DMA-specific flags */
#endif
};

/* include/linux/scatterlist.h:39-43 */
struct sg_table {
    struct scatterlist *sgl;        /* The list */
    unsigned int nents;             /* Number of mapped entries */
    unsigned int orig_nents;        /* Original size of list */
};
```

The `page_link` field overloads the low bits of the page pointer:
- Bit 0 (`SG_CHAIN`): This entry is a chain link to the next SG array
- Bit 1 (`SG_END`): This is the last entry in the list

### Mapping Scatter-Gather Lists

```c
/* kernel/dma/mapping.c:250-260 */
unsigned int dma_map_sg_attrs(struct device *dev, struct scatterlist *sg,
            int nents, enum dma_data_direction dir, unsigned long attrs);

/* kernel/dma/mapping.c:289-300 */
int dma_map_sgtable(struct device *dev, struct sg_table *sgt,
                    enum dma_data_direction dir, unsigned long attrs);
```

`dma_map_sg_attrs` may coalesce adjacent physical segments into fewer DMA
segments (returning a count <= `nents`), or may split segments that cross
IOMMU page boundaries. The IOMMU can also remap non-contiguous pages to
appear contiguous from the device's perspective.

```
+------------------------------------------------------------------+
|             Scatter-Gather DMA Mapping                            |
+------------------------------------------------------------------+
|                                                                   |
|  Before mapping (CPU view):                                       |
|  +--------+  +--------+  +--------+  +--------+                   |
|  | Page A |  | Page C |  | Page Q |  | Page R |                   |
|  | 0x1000 |  | 0x3000 |  | 0x5000 |  | 0x6000 |                   |
|  +--------+  +--------+  +--------+  +--------+                   |
|   sg[0]       sg[1]       sg[2]       sg[3]                       |
|                                                                   |
|  After dma_map_sg (with IOMMU - may coalesce):                    |
|  +------------------+  +------------------+                       |
|  | IOVA 0x10000     |  | IOVA 0x12000     |                       |
|  | len = 2 pages    |  | len = 2 pages    |                       |
|  +------------------+  +------------------+                       |
|   sg_dma[0]              sg_dma[1]                                |
|   Returns nents = 2  (coalesced from 4)                           |
|                                                                   |
|  After dma_map_sg (direct, no coalescing):                        |
|  sg[0].dma_address = 0x1000  sg[2].dma_address = 0x5000           |
|  sg[1].dma_address = 0x3000  sg[3].dma_address = 0x6000           |
|   Returns nents = 4                                               |
|                                                                   |
+------------------------------------------------------------------+
```

### SG Synchronization

```c
/* include/linux/dma-mapping.h:437-458 */
static inline void dma_sync_sgtable_for_cpu(struct device *dev,
        struct sg_table *sgt, enum dma_data_direction dir)
{
    dma_sync_sg_for_cpu(dev, sgt->sgl, sgt->orig_nents, dir);
}

static inline void dma_sync_sgtable_for_device(struct device *dev,
        struct sg_table *sgt, enum dma_data_direction dir)
{
    dma_sync_sg_for_device(dev, sgt->sgl, sgt->orig_nents, dir);
}
```

---

## The dma_map_ops Abstraction

The `dma_map_ops` structure is the vtable that allows different DMA
implementation backends (direct, IOMMU, platform-specific) to be plugged
in transparently.

```c
/* include/linux/dma-map-ops.h:16-71 */
struct dma_map_ops {
    void *(*alloc)(struct device *dev, size_t size,
            dma_addr_t *dma_handle, gfp_t gfp,
            unsigned long attrs);
    void (*free)(struct device *dev, size_t size, void *vaddr,
            dma_addr_t dma_handle, unsigned long attrs);
    struct page *(*alloc_pages_op)(struct device *dev, size_t size,
            dma_addr_t *dma_handle, enum dma_data_direction dir,
            gfp_t gfp);
    void (*free_pages)(struct device *dev, size_t size, struct page *vaddr,
            dma_addr_t dma_handle, enum dma_data_direction dir);
    int (*mmap)(struct device *, struct vm_area_struct *,
            void *, dma_addr_t, size_t, unsigned long attrs);

    int (*get_sgtable)(struct device *dev, struct sg_table *sgt,
            void *cpu_addr, dma_addr_t dma_addr, size_t size,
            unsigned long attrs);

    dma_addr_t (*map_page)(struct device *dev, struct page *page,
            unsigned long offset, size_t size,
            enum dma_data_direction dir, unsigned long attrs);
    void (*unmap_page)(struct device *dev, dma_addr_t dma_handle,
            size_t size, enum dma_data_direction dir,
            unsigned long attrs);

    int (*map_sg)(struct device *dev, struct scatterlist *sg, int nents,
            enum dma_data_direction dir, unsigned long attrs);
    void (*unmap_sg)(struct device *dev, struct scatterlist *sg, int nents,
            enum dma_data_direction dir, unsigned long attrs);

    dma_addr_t (*map_resource)(struct device *dev, phys_addr_t phys_addr,
            size_t size, enum dma_data_direction dir,
            unsigned long attrs);
    void (*unmap_resource)(struct device *dev, dma_addr_t dma_handle,
            size_t size, enum dma_data_direction dir,
            unsigned long attrs);

    void (*sync_single_for_cpu)(struct device *dev, dma_addr_t dma_handle,
            size_t size, enum dma_data_direction dir);
    void (*sync_single_for_device)(struct device *dev,
            dma_addr_t dma_handle, size_t size,
            enum dma_data_direction dir);
    void (*sync_sg_for_cpu)(struct device *dev, struct scatterlist *sg,
            int nents, enum dma_data_direction dir);
    void (*sync_sg_for_device)(struct device *dev, struct scatterlist *sg,
            int nents, enum dma_data_direction dir);

    void (*cache_sync)(struct device *dev, void *vaddr, size_t size,
            enum dma_data_direction direction);
    int (*dma_supported)(struct device *dev, u64 mask);
    u64 (*get_required_mask)(struct device *dev);
    size_t (*max_mapping_size)(struct device *dev);
    size_t (*opt_mapping_size)(void);
    unsigned long (*get_merge_boundary)(struct device *dev);
};
```

### Getting and Setting DMA Ops

```c
/* include/linux/dma-map-ops.h:76-87 */
static inline const struct dma_map_ops *get_dma_ops(struct device *dev)
{
    if (dev->dma_ops)
        return dev->dma_ops;
    return get_arch_dma_ops();     /* Architecture default */
}

static inline void set_dma_ops(struct device *dev,
                               const struct dma_map_ops *dma_ops)
{
    dev->dma_ops = dma_ops;
}
```

### Dispatch Logic

The DMA mapping layer uses a three-way dispatch. Every mapping function
follows this pattern:

```c
/* Simplified dispatch pattern (kernel/dma/mapping.c) */
if (dma_map_direct(dev, ops))
    /* Path 1: Direct physical mapping (identity or offset) */
    result = dma_direct_*(...);
else if (use_dma_iommu(dev))
    /* Path 2: IOMMU-based mapping */
    result = iommu_dma_*(...);
else
    /* Path 3: Custom platform dma_map_ops */
    result = ops->*(...);
```

```
+------------------------------------------------------------------+
|              DMA Mapping Dispatch Architecture                    |
+------------------------------------------------------------------+
|                                                                   |
|  Driver calls:  dma_map_page_attrs(dev, ...)                      |
|                         |                                         |
|                         v                                         |
|              +---------------------+                              |
|              |  get_dma_ops(dev)   |                              |
|              +-----+-------+------+                               |
|                    |       |                                      |
|         ops==NULL  |       |  ops!=NULL                           |
|         (common)   |       |                                      |
|                    v       v                                      |
|        +----------+--+ +--+-----------+                           |
|        | dma_map_     | | use_dma_    |                           |
|        | direct()? ---+-+ iommu()?    |                           |
|        +--+--------+  | +--+------+--+                            |
|           |        |  |    |      |                               |
|       yes |     no |  | yes|   no |                               |
|           v        |  |    v      v                               |
|   +-------+------+ |  | +--+--+ +-+--------+                      |
|   | dma_direct_  | |  | |iommu| |ops->     |                      |
|   | map_page()   | |  | |_dma_| |map_page()|                      |
|   | (phys->dma   | |  | |map_ | |          |                      |
|   |  + SWIOTLB)  | |  | |page | |(custom)  |                      |
|   +--------------+ |  | +-----+ +----------+                      |
|                    |  |                                           |
|                    +--+                                           |
|                                                                   |
+------------------------------------------------------------------+
```

---

## Direct DMA (No IOMMU)

When no IOMMU is present (or bypassed), the DMA layer maps physical memory
directly, applying a simple offset between CPU physical addresses and bus
addresses.

### Key Operations

```c
/* kernel/dma/direct.h:83-112 */
static inline dma_addr_t dma_direct_map_page(struct device *dev,
        struct page *page, unsigned long offset, size_t size,
        enum dma_data_direction dir, unsigned long attrs)
{
    phys_addr_t phys = page_to_phys(page) + offset;
    dma_addr_t dma_addr = phys_to_dma(dev, phys);

    if (is_swiotlb_force_bounce(dev)) {
        if (is_pci_p2pdma_page(page))
            return DMA_MAPPING_ERROR;
        return swiotlb_map(dev, phys, size, dir, attrs);
    }

    if (unlikely(!dma_capable(dev, dma_addr, size, true)) ||
        dma_kmalloc_needs_bounce(dev, size, dir)) {
        if (is_pci_p2pdma_page(page))
            return DMA_MAPPING_ERROR;
        if (is_swiotlb_active(dev))
            return swiotlb_map(dev, phys, size, dir, attrs);

        dev_WARN_ONCE(dev, 1,
                     "DMA addr %pad+%zu overflow (mask %llx, bus limit %llx).\n",
                     &dma_addr, size, *dev->dma_mask, dev->bus_dma_limit);
        return DMA_MAPPING_ERROR;
    }

    if (!dev_is_dma_coherent(dev) && !(attrs & DMA_ATTR_SKIP_CPU_SYNC))
        arch_sync_dma_for_device(phys, size, dir);
    return dma_addr;
}
```

The direct mapping path does the following:
1. Converts CPU physical address to DMA address via `phys_to_dma()`
2. Checks if the device can address the result (`dma_capable()`)
3. If not addressable, bounces through SWIOTLB
4. Performs cache sync for non-coherent devices

### DMA Range Map (CPU<->DMA Offset)

For platforms where bus addresses differ from physical addresses by a
constant offset, the `dma_range_map` on struct device stores the mapping:

```c
/* kernel/dma/direct.c:658-680 */
int dma_direct_set_offset(struct device *dev, phys_addr_t cpu_start,
                          dma_addr_t dma_start, u64 size)
{
    struct bus_dma_region *map;
    u64 offset = (u64)cpu_start - (u64)dma_start;

    if (dev->dma_range_map)
        return -EINVAL;
    if (!offset)
        return 0;

    map = kcalloc(2, sizeof(*map), GFP_KERNEL);
    if (!map)
        return -ENOMEM;
    map[0].cpu_start = cpu_start;
    map[0].dma_start = dma_start;
    map[0].size = size;
    dev->dma_range_map = map;
    return 0;
}
```

### Zone Selection

Direct DMA allocation uses GFP zone flags to allocate from memory that the
device can address:

```c
/* kernel/dma/direct.c:47-67 */
static gfp_t dma_direct_optimal_gfp_mask(struct device *dev, u64 *phys_limit)
{
    u64 dma_limit = min_not_zero(
        dev->coherent_dma_mask,
        dev->bus_dma_limit);

    *phys_limit = dma_to_phys(dev, dma_limit);
    if (*phys_limit <= zone_dma_limit)
        return GFP_DMA;           /* First 16 MB (arch-dependent) */
    if (*phys_limit <= DMA_BIT_MASK(32))
        return GFP_DMA32;         /* First 4 GB */
    return 0;                      /* Any memory */
}
```

---

## IOMMU-Backed DMA

When an IOMMU (Input/Output Memory Management Unit) is present, the DMA
mapping layer can use it to remap device addresses through IOMMU page tables.
This provides:

- **Address translation**: Any physical page can be mapped to any device
  address, eliminating addressing limitations
- **Scatter-gather coalescing**: Non-contiguous physical pages appear
  contiguous to the device
- **Device isolation**: Devices can only access mapped memory, preventing
  errant DMA from corrupting other memory

```
+------------------------------------------------------------------+
|                   IOMMU-Backed DMA                                |
+------------------------------------------------------------------+
|                                                                   |
|  CPU                            Device                            |
|  +---------+                    +--------+                        |
|  | Virtual |                    | DMA    |                        |
|  | Address |                    | Engine |                        |
|  +----+----+                    +---+----+                        |
|       |                             |                             |
|       | CPU MMU                     | IOVA (I/O Virtual Address)  |
|       v                             v                             |
|  +----+----+                  +-----+------+                      |
|  | Physical|                  |   IOMMU    |                      |
|  | Address |                  | Page Tables|                      |
|  +----+----+                  +-----+------+                      |
|       |                             |                             |
|       v                             v                             |
|  +----+-----------------------------+----+                        |
|  |            Physical RAM               |                        |
|  +---------------------------------------+                        |
|                                                                   |
|  Benefits:                                                        |
|  - 32-bit device can access all RAM via IOVA remapping            |
|  - Non-contiguous pages look contiguous to device                 |
|  - Device isolation (protection from DMA attacks)                 |
|                                                                   |
+------------------------------------------------------------------+
```

The IOMMU DMA path is selected when `use_dma_iommu(dev)` returns true.
The implementation lives in `drivers/iommu/dma-iommu.c` and manages
IOVA (I/O Virtual Address) allocation, IOMMU page table programming,
and flush operations.

When the DMA mask is large enough to address all memory, the IOMMU can
be bypassed for performance (the `dma_ops_bypass` path):

```c
/* kernel/dma/mapping.c:120-135 */
static bool dma_go_direct(struct device *dev, dma_addr_t mask,
        const struct dma_map_ops *ops)
{
    if (use_dma_iommu(dev))
        return false;
    if (likely(!ops))
        return true;
#ifdef CONFIG_DMA_OPS_BYPASS
    if (dev->dma_ops_bypass)
        return min_not_zero(mask, dev->bus_dma_limit) >=
                    dma_direct_get_required_mask(dev);
#endif
    return false;
}
```

---

## Bounce Buffers (SWIOTLB)

SWIOTLB (Software I/O Translation Lookaside Buffer) provides bounce
buffering for devices that cannot address all of physical memory. When a
DMA mapping targets memory outside a device's addressable range, SWIOTLB
copies the data to/from a pre-allocated buffer in low memory that the
device can reach.

### SWIOTLB Architecture

```
+------------------------------------------------------------------+
|                    SWIOTLB Bounce Buffering                       |
+------------------------------------------------------------------+
|                                                                   |
|  High Memory (above device DMA mask)                              |
|  +---------------------+                                          |
|  | Original Buffer     |<--- CPU accesses here                    |
|  | phys: 0x1_0000_0000 |                                          |
|  +-----+---------------+                                          |
|        |                                                          |
|        | copy on map (DMA_TO_DEVICE)                              |
|        | copy on unmap/sync (DMA_FROM_DEVICE)                     |
|        v                                                          |
|  Low Memory (within device DMA mask)                              |
|  +---------------------+                                          |
|  | SWIOTLB Bounce Buf  |<--- Device DMA targets here              |
|  | phys: 0x0100_0000   |                                          |
|  +---------------------+                                          |
|                                                                   |
|  The SWIOTLB pool is allocated at boot from low memory            |
|  Default size: 64 MB, configurable via swiotlb= boot parameter    |
|                                                                   |
+------------------------------------------------------------------+
```

### Core Data Structures

```c
/* include/linux/swiotlb.h:70-85 */
struct io_tlb_pool {
    phys_addr_t start;              /* Pool start physical address */
    phys_addr_t end;                /* Pool end physical address */
    void *vaddr;                    /* Virtual address (may be remapped) */
    unsigned long nslabs;           /* Number of IO TLB slots */
    bool late_alloc;                /* Allocated via page allocator */
    unsigned int nareas;            /* Number of areas */
    unsigned int area_nslabs;       /* Slots per area */
    struct io_tlb_area *areas;      /* Per-area descriptors */
    struct io_tlb_slot *slots;      /* Per-slot descriptors */
#ifdef CONFIG_SWIOTLB_DYNAMIC
    struct list_head node;          /* List node for pool list */
    struct rcu_head rcu;            /* RCU for deferred free */
    bool transient;                 /* Transient pool flag */
#endif
};

/* include/linux/swiotlb.h:108-126 */
struct io_tlb_mem {
    struct io_tlb_pool defpool;     /* Default (initial) pool */
    unsigned long nslabs;           /* Total slots across all pools */
    struct dentry *debugfs;         /* Debugfs entry */
    bool force_bounce;              /* Force all DMA through bounce */
    bool for_alloc;                 /* Used for memory allocation */
#ifdef CONFIG_SWIOTLB_DYNAMIC
    bool can_grow;                  /* Can allocate more pools */
    u64 phys_limit;                 /* Maximum physical address */
    spinlock_t lock;                /* Protects pool list */
    struct list_head pools;         /* List of all pools */
    struct work_struct dyn_alloc;   /* Deferred pool allocation */
#endif
};
```

### Per-Slot Tracking

```c
/* kernel/dma/swiotlb.c:67-80 */
struct io_tlb_slot {
    phys_addr_t orig_addr;          /* Original physical address */
    size_t alloc_size;              /* Allocated buffer size */
    unsigned short list;            /* Free list counter */
    unsigned short pad_slots;       /* Preceding padding slots */
};
```

### Per-Area Locking

```c
/* kernel/dma/swiotlb.c:115-119 */
struct io_tlb_area {
    unsigned long used;             /* Used slot count */
    unsigned int index;             /* Next search start index */
    spinlock_t lock;                /* Protects this area */
};
```

Each SWIOTLB pool is divided into multiple areas, each with its own lock,
to reduce contention on multi-CPU systems. Slots are 2 KB each
(`IO_TLB_SIZE = 1 << IO_TLB_SHIFT = 1 << 11 = 2048`).

### Key Parameters

```c
/* include/linux/swiotlb.h:26-36 */
#define IO_TLB_SEGSIZE  128             /* Max contiguous slabs per mapping */
#define IO_TLB_SHIFT    11              /* log2(slab size) = 2 KB */
#define IO_TLB_SIZE     (1 << IO_TLB_SHIFT)
#define IO_TLB_DEFAULT_SIZE (64UL<<20)  /* Default pool: 64 MB */
```

### SWIOTLB Bounce Path

The bounce path is triggered from `dma_direct_map_page()` when the
physical address falls outside the device's DMA mask:

```c
/* kernel/dma/direct.h:90-101 (simplified) */
static inline dma_addr_t dma_direct_map_page(...)
{
    phys_addr_t phys = page_to_phys(page) + offset;
    dma_addr_t dma_addr = phys_to_dma(dev, phys);

    if (is_swiotlb_force_bounce(dev))
        return swiotlb_map(dev, phys, size, dir, attrs);

    if (unlikely(!dma_capable(dev, dma_addr, size, true)))
        if (is_swiotlb_active(dev))
            return swiotlb_map(dev, phys, size, dir, attrs);
    /* ... */
}
```

`swiotlb_map()` allocates a bounce buffer slot, copies data for
`DMA_TO_DEVICE`, and returns the DMA address of the bounce buffer. On
unmap, data is copied back for `DMA_FROM_DEVICE`.

### Dynamic SWIOTLB

With `CONFIG_SWIOTLB_DYNAMIC`, the SWIOTLB can allocate additional pools
at runtime when the default pool is exhausted, up to `phys_limit`. Pool
allocation happens asynchronously via `swiotlb_dyn_alloc()` work.

---

## DMA Pools

DMA pools (`struct dma_pool`) provide efficient allocation of small,
fixed-size DMA-coherent buffers. They are built on top of
`dma_alloc_coherent()`, splitting full pages into smaller blocks.

### Data Structures

```c
/* mm/dmapool.c:43-62 */
struct dma_block {
    struct dma_block *next_block;
    dma_addr_t dma;
};

struct dma_pool {
    struct list_head page_list;     /* List of allocated pages */
    spinlock_t lock;                /* Protects pool state */
    struct dma_block *next_block;   /* Head of free list */
    size_t nr_blocks;               /* Total blocks across all pages */
    size_t nr_active;               /* Currently allocated blocks */
    size_t nr_pages;                /* Number of backing pages */
    struct device *dev;             /* Owning device */
    unsigned int size;              /* Block size */
    unsigned int allocation;        /* Page allocation size */
    unsigned int boundary;          /* Must not cross this boundary */
    int node;                       /* NUMA node */
    char name[32];                  /* Pool name (for debugfs) */
    struct list_head pools;         /* Link in dev->dma_pools */
};

struct dma_page {
    struct list_head page_list;     /* Link in pool->page_list */
    void *vaddr;                    /* CPU virtual address */
    dma_addr_t dma;                 /* DMA base address */
};
```

### API

```c
/* include/linux/dmapool.h */
struct dma_pool *dma_pool_create(const char *name, struct device *dev,
        size_t size, size_t align, size_t boundary);

void dma_pool_destroy(struct dma_pool *pool);

void *dma_pool_alloc(struct dma_pool *pool, gfp_t mem_flags,
                     dma_addr_t *handle);

void dma_pool_free(struct dma_pool *pool, void *vaddr, dma_addr_t addr);

/* Managed variants */
struct dma_pool *dmam_pool_create(const char *name, struct device *dev,
                                  size_t size, size_t align, size_t allocation);
void dmam_pool_destroy(struct dma_pool *pool);
```

### Usage Example

```c
/* Creating a pool for 64-byte descriptors, 64-byte aligned,
   that must not cross 4K boundaries */
pool = dma_pool_create("my_desc", dev, 64, 64, 4096);

/* Allocate a descriptor */
desc = dma_pool_alloc(pool, GFP_KERNEL, &desc_dma);
/* desc is CPU address, desc_dma is device DMA address */

/* Use the descriptor... */

/* Free */
dma_pool_free(pool, desc, desc_dma);

/* Destroy pool when done */
dma_pool_destroy(pool);
```

---

## Coherent Memory Pools

Per-device coherent memory pools allow devices to use dedicated memory
regions (often specified via device tree `reserved-memory` nodes) for
coherent DMA allocations.

### Data Structure

```c
/* kernel/dma/coherent.c:13-21 */
struct dma_coherent_mem {
    void            *virt_base;             /* Kernel virtual base */
    dma_addr_t      device_base;            /* DMA base address */
    unsigned long   pfn_base;               /* Physical page frame base */
    int             size;                   /* Size in pages */
    unsigned long   *bitmap;                /* Allocation bitmap */
    spinlock_t      spinlock;               /* Protects bitmap */
    bool            use_dev_dma_pfn_offset; /* Use device DMA PFN offset */
};
```

### Declaration and Allocation

```c
/* kernel/dma/coherent.c:117-131 */
int dma_declare_coherent_memory(struct device *dev, phys_addr_t phys_addr,
                                dma_addr_t device_addr, size_t size);

/* kernel/dma/coherent.c:187-197 */
int dma_alloc_from_dev_coherent(struct device *dev, ssize_t size,
        dma_addr_t *dma_handle, void **ret);
```

When a device has a declared coherent memory region, `dma_alloc_attrs()`
checks it first:

```c
/* kernel/dma/mapping.c:608 */
if (dma_alloc_from_dev_coherent(dev, size, dma_handle, &cpu_addr))
    return cpu_addr;
```

The allocation uses a simple bitmap allocator within the reserved region.

### Global Coherent Pool

For architectures without hardware cache coherence and no per-device pools,
a global coherent memory pool can be configured:

```c
/* kernel/dma/coherent.c:311-321 */
int dma_init_global_coherent(phys_addr_t phys_addr, size_t size);
```

This is typically set up via device tree with the `linux,dma-default`
property on a `shared-dma-pool` reserved memory node.

---

## Atomic DMA Pools

When `dma_alloc_coherent()` is called from a context that cannot block
(e.g., interrupt handlers), it must allocate from pre-allocated atomic
pools instead of the page allocator.

### Pool Organization

```c
/* kernel/dma/pool.c:16-21 */
static struct gen_pool *atomic_pool_dma __ro_after_init;
static struct gen_pool *atomic_pool_dma32 __ro_after_init;
static struct gen_pool *atomic_pool_kernel __ro_after_init;
```

Three pools are maintained for different memory zones:
- `atomic_pool_dma`: For allocations requiring ZONE_DMA (usually <16 MB)
- `atomic_pool_dma32`: For allocations requiring ZONE_DMA32 (<4 GB)
- `atomic_pool_kernel`: For allocations from any zone

### Pool Sizing

```c
/* kernel/dma/pool.c:187-221 */
static int __init dma_atomic_pool_init(void)
{
    /* Default: 128KB per 1GB of RAM, min 128KB, max MAX_PAGE_ORDER */
    if (!atomic_pool_size) {
        unsigned long pages = totalram_pages() / (SZ_1G / SZ_128K);
        pages = min_t(unsigned long, pages, MAX_ORDER_NR_PAGES);
        atomic_pool_size = max_t(size_t, pages << PAGE_SHIFT, SZ_128K);
    }
    /* ... create pools ... */
}
postcore_initcall(dma_atomic_pool_init);
```

The `coherent_pool=` kernel command-line parameter overrides the default
pool size.

### Pool Selection and Expansion

```c
/* kernel/dma/pool.c:224-238 */
static inline struct gen_pool *dma_guess_pool(struct gen_pool *prev, gfp_t gfp)
{
    if (prev == NULL) {
        if (IS_ENABLED(CONFIG_ZONE_DMA32) && (gfp & GFP_DMA32))
            return atomic_pool_dma32;
        if (atomic_pool_dma && (gfp & GFP_DMA))
            return atomic_pool_dma;
        return atomic_pool_kernel;
    }
    /* Fall through to next pool if previous failed */
    if (prev == atomic_pool_kernel)
        return atomic_pool_dma32 ? atomic_pool_dma32 : atomic_pool_dma;
    if (prev == atomic_pool_dma32)
        return atomic_pool_dma;
    return NULL;
}
```

When pool capacity drops below `atomic_pool_size`, a background work item
dynamically expands the pool:

```c
/* kernel/dma/pool.c:257-258 */
if (gen_pool_avail(pool) < atomic_pool_size)
    schedule_work(&atomic_pool_work);
```

---

## Cache Coherency

On architectures without hardware cache coherence between CPU and devices
(e.g., ARM without CCI/CCN), the DMA subsystem must explicitly manage CPU
caches. The kernel provides architecture callbacks for this:

### Architecture Cache Operations

```c
/* include/linux/dma-map-ops.h:347-389 */

/* Flush CPU caches so device sees latest data */
void arch_sync_dma_for_device(phys_addr_t paddr, size_t size,
        enum dma_data_direction dir);

/* Invalidate CPU caches so CPU sees what device wrote */
void arch_sync_dma_for_cpu(phys_addr_t paddr, size_t size,
        enum dma_data_direction dir);

/* Invalidate all CPU caches (some architectures) */
void arch_sync_dma_for_cpu_all(void);

/* Prepare a page for coherent DMA (clean + invalidate) */
void arch_dma_prep_coherent(struct page *page, size_t size);
```

### Coherence Check

```c
/* include/linux/dma-map-ops.h:234-237 */
static inline bool dev_is_dma_coherent(struct device *dev)
{
    return dev->dma_coherent;
}
```

### Sync Implementation for Direct DMA

```c
/* kernel/dma/direct.h:56-81 */
static inline void dma_direct_sync_single_for_device(struct device *dev,
        dma_addr_t addr, size_t size, enum dma_data_direction dir)
{
    phys_addr_t paddr = dma_to_phys(dev, addr);

    /* Copy from bounce buffer back to original if needed */
    swiotlb_sync_single_for_device(dev, paddr, size, dir);

    if (!dev_is_dma_coherent(dev))
        arch_sync_dma_for_device(paddr, size, dir);
}

static inline void dma_direct_sync_single_for_cpu(struct device *dev,
        dma_addr_t addr, size_t size, enum dma_data_direction dir)
{
    phys_addr_t paddr = dma_to_phys(dev, addr);

    if (!dev_is_dma_coherent(dev)) {
        arch_sync_dma_for_cpu(paddr, size, dir);
        arch_sync_dma_for_cpu_all();
    }

    /* Copy from bounce buffer to original if needed */
    swiotlb_sync_single_for_cpu(dev, paddr, size, dir);

    if (dir == DMA_FROM_DEVICE)
        arch_dma_mark_clean(paddr, size);
}
```

### Sync Skip Optimization

To avoid unnecessary sync calls on coherent devices, the subsystem
tracks whether sync is needed:

```c
/* include/linux/dma-mapping.h:291-295 */
static inline bool dma_dev_need_sync(const struct device *dev)
{
    /* Always call DMA sync operations when debugging is enabled */
    return !dev->dma_skip_sync || IS_ENABLED(CONFIG_DMA_API_DEBUG);
}
```

This flag is set during `dma_set_mask()` and reset if a SWIOTLB bounce
buffer is ever used (since bounce buffers always need sync):

```c
/* kernel/dma/mapping.c:446-466 */
static void dma_setup_need_sync(struct device *dev)
{
    const struct dma_map_ops *ops = get_dma_ops(dev);

    if (dma_map_direct(dev, ops) || use_dma_iommu(dev))
        dev->dma_skip_sync = dev_is_dma_coherent(dev);
    else if (!ops->sync_single_for_device && !ops->sync_single_for_cpu &&
             !ops->sync_sg_for_device && !ops->sync_sg_for_cpu)
        dev->dma_skip_sync = true;
    else
        dev->dma_skip_sync = false;
}
```

### Bounce Buffer Alignment for Non-Coherent DMA

On non-coherent architectures, kmalloc buffers may not be aligned to cache
line boundaries, causing corruption when DMA cache maintenance affects
neighboring data. The subsystem detects this:

```c
/* include/linux/dma-map-ops.h:310-314 */
static inline bool dma_kmalloc_needs_bounce(struct device *dev, size_t size,
                                            enum dma_data_direction dir)
{
    return !dma_kmalloc_safe(dev, dir) && !dma_kmalloc_size_aligned(size);
}
```

When `CONFIG_DMA_BOUNCE_UNALIGNED_KMALLOC` is enabled, such buffers are
automatically bounced through SWIOTLB to ensure proper alignment.

---

## DMA Fences

DMA fences (`struct dma_fence`) are synchronization primitives used primarily
by the GPU/display subsystem (DRM/KMS) and `dma-buf` framework to track
when asynchronous hardware operations complete. They are distinct from the
DMA mapping API but are part of the broader DMA subsystem.

### Data Structure

```c
/* include/linux/dma-fence.h:66-96 */
struct dma_fence {
    spinlock_t *lock;
    const struct dma_fence_ops *ops;

    union {
        struct list_head cb_list;       /* Callbacks before signal */
        ktime_t timestamp;              /* Set on signal */
        struct rcu_head rcu;            /* Used on release */
    };
    u64 context;                        /* Execution context ID */
    u64 seqno;                          /* Sequence number */
    unsigned long flags;                /* DMA_FENCE_FLAG_* */
    struct kref refcount;
    int error;
};
```

### Operations

```c
/* include/linux/dma-fence.h:126 */
struct dma_fence_ops {
    bool use_64bit_seqno;

    const char *(*get_driver_name)(struct dma_fence *fence);
    const char *(*get_timeline_name)(struct dma_fence *fence);
    bool (*enable_signaling)(struct dma_fence *fence);
    bool (*signaled)(struct dma_fence *fence);
    signed long (*wait)(struct dma_fence *fence, bool intr,
                        signed long timeout);
    void (*release)(struct dma_fence *fence);
    /* ... */
};
```

### Key Operations

```c
/* Signal that the operation is complete */
int dma_fence_signal(struct dma_fence *fence);

/* Wait for fence to be signaled */
signed long dma_fence_wait_timeout(struct dma_fence *fence, bool intr,
                                   signed long timeout);

/* Add a callback for when fence signals */
int dma_fence_add_callback(struct dma_fence *fence,
                           struct dma_fence_cb *cb,
                           dma_fence_func_t func);

/* Context allocation for a new timeline */
u64 dma_fence_context_alloc(unsigned num);
```

Fences are used by `dma-buf` for buffer sharing between devices (e.g.,
GPU renders to a buffer, display controller reads it). The fence ensures
the display does not read until the GPU has finished writing.

---

## DMA Debugging

The DMA debugging infrastructure (`CONFIG_DMA_API_DEBUG`) tracks all DMA
mappings and validates correct API usage.

### Debug Entry Tracking

```c
/* kernel/dma/debug.c:53-82 */
struct dma_debug_entry {
    struct list_head list;
    struct device    *dev;
    u64              dev_addr;          /* DMA address */
    u64              size;              /* Mapping size */
    int              type;              /* single, sg, coherent, resource */
    int              direction;         /* DMA direction */
    int              sg_call_ents;      /* nents from dma_map_sg */
    int              sg_mapped_ents;    /* Mapped entries from dma_map_sg */
    phys_addr_t      paddr;            /* Physical address */
    enum map_err_types  map_err_type;  /* Error check tracking */
#ifdef CONFIG_STACKTRACE
    unsigned int     stack_len;
    unsigned long    stack_entries[DMA_DEBUG_STACKTRACE_ENTRIES];
#endif
};
```

Entries are stored in a hash table keyed by DMA address:

```c
/* kernel/dma/debug.c:29-33 */
#define HASH_SIZE       16384ULL
#define PREALLOC_DMA_DEBUG_ENTRIES (1 << 16)    /* 65536 pre-allocated */
```

### What the Debug Layer Checks

The debug layer validates:

1. **Missing unmap**: Warns if a device is removed with outstanding DMA
   mappings
2. **Double free**: Detects unmapping the same address twice
3. **Wrong direction**: Catches direction mismatch between map and unmap
4. **Missing error check**: Warns if `dma_mapping_error()` is not called
   after mapping
5. **Sync without map**: Detects sync calls on addresses that are not
   currently mapped
6. **Size mismatch**: Checks that unmap size matches map size
7. **Device mismatch**: Catches unmapping from a different device than
   the one that mapped

### Debug Functions

The debug layer hooks into every DMA operation:

```c
/* kernel/dma/debug.h */
void debug_dma_map_page(struct device *dev, struct page *page,
        size_t offset, size_t size, int direction,
        dma_addr_t dma_addr, unsigned long attrs);
void debug_dma_unmap_page(struct device *dev, dma_addr_t addr,
        size_t size, int direction);
void debug_dma_map_sg(struct device *dev, struct scatterlist *sg,
        int nents, int mapped_ents, int direction, unsigned long attrs);
void debug_dma_unmap_sg(struct device *dev, struct scatterlist *sglist,
        int nelems, int dir);
void debug_dma_alloc_coherent(struct device *dev, size_t size,
        dma_addr_t dma_addr, void *virt, unsigned long attrs);
void debug_dma_free_coherent(struct device *dev, size_t size,
        void *virt, dma_addr_t addr);
void debug_dma_sync_single_for_cpu(struct device *dev,
        dma_addr_t dma_handle, size_t size, int direction);
void debug_dma_sync_single_for_device(struct device *dev,
        dma_addr_t dma_handle, size_t size, int direction);
```

### Enabling DMA Debugging

Enable via kernel config `CONFIG_DMA_API_DEBUG=y` or boot parameter
`dma_debug=1`. Mappings can be dumped via debugfs:

```c
/* include/linux/dma-map-ops.h:424-426 */
void dma_debug_add_bus(const struct bus_type *bus);
void debug_dma_dump_mappings(struct device *dev);
```

---

## Source Files

### Core DMA Mapping (`kernel/dma/`)

| File | Description |
|------|-------------|
| `mapping.c` | Central dispatch: `dma_map_page_attrs()`, `dma_alloc_attrs()`, `dma_map_sg_attrs()`, `dma_set_mask()`, managed DMA (`dmam_*`) |
| `direct.c` | Direct (no-IOMMU) DMA: `dma_direct_alloc()`, `dma_direct_map_page()`, zone selection, CMA/contiguous allocation |
| `direct.h` | Inline direct mapping functions: `dma_direct_map_page()`, `dma_direct_sync_single_for_{cpu,device}()` |
| `swiotlb.c` | Software bounce buffering: `swiotlb_tbl_map_single()`, slot allocation, area-based locking, dynamic pool growth |
| `pool.c` | Atomic DMA pools: `dma_alloc_from_pool()`, `dma_free_from_pool()`, per-zone `gen_pool` management |
| `coherent.c` | Per-device coherent memory: `dma_declare_coherent_memory()`, bitmap allocator, global coherent pool |
| `debug.c` | DMA API debug: hash table tracking, validation checks, debugfs interface |
| `debug.h` | Debug function declarations: `debug_dma_map_page()`, `debug_dma_alloc_coherent()`, etc. |

### DMA Pool Allocator (`mm/`)

| File | Description |
|------|-------------|
| `mm/dmapool.c` | Small coherent buffer pool: `dma_pool_create()`, `dma_pool_alloc()`, free-list management |

### IOMMU DMA (`drivers/iommu/`)

| File | Description |
|------|-------------|
| `drivers/iommu/dma-iommu.c` | IOMMU-backed DMA: IOVA allocation, page table programming, `iommu_dma_map_page()`, `iommu_dma_alloc()` |

### DMA Fence (`drivers/dma-buf/`)

| File | Description |
|------|-------------|
| `drivers/dma-buf/dma-fence.c` | Fence implementation: `dma_fence_signal()`, `dma_fence_wait()`, callback management |
| `drivers/dma-buf/dma-fence-array.c` | Array of fences that signal when all complete |
| `drivers/dma-buf/dma-fence-chain.c` | Chain of fences for timeline semantics |

### Header Files

| File | Description |
|------|-------------|
| `include/linux/dma-mapping.h` | Public DMA API: `dma_map_single()`, `dma_alloc_coherent()`, `dma_set_mask()`, DMA attributes, sync wrappers |
| `include/linux/dma-map-ops.h` | `struct dma_map_ops`, `get_dma_ops()`, CMA helpers, arch sync callbacks, `dev_is_dma_coherent()` |
| `include/linux/dma-direction.h` | `enum dma_data_direction` |
| `include/linux/dma-direct.h` | Direct DMA helpers: `phys_to_dma()`, `dma_to_phys()`, `dma_capable()` |
| `include/linux/scatterlist.h` | `struct scatterlist`, `struct sg_table`, SG iteration macros |
| `include/linux/swiotlb.h` | `struct io_tlb_pool`, `struct io_tlb_mem`, `swiotlb_find_pool()`, SWIOTLB API |
| `include/linux/dmapool.h` | `dma_pool_create()`, `dma_pool_alloc()`, `dma_pool_free()` |
| `include/linux/dma-fence.h` | `struct dma_fence`, `struct dma_fence_ops`, fence API |
| `include/linux/device.h` | `struct device` DMA fields, `struct device_dma_parameters` |

---

## Summary

The DMA mapping subsystem provides a unified abstraction for device memory
access across diverse hardware configurations:

| Component | Purpose |
|-----------|---------|
| `dma_map_single/page` | Map CPU buffer for device access (streaming) |
| `dma_map_sg` | Map scatter-gather list for device access |
| `dma_alloc_coherent` | Allocate coherent CPU+device shared memory |
| `dma_sync_*` | Transfer buffer ownership between CPU and device |
| `dma_set_mask` | Declare device addressing capabilities |
| `dma_map_ops` | Backend abstraction (direct, IOMMU, custom) |
| Direct DMA | Identity/offset mapping without IOMMU |
| IOMMU DMA | IOVA-based remapping with page tables |
| SWIOTLB | Bounce buffering for limited-address devices |
| DMA pools | Small coherent buffer allocator |
| Atomic pools | Non-blocking coherent allocation |
| DMA fences | Async hardware completion synchronization |
| DMA debug | Runtime validation of API usage |

Key design principles:
- **Three-way dispatch**: Every operation routes through direct, IOMMU, or custom ops
- **Ownership model**: Streaming buffers have explicit CPU/device ownership
- **Lazy sync**: Skip unnecessary cache operations on coherent hardware
- **Graceful degradation**: SWIOTLB provides a fallback when addresses exceed device capabilities
- **Managed resources**: `dmam_*` variants integrate with devres for automatic cleanup
