# Linux Kernel Permissions and Credentials

## Table of Contents
1. [Overview](#1-overview)
2. [Task Credentials (struct cred)](#2-task-credentials-struct-cred)
3. [UID/GID Types and Namespace Mapping](#3-uidgid-types-and-namespace-mapping)
4. [Filesystem Permission Bits](#4-filesystem-permission-bits)
5. [DAC Permission Checking Flow](#5-dac-permission-checking-flow)
6. [POSIX ACLs](#6-posix-acls)
7. [Supplementary Groups](#7-supplementary-groups)
8. [Capability Override Rules](#8-capability-override-rules)
9. [setuid/setgid Syscalls](#9-setuidsetgid-syscalls)
10. [Setuid/Setgid Binaries and execve](#10-setuidsetgid-binaries-and-execve)
11. [Capability Transformation During exec](#11-capability-transformation-during-exec)
12. [Root (UID 0) and Capabilities](#12-root-uid-0-and-capabilities)
13. [no_new_privs](#13-no_new_privs)
14. [Dumpability and /proc Restrictions](#14-dumpability-and-proc-restrictions)
15. [umask](#15-umask)
16. [chmod and chown](#16-chmod-and-chown)
17. [access() Syscall](#17-access-syscall)
18. [Credential Override Mechanism](#18-credential-override-mechanism)
19. [Securebits](#19-securebits)
20. [Key Source Files](#20-key-source-files)

---

## 1. Overview

The Linux kernel implements a multi-layered permission model governing which
tasks can access which resources. The core layers are:

```
┌───────────────────────────────────────────────────────────────────┐
│                     Permission Check Layers                       │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Syscall Entry                                                    │
│       │                                                           │
│       ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │  1. DAC (Discretionary Access Control)                   │      │
│  │     owner/group/other mode bits + POSIX ACLs             │      │
│  │     Checked against: fsuid, fsgid, supplementary groups  │      │
│  └────────────────────┬────────────────────────────────────┘      │
│                       │ denied?                                   │
│                       ▼                                           │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │  2. Capability Override                                  │      │
│  │     CAP_DAC_OVERRIDE, CAP_DAC_READ_SEARCH, CAP_FOWNER   │      │
│  │     Checked against: cap_effective in struct cred        │      │
│  └────────────────────┬────────────────────────────────────┘      │
│                       │                                           │
│                       ▼                                           │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │  3. LSM (Mandatory Access Control)                       │      │
│  │     SELinux, AppArmor, Landlock, BPF LSM                 │      │
│  │     Always checked regardless of DAC outcome             │      │
│  └─────────────────────────────────────────────────────────┘      │
│                                                                   │
│  All three layers must grant access. DAC + capability override    │
│  is checked first; then LSM hooks are called independently.       │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

Every task carries a `struct cred` containing four UID/GID pairs, five
capability sets, and LSM security blobs. Filesystem objects carry an owner UID,
owner GID, a 12-bit mode field, and optional POSIX ACLs. The VFS permission
checking code in `fs/namei.c` ties these together.

---

## 2. Task Credentials (struct cred)

### Definition

`include/linux/cred.h:113`

```c
struct cred {
    atomic_long_t   usage;
    kuid_t          uid;            /* real UID */
    kgid_t          gid;            /* real GID */
    kuid_t          suid;           /* saved UID */
    kgid_t          sgid;           /* saved GID */
    kuid_t          euid;           /* effective UID */
    kgid_t          egid;           /* effective GID */
    kuid_t          fsuid;          /* UID for VFS ops */
    kgid_t          fsgid;          /* GID for VFS ops */
    unsigned        securebits;     /* SUID-less security management */
    kernel_cap_t    cap_inheritable;
    kernel_cap_t    cap_permitted;
    kernel_cap_t    cap_effective;
    kernel_cap_t    cap_bset;       /* capability bounding set */
    kernel_cap_t    cap_ambient;
    void            *security;      /* LSM security blob */
    struct user_struct *user;
    struct user_namespace *user_ns;
    struct ucounts  *ucounts;
    struct group_info *group_info;  /* supplementary groups */
    union {
        int non_rcu;
        struct rcu_head rcu;
    };
} __randomize_layout;
```

### The Four UID/GID Pairs

```
┌─────────────────────────────────────────────────────────────────┐
│                    UID/GID Pairs in struct cred                   │
├──────────┬──────────────────────────────────────────────────────┤
│ uid/gid  │ Real ID. Identifies who the process actually belongs │
│          │ to. Used for signal delivery permission checks and   │
│          │ resource accounting. Set at login, inherited across  │
│          │ fork(). Only changeable with CAP_SETUID/CAP_SETGID. │
├──────────┼──────────────────────────────────────────────────────┤
│ euid/    │ Effective ID. Used for most privilege checks:        │
│ egid     │ capability evaluation, file creation ownership,     │
│          │ IPC permissions. Set by setuid binaries on exec.     │
├──────────┼──────────────────────────────────────────────────────┤
│ suid/    │ Saved ID. Preserves the effective ID across an exec │
│ sgid     │ so a setuid program can drop privileges temporarily │
│          │ (set euid=uid) then regain them (set euid=suid).    │
├──────────┼──────────────────────────────────────────────────────┤
│ fsuid/   │ Filesystem ID. Used specifically by VFS permission  │
│ fsgid    │ checks (acl_permission_check). Normally shadows     │
│          │ euid/egid — set automatically when euid/egid change.│
│          │ Can be set independently via setfsuid()/setfsgid().  │
│          │ Exists so NFS servers can access files as a client   │
│          │ user without granting that user signal privileges.   │
└──────────┴──────────────────────────────────────────────────────┘
```

### Two Credential Pointers Per Task

Every `task_struct` has two credential pointers:

- `task->real_cred` — **objective context**: used when *other* tasks act upon
  this task (e.g., signal delivery permission checks)
- `task->cred` — **subjective context**: used when this task acts upon other
  objects (e.g., file access)

They normally point to the same `struct cred`. They diverge only when
`override_creds()` is used (see [Credential Override](#18-credential-override-mechanism)).

### Credential Lifecycle

`kernel/cred.c`

```c
struct cred *prepare_creds(void);           // copy current->cred
int commit_creds(struct cred *new);         // install atomically via RCU
void abort_creds(struct cred *new);         // discard prepared creds
struct cred *prepare_exec_creds(void);      // copy for execve
struct cred *prepare_kernel_cred(...);      // creds for kernel service
```

**`prepare_creds()`** (`cred.c:179`): Allocates a new `struct cred` from
`cred_jar` slab cache, copies all fields from `current->cred`, increments
refcounts on sub-objects (`group_info`, `user`, `user_ns`), and calls
`security_prepare_creds()` for LSM blob allocation.

**`commit_creds()`** (`cred.c:368`): Installs new credentials atomically:

```c
int commit_creds(struct cred *new)
{
    struct task_struct *task = current;
    const struct cred *old = task->real_cred;

    get_cred(new);  /* extra ref for subjective pointer */

    /* If euid/egid/fsuid/fsgid changed or caps grew, downgrade dumpability */
    if (!uid_eq(old->euid, new->euid) || !gid_eq(old->egid, new->egid) ||
        !uid_eq(old->fsuid, new->fsuid) || !gid_eq(old->fsgid, new->fsgid) ||
        !cred_cap_issubset(old, new)) {
        if (task->mm)
            set_dumpable(task->mm, suid_dumpable);
        task->pdeath_signal = 0;    /* clear parent-death signal */
    }

    rcu_assign_pointer(task->real_cred, new);
    rcu_assign_pointer(task->cred, new);
    put_cred_many(old, 2);
    return 0;
}
```

### RCU Protection

```c
/* Read another task's credentials safely: */
rcu_read_lock();
cred = __task_cred(task);   /* rcu_dereference(task->real_cred) */
/* use cred fields... */
rcu_read_unlock();

/* Read current's credentials (no RCU needed — only current modifies its own): */
current_cred()              /* current->cred */
current_uid()               /* current_cred()->uid */
current_euid()              /* current_cred()->euid */
current_fsuid()             /* current_cred()->fsuid */
```

### Credential Flow Through Task Lifecycle

- **`fork()`**: `copy_creds()` creates a copy; `CLONE_THREAD` shares the same
  cred pointer (threads share credentials)
- **`execve()`**: `prepare_exec_creds()` copies, `bprm_fill_uid()` applies
  setuid, `cap_bprm_creds_from_file()` applies capabilities,
  `commit_creds()` installs
- **`setuid()`**: `prepare_creds()` → modify UID fields →
  `security_task_fix_setuid()` → `commit_creds()`

---

## 3. UID/GID Types and Namespace Mapping

### Type-Safe Wrappers

`include/linux/uidgid_types.h:7`

```c
typedef struct { uid_t val; } kuid_t;
typedef struct { gid_t val; } kgid_t;
```

Wrapping in structs prevents accidental mixing of raw `uid_t` with `kuid_t` at
compile time. All kernel-internal operations use `kuid_t`/`kgid_t`.

### Key Constants

`include/linux/uidgid.h:47`

```c
#define GLOBAL_ROOT_UID  KUIDT_INIT(0)
#define GLOBAL_ROOT_GID  KGIDT_INIT(0)
#define INVALID_UID      KUIDT_INIT(-1)
#define INVALID_GID      KGIDT_INIT(-1)
```

### Namespace Mapping Functions

`include/linux/uidgid.h:115`

```c
kuid_t make_kuid(struct user_namespace *from, uid_t uid);   /* userspace → kernel */
uid_t from_kuid(struct user_namespace *to, kuid_t uid);     /* kernel → userspace */
uid_t from_kuid_munged(struct user_namespace *to, kuid_t uid); /* returns overflowuid if unmapped */
bool uid_eq(kuid_t left, kuid_t right);
bool uid_valid(kuid_t uid);
```

Within user namespaces, a UID in one namespace maps to a different kernel-global
`kuid_t`. The `uid_map`/`gid_map` in `struct user_namespace` defines these
mappings. All permission checks use `kuid_t`/`kgid_t`, ensuring consistent
comparison across namespace boundaries.

### VFS UID/GID Types

For idmapped mounts, the VFS uses `vfsuid_t`/`vfsgid_t` wrappers. These
account for mount-level ID mapping on top of namespace mapping:

```c
vfsuid_t vfsuid = i_uid_into_vfsuid(idmap, inode);
bool match = vfsuid_eq_kuid(vfsuid, current_fsuid());
```

---

## 4. Filesystem Permission Bits

### Mode Bits

`include/uapi/linux/stat.h:9`

```c
/* File type (upper 4 bits of i_mode) */
#define S_IFMT   00170000
#define S_IFREG  0100000     /* regular file */
#define S_IFDIR  0040000     /* directory */
#define S_IFLNK  0120000     /* symbolic link */
#define S_IFCHR  0020000     /* character device */
#define S_IFBLK  0060000     /* block device */
#define S_IFIFO  0010000     /* FIFO/pipe */
#define S_IFSOCK 0140000     /* socket */

/* Special permission bits */
#define S_ISUID  0004000     /* set-user-ID on execution */
#define S_ISGID  0002000     /* set-group-ID on execution / mandatory locking */
#define S_ISVTX  0001000     /* sticky bit (restricted deletion in dirs) */

/* Owner permissions */
#define S_IRWXU  00700
#define S_IRUSR  00400       /* owner read */
#define S_IWUSR  00200       /* owner write */
#define S_IXUSR  00100       /* owner execute */

/* Group permissions */
#define S_IRWXG  00070
#define S_IRGRP  00040
#define S_IWGRP  00020
#define S_IXGRP  00010

/* Other permissions */
#define S_IRWXO  00007
#define S_IROTH  00004
#define S_IWOTH  00002
#define S_IXOTH  00001
```

Additional kernel-internal constants:
- `S_IRWXUGO` = `S_IRWXU | S_IRWXG | S_IRWXO` = 0777
- `S_IALLUGO` = `S_ISUID | S_ISGID | S_ISVTX | S_IRWXUGO` = 07777
- `S_IXUGO` = `S_IXUSR | S_IXGRP | S_IXOTH` = 0111

### Permission-Relevant Inode Fields

`include/linux/fs.h:766`

```c
struct inode {
    umode_t           i_mode;        /* file type + 12-bit permission field */
    kuid_t            i_uid;         /* owner UID */
    kgid_t            i_gid;         /* owner GID */
    unsigned int      i_flags;       /* includes S_IMMUTABLE, S_APPEND */
    struct posix_acl  *i_acl;        /* access ACL (cached) */
    struct posix_acl  *i_default_acl; /* default ACL for directories */
    /* ... */
};
```

### Special Bit Semantics

```
┌───────────┬──────────────────────────────────────────────────────┐
│ S_ISUID   │ On exec: set euid to file owner (see §10)           │
│           │ On regular files only. Requires exec permission.     │
├───────────┼──────────────────────────────────────────────────────┤
│ S_ISGID   │ On exec (with S_IXGRP): set egid to file group      │
│           │ On directories: new files inherit parent's group     │
│           │ On files without S_IXGRP: mandatory locking (legacy) │
├───────────┼──────────────────────────────────────────────────────┤
│ S_ISVTX   │ Sticky bit. On directories: only file owner, dir    │
│ (sticky)  │ owner, or CAP_FOWNER can delete/rename files within │
└───────────┴──────────────────────────────────────────────────────┘
```

---

## 5. DAC Permission Checking Flow

### Call Chain

```
vfs_read() / vfs_open() / path_lookup()
  └─ inode_permission(idmap, inode, mask)          fs/namei.c:623
       ├─ sb_permission()                          superblock-level checks (readonly)
       ├─ IS_IMMUTABLE check                       immutable files reject MAY_WRITE
       ├─ do_inode_permission()                    fs/namei.c:573
       │    ├─ inode->i_op->permission()           filesystem-specific (rare)
       │    └─ generic_permission()                fs/namei.c:516 (common path)
       │         ├─ acl_permission_check()         fs/namei.c:433 (core DAC)
       │         │    ├─ owner match → check owner bits
       │         │    ├─ POSIX ACL → posix_acl_permission()
       │         │    ├─ group match → check group bits
       │         │    └─ other → check other bits
       │         └─ capability override             if DAC denied
       │              ├─ CAP_DAC_OVERRIDE
       │              └─ CAP_DAC_READ_SEARCH
       ├─ devcgroup_inode_permission()             device cgroup check
       └─ security_inode_permission()              LSM hook (SELinux, AppArmor, etc.)
```

### `acl_permission_check()` — Core DAC Logic

`fs/namei.c:433`

```c
static int acl_permission_check(struct mnt_idmap *idmap,
                                struct inode *inode, int mask)
{
    unsigned int mode = inode->i_mode;
    vfsuid_t vfsuid;

    /* Fast path: if "other" bits grant everything and no ACLs, allow */
    if (!((mask & 7) * 0111 & ~mode)) {
        if (no_acl_inode(inode))
            return 0;
        if (!IS_POSIXACL(inode))
            return 0;
    }

    /* Owner check: ACLs don't apply to the owner */
    vfsuid = i_uid_into_vfsuid(idmap, inode);
    if (likely(vfsuid_eq_kuid(vfsuid, current_fsuid()))) {
        mask &= 7;
        mode >>= 6;
        return (mask & ~mode) ? -EACCES : 0;
    }

    /* POSIX ACL check (if present) */
    if (IS_POSIXACL(inode) && (mode & S_IRWXG)) {
        int error = check_acl(idmap, inode, mask);
        if (error != -EAGAIN)
            return error;
    }

    mask &= 7;

    /* Group check: if group bits differ from other, check group membership */
    if (mask & (mode ^ (mode >> 3))) {
        vfsgid_t vfsgid = i_gid_into_vfsgid(idmap, inode);
        if (vfsgid_in_group_p(vfsgid))
            mode >>= 3;
    }

    return (mask & ~mode) ? -EACCES : 0;
}
```

**Evaluation order**: owner → POSIX ACL (if present) → group → other.

The `(mask & 7) * 0111` expression replicates the requested permission bits
across all three positions (owner/group/other). If *all* positions grant the
requested access, the fast path returns immediately.

### `generic_permission()` — DAC + Capability Override

`fs/namei.c:516`

```c
int generic_permission(struct mnt_idmap *idmap,
                       struct inode *inode, int mask)
{
    int ret;

    ret = acl_permission_check(idmap, inode, mask);
    if (ret != -EACCES)
        return ret;

    /* DAC denied — try capability overrides */

    if (S_ISDIR(inode->i_mode)) {
        /* Directories: CAP_DAC_READ_SEARCH overrides read+search */
        if (!(mask & MAY_WRITE))
            if (capable_wrt_inode_uidgid(idmap, inode, CAP_DAC_READ_SEARCH))
                return 0;
        /* CAP_DAC_OVERRIDE overrides everything on directories */
        if (capable_wrt_inode_uidgid(idmap, inode, CAP_DAC_OVERRIDE))
            return 0;
        return -EACCES;
    }

    /* Files: CAP_DAC_READ_SEARCH overrides read-only denial */
    if (mask == MAY_READ)
        if (capable_wrt_inode_uidgid(idmap, inode, CAP_DAC_READ_SEARCH))
            return 0;

    /* CAP_DAC_OVERRIDE overrides read+write; overrides exec only if
       at least one execute bit is set on the file */
    if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
        if (capable_wrt_inode_uidgid(idmap, inode, CAP_DAC_OVERRIDE))
            return 0;

    return -EACCES;
}
```

### MAY_* Flags

`include/linux/fs.h`

```c
#define MAY_EXEC        0x00000001
#define MAY_WRITE       0x00000002
#define MAY_READ        0x00000004
#define MAY_APPEND      0x00000008
#define MAY_ACCESS      0x00000010   /* used by access() syscall */
#define MAY_OPEN        0x00000020
#define MAY_CHDIR       0x00000040
#define MAY_NOT_BLOCK   0x00000080   /* RCU path walk */
```

---

## 6. POSIX ACLs

### Data Structures

`include/linux/posix_acl.h:20`

```c
struct posix_acl_entry {
    short           e_tag;      /* ACL_USER_OBJ, ACL_USER, ACL_GROUP_OBJ, etc. */
    unsigned short  e_perm;     /* ACL_READ | ACL_WRITE | ACL_EXECUTE */
    union {
        kuid_t      e_uid;      /* for ACL_USER entries */
        kgid_t      e_gid;      /* for ACL_GROUP entries */
    };
};

struct posix_acl {
    refcount_t              a_refcount;
    unsigned int            a_count;
    struct rcu_head         a_rcu;
    struct posix_acl_entry  a_entries[];
};
```

### ACL Tag Types

`include/uapi/linux/posix_acl.h:28`

```c
#define ACL_USER_OBJ    0x01    /* file owner permissions */
#define ACL_USER        0x02    /* named user permissions */
#define ACL_GROUP_OBJ   0x04    /* file group permissions */
#define ACL_GROUP       0x08    /* named group permissions */
#define ACL_MASK        0x10    /* max perms for ACL_USER, ACL_GROUP, ACL_GROUP_OBJ */
#define ACL_OTHER       0x20    /* everyone else */
```

### Checking Algorithm

`fs/posix_acl.c:373` — `posix_acl_permission()`

The algorithm walks ACL entries in order:

```
1. ACL_USER_OBJ:  if fsuid == file owner   → check permission (no mask)
2. ACL_USER:      if fsuid == entry uid     → check permission & ACL_MASK
3. ACL_GROUP_OBJ: if fsgid == file group    → mark found, accumulate perms
4. ACL_GROUP:     if in supplementary group → mark found, accumulate perms
5. ACL_MASK:      record mask value
6. ACL_OTHER:     if no group matched       → check permission (no mask)
                  if group matched          → check accumulated perms & mask
```

Key subtlety: named user matches (ACL_USER) are masked by ACL_MASK, but
owner matches (ACL_USER_OBJ) are NOT masked. Group matches are accumulated
across all matching group entries and then masked.

### ACL Caching

ACLs are cached in `inode->i_acl` and `inode->i_default_acl`. The RCU path
walk (`MAY_NOT_BLOCK`) uses `get_cached_acl_rcu()` for lock-free access. If
the ACL is not cached, the path walk falls back to the blocking path.

---

## 7. Supplementary Groups

### struct group_info

`include/linux/cred.h:28`

```c
struct group_info {
    refcount_t  usage;
    int         ngroups;
    kgid_t      gid[];     /* flexible array, kept sorted */
};
```

The array is kept sorted via `groups_sort()` (`kernel/groups.c:84`) to enable
binary search.

### Group Membership Check

`kernel/groups.c:227`

```c
int in_group_p(kgid_t grp)
{
    const struct cred *cred = current_cred();
    int retval = 1;
    if (!gid_eq(grp, cred->fsgid))
        retval = groups_search(cred->group_info, grp);
    return retval;
}
```

First checks `fsgid`, then binary-searches the sorted supplementary group
array. `in_egroup_p()` does the same but checks `egid` instead.

`groups_search()` (`kernel/groups.c:92`) implements a standard binary search
over the sorted `gid[]` array.

### setgroups Syscall

`kernel/groups.c:198` — requires `CAP_SETGID` (checked via `may_setgroups()`).
Maximum group count is `NGROUPS_MAX` (65536).

---

## 8. Capability Override Rules

When DAC denies access, `generic_permission()` checks capabilities. The
relevant capabilities and their override scope:

```
┌──────────────────────┬────────────────────────────────────────────────┐
│ CAP_DAC_OVERRIDE     │ Override read+write on any file.              │
│                      │ Override execute only if at least one exec    │
│                      │ bit (S_IXUGO) is set on the file.            │
│                      │ Override all permissions on directories.      │
├──────────────────────┼────────────────────────────────────────────────┤
│ CAP_DAC_READ_SEARCH  │ Override read on files.                       │
│                      │ Override read+execute (search) on dirs.       │
├──────────────────────┼────────────────────────────────────────────────┤
│ CAP_FOWNER           │ Override checks that require fsuid == i_uid.  │
│                      │ Needed for: chmod on non-owned files,         │
│                      │ setting xattrs, removing sticky-bit files.    │
├──────────────────────┼────────────────────────────────────────────────┤
│ CAP_CHOWN            │ Override chown/chgrp restrictions.            │
├──────────────────────┼────────────────────────────────────────────────┤
│ CAP_FSETID           │ Don't clear S_ISUID/S_ISGID on chown.        │
│                      │ Allow S_ISGID on files where group doesn't   │
│                      │ match.                                        │
└──────────────────────┴────────────────────────────────────────────────┘
```

### `capable_wrt_inode_uidgid()`

`kernel/capability.c:473`

```c
bool capable_wrt_inode_uidgid(struct mnt_idmap *idmap,
                              const struct inode *inode, int cap)
{
    struct user_namespace *ns = current_user_ns();
    return ns_capable(ns, cap) &&
           privileged_wrt_inode_uidgid(ns, idmap, inode);
}
```

Two conditions: (1) the task has the capability in its user namespace, AND
(2) the inode's UID/GID are mapped into that namespace. This prevents a
process with `CAP_DAC_OVERRIDE` in a user namespace from accessing files
owned by unmapped UIDs.

---

## 9. setuid/setgid Syscalls

All implemented in `kernel/sys.c`. They follow the same pattern:
`prepare_creds()` → modify UID fields → `security_task_fix_setuid()` →
`commit_creds()`.

### setuid(uid)

`kernel/sys.c:651`

```c
long __sys_setuid(uid_t uid)
{
    kuid = make_kuid(ns, uid);
    new = prepare_creds();
    old = current_cred();

    if (ns_capable_setid(old->user_ns, CAP_SETUID)) {
        /* Privileged: set real + saved + effective + fs */
        new->suid = new->uid = kuid;
    } else if (!uid_eq(kuid, old->uid) && !uid_eq(kuid, new->suid)) {
        goto error;     /* Unprivileged: can only set to real or saved */
    }
    new->fsuid = new->euid = kuid;
    return commit_creds(new);
}
```

### Unprivileged UID Change Rules

```
┌──────────────────┬───────────────────────────────────────────────────┐
│ Syscall          │ Unprivileged rules (no CAP_SETUID)               │
├──────────────────┼───────────────────────────────────────────────────┤
│ setuid(uid)      │ Can set euid+fsuid to current {uid, suid}.       │
│                  │ Cannot change uid (real).                         │
├──────────────────┼───────────────────────────────────────────────────┤
│ setreuid(r, e)   │ ruid: can set to old {uid, euid}                 │
│                  │ euid: can set to old {uid, euid, suid}            │
│                  │ If ruid changes or euid != old uid: suid = euid   │
├──────────────────┼───────────────────────────────────────────────────┤
│ setresuid(r,e,s) │ Each can be set to any of current {uid,euid,suid}│
│                  │ Any other value requires CAP_SETUID.              │
├──────────────────┼───────────────────────────────────────────────────┤
│ setfsuid(uid)    │ Can set fsuid to current {uid, euid, suid, fsuid}│
│                  │ Returns the old fsuid value.                      │
└──────────────────┴───────────────────────────────────────────────────┘
```

The `setgid`/`setregid`/`setresgid`/`setfsgid` variants mirror these exactly
using `CAP_SETGID`.

### Capability Adjustment on UID Change

`security/commoncap.c:1120` — `cap_emulate_setxuid()`:

Called via the `security_task_fix_setuid()` hook after every setuid-family
syscall (unless `SECURE_NO_SETUID_FIXUP` is set):

```c
static inline void cap_emulate_setxuid(struct cred *new, const struct cred *old)
{
    kuid_t root_uid = make_kuid(old->user_ns, 0);

    /* All UIDs were root, now none are: clear permitted + effective */
    if ((uid_eq(old->uid, root_uid) || uid_eq(old->euid, root_uid) ||
         uid_eq(old->suid, root_uid)) &&
        (!uid_eq(new->uid, root_uid) && !uid_eq(new->euid, root_uid) &&
         !uid_eq(new->suid, root_uid))) {
        if (!issecure(SECURE_KEEP_CAPS))
            cap_clear(new->cap_permitted);
        cap_clear(new->cap_effective);
        cap_clear(new->cap_ambient);
    }

    /* euid root → non-root: clear effective */
    if (uid_eq(old->euid, root_uid) && !uid_eq(new->euid, root_uid))
        cap_clear(new->cap_effective);

    /* euid non-root → root: effective = permitted */
    if (!uid_eq(old->euid, root_uid) && uid_eq(new->euid, root_uid))
        new->cap_effective = new->cap_permitted;
}
```

This implements the rule: dropping all root UIDs clears capabilities;
dropping only effective root clears effective caps; regaining effective root
restores effective from permitted.

---

## 10. Setuid/Setgid Binaries and execve

### `bprm_fill_uid()` — The Setuid/Setgid Handler

`fs/exec.c:1528`

```c
static void bprm_fill_uid(struct linux_binprm *bprm, struct file *file)
{
    struct inode *inode = file_inode(file);
    unsigned int mode;
    vfsuid_t vfsuid;
    vfsgid_t vfsgid;

    if (!mnt_may_suid(file->f_path.mnt))        /* nosuid mount → skip */
        return;
    if (task_no_new_privs(current))              /* no_new_privs → skip */
        return;

    mode = READ_ONCE(inode->i_mode);
    if (!(mode & (S_ISUID|S_ISGID)))            /* no suid/sgid bits → skip */
        return;

    /* Lock inode, re-read mode atomically to prevent TOCTOU */
    inode_lock(inode);
    mode = inode->i_mode;
    vfsuid = i_uid_into_vfsuid(idmap, inode);
    vfsgid = i_gid_into_vfsgid(idmap, inode);
    err = inode_permission(idmap, inode, MAY_EXEC);
    inode_unlock(inode);

    if (err)                                     /* no exec permission → skip */
        return;

    /* Skip if UID/GID not mapped in this namespace */
    if (!vfsuid_has_mapping(bprm->cred->user_ns, vfsuid) ||
        !vfsgid_has_mapping(bprm->cred->user_ns, vfsgid))
        return;

    if (mode & S_ISUID)
        bprm->cred->euid = vfsuid_into_kuid(vfsuid);   /* euid = file owner */

    if ((mode & (S_ISGID | S_IXGRP)) == (S_ISGID | S_IXGRP))
        bprm->cred->egid = vfsgid_into_kgid(vfsgid);   /* egid = file group */
}
```

### Conditions That Block Setuid

| Condition | Effect |
|-----------|--------|
| `nosuid` mount | `mnt_may_suid()` returns false |
| `no_new_privs` flag | `task_no_new_privs()` returns true |
| No exec permission on file | `inode_permission()` returns error |
| UID/GID unmapped in user namespace | `vfsuid_has_mapping()` returns false |
| Being ptraced (without capable tracer) | `cap_bprm_creds_from_file()` reverts euid/egid |

### The Complete execve Credential Flow

```
do_execveat_common()
  └─ bprm_execve()
       ├─ prepare_bprm_creds()
       │    └─ prepare_exec_creds()       copy current creds, set suid=fsuid=euid
       ├─ check_unsafe_exec()             set bprm->unsafe flags
       │    ├─ LSM_UNSAFE_PTRACE          if ptraced
       │    ├─ LSM_UNSAFE_NO_NEW_PRIVS    if no_new_privs
       │    └─ LSM_UNSAFE_SHARE           if fs_struct shared
       ├─ security_bprm_creds_for_exec()  LSMs set up security labels
       └─ exec_binprm() → search_binary_handler()
            └─ load_binary() → begin_new_exec()
                 ├─ bprm_creds_from_file()
                 │    ├─ bprm_fill_uid()                    apply S_ISUID/S_ISGID
                 │    └─ security_bprm_creds_from_file()
                 │         └─ cap_bprm_creds_from_file()    capability transformation
                 ├─ would_dump()                            non-readable → force nondump
                 ├─ set_dumpable()                          if uid != euid
                 ├─ security_bprm_committing_creds()
                 ├─ commit_creds(bprm->cred)                install new creds
                 └─ security_bprm_committed_creds()
```

### After Setuid Exec

```
uid  = original real UID (unchanged)
euid = file owner UID (from S_ISUID)
suid = euid (set by cap_bprm_creds_from_file)
fsuid = euid (set by cap_bprm_creds_from_file)
```

The program can then:
- Drop privileges: `seteuid(getuid())` → euid = uid (original user)
- Regain privileges: `seteuid(saved_uid)` → euid = suid (file owner)
- Permanently drop: `setuid(getuid())` → all UIDs = uid (irreversible)

---

## 11. Capability Transformation During exec

### `cap_bprm_creds_from_file()`

`security/commoncap.c:919`

This function computes the new capability sets using the POSIX formula:

```
pP' = (X & fP) | (pI & fI) | pA'
pE' = fE ? pP' : pA'

Where:
  pP' = new permitted capabilities
  X   = capability bounding set (cap_bset)
  fP  = file permitted capabilities (from security.capability xattr)
  pI  = process inheritable capabilities
  fI  = file inheritable capabilities
  pA' = new ambient capabilities (cleared if file caps present or UID changed)
  fE  = file effective flag (from xattr)
  pE' = new effective capabilities
```

### File Capabilities

Stored as `security.capability` extended attribute in `struct vfs_ns_cap_data`
format. Read by `get_file_caps()` (`security/commoncap.c:763`) during exec.

File caps are ignored when:
- `nosuid` mount
- `CONFIG_SECURITY_FS_CAPS` disabled
- Process not in the file's superblock user namespace

### Privilege Downgrade

If `no_new_privs` or unsafe ptrace is detected and privileges would escalate:

```c
if ((id_changed || __cap_gained(permitted, new, old)) &&
    ((bprm->unsafe & ~LSM_UNSAFE_PTRACE) ||
     !ptracer_capable(current, new->user_ns))) {
    if (!ns_capable(new->user_ns, CAP_SETUID) ||
        (bprm->unsafe & LSM_UNSAFE_NO_NEW_PRIVS)) {
        new->euid = new->uid;     /* revert setuid */
        new->egid = new->gid;     /* revert setgid */
    }
    new->cap_permitted = cap_intersect(new->cap_permitted, old->cap_permitted);
}
```

### `secureexec` (AT_SECURE)

`security/commoncap.c:999`

Set to 1 when the exec involves a privilege transition (UID change, capability
gain). Passed to the dynamic linker via the ELF auxiliary vector `AT_SECURE=1`,
causing glibc to sanitize the environment (`LD_PRELOAD`, `LD_LIBRARY_PATH`,
`LD_AUDIT`, etc.).

---

## 12. Root (UID 0) and Capabilities

### How Root Gets Capabilities

`security/commoncap.c:828` — `handle_privileged_root()`

When `SECURE_NOROOT` is NOT set (the default), UID 0 is treated specially
during exec:

```c
static void handle_privileged_root(struct linux_binprm *bprm, bool has_fcap,
                                   bool *effective, kuid_t root_uid)
{
    if (!root_privileged())     /* SECURE_NOROOT set → root is not special */
        return;
    if (has_fcap && __is_suid(root_uid, new))
        return;                 /* file caps + suid-root: file caps win */

    /* If euid or ruid is root: pP = cap_bset | cap_inheritable */
    if (__is_eff(root_uid, new) || __is_real(root_uid, new))
        new->cap_permitted = cap_combine(old->cap_bset, old->cap_inheritable);

    /* If euid is root: all permitted become effective */
    if (__is_eff(root_uid, new))
        *effective = true;
}
```

This is why root traditionally has "all privileges" — the capability bounding
set starts as all-ones, so root's `cap_permitted` becomes all-ones, and since
euid is 0, all permitted become effective.

### UID 0 Detection

`security/commoncap.c:807`

```c
static inline bool __is_real(kuid_t uid, struct cred *cred)
{ return uid_eq(cred->uid, uid); }

static inline bool __is_eff(kuid_t uid, struct cred *cred)
{ return uid_eq(cred->euid, uid); }

static inline bool __is_suid(kuid_t uid, struct cred *cred)
{ return !__is_real(uid, cred) && __is_eff(uid, cred); }
```

### Capability Changes on Runtime UID Transitions

`security/commoncap.c:1159` — `cap_task_fix_setuid()`:

| Transition | Capability effect |
|-----------|-------------------|
| All UIDs root → all UIDs non-root | Clear permitted, effective, ambient (unless `SECURE_KEEP_CAPS`) |
| euid root → euid non-root | Clear effective |
| euid non-root → euid root | effective = permitted |
| fsuid root → fsuid non-root | Drop fs-related effective caps |
| fsuid non-root → fsuid root | Raise fs-related effective caps from permitted |

---

## 13. no_new_privs

### Flag Storage and API

Stored in `task->atomic_flags` bit `PFA_NO_NEW_PRIVS` (`include/linux/sched.h:1847`).

**One-way latch**: once set, cannot be cleared. There is no `TASK_PFA_CLEAR`
macro defined for this flag.

```c
/* kernel/sys.c:2705 */
case PR_SET_NO_NEW_PRIVS:
    if (arg2 != 1 || arg3 || arg4 || arg5)
        return -EINVAL;
    task_set_no_new_privs(current);
    break;

case PR_GET_NO_NEW_PRIVS:
    return task_no_new_privs(current) ? 1 : 0;
```

### Effects

| Context | Effect |
|---------|--------|
| `bprm_fill_uid()` | S_ISUID and S_ISGID bits on executables are completely ignored |
| `cap_bprm_creds_from_file()` | euid/egid reverted to uid/gid; `cap_permitted` intersected with old |
| Seccomp | Required for installing seccomp filters without `CAP_SYS_ADMIN` |
| Landlock | Required for `landlock_restrict_self()` without `CAP_SYS_ADMIN` |

### Inheritance

The flag is inherited across `fork()` and preserved across `execve()`. A
process cannot escape `no_new_privs` through any combination of exec and
UID changes.

---

## 14. Dumpability and /proc Restrictions

### Dumpability Values

`include/linux/sched/coredump.h:7`

```c
#define SUID_DUMP_DISABLE   0   /* no dumping */
#define SUID_DUMP_USER      1   /* dump as user of process */
#define SUID_DUMP_ROOT      2   /* dump as root */
```

### When Dumpability Changes

**During `commit_creds()`** (`kernel/cred.c:381`): If euid, egid, fsuid, fsgid
changed, or capabilities increased → `set_dumpable(mm, suid_dumpable)` and
`pdeath_signal = 0`.

**During `begin_new_exec()`** (`fs/exec.c:1206`): If euid != uid or egid !=
gid after the exec → `set_dumpable(mm, suid_dumpable)`.

**`would_dump()`** (`fs/exec.c:1293`): If the binary file is not readable by
the calling user → force non-dumpable via `BINPRM_FLAGS_ENFORCE_NONDUMP`.

### /proc Ownership

`fs/proc/base.c:1906`

When a process is non-dumpable, its `/proc/[pid]/` entries (except the
directory itself) are owned by root. This prevents the original user from
reading `/proc/[pid]/mem`, `/proc/[pid]/maps`, `/proc/[pid]/environ`, etc.
after a setuid exec.

### `suid_dumpable` Sysctl

`fs/exec.c:86` — `/proc/sys/fs/suid_dumpable`

```
0 (default) = SUID_DUMP_DISABLE  — no core dumps for setuid processes
1           = SUID_DUMP_USER     — dump owned by current user (debug mode)
2           = SUID_DUMP_ROOT     — dump owned by root
```

### prctl

```c
prctl(PR_SET_DUMPABLE, SUID_DUMP_DISABLE);  /* or SUID_DUMP_USER */
prctl(PR_GET_DUMPABLE);
```

Only 0 and 1 can be set via prctl. Value 2 is only settable via sysctl.

---

## 15. umask

### Storage

`include/linux/fs_struct.h:10`

```c
struct fs_struct {
    int users;
    seqlock_t seq;
    int umask;          /* stored here */
    int in_exec;
    struct path root, pwd;
};
```

### Syscall

`kernel/sys.c:1960`

```c
SYSCALL_DEFINE1(umask, int, mask)
{
    mask = xchg(&current->fs->umask, mask & S_IRWXUGO);
    return mask;    /* returns old umask */
}
```

Atomically sets the new umask (masked to 0777) and returns the old value.

### Application During File Creation

`include/linux/namei.h:200` — `mode_strip_umask()`

```c
static inline umode_t mode_strip_umask(const struct inode *dir, umode_t mode)
{
    if (!IS_POSIXACL(dir) && !(dir->i_sb->s_iflags & SB_I_NOUMASK))
        mode &= ~current_umask();
    return mode;
}
```

The umask is NOT applied when the parent directory has POSIX ACLs — in that
case, the directory's default ACL controls the initial permissions instead.

---

## 16. chmod and chown

### chmod

`fs/open.c:620` — `chmod_common()`

```c
int chmod_common(const struct path *path, umode_t mode)
{
    newattrs.ia_mode = (mode & S_IALLUGO) | (inode->i_mode & ~S_IALLUGO);
    newattrs.ia_valid = ATTR_MODE | ATTR_CTIME;
    error = notify_change(mnt_idmap(path->mnt), path->dentry,
                          &newattrs, &delegated_inode);
}
```

Permission check in `setattr_prepare()` (`fs/attr.c:161`): requires
`inode_owner_or_capable()` — must be file owner or have `CAP_FOWNER`.

S_ISGID is automatically stripped if the caller is not in the owning group
and lacks `CAP_FSETID`.

### chown

`fs/open.c:741` — `chown_common()`

Permission checking via `chown_ok()` and `chgrp_ok()` in `fs/attr.c`:

```
┌───────────────────────────────────────────────────────────────────┐
│ chown_ok():                                                       │
│   Owner can chown to self (no-op). Otherwise requires CAP_CHOWN.  │
│                                                                   │
│ chgrp_ok():                                                       │
│   Owner can chgrp to own group or any supplementary group.        │
│   Otherwise requires CAP_CHOWN.                                   │
│                                                                   │
│ inode_owner_or_capable() — fs/inode.c:2695:                       │
│   Returns true if fsuid == file owner, or if the task has         │
│   CAP_FOWNER and the inode's UID is mapped in the task's ns.     │
└───────────────────────────────────────────────────────────────────┘
```

### Automatic Security Bit Clearing

On `chown` of a non-directory: `ATTR_KILL_SUID` is set, which clears S_ISUID.
S_ISGID is also cleared (via `setattr_should_drop_sgid()`). This prevents a
user from creating a file, setting it setuid, then chowning it to root.

---

## 17. access() Syscall

`fs/open.c:465` — `do_faccessat()`

The `access()` syscall checks permissions using the **real** UID/GID instead
of the effective UID/GID. This allows setuid programs to check whether the
*original* user has access to a file.

### Credential Override

`fs/open.c:416` — `access_override_creds()`

```c
static const struct cred *access_override_creds(void)
{
    struct cred *override_cred = prepare_creds();

    override_cred->fsuid = override_cred->uid;   /* use REAL uid */
    override_cred->fsgid = override_cred->gid;   /* use REAL gid */

    if (!issecure(SECURE_NO_SETUID_FIXUP)) {
        kuid_t root_uid = make_kuid(override_cred->user_ns, 0);
        if (!uid_eq(override_cred->uid, root_uid))
            cap_clear(override_cred->cap_effective);  /* non-root: clear caps */
        else
            override_cred->cap_effective =
                override_cred->cap_permitted;         /* root: full caps */
    }

    return override_creds(override_cred);
}
```

This temporarily swaps `task->cred` to use real UIDs, then the normal
`inode_permission()` path runs. After the check, `revert_creds()` restores
the original credentials.

### Syscall Variants

```c
access(filename, mode);                              /* always uses real IDs */
faccessat(dfd, filename, mode);                      /* uses real IDs */
faccessat2(dfd, filename, mode, flags);              /* AT_EACCESS: use effective IDs */
```

**TOCTOU warning**: `access()` checks are inherently racy — the file's
permissions may change between the `access()` check and the subsequent
`open()`. Using `access()` before `open()` is generally a security
anti-pattern; instead, attempt the operation directly and handle errors.

---

## 18. Credential Override Mechanism

`include/linux/cred.h:179`

```c
static inline const struct cred *override_creds(const struct cred *override_cred)
{
    return rcu_replace_pointer(current->cred, override_cred, 1);
}

static inline const struct cred *revert_creds(const struct cred *revert_cred)
{
    return rcu_replace_pointer(current->cred, revert_cred, 1);
}
```

This replaces only `task->cred` (subjective context) while leaving
`task->real_cred` untouched. Used by kernel threads and the `access()` syscall
to temporarily assume different permissions.

### Scoped Helper

```c
DEFINE_CLASS(override_creds, const struct cred *,
             revert_creds(_T),
             override_creds(override_cred),
             const struct cred *override_cred)

#define scoped_with_creds(cred)     scoped_class(override_creds, ..., cred)
#define scoped_with_kernel_creds()  scoped_with_creds(kernel_cred())
```

Uses C cleanup attributes to automatically revert credentials when the scope
exits.

### `prepare_kernel_cred()`

`kernel/cred.c:558` — creates credentials for kernel services, typically
copying from `&init_task` to get root-level access.

---

## 19. Securebits

`security/commoncap.c:1307` — managed via `prctl(PR_SET_SECUREBITS)`

```
┌──────────────────────────┬─────────────────────────────────────────┐
│ SECURE_NOROOT            │ UID 0 has no special privilege.         │
│                          │ Root behaves like any other user.       │
├──────────────────────────┼─────────────────────────────────────────┤
│ SECURE_NO_SETUID_FIXUP   │ Don't adjust capabilities when         │
│                          │ setuid/setgid syscalls change UIDs.     │
├──────────────────────────┼─────────────────────────────────────────┤
│ SECURE_KEEP_CAPS         │ Don't clear caps when all UIDs become  │
│                          │ non-root. Cleared after every exec.     │
├──────────────────────────┼─────────────────────────────────────────┤
│ SECURE_NO_CAP_AMBIENT    │ Prevent raising ambient capabilities.   │
│ _RAISE                   │                                         │
├──────────────────────────┼─────────────────────────────────────────┤
│ *_LOCKED variants        │ Each bit has a corresponding _LOCKED    │
│                          │ bit that prevents future changes.       │
└──────────────────────────┴─────────────────────────────────────────┘
```

`SECURE_KEEP_CAPS` is automatically cleared by `cap_bprm_creds_from_file()`
(`security/commoncap.c:994`) after every exec, preventing it from persisting
across exec boundaries.

---

## 20. Key Source Files

| File | Purpose |
|------|---------|
| `include/linux/cred.h` | `struct cred`, accessor macros, lifecycle prototypes |
| `kernel/cred.c` | `prepare_creds()`, `commit_creds()`, `prepare_kernel_cred()` |
| `include/linux/uidgid_types.h` | `kuid_t`, `kgid_t` type definitions |
| `include/linux/uidgid.h` | `make_kuid()`, `from_kuid()`, comparison helpers |
| `include/uapi/linux/stat.h` | `S_ISUID`, `S_ISGID`, `S_IRWXU`, permission bit constants |
| `include/linux/fs.h` | `struct inode` (i_mode, i_uid, i_gid), MAY_* flags |
| `fs/namei.c` | `inode_permission()`, `generic_permission()`, `acl_permission_check()` |
| `fs/posix_acl.c` | `posix_acl_permission()`, ACL checking logic |
| `include/linux/posix_acl.h` | `struct posix_acl`, `struct posix_acl_entry` |
| `kernel/sys.c` | `setuid()`, `setgid()`, `setreuid()`, `setresuid()`, `umask()`, `prctl()` |
| `fs/exec.c` | `bprm_fill_uid()`, `begin_new_exec()`, setuid exec flow |
| `security/commoncap.c` | `cap_bprm_creds_from_file()`, `handle_privileged_root()`, `cap_emulate_setxuid()` |
| `kernel/capability.c` | `capable_wrt_inode_uidgid()`, `ns_capable()` |
| `kernel/groups.c` | `groups_search()`, `in_group_p()`, supplementary groups |
| `fs/open.c` | `chmod_common()`, `chown_common()`, `do_faccessat()` |
| `fs/attr.c` | `setattr_prepare()`, `chown_ok()`, `chgrp_ok()` |
| `fs/inode.c` | `inode_owner_or_capable()` |
| `include/linux/fs_struct.h` | `struct fs_struct` (umask storage) |
| `include/linux/sched/coredump.h` | `SUID_DUMP_*` dumpability constants |
| `include/linux/security.h` | `LSM_UNSAFE_*` flags |
| `include/uapi/linux/capability.h` | `CAP_*` constants |

---

## Call Flow Example: Reading a File as a Setuid Process

```
User runs /usr/bin/passwd (owned by root, S_ISUID set)

1. execve("/usr/bin/passwd")
   └─ bprm_fill_uid():  euid = 0 (root), uid = 1000 (user)
   └─ cap_bprm_creds_from_file():
        handle_privileged_root():  cap_permitted = cap_bset (all caps)
        suid = fsuid = euid = 0
   └─ commit_creds()
   └─ set_dumpable(SUID_DUMP_DISABLE)  [because euid != uid]
   └─ bprm->secureexec = 1  [AT_SECURE → glibc sanitizes env]

2. passwd opens /etc/shadow (mode 0640, owner root:shadow)
   └─ inode_permission(inode, MAY_READ|MAY_WRITE)
        └─ acl_permission_check():
             fsuid(0) == i_uid(0)?  YES → check owner bits
             mode >> 6 = rw- → MAY_READ|MAY_WRITE granted
        └─ security_inode_permission() → SELinux/AppArmor check

3. passwd temporarily drops privileges
   └─ seteuid(getuid())  →  euid = 1000, suid = 0 (saved)
   └─ cap_emulate_setxuid():  clear cap_effective [euid was root, now isn't]
   └─ commit_creds()

4. passwd regains privileges
   └─ seteuid(0)  →  euid = 0 (from suid)
   └─ cap_emulate_setxuid():  cap_effective = cap_permitted
   └─ commit_creds()
```
