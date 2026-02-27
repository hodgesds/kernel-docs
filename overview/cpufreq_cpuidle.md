# CPUFreq and CPUIdle Subsystems

This document provides a comprehensive overview of the Linux kernel's CPU
frequency scaling (CPUFreq) and CPU idle state management (CPUIdle)
subsystems. These two subsystems work together to balance performance and
power consumption: CPUFreq controls the operating frequency and voltage of
CPUs (P-states / DVFS), while CPUIdle manages which low-power idle state a
CPU enters when it has no work (C-states).

---

## Table of Contents

1. [CPUFreq Architecture Overview](#1-cpufreq-architecture-overview)
2. [CPUFreq Core Data Structures](#2-cpufreq-core-data-structures)
3. [CPUFreq Driver Interface](#3-cpufreq-driver-interface)
4. [Frequency Tables](#4-frequency-tables)
5. [CPUFreq Governors](#5-cpufreq-governors)
6. [Schedutil Governor -- Scheduler-Driven DVFS](#6-schedutil-governor----scheduler-driven-dvfs)
7. [Intel P-State Driver](#7-intel-p-state-driver)
8. [CPUFreq Notifiers and QoS](#8-cpufreq-notifiers-and-qos)
9. [CPUFreq Sysfs Interface](#9-cpufreq-sysfs-interface)
10. [CPUIdle Architecture Overview](#10-cpuidle-architecture-overview)
11. [CPUIdle Core Data Structures](#11-cpuidle-core-data-structures)
12. [CPUIdle Governors](#12-cpuidle-governors)
13. [Idle Loop Integration](#13-idle-loop-integration)
14. [CPUIdle Sysfs Interface](#14-cpuidle-sysfs-interface)
15. [Energy-Aware Scheduling Interaction](#15-energy-aware-scheduling-interaction)
16. [Locking Summary](#16-locking-summary)
17. [Key Data Flows](#17-key-data-flows)
18. [Source File Reference](#18-source-file-reference)

---

## 1. CPUFreq Architecture Overview

CPUFreq implements Dynamic Voltage and Frequency Scaling (DVFS) through
three layers:

```
  +------------------------------------------------------------------+
  |                        User Space                                 |
  |    /sys/devices/system/cpu/cpuN/cpufreq/                         |
  +------------------+-----------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |               CPUFreq Core  (drivers/cpufreq/cpufreq.c)         |
  |                                                                   |
  |  cpufreq_policy_list  <-->  [policy0] <--> [policy1] <--> ...    |
  |                                |                                  |
  |  cpufreq_governor_list <-->  [governor]                          |
  |                                |                                  |
  +------------------+-----------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |            CPUFreq Driver  (platform-specific)                   |
  |                                                                   |
  |  intel_pstate / intel_cpufreq / acpi-cpufreq / amd-pstate / ...  |
  |                                                                   |
  |  .init()  .verify()  .target_index()  .fast_switch()             |
  |  .setpolicy()  .adjust_perf()  .get()                            |
  +------------------------------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |                     Hardware (MSRs, MMIO, firmware)              |
  +------------------------------------------------------------------+
```

A **policy** groups CPUs that share a clock domain and must scale together.
A **governor** decides when and how to change frequency. A **driver**
interfaces with hardware to execute the actual frequency change. Only one
driver can be registered at a time. The global driver pointer is protected
by `cpufreq_driver_lock` (`drivers/cpufreq/cpufreq.c:60`).

```c
/* drivers/cpufreq/cpufreq.c:58-60 */
static struct cpufreq_driver *cpufreq_driver;
static DEFINE_PER_CPU(struct cpufreq_policy *, cpufreq_cpu_data);
static DEFINE_RWLOCK(cpufreq_driver_lock);
```

---

## 2. CPUFreq Core Data Structures

### 2.1 struct cpufreq_policy

**Location:** `include/linux/cpufreq.h:55`

The central structure representing one frequency-scaling domain. One policy
covers one or more CPUs that share a clock and must scale together.

```c
struct cpufreq_policy {
    /* CPUs sharing clock, require sw coordination */
    cpumask_var_t       cpus;           /* Online CPUs only */
    cpumask_var_t       related_cpus;   /* Online + Offline CPUs */
    cpumask_var_t       real_cpus;      /* Related and present */

    unsigned int        shared_type;    /* ACPI: NONE/HW/ALL/ANY */
    unsigned int        cpu;            /* cpu managing this policy, must be online */

    struct clk         *clk;
    struct cpufreq_cpuinfo cpuinfo;     /* HW min/max freq and transition latency */

    unsigned int        min;            /* in kHz - current policy min */
    unsigned int        max;            /* in kHz - current policy max */
    unsigned int        cur;            /* in kHz - current actual freq */
    unsigned int        suspend_freq;   /* freq to set during suspend */

    unsigned int        policy;         /* POWERSAVE or PERFORMANCE (setpolicy drivers) */
    unsigned int        last_policy;    /* policy before unplug */
    struct cpufreq_governor *governor;  /* active governor */
    void               *governor_data;  /* governor private state */

    struct work_struct  update;         /* deferred policy update work */

    struct freq_constraints constraints; /* PM QoS frequency constraints */
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    struct cpufreq_frequency_table *freq_table;  /* available freq steps */
    enum cpufreq_table_sorting freq_table_sorted;

    struct list_head    policy_list;     /* global policy list linkage */
    struct kobject      kobj;            /* sysfs representation */

    struct rw_semaphore rwsem;          /* read: policy readers; write: hotplug/updates */

    bool                fast_switch_possible;  /* driver supports fast switching */
    bool                fast_switch_enabled;   /* governor enabled fast switch */
    bool                strict_target;         /* governor wants exact freq */
    bool                efficiencies_available; /* inefficient freqs marked */

    unsigned int        transition_delay_us;   /* min interval between switches */
    bool                dvfs_possible_from_any_cpu; /* cross-CPU DVFS */
    bool                boost_enabled;         /* per-policy boost flag */

    unsigned int        cached_target_freq;    /* last resolved target freq */
    unsigned int        cached_resolved_idx;   /* index of last resolved freq */

    /* Synchronization for frequency transitions */
    bool                transition_ongoing;    /* freq transition in progress */
    spinlock_t          transition_lock;
    wait_queue_head_t   transition_wait;
    struct task_struct *transition_task;        /* task doing the transition */

    struct cpufreq_stats *stats;        /* time-in-state statistics */
    void               *driver_data;    /* driver private data */

    struct thermal_cooling_device *cdev; /* cooling device if CPUFREQ_IS_COOLING_DEV */

    struct notifier_block nb_min;       /* QoS min notifier */
    struct notifier_block nb_max;       /* QoS max notifier */
};
```

Key semantics:
- `cpus` vs `related_cpus`: `cpus` contains only currently online CPUs.
  `related_cpus` includes offline CPUs. The `cpu` field identifies which CPU
  manages the policy.
- `rwsem`: Primary synchronization. Read-locked by code reading the policy;
  write-locked by hotplug and `cpufreq_update_policy()`.
- `fast_switch_possible` / `fast_switch_enabled`: When both true, the
  governor calls `cpufreq_driver_fast_switch()` directly from the scheduler
  hot path without locks.
- `transition_lock` + `transition_wait`: Used by
  `cpufreq_freq_transition_begin/end()` to serialize frequency transitions
  (`drivers/cpufreq/cpufreq.c:413`).

### 2.2 struct cpufreq_cpuinfo

**Location:** `include/linux/cpufreq.h:47`

```c
struct cpufreq_cpuinfo {
    unsigned int    max_freq;           /* hardware max freq (kHz) */
    unsigned int    min_freq;           /* hardware min freq (kHz) */
    unsigned int    transition_latency; /* in nanoseconds */
};
```

### 2.3 struct cpufreq_freqs

**Location:** `include/linux/cpufreq.h:184`

Passed to transition notifiers during frequency changes:

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy;
    unsigned int old;       /* old frequency (kHz) */
    unsigned int new;       /* new frequency (kHz) */
    u8 flags;               /* flags of cpufreq_driver */
};
```

---

## 3. CPUFreq Driver Interface

### 3.1 struct cpufreq_driver

**Location:** `include/linux/cpufreq.h:336`

Hardware-specific driver interface registered with
`cpufreq_register_driver()`.

```c
struct cpufreq_driver {
    char        name[CPUFREQ_NAME_LEN];
    u16         flags;
    void       *driver_data;

    /* mandatory */
    int   (*init)(struct cpufreq_policy *policy);
    int   (*verify)(struct cpufreq_policy_data *policy);

    /* choose one: setpolicy OR target/target_index */
    int   (*setpolicy)(struct cpufreq_policy *policy);

    int   (*target)(struct cpufreq_policy *policy,        /* Deprecated */
                    unsigned int target_freq, unsigned int relation);
    int   (*target_index)(struct cpufreq_policy *policy, unsigned int index);
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);
    void  (*adjust_perf)(unsigned int cpu, unsigned long min_perf,
                         unsigned long target_perf, unsigned long capacity);

    /* intermediate frequency support */
    unsigned int (*get_intermediate)(struct cpufreq_policy *policy,
                                    unsigned int index);
    int   (*target_intermediate)(struct cpufreq_policy *policy,
                                 unsigned int index);

    unsigned int (*get)(unsigned int cpu);       /* read current HW freq */
    void  (*update_limits)(unsigned int cpu);    /* firmware notification */

    int   (*online)(struct cpufreq_policy *policy);
    int   (*offline)(struct cpufreq_policy *policy);
    void  (*exit)(struct cpufreq_policy *policy);
    int   (*suspend)(struct cpufreq_policy *policy);
    int   (*resume)(struct cpufreq_policy *policy);
    void  (*ready)(struct cpufreq_policy *policy);

    struct freq_attr **attr;             /* additional sysfs attributes */

    bool  boost_enabled;
    int   (*set_boost)(struct cpufreq_policy *policy, int state);

    void  (*register_em)(struct cpufreq_policy *policy);
};
```

### 3.2 Driver Types: setpolicy vs target

There are two fundamental driver categories:

**setpolicy drivers** (e.g., `intel_pstate` in active mode):
- Implement `->setpolicy()`. The hardware or firmware autonomously manages
  frequency within `policy->min` and `policy->max`.
- The `policy->policy` field is set to `CPUFREQ_POLICY_PERFORMANCE` or
  `CPUFREQ_POLICY_POWERSAVE`.
- No traditional governor is attached; available "governors" are
  `performance` and `powersave`, which merely set the policy mode.
- The `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` flag prevents dynamic governors.

**target drivers** (e.g., `acpi-cpufreq`, `intel_cpufreq`):
- Implement `->target_index()` and/or `->fast_switch()`.
- An external governor selects specific frequencies.
- The governor calls `__cpufreq_driver_target()` which resolves the target
  frequency to a table index via `cpufreq_frequency_table_target()`.

### 3.3 Driver Flags

**Location:** `include/linux/cpufreq.h:419-466`

```c
#define CPUFREQ_NEED_UPDATE_LIMITS        BIT(0)  /* invoke driver even if target unchanged */
#define CPUFREQ_CONST_LOOPS               BIT(1)  /* loops_per_jiffy unaffected by freq */
#define CPUFREQ_IS_COOLING_DEV            BIT(2)  /* auto-register as thermal cooling device */
#define CPUFREQ_HAVE_GOVERNOR_PER_POLICY  BIT(3)  /* per-policy governor tunables */
#define CPUFREQ_ASYNC_NOTIFICATION        BIT(4)  /* driver sends POSTCHANGE from outside target */
#define CPUFREQ_NEED_INITIAL_FREQ_CHECK   BIT(5)  /* verify CPU runs at table freq */
#define CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING BIT(6)  /* disallow dynamic governors */
```

### 3.4 Driver Registration

**Location:** `drivers/cpufreq/cpufreq.c:2921`

```c
int cpufreq_register_driver(struct cpufreq_driver *driver_data);
```

Registration validates the driver, acquires `cpufreq_driver_lock`, sets the
global `cpufreq_driver` pointer, and enables frequency invariance for
target-style drivers via a static branch. Only one driver can be registered
at a time; registering a second returns `-EEXIST`.

### 3.5 Frequency Change Paths

**Normal path (via `target_index`):**

```
Governor
  -> cpufreq_driver_target()                  [takes policy->rwsem]
    -> __cpufreq_driver_target()              [cpufreq.c:2345]
      -> __resolve_freq(policy, target_freq)  [find table index]
        -> cpufreq_frequency_table_target()   [cpufreq.h:1019]
      -> __target_index(policy, index)        [cpufreq.c:2285]
        -> cpufreq_freq_transition_begin()    [PRECHANGE notifier]
        -> driver->target_index(policy, idx)  [hardware write]
        -> cpufreq_freq_transition_end()      [POSTCHANGE notifier]
```

**Fast switch path (scheduler hot path):**

```
Scheduler tick / task wakeup
  -> cpufreq_update_util()
    -> sugov_update_single_freq()
      -> get_next_freq()
      -> cpufreq_driver_fast_switch(policy, target_freq)
        -> driver->fast_switch()              [no rwsem, no notifiers]
        -> policy->cur = new_freq
        -> arch_set_freq_scale()
```

The fast switch path runs directly in the scheduler context under
`rq->lock`, bypassing the rwsem and transition notifiers for minimal
latency.

---

## 4. Frequency Tables

### 4.1 struct cpufreq_frequency_table

**Location:** `include/linux/cpufreq.h:693`

```c
struct cpufreq_frequency_table {
    unsigned int    flags;          /* CPUFREQ_BOOST_FREQ, CPUFREQ_INEFFICIENT_FREQ */
    unsigned int    driver_data;    /* driver specific data, not used by core */
    unsigned int    frequency;      /* kHz - doesn't need to be in ascending order */
};
```

Special frequency values:
- `CPUFREQ_ENTRY_INVALID` (`~0u`) -- skip this entry
- `CPUFREQ_TABLE_END` (`~1u`) -- terminate the table

Flag values:
- `CPUFREQ_BOOST_FREQ` (`BIT(0)`) -- this is a boost frequency
- `CPUFREQ_INEFFICIENT_FREQ` (`BIT(1)`) -- this frequency is dominated by
  a more efficient one (same performance, higher power)

### 4.2 Iteration Macros

**Location:** `include/linux/cpufreq.h:700-762`

```c
/* Iterate over all entries */
cpufreq_for_each_entry(pos, table)

/* Iterate skipping CPUFREQ_ENTRY_INVALID */
cpufreq_for_each_valid_entry(pos, table)

/* Iterate skipping invalid and optionally inefficient entries */
cpufreq_for_each_efficient_entry_idx(pos, table, idx, efficiencies)
```

### 4.3 Table Sorting

Tables can be `CPUFREQ_TABLE_SORTED_ASCENDING`,
`CPUFREQ_TABLE_SORTED_DESCENDING`, or `CPUFREQ_TABLE_UNSORTED`. Sorted
tables enable optimized binary-like lookups via
`cpufreq_table_find_index_{al,dl,ah,dh,ac,dc}()` helper functions. The
sorting order is detected during `cpufreq_table_validate_and_sort()`.

### 4.4 Frequency Relations

When resolving a target frequency to a table entry, one of these relations
is used (`include/linux/cpufreq.h:295-303`):

```c
#define CPUFREQ_RELATION_L  0   /* lowest freq at or above target */
#define CPUFREQ_RELATION_H  1   /* highest freq at or below target */
#define CPUFREQ_RELATION_C  2   /* closest freq to target */
#define CPUFREQ_RELATION_E  BIT(2)  /* prefer efficient freq if available */
```

`CPUFREQ_RELATION_E` can be OR-ed with any of the above to prefer
efficient frequencies. If no in-limits efficient frequency is found, the
lookup retries without the efficiency constraint.

---

## 5. CPUFreq Governors

### 5.1 struct cpufreq_governor

**Location:** `include/linux/cpufreq.h:580`

```c
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];
    int   (*init)(struct cpufreq_policy *policy);
    void  (*exit)(struct cpufreq_policy *policy);
    int   (*start)(struct cpufreq_policy *policy);
    void  (*stop)(struct cpufreq_policy *policy);
    void  (*limits)(struct cpufreq_policy *policy);
    ssize_t (*show_setspeed)(struct cpufreq_policy *policy, char *buf);
    int   (*store_setspeed)(struct cpufreq_policy *policy, unsigned int freq);
    struct list_head    governor_list;
    struct module      *owner;
    u8                  flags;
};
```

Governor lifecycle: `init` -> `start` -> [running] -> `stop` -> `exit`.
The `limits` callback is invoked when `policy->min` or `policy->max`
changes while the governor is running.

Governor flags (`include/linux/cpufreq.h:598-601`):

```c
#define CPUFREQ_GOV_DYNAMIC_SWITCHING  BIT(0)  /* changes freq dynamically */
#define CPUFREQ_GOV_STRICT_TARGET      BIT(1)  /* wants exact requested freq */
```

### 5.2 Per-Policy vs Global Governors

When `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` is set, each policy gets its own
sysfs directory for governor tunables (`cpu/cpuN/cpufreq/<governor>/`).
Without the flag, all policies share a single global governor tunable
directory. `struct gov_attr_set` handles this sharing with a reference-
counted `usage_count` (`include/linux/cpufreq.h:655`).

### 5.3 Available Governors

**performance** (`drivers/cpufreq/cpufreq_performance.c`):
- Always sets `policy->max`. Serves as the fallback governor.
- No dynamic switching; no tunables.

**powersave** (`drivers/cpufreq/cpufreq_powersave.c`):
- Always sets `policy->min`.
- No dynamic switching; no tunables.

**ondemand** (`drivers/cpufreq/cpufreq_ondemand.c`):
- Samples CPU idle time at configurable intervals using the dbs
  (demand-based switching) framework (`cpufreq_governor.c`).
- Default up-threshold: 80%. Above threshold, jumps to max; below,
  steps down.
- Tunables: `sampling_rate`, `up_threshold`, `sampling_down_factor`,
  `powersave_bias`, `io_is_busy`.

**conservative** (`drivers/cpufreq/cpufreq_conservative.c`):
- Like ondemand but steps frequency gradually up/down rather than
  jumping to max.
- Tunables: `up_threshold`, `down_threshold`, `freq_step`,
  `sampling_rate`, `sampling_down_factor`.

**schedutil** (`kernel/sched/cpufreq_schedutil.c`):
- Scheduler-driven governor; see Section 6.

**userspace** (`drivers/cpufreq/cpufreq_userspace.c`):
- Allows userspace to set the frequency directly via
  `scaling_setspeed` sysfs attribute.

---

## 6. Schedutil Governor -- Scheduler-Driven DVFS

**Location:** `kernel/sched/cpufreq_schedutil.c`

The schedutil governor is directly integrated with the scheduler's
utilization tracking via the `update_util_data` callback hook. It is called
from `cpufreq_update_util()` in the scheduler hot path (task wakeup,
scheduler tick), making frequency decisions with zero sampling overhead.

### 6.1 Key Data Structures

```c
/* kernel/sched/cpufreq_schedutil.c:11 */
struct sugov_tunables {
    struct gov_attr_set attr_set;
    unsigned int        rate_limit_us;  /* min interval between freq updates */
};

/* kernel/sched/cpufreq_schedutil.c:16 */
struct sugov_policy {
    struct cpufreq_policy  *policy;
    struct sugov_tunables  *tunables;
    struct list_head        tunables_hook;

    raw_spinlock_t          update_lock;
    u64                     last_freq_update_time;
    s64                     freq_update_delay_ns;
    unsigned int            next_freq;
    unsigned int            cached_raw_freq;

    /* For slow path (when fast switch unavailable): */
    struct irq_work         irq_work;
    struct kthread_work     work;
    struct mutex            work_lock;
    struct kthread_worker   worker;
    struct task_struct     *thread;
    bool                    work_in_progress;

    bool                    limits_changed;
    bool                    need_freq_update;
};

/* kernel/sched/cpufreq_schedutil.c:40 */
struct sugov_cpu {
    struct update_util_data update_util;
    struct sugov_policy    *sg_policy;
    unsigned int            cpu;

    bool                    iowait_boost_pending;
    unsigned int            iowait_boost;
    u64                     last_update;

    unsigned long           util;
    unsigned long           bw_min;
};
```

### 6.2 Frequency Computation

The core formula is in `get_next_freq()` (`cpufreq_schedutil.c:165`):

```
next_freq = C * ref_freq * util / max
```

Where `C = 1.25` (providing a tipping point at 80% utilization), `ref_freq`
is determined by `get_capacity_ref_freq()`, `util` is the CPU's current
utilization from PELT, and `max` is the CPU's capacity.

```c
/* cpufreq_schedutil.c:165 */
static unsigned int get_next_freq(struct sugov_policy *sg_policy,
                                  unsigned long util, unsigned long max)
{
    struct cpufreq_policy *policy = sg_policy->policy;
    unsigned int freq;

    freq = get_capacity_ref_freq(policy);
    freq = map_util_freq(util, freq, max);    /* = util * freq / max */
    ...
    return cpufreq_driver_resolve_freq(policy, freq);
}
```

### 6.3 Update Paths

Schedutil selects its callback at `start()` time (`cpufreq_schedutil.c:847`):

```c
if (policy_is_shared(policy))
    uu = sugov_update_shared;           /* shared policy: multiple CPUs */
else if (policy->fast_switch_enabled && cpufreq_driver_has_adjust_perf())
    uu = sugov_update_single_perf;      /* fast path with adjust_perf */
else
    uu = sugov_update_single_freq;      /* single CPU, freq-based */
```

**Fast switch path** (`sugov_update_single_freq`, `cpufreq_schedutil.c:391`):
1. Called from scheduler via `update_util_data` hook under `rq->lock`
2. Check rate limit (`freq_update_delay_ns`)
3. Compute iowait boost and get CPU utilization
4. Call `get_next_freq()` to compute target frequency
5. If `fast_switch_enabled`: call `cpufreq_driver_fast_switch()` directly
6. Otherwise: queue `irq_work` -> `kthread_work` via `sugov_deferred_update()`

**Slow switch path** (`sugov_work`, `cpufreq_schedutil.c:514`):
1. `irq_work` handler queues `kthread_work` on dedicated `sugov_worker`
2. Worker thread calls `__cpufreq_driver_target()` under `work_lock`
3. Worker runs at `SCHED_DEADLINE` priority with `SCHED_FLAG_SUGOV` flag

**adjust_perf path** (`sugov_update_single_perf`, `cpufreq_schedutil.c:432`):
- Used when the driver provides `->adjust_perf()` (e.g., `intel_pstate`
  passive mode with HWP)
- Passes min_perf, target_perf, and capacity directly to the driver
- Avoids the frequency-to-index resolution, letting the hardware manage
  the mapping

### 6.4 IO Wait Boost

When a task wakes up after IO (`SCHED_CPUFREQ_IOWAIT` flag), schedutil
applies a boost that doubles at each successive IO wakeup within a tick
(`cpufreq_schedutil.c:250`):

```
IOWAIT_BOOST_MIN (1/8 of max capacity) -> 1/4 -> 1/2 -> full capacity
```

The boost resets if more than one tick passes without an IO wakeup.

### 6.5 Governor Registration

```c
/* cpufreq_schedutil.c:894 */
struct cpufreq_governor schedutil_gov = {
    .name   = "schedutil",
    .owner  = THIS_MODULE,
    .flags  = CPUFREQ_GOV_DYNAMIC_SWITCHING,
    .init   = sugov_init,
    .exit   = sugov_exit,
    .start  = sugov_start,
    .stop   = sugov_stop,
    .limits = sugov_limits,
};
```

When schedutil starts or stops, it triggers `sugov_eas_rebuild_sd()` which
schedules `rebuild_sched_domains_energy()` to ensure the Energy Aware
Scheduler (EAS) is properly enabled or disabled.

---

## 7. Intel P-State Driver

**Location:** `drivers/cpufreq/intel_pstate.c`

The `intel_pstate` driver supports two operational modes:

### 7.1 Active Mode (intel_pstate)

Uses the `setpolicy` interface. The driver internally manages frequency
using Hardware P-states (HWP) or a built-in P-state selection algorithm.

```c
/* intel_pstate.c:3020 */
static struct cpufreq_driver intel_pstate = {
    .flags      = CPUFREQ_CONST_LOOPS,
    .verify     = intel_pstate_verify_policy,
    .setpolicy  = intel_pstate_set_policy,     /* setpolicy driver */
    .suspend    = intel_pstate_suspend,
    .resume     = intel_pstate_resume,
    .init       = intel_pstate_cpu_init,
    .exit       = intel_pstate_cpu_exit,
    .offline    = intel_pstate_cpu_offline,
    .online     = intel_pstate_cpu_online,
    .update_limits = intel_pstate_update_limits,
    .name       = "intel_pstate",
};
```

In active mode with HWP, the driver writes `MSR_HWP_REQUEST` with min/max
performance ratios and an Energy Performance Preference (EPP) value. The
hardware autonomously selects the operating point. The `setpolicy` callback
(`intel_pstate.c:2821`) translates `CPUFREQ_POLICY_PERFORMANCE` /
`CPUFREQ_POLICY_POWERSAVE` into appropriate HWP EPP settings.

### 7.2 Passive Mode (intel_cpufreq)

Uses the `target`/`fast_switch` interface and works with any governor
(typically schedutil).

```c
/* intel_pstate.c:3331 */
static struct cpufreq_driver intel_cpufreq = {
    .flags      = CPUFREQ_CONST_LOOPS,
    .verify     = intel_cpufreq_verify_policy,
    .target     = intel_cpufreq_target,
    .fast_switch = intel_cpufreq_fast_switch,  /* target driver */
    .init       = intel_cpufreq_cpu_init,
    .exit       = intel_cpufreq_cpu_exit,
    .offline    = intel_cpufreq_cpu_offline,
    .online     = intel_pstate_cpu_online,
    .suspend    = intel_cpufreq_suspend,
    .resume     = intel_pstate_resume,
    .update_limits = intel_pstate_update_limits,
    .name       = "intel_cpufreq",
};
```

### 7.3 Per-CPU Data

```c
/* intel_pstate.c:227 */
struct cpudata {
    int cpu;
    unsigned int policy;                /* CPUFREQ_POLICY_{PERFORMANCE,POWERSAVE} */
    struct update_util_data update_util;
    bool   update_util_set;

    struct pstate_data pstate;          /* P-state limits */
    struct vid_data vid;                /* voltage (Atom only) */

    u64    last_update;
    u64    prev_aperf, prev_mperf, prev_tsc;
    struct sample sample;               /* APERF/MPERF performance sample */
    int32_t min_perf_ratio;
    int32_t max_perf_ratio;

    unsigned int iowait_boost;
    s16 epp_powersave;                  /* saved EPP for policy switch */
    s16 epp_policy;
    s16 epp_default;
    s16 epp_cached;
    u64 hwp_req_cached;                 /* cached HWP Request MSR */
    u64 hwp_cap_cached;                 /* cached HWP Capabilities MSR */
    unsigned int capacity_perf;         /* highest perf for scale invariance */
    u32 hwp_boost_min;
    bool suspended;
};
```

### 7.4 P-State Data

```c
/* intel_pstate.c:137 */
struct pstate_data {
    int current_pstate;
    int min_pstate;
    int max_pstate;
    int max_pstate_physical;
    int perf_ctl_scaling;    /* PERF_CTL P-state to frequency scaling */
    int scaling;
    int turbo_pstate;
    unsigned int min_freq, max_freq, turbo_freq;  /* in cpufreq units */
};
```

### 7.5 HWP (Hardware P-states)

When `hwp_active` is true (`intel_pstate.c:296`), the driver uses Intel
Hardware P-state (HWP) feature:

- `MSR_HWP_CAPABILITIES` -- reports guaranteed, highest, lowest, and
  most-efficient performance levels
- `MSR_HWP_REQUEST` -- writes desired min, max, and EPP values
- The CPU hardware autonomously selects the P-state within the configured
  bounds, guided by the EPP (Energy Performance Preference, 0-255,
  0=performance, 255=power-saving)

The `intel_pstate_hwp_set()` function (`intel_pstate.c:1148`) programs the
HWP request MSR based on the current policy and performance limits.

### 7.6 Hybrid Scaling

For hybrid architectures (e.g., Alder Lake), different core types have
different frequency-to-performance scaling factors. The driver uses
`HYBRID_SCALING_FACTOR` constants (`intel_pstate.c:305-307`) to properly
map frequencies for big vs little cores.

---

## 8. CPUFreq Notifiers and QoS

### 8.1 Transition Notifiers

**Location:** `include/linux/cpufreq.h:509-526`

```c
#define CPUFREQ_TRANSITION_NOTIFIER  (0)
#define CPUFREQ_POLICY_NOTIFIER      (1)

#define CPUFREQ_PRECHANGE            (0)
#define CPUFREQ_POSTCHANGE           (1)

int cpufreq_register_notifier(struct notifier_block *nb, unsigned int list);
int cpufreq_unregister_notifier(struct notifier_block *nb, unsigned int list);
```

**Transition notifiers** are invoked around frequency changes:
- `CPUFREQ_PRECHANGE` -- before the frequency change (from
  `cpufreq_freq_transition_begin()`, `cpufreq.c:413`)
- `CPUFREQ_POSTCHANGE` -- after the frequency change (from
  `cpufreq_freq_transition_end()`, `cpufreq.c:447`)

The transition notifier list uses SRCU (`srcu_notifier_head`) for lockless
read access. Note: fast-switch path does NOT call transition notifiers.

**Policy notifiers** are invoked on policy creation/removal:
- `CPUFREQ_CREATE_POLICY` / `CPUFREQ_REMOVE_POLICY`

### 8.2 PM QoS Frequency Constraints

Each policy has `freq_constraints` with `min_freq_req` and `max_freq_req`.
When userspace or kernel subsystems (e.g., thermal) update `scaling_min_freq`
or `scaling_max_freq`, these flow through `freq_qos_add_request()` /
`freq_qos_update_request()`, which in turn trigger `cpufreq_update_policy()`
to apply the new constraints.

The notifier blocks `nb_min` and `nb_max` in `struct cpufreq_policy` listen
for QoS changes and call `cpufreq_update_limits()` to re-evaluate the
active frequency.

---

## 9. CPUFreq Sysfs Interface

Per-CPU sysfs directory: `/sys/devices/system/cpu/cpuN/cpufreq/`

| File | R/W | Description |
|---|---|---|
| `cpuinfo_min_freq` | R | Hardware minimum frequency (kHz) |
| `cpuinfo_max_freq` | R | Hardware maximum frequency (kHz) |
| `cpuinfo_transition_latency` | R | Transition latency (ns) |
| `scaling_min_freq` | RW | Policy minimum frequency (kHz) |
| `scaling_max_freq` | RW | Policy maximum frequency (kHz) |
| `scaling_cur_freq` | R | Current frequency (kHz) |
| `scaling_governor` | RW | Active governor name |
| `scaling_available_governors` | R | List of available governors |
| `scaling_available_frequencies` | R | List of available frequencies |
| `scaling_driver` | R | Name of the active driver |
| `scaling_setspeed` | RW | Set frequency directly (userspace governor) |
| `related_cpus` | R | CPUs in the same frequency domain |
| `affected_cpus` | R | Online CPUs sharing this policy |
| `local_boost` | RW | Per-policy boost enable/disable |

Global sysfs: `/sys/devices/system/cpu/cpufreq/`

| File | R/W | Description |
|---|---|---|
| `boost` | RW | Global frequency boost enable/disable |

Governor tunables appear under the governor directory. For schedutil:
`/sys/devices/system/cpu/cpufreq/schedutil/rate_limit_us` (or per-policy
if `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` is set).

---

## 10. CPUIdle Architecture Overview

CPUIdle manages which C-state (idle state) a CPU enters when the scheduler
has no work to run. The architecture mirrors CPUFreq: a **driver** declares
available states, and a **governor** picks which one to enter each time.

```
  +------------------------------------------------------------------+
  |                    Scheduler Idle Loop                            |
  |              (kernel/sched/idle.c: do_idle)                      |
  +------------------+-----------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |               CPUIdle Core  (drivers/cpuidle/cpuidle.c)          |
  |                                                                   |
  |  cpuidle_select()   --> governor->select()                       |
  |  cpuidle_enter()    --> cpuidle_enter_state()                    |
  |  cpuidle_reflect()  --> governor->reflect()                      |
  +------------------+-----------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |            CPUIdle Governor                                      |
  |                                                                   |
  |  menu  (rating=20)  |  teo  (rating=19)  |  ladder | haltpoll   |
  +------------------+-----------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |            CPUIdle Driver  (platform-specific)                   |
  |                                                                   |
  |  acpi_idle / intel_idle / psci / ...                             |
  |  states[0..N]: enter(), exit_latency_ns, target_residency_ns    |
  +------------------------------------------------------------------+
                     |
  +------------------v-----------------------------------------------+
  |               Hardware (MWAIT, WFI, HLT, PSCI)                  |
  +------------------------------------------------------------------+
```

States must be ordered by decreasing power consumption: state 0 is the
shallowest (polling or C1), higher indices are progressively deeper.
Maximum states: `CPUIDLE_STATE_MAX = 10`.

---

## 11. CPUIdle Core Data Structures

### 11.1 struct cpuidle_state

**Location:** `include/linux/cpuidle.h:49`

Describes one C-state that hardware supports:

```c
struct cpuidle_state {
    char        name[CPUIDLE_NAME_LEN];     /* e.g., "C1", "C1E", "C6" */
    char        desc[CPUIDLE_DESC_LEN];

    s64         exit_latency_ns;            /* time to wake from this state */
    s64         target_residency_ns;        /* min time to justify entry */
    unsigned int flags;
    unsigned int exit_latency;              /* in US (microseconds) */
    int         power_usage;                /* in mW */
    unsigned int target_residency;          /* in US */

    int (*enter)(struct cpuidle_device *dev,
                 struct cpuidle_driver *drv, int index);
    void (*enter_dead)(struct cpuidle_device *dev, int index);
    int (*enter_s2idle)(struct cpuidle_device *dev,
                        struct cpuidle_driver *drv, int index);
};
```

- `exit_latency_ns` -- how long it takes to resume from this state
- `target_residency_ns` -- how long the CPU must stay idle to justify the
  entry overhead. Governors check `predicted_idle_duration >= target_residency`
  before selecting a state.
- `enter()` -- main idle entry function, called from `cpuidle_enter_state()`
- `enter_dead()` -- used for CPU offlining (`cpuidle_play_dead()`)
- `enter_s2idle()` -- used during suspend-to-idle; must not re-enable
  interrupts at any point

### 11.2 State Flags

**Location:** `include/linux/cpuidle.h:80-87`

```c
#define CPUIDLE_FLAG_NONE           (0x00)
#define CPUIDLE_FLAG_POLLING        BIT(0)  /* state is busy-wait polling loop */
#define CPUIDLE_FLAG_COUPLED        BIT(1)  /* state applies to multiple cpus */
#define CPUIDLE_FLAG_TIMER_STOP     BIT(2)  /* local timer stops in this state */
#define CPUIDLE_FLAG_UNUSABLE       BIT(3)  /* avoid using this state */
#define CPUIDLE_FLAG_OFF            BIT(4)  /* disable this state by default */
#define CPUIDLE_FLAG_TLB_FLUSHED    BIT(5)  /* state flushes TLBs */
#define CPUIDLE_FLAG_RCU_IDLE       BIT(6)  /* state handles its own RCU idle */
```

Disable flags in `cpuidle_state_usage`:
```c
#define CPUIDLE_STATE_DISABLED_BY_USER    BIT(0)
#define CPUIDLE_STATE_DISABLED_BY_DRIVER  BIT(1)
```

### 11.3 struct cpuidle_state_usage

**Location:** `include/linux/cpuidle.h:36`

Per-CPU, per-state counters:

```c
struct cpuidle_state_usage {
    unsigned long long  disable;       /* disable bitmask */
    unsigned long long  usage;         /* times entered */
    u64                 time_ns;       /* total time in state */
    unsigned long long  above;         /* too deep: residency < target */
    unsigned long long  below;         /* too shallow: deeper state available */
    unsigned long long  rejected;      /* idle entry rejected */
#ifdef CONFIG_SUSPEND
    unsigned long long  s2idle_usage;  /* s2idle entry count */
    unsigned long long  s2idle_time;   /* s2idle total time (US) */
#endif
};
```

### 11.4 struct cpuidle_device

**Location:** `include/linux/cpuidle.h:93`

Per-CPU idle device state:

```c
struct cpuidle_device {
    unsigned int        registered:1;
    unsigned int        enabled:1;
    unsigned int        poll_time_limit:1;
    unsigned int        cpu;
    ktime_t             next_hrtimer;          /* next hrtimer expiry */

    int                 last_state_idx;
    u64                 last_residency_ns;     /* actual time in last state */
    u64                 poll_limit_ns;
    u64                 forced_idle_latency_limit_ns;  /* governor override */
    struct cpuidle_state_usage states_usage[CPUIDLE_STATE_MAX];

    struct cpuidle_state_kobj *kobjs[CPUIDLE_STATE_MAX];
    struct cpuidle_driver_kobj *kobj_driver;
    struct cpuidle_device_kobj *kobj_dev;
    struct list_head    device_list;
};
```

`forced_idle_latency_limit_ns` is set by `cpuidle_use_deepest_state()` to
bypass the governor and enter the deepest state within the given latency
constraint (used by `play_idle_precise()` for idle injection).

### 11.5 struct cpuidle_driver

**Location:** `include/linux/cpuidle.h:152`

```c
struct cpuidle_driver {
    const char         *name;
    struct module      *owner;
    unsigned int        bctimer:1;          /* broadcast timer setup */
    struct cpuidle_state states[CPUIDLE_STATE_MAX];  /* shallow -> deep */
    int                 state_count;
    int                 safe_state_index;   /* failsafe shallow state */
    struct cpumask     *cpumask;            /* CPUs this driver handles */
    const char         *governor;           /* preferred governor name */
};
```

### 11.6 struct cpuidle_governor

**Location:** `include/linux/cpuidle.h:288`

```c
struct cpuidle_governor {
    char                name[CPUIDLE_NAME_LEN];
    struct list_head    governor_list;
    unsigned int        rating;             /* higher = preferred */

    int  (*enable)(struct cpuidle_driver *drv, struct cpuidle_device *dev);
    void (*disable)(struct cpuidle_driver *drv, struct cpuidle_device *dev);
    int  (*select)(struct cpuidle_driver *drv, struct cpuidle_device *dev,
                   bool *stop_tick);
    void (*reflect)(struct cpuidle_device *dev, int index);
};
```

- `select()` -- called every time a CPU goes idle; returns the chosen state
  index. Writes `*stop_tick = false` if the scheduler tick should keep
  running.
- `reflect()` -- called after wakeup for governor learning and metric
  updates.
- `rating` -- the governor with the highest rating is selected by default.
  The menu governor has rating 20; TEO has rating 19.

---

## 12. CPUIdle Governors

### 12.1 Menu Governor

**Location:** `drivers/cpuidle/governors/menu.c`

The menu governor is the default governor on tickless (NO_HZ) systems
(rating = 20). It uses two independent prediction methods:

**Energy break-even prediction:**
1. Reads the time till the next timer event via
   `tick_nohz_get_sleep_length()`
2. Applies a correction factor from a 6-bucket histogram indexed by the
   magnitude of the expected duration (`which_bucket()`, `menu.c:90`):
   - Buckets: <10us, <100us, <1ms, <10ms, <100ms, >=100ms
3. The correction factor is an exponentially decaying average
   (`DECAY = 8`) of the ratio `actual_idle_time / predicted_timer_time`

**Repeatable-interval detector** (`get_typical_interval()`, `menu.c:117`):
- Tracks the last 8 (`INTERVALS`) idle durations
- If the standard deviation is below a threshold (variance <= 400 us^2 or
  avg^2 > 36 * variance), uses the average as prediction
- Handles workloads with fixed-rate wakeups (hardware interrupt coalescing,
  input devices)

**State selection** (`menu_select()`, `menu.c:208`):
- Uses the minimum of the timer-corrected prediction and the typical
  interval
- Selects the deepest state whose `target_residency_ns <= predicted_ns`
  and `exit_latency_ns <= latency_req` (from PM QoS)
- Decides whether to stop the tick based on predicted idle duration vs
  tick period

**Per-CPU data:**
```c
/* menu.c:79 */
struct menu_device {
    int             needs_update;
    int             tick_wakeup;
    u64             next_timer_ns;
    unsigned int    bucket;
    unsigned int    correction_factor[BUCKETS]; /* 6 buckets */
    unsigned int    intervals[INTERVALS];       /* last 8 intervals */
    int             interval_ptr;
};
```

### 12.2 TEO (Timer Events Oriented) Governor

**Location:** `drivers/cpuidle/governors/teo.c`

The TEO governor (rating = 19) is based on the observation that timer
interrupts dominate CPU wakeup patterns. It uses bins aligned with state
`target_residency` boundaries.

**Metrics:**
- **hits** -- the sleep length and measured idle duration fall into the same
  bin (CPU woke up "on time" from a timer)
- **intercepts** -- measured idle duration falls into a shallower bin than
  the sleep length (CPU was woken up early by a non-timer source)

Both metrics use exponential decay (shift right by `DECAY_SHIFT = 3`) and
pulse increments of `PULSE = 1024`.

**Per-CPU data:**
```c
/* teo.c:120 */
struct teo_bin {
    unsigned int intercepts;
    unsigned int hits;
};

/* teo.c:133 */
struct teo_cpu {
    s64 time_span_ns;
    s64 sleep_length_ns;
    struct teo_bin state_bins[CPUIDLE_STATE_MAX];
    unsigned int total;         /* grand total of all metrics */
    unsigned int tick_hits;     /* hits after TICK_NSEC */
};
```

**Selection algorithm** (`teo_select()`, `teo.c:282`):

1. Find the deepest enabled state (candidate). Compute sum of hits and
   intercepts for all shallower states.
2. If intercepts for shallower states exceed half of the total for the
   candidate and deeper bins, a shallower state is likely better. Traverse
   downward to find where the majority of intercepts accumulate.
3. Apply latency constraint (`exit_latency_ns <= latency_req`).
4. If the candidate has a small enough target residency, return it without
   querying the sleep length (optimization).
5. Otherwise, query `tick_nohz_get_sleep_length()`. If the sleep length
   is shorter than the candidate's target residency, find a shallower
   matching state.
6. Decide whether to stop the tick based on intercept patterns.

**TEO vs Menu:**
- TEO avoids the costly `tick_nohz_get_sleep_length()` call when recent
  wakeup patterns suggest a shallow state
- TEO's bin-based approach is simpler and more predictable than menu's
  correction factors
- Menu tends to be slightly more accurate for highly variable workloads

### 12.3 Ladder Governor

**Location:** `drivers/cpuidle/governors/ladder.c`

A simple step-up/step-down governor primarily used on periodic-tick systems
(not tickless). Steps to a deeper state when residency exceeds the
threshold, steps shallower when residency is too short. Rating varies.

### 12.4 Haltpoll Governor

**Location:** `drivers/cpuidle/governors/haltpoll.c`

Designed for virtualized environments. Polls for a configurable duration
before entering a deeper halt state, reducing VM exit/entry overhead.

---

## 13. Idle Loop Integration

The scheduler's idle path is the glue between cpuidle and the rest of the
kernel.

### 13.1 do_idle()

**Location:** `kernel/sched/idle.c:252`

The generic idle loop, reached from `cpu_startup_entry()`:

```c
static void do_idle(void)
{
    __current_set_polling();
    tick_nohz_idle_enter();

    while (!need_resched()) {
        local_irq_disable();
        arch_cpu_idle_enter();

        if (cpu_idle_force_poll || tick_check_broadcast_expired())
            cpu_idle_poll();         /* busy-poll loop */
        else
            cpuidle_idle_call();     /* governor-directed idle */

        arch_cpu_idle_exit();
    }

    tick_nohz_idle_exit();
    __current_clr_polling();
    flush_smp_call_function_queue();
    schedule_idle();
}
```

### 13.2 cpuidle_idle_call()

**Location:** `kernel/sched/idle.c:167`

The main cpuidle entry point from the idle loop:

```c
static void cpuidle_idle_call(void)
{
    struct cpuidle_device *dev = cpuidle_get_device();
    struct cpuidle_driver *drv = cpuidle_get_cpu_driver(dev);

    if (cpuidle_not_available(drv, dev)) {
        tick_nohz_idle_stop_tick();
        default_idle_call();              /* fallback: arch_cpu_idle() */
        return;
    }

    if (idle_should_enter_s2idle() || dev->forced_idle_latency_limit_ns) {
        /* Bypass governor: go for deepest state */
        if (idle_should_enter_s2idle()) {
            call_cpuidle_s2idle(drv, dev); /* suspend-to-idle */
        }
        tick_nohz_idle_stop_tick();
        next_state = cpuidle_find_deepest_state(drv, dev, max_latency_ns);
        call_cpuidle(drv, dev, next_state);
    } else {
        /* Normal path: ask governor */
        next_state = cpuidle_select(drv, dev, &stop_tick);

        if (stop_tick || tick_nohz_tick_stopped())
            tick_nohz_idle_stop_tick();
        else
            tick_nohz_idle_retain_tick();

        entered_state = call_cpuidle(drv, dev, next_state);
        cpuidle_reflect(dev, entered_state);
    }
}
```

### 13.3 cpuidle_enter_state()

**Location:** `drivers/cpuidle/cpuidle.c:215`

The core function that actually enters an idle state:

```
cpuidle_enter_state(dev, drv, index)
  1. Handle CPUIDLE_FLAG_TIMER_STOP:
     - Call tick_broadcast_enter(); if fails, find shallower state
  2. Handle CPUIDLE_FLAG_TLB_FLUSHED:
     - Call leave_mm() to flush TLBs
  3. sched_idle_set_state(target_state)     -- record planned state
  4. Record time_start = local_clock()
  5. stop_critical_timings()
  6. If !CPUIDLE_FLAG_RCU_IDLE:
     - ct_cpuidle_enter()                   -- put RCU in idle mode
  7. target_state->enter(dev, drv, index)   -- HARDWARE IDLE
  8. If !CPUIDLE_FLAG_RCU_IDLE:
     - ct_cpuidle_exit()                    -- exit RCU idle
  9. start_critical_timings()
  10. Record time_end, compute last_residency_ns
  11. sched_idle_set_state(NULL)             -- clear planned state
  12. If broadcast: tick_broadcast_exit()
  13. Update states_usage counters:
      - usage++, time_ns += diff
      - If diff < target_residency: above++ (entered too deep)
      - If deeper state was available: below++ (too shallow)
      - If enter() returned error: rejected++
```

---

## 14. CPUIdle Sysfs Interface

Per-CPU sysfs directory: `/sys/devices/system/cpu/cpuN/cpuidle/`

**Per-state subdirectory:** `.../cpuidle/stateN/`

| File | R/W | Description |
|---|---|---|
| `name` | R | State name (e.g., "C1", "C1E") |
| `desc` | R | State description |
| `latency` | R | Exit latency in microseconds |
| `residency` | R | Target residency in microseconds |
| `power` | R | Power consumption in mW (-1 if unknown) |
| `usage` | R | Number of times this state was entered |
| `time` | R | Total time spent in this state (us) |
| `above` | R | Times entered when shallower state sufficed |
| `below` | R | Times a deeper state was available |
| `rejected` | R | Times idle entry was rejected by driver |
| `disable` | RW | Disable this state (0=enabled, 1=disabled by user) |
| `default_status` | R | Whether state is disabled by driver default |
| `s2idle/usage` | R | Suspend-to-idle entry count |
| `s2idle/time` | R | Suspend-to-idle total time (us) |

**Driver subdirectory:** `.../cpuidle/driver/`

| File | R/W | Description |
|---|---|---|
| `name` | R | Driver name |

**Global:** `/sys/devices/system/cpu/cpuidle/`

| File | R/W | Description |
|---|---|---|
| `current_driver` | R | Currently loaded cpuidle driver |
| `current_governor` | R | Currently active cpuidle governor |
| `current_governor_ro` | R | Governor name (read-only variant) |
| `available_governors` | R | List of compiled-in governors |

---

## 15. Energy-Aware Scheduling Interaction

CPUFreq and CPUIdle are tightly coupled with the Energy Aware Scheduler
(EAS) in the CFS scheduling class:

### 15.1 Schedutil as EAS Prerequisite

EAS requires the schedutil governor to be active. When schedutil starts or
stops, it calls `sugov_eas_rebuild_sd()` which schedules
`rebuild_sched_domains_energy()` (`cpufreq_schedutil.c:619`). This
rebuilds scheduling domains with energy model information, enabling or
disabling EAS accordingly.

### 15.2 Energy Model Registration

CPUFreq drivers can register with the Energy Model via
`driver->register_em(policy)`. The common helper
`cpufreq_register_em_with_opp()` (`include/linux/cpufreq.h:1209`) calls
`dev_pm_opp_of_register_em()` to build the power/frequency table from
Operating Performance Points (OPPs).

### 15.3 Frequency Invariance

CPUFreq provides frequency-invariant capacity scaling:

```c
/* drivers/cpufreq/cpufreq.c:62 */
static DEFINE_STATIC_KEY_FALSE(cpufreq_freq_invariance);
```

When a target-style driver registers, this static branch is enabled
(`cpufreq.c:2965`), allowing the scheduler to use
`arch_scale_freq_capacity()` for accurate load tracking regardless of the
current operating frequency.

After each frequency transition, `cpufreq_freq_transition_end()` calls
`arch_set_freq_scale()` (`cpufreq.c:455`) to update the per-CPU frequency
scale factor.

### 15.4 CPUFreq Pressure

```c
/* include/linux/cpufreq.h:245 */
DECLARE_PER_CPU(unsigned long, cpufreq_pressure);
```

When thermal throttling or other constraints reduce the maximum available
frequency below the CPU's rated capacity, the `cpufreq_pressure` per-CPU
variable reflects this reduction. The scheduler uses
`cpufreq_get_pressure()` to account for reduced capacity in task placement
decisions.

### 15.5 Idle State Awareness in Scheduling

The scheduler tracks the current idle state of each CPU via
`sched_idle_set_state()` (`kernel/sched/idle.c:17`), which is called by
`cpuidle_enter_state()` before and after entering an idle state. This
information is used by the scheduler's `idle_get_state()` to factor in
the cost of waking a CPU from a deep idle state when selecting a target
CPU for task wakeup.

---

## 16. Locking Summary

| Subsystem | Lock | Type | Protects |
|---|---|---|---|
| CPUFreq | `cpufreq_driver_lock` | rwlock | `cpufreq_driver` pointer, per-CPU data |
| CPUFreq | `policy->rwsem` | rw_semaphore | policy fields during reads/writes |
| CPUFreq | `policy->transition_lock` | spinlock | `transition_ongoing` state |
| CPUFreq | `cpufreq_governor_mutex` | mutex | governor list iteration |
| Schedutil | `sg_policy->update_lock` | raw_spinlock | next_freq, deferred update state |
| Schedutil | `sg_policy->work_lock` | mutex | slow-path frequency change |
| Schedutil | `global_tunables_lock` | mutex | global tunables pointer |
| Intel PState | `intel_pstate_driver_lock` | mutex | driver registration |
| Intel PState | `intel_pstate_limits_lock` | mutex | performance limits update |
| CPUIdle | `cpuidle_lock` | mutex | device list, enabled state, driver registration |
| CPUIdle | (none on hot path) | -- | `select()`/`enter()` run locklessly per-CPU |

Note: The cpuidle `select()` and `enter()` paths are intentionally
lock-free on the hot path since they run on the local CPU with interrupts
disabled. Synchronization is only needed for device registration/
unregistration and governor changes.

---

## 17. Key Data Flows

### 17.1 Task Wakeup -> Frequency Scale-Up (Schedutil)

```
Task becomes runnable
  -> scheduler calls cpufreq_update_util()     [kernel/sched/cpufreq.c]
    -> sugov_update_single_freq() via update_util_data hook
      -> sugov_should_update_freq(): check rate_limit_us
      -> sugov_iowait_boost(): apply IO boost if applicable
      -> sugov_get_util(): read CPU utilization from PELT
      -> get_next_freq(): next_freq = 1.25 * ref_freq * util / max
      -> if fast_switch_enabled:
           cpufreq_driver_fast_switch(policy, next_freq)
             -> driver->fast_switch()           [direct, under rq->lock]
             -> policy->cur = new_freq
             -> arch_set_freq_scale()
         else:
           sugov_deferred_update()
             -> irq_work_queue()
               -> sugov_irq_work()
                 -> kthread_queue_work()
                   -> sugov_work()              [SCHED_DEADLINE kthread]
                     -> __cpufreq_driver_target()
```

### 17.2 CPU Goes Idle -> C-State Entry

```
schedule() finds no runnable tasks
  -> do_idle()                                  [kernel/sched/idle.c:252]
    -> tick_nohz_idle_enter()
    -> local_irq_disable()
    -> cpuidle_idle_call()                      [idle.c:167]
      -> cpuidle_select(drv, dev, &stop_tick)   [governor->select()]
        -> menu_select() or teo_select()
          -> predict idle duration
          -> select deepest state within prediction and latency_req
          -> decide whether to stop tick
      -> tick_nohz_idle_stop_tick() or tick_nohz_idle_retain_tick()
      -> cpuidle_enter(drv, dev, next_state)    [cpuidle.c:373]
        -> cpuidle_enter_state()                [cpuidle.c:215]
          -> sched_idle_set_state(target_state)
          -> ct_cpuidle_enter()                 [RCU idle mode]
          -> target_state->enter()              [HARDWARE IDLE -- CPU halts]
          -> [WAKEUP INTERRUPT]
          -> ct_cpuidle_exit()
          -> compute last_residency_ns
          -> update states_usage counters
      -> cpuidle_reflect(dev, entered_state)    [governor->reflect()]
        -> update prediction metrics
    -> tick_nohz_idle_exit()
    -> schedule_idle()
```

### 17.3 Thermal Throttling -> Frequency Reduction

```
Thermal zone temperature exceeds trip point
  -> thermal governor (step_wise/power_allocator)
    -> set cooling device state
      -> cpufreq cooling device: cpufreq_set_cur_state()
        -> freq_qos_update_request(&policy->max_freq_req, new_max)
          -> policy->nb_max notifier fires
            -> cpufreq_update_limits()
              -> refresh_frequency_limits(policy)
                -> cpufreq_governor_limits(policy)
                  -> governor->limits(policy)
                    -> schedutil: cpufreq_policy_apply_limits()
                      -> __cpufreq_driver_target(policy, policy->max, ...)
```

### 17.4 Intel HWP Flow (Active Mode)

```
User writes "performance" to scaling_governor
  -> store_scaling_governor()                   [cpufreq.c:824]
    -> cpufreq_set_policy(policy, new_gov, CPUFREQ_POLICY_PERFORMANCE)
      -> intel_pstate_set_policy()              [intel_pstate.c:2821]
        -> cpu->policy = CPUFREQ_POLICY_PERFORMANCE
        -> intel_pstate_update_perf_limits()
          -> cpu->min_perf_ratio = cpu->max_perf_ratio
        -> intel_pstate_hwp_set(cpu)            [intel_pstate.c:1148]
          -> rdmsrl(MSR_HWP_REQUEST, value)
          -> value = (max << 8) | min           [set min = max]
          -> value |= epp_for_performance       [EPP = 0]
          -> wrmsrl(MSR_HWP_REQUEST, value)
```

---

## 18. Source File Reference

### CPUFreq Core

```
include/linux/cpufreq.h                   - All public structs and API
drivers/cpufreq/cpufreq.c                 - Core framework
drivers/cpufreq/freq_table.c              - Frequency table helpers
drivers/cpufreq/cpufreq_governor.c        - Shared dbs governor framework
```

### CPUFreq Governors

```
drivers/cpufreq/cpufreq_performance.c     - Performance governor
drivers/cpufreq/cpufreq_powersave.c       - Powersave governor
drivers/cpufreq/cpufreq_ondemand.c        - Ondemand governor
drivers/cpufreq/cpufreq_conservative.c    - Conservative governor
drivers/cpufreq/cpufreq_userspace.c       - Userspace governor
kernel/sched/cpufreq_schedutil.c          - Schedutil governor
```

### CPUFreq Drivers

```
drivers/cpufreq/intel_pstate.c            - Intel HWP / P-state driver
drivers/cpufreq/acpi-cpufreq.c            - ACPI P-state driver
drivers/cpufreq/amd-pstate.c              - AMD CPPC driver
drivers/cpufreq/cpufreq-dt.c              - Device-tree based driver
```

### CPUIdle Core

```
include/linux/cpuidle.h                   - All structs and API
drivers/cpuidle/cpuidle.c                 - Core: cpuidle_enter_state(), select, reflect
drivers/cpuidle/driver.c                  - Driver registration
drivers/cpuidle/governor.c                - Governor registration
drivers/cpuidle/sysfs.c                   - Sysfs interface
drivers/cpuidle/cpuidle.h                 - Internal declarations
```

### CPUIdle Governors

```
drivers/cpuidle/governors/menu.c          - Menu governor (rating=20)
drivers/cpuidle/governors/teo.c           - TEO governor (rating=19)
drivers/cpuidle/governors/ladder.c        - Ladder governor
drivers/cpuidle/governors/haltpoll.c      - HaltPoll governor (virtualization)
drivers/cpuidle/governors/gov.h           - Shared governor header
```

### CPUIdle Drivers

```
drivers/idle/intel_idle.c                 - Intel MWAIT-based idle driver
drivers/cpuidle/cpuidle-psci.c            - ARM PSCI idle driver
drivers/cpuidle/cpuidle-haltpoll.c        - KVM halt-polling idle driver
```

### Idle Loop

```
kernel/sched/idle.c                       - do_idle(), cpuidle_idle_call()
```
