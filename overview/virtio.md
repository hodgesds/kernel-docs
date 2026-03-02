# Linux Kernel Virtio Subsystem Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Virtio Architecture](#virtio-architecture)
3. [Virtqueues and Vring Layout](#virtqueues-and-vring-layout)
4. [Split Virtqueue Ring Format](#split-virtqueue-ring-format)
5. [Packed Virtqueue Ring Format](#packed-virtqueue-ring-format)
6. [Virtio Device and Driver](#virtio-device-and-driver)
7. [Feature Negotiation](#feature-negotiation)
8. [Virtqueue Operations API](#virtqueue-operations-api)
9. [Transport Backends](#transport-backends)
10. [Notification Suppression](#notification-suppression)
11. [DMA Considerations](#dma-considerations)
12. [Vhost Kernel Acceleration](#vhost-kernel-acceleration)
13. [Vringh (Host-Side Vring Access)](#vringh-host-side-vring-access)
14. [Virtio Device Types](#virtio-device-types)
15. [Source Files](#source-files)

---

## Introduction

Virtio is a standardized interface for virtual I/O devices in paravirtualized
hypervisors. It provides a common framework for guest operating systems to
communicate with virtual devices implemented by a hypervisor or host kernel,
achieving near-native performance by avoiding the overhead of full hardware
emulation.

Key design principles:

- **Guest/host split**: The guest runs virtio drivers; the host provides
  virtio device backends. They share a common ABI for data exchange.
- **Transport independence**: The virtio protocol sits above transport layers
  (PCI, MMIO, Channel I/O) that handle device discovery and configuration.
- **Efficient data transfer**: Shared-memory virtqueues (vrings) enable
  zero-copy data paths with minimal exit/notification overhead.
- **Feature negotiation**: Both sides advertise capabilities and negotiate a
  common subset, enabling forward and backward compatibility.

```
+---------------------------------------------------------------+
|                    Virtio Architecture                        |
+---------------------------------------------------------------+
|                                                               |
|   Guest (Linux Kernel)            Host (QEMU / KVM / vhost)   |
|   +-------------------------+    +-------------------------+  |
|   |  virtio-net driver      |    |  virtio-net backend     |  |
|   |  virtio-blk driver      |    |  virtio-blk backend     |  |
|   |  virtio-scsi driver     |    |  virtio-scsi backend    |  |
|   +----------+--------------+    +-----------+-------------+  |
|              |                               |                |
|   +----------+--------------+    +-----------+-------------+  |
|   |    Virtqueue API        |    |   Vring access logic    |  |
|   |    (virtio_ring.c)      |    |   (QEMU / vhost.c)      |  |
|   +----------+--------------+    +-----------+-------------+  |
|              |                               |                |
|   +----------+-------------------------------+-------------+  |
|   |              Shared Memory (Vrings)                    |  |
|   |  +--------+   +--------+   +--------+                  |  |
|   |  | desc[] |   | avail  |   | used   |                  |  |
|   |  +--------+   +--------+   +--------+                  |  |
|   +--------------------------------------------------------+  |
|              |                               |                |
|   +----------+--------------+    +-----------+-------------+  |
|   | Transport (PCI/MMIO/CCW)|    | Transport (PCI/MMIO/CCW)|  |
|   +-------------------------+    +-------------------------+  |
|                                                               |
+---------------------------------------------------------------+
```

---

## Virtio Architecture

The virtio subsystem is organized into several layers in the Linux kernel:

```
+---------------------------------------------------------------+
|                    Layer Diagram                              |
+---------------------------------------------------------------+
|                                                               |
|   Device Drivers (consumers)                                  |
|   +---------------------------------------------------+       |
|   | virtio-net  | virtio-blk | virtio-scsi | virtio-* |       |
|   +------+------+------+-----+------+------+----+-----+       |
|          |             |             |           |            |
|   +------+-------------+-------------+-----------+------+     |
|   |              struct virtio_driver                   |     |
|   |              (include/linux/virtio.h:218)           |     |
|   +------+----------------------------------------------+     |
|          |                                                    |
|   +------+----------------------------------------------+     |
|   |         Virtio Core Bus (drivers/virtio/virtio.c)   |     |
|   |         virtio_bus, register_virtio_device(),       |     |
|   |         virtio_dev_probe(), feature negotiation     |     |
|   +------+----------------------------------------------+     |
|          |                                                    |
|   +------+----------------------------------------------+     |
|   |         Virtqueue / Vring (drivers/virtio/virtio_ring.c)  |
|   |         struct vring_virtqueue, virtqueue_add_*(),  |     |
|   |         virtqueue_kick(), virtqueue_get_buf()       |     |
|   +------+----------------------------------------------+     |
|          |                                                    |
|   +------+----------------------------------------------+     |
|   | Transport Layer                                     |     |
|   | +-------------+ +-------------+ +-----------+       |     |
|   | | virtio-pci  | | virtio-mmio | | virtio-ccw|       |     |
|   | | (modern +   | | (platform   | | (s390     |       |     |
|   | |  legacy)    | |  device)    | |  channel) |       |     |
|   | +-------------+ +-------------+ +-----------+       |     |
|   +-----------------------------------------------------+     |
|                                                               |
+---------------------------------------------------------------+
```

### Status Bits (Device Initialization Sequence)

Defined in `include/uapi/linux/virtio_config.h`:

```c
/* include/uapi/linux/virtio_config.h */
#define VIRTIO_CONFIG_S_ACKNOWLEDGE     1     /* Guest OS has found the device */
#define VIRTIO_CONFIG_S_DRIVER          2     /* Guest OS knows how to drive it */
#define VIRTIO_CONFIG_S_DRIVER_OK       4     /* Driver is set up and ready */
#define VIRTIO_CONFIG_S_FEATURES_OK     8     /* Feature negotiation complete */
#define VIRTIO_CONFIG_S_NEEDS_RESET     0x40  /* Device experienced an error */
#define VIRTIO_CONFIG_S_FAILED          0x80  /* Guest has given up on device */
```

The initialization sequence is:

```
1. Reset device                          (status = 0)
2. Set ACKNOWLEDGE                       (status |= 1)
3. Set DRIVER                            (status |= 2)
4. Read device features
5. Negotiate features (driver & device)
6. Set FEATURES_OK                       (status |= 8)
7. Re-read status to confirm FEATURES_OK
8. Device-specific setup (find_vqs, etc.)
9. Set DRIVER_OK                         (status |= 4)
```

This sequence is implemented in `virtio_dev_probe()` at
`drivers/virtio/virtio.c:270`.

---

## Virtqueues and Vring Layout

A virtqueue is the mechanism for bulk data transport between guest and host.
Each virtio device has one or more virtqueues. The guest-visible API is the
`struct virtqueue`, while the underlying shared-memory layout is the vring.

### struct virtqueue (include/linux/virtio.h:31)

The virtqueue is the driver-facing handle for a single queue. Drivers interact
with it through the virtqueue_add_*/kick/get_buf API. The `priv` field points
to the transport-specific implementation (the `vring_virtqueue`).

```c
/* include/linux/virtio.h:31 */
struct virtqueue {
    struct list_head list;
    void (*callback)(struct virtqueue *vq);
    const char *name;
    struct virtio_device *vdev;
    unsigned int index;
    unsigned int num_free;
    unsigned int num_max;
    bool reset;
    void *priv;
};
```

### struct vring_virtqueue (drivers/virtio/virtio_ring.c:162)

The internal implementation structure that wraps `struct virtqueue` and contains
the actual vring state. This is private to the virtio_ring module and never
exposed to drivers.

```c
/* drivers/virtio/virtio_ring.c:162 */
struct vring_virtqueue {
    struct virtqueue vq;

    bool packed_ring;           /* Split or packed format? */
    bool use_dma_api;           /* Use DMA API for mappings? */
    bool weak_barriers;         /* Can we use lighter barriers? */
    bool broken;                /* Device misbehaved? */
    bool indirect;              /* Host supports indirect descriptors */
    bool event;                 /* Host publishes avail event idx */

    unsigned int free_head;     /* Head of free descriptor list */
    unsigned int num_added;     /* Descriptors added since last sync */
    u16 last_used_idx;          /* Last used index we've seen */
    bool event_triggered;       /* Hint for event idx optimization */

    union {
        struct vring_virtqueue_split split;
        struct vring_virtqueue_packed packed;
    };

    bool (*notify)(struct virtqueue *vq);  /* Transport notification */
    bool we_own_ring;
    struct device *dma_dev;
};
```

---

## Split Virtqueue Ring Format

The split vring is the original (legacy and v1.0) format. It consists of three
physically separate areas in shared memory: a descriptor table, an available
ring, and a used ring.

### Vring Memory Layout (Split)

```
+---------------------------------------------------------------+
|                   Split Vring Layout                          |
+---------------------------------------------------------------+
|                                                               |
|  Descriptor Table (16 bytes each)                             |
|  +--------+--------+--------+--------+-----+--------+         |
|  | desc 0 | desc 1 | desc 2 | desc 3 | ... | desc N |         |
|  +--------+--------+--------+--------+-----+--------+         |
|                                                               |
|  Available Ring (driver -> device)                            |
|  +-------+-------+----------------------------+----------+    |
|  | flags | idx   | ring[0] ring[1] ... ring[N] | used_evt |   |
|  +-------+-------+----------------------------+----------+    |
|                                                               |
|  Padding to VRING_USED_ALIGN_SIZE (4 bytes)                   |
|                                                               |
|  Used Ring (device -> driver)                                 |
|  +-------+-------+----------------------------+----------+    |
|  | flags | idx   | ring[0] ring[1] ... ring[N] | avail_evt|   |
|  +-------+-------+----------------------------+----------+    |
|                                                               |
+---------------------------------------------------------------+
```

### struct vring_desc (include/uapi/linux/virtio_ring.h:107)

Each descriptor points to a buffer in guest memory. Descriptors can be chained
via the `next` field to form scatter-gather lists.

```c
/* include/uapi/linux/virtio_ring.h:107 */
struct vring_desc {
    __virtio64 addr;     /* Guest-physical buffer address */
    __virtio32 len;      /* Buffer length */
    __virtio16 flags;    /* VRING_DESC_F_NEXT, _WRITE, _INDIRECT */
    __virtio16 next;     /* Next descriptor in chain (if F_NEXT set) */
};
```

Descriptor flags (`include/uapi/linux/virtio_ring.h:41`):

```c
#define VRING_DESC_F_NEXT       1   /* Buffer continues via next field */
#define VRING_DESC_F_WRITE      2   /* Buffer is write-only (device writes) */
#define VRING_DESC_F_INDIRECT   4   /* Buffer contains indirect desc table */
```

### Available Ring (include/uapi/linux/virtio_ring.h:114)

The driver writes descriptor chain heads here to offer buffers to the device.

```c
/* include/uapi/linux/virtio_ring.h:114 */
struct vring_avail {
    __virtio16 flags;    /* VRING_AVAIL_F_NO_INTERRUPT */
    __virtio16 idx;      /* Where the driver would put the next desc entry */
    __virtio16 ring[];   /* Descriptor chain head indices */
};
```

### Used Ring (include/uapi/linux/virtio_ring.h:121)

The device writes consumed descriptor chain heads here to return buffers.

```c
/* include/uapi/linux/virtio_ring.h:121 */
struct vring_used_elem {
    __virtio32 id;       /* Index of start of used descriptor chain */
    __virtio32 len;      /* Bytes written to the descriptor chain */
};

/* include/uapi/linux/virtio_ring.h:131 */
struct vring_used {
    __virtio16 flags;    /* VRING_USED_F_NO_NOTIFY */
    __virtio16 idx;      /* Where the device would put the next used entry */
    vring_used_elem_t ring[];
};
```

### Descriptor Chain Example

```
Driver wants to send a request header + receive a response:

  avail ring              Descriptor Table
  +-------+             +---+---+---+---+---+---+
  | idx=3 |             | 0 | 1 | 2 | 3 | 4 | 5 |
  +-------+             +---+---+---+---+---+---+
  | ring: |                   |           |
  | [0]=1 |         desc[1]   |    desc[3]|
  | [1]=3 |  +--------+------+    +------+--------+
  | [2]=_ |  | addr   | req_hdr   | addr | data   |
  +-------+  | len    | 16        | len  | 4096   |
             | flags  | NEXT      | flags| WRITE  |
             | next   | 4         | next | -      |
             +--------+           +------+--------+
                  |
            desc[4]
             +--------+
             | addr   | status_byte
             | len    | 1
             | flags  | WRITE
             | next   | -
             +--------+

Chain 1: desc[1] -> desc[4]  (out: req_hdr, in: status)
Chain 2: desc[3]             (in: data buffer)
```

### struct vring (include/uapi/linux/virtio_ring.h:158)

Convenience structure that ties the three areas together:

```c
/* include/uapi/linux/virtio_ring.h:158 */
struct vring {
    unsigned int num;         /* Number of descriptors (power of 2) */
    vring_desc_t *desc;       /* Descriptor table */
    vring_avail_t *avail;     /* Available ring */
    vring_used_t *used;       /* Used ring */
};
```

### Split Ring Internal State (drivers/virtio/virtio_ring.c:97)

```c
/* drivers/virtio/virtio_ring.c:97 */
struct vring_virtqueue_split {
    struct vring vring;                      /* Actual memory layout */
    u16 avail_flags_shadow;                  /* Cached avail->flags */
    u16 avail_idx_shadow;                    /* Cached avail->idx */
    struct vring_desc_state_split *desc_state;   /* Per-descriptor state */
    struct vring_desc_extra *desc_extra;     /* Extra per-desc metadata */
    dma_addr_t queue_dma_addr;
    size_t queue_size_in_bytes;
    u32 vring_align;
    bool may_reduce_num;
};
```

---

## Packed Virtqueue Ring Format

The packed virtqueue (negotiated via `VIRTIO_F_RING_PACKED`, bit 34) replaces
the three separate areas of the split format with a single unified descriptor
ring. This improves cache locality and enables more efficient hardware
implementations.

### Packed Descriptor (include/uapi/linux/virtio_ring.h:239)

```c
/* include/uapi/linux/virtio_ring.h:239 */
struct vring_packed_desc {
    __le64 addr;       /* Buffer Address */
    __le32 len;        /* Buffer Length */
    __le16 id;         /* Buffer ID */
    __le16 flags;      /* Descriptor flags (incl. avail/used bits) */
};
```

### Packed Event Suppression (include/uapi/linux/virtio_ring.h:232)

```c
/* include/uapi/linux/virtio_ring.h:232 */
struct vring_packed_desc_event {
    __le16 off_wrap;   /* Event offset + wrap counter */
    __le16 flags;      /* ENABLE / DISABLE / DESC */
};
```

### Packed Ring vs Split Ring

```
+---------------------------------------------------------------+
|                Split vs Packed Comparison                     |
+---------------------------------------------------------------+
|                                                               |
|  Split Ring:                                                  |
|  - Three separate memory areas (desc, avail, used)            |
|  - Avail/used indices are free-running 16-bit counters        |
|  - Cache-unfriendly: driver writes avail, reads used          |
|  - Descriptors referenced by index in avail/used rings        |
|                                                               |
|  Packed Ring:                                                 |
|  - Single descriptor ring for everything                      |
|  - Avail/used state encoded in per-descriptor flag bits       |
|  - Wrap counter distinguishes old from new entries            |
|  - Better cache locality: all data in one contiguous area     |
|  - Two separate event suppression structures for              |
|    driver and device                                          |
|  - Descriptors consumed in-place (no separate used ring)      |
|                                                               |
+---------------------------------------------------------------+
```

The packed ring uses two flag bits per descriptor to track availability:

```c
/* include/uapi/linux/virtio_ring.h:51 */
#define VRING_PACKED_DESC_F_AVAIL   7    /* Bit position for avail flag */
#define VRING_PACKED_DESC_F_USED    15   /* Bit position for used flag */
```

The driver toggles the avail flag when making a descriptor available. The
device toggles the used flag when consuming it. A wrap counter inverts the
polarity each time the ring wraps around.

### Packed Ring Internal State (drivers/virtio/virtio_ring.c:126)

```c
/* drivers/virtio/virtio_ring.c:126 */
struct vring_virtqueue_packed {
    struct {
        unsigned int num;
        struct vring_packed_desc *desc;
        struct vring_packed_desc_event *driver;   /* Driver event suppression */
        struct vring_packed_desc_event *device;   /* Device event suppression */
    } vring;

    bool avail_wrap_counter;       /* Current wrap counter state */
    u16 avail_used_flags;          /* Pre-computed avail/used flag values */
    u16 next_avail_idx;            /* Next descriptor index to use */
    u16 event_flags_shadow;        /* Cached driver event flags */

    struct vring_desc_state_packed *desc_state;
    struct vring_desc_extra *desc_extra;

    dma_addr_t ring_dma_addr;
    dma_addr_t driver_event_dma_addr;
    dma_addr_t device_event_dma_addr;
    size_t ring_size_in_bytes;
    size_t event_size_in_bytes;
};
```

---

## Virtio Device and Driver

### struct virtio_device (include/linux/virtio.h:149)

Represents a single virtio device instance. Created by the transport layer
(PCI, MMIO, etc.) and registered on the virtio bus.

```c
/* include/linux/virtio.h:149 */
struct virtio_device {
    int index;                              /* Unique device index */
    bool failed;                            /* Saved FAILED status for restore */
    bool config_core_enabled;               /* Config change reporting by core */
    bool config_driver_disabled;            /* Config change reporting by driver */
    bool config_change_pending;             /* Pending config change event */
    spinlock_t config_lock;                 /* Protects config change state */
    spinlock_t vqs_list_lock;               /* Protects vqs list */
    struct device dev;                      /* Embedded struct device */
    struct virtio_device_id id;             /* Device/vendor ID */
    const struct virtio_config_ops *config; /* Transport operations */
    const struct vringh_config_ops *vringh_config;  /* Host vring ops */
    struct list_head vqs;                   /* List of virtqueues */
    u64 features;                           /* Negotiated feature bits */
    void *priv;                             /* Driver private data */
};

#define dev_to_virtio(_dev) container_of_const(_dev, struct virtio_device, dev)
```

### struct virtio_device_id (include/linux/mod_devicetable.h:449)

Used for matching devices to drivers on the virtio bus:

```c
/* include/linux/mod_devicetable.h:449 */
struct virtio_device_id {
    __u32 device;
    __u32 vendor;
};
#define VIRTIO_DEV_ANY_ID   0xffffffff
```

### struct virtio_driver (include/linux/virtio.h:218)

The driver counterpart, registered by device drivers (virtio-net, virtio-blk,
etc.) onto the virtio bus.

```c
/* include/linux/virtio.h:218 */
struct virtio_driver {
    struct device_driver driver;
    const struct virtio_device_id *id_table;
    const unsigned int *feature_table;         /* Features this driver supports */
    unsigned int feature_table_size;
    const unsigned int *feature_table_legacy;  /* Legacy-only features */
    unsigned int feature_table_size_legacy;
    int (*validate)(struct virtio_device *dev); /* Validate features/config */
    int (*probe)(struct virtio_device *dev);    /* Device found */
    void (*scan)(struct virtio_device *dev);    /* Post-probe scan (virtio-scsi) */
    void (*remove)(struct virtio_device *dev);  /* Device removed */
    void (*config_changed)(struct virtio_device *dev);  /* Config space changed */
    int (*freeze)(struct virtio_device *dev);   /* Suspend/hibernate */
    int (*restore)(struct virtio_device *dev);  /* Resume */
};

#define drv_to_virtio(__drv) container_of_const(__drv, struct virtio_driver, driver)
```

### struct virtio_config_ops (include/linux/virtio_config.h:108)

The transport-level operations that each transport backend must implement.
This is the vtable that connects the generic virtio core to transport-specific
code (PCI, MMIO, etc.).

```c
/* include/linux/virtio_config.h:108 */
struct virtio_config_ops {
    void (*get)(struct virtio_device *vdev, unsigned offset,
                void *buf, unsigned len);
    void (*set)(struct virtio_device *vdev, unsigned offset,
                const void *buf, unsigned len);
    u32 (*generation)(struct virtio_device *vdev);
    u8 (*get_status)(struct virtio_device *vdev);
    void (*set_status)(struct virtio_device *vdev, u8 status);
    void (*reset)(struct virtio_device *vdev);
    int (*find_vqs)(struct virtio_device *vdev, unsigned int nvqs,
                    struct virtqueue *vqs[],
                    struct virtqueue_info vqs_info[],
                    struct irq_affinity *desc);
    void (*del_vqs)(struct virtio_device *);
    void (*synchronize_cbs)(struct virtio_device *);
    u64 (*get_features)(struct virtio_device *vdev);
    int (*finalize_features)(struct virtio_device *vdev);
    const char *(*bus_name)(struct virtio_device *vdev);
    int (*set_vq_affinity)(struct virtqueue *vq,
                           const struct cpumask *cpu_mask);
    const struct cpumask *(*get_vq_affinity)(struct virtio_device *vdev,
                                             int index);
    bool (*get_shm_region)(struct virtio_device *vdev,
                           struct virtio_shm_region *region, u8 id);
    int (*disable_vq_and_reset)(struct virtqueue *vq);
    int (*enable_vq_after_reset)(struct virtqueue *vq);
};
```

### The Virtio Bus (drivers/virtio/virtio.c:380)

Virtio registers its own bus type. The bus `match` function iterates over the
driver's `id_table` and compares device/vendor IDs. The bus `probe` function
drives the full initialization sequence including feature negotiation.

```c
/* drivers/virtio/virtio.c:380 */
static const struct bus_type virtio_bus = {
    .name  = "virtio",
    .match = virtio_dev_match,
    .dev_groups = virtio_dev_groups,
    .uevent = virtio_uevent,
    .probe = virtio_dev_probe,
    .remove = virtio_dev_remove,
};
```

The bus is registered at boot via `core_initcall(virtio_init)` at
`drivers/virtio/virtio.c:618`.

---

## Feature Negotiation

Feature negotiation is central to virtio's extensibility. Both device and
driver advertise feature bits; only the intersection is activated.

### Feature Negotiation Flow

```
+---------------------------------------------------------------+
|                Feature Negotiation Flow                       |
+---------------------------------------------------------------+
|                                                               |
|  1. Device publishes feature bits                             |
|     device_features = dev->config->get_features(dev)          |
|     (drivers/virtio/virtio.c:283)                             |
|                                                               |
|  2. Driver computes its features from feature_table[]         |
|     driver_features = OR of all bits in drv->feature_table    |
|     (drivers/virtio/virtio.c:287)                             |
|                                                               |
|  3. Intersection: dev->features = driver_features &           |
|                                   device_features             |
|     (drivers/virtio/virtio.c:306)                             |
|                                                               |
|  4. Transport features (bits 28-41) always preserved          |
|     (drivers/virtio/virtio.c:314)                             |
|                                                               |
|  5. Finalize: dev->config->finalize_features(dev)             |
|     Writes negotiated bits to device                          |
|     (drivers/virtio/virtio.c:318)                             |
|                                                               |
|  6. Optional validate: drv->validate(dev)                     |
|     Driver can reject or modify features                      |
|     (drivers/virtio/virtio.c:322)                             |
|                                                               |
|  7. Set FEATURES_OK, re-read to confirm device accepted       |
|     virtio_features_ok() at drivers/virtio/virtio.c:204       |
|                                                               |
+---------------------------------------------------------------+
```

### Transport Feature Bits (include/uapi/linux/virtio_config.h)

Feature bits 28-41 are reserved for transport-level features:

```c
/* include/uapi/linux/virtio_config.h */
#define VIRTIO_TRANSPORT_F_START        28
#define VIRTIO_TRANSPORT_F_END          42

/* Transport features */
#define VIRTIO_RING_F_INDIRECT_DESC     28  /* Indirect descriptor support */
#define VIRTIO_RING_F_EVENT_IDX         29  /* Event index for notification suppression */
#define VIRTIO_F_VERSION_1              32  /* Virtio 1.0 compliant device */
#define VIRTIO_F_ACCESS_PLATFORM        33  /* Device respects platform DMA/IOMMU */
#define VIRTIO_F_RING_PACKED            34  /* Packed virtqueue layout */
#define VIRTIO_F_IN_ORDER               35  /* Buffers used in submission order */
#define VIRTIO_F_ORDER_PLATFORM         36  /* Memory ordering per platform */
#define VIRTIO_F_SR_IOV                 37  /* Single Root I/O Virtualization */
#define VIRTIO_F_NOTIFICATION_DATA      38  /* Extra data in notifications */
#define VIRTIO_F_NOTIF_CONFIG_DATA      39  /* Device-provided VQ notification data */
#define VIRTIO_F_RING_RESET             40  /* Per-queue reset support */
#define VIRTIO_F_ADMIN_VQ               41  /* Administration virtqueues */
```

---

## Virtqueue Operations API

Drivers interact with virtqueues through a well-defined API declared in
`include/linux/virtio.h` and implemented in `drivers/virtio/virtio_ring.c`.

### Adding Buffers

```c
/* Expose scatter-gather lists to the device */
int virtqueue_add_sgs(struct virtqueue *vq,          /* line 2300 */
                      struct scatterlist *sgs[],
                      unsigned int out_sgs,
                      unsigned int in_sgs,
                      void *data, gfp_t gfp);

/* Expose a single output buffer */
int virtqueue_add_outbuf(struct virtqueue *vq,       /* line 2334 */
                         struct scatterlist *sg,
                         unsigned int num,
                         void *data, gfp_t gfp);

/* Expose a single input buffer */
int virtqueue_add_inbuf(struct virtqueue *vq,        /* line 2379 */
                        struct scatterlist *sg,
                        unsigned int num,
                        void *data, gfp_t gfp);
```

### Notifying the Device

```c
/* Combined kick: check if notification needed, then notify */
bool virtqueue_kick(struct virtqueue *vq);           /* line 2510 */

/* Two-phase kick for better batching: */
bool virtqueue_kick_prepare(struct virtqueue *vq);   /* line 2465 */
bool virtqueue_notify(struct virtqueue *vq);         /* line 2482 */
```

`virtqueue_kick()` checks whether the device actually needs a notification
(via event index or flags), and only sends one if required. This is the main
optimization path for reducing VM exits.

### Retrieving Used Buffers

```c
/* Get next completed buffer; returns the data token from add_* */
void *virtqueue_get_buf(struct virtqueue *vq,        /* line 2545 */
                        unsigned int *len);
```

### Callback Control

```c
/* Disable callbacks (optimization, not guaranteed) */
void virtqueue_disable_cb(struct virtqueue *vq);     /* line 2559 */

/* Re-enable callbacks; returns last_used_idx for poll check */
unsigned int virtqueue_enable_cb_prepare(             /* line 2582 */
    struct virtqueue *vq);

/* Re-enable callbacks (simple version) */
bool virtqueue_enable_cb(struct virtqueue *vq);      /* line 2627 */

/* Re-enable with delayed notification (3/4 of ring) */
bool virtqueue_enable_cb_delayed(struct virtqueue *vq); /* line 2648 */

/* Poll for completed buffers without callbacks */
bool virtqueue_poll(struct virtqueue *vq,            /* line 2603 */
                    unsigned int last_used_idx);
```

### Interrupt Handler

```c
/* Called from transport interrupt handler */
irqreturn_t vring_interrupt(int irq, void *_vq);    /* line 2690 */
```

### Typical Driver Usage Pattern

```c
/* NAPI-style polling loop used by virtio-net and similar drivers */

/* In interrupt handler (callback): */
void my_vq_callback(struct virtqueue *vq)
{
    virtqueue_disable_cb(vq);        /* Stop interrupts */
    schedule_work(&my_work);         /* Defer to process context */
}

/* In worker / NAPI poll: */
void my_poll(void)
{
    while ((buf = virtqueue_get_buf(vq, &len)) != NULL) {
        process_buffer(buf, len);
    }

    /* Re-enable and check for race */
    if (virtqueue_enable_cb(vq)) {
        /* No race - callbacks re-armed */
    } else {
        /* New buffers arrived while re-enabling, poll again */
        virtqueue_disable_cb(vq);
        goto poll_again;
    }
}
```

---

## Transport Backends

Each transport provides `virtio_config_ops` and handles device discovery,
interrupt delivery, and vring memory management.

### Virtio over PCI

The most common transport for x86/KVM virtual machines. Supports both legacy
(pre-1.0) and modern (1.0+) PCI device layouts.

**Files**: `drivers/virtio/virtio_pci_common.c`, `virtio_pci_modern.c`,
`virtio_pci_legacy.c`

```c
/* drivers/virtio/virtio_pci_common.h:60 */
struct virtio_pci_device {
    struct virtio_device vdev;
    struct pci_dev *pci_dev;
    union {
        struct virtio_pci_legacy_device ldev;
        struct virtio_pci_modern_device mdev;
    };
    bool is_legacy;

    u8 __iomem *isr;             /* ISR register for interrupt status */

    spinlock_t lock;
    struct list_head virtqueues;
    struct list_head slow_virtqueues;
    struct virtio_pci_vq_info **vqs;

    struct virtio_pci_admin_vq admin_vq;   /* Admin virtqueue */

    /* MSI-X support */
    int msix_enabled;
    int intx_enabled;
    unsigned int msix_vectors;
    unsigned int msix_used_vectors;
    bool per_vq_vectors;         /* One MSI-X vector per virtqueue */

    struct virtqueue *(*setup_vq)(...);
    void (*del_vq)(...);
    u16 (*config_vector)(...);
};
```

PCI Virtio devices use PCI vendor ID 0x1AF4 (Red Hat) with device IDs
0x1000-0x103F (legacy) or 0x1040-0x107F (modern). MSI-X is used to assign
separate interrupt vectors to each virtqueue and the config change notification.

### Virtio over MMIO (Memory-Mapped I/O)

Used primarily in embedded/ARM platforms (e.g., QEMU's `-device virtio-*-device`
on ARM virt machine).

**File**: `drivers/virtio/virtio_mmio.c`

```c
/* drivers/virtio/virtio_mmio.c:85 */
struct virtio_mmio_device {
    struct virtio_device vdev;
    struct platform_device *pdev;

    void __iomem *base;          /* MMIO register base */
    unsigned long version;       /* MMIO transport version (1 or 2) */

    spinlock_t lock;
    struct list_head virtqueues;
};
```

MMIO devices are described via Device Tree (`compatible = "virtio,mmio"`)
or platform device registration. All device configuration is done through
memory-mapped registers at fixed offsets.

### Virtio over CCW (Channel I/O)

Used on IBM s390 (System z) platforms. Device discovery uses the s390 channel
subsystem.

**File**: `drivers/s390/virtio/virtio_ccw.c`

### Transport Comparison

```
+---------------------------------------------------------------+
|              Transport Comparison                             |
+---------------------------------------------------------------+
|                                                               |
|  Property       | PCI          | MMIO        | CCW            |
|  ---------------+--------------+-------------+----------      |
|  Platform       | x86, ARM     | ARM, RISC-V | s390           |
|  Discovery      | PCI bus scan | DT / ACPI   | Channel I/O    |
|  Config space   | PCI BARs     | Fixed MMIO  | CCW commands   |
|  Interrupts     | MSI-X/INTx   | Single IRQ  | Channel IRQ    |
|  Per-VQ vectors | Yes (MSI-X)  | No          | Yes            |
|  Hypervisor     | QEMU, etc.   | QEMU, kvmtool| z/VM, KVM     |
|                                                               |
+---------------------------------------------------------------+
```

---

## Notification Suppression

Notifications (guest-to-host kicks and host-to-guest interrupts) are the most
expensive operations in virtio because they trigger VM exits. Virtio provides
two mechanisms to reduce them.

### Flags-Based Suppression

The simplest mechanism uses flag fields in the available and used rings:

```c
/* include/uapi/linux/virtio_ring.h:57 */
#define VRING_USED_F_NO_NOTIFY      1  /* Host -> Guest: don't kick me */
#define VRING_AVAIL_F_NO_INTERRUPT  1  /* Guest -> Host: don't interrupt me */
```

- The device sets `VRING_USED_F_NO_NOTIFY` in `used->flags` to tell the
  driver "don't notify me when you add buffers." The driver checks this in
  `virtqueue_kick_prepare()`.
- The driver sets `VRING_AVAIL_F_NO_INTERRUPT` in `avail->flags` to tell
  the device "don't interrupt me when you consume buffers."

### Event Index Suppression (VIRTIO_RING_F_EVENT_IDX)

A more precise mechanism negotiated via feature bit 29. Instead of binary
on/off, each side publishes the index at which it wants to be notified:

```c
/* include/uapi/linux/virtio_ring.h:196 */
#define vring_used_event(vr)  ((vr)->avail->ring[(vr)->num])
#define vring_avail_event(vr) (*(__virtio16 *)&(vr)->used->ring[(vr)->num])
```

The `vring_need_event()` function at `include/uapi/linux/virtio_ring.h:222`
determines whether a notification is needed:

```c
static inline int vring_need_event(__u16 event_idx,
                                   __u16 new_idx, __u16 old)
{
    return (__u16)(new_idx - event_idx - 1) < (__u16)(new_idx - old);
}
```

This triggers a notification only when the index crosses the event threshold,
reducing notifications to approximately one per batch of operations.

### Packed Ring Event Suppression

The packed ring uses dedicated event suppression structures:

```c
/* include/uapi/linux/virtio_ring.h:64 */
#define VRING_PACKED_EVENT_FLAG_ENABLE   0x0  /* Enable all events */
#define VRING_PACKED_EVENT_FLAG_DISABLE  0x1  /* Disable all events */
#define VRING_PACKED_EVENT_FLAG_DESC     0x2  /* Enable for specific desc */
```

---

## DMA Considerations

Virtio has complex DMA requirements due to historical legacy and the variety
of deployment scenarios.

### The VIRTIO_F_ACCESS_PLATFORM Feature

```c
/* include/uapi/linux/virtio_config.h:76 */
#define VIRTIO_F_ACCESS_PLATFORM    33
```

This feature bit has reverse polarity for backward compatibility:

- **If NOT set** (most legacy/emulated devices): The device can access all
  guest memory directly. The driver bypasses the DMA API and uses
  guest-physical addresses directly in descriptors. This is the "DMA quirk."

- **If SET** (hardware virtio, VFIO, platform IOMMU): The device respects
  the platform's IOMMU. The driver must use the DMA API for all buffer
  mappings, and descriptors contain DMA-mapped addresses.

The decision logic is in `vring_use_dma_api()` at
`drivers/virtio/virtio_ring.c:271`:

```c
static bool vring_use_dma_api(const struct virtio_device *vdev)
{
    if (!virtio_has_dma_quirk(vdev))
        return true;       /* ACCESS_PLATFORM set: use DMA API */
    if (xen_domain())
        return true;       /* Xen always needs DMA API */
    return false;          /* Legacy: bypass DMA API */
}
```

### DMA Helper API

`virtio_ring.c` provides wrappers for drivers that need fine-grained DMA
control (e.g., for pre-mapped buffers):

```c
/* include/linux/virtio.h:251 */
dma_addr_t virtqueue_dma_map_single_attrs(struct virtqueue *vq, void *ptr,
                                          size_t size, enum dma_data_direction dir,
                                          unsigned long attrs);
void virtqueue_dma_unmap_single_attrs(struct virtqueue *vq, dma_addr_t addr,
                                      size_t size, enum dma_data_direction dir,
                                      unsigned long attrs);
struct device *virtqueue_dma_dev(struct virtqueue *vq);  /* line 2443 */
```

---

## Vhost Kernel Acceleration

Vhost moves the virtio device backend from userspace (QEMU) into the host
kernel, eliminating context switches on the data path. The userspace QEMU
process still handles configuration, but bulk I/O is processed by a kernel
thread.

### Architecture

```
+---------------------------------------------------------------+
|                 Vhost Architecture                            |
+---------------------------------------------------------------+
|                                                               |
|   Guest Kernel                Host Kernel                     |
|   +------------------+       +------------------+             |
|   | virtio-net drv   |       | vhost-net module |             |
|   +--------+---------+       +--------+---------+             |
|            |                          |                       |
|   +--------+---------+       +--------+---------+             |
|   | virtqueue_kick() |       | vhost worker     |             |
|   |   (VM exit)      |------>| thread polls     |             |
|   +--------+---------+       | eventfd/vring    |             |
|            |                 +--------+---------+             |
|   +--------+-----------------------------------+              |
|   |         Shared Memory Vrings               |              |
|   |  (mapped into both guest and host kernel)  |              |
|   +--------------------------------------------+              |
|                                                               |
|   Host Userspace                                              |
|   +------------------+                                        |
|   | QEMU             |  <- Handles config, feature            |
|   | (control plane)  |     negotiation, setup                 |
|   +------------------+                                        |
|                                                               |
+---------------------------------------------------------------+
```

### Vhost Kernel Module (drivers/vhost/)

The vhost framework provides generic infrastructure for in-kernel virtio
backends.

```c
/* drivers/vhost/vhost.h:81 */
struct vhost_virtqueue {
    struct vhost_dev *dev;
    struct vhost_worker __rcu *worker;

    struct mutex mutex;
    unsigned int num;
    vring_desc_t __user *desc;          /* Userspace-mapped descriptor table */
    vring_avail_t __user *avail;        /* Userspace-mapped available ring */
    vring_used_t __user *used;          /* Userspace-mapped used ring */

    struct file *kick;                  /* eventfd for guest -> host notify */
    struct vhost_vring_call call_ctx;   /* eventfd for host -> guest interrupt */
    struct eventfd_ctx *error_ctx;

    vhost_work_fn_t handle_kick;        /* Work function for incoming kicks */
    u16 last_avail_idx;
    u16 last_used_idx;
    /* ... */
};
```

### Vhost Implementations

| Module      | File                     | Purpose                        |
|-------------|--------------------------|--------------------------------|
| vhost-net   | `drivers/vhost/net.c`    | Network I/O (tap/macvtap)      |
| vhost-scsi  | `drivers/vhost/scsi.c`   | SCSI target emulation          |
| vhost-vsock | `drivers/vhost/vsock.c`  | AF_VSOCK host transport        |
| vhost-vdpa  | `drivers/vhost/vdpa.c`   | vDPA device proxy              |
| vhost core  | `drivers/vhost/vhost.c`  | Shared infrastructure          |

### Eventfd-Based Notification

Vhost uses Linux eventfds to bridge guest notifications and host processing:

- **Kick (guest -> host)**: KVM's `ioeventfd` mechanism traps the guest's
  MMIO/PIO write to the virtqueue notification register and signals an eventfd.
  The vhost worker thread polls this eventfd and processes the vring.
- **Call (host -> guest)**: The vhost worker signals an `irqfd` eventfd, which
  KVM translates into a virtual interrupt injection into the guest.

---

## Vringh (Host-Side Vring Access)

The vringh API allows kernel code to access vrings as a host (consuming from
the available ring, writing to the used ring). This is used by vhost and other
in-kernel virtio backends.

### struct vringh (include/linux/vringh.h:25)

```c
/* include/linux/vringh.h:25 */
struct vringh {
    bool little_endian;
    bool event_indices;
    bool weak_barriers;
    bool use_va;                 /* Use virtual addresses vs physical */

    u16 last_avail_idx;          /* Where we're up to in avail ring */
    u16 last_used_idx;           /* Last used index we wrote */
    u32 completed;               /* Descriptors completed since last notify check */

    struct vring vring;          /* The underlying vring */
    struct vhost_iotlb *iotlb;   /* IOMMU translation table */
    spinlock_t *iotlb_lock;

    void (*notify)(struct vringh *);  /* Guest notification callback */
};
```

### Vringh Access Modes

The vringh API comes in three flavors depending on the memory access model:

```c
/* Userspace vrings (copy_from_user / copy_to_user) */
int vringh_init_user(struct vringh *vrh, ...);
int vringh_getdesc_user(struct vringh *vrh, ...);
int vringh_complete_user(struct vringh *vrh, u16 head, u32 len);

/* Kernelspace vrings (direct memcpy) */
int vringh_init_kern(struct vringh *vrh, ...);
int vringh_getdesc_kern(struct vringh *vrh, ...);
int vringh_complete_kern(struct vringh *vrh, u16 head, u32 len);

/* IOTLB-translated vrings (vhost IOMMU) */
int vringh_init_iotlb(struct vringh *vrh, ...);
int vringh_getdesc_iotlb(struct vringh *vrh, ...);
int vringh_complete_iotlb(struct vringh *vrh, u16 head, u32 len);
```

---

## Virtio Device Types

Standard virtio device IDs are defined in `include/uapi/linux/virtio_ids.h`:

```c
/* include/uapi/linux/virtio_ids.h */
#define VIRTIO_ID_NET           1   /* virtio net */
#define VIRTIO_ID_BLOCK         2   /* virtio block */
#define VIRTIO_ID_CONSOLE       3   /* virtio console */
#define VIRTIO_ID_RNG           4   /* virtio rng */
#define VIRTIO_ID_BALLOON       5   /* virtio balloon */
#define VIRTIO_ID_SCSI          8   /* virtio scsi */
#define VIRTIO_ID_9P            9   /* 9p virtio console */
#define VIRTIO_ID_GPU           16  /* virtio GPU */
#define VIRTIO_ID_INPUT         18  /* virtio input */
#define VIRTIO_ID_VSOCK         19  /* virtio vsock transport */
#define VIRTIO_ID_CRYPTO        20  /* virtio crypto */
#define VIRTIO_ID_MEM           24  /* virtio mem */
#define VIRTIO_ID_SOUND         25  /* virtio sound */
#define VIRTIO_ID_FS            26  /* virtio filesystem */
/* ... more device types ... */
```

### virtio-net

The most widely used virtio device. Provides a virtual network interface with
features like checksum offload, TSO, multi-queue, and RSS.

- **Driver**: `drivers/net/virtio_net.c`
- **Virtqueues**: Pairs of RX/TX queues (one pair per CPU in multiqueue mode),
  plus an optional control virtqueue.
- **Vhost backend**: `drivers/vhost/net.c` (kernel) or QEMU userspace.
- **Key features**: `VIRTIO_NET_F_MRG_RXBUF` (mergeable receive buffers),
  `VIRTIO_NET_F_MQ` (multiqueue), `VIRTIO_NET_F_CTRL_VQ` (control channel).

### virtio-blk

Block device emulation providing a virtual disk.

- **Driver**: `drivers/block/virtio_blk.c`
- **Virtqueues**: One or more request queues (multiqueue since v1.0).
- **Protocol**: Each request is a descriptor chain: header (command) ->
  data buffers -> status byte.
- **Key features**: `VIRTIO_BLK_F_MQ` (multiqueue), `VIRTIO_BLK_F_FLUSH`
  (cache flush), `VIRTIO_BLK_F_DISCARD`.

### virtio-scsi

SCSI host bus adapter emulation, providing full SCSI semantics.

- **Driver**: `drivers/scsi/virtio_scsi.c`
- **Virtqueues**: Control VQ + event VQ + one or more request VQs.
- **Advantage over virtio-blk**: Supports SCSI passthrough, multiple LUNs,
  hot-plug, and full SCSI command set.
- **Uses `scan` callback**: The `virtio_driver.scan` field was added
  specifically for virtio-scsi to trigger SCSI bus scanning after probe.

### Virtqueue Layout by Device Type

```
+---------------------------------------------------------------+
|          Virtqueue Layout by Device Type                      |
+---------------------------------------------------------------+
|                                                               |
|  virtio-net (multiqueue):                                     |
|    VQ 0: receiveq0      (RX, device -> driver)                |
|    VQ 1: transmitq0     (TX, driver -> device)                |
|    VQ 2: receiveq1                                            |
|    VQ 3: transmitq1                                           |
|    ...                                                        |
|    VQ 2N:   receiveqN                                         |
|    VQ 2N+1: transmitqN                                        |
|    VQ 2N+2: control     (driver -> device, for config cmds)   |
|                                                               |
|  virtio-blk:                                                  |
|    VQ 0: requestq0      (driver <-> device)                   |
|    VQ 1: requestq1      (if multiqueue)                       |
|    ...                                                        |
|                                                               |
|  virtio-scsi:                                                 |
|    VQ 0: controlq       (driver -> device, config commands)   |
|    VQ 1: eventq         (device -> driver, async events)      |
|    VQ 2: requestq0      (driver <-> device, SCSI commands)    |
|    VQ 3: requestq1      (if multiqueue)                       |
|    ...                                                        |
|                                                               |
|  virtio-console:                                              |
|    VQ 0: port0_receiveq                                       |
|    VQ 1: port0_transmitq                                      |
|    VQ 2: control_receiveq (if VIRTIO_CONSOLE_F_MULTIPORT)     |
|    VQ 3: control_transmitq                                    |
|    ...                                                        |
|                                                               |
+---------------------------------------------------------------+
```

---

## Source Files

### Core Virtio Framework

| File | Description |
|------|-------------|
| `drivers/virtio/virtio.c` | Virtio bus registration, `virtio_dev_probe()`, feature negotiation, device registration/unregistration, PM freeze/restore |
| `drivers/virtio/virtio_ring.c` | Vring implementation: `struct vring_virtqueue`, split and packed ring add/kick/get_buf, `vring_interrupt()`, `vring_create_virtqueue()` |
| `drivers/virtio/virtio_debug.c` | Debugfs support for filtering features at runtime |

### Transport Backends

| File | Description |
|------|-------------|
| `drivers/virtio/virtio_pci_common.c` | Shared PCI transport code: `vp_find_vqs()`, `vp_notify()`, MSI-X setup |
| `drivers/virtio/virtio_pci_modern.c` | Modern (v1.0+) PCI virtio device support with capability-based config |
| `drivers/virtio/virtio_pci_legacy.c` | Legacy (pre-1.0) PCI virtio device support |
| `drivers/virtio/virtio_pci_modern_dev.c` | Modern PCI device access helpers (read/write BAR regions) |
| `drivers/virtio/virtio_pci_legacy_dev.c` | Legacy PCI device access helpers |
| `drivers/virtio/virtio_mmio.c` | MMIO transport: platform device driver, register-based config access |
| `drivers/s390/virtio/virtio_ccw.c` | s390 Channel I/O transport |
| `drivers/virtio/virtio_vdpa.c` | vDPA (virtio data path acceleration) transport |

### Vhost (In-Kernel Backends)

| File | Description |
|------|-------------|
| `drivers/vhost/vhost.c` | Core vhost framework: worker threads, vring access, eventfd, memory translation |
| `drivers/vhost/vhost.h` | `struct vhost_virtqueue`, `struct vhost_dev`, `struct vhost_worker` |
| `drivers/vhost/net.c` | vhost-net: kernel-accelerated virtio-net backend |
| `drivers/vhost/scsi.c` | vhost-scsi: kernel-accelerated SCSI target |
| `drivers/vhost/vsock.c` | vhost-vsock: AF_VSOCK host transport |
| `drivers/vhost/vdpa.c` | vhost-vdpa: userspace interface for vDPA devices |
| `drivers/vhost/vringh.c` | Host-side vring helpers implementation |
| `drivers/vhost/iotlb.c` | IOMMU translation table for vhost |

### Device Drivers

| File | Description |
|------|-------------|
| `drivers/net/virtio_net.c` | virtio-net network driver |
| `drivers/block/virtio_blk.c` | virtio-blk block driver |
| `drivers/scsi/virtio_scsi.c` | virtio-scsi SCSI HBA driver |
| `drivers/virtio/virtio_balloon.c` | Memory balloon driver |
| `drivers/virtio/virtio_mem.c` | Virtio memory device (hot-plug) |
| `drivers/virtio/virtio_input.c` | Virtio input device driver |

### Header Files

| File | Description |
|------|-------------|
| `include/linux/virtio.h` | `struct virtqueue`, `struct virtio_device`, `struct virtio_driver`, virtqueue API declarations |
| `include/linux/virtio_config.h` | `struct virtio_config_ops`, feature test helpers, config space accessors |
| `include/linux/vringh.h` | `struct vringh`, host-side vring access API |
| `include/uapi/linux/virtio_ring.h` | `struct vring_desc`, `struct vring_avail`, `struct vring_used`, `struct vring_packed_desc`, ring layout constants |
| `include/uapi/linux/virtio_config.h` | Status bits, transport feature bit definitions |
| `include/uapi/linux/virtio_ids.h` | Standard virtio device type IDs |
| `include/uapi/linux/virtio_pci.h` | PCI configuration space layout |
| `include/uapi/linux/virtio_mmio.h` | MMIO register map |
| `include/linux/mod_devicetable.h` | `struct virtio_device_id` (line 449) |

---

## Summary

The virtio subsystem provides a layered, extensible framework for
paravirtualized I/O:

| Component | Purpose |
|-----------|---------|
| `struct virtio_device` | Device instance with features and config ops |
| `struct virtio_driver` | Driver with feature table and probe/remove callbacks |
| `struct virtqueue` | Driver-facing queue handle |
| `struct vring_virtqueue` | Internal split/packed ring implementation |
| `struct virtio_config_ops` | Transport-specific operations vtable |
| `virtio_bus` | Bus type for device/driver matching |
| Feature negotiation | Forward/backward compatible capability exchange |
| Split vring | Three-area layout (desc + avail + used) |
| Packed vring | Single-area layout with in-place avail/used flags |
| Event index | Fine-grained notification suppression |
| Vhost | In-kernel device backends for near-native performance |
| Vringh | Host-side vring access API |

Key design benefits:

- **Performance**: Shared-memory rings avoid data copying; notification
  suppression minimizes VM exits; vhost eliminates userspace context switches.
- **Portability**: Transport-independent protocol works across PCI, MMIO,
  and channel I/O; feature negotiation handles version differences.
- **Simplicity**: Clean driver API (add/kick/get_buf) hides ring complexity;
  `module_virtio_driver()` macro reduces boilerplate.
- **Ecosystem**: Standardized by OASIS; implementations in Linux, QEMU, DPDK,
  FreeBSD, and hardware (vDPA).
