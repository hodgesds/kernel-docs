# Linux Kernel BPF (eBPF) Overview

## Table of Contents
1. [Introduction](#introduction)
2. [BPF Architecture](#bpf-architecture)
3. [BPF Instruction Set](#bpf-instruction-set)
4. [BPF Programs](#bpf-programs)
5. [BPF Verifier](#bpf-verifier)
6. [BPF JIT Compiler](#bpf-jit-compiler)
7. [BPF Maps](#bpf-maps)
8. [BPF Helpers](#bpf-helpers)
9. [BTF (BPF Type Format)](#btf-bpf-type-format)
10. [Program Types and Attach Points](#program-types-and-attach-points)
11. [XDP and TC](#xdp-and-tc)

---

## Introduction

BPF (Berkeley Packet Filter), extended to eBPF (extended BPF), is a
revolutionary technology that allows running sandboxed programs in the Linux
kernel. It enables:

- High-performance networking (XDP, TC)
- System observability (tracing, profiling)
- Security enforcement (seccomp, LSM)
- All without modifying kernel source or loading kernel modules

### BPF Evolution

```
┌─────────────────────────────────────────────────────────────┐
│                     BPF Evolution                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Classic BPF (cBPF) - 1992                                  │
│  ─────────────────────────                                  │
│  - 2 registers (A, X)                                       │
│  - 32-bit                                                   │
│  - Limited instruction set                                  │
│  - Only packet filtering (tcpdump)                          │
│                                                             │
│  Extended BPF (eBPF) - 2014+                                │
│  ─────────────────────────                                  │
│  - 11 registers (R0-R10)                                    │
│  - 64-bit                                                   │
│  - Rich instruction set                                     │
│  - Maps for persistent storage                              │
│  - Helper functions for kernel interaction                  │
│  - JIT compilation for native performance                   │
│  - Verifier for safety guarantees                           │
│  - Many program types and attach points                     │
│                                                             │
│  Modern eBPF (2020+)                                        │
│  ───────────────────                                        │
│  - BTF for type information                                 │
│  - CO-RE for portable programs                              │
│  - Bounded loops                                            │
│  - Global variables                                         │
│  - BPF-to-BPF calls                                         │
│  - Signed/unsigned comparisons                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## BPF Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   BPF Architecture                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User Space                                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  BPF Program (C)                                     │   │
│  │      │                                               │   │
│  │      ▼ clang -target bpf                             │   │
│  │  BPF Bytecode (ELF)                                  │   │
│  │      │                                               │   │
│  │      ▼ libbpf / bpf() syscall                        │   │
│  └──────┼───────────────────────────────────────────────┘   │
│         │                                                   │
│  ───────┼───────────────────────────────────────────────    │
│         │ bpf(BPF_PROG_LOAD)                                │
│         ▼                                                   │
│  Kernel Space                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ┌──────────────┐                                   │    │
│  │  │   Verifier   │ ← Safety and termination check    │    │
│  │  └──────┬───────┘                                   │    │
│  │         │ OK                                        │    │
│  │         ▼                                           │    │
│  │  ┌──────────────┐                                   │    │
│  │  │ JIT Compiler │ ← Convert to native code          │    │
│  │  └──────┬───────┘                                   │    │
│  │         │                                           │    │
│  │         ▼                                           │    │
│  │  ┌──────────────┐    ┌──────────────┐               │    │
│  │  │  BPF Program │──▶│   BPF Maps   │               │    │
│  │  │  (attached)  │◀──│  (storage)   │               │    │
│  │  └──────────────┘    └──────────────┘               │    │
│  │         │                   ▲                       │    │
│  │         │                   │                       │    │
│  │         ▼                   │                       │    │
│  │  ┌──────────────┐           │                       │    │
│  │  │   Helpers    │───────────┘                       │    │
│  │  │ (kernel API) │                                   │    │
│  │  └──────────────┘                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## BPF Instruction Set

BPF uses a RISC-like instruction set optimized for JIT compilation.

### Registers

```
┌─────────────────────────────────────────────────────────────┐
│                   BPF Registers                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Register    Purpose                                        │
│  ────────    ───────                                        │
│  R0          Return value from helpers and program          │
│  R1          First argument to helpers / context pointer    │
│  R2          Second argument                                │
│  R3          Third argument                                 │
│  R4          Fourth argument                                │
│  R5          Fifth argument                                 │
│  R6          Callee-saved (preserved across calls)          │
│  R7          Callee-saved                                   │
│  R8          Callee-saved                                   │
│  R9          Callee-saved                                   │
│  R10         Frame pointer (read-only, stack base)          │
│                                                             │
│  All registers are 64-bit                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Instruction Format

The bpf_insn structure represents a single BPF instruction in the virtual
machine's bytecode. Each instruction is exactly 8 bytes, making the bytecode
format simple and efficient to parse. The structure uses bit fields for the
source and destination register indices (4 bits each, supporting the 11 BPF
registers), while the offset and immediate fields provide operand data for
memory operations, jumps, and arithmetic. This compact encoding is designed to
map efficiently onto modern CPU architectures during JIT compilation.

```c
/* include/uapi/linux/bpf.h */
struct bpf_insn {
    __u8    code;       /* Opcode */
    __u8    dst_reg:4;  /* Destination register */
    __u8    src_reg:4;  /* Source register */
    __s16   off;        /* Signed offset */
    __s32   imm;        /* Signed immediate */
};

/* Instruction classes */
#define BPF_LD      0x00    /* Load from immediate */
#define BPF_LDX     0x01    /* Load from register */
#define BPF_ST      0x02    /* Store immediate */
#define BPF_STX     0x03    /* Store from register */
#define BPF_ALU     0x04    /* 32-bit arithmetic */
#define BPF_JMP     0x05    /* 64-bit jumps */
#define BPF_JMP32   0x06    /* 32-bit jumps */
#define BPF_ALU64   0x07    /* 64-bit arithmetic */

/* ALU/ALU64 operations */
#define BPF_ADD     0x00    /* dst += src */
#define BPF_SUB     0x10    /* dst -= src */
#define BPF_MUL     0x20    /* dst *= src */
#define BPF_DIV     0x30    /* dst /= src */
#define BPF_OR      0x40    /* dst |= src */
#define BPF_AND     0x50    /* dst &= src */
#define BPF_LSH     0x60    /* dst <<= src */
#define BPF_RSH     0x70    /* dst >>= src (logical) */
#define BPF_NEG     0x80    /* dst = -dst */
#define BPF_MOD     0x90    /* dst %= src */
#define BPF_XOR     0xa0    /* dst ^= src */
#define BPF_MOV     0xb0    /* dst = src */
#define BPF_ARSH    0xc0    /* dst >>= src (arithmetic) */
#define BPF_END     0xd0    /* Endianness conversion */

/* Jump operations */
#define BPF_JA      0x00    /* Unconditional jump */
#define BPF_JEQ     0x10    /* Jump if equal */
#define BPF_JGT     0x20    /* Jump if greater (unsigned) */
#define BPF_JGE     0x30    /* Jump if greater or equal */
#define BPF_JSET    0x40    /* Jump if (dst & src) */
#define BPF_JNE     0x50    /* Jump if not equal */
#define BPF_JSGT    0x60    /* Jump if greater (signed) */
#define BPF_JSGE    0x70    /* Jump if greater or equal (signed) */
#define BPF_CALL    0x80    /* Function call */
#define BPF_EXIT    0x90    /* Program exit */
#define BPF_JLT     0xa0    /* Jump if less than */
#define BPF_JLE     0xb0    /* Jump if less or equal */
#define BPF_JSLT    0xc0    /* Jump if less than (signed) */
#define BPF_JSLE    0xd0    /* Jump if less or equal (signed) */
```

### Example Instructions

```c
/* r1 = r2 */
BPF_MOV64_REG(BPF_REG_1, BPF_REG_2)

/* r1 = 42 */
BPF_MOV64_IMM(BPF_REG_1, 42)

/* r0 = *(u64 *)(r1 + 8) */
BPF_LDX_MEM(BPF_DW, BPF_REG_0, BPF_REG_1, 8)

/* *(u32 *)(r10 - 4) = r3 */
BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_3, -4)

/* if r1 > r2 goto +5 */
BPF_JMP_REG(BPF_JGT, BPF_REG_1, BPF_REG_2, 5)

/* call helper function */
BPF_CALL_HELPER(BPF_FUNC_map_lookup_elem)

/* exit program */
BPF_EXIT_INSN()
```

---

## BPF Programs

### bpf_prog Structure

The bpf_prog structure is the kernel's internal representation of a loaded BPF
program. It contains all the metadata and state needed to execute the program,
including the actual instructions, JIT-compiled native code pointer, program
type, and various flags indicating program capabilities and requirements. The
structure tracks whether the program has been JIT-compiled, its GPL
compatibility status, and maintains per-CPU statistics. The bpf_func pointer
holds the address of the JIT-compiled native code (or the interpreter
function), while the aux field links to additional auxiliary data such as BTF
information and attached maps.

```c
/* include/linux/bpf.h */
struct bpf_prog {
    u16                     pages;          /* Number of allocated pages */
    u16                     jited:1;        /* JIT compiled? */
    u16                     jit_requested:1;
    u16                     gpl_compatible:1;
    u16                     cb_access:1;
    u16                     dst_needed:1;
    u16                     blinded:1;
    u16                     is_func:1;
    u16                     kprobe_override:1;
    u16                     has_callchain_buf:1;
    u16                     enforce_expected_attach_type:1;
    u16                     call_get_stack:1;
    u16                     call_get_func_ip:1;
    u16                     tstamp_type_access:1;

    enum bpf_prog_type      type;           /* Program type */
    enum bpf_attach_type    expected_attach_type;

    u32                     len;            /* Number of instructions */
    u32                     jited_len;      /* Size of JITed image */

    u8                      tag[BPF_TAG_SIZE];

    struct bpf_prog_stats __percpu *stats;
    int __percpu            *active;

    unsigned int (*bpf_func)(const void *ctx,
                             const struct bpf_insn *insn);

    struct bpf_prog_aux     *aux;           /* Auxiliary info */

    struct sock_fprog_kern  *orig_prog;
    struct bpf_insn         insnsi[];       /* Instructions */
};
```

### Loading a BPF Program

```c
/* User-space loading example */
#include <bpf/libbpf.h>
#include <bpf/bpf.h>

int load_bpf_program(void)
{
    struct bpf_object *obj;
    struct bpf_program *prog;
    struct bpf_link *link;
    int err;

    /* Open BPF object file */
    obj = bpf_object__open_file("my_prog.bpf.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object\n");
        return -1;
    }

    /* Load into kernel */
    err = bpf_object__load(obj);
    if (err) {
        fprintf(stderr, "Failed to load BPF object\n");
        goto cleanup;
    }

    /* Find program by name */
    prog = bpf_object__find_program_by_name(obj, "my_func");
    if (!prog) {
        fprintf(stderr, "Program not found\n");
        err = -1;
        goto cleanup;
    }

    /* Attach to hook point */
    link = bpf_program__attach(prog);
    if (libbpf_get_error(link)) {
        fprintf(stderr, "Failed to attach\n");
        err = -1;
        goto cleanup;
    }

    /* Keep running... */

cleanup:
    bpf_object__close(obj);
    return err;
}
```

---

## BPF Verifier

The verifier is a critical component that ensures BPF programs are safe to run.

### Verifier Guarantees

```
┌─────────────────────────────────────────────────────────────┐
│                  Verifier Guarantees                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Memory Safety                                           │
│     - No out-of-bounds access                               │
│     - No null pointer dereference                           │
│     - No use of uninitialized registers                     │
│     - Stack bounds checking                                 │
│                                                             │
│  2. Type Safety                                             │
│     - Pointer arithmetic restrictions                       │
│     - Context access validation                             │
│     - Map value type checking                               │
│                                                             │
│  3. Termination                                             │
│     - No infinite loops                                     │
│     - Bounded program complexity                            │
│     - Maximum instruction limit                             │
│                                                             │
│  4. Privilege Checking                                      │
│     - Helper function permissions                           │
│     - Map access permissions                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Verifier Algorithm

```
┌─────────────────────────────────────────────────────────────┐
│                  Verifier Algorithm                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. CFG Construction                                        │
│     - Build control flow graph                              │
│     - Identify basic blocks                                 │
│     - Check for unreachable code                            │
│                                                             │
│  2. DAG Check                                               │
│     - Ensure no back edges (no loops)                       │
│     - Or verify bounded loops                               │
│                                                             │
│  3. Symbolic Execution                                      │
│     - Track register states                                 │
│     - Track stack slot states                               │
│     - Explore all paths (with pruning)                      │
│                                                             │
│  Register State Tracking:                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Type: SCALAR / PTR_TO_CTX / PTR_TO_MAP_VALUE / ...    │  │
│  │ Range: [min_value, max_value]                         │  │
│  │ Tnum: Known bits and unknown bits                     │  │
│  │ Ref: Reference type info                              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  4. Bound Checking                                          │
│     - Verify memory access within bounds                    │
│     - Check pointer + offset validity                       │
│                                                             │
│  5. Privilege Verification                                  │
│     - Check helper calls are allowed                        │
│     - Verify map operations permitted                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Verifier Errors

```c
/* Common verifier errors */

/* Invalid memory access */
"R1 invalid mem access 'inv'"
/* Solution: Check bounds before access */

/* Uninitialized register */
"R3 !read_ok"
/* Solution: Initialize register before use */

/* Invalid pointer arithmetic */
"R2 pointer arithmetic prohibited"
/* Solution: Cast to scalar or use allowed operations */

/* Loop not bounded */
"back-edge from insn X to Y"
/* Solution: Use bounded loop or unroll */

/* Exceeded complexity limit */
"processed X insns ... limit is Y"
/* Solution: Simplify program logic */
```

---

## BPF JIT Compiler

The JIT compiles BPF bytecode to native machine code for better performance.

### JIT Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     BPF JIT                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  BPF Bytecode                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ r1 = r2                                               │  │
│  │ r0 = *(u64 *)(r1 + 0)                                 │  │
│  │ r0 += 1                                               │  │
│  │ exit                                                  │  │
│  └────────────────────┬──────────────────────────────────┘  │
│                       │                                     │
│                       ▼ JIT Compile                         │
│                                                             │
│  Native x86-64                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ mov rsi, rdx        ; r1 = r2 (rsi=R1, rdx=R2)        │  │
│  │ mov rax, [rsi]      ; r0 = *(u64 *)(r1 + 0)           │  │
│  │ add rax, 1          ; r0 += 1                         │  │
│  │ ret                 ; exit (return r0)                │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  BPF Register to x86-64 Register Mapping:                   │
│  R0 → rax (return value)                                    │
│  R1 → rdi (1st arg)                                         │
│  R2 → rsi (2nd arg)                                         │
│  R3 → rdx (3rd arg)                                         │
│  R4 → rcx (4th arg)                                         │
│  R5 → r8  (5th arg)                                         │
│  R6 → rbx (callee-saved)                                    │
│  R7 → r13 (callee-saved)                                    │
│  R8 → r14 (callee-saved)                                    │
│  R9 → r15 (callee-saved)                                    │
│  R10→ rbp (frame pointer)                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Enabling JIT

```bash
# Check JIT status
cat /proc/sys/net/core/bpf_jit_enable
# 0 = disabled, 1 = enabled, 2 = debug mode

# Enable JIT
echo 1 > /proc/sys/net/core/bpf_jit_enable

# Enable JIT hardening (for security)
echo 1 > /proc/sys/net/core/bpf_jit_harden
```

---

## BPF Maps

Maps are key-value stores for sharing data between BPF programs and user space.

### Map Types

The bpf_map_type enumeration defines all available BPF map types in the kernel.
Maps are the primary mechanism for storing persistent data that survives across
BPF program invocations and for sharing data between BPF programs and user
space. Each map type is optimized for specific access patterns and use cases:
HASH maps provide O(1) key-value lookups with arbitrary keys, ARRAY maps offer
fast indexed access, PERCPU variants eliminate locking overhead for
high-performance counters, and specialized types like RINGBUF enable efficient
data transfer to user space. Choosing the right map type is crucial for both
performance and functionality of BPF applications.

```c
/* include/uapi/linux/bpf.h */
enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,              /* Hash table */
    BPF_MAP_TYPE_ARRAY,             /* Array (fixed key 0..n-1) */
    BPF_MAP_TYPE_PROG_ARRAY,        /* Array of program FDs */
    BPF_MAP_TYPE_PERF_EVENT_ARRAY,  /* Per-CPU perf buffers */
    BPF_MAP_TYPE_PERCPU_HASH,       /* Per-CPU hash table */
    BPF_MAP_TYPE_PERCPU_ARRAY,      /* Per-CPU array */
    BPF_MAP_TYPE_STACK_TRACE,       /* Stack trace storage */
    BPF_MAP_TYPE_CGROUP_ARRAY,      /* Cgroup FD array */
    BPF_MAP_TYPE_LRU_HASH,          /* LRU hash table */
    BPF_MAP_TYPE_LRU_PERCPU_HASH,   /* Per-CPU LRU hash */
    BPF_MAP_TYPE_LPM_TRIE,          /* Longest prefix match trie */
    BPF_MAP_TYPE_ARRAY_OF_MAPS,     /* Array of maps */
    BPF_MAP_TYPE_HASH_OF_MAPS,      /* Hash of maps */
    BPF_MAP_TYPE_DEVMAP,            /* Device redirect map */
    BPF_MAP_TYPE_SOCKMAP,           /* Socket map */
    BPF_MAP_TYPE_CPUMAP,            /* CPU redirect map */
    BPF_MAP_TYPE_XSKMAP,            /* AF_XDP socket map */
    BPF_MAP_TYPE_SOCKHASH,          /* Socket hash */
    BPF_MAP_TYPE_CGROUP_STORAGE,    /* Per-cgroup storage */
    BPF_MAP_TYPE_REUSEPORT_SOCKARRAY,
    BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE,
    BPF_MAP_TYPE_QUEUE,             /* FIFO queue */
    BPF_MAP_TYPE_STACK,             /* LIFO stack */
    BPF_MAP_TYPE_SK_STORAGE,        /* Per-socket storage */
    BPF_MAP_TYPE_DEVMAP_HASH,       /* Device hash map */
    BPF_MAP_TYPE_STRUCT_OPS,        /* Struct operations */
    BPF_MAP_TYPE_RINGBUF,           /* Ring buffer */
    BPF_MAP_TYPE_INODE_STORAGE,     /* Per-inode storage */
    BPF_MAP_TYPE_TASK_STORAGE,      /* Per-task storage */
    BPF_MAP_TYPE_BLOOM_FILTER,      /* Bloom filter */
    /* ... more types */
};
```

### Map Definition and Usage

```c
/* BPF program side (libbpf) */
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10000);
    __type(key, __u32);
    __type(value, __u64);
} my_map SEC(".maps");

SEC("xdp")
int my_prog(struct xdp_md *ctx)
{
    __u32 key = 0;
    __u64 *value;

    /* Lookup */
    value = bpf_map_lookup_elem(&my_map, &key);
    if (value)
        (*value)++;

    /* Update */
    __u64 new_val = 1;
    bpf_map_update_elem(&my_map, &key, &new_val, BPF_ANY);

    /* Delete */
    bpf_map_delete_elem(&my_map, &key);

    return XDP_PASS;
}

/* User-space side */
int fd = bpf_obj_get("/sys/fs/bpf/my_map");

__u32 key = 0;
__u64 value;

/* Lookup */
bpf_map_lookup_elem(fd, &key, &value);

/* Update */
value = 42;
bpf_map_update_elem(fd, &key, &value, BPF_ANY);

/* Delete */
bpf_map_delete_elem(fd, &key);

/* Iterate */
__u32 next_key;
while (bpf_map_get_next_key(fd, &key, &next_key) == 0) {
    bpf_map_lookup_elem(fd, &next_key, &value);
    key = next_key;
}
```

### Map Types Comparison

```
┌─────────────────────────────────────────────────────────────┐
│                    Map Type Selection                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  HASH                                                       │
│  - Arbitrary keys                                           │
│  - O(1) lookup                                              │
│  - Dynamic size (up to max_entries)                         │
│  - Use for: counters per flow, caching                      │
│                                                             │
│  ARRAY                                                      │
│  - Integer keys (0 to max_entries-1)                        │
│  - O(1) lookup                                              │
│  - Pre-allocated, all entries exist                         │
│  - Use for: per-CPU counters, configuration                 │
│                                                             │
│  PERCPU_*                                                   │
│  - Separate value per CPU                                   │
│  - No locking needed                                        │
│  - Use for: counters, statistics                            │
│                                                             │
│  LRU_HASH                                                   │
│  - Automatic eviction of old entries                        │
│  - Use for: caches, connection tracking                     │
│                                                             │
│  RINGBUF                                                    │
│  - Efficient BPF-to-user data transfer                      │
│  - Variable-size records                                    │
│  - Better than perf buffer for high throughput              │
│                                                             │
│  LPM_TRIE                                                   │
│  - Longest prefix match                                     │
│  - Use for: IP routing, CIDR lookups                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## BPF Helpers

Helpers are kernel functions that BPF programs can call.

### Common Helpers

```c
/* Map operations */
void *bpf_map_lookup_elem(void *map, const void *key);
long bpf_map_update_elem(void *map, const void *key,
                         const void *value, u64 flags);
long bpf_map_delete_elem(void *map, const void *key);
long bpf_map_push_elem(void *map, const void *value, u64 flags);
long bpf_map_pop_elem(void *map, void *value);
long bpf_map_peek_elem(void *map, void *value);

/* Packet access (XDP/TC) */
long bpf_xdp_adjust_head(struct xdp_md *xdp, int delta);
long bpf_xdp_adjust_tail(struct xdp_md *xdp, int delta);
long bpf_skb_store_bytes(struct __sk_buff *skb, u32 offset,
                         const void *from, u32 len, u64 flags);
long bpf_skb_load_bytes(const struct __sk_buff *skb, u32 offset,
                        void *to, u32 len);

/* Redirect */
long bpf_redirect(u32 ifindex, u64 flags);
long bpf_redirect_map(void *map, u32 key, u64 flags);

/* Tracing */
long bpf_probe_read(void *dst, u32 size, const void *unsafe_ptr);
long bpf_probe_read_user(void *dst, u32 size, const void *unsafe_ptr);
long bpf_probe_read_kernel(void *dst, u32 size, const void *unsafe_ptr);
long bpf_probe_read_str(void *dst, u32 size, const void *unsafe_ptr);

/* Time */
u64 bpf_ktime_get_ns(void);
u64 bpf_ktime_get_boot_ns(void);
u64 bpf_ktime_get_coarse_ns(void);

/* Random */
u32 bpf_get_prandom_u32(void);

/* Process info */
u64 bpf_get_current_pid_tgid(void);
u64 bpf_get_current_uid_gid(void);
long bpf_get_current_comm(void *buf, u32 size);
struct task_struct *bpf_get_current_task(void);

/* Debugging */
long bpf_trace_printk(const char *fmt, u32 fmt_size, ...);
/* Output visible in /sys/kernel/debug/tracing/trace_pipe */

/* Stack traces */
long bpf_get_stackid(void *ctx, void *map, u64 flags);
long bpf_get_stack(void *ctx, void *buf, u32 size, u64 flags);

/* Ring buffer */
void *bpf_ringbuf_reserve(void *ringbuf, u64 size, u64 flags);
void bpf_ringbuf_submit(void *data, u64 flags);
void bpf_ringbuf_discard(void *data, u64 flags);
long bpf_ringbuf_output(void *ringbuf, void *data, u64 size, u64 flags);

/* Spin locks */
long bpf_spin_lock(struct bpf_spin_lock *lock);
long bpf_spin_unlock(struct bpf_spin_lock *lock);

/* ... many more helpers ... */
```

### Helper Categories by Program Type

```
┌─────────────────────────────────────────────────────────────┐
│              Helpers by Program Type                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  XDP Programs:                                              │
│  - bpf_xdp_adjust_head/tail                                 │
│  - bpf_redirect, bpf_redirect_map                           │
│  - bpf_fib_lookup                                           │
│  - Map helpers                                              │
│                                                             │
│  TC Programs:                                               │
│  - bpf_skb_* (load, store, change_proto, etc.)              │
│  - bpf_redirect                                             │
│  - bpf_clone_redirect                                       │
│  - bpf_csum_* helpers                                       │
│                                                             │
│  Tracing Programs (kprobe, tracepoint):                     │
│  - bpf_probe_read_*                                         │
│  - bpf_get_current_* (pid, comm, etc.)                      │
│  - bpf_perf_event_output                                    │
│  - bpf_get_stack                                            │
│                                                             │
│  Socket Programs:                                           │
│  - bpf_sock_* helpers                                       │
│  - bpf_sk_storage_*                                         │
│  - bpf_bind, bpf_connect                                    │
│                                                             │
│  Cgroup Programs:                                           │
│  - bpf_sysctl_*                                             │
│  - bpf_get_local_storage                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## BTF (BPF Type Format)

BTF provides type information for BPF programs, enabling:
- Better debugging
- CO-RE (Compile Once, Run Everywhere)
- Readable map/program introspection

### BTF Structure

The btf_header structure is the header of a BTF (BPF Type Format) blob, which
provides rich type information for BPF programs and maps. BTF enables
debugging, introspection, and most importantly, CO-RE (Compile Once, Run
Everywhere) functionality that allows BPF programs to run across different
kernel versions. The header contains a magic number for validation, version
information, and offsets to the type and string sections that follow. The type
section contains encoded type descriptors (structs, unions, enums, etc.), while
the string section holds all type and field names. The kernel uses BTF to
validate map value types and to generate human-readable output from bpftool.

```c
/* include/uapi/linux/btf.h */
struct btf_header {
    __u16   magic;          /* BTF_MAGIC (0xeB9F) */
    __u8    version;
    __u8    flags;
    __u32   hdr_len;

    /* Offsets relative to end of header */
    __u32   type_off;       /* Offset of type section */
    __u32   type_len;       /* Length of type section */
    __u32   str_off;        /* Offset of string section */
    __u32   str_len;        /* Length of string section */
};

/* BTF type kinds */
enum {
    BTF_KIND_UNKN           = 0,
    BTF_KIND_INT            = 1,    /* Integer */
    BTF_KIND_PTR            = 2,    /* Pointer */
    BTF_KIND_ARRAY          = 3,    /* Array */
    BTF_KIND_STRUCT         = 4,    /* Struct */
    BTF_KIND_UNION          = 5,    /* Union */
    BTF_KIND_ENUM           = 6,    /* Enum */
    BTF_KIND_FWD            = 7,    /* Forward declaration */
    BTF_KIND_TYPEDEF        = 8,    /* Typedef */
    BTF_KIND_VOLATILE       = 9,    /* Volatile */
    BTF_KIND_CONST          = 10,   /* Const */
    BTF_KIND_RESTRICT       = 11,   /* Restrict */
    BTF_KIND_FUNC           = 12,   /* Function */
    BTF_KIND_FUNC_PROTO     = 13,   /* Function prototype */
    BTF_KIND_VAR            = 14,   /* Variable */
    BTF_KIND_DATASEC        = 15,   /* Data section */
    BTF_KIND_FLOAT          = 16,   /* Floating point */
    BTF_KIND_DECL_TAG       = 17,   /* Declaration tag */
    BTF_KIND_TYPE_TAG       = 18,   /* Type tag */
    BTF_KIND_ENUM64         = 19,   /* 64-bit enum */
};
```

### CO-RE (Compile Once, Run Everywhere)

```c
/* BPF CO-RE allows accessing kernel structures portably */

/* Old way (fragile): */
struct task_struct *task = (void *)bpf_get_current_task();
// Direct offset access - breaks across kernel versions

/* CO-RE way (portable): */
#include <vmlinux.h>  /* Generated from BTF */

struct task_struct *task = (void *)bpf_get_current_task();
pid_t pid = BPF_CORE_READ(task, pid);
/* Compiler records field access, libbpf relocates at load time */

/* Field existence check */
if (bpf_core_field_exists(task->field_that_may_not_exist)) {
    /* Access the field */
}

/* Type existence check */
if (bpf_core_type_exists(struct new_kernel_struct)) {
    /* Use the type */
}
```

---

## Program Types and Attach Points

### Major Program Types

The bpf_prog_type enumeration defines all the different types of BPF programs
that the kernel supports. Each program type determines where the program can be
attached, what context it receives, which helper functions it can call, and
what return value semantics apply. For example, XDP programs run at the
earliest point in the network receive path and return actions like DROP or
PASS, while KPROBE programs attach to kernel functions for tracing and receive
CPU register state. The program type is specified when loading a program and
the verifier uses it to enforce appropriate restrictions and validate helper
calls.

```c
enum bpf_prog_type {
    BPF_PROG_TYPE_UNSPEC,
    BPF_PROG_TYPE_SOCKET_FILTER,    /* Socket filtering */
    BPF_PROG_TYPE_KPROBE,           /* Kernel function tracing */
    BPF_PROG_TYPE_SCHED_CLS,        /* TC classifier */
    BPF_PROG_TYPE_SCHED_ACT,        /* TC action */
    BPF_PROG_TYPE_TRACEPOINT,       /* Kernel tracepoints */
    BPF_PROG_TYPE_XDP,              /* eXpress Data Path */
    BPF_PROG_TYPE_PERF_EVENT,       /* Perf events */
    BPF_PROG_TYPE_CGROUP_SKB,       /* Cgroup socket buffer */
    BPF_PROG_TYPE_CGROUP_SOCK,      /* Cgroup socket */
    BPF_PROG_TYPE_LWT_IN,           /* Lightweight tunnel */
    BPF_PROG_TYPE_LWT_OUT,
    BPF_PROG_TYPE_LWT_XMIT,
    BPF_PROG_TYPE_SOCK_OPS,         /* Socket operations */
    BPF_PROG_TYPE_SK_SKB,           /* Socket buffer program */
    BPF_PROG_TYPE_CGROUP_DEVICE,    /* Cgroup device access */
    BPF_PROG_TYPE_SK_MSG,           /* Socket message */
    BPF_PROG_TYPE_RAW_TRACEPOINT,   /* Raw tracepoints */
    BPF_PROG_TYPE_CGROUP_SOCK_ADDR, /* Cgroup socket address */
    BPF_PROG_TYPE_LWT_SEG6LOCAL,    /* SRv6 */
    BPF_PROG_TYPE_LIRC_MODE2,       /* IR remote control */
    BPF_PROG_TYPE_SK_REUSEPORT,     /* SO_REUSEPORT selection */
    BPF_PROG_TYPE_FLOW_DISSECTOR,   /* Flow dissection */
    BPF_PROG_TYPE_CGROUP_SYSCTL,    /* Sysctl access */
    BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE,
    BPF_PROG_TYPE_CGROUP_SOCKOPT,   /* Socket options */
    BPF_PROG_TYPE_TRACING,          /* fentry/fexit/fmod_ret */
    BPF_PROG_TYPE_STRUCT_OPS,       /* Kernel struct ops */
    BPF_PROG_TYPE_EXT,              /* Extension program */
    BPF_PROG_TYPE_LSM,              /* LSM hook */
    BPF_PROG_TYPE_SK_LOOKUP,        /* Socket lookup */
    BPF_PROG_TYPE_SYSCALL,          /* System call */
    /* ... more types ... */
};
```

### Program Type Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                Program Type Use Cases                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Networking:                                                │
│  ─────────                                                  │
│  XDP              - DDoS mitigation, load balancing         │
│  SCHED_CLS (TC)   - Traffic shaping, filtering              │
│  SOCKET_FILTER    - Packet capture (tcpdump)                │
│  SK_SKB           - Socket data plane                       │
│  SOCK_OPS         - TCP tuning, congestion control          │
│                                                             │
│  Tracing:                                                   │
│  ───────                                                    │
│  KPROBE           - Function entry/exit tracing             │
│  TRACEPOINT       - Stable kernel events                    │
│  PERF_EVENT       - CPU profiling                           │
│  RAW_TRACEPOINT   - Low-overhead tracing                    │
│  TRACING          - fentry/fexit (modern, BTF-based)        │
│                                                             │
│  Security:                                                  │
│  ────────                                                   │
│  LSM               - Security policy enforcement            │
│  CGROUP_*          - Container security                     │
│                                                             │
│  Schedulers:                                                │
│  ──────────                                                 │
│  STRUCT_OPS        - Custom TCP congestion control          │
│  STRUCT_OPS        - sched_ext (custom CPU scheduling)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## XDP and TC

### XDP (eXpress Data Path)

The xdp_md structure is the context passed to XDP programs, providing access to
packet data at the earliest possible point in the network receive path, before
any socket buffer (skb) allocation. This makes XDP ideal for high-performance
packet processing like DDoS mitigation and load balancing. The data and
data_end fields define the packet boundaries for safe access, while
ingress_ifindex identifies the receiving interface. XDP programs return one of
the xdp_action values to tell the driver what to do with the packet: XDP_DROP
discards it immediately, XDP_PASS sends it up the network stack, XDP_TX
transmits it back out the same interface, and XDP_REDIRECT sends it to another
interface or CPU.

```c
/* XDP context */
struct xdp_md {
    __u32 data;             /* Packet data start */
    __u32 data_end;         /* Packet data end */
    __u32 data_meta;        /* Metadata before packet */
    __u32 ingress_ifindex;  /* Incoming interface */
    __u32 rx_queue_index;   /* RX queue */
    __u32 egress_ifindex;   /* For TX */
};

/* XDP return codes */
enum xdp_action {
    XDP_ABORTED = 0,    /* Error, drop with trace */
    XDP_DROP,           /* Silent drop */
    XDP_PASS,           /* Pass to network stack */
    XDP_TX,             /* Transmit on same interface */
    XDP_REDIRECT,       /* Redirect to another interface/CPU */
};

/* Example XDP program */
SEC("xdp")
int xdp_drop_icmp(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_ABORTED;

    if (eth->h_proto != bpf_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_ABORTED;

    if (ip->protocol == IPPROTO_ICMP)
        return XDP_DROP;

    return XDP_PASS;
}
```

### TC (Traffic Control)

The __sk_buff structure is the context for TC (Traffic Control) BPF programs,
providing a rich view of the socket buffer (skb) that represents a network
packet. Unlike XDP which operates before skb allocation, TC programs run later
in the network stack with access to more metadata including protocol
information, VLAN tags, packet marks, and priority fields. The structure
exposes both packet data (via data/data_end pointers) and extensive metadata
that can be read or modified. TC programs are attached to the ingress or egress
paths using the tc command and return TC_ACT_* values: TC_ACT_OK passes the
packet, TC_ACT_SHOT drops it, and TC_ACT_REDIRECT sends it elsewhere. TC is
commonly used for traffic shaping, policing, and advanced packet mangling.

```c
/* TC context (__sk_buff) */
struct __sk_buff {
    __u32 len;              /* Packet length */
    __u32 pkt_type;         /* Packet type */
    __u32 mark;             /* Skb mark */
    __u32 queue_mapping;    /* TX queue */
    __u32 protocol;         /* L3 protocol */
    __u32 vlan_present;
    __u32 vlan_tci;
    __u32 vlan_proto;
    __u32 priority;
    __u32 ingress_ifindex;
    __u32 ifindex;
    __u32 tc_index;
    __u32 cb[5];            /* Control block */
    __u32 hash;
    __u32 tc_classid;       /* TC class ID */
    __u32 data;             /* Packet data start */
    __u32 data_end;         /* Packet data end */
    __u32 napi_id;
    __u32 family;           /* Address family */
    /* ... more fields ... */
};

/* TC return codes */
#define TC_ACT_UNSPEC       (-1)
#define TC_ACT_OK           0
#define TC_ACT_RECLASSIFY   1
#define TC_ACT_SHOT         2   /* Drop */
#define TC_ACT_PIPE         3
#define TC_ACT_STOLEN       4
#define TC_ACT_QUEUED       5
#define TC_ACT_REPEAT       6
#define TC_ACT_REDIRECT     7

/* Example TC program */
SEC("tc")
int tc_classify(struct __sk_buff *skb)
{
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return TC_ACT_SHOT;

    /* Set TC class for QoS */
    if (eth->h_proto == bpf_htons(ETH_P_IP)) {
        struct iphdr *ip = (void *)(eth + 1);
        if ((void *)(ip + 1) > data_end)
            return TC_ACT_SHOT;

        /* High priority for port 80 */
        if (ip->protocol == IPPROTO_TCP) {
            skb->tc_classid = 0x10001;  /* classid 1:1 */
        }
    }

    return TC_ACT_OK;
}
```

---

## Summary

eBPF provides a powerful, safe, and efficient way to extend the Linux kernel:

| Component | Purpose |
|-----------|---------|
| Instruction Set | RISC-like 64-bit with 11 registers |
| Verifier | Safety and termination guarantee |
| JIT Compiler | Native code performance |
| Maps | Shared storage between BPF and user space |
| Helpers | Kernel API access |
| BTF | Type information for portability |

Key use cases:
- **Networking**: XDP for packet processing, TC for traffic control
- **Observability**: Tracing, profiling, metrics
- **Security**: LSM hooks, seccomp
- **Scheduling**: sched_ext for custom CPU scheduling

eBPF enables kernel-level programmability without kernel module risks.
