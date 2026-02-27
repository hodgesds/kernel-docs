# Linux Kernel Entry/Exit Subsystem

## Overview

The kernel entry/exit subsystem manages every transition between user mode and
kernel mode on x86-64. This includes system calls (SYSCALL, SYSENTER, INT
0x80), hardware interrupts, CPU exceptions, and NMIs. Each transition must
save processor state into a `struct pt_regs` on the kernel stack, establish
kernel context (GS base, page tables, RCU, lockdep), perform the requested
work, and then reverse all of those steps on exit -- while defending against
Spectre, Meltdown, MDS, and BHI side-channel attacks at every boundary.

The code lives in two layers: architecture-specific assembly stubs
(`arch/x86/entry/entry_64.S`, `arch/x86/entry/entry_64_compat.S`) and a
generic C framework shared across architectures (`kernel/entry/common.c`,
`include/linux/entry-common.h`).

---

## 1. Processor State: `struct pt_regs`

### 1.1 x86-64 Layout

`arch/x86/include/asm/ptrace.h:103-170`

```c
struct pt_regs {
    /*
     * C ABI says these regs are callee-preserved. They aren't saved on
     * kernel entry unless syscall needs a complete, fully filled
     * "struct pt_regs".
     */
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
    unsigned long bp;
    unsigned long bx;

    /* These regs are callee-clobbered. Always saved on kernel entry. */
    unsigned long r11;
    unsigned long r10;
    unsigned long r9;
    unsigned long r8;
    unsigned long ax;
    unsigned long cx;
    unsigned long dx;
    unsigned long si;
    unsigned long di;

    /*
     * orig_ax is used on entry for:
     * - the syscall number (syscall, sysenter, int80)
     * - error_code stored by the CPU on traps and exceptions
     * - the interrupt number for device interrupts
     */
    unsigned long orig_ax;

    /* The IRETQ return frame starts here */
    unsigned long ip;
    union {
        u16             cs;
        u64             csx;
        struct fred_cs  fred_cs;
    };
    unsigned long flags;
    unsigned long sp;
    union {
        u16             ss;
        u64             ssx;
        struct fred_ss  fred_ss;
    };
};
```

### 1.2 Stack Frame Diagram

When the CPU enters the kernel, hardware pushes the IRET frame. Software then
pushes `orig_ax` and the general-purpose registers to complete `pt_regs`:

```
High addresses (bottom of kernel stack)
  ┌──────────────────────────┐
  │  SS                      │  ← pushed by CPU (IRET frame)
  │  RSP                     │
  │  RFLAGS                  │
  │  CS                      │
  │  RIP                     │
  ├──────────────────────────┤
  │  orig_ax (syscall nr /   │  ← pushed by software
  │           error code)    │
  ├──────────────────────────┤
  │  rdi                     │  ← PUSH_AND_CLEAR_REGS macro
  │  rsi                     │
  │  rdx                     │
  │  rcx                     │
  │  rax                     │
  │  r8                      │
  │  r9                      │
  │  r10                     │
  │  r11                     │
  │  rbx                     │
  │  rbp                     │
  │  r12                     │
  │  r13                     │
  │  r14                     │
  │  r15                     │  ← RSP points here (= &pt_regs)
  └──────────────────────────┘
Low addresses (top of kernel stack)
```

The `PUSH_AND_CLEAR_REGS` macro (`arch/x86/entry/calling.h:125-128`) pushes
all general-purpose registers and then zeroes them to prevent speculative
information leaks:

```asm
.macro PUSH_AND_CLEAR_REGS ...
    PUSH_REGS ...       /* push r15..rdi */
    CLEAR_REGS ...      /* xorl each register for nospec */
.endm
```

`CLEAR_REGS` (`arch/x86/entry/calling.h:100-123`) zeroes rsi, rdx, rcx,
r8-r15, rbx, and optionally rbp to sanitize register values against
speculative execution gadgets.

---

## 2. System Call Entry Paths

### 2.1 64-bit SYSCALL: `entry_SYSCALL_64`

`arch/x86/entry/entry_64.S:87-170`

This is the primary entry point for 64-bit system calls. The SYSCALL
instruction saves RIP to RCX, RFLAGS to R11, then loads new SS, CS, and RIP
from MSRs. It does not save RSP or push anything on the stack.

Register convention on entry:
```
rax   system call number
rcx   return address (saved by SYSCALL)
r11   saved rflags   (saved by SYSCALL)
rdi   arg0
rsi   arg1
rdx   arg2
r10   arg3
r8    arg4
r9    arg5
```

The entry stub performs these steps:

```
entry_SYSCALL_64:                          (entry_64.S:87)
  1. swapgs                                 -- switch to kernel GS base
  2. Save user RSP in TSS scratch space     -- movq %rsp, PER_CPU_VAR(cpu_tss_rw + TSS_sp2)
  3. SWITCH_TO_KERNEL_CR3                   -- KPTI: load kernel page tables
  4. Load kernel stack pointer              -- movq PER_CPU_VAR(pcpu_hot + X86_top_of_stack), %rsp
  5. Construct pt_regs on stack:
       push $__USER_DS                      -- pt_regs->ss
       push saved user RSP                  -- pt_regs->sp
       push %r11                            -- pt_regs->flags
       push $__USER_CS                      -- pt_regs->cs
       push %rcx                            -- pt_regs->ip
       push %rax                            -- pt_regs->orig_ax
       PUSH_AND_CLEAR_REGS rax=$-ENOSYS    -- remaining regs (ax preset to -ENOSYS)
  6. IBRS_ENTER                             -- Spectre v2: set IBRS
  7. UNTRAIN_RET                            -- retpoline: untrain return predictor
  8. CLEAR_BRANCH_HISTORY                   -- BHI: clear branch history buffer
  9. call do_syscall_64                     -- C handler (IRQs disabled)
 10. Return path: SYSRET (fast) or IRET (slow)
```

### 2.2 C Handler: `do_syscall_64()`

`arch/x86/entry/common.c:76-130`

```c
__visible noinstr bool do_syscall_64(struct pt_regs *regs, int nr)
{
    add_random_kstack_offset();
    nr = syscall_enter_from_user_mode(regs, nr);

    instrumentation_begin();

    if (!do_syscall_x64(regs, nr) && !do_syscall_x32(regs, nr) && nr != -1) {
        regs->ax = __x64_sys_ni_syscall(regs);
    }

    instrumentation_end();
    syscall_exit_to_user_mode(regs);

    /* Return true = use SYSRET, false = use IRET */
    /* Checks: RCX==RIP, R11==RFLAGS, CS/SS match, RIP canonical, no RF/TF */
    ...
    return true;
}
```

The function returns a boolean: `true` means the fast SYSRET path is safe,
`false` forces the slower IRET path. SYSRET is unsafe when:
- RCX != RIP or R11 != RFLAGS (ptrace modified them)
- CS/SS do not match MSR_STAR values
- RIP >= TASK_SIZE_MAX (non-canonical RIP causes #GP in kernel mode on Intel)
- RF or TF flags are set

`do_syscall_x64()` (`arch/x86/entry/common.c:42-56`) dispatches the syscall:
```c
static __always_inline bool do_syscall_x64(struct pt_regs *regs, int nr)
{
    unsigned int unr = nr;
    if (likely(unr < NR_syscalls)) {
        unr = array_index_nospec(unr, NR_syscalls);  /* Spectre v1 */
        regs->ax = x64_sys_call(regs, unr);
        return true;
    }
    return false;
}
```

`array_index_nospec()` clamps the syscall number to prevent Spectre v1 bounds
bypass.

### 2.3 SYSRET vs. IRET Return

**SYSRET** (`entry_64.S:137-170`): Fast path. Restores registers, switches to
the trampoline stack (TSS.sp0), performs STACKLEAK_ERASE, switches to user CR3
(`SWITCH_TO_USER_CR3_STACK`), swapgs, `CLEAR_CPU_BUFFERS`, and executes
`sysretq`.

**IRET** (`entry_64.S:559-616`): Slow path via
`swapgs_restore_regs_and_return_to_usermode`. Used for all returns when SYSRET
conditions are not met, and for all interrupt/exception returns to user mode.
With KPTI enabled, copies the IRET frame to a trampoline stack before
switching CR3.

### 2.4 32-bit Compatibility Entry Points

Three entry points exist for 32-bit processes on a 64-bit kernel:

**SYSENTER** -- `entry_SYSENTER_compat` (`arch/x86/entry/entry_64_compat.S:50-134`):
Used by Intel CPUs via the vDSO `__kernel_vsyscall`. SYSENTER does not save
RIP, RSP, or RFLAGS -- the vDSO saves ESP in EBP, and the kernel fills in
placeholder values. Calls `do_SYSENTER_32()`.

**32-bit SYSCALL** -- `entry_SYSCALL_compat` (`arch/x86/entry/entry_64_compat.S:183-285`):
Used by AMD CPUs via the vDSO. SYSCALL saves RIP to RCX and RFLAGS to R11.
Calls `do_fast_syscall_32()`.

**INT 0x80** -- `int80_emulation` (`arch/x86/entry/entry_64_compat.S:294-299`):
Legacy slow path. The assembly stub clears branch history (`CLEAR_BRANCH_HISTORY`)
then jumps to `do_int80_emulation()` (`arch/x86/entry/common.c:210-257`).

32-bit argument mapping (`arch/x86/include/asm/syscall_wrapper.h:80-83`):
```
eax: syscall number
ebx: arg1    ecx: arg2    edx: arg3
esi: arg4    edi: arg5    ebp: arg6
```

### 2.5 Syscall Wrapper Architecture

`arch/x86/include/asm/syscall_wrapper.h`

On x86, `SYSCALL_DEFINEx()` generates multiple stub functions that extract
arguments from `struct pt_regs` according to each ABI:

```
__x64_sys_*()         -- 64-bit: args from rdi, rsi, rdx, r10, r8, r9
__ia32_sys_*()        -- 32-bit: args from bx, cx, dx, si, di, bp
__ia32_compat_sys_*() -- 32-bit compat syscall
__x64_compat_sys_*()  -- x32 compat syscall
```

This prevents raw user register values from leaking into the C call chain.
The stub extracts arguments, passes them through `__se_sys_*()` for
sign-extension, and finally calls `__do_sys_*()`.

---

## 3. IDT Exception and Interrupt Entry

### 3.1 IDT Entry Macros

`arch/x86/include/asm/idtentry.h`

The kernel uses a family of macros to define IDT entry points. Each macro
emits both an assembly stub and a C handler:

| Macro | Use Case |
|-------|----------|
| `DEFINE_IDTENTRY` | Simple exceptions (no error code) |
| `DEFINE_IDTENTRY_ERRORCODE` | Exceptions with hardware error code |
| `DEFINE_IDTENTRY_RAW` | Raw entry, manual enter/exit |
| `DEFINE_IDTENTRY_IRQ` | Device interrupts |
| `DEFINE_IDTENTRY_SYSVEC` | System vector interrupts (APIC timer, IPI) |
| `DEFINE_IDTENTRY_IST` | IST-stack exceptions (#MC, #DB) |
| `DEFINE_IDTENTRY_NMI` | Non-maskable interrupts |

**Standard pattern** (`arch/x86/include/asm/idtentry.h:53-66`):

```c
#define DEFINE_IDTENTRY(func)                                   \
static __always_inline void __##func(struct pt_regs *regs);     \
                                                                \
__visible noinstr void func(struct pt_regs *regs)               \
{                                                               \
    irqentry_state_t state = irqentry_enter(regs);              \
                                                                \
    instrumentation_begin();                                    \
    __##func(regs);                                             \
    instrumentation_end();                                      \
    irqentry_exit(regs, state);                                 \
}                                                               \
                                                                \
static __always_inline void __##func(struct pt_regs *regs)
```

The outer wrapper is `noinstr` (no instrumentation allowed). The actual
handler body runs between `instrumentation_begin()`/`instrumentation_end()`.

**IRQ entry** (`arch/x86/include/asm/idtentry.h:206-222`):

```c
#define DEFINE_IDTENTRY_IRQ(func)                               \
__visible noinstr void func(struct pt_regs *regs,               \
                            unsigned long error_code)           \
{                                                               \
    irqentry_state_t state = irqentry_enter(regs);              \
    u32 vector = (u32)(u8)error_code;                           \
                                                                \
    kvm_set_cpu_l1tf_flush_l1d();                               \
    instrumentation_begin();                                    \
    run_irq_on_irqstack_cond(__##func, regs, vector);           \
    instrumentation_end();                                      \
    irqentry_exit(regs, state);                                 \
}
```

### 3.2 IDT Entry Declarations

`arch/x86/include/asm/idtentry.h:600-780`

Key exception entry points:

```c
/* No error code */
DECLARE_IDTENTRY(X86_TRAP_DE,     exc_divide_error);
DECLARE_IDTENTRY(X86_TRAP_OF,     exc_overflow);
DECLARE_IDTENTRY(X86_TRAP_BR,     exc_bounds);
DECLARE_IDTENTRY(X86_TRAP_NM,     exc_device_not_available);

/* With error code */
DECLARE_IDTENTRY_ERRORCODE(X86_TRAP_TS, exc_invalid_tss);
DECLARE_IDTENTRY_ERRORCODE(X86_TRAP_NP, exc_segment_not_present);
DECLARE_IDTENTRY_ERRORCODE(X86_TRAP_SS, exc_stack_segment);
DECLARE_IDTENTRY_ERRORCODE(X86_TRAP_GP, exc_general_protection);
DECLARE_IDTENTRY_ERRORCODE(X86_TRAP_AC, exc_alignment_check);

/* Raw (manual state management) */
DECLARE_IDTENTRY_RAW(X86_TRAP_UD,       exc_invalid_op);
DECLARE_IDTENTRY_RAW(X86_TRAP_BP,       exc_int3);
DECLARE_IDTENTRY_RAW_ERRORCODE(X86_TRAP_PF, exc_page_fault);

/* IST stack */
DECLARE_IDTENTRY_MCE(X86_TRAP_MC,   exc_machine_check);
DECLARE_IDTENTRY_DEBUG(X86_TRAP_DB, exc_debug);
DECLARE_IDTENTRY_DF(X86_TRAP_DF,    exc_double_fault);
DECLARE_IDTENTRY_NMI(X86_TRAP_NMI,  exc_nmi);

/* INT 0x80 (IA32 emulation) */
DECLARE_IDTENTRY_RAW(IA32_SYSCALL_VECTOR, int80_emulation);

/* Device interrupts */
DECLARE_IDTENTRY_IRQ(X86_TRAP_OTHER, common_interrupt);
DECLARE_IDTENTRY_IRQ(X86_TRAP_OTHER, spurious_interrupt);
```

### 3.3 Assembly Entry: `idtentry` Macro

`arch/x86/entry/entry_64.S:329-365`

For IDT exceptions (not SYSCALL), the CPU pushes an IRET frame (and possibly
an error code), then jumps to the IDT stub:

```asm
.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
    ENDBR
    ASM_CLAC                          /* clear AC flag for SMAP */
    cld                               /* clear direction flag */

    .if \has_error_code == 0
        pushq   $-1                   /* ORIG_RAX: no syscall to restart */
    .endif

    idtentry_body \cfunc \has_error_code
SYM_CODE_END(\asmsym)
.endm
```

The `idtentry_body` macro (`entry_64.S:288-317`) calls `error_entry` to save
registers and switch stacks, then invokes the C handler, then jumps to
`error_return`.

### 3.4 `error_entry` -- Register Save and Stack Switch

`arch/x86/entry/entry_64.S:1003-1086`

```asm
SYM_CODE_START(error_entry)
    PUSH_AND_CLEAR_REGS save_ret=1    /* save all regs + return address */

    testb   $3, CS+8(%rsp)            /* check CPL in saved CS */
    jz      .Lerror_kernelspace       /* kernel entry: skip swapgs/CR3 */

    /* User mode entry: */
    swapgs
    FENCE_SWAPGS_USER_ENTRY           /* Spectre v1: lfence after swapgs */
    SWITCH_TO_KERNEL_CR3              /* KPTI: switch page tables */
    IBRS_ENTER                        /* Spectre v2: set IBRS */
    UNTRAIN_RET_FROM_CALL             /* retpoline mitigation */

    leaq    8(%rsp), %rdi             /* pt_regs pointer */
    jmp     sync_regs                 /* copy to thread stack */

.Lerror_kernelspace:
    /* Handle special cases: IRET faults, gs_change faults */
    FENCE_SWAPGS_KERNEL_ENTRY         /* lfence to prevent speculative swapgs */
    ...
SYM_CODE_END(error_entry)
```

### 3.5 `error_return` -- Return Dispatch

`arch/x86/entry/entry_64.S:1088-1094`

```asm
SYM_CODE_START_LOCAL(error_return)
    testb   $3, CS(%rsp)              /* returning to user or kernel? */
    jz      restore_regs_and_return_to_kernel
    jmp     swapgs_restore_regs_and_return_to_usermode
SYM_CODE_END(error_return)
```

---

## 4. Generic Entry/Exit Framework

### 4.1 `irqentry_state_t` -- Opaque State Token

`include/linux/entry-common.h:459-464`

```c
typedef struct irqentry_state {
    union {
        bool    exit_rcu;   /* used by irqentry_enter/exit */
        bool    lockdep;    /* used by irqentry_nmi_enter/exit */
    };
} irqentry_state_t;
```

This token is returned by `irqentry_enter()` and passed to `irqentry_exit()`
so the exit path can undo exactly the state changes made on entry.

### 4.2 `enter_from_user_mode()`

`include/linux/entry-common.h:107-119`

Called on every user-to-kernel transition (syscalls, interrupts, exceptions):

```c
static __always_inline void enter_from_user_mode(struct pt_regs *regs)
{
    arch_enter_from_user_mode(regs);    /* arch-specific sanity checks */
    lockdep_hardirqs_off(CALLER_ADDR0); /* tell lockdep: IRQs are off */

    CT_WARN_ON(__ct_state() != CT_STATE_USER);
    user_exit_irqoff();                 /* context tracking: user -> kernel */

    instrumentation_begin();
    kmsan_unpoison_entry_regs(regs);
    trace_hardirqs_off_finish();
    instrumentation_end();
}
```

This function:
1. Informs lockdep that IRQs are disabled
2. Transitions context tracking from user to kernel (reactivates RCU)
3. Traces the IRQ-off state transition

### 4.3 `irqentry_enter()`

`kernel/entry/common.c:236-301`

The generic entry point for interrupts and exceptions:

```c
noinstr irqentry_state_t irqentry_enter(struct pt_regs *regs)
{
    irqentry_state_t ret = { .exit_rcu = false };

    if (user_mode(regs)) {
        irqentry_enter_from_user_mode(regs);
        return ret;
    }

    /* Kernel mode: handle idle task RCU state */
    if (!IS_ENABLED(CONFIG_TINY_RCU) && is_idle_task(current)) {
        lockdep_hardirqs_off(CALLER_ADDR0);
        ct_irq_enter();             /* reactivate RCU for idle */
        ...
        ret.exit_rcu = true;
        return ret;
    }

    /* Normal kernel mode: RCU is already watching */
    lockdep_hardirqs_off(CALLER_ADDR0);
    rcu_irq_enter_check_tick();     /* check if tick needs restart */
    trace_hardirqs_off_finish();
    return ret;
}
```

Three cases are handled:
- **User mode**: Full `enter_from_user_mode()` transition
- **Kernel idle**: RCU may be idle; must call `ct_irq_enter()` to reactivate
- **Kernel normal**: RCU is watching; just check the tick

### 4.4 `irqentry_exit()`

`kernel/entry/common.c:328-367`

```c
noinstr void irqentry_exit(struct pt_regs *regs, irqentry_state_t state)
{
    lockdep_assert_irqs_disabled();

    if (user_mode(regs)) {
        irqentry_exit_to_user_mode(regs);   /* full exit-to-user work */
    } else if (!regs_irqs_disabled(regs)) {
        if (state.exit_rcu) {
            /* Idle task: undo ct_irq_enter() */
            trace_hardirqs_on_prepare();
            lockdep_hardirqs_on_prepare();
            ct_irq_exit();
            lockdep_hardirqs_on(CALLER_ADDR0);
            return;
        }
        /* Normal kernel: maybe preempt */
        if (IS_ENABLED(CONFIG_PREEMPTION))
            irqentry_exit_cond_resched();
        trace_hardirqs_on();
    } else {
        /* IRQs were disabled: just undo RCU if needed */
        if (state.exit_rcu)
            ct_irq_exit();
    }
}
```

### 4.5 NMI Entry/Exit

`kernel/entry/common.c:369-404`

NMIs require special handling because they can interrupt any code, including
the entry/exit paths themselves:

```c
irqentry_state_t noinstr irqentry_nmi_enter(struct pt_regs *regs)
{
    irqentry_state_t irq_state;

    irq_state.lockdep = lockdep_hardirqs_enabled();

    __nmi_enter();                      /* increment NMI preempt count */
    lockdep_hardirqs_off(CALLER_ADDR0);
    lockdep_hardirq_enter();
    ct_nmi_enter();                     /* context tracking for NMI */
    ...
    return irq_state;
}

void noinstr irqentry_nmi_exit(struct pt_regs *regs, irqentry_state_t irq_state)
{
    ...
    ct_nmi_exit();
    lockdep_hardirq_exit();
    if (irq_state.lockdep)
        lockdep_hardirqs_on(CALLER_ADDR0);
    __nmi_exit();
}
```

---

## 5. Syscall Entry/Exit Work

### 5.1 `syscall_enter_from_user_mode()`

`include/linux/entry-common.h:191-203`

Combines state establishment with work checking:

```c
static __always_inline long syscall_enter_from_user_mode(
        struct pt_regs *regs, long syscall)
{
    long ret;

    enter_from_user_mode(regs);       /* lockdep, context tracking, trace */

    instrumentation_begin();
    local_irq_enable();
    ret = syscall_enter_from_user_mode_work(regs, syscall);
    instrumentation_end();

    return ret;
}
```

### 5.2 Syscall Entry Work Flags

`include/linux/entry-common.h:44-50`

```c
#define SYSCALL_WORK_ENTER  (SYSCALL_WORK_SECCOMP |             \
                             SYSCALL_WORK_SYSCALL_TRACEPOINT |  \
                             SYSCALL_WORK_SYSCALL_TRACE |       \
                             SYSCALL_WORK_SYSCALL_EMU |         \
                             SYSCALL_WORK_SYSCALL_AUDIT |       \
                             SYSCALL_WORK_SYSCALL_USER_DISPATCH | \
                             ARCH_SYSCALL_WORK_ENTER)
```

When any of these flags are set in `current_thread_info()->syscall_work`,
`syscall_trace_enter()` (`kernel/entry/common.c:28-72`) is called:

```c
long syscall_trace_enter(struct pt_regs *regs, long syscall,
                         unsigned long work)
{
    /* 1. Syscall User Dispatch */
    if (work & SYSCALL_WORK_SYSCALL_USER_DISPATCH)
        if (syscall_user_dispatch(regs)) return -1L;

    /* 2. ptrace */
    if (work & (SYSCALL_WORK_SYSCALL_TRACE | SYSCALL_WORK_SYSCALL_EMU))
        ret = ptrace_report_syscall_entry(regs);

    /* 3. seccomp (after ptrace to catch tracer changes) */
    if (work & SYSCALL_WORK_SECCOMP)
        ret = __secure_computing(NULL);

    /* 4. Tracepoints / BPF */
    if (work & SYSCALL_WORK_SYSCALL_TRACEPOINT)
        trace_sys_enter(regs, syscall);

    /* 5. Audit */
    syscall_enter_audit(regs, syscall);

    return ret ? : syscall;
}
```

Returning -1 skips the syscall entirely (e.g., seccomp KILL, ptrace
SYSCALL_EMU).

### 5.3 `syscall_exit_to_user_mode()`

`kernel/entry/common.c:215-221`

```c
__visible noinstr void syscall_exit_to_user_mode(struct pt_regs *regs)
{
    instrumentation_begin();
    __syscall_exit_to_user_mode_work(regs);
    instrumentation_end();
    exit_to_user_mode();
}
```

This calls `syscall_exit_to_user_mode_prepare()` (`kernel/entry/common.c:180-201`):

```c
static void syscall_exit_to_user_mode_prepare(struct pt_regs *regs)
{
    CT_WARN_ON(ct_state() != CT_STATE_KERNEL);

    rseq_syscall(regs);           /* restartable sequences */

    if (unlikely(work & SYSCALL_WORK_EXIT))
        syscall_exit_work(regs, work);
}
```

Syscall exit work (`kernel/entry/common.c:149-174`) handles:
- `audit_syscall_exit()`
- `trace_sys_exit()` tracepoint
- `ptrace_report_syscall_exit()` (single-step reporting)
- Syscall User Dispatch cleanup

---

## 6. Exit-to-User-Mode Loop

### 6.1 TIF Flags Checked on Exit

`include/linux/entry-common.h:65-69`

```c
#define EXIT_TO_USER_MODE_WORK                      \
    (_TIF_SIGPENDING | _TIF_NOTIFY_RESUME |         \
     _TIF_UPROBE | _TIF_NEED_RESCHED |              \
     _TIF_NEED_RESCHED_LAZY | _TIF_PATCH_PENDING |  \
     _TIF_NOTIFY_SIGNAL | ARCH_EXIT_TO_USER_MODE_WORK)
```

x86 TIF flag definitions (`arch/x86/include/asm/thread_info.h:87-109`):

```c
#define TIF_NOTIFY_RESUME       1
#define TIF_SIGPENDING          2
#define TIF_NEED_RESCHED        3
#define TIF_NEED_RESCHED_LAZY   4
#define TIF_SINGLESTEP          5
#define TIF_UPROBE              12
#define TIF_PATCH_PENDING       13
#define TIF_NOTIFY_SIGNAL       17
```

### 6.2 `exit_to_user_mode_loop()`

`kernel/entry/common.c:90-134`

This loop runs with interrupts enabled until all pending work is consumed:

```c
unsigned long exit_to_user_mode_loop(struct pt_regs *regs,
                                     unsigned long ti_work)
{
    while (ti_work & EXIT_TO_USER_MODE_WORK) {

        local_irq_enable_exit_to_user(ti_work);

        if (ti_work & (_TIF_NEED_RESCHED | _TIF_NEED_RESCHED_LAZY))
            schedule();

        if (ti_work & _TIF_UPROBE)
            uprobe_notify_resume(regs);

        if (ti_work & _TIF_PATCH_PENDING)
            klp_update_patch_state(current);

        if (ti_work & (_TIF_SIGPENDING | _TIF_NOTIFY_SIGNAL))
            arch_do_signal_or_restart(regs);

        if (ti_work & _TIF_NOTIFY_RESUME)
            resume_user_mode_work(regs);

        arch_exit_to_user_mode_work(regs, ti_work);

        local_irq_disable_exit_to_user();
        tick_nohz_user_enter_prepare();
        ti_work = read_thread_flags();     /* re-read: work may have been added */
    }

    return ti_work;
}
```

The loop order matters:
1. **Schedule** first -- honor `need_resched` before delivering signals
2. **Uprobes** -- breakpoint notifications
3. **Live patching** -- apply pending patches before returning
4. **Signals** -- deliver pending signals (may modify pt_regs for handler)
5. **Notify resume** -- callbacks for things like rseq, task_work
6. **Re-check** -- disable interrupts, re-read flags, loop if more work

### 6.3 `exit_to_user_mode()`

`include/linux/entry-common.h:357-367`

The final transition, called with interrupts disabled:

```c
static __always_inline void exit_to_user_mode(void)
{
    instrumentation_begin();
    trace_hardirqs_on_prepare();
    lockdep_hardirqs_on_prepare();
    instrumentation_end();

    user_enter_irqoff();           /* context tracking: kernel -> user */
    arch_exit_to_user_mode();      /* arch speculation mitigations */
    lockdep_hardirqs_on(CALLER_ADDR0);
}
```

---

## 7. Complete Entry/Exit Flow Diagram

```
                     USER MODE
                        |
          ╔═════════════╧═════════════╗
          ║   SYSCALL / INT / trap    ║
          ╚═════════════╤═════════════╝
                        |
         ┌──────────────┴──────────────┐
         │ 1. CPU: push IRET frame     │
         │    (SS, RSP, RFLAGS, CS, IP)│
         │    [+ error code if trap]   │
         │                             │
         │ 2. ASM stub:                │
         │    swapgs                   │
         │    SWITCH_TO_KERNEL_CR3     │
         │    switch to kernel stack   │
         │    PUSH_AND_CLEAR_REGS     │
         │    IBRS_ENTER              │
         │    UNTRAIN_RET             │
         │    CLEAR_BRANCH_HISTORY    │
         └──────────────┬──────────────┘
                        |
         ┌──────────────┴──────────────┐
         │ 3. enter_from_user_mode()   │
         │    - lockdep: IRQs off      │
         │    - context tracking:      │
         │      user_exit_irqoff()     │
         │      (RCU reactivated)      │
         │    - trace: hardirqs off    │
         └──────────────┬──────────────┘
                        |
         ┌──────────────┴──────────────┐
         │ 4. instrumentation_begin()  │
         │    local_irq_enable()       │
         │    syscall_enter_work()     │
         │    - ptrace, seccomp, audit │
         │                             │
         │ 5. Execute syscall / handle │
         │    exception / run IRQ      │
         │                             │
         │ 6. syscall_exit_work()      │
         │    - audit, trace, ptrace   │
         │    instrumentation_end()    │
         └──────────────┬──────────────┘
                        |
         ┌──────────────┴──────────────┐
         │ 7. exit_to_user_mode_loop() │
         │    while (TIF work pending) │
         │      schedule()             │
         │      signals                │
         │      uprobes, livepatch     │
         │      notify_resume          │
         └──────────────┬──────────────┘
                        |
         ┌──────────────┴──────────────┐
         │ 8. exit_to_user_mode()      │
         │    - trace: hardirqs on     │
         │    - context tracking:      │
         │      user_enter_irqoff()    │
         │      (RCU may go idle)      │
         │    - arch mitigations       │
         │    - lockdep: IRQs on       │
         └──────────────┬──────────────┘
                        |
         ┌──────────────┴──────────────┐
         │ 9. ASM return:              │
         │    IBRS_EXIT                │
         │    POP_REGS                 │
         │    SWITCH_TO_USER_CR3       │
         │    swapgs                   │
         │    CLEAR_CPU_BUFFERS        │
         │    SYSRET / IRET            │
         └──────────────┬──────────────┘
                        |
                     USER MODE
```

---

## 8. KPTI (Kernel Page Table Isolation)

### 8.1 Purpose

KPTI mitigates the Meltdown vulnerability (CVE-2017-5754) by maintaining two
sets of page tables per process:
- **Kernel page tables**: Map all of kernel memory. Used while in kernel mode.
- **User page tables**: Map only user memory plus a minimal set of kernel
  pages (entry/exit trampolines, per-CPU data). Used while in user mode.

### 8.2 CR3 Switching

`arch/x86/entry/calling.h:150-290`

The kernel and user page table directories are placed 4K apart in physical
memory. Bit 12 of CR3 selects between them:

```c
#define PTI_USER_PGTABLE_BIT    PAGE_SHIFT          /* bit 12 */
#define PTI_USER_PGTABLE_MASK   (1 << PTI_USER_PGTABLE_BIT)
```

**Entry** -- `SWITCH_TO_KERNEL_CR3` (`calling.h:172-178`):
```asm
.macro SWITCH_TO_KERNEL_CR3 scratch_reg:req
    ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_PTI
    mov     %cr3, \scratch_reg
    ADJUST_KERNEL_CR3 \scratch_reg    /* clear bit 12 + PCID bits */
    mov     \scratch_reg, %cr3
.Lend_\@:
.endm
```

**Exit** -- `SWITCH_TO_USER_CR3` (`calling.h:183-213`):
```asm
.macro SWITCH_TO_USER_CR3 scratch_reg:req scratch_reg2:req
    mov     %cr3, \scratch_reg
    /* Check if PCID ASID needs flush */
    ...
    SET_NOFLUSH_BIT \scratch_reg      /* or: flush if needed */
    orq     $(PTI_USER_PCID_MASK), \scratch_reg   /* flip ASID */
    orq     $(PTI_USER_PGTABLE_MASK), \scratch_reg /* flip PGD */
    mov     \scratch_reg, %cr3
.endm
```

When PCID is available, the kernel uses the NOFLUSH bit to avoid TLB flushes
on CR3 switches, using separate PCIDs for kernel and user page tables.

### 8.3 Trampoline Stack

With KPTI, the IRET return to user mode cannot happen from the kernel stack
(which is unmapped in user page tables). Instead, the return path
(`swapgs_restore_regs_and_return_to_usermode`, `entry_64.S:583-616`):

1. Pops all general registers except RDI
2. Switches RSP to the per-CPU trampoline stack (`TSS.sp0`)
3. Copies the IRET frame to the trampoline stack
4. Switches CR3 to user page tables
5. Restores RDI, executes swapgs + IRET

---

## 9. Spectre/Meltdown/MDS Mitigations at Entry/Exit

### 9.1 Spectre v1: Bounds Check Bypass

- `array_index_nospec()` in syscall dispatch (`arch/x86/entry/common.c:51`)
  clamps the syscall number after the bounds check
- `FENCE_SWAPGS_USER_ENTRY` / `FENCE_SWAPGS_KERNEL_ENTRY`
  (`arch/x86/entry/calling.h:362-367`) insert LFENCE after conditional swapgs
  to prevent speculative use of wrong GS base

### 9.2 Spectre v2: Branch Target Injection

**IBRS** (`arch/x86/entry/calling.h:304-327`):
```asm
.macro IBRS_ENTER
    ALTERNATIVE "jmp .Lend_\@", "", X86_FEATURE_KERNEL_IBRS
    movl    $MSR_IA32_SPEC_CTRL, %ecx
    movq    PER_CPU_VAR(x86_spec_ctrl_current), %rdx
    movl    %edx, %eax
    shr     $32, %rdx
    wrmsr
.Lend_\@:
.endm
```

On entry, IBRS is enabled via MSR write to restrict indirect branch
speculation. On exit (`IBRS_EXIT`), it is cleared.

**Retpoline**: `UNTRAIN_RET` replaces indirect branches with a retpoline
sequence that captures speculative execution in an infinite loop.

**CLEAR_BRANCH_HISTORY** (`arch/x86/include/asm/nospec-branch.h:339-341`):
```asm
.macro CLEAR_BRANCH_HISTORY
    ALTERNATIVE "", "call clear_bhb_loop", X86_FEATURE_CLEAR_BHB_LOOP
.endm
```

Clears the Branch History Buffer to prevent BHI (Branch History Injection)
attacks. Called at every syscall entry point.

### 9.3 Meltdown

Mitigated by KPTI (Section 8). The kernel page tables are not mapped when
executing in user mode, so speculative reads of kernel memory fault.

### 9.4 MDS/TAA: Microarchitectural Data Sampling

`CLEAR_CPU_BUFFERS` (`arch/x86/include/asm/nospec-branch.h:325-336`):
```asm
.macro CLEAR_CPU_BUFFERS
    ALTERNATIVE "", "verw mds_verw_sel(%rip)", X86_FEATURE_CLEAR_CPU_BUF
.endm
```

The `VERW` instruction (with a memory operand) is overloaded by microcode to
flush CPU-internal buffers. It is called just before `sysretq` and `iretq`
to prevent user mode from reading stale kernel data from fill buffers.

### 9.5 Register Clearing

`CLEAR_REGS` (`arch/x86/entry/calling.h:100-123`) zeroes all general-purpose
registers after saving them to `pt_regs`, preventing speculative use of
user-provided values in kernel context.

### 9.6 Stack Randomization

`add_random_kstack_offset()` is called at the start of every `do_syscall_64()`
and `do_int80_emulation()` to randomize the kernel stack offset, making
stack-based side channels harder to exploit.

---

## 10. The `noinstr` Requirement

### 10.1 Definition

`include/linux/compiler_types.h:370-375`

```c
#define __noinstr_section(section)                          \
    noinline notrace __attribute((__section__(section)))    \
    __no_kcsan __no_sanitize_address __no_profile           \
    __no_sanitize_coverage __no_sanitize_memory __signed_wrap

#define noinstr __noinstr_section(".noinstr.text")
```

### 10.2 Purpose

Between `enter_from_user_mode()` and `instrumentation_begin()`, and between
`instrumentation_end()` and `exit_to_user_mode()`, the kernel is in a
transitional state where:

- RCU may not be watching
- Lockdep state may be inconsistent
- Tracing is not yet set up

Any instrumentation (ftrace, KASAN, KCSAN, GCOV, etc.) in this window could:
- Take RCU-dependent locks without RCU watching
- Generate false lockdep warnings
- Call into code that assumes full kernel context

The `noinstr` annotation ensures the compiler places these functions in
`.noinstr.text` and disables all forms of instrumentation. The `objtool` tool
validates at build time that no `noinstr` function calls into instrumentable
code outside of `instrumentation_begin()`/`instrumentation_end()` regions.

### 10.3 Pattern

Every entry point handler follows this pattern:

```c
__visible noinstr void handler(struct pt_regs *regs)
{
    irqentry_state_t state = irqentry_enter(regs);  /* noinstr */

    instrumentation_begin();    /* === instrumentable code allowed === */
    __handler(regs);            /* the actual work */
    instrumentation_end();      /* === end instrumentable code === */

    irqentry_exit(regs, state);                     /* noinstr */
}
```

---

## 11. Context Tracking

### 11.1 States

Context tracking (`include/linux/context_tracking.h`) tracks whether the CPU
is executing user code, kernel code, or idle code. This is critical for
`NO_HZ_FULL` (tickless) operation where the timer tick is stopped when a CPU
runs in user mode.

Key transitions at entry/exit:
- `user_exit_irqoff()` -- called on entry from user mode; reactivates RCU
- `user_enter_irqoff()` -- called on exit to user mode; may deactivate RCU
- `ct_irq_enter()` / `ct_irq_exit()` -- for interrupts hitting idle tasks

### 11.2 RCU Implications

When a CPU runs in user mode with `NO_HZ_FULL`, RCU considers the CPU to be
in an extended quiescent state (EQS). On kernel entry,
`user_exit_irqoff()` calls `ct_user_exit()` to notify RCU that the CPU
is active again. On exit, `user_enter_irqoff()` reverses this.

For interrupts hitting idle CPUs, `irqentry_enter()` sets `exit_rcu = true`
in the state token. `irqentry_exit()` then calls `ct_irq_exit()` to restore
the quiescent state.

---

## 12. Paranoid Entry (IST and NMI)

### 12.1 Paranoid vs. Normal Entry

Some exceptions (#DB, #MC, #DF, NMI) can arrive at any point, including
during other entry/exit code. These use the paranoid entry path with IST
(Interrupt Stack Table) stacks:

`arch/x86/entry/entry_64.S:397-430` (`idtentry_mce_db` macro):
```asm
    testb   $3, CS-ORIG_RAX(%rsp)     /* from user or kernel? */
    jnz     .Lfrom_usermode_switch_stack_\@

    /* Kernel mode: paranoid entry */
    call    paranoid_entry             /* SAVE_AND_SWITCH_TO_KERNEL_CR3 */
    call    \cfunc
    jmp     paranoid_exit

.Lfrom_usermode_switch_stack_\@:
    idtentry_body noist_\cfunc         /* normal path for user mode */
```

`paranoid_entry` saves/restores CR3 independently because the exception may
have arrived while the CPU was in the middle of a CR3 switch.
`paranoid_exit` restores the original CR3 using `PARANOID_RESTORE_CR3`.

### 12.2 NMI Nesting

NMIs on x86-64 can nest in a narrow window after IRET (the "NMI executing"
problem). The kernel handles this by using the IST stack and checking for
re-entrant NMI delivery at the assembly level.

---

## 13. Key Source Files

| File | Purpose |
|------|---------|
| `arch/x86/entry/entry_64.S` | 64-bit SYSCALL entry, IDT stubs, error_entry, return paths |
| `arch/x86/entry/entry_64_compat.S` | 32-bit SYSENTER, SYSCALL, INT 0x80 compat stubs |
| `arch/x86/entry/common.c` | `do_syscall_64()`, `do_int80_emulation()`, `do_fast_syscall_32()` |
| `arch/x86/entry/calling.h` | PUSH/POP_REGS, KPTI CR3 macros, IBRS, FENCE_SWAPGS |
| `arch/x86/include/asm/idtentry.h` | DEFINE_IDTENTRY* macros, IDT entry declarations |
| `arch/x86/include/asm/ptrace.h` | `struct pt_regs` definition |
| `arch/x86/include/asm/syscall_wrapper.h` | Syscall stub generation, ABI register mapping |
| `arch/x86/include/asm/thread_info.h` | TIF_* flag definitions |
| `arch/x86/include/asm/nospec-branch.h` | CLEAR_CPU_BUFFERS, CLEAR_BRANCH_HISTORY, UNTRAIN_RET |
| `include/linux/entry-common.h` | Generic entry/exit API, EXIT_TO_USER_MODE_WORK |
| `kernel/entry/common.c` | `irqentry_enter/exit()`, `exit_to_user_mode_loop()`, syscall trace |
| `include/linux/thread_info.h` | SYSCALL_WORK_* flag definitions |
| `include/linux/compiler_types.h` | `noinstr` macro definition |

---

## 14. Subsystem Invariants

- Every user-to-kernel transition must call `enter_from_user_mode()` (or its
  wrappers) before any instrumentable code runs.
- Every kernel-to-user transition must call `exit_to_user_mode()` as the last
  step, after disabling interrupts.
- Code between `enter_from_user_mode()` and `instrumentation_begin()` must be
  `noinstr` -- no tracing, no sanitizers, no RCU-dependent operations.
- KPTI CR3 switches happen in assembly, before/after C code runs, ensuring
  kernel page tables are never visible to user-mode speculation.
- `CLEAR_CPU_BUFFERS` (VERW) is called immediately before every `sysretq`
  and `iretq` to user mode.
- `IBRS_ENTER` is called after CR3 and GS are established, before the first
  RET instruction (including UNTRAIN_RET which contains a RET).
- `irqentry_state_t` tokens are opaque and must be passed from enter to the
  matching exit function to correctly undo RCU and lockdep state.
- The `exit_to_user_mode_loop()` re-reads TIF flags with interrupts disabled
  after each iteration to avoid missing newly set flags.
- SYSRET is only used when all safety conditions are verified; any doubt
  triggers the IRET fallback path.
- `array_index_nospec()` is used for all syscall table lookups to prevent
  Spectre v1 bounds-bypass attacks.
