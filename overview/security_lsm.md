# Linux Kernel Security and LSM Subsystem

## Overview

The Linux Security Module (LSM) framework provides a hook-based infrastructure
that allows multiple security modules to enforce mandatory access control
policies. The framework supports stacking multiple LSMs, each intercepting
security-relevant kernel operations. Built-in modules include capabilities
(POSIX), SELinux, AppArmor, Landlock, seccomp, and BPF LSM. The integrity
subsystem (IMA/EVM) provides file measurement and verification.

---

## LSM Framework

### Hook Infrastructure

All hooks are defined using the `LSM_HOOK` macro in
`include/linux/lsm_hook_defs.h`:

```c
LSM_HOOK(<return_type>, <default_value>, <hook_name>, args...)
```

This file is `#include`-d repeatedly with different macro definitions to
generate data structures, static calls, and the dispatch table simultaneously.

### Key Data Structures

#### `struct lsm_id`

`include/linux/lsm_hooks.h:81`

```c
struct lsm_id {
    const char *name;  // e.g. "selinux", "bpf"
    u64 id;            // numeric ID from uapi/linux/lsm.h
};
```

#### `struct security_hook_list`

`include/linux/lsm_hooks.h:95`

```c
struct security_hook_list {
    struct lsm_static_call       *scalls;  // static call slot array
    union security_list_options   hook;    // actual callback pointer
    const struct lsm_id          *lsmid;  // owning LSM
};
```

#### `struct lsm_blob_sizes`

`include/linux/lsm_hooks.h:104`

Each LSM declares how many bytes it needs in each kernel object's security blob:

```c
struct lsm_blob_sizes {
    int lbs_cred;        // bytes in cred->security
    int lbs_file;        // bytes in file->f_security
    int lbs_inode;       // bytes in inode->i_security
    int lbs_sock;        // bytes in sock->sk_security
    int lbs_task;        // bytes in task blob
    int lbs_superblock;  // bytes in super_block blob
    int lbs_ipc;         // bytes in kern_ipc_perm blob
    int lbs_key;         // bytes in key blob
    int lbs_msg_msg;     // bytes in msg_msg blob
    int lbs_xattr_count; // extra xattr slots needed
};
```

The framework accumulates these at init to allocate single contiguous blobs.
Each LSM accesses its portion by adding its pre-computed offset to the base
pointer.

#### `struct lsm_info`

`include/linux/lsm_hooks.h:151`

```c
struct lsm_info {
    const char *name;
    enum lsm_order order;     // LSM_ORDER_FIRST / MUTABLE / LAST
    unsigned long flags;      // LSM_FLAG_LEGACY_MAJOR, LSM_FLAG_EXCLUSIVE
    int *enabled;
    int (*init)(void);
    struct lsm_blob_sizes *blobs;
};
```

LSMs register via `DEFINE_LSM(name)` which places `lsm_info` in the
`.lsm_info.init` linker section. Early LSMs use `DEFINE_EARLY_LSM`.

### Static Call Dispatch

Rather than linked-list traversal, the framework uses kernel static calls. Each
hook has up to `MAX_LSM_COUNT` slots. The dispatch unrolls into a compile-time
loop of static calls guarded by `static_key_false` branch checks — equivalent
to inlined direct calls at runtime with no indirection overhead.

### Hook Registration

`security_add_hooks()` at `security/security.c:614`:

```c
void __init security_add_hooks(struct security_hook_list *hooks, int count,
                               const struct lsm_id *lsmid);
```

Fills the first unused `lsm_static_call` slot for each hook via
`__static_call_update()` and `static_branch_enable()`.

### Initialization Order

`security/security.c:502-544`:

1. **`early_security_init()`**: Before slab available. Runs early LSMs (e.g., lockdown).
2. **`security_init()`** → `ordered_lsm_init()`:
   - Parses `CONFIG_LSM` or `lsm=` boot parameter
   - `LSM_ORDER_FIRST` (capabilities) always first
   - `LSM_ORDER_LAST` (integrity) always last
   - Accumulates blob sizes, creates kmem caches
   - Calls each LSM's `init()` function

### Blob Allocation

Single contiguous blob per kernel object, shared by all LSMs:
- `cred->security` — `lsm_cred_alloc()` at `security.c:699`
- `file->f_security` — from `lsm_file_cache`
- `inode->i_security` — from `lsm_inode_cache`
- `sock->sk_security`, `ipc->security`, `key->security` — similarly

Each LSM's accessor adds its offset to the base pointer:
```c
// Example from SELinux:
static inline struct task_security_struct *selinux_cred(const struct cred *cred)
{
    return cred->security + selinux_blob_sizes.lbs_cred;
}
```

### BPF LSM

`security/bpf/hooks.c`

Registers a stub hook for every hook point in `lsm_hook_defs.h`. These stubs
are BPF trampolines — `BPF_PROG_TYPE_LSM` programs attach to these attachment
points. Enables runtime-programmable security policy without kernel
recompilation.

---

## Key Hook Categories

All signatures in `include/linux/lsm_hook_defs.h`.

### Task Hooks

Protect task lifecycle, signals, scheduling, process control:
```c
task_alloc, task_free, task_kill, task_setnice,
task_setscheduler, task_setrlimit, task_prctl,
task_movememory, ptrace_access_check, ptrace_traceme,
userns_create
```

### File Hooks

Protect open, read/write, mmap, locking, fd operations:
```c
file_permission, file_alloc_security, file_free_security,
file_open, file_ioctl, mmap_file, mmap_addr, file_mprotect,
file_lock, file_fcntl, file_truncate, file_receive
```

`security_file_permission()` is called on every read/write — a hot path.

### Inode Hooks

Protect filesystem namespace operations:
```c
inode_create, inode_link, inode_unlink, inode_symlink,
inode_mkdir, inode_rmdir, inode_rename, inode_permission,
inode_setattr, inode_getattr, inode_setxattr, inode_getxattr,
inode_removexattr, inode_init_security, inode_alloc_security,
inode_free_security
```

### Socket Hooks

Protect network operations:
```c
socket_create, socket_bind, socket_connect, socket_listen,
socket_accept, socket_sendmsg, socket_recvmsg,
socket_sock_rcv_skb, sk_alloc_security, sk_free_security,
unix_stream_connect
```

### Credential Hooks

```c
cred_alloc_blank, cred_free, cred_prepare, cred_transfer,
task_fix_setuid, task_fix_setgid, task_fix_setgroups
```

### Program Execution Hooks (bprm)

Called during `execve()`:
```c
bprm_creds_for_exec      // set up new credentials
bprm_creds_from_file     // file-capability elevation
bprm_check_security      // final permission check
bprm_committing_creds    // about to commit
bprm_committed_creds     // post-exec notification
```

### IPC Hooks

```c
ipc_permission, msg_queue_msgsnd, msg_queue_msgrcv,
shm_shmat, sem_semop
```

### Key Hooks

```c
key_alloc, key_permission, key_getsecurity
```

---

## Capabilities (POSIX)

### Capability Sets in `struct cred`

`include/linux/cred.h:111`

```c
kernel_cap_t cap_inheritable;  // caps children can inherit
kernel_cap_t cap_permitted;    // superset of cap_effective
kernel_cap_t cap_effective;    // caps actually checked
kernel_cap_t cap_bset;         // bounding set (max acquirable via exec)
kernel_cap_t cap_ambient;      // auto-inherited across exec
```

`kernel_cap_t` is `typedef struct { u64 val; } kernel_cap_t;` — 64-bit bitmask.

**Ambient invariant**: `cap_ambient ⊆ (cap_permitted ∩ cap_inheritable)`

### Checking

```c
capable(int cap)                                    // current in init_user_ns
ns_capable(struct user_namespace *ns, int cap)       // current in specific ns
has_capability(struct task_struct *t, int cap)        // another task
has_ns_capability(struct task_struct *t, struct user_namespace *ns, int cap)
```

All route through `security_capable()` → `cap_capable()` in `security/commoncap.c`, which walks the user namespace hierarchy.

### Important Capabilities

```c
CAP_DAC_OVERRIDE     1   // bypass file DAC
CAP_KILL             5   // signal any process
CAP_SETUID           7   // setuid
CAP_NET_BIND_SERVICE 10  // bind ports < 1024
CAP_NET_ADMIN        12  // network administration
CAP_NET_RAW          13  // RAW/PACKET sockets
CAP_SYS_MODULE       16  // load kernel modules
CAP_SYS_RAWIO        17  // raw I/O
CAP_SYS_PTRACE       19  // ptrace
CAP_SYS_ADMIN        21  // wide-ranging admin
CAP_SYS_NICE         23  // scheduling priority
CAP_MAC_OVERRIDE     32  // override MAC
CAP_MAC_ADMIN        33  // MAC administration
```

### File Capabilities

Stored in `security.capability` xattr as `struct vfs_ns_cap_data`. Read during
`execve()` via `cap_bprm_creds_from_file()` in `security/commoncap.c`. File's
permitted/inheritable sets are intersected with bounding set and process's
inheritable set to derive the new permitted set.

### The commoncap Module

`security/commoncap.c` — always loaded at `LSM_ORDER_FIRST`. Implements default
POSIX capability semantics for `capable`, `capget`, `capset`,
`ptrace_access_check`, `bprm_creds_from_file`, etc.

---

## SELinux

`security/selinux/`

### Architecture

Three access control models:
- **Type Enforcement (TE)**: Primary mechanism. `allow <source_type> <target_type>:<class> { permissions }`
- **Role-Based Access Control (RBAC)**: Users → roles → types
- **Multi-Level Security (MLS)**: Sensitivity levels and categories

Security context: `user:role:type:s0:c0,c1`

### Per-Object Security Structures

`security/selinux/include/objsec.h`

**`struct task_security_struct`** (in `cred->security`):
```c
struct task_security_struct {
    u32 osid;           // SID prior to last execve
    u32 sid;            // current SID
    u32 exec_sid;       // SID for next exec transition
    u32 create_sid;     // fscreate SID
    u32 keycreate_sid;  // new key SID
    u32 sockcreate_sid; // new socket SID
};
```

**`struct inode_security_struct`** (in `inode->i_security`):
```c
struct inode_security_struct {
    struct inode *inode;
    u32 task_sid;    // creating task SID
    u32 sid;         // object SID
    u16 sclass;      // SECCLASS_FILE, SECCLASS_DIR, etc.
    unsigned char initialized;
    spinlock_t lock;
};
```

### Access Vector Cache (AVC)

`security/selinux/avc.c`

512-slot hash table (`AVC_CACHE_SLOTS`) keyed by `(ssid, tsid, tclass)`:

```c
struct avc_entry {
    u32 ssid, tsid;
    u16 tclass;
    struct av_decision avd;  // allowed, denied, audit bits
};
```

**`avc_has_perm()`**: Main check function. Hash lookup → on miss, calls
`security_compute_av()` in the policy server → populates cache.

### Key Hooks

- `selinux_inode_permission()` — called for every inode access, checks AVC
- `selinux_file_permission()` — read/write permission checks
- `selinux_bprm_creds_for_exec()` — domain transitions on exec
- `selinux_inode_setxattr()` — relabel checks (RELABELFROM, RELABELTO, ASSOCIATE)

---

## AppArmor

`security/apparmor/`

### Architecture

Profile-based MAC using **path-based** access control (not inode-based like
SELinux). Profiles contain compiled DFA patterns matched against resolved file
paths.

### Key Structures

**`struct aa_profile`** (`include/policy.h:224`):
```c
struct aa_profile {
    struct aa_policy base;
    struct aa_profile __rcu *parent;
    struct aa_ns *ns;
    long mode;                 // APPARMOR_ENFORCE/COMPLAIN/KILL/UNCONFINED
    struct aa_attachment attach;
    struct list_head rules;    // list of aa_ruleset
    struct aa_label label;     // identity in cred->security
};
```

**`struct aa_label`** (`include/label.h:123`):
```c
struct aa_label {
    struct kref count;
    struct rb_node node;        // global label rbtree
    u32 secid;
    int size;                   // number of profiles
    struct aa_profile *vec[];   // sorted profile vector
};
```

**`struct aa_policydb`** (`include/policy.h:84`):
```c
struct aa_policydb {
    struct aa_dfa *dfa;           // compiled finite automaton
    struct { struct aa_perms *perms; u32 size; };
    struct aa_str_table trans;    // transition strings
};
```

### Path Resolution

At hook time, AppArmor computes the full path (`aa_path_name()`) and runs it
through the profile's DFA. Implications:
- Hard links can bypass rules if the alternative path isn't covered
- Bind mounts can expose files under different paths with different permissions
- File moves change permissions dynamically

### Profile Loading

Via `/sys/kernel/security/apparmor/.load`. `apparmor_parser` compiles profile
text into binary DFAs. Domain transitions on exec defined by `ix` (inline),
`px` (profile), `cx` (child), `ux` (unconfined) rules.

---

## Seccomp

`kernel/seccomp.c`

### Modes

```c
SECCOMP_MODE_DISABLED  0  // no seccomp
SECCOMP_MODE_STRICT    1  // only read, write, exit, sigreturn
SECCOMP_MODE_FILTER    2  // BPF filter programs
```

### Task State

```c
struct seccomp {                     // in task_struct
    int mode;
    atomic_t filter_count;
    struct seccomp_filter *filter;   // most recent (singly-linked list)
};
```

### `struct seccomp_filter`

`kernel/seccomp.c:226`

```c
struct seccomp_filter {
    refcount_t refs, users;
    bool log;
    struct action_cache cache;       // per-syscall always-allow bitmap
    struct seccomp_filter *prev;     // parent filter
    struct bpf_prog *prog;           // BPF program
    struct notification *notif;      // for SECCOMP_RET_USER_NOTIF
};
```

Filters form a linked list via `->prev`. Inherited on `fork()`, new filters
prepended. All filters always evaluated.

### Filter Input

```c
struct seccomp_data {
    int nr;                   // syscall number
    __u32 arch;               // AUDIT_ARCH_*
    __u64 instruction_pointer;
    __u64 args[6];            // syscall arguments
};
```

### Return Values (ordered by restrictiveness)

```c
SECCOMP_RET_KILL_PROCESS  0x80000000  // kill entire process
SECCOMP_RET_KILL_THREAD   0x00000000  // kill thread
SECCOMP_RET_TRAP          0x00030000  // deliver SIGSYS
SECCOMP_RET_ERRNO         0x00050000  // return errno
SECCOMP_RET_USER_NOTIF    0x7fc00000  // notify userspace
SECCOMP_RET_TRACE         0x7ff00000  // pass to tracer
SECCOMP_RET_LOG           0x7ffc0000  // allow + log
SECCOMP_RET_ALLOW         0x7fff0000  // allow
```

`min()` over all filter results gives the most restrictive outcome.

### `seccomp_run_filters()`

`kernel/seccomp.c:406` — called from `__secure_computing()` at every syscall entry:

1. Fast-path: check `action_cache` bitmap for always-allow
2. Walk filter chain (`f = f->prev`), run each BPF program
3. Keep lowest (most restrictive) return value
4. Apply the action

### Installation

```c
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &fprog);
syscall(SYS_seccomp, SECCOMP_SET_MODE_FILTER, flags, &fprog);
```

`SECCOMP_FILTER_FLAG_TSYNC` propagates to all threads.

---

## Credentials

### `struct cred` Lifecycle

`include/linux/cred.h:154`

```c
struct cred *prepare_creds(void);          // copy current->cred
int commit_creds(struct cred *);           // install atomically via RCU
void abort_creds(struct cred *);           // discard prepared creds
const struct cred *override_creds(...);    // temporary override
void revert_creds(const struct cred *);    // undo override
struct cred *prepare_kernel_cred(...);     // kernel credentials
```

**`prepare_creds()`**: Allocates new cred, copies all fields, increments
references, allocates LSM blob via `security_prepare_creds()`.

**`commit_creds()`**: Atomically installs via `rcu_assign_pointer()` to both
`task->real_cred` and `task->cred`. Puts old credentials.

### RCU Protection

```c
// Read another task's credentials:
rcu_read_lock();
cred = __task_cred(task);   // rcu_dereference(task->real_cred)
rcu_read_unlock();

// Read current's credentials (no RCU needed):
current_cred()              // safe because only current modifies its own creds
```

### Credential Flow

- **`fork()`**: `copy_creds()` creates a copy; `CLONE_THREAD` shares the pointer
- **`execve()`**: `prepare_exec_creds()` copies, `bprm_creds_*` hooks modify (setuid, file caps, SELinux domain transition), `commit_creds()` installs
- **`setuid()`**: `prepare_creds()`, modify uid, `task_fix_setuid` hook, `commit_creds()`

---

## User Namespaces as Security Boundary

### `struct user_namespace`

`include/linux/user_namespace.h:74`

```c
struct user_namespace {
    struct uid_gid_map uid_map, gid_map;
    struct user_namespace *parent;
    int level;                   // nesting depth (init_user_ns = 0)
    kuid_t owner;                // uid of creator in parent ns
    struct ns_common ns;
};
```

### Capability Mapping

`cap_capable()` in `security/commoncap.c` walks the namespace hierarchy:
1. If target ns == `cred->user_ns`, check `cap_effective` directly
2. If `cred->user_ns` is parent and `euid` matches target ns `owner`, grant all caps
3. Walk up via `ns->parent`

A process with `CAP_SYS_ADMIN` in a user namespace has privilege only over resources within that namespace — no privilege over the host.

### ID Mapping

`make_kuid(ns, uid)` → kernel-global `kuid_t`. `from_kuid(ns, kuid)` → namespace-local UID. All internal kernel operations use `kuid_t`/`kgid_t`.

---

## Landlock

`security/landlock/`

Self-imposed sandboxing allowing unprivileged processes to restrict their own access. Policies are rulesets built programmatically (not by administrators).

### Key Structures

**`struct landlock_ruleset`** (`ruleset.h:174`):
```c
struct landlock_ruleset {
    struct rb_root root_inode;      // inode-based rules rbtree
    struct rb_root root_net_port;   // network port rules rbtree
    struct landlock_hierarchy *hierarchy;
    u32 num_rules, num_layers;
    struct access_masks access_masks[];  // per stacked layer
};
```

**`struct landlock_rule`** (`ruleset.h:128`):
```c
struct landlock_rule {
    struct rb_node node;
    union landlock_key key;     // inode object or port number
    u32 num_layers;
    struct landlock_layer layers[];
};
```

### Access Rights

Filesystem: `LANDLOCK_ACCESS_FS_READ_FILE`, `_WRITE_FILE`, `_EXECUTE`, `_MAKE_REG`, `_MAKE_DIR`, `_REFER` (cross-directory hardlink/rename), etc.

Network: port-based restrictions.

### Syscall Interface

```c
landlock_create_ruleset(attr, size, flags);   // create ruleset fd
landlock_add_rule(fd, rule_type, rule_attr, flags);  // add inode or port rule
landlock_restrict_self(fd, flags);            // install on calling thread
```

Requires `no_new_privs` or `CAP_SYS_ADMIN`. Multiple layers stack — most restrictive wins.

### Rule Matching

At enforcement time:
1. Get domain from `cred->security`
2. Compute required access masks
3. `landlock_find_rule()` — rbtree lookup by inode
4. `landlock_unmask_layers()` — walk rule's layer stack
5. If any layer still has ungranted bits → access denied

---

## Integrity Subsystem

`security/integrity/`

### IMA (Integrity Measurement Architecture)

`security/integrity/ima/`

**Purpose**: Measures file content hashes and extends them into TPM PCR
(typically PCR 10), creating an immutable measurement log. Supports appraisal
(refusing files whose hash doesn't match stored signature).

**Measurement flow**:
1. On file open/exec, check IMA policy for measurement requirement
2. Compute file hash via `integrity_kernel_read()`
3. Create `struct ima_template_entry` with hash and filename
4. Extend TPM PCR via `tpm_pcr_extend()`
5. Add to `ima_measurements` list

**Action flags** (`ima.h:131`):
```c
IMA_MEASURE     0x01  // must measure
IMA_MEASURED    0x02  // has been measured
IMA_APPRAISE   0x04  // must appraise
IMA_APPRAISED  0x08  // has been appraised
IMA_AUDIT      0x40  // must audit
IMA_HASH       0x100 // must hash
```

**Appraisal**: Checks `security.ima` xattr against computed hash. Xattr
contains hash or digital signature (`IMA_XATTR_DIGEST`, `EVM_IMA_XATTR_DIGSIG`,
`IMA_VERITY_DIGSIG`).

### EVM (Extended Verification Module)

`security/integrity/evm/`

**Purpose**: Protects security xattrs from offline tampering by computing HMAC
or digital signature over them.

**Protected xattrs**:
- `security.selinux`
- `security.apparmor`
- `security.ima`
- `security.capability`
- `security.smack`

The `security.evm` xattr stores an HMAC of all protected xattrs plus inode
metadata. On read, EVM verifies this HMAC to detect tampering.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `security/security.c` | LSM core: init, `security_add_hooks()`, blob alloc, all `security_*()` wrappers |
| `include/linux/lsm_hooks.h` | `security_hook_list`, `lsm_info`, `lsm_blob_sizes`, `DEFINE_LSM` |
| `include/linux/lsm_hook_defs.h` | All `LSM_HOOK()` declarations — canonical hook list |
| `include/linux/security.h` | Public `security_*()` prototypes |
| `include/linux/cred.h` | `struct cred`, `prepare_creds()`, `commit_creds()` |
| `security/commoncap.c` | Capabilities module, `cap_capable()` |
| `include/uapi/linux/capability.h` | `CAP_*` constants |
| `security/selinux/hooks.c` | SELinux hook implementations (~7500 lines) |
| `security/selinux/include/objsec.h` | SELinux per-object security structs |
| `security/selinux/avc.c` | Access Vector Cache |
| `security/apparmor/include/policy.h` | `aa_profile`, `aa_ruleset`, `aa_policydb` |
| `security/apparmor/include/label.h` | `aa_label` |
| `security/apparmor/lsm.c` | AppArmor hook implementations |
| `kernel/seccomp.c` | `seccomp_filter`, `seccomp_run_filters()` |
| `include/uapi/linux/seccomp.h` | `SECCOMP_RET_*`, `seccomp_data` |
| `security/landlock/ruleset.h` | `landlock_ruleset`, `landlock_rule` |
| `security/landlock/syscalls.c` | Landlock syscall implementations |
| `security/bpf/hooks.c` | BPF LSM registration and stubs |
| `security/integrity/ima/ima_main.c` | IMA hook implementations |
| `security/integrity/evm/evm_main.c` | EVM hook implementations |
| `security/integrity/integrity.h` | Shared IMA/EVM structures |
| `include/linux/user_namespace.h` | `user_namespace`, `uid_gid_map` |

---

## Call Flow Example: File Permission Check

```
vfs_read()
  └─ rw_verify_area()
       └─ security_file_permission(file, MAY_READ)
            ├─ [static call slot 0] cap_file_permission()  (commoncap)
            ├─ [static call slot 1] selinux_file_permission()
            │    └─ file_has_perm() → inode_has_perm()
            │         └─ avc_has_perm_noaudit(current_sid, isec->sid, FILE__READ)
            │              └─ hash lookup in 512-slot AVC cache
            │                   ├─ hit: check avd.allowed & perms
            │                   └─ miss: security_compute_av() → populate cache
            ├─ [static call slot 2] apparmor_file_permission()
            │    └─ aa_path_name() → DFA match → perms lookup
            └─ First non-zero return → deny; all zero → allow
```
