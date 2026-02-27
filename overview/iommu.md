# Linux Kernel IOMMU Subsystem Overview

## Table of Contents
1. [Introduction](#1-introduction)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Data Structures](#3-core-data-structures)
4. [IOMMU Operations](#4-iommu-operations)
5. [Domain Types](#5-domain-types)
6. [IOMMU Groups](#6-iommu-groups)
7. [DMA API Integration](#7-dma-api-integration)
8. [Page Table Management and IOTLB](#8-page-table-management-and-iotlb)
9. [IOMMUFD Userspace Interface](#9-iommufd-userspace-interface)
10. [VFIO and Device Passthrough](#10-vfio-and-device-passthrough)
11. [Shared Virtual Addressing (SVA)](#11-shared-virtual-addressing-sva)
12. [Hardware Implementations](#12-hardware-implementations)
13. [Fault Handling and I/O Page Faults](#13-fault-handling-and-io-page-faults)
14. [Initialization and Boot Flow](#14-initialization-and-boot-flow)
15. [Key Source Files](#15-key-source-files)

---

## 1. Introduction

The IOMMU (Input/Output Memory Management Unit) subsystem provides hardware-
assisted memory address translation and access control for DMA (Direct Memory
Access) transactions initiated by devices. It sits between I/O devices and
system memory, translating device-visible I/O Virtual Addresses (IOVAs) into
physical addresses, much like how the CPU's MMU translates virtual addresses
for processes.

The IOMMU subsystem serves several critical purposes:

- **DMA Remapping**: Translates device DMA addresses through page tables,
  allowing devices to use virtual address spaces independent of physical
  memory layout
- **Device Isolation**: Prevents devices from accessing memory outside their
  assigned regions, protecting against buggy or malicious hardware/firmware
- **Virtualization Support**: Enables safe device passthrough to virtual
  machines by constraining device DMA to guest-assigned memory
- **Scatter-Gather Optimization**: Maps physically discontiguous pages into
  a contiguous IOVA range, enabling efficient DMA for fragmented memory
- **Shared Virtual Addressing (SVA)**: Allows devices to share process
  address spaces with the CPU, using the same virtual addresses

### IOMMU in the System

```
                     CPU
                      |
                      v
              ┌───────────────┐
              │   MMU (CPU)   │   CPU virtual -> physical
              └───────┬───────┘
                      |
        ┌─────────────┴─────────────┐
        |                           |
   System Memory              ┌─────────┐
        ^                      │  IOMMU  │   IOVA -> physical
        |                      └────┬────┘
        |                           |
        |              ┌────────────┴────────────┐
        |              |            |             |
        |           ┌──┴──┐    ┌───┴───┐    ┌────┴────┐
        └───────────│ NIC │    │  GPU  │    │ NVMe    │
         (DMA)      └─────┘    └───────┘    └─────────┘
                     Devices issue DMA using IOVAs;
                     IOMMU translates to physical addresses
```

---

## 2. Architecture Overview

The Linux IOMMU subsystem is organized around a layered architecture: a
generic core framework (`drivers/iommu/iommu.c`) that provides the unified API,
hardware-specific drivers that implement the `iommu_ops` interface, and the DMA
layer that transparently uses the IOMMU when available.

### Subsystem Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Userspace                                    │
│   VFIO (/dev/vfio)    IOMMUFD (/dev/iommu)    DPDK / SPDK          │
├─────────────────────────────────────────────────────────────────────┤
│                     Kernel IOMMU Core                               │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │  DMA API     │  │  IOMMU API   │  │   IOMMUFD Subsystem    │    │
│  │  (dma-iommu) │  │  (iommu.c)   │  │   (iommufd/)           │    │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬────────────┘    │
│         └──────────────────┴─────────────────────┘                  │
│                            │                                        │
│                   struct iommu_ops                                   │
│                            │                                        │
├────────────┬───────────────┼───────────────┬────────────────────────┤
│            │               │               │                        │
│  ┌─────────┴──┐  ┌────────┴────┐  ┌───────┴───────┐               │
│  │ Intel VT-d │  │  AMD-Vi     │  │   ARM SMMU    │   ...         │
│  │ (intel/)   │  │  (amd/)     │  │   (arm/)      │               │
│  └────────────┘  └─────────────┘  └───────────────┘               │
├─────────────────────────────────────────────────────────────────────┤
│                      IOMMU Hardware                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Core Data Structures

### 3.1 struct iommu_domain

The domain represents an I/O address space with its own page tables. All
devices attached to the same domain share the same IOVA-to-physical mapping.
Domains are the fundamental unit of isolation -- devices in different domains
cannot access each other's memory.

The `type` field encodes the domain behavior using flag bits. The `ops` pointer
provides domain-specific operations (map, unmap, etc.), while `owner` tracks
which IOMMU driver allocated the domain. The `pgsize_bitmap` indicates which
page sizes the hardware supports for this domain, and `iova_cookie` holds the
DMA layer's IOVA allocator state when the domain is used for DMA API
translations. The union at the end holds either a legacy fault handler or SVA
state (the `mm` pointer and associated list linkage for shared address spaces).

```c
/* include/linux/iommu.h:210 */
struct iommu_domain {
    unsigned type;
    const struct iommu_domain_ops *ops;
    const struct iommu_dirty_ops *dirty_ops;
    const struct iommu_ops *owner;
    unsigned long pgsize_bitmap;
    struct iommu_domain_geometry geometry;
    struct iommu_dma_cookie *iova_cookie;
    int (*iopf_handler)(struct iopf_group *group);
    void *fault_data;
    union {
        struct {
            iommu_fault_handler_t handler;
            void *handler_token;
        };
        struct {    /* IOMMU_DOMAIN_SVA */
            struct mm_struct *mm;
            int users;
            struct list_head next;
        };
    };
};
```

### 3.2 struct iommu_group

An IOMMU group represents the smallest set of devices that the IOMMU hardware
can isolate from all other devices. Devices within a group can potentially
access each other's DMA, so they must all be assigned to the same domain. The
grouping is determined by hardware topology -- for example, devices behind a
non-ACS-capable PCI bridge are placed in the same group because the bridge may
route peer-to-peer DMA between them without going through the IOMMU.

The structure is defined privately within the IOMMU core to prevent drivers
from accessing internals directly. The `default_domain` is the kernel-managed
domain (DMA or identity), while `domain` tracks the currently active domain.
The `owner_cnt` and `owner` fields implement exclusive ownership for VFIO
passthrough -- once a userspace driver claims a group, the kernel DMA API
cannot be used for those devices simultaneously.

```c
/* drivers/iommu/iommu.c:48 */
struct iommu_group {
    struct kobject kobj;
    struct kobject *devices_kobj;
    struct list_head devices;
    struct xarray pasid_array;
    struct mutex mutex;
    void *iommu_data;
    void (*iommu_data_release)(void *iommu_data);
    char *name;
    int id;
    struct iommu_domain *default_domain;
    struct iommu_domain *blocking_domain;
    struct iommu_domain *domain;
    struct list_head entry;
    unsigned int owner_cnt;
    void *owner;
};
```

### 3.3 struct iommu_device

This represents one IOMMU hardware instance as registered with the core. Each
physical IOMMU unit in the system (e.g., each DMAR unit reported by ACPI for
Intel VT-d) is represented by one `iommu_device`. The `ops` pointer links to
the driver's operation table, `max_pasids` indicates how many Process Address
Space IDs the hardware supports, and `singleton_group` is an optimization for
drivers where every device gets its own group.

```c
/* include/linux/iommu.h:744 */
struct iommu_device {
    struct list_head list;
    const struct iommu_ops *ops;
    struct fwnode_handle *fwnode;
    struct device *dev;
    struct iommu_group *singleton_group;
    u32 max_pasids;
};
```

### 3.4 struct iommu_domain_geometry

Describes the valid IOVA range for a domain. Hardware IOMMUs often have a
limited address space. The `force_aperture` flag indicates whether DMA outside
this range is forbidden (true) or simply passes through untranslated (false).

```c
/* include/linux/iommu.h:160 */
struct iommu_domain_geometry {
    dma_addr_t aperture_start;
    dma_addr_t aperture_end;
    bool force_aperture;
};
```

### 3.5 struct iommu_iotlb_gather

Used to batch IOTLB (I/O TLB) invalidations. When unmapping pages, the core
accumulates the affected IOVA range and defers the expensive TLB flush until
the entire batch is complete. The `freelist` holds page table pages that can
only be freed after the TLB has been synchronized. The `queued` flag indicates
that the flush will happen via a flush queue rather than synchronously.

```c
/* include/linux/iommu.h:347 */
struct iommu_iotlb_gather {
    unsigned long start;
    unsigned long end;
    size_t pgsize;
    struct list_head freelist;
    bool queued;
};
```

---

## 4. IOMMU Operations

### 4.1 struct iommu_ops (Driver-Level Operations)

This is the primary interface that hardware-specific IOMMU drivers implement.
It covers device probing, domain allocation, group management, and feature
enablement. Each IOMMU driver (Intel, AMD, ARM SMMU, etc.) provides its own
implementation.

Key callbacks include domain allocators for different domain types (paging,
SVA, nested), device probing and release, group assignment logic, and reserved
region reporting. The `identity_domain` and `blocked_domain` are singleton
domains used for passthrough and blocking modes respectively.

```c
/* include/linux/iommu.h:615 */
struct iommu_ops {
    bool (*capable)(struct device *dev, enum iommu_cap);
    void *(*hw_info)(struct device *dev, u32 *length, u32 *type);

    /* Domain allocation and freeing by the iommu driver */
    struct iommu_domain *(*domain_alloc)(unsigned iommu_domain_type);
    struct iommu_domain *(*domain_alloc_paging_flags)(
        struct device *dev, u32 flags,
        const struct iommu_user_data *user_data);
    struct iommu_domain *(*domain_alloc_paging)(struct device *dev);
    struct iommu_domain *(*domain_alloc_sva)(struct device *dev,
                                             struct mm_struct *mm);
    struct iommu_domain *(*domain_alloc_nested)(
        struct device *dev, struct iommu_domain *parent, u32 flags,
        const struct iommu_user_data *user_data);

    struct iommu_device *(*probe_device)(struct device *dev);
    void (*release_device)(struct device *dev);
    void (*probe_finalize)(struct device *dev);
    struct iommu_group *(*device_group)(struct device *dev);

    void (*get_resv_regions)(struct device *dev, struct list_head *list);
    int (*of_xlate)(struct device *dev, const struct of_phandle_args *args);
    bool (*is_attach_deferred)(struct device *dev);

    int (*dev_enable_feat)(struct device *dev, enum iommu_dev_features f);
    int (*dev_disable_feat)(struct device *dev, enum iommu_dev_features f);

    void (*page_response)(struct device *dev, struct iopf_fault *evt,
                          struct iommu_page_response *msg);
    int (*def_domain_type)(struct device *dev);
    void (*remove_dev_pasid)(struct device *dev, ioasid_t pasid,
                             struct iommu_domain *domain);

    struct iommufd_viommu *(*viommu_alloc)(
        struct device *dev, struct iommu_domain *parent_domain,
        struct iommufd_ctx *ictx, unsigned int viommu_type);

    const struct iommu_domain_ops *default_domain_ops;
    unsigned long pgsize_bitmap;
    struct module *owner;
    struct iommu_domain *identity_domain;
    struct iommu_domain *blocked_domain;
    struct iommu_domain *release_domain;
    struct iommu_domain *default_domain;
    u8 user_pasid_table:1;
};
```

### 4.2 struct iommu_domain_ops (Domain-Level Operations)

Per-domain operations that handle the actual page table manipulation. These are
the workhorse callbacks for mapping and unmapping IOVAs, flushing TLBs, and
translating addresses.

```c
/* include/linux/iommu.h:705 */
struct iommu_domain_ops {
    int (*attach_dev)(struct iommu_domain *domain, struct device *dev);
    int (*set_dev_pasid)(struct iommu_domain *domain, struct device *dev,
                         ioasid_t pasid, struct iommu_domain *old);

    int (*map_pages)(struct iommu_domain *domain, unsigned long iova,
                     phys_addr_t paddr, size_t pgsize, size_t pgcount,
                     int prot, gfp_t gfp, size_t *mapped);
    size_t (*unmap_pages)(struct iommu_domain *domain, unsigned long iova,
                          size_t pgsize, size_t pgcount,
                          struct iommu_iotlb_gather *iotlb_gather);

    void (*flush_iotlb_all)(struct iommu_domain *domain);
    int (*iotlb_sync_map)(struct iommu_domain *domain, unsigned long iova,
                          size_t size);
    void (*iotlb_sync)(struct iommu_domain *domain,
                       struct iommu_iotlb_gather *iotlb_gather);
    int (*cache_invalidate_user)(struct iommu_domain *domain,
                                 struct iommu_user_data_array *array);

    phys_addr_t (*iova_to_phys)(struct iommu_domain *domain,
                                dma_addr_t iova);

    bool (*enforce_cache_coherency)(struct iommu_domain *domain);
    int (*set_pgtable_quirks)(struct iommu_domain *domain,
                              unsigned long quirks);
    void (*free)(struct iommu_domain *domain);
};
```

### 4.3 Key Core API Functions

The IOMMU core provides these primary functions for managing domains, groups,
and mappings:

| Function | Location | Purpose |
|---|---|---|
| `iommu_domain_alloc()` | `drivers/iommu/iommu.c` | Allocate an IOMMU domain (legacy) |
| `iommu_paging_domain_alloc_flags()` | `drivers/iommu/iommu.c:2014` | Allocate a paging domain with flags |
| `iommu_domain_free()` | `drivers/iommu/iommu.c:2022` | Free an IOMMU domain |
| `iommu_attach_device()` | `drivers/iommu/iommu.c:2076` | Attach a device to a domain |
| `iommu_detach_device()` | `drivers/iommu/iommu.c:2110` | Detach a device from a domain |
| `iommu_map()` | `drivers/iommu/iommu.c:2478` | Map IOVA to physical address |
| `iommu_unmap()` | `drivers/iommu/iommu.c:2576` | Unmap an IOVA range |
| `iommu_group_alloc()` | `drivers/iommu/iommu.c:965` | Allocate a new IOMMU group |
| `iommu_device_register()` | `drivers/iommu/iommu.c:248` | Register an IOMMU hardware instance |
| `iommu_get_domain_for_dev()` | `drivers/iommu/iommu.c:2129` | Get the current domain for a device |
| `iommu_attach_device_pasid()` | `include/linux/iommu.h:1130` | Attach a domain to a device's PASID |
| `iommu_sva_bind_device()` | `include/linux/iommu.h:1588` | Bind device for Shared Virtual Addressing |

---

## 5. Domain Types

The IOMMU subsystem defines several domain types, each serving a different
purpose. The type is encoded as a bitmask of feature flags.

### Domain Type Flags

```c
/* include/linux/iommu.h:167 */
#define __IOMMU_DOMAIN_PAGING  (1U << 0)  /* Support for iommu_map/unmap */
#define __IOMMU_DOMAIN_DMA_API (1U << 1)  /* Domain for DMA-API use      */
#define __IOMMU_DOMAIN_PT      (1U << 2)  /* Identity mapped             */
#define __IOMMU_DOMAIN_DMA_FQ  (1U << 3)  /* DMA-API uses flush queue    */
#define __IOMMU_DOMAIN_SVA     (1U << 4)  /* Shared process addr space   */
#define __IOMMU_DOMAIN_PLATFORM (1U << 5) /* Legacy platform domain      */
#define __IOMMU_DOMAIN_NESTED  (1U << 6)  /* Nested on stage-2           */
```

### Composite Domain Types

```
┌──────────────────────┬────────────────────────────────────────────────┐
│ Type                 │ Description                                    │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_BLOCKED │ All DMA blocked; used to isolate devices.     │
│ (0)                  │ No translations possible.                     │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_IDENTITY│ DMA addresses = physical addresses.           │
│ (PT)                 │ Passthrough mode, no translation overhead.    │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_        │ User-managed mappings (VFIO, vhost).          │
│ UNMANAGED (PAGING)   │ Caller controls all map/unmap operations.     │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_DMA     │ Kernel DMA API managed. The IOMMU core       │
│ (PAGING | DMA_API)   │ handles IOVA allocation and mapping for       │
│                      │ standard device drivers. Strict TLB flush.    │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_DMA_FQ  │ Like DMA, but with batched (lazy) TLB        │
│ (PAGING|DMA_API|FQ)  │ invalidation via a flush queue for            │
│                      │ higher performance.                           │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_SVA     │ Shares process virtual address space with     │
│                      │ device. Uses CPU page tables via PASID.       │
├──────────────────────┼────────────────────────────────────────────────┤
│ IOMMU_DOMAIN_NESTED  │ Guest-managed stage-1 translation nested on   │
│                      │ host stage-2. For VM device passthrough.      │
└──────────────────────┴────────────────────────────────────────────────┘
```

### Default Domain Selection

At boot, the kernel selects a default domain type for all IOMMU groups. This
can be controlled via kernel command line parameters:

- `iommu.passthrough=1` -- Use `IOMMU_DOMAIN_IDENTITY` (passthrough)
- `iommu.passthrough=0` -- Use `IOMMU_DOMAIN_DMA` (translated)
- `iommu.strict=1` -- Use strict TLB invalidation (`IOMMU_DOMAIN_DMA`)
- `iommu.strict=0` -- Use lazy TLB invalidation (`IOMMU_DOMAIN_DMA_FQ`)

The initialization logic is in `iommu_subsys_init()`
(`drivers/iommu/iommu.c:196`). When memory encryption is active (e.g., AMD
SEV), passthrough mode is automatically disabled because devices must not have
direct access to encrypted memory.

Individual IOMMU drivers can override the default via the `def_domain_type()`
callback in `iommu_ops`.

---

## 6. IOMMU Groups

### 6.1 Purpose and Semantics

An IOMMU group is the smallest granularity of device isolation that the
hardware can guarantee. All devices in a group must be in the same domain
because the hardware cannot distinguish their DMA traffic from each other.

Common reasons devices end up in the same group:

- **Non-ACS PCI bridge**: Devices behind a bridge that does not support Access
  Control Services may perform peer-to-peer DMA without traversing the IOMMU
- **Multi-function PCI devices**: Functions of the same physical device often
  share a request ID
- **Integrated devices**: Platform devices on SoCs that share an IOMMU stream
  ID

### 6.2 Group Lifecycle

```
  Driver probe                     Device enumeration
      │                                   │
      v                                   v
  iommu_probe_device()          ops->device_group(dev)
      │                                   │
      │    ┌──────────────────────────────┘
      v    v
  iommu_group_alloc()  ──or──  iommu_group_get()
      │                            │
      v                            v
  iommu_group_add_device()   (existing group returned)
      │
      v
  iommu_setup_default_domain()
      │
      v
  Device attached to group's default domain
      │
      v
  Group visible in sysfs: /sys/kernel/iommu_groups/<id>/
```

### 6.3 Group Sysfs Interface

Each IOMMU group is exposed in sysfs:

```
/sys/kernel/iommu_groups/
    ├── 0/
    │   ├── devices/
    │   │   ├── 0000:00:02.0 -> ../../../../devices/pci0000:00/...
    │   │   └── 0000:00:02.1 -> ../../../../devices/pci0000:00/...
    │   ├── type                    ← "DMA", "identity", etc.
    │   └── reserved_regions
    ├── 1/
    │   ├── devices/
    │   │   └── 0000:01:00.0
    │   ├── type
    │   └── reserved_regions
    ...
```

### 6.4 DMA Ownership

For device passthrough (VFIO), userspace must claim exclusive DMA ownership
of an IOMMU group. This prevents conflicts between kernel DMA API users and
userspace drivers:

```c
/* include/linux/iommu.h:1123 */
int iommu_group_claim_dma_owner(struct iommu_group *group, void *owner);
void iommu_group_release_dma_owner(struct iommu_group *group);
bool iommu_group_dma_owner_claimed(struct iommu_group *group);

int iommu_device_claim_dma_owner(struct device *dev, void *owner);
void iommu_device_release_dma_owner(struct device *dev);
```

---

## 7. DMA API Integration

### 7.1 How the DMA API Uses IOMMU

When an IOMMU is present and configured in translated mode, the kernel's DMA
API (`dma_map_single()`, `dma_map_sg()`, etc.) transparently uses the IOMMU
to establish mappings. The integration is handled by `drivers/iommu/dma-iommu.c`.

### 7.2 DMA Mapping Flow

```
  Driver calls dma_map_page(dev, page, ...)
      │
      v
  dma_map_page_attrs()             [kernel/dma/mapping.c]
      │
      v
  dev->dma_ops->map_page()         [resolved to iommu_dma_map_page]
      │
      v
  iommu_dma_map_page()             [drivers/iommu/dma-iommu.c]
      │
      ├── 1. Allocate IOVA from domain's IOVA allocator
      │
      ├── 2. iommu_map() ──> domain->ops->map_pages()
      │      Install page table entry: IOVA -> physical address
      │
      └── 3. Return IOVA as DMA address to driver
              (device uses this address for DMA)

  Driver calls dma_unmap_page(dev, dma_addr, ...)
      │
      v
  iommu_dma_unmap_page()
      │
      ├── 1. iommu_unmap() ──> domain->ops->unmap_pages()
      │      Remove page table entry
      │
      ├── 2. Flush IOTLB (or queue for lazy flush)
      │
      └── 3. Free IOVA back to allocator
```

### 7.3 Strict vs. Lazy TLB Invalidation

The IOMMU subsystem supports two TLB invalidation strategies:

- **Strict mode** (`IOMMU_DOMAIN_DMA`): The IOTLB is invalidated synchronously
  on every unmap. This is the safest mode -- a device can never access stale
  mappings -- but incurs higher overhead due to per-unmap TLB flushes.

- **Lazy mode** (`IOMMU_DOMAIN_DMA_FQ`): IOTLB invalidations are batched via
  a flush queue. Unmapped IOVAs are not reused until the flush completes,
  preventing stale access without per-operation flushes. This significantly
  improves performance for high-throughput I/O workloads.

The default is determined at boot time (`drivers/iommu/iommu.c:45`):

```c
static bool iommu_dma_strict __read_mostly =
    IS_ENABLED(CONFIG_IOMMU_DEFAULT_DMA_STRICT);
```

---

## 8. Page Table Management and IOTLB

### 8.1 Page Table Formats

Each IOMMU hardware implementation uses its own page table format, but they
share common concepts:

| Hardware | Page Table Format | Levels | Page Sizes |
|---|---|---|---|
| Intel VT-d | Extended/Second-Level | 3-5 | 4K, 2M, 1G |
| AMD-Vi | v1/v2 page tables | 4-6 | 4K, 2M, 1G |
| ARM SMMUv3 | AArch64 LPAE (stage-1/2) | 2-4 | 4K, 16K, 64K, 2M, 1G |

### 8.2 Supported Page Sizes

Each IOMMU driver advertises its supported page sizes via `pgsize_bitmap` in
`iommu_ops`. The core uses this to select optimal page sizes when mapping:

```c
/* include/linux/iommu.h:658 */
unsigned long pgsize_bitmap;    /* in struct iommu_ops */
```

### 8.3 IOTLB Management

IOMMUs cache address translations in an I/O TLB (IOTLB). After modifying page
tables, the driver must invalidate stale entries. The core provides:

- `flush_iotlb_all()` -- Invalidate the entire IOTLB for a domain
- `iotlb_sync_map()` -- Synchronize after mapping (ensure new entries visible)
- `iotlb_sync()` -- Synchronize after unmapping (flush stale entries)

The `iommu_iotlb_gather` structure accumulates a range of IOVAs to invalidate,
enabling a single batched flush rather than per-page invalidation.

### 8.4 Dirty Page Tracking

For live migration of VMs, the IOMMU can track which pages have been written
by devices. The `iommu_dirty_ops` interface provides this:

```c
/* include/linux/iommu.h:373 */
struct iommu_dirty_ops {
    int (*set_dirty_tracking)(struct iommu_domain *domain, bool enabled);
    int (*read_and_clear_dirty)(struct iommu_domain *domain,
                                unsigned long iova, size_t size,
                                unsigned long flags,
                                struct iommu_dirty_bitmap *dirty);
};
```

---

## 9. IOMMUFD Userspace Interface

### 9.1 Overview

IOMMUFD (`/dev/iommu`) is the modern userspace interface for managing IOMMU
objects. It replaces the older VFIO container/group interface with a more
flexible, object-based API. IOMMUFD gives userspace fine-grained control over
IO page tables and is the foundation for modern device passthrough (VFIO) and
user-space I/O frameworks.

The core implementation lives in `drivers/iommu/iommufd/main.c`.

### 9.2 IOMMUFD Object Model

IOMMUFD manages kernel objects via an xarray, with each object identified by
a numeric ID returned to userspace. Objects form a reference-counted graph:

```
┌──────────────────────────────────────────────────────────────┐
│                    IOMMUFD Object Graph                       │
│                                                              │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐    │
│   │  IOAS    │────>│   HWPT       │────>│  HWPT        │    │
│   │ (IO Addr │     │  (HW Page    │     │  (Nested)    │    │
│   │  Space)  │     │   Table)     │     │              │    │
│   └──────────┘     └──────────────┘     └──────────────┘    │
│        │                  │                                  │
│        v                  v                                  │
│   ┌──────────┐     ┌──────────────┐     ┌──────────────┐    │
│   │  DEVICE  │     │   VIOMMU     │────>│  VDEVICE     │    │
│   │          │     │  (Virtual    │     │              │    │
│   │          │     │   IOMMU)     │     │              │    │
│   └──────────┘     └──────────────┘     └──────────────┘    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 9.3 IOMMUFD ioctl Commands

The ioctl dispatch table is defined in `drivers/iommu/iommufd/main.c:334`:

| ioctl | Handler | Purpose |
|---|---|---|
| `IOMMU_DESTROY` | `iommufd_destroy` | Destroy an IOMMUFD object |
| `IOMMU_IOAS_ALLOC` | `iommufd_ioas_alloc_ioctl` | Allocate an IO Address Space |
| `IOMMU_IOAS_MAP` | `iommufd_ioas_map` | Map user memory into an IOAS |
| `IOMMU_IOAS_MAP_FILE` | `iommufd_ioas_map_file` | Map file-backed memory into IOAS |
| `IOMMU_IOAS_UNMAP` | `iommufd_ioas_unmap` | Unmap a region from an IOAS |
| `IOMMU_IOAS_COPY` | `iommufd_ioas_copy` | Copy mappings between IOASes |
| `IOMMU_IOAS_IOVA_RANGES` | `iommufd_ioas_iova_ranges` | Query allowed IOVA ranges |
| `IOMMU_IOAS_ALLOW_IOVAS` | `iommufd_ioas_allow_iovas` | Restrict IOVA allocation |
| `IOMMU_HWPT_ALLOC` | `iommufd_hwpt_alloc` | Allocate a HW page table |
| `IOMMU_HWPT_INVALIDATE` | `iommufd_hwpt_invalidate` | Invalidate HW page table caches |
| `IOMMU_HWPT_SET_DIRTY_TRACKING` | `iommufd_hwpt_set_dirty_tracking` | Enable/disable dirty tracking |
| `IOMMU_HWPT_GET_DIRTY_BITMAP` | `iommufd_hwpt_get_dirty_bitmap` | Read dirty page bitmap |
| `IOMMU_GET_HW_INFO` | `iommufd_get_hw_info` | Query IOMMU hardware capabilities |
| `IOMMU_VFIO_IOAS` | `iommufd_vfio_ioas` | VFIO compatibility interface |
| `IOMMU_VIOMMU_ALLOC` | `iommufd_viommu_alloc_ioctl` | Allocate a virtual IOMMU |
| `IOMMU_VDEVICE_ALLOC` | `iommufd_vdevice_alloc_ioctl` | Allocate a virtual device |
| `IOMMU_FAULT_QUEUE_ALLOC` | `iommufd_fault_alloc` | Allocate a fault queue |
| `IOMMU_OPTION` | `iommufd_option` | Set/get IOMMUFD options |

### 9.4 IOMMUFD VFIO Compatibility

IOMMUFD can provide `/dev/vfio/vfio` as a backward-compatible interface. When
`CONFIG_IOMMUFD_VFIO_CONTAINER` is enabled, IOMMUFD intercepts the legacy VFIO
container open and provides compatible semantics
(`drivers/iommu/iommufd/main.c:219`).

---

## 10. VFIO and Device Passthrough

### 10.1 Device Passthrough Flow

VFIO uses the IOMMU subsystem to safely assign physical devices to userspace
(typically virtual machines). The flow involves claiming DMA ownership,
creating an unmanaged domain, and establishing mappings controlled by
userspace:

```
  1. Userspace opens /dev/vfio/<group_id>
     │
  2. iommu_group_claim_dma_owner(group, owner)
     │  ─── Switches group from default kernel domain
     │      to blocked domain; kernel drivers detached
     │
  3. Allocate IOMMU_DOMAIN_UNMANAGED domain
     │
  4. iommu_attach_group(domain, group)
     │  ─── All devices in group now use new domain
     │
  5. Userspace maps guest memory:
     │  iommu_map(domain, iova, phys, size, prot)
     │  ─── Establishes guest-physical to host-physical mapping
     │
  6. Guest device issues DMA using IOVA
     │  ─── IOMMU translates to host physical address
     │
  7. On VM teardown:
     │  iommu_unmap(domain, iova, size)
     │  iommu_detach_group(domain, group)
     │  iommu_group_release_dma_owner(group)
     │  ─── Group reverts to default kernel domain
```

### 10.2 Nested Translation (VM passthrough)

For performance-sensitive VM passthrough, nested translation allows the guest
to manage its own stage-1 page tables while the host maintains stage-2:

```
  Guest IOVA ──> [Stage-1: Guest-managed] ──> Guest Physical Address
                                                      │
                                               [Stage-2: Host-managed]
                                                      │
                                                      v
                                              Host Physical Address
```

This is implemented via `IOMMU_DOMAIN_NESTED` domains allocated through
`ops->domain_alloc_nested()`, exposed to userspace through IOMMUFD's
`IOMMU_HWPT_ALLOC` with appropriate flags.

---

## 11. Shared Virtual Addressing (SVA)

### 11.1 Overview

SVA (also called SVM -- Shared Virtual Memory) allows devices to use the same
virtual addresses as a CPU process. The IOMMU shares the process's CPU page
tables (or a copy) so the device can directly access process memory using
userspace pointers. This is enabled through PASIDs (Process Address Space IDs),
which tag device transactions with a process context.

### 11.2 SVA Architecture

```
     Process (mm_struct)
     ┌───────────────────────────────────┐
     │  Virtual Address Space            │
     │  0x7fff00000000 ─── stack         │
     │  0x7f0000000000 ─── shared libs   │
     │  0x400000       ─── code/data     │
     │        │                          │
     │        │ CPU MMU uses these       │
     │        │ page tables              │
     │        │                          │
     │        │    ┌──────────────┐      │
     │        └───>│ Page Tables  │<─────│──── IOMMU also walks
     │             └──────────────┘      │     these tables (via
     │                                   │     PASID tagging)
     └───────────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         v                         v
      CPU access              Device DMA
    (same addresses)        (same addresses)
```

### 11.3 SVA API

```c
/* include/linux/iommu.h:1588 */
struct iommu_sva *iommu_sva_bind_device(struct device *dev,
                                        struct mm_struct *mm);
void iommu_sva_unbind_device(struct iommu_sva *handle);
u32 iommu_sva_get_pasid(struct iommu_sva *handle);
```

SVA requires both the device and the IOMMU to support:
- **PASID** (Process Address Space ID): Tags DMA transactions with a process
  context
- **I/O Page Faults (IOPF)**: Handles page faults from device DMA, similar
  to CPU demand paging
- **ATS** (Address Translation Services): Optional PCI feature allowing
  devices to cache translations

Device features are enabled via:

```c
/* include/linux/iommu.h:1117 */
int iommu_dev_enable_feature(struct device *dev, enum iommu_dev_features f);
int iommu_dev_disable_feature(struct device *dev, enum iommu_dev_features f);
```

Where features include `IOMMU_DEV_FEAT_SVA` and `IOMMU_DEV_FEAT_IOPF`.

---

## 12. Hardware Implementations

### 12.1 Intel VT-d (Virtualization Technology for Directed I/O)

Intel VT-d is implemented in `drivers/iommu/intel/`. It provides:

- **DMA Remapping (DMAR)**: Translates device DMA addresses via multi-level
  page tables. DMAR hardware units are described by the ACPI DMAR table.
- **Interrupt Remapping**: Redirects and validates device interrupts.
- **Scalable Mode**: Modern Intel IOMMUs support scalable mode with a
  two-level structure: a PASID table per device pointing to per-PASID page
  tables. This enables SVA and nested translation.
- **Page Sizes**: 4KB, 2MB, 1GB (when supported by hardware).
- **First-Level / Second-Level**: First-level translation uses CPU-compatible
  page tables (for SVA), second-level uses IOMMU-specific format (for DMA
  remapping and VM passthrough).

Key files:
- `drivers/iommu/intel/iommu.c` -- Core VT-d driver
- `drivers/iommu/intel/iommu.h` -- VT-d data structures
- `drivers/iommu/intel/pasid.c` -- PASID table management
- `drivers/iommu/intel/svm.c` -- Shared Virtual Memory support

### 12.2 AMD-Vi (AMD I/O Virtualization)

AMD-Vi is implemented in `drivers/iommu/amd/`. It provides:

- **Device Table**: Central per-device configuration table indexed by DeviceID,
  containing domain assignment, interrupt remapping, and PASID configuration.
- **I/O Page Tables**: v1 (legacy) and v2 (host CPU-compatible) formats.
- **Guest Virtual APIC (GA) Log**: For interrupt virtualization.
- **PPR (Peripheral Page Request) Log**: For I/O page fault handling.
- **Page Sizes**: 4KB, 2MB, 1GB.

Key files:
- `drivers/iommu/amd/iommu.c` -- Core AMD-Vi driver
- `drivers/iommu/amd/init.c` -- Hardware initialization from IVRS ACPI table
- `drivers/iommu/amd/io_pgtable.c` -- Page table management
- `drivers/iommu/amd/iommu_v2.c` -- v2 page table / PASID support

### 12.3 ARM SMMU (System MMU)

ARM provides two generations of SMMU:

**SMMUv2** (`drivers/iommu/arm/arm-smmu/`):
- Uses Stream IDs to identify devices
- Stage-1 (VA to IPA) and Stage-2 (IPA to PA) translation
- LPAE (Large Physical Address Extension) page table format
- Context banks for domain isolation

**SMMUv3** (`drivers/iommu/arm/arm-smmu-v3/`):
- Command queue and event queue based interface (replaces MMIO)
- Native PASID (SubstreamID) support
- Stall model for I/O page faults
- PCIe ATS support
- Page sizes: 4KB, 16KB, 64KB, 2MB, 1GB (configurable)
- Broadcast TLB maintenance via command queue

Key files:
- `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` -- SMMUv3 driver
- `drivers/iommu/arm/arm-smmu/arm-smmu.c` -- SMMUv2 driver
- `drivers/iommu/io-pgtable-arm.c` -- Shared ARM page table code

### 12.4 Hardware Comparison

```
┌────────────┬────────────────┬────────────────┬─────────────────────┐
│ Feature    │ Intel VT-d     │ AMD-Vi         │ ARM SMMUv3          │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ Config     │ ACPI DMAR      │ ACPI IVRS      │ Device Tree / ACPI  │
│ Source     │ table          │ table          │ IORT table          │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ Device ID  │ PCI BDF        │ PCI BDF /      │ Stream ID           │
│            │ (source-id)    │ DeviceID       │ (from IORT/DT)      │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ Page Table │ Second-level   │ v1 (4-level)   │ LPAE (ARM64-style)  │
│ Format     │ (VT-d format)  │ v2 (CPU fmt)   │ Stage-1 / Stage-2   │
│            │ First-level    │                │                     │
│            │ (CPU format)   │                │                     │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ PASID      │ Yes (scalable  │ Yes (PPR,      │ Yes (SubstreamID)   │
│ Support    │ mode)          │ PASID table)   │                     │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ Nested     │ Yes (first-    │ Yes (v2 on v1) │ Yes (stage-1 on     │
│ Translation│ level on       │                │ stage-2)            │
│            │ second-level)  │                │                     │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ I/O Page   │ PRI (PCI PRQ)  │ PPR Log        │ Stall / PRI         │
│ Faults     │                │                │                     │
├────────────┼────────────────┼────────────────┼─────────────────────┤
│ TLB Inv.   │ MMIO register  │ Command buffer │ Command queue       │
│ Mechanism  │ based          │ / MMIO         │                     │
└────────────┴────────────────┴────────────────┴─────────────────────┘
```

---

## 13. Fault Handling and I/O Page Faults

### 13.1 Fault Types

The IOMMU subsystem handles two categories of faults:

- **Unrecoverable faults**: The device accessed an unmapped or disallowed IOVA.
  These are logged and may result in device reset. Common in DMA remapping mode
  when a device driver has a bug.

- **Recoverable faults (I/O Page Faults / IOPF)**: The device accessed a valid
  virtual address that is not currently present in the IOMMU page tables. The
  IOMMU stalls the transaction, the kernel handles the fault (e.g., by faulting
  in pages from the process's address space), and the transaction is retried.
  This is critical for SVA.

### 13.2 Fault Data Structures

```c
/* include/linux/iommu.h:54 */
enum iommu_fault_type {
    IOMMU_FAULT_PAGE_REQ = 1,   /* Recoverable page request fault */
};

/* include/linux/iommu.h:71 */
struct iommu_fault_page_request {
    u32 flags;
    u32 pasid;
    u32 grpid;        /* Page Request Group Index */
    u32 perm;         /* Requested permissions */
    u64 addr;         /* Faulting address */
    u64 private_data[2];
};

/* include/linux/iommu.h:88 */
struct iommu_fault {
    u32 type;
    struct iommu_fault_page_request prm;
};
```

### 13.3 IOPF Queue and Handling

The I/O Page Fault (IOPF) subsystem processes recoverable faults through a
workqueue. Faults are grouped by Page Request Group ID (grpid) and dispatched
to the appropriate handler:

```c
/* include/linux/iommu.h:147 */
struct iopf_queue {
    struct workqueue_struct *wq;
    struct list_head devices;
    struct mutex lock;
};
```

### 13.4 Page Response

After handling a fault, the kernel sends a response back to the device:

```c
/* include/linux/iommu.h:102 */
enum iommu_page_response_code {
    IOMMU_PAGE_RESP_SUCCESS = 0,   /* Fault handled, retry access */
    IOMMU_PAGE_RESP_INVALID,       /* Cannot handle, don't retry  */
    IOMMU_PAGE_RESP_FAILURE,       /* General error, drop faults  */
};

/* include/linux/iommu.h:114 */
struct iommu_page_response {
    u32 pasid;
    u32 grpid;
    u32 code;
};
```

---

## 14. Initialization and Boot Flow

### 14.1 IOMMU Subsystem Initialization

The IOMMU subsystem initializes early during boot via `subsys_initcall`:

```
  subsys_initcall(iommu_subsys_init)        [drivers/iommu/iommu.c:196]
      │
      ├── Determine default domain type
      │   (passthrough vs. translated, strict vs. lazy)
      │
      ├── If memory encryption active:
      │   Force translated mode (disable passthrough)
      │
      ├── Create iommu_group_kset for sysfs
      │
      └── Register bus notifiers for:
          ├── platform_bus_type
          ├── pci_bus_type
          ├── amba_bustype
          ├── fsl_mc_bus_type
          ├── host1x_context_device_bus_type
          └── cdx_bus_type
```

### 14.2 IOMMU Hardware Registration

Each IOMMU driver registers its hardware instances during its own init:

```
  IOMMU driver init (e.g., intel_iommu_init)
      │
      v
  iommu_device_register(iommu, ops, dev)    [drivers/iommu/iommu.c:248]
      │
      ├── Add to global iommu_device_list
      │
      └── bus_iommu_probe(bus)
          │
          └── For each device on bus:
              │
              ├── ops->probe_device(dev)
              │   ── Identify IOMMU instance for this device
              │
              ├── ops->device_group(dev)
              │   ── Assign device to an IOMMU group
              │
              ├── iommu_setup_default_domain(group)
              │   ── Allocate and attach default domain
              │
              └── ops->probe_finalize(dev)
                  ── Complete device setup (e.g., enable DMA ops)
```

### 14.3 Per-Device IOMMU State

Each device maintains IOMMU state through `dev->iommu` (a `struct dev_iommu`):

```
  struct device
      │
      ├── iommu_group    ← Pointer to the device's IOMMU group
      │
      └── iommu (struct dev_iommu)
          ├── iommu_dev  ← Which IOMMU hardware instance handles this device
          ├── priv       ← Driver-private data (stream table entry, etc.)
          └── fwspec     ← Firmware specification (stream IDs, etc.)
```

---

## 15. Key Source Files

| File | Purpose |
|---|---|
| `include/linux/iommu.h` | Core IOMMU API header: domains, groups, ops |
| `drivers/iommu/iommu.c` | Core IOMMU framework implementation |
| `drivers/iommu/dma-iommu.c` | DMA API integration with IOMMU |
| `drivers/iommu/iova.c` | IOVA (I/O Virtual Address) allocator |
| `drivers/iommu/io-pgtable.c` | Generic I/O page table framework |
| `drivers/iommu/io-pgtable-arm.c` | ARM LPAE page table implementation |
| `drivers/iommu/iommufd/main.c` | IOMMUFD userspace interface core |
| `drivers/iommu/iommufd/io_pagetable.c` | IOMMUFD I/O page table management |
| `drivers/iommu/iommufd/ioas.c` | IOMMUFD IO Address Space objects |
| `drivers/iommu/iommufd/hw_pagetable.c` | IOMMUFD HW page table objects |
| `drivers/iommu/iommufd/viommu.c` | IOMMUFD virtual IOMMU support |
| `drivers/iommu/intel/iommu.c` | Intel VT-d core driver |
| `drivers/iommu/intel/pasid.c` | Intel VT-d PASID table management |
| `drivers/iommu/intel/svm.c` | Intel VT-d shared virtual memory |
| `drivers/iommu/amd/iommu.c` | AMD-Vi core driver |
| `drivers/iommu/amd/init.c` | AMD-Vi hardware initialization |
| `drivers/iommu/amd/io_pgtable.c` | AMD-Vi page table management |
| `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` | ARM SMMUv3 driver |
| `drivers/iommu/arm/arm-smmu/arm-smmu.c` | ARM SMMUv2 driver |
| `include/uapi/linux/iommufd.h` | IOMMUFD userspace API definitions |
| `include/linux/iommufd.h` | IOMMUFD kernel-internal interface |
