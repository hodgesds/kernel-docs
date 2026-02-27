# Linux Kernel KVM (Kernel-based Virtual Machine) Overview

## Table of Contents
1. [Introduction](#1-introduction)
2. [KVM Architecture](#2-kvm-architecture)
3. [Ioctl Interface and Device Model](#3-ioctl-interface-and-device-model)
4. [Core Data Structures](#4-core-data-structures)
5. [VM and vCPU Lifecycle](#5-vm-and-vcpu-lifecycle)
6. [VMCS / VMCB — Hardware Virtualization State](#6-vmcs--vmcb--hardware-virtualization-state)
7. [VM Exits](#7-vm-exits)
8. [Memory Virtualization (EPT/NPT)](#8-memory-virtualization-eptnpt)
9. [Interrupt and Exception Injection](#9-interrupt-and-exception-injection)
10. [I/O Handling](#10-io-handling)
11. [Hardware Backends: VMX and SVM](#11-hardware-backends-vmx-and-svm)
12. [Paravirtual (PV) Features](#12-paravirtual-pv-features)
13. [Nested Virtualization](#13-nested-virtualization)
14. [Key Source Files](#14-key-source-files)

---

## 1. Introduction

KVM (Kernel-based Virtual Machine) turns the Linux kernel itself into a Type-1
hypervisor. It was merged in Linux 2.6.20 (2007) and leverages hardware
virtualization extensions — Intel VT-x and AMD-V — to run unmodified guest
operating systems at near-native speed.

KVM is implemented as a loadable kernel module (`kvm.ko` plus a
hardware-specific module such as `kvm-intel.ko` or `kvm-amd.ko`). It exposes a
`/dev/kvm` character device to userspace, which communicates via a set of
`ioctl()` calls to create VMs, add vCPUs, map memory, and run guest code.

Userspace VMMs (Virtual Machine Monitors) such as QEMU, crosvm, Cloud
Hypervisor, and Firecracker sit on top of the KVM ioctl interface and provide
device emulation, firmware, and management.

```
virt/kvm/kvm_main.c:73-75
  MODULE_AUTHOR("Qumranet");
  MODULE_DESCRIPTION("Kernel-based Virtual Machine (KVM) Hypervisor");
  MODULE_LICENSE("GPL");
```

### Key Design Principles

- **Reuse the kernel**: Scheduling, memory management, and device drivers are
  all inherited from the host Linux kernel rather than reimplemented.
- **Hardware-assisted**: Relies on CPU virtualization extensions (VT-x / AMD-V)
  for efficient guest execution — no binary translation.
- **Split architecture**: A thin kernel module handles privileged operations;
  userspace handles device emulation and VM management.
- **Per-architecture backends**: Common code lives in `virt/kvm/`; architecture-
  specific code lives under `arch/<arch>/kvm/`.

---

## 2. KVM Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         User Space                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │   VMM (QEMU / crosvm / Firecracker / Cloud Hypervisor)        │  │
│  │                                                                │  │
│  │   ┌──────────┐  ┌──────────┐  ┌─────────────────────────┐    │  │
│  │   │  VM mgmt │  │  Device  │  │ vCPU threads            │    │  │
│  │   │  ioctls  │  │  emulat. │  │ (KVM_RUN loop)          │    │  │
│  │   └────┬─────┘  └────┬─────┘  └────────┬────────────────┘    │  │
│  └────────┼──────────────┼─────────────────┼─────────────────────┘  │
│           │              │                 │                         │
│ ──────────┼──────────────┼─────────────────┼───── /dev/kvm ─────── │
│           │   ioctl()    │   ioctl()       │   ioctl(KVM_RUN)      │
├───────────┼──────────────┼─────────────────┼────────────────────────┤
│           ▼              ▼                 ▼     Kernel Space       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    KVM Core (kvm.ko)                         │   │
│  │                   virt/kvm/kvm_main.c                        │   │
│  │                                                              │   │
│  │   ┌──────────┐  ┌──────────┐  ┌───────────────────────┐    │   │
│  │   │  VM mgmt │  │  Memory  │  │   vCPU scheduling     │    │   │
│  │   │  (struct │  │  slots & │  │   & VM entry/exit     │    │   │
│  │   │   kvm)   │  │  MMU     │  │                       │    │   │
│  │   └──────────┘  └──────────┘  └───────────────────────┘    │   │
│  └──────────────────────┬───────────────────────────────────────┘   │
│                         │                                           │
│  ┌──────────────────────▼───────────────────────────────────────┐   │
│  │        Hardware Backend (kvm-intel.ko / kvm-amd.ko)          │   │
│  │        arch/x86/kvm/vmx/  or  arch/x86/kvm/svm/             │   │
│  │                                                              │   │
│  │   ┌──────────┐  ┌──────────┐  ┌───────────────────────┐    │   │
│  │   │  VMCS /  │  │  VM exit │  │  Hardware-specific    │    │   │
│  │   │  VMCB    │  │  handler │  │  feature enablement   │    │   │
│  │   │  mgmt    │  │          │  │  (EPT, VPID, etc.)    │    │   │
│  │   └──────────┘  └──────────┘  └───────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Hardware (CPU)                             │   │
│  │       VT-x (VMXON/VMLAUNCH/VMRESUME) or AMD-V (VMRUN)       │   │
│  │       EPT / NPT page tables in hardware                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Execution Flow Summary

1. Userspace opens `/dev/kvm`, creates a VM fd, then creates vCPU fds.
2. Memory regions are registered via `KVM_SET_USER_MEMORY_REGION`.
3. For each vCPU, the userspace thread calls `ioctl(vcpu_fd, KVM_RUN)`.
4. KVM loads guest state into hardware (VMCS/VMCB) and executes `VMLAUNCH`
   or `VMRESUME` (Intel) / `VMRUN` (AMD).
5. The CPU runs guest code in VMX non-root / guest mode.
6. On a VM exit, the CPU returns to KVM. KVM handles the exit or returns
   to userspace (via `kvm_run`).
7. The loop repeats.

---

## 3. Ioctl Interface and Device Model

KVM uses three levels of file descriptors, each with its own set of ioctls:

```
                  ┌──────────────────────┐
   open("/dev/kvm") → │  System fd (/dev/kvm)│
                  └──────────┬───────────┘
                             │ KVM_CREATE_VM
                             ▼
                  ┌──────────────────────┐
                  │     VM fd            │
                  └──────────┬───────────┘
                             │ KVM_CREATE_VCPU
                             ▼
                  ┌──────────────────────┐
                  │    vCPU fd           │
                  └──────────────────────┘
```

### 3.1 System-level ioctls (`/dev/kvm`)

| Ioctl                        | Purpose                                       |
|------------------------------|-----------------------------------------------|
| `KVM_GET_API_VERSION`        | Returns `KVM_API_VERSION` (currently 12)       |
| `KVM_CREATE_VM`              | Creates a new VM, returns a VM fd              |
| `KVM_CHECK_EXTENSION`        | Query capability support                       |
| `KVM_GET_VCPU_MMAP_SIZE`     | Size of `kvm_run` mmap region                  |

```
include/uapi/linux/kvm.h:17
  #define KVM_API_VERSION 12
```

### 3.2 VM-level ioctls

| Ioctl                           | Purpose                                      |
|---------------------------------|----------------------------------------------|
| `KVM_CREATE_VCPU`              | Create a vCPU, returns a vCPU fd              |
| `KVM_SET_USER_MEMORY_REGION`   | Map host memory into guest physical space     |
| `KVM_SET_USER_MEMORY_REGION2`  | Extended version with guest_memfd support     |
| `KVM_CREATE_IRQCHIP`          | Create in-kernel PIC/IOAPIC/LAPIC             |
| `KVM_SET_GSI_ROUTING`         | Configure IRQ routing table                   |
| `KVM_IRQFD`                   | Bind eventfd to guest IRQ line                |
| `KVM_IOEVENTFD`               | Bind eventfd to guest I/O address             |
| `KVM_SET_TSS_ADDR`            | Set Task State Segment address (x86)          |
| `KVM_SET_IDENTITY_MAP_ADDR`   | Set identity map page (x86)                   |
| `KVM_CREATE_PIT2`             | Create in-kernel i8254 PIT                    |
| `KVM_GET_DIRTY_LOG`           | Retrieve dirty page bitmap for live migration |

### 3.3 vCPU-level ioctls

| Ioctl                    | Purpose                                          |
|--------------------------|--------------------------------------------------|
| `KVM_RUN`                | Enter guest execution (blocks until VM exit)     |
| `KVM_GET_REGS`           | Read general-purpose registers                   |
| `KVM_SET_REGS`           | Write general-purpose registers                  |
| `KVM_GET_SREGS`          | Read segment / control registers                 |
| `KVM_SET_SREGS`          | Write segment / control registers                |
| `KVM_GET_MSRS`           | Read MSRs                                        |
| `KVM_SET_MSRS`           | Write MSRs                                       |
| `KVM_SET_CPUID2`         | Configure CPUID leaves exposed to guest          |
| `KVM_INTERRUPT`          | Inject an external interrupt                     |
| `KVM_NMI`                | Inject an NMI                                    |
| `KVM_SET_SIGNAL_MASK`    | Restrict signals during KVM_RUN                  |
| `KVM_GET_FPU`            | Read FPU state                                   |
| `KVM_SET_FPU`            | Write FPU state                                  |
| `KVM_SET_GUEST_DEBUG`    | Configure hardware guest debugging               |

Implementation entry points:
```
virt/kvm/kvm_main.c:5008   static long kvm_vm_ioctl(struct file *filp, ...)
virt/kvm/kvm_main.c:4285   static long kvm_vcpu_ioctl(struct file *filp, ...)
virt/kvm/kvm_main.c:5340   static int kvm_dev_ioctl_create_vm(unsigned long type)
```

---

## 4. Core Data Structures

### 4.1 `struct kvm` — The Virtual Machine

Represents an entire VM instance including all of its vCPUs, memory, and
devices. One `struct kvm` per VM.

```c
/* include/linux/kvm_host.h:744 */
struct kvm {
#ifdef KVM_HAVE_MMU_RWLOCK
    rwlock_t mmu_lock;
#else
    spinlock_t mmu_lock;
#endif

    struct mutex slots_lock;
    struct mutex slots_arch_lock;
    struct mm_struct *mm;                 /* userspace tied to this vm */
    unsigned long nr_memslot_pages;

    /* The two memslot sets - active and inactive (per address space) */
    struct kvm_memslots __memslots[KVM_MAX_NR_ADDRESS_SPACES][2];
    struct kvm_memslots __rcu *memslots[KVM_MAX_NR_ADDRESS_SPACES];

    struct xarray vcpu_array;
    atomic_t online_vcpus;
    int max_vcpus;
    int created_vcpus;
    int last_boosted_vcpu;
    struct list_head vm_list;
    struct mutex lock;

    struct kvm_io_bus __rcu *buses[KVM_NR_BUSES];

#ifdef CONFIG_HAVE_KVM_IRQCHIP
    struct {
        spinlock_t        lock;
        struct list_head  items;
        struct list_head  resampler_list;
        struct mutex      resampler_lock;
    } irqfds;
#endif

    struct list_head ioeventfds;
    struct kvm_vm_stat stat;
    struct kvm_arch arch;              /* architecture-specific state */
    refcount_t users_count;

#ifdef CONFIG_KVM_MMIO
    struct kvm_coalesced_mmio_ring *coalesced_mmio_ring;
    spinlock_t ring_lock;
    struct list_head coalesced_zones;
#endif

    struct mutex irq_lock;
#ifdef CONFIG_HAVE_KVM_IRQCHIP
    struct kvm_irq_routing_table __rcu *irq_routing;
    struct hlist_head irq_ack_notifier_list;
#endif
    /* ... debugfs, stats, dirty ring fields ... */
};
```

### 4.2 `struct kvm_vcpu` — The Virtual CPU

Each vCPU maps to one host thread. Created by `KVM_CREATE_VCPU`.

```c
/* include/linux/kvm_host.h:316 */
struct kvm_vcpu {
    struct kvm *kvm;
#ifdef CONFIG_PREEMPT_NOTIFIERS
    struct preempt_notifier preempt_notifier;
#endif
    int cpu;                           /* physical CPU this ran on last */
    int vcpu_id;                       /* id given by userspace at creation */
    int vcpu_idx;                      /* index into kvm->vcpu_array */
    int mode;                          /* OUTSIDE_GUEST_MODE, IN_GUEST_MODE, ... */
    u64 requests;                      /* pending request bitmask */
    unsigned long guest_debug;

    struct mutex mutex;
    struct kvm_run *run;               /* shared with userspace via mmap */

    struct rcuwait wait;
    struct pid *pid;
    rwlock_t pid_lock;
    int sigset_active;
    sigset_t sigset;
    unsigned int halt_poll_ns;         /* adaptive halt polling */
    bool valid_wakeup;

#ifdef CONFIG_HAS_IOMEM
    int mmio_needed;
    int mmio_read_completed;
    int mmio_is_write;
    int mmio_cur_fragment;
    int mmio_nr_fragments;
    struct kvm_mmio_fragment mmio_fragments[KVM_MAX_MMIO_FRAGMENTS];
#endif

    bool wants_to_run;
    bool preempted;
    bool ready;
    bool scheduled_out;
    struct kvm_vcpu_arch arch;         /* architecture-specific vCPU state */
    struct kvm_vcpu_stat stat;
    struct kvm_dirty_ring dirty_ring;
    struct kvm_memory_slot *last_used_slot;
    u64 last_used_slot_gen;
};
```

The `mode` field tracks vCPU state transitions:

```c
/* include/linux/kvm_host.h:270 */
enum {
    OUTSIDE_GUEST_MODE,
    IN_GUEST_MODE,
    EXITING_GUEST_MODE,
    READING_SHADOW_PAGE_TABLES,
};
```

### 4.3 `struct kvm_run` — Userspace Communication

The `kvm_run` structure is shared between kernel and userspace via `mmap()` on
the vCPU fd. When `KVM_RUN` returns, the `exit_reason` field tells userspace
why, and the union provides exit-specific data.

```c
/* include/uapi/linux/kvm.h:209 */
struct kvm_run {
    /* in */
    __u8 request_interrupt_window;
    __u8 immediate_exit;
    __u8 padding1[6];

    /* out */
    __u32 exit_reason;
    __u8 ready_for_interrupt_injection;
    __u8 if_flag;
    __u16 flags;

    /* in (pre_kvm_run), out (post_kvm_run) */
    __u64 cr8;
    __u64 apic_base;

    union {
        struct { __u64 hardware_exit_reason; } hw;          /* KVM_EXIT_UNKNOWN */
        struct { __u64 hardware_entry_failure_reason;       /* KVM_EXIT_FAIL_ENTRY */
                 __u32 cpu; } fail_entry;
        struct { __u8 direction; __u8 size; __u16 port;     /* KVM_EXIT_IO */
                 __u32 count; __u64 data_offset; } io;
        struct { __u64 phys_addr; __u8 data[8];             /* KVM_EXIT_MMIO */
                 __u32 len; __u8 is_write; } mmio;
        struct { __u64 nr; __u64 args[6]; __u64 ret;        /* KVM_EXIT_HYPERCALL */
                 __u64 flags; } hypercall;
        /* ... many more exit types ... */
    };
};
```

### 4.4 `struct kvm_memory_slot` — Guest Physical Memory

Each memory slot maps a range of guest physical addresses to a host virtual
address range.

```c
/* include/linux/kvm_host.h:584 */
struct kvm_memory_slot {
    struct hlist_node id_node[2];
    struct interval_tree_node hva_node[2];
    struct rb_node gfn_node[2];
    gfn_t base_gfn;                    /* base guest frame number */
    unsigned long npages;               /* number of pages */
    unsigned long *dirty_bitmap;        /* for dirty page tracking */
    struct kvm_arch_memory_slot arch;
    unsigned long userspace_addr;       /* host virtual address */
    u32 flags;
    short id;
    u16 as_id;                          /* address space id */

#ifdef CONFIG_KVM_PRIVATE_MEM
    struct {
        struct file __rcu *file;
        pgoff_t pgoff;
    } gmem;                             /* guest_memfd for CoCo VMs */
#endif
};
```

Userspace configures slots via:

```c
/* include/uapi/linux/kvm.h:25 */
struct kvm_userspace_memory_region {
    __u32 slot;
    __u32 flags;
    __u64 guest_phys_addr;
    __u64 memory_size;       /* bytes */
    __u64 userspace_addr;    /* start of the userspace allocated memory */
};

/* Flags */
#define KVM_MEM_LOG_DIRTY_PAGES  (1UL << 0)   /* enable dirty tracking */
#define KVM_MEM_READONLY         (1UL << 1)   /* read-only region */
#define KVM_MEM_GUEST_MEMFD      (1UL << 2)   /* private memory (CoCo) */
```

### 4.5 `struct kvm_io_bus` — I/O Device Dispatch

```c
/* include/linux/kvm_host.h:205 */
struct kvm_io_bus {
    int dev_count;
    int ioeventfd_count;
    struct kvm_io_range range[];       /* flexible array of registered devices */
};

enum kvm_bus {
    KVM_MMIO_BUS,
    KVM_PIO_BUS,
    KVM_VIRTIO_CCW_NOTIFY_BUS,
    KVM_FAST_MMIO_BUS,
    KVM_IOCSR_BUS,
    KVM_NR_BUSES
};
```

---

## 5. VM and vCPU Lifecycle

### 5.1 VM Creation

```
Userspace                              Kernel
─────────                              ──────
open("/dev/kvm")  ───────────────►  kvm_dev_open()
                                       returns system fd

ioctl(sys_fd,     ───────────────►  kvm_dev_ioctl_create_vm()
  KVM_CREATE_VM)                      kvm_create_vm()
                                        ├─ allocate struct kvm
                                        ├─ kvm_init_memslots()
                                        ├─ kvm_arch_init_vm()
                                        ├─ hardware_enable_all()
                                        └─ returns VM fd
```

```
virt/kvm/kvm_main.c:5340
  static int kvm_dev_ioctl_create_vm(unsigned long type)
```

### 5.2 vCPU Creation

```
ioctl(vm_fd,      ───────────────►  kvm_vm_ioctl_create_vcpu()      [kvm_main.c:4057]
  KVM_CREATE_VCPU, id)                ├─ kvm_arch_vcpu_precreate()
                                      ├─ kvm_arch_vcpu_create()
                                      │    ├─ allocate kvm_vcpu
                                      │    ├─ kvm_vcpu_init()
                                      │    ├─ kvm_x86_call(vcpu_create)(vcpu)
                                      │    │    └─ vmx_vcpu_create() or svm_vcpu_create()
                                      │    │        ├─ alloc VMCS/VMCB
                                      │    │        └─ initialize hardware state
                                      │    └─ setup LAPIC, MMU, etc.
                                      ├─ create_vcpu_fd()
                                      └─ store in kvm->vcpu_array
```

### 5.3 The KVM_RUN Loop

The heart of KVM. Each vCPU thread calls `KVM_RUN` in a loop:

```
Userspace                              Kernel
─────────                              ──────
while (1) {
  ioctl(vcpu_fd,  ───────────────►  kvm_vcpu_ioctl(KVM_RUN)
    KVM_RUN)                          kvm_arch_vcpu_ioctl_run()
                                        ├─ vcpu_load()
                                        │    └─ kvm_arch_vcpu_load()
                                        │
                                        ├─ ┌── vcpu_run() loop ──────────────┐
                                        │  │ for (;;) {                       │
                                        │  │   if (kvm_vcpu_running())        │
                                        │  │     r = vcpu_enter_guest()       │
                                        │  │       ├─ inject interrupts       │
                                        │  │       ├─ kvm_x86_call(vcpu_run)  │
                                        │  │       │    ├─ VMLAUNCH/VMRESUME  │
                                        │  │       │    │   (guest executes)  │
                                        │  │       │    └─ VM exit occurs     │
                                        │  │       ├─ handle_exit()           │
                                        │  │       └─ return to loop          │
                                        │  │   else                           │
                                        │  │     kvm_vcpu_block() / halt      │
                                        │  │   if (need userspace exit)       │
                                        │  │     break                        │
                                        │  │ }                                │
                                        │  └──────────────────────────────────┘
                                        │
                                        └─ vcpu_put()

  /* process kvm_run->exit_reason */
  switch (run->exit_reason) {
    case KVM_EXIT_IO:     ...
    case KVM_EXIT_MMIO:   ...
    case KVM_EXIT_HLT:    ...
  }
}
```

### 5.4 Adaptive Halt Polling

When a vCPU executes HLT (halt), KVM can either immediately block the host
thread or spin-poll for a short period hoping for a quick wakeup (interrupt):

```c
/* virt/kvm/kvm_main.c:78 */
unsigned int halt_poll_ns = KVM_HALT_POLL_NS_DEFAULT;

/* Default doubles per-vcpu halt_poll_ns. */
unsigned int halt_poll_ns_grow = 2;          /* line 83 */

/* The start value to grow halt_poll_ns from */
unsigned int halt_poll_ns_grow_start = 10000; /* 10us, line 88 */

/* Default halves per-vcpu halt_poll_ns. */
unsigned int halt_poll_ns_shrink = 2;        /* line 93 */
```

The algorithm: if a wakeup arrives during the poll window, double `halt_poll_ns`
(up to the max). If the poll expires without wakeup, halve it. This provides
low-latency wakeup for active workloads without wasting CPU when idle.

### 5.5 VM Destruction

When the last reference to the VM fd is dropped (all vCPU fds closed, VM fd
closed), `kvm_destroy_vm()` tears down all state — frees memory slots, destroys
vCPUs, releases hardware resources, and frees the `struct kvm`.

---

## 6. VMCS / VMCB — Hardware Virtualization State

### 6.1 Intel VMCS (Virtual Machine Control Structure)

The VMCS is a hardware-defined data structure that controls VM entry/exit
behavior on Intel processors. It resides in a special 4KB-aligned memory region
managed by the CPU via `VMREAD`/`VMWRITE` instructions.

```
┌─────────────────────────────────────────────────────────┐
│                     VMCS Layout                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Guest-State Area                                        │
│  ──────────────────                                      │
│  - CR0, CR3, CR4                                         │
│  - RSP, RIP, RFLAGS                                     │
│  - Segment registers (CS, DS, SS, ES, FS, GS, LDTR, TR) │
│  - GDTR, IDTR                                           │
│  - MSRs (IA32_EFER, IA32_PAT, etc.)                     │
│  - Activity state (active, HLT, shutdown, wait-for-SIPI) │
│  - Interruptibility state                                │
│  - Pending debug exceptions                              │
│                                                          │
│  Host-State Area                                         │
│  ────────────────                                        │
│  - CR0, CR3, CR4                                         │
│  - RSP, RIP (entry point after VM exit)                  │
│  - Segment selectors                                     │
│  - MSRs                                                  │
│                                                          │
│  VM-Execution Controls                                   │
│  ──────────────────────                                  │
│  - Pin-based (ext. interrupts, NMI, preemption timer)    │
│  - Primary processor-based (HLT, MWAIT, RDPMC, etc.)    │
│  - Secondary processor-based (EPT, VPID, RDTSCP, etc.)  │
│  - Exception bitmap (which exceptions cause VM exit)     │
│  - I/O bitmap addresses                                  │
│  - MSR bitmaps address                                   │
│  - EPT pointer (EPTP)                                    │
│  - Virtual APIC page address                             │
│                                                          │
│  VM-Exit Controls                                        │
│  ────────────────                                        │
│  - Acknowledge interrupt on exit                         │
│  - Save/load MSRs on exit                                │
│  - Host address-space size                               │
│                                                          │
│  VM-Entry Controls                                       │
│  ─────────────────                                       │
│  - Event injection (interrupt/exception/NMI)             │
│  - Load MSRs on entry                                    │
│  - Guest mode (IA-32e / protected / real)                │
│                                                          │
│  VM-Exit Information (read-only)                         │
│  ───────────────────────────────                         │
│  - Exit reason                                           │
│  - Exit qualification (details about exit cause)         │
│  - Guest-linear and guest-physical addresses             │
│  - Instruction length, instruction info                  │
│  - IDT/GDT vectoring information                         │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

Key source: `arch/x86/kvm/vmx/vmcs.h`, `arch/x86/kvm/vmx/vmx.c`

### 6.2 AMD VMCB (Virtual Machine Control Block)

The AMD equivalent is the VMCB, used with the `VMRUN` instruction. It has two
main areas:

- **Control Area** (offset 0x000): Intercepts, IOPM/MSRPM base addresses, TSC
  offset, guest ASID, TLB control, interrupt shadow, event injection, nested
  paging CR3 (nCR3), and more.
- **State Save Area** (offset 0x400): Guest register state — segment
  registers, CR0/CR2/CR3/CR4, RFLAGS, RIP, RSP, RAX, EFER, PAT, etc.

Key source: `arch/x86/kvm/svm/svm.h`, `arch/x86/kvm/svm/svm.c`

---

## 7. VM Exits

A VM exit transfers control from guest mode back to the hypervisor (KVM). KVM
must determine the reason and either handle it internally or return to
userspace.

### 7.1 Common VM Exit Reasons

| Exit Reason                    | Typical Handling                              |
|-------------------------------|-----------------------------------------------|
| External interrupt             | Return to host interrupt handler               |
| HLT                           | `kvm_emulate_halt()` — block or halt-poll      |
| I/O (IN/OUT)                  | In-kernel device or exit to userspace           |
| MMIO                          | In-kernel device or exit to userspace           |
| EPT violation / NPT fault     | KVM MMU handles page fault                     |
| CR access                     | Emulate CR read/write                           |
| MSR access                    | Handle in-kernel or filter via MSR bitmap       |
| CPUID                         | `kvm_emulate_cpuid()`                           |
| WRMSR/RDMSR                   | MSR emulation                                  |
| Exception (e.g., #PF, #GP)    | Inject into guest or handle internally          |
| Preemption timer               | Preempt guest for scheduler fairness            |
| PAUSE                         | Pause-loop exiting — yield to other vCPUs       |
| XSETBV                        | Emulate XCR0 write                              |
| INVLPG                        | Invalidate shadow/EPT TLB entry                 |
| Task switch                   | Emulate hardware task switch                    |

### 7.2 Exit Reason Codes (Userspace-visible)

```c
/* include/uapi/linux/kvm.h:141 */
#define KVM_EXIT_UNKNOWN          0
#define KVM_EXIT_EXCEPTION        1
#define KVM_EXIT_IO               2
#define KVM_EXIT_HYPERCALL        3
#define KVM_EXIT_DEBUG            4
#define KVM_EXIT_HLT              5
#define KVM_EXIT_MMIO             6
#define KVM_EXIT_IRQ_WINDOW_OPEN  7
#define KVM_EXIT_SHUTDOWN         8
#define KVM_EXIT_FAIL_ENTRY       9
#define KVM_EXIT_INTR             10
#define KVM_EXIT_NMI              16
#define KVM_EXIT_INTERNAL_ERROR   17
#define KVM_EXIT_HYPERV           27
#define KVM_EXIT_X86_RDMSR        29
#define KVM_EXIT_X86_WRMSR        30
#define KVM_EXIT_DIRTY_RING_FULL  31
#define KVM_EXIT_MEMORY_FAULT     39
/* ... and more ... */
```

### 7.3 Exit Handling Flow

```
VM Exit occurs
    │
    ▼
vmx_vmexit() / svm_vmexit()
    │
    ├─ Save guest state from VMCS/VMCB
    ├─ Restore host state
    │
    ▼
vmx_handle_exit() / svm_handle_exit()
    │
    ├─ Read exit reason from VMCS/VMCB
    ├─ Dispatch via exit handler table:
    │     kvm_vmx_exit_handlers[] / svm_exit_handlers[]
    │
    ├─ [Fast path] Handle entirely in kernel:
    │     ├─ EPT fault → kvm_mmu_page_fault()
    │     ├─ CPUID    → kvm_emulate_cpuid()
    │     ├─ MSR      → kvm_emulate_rdmsr/wrmsr()
    │     ├─ I/O      → kvm_fast_pio() or kernel device
    │     └─ return 1 (re-enter guest)
    │
    └─ [Slow path] Need userspace:
          ├─ Fill in kvm_run fields
          ├─ return 0 (exit to userspace)
          └─ Userspace processes the exit
```

---

## 8. Memory Virtualization (EPT/NPT)

### 8.1 Two-Level Address Translation

Hardware-assisted memory virtualization (Intel EPT = Extended Page Tables,
AMD NPT = Nested Page Tables) provides a second level of address translation
in hardware, eliminating the need for shadow page tables.

```
┌─────────────────────────────────────────────────────────────────┐
│                Two-Level Address Translation                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Guest Virtual Addr (GVA)                                        │
│         │                                                        │
│         │  Guest Page Tables (CR3)                                │
│         │  (controlled by guest OS)                               │
│         ▼                                                        │
│  Guest Physical Addr (GPA)                                       │
│         │                                                        │
│         │  EPT / NPT Page Tables (EPTP)                          │
│         │  (controlled by KVM)                                    │
│         ▼                                                        │
│  Host Physical Addr (HPA)                                        │
│                                                                  │
│  Without EPT/NPT (Shadow Page Tables):                          │
│  ──────────────────────────────────────                          │
│  GVA ──► GPA (guest PT) ──► HPA  →  collapsed into              │
│  GVA ──────────────────────► HPA     shadow page table           │
│                                                                  │
│  Shadow PTs require VM exits on every guest PT modification.     │
│  EPT/NPT remove this overhead entirely.                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 EPT/NPT Page Table Structure

EPT uses the same 4-level (or 5-level) page table format as regular x86 paging:

```
EPT PML4 (512 entries)
    │
    └─► EPT PDPT (512 entries)
            │
            └─► EPT PD (512 entries)
                    │
                    ├─► EPT PT (512 entries, 4KB pages)
                    └─► 2MB large page (direct mapping)
```

Each EPT PTE contains:
- Host physical address of the mapped page
- Read/Write/Execute permissions
- Memory type (UC, WB, WC, etc.)
- Accessed/Dirty bits (if supported)

### 8.3 Memory Slot Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              Memory Slot Architecture                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Userspace (QEMU)                                              │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  mmap'd anonymous memory:                              │   │
│  │                                                        │   │
│  │  slot 0: RAM         0x0000_0000 - 0x0009_FFFF (640K)  │   │
│  │  slot 1: RAM         0x0010_0000 - 0x7FFF_FFFF (2G-1M) │   │
│  │  slot 2: VGA FB      0x000A_0000 - 0x000B_FFFF         │   │
│  │  slot 3: ROM (RO)    0xFFFC_0000 - 0xFFFF_FFFF (BIOS)  │   │
│  └────────────────────────────────────────────────────────┘   │
│        │                                                       │
│        │  KVM_SET_USER_MEMORY_REGION                           │
│        ▼                                                       │
│  Kernel (KVM)                                                  │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  struct kvm_memory_slot[]                               │   │
│  │    base_gfn ──► npages ──► userspace_addr               │   │
│  │                                                         │   │
│  │  EPT/NPT page tables                                   │   │
│  │    GPA → HPA mappings (populated on demand / faults)    │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 8.4 EPT Violation Handling

When the guest accesses a GPA that has no EPT mapping (or violates permissions),
an EPT violation VM exit occurs:

1. `vmx_handle_exit()` dispatches to `handle_ept_violation()`
2. Extract the faulting GPA and access type from VMCS
3. Call `kvm_mmu_page_fault()`
4. Look up the `kvm_memory_slot` for the GPA
5. Translate GPA to HPA via `gfn_to_pfn()` (uses host page tables)
6. Install the EPT mapping
7. Return to guest (or exit to userspace for MMIO)

### 8.5 MMIO Handling via EPT

Accesses to unmapped GPAs (no memory slot) are treated as MMIO. KVM:
1. Marks the access as MMIO in the EPT (using reserved bits as a "MMIO spte")
2. On subsequent accesses, detects the MMIO marker
3. Emulates the instruction and dispatches to the I/O bus or exits to userspace

### 8.6 Dirty Page Tracking

For live migration, KVM tracks dirty pages via:

- **Dirty bitmap**: `KVM_GET_DIRTY_LOG` returns a bitmap of dirty pages per slot
- **Dirty ring**: `KVM_DIRTY_LOG_PAGE_OFFSET` — per-vCPU ring buffer of dirty
  page records, avoiding bitmap scanning overhead
- **PML (Page Modification Logging)**: Intel hardware feature that logs dirty
  GPAs automatically, reducing VM exits

### 8.7 MMU Notifiers

KVM integrates with the Linux MMU notifier framework to stay synchronized with
host memory management operations (e.g., page migration, swap, KSM):

```c
/* include/linux/kvm_host.h:258 */
struct kvm_gfn_range {
    struct kvm_memory_slot *slot;
    gfn_t start;
    gfn_t end;
    union kvm_mmu_notifier_arg arg;
    bool may_block;
};
```

When the host unmaps or changes a page, the MMU notifier calls
`kvm_unmap_gfn_range()`, which zaps the corresponding EPT/shadow entries,
forcing re-faults that will pick up the new mapping.

---

## 9. Interrupt and Exception Injection

### 9.1 Interrupt Delivery Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                Interrupt Delivery Path                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Userspace (QEMU)                                                │
│  ┌────────┐    KVM_IRQFD        ┌──────────┐                   │
│  │ Device  │ ─────eventfd──────►│ irqfd in │                   │
│  │ Model   │                    │ KVM core │                   │
│  └────────┘                     └────┬─────┘                   │
│                                      │                          │
│  ──────────────────────────────────── │ ────────────────────── │
│                                      ▼        Kernel            │
│                               ┌──────────────┐                  │
│  ┌─────────┐    ┌──────────┐ │   KVM IRQ    │                  │
│  │ In-kernel│───►│  IOAPIC  │─┤   Routing    │                  │
│  │ devices  │    │          │ │   Table      │                  │
│  │ (PIT,etc)│    └──────────┘ └──────┬───────┘                  │
│  └─────────┘                         │                          │
│                                      ▼                          │
│                               ┌──────────────┐                  │
│                               │   LAPIC      │                  │
│                               │  (in-kernel) │                  │
│                               │  per-vCPU    │                  │
│                               └──────┬───────┘                  │
│                                      │                          │
│                                      ▼                          │
│                               ┌──────────────────┐              │
│                               │ Inject via VMCS/ │              │
│                               │ VMCB entry field │              │
│                               │ on next VM entry │              │
│                               └──────────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Injection Mechanisms

**Event injection via VMCS/VMCB**: Before VM entry, KVM writes the interrupt
vector and type into the VM-Entry Interruption-Information field (Intel) or
the EVENTINJ field (AMD). The CPU delivers the interrupt upon VM entry as if
it were a real hardware interrupt.

**Virtual interrupt delivery (Intel APICv / AMD AVIC)**: Hardware can deliver
interrupts directly to the guest without a VM exit:
- Posted interrupts: KVM writes to a Posted Interrupt Descriptor (PID);
  hardware picks up the interrupt and delivers it on the next opportunity
- Virtual APIC page: Hardware maintains a virtual APIC page with registers
  that the guest can read/write without VM exits
- AVIC (AMD Virtual Interrupt Controller): Similar concept for AMD

### 9.3 Interrupt Windows

If the guest has interrupts disabled (IF=0), KVM cannot inject immediately.
Instead, it sets the "interrupt window exiting" VMCS control. The CPU will
cause a VM exit as soon as the guest enables interrupts, allowing KVM to
inject the pending interrupt.

### 9.4 IRQ Source IDs

```c
/* include/linux/kvm_host.h:191 */
#define KVM_USERSPACE_IRQ_SOURCE_ID          0
#define KVM_IRQFD_RESAMPLE_IRQ_SOURCE_ID     1
```

### 9.5 Exception Injection

KVM can inject exceptions (e.g., #PF, #GP, #UD) into the guest via the same
VMCS/VMCB event injection mechanism. This is used for:
- Re-injecting exceptions that KVM intercepted but wants the guest to handle
- Injecting #PF for async page faults (PV feature)
- Injecting #UD for unsupported instructions

---

## 10. I/O Handling

### 10.1 PIO (Port I/O)

When the guest executes an `IN` or `OUT` instruction and the I/O bitmap causes
a VM exit:

1. KVM reads the port number, size, direction, and data from the VMCS/VMCB
2. Tries in-kernel I/O bus: `kvm_io_bus_write(vcpu, KVM_PIO_BUS, ...)`
3. If no in-kernel device handles it, fills `kvm_run->io` and exits to
   userspace with `KVM_EXIT_IO`
4. Userspace emulates the device and re-enters via `KVM_RUN`

### 10.2 MMIO (Memory-Mapped I/O)

When the guest accesses a GPA with no memory slot (or marked MMIO):

1. EPT violation or instruction emulation triggers MMIO path
2. KVM decodes the faulting instruction to determine address, size, direction
3. Tries in-kernel I/O bus: `kvm_io_bus_write(vcpu, KVM_MMIO_BUS, ...)`
4. If unhandled, fills `kvm_run->mmio` and exits with `KVM_EXIT_MMIO`

```c
/* From kvm_run union — include/uapi/linux/kvm.h:259 */
struct {
    __u64 phys_addr;
    __u8  data[8];
    __u32 len;
    __u8  is_write;
} mmio;
```

### 10.3 ioeventfd

`KVM_IOEVENTFD` allows userspace to bind an `eventfd` to a specific guest
I/O address. When the guest writes to that address, KVM signals the eventfd
without exiting to userspace — enabling fast I/O notification for virtio
and similar frameworks.

### 10.4 Coalesced MMIO

For high-frequency writes to device registers (e.g., VGA framebuffer), KVM
can buffer writes in a shared ring (`kvm_coalesced_mmio_ring`) and deliver
them in batch, reducing VM exit overhead.

### 10.5 In-Kernel Devices

KVM can emulate several devices entirely in-kernel for performance:

| Device      | Purpose                           | Created via              |
|------------|-----------------------------------|--------------------------|
| Local APIC  | Per-CPU interrupt controller      | `KVM_CREATE_IRQCHIP`     |
| IOAPIC      | I/O interrupt routing             | `KVM_CREATE_IRQCHIP`     |
| PIC (i8259) | Legacy interrupt controller       | `KVM_CREATE_IRQCHIP`     |
| PIT (i8254) | Programmable interval timer       | `KVM_CREATE_PIT2`        |
| kvmclock    | Paravirtual clock source          | Automatic with PV        |

---

## 11. Hardware Backends: VMX and SVM

KVM uses a `kvm_x86_ops` function pointer table to abstract the hardware
backend. The VMX module (`kvm-intel.ko`) and SVM module (`kvm-amd.ko`) each
register their implementation.

### 11.1 Intel VMX (Virtual Machine Extensions)

**Key instructions**:
- `VMXON` — Enter VMX operation (root mode)
- `VMXOFF` — Leave VMX operation
- `VMLAUNCH` — Launch VM for the first time (current VMCS)
- `VMRESUME` — Resume VM (after first launch)
- `VMREAD` / `VMWRITE` — Read/write VMCS fields
- `VMPTRLD` — Load VMCS pointer
- `VMCLEAR` — Clear/deactivate a VMCS
- `INVEPT` — Invalidate EPT-based TLB entries
- `INVVPID` — Invalidate VPID-tagged TLB entries

**Key files**:
```
arch/x86/kvm/vmx/vmx.c        — Main VMX module, entry/exit handling
arch/x86/kvm/vmx/vmx.h        — VMX data structures
arch/x86/kvm/vmx/vmcs.h       — VMCS field definitions
arch/x86/kvm/vmx/vmenter.S    — Assembly VM entry/exit stubs
arch/x86/kvm/vmx/capabilities.h — Feature detection
arch/x86/kvm/vmx/nested.c     — Nested VMX (L1 hypervisor support)
arch/x86/kvm/vmx/posted_intr.c — Posted interrupt support
arch/x86/kvm/vmx/pmu_intel.c  — Virtual PMU
```

**Key VMX features**:
- **VPID (Virtual Processor ID)**: Tags TLB entries with a per-vCPU ID,
  avoiding full TLB flush on VM entry/exit
- **EPT**: Hardware two-level page table translation
- **Unrestricted guest**: Run real-mode and unpaged guests directly in VMX
  non-root, without emulation
- **APICv**: Hardware-assisted APIC virtualization
- **PML (Page Modification Logging)**: Automatic dirty page tracking
- **Preemption timer**: VMCS-based timer that causes VM exit after N TSC ticks
- **MSR bitmaps**: Selectively intercept MSR accesses
- **I/O bitmaps**: Selectively intercept port I/O

### 11.2 AMD SVM (Secure Virtual Machine)

**Key instructions**:
- `VMRUN` — Run the guest (loads VMCB, enters guest mode)
- `VMSAVE` — Save additional host state
- `VMLOAD` — Restore additional host state
- `STGI` / `CLGI` — Set/Clear GIF (Global Interrupt Flag)
- `INVLPGA` — Invalidate TLB entry by address and ASID

**Key files**:
```
arch/x86/kvm/svm/svm.c        — Main SVM module
arch/x86/kvm/svm/svm.h        — SVM data structures, VMCB definition
arch/x86/kvm/svm/vmenter.S    — Assembly VM entry/exit
arch/x86/kvm/svm/nested.c     — Nested SVM
arch/x86/kvm/svm/avic.c       — AMD AVIC (virtual interrupt controller)
arch/x86/kvm/svm/sev.c        — AMD SEV (Secure Encrypted Virtualization)
```

**Key SVM features**:
- **NPT (Nested Page Tables)**: AMD's equivalent of EPT
- **ASID**: Address Space Identifier for TLB tagging
- **AVIC**: Hardware-accelerated interrupt delivery (AMD's APICv)
- **Decode assist**: Hardware provides decoded instruction info on intercepts
- **VMCB Clean bits**: Skip reloading unchanged VMCB fields for fast VMRUN
- **vGIF (Virtual GIF)**: Virtualizes GIF without intercepts
- **LBR virtualization**: Last Branch Record support for guests
- **SEV / SEV-ES / SEV-SNP**: Memory encryption for confidential VMs

### 11.3 `kvm_x86_ops` Interface

The x86 architecture code provides a function table that abstracts the
hardware differences. Key callbacks include:

| Callback                   | Purpose                                    |
|---------------------------|--------------------------------------------|
| `vcpu_create`             | Allocate and initialize hardware vCPU state |
| `vcpu_free`               | Free hardware vCPU state                    |
| `vcpu_load`               | Load vCPU state onto physical CPU           |
| `vcpu_put`                | Unload vCPU state from physical CPU         |
| `vcpu_run`                | Enter guest mode (VMLAUNCH/VMRESUME/VMRUN) |
| `handle_exit`             | Process VM exit reason                      |
| `set_cr0` / `set_cr3` / `set_cr4` | CR register write emulation         |
| `get_cpl`                 | Get current privilege level                 |
| `inject_irq`              | Inject interrupt into guest                 |
| `inject_nmi`              | Inject NMI into guest                       |
| `queue_exception`         | Queue exception for injection               |
| `set_msr` / `get_msr`    | MSR emulation                               |
| `tlb_flush_all`           | Flush all TLB entries                       |
| `tlb_flush_current`       | Flush current ASID/VPID TLB entries         |

---

## 12. Paravirtual (PV) Features

While KVM runs unmodified guests, performance improves significantly when
the guest cooperates. Linux guests include KVM PV drivers:

### 12.1 kvmclock

A paravirtual clock source that avoids the overhead and inaccuracy of
emulating hardware timers (PIT, HPET, TSC with unstable TSC). The hypervisor
shares a `pvclock_vcpu_time_info` structure with the guest via an MSR-specified
memory page, providing a precise time reference.

**MSRs**: `MSR_KVM_WALL_CLOCK_NEW`, `MSR_KVM_SYSTEM_TIME_NEW`

### 12.2 PV Spinlocks

Guest spinlock contention causes "lock holder preemption" problems. KVM PV
spinlocks use:
- `KVM_HC_KICK_CPU` hypercall to wake a specific vCPU
- `HLT`-based wait instead of spinning when the lock holder is preempted

### 12.3 PV TLB Flush

Instead of sending IPIs for TLB flushes (which cause VM exits on every target
vCPU), the guest can request KVM to perform the flush via a hypercall or
shared memory flag.

### 12.4 Async Page Fault

When the host needs to swap in a page for the guest, a normal page fault
would block the entire vCPU. KVM's async page fault mechanism:

1. KVM injects a special #PF into the guest with a "not present" error code
2. The guest PV handler schedules another process while waiting
3. When the page is ready, KVM injects a "page ready" notification
4. The guest reschedules the waiting process

```c
/* include/linux/kvm_host.h:234 */
struct kvm_async_pf {
    struct work_struct work;
    struct list_head link;
    struct list_head queue;
    struct kvm_vcpu *vcpu;
    gpa_t cr2_or_gpa;
    unsigned long addr;
    struct kvm_arch_async_pf arch;
    bool   wakeup_all;
    bool notpresent_injected;
};
```

### 12.5 PV EOI (End of Interrupt)

Avoids a VM exit on EOI write to the LAPIC by using a shared memory flag.
The guest sets a flag before writing EOI; if the flag is set, KVM knows no
exit is needed because there is no level-triggered interrupt to re-evaluate.

### 12.6 Steal Time

KVM reports to the guest how much CPU time was "stolen" (i.e., the vCPU was
runnable but the host scheduler ran something else). This allows the guest
scheduler to make better decisions and avoid penalizing processes for stolen
time.

### 12.7 PV IPI

Instead of sending Inter-Processor Interrupts via the LAPIC (which requires
multiple MMIO exits), the guest can use a `KVM_HC_SEND_IPI` hypercall to
send IPIs to multiple vCPUs in a single VM exit.

---

## 13. Nested Virtualization

Nested virtualization allows running a hypervisor inside a KVM guest (L1 guest
running an L2 guest). This is essential for:
- Testing hypervisors in VMs
- Cloud environments where tenants run their own hypervisors
- Recursive virtualization (e.g., WSL2 inside a VM)

### 13.1 Architecture

```
┌────────────────────────────────────────────────────────────┐
│  L0: Host KVM (bare metal)                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  L1: Guest hypervisor (e.g., KVM in a VM)            │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │  L2: Nested guest (OS running inside L1's VM)  │  │  │
│  │  │                                                │  │  │
│  │  │  VMLAUNCH/VMRUN by L1 is intercepted by L0     │  │  │
│  │  │  L0 merges L1's VMCS with its own controls     │  │  │
│  │  │  L2 actually runs under L0's direct control    │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### 13.2 VMCS Shadowing and Merging

When L1 executes VMLAUNCH/VMRESUME:

1. L0 intercepts the instruction (it causes a VM exit from L1)
2. L0 reads L1's VMCS (the "VMCS12" — L1's description of L2)
3. L0 creates a merged "VMCS02" that combines:
   - Guest state from VMCS12 (L2's register state)
   - Host state from L0's own state
   - Execution controls merged from VMCS12 and VMCS01 (OR of intercepts)
4. L0 runs L2 using VMCS02

On L2 VM exit:
- If the exit reason is in L1's intercept list, L0 reflects it to L1
  (synthetic VM exit into L1)
- If only L0 needs to handle it (e.g., EPT violation on L0's pages),
  L0 handles it directly and re-enters L2

### 13.3 Nested Memory Virtualization

With nested EPT, three levels of translation exist:

```
L2 GVA → L2 GPA   (L2's page tables, controlled by L2 guest OS)
L2 GPA → L1 GPA   (L1's EPT, controlled by L1 hypervisor)
L1 GPA → L0 HPA   (L0's EPT, controlled by L0/KVM)
```

KVM can either:
- **Shadow L1's EPT**: Build a combined "shadow EPT" that maps L2 GPA directly
  to HPA. Faster execution but more VM exits to maintain consistency.
- **Use nested EPT (if hardware supports it)**: Not currently available in
  hardware — all implementations use shadow EPT.

### 13.4 Source Files

```
arch/x86/kvm/vmx/nested.c    — Nested VMX implementation
arch/x86/kvm/svm/nested.c    — Nested SVM implementation
```

---

## 14. Key Source Files

### 14.1 Architecture-Independent (Common)

| File                              | Purpose                                     |
|----------------------------------|---------------------------------------------|
| `virt/kvm/kvm_main.c`           | Core KVM module: VM/vCPU lifecycle, ioctls  |
| `virt/kvm/eventfd.c`            | irqfd and ioeventfd implementation           |
| `virt/kvm/async_pf.c`           | Asynchronous page fault framework            |
| `virt/kvm/coalesced_mmio.c`     | Coalesced MMIO ring buffer                   |
| `virt/kvm/vfio.c`               | VFIO integration for device passthrough      |
| `virt/kvm/binary_stats.c`       | Binary statistics export                     |
| `virt/kvm/guest_memfd.c`        | Private memory for confidential VMs          |
| `include/linux/kvm_host.h`      | Core KVM data structures                     |
| `include/uapi/linux/kvm.h`      | Userspace API (ioctls, structs)              |
| `include/linux/kvm_types.h`     | KVM type definitions (gfn_t, gpa_t, etc.)   |

### 14.2 x86 Architecture-Specific

| File                              | Purpose                                     |
|----------------------------------|---------------------------------------------|
| `arch/x86/kvm/x86.c`            | x86 common code (ioctls, emulation, MSRs)  |
| `arch/x86/kvm/mmu/mmu.c`        | MMU — shadow & EPT page table management    |
| `arch/x86/kvm/mmu/tdp_mmu.c`    | TDP (Two-Dimensional Paging) MMU            |
| `arch/x86/kvm/mmu/spte.c`       | Shadow PTE manipulation                      |
| `arch/x86/kvm/mmu/page_track.c` | Page write tracking                          |
| `arch/x86/kvm/lapic.c`          | In-kernel Local APIC emulation               |
| `arch/x86/kvm/ioapic.c`         | In-kernel IOAPIC emulation                   |
| `arch/x86/kvm/i8254.c`          | In-kernel PIT emulation                      |
| `arch/x86/kvm/irq.c`            | IRQ handling and injection                   |
| `arch/x86/kvm/cpuid.c`          | CPUID emulation and filtering                |
| `arch/x86/kvm/emulate.c`        | x86 instruction emulator                     |
| `arch/x86/kvm/pmu.c`            | Virtual Performance Monitoring Unit          |
| `arch/x86/kvm/hyperv.c`         | Hyper-V enlightenments emulation             |
| `arch/x86/kvm/xen.c`            | Xen HVM emulation                            |
| `arch/x86/kvm/smm.c`            | System Management Mode                       |

### 14.3 Intel VMX Backend

| File                                | Purpose                                   |
|-------------------------------------|-------------------------------------------|
| `arch/x86/kvm/vmx/vmx.c`           | VMX core: entry/exit, VMCS management     |
| `arch/x86/kvm/vmx/vmx.h`           | VMX data structures                       |
| `arch/x86/kvm/vmx/vmcs.h`          | VMCS field encodings                      |
| `arch/x86/kvm/vmx/vmenter.S`       | Assembly VM entry/exit routines           |
| `arch/x86/kvm/vmx/nested.c`        | Nested VMX                                |
| `arch/x86/kvm/vmx/posted_intr.c`   | Posted interrupts / APICv                 |
| `arch/x86/kvm/vmx/capabilities.h`  | VMX capability detection                  |
| `arch/x86/kvm/vmx/pmu_intel.c`     | Intel vPMU                                |

### 14.4 AMD SVM Backend

| File                              | Purpose                                     |
|----------------------------------|---------------------------------------------|
| `arch/x86/kvm/svm/svm.c`        | SVM core: entry/exit, VMCB management       |
| `arch/x86/kvm/svm/svm.h`        | SVM data structures, VMCB layout            |
| `arch/x86/kvm/svm/vmenter.S`    | Assembly VM entry/exit routines             |
| `arch/x86/kvm/svm/nested.c`     | Nested SVM                                  |
| `arch/x86/kvm/svm/avic.c`       | AVIC (virtual interrupt controller)         |
| `arch/x86/kvm/svm/sev.c`        | SEV / SEV-ES / SEV-SNP encrypted VMs        |

### 14.5 Key Functions Reference

| Function                             | File                    | Line  | Purpose                              |
|--------------------------------------|-------------------------|-------|--------------------------------------|
| `kvm_dev_ioctl_create_vm()`         | `virt/kvm/kvm_main.c`  | 5340  | Create a new VM                      |
| `kvm_vm_ioctl_create_vcpu()`        | `virt/kvm/kvm_main.c`  | 4057  | Create a new vCPU                    |
| `kvm_vcpu_ioctl()`                  | `virt/kvm/kvm_main.c`  | 4285  | vCPU ioctl dispatcher                |
| `kvm_vm_ioctl()`                    | `virt/kvm/kvm_main.c`  | 5008  | VM ioctl dispatcher                  |
| `kvm_io_bus_write()`               | `virt/kvm/kvm_main.c`  | —     | Write to in-kernel I/O device        |
| `kvm_io_bus_read()`                | `virt/kvm/kvm_main.c`  | —     | Read from in-kernel I/O device       |
| `kvm_arch_vcpu_ioctl_run()`        | `arch/x86/kvm/x86.c`   | —     | x86 KVM_RUN entry point              |
| `vcpu_enter_guest()`               | `arch/x86/kvm/x86.c`   | —     | Main guest entry function            |
| `kvm_mmu_page_fault()`             | `arch/x86/kvm/mmu/`    | —     | Handle guest page fault / EPT fault  |
| `kvm_emulate_cpuid()`              | `arch/x86/kvm/cpuid.c` | —     | Handle CPUID VM exit                 |

---

## Summary

KVM transforms Linux into a high-performance hypervisor by combining:

1. **Hardware virtualization extensions** (VT-x/AMD-V) for efficient guest
   execution with minimal overhead
2. **Linux kernel reuse** — scheduling, memory management, and drivers come
   "for free"
3. **A clean ioctl-based interface** (`/dev/kvm`) that separates privileged
   operations (kernel) from device emulation (userspace)
4. **EPT/NPT hardware page tables** for near-native memory performance
5. **Paravirtual features** (kvmclock, PV spinlocks, async PF) for workloads
   that can cooperate with the hypervisor
6. **Nested virtualization** for running hypervisors inside VMs
7. **Confidential computing support** (SEV, TDX, guest_memfd) for encrypted
   VM memory

The architecture's modularity — common code in `virt/kvm/`, x86-specific code
in `arch/x86/kvm/`, and pluggable hardware backends (VMX/SVM) — makes KVM both
maintainable and extensible across multiple architectures (x86, ARM, RISC-V,
s390, MIPS, LoongArch).
