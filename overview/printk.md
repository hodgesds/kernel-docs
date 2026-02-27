# Linux Kernel printk / Logging Subsystem

## Overview

The Linux kernel logging subsystem is built around `printk()`, a printf-like
function callable from virtually any context -- process, interrupt, NMI, even
early boot before the memory allocator is running. At its core, `printk()`
formats a message, stores it in a lockless ring buffer, and then dispatches it
to registered console drivers. User space reads the log through `/dev/kmsg`
(structured) or the `syslog(2)` syscall (legacy). The entire implementation
lives under `kernel/printk/`, with key headers in `include/linux/printk.h`,
`include/linux/console.h`, and `include/linux/kern_levels.h`.

---

## 1. Architecture Overview

The data flow of a kernel log message follows this path:

```
  caller (pr_info, dev_err, printk, ...)
    |
    v
  _printk() / vprintk_emit()          kernel/printk/printk.c:2514
    |
    v
  vprintk_store()                      kernel/printk/printk.c:2377
    |  - formats text via vsnprintf
    |  - reserves descriptor + data block in ring buffer
    |  - calls prb_reserve() / prb_final_commit()
    v
  printk_ringbuffer (prb)              kernel/printk/printk.c:561
    |
    +---> console drivers (write / write_atomic / write_thread)
    |       dispatched via legacy console_unlock() loop or nbcon kthreads
    |
    +---> /dev/kmsg readers (devkmsg_read)
    |       kernel/printk/printk.c:838
    |
    +---> syslog(2) / klogd
    |
    +---> trace_console() tracepoint
            kernel/printk/printk.c:2371
```

Key global state at `kernel/printk/printk.c`:

- `prb` (line 561) -- pointer to the active `printk_ringbuffer`, initially
  the statically allocated `printk_rb_static`, later replaced by a
  dynamically allocated buffer in `setup_log_buf()`.
- `console_list` (line 97) -- SRCU-protected `hlist_head` of registered
  `struct console` instances.
- `console_sem` (line 96) -- semaphore serializing legacy console output.
- `console_mutex` (line 90) -- mutex protecting console list mutations.

---

## 2. Core Data Structures

### `struct printk_ringbuffer`

`kernel/printk/printk_ringbuffer.h:90`

The top-level ring buffer object, composed of two sub-rings:

```c
struct printk_ringbuffer {
    struct prb_desc_ring    desc_ring;       // descriptor ring (meta-data)
    struct prb_data_ring    text_data_ring;  // text data ring (message bytes)
    atomic_long_t           fail;            // count of failed prb_reserve() calls
};
```

### `struct prb_desc_ring`

`kernel/printk/printk_ringbuffer.h:75`

Ring of descriptors, each tracking one log record:

```c
struct prb_desc_ring {
    unsigned int        count_bits;         // log2(number of descriptors)
    struct prb_desc     *descs;             // array of descriptors
    struct printk_info  *infos;             // parallel array of record metadata
    atomic_long_t       head_id;            // newest descriptor ID
    atomic_long_t       tail_id;            // oldest descriptor ID
    atomic_long_t       last_finalized_seq; // last fully committed sequence
};
```

### `struct prb_desc`

`kernel/printk/printk_ringbuffer.h:61`

A single descriptor -- the synchronization pivot for lockless operation:

```c
struct prb_desc {
    atomic_long_t           state_var;      // ID + state (2 high bits)
    struct prb_data_blk_lpos text_blk_lpos; // logical position of text data
};
```

The `state_var` encodes both the descriptor ID and its state in the top 2 bits.
The state machine has four states (`kernel/printk/printk_ringbuffer.h:116`):

```c
enum desc_state {
    desc_miss      = -1, // ID mismatch (pseudo state)
    desc_reserved  = 0x0, // in use by writer
    desc_committed = 0x1, // committed, can be reopened
    desc_finalized = 0x2, // complete, available for reading
    desc_reusable  = 0x3, // free, ready for recycling
};
```

### `struct printk_info`

`kernel/printk/printk_ringbuffer.h:18`

Per-record metadata, stored in a parallel array indexed identically to
descriptors:

```c
struct printk_info {
    u64     seq;            // sequence number
    u64     ts_nsec;        // timestamp in nanoseconds
    u16     text_len;       // length of text message
    u8      facility;       // syslog facility
    u8      flags:5;        // internal record flags (LOG_CONT, LOG_NEWLINE, etc.)
    u8      level:3;        // syslog level (0-7)
    u32     caller_id;      // thread id or processor id
    struct dev_printk_info dev_info; // subsystem + device name
};
```

### `struct console`

`include/linux/console.h:335`

The descriptor for every registered console driver:

```c
struct console {
    char            name[16];
    void            (*write)(struct console *co, const char *s, unsigned int count);
    int             (*read)(struct console *co, char *s, unsigned int count);
    struct tty_driver *(*device)(struct console *co, int *index);
    void            (*unblank)(void);
    int             (*setup)(struct console *co, char *options);
    int             (*exit)(struct console *co);
    int             (*match)(struct console *co, char *name, int idx, char *options);
    short           flags;      // CON_ENABLED, CON_BOOT, CON_NBCON, ...
    short           index;
    int             cflag;
    u64             seq;        // next ringbuffer record to print (legacy consoles)
    unsigned long   dropped;    // unreported dropped records
    void            *data;
    struct hlist_node node;     // console_list linkage
    int             level;      // per-console loglevel
    struct device   *classdev;

    /* nbcon-specific members */
    void (*write_atomic)(struct console *con, struct nbcon_write_context *wctxt);
    void (*write_thread)(struct console *con, struct nbcon_write_context *wctxt);
    void (*device_lock)(struct console *con, unsigned long *flags);
    void (*device_unlock)(struct console *con, unsigned long flags);
    atomic_t        nbcon_state;
    atomic_long_t   nbcon_seq;
    struct nbcon_context nbcon_device_ctxt;
    atomic_long_t   nbcon_prev_seq;
    struct printk_buffers *pbufs;
    struct task_struct    *kthread;
    struct rcuwait        rcuwait;
    struct irq_work       irq_work;
};
```

Registered via `register_console()` (`kernel/printk/printk.c:4315`),
unregistered via `unregister_console()` (`kernel/printk/printk.c:4584`).

---

## 3. The Ring Buffer -- Lockless Design

`kernel/printk/printk_ringbuffer.c`

The printk ring buffer is designed to be fully lockless, allowing writers and
readers to operate concurrently from any context, including NMI. The design
documentation at the top of `printk_ringbuffer.c` (lines 11-300) describes
the approach in detail.

### Two-ring architecture

The ring buffer consists of two sub-rings:

1. **Descriptor ring** (`prb_desc_ring`) -- fixed-size array of `prb_desc`
   entries. Each descriptor contains an `atomic_long_t state_var` that
   encodes the descriptor ID and a 2-bit state field in the high bits.
   A parallel `printk_info` array stores record metadata.

2. **Text data ring** (`prb_data_ring`) -- byte array where message text is
   stored. Each data block starts with an `unsigned long` ID that maps back
   to a descriptor, followed by the text content. The ring tracks
   `head_lpos` and `tail_lpos` as atomic logical positions.

### Descriptor state machine

All synchronization pivots on atomic transitions of `state_var`:

```
  reserved  --prb_commit()-->   committed  --finalize-->  finalized
      ^                             |                         |
      |                             v                         v
      +--- prb_reserve() <--- reusable <-------- recycled ----+
```

- `prb_reserve()` atomically transitions a descriptor from `reusable` to
  `reserved`, pushing the head forward.
- `prb_commit()` transitions `reserved` to `committed` -- data is consistent
  but may be reopened (e.g., for `KERN_CONT` continuation).
- `prb_final_commit()` transitions directly to `finalized` -- data is
  immutable and available to readers.
- When the ring wraps, tail descriptors are transitioned to `reusable` to
  make room.

### Writer protocol

```c
struct prb_reserved_entry e;
struct printk_record r;

prb_rec_init_wr(&r, text_len);
if (prb_reserve(&e, prb, &r)) {
    // fill r.text_buf and r.info
    prb_final_commit(&e);
}
```

### Reader protocol

```c
struct printk_info info;
struct printk_record r;
char text_buf[1024];
u64 seq;

prb_rec_init_rd(&r, &info, text_buf, sizeof(text_buf));
prb_for_each_record(0, prb, &seq, &r) {
    // process info.seq, info.ts_nsec, text_buf
}
```

### Static allocation

The initial ring buffer is statically defined at `kernel/printk/printk.c:556`:

```c
#define PRB_AVGBITS 5  /* 32 character average length */
_DEFINE_PRINTKRB(printk_rb_static, CONFIG_LOG_BUF_SHIFT - PRB_AVGBITS,
                 PRB_AVGBITS, &__log_buf[0]);
```

The static `__log_buf` (line 542) is `1 << CONFIG_LOG_BUF_SHIFT` bytes
(typically 128 KiB to 1 MiB). `setup_log_buf()` (line 1187) may later
allocate a larger buffer from memblock and swap the `prb` pointer to
`printk_rb_dynamic`.

---

## 4. printk API Variants

### The core `printk()` macro

`include/linux/printk.h:509`

```c
#define printk(fmt, ...) printk_index_wrap(_printk, fmt, ##__VA_ARGS__)
```

This wraps `_printk()`, which calls `vprintk_emit()`. The `printk_index_wrap`
macro also generates a compile-time printk index entry (under
`CONFIG_PRINTK_INDEX`) for tooling to enumerate all printk call sites.

### `pr_*()` family

`include/linux/printk.h:521-594`

Convenience macros that prepend a log level and apply `pr_fmt()`:

| Macro | Level | Numeric |
|-------|-------|---------|
| `pr_emerg()` | `KERN_EMERG` | 0 |
| `pr_alert()` | `KERN_ALERT` | 1 |
| `pr_crit()` | `KERN_CRIT` | 2 |
| `pr_err()` | `KERN_ERR` | 3 |
| `pr_warn()` | `KERN_WARNING` | 4 |
| `pr_notice()` | `KERN_NOTICE` | 5 |
| `pr_info()` | `KERN_INFO` | 6 |
| `pr_debug()` | `KERN_DEBUG` | 7 |
| `pr_cont()` | `KERN_CONT` | continuation |

The `pr_fmt()` macro (`include/linux/printk.h:398`) defaults to the identity
but is commonly overridden per-file:

```c
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt
```

### `dev_*()` family

`include/linux/dev_printk.h:148-171`

Device-aware logging that automatically includes device identity:

```c
#define dev_err(dev, fmt, ...) \
    dev_printk_index_wrap(_dev_err, KERN_ERR, dev, dev_fmt(fmt), ##__VA_ARGS__)
```

These call `dev_vprintk_emit()`, which fills a `struct dev_printk_info` with
the device's subsystem and bus-id, then calls `vprintk_emit()`. The
information is stored in `printk_info.dev_info` and exposed through
`/dev/kmsg`. Similarly: `dev_warn()`, `dev_info()`, `dev_dbg()`,
`dev_notice()`, `dev_crit()`, `dev_alert()`, `dev_emerg()`.

### One-time variants

`include/linux/printk.h:647-689`

```c
#define printk_once(fmt, ...)  DO_ONCE_LITE(printk, fmt, ##__VA_ARGS__)
#define pr_err_once(fmt, ...)  printk_once(KERN_ERR pr_fmt(fmt), ##__VA_ARGS__)
```

Prints the message only on the first invocation using a static flag.

### Deferred printk

`include/linux/printk.h:510-511`

```c
#define printk_deferred(fmt, ...) \
    printk_index_wrap(_printk_deferred, fmt, ##__VA_ARGS__)
```

For use in contexts (scheduler, timekeeping) where calling `up()` on the
console semaphore would be unsafe. Stores the record but defers console
flushing to an IRQ work context.

---

## 5. Console Drivers -- Legacy vs nbcon

### Legacy consoles

Legacy consoles implement the `write()` callback in `struct console`.
All legacy console output is serialized through the `console_sem` semaphore
(`kernel/printk/printk.c:96`). The typical flow is:

1. `vprintk_emit()` stores the record and attempts `console_trylock()`.
2. If the lock is acquired, `console_unlock()` iterates all consoles,
   calling `con->write()` for each pending record.
3. If the lock is not acquired, the current holder will pick up new records
   when it calls `console_unlock()`.

This design has a well-known problem: under high printk load, the
`console_unlock()` holder can be stuck printing for a long time, starving
the original task.

### nbcon (non-blocking console)

`kernel/printk/nbcon.c`

The nbcon framework was introduced to replace the legacy console_lock model.
An nbcon console sets `CON_NBCON` in its flags and provides:

- `write_atomic()` -- outputs text from any context, including NMI. Uses
  lockless ownership via `nbcon_state`.
- `write_thread()` -- outputs text from a dedicated per-console kthread,
  using the driver's `device_lock()` for serialization.
- `device_lock()` / `device_unlock()` -- driver-provided synchronization
  callbacks for thread-context printing.

#### Ownership model

`include/linux/console.h:220-231`

```c
struct nbcon_state {
    union {
        unsigned int atom;
        struct {
            unsigned int prio           :  2; // current owner's priority
            unsigned int req_prio       :  2; // handover request priority
            unsigned int unsafe         :  1; // in non-takeover-safe region
            unsigned int unsafe_takeover:  1; // hostile takeover occurred
            unsigned int cpu            : 24; // owning CPU
        };
    };
};
```

The `nbcon_state` is atomically updated via `cmpxchg`. Three acquisition
methods exist (`kernel/printk/nbcon.c:59-117`):

1. **Direct acquire** -- console is unowned or owned at lower priority and
   in a safe state.
2. **Friendly handover** -- `req_prio` is set; the current owner sees it
   upon exiting an unsafe section and voluntarily releases.
3. **Hostile takeover** -- only during `panic()` as a last resort, even if
   the console is in an unsafe state.

#### Priority levels

`include/linux/console.h:253-259`

```c
enum nbcon_prio {
    NBCON_PRIO_NONE = 0,    // not owned
    NBCON_PRIO_NORMAL,      // normal printing (kthread)
    NBCON_PRIO_EMERGENCY,   // WARN/OOPS context
    NBCON_PRIO_PANIC,       // panic() context
    NBCON_PRIO_MAX,
};
```

#### Console flags

`include/linux/console.h:190-200`

```c
enum cons_flags {
    CON_PRINTBUFFER  = BIT(0), // replay buffered messages on register
    CON_CONSDEV      = BIT(1), // backing /dev/console
    CON_ENABLED      = BIT(2), // allowed to print
    CON_BOOT         = BIT(3), // early boot console, auto-unregistered
    CON_ANYTIME      = BIT(4), // usable even on offline CPUs
    CON_BRL          = BIT(5), // braille device
    CON_EXTENDED     = BIT(6), // supports extended /dev/kmsg format
    CON_SUSPENDED    = BIT(7), // suspended (e.g., during PM)
    CON_NBCON        = BIT(8), // non-blocking console
};
```

---

## 6. /dev/kmsg Interface and dmesg

### /dev/kmsg

`kernel/printk/printk.c:1000-1007`

```c
const struct file_operations kmsg_fops = {
    .open      = devkmsg_open,
    .read      = devkmsg_read,
    .write_iter = devkmsg_write,
    .llseek    = devkmsg_llseek,
    .poll      = devkmsg_poll,
    .release   = devkmsg_release,
};
```

Each open file descriptor is backed by a `struct devkmsg_user`
(`kernel/printk/printk.c:756`):

```c
struct devkmsg_user {
    atomic64_t          seq;    // current read position
    struct ratelimit_state rs;  // write ratelimit
    struct mutex        lock;   // serializes reads
    struct printk_buffers pbufs; // formatting buffers
};
```

**Reading**: `devkmsg_read()` (line 838) calls `printk_get_next_message()`
with the user's current sequence number. The message is formatted in the
extended format:
```
<priority>,<seq>,<timestamp>,<flags>;<message text>\n
```
If the reader falls behind and records are overwritten, `read()` returns
`-EPIPE` and the sequence is reset to the oldest available record.
Blocking reads use `wait_event_interruptible()` on `log_wait`.

**Writing**: `devkmsg_write()` (line 776) allows user space to inject
messages into the kernel log (facility `LOG_USER` by default). Writing is
ratelimited by default, controlled by `printk.devkmsg=` boot parameter.

**Seeking**: `devkmsg_llseek()` (line 908) supports:
- `SEEK_SET` -- jump to oldest record
- `SEEK_END` -- jump past newest record
- `SEEK_DATA` -- jump to the record after the last `dmesg -c` clear

### dmesg(1)

The `dmesg` command reads from `/dev/kmsg` (modern) or via `syslog(2)`
syscall (`SYSLOG_ACTION_READ_ALL`). The syslog interface tracks a global
`syslog_seq` position (`kernel/printk/printk.c:2620`) and is serialized
by `syslog_lock`.

Access is controlled by `dmesg_restrict` (`kernel/printk/printk.c:194`):
when set to 1, reading requires `CAP_SYSLOG`.

---

## 7. Log Levels and Dynamic Debug

### Log levels

`include/linux/kern_levels.h`

Eight standard syslog levels encoded as a single ASCII digit after a
`KERN_SOH` (`\001`) byte prefix:

```c
#define KERN_EMERG   KERN_SOH "0"  /* system is unusable */
#define KERN_ALERT   KERN_SOH "1"  /* action must be taken immediately */
#define KERN_CRIT    KERN_SOH "2"  /* critical conditions */
#define KERN_ERR     KERN_SOH "3"  /* error conditions */
#define KERN_WARNING KERN_SOH "4"  /* warning conditions */
#define KERN_NOTICE  KERN_SOH "5"  /* normal but significant condition */
#define KERN_INFO    KERN_SOH "6"  /* informational */
#define KERN_DEBUG   KERN_SOH "7"  /* debug-level messages */
#define KERN_CONT    KERN_SOH "c"  /* continuation of previous line */
```

### Console loglevel

`include/linux/printk.h:65-74`

The four-element `console_printk[]` array controls filtering:

```c
int console_printk[4] = {
    CONSOLE_LOGLEVEL_DEFAULT,   /* console_loglevel */
    MESSAGE_LOGLEVEL_DEFAULT,   /* default_message_loglevel */
    CONSOLE_LOGLEVEL_MIN,       /* minimum_console_loglevel */
    CONSOLE_LOGLEVEL_DEFAULT,   /* default_console_loglevel */
};
```

- `console_loglevel` -- only messages with level < this value are printed to
  the console. Adjustable via `/proc/sys/kernel/printk` or the `loglevel=`
  boot parameter.
- `default_message_loglevel` -- assigned when a `printk()` call omits a
  level prefix (typically 4, `KERN_WARNING`).

Messages are always stored in the ring buffer regardless of console loglevel.

### Per-console loglevel

`struct console` has a `level` field (line 353 of `include/linux/console.h`)
and `per_console_loglevel_is_set()` checks whether a per-console override is
active. This allows individual consoles to filter at different levels.

### Dynamic debug

`include/linux/dynamic_debug.h`

When `CONFIG_DYNAMIC_DEBUG` is enabled, `pr_debug()` and `dev_dbg()` expand
to `dynamic_pr_debug()` / `dynamic_dev_dbg()` instead of being compiled
out. Each call site generates a `struct _ddebug` descriptor:

```c
struct _ddebug {
    const char *modname;
    const char *function;
    const char *filename;
    const char *format;
    unsigned int lineno:18;
    unsigned int class_id:6;
    unsigned int flags:8;
    // optional static_key for jump-label patching
};
```

These descriptors are collected in the `__dyndbg` ELF section. At runtime,
the user controls them through:

```
/sys/kernel/debug/dynamic_debug/control
```

Examples:
```
echo 'file drivers/usb/core/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module snd_hda_intel +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func probe +pflt' > /sys/kernel/debug/dynamic_debug/control
```

Flag bits (`include/linux/dynamic_debug.h:34-40`):
- `p` -- enable `printk()` output
- `f` -- include function name
- `l` -- include line number
- `m` -- include module name
- `t` -- include thread ID
- `s` -- include source filename

When `CONFIG_JUMP_LABEL` is available, dynamic debug uses static keys to
achieve near-zero overhead when disabled.

---

## 8. Rate Limiting and Suppression

### `printk_ratelimited()`

`include/linux/printk.h:696-704`

```c
#define printk_ratelimited(fmt, ...)                    \
({                                                      \
    static DEFINE_RATELIMIT_STATE(_rs,                  \
                  DEFAULT_RATELIMIT_INTERVAL,           \
                  DEFAULT_RATELIMIT_BURST);             \
    if (__ratelimit(&_rs))                              \
        printk(fmt, ##__VA_ARGS__);                     \
})
```

Each call site gets its own static `ratelimit_state` with defaults of
5-second interval and 10-burst count. The full family:
`pr_err_ratelimited()`, `pr_warn_ratelimited()`, `pr_info_ratelimited()`,
`pr_debug_ratelimited()`, `dev_err_ratelimited()`, etc.

### `printk_once()` / `pr_*_once()`

`include/linux/printk.h:647-656`

```c
#define printk_once(fmt, ...) DO_ONCE_LITE(printk, fmt, ##__VA_ARGS__)
```

Uses a per-call-site static boolean to emit the message exactly once. The
`*_once` variants exist for all log levels.

### `/dev/kmsg` write ratelimiting

`kernel/printk/printk.c:127-142`

User-space writes to `/dev/kmsg` are ratelimited by default. The
`printk.devkmsg=` boot parameter controls behavior:

- `ratelimit` (default) -- apply per-fd `ratelimit_state`
- `on` -- allow unlimited writes
- `off` -- reject all user writes (`devkmsg_open` returns `-EPERM`)

Once set via kernel command line, the `DEVKMSG_LOG_MASK_LOCK` bit prevents
runtime changes via sysctl.

### Message suppression

`kernel/printk/printk.c:105`

```c
int __read_mostly suppress_printk;
```

Set after panic to suppress non-essential messages. `vprintk_emit()` checks
this at entry (line 2524). Also, messages from non-panic CPUs are silently
dropped when another CPU is in panic (line 2532).

---

## 9. Early Boot Logging (earlycon)

### `early_printk()`

`kernel/printk/printk.c:2628-2644`

Before a real console is registered, `CONFIG_EARLY_PRINTK` provides
`early_printk()`, which directly invokes `early_console->write()`:

```c
struct console *early_console;

asmlinkage __visible void early_printk(const char *fmt, ...)
{
    va_list ap;
    char buf[512];
    int n;
    if (!early_console)
        return;
    va_start(ap, fmt);
    n = vscnprintf(buf, sizeof(buf), fmt, ap);
    va_end(ap);
    early_console->write(early_console, buf, n);
}
```

This is architecture-specific and used only for the earliest boot output
before the regular printk infrastructure is functional.

### earlycon framework

`include/linux/serial_core.h:1047-1078`

The `earlycon` framework provides a standardized way for serial drivers to
register early consoles:

```c
struct earlycon_device {
    struct console  *con;
    struct uart_port port;
    char  options[32];     // e.g., "115200n8"
    unsigned int baud;
};

struct earlycon_id {
    char  name[15];
    char  name_term;
    char  compatible[128];
    int   (*setup)(struct earlycon_device *, const char *options);
};
```

Drivers register with:

```c
EARLYCON_DECLARE(name, setup_func);
OF_EARLYCON_DECLARE(name, "compatible-string", setup_func);
```

These entries are collected in the `__earlycon_table` linker section. The
kernel matches them against the `earlycon=` boot parameter or the device
tree `stdout-path` / ACPI SPCR table.

### Boot consoles

Consoles registered with `CON_BOOT` flag (`include/linux/console.h:194`)
are automatically unregistered when the first real console driver registers,
unless `keep_bootcon` is specified on the command line. The `CON_PRINTBUFFER`
flag causes a boot console's successor to replay all buffered messages.

### Static log buffer

The initial log buffer is a statically allocated char array
(`kernel/printk/printk.c:542`):

```c
#define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)
static char __log_buf[__LOG_BUF_LEN] __aligned(LOG_ALIGN);
```

`setup_log_buf()` (line 1187) is called during early init to optionally
allocate a larger buffer using `memblock_alloc`. The `log_buf_len=` boot
parameter controls the desired size (max 2 GiB).

---

## 10. Integration with Tracing

### `trace_console()` tracepoint

`kernel/printk/printk.c:2371`

Every message that is stored in the ring buffer triggers the `console`
tracepoint:

```c
trace_console(text, text_len);
```

This tracepoint is defined via `#include <trace/events/printk.h>` (line 58)
and exported at line 76:

```c
EXPORT_TRACEPOINT_SYMBOL_GPL(console);
```

This allows BPF programs and ftrace consumers to observe all printk messages
in real time without polling `/dev/kmsg`.

### printk and ftrace

The kernel provides `trace_printk()` (not to be confused with `printk()`),
which writes directly into the ftrace ring buffer instead of the printk ring
buffer. It is intended for high-frequency debugging inside the kernel where
`printk()` would be too slow. `trace_printk()` messages appear in
`/sys/kernel/tracing/trace` prefixed with the CPU and timestamp.

Important: `trace_printk()` should never be left in production code -- the
kernel prints a loud banner at boot if any `trace_printk()` call sites exist.

### Tracing integration via kprobes / BPF

BPF programs can:

1. Attach to the `console` tracepoint to capture all printk output.
2. Use `bpf_trace_printk()` helper to emit messages into the ftrace ring
   buffer (and `/sys/kernel/tracing/trace_pipe`).
3. Attach kprobes to `vprintk_emit()` or `vprintk_store()` to intercept
   messages before they reach the ring buffer.

### irq_work for deferred flushing

`kernel/printk/internal.h:94`

When printk is called from contexts that cannot directly flush consoles
(scheduler, hardirq with deferred printing), the subsystem uses `irq_work`
to schedule flushing. For nbcon consoles, each console has its own
`irq_work` field (`struct console::irq_work`, `include/linux/console.h:471`)
used to wake its printer kthread from interrupt context.

---

## Key Source Files Reference

| File | Purpose |
|------|---------|
| `kernel/printk/printk.c` | Core printk implementation, syslog, /dev/kmsg, console management |
| `kernel/printk/printk_ringbuffer.c` | Lockless ring buffer implementation |
| `kernel/printk/printk_ringbuffer.h` | Ring buffer data structures and macros |
| `kernel/printk/nbcon.c` | Non-blocking console framework |
| `kernel/printk/internal.h` | Internal definitions, `printk_buffers`, `console_flush_type` |
| `kernel/printk/printk_safe.c` | Safe printk entry for recursion protection |
| `kernel/printk/sysctl.c` | sysctl integration for printk tunables |
| `include/linux/printk.h` | Public printk API, pr_* macros, ratelimit macros |
| `include/linux/console.h` | `struct console`, nbcon state, console flags |
| `include/linux/kern_levels.h` | KERN_EMERG..KERN_DEBUG level definitions |
| `include/linux/dev_printk.h` | dev_err/dev_info/dev_dbg family, `dev_printk_info` |
| `include/linux/dynamic_debug.h` | `struct _ddebug`, dynamic debug flags |
| `include/linux/serial_core.h` | `earlycon_device`, `earlycon_id`, EARLYCON_DECLARE |
