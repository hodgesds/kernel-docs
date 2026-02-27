# kernel-docs

Extended documentation for various Linux kernel subsystems and APIs. These docs are designed to be consumable by both humans and LLMs for quickly getting context on kernel internals.

## Overview

| Topic | Description |
|-------|-------------|
| [Audit](overview/audit.md) | Syscall auditing, filter rules, file watches, and SELinux integration |
| [Block Layer](overview/block_layer.md) | Linux block I/O layer, request queues, and block device drivers |
| [BPF (eBPF)](overview/bpf.md) | BPF subsystem, program types, maps, and the verifier |
| [Cgroups](overview/cgroups.md) | Control groups v1/v2, resource controllers, and hierarchy management |
| [CPU Hotplug](overview/cpu_hotplug.md) | CPU hotplug state machine, bring-up/tear-down, and callbacks |
| [CPUFreq / CPUIdle](overview/cpufreq_cpuidle.md) | CPU frequency scaling, idle states, governors, and schedutil |
| [Cryptographic Subsystem](overview/crypto.md) | Crypto API, ciphers, hashes, and cryptographic transformations |
| [Device Model](overview/device_model.md) | Buses, drivers, devices, and sysfs |
| [DMA Mapping](overview/dma_mapping.md) | DMA address translation, streaming/coherent mappings, and SWIOTLB |
| [VFS / Filesystems](overview/fs.md) | Virtual File System, inodes, dentries, and filesystem operations |
| [IOMMU](overview/iommu.md) | DMA remapping, device isolation, IOMMUFD, and SVA |
| [Interrupts](overview/interrupts.md) | Hardware interrupts, IRQ handling, and softirqs |
| [io_uring](overview/io_uring.md) | High-performance asynchronous I/O via shared-memory ring buffers |
| [IPC](overview/ipc.md) | Inter-process communication: pipes, signals, shared memory, and message queues |
| [Jump Labels / Static Calls](overview/jump_labels.md) | Zero-overhead branch/call primitives and text patching |
| [Kernel Entry/Exit](overview/kernel_entry_exit.md) | Syscall entry, IDT exceptions, pt_regs, KPTI, and Spectre mitigations |
| [Kprobes / Uprobes](overview/kprobes_uprobes.md) | Dynamic kernel and userspace probes, optprobes, and BPF attachment |
| [KVM](overview/kvm.md) | Kernel-based Virtual Machine, vCPU lifecycle, EPT/NPT, and VM exits |
| [Live Patching](overview/livepatch.md) | Runtime function replacement via ftrace, consistency model |
| [Lockdep](overview/lockdep.md) | Lock dependency validator, deadlock detection, and IRQ state tracking |
| [Locking & Synchronization](overview/locking.md) | Spinlocks, mutexes, RW locks, and synchronization primitives |
| [Memory Management](overview/memory_management.md) | Page allocator, slab, vmalloc, page tables, and virtual memory |
| [Module Loading](overview/modules.md) | Module lifecycle, ELF parsing, symbol resolution, and signing |
| [Namespaces](overview/namespaces.md) | 8 namespace types, nsproxy, and container isolation primitives |
| [Netfilter](overview/netfilter.md) | Netfilter hooks, nf_tables, connection tracking, and NAT |
| [Networking](overview/networking.md) | Network stack, sk_buff, sockets, and protocol handling |
| [Perf Events](overview/perf_event.md) | Performance counters, hardware/software events, and profiling |
| [Power Management](overview/power_management.md) | Suspend/resume, runtime PM, and power domains |
| [printk / Logging](overview/printk.md) | Lockless ring buffer, console drivers, and dynamic debug |
| [Process Lifecycle](overview/process_lifecycle.md) | Process creation, scheduling states, and termination |
| [Ptrace](overview/ptrace.md) | Process tracing, stop states, syscall tracing, and YAMA LSM |
| [RCU](overview/rcu.md) | Read-Copy-Update synchronization mechanism |
| [Scheduler](overview/sched.md) | CFS, scheduling classes, runqueues, and sched_ext |
| [Seccomp](overview/seccomp.md) | Syscall filtering, BPF programs, and user notification |
| [Security / LSM](overview/security_lsm.md) | Linux Security Modules, hooks, and access control frameworks |
| [Signals](overview/signals.md) | Signal delivery, handlers, masks, signalfd, and job control |
| [SMP Topology](overview/smp_topology.md) | CPU topology, per-CPU data, IPIs, NUMA, and scheduling domains |
| [Timers](overview/timers.md) | Timer subsystem, hrtimers, and time management |
| [Tracing](overview/tracing.md) | Ftrace, tracepoints, kprobes, and kernel tracing infrastructure |
| [Virtio](overview/virtio.md) | Virtqueues, vring, transport backends, and vhost acceleration |
