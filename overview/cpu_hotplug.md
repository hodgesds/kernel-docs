# Linux Kernel CPU Hotplug Subsystem

## Table of Contents
1. [Introduction](#1-introduction)
2. [Architecture Overview](#2-architecture-overview)
3. [Hotplug State Machine](#3-hotplug-state-machine)
4. [Core Data Structures](#4-core-data-structures)
5. [CPU Bring-Up Sequence](#5-cpu-bring-up-sequence)
6. [CPU Tear-Down Sequence](#6-cpu-tear-down-sequence)
7. [Hotplug Callback Registration API](#7-hotplug-callback-registration-api)
8. [Hotplug Locking](#8-hotplug-locking)
9. [Per-CPU Hotplug Thread Management](#9-per-cpu-hotplug-thread-management)
10. [Sysfs Interface](#10-sysfs-interface)
11. [Suspend/Resume: Freeze and Thaw](#11-suspendresume-freeze-and-thaw)
12. [SMT (Hyper-Threading) Control](#12-smt-hyper-threading-control)
13. [Interaction with the Scheduler](#13-interaction-with-the-scheduler)
14. [Interaction with Workqueues](#14-interaction-with-workqueues)
15. [Migration During Offline](#15-migration-during-offline)
16. [Parallel Bringup](#16-parallel-bringup)
17. [Key Source Files](#17-key-source-files)

---

## 1. Introduction

CPU hotplug is the kernel subsystem that manages the dynamic onlining and
offlining of CPUs at runtime. It enables:

- Adding or removing CPUs without rebooting (physical hotplug on supported
  hardware, or logical offlining for power management)
- Suspend/resume (all secondary CPUs are offlined during suspend and
  restored on resume)
- SMT (Simultaneous Multi-Threading / Hyper-Threading) control, allowing
  individual threads to be disabled for security or performance reasons
- Workload isolation by offlining unused CPUs
- Orderly shutdown and kexec by offlining all non-boot CPUs

The subsystem is built around a linear state machine with well-defined
callback points that subsystems register into. The core implementation
lives in `kernel/cpu.c`, with per-CPU thread management in
`kernel/smpboot.c`, state definitions in `include/linux/cpuhotplug.h`,
and locking primitives in `include/linux/cpuhplock.h`.

---

## 2. Architecture Overview

```
                           User Space
                    ________________________
                   |                        |
                   |  echo 0 > /sys/devices/system/cpu/cpu3/online
                   |  echo 1 > /sys/devices/system/cpu/cpu3/online
                   |________________________|
                              |
                              | device_online() / device_offline()
                              v
                    ________________________
                   |   drivers/base/cpu.c   |
                   |  cpu_subsys_online()   |
                   |  cpu_subsys_offline()  |
                   |________________________|
                              |
                              | cpu_device_up() / cpu_device_down()
                              v
                    ________________________
                   |     kernel/cpu.c       |
                   |  cpu_up() / cpu_down() |
                   |                        |
                   |  State Machine Engine  |
                   |  cpuhp_up_callbacks()  |
                   | cpuhp_down_callbacks() |
                   |________________________|
                          /        \
                         /          \
             ___________/            \___________
            |                                    |
    PREPARE callbacks                    AP callbacks
    (run on control CPU)              (run on target CPU)
            |                                    |
    CPUHP_OFFLINE..                     CPUHP_AP_OFFLINE..
    CPUHP_BRINGUP_CPU                   CPUHP_AP_ACTIVE
            |                                    |
    __cpu_up() / arch                   cpuhp/%u kthread
    kicks AP alive                      runs STARTING + ONLINE
                                        state callbacks
```

---

## 3. Hotplug State Machine

### 3.1 State Sections

The state machine is divided into three sections, each with different
execution contexts and constraints
(`include/linux/cpuhotplug.h:25-56`):

```
CPUHP_OFFLINE (0)
  |
  |  PREPARE section: callbacks run on a control CPU (the CPU issuing
  |  the hotplug request). These may sleep and may fail.
  |
  |    CPUHP_CREATE_THREADS          - create smpboot kthreads
  |    CPUHP_PERF_PREPARE            - perf event init
  |    CPUHP_WORKQUEUE_PREP          - workqueue per-CPU pool prep
  |    CPUHP_HRTIMERS_PREPARE        - hrtimer base init
  |    CPUHP_SMPCFD_PREPARE          - SMP call function dead
  |    CPUHP_RCUTREE_PREP            - RCU tree prepare
  |    CPUHP_TIMERS_PREPARE          - timer base init
  |    CPUHP_BP_PREPARE_DYN          - dynamic prepare slots (20 slots)
  |    CPUHP_BP_PREPARE_DYN_END
  |    CPUHP_BP_KICK_AP              - kick AP alive (parallel bringup)
  |    CPUHP_BRINGUP_CPU             - arch-specific CPU bringup
  |
  |  STARTING section: callbacks run on the hotplugged CPU with
  |  interrupts disabled. These MUST NOT fail.
  |
  |    CPUHP_AP_IDLE_DEAD            - final idle death state
  |    CPUHP_AP_OFFLINE              - AP offline marker
  |    CPUHP_AP_SCHED_STARTING       - scheduler starting/dying
  |    CPUHP_AP_RCUTREE_DYING        - RCU tree dying
  |    CPUHP_AP_IRQ_*_STARTING       - IRQ controller setup
  |    CPUHP_AP_PERF_*_STARTING      - perf hardware setup
  |    CPUHP_AP_*_TIMER_STARTING     - per-CPU timer setup
  |    CPUHP_AP_SMPCFD_DYING         - SMP call function dying
  |    CPUHP_AP_HRTIMERS_DYING       - hrtimer dying
  |    CPUHP_AP_TICK_DYING           - tick dying
  |    CPUHP_AP_ONLINE               - AP is now online
  |    CPUHP_TEARDOWN_CPU            - takedown_cpu (stop_machine)
  |
  |  ONLINE section: callbacks run on the hotplugged CPU from its
  |  per-CPU hotplug kthread. Interrupts and preemption enabled.
  |  These may fail on bringup (teardown must not fail).
  |
  |    CPUHP_AP_ONLINE_IDLE          - AP entered idle loop
  |    CPUHP_AP_SCHED_WAIT_EMPTY     - wait for runqueue drain
  |    CPUHP_AP_SMPBOOT_THREADS      - unpark/park smpboot kthreads
  |    CPUHP_AP_IRQ_AFFINITY_ONLINE  - IRQ affinity online
  |    CPUHP_AP_WORKQUEUE_ONLINE     - workqueue pool online
  |    CPUHP_AP_RCUTREE_ONLINE       - RCU tree online
  |    CPUHP_AP_ONLINE_DYN           - dynamic online slots (40 slots)
  |    CPUHP_AP_ONLINE_DYN_END
  |    CPUHP_AP_ACTIVE               - scheduler activates CPU
  |
  v
CPUHP_ONLINE
```

### 3.2 State Enum

The complete state enum is defined in `include/linux/cpuhotplug.h:57-250`:

```c
enum cpuhp_state {
    CPUHP_INVALID = -1,

    /* PREPARE section invoked on a control CPU */
    CPUHP_OFFLINE = 0,
    CPUHP_CREATE_THREADS,
    CPUHP_PERF_PREPARE,
    /* ... many subsystem-specific PREPARE states ... */
    CPUHP_TIMERS_PREPARE,
    CPUHP_BP_PREPARE_DYN,
    CPUHP_BP_PREPARE_DYN_END        = CPUHP_BP_PREPARE_DYN + 20,
    CPUHP_BP_KICK_AP,
    CPUHP_BRINGUP_CPU,

    /* STARTING section on the hotplugged CPU, IRQs disabled */
    CPUHP_AP_IDLE_DEAD,
    CPUHP_AP_OFFLINE,
    CPUHP_AP_SCHED_STARTING,
    CPUHP_AP_RCUTREE_DYING,
    /* ... IRQ controller, perf, timer states ... */
    CPUHP_AP_SMPCFD_DYING,
    CPUHP_AP_HRTIMERS_DYING,
    CPUHP_AP_TICK_DYING,
    CPUHP_AP_ONLINE,
    CPUHP_TEARDOWN_CPU,

    /* ONLINE section on the hotplugged CPU, preemptible */
    CPUHP_AP_ONLINE_IDLE,
    CPUHP_AP_SCHED_WAIT_EMPTY,
    CPUHP_AP_SMPBOOT_THREADS,
    CPUHP_AP_WORKQUEUE_ONLINE,
    CPUHP_AP_RCUTREE_ONLINE,
    CPUHP_AP_ONLINE_DYN,
    CPUHP_AP_ONLINE_DYN_END         = CPUHP_AP_ONLINE_DYN + 40,
    CPUHP_AP_ACTIVE,
    CPUHP_ONLINE,
};
```

### 3.3 Bringup vs. Teardown Direction

During a CPU online operation, the state machine invokes startup callbacks
sequentially from `CPUHP_OFFLINE + 1` to `CPUHP_ONLINE`. During a CPU
offline operation, teardown callbacks are invoked in the reverse order
from `CPUHP_ONLINE - 1` down to `CPUHP_OFFLINE`.

The comment at `include/linux/cpuhotplug.h:7-23` shows the flow:

```
CPU-up                             CPU-down

BP          AP                     BP          AP

OFFLINE                            OFFLINE
  |                                  ^
  v                                  |
BRINGUP_CPU -> AP_OFFLINE          BRINGUP_CPU <- AP_IDLE_DEAD (play_dead)
                 |                               AP_OFFLINE
                 v (IRQ-off)       ,---------------^
               AP_ONLINE           | (stop_machine)
                 |               TEARDOWN_CPU <- AP_ONLINE_IDLE
                 |                                 ^
                 v                                 |
               AP_ACTIVE                         AP_ACTIVE
```

---

## 4. Core Data Structures

### 4.1 struct cpuhp_cpu_state

Per-CPU hotplug state tracking, one per logical CPU
(`kernel/cpu.c:47-84`):

```c
struct cpuhp_cpu_state {
    enum cpuhp_state    state;          /* Current cpu state */
    enum cpuhp_state    target;         /* Target state for transition */
    enum cpuhp_state    fail;           /* Failed state (for injection) */
#ifdef CONFIG_SMP
    struct task_struct  *thread;        /* Per-CPU hotplug kthread (cpuhp/N) */
    bool                should_run;     /* Thread should execute */
    bool                rollback;       /* Performing rollback */
    bool                single;         /* Single callback invocation */
    bool                bringup;        /* Bringup or teardown selector */
    struct hlist_node   *node;          /* Multi-instance node */
    struct hlist_node   *last;          /* Multi-instance rollback cursor */
    enum cpuhp_state    cb_state;       /* Callback state for single mode */
    int                 result;         /* Operation result */
    atomic_t            ap_sync_state;  /* AP synchronization state */
    struct completion   done_up;        /* Signal bringup completion */
    struct completion   done_down;      /* Signal teardown completion */
#endif
};
```

Allocated as a per-CPU variable (`kernel/cpu.c:86-88`):

```c
static DEFINE_PER_CPU(struct cpuhp_cpu_state, cpuhp_state) = {
    .fail = CPUHP_INVALID,
};
```

### 4.2 struct cpuhp_step

Represents one state in the hotplug state machine, holding the startup
and teardown callback functions (`kernel/cpu.c:117-142`):

```c
struct cpuhp_step {
    const char          *name;
    union {
        int     (*single)(unsigned int cpu);
        int     (*multi)(unsigned int cpu,
                         struct hlist_node *node);
    } startup;
    union {
        int     (*single)(unsigned int cpu);
        int     (*multi)(unsigned int cpu,
                         struct hlist_node *node);
    } teardown;
    struct hlist_head   list;           /* Multi-instance list */
    bool                cant_stop;      /* Cannot be stopped at this step */
    bool                multi_instance; /* Supports multiple instances */
};
```

The global state table is the static array (`kernel/cpu.c:2062-2268`):

```c
static struct cpuhp_step cpuhp_hp_states[] = {
    [CPUHP_OFFLINE] = {
        .name           = "offline",
        .startup.single = NULL,
        .teardown.single = NULL,
    },
    [CPUHP_CREATE_THREADS] = {
        .name           = "threads:prepare",
        .startup.single = smpboot_create_threads,
        .teardown.single = NULL,
        .cant_stop      = true,
    },
    /* ... many more entries ... */
    [CPUHP_AP_SCHED_STARTING] = {
        .name           = "sched:starting",
        .startup.single = sched_cpu_starting,
        .teardown.single = sched_cpu_dying,
    },
    [CPUHP_TEARDOWN_CPU] = {
        .name           = "cpu:teardown",
        .startup.single = NULL,
        .teardown.single = takedown_cpu,
        .cant_stop      = true,
    },
    [CPUHP_AP_SMPBOOT_THREADS] = {
        .name           = "smpboot/threads:online",
        .startup.single = smpboot_unpark_threads,
        .teardown.single = smpboot_park_threads,
    },
    [CPUHP_AP_WORKQUEUE_ONLINE] = {
        .name           = "workqueue:online",
        .startup.single = workqueue_online_cpu,
        .teardown.single = workqueue_offline_cpu,
    },
    [CPUHP_AP_ACTIVE] = {
        .name           = "sched:active",
        .startup.single = sched_cpu_activate,
        .teardown.single = sched_cpu_deactivate,
    },
    [CPUHP_ONLINE] = {
        .name           = "online",
    },
};
```

### 4.3 struct cpu

The device-model representation of a CPU (`include/linux/cpu.h:27-31`):

```c
struct cpu {
    int node_id;        /* The NUMA node which contains the CPU */
    int hotpluggable;   /* Creates sysfs control file if hotpluggable */
    struct device dev;
};
```

### 4.4 struct smp_hotplug_thread

Descriptor for per-CPU kthreads that are hotplug-aware
(`include/linux/smpboot.h:31-43`):

```c
struct smp_hotplug_thread {
    struct task_struct      * __percpu *store;
    struct list_head        list;
    int     (*thread_should_run)(unsigned int cpu);
    void    (*thread_fn)(unsigned int cpu);
    void    (*create)(unsigned int cpu);
    void    (*setup)(unsigned int cpu);
    void    (*cleanup)(unsigned int cpu, bool online);
    void    (*park)(unsigned int cpu);
    void    (*unpark)(unsigned int cpu);
    bool    selfparking;
    const char              *thread_comm;
};
```

### 4.5 AP Synchronization States

The boot processor (BP) and application processor (AP) synchronize
through atomic state transitions during bringup and teardown
(`kernel/cpu.c:282-289`):

```c
enum cpuhp_sync_state {
    SYNC_STATE_DEAD,
    SYNC_STATE_KICKED,
    SYNC_STATE_SHOULD_DIE,
    SYNC_STATE_ALIVE,
    SYNC_STATE_SHOULD_ONLINE,
    SYNC_STATE_ONLINE,
};
```

---

## 5. CPU Bring-Up Sequence

### 5.1 Entry Points

There are several entry points for bringing a CPU online:

| Entry Point | Context | Source |
|-------------|---------|--------|
| `add_cpu(cpu)` | User-initiated, takes device_hotplug_lock | `kernel/cpu.c:1743-1753` |
| `cpu_device_up(dev)` | Device core (sysfs) | `kernel/cpu.c:1738-1741` |
| `cpu_up(cpu, target)` | Internal, takes cpu_add_remove_lock | `kernel/cpu.c:1697-1726` |
| `_cpu_up(cpu, tasks_frozen, target)` | Inner implementation | `kernel/cpu.c:1632-1695` |
| `bringup_nonboot_cpus(max)` | Boot-time bulk bringup | `kernel/cpu.c:1886-1897` |

### 5.2 Detailed Bringup Flow

```
add_cpu(cpu) or echo 1 > .../cpuN/online
  |
  v
cpu_up(cpu, CPUHP_ONLINE)                    [kernel/cpu.c:1697]
  |-- cpu_maps_update_begin()                 acquire cpu_add_remove_lock
  |-- check cpu_possible, cpu_bootable
  |
  v
_cpu_up(cpu, 0, CPUHP_ONLINE)               [kernel/cpu.c:1632]
  |-- cpus_write_lock()                       acquire cpu_hotplug_lock (write)
  |-- check cpu_present
  |-- idle = idle_thread_get(cpu)             get pre-forked idle thread
  |-- scs_task_reset(idle)                    reset shadow call stack
  |-- cpuhp_set_state(cpu, st, target)        set target = CPUHP_ONLINE
  |
  |-- cpuhp_up_callbacks(cpu, st, CPUHP_BRINGUP_CPU)
  |     |                                     Run PREPARE callbacks on control CPU:
  |     |-- CPUHP_CREATE_THREADS              smpboot_create_threads()
  |     |-- CPUHP_PERF_PREPARE                perf_event_init_cpu()
  |     |-- CPUHP_WORKQUEUE_PREP              workqueue_prepare_cpu()
  |     |-- CPUHP_HRTIMERS_PREPARE            hrtimers_prepare_cpu()
  |     |-- CPUHP_RCUTREE_PREP                rcutree_prepare_cpu()
  |     |-- CPUHP_TIMERS_PREPARE              timers_prepare_cpu()
  |     |-- CPUHP_BRINGUP_CPU                 bringup_cpu() -> __cpu_up()
  |     |     |                               (arch kicks AP alive via INIT-SIPI-SIPI)
  |     |     |
  |     |     |    === AP starts executing ===
  |     |     |    AP: start_secondary()
  |     |     |    AP: cpuhp_ap_sync_alive()  -> SYNC_STATE_ALIVE
  |     |     |    AP: notify_cpu_starting()
  |     |     |      |-- rcutree_report_cpu_starting()
  |     |     |      |-- CPUHP_AP_SCHED_STARTING  sched_cpu_starting()
  |     |     |      |-- CPUHP_AP_RCUTREE_DYING   (startup: no-op)
  |     |     |      |-- CPUHP_AP_IRQ_*           IRQ controller init
  |     |     |      |-- CPUHP_AP_*_TIMER         timer init
  |     |     |      |-- ... up to CPUHP_AP_ONLINE
  |     |     |    AP: set_cpu_online(cpu, true)
  |     |     |    AP: local_irq_enable()
  |     |     |    AP: cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)
  |     |     |    AP: cpuhp_online_idle()    -> SYNC_STATE_ONLINE
  |     |     |      |-- stop_machine_unpark()
  |     |     |      |-- complete(done_up)     signal BP
  |     |     |
  |     |     |-- bringup_wait_for_ap_online() wait for done_up
  |     |     |-- kthread_unpark(cpuhp thread)
  |
  |-- cpuhp_kick_ap_work(cpu)                  kick AP cpuhp thread
  |     |                                      Run ONLINE callbacks on target CPU:
  |     |-- CPUHP_AP_SMPBOOT_THREADS           smpboot_unpark_threads()
  |     |-- CPUHP_AP_WORKQUEUE_ONLINE          workqueue_online_cpu()
  |     |-- CPUHP_AP_RCUTREE_ONLINE            rcutree_online_cpu()
  |     |-- CPUHP_AP_ONLINE_DYN..DYN_END       dynamic callbacks
  |     |-- CPUHP_AP_ACTIVE                    sched_cpu_activate()
  |     |     |-- set_cpu_active(cpu, true)
  |     |     |-- cpuset_cpu_active()
  |     |     |-- rebuild sched domains
  |
  |-- cpus_write_unlock()
  |-- arch_smt_update()
  v
  CPU is now fully online
```

### 5.3 Key Bringup Functions

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `cpu_up()` | `kernel/cpu.c:1697` | Validates and initiates CPU bringup |
| `_cpu_up()` | `kernel/cpu.c:1632` | Core bringup with locking |
| `cpuhp_up_callbacks()` | `kernel/cpu.c:1014` | Walk PREPARE states with rollback |
| `bringup_cpu()` | `kernel/cpu.c:857` | Arch-specific CPU start |
| `bringup_wait_for_ap_online()` | `kernel/cpu.c:792` | Wait for AP to reach ONLINE_IDLE |
| `cpuhp_kick_ap_work()` | `kernel/cpu.c:1175` | Trigger AP hotplug thread |
| `notify_cpu_starting()` | `kernel/cpu.c:1592` | Invoke STARTING callbacks on AP |
| `cpuhp_online_idle()` | `kernel/cpu.c:1611` | AP signals bringup completion |

---

## 6. CPU Tear-Down Sequence

### 6.1 Detailed Teardown Flow

```
remove_cpu(cpu) or echo 0 > .../cpuN/online
  |
  v
cpu_down(cpu, CPUHP_OFFLINE)                 [kernel/cpu.c:1502]
  |-- cpu_maps_update_begin()
  |-- cpu_down_maps_locked()                  [kernel/cpu.c:1476]
  |     |-- check cpu_hotplug_disabled
  |     |-- work_on_cpu(other_cpu, ...)        run on different CPU
  |
  v
_cpu_down(cpu, 0, CPUHP_OFFLINE)             [kernel/cpu.c:1399]
  |-- check num_online_cpus() > 1             cannot offline last CPU
  |-- cpus_write_lock()
  |-- cpuhp_set_state(cpu, st, CPUHP_OFFLINE)
  |
  |-- cpuhp_kick_ap_work(cpu)                  target = CPUHP_TEARDOWN_CPU
  |     |                                      Run ONLINE teardown on target CPU:
  |     |-- CPUHP_AP_ACTIVE teardown           sched_cpu_deactivate()
  |     |     |-- set_cpu_active(cpu, false)
  |     |     |-- balance_push_set(cpu, true)   push tasks away
  |     |     |-- synchronize_rcu()
  |     |     |-- rebuild sched domains
  |     |-- CPUHP_AP_ONLINE_DYN..DYN_END       dynamic teardown
  |     |-- CPUHP_AP_RCUTREE_ONLINE            rcutree_offline_cpu()
  |     |-- CPUHP_AP_WORKQUEUE_ONLINE          workqueue_offline_cpu()
  |     |-- CPUHP_AP_SMPBOOT_THREADS           smpboot_park_threads()
  |     |-- CPUHP_AP_SCHED_WAIT_EMPTY          sched_cpu_wait_empty()
  |     |     |-- balance_hotplug_wait()        drain tasks
  |
  |-- cpuhp_down_callbacks(cpu, st, CPUHP_OFFLINE)
  |     |                                      Run PREPARE teardown on control CPU:
  |     |-- CPUHP_TEARDOWN_CPU                 takedown_cpu()
  |     |     |-- kthread_park(cpuhp thread)
  |     |     |-- irq_lock_sparse()
  |     |     |-- stop_machine(take_cpu_down)   on the dying CPU:
  |     |     |     |-- __cpu_disable()          move IRQs away
  |     |     |     |-- CPUHP_AP_SCHED_STARTING  sched_cpu_dying()
  |     |     |     |-- CPUHP_AP_RCUTREE_DYING   rcutree_dying_cpu()
  |     |     |     |-- CPUHP_AP_SMPCFD_DYING    smpcfd_dying_cpu()
  |     |     |     |-- CPUHP_AP_HRTIMERS_DYING  hrtimers_cpu_dying()
  |     |     |     |-- CPUHP_AP_TICK_DYING      tick_cpu_dying()
  |     |     |     |-- stop_machine_park(cpu)
  |     |     |
  |     |     |     AP: enters idle loop -> play_dead (arch_cpu_idle_dead)
  |     |     |     AP: cpuhp_report_idle_dead()
  |     |     |       |-- rcutree_report_cpu_dead()
  |     |     |       |-- st->state = CPUHP_AP_IDLE_DEAD
  |     |     |       |-- smp_call_function_single() to signal BP
  |     |     |
  |     |     |-- wait_for_ap_thread(st, false)  wait for AP death
  |     |     |-- irq_unlock_sparse()
  |     |     |-- __cpu_die(cpu)                arch-specific cleanup
  |     |     |-- cpuhp_bp_sync_dead(cpu)       wait for SYNC_STATE_DEAD
  |     |     |-- rcutree_migrate_callbacks()   migrate RCU callbacks
  |     |
  |     |-- CPUHP_TIMERS_PREPARE               timers_dead_cpu()
  |     |-- CPUHP_RCUTREE_PREP                 rcutree_dead_cpu()
  |     |-- CPUHP_SMPCFD_PREPARE               smpcfd_dead_cpu()
  |     |-- CPUHP_PERF_PREPARE                 perf_event_exit_cpu()
  |
  |-- cpus_write_unlock()
  |-- lockup_detector_cleanup()
  |-- arch_smt_update()
  v
  CPU is now offline
```

### 6.2 Key Teardown Functions

| Function | File:Line | Purpose |
|----------|-----------|---------|
| `cpu_down()` | `kernel/cpu.c:1502` | Entry point for CPU offlining |
| `_cpu_down()` | `kernel/cpu.c:1399` | Core teardown with locking |
| `takedown_cpu()` | `kernel/cpu.c:1295` | Orchestrate CPU removal |
| `take_cpu_down()` | `kernel/cpu.c:1268` | Runs on dying CPU via stop_machine |
| `cpuhp_down_callbacks()` | `kernel/cpu.c:1376` | Walk states downward with rollback |
| `cpuhp_report_idle_dead()` | `kernel/cpu.c:1360` | AP reports death from idle |
| `clear_tasks_mm_cpumask()` | `kernel/cpu.c:1238` | Clear mm cpumasks for dead CPU |
| `finish_cpu()` | `kernel/cpu.c:902` | Clean up idle thread mm state |

### 6.3 The take_cpu_down / stop_machine Interaction

The critical transition happens at `CPUHP_TEARDOWN_CPU`. The
`takedown_cpu()` function (`kernel/cpu.c:1295-1351`) uses `stop_machine`
to execute `take_cpu_down()` on the dying CPU with all other CPUs
spinning with interrupts disabled:

```c
/* kernel/cpu.c:1268-1293 */
static int take_cpu_down(void *_param)
{
    struct cpuhp_cpu_state *st = this_cpu_ptr(&cpuhp_state);
    enum cpuhp_state target = max((int)st->target, CPUHP_AP_OFFLINE);
    int err, cpu = smp_processor_id();

    /* Ensure this CPU doesn't handle any more interrupts. */
    err = __cpu_disable();
    if (err < 0)
        return err;

    /* Invoke the DYING callbacks. DYING must not fail! */
    cpuhp_invoke_callback_range_nofail(false, cpu, st, target);

    /* Park the stopper thread */
    stop_machine_park(cpu);
    return 0;
}
```

After `take_cpu_down()` completes, the dying CPU enters `play_dead`
(the arch-specific idle-dead loop), eventually calling
`cpuhp_report_idle_dead()` to signal completion back to the control CPU.

---

## 7. Hotplug Callback Registration API

### 7.1 Primary API Functions

The API is declared in `include/linux/cpuhotplug.h:252-472`:

```c
/* Register callbacks and invoke startup on already-online CPUs */
int cpuhp_setup_state(enum cpuhp_state state, const char *name,
                      int (*startup)(unsigned int cpu),
                      int (*teardown)(unsigned int cpu));

/* Register without invoking on currently online CPUs */
int cpuhp_setup_state_nocalls(enum cpuhp_state state, const char *name,
                              int (*startup)(unsigned int cpu),
                              int (*teardown)(unsigned int cpu));

/* Variants for use when cpus_read_lock is already held */
int cpuhp_setup_state_cpuslocked(enum cpuhp_state state, const char *name,
                                 int (*startup)(unsigned int cpu),
                                 int (*teardown)(unsigned int cpu));

int cpuhp_setup_state_nocalls_cpuslocked(enum cpuhp_state state,
                                         const char *name,
                                         int (*startup)(unsigned int cpu),
                                         int (*teardown)(unsigned int cpu));

/* Multi-instance: per-device or per-object callbacks */
int cpuhp_setup_state_multi(enum cpuhp_state state, const char *name,
                            int (*startup)(unsigned int cpu,
                                           struct hlist_node *node),
                            int (*teardown)(unsigned int cpu,
                                            struct hlist_node *node));

/* Add/remove instances for multi-instance states */
int cpuhp_state_add_instance(enum cpuhp_state state,
                             struct hlist_node *node);
int cpuhp_state_add_instance_nocalls(enum cpuhp_state state,
                                     struct hlist_node *node);

/* Remove callbacks */
void cpuhp_remove_state(enum cpuhp_state state);
void cpuhp_remove_state_nocalls(enum cpuhp_state state);
void cpuhp_remove_multi_state(enum cpuhp_state state);
int cpuhp_state_remove_instance(enum cpuhp_state state,
                                struct hlist_node *node);
```

### 7.2 Dynamic State Allocation

When a subsystem does not need strict ordering relative to other states,
it should use `CPUHP_AP_ONLINE_DYN` or `CPUHP_BP_PREPARE_DYN` as the
state argument. The framework allocates a free slot from the dynamic
range and returns the slot number (`kernel/cpu.c:2283-2307`):

```c
static int cpuhp_reserve_state(enum cpuhp_state state)
{
    enum cpuhp_state i, end;
    struct cpuhp_step *step;

    switch (state) {
    case CPUHP_AP_ONLINE_DYN:
        step = cpuhp_hp_states + CPUHP_AP_ONLINE_DYN;
        end = CPUHP_AP_ONLINE_DYN_END;
        break;
    case CPUHP_BP_PREPARE_DYN:
        step = cpuhp_hp_states + CPUHP_BP_PREPARE_DYN;
        end = CPUHP_BP_PREPARE_DYN_END;
        break;
    default:
        return -EINVAL;
    }

    for (i = state; i <= end; i++, step++) {
        if (!step->name)
            return i;
    }
    WARN(1, "No more dynamic states available for CPU hotplug\n");
    return -ENOSPC;
}
```

Dynamic ranges:
- `CPUHP_BP_PREPARE_DYN`: 20 slots for PREPARE-section callbacks
- `CPUHP_AP_ONLINE_DYN`: 40 slots for ONLINE-section callbacks

### 7.3 Usage Example

```c
/* Simple single-instance registration */
static int my_cpu_online(unsigned int cpu)
{
    /* Initialize per-CPU resources */
    return 0;
}

static int my_cpu_offline(unsigned int cpu)
{
    /* Clean up per-CPU resources */
    return 0;
}

/* Registration with dynamic slot allocation */
int ret;
ret = cpuhp_setup_state(CPUHP_AP_ONLINE_DYN,
                        "mysubsystem:online",
                        my_cpu_online,
                        my_cpu_offline);
if (ret < 0)
    return ret;

/* ret contains the allocated dynamic state number */
int my_hp_state = ret;

/* Later, to unregister: */
cpuhp_remove_state(my_hp_state);
```

### 7.4 Multi-Instance Example

```c
/* For per-device hotplug callbacks */
struct my_device {
    struct hlist_node cpuhp_node;
    /* ... */
};

static int my_dev_cpu_online(unsigned int cpu, struct hlist_node *node)
{
    struct my_device *dev = container_of(node, struct my_device, cpuhp_node);
    /* Setup device on this CPU */
    return 0;
}

/* Register the multi-instance state (once) */
cpuhp_setup_state_multi(CPUHP_AP_ONLINE_DYN,
                        "mydriver:online",
                        my_dev_cpu_online,
                        my_dev_cpu_offline);

/* Add per-device instance */
cpuhp_state_add_instance(my_hp_state, &dev->cpuhp_node);

/* Remove per-device instance */
cpuhp_state_remove_instance(my_hp_state, &dev->cpuhp_node);
```

### 7.5 Callback Invocation

The `cpuhp_invoke_callback()` function (`kernel/cpu.c:169-249`) dispatches
callbacks. For single-instance states it calls the function directly; for
multi-instance states it iterates the `hlist` of registered nodes:

```c
static int cpuhp_invoke_callback(unsigned int cpu, enum cpuhp_state state,
                                 bool bringup, struct hlist_node *node,
                                 struct hlist_node **lastp)
{
    struct cpuhp_cpu_state *st = per_cpu_ptr(&cpuhp_state, cpu);
    struct cpuhp_step *step = cpuhp_get_step(state);

    if (!step->multi_instance) {
        cb = bringup ? step->startup.single : step->teardown.single;
        ret = cb(cpu);
        return ret;
    }

    /* Multi-instance: iterate over all registered nodes */
    cbm = bringup ? step->startup.multi : step->teardown.multi;
    hlist_for_each(node, &step->list) {
        ret = cbm(cpu, node);
        if (ret) {
            /* Rollback previously successful instances */
            ...
        }
    }
    return 0;
}
```

---

## 8. Hotplug Locking

### 8.1 cpu_hotplug_lock (percpu-rwsem)

The primary lock protecting CPU state transitions is a per-CPU reader-
writer semaphore (`kernel/cpu.c:484`):

```c
DEFINE_STATIC_PERCPU_RWSEM(cpu_hotplug_lock);
```

API (`include/linux/cpuhplock.h:17-27`, `kernel/cpu.c:488-514`):

```c
void cpus_read_lock(void);      /* Prevent CPU state changes */
void cpus_read_unlock(void);
int  cpus_read_trylock(void);   /* Non-blocking attempt */
void cpus_write_lock(void);     /* Exclusive: held during state changes */
void cpus_write_unlock(void);
void lockdep_assert_cpus_held(void);
```

The lock guard macro (`include/linux/cpuhplock.h:47`):

```c
DEFINE_LOCK_GUARD_0(cpus_read_lock, cpus_read_lock(), cpus_read_unlock())
```

**Readers** (`cpus_read_lock`): Use this to ensure CPUs remain in their
current state during an operation. Multiple readers can hold this
concurrently. This is a very fast operation on the read path (per-CPU
counter increment).

**Writers** (`cpus_write_lock`): Held exclusively during actual CPU state
transitions. Called by `_cpu_up()` and `_cpu_down()` before modifying
CPU state.

### 8.2 cpu_add_remove_lock

A mutex serializing updates to `cpu_online_mask` and `cpu_present_mask`
(`kernel/cpu.c:458`):

```c
static DEFINE_MUTEX(cpu_add_remove_lock);
```

Acquired via `cpu_maps_update_begin()` / `cpu_maps_update_done()`.
The `cpu_hotplug_disabled` counter is protected by this lock
(`kernel/cpu.c:480`).

### 8.3 cpuhp_state_mutex

A mutex serializing the installation and removal of hotplug callbacks
(`kernel/cpu.c:144`):

```c
static DEFINE_MUTEX(cpuhp_state_mutex);
```

Held during `__cpuhp_setup_state()` and `__cpuhp_remove_state()` to
protect the `cpuhp_hp_states[]` array.

### 8.4 Locking Hierarchy

```
cpu_add_remove_lock          (outermost - serializes hotplug requests)
  |
  +-- cpu_hotplug_lock       (percpu-rwsem - protects state transitions)
        |
        +-- cpuhp_state_mutex  (protects callback registration)
```

### 8.5 Disable/Enable Hotplug

Subsystems can temporarily prevent CPU hotplug operations
(`kernel/cpu.c:562-583`):

```c
void cpu_hotplug_disable(void)
{
    cpu_maps_update_begin();
    cpu_hotplug_disabled++;
    cpu_maps_update_done();
}

void cpu_hotplug_enable(void)
{
    cpu_maps_update_begin();
    __cpu_hotplug_enable();
    cpu_maps_update_done();
}
```

When `cpu_hotplug_disabled > 0`, `cpu_up()` and `cpu_down()` return
`-EBUSY`.

---

## 9. Per-CPU Hotplug Thread Management

### 9.1 The cpuhp/N Thread

Each CPU has a dedicated hotplug kthread named `cpuhp/N` that executes
ONLINE-section callbacks on the target CPU. It is registered as an
smpboot thread (`kernel/cpu.c:1194-1200`):

```c
static struct smp_hotplug_thread cpuhp_threads = {
    .store          = &cpuhp_state.thread,
    .thread_should_run = cpuhp_should_run,
    .thread_fn      = cpuhp_thread_fun,
    .thread_comm    = "cpuhp/%u",
    .selfparking    = true,
};
```

The `cpuhp_thread_fun()` function (`kernel/cpu.c:1058-1121`) executes
one state callback per invocation. It operates in three modes:

```
1. single: Runs a single callback at st->cb_state (for instance
           add/remove at runtime)
2. up:     Increments st->state and runs the callback, repeating
           while st->state < st->target
3. down:   Decrements st->state and runs the callback, repeating
           while st->state > st->target
```

For STARTING-section states (atomic states), the thread disables local
IRQs before invoking the callback (`kernel/cpu.c:1092-1095`):

```c
if (cpuhp_is_atomic_state(state)) {
    local_irq_disable();
    st->result = cpuhp_invoke_callback(cpu, state, bringup, st->node, &st->last);
    local_irq_enable();
}
```

### 9.2 The smpboot Thread Framework

The smpboot framework (`kernel/smpboot.c`) manages per-CPU kthreads
that need to be automatically parked when their CPU goes offline and
unparked when it comes back online. The hotplug state machine uses
`CPUHP_AP_SMPBOOT_THREADS` to trigger parking/unparking.

Thread lifecycle (`kernel/smpboot.c:84-94`):

```c
struct smpboot_thread_data {
    unsigned int            cpu;
    unsigned int            status;
    struct smp_hotplug_thread   *ht;
};

enum {
    HP_THREAD_NONE = 0,
    HP_THREAD_ACTIVE,
    HP_THREAD_PARKED,
};
```

The main thread loop (`kernel/smpboot.c:106-167`) handles state
transitions:

```
+-- HP_THREAD_NONE   -> call ht->setup(), transition to ACTIVE
+-- HP_THREAD_PARKED -> call ht->unpark(), transition to ACTIVE
+-- HP_THREAD_ACTIVE -> call ht->thread_fn() or sleep
+-- kthread_should_park() -> call ht->park(), kthread_parkme()
+-- kthread_should_stop() -> call ht->cleanup(), return
```

### 9.3 Registration and Lifecycle

Register a per-CPU thread (`kernel/smpboot.c:288-309`):

```c
int smpboot_register_percpu_thread(struct smp_hotplug_thread *plug_thread)
{
    cpus_read_lock();
    mutex_lock(&smpboot_threads_lock);
    for_each_online_cpu(cpu) {
        __smpboot_create_thread(plug_thread, cpu);
        smpboot_unpark_thread(plug_thread, cpu);
    }
    list_add(&plug_thread->list, &hotplug_threads);
    mutex_unlock(&smpboot_threads_lock);
    cpus_read_unlock();
    return ret;
}
```

Key per-CPU kthreads managed by this framework:
- `cpuhp/%u` -- hotplug state machine
- `migration/%u` -- stop_machine stopper thread
- `ksoftirqd/%u` -- softirq processing
- `kcompactd/%u` -- memory compaction
- `watchdog/%u` -- NMI watchdog

### 9.4 Idle Thread Management

Each CPU requires a pre-forked idle thread. These are created during
early boot by `idle_threads_init()` (`kernel/smpboot.c:66-76`):

```c
void __init idle_threads_init(void)
{
    unsigned int cpu, boot_cpu;
    boot_cpu = smp_processor_id();
    for_each_possible_cpu(cpu) {
        if (cpu != boot_cpu)
            idle_init(cpu);    /* calls fork_idle(cpu) */
    }
}
```

The idle threads are stored per-CPU and reused across hotplug cycles
(`kernel/smpboot.c:28`):

```c
static DEFINE_PER_CPU(struct task_struct *, idle_threads);
```

---

## 10. Sysfs Interface

### 10.1 Directory Layout

```
/sys/devices/system/cpu/
  |-- online                    cpumask of online CPUs (e.g., "0-7")
  |-- offline                   cpumask of offline CPUs
  |-- possible                  cpumask of possible CPUs
  |-- present                   cpumask of present CPUs
  |-- kernel_max                NR_CPUS - 1
  |
  |-- cpu0/
  |   |-- online                1 or 0 (read/write for hotpluggable CPUs)
  |   |-- topology/             CPU topology info
  |   |   |-- core_id
  |   |   |-- physical_package_id
  |   |   |-- thread_siblings
  |   |   |-- core_siblings
  |   |-- cache/
  |   |-- cpufreq -> ...
  |
  |-- cpu1/
  |   |-- online                writable for hotpluggable CPUs
  |   |-- ...
  |
  |-- smt/
      |-- control               on/off/forceoff/notsupported
      |-- active                1 or 0
```

### 10.2 Online/Offline via Sysfs

The `online` attribute is managed by the device core's `dev_online_store()`
handler, which calls through to the `cpu_subsys` bus type
(`drivers/base/cpu.c:377-388`):

```c
const struct bus_type cpu_subsys = {
    .name = "cpu",
    .dev_name = "cpu",
    .match = cpu_subsys_match,
#ifdef CONFIG_HOTPLUG_CPU
    .online = cpu_subsys_online,
    .offline = cpu_subsys_offline,
#endif
};
```

The `cpu_subsys_online()` function (`drivers/base/cpu.c:48-87`) calls
`cpu_device_up()`, with retry logic for transient `-EBUSY` errors:

```c
static int cpu_subsys_online(struct device *dev)
{
    struct cpu *cpu = container_of(dev, struct cpu, dev);
    int ret, retries = 0;

retry:
    ret = cpu_device_up(dev);
    if (ret == -EBUSY) {
        retries++;
        if (retries > 5)
            return ret;
        msleep(10 * (1 << retries));
        goto retry;
    }
    return ret;
}
```

### 10.3 Global cpumask Attributes

The global attributes (`drivers/base/cpu.c:228-232`) expose the
standard cpumasks:

```c
static struct cpu_attr cpu_attrs[] = {
    _CPU_ATTR(online, &__cpu_online_mask),
    _CPU_ATTR(possible, &__cpu_possible_mask),
    _CPU_ATTR(present, &__cpu_present_mask),
};
```

### 10.4 Hotpluggability

A CPU is hotpluggable if `cpu->hotpluggable` is set. The device core
checks `dev->offline_disabled` to decide whether to expose the `online`
attribute as writable. During registration (`drivers/base/cpu.c:399-418`):

```c
int register_cpu(struct cpu *cpu, int num)
{
    cpu->dev.offline_disabled = !cpu->hotpluggable;
    cpu->dev.offline = !cpu_online(num);
    /* ... */
    device_register(&cpu->dev);
}
```

---

## 11. Suspend/Resume: Freeze and Thaw

### 11.1 Overview

During system suspend (S3/S4), all secondary CPUs are offlined and then
re-onlined on resume. This uses the hotplug state machine with
`tasks_frozen = 1` to indicate that tasks are already frozen.

### 11.2 freeze_secondary_cpus

Takes down all non-primary CPUs (`kernel/cpu.c:1902-1958`):

```c
int freeze_secondary_cpus(int primary)
{
    cpu_maps_update_begin();

    /* Select primary CPU (boot CPU or first online) */
    if (primary == -1)
        primary = cpumask_first(cpu_online_mask);

    cpumask_clear(frozen_cpus);
    pr_info("Disabling non-boot CPUs ...\n");

    /* Take CPUs down in reverse order (highest first) */
    for (cpu = nr_cpu_ids - 1; cpu >= 0; cpu--) {
        if (!cpu_online(cpu) || cpu == primary)
            continue;

        if (pm_wakeup_pending()) {
            error = -EBUSY;
            break;
        }

        error = _cpu_down(cpu, 1, CPUHP_OFFLINE);  /* tasks_frozen=1 */
        if (!error)
            cpumask_set_cpu(cpu, frozen_cpus);
        else
            break;
    }

    /* Prevent re-onlining during suspend */
    cpu_hotplug_disabled++;
    cpu_maps_update_done();
    return error;
}
```

### 11.3 thaw_secondary_cpus

Re-onlines previously frozen CPUs on resume (`kernel/cpu.c:1968-1998`):

```c
void thaw_secondary_cpus(void)
{
    cpu_maps_update_begin();
    __cpu_hotplug_enable();

    pr_info("Enabling non-boot CPUs ...\n");
    arch_thaw_secondary_cpus_begin();

    for_each_cpu(cpu, frozen_cpus) {
        error = _cpu_up(cpu, 1, CPUHP_ONLINE);  /* tasks_frozen=1 */
        if (!error)
            pr_info("CPU%d is up\n", cpu);
        else
            pr_warn("Error taking CPU%d up: %d\n", cpu, error);
    }

    arch_thaw_secondary_cpus_end();
    cpumask_clear(frozen_cpus);
    cpu_maps_update_done();
}
```

### 11.4 PM Notifier Integration

CPU hotplug is disabled during suspend preparation and re-enabled after
resume via a PM notifier (`kernel/cpu.c:2019-2053`):

```c
static int cpu_hotplug_pm_callback(struct notifier_block *nb,
                                   unsigned long action, void *ptr)
{
    switch (action) {
    case PM_SUSPEND_PREPARE:
    case PM_HIBERNATION_PREPARE:
        cpu_hotplug_disable();
        break;

    case PM_POST_SUSPEND:
    case PM_POST_HIBERNATION:
        cpu_hotplug_enable();
        break;
    }
    return NOTIFY_OK;
}
```

### 11.5 The cpuhp_tasks_frozen Flag

The global variable `cpuhp_tasks_frozen` (`kernel/cpu.c:459`) tells
hotplug callbacks whether the operation is part of a suspend/resume
cycle. Callbacks can check this to adjust their behavior (e.g., skip
work that is unnecessary when tasks are already frozen).

```c
bool cpuhp_tasks_frozen;
EXPORT_SYMBOL_GPL(cpuhp_tasks_frozen);
```

---

## 12. SMT (Hyper-Threading) Control

### 12.1 SMT Control States

SMT control is defined in `include/linux/cpu_smt.h:5-11`:

```c
enum cpuhp_smt_control {
    CPU_SMT_ENABLED,           /* SMT is enabled */
    CPU_SMT_DISABLED,          /* SMT is disabled (can be re-enabled) */
    CPU_SMT_FORCE_DISABLED,    /* SMT is irreversibly disabled */
    CPU_SMT_NOT_SUPPORTED,     /* Hardware does not support SMT */
    CPU_SMT_NOT_IMPLEMENTED,   /* Kernel not compiled with SMT support */
};
```

### 12.2 SMT Bootability Check

The `cpu_bootable()` function (`kernel/cpu.c:671-694`) determines
whether a CPU should be allowed to come online based on SMT policy:

```c
static inline bool cpu_bootable(unsigned int cpu)
{
    if (cpu_smt_control == CPU_SMT_ENABLED && cpu_smt_thread_allowed(cpu))
        return true;

    if (cpu_smt_control == CPU_SMT_NOT_IMPLEMENTED)
        return true;

    if (cpu_smt_control == CPU_SMT_NOT_SUPPORTED)
        return true;

    if (topology_is_primary_thread(cpu))
        return true;

    /*
     * On x86 it's required to boot all logical CPUs at least once
     * so that init code can set CR4.MCE on each CPU. Otherwise, a
     * broadcasted MCE with CR4.MCE=0b on any core will shutdown
     * the machine.
     */
    return !cpumask_test_cpu(cpu, &cpus_booted_once_mask);
}
```

### 12.3 Command-Line Control

The `nosmt` kernel parameter disables SMT at boot (`kernel/cpu.c:650-655`):

```c
static int __init smt_cmdline_disable(char *str)
{
    cpu_smt_disable(str && !strcmp(str, "force"));
    return 0;
}
early_param("nosmt", smt_cmdline_disable);
```

Usage:
- `nosmt` -- disable SMT (can be re-enabled at runtime)
- `nosmt=force` -- permanently disable SMT

### 12.4 Sysfs SMT Control

The `/sys/devices/system/cpu/smt/control` file allows runtime SMT
management:
- Write `on` to enable SMT
- Write `off` to disable SMT
- Write `forceoff` to permanently disable SMT
- Read to see current state

---

## 13. Interaction with the Scheduler

The scheduler registers callbacks at multiple points in the hotplug
state machine:

### 13.1 Scheduler Hotplug Callbacks

| State | Startup | Teardown | File:Line |
|-------|---------|----------|-----------|
| `CPUHP_AP_SCHED_STARTING` | `sched_cpu_starting()` | `sched_cpu_dying()` | `kernel/sched/core.c:8338,8401` |
| `CPUHP_AP_SCHED_WAIT_EMPTY` | (none) | `sched_cpu_wait_empty()` | `kernel/sched/core.c:8359` |
| `CPUHP_AP_ACTIVE` | `sched_cpu_activate()` | `sched_cpu_deactivate()` | `kernel/sched/core.c:8229,8267` |

### 13.2 sched_cpu_activate

Called at `CPUHP_AP_ACTIVE` startup to make a CPU available for
scheduling (`kernel/sched/core.c:8229-8265`):

```c
int sched_cpu_activate(unsigned int cpu)
{
    struct rq *rq = cpu_rq(cpu);

    /* Allow regular task scheduling */
    balance_push_set(cpu, false);

    /* Track SMT presence */
    sched_smt_present_inc(cpu);
    set_cpu_active(cpu, true);

    if (sched_smp_initialized) {
        sched_update_numa(cpu, true);
        sched_domains_numa_masks_set(cpu);
        cpuset_cpu_active();           /* rebuild sched domains */
    }

    scx_rq_activate(rq);              /* sched_ext notification */
    sched_set_rq_online(rq, cpu);

    return 0;
}
```

### 13.3 sched_cpu_deactivate

Called at `CPUHP_AP_ACTIVE` teardown to remove a CPU from scheduling
(`kernel/sched/core.c:8267-8328`):

```c
int sched_cpu_deactivate(unsigned int cpu)
{
    struct rq *rq = cpu_rq(cpu);

    nohz_balance_exit_idle(rq);
    set_cpu_active(cpu, false);

    /*
     * Enable balance_push: from this point, the CPU will actively
     * push away all non-per-CPU and non-migrate_disable tasks.
     */
    balance_push_set(cpu, true);

    /*
     * Wait for all preempt-disabled and RCU users to observe
     * the !cpu_active state, so ttwu no longer targets this CPU.
     */
    synchronize_rcu();

    sched_set_rq_offline(rq, cpu);
    scx_rq_deactivate(rq);
    sched_smt_present_dec(cpu);

    /* Rebuild scheduling domains */
    ret = cpuset_cpu_inactive(cpu);
    if (ret) {
        /* Rollback on failure */
        set_cpu_active(cpu, true);
        balance_push_set(cpu, false);
        return ret;
    }
    sched_domains_numa_masks_clear(cpu);
    return 0;
}
```

### 13.4 sched_cpu_dying

Called in the STARTING section (IRQs disabled, via `stop_machine`) to
verify the CPU's runqueue has been properly drained
(`kernel/sched/core.c:8401-8421`):

```c
int sched_cpu_dying(unsigned int cpu)
{
    struct rq *rq = cpu_rq(cpu);

    sched_tick_stop(cpu);

    rq_lock_irqsave(rq, &rf);
    if (rq->nr_running != 1 || rq_has_pinned_tasks(rq)) {
        WARN(true, "Dying CPU not properly vacated!");
        dump_rq_tasks(rq, KERN_WARNING);
    }
    rq_unlock_irqrestore(rq, &rf);

    calc_load_migrate(rq);
    update_max_interval();
    return 0;
}
```

---

## 14. Interaction with Workqueues

Workqueues are tightly integrated with CPU hotplug at two points:

| State | Startup | Teardown |
|-------|---------|----------|
| `CPUHP_WORKQUEUE_PREP` | `workqueue_prepare_cpu()` | (none) |
| `CPUHP_AP_WORKQUEUE_ONLINE` | `workqueue_online_cpu()` | `workqueue_offline_cpu()` |

**PREPARE phase** (`workqueue_prepare_cpu`): Allocates and initializes
the per-CPU worker pools for the incoming CPU.

**ONLINE phase** (`workqueue_online_cpu`): Activates the worker pools,
allowing work items to be scheduled on the new CPU.

**OFFLINE phase** (`workqueue_offline_cpu`): Deactivates the worker pools.
Pending per-CPU work items are drained or migrated. Unbound work items
are redistributed across remaining online CPUs respecting NUMA affinity.

---

## 15. Migration During Offline

### 15.1 Task Migration

When a CPU goes offline, all runnable tasks must be migrated away.
This happens through the `balance_push` mechanism:

1. `sched_cpu_deactivate()` calls `balance_push_set(cpu, true)`,
   which installs a special `balance_push` callback on the dying
   CPU's runqueue.

2. After `synchronize_rcu()`, the `ttwu` (try-to-wake-up) path
   will no longer target this CPU for new wakeups.

3. The `balance_push` callback actively pushes all non-pinned tasks
   to other CPUs whenever the scheduler runs on the dying CPU.

4. `sched_cpu_wait_empty()` blocks at `CPUHP_AP_SCHED_WAIT_EMPTY`
   until `balance_hotplug_wait()` confirms the runqueue is drained.

5. `sched_cpu_dying()` runs inside `stop_machine` and verifies that
   only the stopper task remains on the runqueue (nr_running == 1).

### 15.2 IRQ Migration

During `take_cpu_down()`, the call to `__cpu_disable()` migrates all
IRQ affinities away from the dying CPU. The `irq_lock_sparse()` /
`irq_unlock_sparse()` calls in `takedown_cpu()` prevent IRQ
allocation/deallocation during this transition.

### 15.3 Timer Migration

Timers and hrtimers are migrated by their respective teardown callbacks:
- `CPUHP_AP_HRTIMERS_DYING` -- `hrtimers_cpu_dying()` migrates
  pending hrtimers
- `CPUHP_AP_TICK_DYING` -- `tick_cpu_dying()` handles clockevent
  shutdown
- `CPUHP_TIMERS_PREPARE` teardown -- `timers_dead_cpu()` migrates
  pending timer_list timers

### 15.4 RCU Callback Migration

After the CPU is dead, `takedown_cpu()` calls
`rcutree_migrate_callbacks(cpu)` (`kernel/cpu.c:1348`) to move pending
RCU callbacks from the dead CPU to an online CPU. This must happen
before any further teardown that might depend on RCU.

### 15.5 Workqueue Work Migration

When a CPU goes offline, `workqueue_offline_cpu()` ensures that:
- Pending per-CPU bound work items are completed or drained
- Worker threads for the offline CPU's pools are managed
- Unbound work items redistribute across NUMA-local CPUs

---

## 16. Parallel Bringup

### 16.1 Overview

When `CONFIG_HOTPLUG_PARALLEL` is enabled, the kernel can bring up
multiple CPUs in parallel during boot, avoiding the serial wait for
each AP to respond to the startup IPI (`kernel/cpu.c:1802-1884`).

### 16.2 How It Works

```
1. Run all BP PREPARE states for each AP up to CPUHP_BP_KICK_AP
   (this sends the startup IPI to all APs in parallel)

2. APs proceed through low-level bringup code concurrently and
   wait in cpuhp_ap_sync_alive() for release

3. Release each AP one-by-one for final ONLINE onboarding
   via cpuhp_bringup_mask(mask, ncpus, CPUHP_ONLINE)
```

The feature is controlled by the kernel command line parameter
`cpuhp.parallel` (`kernel/cpu.c:1805-1809`):

```c
static bool __cpuhp_parallel_bringup __ro_after_init = true;

static int __init parallel_bringup_parse_param(char *arg)
{
    return kstrtobool(arg, &__cpuhp_parallel_bringup);
}
early_param("cpuhp.parallel", parallel_bringup_parse_param);
```

### 16.3 SMT-Aware Parallel Bringup

On x86, primary SMT threads must be brought up before secondary threads
to allow microcode updates. The `cpuhp_bringup_cpus_parallel()` function
handles this by doing a two-phase bringup (`kernel/cpu.c:1847-1881`):

```
Phase 1: Bring up primary threads (one per core)
Phase 2: Bring up secondary threads (remaining SMT siblings)
```

---

## 17. Key Source Files

| File | Purpose |
|------|---------|
| `kernel/cpu.c` | Core hotplug state machine, cpu_up/cpu_down, freeze/thaw, SMT control |
| `kernel/smpboot.c` | Per-CPU hotplug kthread framework, idle thread init |
| `include/linux/cpuhotplug.h` | `enum cpuhp_state`, callback registration API |
| `include/linux/cpu.h` | `struct cpu`, cpu_startup_entry, freeze/thaw declarations |
| `include/linux/cpuhplock.h` | cpus_read_lock/cpus_write_lock API |
| `include/linux/cpu_smt.h` | SMT control enum and API |
| `include/linux/smpboot.h` | `struct smp_hotplug_thread` descriptor |
| `drivers/base/cpu.c` | Sysfs CPU device, online/offline store, register_cpu |
| `kernel/sched/core.c` | Scheduler hotplug callbacks (activate/deactivate/dying) |
| `kernel/stop_machine.c` | stop_machine used by CPUHP_TEARDOWN_CPU |
| `arch/x86/kernel/smpboot.c` | x86 AP startup (start_secondary, INIT-SIPI-SIPI) |
| `kernel/sched/topology.c` | Scheduling domain rebuild on hotplug events |
