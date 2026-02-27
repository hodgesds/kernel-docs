# kernel-docs

Extended documentation for various Linux kernel subsystems and APIs. These docs are designed to be consumable by both humans and LLMs for quickly getting context on kernel internals.

## Overview

| Topic | Description |
|-------|-------------|
| [Block Layer](overview/block_layer.md) | Linux block I/O layer, request queues, and block device drivers |
| [BPF (eBPF)](overview/bpf.md) | BPF subsystem, program types, maps, and the verifier |
| [Cryptographic Subsystem](overview/crypto.md) | Crypto API, ciphers, hashes, and cryptographic transformations |
| [Device Model](overview/device_model.md) | Buses, drivers, devices, and sysfs |
| [VFS / Filesystems](overview/fs.md) | Virtual File System, inodes, dentries, and filesystem operations |
| [Interrupts](overview/interrupts.md) | Hardware interrupts, IRQ handling, and softirqs |
| [io_uring](overview/io_uring.md) | High-performance asynchronous I/O via shared-memory ring buffers |
| [IPC](overview/ipc.md) | Inter-process communication: pipes, signals, shared memory, and message queues |
| [Locking & Synchronization](overview/locking.md) | Spinlocks, mutexes, RW locks, and synchronization primitives |
| [Memory Management](overview/memory_management.md) | Page allocator, slab, vmalloc, page tables, and virtual memory |
| [Networking](overview/networking.md) | Network stack, sk_buff, sockets, and protocol handling |
| [Perf Events](overview/perf_event.md) | Performance counters, hardware/software events, and profiling |
| [Power Management](overview/power_management.md) | CPU frequency/idle management, suspend, and runtime PM |
| [Process Lifecycle](overview/process_lifecycle.md) | Process creation, scheduling states, and termination |
| [RCU](overview/rcu.md) | Read-Copy-Update synchronization mechanism |
| [Scheduler](overview/sched.md) | CFS, scheduling classes, runqueues, and sched_ext |
| [Security / LSM](overview/security_lsm.md) | Linux Security Modules, hooks, and access control frameworks |
| [Timers](overview/timers.md) | Timer subsystem, hrtimers, and time management |
| [Tracing](overview/tracing.md) | Ftrace, tracepoints, kprobes, and kernel tracing infrastructure |
