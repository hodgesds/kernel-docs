# Linux Kernel Timer Subsystem Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Jiffies and HZ](#jiffies-and-hz)
3. [Low-Resolution Timers](#low-resolution-timers)
4. [High-Resolution Timers (hrtimers)](#high-resolution-timers-hrtimers)
5. [Clocksource](#clocksource)
6. [Clockevents](#clockevents)
7. [NO_HZ and Tickless Operation](#no_hz-and-tickless-operation)
8. [Sleep and Delay Functions](#sleep-and-delay-functions)
9. [Timekeeping](#timekeeping)

---

## Introduction

The Linux timer subsystem provides infrastructure for:

- Tracking system time
- Scheduling deferred work
- Periodic tick generation
- High-precision timing
- Power-efficient tickless operation

### Timer Subsystem Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 Timer Subsystem Architecture                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              User Space                             │    │
│  │  nanosleep(), clock_gettime(), timerfd, alarm()     │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│  ────────────────────────┼──────────────────────────────    │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              High-Level Timer APIs                  │    │
│  │                                                     │    │
│  │   ┌─────────────┐        ┌─────────────────────┐    │    │
│  │   │ timer_list  │        │    hrtimer          │    │    │
│  │   │ (jiffies    │        │ (nanosecond         │    │    │
│  │   │  resolution)│        │  resolution)        │    │    │
│  │   └──────┬──────┘        └──────────┬──────────┘    │    │
│  │          │                          │               │    │
│  └──────────┼──────────────────────────┼───────────────┘    │
│             │                          │                    │
│  ┌──────────┴──────────────────────────┴───────────────┐    │
│  │              Timekeeping Core                       │    │
│  │                                                     │    │
│  │   ┌─────────────────┐    ┌─────────────────────┐    │    │
│  │   │   Clocksource   │    │    Clockevent       │    │    │
│  │   │ (time reading)  │    │ (interrupt gen)     │    │    │
│  │   └────────┬────────┘    └──────────┬──────────┘    │    │
│  │            │                        │               │    │
│  └────────────┼────────────────────────┼───────────────┘    │
│               │                        │                    │
│  ─────────────┼────────────────────────┼────────────────    │
│               │                        │                    │
│  ┌────────────┴────────────────────────┴───────────────┐    │
│  │              Hardware                               │    │
│  │                                                     │    │
│  │   TSC    HPET    ACPI_PM    Local APIC   PIT        │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Jiffies and HZ

Jiffies is the fundamental time unit in the kernel, representing tick counts
since boot.

### Jiffies Configuration

```c
/* include/linux/jiffies.h */

/* HZ - timer interrupts per second (from asm/param.h) */
/* Common values: 100, 250, 300, 1000 */

/* Jiffies counter */
extern unsigned long volatile jiffies;      /* 32-bit on 32-bit systems */
extern u64 jiffies_64;                      /* Always 64-bit */

/* INITIAL_JIFFIES starts at -300*HZ to expose wraparound bugs early */

/* Convert between time units (functions, not simple macros) */
extern unsigned int jiffies_to_msecs(const unsigned long j);
extern unsigned int jiffies_to_usecs(const unsigned long j);
unsigned long msecs_to_jiffies(const unsigned int m);  /* compile-time folded */
unsigned long usecs_to_jiffies(const unsigned int u);
#define secs_to_jiffies(_secs)  ((_secs) * HZ)

/* 64-bit variants */
extern u64 jiffies64_to_nsecs(u64 j);
extern u64 jiffies64_to_msecs(u64 j);

/* Safe jiffies comparison (handles wraparound, includes typecheck()) */
#define time_after(a, b)        (typecheck(unsigned long, a) && \
                                 typecheck(unsigned long, b) && \
                                 ((long)((b) - (a)) < 0))
#define time_before(a, b)       time_after(b, a)
#define time_after_eq(a, b)     ((long)((a) - (b)) >= 0)
#define time_before_eq(a, b)    time_after_eq(b, a)
#define time_in_range(a, b, c)  (time_after_eq(a, b) && time_before_eq(a, c))
#define time_in_range_open(a, b, c) (time_after_eq(a, b) && time_before(a, c))

/* 64-bit comparison variants also available:
 * time_after64, time_before64, time_in_range64, etc. */
```

### Jiffies Resolution

```
┌─────────────────────────────────────────────────────────────┐
│                  HZ and Resolution                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  HZ     Tick Period    Timer Resolution    Overhead         │
│  ───    ───────────    ────────────────    ────────         │
│  100    10 ms          10 ms               Low              │
│  250    4 ms           4 ms                Medium           │
│  300    3.33 ms        3.33 ms             Medium           │
│  1000   1 ms           1 ms                High             │
│                                                             │
│  Trade-off:                                                 │
│  - Higher HZ = Better timer resolution                      │
│  - Higher HZ = More interrupt overhead                      │
│  - Higher HZ = Better interactive response                  │
│  - Lower HZ = Better throughput, less power                 │
│                                                             │
│  Modern approach: Use NO_HZ (tickless) to get best of both  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Low-Resolution Timers

Low-resolution timers (timer_list) are the traditional timer mechanism with
jiffies granularity.

### Timer Structure

The timer_list structure is the fundamental building block for low-resolution
timers in the Linux kernel. It represents a single timer instance that will
execute a callback function when the specified expiration time (measured in
jiffies) is reached. Each timer is linked into a hierarchical timer wheel data
structure via its hlist_node entry, allowing efficient O(1) insertion and
expiration processing. The structure is typically embedded within driver or
subsystem-specific data structures, and the callback function receives a
pointer to the timer_list, enabling container_of() to retrieve the enclosing
structure. Key fields include the expiration time, callback function pointer,
and flags that control CPU affinity and deferral behavior.

```c
/* include/linux/timer_types.h (split from timer.h) */
struct timer_list {
    struct hlist_node       entry;
    unsigned long           expires;    /* Expiration time in jiffies */
    void (*function)(struct timer_list *);
    u32                     flags;

#ifdef CONFIG_LOCKDEP
    struct lockdep_map      lockdep_map;
#endif
};

/* Timer flags (include/linux/timer.h) */
#define TIMER_CPUMASK           0x0003FFFF  /* CPU binding mask */
#define TIMER_MIGRATING         0x00040000
#define TIMER_BASEMASK          (TIMER_CPUMASK | TIMER_MIGRATING)
#define TIMER_DEFERRABLE        0x00080000  /* Can be deferred for power */
#define TIMER_PINNED            0x00100000  /* Pinned to CPU */
#define TIMER_IRQSAFE           0x00200000  /* Safe in hardirq context */
```

### Timer API

```c
/* Static initialization */
#define DEFINE_TIMER(_name, _function) \
    struct timer_list _name = __TIMER_INITIALIZER(_function, 0)

/* Dynamic initialization */
timer_setup(&my_timer, my_callback, flags);

/* Modify expiration time and start */
mod_timer(&my_timer, jiffies + msecs_to_jiffies(100));

/* Add timer (must set expires first) */
my_timer.expires = jiffies + HZ;  /* 1 second */
add_timer(&my_timer);

/* Delete timer (prefer timer_delete* in new code) */
timer_delete(&my_timer);              /* May return while callback running */
timer_delete_sync(&my_timer);         /* Wait for callback to complete */
del_timer(&my_timer);                 /* Legacy alias for timer_delete */
del_timer_sync(&my_timer);            /* Legacy alias for timer_delete_sync */

/* Check if timer pending */
int pending = timer_pending(&my_timer);

/* Shutdown timer (prevents reactivation) */
timer_shutdown(&my_timer);
timer_shutdown_sync(&my_timer);

/* Reduce timer expiration (only moves earlier, never later) */
timer_reduce(&my_timer, jiffies + msecs_to_jiffies(50));

/* Add timer to specific base */
add_timer_local(&my_timer);           /* Local CPU only */
add_timer_global(&my_timer);          /* Can migrate between CPUs */

/* Get enclosing struct from timer callback */
#define from_timer(var, callback_timer, timer_fieldname) \
    container_of(callback_timer, typeof(*var), timer_fieldname)
```

### Timer Wheel (Hierarchical)

```
┌─────────────────────────────────────────────────────────────┐
│                 Timer Wheel Structure                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Modern kernel uses hierarchical timer wheel:               │
│  LVL_BITS=6 (64 buckets/level), LVL_DEPTH=9 (HZ>100)        │
│  Total: 576 buckets, 3 bases per CPU (NO_HZ_COMMON)         │
│                                                             │
│  Granularity table (HZ=1000):                               │
│  Level  Offset  Granularity   Range                         │
│    0       0       1 ms       0 ms  -    63 ms              │
│    1      64       8 ms      64 ms  -   511 ms              │
│    2     128      64 ms     512 ms  -  4095 ms  (~4s)       │
│    3     192     512 ms    4096 ms  - 32767 ms  (~32s)      │
│    4     256    4096 ms       ~4s   -   ~4 min              │
│    5     320   32768 ms      ~32s   -   ~34 min             │
│    6     384  262144 ms      ~4m    -   ~4 hours            │
│    7     448 2097152 ms      ~34m   -   ~1 day              │
│    8     512 16777216 ms     ~4h    -   ~12 days            │
│                                                             │
│  Three bases per CPU (CONFIG_NO_HZ_COMMON):                 │
│  BASE_LOCAL  (0) - Timers processed on local CPU only       │
│  BASE_GLOBAL (1) - Timers that can migrate to non-idle CPUs │
│  BASE_DEF    (2) - Deferrable timers                        │
│                                                             │
│  Key properties:                                            │
│  - O(1) insertion (hash to bucket)                          │
│  - NO cascading (unlike classic timer wheel)                │
│  - pending_map bitmap enables fast next-expiry scanning     │
│  - Near timers have better resolution                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Timer Example

```c
#include <linux/timer.h>

static struct timer_list my_timer;
static int counter = 0;

static void my_timer_callback(struct timer_list *t)
{
    counter++;
    pr_info("Timer expired! count=%d\n", counter);

    /* Re-arm timer for next expiration */
    mod_timer(&my_timer, jiffies + HZ);  /* 1 second */
}

static int __init my_init(void)
{
    timer_setup(&my_timer, my_timer_callback, 0);
    mod_timer(&my_timer, jiffies + HZ);

    return 0;
}

static void __exit my_exit(void)
{
    del_timer_sync(&my_timer);
}
```

---

## High-Resolution Timers (hrtimers)

High-resolution timers provide nanosecond resolution and are independent of the
system tick.

### hrtimer Structure

The hrtimer structure represents a high-resolution timer capable of
nanosecond-precision timing, independent of the system tick frequency (HZ).
Unlike the timer wheel used by timer_list, hrtimers are organized in a
red-black tree (via the timerqueue_node), which provides O(log n) insertion and
efficient retrieval of the next expiring timer. Each hrtimer is associated with
a clock base (monotonic, realtime, boottime, or TAI) and can be configured to
run in either hard IRQ context or soft IRQ context. The callback function
returns an hrtimer_restart enum to indicate whether the timer should be
restarted or has completed. The is_rel, is_soft, and is_hard flags control
timing mode and execution context.

```c
/* include/linux/hrtimer.h */
struct hrtimer {
    struct timerqueue_node      node;       /* RB-tree node */
    ktime_t                     _softexpires;
    enum hrtimer_restart        (*function)(struct hrtimer *);
    struct hrtimer_clock_base   *base;
    u8                          state;
    u8                          is_rel;
    u8                          is_soft;
    u8                          is_hard;
};

/* Clock bases */
enum hrtimer_base_type {
    HRTIMER_BASE_MONOTONIC,         /* Monotonic (CLOCK_MONOTONIC) */
    HRTIMER_BASE_REALTIME,          /* Wall clock (CLOCK_REALTIME) */
    HRTIMER_BASE_BOOTTIME,          /* Boot time (includes suspend) */
    HRTIMER_BASE_TAI,               /* International Atomic Time */
    HRTIMER_BASE_MONOTONIC_SOFT,    /* Soft IRQ versions */
    HRTIMER_BASE_REALTIME_SOFT,
    HRTIMER_BASE_BOOTTIME_SOFT,
    HRTIMER_BASE_TAI_SOFT,
    HRTIMER_MAX_CLOCK_BASES,
};

/* Return values from callback */
enum hrtimer_restart {
    HRTIMER_NORESTART,      /* Timer is done */
    HRTIMER_RESTART,        /* Restart timer */
};
```

### hrtimer API

```c
/* Initialize (traditional) */
hrtimer_init(&hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
hr_timer.function = my_hrtimer_callback;

/* Initialize (preferred in new code) */
hrtimer_setup(&hr_timer, my_hrtimer_callback,
              CLOCK_MONOTONIC, HRTIMER_MODE_REL);

/* Timer modes (can be combined with |) */
enum hrtimer_mode {
    HRTIMER_MODE_ABS      = 0x00,  /* Absolute time */
    HRTIMER_MODE_REL      = 0x01,  /* Relative to now */
    HRTIMER_MODE_PINNED   = 0x02,  /* Pinned to CPU */
    HRTIMER_MODE_SOFT     = 0x04,  /* Soft IRQ context */
    HRTIMER_MODE_HARD     = 0x08,  /* Hard IRQ context */
    /* Combinations: REL_PINNED, ABS_SOFT, REL_SOFT, ABS_HARD, etc. */
};

/* Start timer */
hrtimer_start(&hr_timer, ktime_set(1, 0), HRTIMER_MODE_REL);
/* ktime_set(seconds, nanoseconds) */

/* Start with range (for power efficiency) */
hrtimer_start_range_ns(&hr_timer, ns, slack_ns, mode);

/* Cancel timer */
hrtimer_cancel(&hr_timer);          /* Wait for callback */
hrtimer_try_to_cancel(&hr_timer);   /* Don't wait */

/* Forward timer expiration */
hrtimer_forward(&hr_timer, now, interval);
hrtimer_forward_now(&hr_timer, interval);

/* Check state */
int active = hrtimer_active(&hr_timer);
int callback_running = hrtimer_callback_running(&hr_timer);

/* Get remaining time */
ktime_t remaining = hrtimer_get_remaining(&hr_timer);
```

### ktime_t Operations

```c
/* include/linux/ktime.h */
typedef s64 ktime_t;  /* Time in nanoseconds */

/* Create ktime values */
ktime_t kt = ktime_set(secs, nsecs);
ktime_t kt = ns_to_ktime(nsecs);
ktime_t kt = ms_to_ktime(msecs);
ktime_t kt = KTIME_MAX;

/* Arithmetic */
ktime_t sum = ktime_add(a, b);
ktime_t diff = ktime_sub(a, b);
ktime_t added = ktime_add_ns(kt, nsecs);
ktime_t added = ktime_add_us(kt, usecs);
ktime_t added = ktime_add_ms(kt, msecs);

/* Comparison */
int cmp = ktime_compare(a, b);  /* -1, 0, or 1 */
bool after = ktime_after(a, b);
bool before = ktime_before(a, b);

/* Convert to other units */
s64 ns = ktime_to_ns(kt);
s64 us = ktime_to_us(kt);
s64 ms = ktime_to_ms(kt);
struct timespec64 ts = ktime_to_timespec64(kt);

/* Get current time */
ktime_t now = ktime_get();              /* Monotonic */
ktime_t now = ktime_get_real();         /* Wall clock */
ktime_t now = ktime_get_boottime();     /* Boot time */
```

### hrtimer Example

```c
#include <linux/hrtimer.h>
#include <linux/ktime.h>

static struct hrtimer hr_timer;
static ktime_t interval;

static enum hrtimer_restart my_hrtimer_callback(struct hrtimer *timer)
{
    ktime_t now = ktime_get();

    pr_info("hrtimer callback at %lld ns\n", ktime_to_ns(now));

    /* Forward and restart */
    hrtimer_forward_now(timer, interval);
    return HRTIMER_RESTART;
}

static int __init my_init(void)
{
    interval = ktime_set(0, 500000000);  /* 500 ms */

    hrtimer_init(&hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
    hr_timer.function = my_hrtimer_callback;

    hrtimer_start(&hr_timer, interval, HRTIMER_MODE_REL);

    return 0;
}

static void __exit my_exit(void)
{
    hrtimer_cancel(&hr_timer);
}
```

### Timer Comparison

```
┌─────────────────────────────────────────────────────────────┐
│            timer_list vs hrtimer Comparison                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Feature          timer_list          hrtimer               │
│  ────────         ──────────          ───────               │
│  Resolution       Jiffies (1-10ms)    Nanoseconds           │
│  Data structure   Timer wheel         Red-black tree        │
│  Callback context Softirq             Hardirq or softirq    │
│  Overhead         Very low            Low                   │
│  Use case         Timeouts, delays    Precise timing        │
│                                                             │
│  When to use timer_list:                                    │
│  - Coarse-grained timeouts                                  │
│  - Network retransmits                                      │
│  - Watchdogs with second+ resolution                        │
│                                                             │
│  When to use hrtimer:                                       │
│  - Sub-millisecond precision needed                         │
│  - Multimedia, audio timing                                 │
│  - CPU scheduling (CFS bandwidth)                           │
│  - High-precision delays                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Clocksource

Clocksources provide a monotonically increasing counter for timekeeping.

### Clocksource Structure

The clocksource structure abstracts hardware timing devices that provide
monotonically increasing counters for system timekeeping. Each clocksource
registers with the kernel and is rated according to its accuracy, stability,
and resolution, with the kernel automatically selecting the highest-rated
available source. The read() function pointer returns the current counter
value, which is converted to nanoseconds using the mult and shift fields for
efficient fixed-point arithmetic. Common clocksources on x86 include TSC
(highest performance), HPET, ACPI PM timer, and the legacy PIT. The structure
includes callbacks for power management (suspend/resume) and stability
tracking, as some clocksources (like older TSC implementations) can become
unreliable under frequency scaling or deep sleep states.

```c
/* include/linux/clocksource.h */
struct clocksource {
    u64 (*read)(struct clocksource *cs);    /* Read counter */
    u64                     mask;            /* Bitmask for counter */
    u32                     mult;            /* Cycle to ns multiplier */
    u32                     shift;           /* Shift for mult */

    u64                     max_idle_ns;     /* Max idle time */
    u32                     maxadj;          /* Max adjustment */

    const char              *name;
    struct list_head        list;

    int                     rating;          /* Quality rating */
    enum clocksource_ids    id;

    int (*enable)(struct clocksource *cs);
    void (*disable)(struct clocksource *cs);

    unsigned long           flags;

    void (*suspend)(struct clocksource *cs);
    void (*resume)(struct clocksource *cs);
    void (*mark_unstable)(struct clocksource *cs);
    void (*tick_stable)(struct clocksource *cs);

    struct module           *owner;
};

/* Rating values */
#define CLOCKSOURCE_RATING_PERFECT      400  /* TSC when stable */
#define CLOCKSOURCE_RATING_HIGH         350  /* HPET */
#define CLOCKSOURCE_RATING_GOOD         250  /* ACPI PM timer */
#define CLOCKSOURCE_RATING_LOW          100  /* PIT */
```

### Common Clocksources (x86)

```
┌─────────────────────────────────────────────────────────────┐
│                 x86 Clocksources                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TSC (Time Stamp Counter)                                   │
│  ─────────────────────────                                  │
│  - CPU cycle counter                                        │
│  - Highest resolution and lowest latency                    │
│  - Rating: 300-400 (if stable)                              │
│  - Issues: Varies with CPU frequency (older CPUs)           │
│  - Modern: Invariant TSC, nonstop TSC                       │
│                                                             │
│  HPET (High Precision Event Timer)                          │
│  ─────────────────────────────────                          │
│  - Dedicated timer chip                                     │
│  - 10+ MHz frequency                                        │
│  - Rating: 250                                              │
│  - Higher latency than TSC                                  │
│                                                             │
│  ACPI PM Timer                                              │
│  ─────────────                                              │
│  - 3.58 MHz, power-management timer                         │
│  - Rating: 200                                              │
│  - Very stable, slow                                        │
│                                                             │
│  PIT (Programmable Interval Timer)                          │
│  ─────────────────────────────────                          │
│  - Legacy 8254 timer                                        │
│  - Rating: 110                                              │
│  - ~1.19 MHz                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Viewing Clocksources

```bash
# View available clocksources
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# Output: tsc hpet acpi_pm

# View current clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# Output: tsc

# Change clocksource
echo "hpet" > /sys/devices/system/clocksource/clocksource0/current_clocksource
```

---

## Clockevents

Clockevents generate interrupts at specified times for timer expiration.

### Clockevent Structure

The clock_event_device structure represents hardware capable of generating
interrupts at programmed times, serving as the interrupt source for both the
periodic system tick and high-resolution timer expiration. Unlike clocksources
(which passively read time), clockevents actively drive the timer subsystem by
generating interrupts when timers expire. The structure supports multiple
operating modes: periodic mode generates fixed-interval interrupts for the
traditional tick, while one-shot mode allows programming individual interrupts
for each timer expiration, enabling tickless (NO_HZ) operation. Key function
    pointers include set_next_event() for programming the next interrupt and
    event_handler() which is called when the interrupt fires. The features
    field indicates hardware capabilities, and the rating helps the kernel
    select the best available device for each CPU.

```c
/* include/linux/clockchips.h */
struct clock_event_device {
    void (*event_handler)(struct clock_event_device *);
    int (*set_next_event)(unsigned long evt,
                          struct clock_event_device *);
    int (*set_next_ktime)(ktime_t expires,
                          struct clock_event_device *);
    ktime_t                 next_event;
    u64                     max_delta_ns;
    u64                     min_delta_ns;
    u32                     mult;
    u32                     shift;

    enum clock_event_state  state_use_accessors;

    unsigned int            features;

    const char              *name;
    int                     rating;

    int                     irq;
    int                     bound_on;
    const struct cpumask    *cpumask;

    struct list_head        list;
    struct module           *owner;

    int (*set_state_periodic)(struct clock_event_device *);
    int (*set_state_oneshot)(struct clock_event_device *);
    int (*set_state_oneshot_stopped)(struct clock_event_device *);
    int (*set_state_shutdown)(struct clock_event_device *);
    int (*tick_resume)(struct clock_event_device *);

    void (*suspend)(struct clock_event_device *);
    void (*resume)(struct clock_event_device *);
};

/* Features */
#define CLOCK_EVT_FEAT_PERIODIC     0x000001
#define CLOCK_EVT_FEAT_ONESHOT      0x000002
#define CLOCK_EVT_FEAT_KTIME        0x000004
#define CLOCK_EVT_FEAT_C3STOP       0x000008  /* Stops in C3 power state */
#define CLOCK_EVT_FEAT_DUMMY        0x000010
```

### Clockevent Modes

```
┌─────────────────────────────────────────────────────────────┐
│                  Clockevent Modes                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Periodic Mode:                                             │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┐                  │
│  │IRQ │    │IRQ │    │IRQ │    │IRQ │    │                  │
│  └────┴────┴────┴────┴────┴────┴────┴────┘                  │
│  Fixed interval interrupts (traditional tick)               │
│  Used when NO_HZ is disabled                                │
│                                                             │
│  One-shot Mode:                                             │
│  ┌────┐           ┌────┐    ┌────┐                          │
│  │IRQ │           │IRQ │    │IRQ │                          │
│  └────┴───────────┴────┴────┴────┴─────────────             │
│  Program each interrupt individually                        │
│  Used for hrtimers and NO_HZ                                │
│                                                             │
│  Shutdown/Stopped:                                          │
│  No interrupts generated                                    │
│  Used in deep idle states                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## NO_HZ and Tickless Operation

NO_HZ (tickless) operation eliminates periodic timer interrupts when idle.

### NO_HZ Modes

```c
/* Kernel config options */
CONFIG_NO_HZ_IDLE       /* Disable tick when idle (default) */
CONFIG_NO_HZ_FULL       /* Disable tick for busy CPUs too */
CONFIG_HZ_PERIODIC      /* Traditional periodic tick */
```

### Tickless Operation

```
┌─────────────────────────────────────────────────────────────┐
│                 Tickless vs Periodic                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CONFIG_HZ_PERIODIC (Traditional):                          │
│  ─────────────────────────────────                          │
│  ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐          │
│  │T │  │T │  │T │  │T │  │T │  │T │  │T │  │T │  │          │
│  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘          │
│     Tick every 1/HZ, even when idle                         │
│     Wastes power on idle systems                            │
│                                                             │
│  CONFIG_NO_HZ_IDLE (Tickless idle):                         │
│  ──────────────────────────────────                         │
│  ┌──┬──┬──┐                           ┌──┬──┐               │
│  │T │  │T │ (work)    (idle)          │T │  │ (work)        │
│  └──┴──┴──┘                           └──┴──┘               │
│     Tick only when there's work                             │
│     CPU can sleep deeply when idle                          │
│                                                             │
│  CONFIG_NO_HZ_FULL (Full tickless):                         │
│  ──────────────────────────────────                         │
│  ┌──┐                                                       │
│  │T │ (brief)      (user-space work, no tick)               │
│  └──┘                                                       │
│     Single task runs without timer interrupts               │
│     Best for latency-sensitive workloads                    │
│     Requires isolcpus= and nohz_full= boot params           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### tick_nohz Functions

```c
/* include/linux/tick.h */

/* Called when CPU enters idle */
void tick_nohz_idle_enter(void);

/* Called when CPU exits idle */
void tick_nohz_idle_exit(void);

/* Called in idle loop */
void tick_nohz_idle_stop_tick(void);
void tick_nohz_idle_restart_tick(void);

/* Get next timer deadline */
ktime_t tick_nohz_get_sleep_length(ktime_t *delta_next);

/* For full dynticks */
bool tick_nohz_full_running;
int tick_nohz_full_cpu(int cpu);

/* From cpuidle */
void tick_nohz_idle_stop_tick(void);
void tick_nohz_idle_retain_tick(void);
```

---

## Sleep and Delay Functions

### Delay Functions (Busy-Wait)

```c
/* include/linux/delay.h */

/* Microsecond delay (busy wait) */
void udelay(unsigned long usecs);       /* Up to ~1000 us */
void ndelay(unsigned long nsecs);       /* Rounds up to microseconds internally */
void mdelay(unsigned long msecs);       /* Milliseconds (use sparingly!) */

/* These spin and waste CPU, blocking all other work */
/* Use only for very short delays in atomic context */

/* Example: hardware register timing */
writel(CMD_START, base + CMD_REG);
udelay(10);  /* Wait 10us for hardware */
status = readl(base + STATUS_REG);
```

### Sleep Functions (Yield CPU)

```c
/* include/linux/delay.h */

/* Sleep functions - yield CPU to scheduler */
void msleep(unsigned int msecs);
void ssleep(unsigned int seconds);

/* Interruptible versions */
unsigned long msleep_interruptible(unsigned int msecs);
/* Returns remaining time if interrupted */

/* High-resolution sleep */
void usleep_range(unsigned long min, unsigned long max);
/* Provides slack for timer coalescing, saves power */

/* Additional sleep variants */
void usleep_range_state(unsigned long min, unsigned long max,
                        unsigned int state);  /* Specify task state */
void usleep_range_idle(unsigned long min, unsigned long max);
/* TASK_IDLE state - doesn't contribute to load average */

/* Unified smart sleep function (new, preferred) */
static inline void fsleep(unsigned long usecs);
/*
 * Auto-selects the best sleep mechanism:
 * <= 10 us:              udelay() (avoid hrtimer overhead)
 * < USLEEP_RANGE_UPPER:  usleep_range() with ~25% slack
 * >= USLEEP_RANGE_UPPER: msleep()
 */

/* Example */
msleep(100);                          /* Sleep ~100ms */
usleep_range(1000, 1100);             /* Sleep 1-1.1ms */
fsleep(500);                          /* Auto: udelay for this range */
fsleep(50000);                        /* Auto: usleep_range */
fsleep(200000);                       /* Auto: msleep */
```

### schedule_timeout

```c
/* Lowest-level sleep primitive */
signed long schedule_timeout(signed long timeout);

/* Interruptible version */
signed long schedule_timeout_interruptible(signed long timeout);
signed long schedule_timeout_killable(signed long timeout);
signed long schedule_timeout_uninterruptible(signed long timeout);

/* Usage */
set_current_state(TASK_INTERRUPTIBLE);
timeout = schedule_timeout(timeout);
/* Returns 0 if timeout expired, remaining jiffies if woken early */
```

### Sleep Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│                 Sleep/Delay Selection                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Duration        Atomic OK?     Recommended                 │
│  ────────        ──────────     ───────────                 │
│  < 10 us         Yes            udelay()                    │
│  10-20 us        Yes            udelay() or usleep_range()  │
│  > 20 us         No             usleep_range() or fsleep()  │
│  > 10 ms         No             msleep() or fsleep()        │
│  Any (auto)      No             fsleep() (unified)          │
│                                                             │
│  Key rules:                                                 │
│  - Atomic context (spinlock, IRQ): Only udelay/ndelay       │
│  - Process context: Prefer sleep functions                  │
│  - usleep_range() for power efficiency                      │
│  - Never use mdelay() if you can help it                    │
│                                                             │
│  Context check:                                             │
│  if (in_atomic() || irqs_disabled())                        │
│      udelay(delay);                                         │
│  else                                                       │
│      usleep_range(delay, delay + 100);                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Timekeeping

Timekeeping maintains the system's notion of current time.

### Time Types

```c
/* include/linux/time.h */

/* Wall clock time (can jump) */
void ktime_get_real_ts64(struct timespec64 *ts);
ktime_t ktime_get_real(void);
time64_t ktime_get_real_seconds(void);

/* Monotonic time (since boot, no jumps) */
void ktime_get_ts64(struct timespec64 *ts);
ktime_t ktime_get(void);

/* Boot time (monotonic + suspend time) */
void ktime_get_boottime_ts64(struct timespec64 *ts);
ktime_t ktime_get_boottime(void);

/* Raw monotonic (not adjusted by NTP) */
void ktime_get_raw_ts64(struct timespec64 *ts);
ktime_t ktime_get_raw(void);

/* Coarse versions (lower precision, faster) */
void ktime_get_coarse_ts64(struct timespec64 *ts);
void ktime_get_coarse_real_ts64(struct timespec64 *ts);
ktime_t ktime_get_coarse(void);
ktime_t ktime_get_coarse_real(void);
```

### Time Structures

The timespec64 and timeval structures are the standard representations for time
values throughout the kernel. The timespec64 structure stores time as a
seconds/nanoseconds pair and is the preferred 64-bit safe representation for
all new code, replacing the older 32-bit timespec that suffered from the Year
2038 problem. The timeval structure uses a seconds/microseconds pair and is
primarily maintained for legacy compatibility with older userspace APIs. These
structures are used extensively for storing timestamps, specifying timeouts,
and interfacing with system calls like clock_gettime() and nanosleep().
Conversion functions like timespec64_to_ns() and ns_to_timespec64() allow easy
transformation between these representations and raw nanosecond values.

```c
/* 64-bit time structures */
struct timespec64 {
    time64_t    tv_sec;         /* Seconds */
    long        tv_nsec;        /* Nanoseconds */
};

struct timeval {
    time_t      tv_sec;         /* Seconds */
    suseconds_t tv_usec;        /* Microseconds */
};

/* Time conversion */
struct timespec64 ts = ns_to_timespec64(nsec);
s64 nsec = timespec64_to_ns(&ts);
ktime_t kt = timespec64_to_ktime(ts);
struct timespec64 ts = ktime_to_timespec64(kt);
```

### Clock Types Comparison

```
┌─────────────────────────────────────────────────────────────┐
│                    Clock Types                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Clock Type        Adjusted?    Suspend?    Use Case        │
│  ──────────        ─────────    ────────    ────────        │
│  CLOCK_REALTIME    Yes (NTP)    Jumps       Wall clock      │
│  CLOCK_MONOTONIC   Yes (NTP)    Stops       Intervals       │
│  CLOCK_BOOTTIME    Yes (NTP)    Continues   Suspend-aware   │
│  CLOCK_MONOTONIC_  No           Stops       Raw intervals   │
│    RAW                                                      │
│  CLOCK_TAI         Yes          Jumps       Atomic time     │
│                                                             │
│  Usage guidelines:                                          │
│  - Timestamps for logs: CLOCK_REALTIME                      │
│  - Measuring intervals: CLOCK_MONOTONIC                     │
│  - Timers across suspend: CLOCK_BOOTTIME                    │
│  - Hardware correlation: CLOCK_MONOTONIC_RAW                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Timekeeper Internals

The `struct timekeeper` (include/linux/timekeeper_internal.h) is the kernel's
central timekeeping structure, cacheline-optimized for fast reads:

```c
struct timekeeper {
    struct tk_read_base tkr_mono;       /* CLOCK_MONOTONIC readout base */
    u64             xtime_sec;          /* CLOCK_REALTIME seconds */
    unsigned long   ktime_sec;          /* CLOCK_MONOTONIC seconds */
    struct timespec64 wall_to_monotonic;
    ktime_t         offs_real;          /* Monotonic → realtime offset */
    ktime_t         offs_boot;          /* Monotonic → boottime offset */
    ktime_t         offs_tai;           /* Monotonic → TAI offset */
    s32             tai_offset;
    struct tk_read_base tkr_raw;        /* CLOCK_MONOTONIC_RAW readout */
    /* ... NTP and internal fields ... */
};

struct tk_read_base {
    struct clocksource  *clock;
    u64                 mask;
    u64                 cycle_last;
    u32                 mult;
    u32                 shift;
    u64                 xtime_nsec;
    ktime_t             base;
    u64                 base_real;       /* For NMI-safe CLOCK_REALTIME */
};
```

**VDSO fast path**: `clock_gettime()` is implemented in userspace via VDSO
(Virtual Dynamic Shared Object), avoiding syscall overhead entirely. The kernel
updates VDSO data each tick via `update_vsyscall()`.

**NMI-safe timekeeper**: A `struct tk_fast` with double-buffered `tk_read_base`
arrays (`tk_fast_mono`, `tk_fast_raw`) allows lock-free reads from NMI context
using a seqcount latch design.

---

## POSIX Timers

POSIX timers provide userspace timer functionality via `timer_create()`,
`timer_settime()`, `timer_gettime()`, and `timer_delete()`.

### Implementation (kernel/time/posix-timers.c)

```c
/* Supported clocks dispatch via struct k_clock */
/* Each clock type registers timer_create, timer_set, timer_get,
 * timer_del, clock_gettime, clock_settime, nsleep callbacks */

/* Timers stored in 9-bit hash table keyed on (signal_struct, timer_id) */
static DEFINE_HASHTABLE(posix_timers_hashtable, 9);

/* Supported clocks: CLOCK_REALTIME, CLOCK_MONOTONIC,
 * CLOCK_BOOTTIME, CLOCK_TAI, CLOCK_PROCESS_CPUTIME_ID,
 * CLOCK_THREAD_CPUTIME_ID */

/* clock_nanosleep() ultimately calls hrtimer_nanosleep()
 * for all hardware clocks */
```

---

## timerfd

timerfd provides timer notifications via file descriptors, integrable with
poll/epoll/io_uring for event-driven programming.

### Implementation (fs/timerfd.c)

```c
struct timerfd_ctx {
    union {
        struct hrtimer tmr;     /* Regular clocks */
        struct alarm alarm;     /* CLOCK_*_ALARM clocks */
    } t;
    ktime_t tintv;              /* Interval for periodic timers */
    ktime_t moffs;              /* Monotonic offset at last set */
    wait_queue_head_t wqh;
    u64 ticks;                  /* Expiration count (read resets) */
    int clockid;
    short unsigned expired;
    short unsigned settime_flags;
    struct rcu_head rcu;
    struct list_head clist;     /* Cancel list for CANCEL_ON_SET */
    spinlock_t cancel_lock;
    bool might_cancel;
};

/* Supported flags */
#define TFD_TIMER_ABSTIME       (1 << 0)
#define TFD_TIMER_CANCEL_ON_SET (1 << 1)  /* Cancel on clock_settime */
#define TFD_CLOEXEC             O_CLOEXEC
#define TFD_NONBLOCK            O_NONBLOCK

/* Syscalls */
int timerfd_create(int clockid, int flags);
int timerfd_settime(int fd, int flags,
                    const struct itimerspec64 *new,
                    struct itimerspec64 *old);
int timerfd_gettime(int fd, struct itimerspec64 *cur);

/* Additional clocks supported:
 * CLOCK_REALTIME_ALARM, CLOCK_BOOTTIME_ALARM
 * (wakeup-capable, use alarm subsystem instead of hrtimers) */
```

---

## Summary

The Linux timer subsystem provides multiple layers of abstraction:

| Component | Purpose | Resolution |
|-----------|---------|------------|
| jiffies | System tick counter | 1-10 ms (HZ dependent) |
| timer_list | Coarse-grained timeouts | Jiffies (9-level wheel) |
| hrtimer | High-precision timers | Nanoseconds |
| clocksource | Time reading | Hardware dependent |
| clockevent | Interrupt generation | Hardware dependent |
| NO_HZ | Power-efficient ticks | N/A |
| POSIX timers | Userspace timer API | Nanoseconds |
| timerfd | Timer via file descriptor | Nanoseconds |

Key principles:
- Use the coarsest-grained timer that meets your needs
- Prefer sleep over busy-wait
- Use fsleep() for auto-selecting the best sleep mechanism
- Use usleep_range() for power efficiency
- hrtimers for sub-millisecond precision
- Prefer hrtimer_setup() over hrtimer_init() in new code
- Prefer timer_delete_sync() over del_timer_sync() in new code
- Consider NO_HZ impact for latency-sensitive applications
