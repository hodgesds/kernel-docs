# Linux Kernel Kprobes and Uprobes Subsystem

## Table of Contents
1. [Overview](#1-overview)
2. [Kprobes Architecture](#2-kprobes-architecture)
3. [Kprobe Core Data Structures](#3-kprobe-core-data-structures)
4. [Kprobe Registration and Lifecycle](#4-kprobe-registration-and-lifecycle)
5. [x86 Breakpoint Mechanism (INT3)](#5-x86-breakpoint-mechanism-int3)
6. [Single-Stepping and Boosting](#6-single-stepping-and-boosting)
7. [Kretprobes (Return Probes)](#7-kretprobes-return-probes)
8. [Optprobes (Jump Optimization)](#8-optprobes-jump-optimization)
9. [Ftrace-Based Kprobes](#9-ftrace-based-kprobes)
10. [Kprobe Blacklisting](#10-kprobe-blacklisting)
11. [Kprobe Aggregation (Multi-Handler)](#11-kprobe-aggregation-multi-handler)
12. [Uprobes Architecture](#12-uprobes-architecture)
13. [Uprobe Core Data Structures](#13-uprobe-core-data-structures)
14. [Uprobe Registration and Inode-Based Tracking](#14-uprobe-registration-and-inode-based-tracking)
15. [XOL (Execute Out-of-Line) Area](#15-xol-execute-out-of-line-area)
16. [Uprobe Hit Handling (handle_swbp)](#16-uprobe-hit-handling-handle_swbp)
17. [Uretprobes (User Return Probes)](#17-uretprobes-user-return-probes)
18. [Uprobe Consumers](#18-uprobe-consumers)
19. [BPF Attachment to Kprobes/Uprobes](#19-bpf-attachment-to-kprobesuprobes)
20. [perf_event Integration](#20-perf_event-integration)
21. [Jprobes (Deprecated)](#21-jprobes-deprecated)
22. [Debugfs Interface](#22-debugfs-interface)
23. [Architecture-Specific Details](#23-architecture-specific-details)

---

## 1. Overview

Kprobes and uprobes are dynamic instrumentation frameworks in the Linux kernel
that enable tracing and debugging of kernel and user-space code without
recompilation or rebooting. They form the foundation for many higher-level
tracing tools including ftrace trace events, perf, BPF, and SystemTap.

```
                        Dynamic Probing Infrastructure
 ┌────────────────────────────────────────────────────────────────────────┐
 │                                                                        │
 │   User Space                                                           │
 │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
 │   │  perf probe  │  │  bpftrace    │  │  trace-cmd   │                 │
 │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                 │
 │          │                 │                  │                         │
 │   ───────┼─────────────────┼──────────────────┼────────────────────    │
 │          │ perf_event /    │ BPF_PROG_LOAD    │ tracefs                │
 │          │ syscall         │                  │                         │
 │   Kernel Space             ▼                  ▼                        │
 │          │          ┌─────────────┐    ┌─────────────┐                 │
 │          └─────────►│ trace_kprobe│    │ trace_uprobe│                 │
 │                     └──────┬──────┘    └──────┬──────┘                 │
 │                            │                  │                         │
 │                     ┌──────▼──────┐    ┌──────▼──────┐                 │
 │                     │   kprobes   │    │   uprobes   │                 │
 │                     │  (kernel)   │    │ (userspace) │                 │
 │                     └──────┬──────┘    └──────┬──────┘                 │
 │                            │                  │                         │
 │                     ┌──────▼──────┐    ┌──────▼──────┐                 │
 │                     │  INT3 / BRK │    │  INT3 / BRK │                 │
 │                     │ (breakpoint)│    │ (breakpoint) │                │
 │                     └─────────────┘    └─────────────┘                 │
 │                                                                        │
 └────────────────────────────────────────────────────────────────────────┘
```

**Key source files:**
- `kernel/kprobes.c` -- core kprobe infrastructure
- `include/linux/kprobes.h` -- kprobe API and data structures
- `kernel/events/uprobes.c` -- core uprobe infrastructure
- `include/linux/uprobes.h` -- uprobe API and data structures
- `arch/x86/kernel/kprobes/core.c` -- x86 kprobe arch support
- `arch/x86/kernel/kprobes/opt.c` -- x86 optprobe (JMP optimization)
- `arch/x86/kernel/kprobes/ftrace.c` -- x86 ftrace-based kprobes
- `arch/x86/kernel/uprobes.c` -- x86 uprobe arch support
- `kernel/trace/trace_kprobe.c` -- kprobe trace events

---

## 2. Kprobes Architecture

Kprobes enables dynamic insertion of probe points at virtually any kernel
instruction. The mechanism works by:

1. Saving the original instruction at the probe point
2. Replacing the first byte with a breakpoint instruction (INT3 / 0xCC on x86)
3. When the breakpoint fires, executing the user's pre-handler
4. Single-stepping the saved original instruction (or emulating/boosting it)
5. Executing the user's post-handler
6. Resuming normal execution

```
                     Kprobe Execution Flow (x86)

   Original code:           Probed code:
   ┌─────────────┐          ┌─────────────┐
   │ mov %rdi,%r9│          │ INT3 (0xCC) │◄── breakpoint replaces 1st byte
   │ ...         │          │ ...         │
   └─────────────┘          └─────────────┘
                                  │
                                  ▼ INT3 trap
                            ┌─────────────┐
                            │kprobe_int3_ │
                            │  handler()  │
                            └──────┬──────┘
                                   │
                    ┌──────────────►├──────────────────┐
                    │               │                  │
              ┌─────▼─────┐  ┌─────▼─────┐   ┌───────▼───────┐
              │ pre_handler│  │  emulate  │   │  single-step  │
              │  callback  │  │  (boost)  │   │   copy of     │
              └─────┬──────┘  │  JMP back │   │  original insn│
                    │         └───────────┘   └───────┬───────┘
                    │                                 │ INT3 (end)
                    │                          ┌──────▼──────┐
                    │                          │ post_handler│
                    │                          │  callback   │
                    │                          └──────┬──────┘
                    │                                 │
                    └────────────┬─────────────────────┘
                                 │
                          ┌──────▼──────┐
                          │   resume    │
                          │  execution  │
                          └─────────────┘
```

---

## 3. Kprobe Core Data Structures

### `struct kprobe`

`include/linux/kprobes.h:59`

The fundamental probe descriptor. Each probe point is represented by one
`kprobe` instance:

```c
struct kprobe {
    struct hlist_node hlist;           /* hash table linkage (kprobe_table[]) */

    /* list of kprobes for multi-handler support */
    struct list_head list;

    /* count the number of times this probe was temporarily disarmed */
    unsigned long nmissed;

    /* location of the probe point */
    kprobe_opcode_t *addr;

    /* Allow user to indicate symbol name of the probe point */
    const char *symbol_name;

    /* Offset into the symbol */
    unsigned int offset;

    /* Called before addr is executed. */
    kprobe_pre_handler_t pre_handler;

    /* Called after addr is executed, unless... */
    kprobe_post_handler_t post_handler;

    /* Saved opcode (which has been replaced with breakpoint) */
    kprobe_opcode_t opcode;

    /* copy of the original instruction */
    struct arch_specific_insn ainsn;

    /* Various status flags (KPROBE_FLAG_*) */
    u32 flags;
};
```

**Handler type signatures** (`include/linux/kprobes.h:53`):

```c
typedef int (*kprobe_pre_handler_t) (struct kprobe *, struct pt_regs *);
typedef void (*kprobe_post_handler_t) (struct kprobe *, struct pt_regs *,
                                       unsigned long flags);
```

**Kprobe flags** (`include/linux/kprobes.h:97`):

| Flag | Value | Meaning |
|------|-------|---------|
| `KPROBE_FLAG_GONE` | 1 | Breakpoint has already gone (module unloaded) |
| `KPROBE_FLAG_DISABLED` | 2 | Probe is temporarily disabled |
| `KPROBE_FLAG_OPTIMIZED` | 4 | Probe is using jump optimization (optprobe) |
| `KPROBE_FLAG_FTRACE` | 8 | Probe is using ftrace mechanism |
| `KPROBE_FLAG_ON_FUNC_ENTRY` | 16 | Probe is at function entry point |

### `struct arch_specific_insn` (x86)

`arch/x86/include/asm/kprobes.h:54`

Per-architecture instruction analysis and single-step support:

```c
struct arch_specific_insn {
    /* copy of the original instruction (on executable page) */
    kprobe_opcode_t *insn;
    unsigned boostable:1;       /* 1 = can skip single-step via jmp */
    unsigned char size;         /* size of the instruction */
    union {
        unsigned char opcode;   /* for IF-modifier emulation */
        struct {
            unsigned char type; /* conditional jump type */
        } jcc;
        struct {
            unsigned char type; /* loop type */
            unsigned char asize;/* address size */
        } loop;
        struct {
            unsigned char reg;  /* register for indirect call/jmp */
        } indirect;
    };
    s32 rel32;                  /* relative offset for branches */
    void (*emulate_op)(struct kprobe *p, struct pt_regs *regs);
    int tp_len;                 /* number of bytes of text poked */
};
```

### `struct kprobe_ctlblk` (x86)

`arch/x86/include/asm/kprobes.h:108`

Per-CPU control block that tracks the currently active kprobe during
breakpoint handling:

```c
struct kprobe_ctlblk {
    unsigned long kprobe_status;       /* KPROBE_HIT_ACTIVE, _SS, etc. */
    unsigned long kprobe_old_flags;
    unsigned long kprobe_saved_flags;
    struct prev_kprobe prev_kprobe;    /* for re-entrant probe handling */
};
```

Declared per-CPU at `arch/x86/kernel/kprobes/core.c:62`:

```c
DEFINE_PER_CPU(struct kprobe *, current_kprobe) = NULL;
DEFINE_PER_CPU(struct kprobe_ctlblk, kprobe_ctlblk);
```

### Kprobe Hash Table

`kernel/kprobes.c:48-49,61`

```c
#define KPROBE_HASH_BITS 6
#define KPROBE_TABLE_SIZE (1 << KPROBE_HASH_BITS)  /* 64 buckets */

static struct hlist_head kprobe_table[KPROBE_TABLE_SIZE];
```

Probes are looked up by hashing the `addr` field. Protected by `kprobe_mutex`
for modification and RCU for lockless read during breakpoint handling.

---

## 4. Kprobe Registration and Lifecycle

### `register_kprobe()`

`kernel/kprobes.c:1621`

The primary registration path:

```
register_kprobe(p)
    │
    ├── _kprobe_addr()           ← Resolve symbol_name + offset to addr
    │       └── kprobe_lookup_name()  ← kallsyms_lookup_name()
    │       └── arch_adjust_kprobe_addr()  ← Skip ENDBR/BTI on x86
    │
    ├── warn_kprobe_rereg()      ← Reject double registration
    │
    ├── check_kprobe_address_safe()
    │       ├── check_ftrace_location()  ← Set KPROBE_FLAG_FTRACE if on NOP5
    │       ├── core_kernel_text()       ← Ensure in text section
    │       ├── within_kprobe_blacklist()
    │       ├── jump_label_text_reserved()
    │       └── static_call_text_reserved()
    │
    ├── [if existing probe at addr] register_aggr_kprobe()
    │       └── Creates aggregator kprobe for multi-handler
    │
    ├── prepare_kprobe(p)
    │       ├── [ftrace] arch_prepare_kprobe_ftrace()
    │       └── [normal] arch_prepare_kprobe()
    │               ├── get_insn_slot()        ← Allocate executable page slot
    │               ├── __copy_instruction()   ← Copy + fixup RIP-relative
    │               ├── prepare_emulation()    ← Set emulate_op for branches
    │               └── prepare_singlestep()   ← Add JMP-back or INT3 suffix
    │
    ├── hlist_add_head_rcu()     ← Insert into kprobe_table[]
    │
    ├── arm_kprobe(p)
    │       ├── [ftrace] arm_kprobe_ftrace()
    │       │       └── ftrace_set_filter_ip() + register_ftrace_function()
    │       └── [normal] arch_arm_kprobe()
    │               └── text_poke(addr, INT3, 1)  ← Write breakpoint
    │
    └── try_to_optimize_kprobe() ← Attempt JMP optimization (optprobe)
```

### `unregister_kprobe()`

`kernel/kprobes.c:1849`

Two-phase removal via `__unregister_kprobe_top()` (under `kprobe_mutex`) and
`__unregister_kprobe_bottom()` (after `synchronize_rcu()`):

```c
void unregister_kprobe(struct kprobe *p)
{
    unregister_kprobes(&p, 1);
}
```

The top-half disables and disarms the probe. The bottom-half frees the
instruction slot and any aggregator after an RCU grace period ensures no
CPU is in the breakpoint handler.

---

## 5. x86 Breakpoint Mechanism (INT3)

### Arming: Writing the Breakpoint

`arch/x86/kernel/kprobes/core.c:815`

```c
void arch_arm_kprobe(struct kprobe *p)
{
    u8 int3 = INT3_INSN_OPCODE;     /* 0xCC */
    text_poke(p->addr, &int3, 1);   /* atomic code patching */
    text_poke_sync();                /* IPI to synchronize icache */
}
```

Only the first byte of the instruction is replaced. The original first byte
is saved in `p->opcode`.

### INT3 Handler Entry

`arch/x86/kernel/kprobes/core.c:1006`

When the CPU executes INT3, it vectors to the kernel's debug trap handler
which calls `kprobe_int3_handler()`:

```c
int kprobe_int3_handler(struct pt_regs *regs)
{
    addr = (kprobe_opcode_t *)(regs->ip - sizeof(kprobe_opcode_t));
    kcb = get_kprobe_ctlblk();
    p = get_kprobe(addr);            /* hash table lookup */

    if (p) {
        if (kprobe_running()) {
            reenter_kprobe(p, regs, kcb);  /* nested probe */
        } else {
            set_current_kprobe(p, regs, kcb);
            kcb->kprobe_status = KPROBE_HIT_ACTIVE;

            if (!p->pre_handler || !p->pre_handler(p, regs))
                setup_singlestep(p, regs, kcb, 0);
            else
                reset_current_kprobe();
        }
        return 1;
    }
    /* ... handle post-single-step INT3 ... */
}
```

### Kprobe Status States

```
 ┌──────────────────┐       pre_handler
 │  KPROBE_HIT_     │──────────────────►┌──────────────────┐
 │     ACTIVE        │                   │  KPROBE_HIT_SS   │
 └──────────────────┘                   └────────┬─────────┘
                                                 │ single-step done
                                        ┌────────▼─────────┐
                                        │ KPROBE_HIT_SSDONE│
                                        │  post_handler    │
                                        └──────────────────┘

                      re-entrant hit
 ┌──────────────────┐
 │ KPROBE_REENTER   │  (nested probe, skip handlers)
 └──────────────────┘
```

---

## 6. Single-Stepping and Boosting

After the pre-handler runs, the kernel must execute the original instruction
that was replaced by INT3. There are three strategies on x86:

### Strategy 1: Instruction Emulation

`arch/x86/kernel/kprobes/core.c:639`

For control-flow instructions (call, jmp, jcc, loop, ret, push/popf), the
instruction is emulated directly without executing a copy. The function
`prepare_emulation()` sets `p->ainsn.emulate_op` to the appropriate emulator:

| Opcode | Emulator Function |
|--------|-------------------|
| `0xE8` (near call) | `kprobe_emulate_call()` |
| `0xE9/0xEB` (jmp) | `kprobe_emulate_jmp()` |
| `0x70-0x7F` (jcc) | `kprobe_emulate_jcc()` |
| `0xC2/0xC3` (ret) | `kprobe_emulate_ret()` |
| `0xFA/0xFB` (cli/sti) | `kprobe_emulate_ifmodifiers()` |
| `0xFF /2` (call indirect) | `kprobe_emulate_call_indirect()` |
| `0xFF /4` (jmp indirect) | `kprobe_emulate_jmp_indirect()` |
| `0xE0-0xE3` (loop/jcxz) | `kprobe_emulate_loop()` |

### Strategy 2: Boosting (Direct Execution)

`arch/x86/kernel/kprobes/core.c:464`

For non-control-flow instructions that are "boostable" (determined by
`can_boost()` at `core.c:140`), the copied instruction has a relative JMP
appended that jumps back to the instruction after the probe point. The CPU
executes the copy directly, skipping single-step overhead:

```c
if (!IS_ENABLED(CONFIG_PREEMPTION) &&
    !p->post_handler && can_boost(insn, p->addr) &&
    MAX_INSN_SIZE - len >= JMP32_INSN_SIZE) {
    synthesize_reljump(buf + len, p->ainsn.insn + len,
                       p->addr + insn->length);
    p->ainsn.boostable = 1;
}
```

```
  Boosted single-step (no trap overhead):

  Insn copy slot:
  ┌──────────────────────────────────┐
  │ original insn (copied)           │
  │ JMP rel32 → (p->addr + insn_len)│  ← jumps back after probe point
  └──────────────────────────────────┘
```

### Strategy 3: INT3 Single-Step

For instructions that cannot be emulated or boosted, an INT3 is placed
immediately after the copied instruction. The CPU single-steps into the copy,
hits the trailing INT3, and the handler fixes up `regs->ip` to point after
the original probe point.

```
  INT3 single-step:

  Insn copy slot:
  ┌──────────────────────────────────┐
  │ original insn (copied)           │
  │ INT3 (0xCC)                      │  ← traps back into handler
  └──────────────────────────────────┘
```

### Instruction Slot Management

`kernel/kprobes.c:89-198`

Copied instructions live on special executable pages managed by
`struct kprobe_insn_cache`:

```c
struct kprobe_insn_cache {
    struct mutex mutex;
    void *(*alloc)(void);       /* allocate insn page */
    void (*free)(void *);       /* free insn page */
    const char *sym;            /* symbol for insn pages */
    struct list_head pages;     /* list of kprobe_insn_page */
    size_t insn_size;           /* size of instruction slot */
    int nr_garbage;
};
```

Pages are allocated via `execmem_alloc(EXECMEM_KPROBES, PAGE_SIZE)` to ensure
they are within +/- 2GB of the kernel image (required for RIP-relative
addressing fixups on x86-64).

---

## 7. Kretprobes (Return Probes)

### `struct kretprobe`

`include/linux/kprobes.h:146`

Kretprobes instrument function returns by hijacking the return address:

```c
struct kretprobe {
    struct kprobe kp;                /* entry kprobe */
    kretprobe_handler_t handler;     /* return handler */
    kretprobe_handler_t entry_handler;/* optional entry handler */
    int maxactive;                   /* max concurrent instances */
    int nmissed;                     /* missed due to maxactive limit */
    size_t data_size;                /* per-instance private data size */
#ifdef CONFIG_KRETPROBE_ON_RETHOOK
    struct rethook *rh;
#else
    struct kretprobe_holder *rph;    /* object pool holder */
#endif
};
```

### `struct kretprobe_instance`

`include/linux/kprobes.h:162`

Each active invocation of a probed function gets an instance:

```c
struct kretprobe_instance {
#ifdef CONFIG_KRETPROBE_ON_RETHOOK
    struct rethook_node node;
#else
    struct rcu_head rcu;
    struct llist_node llist;
    struct kretprobe_holder *rph;
    kprobe_opcode_t *ret_addr;   /* original return address */
    void *fp;                    /* frame pointer */
#endif
    char data[];                 /* user private data (data_size bytes) */
};
```

### Kretprobe Mechanism

```
  Kretprobe Flow:

  1. register_kretprobe() installs a kprobe at function entry
     with pre_handler = pre_handler_kretprobe

  2. On function entry (INT3 hit):
     ┌────────────────────────────────┐
     │ pre_handler_kretprobe()        │
     │   ├── Pop ri from object pool  │
     │   ├── entry_handler(ri, regs)  │  ← optional entry callback
     │   └── arch_prepare_kretprobe() │
     │         └── Save return addr   │
     │         └── Replace with       │
     │             trampoline addr    │
     └────────────────────────────────┘

  3. Function executes normally...

  4. On function return (RET to trampoline):
     ┌────────────────────────────────┐
     │ __kretprobe_trampoline_handler │
     │   ├── Find correct ret addr    │
     │   ├── Call rp->handler(ri,regs)│  ← user return callback
     │   ├── Restore original ret addr│
     │   └── Recycle ri to pool       │
     └────────────────────────────────┘
```

### `register_kretprobe()`

`kernel/kprobes.c:2203`

```c
int register_kretprobe(struct kretprobe *rp)
{
    /* Must be at function entry */
    ret = kprobe_on_func_entry(rp->kp.addr, rp->kp.symbol_name, rp->kp.offset);

    /* Check kretprobe blacklist (__switch_to, etc.) */
    for (i = 0; kretprobe_blacklist[i].name != NULL; i++)
        if (kretprobe_blacklist[i].addr == addr) return -EINVAL;

    rp->kp.pre_handler = pre_handler_kretprobe;
    rp->kp.post_handler = NULL;

    /* Default maxactive = max(10, 2 * num_possible_cpus) */
    if (rp->maxactive <= 0)
        rp->maxactive = max_t(unsigned int, 10, 2*num_possible_cpus());

    /* Allocate object pool for kretprobe_instance */
    /* ... (rethook or objpool based) ... */

    ret = register_kprobe(&rp->kp);  /* Register the entry probe */
}
```

### Kretprobe Blacklist

`arch/x86/kernel/kprobes/core.c:101`

Certain functions must never have return probes because their return
semantics are special (e.g., `__switch_to` changes the stack):

```c
struct kretprobe_blackpoint kretprobe_blacklist[] = {
    {"__switch_to", },
    {NULL, NULL}
};
```

---

## 8. Optprobes (Jump Optimization)

### Overview

When `CONFIG_OPTPROBES` is enabled, kprobes can be "optimized" by replacing the
INT3 breakpoint with a near JMP (5-byte `0xE9` relative jump) to a detour
buffer. This avoids the expensive INT3 trap on every hit.

### `struct optimized_kprobe`

`include/linux/kprobes.h:338`

```c
struct optimized_kprobe {
    struct kprobe kp;
    struct list_head list;              /* queue for optimizer */
    struct arch_optimized_insn optinsn; /* arch-specific detour info */
};
```

### `struct arch_optimized_insn` (x86)

`arch/x86/include/asm/kprobes.h:85`

```c
struct arch_optimized_insn {
    kprobe_opcode_t copied_insn[DISP32_SIZE]; /* original bytes (4 bytes) */
    kprobe_opcode_t *insn;   /* detour code buffer */
    size_t size;             /* size of instructions copied */
};
```

### Optimization Flow

`kernel/kprobes.c:506-638`

```
  Optprobe Mechanism:

  Original code (5+ bytes):
  ┌─────────────────────────────┐
  │ insn1 (probed)              │
  │ insn2 (may be overwritten)  │
  └─────────────────────────────┘
                │
                ▼ replaced with
  ┌─────────────────────────────┐
  │ JMP rel32 → detour_buffer   │  ← 5-byte near jump
  └─────────────────────────────┘
                │
                ▼
  Detour buffer (executable page):
  ┌─────────────────────────────────────┐
  │ Save all registers (SAVE_REGS)      │
  │ Set arg1 = &optimized_kprobe        │
  │ CALL opt_pre_handler                │
  │   └─ iterate p->list, call handlers │
  │ Restore registers (RESTORE_REGS)    │
  │ Execute original instructions       │
  │ JMP back to original code + N       │
  └─────────────────────────────────────┘
```

### Optimizer Work Queue

`kernel/kprobes.c:507-513`

Optimization is deferred to a work queue to allow safe text patching:

```c
static LIST_HEAD(optimizing_list);
static LIST_HEAD(unoptimizing_list);
static LIST_HEAD(freeing_list);

static DECLARE_DELAYED_WORK(optimizing_work, kprobe_optimizer);
#define OPTIMIZE_DELAY 5
```

The `kprobe_optimizer()` worker (`kernel/kprobes.c:601`) performs:
1. Unoptimize probes on `unoptimizing_list`
2. `synchronize_rcu_tasks()` -- wait for preempted tasks to reschedule
3. Optimize probes on `optimizing_list`
4. Free cleaned probes

### Optimization Constraints

A kprobe **cannot** be optimized if:
- It has a `post_handler` (needs single-step semantics)
- It is ftrace-based (`KPROBE_FLAG_FTRACE`)
- The instructions at `addr` through `addr+4` cross a function boundary
- Another kprobe exists within the overwritten region
- The address is in an exception table or jump label region

### Sysctl Control

`kernel/kprobes.c:957`

```
/proc/sys/debug/kprobes-optimization
```

Write 0 to disable optimization, 1 to enable. When disabled, all optimized
probes revert to INT3 breakpoints.

---

## 9. Ftrace-Based Kprobes

### Overview

When `CONFIG_KPROBES_ON_FTRACE` is enabled and a kprobe is placed on a
function entry that coincides with an ftrace NOP site, the kprobe uses
ftrace's callback mechanism instead of INT3. This is more efficient because
ftrace already has infrastructure for function entry interception.

### Detection

`kernel/kprobes.c:1535`

```c
static int check_ftrace_location(struct kprobe *p)
{
    unsigned long addr = (unsigned long)p->addr;
    if (ftrace_location(addr) == addr) {
        p->flags |= KPROBE_FLAG_FTRACE;
    }
}
```

### Ftrace Ops

`kernel/kprobes.c:1054`

Two `ftrace_ops` are used -- one for normal probes and one for IP-modifying
probes (those with `post_handler`):

```c
static struct ftrace_ops kprobe_ftrace_ops __read_mostly = {
    .func = kprobe_ftrace_handler,
    .flags = FTRACE_OPS_FL_SAVE_REGS,
};

static struct ftrace_ops kprobe_ipmodify_ops __read_mostly = {
    .func = kprobe_ftrace_handler,
    .flags = FTRACE_OPS_FL_SAVE_REGS | FTRACE_OPS_FL_IPMODIFY,
};
```

### Handler

`arch/x86/kernel/kprobes/ftrace.c:17`

```c
void kprobe_ftrace_handler(unsigned long ip, unsigned long parent_ip,
                           struct ftrace_ops *ops, struct ftrace_regs *fregs)
{
    struct pt_regs *regs = ftrace_get_regs(fregs);
    p = get_kprobe((kprobe_opcode_t *)ip);

    if (kprobe_running()) {
        kprobes_inc_nmissed_count(p);  /* recursive -- skip */
    } else {
        /* Adjust IP to look like breakpoint hit */
        instruction_pointer_set(regs, ip + INT3_INSN_SIZE);
        __this_cpu_write(current_kprobe, p);
        kcb->kprobe_status = KPROBE_HIT_ACTIVE;

        if (!p->pre_handler || !p->pre_handler(p, regs)) {
            if (p->post_handler) {
                instruction_pointer_set(regs, ip + MCOUNT_INSN_SIZE);
                kcb->kprobe_status = KPROBE_HIT_SSDONE;
                p->post_handler(p, regs, 0);
            }
            instruction_pointer_set(regs, orig_ip);
        }
        __this_cpu_write(current_kprobe, NULL);
    }
}
```

Note: ftrace-based kprobes **cannot** be optimized to optprobes (`try_to_optimize_kprobe` checks `kprobe_ftrace(p)`).

---

## 10. Kprobe Blacklisting

### Mechanisms

Certain addresses must never be probed to avoid deadlocks, infinite
recursion, or corruption. There are multiple layers of protection:

**1. `__kprobes` section** (`kernel/kprobes.c:1374`):

Functions annotated with `__kprobes` or `NOKPROBE_SYMBOL()` are placed in
the `.kprobes.text` section and are automatically blacklisted:

```c
bool __weak arch_within_kprobe_blacklist(unsigned long addr)
{
    return addr >= (unsigned long)__kprobes_text_start &&
           addr < (unsigned long)__kprobes_text_end;
}
```

**2. Blacklist entries** (`kernel/kprobes.c:77-80`):

A linked list of `struct kprobe_blacklist_entry` ranges populated at boot
from the `_kprobe_blacklist` section:

```c
struct kprobe_blacklist_entry {
    struct list_head list;
    unsigned long start_addr;
    unsigned long end_addr;
};

static LIST_HEAD(kprobe_blacklist);
```

**3. Dynamic blacklist additions** (`kernel/kprobes.c:2460`):

Modules can contribute their own blacklist entries via the
`kprobe_blacklist` section in their ELF.

**4. Other reserved regions** (`kernel/kprobes.c:1560`):

- `jump_label_text_reserved()` -- static keys
- `static_call_text_reserved()` -- static calls
- `find_bug()` -- BUG() locations
- `is_cfi_preamble_symbol()` -- CFI prefix functions (`__cfi_*`, `__pfx_*`)

### Viewable via debugfs

```
/sys/kernel/debug/kprobes/blacklist
```

---

## 11. Kprobe Aggregation (Multi-Handler)

When multiple kprobes are registered at the same address, an "aggregator"
kprobe is created to dispatch to all handlers:

`kernel/kprobes.c:1193-1221`

```c
static int aggr_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    struct kprobe *kp;
    list_for_each_entry_rcu(kp, &p->list, list) {
        if (kp->pre_handler && likely(!kprobe_disabled(kp))) {
            set_kprobe_instance(kp);
            if (kp->pre_handler(kp, regs))
                return 1;
        }
        reset_kprobe_instance();
    }
    return 0;
}

static void aggr_post_handler(struct kprobe *p, struct pt_regs *regs,
                              unsigned long flags)
{
    struct kprobe *kp;
    list_for_each_entry_rcu(kp, &p->list, list) {
        if (kp->post_handler && likely(!kprobe_disabled(kp))) {
            set_kprobe_instance(kp);
            kp->post_handler(kp, regs, flags);
            reset_kprobe_instance();
        }
    }
}
```

```
  Aggregation:

  kprobe_table[hash]
       │
       ▼
  ┌──────────────────┐
  │ aggr_kprobe (ap) │  pre_handler = aggr_pre_handler
  │   .list ──────────┼──► kprobe_A ──► kprobe_B ──► kprobe_C
  └──────────────────┘
```

When `CONFIG_OPTPROBES` is enabled, the aggregator is actually an
`optimized_kprobe` (allocated via `alloc_aggr_kprobe()` at
`kernel/kprobes.c:824`).

---

## 12. Uprobes Architecture

Uprobes provides dynamic instrumentation of user-space programs. Unlike
kprobes, uprobes are **inode-based**: a probe is associated with an
(inode, offset) tuple rather than a virtual address. This means a single
uprobe definition applies to all processes that map the same binary.

```
  Uprobe Architecture:

  ┌────────────────────────────────────────────────────────┐
  │  uprobe_register(inode, offset, ref_ctr_offset, uc)   │
  │                                                        │
  │  uprobes_tree (rb_tree)                                │
  │      │                                                 │
  │  ┌───▼─────────┐                                       │
  │  │  uprobe      │  key = (inode, offset)               │
  │  │  .consumers ─┼──► uc1 ──► uc2 ──► ...              │
  │  │  .arch.insn  │  saved original instruction          │
  │  │  .arch.ixol  │  potentially modified for XOL        │
  │  └──────────────┘                                      │
  │                                                        │
  │  For each process mapping the inode:                   │
  │  ┌─────────────────────────────────────────┐           │
  │  │ Process address space                   │           │
  │  │                                         │           │
  │  │  .text VMA:   ... INT3 (0xCC) ...       │  ← swbp  │
  │  │                                         │           │
  │  │  XOL area:    [trampoline] [slot1] ...  │  ← exec   │
  │  │  (VM_EXEC | VM_DONTCOPY)                │    OOL    │
  │  └─────────────────────────────────────────┘           │
  └────────────────────────────────────────────────────────┘
```

---

## 13. Uprobe Core Data Structures

### `struct uprobe`

`kernel/events/uprobes.c:61`

The internal uprobe descriptor (not exposed to consumers):

```c
struct uprobe {
    struct rb_node      rb_node;        /* node in uprobes_tree */
    refcount_t          ref;
    struct rw_semaphore register_rwsem;
    struct rw_semaphore consumer_rwsem;
    struct list_head    pending_list;
    struct list_head    consumers;       /* list of uprobe_consumer */
    struct inode        *inode;          /* backing file inode */
    union {
        struct rcu_head     rcu;
        struct work_struct  work;
    };
    loff_t              offset;          /* offset within inode */
    loff_t              ref_ctr_offset;  /* SDT reference counter offset */
    unsigned long       flags;

    struct arch_uprobe  arch;            /* arch-specific (insn + ixol) */
};
```

### `struct uprobe_consumer`

`include/linux/uprobes.h:42`

The public interface that callers use to receive probe callbacks:

```c
struct uprobe_consumer {
    int (*handler)(struct uprobe_consumer *self, struct pt_regs *regs,
                   __u64 *data);
    int (*ret_handler)(struct uprobe_consumer *self,
                       unsigned long func,
                       struct pt_regs *regs, __u64 *data);
    bool (*filter)(struct uprobe_consumer *self, struct mm_struct *mm);

    struct list_head cons_node;
    __u64 id;    /* set when uprobe_consumer is registered */
};
```

**Handler return values:**
- `0` -- normal, continue processing
- `UPROBE_HANDLER_REMOVE` (1) -- remove breakpoint from current process
- `UPROBE_HANDLER_IGNORE` (2) -- skip ret_handler for this consumer

### `struct arch_uprobe` (x86)

`arch/x86/include/asm/uprobes.h:25`

```c
struct arch_uprobe {
    union {
        u8  insn[MAX_UINSN_BYTES];  /* original instruction (16 bytes) */
        u8  ixol[MAX_UINSN_BYTES];  /* instruction for XOL execution */
    };

    const struct uprobe_xol_ops *ops;  /* XOL fixup operations */

    union {
        struct {
            s32 offs;
            u8  ilen;
            u8  opc1;
        } branch;               /* for branch instructions */
        struct {
            u8  fixups;         /* UPROBE_FIX_IP | UPROBE_FIX_CALL | ... */
            u8  ilen;
        } defparam;             /* for default (non-branch) instructions */
        struct {
            u8  reg_offset;
            u8  ilen;
        } push;                 /* for push instructions */
    };
};
```

Key constants (`arch/x86/include/asm/uprobes.h`):

```c
#define UPROBE_SWBP_INSN       0xcc     /* INT3 */
#define UPROBE_SWBP_INSN_SIZE  1
#define UPROBE_XOL_SLOT_BYTES  128      /* cache-aligned slot size */
#define MAX_UINSN_BYTES        16
```

### `struct uprobe_task`

`include/linux/uprobes.h:122`

Per-task metadata during single-stepping:

```c
struct uprobe_task {
    enum uprobe_task_state  state;    /* UTASK_RUNNING, _SSTEP, etc. */

    unsigned int            depth;             /* uretprobe nesting depth */
    struct return_instance  *return_instances;  /* uretprobe stack */

    struct return_instance  *ri_pool;
    struct timer_list       ri_timer;
    seqcount_t              ri_seqcount;

    union {
        struct {
            struct arch_uprobe_task autask;
            unsigned long           vaddr;     /* breakpoint vaddr */
        };
        struct {
            struct callback_head    dup_xol_work;
            unsigned long           dup_xol_addr;
        };
    };

    struct uprobe            *active_uprobe;
    unsigned long            xol_vaddr;         /* XOL slot address */
    struct arch_uprobe       *auprobe;
};
```

### Global Uprobe Tree

`kernel/events/uprobes.c:38-46`

```c
static struct rb_root uprobes_tree = RB_ROOT;
static DEFINE_RWLOCK(uprobes_treelock);
static seqcount_rwlock_t uprobes_seqcount = SEQCNT_RWLOCK_ZERO(...);
```

Uprobes are stored in an RB-tree sorted by (inode, offset). The seqcount
enables lockless RCU lookups in the hot path (`find_uprobe_rcu()` at
`kernel/events/uprobes.c:871`).

---

## 14. Uprobe Registration and Inode-Based Tracking

### `uprobe_register()`

`kernel/events/uprobes.c:1358`

```c
struct uprobe *uprobe_register(struct inode *inode,
                loff_t offset, loff_t ref_ctr_offset,
                struct uprobe_consumer *uc)
```

Registration flow:

```
uprobe_register(inode, offset, ref_ctr_offset, uc)
    │
    ├── Validate: uc must have handler or ret_handler
    ├── Validate: inode mapping supports read_folio or shmem
    ├── Validate: offset aligned to UPROBE_SWBP_INSN_SIZE
    │
    ├── alloc_uprobe(inode, offset, ref_ctr_offset)
    │       ├── kzalloc(sizeof(struct uprobe))
    │       ├── Initialize rb_node, consumers list, rwsems
    │       └── insert_uprobe() → uprobes_tree
    │           └── If already exists, reuse existing uprobe
    │
    ├── consumer_add(uprobe, uc)
    │       └── list_add_rcu() to uprobe->consumers
    │       └── Assign unique uc->id
    │
    └── register_for_each_vma(uprobe, uc)
            ├── build_map_info()
            │       └── Walk inode->i_mapping->i_mmap (VMA interval tree)
            │       └── Find all VMAs mapping this file at this offset
            │
            └── For each matching VMA:
                    ├── mmap_write_lock(mm)
                    ├── consumer_filter(uc, mm)  ← check if consumer wants this mm
                    └── install_breakpoint(uprobe, mm, vma, vaddr)
                            ├── prepare_uprobe()
                            │       ├── copy_insn() ← read instruction from file
                            │       └── arch_uprobe_analyze_insn()
                            ├── set_bit(MMF_HAS_UPROBES)
                            └── set_swbp() → uprobe_write_opcode()
                                    └── Replace instruction with INT3 in page
```

### How Breakpoints Are Installed in User Pages

`kernel/events/uprobes.c:473`

`uprobe_write_opcode()` performs a COW (copy-on-write) page replacement:

1. Read the target page via `get_user_page_vma_remote()`
2. Verify the opcode state (not already probed/unprobed)
3. Update reference counter if `ref_ctr_offset` is set
4. Allocate a new anonymous page
5. Copy the old page content
6. Write the breakpoint opcode into the new page
7. Replace the old page with `__replace_page()`

This ensures the original file page is never modified -- the breakpoint
lives only in the process's anonymous page.

### Lazy Installation via `uprobe_mmap()`

`kernel/events/uprobes.c:1565`

When a process maps (mmap) a file that has registered uprobes, the
`uprobe_mmap()` hook is called from `mmap_region()`. It walks the
uprobes_tree to find probes in the mapped range and installs breakpoints:

```c
int uprobe_mmap(struct vm_area_struct *vma)
{
    if (no_uprobe_events()) return 0;  /* fast path: tree empty */
    build_probe_list(inode, vma, ...);
    list_for_each_entry_safe(uprobe, ...) {
        if (filter_chain(uprobe, vma->vm_mm))
            install_breakpoint(uprobe, vma->vm_mm, vma, vaddr);
    }
}
```

---

## 15. XOL (Execute Out-of-Line) Area

### `struct xol_area`

`kernel/events/uprobes.c:108`

Each probed process gets a per-mm XOL area -- an anonymous executable
mapping where copies of probed instructions are executed:

```c
struct xol_area {
    wait_queue_head_t   wq;       /* if all slots are busy */
    unsigned long       *bitmap;  /* 0 = free slot */
    struct page         *page;    /* single page of instruction slots */
    unsigned long       vaddr;    /* virtual address of the XOL page */
};
```

### XOL Area Layout

```
  XOL Area (1 page, UINSNS_PER_PAGE slots):

  ┌──────────────────────────────────────┐  ← area->vaddr
  │ Slot 0: uretprobe trampoline (INT3) │  ← reserved, always allocated
  ├──────────────────────────────────────┤
  │ Slot 1: copy of probed insn A       │  ← 128 bytes per slot
  ├──────────────────────────────────────┤
  │ Slot 2: copy of probed insn B       │
  ├──────────────────────────────────────┤
  │ ...                                  │
  └──────────────────────────────────────┘

  VMA flags: VM_EXEC | VM_MAYEXEC | VM_DONTCOPY | VM_IO
  Mapped via _install_special_mapping() with name "[uprobes]"
```

### Creation

`kernel/events/uprobes.c:1714`

```c
static struct xol_area *__create_xol_area(unsigned long vaddr)
{
    area = kzalloc(sizeof(*area), GFP_KERNEL);
    area->bitmap = kcalloc(BITS_TO_LONGS(UINSNS_PER_PAGE), ...);
    area->page = alloc_page(GFP_HIGHUSER | __GFP_ZERO);

    /* Reserve slot 0 for uretprobe trampoline */
    set_bit(0, area->bitmap);
    insns = arch_uprobe_trampoline(&insns_size);
    arch_uprobe_copy_ixol(area->page, 0, insns, insns_size);

    xol_add_vma(mm, area);  /* mmap the page into user space */
}
```

The XOL vaddr is placed near `TASK_SIZE - PAGE_SIZE` (high end of the
address space).

### Slot Allocation

`kernel/events/uprobes.c:1826`

When a thread hits a uprobe, it must obtain an XOL slot to execute the
original instruction. If all slots are busy, the thread waits:

```c
static bool xol_get_insn_slot(struct uprobe *uprobe, struct uprobe_task *utask)
{
    struct xol_area *area = get_xol_area();
    wait_event(area->wq,
        (slot_nr = xol_get_slot_nr(area)) < UINSNS_PER_PAGE);

    utask->xol_vaddr = area->vaddr + slot_nr * UPROBE_XOL_SLOT_BYTES;
    arch_uprobe_copy_ixol(area->page, utask->xol_vaddr,
                          &uprobe->arch.ixol, sizeof(uprobe->arch.ixol));
}
```

---

## 16. Uprobe Hit Handling (handle_swbp)

### `handle_swbp()`

`kernel/events/uprobes.c:2651`

When user-space executes INT3 at a probed address, the kernel's trap handler
calls `handle_swbp()`:

```
  handle_swbp(regs)
      │
      ├── bp_vaddr = uprobe_get_swbp_addr(regs)
      │
      ├── [if bp_vaddr == trampoline_vaddr]
      │       └── uprobe_handle_trampoline()   ← uretprobe return path
      │
      ├── rcu_read_lock_trace()
      ├── find_active_uprobe_rcu(bp_vaddr)
      │       ├── find_uprobe_rcu(inode, offset)  ← RB-tree lookup
      │       └── Verify INT3 is still at that address
      │
      ├── instruction_pointer_set(regs, bp_vaddr)  ← fix IP
      │
      ├── get_utask()   ← allocate uprobe_task if needed
      │
      ├── handler_chain(uprobe, regs)
      │       ├── For each uprobe_consumer:
      │       │       ├── uc->handler(uc, regs, &cookie)
      │       │       └── [if ret_handler] alloc return_instance
      │       │
      │       └── [if ret_handler consumers]
      │               └── prepare_uretprobe(uprobe, regs, ri)
      │
      ├── arch_uprobe_skip_sstep()  ← try emulation first
      │
      └── [if can't skip] pre_ssout(uprobe, regs, bp_vaddr)
              ├── xol_get_insn_slot()  ← allocate XOL slot
              ├── arch_uprobe_pre_xol()
              │       └── Set IP to XOL slot, enable TF (single-step)
              └── utask->state = UTASK_SSTEP
```

After `handle_swbp()` returns, the thread resumes in user-space at the XOL
slot. It single-steps through the copied instruction, traps again (debug
exception), and the kernel calls `handle_singlestep()`:

```c
static void handle_singlestep(struct uprobe_task *utask, struct pt_regs *regs)
{
    if (utask->state == UTASK_SSTEP_ACK)
        arch_uprobe_post_xol(&uprobe->arch, regs);  /* fixup IP, flags */
    else if (utask->state == UTASK_SSTEP_TRAPPED)
        arch_uprobe_abort_xol(&uprobe->arch, regs);

    xol_free_insn_slot(utask);
}
```

---

## 17. Uretprobes (User Return Probes)

### Mechanism

Uretprobes work similarly to kretprobes: the return address is hijacked
to point to a trampoline (slot 0 of the XOL area), and when the function
returns, the trampoline's INT3 triggers the return handler.

### `struct return_instance`

`include/linux/uprobes.h:155`

```c
struct return_instance {
    struct hprobe           hprobe;
    unsigned long           func;            /* probed function address */
    unsigned long           stack;           /* stack pointer at entry */
    unsigned long           orig_ret_vaddr;  /* original return address */
    bool                    chained;         /* nested call */
    int                     cons_cnt;        /* number of session consumers */
    struct return_instance  *next;           /* stack of instances */
    struct rcu_head         rcu;
    struct return_consumer  consumer;
    struct return_consumer  *extra_consumers;
};
```

### Hybrid Lifetime (hprobe)

`include/linux/uprobes.h:113`

To handle the fact that uprobes can be unregistered while return instances
are outstanding, a "hybrid probe" (`struct hprobe`) manages the lifecycle:

```c
struct hprobe {
    enum hprobe_state state;   /* LEASED, STABLE, GONE, CONSUMED */
    int srcu_idx;
    struct uprobe *uprobe;
};
```

States:
- `HPROBE_LEASED` -- uprobe is SRCU-protected (short-term)
- `HPROBE_STABLE` -- uprobe is refcounted (long-term, after SRCU expiry)
- `HPROBE_GONE` -- uprobe has been freed, pointer invalid
- `HPROBE_CONSUMED` -- handler has been called

### `prepare_uretprobe()`

`kernel/events/uprobes.c:2194`

```c
static void prepare_uretprobe(struct uprobe *uprobe, struct pt_regs *regs,
                              struct return_instance *ri)
{
    trampoline_vaddr = uprobe_get_trampoline_vaddr();
    orig_ret_vaddr = arch_uretprobe_hijack_return_addr(trampoline_vaddr, regs);

    ri->func = instruction_pointer(regs);
    ri->stack = user_stack_pointer(regs);
    ri->orig_ret_vaddr = orig_ret_vaddr;

    hprobe_init_leased(&ri->hprobe, uprobe, srcu_idx);
    ri->next = utask->return_instances;
    rcu_assign_pointer(utask->return_instances, ri);
}
```

Maximum nesting depth: `MAX_URETPROBE_DEPTH = 64` (`include/linux/uprobes.h:40`)

### Return Handler Path

`kernel/events/uprobes.c:2578`

When the function returns, it jumps to the trampoline (slot 0 of XOL area
containing INT3). `uprobe_handle_trampoline()` is called, which:

1. Finds the matching `return_instance` by stack pointer
2. Calls `handle_uretprobe_chain()` for each consumer's `ret_handler`
3. Restores the original return address
4. Frees the return_instance

---

## 18. Uprobe Consumers

### Session Semantics

If an `uprobe_consumer` has both `handler` and `ret_handler`, it operates in
"session" mode. The `handler` produces a cookie (`__u64 *data`) that is
passed to the `ret_handler` for correlation:

```c
struct uprobe_consumer {
    int (*handler)(struct uprobe_consumer *self, struct pt_regs *regs,
                   __u64 *data);
    int (*ret_handler)(struct uprobe_consumer *self,
                       unsigned long func,
                       struct pt_regs *regs, __u64 *data);
    bool (*filter)(struct uprobe_consumer *self, struct mm_struct *mm);
};
```

### Filter Callback

The `filter` callback allows selective per-process probing. When
`uprobe_register()` iterates VMAs, it calls `consumer_filter()` to decide
whether to install the breakpoint in each process. This is used by perf to
probe only specific PIDs.

### Consumer Lifecycle

```c
/* Register */
uprobe = uprobe_register(inode, offset, 0, &my_consumer);

/* Unregister */
uprobe_unregister_nosync(uprobe, &my_consumer);
uprobe_unregister_sync();  /* Wait for RCU grace periods */
```

The sync/nosync split (`kernel/events/uprobes.c:1301-1337`) allows batching
multiple unregistrations before the expensive `synchronize_rcu_tasks_trace()`
and `synchronize_srcu()` calls.

---

## 19. BPF Attachment to Kprobes/Uprobes

### BPF Kprobe Programs

BPF programs of type `BPF_PROG_TYPE_KPROBE` can be attached to kprobes
and kretprobes. The kernel creates `trace_kprobe` events
(`kernel/trace/trace_kprobe.c:58`):

```c
struct trace_kprobe {
    struct dyn_event    devent;
    struct kretprobe    rp;     /* Use rp.kp for kprobe use */
    unsigned long __percpu *nhit;
    const char          *symbol;
    struct trace_probe  tp;
};
```

BPF attaches via perf_event with `perf_event_attr.type = PERF_TYPE_TRACEPOINT`
or via `bpf_program__attach_kprobe()` in libbpf.

### BPF Uprobe Programs

Similarly, `BPF_PROG_TYPE_KPROBE` programs can attach to uprobes (the
program type is the same). The attachment creates a `trace_uprobe` event
and registers an `uprobe_consumer` whose handler invokes the BPF program.

### Attachment Path

```
  bpf(BPF_LINK_CREATE) / perf_event_open()
      │
      ├── [kprobe]  → perf_trace_event_init()
      │                  → perf_kprobe_init()
      │                      → register_kprobe() / register_kretprobe()
      │
      └── [uprobe]  → perf_trace_event_init()
                         → perf_uprobe_init()
                             → uprobe_register()
```

### BPF Cookie

Recent kernels support `bpf_get_attach_cookie()` which retrieves a 64-bit
cookie value associated with the BPF link. This is passed through the
uprobe consumer's `__u64 *data` mechanism.

---

## 20. perf_event Integration

### perf kprobe/uprobe Events

`perf_event_open()` can create kprobe and uprobe events directly:

```c
struct perf_event_attr attr = {
    .type = PERF_TYPE_TRACEPOINT,     /* or use dynamic PMU */
    .config = event_id,               /* from tracefs */
};
```

Or using the kprobe/uprobe PMU:

```c
/* kprobe PMU */
attr.type = perf_kprobe_type;  /* from /sys/bus/event_source/devices/kprobe/type */
attr.kprobe_func = "do_sys_open";
attr.probe_offset = 0;

/* uprobe PMU */
attr.type = perf_uprobe_type;
attr.uprobe_path = "/usr/bin/bash";
attr.probe_offset = 0x12345;
```

### perf_event Text Poke Notifications

The kprobes subsystem emits perf text poke events when arming/disarming
probes, enabling perf tools to track code modifications:

```c
/* In arch_arm_kprobe(): */
perf_event_text_poke(p->addr, &p->opcode, 1, &int3, 1);

/* In arch_copy_kprobe(): */
perf_event_text_poke(p->ainsn.insn, NULL, 0, buf, len);
```

---

## 21. Jprobes (Deprecated)

Jprobes (jump probes) were a kprobes variant that allowed intercepting
function calls with access to the function arguments through a handler with
the same prototype as the probed function. They were removed in Linux 4.15
(commit removing jprobes was merged in 2017).

Jprobes worked by:
1. Setting the instruction pointer to the user's handler function
2. The handler could inspect arguments via its own parameters
3. The handler called `jprobe_return()` to resume execution

They were superseded by:
- BPF programs attached to kprobes (which can read function arguments)
- ftrace with `FTRACE_OPS_FL_SAVE_REGS`
- `kprobe.pre_handler` with direct `pt_regs` access

---

## 22. Debugfs Interface

`kernel/kprobes.c:3032`

```
/sys/kernel/debug/kprobes/
├── list           ← All registered kprobes with status
│                     Format: addr  type  symbol+offset  [FLAGS]
│                     Flags: [GONE] [DISABLED] [OPTIMIZED] [FTRACE]
│
├── enabled        ← Read/write: 0 = all disarmed, 1 = all armed
│
└── blacklist      ← All blacklisted address ranges
                     Format: 0x%px-0x%px  %ps
```

The `enabled` file controls global arming/disarming via
`arm_all_kprobes()` / `disarm_all_kprobes()`:

```c
/* kernel/kprobes.c:2991 */
/* Write: echo 0 > enabled  → disarms all kprobes */
/* Write: echo 1 > enabled  → re-arms all kprobes */
```

---

## 23. Architecture-Specific Details

### x86 Specifics

**Kprobes:**
- Breakpoint: `INT3` (0xCC), single byte
- Instruction copy: on executable pages allocated via `execmem_alloc()`
  within +/- 2GB for RIP-relative addressing fixup
- IBT (Indirect Branch Tracking): `arch_adjust_kprobe_addr()` skips ENDBR
  instructions at function entry (`core.c:373`)
- Optprobes: replaces up to `MAX_OPTIMIZED_LENGTH` (20) bytes with a 5-byte
  JMP to a detour buffer
- Emulation: most control-flow instructions are emulated in-kernel rather
  than single-stepped

**Uprobes:**
- Same `INT3` (0xCC) breakpoint
- XOL slots: 128 bytes each, cache-aligned (`UPROBE_XOL_SLOT_BYTES`)
- RIP-relative fixup for 64-bit code
- Post-XOL fixups: `UPROBE_FIX_IP`, `UPROBE_FIX_CALL`, `UPROBE_FIX_RIP_*`
- Trampoline: single INT3 in slot 0 of XOL area

### arm64 Specifics

**Kprobes:**
- Breakpoint: `BRK #0x004` (4 bytes, fixed-width ISA)
- All instructions are 4 bytes, simplifying instruction boundary detection
- Single-step uses hardware debug registers (MDSCR_EL1.SS)
- No optprobes on arm64 (as of this writing)

**Uprobes:**
- Breakpoint: `BRK #0x004`
- XOL area works similarly to x86
- Fixed instruction width eliminates alignment concerns

### Key Differences Summary

| Feature | x86 | arm64 |
|---------|-----|-------|
| Breakpoint insn | `INT3` (1 byte) | `BRK` (4 bytes) |
| Instruction width | Variable (1-15 bytes) | Fixed (4 bytes) |
| Optprobes | Yes (JMP rel32) | No |
| RIP-relative fixup | Required (x86-64) | Not applicable |
| Emulation | Extensive (call, jmp, jcc, loop, ret) | Simpler |
| Boosting | Yes (append JMP to insn copy) | N/A |
| IBT/BTI skip | ENDBR skip | BTI skip |

---

## Initialization

### Kprobes Init

`kernel/kprobes.c:2712` (registered as `early_initcall`):

```c
static int __init init_kprobes(void)
{
    for (i = 0; i < KPROBE_TABLE_SIZE; i++)
        INIT_HLIST_HEAD(&kprobe_table[i]);

    populate_kprobe_blacklist(__start_kprobe_blacklist,
                              __stop_kprobe_blacklist);

    /* Resolve kretprobe blacklist symbols */
    for (i = 0; kretprobe_blacklist[i].name != NULL; i++)
        kretprobe_blacklist[i].addr = kprobe_lookup_name(...);

    kprobes_all_disarmed = false;
    arch_init_kprobes();
    register_die_notifier(&kprobe_exceptions_nb);  /* priority 0x7fffffff */
    kprobe_register_module_notifier();
    kprobes_initialized = (err == 0);
}
```

Optprobe optimization is deferred to `subsys_initcall` (`init_optprobes` at
`kernel/kprobes.c:2758`) because it depends on `synchronize_rcu_tasks()`
which requires ksoftirqd.

### Uprobes Init

`include/linux/uprobes.h:187`:

```c
extern void __init uprobes_init(void);
```

Initializes the mmap mutex array and SRCU for uretprobe lifetime management.
