# Linux Kernel Jump Labels, Static Keys, and Static Calls Overview

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Static Keys](#static-keys)
3. [Jump Label Implementation](#jump-label-implementation)
4. [Static Calls](#static-calls)
5. [x86 Implementation](#x86-implementation)
6. [Use in Hot Paths](#use-in-hot-paths)
7. [Reference Counting with static_key_count](#reference-counting-with-static_key_count)
8. [text_poke_bp -- Breakpoint-Based Safe Patching](#text_poke_bp----breakpoint-based-safe-patching)
9. [Module Integration](#module-integration)
10. [Performance Benefits](#performance-benefits)

---

## Architecture Overview

Jump labels, static keys, and static calls are three related kernel mechanisms
that eliminate runtime overhead from conditional branches and indirect function
calls by patching live kernel text. Instead of evaluating conditions or
dereferencing function pointers at every execution, these mechanisms modify the
machine code itself so that the fast path executes a NOP (no operation) or a
direct CALL/JMP instruction.

```
+------------------------------------------------------------------+
|              Zero-Overhead Branch/Call Primitives                 |
+------------------------------------------------------------------+
|                                                                  |
|  Traditional approach:                                           |
|  ---------------------                                           |
|  if (some_flag)         -->  load flag, compare, branch          |
|      do_something();         (memory load + branch prediction)   |
|                                                                  |
|  func_ptr(args);        -->  load pointer, indirect call         |
|                              (retpoline penalty on x86)          |
|                                                                  |
|  Static key approach:                                            |
|  --------------------                                            |
|  if (static_branch_unlikely(&key))                               |
|      do_something();    -->  NOP (5 bytes, zero cost)            |
|                              patched to JMP when enabled         |
|                                                                  |
|  Static call approach:                                           |
|  --------------------                                            |
|  static_call(name)()    -->  CALL func (direct, 5 bytes)         |
|                              patched to CALL other_func          |
|                              when updated                        |
|                                                                  |
+------------------------------------------------------------------+
|                                                                  |
|  Key insight: modifying code is SLOW (machine-wide sync),        |
|  but the resulting branch/call is FAST (zero overhead).          |
|  Perfect for rarely-toggled, frequently-executed paths.          |
|                                                                  |
+------------------------------------------------------------------+
```

The three mechanisms serve complementary roles:

- **Static keys / Jump labels**: Replace conditional branches with NOP or JMP
  instructions. A 5-byte NOP replaces `if (flag)` checks that are almost always
  false (or true). Toggling the key patches all associated sites.
- **Static calls**: Replace indirect function pointer calls with direct CALL or
  JMP instructions. This avoids retpoline overhead on speculative-execution
  mitigated systems and eliminates pointer dereference cost.
- **Text patching infrastructure**: The underlying mechanism (on x86,
  `text_poke_bp`) that safely modifies kernel text on live SMP systems using
  INT3 breakpoints as a synchronization primitive.

All three are defined across:

| Component | Header | Implementation |
|-----------|--------|----------------|
| Static keys | `include/linux/jump_label.h` | `kernel/jump_label.c` |
| Jump labels (x86) | `arch/x86/include/asm/jump_label.h` | `arch/x86/kernel/jump_label.c` |
| Static calls | `include/linux/static_call.h`, `include/linux/static_call_types.h` | `kernel/static_call_inline.c`, `kernel/static_call.c` |
| Static calls (x86) | `arch/x86/include/asm/static_call.h` | `arch/x86/kernel/static_call.c` |
| Text patching (x86) | `arch/x86/include/asm/text-patching.h` | `arch/x86/kernel/alternative.c` |

---

## Static Keys

Static keys are the user-facing API built on top of jump labels. They provide
type-safe boolean toggles that compile down to either NOP or JMP instructions.

### Core Data Structure

From `include/linux/jump_label.h:85`:

```c
struct static_key {
    atomic_t enabled;
#ifdef CONFIG_JUMP_LABEL
    /*
     * bit 0 => 1 if key is initially true
     *          0 if initially false
     * bit 1 => 1 if points to struct static_key_mod
     *          0 if points to struct jump_entry
     */
    union {
        unsigned long type;
        struct jump_entry *entries;
        struct static_key_mod *next;
    };
#endif
};
```

The `enabled` field is an atomic reference count. The union encodes both a
pointer (to jump entries or module chain) and flag bits in the two lowest bits:

- **Bit 0 (`JUMP_TYPE_TRUE`)**: Initial key state (true=1, false=0)
- **Bit 1 (`JUMP_TYPE_LINKED`)**: Whether the key uses a linked list of
  `static_key_mod` structures (set when modules reference the key)

### Type Wrappers

From `include/linux/jump_label.h:362`:

```c
struct static_key_true {
    struct static_key key;
};

struct static_key_false {
    struct static_key key;
};
```

These wrappers enable compile-time type differentiation so the macros can emit
the correct instruction (NOP vs JMP) at each branch site.

### Definition Macros

From `include/linux/jump_label.h:373`:

```c
#define DEFINE_STATIC_KEY_TRUE(name)  \
    struct static_key_true name = STATIC_KEY_TRUE_INIT

#define DEFINE_STATIC_KEY_FALSE(name) \
    struct static_key_false name = STATIC_KEY_FALSE_INIT

/* Read-only after init variants */
#define DEFINE_STATIC_KEY_TRUE_RO(name)  \
    struct static_key_true name __ro_after_init = STATIC_KEY_TRUE_INIT

#define DEFINE_STATIC_KEY_FALSE_RO(name) \
    struct static_key_false name __ro_after_init = STATIC_KEY_FALSE_INIT

/* Array variants */
#define DEFINE_STATIC_KEY_ARRAY_TRUE(name, count)      \
    struct static_key_true name[count] = {             \
        [0 ... (count) - 1] = STATIC_KEY_TRUE_INIT,   \
    }

/* Config-dependent: picks TRUE or FALSE based on Kconfig */
#define DEFINE_STATIC_KEY_MAYBE(cfg, name) \
    __PASTE(_DEFINE_STATIC_KEY_, IS_ENABLED(cfg))(name)
```

### Branch Macros

From `include/linux/jump_label.h:485` (with `CONFIG_JUMP_LABEL` enabled):

```c
#define static_branch_likely(x)                         \
({                                                      \
    bool branch;                                        \
    if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))  \
        branch = !arch_static_branch(&(x)->key, true);                    \
    else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
        branch = !arch_static_branch_jump(&(x)->key, true);               \
    else                                                \
        branch = ____wrong_branch_error();              \
    likely_notrace(branch);                             \
})

#define static_branch_unlikely(x)                       \
({                                                      \
    bool branch;                                        \
    if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))  \
        branch = arch_static_branch_jump(&(x)->key, false);               \
    else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
        branch = arch_static_branch(&(x)->key, false);                    \
    else                                                \
        branch = ____wrong_branch_error();              \
    unlikely_notrace(branch);                           \
})
```

The key insight is the truth table from the header comment at
`include/linux/jump_label.h:434`:

```
 type\branch |  likely (1)         |  unlikely (0)
 ------------+---------------------+------------------
  true  (1)  |  NOP                |  JMP L
             |  <br-stmts>         |  1: ...
             |  L: ...             |  L: <br-stmts>
             |                     |     jmp 1b
 ------------+---------------------+------------------
  false (0)  |  JMP L              |  NOP
             |  <br-stmts>         |  1: ...
             |  L: ...             |  L: <br-stmts>
             |                     |     jmp 1b

 Logic:
   dynamic:  instruction = enabled ^ branch
   static:   instruction = type ^ branch
```

When the initial state matches the expected likelihood, a NOP is emitted
(zero cost). When they differ, a JMP is emitted to take the out-of-line path.

### Control API

```c
/* Boolean enable/disable (non-refcounted) */
static_branch_enable(x)       /* Set key to true, patch NOP->JMP or JMP->NOP */
static_branch_disable(x)      /* Set key to false */

/* Reference-counted (branch enabled when count != 0) */
static_branch_inc(x)          /* Increment; first inc patches code */
static_branch_dec(x)          /* Decrement; last dec patches code */
```

---

## Jump Label Implementation

The jump label subsystem provides the low-level mechanism that static keys build
upon. It manages a table of patch sites, sorts them by key, and coordinates
code patching when a static key changes state.

### The Jump Entry Table

From `include/linux/jump_label.h:117`:

```c
struct jump_entry {
    s32 code;       /* Relative offset to the instruction site */
    s32 target;     /* Relative offset to the branch target */
    long key;       /* Relative offset to the static_key (low bits: flags) */
};
```

All fields use PC-relative offsets (`CONFIG_HAVE_ARCH_JUMP_LABEL_RELATIVE`) so
the table is position-independent under KASLR. The accessor functions at
`include/linux/jump_label.h:123` decode these:

```c
static inline unsigned long jump_entry_code(const struct jump_entry *entry)
{
    return (unsigned long)&entry->code + entry->code;
}

static inline unsigned long jump_entry_target(const struct jump_entry *entry)
{
    return (unsigned long)&entry->target + entry->target;
}

static inline struct static_key *jump_entry_key(const struct jump_entry *entry)
{
    long offset = entry->key & ~3L;
    return (struct static_key *)((unsigned long)&entry->key + offset);
}
```

The low bits of `key` encode flags:
- **Bit 0**: Branch direction (0=unlikely, 1=likely)
- **Bit 1**: Whether the site is in an `__init` section

### Initialization

From `kernel/jump_label.c:525`:

```c
void __init jump_label_init(void)
{
    struct jump_entry *iter_start = __start___jump_table;
    struct jump_entry *iter_stop = __stop___jump_table;
    struct static_key *key = NULL;
    struct jump_entry *iter;

    /* ... */
    jump_label_sort_entries(iter_start, iter_stop);

    for (iter = iter_start; iter < iter_stop; iter++) {
        struct static_key *iterk;
        bool in_init;

        /* rewrite NOPs */
        if (jump_label_type(iter) == JUMP_LABEL_NOP)
            arch_jump_label_transform_static(iter, JUMP_LABEL_NOP);

        in_init = init_section_contains((void *)jump_entry_code(iter), 1);
        jump_entry_set_init(iter, in_init);

        iterk = jump_entry_key(iter);
        if (iterk == key)
            continue;

        key = iterk;
        static_key_set_entries(key, iter);
    }
    static_key_initialized = true;
}
```

At boot, the linker-generated `__jump_table` section (bounded by
`__start___jump_table` and `__stop___jump_table`) is sorted by key pointer. Each
key is then linked to its first jump entry. Entries in `__init` sections are
flagged so they can be skipped after init memory is freed.

### Update Path

From `kernel/jump_label.c:896`:

```c
static void jump_label_update(struct static_key *key)
{
    struct jump_entry *stop = __stop___jump_table;
    bool init = system_state < SYSTEM_RUNNING;
    struct jump_entry *entry;

    if (static_key_linked(key)) {
        __jump_label_mod_update(key);
        return;
    }
    /* ... */
    entry = static_key_entries(key);
    if (entry)
        __jump_label_update(key, entry, stop, init);
}
```

The update iterates over all jump entries for a given key and calls the
arch-specific transform function for each site. The NOP/JMP decision is computed
by `jump_label_type()` at `kernel/jump_label.c:455`:

```c
static enum jump_label_type jump_label_type(struct jump_entry *entry)
{
    struct static_key *key = jump_entry_key(entry);
    bool enabled = static_key_enabled(key);
    bool branch = jump_entry_is_branch(entry);

    /* instruction = enabled ^ branch */
    return enabled ^ branch;
}
```

### Sealing Read-Only Keys

From `kernel/jump_label.c:582`:

```c
void jump_label_init_ro(void)
{
    /* ... */
    for (iter = iter_start; iter < iter_stop; iter++) {
        struct static_key *iterk = jump_entry_key(iter);

        if (!is_kernel_ro_after_init((unsigned long)iterk))
            continue;

        if (static_key_sealed(iterk))
            continue;

        static_key_seal(iterk);
    }
}
```

Keys marked `__ro_after_init` (via `DEFINE_STATIC_KEY_TRUE_RO` /
`DEFINE_STATIC_KEY_FALSE_RO`) are sealed after init. Sealed keys cannot be
modified at runtime, avoiding the need to maintain module tracking structures.

---

## Static Calls

Static calls replace indirect function pointer calls (`func_ptr(args)`) with
direct call instructions. This eliminates the performance penalty of retpoline
mitigations for Spectre v2 and removes the overhead of pointer dereference and
indirect branch prediction.

### Core Data Structures

From `include/linux/static_call_types.h:32`:

```c
struct static_call_site {
    s32 addr;   /* Relative offset to the call site instruction */
    s32 key;    /* Relative offset to static_call_key (low bits: flags) */
};
```

Flags in the low bits of `key` (from `include/linux/static_call_types.h:24`):

```c
#define STATIC_CALL_SITE_TAIL   1UL   /* tail call (JMP, not CALL) */
#define STATIC_CALL_SITE_INIT   2UL   /* site is in init section */
#define STATIC_CALL_SITE_FLAGS  3UL
```

The key structure (from `include/linux/static_call_types.h:61`, with
`CONFIG_HAVE_STATIC_CALL_INLINE`):

```c
struct static_call_key {
    void *func;
    union {
        /* bit 0: 0 = mods, 1 = sites */
        unsigned long type;
        struct static_call_mod *mods;
        struct static_call_site *sites;
    };
};
```

Module tracking (from `include/linux/static_call.h:169`):

```c
struct static_call_mod {
    struct static_call_mod *next;
    struct module *mod;          /* for vmlinux, mod == NULL */
    struct static_call_site *sites;
};
```

### API Overview

From `include/linux/static_call.h:15`:

```c
/*
 * API overview:
 *
 *   DECLARE_STATIC_CALL(name, func);
 *   DEFINE_STATIC_CALL(name, func);
 *   DEFINE_STATIC_CALL_NULL(name, typename);
 *   DEFINE_STATIC_CALL_RET0(name, typename);
 *
 *   static_call(name)(args...);
 *   static_call_cond(name)(args...);
 *   static_call_update(name, func);
 *   static_call_query(name);
 */
```

### Definition and Usage

From `include/linux/static_call.h:187`:

```c
#define DEFINE_STATIC_CALL(name, _func)                 \
    DECLARE_STATIC_CALL(name, _func);                   \
    struct static_call_key STATIC_CALL_KEY(name) = {    \
        .func = _func,                                  \
        .type = 1,                                      \
    };                                                  \
    ARCH_DEFINE_STATIC_CALL_TRAMP(name, _func)
```

Each static call has two components:
1. A **key** (`__SCK__name`) holding the current function pointer and site list
2. A **trampoline** (`__SCT__name`) -- a JMP instruction that always points to
   the current target function

Usage example from the header:

```c
/* Define with initial target func_a */
DEFINE_STATIC_CALL(my_name, func_a);

/* Call func_a() -- compiles to direct CALL */
static_call(my_name)(arg1, arg2);

/* Update to point to func_b() -- patches all sites */
static_call_update(my_name, &func_b);

/* Now calls func_b() */
static_call(my_name)(arg1, arg2);
```

### Update Mechanism

From `kernel/static_call_inline.c:134`:

```c
void __static_call_update(struct static_call_key *key, void *tramp, void *func)
{
    struct static_call_site *site, *stop;
    struct static_call_mod *site_mod, first;

    cpus_read_lock();
    static_call_lock();

    if (key->func == func)
        goto done;

    key->func = func;

    arch_static_call_transform(NULL, tramp, func, false);

    /* ... iterate all sites across vmlinux and modules ... */
    for (site_mod = &first; site_mod; site_mod = site_mod->next) {
        /* ... */
        for (site = site_mod->sites;
             site < stop && static_call_key(site) == key; site++) {
            void *site_addr = static_call_addr(site);
            /* ... */
            arch_static_call_transform(site_addr, NULL, func,
                                       static_call_is_tail(site));
        }
    }

done:
    static_call_unlock();
    cpus_read_unlock();
}
```

The update first patches the trampoline (so any in-flight calls via the
trampoline get the new target), then iterates all inline call sites and patches
each one. This requires `cpus_read_lock()` and a mutex for serialization.

### NULL and RET0 Variants

From `include/linux/static_call.h:195`:

```c
#define DEFINE_STATIC_CALL_NULL(name, _func)            \
    DECLARE_STATIC_CALL(name, _func);                   \
    struct static_call_key STATIC_CALL_KEY(name) = {    \
        .func = NULL,                                   \
        .type = 1,                                      \
    };                                                  \
    ARCH_DEFINE_STATIC_CALL_NULL_TRAMP(name)

#define DEFINE_STATIC_CALL_RET0(name, _func)            \
    DECLARE_STATIC_CALL(name, _func);                   \
    struct static_call_key STATIC_CALL_KEY(name) = {    \
        .func = __static_call_return0,                  \
        .type = 1,                                      \
    };                                                  \
    ARCH_DEFINE_STATIC_CALL_RET0_TRAMP(name)
```

- **NULL**: The trampoline is patched to a RET instruction. On inline
  architectures, the call site becomes a NOP. `static_call_cond()` provides a
  safe way to call potentially-NULL static calls.
- **RET0**: On x86, the 5-byte CALL instruction is replaced with a 5-byte
  `xor %eax, %eax` (`0x2e 0x2e 0x2e 0x31 0xc0`), completely eliminating the
  function call while returning 0.

---

## x86 Implementation

### Jump Label Patching (x86)

The x86 jump label implementation uses either 2-byte (`JMP8`) or 5-byte
(`JMP32`) short/near jumps, with corresponding NOP sizes.

From `arch/x86/include/asm/jump_label.h:34`:

```c
static __always_inline bool arch_static_branch(struct static_key * const key,
                                                const bool branch)
{
    asm goto(ARCH_STATIC_BRANCH_ASM("%c0 + %c1", "%l[l_yes]")
        : :  "i" (key), "i" (branch) : : l_yes);

    return false;
l_yes:
    return true;
}
```

The `ARCH_STATIC_BRANCH_ASM` macro (at line 29) emits a 5-byte NOP and a
`__jump_table` section entry:

```c
#define ARCH_STATIC_BRANCH_ASM(key, label)            \
    "1: .byte " __stringify(BYTES_NOP5) "\n\t"        \
    JUMP_TABLE_ENTRY(key, label)
```

The jump table entry records the instruction address, target label, and key
reference using PC-relative offsets:

```c
#define JUMP_TABLE_ENTRY(key, label)                  \
    ".pushsection __jump_table,  \"aw\" \n\t"         \
    _ASM_ALIGN "\n\t"                                 \
    ".long 1b - . \n\t"                               \
    ".long " label " - . \n\t"                        \
    _ASM_PTR " " key " - . \n\t"                      \
    ".popsection \n\t"
```

The arch-level transform function at `arch/x86/kernel/jump_label.c:82`:

```c
static __always_inline void
__jump_label_transform(struct jump_entry *entry,
                       enum jump_label_type type, int init)
{
    const struct jump_label_patch jlp = __jump_label_patch(entry, type);

    if (init || system_state == SYSTEM_BOOTING) {
        text_poke_early((void *)jump_entry_code(entry), jlp.code, jlp.size);
        return;
    }

    text_poke_bp((void *)jump_entry_code(entry), jlp.code, jlp.size, NULL);
}
```

During boot (`text_poke_early`), code can be patched directly because only one
CPU is running. After boot, `text_poke_bp` uses the INT3 breakpoint protocol
for safe SMP patching.

### Batch Patching (x86)

x86 supports batch patching via `HAVE_JUMP_LABEL_BATCH`. From
`arch/x86/kernel/jump_label.c:123`:

```c
bool arch_jump_label_transform_queue(struct jump_entry *entry,
                                     enum jump_label_type type)
{
    /* ... */
    jlp = __jump_label_patch(entry, type);
    text_poke_queue((void *)jump_entry_code(entry), jlp.code, jlp.size, NULL);
    return true;
}

void arch_jump_label_transform_apply(void)
{
    mutex_lock(&text_mutex);
    text_poke_finish();
    mutex_unlock(&text_mutex);
}
```

Batch patching queues multiple sites and applies them together with a single
sync cycle, significantly reducing overhead when many sites share a key.

### Static Call Patching (x86)

From `arch/x86/kernel/static_call.c:7`:

```c
enum insn_type {
    CALL = 0,  /* site call */
    NOP = 1,   /* site cond-call */
    JMP = 2,   /* tramp / site tail-call */
    RET = 3,   /* tramp / site cond-tail-call */
    JCC = 4,
};
```

The trampoline signature is a 3-byte UD1 instruction placed after the 5-byte
JMP, serving as both a unique identifier and a speculation barrier
(`arch/x86/kernel/static_call.c:20`):

```c
static const u8 tramp_ud[] = { 0x0f, 0xb9, 0xcc };
```

The RET0 optimization uses a 5-byte `xor %eax, %eax` with CS segment override
prefixes (`arch/x86/kernel/static_call.c:25`):

```c
static const u8 xor5rax[] = { 0x2e, 0x2e, 0x2e, 0x31, 0xc0 };
```

The core transform function at `arch/x86/kernel/static_call.c:53`:

```c
static void __ref __static_call_transform(void *insn, enum insn_type type,
                                          void *func, bool modinit)
{
    const void *emulate = NULL;
    int size = CALL_INSN_SIZE;
    const void *code;

    switch (type) {
    case CALL:
        func = callthunks_translate_call_dest(func);
        code = text_gen_insn(CALL_INSN_OPCODE, insn, func);
        if (func == &__static_call_return0) {
            emulate = code;
            code = &xor5rax;    /* Replace CALL with XOR EAX,EAX */
        }
        break;

    case NOP:
        code = x86_nops[5];
        break;

    case JMP:
        code = text_gen_insn(JMP32_INSN_OPCODE, insn, func);
        break;

    case RET:
        if (cpu_feature_enabled(X86_FEATURE_RETHUNK))
            code = text_gen_insn(JMP32_INSN_OPCODE, insn, x86_return_thunk);
        else
            code = &retinsn;
        break;
    /* ... JCC handling ... */
    }

    if (system_state == SYSTEM_BOOTING || modinit)
        return text_poke_early(insn, code, size);

    text_poke_bp(insn, code, size, emulate);
}
```

### Instruction Encoding Summary

```
+--------------------------------------------------------------+
|                x86 Instruction Encodings                     |
+--------------------------------------------------------------+
|                                                              |
|  NOP (5 bytes):    0x0f 0x1f 0x44 0x00 0x00                 |
|  NOP (2 bytes):    0x66 0x90                                 |
|  JMP32 (5 bytes):  0xe9 [disp32]                             |
|  JMP8 (2 bytes):   0xeb [disp8]                              |
|  CALL (5 bytes):   0xe8 [disp32]                             |
|  RET (1 byte):     0xc3                                      |
|  INT3 (1 byte):    0xcc                                      |
|  XOR EAX,EAX:      0x2e 0x2e 0x2e 0x31 0xc0 (5 bytes)       |
|  UD1 (tramp sig):  0x0f 0xb9 0xcc (3 bytes)                 |
|                                                              |
+--------------------------------------------------------------+
```

---

## Use in Hot Paths

Static keys and static calls are used extensively throughout the kernel in
performance-critical hot paths where the overhead of a conditional branch or
indirect call would be unacceptable.

### Tracepoints

Every tracepoint uses a static key to gate its check. From
`include/linux/tracepoint-defs.h:39`:

```c
struct tracepoint {
    const char *name;
    struct static_key_false key;
    struct static_call_key *static_call_key;
    void *static_call_tramp;
    void *iterator;
    void *probestub;
    struct tracepoint_func __rcu *funcs;
    struct tracepoint_ext *ext;
};
```

The hot path check from `include/linux/tracepoint.h:266`:

```c
/* This compiles to a NOP when the tracepoint is disabled */
trace_##name##_enabled(void)
{
    return static_branch_unlikely(&__tracepoint_##name.key);
}

/* The trace macro itself */
static inline void trace_##name(proto)
{
    if (static_branch_unlikely(&__tracepoint_##name.key)) {
        /* ... call tracepoint callbacks ... */
    }
}
```

When no tracer is attached, the `static_branch_unlikely()` compiles to a 5-byte
NOP -- zero overhead in the uninstrumented case. Tracepoints also use static
calls for the actual callback dispatch, avoiding indirect call overhead when
a single callback is registered.

### Scheduler

The scheduler uses static keys for optional features. For example, from
`kernel/sched/psi.c`:

```c
DEFINE_STATIC_KEY_TRUE(psi_disabled);
DEFINE_STATIC_KEY_FALSE(psi_cgroups_enabled);

/* Hot path check -- NOP when PSI is disabled */
if (static_branch_likely(&psi_disabled))
    return;
```

Energy-aware scheduling in `kernel/sched/topology.c` uses:

```c
DEFINE_STATIC_KEY_FALSE(sched_energy_present);

/* Toggled at runtime based on hardware topology */
static_branch_enable_cpuslocked(&sched_energy_present);
```

### BPF

BPF uses static keys to gate feature checks and static calls for dispatching
to JIT-compiled programs. The BPF JIT on x86 also uses `text_poke_bp` for
patching trampolines.

### Ftrace

Function tracing uses the same text patching infrastructure to convert
function entry NOPs to CALL instructions when tracing is enabled. Ftrace was
one of the original motivations for the jump label mechanism.

### RCU

RCU uses static keys to select between different implementations at runtime
(e.g., preemptible vs. non-preemptible RCU) and to gate debugging checks.

### Networking

The networking stack uses static keys extensively for protocol-specific
features. The `net_device` fastpath checks for optional features like GRO,
XPS, and various offloads use static keys to avoid per-packet overhead
for disabled features. The `static_key_slow_dec_deferred()` API was
specifically designed for networking's needs, where userspace-controlled
toggles could cause high-frequency patching.

---

## Reference Counting with static_key_count

Static keys support reference counting, allowing multiple users to
independently enable/disable a key. The code is patched only on transitions
between 0 and non-zero.

### The Count Function

From `kernel/jump_label.c:104`:

```c
int static_key_count(struct static_key *key)
{
    /*
     * -1 means the first static_key_slow_inc() is in progress.
     *  static_key_enabled() must return true, so return 1 here.
     */
    int n = atomic_read(&key->enabled);

    return n >= 0 ? n : 1;
}
```

The special value `-1` is used as a transitional state during the first
increment. This prevents concurrent incrementors from bypassing the code
patching step.

### Increment (Enable Path)

From `kernel/jump_label.c:151`:

```c
bool static_key_slow_inc_cpuslocked(struct static_key *key)
{
    lockdep_assert_cpus_held();

    /* Fast path: already enabled, just bump the count */
    if (static_key_fast_inc_not_disabled(key))
        return true;

    guard(mutex)(&jump_label_mutex);
    /* Try to mark it as 'enabling in progress' */
    if (!atomic_cmpxchg(&key->enabled, 0, -1)) {
        jump_label_update(key);            /* Patch all sites */
        /*
         * Ensure that when static_key_fast_inc_not_disabled()
         * observes the positive value, they must also observe
         * all the text changes.
         */
        atomic_set_release(&key->enabled, 1);
    } else {
        if (WARN_ON_ONCE(!static_key_fast_inc_not_disabled(key)))
            return false;
    }
    return true;
}
```

The fast path (`static_key_fast_inc_not_disabled`) at line 127 uses an atomic
compare-and-swap loop that only succeeds when the count is positive. This means
only the 0->1 transition (first enabler) takes the slow path and actually
patches code.

### Decrement (Disable Path)

From `kernel/jump_label.c:292`:

```c
static void __static_key_slow_dec_cpuslocked(struct static_key *key)
{
    /* ... */
    if (static_key_dec_not_one(key))
        return;                           /* Still other users, no patching */

    guard(mutex)(&jump_label_mutex);
    /* ... */
    if (atomic_dec_and_test(&key->enabled))
        jump_label_update(key);           /* Last user, patch all sites */
}
```

Similarly, decrement only patches on the 1->0 transition.

### Deferred Decrement

From `kernel/jump_label.c:346`:

```c
void __static_key_slow_dec_deferred(struct static_key *key,
                                    struct delayed_work *work,
                                    unsigned long timeout)
{
    STATIC_KEY_CHECK_USE(key);

    if (static_key_dec_not_one(key))
        return;

    schedule_delayed_work(work, timeout);
}
```

This defers the final 1->0 transition by scheduling it as delayed work. This
prevents high-frequency toggling from causing excessive code patching, which is
important when the control is exposed to userspace (e.g., sysctl knobs).

---

## text_poke_bp -- Breakpoint-Based Safe Patching

The INT3 breakpoint protocol is the core mechanism that makes live code patching
safe on SMP x86 systems. It avoids `stop_machine()` entirely.

### The Algorithm

From `arch/x86/kernel/alternative.c:2267`:

```
 text_poke_bp_batch() -- update instructions on live kernel on SMP

 The way it is done:
   - For each entry in the vector:
       - add a int3 trap to the address that will be patched
   - sync cores
   - For each entry in the vector:
       - update all but the first byte of the patched range
   - sync cores
   - For each entry in the vector:
       - replace the first byte (int3) by the first byte of
         replacing opcode
   - sync cores
```

### Implementation

From `arch/x86/kernel/alternative.c:2287`:

```c
static void text_poke_bp_batch(struct text_poke_loc *tp, unsigned int nr_entries)
{
    unsigned char int3 = INT3_INSN_OPCODE;
    unsigned int i;
    int do_sync;

    lockdep_assert_held(&text_mutex);

    /* Step 1: Write INT3 at each site */
    for (i = 0; i < nr_entries; i++) {
        tp[i].old = *(u8 *)text_poke_addr(&tp[i]);
        text_poke(text_poke_addr(&tp[i]), &int3, INT3_INSN_SIZE);
    }

    text_poke_sync();    /* IPI all cores to serialize */

    /* Step 2: Update bytes 2-5 (everything after the INT3) */
    for (do_sync = 0, i = 0; i < nr_entries; i++) {
        /* ... */
        if (len - INT3_INSN_SIZE > 0) {
            text_poke(text_poke_addr(&tp[i]) + INT3_INSN_SIZE,
                      new + INT3_INSN_SIZE,
                      len - INT3_INSN_SIZE);
            do_sync++;
        }
    }

    /* ... emit perf events, sync, then: */

    /* Step 3: Replace INT3 with the real first byte (opcode) */
    for (i = 0; i < nr_entries; i++) {
        /* ... */
        text_poke(text_poke_addr(&tp[i]), &tp[i].text[0], INT3_INSN_SIZE);
    }

    text_poke_sync();    /* Final sync */
}
```

### Why This Works

```
+------------------------------------------------------------------+
|              INT3 Breakpoint Patching Protocol                   |
+------------------------------------------------------------------+
|                                                                  |
|  Original instruction:  [OP] [B2] [B3] [B4] [B5]                |
|                                                                  |
|  Step 1: Write INT3:    [CC] [B2] [B3] [B4] [B5]                |
|           IPI-sync all cores                                     |
|           Any CPU hitting this address traps to INT3 handler     |
|           which emulates the OLD instruction                     |
|                                                                  |
|  Step 2: Write tail:    [CC] [B2'] [B3'] [B4'] [B5']            |
|           IPI-sync all cores                                     |
|           INT3 handler now emulates the NEW instruction          |
|                                                                  |
|  Step 3: Write opcode:  [OP'] [B2'] [B3'] [B4'] [B5']           |
|           IPI-sync all cores                                     |
|           Normal execution of the new instruction                |
|                                                                  |
+------------------------------------------------------------------+
|                                                                  |
|  Key safety properties:                                          |
|  - INT3 is a single byte: atomically visible to all cores        |
|  - INT3 handler is registered in the IDT (poke_int3_handler)     |
|  - The emulate pointer allows correct behavior during patch      |
|  - IPI sync ensures all cores see each stage before the next     |
|  - No stop_machine() needed: only the patching core needs        |
|    the text_mutex; others continue running normally              |
|                                                                  |
+------------------------------------------------------------------+
```

### INT3 Emulation Helpers

From `arch/x86/include/asm/text-patching.h:136`:

```c
static __always_inline
void int3_emulate_jmp(struct pt_regs *regs, unsigned long ip)
{
    regs->ip = ip;
}

static __always_inline
void int3_emulate_call(struct pt_regs *regs, unsigned long func)
{
    int3_emulate_push(regs, regs->ip - INT3_INSN_SIZE + CALL_INSN_SIZE);
    int3_emulate_jmp(regs, func);
}
```

These functions allow the INT3 handler to emulate the original or new
instruction transparently, so the interrupted code behaves correctly during the
patching window.

### Batching

The `text_poke_queue()` / `text_poke_finish()` API allows batching multiple
patch sites. All queued patches are applied in a single `text_poke_bp_batch()`
call, requiring only three sync cycles total regardless of the number of sites.
This is used by jump label batch mode (`HAVE_JUMP_LABEL_BATCH`) and ftrace.

---

## Module Integration

Both jump labels and static calls must handle module load and unload events,
since modules can both define new static keys/calls and reference ones defined
in vmlinux or other modules.

### Jump Label Module Support

From `kernel/jump_label.c:622`:

```c
struct static_key_mod {
    struct static_key_mod *next;
    struct jump_entry *entries;
    struct module *mod;
};
```

When a module is loaded, `jump_label_add_module()` (at line 700) processes the
module's `__jump_table` section:

```c
static int jump_label_add_module(struct module *mod)
{
    struct jump_entry *iter_start = mod->jump_entries;
    struct jump_entry *iter_stop = iter_start + mod->num_jump_entries;
    /* ... */

    jump_label_sort_entries(iter_start, iter_stop);

    for (iter = iter_start; iter < iter_stop; iter++) {
        /* ... */
        key = iterk;
        if (within_module((unsigned long)key, mod)) {
            /* Key is defined in this module */
            static_key_set_entries(key, iter);
            continue;
        }

        /* Key is in vmlinux or another module -- link into chain */
        if (static_key_sealed(key))
            goto do_poke;  /* Just patch, no tracking needed */

        jlm = kzalloc(sizeof(struct static_key_mod), GFP_KERNEL);
        /* ... build linked list of static_key_mod ... */
        jlm->mod = mod;
        jlm->entries = iter;
        jlm->next = static_key_mod(key);
        static_key_set_mod(key, jlm);
        static_key_set_linked(key);

        /* Patch if current state differs from initial state */
do_poke:
        if (jump_label_type(iter) != jump_label_init_type(iter))
            __jump_label_update(key, iter, iter_stop, true);
    }
    return 0;
}
```

Module unload (`jump_label_del_module` at line 772) removes the module's
`static_key_mod` from each key's linked list. If only one entry remains, the
list is folded back into a direct pointer.

The module notifier is registered at `kernel/jump_label.c:855`:

```c
static struct notifier_block jump_label_module_nb = {
    .notifier_call = jump_label_module_notify,
    .priority = 1, /* higher than tracepoints */
};
```

### Static Call Module Support

Static calls handle modules similarly. From `kernel/static_call_inline.c:364`:

```c
static int static_call_add_module(struct module *mod)
{
    struct static_call_site *start = mod->static_call_sites;
    struct static_call_site *stop = start + mod->num_static_call_sites;
    struct static_call_site *site;

    for (site = start; site != stop; site++) {
        unsigned long s_key = __static_call_key(site);
        unsigned long addr = s_key & ~STATIC_CALL_SITE_FLAGS;
        unsigned long key;

        /*
         * If the key is exported, 'addr' points to the key.
         * Otherwise, 'addr' points to the trampoline and we
         * need to look up the key via tramp_key_lookup().
         */
        if (!kernel_text_address(addr))
            continue;

        key = tramp_key_lookup(addr);
        /* ... fix up the site's key reference ... */
    }

    return __static_call_init(mod, start, stop);
}
```

The `EXPORT_STATIC_CALL_TRAMP` variants deliberately do not export the key,
preventing modules from calling `static_call_update()`. The trampoline-to-key
mapping is maintained via the `static_call_tramp_key` section
(`include/linux/static_call.h:176`):

```c
struct static_call_tramp_key {
    s32 tramp;
    s32 key;
};
```

The module notifier at `kernel/static_call_inline.c:471`:

```c
static struct notifier_block static_call_module_nb = {
    .notifier_call = static_call_module_notify,
};
```

### Module Lifecycle

```
+------------------------------------------------------------------+
|                   Module Load/Unload Flow                        |
+------------------------------------------------------------------+
|                                                                  |
|  MODULE_STATE_COMING (load):                                     |
|  1. Sort the module's jump/static_call entries by key            |
|  2. For each entry referencing a vmlinux key:                    |
|     a. Allocate static_key_mod / static_call_mod                 |
|     b. Link into the key's chain (set JUMP_TYPE_LINKED)          |
|     c. Patch if current state != initial state                   |
|  3. For entries with keys defined in the module:                 |
|     a. Set the key's entries pointer directly                    |
|                                                                  |
|  MODULE_STATE_GOING (unload):                                    |
|  1. For each entry referencing a vmlinux key:                    |
|     a. Remove this module's static_key_mod from the chain        |
|     b. If only one entry remains, fold back to direct pointer    |
|  2. Free allocated tracking structures                           |
|                                                                  |
|  Locking: cpus_read_lock() + jump_label_mutex/static_call_mutex  |
|                                                                  |
+------------------------------------------------------------------+
```

---

## Performance Benefits

### Quantitative Impact

```
+------------------------------------------------------------------+
|              Performance Comparison (x86_64)                     |
+------------------------------------------------------------------+
|                                                                  |
|  Mechanism            | Instruction Bytes | Cycles (approx)      |
|  ---------------------+-------------------+----------------------|
|  Conditional branch   | 7-10 (load+cmp+jcc)| 1-20 (predict miss) |
|  Static key (NOP)     | 5 (nop)           | ~0 (fused by CPU)    |
|  Static key (JMP)     | 5 (jmp)           | ~0 (unconditional)   |
|  Indirect call        | 3-7 (call *reg)   | 20-100 (retpoline)   |
|  Static call (direct) | 5 (call rel32)    | 1-3 (well predicted) |
|  Static call (RET0)   | 5 (xor eax,eax)   | ~0 (no call at all)  |
|                                                                  |
+------------------------------------------------------------------+
```

### Key Advantages

**Static keys eliminate branch prediction overhead entirely.** A 5-byte NOP is
simply not a branch -- there is nothing to predict, no branch target buffer
entry consumed, no possibility of a misprediction penalty. This is particularly
valuable for tracepoints, where thousands of disabled tracepoints throughout the
kernel would otherwise pollute the branch predictor.

**Static calls avoid retpoline penalties.** On Spectre v2 mitigated systems,
indirect calls go through retpolines that cost 20-100+ cycles per call. A
direct call instruction is perfectly predicted and costs only the function call
overhead itself. For high-frequency dispatch points like timer callbacks,
scheduler hooks, and tracepoint callbacks, this difference is significant.

**The patching cost is amortized.** While modifying code is expensive (requiring
IPIs to synchronize all cores), it happens only when the state changes -- which
is typically rare (sysctl toggle, module load, debugger attach). The per-
execution savings are paid every time the hot path runs, which can be millions
of times per second.

**Cache efficiency.** A NOP or direct call consumes the same number of
instruction cache bytes as the code it replaces, with no additional data cache
footprint. Conditional branches require loading the flag variable from data
cache; indirect calls require loading the function pointer. Eliminating these
loads frees data cache lines for other uses.

### When to Use

| Scenario | Mechanism | Example |
|----------|-----------|---------|
| Feature rarely enabled, checked on hot path | `DEFINE_STATIC_KEY_FALSE` + `static_branch_unlikely` | Tracepoints, debug checks |
| Feature usually enabled, rarely disabled | `DEFINE_STATIC_KEY_TRUE` + `static_branch_likely` | PSI accounting |
| Multiple independent enablers | `static_branch_inc/dec` | Network protocol features |
| Indirect call on hot path | `DEFINE_STATIC_CALL` | Timer callbacks, scheduler hooks |
| Optional callback (may be NULL) | `DEFINE_STATIC_CALL_NULL` + `static_call_cond` | Optional subsystem hooks |
| Do-nothing return 0 callback | `DEFINE_STATIC_CALL_RET0` | Disabled feature stubs |

### Trade-offs

- **Update cost**: Modifying a static key or call requires `cpus_read_lock()`,
  a mutex, IPIs to all cores, and patching every site. This makes them
  unsuitable for frequently toggled state.
- **Memory**: Each jump entry is 12 bytes (three `s32` fields). Each static call
  site is 8 bytes (two `s32` fields). With thousands of tracepoints, this adds
  up to tens of kilobytes.
- **Complexity**: The interaction between the compiler, linker sections, objtool
  annotations, and runtime patching adds significant implementation complexity.
- **Architecture support required**: Full functionality requires
  `CONFIG_JUMP_LABEL` (most architectures) and
  `CONFIG_HAVE_STATIC_CALL_INLINE` (currently x86 only for full inline
  patching). Other architectures fall back to trampoline-only or conditional
  branch implementations.
