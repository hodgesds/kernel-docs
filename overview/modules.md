# Linux Kernel Module Loading Subsystem Overview

## Table of Contents
1. [Introduction](#1-introduction)
2. [Module Lifecycle](#2-module-lifecycle)
3. [Core Data Structures](#3-core-data-structures)
4. [System Call Entry Points](#4-system-call-entry-points)
5. [The load_module() Pipeline](#5-the-load_module-pipeline)
6. [ELF Parsing and Validation](#6-elf-parsing-and-validation)
7. [Memory Layout and Allocation](#7-memory-layout-and-allocation)
8. [Symbol Resolution and Dependencies](#8-symbol-resolution-and-dependencies)
9. [Relocations](#9-relocations)
10. [Module Signing and Verification](#10-module-signing-and-verification)
11. [Module Parameters](#11-module-parameters)
12. [Module Initialization and Teardown](#12-module-initialization-and-teardown)
13. [Module Unloading](#13-module-unloading)
14. [Sysfs and Procfs Interfaces](#14-sysfs-and-procfs-interfaces)
15. [Kallsyms Integration](#15-kallsyms-integration)
16. [Module Tainting](#16-module-tainting)
17. [Live Patching Hooks](#17-live-patching-hooks)
18. [Locking and Concurrency](#18-locking-and-concurrency)
19. [Source File Map](#19-source-file-map)
20. [Summary](#20-summary)

---

## 1. Introduction

The Linux kernel module subsystem allows code to be dynamically loaded into
and unloaded from a running kernel without rebooting. Modules are compiled
as ELF relocatable object files (`.ko`) and undergo extensive validation,
relocation, and linking at load time. The subsystem handles:

- Parsing and validating ELF object files
- Resolving undefined symbols against the kernel and other modules
- Applying architecture-specific relocations
- Verifying cryptographic signatures
- Managing module dependencies and reference counts
- Exposing module metadata through sysfs, procfs, and kallsyms
- Supporting kernel live patching via preserved ELF information

### High-Level Architecture

```
                        User Space
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  insmod / modprobe                                      │
  │      │                                                  │
  │      ▼                                                  │
  │  finit_module(fd, args, flags)                          │
  │  init_module(umod, len, args)                           │
  │                                                         │
  └──────┼──────────────────────────────────────────────────┘
         │  syscall
  ───────┼──────────────────────────────────────────────────
         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Kernel: load_module()                                  │
  │                                                         │
  │  1. module_sig_check()         Verify signature         │
  │  2. elf_validity_cache_copy()  Parse & validate ELF     │
  │  3. early_mod_check()          Vermagic, blacklist      │
  │  4. layout_and_allocate()      Compute layout, alloc    │
  │  5. add_unformed_module()      Insert into module list  │
  │  6. simplify_symbols()         Resolve symbols          │
  │  7. apply_relocations()        Patch code references    │
  │  8. post_relocation()          Kallsyms, finalize       │
  │  9. complete_formation()       Set R/O, R/X perms       │
  │  10. mod_sysfs_setup()         Create sysfs entries     │
  │  11. do_init_module()          Call mod->init()          │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

---

## 2. Module Lifecycle

A module transitions through four states defined in the `module_state` enum.

```c
/* include/linux/module.h:318 */
enum module_state {
    MODULE_STATE_LIVE,      /* Normal state. */
    MODULE_STATE_COMING,    /* Fully formed, running module_init. */
    MODULE_STATE_GOING,     /* Going away. */
    MODULE_STATE_UNFORMED,  /* Still setting it up. */
};
```

### State Transitions

```
  ┌──────────────────────────────────────────────────────────┐
  │              Module State Machine                        │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  load_module() called                                    │
  │         │                                                │
  │         ▼                                                │
  │  ┌──────────────┐                                        │
  │  │   UNFORMED   │  add_unformed_module(): inserted into  │
  │  │              │  modules list, invisible to most       │
  │  └──────┬───────┘  lookups                               │
  │         │                                                │
  │         │ complete_formation()                           │
  │         ▼                                                │
  │  ┌──────────────┐                                        │
  │  │   COMING     │  Visible to kallsyms, notifiers run;   │
  │  │              │  module_init() is called                │
  │  └──────┬───────┘                                        │
  │         │                                                │
  │         │ do_init_module() success                       │
  │         ▼                                                │
  │  ┌──────────────┐                                        │
  │  │    LIVE      │  Fully operational, refcount tracked   │
  │  │              │  Init sections freed                   │
  │  └──────┬───────┘                                        │
  │         │                                                │
  │         │ delete_module() syscall                        │
  │         ▼                                                │
  │  ┌──────────────┐                                        │
  │  │   GOING      │  module_exit() called, free_module()   │
  │  │              │  removes all resources                 │
  │  └──────────────┘                                        │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

The helper `module_is_live()` checks whether a module is not in the GOING
state:

```c
/* include/linux/module.h:613 */
static inline bool module_is_live(struct module *mod)
{
    return mod->state != MODULE_STATE_GOING;
}
```

---

## 3. Core Data Structures

### struct module (include/linux/module.h:410)

The central structure representing a loaded module. It contains all metadata
about the module: its name, state, exported symbols, parameters, memory
regions, and linkage into the global module list. Key fields are shown below
(some conditionally compiled fields omitted for brevity).

```c
/* include/linux/module.h:410 */
struct module {
    enum module_state state;

    /* Member of list of modules */
    struct list_head list;

    /* Unique handle for this module */
    char name[MODULE_NAME_LEN];

    /* Sysfs stuff. */
    struct module_kobject mkobj;
    struct module_attribute *modinfo_attrs;
    const char *version;
    const char *srcversion;
    struct kobject *holders_dir;

    /* Exported symbols */
    const struct kernel_symbol *syms;
    const s32 *crcs;
    unsigned int num_syms;

    /* Kernel parameters. */
    struct kernel_param *kp;
    unsigned int num_kp;

    /* GPL-only exported symbols. */
    unsigned int num_gpl_syms;
    const struct kernel_symbol *gpl_syms;
    const s32 *gpl_crcs;
    bool using_gplonly_symbols;

    /* Signature was verified. */
    bool sig_ok;                        /* #ifdef CONFIG_MODULE_SIG */

    /* Exception table */
    unsigned int num_exentries;
    struct exception_table_entry *extable;

    /* Startup function. */
    int (*init)(void);

    /* Memory regions (text, data, rodata, init_text, etc.) */
    struct module_memory mem[MOD_MEM_NUM_TYPES];

    unsigned long taints;               /* same bits as kernel:taint_flags */

    /* Kallsyms data: protected by RCU and/or module_mutex */
    struct mod_kallsyms __rcu *kallsyms;
    struct mod_kallsyms core_kallsyms;

    /* Per-cpu data. */
    void __percpu *percpu;
    unsigned int percpu_size;

    /* What modules depend on me? */
    struct list_head source_list;       /* #ifdef CONFIG_MODULE_UNLOAD */
    /* What modules do I depend on? */
    struct list_head target_list;       /* #ifdef CONFIG_MODULE_UNLOAD */

    /* Destruction function. */
    void (*exit)(void);                 /* #ifdef CONFIG_MODULE_UNLOAD */
    atomic_t refcnt;                    /* #ifdef CONFIG_MODULE_UNLOAD */

    /* Livepatch */
    bool klp;                           /* #ifdef CONFIG_LIVEPATCH */
    bool klp_alive;
    struct klp_modinfo *klp_info;
} ____cacheline_aligned __randomize_layout;
```

### struct module_memory (include/linux/module.h:368)

Each module has up to seven memory regions, one per `mod_mem_type`. The
separation allows applying different page permissions (R/O, R/W, R/X) to
each region.

```c
/* include/linux/module.h:368 */
struct module_memory {
    void *base;           /* Final mapped address */
    void *rw_copy;        /* Writable shadow (for ROX text) */
    bool is_rox;          /* Read-only-execute mapping? */
    unsigned int size;

    struct mod_tree_node mtn;   /* #ifdef CONFIG_MODULES_TREE_LOOKUP */
};
```

```c
/* include/linux/module.h:330 */
enum mod_mem_type {
    MOD_TEXT = 0,          /* .text */
    MOD_DATA,              /* .data */
    MOD_RODATA,            /* .rodata */
    MOD_RO_AFTER_INIT,     /* .data..ro_after_init */
    MOD_INIT_TEXT,         /* .init.text */
    MOD_INIT_DATA,         /* .init.data */
    MOD_INIT_RODATA,       /* .init.rodata */
    MOD_MEM_NUM_TYPES,
    MOD_INVALID = -1,
};
```

### struct load_info (kernel/module/internal.h:61)

Transient structure used during module loading. It holds the ELF image,
section header cache, and indices into important sections. It is allocated
on-stack or as a local in the load path and freed after load_module()
completes.

```c
/* kernel/module/internal.h:61 */
struct load_info {
    const char *name;
    struct module *mod;          /* pointer into temporary copy */
    Elf_Ehdr *hdr;               /* ELF header */
    unsigned long len;           /* total image length */
    Elf_Shdr *sechdrs;           /* section headers */
    char *secstrings, *strtab;   /* section name table, symbol string table */
    unsigned long symoffs, stroffs, init_typeoffs, core_typeoffs;
    bool sig_ok;

    unsigned long mod_kallsyms_init_off;  /* #ifdef CONFIG_KALLSYMS */

    struct {
        unsigned int sym;   /* .symtab section index */
        unsigned int str;   /* .strtab section index */
        unsigned int mod;   /* .gnu.linkonce.this_module section index */
        unsigned int vers;  /* __versions section index */
        unsigned int info;  /* .modinfo section index */
        unsigned int pcpu;  /* .data..percpu section index */
    } index;
};
```

### struct kernel_symbol (kernel/module/internal.h:35)

Represents one exported symbol. On architectures supporting PC-relative
relocations, offsets are stored instead of absolute pointers to save space
in the symbol table.

```c
/* kernel/module/internal.h:35 */
struct kernel_symbol {
#ifdef CONFIG_HAVE_ARCH_PREL32_RELOCATIONS
    int value_offset;
    int name_offset;
    int namespace_offset;
#else
    unsigned long value;
    const char *name;
    const char *namespace;
#endif
};
```

### struct module_use (include/linux/module.h:312)

Links two modules in a dependency relationship. If module A uses symbols
from module B, a `module_use` entry connects A's `target_list` to B's
`source_list`.

```c
/* include/linux/module.h:312 */
struct module_use {
    struct list_head source_list;
    struct list_head target_list;
    struct module *source, *target;
};
```

### struct module_kobject (include/linux/module.h:45)

The kobject embedded in each module, anchoring it in `/sys/module/<name>/`.

```c
/* include/linux/module.h:45 */
struct module_kobject {
    struct kobject kobj;
    struct module *mod;
    struct kobject *drivers_dir;
    struct module_param_attrs *mp;
    struct completion *kobj_completion;
};
```

---

## 4. System Call Entry Points

Three system calls provide the user-space API for loading and unloading
modules.

### init_module (kernel/module/main.c:3434)

The original loading syscall. User space passes the entire module image as a
memory buffer.

```c
/* kernel/module/main.c:3434 */
SYSCALL_DEFINE3(init_module, void __user *, umod,
                unsigned long, len, const char __user *, uargs)
{
    int err;
    struct load_info info = { };

    err = may_init_module();    /* requires CAP_SYS_MODULE */
    if (err)
        return err;

    err = copy_module_from_user(umod, len, &info);
    if (err)
        return err;

    return load_module(&info, uargs, 0);
}
```

### finit_module (kernel/module/main.c:3588)

The modern file-descriptor-based loading syscall. Avoids copying the entire
module through user memory. Supports compressed modules and idempotent
loading (multiple callers opening the same file converge on one load).

```c
/* kernel/module/main.c:3588 */
SYSCALL_DEFINE3(finit_module, int, fd, const char __user *, uargs, int, flags)
{
    int err = may_init_module();
    if (err)
        return err;

    if (flags & ~(MODULE_INIT_IGNORE_MODVERSIONS
                  |MODULE_INIT_IGNORE_VERMAGIC
                  |MODULE_INIT_COMPRESSED_FILE))
        return -EINVAL;

    CLASS(fd, f)(fd);
    if (fd_empty(f))
        return -EBADF;
    return idempotent_init_module(fd_file(f), uargs, flags);
}
```

The `idempotent_init_module()` function at line 3569 uses a hash table
keyed on the file inode to ensure that concurrent loads of the same module
file converge: only the first caller does the actual load, and subsequent
callers wait for the result.

### delete_module (kernel/module/main.c:732)

Unloads a module. Requires `CAP_SYS_MODULE`, the module must be `LIVE`,
have a zero reference count, and no other modules depending on it.

```c
/* kernel/module/main.c:732 */
SYSCALL_DEFINE2(delete_module, const char __user *, name_user,
                unsigned int, flags)
{
    struct module *mod;
    char name[MODULE_NAME_LEN];
    int ret, forced = 0;

    if (!capable(CAP_SYS_MODULE) || modules_disabled)
        return -EPERM;

    /* ... copy name from user, acquire module_mutex ... */

    mod = find_module(name);
    if (!mod)                          { ret = -ENOENT; goto out; }
    if (!list_empty(&mod->source_list)) { ret = -EWOULDBLOCK; goto out; }
    if (mod->state != MODULE_STATE_LIVE) { ret = -EBUSY; goto out; }

    /* If it has init but no exit, it can't be removed */
    if (mod->init && !mod->exit) {
        forced = try_force_unload(flags);
        if (!forced) { ret = -EBUSY; goto out; }
    }

    ret = try_stop_module(mod, flags, &forced);
    if (ret != 0)
        goto out;

    mutex_unlock(&module_mutex);

    if (mod->exit != NULL)
        mod->exit();

    blocking_notifier_call_chain(&module_notify_list,
                                 MODULE_STATE_GOING, mod);
    klp_module_going(mod);
    ftrace_release_mod(mod);
    free_module(mod);
    wake_up_all(&module_wq);
    return 0;
}
```

---

## 5. The load_module() Pipeline

The `load_module()` function (kernel/module/main.c:3222) orchestrates the
entire loading process. It is a long sequential pipeline where each step may
fail, triggering cleanup via a cascade of `goto` labels.

```
  load_module()                                  kernel/module/main.c:3222
  │
  ├── module_sig_check()                         Verify cryptographic signature
  │
  ├── elf_validity_cache_copy()                  Validate ELF, cache sections
  │
  ├── early_mod_check()                          :3186
  │   ├── blacklisted()                          Check /proc/cmdline blacklist
  │   ├── rewrite_section_headers()              :2310 - Set sh_addr to tmpimage
  │   ├── check_modstruct_version()              CRC of struct module itself
  │   ├── check_modinfo()                        :2403 - Vermagic, livepatch
  │   └── module_patient_check_exists()          Wait for duplicate loads
  │
  ├── layout_and_allocate()                      :2710
  │   ├── module_frob_arch_sections()            Arch adjustments
  │   ├── module_enforce_rwx_sections()          Reject W+X sections
  │   ├── layout_sections()                      Compute per-type sizes
  │   ├── layout_symtab()                        Reserve kallsyms space
  │   └── move_module()                          :2554 - Allocate & copy sections
  │
  ├── add_unformed_module()                      :3086 - Insert into modules list
  │
  ├── module_augment_kernel_taints()             :2337 - Set taint flags
  │
  ├── percpu_modalloc()                          Allocate per-CPU data
  │
  ├── module_unload_init()                       Init refcount & dep lists
  │
  ├── find_module_sections()                     :2429 - Locate ELF sections
  │
  ├── check_export_symbol_versions()             Verify CRC of exports
  │
  ├── simplify_symbols()                         :1444 - Resolve SHN_UNDEF
  │
  ├── apply_relocations()                        :1515 - SHT_REL / SHT_RELA
  │
  ├── post_relocation()                          :2792
  │   ├── sort_extable()                         Sort exception table
  │   ├── add_kallsyms()                         Install symbol table
  │   └── module_finalize()                      Arch-specific finalization
  │
  ├── complete_formation()                       :3107
  │   ├── verify_exported_symbols()              No duplicate exports
  │   ├── module_enable_rodata_ro()              Mark R/O pages
  │   ├── module_enable_data_nx()                Mark NX on data
  │   ├── module_enable_text_rox()               Mark R/X on text
  │   └── state = MODULE_STATE_COMING
  │
  ├── prepare_coming_module()                    :3148
  │   ├── ftrace_module_enable()                 Register ftrace hooks
  │   ├── klp_module_coming()                    Notify live patching
  │   └── blocking_notifier_call_chain()         MODULE_STATE_COMING
  │
  ├── parse_args()                               Process module parameters
  │
  ├── mod_sysfs_setup()                          Create /sys/module/<name>/
  │
  ├── copy_module_elf()                          Preserve ELF for livepatch
  │
  └── do_init_module()                           :2881 - Call mod->init()
```

---

## 6. ELF Parsing and Validation

### ELF Validity Check

The function `elf_validity_cache_copy()` performs comprehensive validation
of the module ELF image. It verifies:

- ELF magic number and class (ELF64 vs ELF32)
- Machine type matches current architecture
- File type is `ET_REL` (relocatable object)
- Section headers are within bounds
- The `.gnu.linkonce.this_module` section exists and matches `sizeof(struct module)`
- The `.modinfo` section contains required tags
- Symbol and string table sections exist

### Section Header Rewriting

After initial validation, `rewrite_section_headers()` (kernel/module/main.c:2310)
updates each section's `sh_addr` to point into the temporary in-memory copy
of the ELF file. This allows subsequent code to use `sh_addr` as a direct
pointer during processing.

```c
/* kernel/module/main.c:2310 */
static int rewrite_section_headers(struct load_info *info, int flags)
{
    unsigned int i;

    info->sechdrs[0].sh_addr = 0;

    for (i = 1; i < info->hdr->e_shnum; i++) {
        Elf_Shdr *shdr = &info->sechdrs[i];
        shdr->sh_addr = (size_t)info->hdr + shdr->sh_offset;
    }

    /* Track but don't keep modinfo and version sections. */
    info->sechdrs[info->index.vers].sh_flags &= ~(unsigned long)SHF_ALLOC;
    info->sechdrs[info->index.info].sh_flags &= ~(unsigned long)SHF_ALLOC;

    return 0;
}
```

### Vermagic Check

`check_modinfo()` (kernel/module/main.c:2403) validates the vermagic string
embedded in the module's `.modinfo` section against the running kernel's
vermagic. This catches ABI mismatches (different kernel version, SMP
configuration, preemption model, etc.). The `MODULE_INIT_IGNORE_VERMAGIC`
flag can bypass this check (used by `modprobe --force`).

---

## 7. Memory Layout and Allocation

### Layout Computation

`layout_sections()` iterates over all ELF sections with `SHF_ALLOC` and
assigns each to one of the seven `mod_mem_type` bins based on section flags:

```
  Section Flags               mod_mem_type        Permissions
  ───────────────────────────  ──────────────────  ─────────────
  SHF_ALLOC | SHF_EXECINSTR   MOD_TEXT            R-X
  SHF_ALLOC | SHF_WRITE       MOD_DATA            RW-
  SHF_ALLOC (read-only)       MOD_RODATA          R--
  SHF_RO_AFTER_INIT           MOD_RO_AFTER_INIT   RW- -> R--
  .init.text                  MOD_INIT_TEXT        R-X (freed)
  .init.data                  MOD_INIT_DATA        RW- (freed)
  .init.rodata                MOD_INIT_RODATA      R-- (freed)
```

The `module_get_offset_and_type()` function (kernel/module/main.c:1558)
computes each section's offset within its bin, respecting alignment. It
encodes both the offset and the `mod_mem_type` into `sh_entsize`, using the
top 4 bits for the type:

```c
/* kernel/module/main.c:1558 */
long module_get_offset_and_type(struct module *mod, enum mod_mem_type type,
                                Elf_Shdr *sechdr, unsigned int section)
{
    long offset;
    long mask = ((unsigned long)(type) & SH_ENTSIZE_TYPE_MASK)
                << SH_ENTSIZE_TYPE_SHIFT;

    mod->mem[type].size += arch_mod_section_prepend(mod, section);
    offset = ALIGN(mod->mem[type].size, sechdr->sh_addralign ?: 1);
    mod->mem[type].size = offset + sechdr->sh_size;

    return offset | mask;
}
```

### Memory Allocation and Copying

`move_module()` (kernel/module/main.c:2554) allocates memory for each
non-empty `mod_mem_type` via `module_memory_alloc()` (which uses the
`execmem` allocator), then copies each `SHF_ALLOC` section into its final
location. After copying, it updates `sh_addr` to point to the final
address. This is the point at which the module transitions from the
temporary vmalloc'd image to its permanent home.

```
  ┌─────────────────────────────────────────────────────────┐
  │          Module Memory Layout (after move_module)       │
  ├─────────────────────────────────────────────────────────┤
  │                                                         │
  │  mod->mem[MOD_TEXT]       ┌─────────────────────────┐   │
  │     .base ───────────────▶│ .text (R-X)             │   │
  │                           │ .text.unlikely          │   │
  │                           └─────────────────────────┘   │
  │                                                         │
  │  mod->mem[MOD_RODATA]     ┌─────────────────────────┐   │
  │     .base ───────────────▶│ .rodata (R--)           │   │
  │                           │ __ksymtab               │   │
  │                           │ __kcrctab               │   │
  │                           └─────────────────────────┘   │
  │                                                         │
  │  mod->mem[MOD_DATA]       ┌─────────────────────────┐   │
  │     .base ───────────────▶│ .data (RW-)             │   │
  │                           │ .gnu.linkonce.this_module│   │
  │                           │ core_kallsyms (symtab)  │   │
  │                           └─────────────────────────┘   │
  │                                                         │
  │  mod->mem[MOD_INIT_TEXT]  ┌─────────────────────────┐   │
  │     .base ───────────────▶│ .init.text (R-X)        │   │
  │                           └─────────────────────────┘   │
  │                           Freed after init completes    │
  │                                                         │
  │  mod->mem[MOD_INIT_DATA]  ┌─────────────────────────┐   │
  │     .base ───────────────▶│ .init.data (RW-)        │   │
  │                           │ init kallsyms (full)    │   │
  │                           └─────────────────────────┘   │
  │                           Freed after init completes    │
  │                                                         │
  └─────────────────────────────────────────────────────────┘
```

### Module Tree for Address Lookup

Modules are inserted into a latched RB-tree (`mod_tree`, defined at
kernel/module/main.c:83) for fast address-to-module lookups. The functions
`mod_tree_insert()`, `mod_tree_remove()`, and `mod_find()` manage this
tree, which is used by `__module_address()` and `__module_text_address()`.

---

## 8. Symbol Resolution and Dependencies

### The find_symbol() Function

Symbol resolution is central to module loading. The internal function
`find_symbol()` (declared in kernel/module/internal.h:112) searches for a
named symbol across:
1. The kernel's built-in symbol tables (`__ksymtab` and `__ksymtab_gpl`)
2. All loaded modules' exported symbol tables

It uses binary search via `bsearch()` on the sorted kernel symbol tables.

### The find_symbol_arg Structure

```c
/* kernel/module/internal.h:97 */
struct find_symbol_arg {
    /* Input */
    const char *name;
    bool gplok;         /* Can use GPL-only symbols? */
    bool warn;          /* Warn on failure? */

    /* Output */
    struct module *owner;         /* Module owning the symbol */
    const s32 *crc;               /* Symbol version CRC */
    const struct kernel_symbol *sym;  /* The symbol itself */
    enum mod_license license;     /* GPL_ONLY or NOT_GPL_ONLY */
};
```

### resolve_symbol() (kernel/module/main.c:1143)

Called for each undefined symbol during `simplify_symbols()`. It looks up
the symbol, checks GPL compatibility, verifies the CRC version, validates
namespace imports, and creates a dependency reference via `ref_module()`.

```c
/* kernel/module/main.c:1143 */
static const struct kernel_symbol *resolve_symbol(struct module *mod,
                                                  const struct load_info *info,
                                                  const char *name,
                                                  char ownername[])
{
    struct find_symbol_arg fsa = {
        .name   = name,
        .gplok  = !(mod->taints & (1 << TAINT_PROPRIETARY_MODULE)),
        .warn   = true,
    };

    mutex_lock(&module_mutex);
    if (!find_symbol(&fsa))
        goto unlock;

    if (fsa.license == GPL_ONLY)
        mod->using_gplonly_symbols = true;

    if (!check_version(info, name, mod, fsa.crc)) {
        fsa.sym = ERR_PTR(-EINVAL);
        goto getname;
    }

    err = ref_module(mod, fsa.owner);     /* Create dependency */
    /* ... */
}
```

### resolve_symbol_wait() (kernel/module/main.c:1198)

Wraps `resolve_symbol()` with a 30-second wait-with-timeout. If a depended
module is still loading (returning `-EBUSY`), it waits on `module_wq` for
that module to finish initialization.

### Module Dependencies

Dependencies are tracked through the `module_use` linked lists. When module
A resolves a symbol from module B:
- `ref_module(A, B)` creates a `module_use` and adds it to both:
  - A's `target_list` (modules A depends on)
  - B's `source_list` (modules that depend on B)

During unloading, `delete_module()` checks `mod->source_list`: if any
other module depends on the target module, unloading is refused with
`-EWOULDBLOCK`.

### Symbol Version (CRC) Checking

When `CONFIG_MODVERSIONS` is enabled, each exported symbol has an associated
CRC. `check_version()` (kernel/module/internal.h:386) compares the CRC the
module was compiled against (from `__versions` section) with the current
kernel's CRC for that symbol. A mismatch indicates an ABI change and the
load fails with `-ENOEXEC`.

---

## 9. Relocations

### apply_relocations() (kernel/module/main.c:1515)

After symbols are resolved and `st_value` updated to point to the actual
addresses, `apply_relocations()` processes all relocation sections. It
iterates over every section of type `SHT_REL` or `SHT_RELA` and calls the
architecture-specific `apply_relocate()` or `apply_relocate_add()` to patch
code and data references.

```c
/* kernel/module/main.c:1515 */
static int apply_relocations(struct module *mod, const struct load_info *info)
{
    unsigned int i;
    int err = 0;

    for (i = 1; i < info->hdr->e_shnum; i++) {
        unsigned int infosec = info->sechdrs[i].sh_info;

        if (infosec >= info->hdr->e_shnum)
            continue;
        if (!(info->sechdrs[infosec].sh_flags & SHF_ALLOC))
            continue;

        if (info->sechdrs[i].sh_flags & SHF_RELA_LIVEPATCH)
            err = klp_apply_section_relocs(mod, ...);
        else if (info->sechdrs[i].sh_type == SHT_REL)
            err = apply_relocate(info->sechdrs, info->strtab,
                                 info->index.sym, i, mod);
        else if (info->sechdrs[i].sh_type == SHT_RELA)
            err = apply_relocate_add(info->sechdrs, info->strtab,
                                     info->index.sym, i, mod);
        if (err < 0)
            break;
    }
    return err;
}
```

Livepatch relocation sections (flagged with `SHF_RELA_LIVEPATCH`) are
handled specially by `klp_apply_section_relocs()`, which resolves symbols
against target modules rather than the module being loaded.

---

## 10. Module Signing and Verification

### Overview

Module signature verification uses PKCS#7 / X.509 cryptography to ensure
that only trusted code is loaded into the kernel. The signature is appended
to the `.ko` file at build time by `scripts/sign-file`.

### Signature Data Format

```c
/* include/linux/module_signature.h:33 */
struct module_signature {
    u8      algo;           /* Public-key crypto algorithm [0] */
    u8      hash;           /* Digest algorithm [0] */
    u8      id_type;        /* Key identifier type [PKEY_ID_PKCS7] */
    u8      signer_len;     /* Length of signer's name [0] */
    u8      key_id_len;     /* Length of key identifier [0] */
    u8      __pad[3];
    __be32  sig_len;        /* Length of signature data */
};
```

The module file layout with a signature appended:

```
  ┌─────────────────────────────────────────────────┐
  │  ELF relocatable object (.ko)                   │
  │  ┌─────────────────────────────────────────────┐│
  │  │ ELF Header                                  ││
  │  │ Section Headers                             ││
  │  │ .text, .data, .rodata, .symtab, ...         ││
  │  └─────────────────────────────────────────────┘│
  │  ┌─────────────────────────────────────────────┐│
  │  │ Signer's name (optional, currently 0 len)   ││
  │  │ Key identifier (optional, currently 0 len)  ││
  │  │ PKCS#7 signature data (sig_len bytes)       ││
  │  │ struct module_signature (12 bytes)           ││
  │  └─────────────────────────────────────────────┘│
  │  MODULE_SIG_STRING marker:                      │
  │  "~Module signature appended~\n"                │
  └─────────────────────────────────────────────────┘
```

### Verification Flow

`module_sig_check()` (kernel/module/signing.c:70) is the first step in
`load_module()`. It performs the following:

1. Checks for the `MODULE_SIG_STRING` sentinel at the end of the file
2. Strips the sentinel and reads the `module_signature` trailer
3. Calls `mod_verify_sig()` which:
   - Validates the signature structure via `mod_check_sig()`
   - Calls `verify_pkcs7_signature()` to verify the PKCS#7 signature
     against `VERIFY_USE_SECONDARY_KEYRING`
4. If verification succeeds, sets `info->sig_ok = true`
5. If verification fails and `sig_enforce` is true, rejects with `-EKEYREJECTED`
6. If not enforcing, falls back to `security_locked_down(LOCKDOWN_MODULE_SIGNATURE)`

```c
/* kernel/module/signing.c:70 */
int module_sig_check(struct load_info *info, int flags)
{
    int err = -ENODATA;
    const unsigned long markerlen = sizeof(MODULE_SIG_STRING) - 1;
    const void *mod = info->hdr;

    if (info->len > markerlen &&
        memcmp(mod + info->len - markerlen,
               MODULE_SIG_STRING, markerlen) == 0) {
        info->len -= markerlen;
        err = mod_verify_sig(mod, info);
        if (!err) {
            info->sig_ok = true;
            return 0;
        }
    }

    if (is_module_sig_enforced()) {
        pr_notice("Loading of %s is rejected\n", reason);
        return -EKEYREJECTED;
    }

    return security_locked_down(LOCKDOWN_MODULE_SIGNATURE);
}
```

### Configuration Options

- `CONFIG_MODULE_SIG`: Enable module signature checking infrastructure
- `CONFIG_MODULE_SIG_FORCE`: Reject unsigned/badly-signed modules (also
  controllable via `module.sig_enforce` boot parameter)
- `CONFIG_MODULE_SIG_SHA256` (or SHA384/SHA512): Hash algorithm for signing

---

## 11. Module Parameters

Module parameters allow users to pass configuration values to modules at
load time (via `modprobe` arguments) or at runtime (via
`/sys/module/<name>/parameters/`).

### kernel_param Structure (include/linux/moduleparam.h:69)

```c
/* include/linux/moduleparam.h:69 */
struct kernel_param {
    const char *name;
    struct module *mod;
    const struct kernel_param_ops *ops;
    const u16 perm;         /* sysfs file permissions */
    s8 level;               /* initcall level (for built-in) */
    u8 flags;               /* KERNEL_PARAM_FL_UNSAFE, FL_HWPARAM */
    union {
        void *arg;
        const struct kparam_string *str;
        const struct kparam_array *arr;
    };
};
```

### kernel_param_ops Structure (include/linux/moduleparam.h:47)

```c
/* include/linux/moduleparam.h:47 */
struct kernel_param_ops {
    unsigned int flags;                /* KERNEL_PARAM_OPS_FL_NOARG */
    int (*set)(const char *val, const struct kernel_param *kp);
    int (*get)(char *buffer, const struct kernel_param *kp);
    void (*free)(void *arg);           /* Called on module unload */
};
```

### Parameter Processing

Parameters are defined in the module source with macros like `module_param()`,
which create `struct kernel_param` entries in the `__param` ELF section.
During loading:

1. `find_module_sections()` locates the `__param` section and sets
   `mod->kp` and `mod->num_kp`
2. `parse_args()` (kernel/module/main.c:3347) processes the user-supplied
   argument string, matching each `key=value` pair against `mod->kp`
3. `mod_sysfs_setup()` calls `module_param_sysfs_setup()` to create
   per-parameter files under `/sys/module/<name>/parameters/`

Parameters with `perm != 0` are exposed as sysfs files and can be read
or written at runtime, protected by `mod->param_lock`.

---

## 12. Module Initialization and Teardown

### do_init_module() (kernel/module/main.c:2881)

Called at the end of `load_module()`, this function runs the module's
initialization code and transitions the module to `MODULE_STATE_LIVE`.

```c
/* kernel/module/main.c:2881 */
static noinline int do_init_module(struct module *mod)
{
    int ret = 0;

    /* Run C++ style constructors if present */
    do_mod_ctors(mod);

    /* Start the module */
    if (mod->init != NULL)
        ret = do_one_initcall(mod->init);
    if (ret < 0)
        goto fail;

    /* Now it's a first class citizen! */
    mod->state = MODULE_STATE_LIVE;
    blocking_notifier_call_chain(&module_notify_list,
                                 MODULE_STATE_LIVE, mod);

    /* Delay uevent until module has finished its init routine */
    kobject_uevent(&mod->mkobj.kobj, KOBJ_ADD);

    /* Wait for async probing if not requested to be async */
    if (!mod->async_probe_requested)
        async_synchronize_full();

    mutex_lock(&module_mutex);
    module_put(mod);                /* Drop initial reference */

    /* Switch to core kallsyms now init is done */
    rcu_assign_pointer(mod->kallsyms, &mod->core_kallsyms);

    /* Set ro_after_init sections truly read-only */
    module_enable_rodata_ro(mod, true);

    /* Remove init sections from module tree */
    mod_tree_remove_init(mod);

    /* Clear init section pointers - memory will be freed */
    for_class_mod_mem_type(type, init) {
        mod->mem[type].base = NULL;
        mod->mem[type].size = 0;
    }

    /* Schedule async free of init sections (needs RCU grace period) */
    if (llist_add(&freeinit->node, &init_free_list))
        schedule_work(&init_free_wq);

    mutex_unlock(&module_mutex);
    wake_up_all(&module_wq);
    return 0;
}
```

Key points:
- After `init()` succeeds, state changes to `MODULE_STATE_LIVE`
- The full kallsyms symbol table (in init data) is replaced by the
  core-only subset via `rcu_assign_pointer()`
- Init text/data/rodata memory is freed asynchronously via a work queue
  after an RCU grace period (to avoid racing with kallsyms walkers)

### Module Notifier Chain

The kernel maintains a blocking notifier chain (`module_notify_list`) that
other subsystems can register on. Notifications are sent for:
- `MODULE_STATE_COMING`: After formation, before init
- `MODULE_STATE_LIVE`: After successful init
- `MODULE_STATE_GOING`: During unload

Subscribers include ftrace, livepatch, tracepoints, and BPF.

---

## 13. Module Unloading

### Reference Counting

When `CONFIG_MODULE_UNLOAD` is enabled, each module maintains an atomic
reference count (`mod->refcnt`). The key APIs are:

```c
/* kernel/module/main.c:854 */
void __module_get(struct module *module)
{
    if (module)
        atomic_inc(&module->refcnt);
}

/* kernel/module/main.c:863 */
bool try_module_get(struct module *module)
{
    if (module) {
        if (likely(module_is_live(module) &&
                   atomic_inc_not_zero(&module->refcnt) != 0))
            return true;        /* Got it */
        else
            return false;       /* Module going away */
    }
    return true;                /* NULL module = built-in, always OK */
}

/* kernel/module/main.c:879 */
void module_put(struct module *module)
{
    if (module) {
        int ret = atomic_dec_if_positive(&module->refcnt);
        WARN_ON(ret < 0);      /* Already zero */
    }
}
```

`try_module_get()` is the safe version: it returns false if the module is
in `MODULE_STATE_GOING`, preventing new references to a dying module.

### Unload Sequence

```
  delete_module("mymod", flags)
  │
  ├── find_module("mymod")                    Locate in module list
  ├── Check source_list empty                 No dependents
  ├── Check state == MODULE_STATE_LIVE
  ├── try_stop_module()                       Set refcnt to 0 or wait
  │
  ├── mod->exit()                             Run cleanup function
  │
  ├── blocking_notifier_call_chain(GOING)     Notify subsystems
  ├── klp_module_going()                      Livepatch cleanup
  ├── ftrace_release_mod()                    Ftrace cleanup
  │
  └── free_module()
      ├── mod_sysfs_teardown()                Remove sysfs entries
      ├── mod_tree_remove()                   Remove from address tree
      ├── list_del_rcu() from modules list    RCU-safe removal
      ├── synchronize_rcu()                   Wait for readers
      ├── module_arch_cleanup()               Arch cleanup
      └── free_mod_mem()                      Free all memory regions
```

---

## 14. Sysfs and Procfs Interfaces

### Sysfs: /sys/module/

Each loaded module creates a kobject hierarchy under `/sys/module/<name>/`.
The setup is done by `mod_sysfs_setup()` (kernel/module/sysfs.c:371).

```
  /sys/module/<name>/
  │
  ├── holders/                  Symlinks to modules that depend on us
  │   └── <depmod_name> -> ../../<depmod_name>/
  │
  ├── parameters/               Module parameter files
  │   ├── param1                (perm from module_param)
  │   └── param2
  │
  ├── sections/                 ELF section addresses (root-only)
  │   ├── .text                 Hex address of .text
  │   ├── .data
  │   ├── .bss
  │   └── ...
  │
  ├── notes/                    ELF SHT_NOTE section contents
  │   └── .note.gnu.build-id
  │
  ├── refcnt                    Current reference count
  ├── initstate                 "live", "coming", or "going"
  ├── taint                     Taint flags string
  ├── srcversion                Source checksum
  └── version                   Module version string
```

The `module_kobject` is initialized in `mod_sysfs_init()`
(kernel/module/sysfs.c:339), which creates the kobject under the global
`module_kset`. Section attributes are added by `add_sect_attrs()`
(kernel/module/sysfs.c:72), and note attributes by `add_notes_attrs()`
(kernel/module/sysfs.c:166).

### Procfs: /proc/modules

The `/proc/modules` file (kernel/module/procfs.c:147) provides a
one-line-per-module summary used by `lsmod`. Format:

```
name  size  refcount  dependents  state  address
ext4  802816  1  -  Live  0xffffffffc0400000
```

The `m_show()` function (kernel/module/procfs.c:74) generates each line:

```c
/* kernel/module/procfs.c:74 */
static int m_show(struct seq_file *m, void *p)
{
    struct module *mod = list_entry(p, struct module, list);

    if (mod->state == MODULE_STATE_UNFORMED)
        return 0;

    seq_printf(m, "%s %u", mod->name, module_total_size(mod));
    print_unload_info(m, mod);

    seq_printf(m, " %s",
               mod->state == MODULE_STATE_GOING ? "Unloading" :
               mod->state == MODULE_STATE_COMING ? "Loading" :
               "Live");

    value = m->private ? NULL : mod->mem[MOD_TEXT].base;
    seq_printf(m, " 0x%px", value);

    if (mod->taints)
        seq_printf(m, " %s", module_flags(mod, buf, true));

    seq_puts(m, "\n");
    return 0;
}
```

---

## 15. Kallsyms Integration

### Overview

Modules integrate with the kernel's symbol lookup infrastructure (kallsyms)
so that stack traces, `/proc/kallsyms`, and debuggers can resolve addresses
within module code to symbolic names.

### mod_kallsyms Structure (include/linux/module.h:386)

```c
/* include/linux/module.h:386 */
struct mod_kallsyms {
    Elf_Sym *symtab;            /* Symbol table */
    unsigned int num_symtab;    /* Number of symbols */
    char *strtab;               /* String table */
    char *typetab;              /* Per-symbol type (t,d,r,...) */
};
```

### Two-Phase Symbol Table

During loading, modules maintain two symbol tables:

1. **Init phase** (`mod->kallsyms` points to init data): The full symbol
   table from the ELF file, including symbols in init sections. Set up by
   `add_kallsyms()` (kernel/module/kallsyms.c:170).

2. **Core phase** (`mod->core_kallsyms`): A stripped-down table containing
   only symbols from core (non-init) sections. Prepared by `layout_symtab()`
   and populated by `add_kallsyms()`.

The switch happens atomically via RCU in `do_init_module()`:

```c
/* kernel/module/main.c:2949 */
rcu_assign_pointer(mod->kallsyms, &mod->core_kallsyms);
```

### Address-to-Symbol Lookup

`find_kallsyms_symbol()` (kernel/module/kallsyms.c:256) finds the nearest
preceding symbol for a given address by scanning the module's symbol table.
It is called from `module_address_lookup()` (kernel/module/kallsyms.c:324),
which disables preemption and uses `__module_address()` to first identify
which module owns the address.

### Name-to-Address Lookup

`module_kallsyms_lookup_name()` (kernel/module/kallsyms.c:454) supports the
`module:symbol` syntax. It calls `__module_kallsyms_lookup_name()` which
either:
- Searches a specific module if a colon-separated module name is given
- Iterates all modules otherwise

---

## 16. Module Tainting

### Overview

Modules can taint the kernel, marking it as running with potentially
unsupported or problematic code. Taints are stored as a bitmask in
`mod->taints` and also propagated to the global kernel taint flags.

### Taint Sources

`module_augment_kernel_taints()` (kernel/module/main.c:2337) applies taints
based on module properties:

| Taint Flag               | Condition                                    |
|--------------------------|----------------------------------------------|
| `TAINT_OOT_MODULE`       | No `intree` tag in `.modinfo`                |
| `TAINT_PROPRIETARY_MODULE` | Non-GPL `license` in `.modinfo`            |
| `TAINT_CRAP`             | `staging` tag in `.modinfo`                  |
| `TAINT_LIVEPATCH`        | `livepatch` tag in `.modinfo`                |
| `TAINT_UNSIGNED_MODULE`  | Signature verification failed                |
| `TAINT_TEST`             | `test` tag in `.modinfo`                     |
| `TAINT_FORCED_MODULE`    | Loaded with `--force` (bad vermagic/version) |

### GPL Symbol Enforcement

Proprietary (non-GPL) modules are blocked from using `EXPORT_SYMBOL_GPL`
symbols. In `resolve_symbol()`, the `gplok` flag is derived from:

```c
.gplok = !(mod->taints & (1 << TAINT_PROPRIETARY_MODULE)),
```

If a proprietary module tries to use a GPL-only symbol, `find_symbol()`
returns `NULL` and the load fails with "Unknown symbol".

### Unloaded Module Taint Tracking

When `CONFIG_MODULE_UNLOAD_TAINT_TRACKING` is enabled, the kernel records
the taints of unloaded modules in a `struct mod_unload_taint` list so that
post-mortem analysis can determine which tainted modules were loaded even
after they have been removed.

---

## 17. Live Patching Hooks

### Overview

The kernel live patching infrastructure (kpatch/livepatch) hooks into the
module subsystem at several points. When a livepatch module is loaded, it
must be able to look up symbols in target modules and apply patches to their
functions.

### Livepatch Module Identification

`check_modinfo_livepatch()` (kernel/module/main.c:2246) detects the
`livepatch` tag in `.modinfo` and sets `mod->klp = true`.

### ELF Preservation

For livepatch modules, `copy_module_elf()` (kernel/module/livepatch.c:18)
preserves ELF metadata that is normally discarded after loading. This data
is stored in `struct klp_modinfo`:

```c
/* include/linux/module.h:402 */
struct klp_modinfo {
    Elf_Ehdr hdr;              /* Copy of ELF header */
    Elf_Shdr *sechdrs;         /* Copy of section header table */
    char *secstrings;          /* Copy of section string table */
    unsigned int symndx;       /* Symbol table section index */
};
```

The preserved data allows the livepatch subsystem to perform symbol
resolution against the module's ELF sections even after init sections have
been freed.

### Livepatch Relocation Handling

Relocation sections marked with `SHF_RELA_LIVEPATCH` are processed by
`klp_apply_section_relocs()` instead of the normal relocation path. This
allows livepatch to resolve symbols against arbitrary modules rather than
the module being loaded.

### Module Coming/Going Hooks

The `prepare_coming_module()` function (kernel/module/main.c:3148) calls
`klp_module_coming()` to notify livepatch when a new module is loaded. This
allows livepatch to apply any pending patches to the newly loaded module.
Similarly, `delete_module()` calls `klp_module_going()` before freeing.

---

## 18. Locking and Concurrency

### module_mutex (kernel/module/main.c:75)

The primary mutex protecting module operations:

```c
/* kernel/module/main.c:75 */
DEFINE_MUTEX(module_mutex);
```

It protects:
- The `modules` linked list
- `module_use` dependency links
- `mod_tree` address range tracking
- Module state transitions

### modules List and RCU

The global module list is defined at kernel/module/main.c:76:

```c
LIST_HEAD(modules);
```

Insertions and deletions use RCU list operations (`list_add_rcu()`,
`list_del_rcu()`) to allow lock-free read-side traversal. Readers can
iterate with either:
- `preempt_disable()` + `list_for_each_entry_rcu()` (used by kallsyms)
- `module_mutex` held

### module_wq (kernel/module/main.c:133)

A wait queue used to coordinate between concurrent module load/unload
operations:

```c
static DECLARE_WAIT_QUEUE_HEAD(module_wq);
```

Waiters include:
- `resolve_symbol_wait()`: Waits for a depended module to finish loading
- `module_patient_check_exists()`: Waits for a duplicate module to finish
  loading or fail
- Woken by `do_init_module()` and `free_module()` via
  `wake_up_all(&module_wq)`

### Idempotent Loading (kernel/module/main.c:3457)

The `finit_module()` path uses a hash table (`idem_hash`) with a spinlock
(`idem_lock`) to deduplicate concurrent loads of the same module file. Only
the first caller per inode actually loads; others wait on a per-entry
completion.

### Locking Summary

| Lock              | Type     | Protects                              |
|-------------------|----------|---------------------------------------|
| `module_mutex`    | mutex    | modules list, module_use, mod_tree    |
| `idem_lock`       | spinlock | idempotent loading hash table         |
| `mod->param_lock` | mutex    | sysfs parameter read/write            |
| RCU               | (none)   | Lock-free module list read traversal  |

---

## 19. Source File Map

```
  kernel/module/main.c          Primary module loading/unloading logic,
                                syscall definitions, load_module() pipeline,
                                symbol resolution, relocation dispatch
                                (~3600 lines)

  kernel/module/internal.h      Internal structs: load_info, kernel_symbol,
                                find_symbol_arg, module tree, and internal
                                function declarations

  include/linux/module.h        Public API: struct module, module_state,
                                module_memory, mod_kallsyms, try_module_get,
                                module_put, MODULE_* macros

  include/linux/moduleparam.h   Module parameter infrastructure: kernel_param,
                                kernel_param_ops, module_param() macros

  include/linux/module_signature.h  module_signature struct, MODULE_SIG_STRING

  kernel/module/signing.c       Signature verification: module_sig_check(),
                                mod_verify_sig(), sig_enforce parameter

  kernel/module/kallsyms.c      Kallsyms integration: add_kallsyms(),
                                layout_symtab(), module_address_lookup(),
                                module_kallsyms_lookup_name()

  kernel/module/sysfs.c         Sysfs interface: mod_sysfs_setup(),
                                section/notes attributes, holders links,
                                modinfo attributes

  kernel/module/procfs.c        /proc/modules: seq_file implementation

  kernel/module/livepatch.c     Live patching ELF preservation:
                                copy_module_elf(), free_module_elf()

  kernel/module/strict_rwx.c    Memory protection: module_enable_rodata_ro(),
                                module_enable_data_nx(), module_enable_text_rox()

  kernel/module/decompress.c    Module decompression (gzip/xz/zstd)

  kernel/module/tracking.c      Taint tracking for unloaded modules

  kernel/module/stats.c         Module loading statistics (counts, sizes)

  kernel/module/kmod.c          Automatic module loading (request_module)

  kernel/module/dups.c          Duplicate load request detection

  kernel/module/tree_lookup.c   Latched RB-tree for address lookups

  kernel/module/version.c       Module version / CRC checking
```

---

## 20. Summary

The Linux kernel module subsystem provides a complete infrastructure for
dynamically extending the kernel with loadable code.

| Component              | Purpose                                        |
|------------------------|------------------------------------------------|
| System calls           | `init_module`, `finit_module`, `delete_module`  |
| ELF processing         | Parse, validate, layout, relocate `.ko` files   |
| Symbol resolution      | Link undefined symbols to kernel/module exports |
| Dependency tracking    | `module_use` lists, refcount management          |
| Signature verification | PKCS#7 cryptographic module authentication      |
| Module parameters      | Runtime-configurable values via sysfs            |
| Memory management      | 7-region layout with per-region R/W/X perms     |
| Sysfs/Procfs           | `/sys/module/`, `/proc/modules` interfaces      |
| Kallsyms               | Address-to-symbol and symbol-to-address lookups |
| Tainting               | Track proprietary, out-of-tree, unsigned modules|
| Live patching          | ELF preservation, special relocations, hooks    |

Key design decisions:
- **Security**: Modules require `CAP_SYS_MODULE` and can optionally require
  cryptographic signatures. Non-GPL modules are blocked from GPL-only symbols.
- **Safety**: Vermagic and CRC version checks prevent loading modules compiled
  against incompatible kernels. W+X sections are rejected.
- **Concurrency**: RCU protects lock-free reads of the module list and
  kallsyms tables. The idempotent loading mechanism prevents wasted work
  from concurrent identical loads.
- **Memory efficiency**: Init sections are freed after initialization.
  Kallsyms tables are trimmed to core-only symbols after init.
