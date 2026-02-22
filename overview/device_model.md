# Linux Kernel Device Model Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Kobject and Kset](#kobject-and-kset)
3. [Sysfs Interface](#sysfs-interface)
4. [Bus/Driver/Device Model](#busdriverdevice-model)
5. [Platform Devices](#platform-devices)
6. [Device Resource Management](#device-resource-management)
7. [Character Devices](#character-devices)
8. [Device Classes](#device-classes)
9. [Device Tree](#device-tree)

---

## Introduction

The Linux Device Model is a unified framework for representing devices,
drivers, and buses in the kernel. It provides:

- Consistent device hierarchy visible through sysfs
- Hotplug and device discovery infrastructure
- Power management integration
- Driver binding and unbinding
- Reference counting for device lifetimes

### Device Model Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                 Device Model Hierarchy                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /sys                                                       │
│   ├── bus/                    ← Bus types                   │
│   │   ├── pci/                                              │
│   │   │   ├── devices/        ← Symlinks to devices         │
│   │   │   └── drivers/        ← Registered drivers          │
│   │   ├── usb/                                              │
│   │   ├── platform/                                         │
│   │   └── ...                                               │
│   │                                                         │
│   ├── class/                  ← Device classes              │
│   │   ├── net/                ← Network devices             │
│   │   ├── block/              ← Block devices               │
│   │   ├── input/              ← Input devices               │
│   │   └── ...                                               │
│   │                                                         │
│   ├── devices/                ← Physical device hierarchy   │
│   │   ├── system/                                           │
│   │   │   └── cpu/            ← CPUs                        │
│   │   ├── pci0000:00/         ← PCI root bus                │
│   │   │   └── 0000:00:1f.0/   ← PCI device                  │
│   │   └── platform/           ← Platform devices            │
│   │                                                         │
│   ├── kernel/                 ← Kernel subsystems           │
│   ├── module/                 ← Loaded modules              │
│   └── firmware/               ← Firmware interfaces         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Kobject and Kset

Kobject is the fundamental building block of the device model. Every object in
sysfs is backed by a kobject.

### Kobject Structure

The kobject structure is the fundamental building block of the Linux device
model, providing reference counting, sysfs representation, and hierarchical
organization for kernel objects. Every object that appears in sysfs is backed
by a kobject, which manages the object's lifecycle through its embedded kref.
The key fields include `name` for the sysfs directory name, `parent` to
establish hierarchy, `kset` for grouping related objects, and `sd` which links
to the sysfs directory node. Kobjects are typically embedded within larger
structures rather than used standalone.

The kobj_type structure defines the behavior and operations for a kobject. It
specifies the `release` callback invoked when the reference count drops to
zero, `sysfs_ops` for handling attribute reads/writes, and `default_groups` for
automatic attribute creation. This type information is shared among all
kobjects of the same kind.

The kset structure represents a collection of related kobjects, providing a
container with its own kobject for sysfs representation. It maintains a list of
member kobjects protected by a spinlock and can specify `uevent_ops` to
customize hotplug event handling for its members.

```c
/* include/linux/kobject.h */
struct kobject {
    const char              *name;          /* Object name */
    struct list_head        entry;          /* Link in kset list */
    struct kobject          *parent;        /* Parent in hierarchy */
    struct kset             *kset;          /* Containing kset */
    struct kobj_type        *ktype;         /* Type information */
    struct kernfs_node      *sd;            /* Sysfs directory */
    struct kref             kref;           /* Reference count */
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work     release;
#endif
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};

/* Kobject type - defines operations */
struct kobj_type {
    void (*release)(struct kobject *kobj);  /* Called when refcount = 0 */
    const struct sysfs_ops *sysfs_ops;      /* Sysfs operations */
    const struct attribute_group **default_groups;
    const struct kobj_ns_type_operations *(*child_ns_type)(
                                            struct kobject *kobj);
    const void *(*namespace)(struct kobject *kobj);
    void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};

/* Kset - collection of kobjects */
struct kset {
    struct list_head        list;           /* List of kobjects */
    spinlock_t              list_lock;      /* Protects list */
    struct kobject          kobj;           /* Embedded kobject */
    const struct kset_uevent_ops *uevent_ops;
};
```

### Kobject Operations

```c
/* Initialize kobject */
void kobject_init(struct kobject *kobj, struct kobj_type *ktype);

/* Add to sysfs hierarchy */
int kobject_add(struct kobject *kobj, struct kobject *parent,
                const char *fmt, ...);

/* Combined init + add */
int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,
                         struct kobject *parent, const char *fmt, ...);

/* Reference counting */
struct kobject *kobject_get(struct kobject *kobj);
void kobject_put(struct kobject *kobj);  /* May call release */

/* Create/destroy */
struct kobject *kobject_create_and_add(const char *name,
                                        struct kobject *parent);
void kobject_del(struct kobject *kobj);

/* Uevents (hotplug notifications) */
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
                       char *envp[]);

/* Actions for uevents */
enum kobject_action {
    KOBJ_ADD,
    KOBJ_REMOVE,
    KOBJ_CHANGE,
    KOBJ_MOVE,
    KOBJ_ONLINE,
    KOBJ_OFFLINE,
    KOBJ_BIND,
    KOBJ_UNBIND,
};
```

### Kobject Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                   Kobject Lifecycle                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Allocate containing structure                           │
│     my_obj = kzalloc(sizeof(*my_obj), GFP_KERNEL);          │
│                                                             │
│  2. Initialize kobject                                      │
│     kobject_init(&my_obj->kobj, &my_ktype);                 │
│         │                                                   │
│         ▼                                                   │
│  3. Add to hierarchy (appears in sysfs)                     │
│     kobject_add(&my_obj->kobj, parent, "name");             │
│         │                                                   │
│         ├─── kobject_get() increments refcount              │
│         │                                                   │
│         ▼                                                   │
│  4. Send uevent (notifies userspace)                        │
│     kobject_uevent(&my_obj->kobj, KOBJ_ADD);                │
│         │                                                   │
│         │ ... object in use ...                             │
│         │                                                   │
│         ▼                                                   │
│  5. Remove from sysfs                                       │
│     kobject_del(&my_obj->kobj);                             │
│         │                                                   │
│         ▼                                                   │
│  6. Drop reference                                          │
│     kobject_put(&my_obj->kobj);                             │
│         │                                                   │
│         ├─── If refcount → 0: ktype->release() called       │
│         │                                                   │
│         ▼                                                   │
│  7. release() frees containing structure                    │
│     kfree(my_obj);                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Sysfs Interface

Sysfs exposes kernel objects to userspace as a virtual filesystem.

### Attribute Types

The attribute structure is the basic building block for sysfs file
representation, defining a single readable or writable file within a sysfs
directory. Each attribute has a `name` that becomes the filename and a `mode`
that sets the file permissions. Attributes are typically not used directly but
are embedded in higher-level attribute types like device_attribute.

The sysfs_ops structure defines the read (`show`) and write (`store`) callbacks
used to handle userspace access to sysfs attributes. The `show` function
formats data into a buffer for reading, while `store` parses user input for
writing.

The bin_attribute structure extends the basic attribute to support binary data
of arbitrary size. It includes `size` for the expected data length and provides
`read`, `write`, and `mmap` callbacks for handling binary file operations,
commonly used for firmware uploads or memory-mapped register access.

The attribute_group structure organizes multiple attributes together,
optionally placing them in a named subdirectory. It includes `is_visible`
callbacks for conditionally showing attributes based on device capabilities and
arrays for both regular and binary attributes.

```c
/* include/linux/sysfs.h */

/* Basic attribute */
struct attribute {
    const char              *name;      /* Attribute filename */
    umode_t                 mode;       /* File permissions */
};

/* Attribute operations */
struct sysfs_ops {
    ssize_t (*show)(struct kobject *kobj, struct attribute *attr,
                    char *buf);
    ssize_t (*store)(struct kobject *kobj, struct attribute *attr,
                     const char *buf, size_t count);
};

/* Binary attribute (for binary data) */
struct bin_attribute {
    struct attribute        attr;
    size_t                  size;
    void                    *private;
    ssize_t (*read)(struct file *filp, struct kobject *kobj,
                    struct bin_attribute *attr,
                    char *buf, loff_t off, size_t size);
    ssize_t (*write)(struct file *filp, struct kobject *kobj,
                     struct bin_attribute *attr,
                     char *buf, loff_t off, size_t size);
    int (*mmap)(struct file *filp, struct kobject *kobj,
                struct bin_attribute *attr, struct vm_area_struct *vma);
};

/* Attribute group */
struct attribute_group {
    const char              *name;      /* Subdirectory name (or NULL) */
    umode_t                 (*is_visible)(struct kobject *kobj,
                                          struct attribute *attr, int n);
    umode_t                 (*is_bin_visible)(struct kobject *kobj,
                                              struct bin_attribute *attr, int n);
    struct attribute        **attrs;
    struct bin_attribute    **bin_attrs;
};
```

### Device Attributes

The device_attribute structure is a specialized attribute type for use with
struct device objects. It wraps the basic attribute structure and provides
device-aware `show` and `store` callbacks that receive the struct device
pointer, allowing drivers to access device-specific data. This is the most
common way for drivers to expose configuration and status information through
sysfs, typically created using the DEVICE_ATTR family of macros.

```c
/* include/linux/device.h */

/* Device attribute */
struct device_attribute {
    struct attribute        attr;
    ssize_t (*show)(struct device *dev, struct device_attribute *attr,
                    char *buf);
    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                     const char *buf, size_t count);
};

/* Macros for declaring device attributes */
#define DEVICE_ATTR(_name, _mode, _show, _store) \
    struct device_attribute dev_attr_##_name = \
        __ATTR(_name, _mode, _show, _store)

#define DEVICE_ATTR_RO(_name) \
    struct device_attribute dev_attr_##_name = __ATTR_RO(_name)

#define DEVICE_ATTR_RW(_name) \
    struct device_attribute dev_attr_##_name = __ATTR_RW(_name)

#define DEVICE_ATTR_WO(_name) \
    struct device_attribute dev_attr_##_name = __ATTR_WO(_name)
```

### Attribute Example

```c
/* Read-only attribute showing device status */
static ssize_t status_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%s\n", mydev->status);
}
static DEVICE_ATTR_RO(status);

/* Read-write attribute for device setting */
static ssize_t setting_show(struct device *dev,
                            struct device_attribute *attr, char *buf)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", mydev->setting);
}

static ssize_t setting_store(struct device *dev,
                             struct device_attribute *attr,
                             const char *buf, size_t count)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    int val;

    if (kstrtoint(buf, 10, &val))
        return -EINVAL;

    mydev->setting = val;
    return count;
}
static DEVICE_ATTR_RW(setting);

/* Group attributes */
static struct attribute *my_attrs[] = {
    &dev_attr_status.attr,
    &dev_attr_setting.attr,
    NULL,
};
ATTRIBUTE_GROUPS(my);

/* Use in device registration */
my_device_class->dev_groups = my_groups;
```

---

## Bus/Driver/Device Model

The bus/driver/device model is the core of device management.

### Bus Type

The bus_type structure represents a type of bus in the Linux device model, such
as PCI, USB, or platform. It serves as the matchmaker between devices and
drivers, providing the `match` callback to determine if a driver can handle a
specific device. Key fields include `name` for identification in sysfs, `probe`
and `remove` for device binding, and power management callbacks. The bus
maintains lists of registered devices and drivers, and attribute groups for
exposing bus-specific, device, and driver attributes through sysfs.

```c
/* include/linux/device/bus.h */
struct bus_type {
    const char              *name;          /* Bus name */
    const char              *dev_name;      /* Device name template */

    struct device           *dev_root;      /* Default parent */

    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    /* Match device to driver */
    int (*match)(struct device *dev, struct device_driver *drv);

    /* Called on successful match */
    int (*probe)(struct device *dev);
    void (*sync_state)(struct device *dev);

    int (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    int (*num_vf)(struct device *dev);
    int (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);

    const struct dev_pm_ops *pm;

    struct subsys_private *p;
};

/* Register/unregister bus */
int bus_register(struct bus_type *bus);
void bus_unregister(struct bus_type *bus);
```

### Device Driver

The device_driver structure represents a driver that can be bound to one or
more devices on a bus. It contains the driver's `name`, a pointer to the `bus`
it operates on, and critical lifecycle callbacks including `probe` (called when
binding to a device), `remove` (when unbinding), and `shutdown` (during system
shutdown). The structure also holds match tables (`of_match_table` for Device
Tree, `acpi_match_table` for ACPI) that specify which devices this driver
supports. The `owner` field tracks the module to prevent unloading while
devices are bound.

```c
/* include/linux/device/driver.h */
struct device_driver {
    const char              *name;          /* Driver name */
    struct bus_type         *bus;           /* Bus this driver works with */

    struct module           *owner;
    const char              *mod_name;

    bool suppress_bind_attrs;               /* Disable bind/unbind via sysfs */
    enum probe_type probe_type;

    const struct of_device_id *of_match_table;      /* Device tree matching */
    const struct acpi_device_id *acpi_match_table;  /* ACPI matching */

    int (*probe)(struct device *dev);       /* Bind driver to device */
    void (*sync_state)(struct device *dev);
    int (*remove)(struct device *dev);      /* Unbind driver from device */
    void (*shutdown)(struct device *dev);   /* Shutdown device */
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    const struct attribute_group **groups;
    const struct attribute_group **dev_groups;

    const struct dev_pm_ops *pm;
    void (*coredump)(struct device *dev);

    struct driver_private *p;
};

/* Register/unregister driver */
int driver_register(struct device_driver *drv);
void driver_unregister(struct device_driver *drv);
```

### Device Structure

The device structure is the core representation of any device in the Linux
kernel, serving as the universal base for all device types. It embeds a kobject
for sysfs representation and reference counting, and contains pointers to its
    `parent` device, `bus`, and bound `driver`. Key fields include
    `platform_data` and `driver_data` for passing information between
    subsystems and drivers, `of_node` for Device Tree integration, and `devt`
    for the major/minor device numbers. The structure also manages power state
        through `power` and `pm_domain`, DMA configuration, and maintains a
        list of managed resources (`devres_head`) for automatic cleanup.
        Devices are typically not allocated directly but embedded within
        bus-specific device structures like platform_device or pci_dev.

```c
/* include/linux/device.h */
struct device {
    struct kobject          kobj;
    struct device           *parent;

    struct device_private   *p;

    const char              *init_name;     /* Initial name */
    const struct device_type *type;

    struct bus_type         *bus;           /* Bus this device is on */
    struct device_driver    *driver;        /* Which driver is bound */

    void                    *platform_data;
    void                    *driver_data;   /* Driver-specific data */

    struct mutex            mutex;

    struct dev_links_info   links;
    struct dev_pm_info      power;
    struct dev_pm_domain    *pm_domain;

    u64                     *dma_mask;
    u64                     coherent_dma_mask;
    u64                     bus_dma_limit;

    const struct dma_map_ops *dma_ops;
    struct dma_coherent_mem *dma_mem;

    struct device_dma_parameters *dma_parms;

    struct list_head        dma_pools;

    struct dev_archdata     archdata;

    struct device_node      *of_node;       /* Device tree node */
    struct fwnode_handle    *fwnode;

    int                     numa_node;
    dev_t                   devt;           /* Major:minor numbers */
    u32                     id;

    spinlock_t              devres_lock;
    struct list_head        devres_head;    /* Resource management list */

    struct class            *class;
    const struct attribute_group **groups;

    void (*release)(struct device *dev);

    struct iommu_group      *iommu_group;
    struct dev_iommu        *iommu;

    bool                    offline_disabled:1;
    bool                    offline:1;
    bool                    of_node_reused:1;
    bool                    state_synced:1;
    bool                    can_match:1;
};
```

### Driver Binding Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   Driver Binding Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Device Registration:                                       │
│  ───────────────────                                        │
│  device_add(dev)                                            │
│      │                                                      │
│      ├─── Add to bus's device list                          │
│      │                                                      │
│      ├─── Create sysfs entries                              │
│      │                                                      │
│      └─── Try to bind to driver                             │
│           │                                                 │
│           ▼                                                 │
│      For each driver on bus:                                │
│           │                                                 │
│           ├─── bus->match(dev, drv)                         │
│           │         │                                       │
│           │         ├── No match: try next driver           │
│           │         │                                       │
│           │         ▼ Match!                                │
│           │                                                 │
│           └─── really_probe(dev, drv)                       │
│                     │                                       │
│                     ├── drv->probe(dev)                     │
│                     │         │                             │
│                     │         ├── Success: device bound     │
│                     │         │                             │
│                     │         └── Failure: try next driver  │
│                     │                                       │
│                     └── Send KOBJ_BIND uevent               │
│                                                             │
│  ═══════════════════════════════════════════════════════    │
│                                                             │
│  Driver Registration:                                       │
│  ────────────────────                                       │
│  driver_register(drv)                                       │
│      │                                                      │
│      ├─── Add to bus's driver list                          │
│      │                                                      │
│      └─── Try to bind to unbound devices                    │
│           │                                                 │
│           ▼                                                 │
│      For each unbound device on bus:                        │
│           │                                                 │
│           └─── [Same matching flow as above]                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Platform Devices

Platform devices represent devices that cannot be probed/enumerated
automatically.

### Platform Device Structure

The platform_device structure represents devices that are not discoverable
through hardware enumeration mechanisms like PCI or USB. These are typically
SoC-integrated peripherals described in Device Tree or ACPI tables, or legacy
devices with fixed addresses. It wraps a struct device and adds
platform-specific fields including `name` and `id` for identification, a
`resource` array describing memory regions and IRQs, and `id_entry` for driver
matching. Platform devices are commonly created from Device Tree nodes or
registered explicitly by board support code.

The resource structure describes a hardware resource such as a memory-mapped
I/O region, I/O port range, IRQ, or DMA channel. Each resource has `start` and
`end` addresses defining its range, a `name` for identification, and `flags`
indicating the resource type (IORESOURCE_MEM, IORESOURCE_IRQ, etc.). Resources
form a hierarchical tree through parent/sibling/child pointers for managing
resource allocation and conflicts.

```c
/* include/linux/platform_device.h */
struct platform_device {
    const char              *name;
    int                     id;
    bool                    id_auto;

    struct device           dev;

    u64                     platform_dma_mask;
    struct device_dma_parameters dma_parms;
    u32                     num_resources;
    struct resource         *resource;

    const struct platform_device_id *id_entry;
    char *driver_override;

    struct mfd_cell         *mfd_cell;

    struct pdev_archdata    archdata;
};

/* Resource types */
struct resource {
    resource_size_t         start;
    resource_size_t         end;
    const char              *name;
    unsigned long           flags;      /* IORESOURCE_* */
    unsigned long           desc;
    struct resource         *parent, *sibling, *child;
};

/* Resource flags */
#define IORESOURCE_IO       0x00000100  /* I/O port region */
#define IORESOURCE_MEM      0x00000200  /* Memory region */
#define IORESOURCE_REG      0x00000300  /* Register offsets */
#define IORESOURCE_IRQ      0x00000400  /* Interrupt */
#define IORESOURCE_DMA      0x00000800  /* DMA channel */
#define IORESOURCE_BUS      0x00001000  /* Bus number */
```

### Platform Driver

The platform_driver structure is the driver counterpart to platform_device,
providing a specialized driver interface for platform bus devices. It contains
the essential lifecycle callbacks `probe`, `remove`, `shutdown`, `suspend`, and
`resume` for device management, along with an embedded device_driver for
integration with the driver model. The `id_table` provides an array of
platform_device_id entries for matching devices by name, while Device Tree
matching is handled through the embedded driver's of_match_table.

The platform_device_id structure provides a simple name-based matching
mechanism for platform devices. Each entry contains a device name string and
optional driver_data that can be used to pass variant-specific information to
the driver. This is commonly used alongside Device Tree matching for backward
compatibility.

```c
/* include/linux/platform_device.h */
struct platform_driver {
    int (*probe)(struct platform_device *pdev);
    int (*remove)(struct platform_device *pdev);
    void (*shutdown)(struct platform_device *pdev);
    int (*suspend)(struct platform_device *pdev, pm_message_t state);
    int (*resume)(struct platform_device *pdev);

    struct device_driver    driver;

    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
};

/* Platform device ID table for matching */
struct platform_device_id {
    char name[PLATFORM_NAME_SIZE];
    kernel_ulong_t driver_data;
};

/* Registration macros */
#define platform_driver_register(drv) \
    __platform_driver_register(drv, THIS_MODULE)

#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)
```

### Platform Driver Example

```c
/* Example platform driver */
#include <linux/platform_device.h>
#include <linux/module.h>

struct my_device {
    void __iomem *base;
    int irq;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_device *mydev;
    struct resource *res;
    int ret;

    /* Allocate device structure */
    mydev = devm_kzalloc(&pdev->dev, sizeof(*mydev), GFP_KERNEL);
    if (!mydev)
        return -ENOMEM;

    /* Get and map memory resource */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    mydev->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(mydev->base))
        return PTR_ERR(mydev->base);

    /* Get IRQ */
    mydev->irq = platform_get_irq(pdev, 0);
    if (mydev->irq < 0)
        return mydev->irq;

    /* Request IRQ */
    ret = devm_request_irq(&pdev->dev, mydev->irq, my_irq_handler,
                           0, "my-device", mydev);
    if (ret)
        return ret;

    /* Store driver data */
    platform_set_drvdata(pdev, mydev);

    dev_info(&pdev->dev, "probed successfully\n");
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    dev_info(&pdev->dev, "removed\n");
    return 0;
}

/* Device tree match table */
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

/* Platform device ID table */
static const struct platform_device_id my_id_table[] = {
    { "my-device", 0 },
    { }
};
MODULE_DEVICE_TABLE(platform, my_id_table);

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .of_match_table = my_of_match,
    },
    .id_table = my_id_table,
};
module_platform_driver(my_driver);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("My Platform Driver");
```

---

## Device Resource Management

Devres (device resource management) provides automatic cleanup when devices are
unbound or removed.

### Devres API

```c
/* include/linux/device.h */

/* Managed memory allocation */
void *devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp);
void *devm_kcalloc(struct device *dev, size_t n, size_t size, gfp_t gfp);
char *devm_kstrdup(struct device *dev, const char *s, gfp_t gfp);
void devm_kfree(struct device *dev, const void *p);

/* Managed I/O memory mapping */
void __iomem *devm_ioremap(struct device *dev, resource_size_t offset,
                           resource_size_t size);
void __iomem *devm_ioremap_resource(struct device *dev,
                                     const struct resource *res);
void __iomem *devm_platform_ioremap_resource(struct platform_device *pdev,
                                              unsigned int index);
void devm_iounmap(struct device *dev, void __iomem *addr);

/* Managed IRQ */
int devm_request_irq(struct device *dev, unsigned int irq,
                     irq_handler_t handler, unsigned long irqflags,
                     const char *devname, void *dev_id);
int devm_request_threaded_irq(struct device *dev, unsigned int irq,
                               irq_handler_t handler,
                               irq_handler_t thread_fn,
                               unsigned long irqflags,
                               const char *devname, void *dev_id);
void devm_free_irq(struct device *dev, unsigned int irq, void *dev_id);

/* Managed clocks */
struct clk *devm_clk_get(struct device *dev, const char *id);
struct clk *devm_clk_get_optional(struct device *dev, const char *id);

/* Managed regulators */
struct regulator *devm_regulator_get(struct device *dev, const char *id);

/* Managed GPIO */
struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id,
                                  enum gpiod_flags flags);

/* Managed reset controller */
struct reset_control *devm_reset_control_get(struct device *dev,
                                              const char *id);
```

### Devres Internals

```
┌─────────────────────────────────────────────────────────────┐
│                    Devres Management                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  struct device {                                            │
│      ...                                                    │
│      spinlock_t devres_lock;                                │
│      struct list_head devres_head;  ─┐                      │
│      ...                             │                      │
│  }                                   │                      │
│                                      │                      │
│                                      ▼                      │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│  │  devres 1   │──▶│  devres 2   │──▶│  devres 3   │        │
│  │  (memory)   │   │  (ioremap)  │   │    (irq)    │        │
│  │  release()  │   │  release()  │   │  release()  │        │
│  └─────────────┘   └─────────────┘   └─────────────┘        │
│                                                             │
│  On driver unbind or device removal:                        │
│  ─────────────────────────────────────                      │
│  devres_release_all(dev)                                    │
│      │                                                      │
│      ├── Call release() for devres 3 (free_irq)             │
│      ├── Call release() for devres 2 (iounmap)              │
│      └── Call release() for devres 1 (kfree)                │
│                                                             │
│  Resources automatically cleaned up in reverse order!       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Custom Devres Actions

```c
/* Add custom cleanup action */
int devm_add_action(struct device *dev,
                    void (*action)(void *), void *data);
int devm_add_action_or_reset(struct device *dev,
                              void (*action)(void *), void *data);
void devm_remove_action(struct device *dev,
                        void (*action)(void *), void *data);

/* Example: Custom cleanup */
static void my_cleanup(void *data)
{
    struct my_resource *res = data;
    /* Cleanup code */
    my_resource_release(res);
}

static int my_probe(struct platform_device *pdev)
{
    struct my_resource *res;

    res = my_resource_acquire();
    if (!res)
        return -ENOMEM;

    /* Automatically call my_cleanup on unbind */
    return devm_add_action_or_reset(&pdev->dev, my_cleanup, res);
}
```

---

## Character Devices

Character devices provide byte-stream access to hardware.

### Character Device Structure

The cdev structure represents a character device in the kernel, providing the
bridge between device numbers and file operations. It embeds a kobject for
reference counting and contains a pointer to the file_operations that define
how the device handles open, read, write, and other file operations. The `dev`
field holds the major/minor device number, and `count` specifies how many minor
numbers this cdev handles. Character devices are initialized with cdev_init()
and registered with cdev_add().

The file_operations structure defines the callbacks invoked when userspace
performs operations on a device file. Key operations include `open` and
`release` for managing device access, `read` and `write` for data transfer,
`unlocked_ioctl` for device-specific commands, `mmap` for memory mapping, and
`poll` for event notification. The `owner` field prevents module unloading
while the device is in use.

```c
/* include/linux/cdev.h */
struct cdev {
    struct kobject          kobj;
    struct module           *owner;
    const struct file_operations *ops;
    struct list_head        list;
    dev_t                   dev;        /* Major:minor number */
    unsigned int            count;      /* Minor number range */
};

/* include/linux/fs.h */
struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int (*mmap)(struct file *, struct vm_area_struct *);
    unsigned long mmap_supported_flags;
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);
    int (*lock)(struct file *, int, struct file_lock *);
    unsigned int (*poll)(struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    int (*flock)(struct file *, int, struct file_lock *);
    /* ... more operations ... */
};
```

### Character Device Example

```c
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h>
#include <linux/module.h>

#define DEVICE_NAME "mychar"
#define CLASS_NAME "myclass"

static int major_number;
static struct class *mychar_class;
static struct cdev my_cdev;

static int mychar_open(struct inode *inode, struct file *file)
{
    pr_info("Device opened\n");
    return 0;
}

static int mychar_release(struct inode *inode, struct file *file)
{
    pr_info("Device closed\n");
    return 0;
}

static ssize_t mychar_read(struct file *file, char __user *buf,
                           size_t count, loff_t *offset)
{
    char message[] = "Hello from kernel!\n";
    size_t len = strlen(message);

    if (*offset >= len)
        return 0;

    if (count > len - *offset)
        count = len - *offset;

    if (copy_to_user(buf, message + *offset, count))
        return -EFAULT;

    *offset += count;
    return count;
}

static ssize_t mychar_write(struct file *file, const char __user *buf,
                            size_t count, loff_t *offset)
{
    char kbuf[256];
    size_t len = min(count, sizeof(kbuf) - 1);

    if (copy_from_user(kbuf, buf, len))
        return -EFAULT;

    kbuf[len] = '\0';
    pr_info("Received: %s\n", kbuf);

    return count;
}

static const struct file_operations mychar_fops = {
    .owner = THIS_MODULE,
    .open = mychar_open,
    .release = mychar_release,
    .read = mychar_read,
    .write = mychar_write,
};

static int __init mychar_init(void)
{
    dev_t dev;
    int ret;

    /* Allocate major number */
    ret = alloc_chrdev_region(&dev, 0, 1, DEVICE_NAME);
    if (ret < 0)
        return ret;

    major_number = MAJOR(dev);

    /* Initialize cdev */
    cdev_init(&my_cdev, &mychar_fops);
    my_cdev.owner = THIS_MODULE;

    /* Add cdev to system */
    ret = cdev_add(&my_cdev, dev, 1);
    if (ret < 0)
        goto err_unregister;

    /* Create device class */
    mychar_class = class_create(CLASS_NAME);
    if (IS_ERR(mychar_class)) {
        ret = PTR_ERR(mychar_class);
        goto err_cdev_del;
    }

    /* Create device node /dev/mychar */
    if (IS_ERR(device_create(mychar_class, NULL, dev, NULL, DEVICE_NAME))) {
        ret = -EINVAL;
        goto err_class_destroy;
    }

    pr_info("Registered with major number %d\n", major_number);
    return 0;

err_class_destroy:
    class_destroy(mychar_class);
err_cdev_del:
    cdev_del(&my_cdev);
err_unregister:
    unregister_chrdev_region(dev, 1);
    return ret;
}

static void __exit mychar_exit(void)
{
    dev_t dev = MKDEV(major_number, 0);

    device_destroy(mychar_class, dev);
    class_destroy(mychar_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev, 1);

    pr_info("Unregistered\n");
}

module_init(mychar_init);
module_exit(mychar_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Simple character device");
```

---

## Device Classes

Classes group devices by function rather than physical connection.

### Class Structure

The class structure represents a high-level classification of devices based on
their function rather than their physical connection. Examples include "net"
for network interfaces, "block" for block devices, and "input" for input
    devices. Classes appear in /sys/class/ and provide a unified view of
    similar devices regardless of which bus they are connected to. The
    structure includes attribute groups for class-wide and per-device
    attributes, `devnode` for customizing /dev node creation, and `dev_release`
    for cleanup when devices are removed. Classes are typically created with
        class_create() and devices are added using device_create().

```c
/* include/linux/device/class.h */
struct class {
    const char              *name;
    struct module           *owner;

    const struct attribute_group **class_groups;
    const struct attribute_group **dev_groups;

    int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(struct device *dev, umode_t *mode);

    void (*class_release)(struct class *class);
    void (*dev_release)(struct device *dev);

    int (*shutdown_pre)(struct device *dev);

    const struct kobj_ns_type_operations *ns_type;
    const void *(*namespace)(struct device *dev);

    void (*get_ownership)(struct device *dev, kuid_t *uid, kgid_t *gid);

    const struct dev_pm_ops *pm;

    struct subsys_private *p;
};
```

### Class Example

```c
/* Creating a device class */
static struct class *my_class;

static int __init my_init(void)
{
    /* Create class - appears in /sys/class/myclass/ */
    my_class = class_create("myclass");
    if (IS_ERR(my_class))
        return PTR_ERR(my_class);

    /* Create device within class */
    device_create(my_class, NULL, MKDEV(major, 0), NULL, "mydev0");

    return 0;
}

static void __exit my_exit(void)
{
    device_destroy(my_class, MKDEV(major, 0));
    class_destroy(my_class);
}
```

---

## Device Tree

Device Tree provides hardware description for non-discoverable devices.

### Device Tree Bindings

```dts
/* Example device tree node */
my_device@ff000000 {
    compatible = "vendor,my-device";
    reg = <0xff000000 0x1000>;
    interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk_controller 5>;
    clock-names = "core";
    resets = <&reset_controller 3>;
    reset-names = "device";
    my-custom-property = <0x1234>;
    status = "okay";
};
```

### Device Tree Matching

The of_device_id structure is used to match drivers to devices described in
Device Tree. Each entry specifies a `compatible` string that matches against
the device node's compatible property, with optional `name` and `type` fields
for additional matching criteria. The `data` field can point to device-specific
    configuration data that the driver retrieves after matching. Match tables
    are typically defined as static const arrays terminated by an empty entry
    and registered with MODULE_DEVICE_TABLE for module autoloading.

```c
/* include/linux/of.h */
struct of_device_id {
    char    name[32];
    char    type[32];
    char    compatible[128];
    const void *data;
};

/* Driver match table */
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device-v1", .data = &variant_v1 },
    { .compatible = "vendor,my-device-v2", .data = &variant_v2 },
    { }
};
MODULE_DEVICE_TABLE(of, my_of_match);

/* Reading properties in probe */
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
    const struct of_device_id *match;
    u32 val;
    int ret;

    /* Get matched device ID */
    match = of_match_device(my_of_match, dev);
    if (match)
        variant = match->data;

    /* Read property */
    ret = of_property_read_u32(np, "my-custom-property", &val);
    if (ret)
        return ret;

    /* Read string property */
    const char *str;
    of_property_read_string(np, "my-string-prop", &str);

    /* Check property existence */
    if (of_property_read_bool(np, "some-flag"))
        enable_feature();

    return 0;
}
```

---

## Summary

The Linux Device Model provides a comprehensive framework for device management:

| Component | Purpose |
|-----------|---------|
| kobject | Base object with refcounting and sysfs support |
| kset | Collection of kobjects |
| sysfs | Userspace interface (/sys) |
| bus_type | Matches devices to drivers |
| device_driver | Driver operations |
| device | Device representation |
| platform_device | Non-enumerable devices |
| devres | Automatic resource cleanup |
| cdev | Character device interface |
| class | Functional device grouping |

Key benefits:
- **Unified hierarchy**: Consistent device representation
- **Hotplug support**: Dynamic device addition/removal
- **Resource management**: devres for automatic cleanup
- **Power management**: Integrated PM infrastructure
- **Userspace visibility**: sysfs for configuration and monitoring
