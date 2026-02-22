# Linux VFS (Virtual File System) Deep Dive

This document provides an in-depth exploration of the Linux Virtual File System
layer, covering the core data structures, caching mechanisms, and I/O paths
that make up the filesystem abstraction layer.

## Table of Contents

1. [VFS Architecture Overview](#1-vfs-architecture-overview)
2. [Superblock (struct super_block)](#2-superblock-struct-super_block)
3. [Inodes (struct inode)](#3-inodes-struct-inode)
4. [Dentry Cache (dcache)](#4-dentry-cache-dcache)
5. [File Structures](#5-file-structures)
6. [Page Cache](#6-page-cache)
7. [VFS Path Walking](#7-vfs-path-walking)
8. [Buffered I/O Path](#8-buffered-io-path)

---

## 1. VFS Architecture Overview

The Virtual File System (VFS) is an abstraction layer that provides a uniform
interface for filesystem operations regardless of the underlying filesystem type.
Whether you're reading from an ext4 partition, an NFS network share, a procfs
pseudo-filesystem, or a USB drive formatted as FAT32, the same system calls
(open, read, write, close) work identically from userspace.

The VFS achieves this by defining generic data structures (`super_block`, `inode`,
`dentry`, `file`) and operation tables that each filesystem driver must implement.
When a filesystem is mounted, it registers its specific operations (how to read
an inode, how to look up a name in a directory, etc.) with the VFS. The VFS then
dispatches system calls to the appropriate filesystem-specific code. This design
allows Linux to support dozens of filesystem types while presenting a consistent
POSIX interface to applications.

### Core VFS Objects

```
┌─────────────────────────────────────────────────────────────────┐
│                        Userspace                                │
├─────────────────────────────────────────────────────────────────┤
│                    System Call Interface                        │
│              (open, read, write, close, stat, etc.)             │
├─────────────────────────────────────────────────────────────────┤
│                          VFS Layer                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ file     │  │ dentry   │  │ inode    │  │super_    │         │
│  │ struct   │  │ cache    │  │          │  │block     │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│                     │               │             │             │
│                     └───────────────┴─────────────┘             │
│                              Page Cache                         │
├─────────────────────────────────────────────────────────────────┤
│              Filesystem Drivers (ext4, xfs, btrfs, etc.)        │
├─────────────────────────────────────────────────────────────────┤
│                      Block Layer / Device Drivers               │
└─────────────────────────────────────────────────────────────────┘
```

### Key Relationships

```
file_system_type  ──creates──>  super_block
                                     │
                                     ├── s_root ──> dentry (root)
                                     ├── s_inodes ──> inode list
                                     └── s_op ──> super_operations

super_block  ──contains──>  inode
                              │
                              ├── i_op ──> inode_operations
                              ├── i_fop ──> file_operations
                              ├── i_mapping ──> address_space (page cache)
                              └── i_dentry ──> dentry list (aliases)

dentry  ──points to──>  inode
          │
          ├── d_parent ──> parent dentry
          ├── d_children ──> child dentries
          └── d_op ──> dentry_operations

file  ──references──>  dentry + inode
        │
        ├── f_path ──> path (vfsmount + dentry)
        ├── f_inode ──> cached inode pointer
        ├── f_mapping ──> address_space
        └── f_op ──> file_operations
```

---

## 2. Superblock (struct super_block)

The superblock represents a mounted filesystem instance and is the central
control structure for any filesystem. It contains all the global metadata about
the filesystem including block size, mount flags, filesystem-specific operations,
and manages resources like inodes, dentries, and the underlying block device.

Every mounted filesystem has exactly one superblock, even if the same filesystem
type is mounted multiple times (each mount gets its own superblock). The superblock
is created during mount and destroyed during unmount. It serves as the entry point
for most filesystem operations and maintains lists of all inodes and dentries
belonging to that filesystem instance.

### Key Source Files
- `include/linux/fs.h:1289-1429` - struct super_block definition
- `fs/super.c` - Superblock lifecycle implementation

### 2.1 struct super_block Definition

The super_block structure is the largest and most complex VFS structure. It
contains pointers to all the operation tables, the root dentry, block device
references, and various lists tracking all inodes and dentries. The `s_fs_info`
field points to filesystem-specific data (e.g., ext4_sb_info for ext4).

```c
struct super_block {
    struct list_head    s_list;         /* Global superblocks list */
    dev_t               s_dev;          /* Device identifier */
    unsigned char       s_blocksize_bits;
    unsigned long       s_blocksize;
    loff_t              s_maxbytes;     /* Max file size */
    struct file_system_type *s_type;    /* Filesystem type */
    const struct super_operations *s_op;/* Superblock operations */
    const struct dquot_operations *dq_op;
    const struct quotactl_ops *s_qcop;
    const struct export_operations *s_export_op;
    unsigned long       s_flags;        /* Mount flags (SB_RDONLY, etc.) */
    unsigned long       s_iflags;       /* Internal flags (SB_I_*) */
    unsigned long       s_magic;        /* Filesystem magic number */
    struct dentry       *s_root;        /* Root dentry */
    struct rw_semaphore s_umount;       /* Mount/unmount serialization */
    int                 s_count;        /* Temporary reference count */
    atomic_t            s_active;       /* Active reference count */
    void                *s_security;    /* LSM security blob */
    const struct xattr_handler * const *s_xattr;
    const struct fscrypt_operations *s_cop;
    struct fscrypt_keyring *s_master_keys;
    const struct fsverity_operations *s_vop;
    struct unicode_map *s_encoding;
    __u16 s_encoding_flags;
    struct hlist_bl_head s_roots;       /* Alternate roots (NFS) */
    struct list_head    s_mounts;       /* List of mounts */
    struct block_device *s_bdev;        /* Block device */
    struct file         *s_bdev_file;   /* Block device file handle */
    struct backing_dev_info *s_bdi;     /* Backing device info */
    struct mtd_info     *s_mtd;         /* MTD device (flash) */
    struct hlist_node   s_instances;    /* fs_type instances list */
    unsigned int        s_quota_types;
    struct quota_info   s_dquot;
    struct sb_writers   s_writers;      /* Freeze/thaw locks */
    void                *s_fs_info;     /* Filesystem private data */
    u32                 s_time_gran;    /* Timestamp granularity (ns) */
    time64_t            s_time_min;
    time64_t            s_time_max;
    u32                 s_fsnotify_mask;
    struct fsnotify_sb_info *s_fsnotify_info;
    char                s_id[32];       /* Informational name */
    uuid_t              s_uuid;
    u8                  s_uuid_len;
    char                s_sysfs_name[UUID_STRING_LEN + 1];
    unsigned int        s_max_links;
    struct mutex        s_vfs_rename_mutex;
    const char          *s_subtype;
    const struct dentry_operations *s_d_op; /* Default dentry ops */
    struct shrinker     *s_shrink;      /* Memory reclaim shrinker */
    atomic_long_t       s_remove_count;
    int                 s_readonly_remount;
    errseq_t            s_wb_err;       /* Writeback error tracking */
    struct workqueue_struct *s_dio_done_wq;
    struct hlist_head   s_pins;
    struct user_namespace *s_user_ns;
    struct list_lru     s_dentry_lru;   /* Dentry LRU list */
    struct list_lru     s_inode_lru;    /* Inode LRU list */
    struct rcu_head     rcu;
    struct work_struct  destroy_work;
    struct mutex        s_sync_lock;
    int                 s_stack_depth;  /* Filesystem stacking depth */
    spinlock_t          s_inode_list_lock ____cacheline_aligned_in_smp;
    struct list_head    s_inodes;       /* All inodes list */
    spinlock_t          s_inode_wblist_lock;
    struct list_head    s_inodes_wb;    /* Writeback inodes */
} __randomize_layout;
```

### 2.2 struct super_operations

The super_operations table defines how the VFS interacts with the filesystem
for superblock-level and inode-level operations. Filesystems must implement
at least the inode allocation/destruction methods. The `write_inode` callback
syncs inode metadata to disk, while `evict_inode` cleans up when an inode is
being removed from memory. Not all callbacks need to be implemented—NULL
entries use default VFS behavior.

```c
struct super_operations {
    /* Inode lifecycle */
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*free_inode)(struct inode *);

    /* Inode state */
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *wbc);
    int (*drop_inode)(struct inode *);
    void (*evict_inode)(struct inode *);

    /* Superblock lifecycle */
    void (*put_super)(struct super_block *);
    int (*sync_fs)(struct super_block *sb, int wait);
    int (*freeze_super)(struct super_block *, enum freeze_holder who);
    int (*freeze_fs)(struct super_block *);
    int (*thaw_super)(struct super_block *, enum freeze_holder who);
    int (*unfreeze_fs)(struct super_block *);

    /* Statistics and info */
    int (*statfs)(struct dentry *, struct kstatfs *);
    int (*remount_fs)(struct super_block *, int *, char *);
    void (*umount_begin)(struct super_block *);
    int (*show_options)(struct seq_file *, struct dentry *);
    int (*show_devname)(struct seq_file *, struct dentry *);
    int (*show_path)(struct seq_file *, struct dentry *);
    int (*show_stats)(struct seq_file *, struct dentry *);

    /* Quota operations */
    ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
    ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
    struct dquot __rcu **(*get_dquots)(struct inode *);

    /* Memory reclaim */
    long (*nr_cached_objects)(struct super_block *, struct shrink_control *);
    long (*free_cached_objects)(struct super_block *, struct shrink_control *);

    /* Emergency shutdown */
    void (*shutdown)(struct super_block *sb);
};
```

### 2.3 Mount Flags (s_flags)

| Flag | Value | Description |
|------|-------|-------------|
| `SB_RDONLY` | BIT(0) | Mount read-only |
| `SB_NOSUID` | BIT(1) | Ignore suid/sgid bits |
| `SB_NODEV` | BIT(2) | Disallow device access |
| `SB_NOEXEC` | BIT(3) | Disallow execution |
| `SB_SYNCHRONOUS` | BIT(4) | Synchronous writes |
| `SB_MANDLOCK` | BIT(6) | Allow mandatory locks |
| `SB_DIRSYNC` | BIT(7) | Synchronous directory ops |
| `SB_NOATIME` | BIT(10) | Don't update access times |
| `SB_NODIRATIME` | BIT(11) | Don't update dir access times |
| `SB_SILENT` | BIT(15) | Suppress mount messages |
| `SB_POSIXACL` | BIT(16) | POSIX ACL support |
| `SB_INLINECRYPT` | BIT(17) | Use blk-crypto |
| `SB_I_VERSION` | BIT(23) | Update i_version for NFS |
| `SB_LAZYTIME` | BIT(25) | Lazy timestamp updates |
| `SB_BORN` | BIT(29) | Superblock fully initialized |
| `SB_ACTIVE` | BIT(30) | Superblock is active |

### 2.4 Superblock Lifecycle

```
alloc_super()
    │
    ├── kzalloc(sizeof(struct super_block))
    ├── Initialize locks (s_umount, s_sync_lock, etc.)
    ├── Initialize lists (s_mounts, s_inodes, etc.)
    ├── Set s_count = 1, s_active = 1
    ├── Allocate shrinker (super_cache_scan, super_cache_count)
    └── Initialize LRU lists (s_dentry_lru, s_inode_lru)
           │
           ▼
sget_fc() / sget()
    │
    ├── Search existing superblocks via test() callback
    ├── If found: grab_super() to acquire reference
    ├── If not found: allocate new via alloc_super()
    ├── Call set() callback to initialize
    ├── Add to super_blocks global list
    ├── Add to fs_type->fs_supers
    └── Register shrinker
           │
           ▼
vfs_get_tree()
    │
    ├── Call fs_type->get_tree() (fills superblock)
    ├── Set SB_BORN flag
    └── Superblock ready for use
           │
           ▼
[Normal operation]
           │
           ▼
deactivate_super()
    │
    ├── Decrement s_active
    └── If s_active == 0:
        ├── Free shrinker
        ├── Call fs_type->kill_sb()
        │   └── generic_shutdown_super()
        │       ├── shrink_dcache_for_umount()
        │       ├── sync_filesystem()
        │       ├── evict_inodes()
        │       ├── Call s_op->put_super()
        │       └── Set SB_DYING flag
        ├── Destroy LRU lists
        └── put_filesystem()
```

### 2.5 Superblock Locking

| Lock | Type | Purpose |
|------|------|---------|
| `s_umount` | rw_semaphore | Mount/unmount/remount serialization |
| `sb_lock` | spinlock (global) | Protects super_blocks list and s_count |
| `s_inode_list_lock` | spinlock | Protects s_inodes list |
| `s_inode_wblist_lock` | spinlock | Protects s_inodes_wb list |
| `s_vfs_rename_mutex` | mutex | Serializes renames |
| `s_sync_lock` | mutex | Serializes sync operations |
| `s_writers.rw_sem[]` | percpu_rw_semaphore | Freeze/thaw at multiple levels |

---

## 3. Inodes (struct inode)

The inode (index node) is the kernel's in-memory representation of a filesystem
object such as a file, directory, device node, socket, or symbolic link. It contains
all the metadata about the object including ownership, permissions, timestamps, size,
and pointers to the actual data. The term comes from Unix tradition where inodes
were stored in an "index" area separate from file data.

Inodes are uniquely identified within a filesystem by their inode number (`i_ino`).
The VFS maintains a global hash table for fast inode lookup by (superblock, inode_number)
pairs. Inodes are reference-counted and cached in memory even after the file is closed,
allowing quick access if the file is opened again. The inode also contains the
`address_space` structure that manages the page cache for this file's data.

### Key Source Files
- `include/linux/fs.h:668-783` - struct inode definition
- `fs/inode.c` - Inode lifecycle implementation

### 3.1 struct inode Definition

The inode structure contains all metadata about a filesystem object. Key fields
include `i_mode` (file type and permissions), `i_uid`/`i_gid` (ownership),
`i_size` (file size), timestamps, and `i_ino` (inode number). The `i_mapping`
pointer connects to the page cache, while `i_op` and `i_fop` point to operation
tables. Filesystems typically embed this structure in a larger filesystem-specific
inode (e.g., ext4_inode_info) and use `container_of()` to access private data.

```c
struct inode {
    /* File type and permissions */
    umode_t         i_mode;
    unsigned short  i_opflags;       /* IOP_* fast-path flags */
    kuid_t          i_uid;
    kgid_t          i_gid;
    unsigned int    i_flags;         /* FS-independent flags */

    /* ACL caching */
#ifdef CONFIG_FS_POSIX_ACL
    struct posix_acl *i_acl;
    struct posix_acl *i_default_acl;
#endif

    /* Operations and parent */
    const struct inode_operations *i_op;
    struct super_block *i_sb;
    struct address_space *i_mapping;  /* Page cache mapping */

    /* Security */
#ifdef CONFIG_SECURITY
    void            *i_security;
#endif

    /* Identity */
    unsigned long   i_ino;           /* Inode number */
    union {
        const unsigned int i_nlink;  /* Hard link count */
        unsigned int __i_nlink;
    };
    dev_t           i_rdev;          /* Device number */

    /* Size and timestamps */
    loff_t          i_size;
    time64_t        i_atime_sec;
    time64_t        i_mtime_sec;
    time64_t        i_ctime_sec;
    u32             i_atime_nsec;
    u32             i_mtime_nsec;
    u32             i_ctime_nsec;
    u32             i_generation;    /* Version for NFS */

    /* Locking and state */
    spinlock_t      i_lock;
    unsigned short  i_bytes;         /* Bytes in last block */
    u8              i_blkbits;       /* Block size bits */
    enum rw_hint    i_write_hint;
    blkcnt_t        i_blocks;        /* File size in blocks */

#ifdef __NEED_I_SIZE_ORDERED
    seqcount_t      i_size_seqcount;
#endif

    u32             i_state;         /* State flags (I_NEW, I_DIRTY, etc.) */
    struct rw_semaphore i_rwsem;     /* Directory operations lock */

    /* Dirty tracking */
    unsigned long   dirtied_when;
    unsigned long   dirtied_time_when;

    /* Lists and linkage */
    struct hlist_node i_hash;        /* Hash table linkage */
    struct list_head i_io_list;      /* Backing device IO list */

#ifdef CONFIG_CGROUP_WRITEBACK
    struct bdi_writeback *i_wb;
    int             i_wb_frn_winner;
    u16             i_wb_frn_avg_time;
    u16             i_wb_frn_history;
#endif

    struct list_head i_lru;          /* LRU list */
    struct list_head i_sb_list;      /* Superblock inode list */
    struct list_head i_wb_list;      /* Writeback list */

    union {
        struct hlist_head i_dentry;  /* Dentry aliases */
        struct rcu_head i_rcu;       /* RCU callback */
    };

    /* Reference counts */
    atomic64_t      i_version;
    atomic64_t      i_sequence;
    atomic_t        i_count;         /* Reference count */
    atomic_t        i_dio_count;     /* Direct I/O count */
    atomic_t        i_writecount;    /* Writers count */

#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
    atomic_t        i_readcount;     /* Read-only opens */
#endif

    union {
        const struct file_operations *i_fop;
        void (*free_inode)(struct inode *);
    };

    struct file_lock_context *i_flctx;
    struct address_space i_data;     /* Embedded address_space */
    struct list_head i_devices;

    union {
        struct pipe_inode_info *i_pipe;
        struct cdev *i_cdev;
        char *i_link;                /* Symlink target */
        unsigned i_dir_seq;          /* Directory sequence */
    };

    /* Notification and crypto */
#ifdef CONFIG_FSNOTIFY
    __u32           i_fsnotify_mask;
    struct fsnotify_mark_connector __rcu *i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
    struct fscrypt_inode_info *i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
    struct fsverity_info *i_verity_info;
#endif

    void            *i_private;      /* Filesystem private */
} __randomize_layout;
```

### 3.2 Inode State Flags (i_state)

```c
#define I_NEW               (1 << 0)  /* Being initialized */
#define I_SYNC              (1 << 1)  /* Writeback in progress */
#define I_LRU_ISOLATING     (1 << 2)  /* LRU isolation in progress */
#define I_DIRTY_SYNC        (1 << 3)  /* Metadata dirty */
#define I_DIRTY_DATASYNC    (1 << 4)  /* Data-related metadata dirty */
#define I_DIRTY_PAGES       (1 << 5)  /* Data pages dirty */
#define I_WILL_FREE         (1 << 6)  /* About to be freed */
#define I_FREEING           (1 << 7)  /* Being freed */
#define I_CLEAR             (1 << 8)  /* Cleared (post-eviction) */
#define I_REFERENCED        (1 << 9)  /* Recently referenced (LRU) */
#define I_LINKABLE          (1 << 10) /* Can have nlink=0 while alive */
#define I_DIRTY_TIME        (1 << 11) /* Only timestamps dirty */
#define I_WB_SWITCH         (1 << 12) /* Cgroup writeback switch */
#define I_OVL_INUSE         (1 << 13) /* Used by overlayfs */
#define I_CREATING          (1 << 14) /* Being created */
#define I_DONTCACHE         (1 << 15) /* Evict immediately */
#define I_SYNC_QUEUED       (1 << 16) /* Queued for sync */
#define I_PINNING_NETFS_WB  (1 << 17) /* Pinning netfs writeback */

/* Compound flags */
#define I_DIRTY_INODE       (I_DIRTY_SYNC | I_DIRTY_DATASYNC)
#define I_DIRTY             (I_DIRTY_INODE | I_DIRTY_PAGES)
#define I_DIRTY_ALL         (I_DIRTY | I_DIRTY_TIME)
```

### 3.3 struct inode_operations

The inode_operations table defines operations on directory entries and inode
metadata. The crucial `lookup` callback resolves a name within a directory to
its inode. Creation operations (`create`, `mkdir`, `mknod`, `symlink`, `link`)
add new entries, while `unlink` and `rmdir` remove them. The `rename` callback
handles moving/renaming entries. Attribute operations (`setattr`, `getattr`)
handle chmod, chown, truncate, and stat. For regular files, most operations
go through file_operations instead.

```c
struct inode_operations {
    /* Lookup and link operations */
    struct dentry *(*lookup)(struct inode *, struct dentry *, unsigned int);
    const char *(*get_link)(struct dentry *, struct inode *, struct delayed_call *);
    int (*permission)(struct mnt_idmap *, struct inode *, int);
    struct posix_acl *(*get_inode_acl)(struct inode *, int, bool);
    int (*readlink)(struct dentry *, char __user *, int);

    /* File/directory creation */
    int (*create)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t, bool);
    int (*link)(struct dentry *, struct inode *, struct dentry *);
    int (*unlink)(struct inode *, struct dentry *);
    int (*symlink)(struct mnt_idmap *, struct inode *, struct dentry *, const char *);
    int (*mkdir)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t);
    int (*rmdir)(struct inode *, struct dentry *);
    int (*mknod)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t, dev_t);
    int (*rename)(struct mnt_idmap *, struct inode *, struct dentry *,
                  struct inode *, struct dentry *, unsigned int);

    /* Attribute operations */
    int (*setattr)(struct mnt_idmap *, struct dentry *, struct iattr *);
    int (*getattr)(struct mnt_idmap *, const struct path *, struct kstat *, u32, unsigned int);
    ssize_t (*listxattr)(struct dentry *, char *, size_t);

    /* Advanced operations */
    int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start, u64 len);
    int (*update_time)(struct inode *, int);
    int (*atomic_open)(struct inode *, struct dentry *, struct file *, unsigned, umode_t);
    int (*tmpfile)(struct mnt_idmap *, struct inode *, struct file *, umode_t);

    /* ACL operations */
    struct posix_acl *(*get_acl)(struct mnt_idmap *, struct dentry *, int);
    int (*set_acl)(struct mnt_idmap *, struct dentry *, struct posix_acl *, int);

    /* File attributes (ioctl interface) */
    int (*fileattr_set)(struct mnt_idmap *, struct dentry *, struct fileattr *);
    int (*fileattr_get)(struct dentry *, struct fileattr *);

    struct offset_ctx *(*get_offset_ctx)(struct inode *);
} ____cacheline_aligned;
```

### 3.4 Inode Hash Table

The kernel maintains a global hash table for fast inode lookup by (superblock, inode_number).

```c
/* Global variables (fs/inode.c) */
static unsigned int i_hash_mask __ro_after_init;
static unsigned int i_hash_shift __ro_after_init;
static struct hlist_head *inode_hashtable __ro_after_init;
static __cacheline_aligned_in_smp DEFINE_SPINLOCK(inode_hash_lock);
```

**Hash Function:**
```c
static unsigned long hash(struct super_block *sb, unsigned long hashval)
{
    unsigned long tmp = (hashval * (unsigned long)sb) ^
                        (GOLDEN_RATIO_PRIME + hashval) / L1_CACHE_BYTES;
    tmp = tmp ^ ((tmp ^ GOLDEN_RATIO_PRIME) >> i_hash_shift);
    return tmp & i_hash_mask;
}
```

### 3.5 Inode Lifecycle

```
alloc_inode(sb)
    │
    ├── sb->s_op->alloc_inode() or kmem_cache_alloc(inode_cachep)
    └── inode_init_always():
        ├── Set i_sb, i_blkbits
        ├── Initialize i_count = 1
        ├── Initialize locks (i_lock, i_rwsem)
        ├── Initialize i_data (address_space)
        ├── Set i_mapping = &i_data
        └── security_inode_alloc()
           │
           ▼
new_inode(sb) or iget_locked(sb, ino)
    │
    ├── alloc_inode()
    ├── Add to sb->s_inodes list
    ├── Hash if needed (iget_locked sets I_NEW)
    └── Return inode
           │
           ▼
[Filesystem fills inode fields]
           │
           ▼
unlock_new_inode()
    │
    └── Clear I_NEW, wake waiters
           │
           ▼
[Normal operation - ihold()/iput() manage references]
           │
           ▼
iput() when i_count reaches 0
    │
    ├── Call s_op->drop_inode() or generic_drop_inode()
    ├── If should evict:
    │   ├── Set I_FREEING
    │   ├── Remove from LRU
    │   └── evict():
    │       ├── Wait for writeback
    │       ├── Call s_op->evict_inode()
    │       ├── Remove from hash
    │       └── destroy_inode()
    └── If should cache:
        └── Add to LRU via __inode_add_lru()
```

### 3.6 Inode Reference Counting

```c
/* Increment reference (caller must already hold one) */
void ihold(struct inode *inode);

/* Decrement reference (may trigger eviction) */
void iput(struct inode *inode);

/* Get inode by number (allocate if needed) */
struct inode *iget_locked(struct super_block *sb, unsigned long ino);

/* Lookup inode without allocating */
struct inode *ilookup(struct super_block *sb, unsigned long ino);

/* Mark inode dirty */
void mark_inode_dirty(struct inode *inode);
void mark_inode_dirty_sync(struct inode *inode);
```

---

## 4. Dentry Cache (dcache)

The dentry (directory entry) cache is one of the most performance-critical
components of the VFS. It maps pathname components (like "home", "user", "file.txt")
to their corresponding inodes, allowing the kernel to translate pathnames to
filesystem objects without hitting the disk for every path component.

Each dentry represents one component of a pathname and maintains parent-child
relationships to form a tree structure mirroring the filesystem hierarchy. Dentries
can be positive (pointing to a valid inode) or negative (representing a name that
was looked up but doesn't exist, cached to avoid repeated failed lookups). The
dcache is crucial for path lookup performance and is the primary target of the
RCU-walk optimization that allows lockless path traversal.

### Key Source Files
- `include/linux/dcache.h` - struct dentry and API
- `fs/dcache.c` - Dentry cache implementation

### 4.1 struct dentry Definition

The dentry structure is carefully laid out for cache efficiency. The first
cacheline contains fields accessed during RCU-walk lookup: flags, sequence
counter, hash linkage, parent pointer, name, and inode pointer. Short names
(up to DNAME_INLINE_LEN, typically 32 bytes) are stored inline in `d_iname`
to avoid a separate allocation. The `d_lockref` combines a spinlock and
reference count for efficient atomic operations.

```c
struct dentry {
    /* RCU lookup touched fields (first cacheline) */
    unsigned int d_flags;           /* Protected by d_lock */
    seqcount_spinlock_t d_seq;      /* Per-dentry seqlock */
    struct hlist_bl_node d_hash;    /* Hash table linkage */
    struct dentry *d_parent;        /* Parent directory */
    struct qstr d_name;             /* Name hash and string */
    struct inode *d_inode;          /* Associated inode (NULL = negative) */
    unsigned char d_iname[DNAME_INLINE_LEN]; /* Inline name storage */

    /* Reference and metadata (second cacheline) */
    const struct dentry_operations *d_op;
    struct super_block *d_sb;
    unsigned long d_time;           /* Used by d_revalidate */
    void *d_fsdata;                 /* Filesystem private data */
    struct lockref d_lockref;       /* Combined lock and refcount */

    union {
        struct list_head d_lru;     /* LRU list */
        wait_queue_head_t *d_wait;  /* In-lookup wait queue */
    };

    struct hlist_node d_sib;        /* Child of parent list */
    struct hlist_head d_children;   /* Our children */

    union {
        struct hlist_node d_alias;  /* Inode alias list */
        struct hlist_bl_node d_in_lookup_hash;
        struct rcu_head d_rcu;
    } d_u;
};
```

### 4.2 Dentry Flags (d_flags)

```c
/* Operation flags (set by d_set_d_op) */
#define DCACHE_OP_HASH          BIT(0)   /* d_hash() defined */
#define DCACHE_OP_COMPARE       BIT(1)   /* d_compare() defined */
#define DCACHE_OP_REVALIDATE    BIT(2)   /* d_revalidate() defined */
#define DCACHE_OP_DELETE        BIT(3)   /* d_delete() defined */
#define DCACHE_OP_PRUNE         BIT(4)   /* d_prune() defined */

/* State flags */
#define DCACHE_DISCONNECTED     BIT(5)   /* Not in dcache tree */
#define DCACHE_REFERENCED       BIT(6)   /* Recently accessed */
#define DCACHE_DONTCACHE        BIT(7)   /* Don't cache on final dput */
#define DCACHE_CANT_MOUNT       BIT(8)   /* Cannot be mount point */
#define DCACHE_GENOCIDE         BIT(9)   /* Genocide in progress */
#define DCACHE_SHRINK_LIST      BIT(10)  /* On shrinker list */

#define DCACHE_OP_WEAK_REVALIDATE BIT(11)
#define DCACHE_NFSFS_RENAMED    BIT(12)  /* NFS silly-rename victim */
#define DCACHE_FSNOTIFY_PARENT_WATCHED BIT(14)
#define DCACHE_DENTRY_KILLED    BIT(15)

/* Mount flags */
#define DCACHE_MOUNTED          BIT(16)  /* Is a mountpoint */
#define DCACHE_NEED_AUTOMOUNT   BIT(17)  /* Trigger automount */
#define DCACHE_MANAGE_TRANSIT   BIT(18)  /* Manage transit */
#define DCACHE_MANAGED_DENTRY   (DCACHE_MOUNTED | DCACHE_NEED_AUTOMOUNT | DCACHE_MANAGE_TRANSIT)

#define DCACHE_LRU_LIST         BIT(19)  /* On LRU or shrink list */

/* Entry type (bits 20-22) */
#define DCACHE_ENTRY_TYPE       (7 << 20)
#define DCACHE_MISS_TYPE        (0 << 20)  /* Negative dentry */
#define DCACHE_WHITEOUT_TYPE    (1 << 20)  /* Whiteout (overlay) */
#define DCACHE_DIRECTORY_TYPE   (2 << 20)  /* Directory */
#define DCACHE_AUTODIR_TYPE     (3 << 20)  /* Lookupless directory */
#define DCACHE_REGULAR_TYPE     (4 << 20)  /* Regular file */
#define DCACHE_SPECIAL_TYPE     (5 << 20)  /* Device, socket, FIFO */
#define DCACHE_SYMLINK_TYPE     (6 << 20)  /* Symlink */

#define DCACHE_NOKEY_NAME       BIT(25)  /* Encrypted without key */
#define DCACHE_OP_REAL          BIT(26)  /* d_real() defined */
#define DCACHE_PAR_LOOKUP       BIT(28)  /* Parallel lookup */
#define DCACHE_DENTRY_CURSOR    BIT(29)  /* Directory cursor */
#define DCACHE_NORCU            BIT(30)  /* No RCU delay for free */
```

### 4.3 struct dentry_operations

The dentry_operations table allows filesystems to customize dentry behavior.
The `d_revalidate` callback is critical for network filesystems (NFS, CIFS)
to check if a cached dentry is still valid. The `d_hash` and `d_compare`
callbacks support case-insensitive filesystems. The `d_automount` callback
triggers automounting when the dentry is traversed. Most local filesystems
use NULL (default) dentry operations.

```c
struct dentry_operations {
    /* Validation */
    int (*d_revalidate)(struct dentry *, unsigned int);
    int (*d_weak_revalidate)(struct dentry *, unsigned int);

    /* Custom hash/compare (case-insensitive filesystems) */
    int (*d_hash)(const struct dentry *, struct qstr *);
    int (*d_compare)(const struct dentry *, unsigned int, const char *, const struct qstr *);

    /* Lifecycle */
    int (*d_delete)(const struct dentry *);
    int (*d_init)(struct dentry *);
    void (*d_release)(struct dentry *);
    void (*d_prune)(struct dentry *);
    void (*d_iput)(struct dentry *, struct inode *);

    /* Display */
    char *(*d_dname)(struct dentry *, char *, int);

    /* Mount management */
    struct vfsmount *(*d_automount)(struct path *);
    int (*d_manage)(const struct path *, bool);

    /* Overlay support */
    struct dentry *(*d_real)(struct dentry *, enum d_real_type type);
} ____cacheline_aligned;
```

### 4.4 Dentry Hash Table

```c
/* Global hash table */
static struct hlist_bl_head *dentry_hashtable __ro_after_init;
static unsigned int d_hash_shift __ro_after_init;

/* Hash function */
static inline struct hlist_bl_head *d_hash(unsigned long hashlen)
{
    return dentry_hashtable +
           runtime_const_shift_right_32(hashlen, d_hash_shift);
}
```

### 4.5 Dentry Lookup Functions

```c
/* RCU-mode lookup (no locks, returns seqcount) */
struct dentry *__d_lookup_rcu(const struct dentry *parent,
                              const struct qstr *name,
                              unsigned *seqp);

/* Locked lookup (takes d_lock) */
struct dentry *__d_lookup(const struct dentry *parent,
                          const struct qstr *name);

/* Safe lookup with rename_lock protection */
struct dentry *d_lookup(const struct dentry *parent,
                        const struct qstr *name);

/* Allocate with parallel lookup coordination */
struct dentry *d_alloc_parallel(struct dentry *parent,
                                const struct qstr *name,
                                wait_queue_head_t *wq);
```

### 4.6 Dentry Reference Counting

```c
/* Get reference */
static inline struct dentry *dget(struct dentry *dentry)
{
    if (dentry)
        lockref_get(&dentry->d_lockref);
    return dentry;
}

/* Release reference (may free) */
void dput(struct dentry *dentry);

/* Delete from hash and optionally turn negative */
void d_delete(struct dentry *dentry);

/* Invalidate dentry */
void d_invalidate(struct dentry *dentry);
```

### 4.7 Negative Dentries

A **negative dentry** has `d_inode == NULL` and represents a name that was looked up but doesn't exist.

**Benefits:**
- Accelerates repeated access to non-existent files
- Reduces filesystem lookup overhead

**Policy:**
- Controlled by `sysctl fs.dentry-negative`
- Value 0: Allow negative dentries (default)
- Value 1: Drop instead of caching

### 4.8 Dentry LRU and Reclaim

```c
/* Add to LRU (when refcount drops to 0) */
static void d_lru_add(struct dentry *dentry);

/* Remove from LRU */
static void d_lru_del(struct dentry *dentry);

/* Shrinker callbacks */
long prune_dcache_sb(struct super_block *sb, struct shrink_control *sc);
void shrink_dcache_sb(struct super_block *sb);  /* Force eviction */
```

**LRU Algorithm:**
1. When refcount reaches 0, add to LRU with `DCACHE_LRU_LIST`
2. On access, set `DCACHE_REFERENCED` flag
3. Shrinker scans LRU:
   - If `DCACHE_REFERENCED` set: clear flag, rotate to end
   - Otherwise: evict dentry

---

## 5. File Structures

The `struct file` represents an open file instance and is the kernel-side
counterpart of a userspace file descriptor. While the inode represents the
file itself (persistent across opens), the file structure represents a specific
open of that file and contains per-open state like the current position, access
mode, and credentials of the opener.

Multiple file descriptors can point to the same file structure (via dup()),
and multiple file structures can point to the same inode (multiple opens of
the same file). The file structure is reference-counted; it's allocated when
a file is opened and freed when the last reference is dropped (typically when
the last file descriptor pointing to it is closed). The file operations table
(`f_op`) provides the interface between system calls and filesystem-specific
implementations.

### Key Source Files
- `include/linux/fs.h:1068-1106` - struct file definition
- `include/linux/fs.h:2105-2148` - struct file_operations
- `fs/file_table.c` - File structure management
- `fs/open.c` - File opening logic

### 5.1 struct file Definition

The file structure contains per-open state. Key fields include `f_pos` (current
file position), `f_flags` (open flags like O_RDONLY, O_APPEND), `f_mode` (access
mode), and `f_cred` (credentials of the process that opened the file). The
`f_path` contains the mount and dentry for pathname operations. The `f_mapping`
points to the address_space for page cache access. The structure is allocated
from a dedicated slab cache and is relatively small compared to inodes.

```c
struct file {
    atomic_long_t           f_count;        /* Reference count */
    spinlock_t              f_lock;         /* Protects f_ep, f_flags */
    fmode_t                 f_mode;         /* File mode (access flags) */
    const struct file_operations *f_op;     /* File operations */
    struct address_space    *f_mapping;     /* Page cache mapping */
    void                    *private_data;  /* Filesystem private data */
    struct inode            *f_inode;       /* Cached inode pointer */
    unsigned int            f_flags;        /* O_* flags from open */
    unsigned int            f_iocb_flags;   /* IOCB flags cache */
    const struct cred       *f_cred;        /* Opener's credentials */

    /* Position and locking */
    struct mutex            f_pos_lock;     /* Position lock (FMODE_ATOMIC_POS) */
    loff_t                  f_pos;          /* Current position */

    /* Path context */
    struct path             f_path;         /* vfsmount + dentry */

    /* Eventpoll */
    struct hlist_head       *f_ep;          /* Eventpoll list */

    /* Owner (fasync) */
    struct fown_struct      *f_owner;

    /* Error tracking */
    errseq_t                f_wb_err;       /* Writeback error */
    errseq_t                f_sb_err;       /* Superblock error */

#ifdef CONFIG_EPOLL
    struct hlist_head       f_ep_links;
    struct list_head        f_tfile_llink;
#endif
} __randomize_layout;
```

### 5.2 struct path

The path structure is a simple pair that identifies a location in the filesystem
namespace. It combines a mount point (`vfsmount`) with a dentry, which together
uniquely identify a file even when the same filesystem is mounted multiple times
or when bind mounts create multiple paths to the same file.

```c
struct path {
    struct vfsmount *mnt;   /* Mount structure */
    struct dentry *dentry;  /* Dentry in that mount */
};
```

### 5.3 File Mode Flags (f_mode)

```c
/* Access flags */
#define FMODE_READ              (1 << 0)
#define FMODE_WRITE             (1 << 1)

/* Capabilities */
#define FMODE_CAN_READ          (1 << 2)
#define FMODE_CAN_WRITE         (1 << 3)
#define FMODE_CAN_ODIRECT       (1 << 4)

/* Position handling */
#define FMODE_ATOMIC_POS        (1 << 5)  /* Serialize pread/pwrite */
#define FMODE_LSEEK             (1 << 6)

/* State flags */
#define FMODE_OPENED            (1 << 7)
#define FMODE_CREATED           (1 << 8)
#define FMODE_PREAD             (1 << 9)
#define FMODE_PWRITE            (1 << 10)
#define FMODE_EXEC              (1 << 11)
#define FMODE_NONOTIFY          (1 << 12)
#define FMODE_PATH              (1 << 13)
#define FMODE_NOREUSE           (1 << 14)

/* Seek modes */
#define FMODE_STREAM            (1 << 16)
#define FMODE_UNSIGNED_OFFSET   (1 << 17)

/* Backing capabilities */
#define FMODE_BACKING           (1 << 18)
#define FMODE_NOWAIT            (1 << 19)
#define FMODE_NEED_UNMOUNT      (1 << 20)
```

### 5.4 struct file_operations

The file_operations table is the primary interface between system calls and
filesystem code for open files. The `read_iter`/`write_iter` callbacks handle
data transfer using the modern iov_iter interface (replacing older read/write).
The `llseek` callback implements seeking. The `mmap` callback sets up memory
mappings. The `fsync` callback ensures data is persisted to storage. For
directories, `iterate_shared` lists entries. Many filesystems use generic
implementations (e.g., generic_file_read_iter) for common operations.

```c
struct file_operations {
    struct module *owner;

    /* Seeking */
    loff_t (*llseek)(struct file *, loff_t, int);

    /* Basic I/O */
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);

    /* Vectored I/O (preferred) */
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);

    /* io_uring support */
    int (*uring_cmd)(struct io_uring_cmd *, unsigned int);
    int (*uring_cmd_iopoll)(struct io_uring_cmd *, struct io_comp_batch *, unsigned int);

    /* Directory iteration */
    int (*iterate_shared)(struct file *, struct dir_context *);

    /* Polling */
    __poll_t (*poll)(struct file *, struct poll_table_struct *);

    /* ioctl */
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);

    /* Memory mapping */
    int (*mmap)(struct file *, struct vm_area_struct *);

    /* Lifecycle */
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);

    /* Sync */
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);

    /* Locking */
    int (*lock)(struct file *, int, struct file_lock *);
    int (*flock)(struct file *, int, struct file_lock *);

    /* Splice */
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);

    /* Space management */
    long (*fallocate)(struct file *, int mode, loff_t offset, loff_t len);

    /* Advanced */
    int (*setlease)(struct file *, int, struct file_lease **, void **);
    void (*show_fdinfo)(struct seq_file *, struct file *);
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *, loff_t, size_t, unsigned int);
    loff_t (*remap_file_range)(struct file *, loff_t, struct file *, loff_t, loff_t, unsigned int);
    int (*fadvise)(struct file *, loff_t, loff_t, int);

    /* Polling for io_uring */
    int (*iopoll)(struct kiocb *, struct io_comp_batch *, unsigned int);
} __randomize_layout;
```

### 5.5 File Lifecycle

```c
/* Allocation */
struct file *alloc_empty_file(int flags, const struct cred *cred);

/* Opening */
int do_dentry_open(struct file *f, struct inode *inode,
                   int (*open)(struct inode *, struct file *));

/* Reference counting */
struct file *get_file(struct file *f);  /* Increment */
void fput(struct file *f);              /* Decrement (may close) */

/* __fput cleanup sequence */
1. fsnotify_close(file)
2. eventpoll_release(file)
3. locks_remove_file(file)
4. security_file_release(file)
5. f_op->release(inode, file)
6. fops_put(f_op)  /* Module refcount */
7. path_put(&f->f_path)
8. file_free(file)
```

### 5.6 Open Flags (f_flags)

From `<fcntl.h>`:

| Flag | Description |
|------|-------------|
| `O_RDONLY` | Read only |
| `O_WRONLY` | Write only |
| `O_RDWR` | Read and write |
| `O_CREAT` | Create if doesn't exist |
| `O_EXCL` | Fail if exists (with O_CREAT) |
| `O_TRUNC` | Truncate to zero length |
| `O_APPEND` | Append mode |
| `O_NONBLOCK` | Non-blocking I/O |
| `O_SYNC` | Synchronous writes |
| `O_DSYNC` | Synchronized data writes |
| `O_DIRECT` | Direct I/O (bypass page cache) |
| `O_LARGEFILE` | Large file support |
| `O_DIRECTORY` | Must be directory |
| `O_NOFOLLOW` | Don't follow symlinks |
| `O_NOATIME` | Don't update atime |
| `O_CLOEXEC` | Close on exec |
| `O_PATH` | Path-only handle |
| `O_TMPFILE` | Unnamed temporary file |

---

## 6. Page Cache

The page cache is the kernel's primary mechanism for caching file data in memory.
Rather than reading from and writing to disk for every I/O operation, the kernel
keeps recently accessed file pages in RAM, dramatically improving I/O performance
for repeated accesses. The page cache is unified—it caches all file data regardless
of whether it's accessed via read()/write() or mmap().

Each file's cached pages are managed through an `address_space` structure embedded
in the inode. Pages are stored in an XArray data structure indexed by file offset,
allowing O(log n) lookup. The page cache uses a "lazy" approach: pages are only
read from disk on first access (demand paging), and dirty pages are written back
asynchronously by background writeback threads. This decoupling of application I/O
from disk I/O is fundamental to Linux's I/O performance.

### Key Source Files
- `include/linux/fs.h:500-520` - struct address_space
- `include/linux/fs.h:432-474` - struct address_space_operations
- `include/linux/pagemap.h` - Page cache API
- `mm/filemap.c` - Core implementation
- `mm/readahead.c` - Read-ahead

### 6.1 struct address_space

The address_space structure manages the page cache for a single file (or other
backing store like a block device). The `i_pages` XArray stores cached folios
indexed by file offset. The `nrpages` field tracks the total cached pages. The
`a_ops` pointer provides the interface to the filesystem for reading/writing
pages. The `i_mmap` tree tracks all memory mappings of this file. Each inode
has an embedded address_space (`i_data`), with `i_mapping` pointing to it.

```c
struct address_space {
    struct inode        *host;          /* Owner inode */
    struct xarray       i_pages;        /* Cached pages (indexed by offset) */
    struct rw_semaphore invalidate_lock;/* Invalidation vs page fault */
    gfp_t               gfp_mask;       /* Allocation flags */
    atomic_t            i_mmap_writable;/* Writable mapping count */
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
    atomic_t            nr_thps;        /* Transparent huge pages */
#endif
    struct rb_root_cached i_mmap;       /* Memory mapping tree */
    unsigned long       nrpages;        /* Number of cached pages */
    pgoff_t             writeback_index;/* Writeback start hint */
    const struct address_space_operations *a_ops;
    unsigned long       flags;          /* Error flags (AS_*) */
    errseq_t            wb_err;         /* Writeback error tracking */
    spinlock_t          i_private_lock;
    struct list_head    i_private_list;
    struct rw_semaphore i_mmap_rwsem;   /* Protects i_mmap */
    void *              i_private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;
```

### 6.2 struct address_space_operations

The address_space_operations table defines how pages are read from and written
to the backing store. The `read_folio` callback reads a single page from disk.
The `readahead` callback handles bulk read-ahead. The `writepage`/`writepages`
callbacks write dirty pages back. The `write_begin`/`write_end` pair bracket
buffered write operations. The `direct_IO` callback bypasses the page cache.
The `dirty_folio` callback is called when a page becomes dirty.

```c
struct address_space_operations {
    /* Write operations */
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*writepages)(struct address_space *, struct writeback_control *);

    /* Read operations */
    int (*read_folio)(struct file *, struct folio *);
    void (*readahead)(struct readahead_control *);

    /* Dirty tracking */
    bool (*dirty_folio)(struct address_space *, struct folio *);

    /* Buffered write support */
    int (*write_begin)(struct file *, struct address_space *,
                       loff_t pos, unsigned len,
                       struct folio **foliop, void **fsdata);
    int (*write_end)(struct file *, struct address_space *,
                     loff_t pos, unsigned len, unsigned copied,
                     struct folio *folio, void *fsdata);

    /* Block mapping */
    sector_t (*bmap)(struct address_space *, sector_t);

    /* Page lifecycle */
    void (*invalidate_folio)(struct folio *, size_t offset, size_t len);
    bool (*release_folio)(struct folio *, gfp_t);
    void (*free_folio)(struct folio *);

    /* Direct I/O */
    ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *);

    /* Migration */
    int (*migrate_folio)(struct address_space *, struct folio *dst,
                         struct folio *src, enum migrate_mode);

    /* Writeback support */
    int (*launder_folio)(struct folio *);
    bool (*is_partially_uptodate)(struct folio *, size_t from, size_t count);
    void (*is_dirty_writeback)(struct folio *, bool *, bool *);
    int (*error_remove_folio)(struct address_space *, struct folio *);

    /* Swap support */
    int (*swap_activate)(struct swap_info_struct *, struct file *, sector_t *);
    void (*swap_deactivate)(struct file *);
    int (*swap_rw)(struct kiocb *, struct iov_iter *);
};
```

### 6.3 XArray Storage

The page cache uses an XArray (eXtensible Array) to store folios indexed by
page offset. The XArray replaced the older radix tree and provides better
performance for sparse arrays with RCU-safe operations. It supports "marks"
(tags) to efficiently find pages in specific states without scanning all entries.
The XArray automatically manages memory for internal nodes as the array grows
and shrinks.

```c
struct xarray {
    spinlock_t      xa_lock;    /* Protects array contents */
    gfp_t           xa_flags;   /* Allocation/behavior flags */
    void __rcu *    xa_head;    /* Root pointer (RCU-protected) */
};
```

**XArray Marks (for page cache):**
- `PAGECACHE_TAG_DIRTY` (XA_MARK_0) - Folio needs writeback
- `PAGECACHE_TAG_WRITEBACK` (XA_MARK_1) - Writeback in progress
- `PAGECACHE_TAG_TOWRITE` (XA_MARK_2) - Tagged for current writeback pass

### 6.4 Page Cache Lookup

```c
/* FGP (Find-or-Get Page) flags */
#define FGP_ACCESSED    0x00000001  /* Mark accessed */
#define FGP_LOCK        0x00000002  /* Return locked */
#define FGP_CREAT       0x00000004  /* Create if not found */
#define FGP_WRITE       0x00000008  /* For write access */
#define FGP_NOFS        0x00000010  /* No filesystem operations */
#define FGP_NOWAIT      0x00000020  /* Non-blocking */
#define FGP_FOR_MMAP    0x00000040  /* For mmap */
#define FGP_STABLE      0x00000080  /* Wait for writeback */
#define FGP_WRITEBEGIN  (FGP_LOCK | FGP_WRITE | FGP_CREAT | FGP_STABLE)

/* Main lookup function */
struct folio *__filemap_get_folio(struct address_space *mapping,
                                   pgoff_t index,
                                   fgf_t fgp_flags,
                                   gfp_t gfp);

/* Convenience wrapper */
static inline struct folio *filemap_get_folio(struct address_space *mapping,
                                               pgoff_t index)
{
    return __filemap_get_folio(mapping, index, 0, 0);
}
```

### 6.5 Page Cache Insertion and Removal

```c
/* Add folio to page cache */
int filemap_add_folio(struct address_space *mapping,
                      struct folio *folio,
                      pgoff_t index,
                      gfp_t gfp);

/* Remove folio from page cache */
void filemap_remove_folio(struct folio *folio);

/* Truncate pages in range */
void truncate_inode_pages_range(struct address_space *mapping,
                                loff_t lstart, loff_t lend);
```

### 6.6 Read-ahead

Read-ahead improves sequential read performance by prefetching pages before
they're requested. The `readahead_control` structure is passed to the filesystem's
readahead callback with information about which pages to read. The `file_ra_state`
structure (stored in struct file) tracks the read-ahead window for each open file,
allowing the algorithm to adapt to access patterns. Read-ahead is triggered
automatically on sequential access and can be tuned via fadvise().

```c
struct readahead_control {
    struct file *file;
    struct address_space *mapping;
    struct file_ra_state *ra;
    pgoff_t _index;
    unsigned int _nr_pages;
    unsigned int _batch_count;
    bool _workingset;
    unsigned long _pflags;
};

struct file_ra_state {
    pgoff_t start;           /* Where readahead started */
    unsigned int size;       /* Current window size */
    unsigned int async_size; /* Async trigger size */
    unsigned int ra_pages;   /* Maximum readahead */
    unsigned int mmap_miss;  /* mmap miss counter */
    loff_t prev_pos;         /* Previous read position */
};

/* Core readahead function */
void page_cache_ra_unbounded(struct readahead_control *ractl,
                             unsigned long nr_to_read,
                             unsigned long lookahead_size);

/* Force immediate readahead */
void force_page_cache_ra(struct readahead_control *ractl,
                         unsigned long nr_to_read);

/* Sync readahead (cache miss) */
void page_cache_sync_readahead(struct address_space *mapping,
                               struct file_ra_state *ra,
                               struct file *file,
                               pgoff_t index,
                               unsigned long req_count);

/* Async readahead (hit on marked page) */
void page_cache_async_ra(struct readahead_control *ractl,
                         struct folio *folio,
                         unsigned long req_count);
```

### 6.7 Dirty Page Tracking and Writeback

```c
/* Mark folio dirty */
bool folio_mark_dirty(struct folio *folio);
bool filemap_dirty_folio(struct address_space *mapping, struct folio *folio);

/* Writeback */
int do_writepages(struct address_space *mapping,
                  struct writeback_control *wbc);

/* Write and wait for range */
int filemap_write_and_wait_range(struct address_space *mapping,
                                  loff_t lstart, loff_t lend);

/* Wait for folio writeback */
void folio_wait_writeback(struct folio *folio);
```

### 6.8 Memory-Mapped File Support

```c
/* Page fault handler */
vm_fault_t filemap_fault(struct vm_fault *vmf);

/* Map pages around fault (fault-around) */
vm_fault_t filemap_map_pages(struct vm_fault *vmf,
                              pgoff_t start_pgoff,
                              pgoff_t end_pgoff);

/* Write fault (COW or dirty) */
vm_fault_t filemap_page_mkwrite(struct vm_fault *vmf);

/* Standard vm_operations */
const struct vm_operations_struct generic_file_vm_ops = {
    .fault          = filemap_fault,
    .map_pages      = filemap_map_pages,
    .page_mkwrite   = filemap_page_mkwrite,
};
```

---

## 7. VFS Path Walking

Path walking (or pathname resolution) is the process of translating a pathname
string like "/home/user/file.txt" into the corresponding filesystem object. This
is one of the most frequently executed code paths in the kernel since nearly every
file operation begins with a path lookup. The implementation in `fs/namei.c` is
highly optimized for performance.

The path walker processes each component of the path sequentially, consulting
the dentry cache for cached lookups and falling back to the filesystem's lookup
method for cache misses. Modern Linux uses a two-phase approach: RCU-walk for
the fast path (lockless traversal using RCU and sequence counters) and ref-walk
for the slow path (taking actual reference counts and locks). The path walker
also handles complexities like symbolic links, mount point traversal, and the
various scoping options introduced with openat2().

### Key Source Files
- `fs/namei.c` - Path walking implementation
- `include/linux/namei.h` - LOOKUP_* flags and API

### 7.1 struct nameidata

The nameidata structure is the central state for path walking operations. It
tracks the current position (`path`), the last component being looked up (`last`),
lookup flags and state, and a stack for following nested symlinks. The `seq`,
`m_seq`, and `r_seq` fields are sequence counters for RCU-walk validation.
This structure is allocated on the stack and passed through all path walking
functions. It's not directly visible to filesystems—they interact through
the dentry and inode passed to their lookup callbacks.

```c
struct nameidata {
    struct path     path;           /* Current position */
    struct qstr     last;           /* Last component name */
    struct path     root;           /* Root for absolute/scoped lookups */
    struct inode    *inode;         /* path.dentry.d_inode (cached) */
    unsigned int    flags, state;   /* LOOKUP_* and ND_* flags */
    unsigned        seq, next_seq, m_seq, r_seq;  /* RCU sequence numbers */
    int             last_type;      /* LAST_NORM, LAST_DOT, etc. */
    unsigned        depth;          /* Symlink nesting depth */
    int             total_link_count;
    struct saved {
        struct path link;
        struct delayed_call done;
        const char *name;
        unsigned seq;
    } *stack, internal[EMBEDDED_LEVELS];
    struct filename *name;
    const char      *pathname;
    struct nameidata *saved;
    unsigned        root_seq;
    int             dfd;
    vfsuid_t        dir_vfsuid;
    umode_t         dir_mode;
} __randomize_layout;
```

### 7.2 LOOKUP Flags

```c
/* Behavior modifiers */
#define LOOKUP_FOLLOW       0x0001  /* Follow final symlink */
#define LOOKUP_DIRECTORY    0x0002  /* Require directory */
#define LOOKUP_AUTOMOUNT    0x0004  /* Force automount */
#define LOOKUP_EMPTY        0x4000  /* Accept empty path */
#define LOOKUP_DOWN         0x8000  /* Follow mounts at start */
#define LOOKUP_MOUNTPOINT   0x0080  /* Follow mounts at end */

/* Validation */
#define LOOKUP_REVAL        0x0020  /* Force revalidation */
#define LOOKUP_RCU          0x0040  /* RCU-walk mode */

/* Intent flags */
#define LOOKUP_OPEN         0x0100  /* For open() */
#define LOOKUP_CREATE       0x0200  /* For creation */
#define LOOKUP_EXCL         0x0400  /* Exclusive create */
#define LOOKUP_RENAME_TARGET 0x0800 /* Rename target */

/* Parent lookup */
#define LOOKUP_PARENT       0x0010  /* Stop at parent */

/* Scoping (openat2) */
#define LOOKUP_NO_SYMLINKS  0x010000  /* Block all symlinks */
#define LOOKUP_NO_MAGICLINKS 0x020000 /* Block magic links */
#define LOOKUP_NO_XDEV      0x040000  /* Block mount crossing */
#define LOOKUP_BENEATH      0x080000  /* Stay beneath start */
#define LOOKUP_IN_ROOT      0x100000  /* Treat as chroot */
#define LOOKUP_CACHED       0x200000  /* Cache-only lookup */
```

### 7.3 RCU-Walk vs Ref-Walk

**RCU-Walk (Fast Path):**
- No locks taken
- Uses RCU read-side critical section
- Validates via sequence counters
- Falls back to ref-walk on contention

**Ref-Walk (Slow Path):**
- Takes dentry/mount reference counts
- May take inode locks
- Always succeeds (no fallback needed)

```c
/* Transition from RCU to ref-walk */
static bool try_to_unlazy(struct nameidata *nd);

/* Exit RCU mode */
static void leave_rcu(struct nameidata *nd)
{
    nd->flags &= ~LOOKUP_RCU;
    nd->seq = nd->next_seq = 0;
    rcu_read_unlock();
}
```

### 7.4 Path Walking Flow

```
do_filp_open()
    │
    ├── set_nameidata()
    └── path_openat() with LOOKUP_RCU
        │
        ├── path_init()
        │   ├── Determine starting point (/, cwd, dirfd)
        │   ├── rcu_read_lock() for RCU mode
        │   └── Read mount_lock, rename_lock sequences
        │
        ├── link_path_walk() - Walk each component
        │   │
        │   └── For each component:
        │       ├── may_lookup() - Check MAY_EXEC permission
        │       ├── hash_name() - Parse and hash component
        │       └── walk_component():
        │           ├── handle_dots() for "." and ".."
        │           ├── lookup_fast() - dcache lookup
        │           │   ├── __d_lookup_rcu() in RCU mode
        │           │   └── __d_lookup() in ref mode
        │           ├── lookup_slow() - filesystem lookup
        │           │   ├── inode_lock_shared()
        │           │   └── i_op->lookup()
        │           └── step_into():
        │               ├── handle_mounts() - traverse mounts
        │               └── pick_link() - handle symlinks
        │
        ├── open_last_lookups() - Handle final component
        │
        ├── do_open() - Actually open the file
        │
        └── complete_walk() - Finalize and validate
            │
            ├── try_to_unlazy() - Exit RCU if needed
            └── Validate scoped lookup constraints
```

### 7.5 Symlink Following

```c
/* Symlink limits */
#define MAX_NESTED_LINKS    8   /* Stack depth */
#define MAXSYMLINKS         40  /* Total symlinks in lookup */

/* Begin following symlink */
static const char *pick_link(struct nameidata *nd,
                             struct path *link,
                             struct inode *inode,
                             int flags)
{
    /* Check limits */
    reserve_stack(nd);

    /* Push to stack */
    nd->stack[nd->depth++] = saved;

    /* Security check for trailing symlinks */
    if (WALK_TRAILING)
        may_follow_link(nd, inode);

    /* Get symlink target */
    target = inode->i_op->get_link(dentry, inode, &done);

    /* Handle absolute symlinks */
    if (*target == '/')
        nd_jump_root(nd);

    return target;
}
```

### 7.6 Mount Point Traversal

```c
/* Follow mounts in RCU mode */
static bool __follow_mount_rcu(struct nameidata *nd, struct path *path)
{
    while (DCACHE_MANAGED_DENTRY) {
        if (DCACHE_MANAGE_TRANSIT)
            d_op->d_manage();

        if (DCACHE_MOUNTED) {
            struct mount *mounted = __lookup_mnt();
            if (mounted) {
                path->mnt = &mounted->mnt;
                path->dentry = mounted->mnt.mnt_root;
                nd->state |= ND_JUMPED;
            }
        }

        if (DCACHE_NEED_AUTOMOUNT)
            return false;  /* Need ref-walk for automount */
    }
    return true;
}
```

---

## 8. Buffered I/O Path

Buffered I/O is the default mode for file read and write operations in Linux.
Unlike direct I/O which bypasses the page cache, buffered I/O uses the page cache
as an intermediary between userspace and the storage device. This provides several
benefits: read-ahead can prefetch data before it's needed, writes can be batched
and written asynchronously, and frequently accessed data stays in memory.

The read path checks the page cache first and triggers read-ahead if sequential
access is detected. On cache miss, pages are read from disk and added to the cache.
The write path copies data to page cache pages, marks them dirty, and returns
immediately—the actual disk write happens later via the background writeback mechanism.
This asynchronous writeback is controlled by dirty page limits and the `pdflush`/`flush`
kernel threads. Both read and write operations are implemented through the
`address_space_operations` callbacks provided by the filesystem.

### 8.1 Read Path

The buffered read path uses `generic_file_read_iter()` as the common entry point.
It first checks for direct I/O requests (O_DIRECT), which bypass the page cache.
For buffered reads, `filemap_read()` iterates through the requested range,
fetching pages from the cache or triggering read-ahead to bring them in from disk.
The `kiocb` structure carries per-I/O state like position and flags, while
`iov_iter` describes the destination buffer(s).

```c
ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    if (iocb->ki_flags & IOCB_DIRECT) {
        /* Direct I/O path */
        kiocb_write_and_wait();  /* Flush dirty pages */
        file_update_time();
        return a_ops->direct_IO(iocb, iter);
    }

    /* Buffered I/O path */
    return filemap_read(iocb, iter, already_read);
}

ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter, ssize_t already_read)
{
    while (count > 0) {
        /* Get pages from cache, triggering readahead */
        filemap_get_pages(iocb, iter, &fbatch);

        /* Copy data to userspace */
        for (each folio in batch) {
            copy_folio_to_iter(folio, offset, bytes, iter);
        }
    }

    file_accessed(file);  /* Update atime */
    return bytes_read;
}
```

### 8.2 Write Path

The buffered write path copies data to page cache pages and marks them dirty.
The `generic_file_write_iter()` function takes the inode lock for serialization,
then calls the core write logic. The `write_begin`/`write_end` callbacks bracket
each page write, allowing the filesystem to allocate blocks and handle partial
pages. The `balance_dirty_pages_ratelimited()` call throttles writers when too
many pages are dirty, preventing memory exhaustion. Actual disk writes happen
asynchronously via writeback unless O_SYNC/O_DSYNC is specified.

```c
ssize_t generic_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    inode_lock(inode);

    ret = __generic_file_write_iter(iocb, from);

    inode_unlock(inode);

    if (O_SYNC || O_DSYNC)
        generic_write_sync(iocb, ret);

    return ret;
}

ssize_t generic_perform_write(struct kiocb *iocb, struct iov_iter *i)
{
    while (count > 0) {
        /* Throttle based on dirty page limits */
        balance_dirty_pages_ratelimited(mapping);

        /* Prepare folio for writing */
        a_ops->write_begin(file, mapping, pos, len, &folio, &fsdata);

        /* Copy from userspace */
        bytes = copy_folio_from_iter_atomic(folio, offset, count, i);

        /* Complete the write */
        a_ops->write_end(file, mapping, pos, len, bytes, folio, fsdata);
    }

    return written;
}
```

### 8.3 Writeback Path

The writeback path flushes dirty pages from the page cache to disk. It's driven
by the `writeback_control` structure which specifies what to write (range, sync
mode, number of pages) and tracks progress. Writeback can be triggered by sync
operations, memory pressure, or periodic flushing by the `flush` kernel threads.
The filesystem's `writepages` callback handles the actual I/O, typically batching
multiple pages into larger requests for efficiency.

```c
int do_writepages(struct address_space *mapping, struct writeback_control *wbc)
{
    if (mapping->a_ops->writepages)
        return mapping->a_ops->writepages(mapping, wbc);

    return writeback_use_writepage(mapping, wbc);
}

struct folio *writeback_iter(struct address_space *mapping,
                             struct writeback_control *wbc,
                             struct folio *folio,
                             int *error)
{
    /* Tag dirty pages for this writeback pass */
    if (first_call)
        tag_pages_for_writeback(mapping, index, end);

    /* Find next tagged folio */
    folio = writeback_get_folio(mapping, wbc);

    if (!folio)
        return NULL;  /* Done */

    return folio;
}
```

---

## Summary

The Linux VFS provides a powerful abstraction layer that:

1. **Unifies filesystem access** through common data structures (superblock, inode, dentry, file)
2. **Caches filesystem metadata** in the dentry cache for fast path lookup
3. **Caches file data** in the page cache to minimize disk I/O
4. **Supports multiple access patterns** (buffered, direct, memory-mapped)
5. **Enables high-performance path resolution** through RCU-walk optimization
6. **Provides consistent semantics** across diverse filesystem implementations

The key insight is that VFS structures form a hierarchy:
- `super_block` owns the filesystem
- `inode` represents persistent objects
- `dentry` provides the naming layer
- `file` represents open instances
- `address_space` manages cached data

Understanding these structures and their interactions is essential for kernel
filesystem development, performance optimization, and debugging filesystem
issues.
