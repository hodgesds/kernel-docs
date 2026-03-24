# Linux Kernel Memory Barriers and Architecture-Specific Ordering

## Table of Contents
1. [Introduction](#introduction)
2. [Why Memory Reordering Happens](#why-memory-reordering-happens)
3. [Architecture Memory Models](#architecture-memory-models)
4. [Linux Kernel Barrier API](#linux-kernel-barrier-api)
5. [Barrier-to-Instruction Mapping](#barrier-to-instruction-mapping)
6. [Dependency Types](#dependency-types)
7. [Atomic Operations and Ordering](#atomic-operations-and-ordering)
8. [MMIO and DMA Barriers](#mmio-and-dma-barriers)
9. [Linux Kernel Memory Model (LKMM)](#linux-kernel-memory-model-lkmm)
10. [Lockless Programming Patterns](#lockless-programming-patterns)
11. [Implicit Barriers in Kernel Primitives](#implicit-barriers-in-kernel-primitives)
12. [Common Pitfalls](#common-pitfalls)
13. [Source File Reference](#source-file-reference)

---

## Introduction

Modern CPUs and compilers aggressively reorder memory operations to extract
instruction-level parallelism. On a single CPU this reordering is invisible
to the executing thread because the hardware preserves the illusion of
program order. But on SMP systems, one CPU may observe another CPU's memory
operations in a different order than the program specified. Memory barriers
are the mechanism to enforce ordering guarantees across CPUs.

The Linux kernel provides a portable barrier API that maps to appropriate
hardware instructions on each architecture. Understanding when and which
barriers to use is essential for writing correct lockless code.

Key kernel documentation: `Documentation/memory-barriers.txt` (the
definitive reference, ~3000 lines) and `tools/memory-model/` (the formal
LKMM).

---

## Why Memory Reordering Happens

### Compiler Reordering

The C standard allows the compiler to reorder memory accesses as long as
single-threaded semantics are preserved. The compiler has no concept of
other CPUs or threads observing shared memory:

```c
/* Compiler may reorder these: */
x = 1;
y = 2;

/* Might become: */
y = 2;
x = 1;
```

The compiler may also:
- Cache values in registers instead of re-reading from memory
- Merge or split stores (two byte stores into one word store)
- Invent stores (writing back a value even on an untaken branch)
- Remove stores it considers dead

### CPU Reordering

Even after the compiler emits instructions in a given order, the CPU
hardware may reorder them. Different architectures allow different
categories of reordering:

```
┌──────────────────────────────────────────────────────────────────┐
│            Categories of Memory Reordering                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Store-Store reordering:  W→W   (Store A then Store B)           │
│    CPU writes A and B out of order to different cache lines       │
│                                                                  │
│  Load-Load reordering:    R→R   (Load A then Load B)             │
│    CPU issues reads out of order or from different buffers        │
│                                                                  │
│  Load-Store reordering:   R→W   (Load A then Store B)            │
│    CPU retires a store before a prior load completes              │
│                                                                  │
│  Store-Load reordering:   W→R   (Store A then Load B)            │
│    CPU reads B before the store to A is visible to others         │
│    (caused by store buffers; the hardest to prevent)              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### The Store Buffer Problem

Every modern CPU has store buffers that decouple stores from the cache
coherence protocol. A store sits in the store buffer before being committed
to the cache and becoming visible to other CPUs:

```
CPU 0 executes:           CPU 1 executes:
  X = 1;                    Y = 1;
  smp_mb();                 smp_mb();
  r1 = Y;                   r2 = X;

Without barriers: r1 == 0 && r2 == 0 is possible!

Each CPU's store sits in its own store buffer, invisible to the other.
Each CPU's load reads from cache (old value) before the other's store
drains from the store buffer.
```

This is the classic Store-Load reordering (also called IRIW - Independent
Reads of Independent Writes). On x86, `smp_mb()` emits `LOCK` or `MFENCE`
to drain the store buffer. On ARM64, it uses `DMB ISH`.

---

## Architecture Memory Models

### x86 / x86-64: Total Store Order (TSO)

x86 has a relatively strong memory model called TSO (Total Store Order):

**Guarantees provided:**
- Loads are not reordered with other loads (R→R preserved)
- Stores are not reordered with other stores (W→W preserved)
- Loads are not reordered with older stores to the **same** location
- Stores from a single CPU are seen in a consistent order by all CPUs

**What CAN be reordered:**
- Stores can be reordered after later loads (W→R reordering)
  - This is the only reordering x86 permits between different addresses
  - Caused by the store buffer: a load can bypass a pending store to a
    different address

**Consequence for Linux barriers:**
- `smp_wmb()` compiles to a compiler barrier only (`barrier()`) -- the
  hardware already guarantees W→W order
- `smp_rmb()` compiles to a compiler barrier only -- the hardware already
  guarantees R→R order
- `smp_mb()` requires a real hardware instruction (`LOCK` prefix or `MFENCE`)
  to drain the store buffer and prevent W→R reordering
- `smp_store_release()` / `smp_load_acquire()` compile to plain MOV + compiler
  barrier -- x86 MOV already has release/acquire semantics natively

```
x86 Memory Model Summary:

  ┌─────────────┬──────────────────────────────┐
  │ Reordering  │ Allowed on x86?              │
  ├─────────────┼──────────────────────────────┤
  │ Load-Load   │ No  (hardware guarantee)     │
  │ Load-Store  │ No  (hardware guarantee)     │
  │ Store-Store │ No  (hardware guarantee)     │
  │ Store-Load  │ YES (store buffer bypass)    │
  └─────────────┴──────────────────────────────┘

  Exceptions to TSO:
  - Non-temporal stores (MOVNTI, MOVNTPS) bypass the cache and use
    write-combining buffers; they CAN be reordered with other stores.
    An SFENCE or MFENCE is required after non-temporal stores.
  - String operations (REP STOSB) may use a fast-string protocol
    that allows store-store reordering within the string operation.
  - CLFLUSHOPT can be reordered with other CLFLUSHOPTs and stores.
```

### ARM64 (AArch64): Weakly Ordered

ARM64 has a weakly ordered memory model. Almost all reorderings are
permitted unless explicitly prevented:

**What CAN be reordered (between different addresses):**
- Load-Load (R→R)
- Load-Store (R→W)
- Store-Store (W→W)
- Store-Load (W→R)

**Hardware guarantees:**
- Single-copy atomicity for aligned accesses
- Address dependencies are respected (if a load's address depends on a
  prior load, the second load cannot be speculated before the first)
- DMB, DSB, ISB barrier instructions enforce ordering
- LDAR/STLR instructions provide acquire/release semantics
- The Armv8.4 LDAPR instruction provides RCpc (Release Consistent
  processor consistent) acquire semantics

**ARM64 barrier instructions:**

| Instruction | Scope | Effect |
|-------------|-------|--------|
| `DMB ISH`   | Inner Shareable | Orders memory accesses; does not wait for completion |
| `DMB ISHLD` | Inner Shareable | Orders loads only (load→load, load→store) |
| `DMB ISHST` | Inner Shareable | Orders stores only (store→store) |
| `DSB ISH`   | Inner Shareable | Like DMB but also waits for all prior memory accesses to complete |
| `DSB ISH`   | Inner Shareable | Required before ISB for context synchronization |
| `ISB`       | Local CPU | Instruction synchronization; flushes pipeline |
| `LDAR`      | Per-address | Load-acquire: no subsequent memory ops reordered before this |
| `STLR`      | Per-address | Store-release: no prior memory ops reordered after this |

Shareability domains:
- **NSH** (Non-shareable): local CPU only
- **ISH** (Inner Shareable): CPUs in the same inner-shareable domain
  (typically all CPUs managed by one OS instance)
- **OSH** (Outer Shareable): CPUs across multiple OS instances
  (hypervisor level)
- **SY** (Full System): everything including DMA-capable devices

The Linux kernel exclusively uses **ISH** (Inner Shareable) for SMP
barriers because all kernel CPUs share a single coherence domain. Device
barriers use **OSH** or **SY**.

```
ARM64 Memory Model Summary:

  ┌─────────────┬──────────────────────────────┐
  │ Reordering  │ Allowed on ARM64?            │
  ├─────────────┼──────────────────────────────┤
  │ Load-Load   │ YES                          │
  │ Load-Store  │ YES                          │
  │ Store-Store │ YES                          │
  │ Store-Load  │ YES                          │
  └─────────────┴──────────────────────────────┘

  All reorderings possible → all Linux barriers emit real instructions
```

### RISC-V: RVWMO (RISC-V Weak Memory Ordering)

RISC-V defines its own weak memory model called RVWMO:

**What CAN be reordered:**
- Load-Load, Load-Store, Store-Store, Store-Load (all categories)
- Similar to ARM64 in weakness

**Preserved orderings:**
- Overlapping address accesses are ordered (same-address operations
  preserve program order)
- Address dependencies (ADDR), data dependencies (DATA), and control
  dependencies followed by a store (CTRL+W) are preserved
- AMO (Atomic Memory Operations) with `.aq` and `.rl` suffixes provide
  acquire/release semantics

**RISC-V barrier instructions:**

| Instruction | Effect |
|-------------|--------|
| `FENCE rw,rw` | Full barrier -- orders all prior {r,w} before all subsequent {r,w} |
| `FENCE r,r`   | Read barrier -- orders prior reads before subsequent reads |
| `FENCE w,w`   | Write barrier -- orders prior writes before subsequent writes |
| `FENCE r,rw`  | Acquire-like -- orders prior reads before subsequent reads and writes |
| `FENCE rw,w`  | Release-like -- orders prior reads and writes before subsequent writes |
| `FENCE.TSO`   | TSO fence -- orders all prior {r,w} before subsequent writes, and prior reads before subsequent reads |
| `LR.aq`       | Load-reserved with acquire |
| `SC.rl`       | Store-conditional with release |
| `AMO.aqrl`    | Atomic operation with acquire+release |

The RISC-V `FENCE` instruction is parameterized: predecessor and successor
sets specify which operation types to order. This allows precise expression
of barrier requirements.

```
RISC-V Memory Model Summary:

  ┌─────────────┬──────────────────────────────┐
  │ Reordering  │ Allowed on RISC-V?           │
  ├─────────────┼──────────────────────────────┤
  │ Load-Load   │ YES                          │
  │ Load-Store  │ YES                          │
  │ Store-Store │ YES                          │
  │ Store-Load  │ YES                          │
  └─────────────┴──────────────────────────────┘

  Similar to ARM64 weakness, but with parameterized FENCE instruction
```

### PowerPC / POWER: Very Weakly Ordered

PowerPC has one of the weakest memory models among mainstream
architectures. In addition to the four standard reorderings, PowerPC
permits behaviors not seen on most other architectures:

**Unique PowerPC behaviors:**
- **IRIW (Independent Reads of Independent Writes)**: Two readers can
  disagree on the order of two stores from two different writers. Most
  other architectures guarantee multi-copy atomicity (once a store is
  visible to one CPU, it is visible to all), but POWER historically did
  not (though POWER9+ improved this).
- **Load-Load reordering** is aggressive -- even dependent loads can
  appear reordered under certain conditions

**PowerPC barrier instructions:**

| Instruction | Effect |
|-------------|--------|
| `sync` (hwsync) | Full barrier; heavyweight, orders everything |
| `lwsync` (lightweight sync) | Orders R→R, R→W, W→W but NOT W→R |
| `isync` | Context synchronization; discards prefetched instructions |
| `eieio` | Orders stores to cacheable memory and I/O separately |

```
PowerPC Memory Model Summary:

  ┌─────────────┬──────────────────────────────┐
  │ Reordering  │ Allowed on PowerPC?          │
  ├─────────────┼──────────────────────────────┤
  │ Load-Load   │ YES                          │
  │ Load-Store  │ YES                          │
  │ Store-Store │ YES                          │
  │ Store-Load  │ YES                          │
  │ IRIW        │ YES (pre-POWER9)             │
  └─────────────┴──────────────────────────────┘

  Weakest mainstream model → all Linux barriers emit real instructions
```

### LoongArch: Weakly Ordered

LoongArch is a RISC ISA used in Loongson processors. It has a weakly
ordered memory model similar to ARM64:

**LoongArch barrier instructions:**

| Instruction | Effect |
|-------------|--------|
| `dbar 0` | Full barrier (all memory operations ordered) |
| `dbar 0x700` | Load-load + load-store barrier |
| `dbar 0x500` | Store-store barrier |
| `LL/SC` with hints | Load-linked/store-conditional with acquire/release hints |

### Architecture Comparison

```
┌───────────────────────────────────────────────────────────────────────┐
│          Architecture Memory Model Strength Comparison                │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Strongest ◄─────────────────────────────────────────► Weakest        │
│                                                                       │
│  x86 (TSO)          ARM64          RISC-V (RVWMO)     PowerPC         │
│  ▲                  ▲              ▲                   ▲               │
│  │                  │              │                   │               │
│  Only W→R           All four       All four            All four        │
│  reordering         reorderings    reorderings         reorderings     │
│                                                        + IRIW          │
│                                                                       │
│  Most barriers      All barriers   All barriers        All barriers    │
│  are NOPs           emit real      emit real            emit real      │
│                     instructions   instructions        instructions    │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Linux Kernel Barrier API

The kernel provides a portable set of barrier primitives in
`include/asm-generic/barrier.h`, with architecture overrides in
`arch/*/include/asm/barrier.h`.

### Compiler Barriers

```c
/* include/linux/compiler.h */

barrier();
/* Prevents the compiler from reordering memory accesses across this point.
   Does NOT emit any CPU instruction -- purely an optimizer fence.
   Defined as: asm volatile("" ::: "memory") */

READ_ONCE(x);
/* Prevents compiler from:
   - Caching the value in a register across multiple reads
   - Merging this read with other reads
   - Tearing a read (non-atomic multi-word reads)
   Emits a single load instruction; no CPU barrier. */

WRITE_ONCE(x, val);
/* Prevents compiler from:
   - Merging this store with other stores
   - Tearing a store
   - Inventing additional stores
   Emits a single store instruction; no CPU barrier. */
```

`READ_ONCE()` and `WRITE_ONCE()` are critical for lockless code. Any
shared variable accessed without a lock must use these to prevent the
compiler from optimizing away or splitting the access. They replace the
older `ACCESS_ONCE()` macro.

### SMP Memory Barriers

These are the primary barrier API. On UP (uniprocessor) kernels, `smp_*`
barriers compile down to compiler barriers only. On SMP kernels, they emit
actual hardware instructions:

```c
/* include/asm-generic/barrier.h */

smp_mb();
/* Full memory barrier.
   Orders all prior loads and stores before all subsequent loads and stores.
   Prevents: R→R, R→W, W→R, W→W reordering across this point. */

smp_rmb();
/* Read memory barrier.
   Orders all prior loads before all subsequent loads.
   Prevents: R→R reordering across this point. */

smp_wmb();
/* Write memory barrier.
   Orders all prior stores before all subsequent stores.
   Prevents: W→W reordering across this point. */

smp_mb__before_atomic();
/* Full barrier guaranteed before the next atomic RMW operation.
   Use when the atomic itself doesn't provide sufficient ordering. */

smp_mb__after_atomic();
/* Full barrier guaranteed after the preceding atomic RMW operation. */
```

### Acquire and Release Barriers

One-directional barriers that are more efficient than full barriers and
express intent more clearly:

```c
smp_load_acquire(ptr);
/* Load *ptr with acquire semantics.
   Guarantees: all subsequent memory operations (loads and stores) are
   ordered AFTER this load. Prior operations may float down past it.

   Formally: no memory operation following this in program order can be
   reordered before the load.

   One-way permeable:
     Prior ops ──can─── cross ───down──►
     ┌─────────────────────────────────┐
     │     smp_load_acquire(ptr)       │
     └─────────────────────────────────┘
     Subsequent ops ──blocked from moving up── */

smp_store_release(ptr, val);
/* Store val to *ptr with release semantics.
   Guarantees: all prior memory operations (loads and stores) are
   ordered BEFORE this store. Subsequent operations may float up past it.

   One-way permeable:
     Prior ops ──blocked from moving down──
     ┌─────────────────────────────────┐
     │   smp_store_release(ptr, val)   │
     └─────────────────────────────────┘
     Subsequent ops ──can─── cross ───up──► */
```

Acquire/release pairs form a **happens-before** relationship: if CPU 1
observes the value stored by `smp_store_release()` via `smp_load_acquire()`,
then CPU 1 is guaranteed to see all stores that preceded the release on
the writing CPU.

### Mandatory Barriers (Non-SMP)

These barriers always emit hardware instructions, even on UP. They are
used when ordering is needed for reasons beyond SMP (e.g., I/O, DMA,
ordering against hardware registers):

```c
mb();      /* Full memory barrier (always emits instruction) */
rmb();     /* Read barrier (always emits instruction) */
wmb();     /* Write barrier (always emits instruction) */
```

Use `smp_*` barriers for CPU-to-CPU ordering. Use mandatory barriers for
CPU-to-device ordering.

---

## Barrier-to-Instruction Mapping

### Per-Architecture Instruction Table

| Linux API | x86-64 | ARM64 | RISC-V | PowerPC | LoongArch |
|-----------|--------|-------|--------|---------|-----------|
| `barrier()` | `asm("":::"memory")` | `asm("":::"memory")` | `asm("":::"memory")` | `asm("":::"memory")` | `asm("":::"memory")` |
| `smp_mb()` | `lock addl $0,(%rsp)` | `DMB ISH` | `FENCE rw,rw` | `lwsync` / `sync` | `dbar 0` |
| `smp_rmb()` | `barrier()` (nop) | `DMB ISHLD` | `FENCE r,r` | `lwsync` | `dbar 0x700` |
| `smp_wmb()` | `barrier()` (nop) | `DMB ISHST` | `FENCE w,w` | `lwsync` / `eieio` | `dbar 0x500` |
| `smp_load_acquire()` | MOV + `barrier()` | `LDAR` | `FENCE r,rw` + load / `LR.aq` | `lwsync` + load | `LL` + `dbar` |
| `smp_store_release()` | `barrier()` + MOV | `STLR` | store + `FENCE rw,w` / `SC.rl` | `lwsync` + store | `dbar` + store |
| `mb()` | `MFENCE` | `DSB SY` | `FENCE iorw,iorw` | `sync` | `dbar 0` |
| `rmb()` | `LFENCE` | `DSB LD` | `FENCE ir,ir` | `sync` | `dbar 0x700` |
| `wmb()` | `SFENCE` | `DSB ST` | `FENCE ow,ow` | `sync` | `dbar 0x500` |

Notes on x86:
- `smp_mb()` uses `lock addl $0,(%rsp)` rather than `MFENCE` because the
  locked instruction is faster on most microarchitectures. The `LOCK`
  prefix provides a full barrier as a side-effect: it locks the cache line
  and drains the store buffer.
- `smp_rmb()` and `smp_wmb()` are compiler barriers only because x86 TSO
  already guarantees R→R and W→W ordering.
- `smp_load_acquire()` and `smp_store_release()` are plain MOV instructions
  with compiler barriers. x86 MOVs inherently provide acquire-load and
  release-store semantics under TSO.

### Where the Mappings Are Defined

Each architecture defines its barrier implementations in:
```
arch/x86/include/asm/barrier.h
arch/arm64/include/asm/barrier.h
arch/riscv/include/asm/barrier.h
arch/powerpc/include/asm/barrier.h
arch/loongarch/include/asm/barrier.h
include/asm-generic/barrier.h          (fallback defaults)
```

---

## Dependency Types

The kernel relies on three types of data dependencies to provide ordering
without explicit barriers. Understanding these is critical for lockless
data structures.

### Address Dependency

An **address dependency** exists when the value loaded by one access is
used to compute the address of a subsequent access:

```c
/* Classic RCU pointer dereference pattern */
struct obj *p = READ_ONCE(global_ptr);   /* load 1 */
int val = READ_ONCE(p->field);           /* load 2: address depends on load 1 */
```

**Guarantee:** All architectures guarantee that load 2 cannot be reordered
before load 1 when there is an address dependency. No barrier is needed.

This is the foundation of `rcu_dereference()`:
```c
#define rcu_dereference(p) \
    ({ \
        typeof(p) _p = READ_ONCE(p); \
        smp_read_barrier_depends(); /* no-op on all current architectures */ \
        (_p); \
    })
```

On all current Linux architectures (x86, ARM64, RISC-V, PowerPC),
address dependencies are respected by hardware. The
`smp_read_barrier_depends()` barrier historically existed for DEC Alpha
(which could speculate dependent loads), but Alpha is no longer supported
as of Linux 6.x, so this barrier is now universally a no-op.

### Data Dependency

A **data dependency** exists when the value loaded by one access is used
as the data stored by a subsequent access:

```c
int val = READ_ONCE(x);          /* load */
WRITE_ONCE(y, val + 1);         /* store: data depends on the load */
```

**Guarantee:** The store cannot be performed until the load completes,
because the CPU needs the loaded value to compute the store value. This
is naturally ordered on all architectures.

### Control Dependency

A **control dependency** exists when the value loaded by one access
determines (via a conditional branch) whether a subsequent access occurs:

```c
int val = READ_ONCE(flag);
if (val) {
    WRITE_ONCE(x, 1);    /* control-dependent store */
}
```

**Critical rules for control dependencies:**

1. Control dependencies order a **load before a subsequent store** only.
   They do NOT order load→load:
   ```c
   int val = READ_ONCE(flag);
   if (val) {
       /* This load is NOT ordered after the flag load! */
       int x = READ_ONCE(data);   /* CPU can speculate this */
   }
   ```

2. The compiler must not be able to optimize away the branch. Use
   `READ_ONCE()` for the controlling load and `WRITE_ONCE()` for the
   dependent store:
   ```c
   /* WRONG: compiler may optimize away the branch */
   if (flag)
       x = 1;

   /* CORRECT: READ_ONCE prevents branch elimination */
   if (READ_ONCE(flag))
       WRITE_ONCE(x, 1);
   ```

3. The dependency must flow through a genuine conditional branch, not
   through data manipulation:
   ```c
   /* WRONG: no control dependency, this is a data dependency */
   int val = READ_ONCE(flag);
   WRITE_ONCE(x, val);    /* data dep, not control dep */

   /* CORRECT: genuine branch */
   if (READ_ONCE(flag))
       WRITE_ONCE(x, 1);
   else
       WRITE_ONCE(x, 0);
   ```

4. Control dependencies are fragile -- the compiler may convert an `if`
   into a conditional move (CMOV), which removes the branch and destroys
   the ordering. Using `READ_ONCE()`/`WRITE_ONCE()` prevents this, but
   care is still needed.

```
┌──────────────────────────────────────────────────────────────────┐
│              Dependency Ordering Summary                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Dependency Type    │ Orders Load→Load? │ Orders Load→Store?     │
│  ──────────────────-┼───────────────────┼──────────────────────  │
│  Address            │ YES               │ YES                    │
│  Data               │ n/a               │ YES (naturally)        │
│  Control            │ NO                │ YES (with READ_ONCE)   │
│                                                                  │
│  Key rule: use smp_rmb() if you need load→load ordering          │
│  without an address dependency.                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Atomic Operations and Ordering

### Fully Ordered Atomics

Some atomic operations provide implicit full memory barriers. These
operations order all prior memory accesses before the atomic and all
subsequent memory accesses after it:

```c
/* These provide FULL ordering (implicit smp_mb() before and after): */
atomic_add_return(i, v);
atomic_sub_return(i, v);
atomic_inc_return(v);
atomic_dec_return(v);
atomic_dec_and_test(v);
atomic_inc_and_test(v);
atomic_sub_and_test(i, v);
atomic_add_negative(i, v);
xchg(ptr, new);
cmpxchg(ptr, old, new);
atomic_xchg(v, new);
atomic_cmpxchg(v, old, new);
test_and_set_bit(nr, addr);
test_and_clear_bit(nr, addr);
test_and_change_bit(nr, addr);
```

### Relaxed Atomics (No Ordering)

These atomics provide no ordering guarantees beyond the atomic operation
itself. If ordering is needed, explicit barriers must be added:

```c
/* These provide NO ordering guarantees: */
atomic_read(v);
atomic_set(v, i);
atomic_add(i, v);
atomic_sub(i, v);
atomic_inc(v);
atomic_dec(v);
set_bit(nr, addr);
clear_bit(nr, addr);
change_bit(nr, addr);
```

### Acquire/Release Atomics

The kernel provides `_acquire` and `_release` variants of many atomic
operations:

```c
atomic_cmpxchg_acquire(v, old, new);   /* acquire on success */
atomic_cmpxchg_release(v, old, new);   /* release before the cmpxchg */
atomic_cmpxchg_relaxed(v, old, new);   /* no ordering */

xchg_acquire(ptr, new);
xchg_release(ptr, new);
xchg_relaxed(ptr, new);

atomic_fetch_add_acquire(i, v);
atomic_fetch_add_release(i, v);
atomic_fetch_add_relaxed(i, v);
```

### Using smp_mb__before/after_atomic

When you need ordering around a relaxed atomic, use:

```c
/* Example: set a flag, then ensure a prior store is visible */
WRITE_ONCE(data, new_value);
smp_mb__before_atomic();
set_bit(FLAG_READY, &flags);

/* Example: clear a flag, then ensure subsequent loads see consistent state */
clear_bit(FLAG_BUSY, &flags);
smp_mb__after_atomic();
val = READ_ONCE(result);
```

**Architecture implementations of `smp_mb__before/after_atomic()`:**

| Architecture | Implementation |
|--------------|----------------|
| x86 | `barrier()` -- x86 `LOCK` prefix on the atomic already provides full ordering |
| ARM64 | `DMB ISH` -- atomics use exclusive loads/stores without inherent ordering |
| RISC-V | `FENCE rw,rw` |
| PowerPC | `lwsync` / `sync` |

On x86, `smp_mb__before_atomic()` is a no-op because x86 locked
instructions already act as full barriers. On weakly ordered architectures,
it emits a real barrier instruction.

---

## MMIO and DMA Barriers

### Device Memory Barriers

When communicating with hardware devices, SMP barriers are insufficient.
Device memory operations need mandatory barriers that are never optimized
away, even on UP:

```c
/* MMIO read/write ordering (for memory-mapped I/O regions): */
readb(addr) / readw(addr) / readl(addr) / readq(addr)
writeb(val, addr) / writew(val, addr) / writel(val, addr) / writeq(val, addr)
/* These are ORDERED: writes are visible to the device in program order,
   reads are not speculated. On x86, volatile semantics + uncacheable
   memory type enforce this. On ARM64, these use DSB + device-nGnRnE
   memory type attributes. */

/* Relaxed MMIO (no ordering guarantees): */
readb_relaxed(addr) / readw_relaxed(addr) / readl_relaxed(addr)
writeb_relaxed(val, addr) / writew_relaxed(val, addr) / writel_relaxed(val, addr)
/* Use for performance when ordering is handled explicitly. */
```

### DMA Barriers

For DMA descriptors shared between the CPU and a device:

```c
dma_rmb();
/* Orders loads from DMA-coherent memory (descriptor reads).
   Ensures a descriptor status read completes before reading
   descriptor data fields.

   Weaker than rmb() on some architectures:
   - x86: barrier() (DMA coherent by default)
   - ARM64: DMB OSHLD (outer shareable, for device visibility) */

dma_wmb();
/* Orders stores to DMA-coherent memory (descriptor writes).
   Ensures descriptor data fields are written before setting
   the ownership/status bit that tells the device to process it.

   - x86: barrier() (DMA coherent by default)
   - ARM64: DMB OSHST (outer shareable store barrier) */
```

**Typical DMA descriptor pattern:**

```c
/* Driver writing a DMA descriptor for device consumption */
desc->addr = dma_addr;
desc->len  = buffer_len;
desc->flags = DESC_FLAGS;
dma_wmb();                        /* ensure data fields written before ownership */
desc->status = DESC_HW_OWNED;    /* device can now process this descriptor */

/* Driver reading a DMA descriptor completed by device */
if (desc->status & DESC_COMPLETE) {
    dma_rmb();                    /* ensure status read before data reads */
    len  = desc->len;
    addr = desc->addr;
}
```

### I/O Barrier Summary

```
┌──────────────────────────────────────────────────────────────────┐
│              I/O Barrier Selection                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Ordering between...              Use...                         │
│  ──────────────────────────────   ────────────────               │
│  CPU ↔ CPU (normal memory)        smp_mb/rmb/wmb()               │
│  CPU ↔ DMA-coherent memory        dma_rmb/wmb()                  │
│  CPU ↔ MMIO registers             mb/rmb/wmb()                   │
│  MMIO ↔ MMIO (same device)        readX/writeX (implicit order)  │
│  MMIO ↔ DMA                       mb() or mmiowb()               │
│                                                                  │
│  mmiowb():                                                       │
│  Orders MMIO writes before a spinlock unlock. Prevents            │
│  out-of-order MMIO writes from being observed by another          │
│  CPU that subsequently takes the same spinlock. Internally        │
│  folded into spin_unlock() on architectures that need it.         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Linux Kernel Memory Model (LKMM)

### Overview

The Linux Kernel Memory Model (LKMM) is a formal model that precisely
defines the ordering guarantees of kernel synchronization primitives.
It lives in `tools/memory-model/` and uses the `herd7` tool from the
`diy` toolsuite to analyze litmus tests.

### Litmus Tests

A litmus test is a small multi-threaded program that asks "can this
outcome occur?" The LKMM tooling can answer definitively:

```
C MP+wmb+rmb
(* Message Passing with write and read barriers *)

{
  x = 0;
  y = 0;
}

P0(int *x, int *y)
{
  WRITE_ONCE(*x, 1);
  smp_wmb();
  WRITE_ONCE(*y, 1);
}

P1(int *x, int *y)
{
  int r0;
  int r1;

  r0 = READ_ONCE(*y);
  smp_rmb();
  r1 = READ_ONCE(*x);
}

exists (1:r0=1 /\ 1:r1=0)
(* Can P1 see y==1 but x==0? Answer: No *)
```

### Key LKMM Ordering Relations

The LKMM defines several ordering relations:

| Relation | Meaning |
|----------|---------|
| `sequenced-before` (sb) | Program order within a single thread |
| `reads-from` (rf) | A read takes its value from a specific write |
| `coherence-order` (co) | Total order of writes to same location |
| `from-reads` (fr) | A read sees a write that is co-before another write |
| `happens-before` (hb) | Transitive closure of ordering relations |
| `propagates-before` (pb) | Store propagation order |
| `rcu-order` | Ordering provided by RCU grace periods |

### Running Litmus Tests

```bash
# Install herd7 (from opam)
opam install herd7

# Run a single litmus test
herd7 -conf tools/memory-model/linux-kernel.cfg \
    tools/memory-model/litmus-tests/MP+wmb+rmb.litmus

# Use klitmus7 to generate a kernel module for empirical testing
klitmus7 -o /tmp/mp-test tools/memory-model/litmus-tests/MP+wmb+rmb.litmus
```

### LKMM Categories

The LKMM litmus tests in `tools/memory-model/litmus-tests/` cover common
patterns:

| Pattern | Description |
|---------|-------------|
| MP (Message Passing) | Producer writes data then flag; consumer reads flag then data |
| SB (Store Buffering) | Two CPUs each store then read the other's variable |
| LB (Load Buffering) | Two CPUs each load then store to the other's variable |
| WRC (Write-to-Read Causality) | Tests propagation transitivity |
| ISA2 | Tests cumulative barrier effects |
| IRIW | Independent Reads of Independent Writes |
| CoRR | Coherence of Read-Read: same-address load ordering |

---

## Lockless Programming Patterns

### Pattern 1: Message Passing (Producer-Consumer)

The most common lockless pattern. The producer publishes data and then
sets a flag; the consumer sees the flag and reads the data:

```c
/* Using acquire/release (preferred) */
/* Producer: */                       /* Consumer: */
WRITE_ONCE(data.a, 1);               while (!smp_load_acquire(&ready))
WRITE_ONCE(data.b, 2);                   cpu_relax();
smp_store_release(&ready, 1);         r1 = READ_ONCE(data.a);  /* sees 1 */
                                      r2 = READ_ONCE(data.b);  /* sees 2 */

/* Using explicit barriers (equivalent) */
/* Producer: */                       /* Consumer: */
WRITE_ONCE(data.a, 1);               while (!READ_ONCE(ready))
WRITE_ONCE(data.b, 2);                   cpu_relax();
smp_wmb();                            smp_rmb();
WRITE_ONCE(ready, 1);                r1 = READ_ONCE(data.a);  /* sees 1 */
                                     r2 = READ_ONCE(data.b);  /* sees 2 */
```

### Pattern 2: RCU Publish/Subscribe

RCU uses address dependencies (via `rcu_assign_pointer` /
`rcu_dereference`) to avoid explicit barriers on the read side:

```c
/* Publisher: */
struct obj *new = kmalloc(sizeof(*new), GFP_KERNEL);
new->field = value;
rcu_assign_pointer(global_ptr, new);
/* rcu_assign_pointer includes smp_store_release to ensure
   new->field is visible before the pointer update */

/* Subscriber (in RCU read-side critical section): */
rcu_read_lock();
struct obj *p = rcu_dereference(global_ptr);
/* rcu_dereference includes READ_ONCE + address dependency;
   the hardware ensures p->field reads happen after the pointer load */
if (p)
    use(p->field);
rcu_read_unlock();
```

### Pattern 3: Ring Buffer (Single Producer, Single Consumer)

```c
struct ring {
    unsigned int head;  /* written by producer, read by consumer */
    unsigned int tail;  /* written by consumer, read by producer */
    void *data[RING_SIZE];
};

/* Producer: */
unsigned int head = ring->head;
unsigned int tail = smp_load_acquire(&ring->tail);  /* read consumer's progress */
if (CIRC_SPACE(head, tail, RING_SIZE) >= 1) {
    ring->data[head] = item;
    smp_store_release(&ring->head, (head + 1) & (RING_SIZE - 1));
}

/* Consumer: */
unsigned int tail = ring->tail;
unsigned int head = smp_load_acquire(&ring->head);  /* read producer's progress */
if (CIRC_CNT(head, tail, RING_SIZE) >= 1) {
    item = ring->data[tail];
    smp_store_release(&ring->tail, (tail + 1) & (RING_SIZE - 1));
}
```

### Pattern 4: Seqcount (Write-Side Sequence + Read-Side Retry)

```c
/* Writer: */
write_seqcount_begin(&seq);     /* seq++; smp_wmb(); */
shared_data.x = new_x;
shared_data.y = new_y;
write_seqcount_end(&seq);       /* smp_wmb(); seq++; */

/* Reader: */
unsigned int s;
do {
    s = read_seqcount_begin(&seq);  /* smp_rmb() after reading seq */
    local_x = shared_data.x;
    local_y = shared_data.y;
} while (read_seqcount_retry(&seq, s));  /* smp_rmb(); check if seq changed */
```

### Pattern 5: Store Buffering (Dekker-like mutual exclusion)

This pattern requires `smp_mb()` -- weaker barriers are insufficient:

```c
/* CPU 0: */                        /* CPU 1: */
WRITE_ONCE(flag0, 1);              WRITE_ONCE(flag1, 1);
smp_mb();                          smp_mb();
r0 = READ_ONCE(flag1);            r1 = READ_ONCE(flag0);

/* Without smp_mb(): r0 == 0 && r1 == 0 is possible (store buffering).
   With smp_mb(): at least one CPU must see the other's store.
   This cannot be done with acquire/release -- full barriers are required. */
```

---

## Implicit Barriers in Kernel Primitives

Many kernel synchronization primitives include implicit memory barriers.
Understanding these prevents adding redundant barriers:

```
┌──────────────────────────────────────────────────────────────────┐
│              Implicit Barriers in Kernel Primitives               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Primitive                    Implicit Barrier                    │
│  ────────────────────────     ─────────────────────────────────  │
│  spin_lock()                  ACQUIRE (one-way: ops after lock   │
│                               cannot move before it)             │
│  spin_unlock()                RELEASE (one-way: ops before       │
│                               unlock cannot move after it)       │
│                                                                  │
│  mutex_lock()                 ACQUIRE                            │
│  mutex_unlock()               RELEASE                            │
│                                                                  │
│  smp_call_function()          Full mb() before and after on      │
│                               both caller and callee CPUs        │
│                                                                  │
│  schedule()                   Full mb() (context switch implies  │
│                               full ordering)                     │
│                                                                  │
│  wake_up() / wake_up_process  smp_mb() before checking           │
│                               condition / setting task state     │
│                                                                  │
│  wait_event()                 Pairs with wake_up(); includes     │
│                               set_current_state() barrier        │
│                                                                  │
│  set_current_state(TASK_*)    smp_mb() (via smp_store_mb())      │
│  __set_current_state(TASK_*)  NO barrier (use only when safe)    │
│                                                                  │
│  complete()                   smp_mb() + spin_lock/unlock        │
│  wait_for_completion()        spin_lock/unlock + schedule        │
│                                                                  │
│  atomic_add_return() etc.     Full mb() before and after         │
│  cmpxchg(), xchg()           Full mb() before and after         │
│                                                                  │
│  smp_store_release()         All prior ops ordered before store  │
│  smp_load_acquire()          All later ops ordered after load    │
│                                                                  │
│  rcu_assign_pointer()        smp_store_release() on pointer      │
│  rcu_dereference()           READ_ONCE + address dependency      │
│  synchronize_rcu()           Full ordering (grace period)        │
│                                                                  │
│  I/O accessors (readl, etc.) Ordered w.r.t each other and       │
│                               prior/subsequent memory accesses   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Important:** `spin_lock()` provides ACQUIRE, not a full barrier.
`spin_unlock()` provides RELEASE, not a full barrier. Together they do
NOT form `smp_mb()`. Specifically, a store before `spin_lock()` can be
reordered after the lock acquisition, and a load after `spin_unlock()`
can be reordered before the unlock. Code that needs full barrier semantics
around a lock critical section must add explicit `smp_mb__after_spinlock()`
or `smp_mb()`.

---

## Common Pitfalls

### 1. Missing READ_ONCE/WRITE_ONCE on shared variables

```c
/* BUG: compiler may cache 'done' in a register, infinite loop */
while (!done)
    cpu_relax();

/* FIX: READ_ONCE forces re-read from memory */
while (!READ_ONCE(done))
    cpu_relax();
```

### 2. Assuming control dependencies order loads

```c
/* BUG: control dependency does NOT order load→load */
if (READ_ONCE(flag)) {
    val = READ_ONCE(data);   /* can be speculated before flag is read */
}

/* FIX: use smp_rmb() or smp_load_acquire() */
if (READ_ONCE(flag)) {
    smp_rmb();
    val = READ_ONCE(data);
}
/* Or better: */
if (smp_load_acquire(&flag)) {
    val = READ_ONCE(data);
}
```

### 3. Using smp_wmb() where smp_mb() is needed

```c
/* BUG: smp_wmb only orders W→W, not W→R */
WRITE_ONCE(data, 1);
smp_wmb();
r1 = READ_ONCE(flag);    /* can be reordered before the write! */

/* FIX: use smp_mb() for W→R ordering */
WRITE_ONCE(data, 1);
smp_mb();
r1 = READ_ONCE(flag);
```

### 4. Assuming locks provide full barriers

```c
/* BUG: store before lock can leak into critical section's visibility */
WRITE_ONCE(x, 1);
spin_lock(&lock);
/* x = 1 might not be visible to another CPU yet */
r1 = READ_ONCE(shared);
spin_unlock(&lock);

/* If ordering of x w.r.t. the critical section matters, add a barrier: */
WRITE_ONCE(x, 1);
smp_mb();            /* or restructure to put the store inside the lock */
spin_lock(&lock);
r1 = READ_ONCE(shared);
spin_unlock(&lock);
```

### 5. Using plain C accesses for shared memory

```c
/* BUG: data races are undefined behavior */
shared_var = 1;              /* compiler may tear, merge, or remove */
if (shared_var) { ... }      /* compiler may cache in register */

/* FIX: always use READ_ONCE/WRITE_ONCE for lockless shared access */
WRITE_ONCE(shared_var, 1);
if (READ_ONCE(shared_var)) { ... }
```

### 6. Barrier pairing mistakes

```c
/* BUG: smp_wmb on writer must pair with smp_rmb on reader */
/* Writer: */                     /* Reader: */
WRITE_ONCE(data, 1);             r0 = READ_ONCE(flag);
smp_wmb();                       /* MISSING smp_rmb() here! */
WRITE_ONCE(flag, 1);             r1 = READ_ONCE(data);
/* Without the pairing rmb, the reader can see flag=1 but data=0 */
```

### 7. Redundant barriers

```c
/* WASTEFUL: spin_unlock already provides release semantics */
smp_wmb();             /* redundant! */
spin_unlock(&lock);

/* WASTEFUL: atomic_add_return already provides full ordering */
atomic_add_return(1, &counter);
smp_mb();              /* redundant! */

/* WASTEFUL: smp_store_release already orders prior writes */
smp_wmb();             /* redundant! */
smp_store_release(&ready, 1);
```

---

## Source File Reference

| File | Purpose |
|------|---------|
| `Documentation/memory-barriers.txt` | Definitive kernel memory barrier documentation (~3000 lines) |
| `tools/memory-model/` | LKMM formal model, herd7 configuration, litmus tests |
| `tools/memory-model/linux-kernel.cfg` | herd7 configuration for LKMM |
| `tools/memory-model/linux-kernel.cat` | LKMM axioms in cat language |
| `tools/memory-model/litmus-tests/` | Example litmus tests for common patterns |
| `include/asm-generic/barrier.h` | Generic (fallback) barrier implementations |
| `include/linux/compiler.h` | `barrier()`, `READ_ONCE()`, `WRITE_ONCE()` definitions |
| `include/linux/atomic/atomic-instrumented.h` | Atomic operation wrappers with KASAN/KCSAN instrumentation |
| `include/linux/atomic/atomic-arch-fallback.h` | Fallback atomic implementations with ordering variants |
| `arch/x86/include/asm/barrier.h` | x86 barrier implementations |
| `arch/arm64/include/asm/barrier.h` | ARM64 barrier implementations (DMB, DSB, ISB, LDAR, STLR) |
| `arch/riscv/include/asm/barrier.h` | RISC-V barrier implementations (FENCE variants) |
| `arch/powerpc/include/asm/barrier.h` | PowerPC barrier implementations (sync, lwsync, isync) |
| `arch/loongarch/include/asm/barrier.h` | LoongArch barrier implementations (dbar) |
| `include/linux/rcupdate.h` | `rcu_assign_pointer()`, `rcu_dereference()` with barrier semantics |
| `include/linux/seqlock.h` | Seqcount/seqlock with embedded barriers |
| `include/linux/spinlock.h` | Spinlock acquire/release barrier semantics |
| `include/asm-generic/rwonce.h` | `READ_ONCE()` / `WRITE_ONCE()` implementation |
