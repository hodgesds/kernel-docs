# Power Management Subsystem

## Overview

The Linux kernel power management subsystem spans CPU frequency/idle management, device
runtime PM, system sleep states, thermal management, and energy modeling. It coordinates
hardware power state transitions across the entire platform — from individual device D-states
through CPU C/P-states to full system S-states.

---

## 1. CPUFreq — CPU Frequency Scaling

CPUFreq manages dynamic voltage and frequency scaling (DVFS) for CPUs. A **policy** groups
CPUs that share a clock domain; a **driver** interfaces with the hardware; a **governor**
decides when and how to change frequency.

### Source Files

```
include/linux/cpufreq.h              - All public structs and API
drivers/cpufreq/cpufreq.c            - Core framework
drivers/cpufreq/freq_table.c         - Frequency table helpers
drivers/cpufreq/cpufreq_governor.c   - Shared dbs governor framework
drivers/cpufreq/cpufreq_ondemand.c   - Ondemand governor
drivers/cpufreq/cpufreq_conservative.c - Conservative governor
drivers/cpufreq/cpufreq_performance.c  - Performance governor
drivers/cpufreq/cpufreq_powersave.c    - Powersave governor
kernel/sched/cpufreq_schedutil.c     - Schedutil governor
drivers/cpufreq/acpi-cpufreq.c       - ACPI P-state driver
drivers/cpufreq/intel_pstate.c       - Intel HWP driver
drivers/cpufreq/amd-pstate.c         - AMD CPPC driver
```

### struct cpufreq_policy (include/linux/cpufreq.h)

Central structure representing one frequency-scaling domain. One policy covers one or
more CPUs that share a clock and must scale together.

```c
struct cpufreq_policy {
    cpumask_var_t    cpus;              /* online CPUs sharing this policy */
    cpumask_var_t    related_cpus;      /* online + offline CPUs in domain */
    cpumask_var_t    real_cpus;         /* related and present CPUs */
    unsigned int     shared_type;       /* ACPI sharing: NONE/HW/ALL/ANY */
    unsigned int     cpu;               /* managing CPU (must be online) */
    struct clk      *clk;              /* optional clock handle */

    struct cpufreq_cpuinfo cpuinfo;    /* HW min/max freq and transition latency */
    unsigned int     min;              /* current policy min freq (kHz) */
    unsigned int     max;              /* current policy max freq (kHz) */
    unsigned int     cur;              /* current actual freq (kHz) */
    unsigned int     suspend_freq;     /* freq to set during system suspend */
    unsigned int     policy;           /* POWERSAVE or PERFORMANCE (setpolicy drivers) */

    struct cpufreq_governor *governor; /* active governor */
    void            *governor_data;    /* governor private state */

    struct freq_constraints constraints; /* PM QoS frequency constraints */
    struct cpufreq_frequency_table *freq_table; /* available freq steps */

    struct rw_semaphore rwsem;         /* read: policy readers; write: hotplug/updates */
    bool             fast_switch_possible; /* driver supports fast switching */
    bool             fast_switch_enabled;  /* governor enabled fast switch */
    bool             strict_target;    /* governor wants exact freq */
    unsigned int     transition_delay_us; /* min interval between switches */
    bool             dvfs_possible_from_any_cpu; /* cross-CPU DVFS */

    unsigned int     cached_target_freq;   /* last resolved target freq */
    unsigned int     cached_resolved_idx;  /* index of last resolved freq */

    bool             transition_ongoing; /* freq transition in progress */
    spinlock_t       transition_lock;    /* protects transition state */
    wait_queue_head_t transition_wait;

    struct cpufreq_stats *stats;       /* time-in-state statistics */
    void            *driver_data;      /* driver private data */
    struct thermal_cooling_device *cdev; /* cooling device if CPUFREQ_IS_COOLING_DEV */
};
```

- `cpus` vs `related_cpus`: `cpus` contains only currently online CPUs. `related_cpus`
  includes offline CPUs. The `cpu` field identifies which CPU manages the policy.
- `rwsem`: Primary synchronization. Read-locked by code reading the policy; write-locked
  by hotplug and `cpufreq_update_policy()`.
- `fast_switch_possible` / `fast_switch_enabled`: When both true, the governor calls
  `cpufreq_driver_fast_switch()` directly from the scheduler hot path without locks.
- `transition_lock` + `transition_wait`: Used by `cpufreq_freq_transition_begin/end()`
  to serialize frequency transitions.
- `freq_table`: Null-terminated array of `struct cpufreq_frequency_table`. Entries with
  `frequency == CPUFREQ_ENTRY_INVALID` are skipped; `CPUFREQ_TABLE_END` terminates.

### struct cpufreq_driver (include/linux/cpufreq.h)

Hardware-specific driver interface registered with `cpufreq_register_driver()`.

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    u16     flags;
    void   *driver_data;

    /* mandatory */
    int   (*init)(struct cpufreq_policy *policy);
    int   (*verify)(struct cpufreq_policy_data *policy);

    /* choose one: setpolicy OR target/target_index */
    int   (*setpolicy)(struct cpufreq_policy *policy);
    int   (*target_index)(struct cpufreq_policy *, unsigned int index);
    unsigned int (*fast_switch)(struct cpufreq_policy *, unsigned int target_freq);
    void  (*adjust_perf)(unsigned int cpu, unsigned long min_perf,
                         unsigned long target_perf, unsigned long capacity);

    unsigned int (*get)(unsigned int cpu);      /* read current HW freq */
    void  (*exit)(struct cpufreq_policy *policy);
    int   (*suspend)(struct cpufreq_policy *policy);
    int   (*resume)(struct cpufreq_policy *policy);
    void  (*register_em)(struct cpufreq_policy *policy);
};
```

Driver flags (`u16 flags`):
- `CPUFREQ_NEED_UPDATE_LIMITS` — call driver even if target freq unchanged but limits changed
- `CPUFREQ_IS_COOLING_DEV` — auto-register as thermal cooling device
- `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` — each policy gets its own governor with own tunables
- `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` — prohibit dynamic governors (e.g., HWP-managed)

### Frequency Change Paths

**Normal path (via `target_index`):**
1. Governor calls `cpufreq_driver_target()`.
2. Takes `policy->rwsem` write lock.
3. `__cpufreq_driver_target()` resolves target into a table index via
   `cpufreq_frequency_table_target()`.
4. Calls `driver->target_index(policy, index)`.

**Fast switch path (scheduler hot path):**
1. Governor calls `cpufreq_driver_fast_switch(policy, target_freq)`.
2. Directly calls `driver->fast_switch()` — no rwsem, runs in scheduler context.
3. Updates `policy->cur` on return.

### struct cpufreq_governor (include/linux/cpufreq.h)

```c
struct cpufreq_governor {
    char  name[CPUFREQ_NAME_LEN];
    int  (*init)(struct cpufreq_policy *policy);
    void (*exit)(struct cpufreq_policy *policy);
    int  (*start)(struct cpufreq_policy *policy);
    void (*stop)(struct cpufreq_policy *policy);
    void (*limits)(struct cpufreq_policy *policy);
    u8    flags;
};
```

Governor flags:
- `CPUFREQ_GOV_DYNAMIC_SWITCHING` — governor changes freq dynamically
- `CPUFREQ_GOV_STRICT_TARGET` — governor wants exact requested freq

### Available Governors

**performance** — always sets `policy->max`. Also serves as fallback governor.

**powersave** — always sets `policy->min`.

**ondemand** — samples CPU idle time at configurable intervals. Default up-threshold: 80%.
Above threshold jumps to max; below steps down. Uses dbs framework. Tunables:
`sampling_rate`, `up_threshold`, `sampling_down_factor`, `powersave_bias`.

**conservative** — like ondemand but steps frequency gradually. Tunables: `up_threshold`,
`down_threshold`, `freq_step`.

**schedutil** (`kernel/sched/cpufreq_schedutil.c`) — directly integrated with scheduler
utilization tracking via `update_util_data` callback. Called from `cpufreq_update_util()` in
the scheduler hot path. Computes: `next_freq = 1.25 * max_freq * util / max` (25% headroom).
Uses `fast_switch` when available. Otherwise queues `irq_work` -> `kthread_work` on a
dedicated `sugov_worker` kthread. Tunables: `rate_limit_us`.

### Per-Policy vs Global Governors

When `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` is set, each policy gets its own sysfs directory
for governor tunables (`cpu/cpuN/cpufreq/<governor>/`). Without the flag, all policies share
a single global governor directory. `struct gov_attr_set` handles this sharing with a
`usage_count`.

---

## 2. CPUIdle — CPU Idle State Management

CPUIdle selects which idle state (C-state) a CPU enters when it has no work. A **driver**
declares available states; a **governor** picks which one to enter each time.

### Source Files

```
include/linux/cpuidle.h              - All structs and API
drivers/cpuidle/cpuidle.c            - Core: cpuidle_enter_state(), select, reflect
drivers/cpuidle/governors/menu.c     - Menu governor
drivers/cpuidle/governors/teo.c      - TEO governor
drivers/cpuidle/governors/ladder.c   - Ladder governor
drivers/cpuidle/governors/haltpoll.c - HaltPoll governor (virtualization)
```

### struct cpuidle_state (include/linux/cpuidle.h)

Describes one C-state that hardware supports.

```c
struct cpuidle_state {
    char  name[CPUIDLE_NAME_LEN];      /* e.g., "C1", "C2" */
    char  desc[CPUIDLE_DESC_LEN];
    s64   exit_latency_ns;             /* time to wake from this state */
    s64   target_residency_ns;         /* min time that makes entry worthwhile */
    unsigned int flags;
    int   power_usage;                 /* mW consumed */

    int (*enter)(struct cpuidle_device *dev,
                 struct cpuidle_driver *drv, int index);
    void (*enter_dead)(struct cpuidle_device *dev, int index);
    int (*enter_s2idle)(struct cpuidle_device *dev,
                        struct cpuidle_driver *drv, int index);
};
```

- `exit_latency_ns` — how long it takes to *resume* from this state
- `target_residency_ns` — how long the CPU must stay idle to justify the entry overhead
- Governors ensure predicted idle duration >= `target_residency` before selecting a state

State flags:
- `CPUIDLE_FLAG_POLLING` — state is a busy-wait polling loop
- `CPUIDLE_FLAG_TIMER_STOP` — local timer stops; needs broadcast timer
- `CPUIDLE_FLAG_TLB_FLUSHED` — state flushes TLBs
- `CPUIDLE_FLAG_RCU_IDLE` — state handles its own RCU idle tracking

### struct cpuidle_device (include/linux/cpuidle.h)

Per-CPU idle device state.

```c
struct cpuidle_device {
    unsigned int  registered:1;
    unsigned int  enabled:1;
    unsigned int  cpu;
    ktime_t       next_hrtimer;        /* next hrtimer expiry */
    int           last_state_idx;
    u64           last_residency_ns;   /* actual time in last state */
    u64           poll_limit_ns;
    struct cpuidle_state_usage states_usage[CPUIDLE_STATE_MAX];
};
```

`states_usage[i]` contains per-state counters: `usage` (times entered), `time_ns` (total
time), `above` (residency shorter than target), `below` (deeper state was available).

### struct cpuidle_driver (include/linux/cpuidle.h)

```c
struct cpuidle_driver {
    const char     *name;
    struct cpuidle_state states[CPUIDLE_STATE_MAX]; /* ordered: shallow -> deep */
    int             state_count;
    int             safe_state_index;  /* failsafe shallow state */
    struct cpumask *cpumask;           /* CPUs this driver handles */
};
```

States must be ordered by decreasing power consumption (state 0 = shallowest). Maximum
10 states (`CPUIDLE_STATE_MAX`).

### struct cpuidle_governor (include/linux/cpuidle.h)

```c
struct cpuidle_governor {
    char  name[CPUIDLE_NAME_LEN];
    unsigned int rating;               /* higher = preferred */
    int  (*select)(struct cpuidle_driver *drv, struct cpuidle_device *dev,
                   bool *stop_tick);   /* returns chosen state index */
    void (*reflect)(struct cpuidle_device *dev, int index);
};
```

`select()` is called every time a CPU goes idle. `stop_tick` tells core whether to stop
the local tick. `reflect()` is called after wakeup for governor learning.

### Idle Entry Path

The scheduler idle path (`kernel/sched/idle.c`) calls:

1. `cpuidle_select(drv, dev, &stop_tick)` — governor's `select()`
2. `cpuidle_enter(drv, dev, index)` — calls `cpuidle_enter_state()`
3. `cpuidle_reflect(dev, index)` — governor's `reflect()`

Inside `cpuidle_enter_state()` (`drivers/cpuidle/cpuidle.c`):
1. Handle `CPUIDLE_FLAG_TIMER_STOP` (broadcast timer) and `TLB_FLUSHED` flags
2. Call `sched_idle_set_state(target_state)`
3. Record start time
4. Call `ct_cpuidle_enter()` (put RCU in idle mode) unless `CPUIDLE_FLAG_RCU_IDLE`
5. Call `target_state->enter(dev, drv, index)` — driver's low-level idle entry
6. Record end time, compute `last_residency_ns`
7. Update `states_usage` counters

### Menu Governor

Uses two independent prediction methods:
1. **Energy break-even** — predicts idle duration using next timer, corrected by a
   12-bucket historical correction factor
2. **Repeatable-interval detector** — tracks last 8 intervals; if low std deviation,
   uses their average
3. **PM QoS constraint** — eliminates states exceeding the CPU's QoS latency requirement

Selects deepest state whose `target_residency_ns` <= predicted idle duration.

### TEO (Timer Events Oriented) Governor

Based on observation that timer interrupts dominate wakeup patterns. Uses bins aligned
with state `target_residency` boundaries. Tracks "hits" (timer wakeup at expected time)
vs "intercepts" (early non-timer wakeup). Selects state based on which pattern dominates.

---

## 3. System Sleep (Suspend/Hibernate)

### Source Files

```
kernel/power/suspend.c               - Main suspend state machine
kernel/power/hibernate.c             - Hibernation (snapshot, swap, resume)
kernel/power/main.c                  - /sys/power/state handler, pm_suspend()
drivers/base/power/main.c            - Device PM list (dpm_suspend_*, dpm_resume_*)
include/linux/suspend.h              - platform_suspend_ops, suspend states
include/linux/pm.h                   - dev_pm_ops, dev_pm_info
include/linux/pm_wakeup.h            - wakeup_source
```

### Suspend States

```c
#define PM_SUSPEND_ON       0   /* running */
#define PM_SUSPEND_TO_IDLE  1   /* s2idle: freeze tasks + suspend devices + idle CPUs */
#define PM_SUSPEND_STANDBY  2   /* ACPI S1: light sleep */
#define PM_SUSPEND_MEM      3   /* ACPI S3: suspend to RAM */
```

User writes to `/sys/power/state`: `freeze`, `standby`, `mem`.

### Suspend State Machine

`enter_state()` in `kernel/power/suspend.c` drives the full sequence:

```
enter_state(PM_SUSPEND_MEM)
  suspend_prepare()
    pm_notifier_call_chain(PM_SUSPEND_PREPARE)      <- PM notifiers
    suspend_freeze_processes()                       <- task freezer
  suspend_devices_and_enter()
    platform_suspend_begin()                         <- suspend_ops->begin()
    dpm_suspend_start(PMSG_SUSPEND)
      dpm_prepare()         <- dev_pm_ops::prepare() for all devices
      dpm_suspend()         <- dev_pm_ops::suspend() for all devices
    suspend_enter()
      platform_suspend_prepare()                     <- suspend_ops->prepare()
      dpm_suspend_late()    <- dev_pm_ops::suspend_late()
      dpm_suspend_noirq()   <- dev_pm_ops::suspend_noirq() (IRQs off)
      pm_sleep_disable_secondary_cpus()              <- offline non-boot CPUs
      arch_suspend_disable_irqs()
      syscore_suspend()                              <- syscore_ops::suspend()
      suspend_ops->enter(state)                      <- PLATFORM ENTERS SLEEP
      syscore_resume()
      arch_suspend_enable_irqs()
      pm_sleep_enable_secondary_cpus()
      dpm_resume_noirq() / dpm_resume_early()
    dpm_resume_end()
      dpm_resume()          <- dev_pm_ops::resume()
      dpm_complete()        <- dev_pm_ops::complete()
    platform_resume_end()
  suspend_finish()
    suspend_thaw_processes()
    pm_notifier_call_chain(PM_POST_SUSPEND)
```

For `PM_SUSPEND_TO_IDLE` (s2idle), instead of `suspend_ops->enter()`, the system enters
`s2idle_loop()` which repeatedly calls `cpuidle_enter_s2idle()` on all CPUs until a real
wakeup is detected.

### struct dev_pm_ops (include/linux/pm.h)

Full PM callback set for drivers:

```c
struct dev_pm_ops {
    /* system suspend/resume */
    int  (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int  (*suspend)(struct device *dev);
    int  (*resume)(struct device *dev);
    int  (*suspend_late)(struct device *dev);
    int  (*resume_early)(struct device *dev);
    int  (*suspend_noirq)(struct device *dev);
    int  (*resume_noirq)(struct device *dev);

    /* hibernation */
    int  (*freeze)(struct device *dev);
    int  (*thaw)(struct device *dev);
    int  (*poweroff)(struct device *dev);
    int  (*restore)(struct device *dev);
    /* ... _late/_early/_noirq variants for each */

    /* runtime PM */
    int  (*runtime_suspend)(struct device *dev);
    int  (*runtime_resume)(struct device *dev);
    int  (*runtime_idle)(struct device *dev);
};
```

`prepare()` returning a positive value triggers **direct-complete optimization**: the device
is left runtime-suspended for the entire transition and only `complete()` is called on resume.

DPM flags on `dev->power.driver_flags`:
- `DPM_FLAG_NO_DIRECT_COMPLETE` — force full callback sequence
- `DPM_FLAG_SMART_PREPARE` — honor positive `prepare()` return
- `DPM_FLAG_SMART_SUSPEND` — skip callbacks if runtime-suspended
- `DPM_FLAG_MAY_SKIP_RESUME` — allow skipping noirq/early resume

### struct platform_suspend_ops (include/linux/suspend.h)

Low-level platform interface registered with `suspend_set_ops()`:

```c
struct platform_suspend_ops {
    int  (*valid)(suspend_state_t state);
    int  (*begin)(suspend_state_t state);
    int  (*prepare)(void);
    int  (*prepare_late)(void);
    int  (*enter)(suspend_state_t state);    /* actually enter sleep (mandatory) */
    void (*wake)(void);
    void (*finish)(void);
    bool (*suspend_again)(void);
    void (*end)(void);
    void (*recover)(void);
};
```

### PM Notifiers

Registered with `register_pm_notifier()`. Called at:
- `PM_SUSPEND_PREPARE` / `PM_POST_SUSPEND` — around suspend
- `PM_HIBERNATION_PREPARE` / `PM_POST_HIBERNATION` — around hibernation
- `PM_RESTORE_PREPARE` / `PM_POST_RESTORE` — around hibernate restore

### struct wakeup_source (include/linux/pm_wakeup.h)

```c
struct wakeup_source {
    const char      *name;
    struct list_head entry;             /* global wakeup sources list */
    spinlock_t       lock;
    struct wake_irq *wakeirq;
    struct timer_list timer;
    ktime_t total_time;                 /* total activation time */
    ktime_t max_time;                   /* longest single activation */
    unsigned long event_count;
    unsigned long active_count;
    unsigned long wakeup_count;         /* times it may have aborted suspend */
    struct device  *dev;
    bool active:1;
};
```

Key API:
- `wakeup_source_register(dev, name)` — allocate and register
- `__pm_stay_awake(ws)` — mark active (prevents suspend)
- `__pm_relax(ws)` — deactivate
- `pm_wakeup_event(dev, msec)` — signal a timed wakeup event
- `pm_wakeup_pending()` — check if wakeup should abort suspend

### Hibernate

Hibernate path (`kernel/power/hibernate.c`):
1. `PM_HIBERNATION_PREPARE` notifier, freeze tasks
2. Full device suspend sequence (`dpm_prepare` through `dpm_suspend_noirq`)
3. `create_image()` — snapshot RAM (the hibernation image)
4. Resume devices, write image to swap, power off

On resume: bootloader loads kernel, kernel detects hibernation image on swap, reads and
restores memory state, resumes execution at saved point.

---

## 4. Runtime PM

Runtime PM (RPM) allows individual devices to enter low-power states when idle, independent
of system sleep. It uses a reference-counting model: devices suspend when their usage count
drops to zero.

### Source Files

```
drivers/base/power/runtime.c         - Core RPM state machine
include/linux/pm_runtime.h           - Public API
include/linux/pm.h                   - dev_pm_info, rpm_status
```

### RPM States

```c
enum rpm_status {
    RPM_ACTIVE   = 0,   /* fully operational */
    RPM_RESUMING,       /* runtime_resume() executing */
    RPM_SUSPENDED,      /* runtime_suspend() completed */
    RPM_SUSPENDING,     /* runtime_suspend() executing */
};
```

### Key Fields in dev_pm_info (embedded in struct device)

```c
struct dev_pm_info {
    struct hrtimer   suspend_timer;     /* autosuspend delay timer */
    struct work_struct work;            /* deferred RPM work item */
    wait_queue_head_t wait_queue;
    atomic_t         usage_count;       /* ref count: 0 = can suspend */
    atomic_t         child_count;       /* active children count */
    unsigned int     disable_depth;     /* pm_runtime_disable() nesting */
    bool             ignore_children;   /* suspend regardless of children */
    bool             no_callbacks;      /* no RPM callbacks defined */
    bool             irq_safe;          /* callbacks safe in atomic context */
    bool             use_autosuspend;
    enum rpm_status  runtime_status;
    int              runtime_error;
    int              autosuspend_delay;  /* ms; -1 = never */
    u64              last_busy;
    u64              active_time;
    u64              suspended_time;
};
```

### RPM API

**Getting (preventing suspend):**
- `pm_runtime_get_sync(dev)` — increment usage_count; synchronously resume if suspended
- `pm_runtime_get(dev)` — increment + async resume
- `pm_runtime_get_noresume(dev)` — increment only
- `pm_runtime_get_if_active(dev)` — increment only if currently RPM_ACTIVE

**Putting (allowing suspend):**
- `pm_runtime_put_sync(dev)` — decrement; if 0, synchronously idle+suspend
- `pm_runtime_put(dev)` — decrement; async idle
- `pm_runtime_put_autosuspend(dev)` — decrement; schedule autosuspend after delay
- `pm_runtime_put_noidle(dev)` — decrement only

**Autosuspend:**
- `pm_runtime_set_autosuspend_delay(dev, delay_ms)` — set delay
- `pm_runtime_use_autosuspend(dev)` — enable autosuspend mode
- `pm_runtime_mark_last_busy(dev)` — update `last_busy` timestamp

**Enable/disable:**
- `pm_runtime_enable(dev)` — decrement `disable_depth`; RPM active at depth 0
- `pm_runtime_disable(dev)` — increment `disable_depth`; RPM frozen

### RPM Suspend Check (drivers/base/power/runtime.c)

`rpm_check_suspend_allowed()` returns error if device cannot suspend:
- `-EACCES` — `disable_depth > 0`
- `-EAGAIN` — `usage_count > 0`
- `-EBUSY` — `child_count > 0` and `!ignore_children`
- `-EPERM` — PM QoS resume latency is 0 (must stay active)

### Callback Resolution Order

When RPM needs to call a callback (e.g., `runtime_suspend`), it checks in order:
1. `dev->pm_domain->ops`
2. `dev->type->pm`
3. `dev->class->pm`
4. `dev->bus->pm`
5. `dev->driver->pm` (fallback)

### Device Links and RPM

Device links with `DL_FLAG_PM_RUNTIME` cause RPM to automatically resume supplier
devices before a consumer's `runtime_resume` (via `rpm_get_suppliers()`), and to attempt
supplier suspension after consumer suspends (via `rpm_suspend_suppliers()`).

---

## 5. Generic Power Domain (genpd)

genpd groups devices into power domains that can be collectively powered on/off. Domains
can form hierarchies — a parent cannot power off until all children have.

### Source Files

```
drivers/pmdomain/core.c              - Main genpd implementation
include/linux/pm_domain.h            - generic_pm_domain, API
```

### struct generic_pm_domain (include/linux/pm_domain.h)

```c
struct generic_pm_domain {
    struct device        dev;
    struct dev_pm_domain domain;           /* replaces bus/class PM ops */
    struct list_head     parent_links;
    struct list_head     child_links;
    struct list_head     dev_list;          /* attached devices */
    struct dev_power_governor *gov;
    const char          *name;
    atomic_t             sd_count;         /* sub-domains with power on */
    enum gpd_status      status;           /* GENPD_STATE_ON or _OFF */
    unsigned int         device_count;
    unsigned int         performance_state;
    cpumask_var_t        cpus;             /* for CPU domains */

    int (*power_off)(struct generic_pm_domain *domain);
    int (*power_on)(struct generic_pm_domain *domain);
    int (*set_performance_state)(struct generic_pm_domain *genpd, unsigned int state);

    unsigned int         flags;            /* GENPD_FLAG_* */
    struct genpd_power_state *states;      /* power state array */
    unsigned int         state_count;
    unsigned int         state_idx;        /* state to enter when off */
};
```

GENPD flags:
- `GENPD_FLAG_PM_CLK` — use PM clock framework to gate clocks
- `GENPD_FLAG_IRQ_SAFE` — `power_on/off()` safe in atomic context (uses spinlock)
- `GENPD_FLAG_ALWAYS_ON` — never power off
- `GENPD_FLAG_CPU_DOMAIN` — covers CPUs; enables last-man-standing
- `GENPD_FLAG_RPM_ALWAYS_ON` — never power off at runtime (only system suspend)
- `GENPD_FLAG_MIN_RESIDENCY` — governor uses minimum residency constraint

### struct genpd_power_state (include/linux/pm_domain.h)

```c
struct genpd_power_state {
    s64  power_off_latency_ns;
    s64  power_on_latency_ns;
    s64  residency_ns;           /* min time off to justify entry */
    u64  usage;                  /* times entered */
    u64  rejected;               /* times skipped */
    u64  idle_time;              /* total time in state */
};
```

### Domain Hierarchy

Sub-domains linked via `struct gpd_link`. A parent cannot power off until `sd_count == 0`.
`pm_genpd_add_subdomain()` establishes parent-child links. Power-off propagates leaf to root.

### Device Attachment

`dev_pm_domain_attach(dev, power_on)` or `genpd_dev_pm_attach(dev)` from DT. genpd installs
itself as `dev->pm_domain`, replacing bus/type PM ops. The `generic_pm_domain_data`
per-device structure holds device suspend/resume latencies and performance state.

---

## 6. Thermal Management

The thermal framework monitors temperatures and controls cooling to prevent overheating.
Thermal zones report temperature; cooling devices reduce heat; governors decide how
aggressively to cool.

### Source Files

```
drivers/thermal/thermal_core.c       - Zone/cooling registration, governor dispatch
drivers/thermal/thermal_core.h       - Internal structs
drivers/thermal/gov_step_wise.c      - Step-wise governor
drivers/thermal/gov_power_allocator.c - Power allocator governor (PID)
include/linux/thermal.h              - Public API
```

### struct thermal_zone_device (drivers/thermal/thermal_core.h)

```c
struct thermal_zone_device {
    int id;
    char type[THERMAL_NAME_LENGTH];
    struct device device;
    enum thermal_device_mode mode;     /* enabled or disabled */
    void *devdata;
    int num_trips;
    unsigned long passive_delay_jiffies;   /* poll interval when passive cooling */
    unsigned long polling_delay_jiffies;   /* normal poll interval */
    int temperature;                   /* current temp in milliCelsius */
    int last_temperature;
    int passive;                       /* 1 if above a passive trip */
    struct thermal_zone_device_ops ops;
    struct thermal_zone_params *tzp;   /* governor params */
    struct thermal_governor *governor;
    void *governor_data;
    struct mutex lock;
    struct delayed_work poll_queue;
    struct thermal_trip_desc trips[];   /* flexible array of trip descriptors */
};
```

### struct thermal_trip (include/linux/thermal.h)

```c
struct thermal_trip {
    int temperature;           /* trip point in milliCelsius */
    int hysteresis;            /* hysteresis: trip-down = temp - hysteresis */
    enum thermal_trip_type type;
    u8  flags;
    void *priv;
};
```

Trip types:
- `THERMAL_TRIP_ACTIVE` — trigger active cooling (fans)
- `THERMAL_TRIP_PASSIVE` — trigger passive cooling (freq reduction)
- `THERMAL_TRIP_HOT` — notify drivers to prepare for shutdown
- `THERMAL_TRIP_CRITICAL` — system must shut down immediately

### struct thermal_cooling_device (include/linux/thermal.h)

```c
struct thermal_cooling_device {
    int id;
    const char *type;
    unsigned long max_state;       /* highest cooling state */
    struct device device;
    void *devdata;
    const struct thermal_cooling_device_ops *ops;
    struct mutex lock;
    struct list_head thermal_instances;
};
```

`thermal_cooling_device_ops`:
- `get_max_state()` / `get_cur_state()` / `set_cur_state()` — basic state control
  (0 = no cooling, max = max cooling)
- `get_requested_power()` / `state2power()` / `power2state()` — for power_allocator governor

### struct thermal_governor (drivers/thermal/thermal_core.h)

```c
struct thermal_governor {
    const char *name;
    int  (*bind_to_tz)(struct thermal_zone_device *tz);
    void (*unbind_from_tz)(struct thermal_zone_device *tz);
    void (*trip_crossed)(struct thermal_zone_device *tz,
                         const struct thermal_trip *trip, bool crossed_up);
    void (*manage)(struct thermal_zone_device *tz);
};
```

**step_wise governor** — increases cooling state by 1 on trip crossing up, decreases by 1
when temp drops below `trip->temperature - trip->hysteresis`. Simple, deterministic.

**power_allocator governor** — PID controller distributing a total power budget among
cooling devices. Uses `sustainable_power` from `thermal_zone_params` and PID gains
`k_po`, `k_pu`, `k_i`, `k_d`. Requires cooling devices to implement power-related ops.

---

## 7. Energy Model

The Energy Model (EM) provides power/frequency tables that the Energy Aware Scheduler
(EAS) uses for optimal task placement.

### Source Files

```
kernel/power/energy_model.c          - Registration, update, RCU management
include/linux/energy_model.h         - em_perf_domain, em_perf_state, API
```

### struct em_perf_state (include/linux/energy_model.h)

```c
struct em_perf_state {
    unsigned long performance;  /* CPU capacity at this freq */
    unsigned long frequency;    /* frequency in kHz */
    unsigned long power;        /* power in uW consumed by 1 CPU */
    unsigned long cost;         /* = power * max_freq / frequency (precomputed) */
    unsigned long flags;        /* EM_PERF_STATE_INEFFICIENT if dominated */
};
```

The `cost` field is precomputed so energy estimation in `em_cpu_energy()` reduces to
`energy = ps->cost * sum_util`.

### struct em_perf_domain (include/linux/energy_model.h)

```c
struct em_perf_domain {
    struct em_perf_table __rcu *em_table;   /* RCU-protected table */
    int nr_perf_states;
    int min_perf_state;
    int max_perf_state;
    unsigned long flags;                   /* EM_PERF_DOMAIN_* */
    unsigned long cpus[];                  /* cpumask */
};
```

Flags:
- `EM_PERF_DOMAIN_MICROWATTS` — power values in uW
- `EM_PERF_DOMAIN_SKIP_INEFFICIENCIES` — skip dominated states
- `EM_PERF_DOMAIN_ARTIFICIAL` — synthetic power values

### Registration

```c
int em_dev_register_perf_domain(struct device *dev, unsigned int nr_states,
                                struct em_data_callback *cb,
                                cpumask_t *span, bool microwatts);
```

The callback's `active_power(dev, *power, *freq)` is called once per frequency step.
CPUFreq drivers register via `driver->register_em(policy)`.

### EAS Energy Estimation

`em_cpu_energy(pd, max_util, sum_util, allowed_cpu_cap)`:
1. Cap `max_util` to `allowed_cpu_cap` (thermal constraint)
2. Find lowest performance state >= `max_util`
3. Return `ps->cost * sum_util`

Called under `rcu_read_lock()` from `kernel/sched/fair.c` during task placement.

---

## 8. ACPI PM Integration

ACPI provides hardware PM descriptions on x86 (and some ARM64) platforms.

### CPU C-states (Idle)

ACPI C-states map to cpuidle states:
- **C0** — active
- **C1/C1E** — halt, ~1us exit latency
- **C2** — stop grant, ~10-100us
- **C3** — sleep, L2 may flush, ~100-1000us
- **C6-C10** (Intel) — package C-states with core power-gating

ACPI `_CST` object describes C-states. `acpi_processor_power` driver reads them and
registers with cpuidle. On ARM64, PSCI maps to cpuidle via cpuidle-psci.

### CPU P-states (Frequency)

ACPI `_PSS` describes frequency/voltage pairs. `acpi-cpufreq` driver reads them. Modern
Intel uses `intel_pstate` (MSR-based HWP). AMD uses `amd-pstate` (CPPC).
`_PPC` limits the highest P-state. `_PSD` describes frequency-domain sharing.

### System S-states (Sleep)

- **S0** — running
- **S1** — standby (CPUs off, RAM refreshed) -> `PM_SUSPEND_STANDBY`
- **S3** — suspend to RAM -> `PM_SUSPEND_MEM`
- **S4** — suspend to disk -> hibernation
- **S5** — soft-off

### Device D-states (Device Power)

- **D0** — fully on
- **D1/D2** — intermediate sleep
- **D3hot** — low power but enumerable
- **D3cold** — completely off

PCI subsystem maps `PCI_D0-D3cold` to ACPI D-states. ACPI power resources (`_PR0`-`_PR3`)
describe which hardware rails to toggle for each D-state.

### ACPI Wakeup

`_PRW` object lists power resources and GPE (General Purpose Event) for wakeup signals.
When `device_may_wakeup()`, ACPI enables the corresponding GPE before suspend.

---

## 9. Locking Summary

| Subsystem | Lock | Type | Protects |
|---|---|---|---|
| CPUFreq | `cpufreq_driver_lock` | rwlock | `cpufreq_driver` pointer, per-CPU data |
| CPUFreq | `policy->rwsem` | rw_semaphore | policy fields during reads/writes |
| CPUFreq | `policy->transition_lock` | spinlock | `transition_ongoing` state |
| CPUIdle | `cpuidle_lock` | mutex | device list, enabled state |
| System PM | `system_sleep_lock` | mutex | sleep state transitions |
| Device PM | `device_pm_lock()` | mutex | `dpm_list` device ordering |
| RPM | `dev->power.lock` | spinlock | RPM state machine fields |
| genpd | `genpd->mlock` / `slock` | mutex/spinlock | domain state, device list |
| genpd | `gpd_list_lock` | mutex | global domain list |
| Thermal | `tz->lock` | mutex | thermal_instances list |
| Wakeup | `ws->lock` | spinlock | individual wakeup_source state |

---

## 10. Key Data Flows

### Task Wakeup -> Frequency Scale-Up (Schedutil)

1. Task becomes runnable; scheduler calls `cpufreq_update_util()`
2. `sugov_update_single_freq()` called via `update_util_data` hook
3. Check rate limit (`rate_limit_us`); skip if too soon
4. `get_next_freq()`: `next_freq = 1.25 * max_freq * util / max`
5. If `fast_switch_enabled`: call `cpufreq_driver_fast_switch()` directly
6. Otherwise: queue `irq_work` -> `kthread_work` -> `cpufreq_driver_target()`

### Device Runtime PM Suspend

1. Driver calls `pm_runtime_put(dev)` or `pm_runtime_put_autosuspend(dev)`
2. `usage_count` decremented; if 0, schedule idle check
3. `rpm_idle()` calls `rpm_check_suspend_allowed()` — checks usage, children, QoS
4. If allowed, calls subsystem `runtime_idle()`. If returns 0, schedule suspend.
5. `rpm_suspend()` sets status to `RPM_SUSPENDING`
6. Calls `runtime_suspend()` via callback resolution chain
7. On success, sets `RPM_SUSPENDED`, updates accounting
8. If in genpd: check if all domain devices suspended, then `genpd->power_off()`

### System Suspend -> Device Suspend Ordering

The device PM list (`dpm_list`) is ordered by device registration time. During suspend,
devices are traversed in reverse order (children before parents). During resume, devices
are traversed in forward order (parents before children). This ensures a parent device
is still active when its children are being suspended, and resumed before its children
need it.
