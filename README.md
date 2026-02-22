# kernel-docs

Extended documentation for various Linux kernel subsystems and APIs. These docs are designed to be consumable by both humans and LLMs for quickly getting context on kernel internals.

## Overview

| Topic | Description |
|-------|-------------|
| [Block Layer](overview/block_layer.md) | Linux block I/O layer, request queues, and block device drivers |
| [BPF (eBPF)](overview/bpf.md) | BPF subsystem, program types, maps, and the verifier |
| [Device Model](overview/device_model.md) | Buses, drivers, devices, and sysfs |
| [VFS / Filesystems](overview/fs.md) | Virtual File System, inodes, dentries, and filesystem operations |
| [Locking & Synchronization](overview/locking.md) | Spinlocks, mutexes, RW locks, and synchronization primitives |
| [Memory Management](overview/memory_management.md) | Page allocator, slab, vmalloc, page tables, and virtual memory |
| [Networking](overview/networking.md) | Network stack, sk_buff, sockets, and protocol handling |
| [Process Lifecycle](overview/process_lifecycle.md) | Process creation, scheduling states, and termination |
| [RCU](overview/rcu.md) | Read-Copy-Update synchronization mechanism |
| [Scheduler](overview/sched.md) | CFS, scheduling classes, runqueues, and sched_ext |
| [Timers](overview/timers.md) | Timer subsystem, hrtimers, and time management |
