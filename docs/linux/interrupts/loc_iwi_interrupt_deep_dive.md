# LOC and IWI: Deep Technical Investigation of Linux Kernel Interrupt Types

**Date**: 2026-03-15
**Kernel version context**: Linux v6.12 (stable) unless noted
**Scope**: x86-64

---

## Table of Contents

1. [LOC — Local Timer Interrupt](#1-loc--local-timer-interrupt)
   1. [What Generates the LOC Interrupt](#11-what-generates-the-loc-interrupt)
   2. [APIC Timer Programming Model](#12-apic-timer-programming-model)
   3. [Interrupt Handling Call Chain](#13-interrupt-handling-call-chain)
   4. [hrtimers and the LOC Handler](#14-hrtimers-and-the-loc-handler)
   5. [CONFIG_HZ and the Scheduler Tick](#15-confighz-and-the-scheduler-tick)
   6. [NO_HZ_IDLE: Suppressing LOC on Idle CPUs](#16-no_hz_idle-suppressing-loc-on-idle-cpus)
   7. [NO_HZ_FULL: Suppressing LOC on Single-Task CPUs](#17-no_hz_full-suppressing-loc-on-single-task-cpus)
   8. [The Residual ~1 Hz Tick](#18-the-residual-1-hz-tick)
   9. [Tick Dependency Tracking](#19-tick-dependency-tracking)
   10. [Conditions That Force Tick Re-enablement](#110-conditions-that-force-tick-re-enablement)
   11. [tick_nohz_stop_sched_tick and tick_nohz_restart_sched_tick](#111-tick_nohz_stop_sched_tick-and-tick_nohz_restart_sched_tick)
   12. [TSC-Deadline Mode vs APIC Countdown Mode](#112-tsc-deadline-mode-vs-apic-countdown-mode)
   13. [The task_isolation Patchset](#113-the-task_isolation-patchset)
2. [IWI — IRQ Work Interrupt](#2-iwi--irq-work-interrupt)
   1. [The irq_work Subsystem: Problem Statement](#21-the-irq_work-subsystem-problem-statement)
   2. [struct irq_work and State Machine](#22-struct-irq_work-and-state-machine)
   3. [IRQ_WORK_LAZY and IRQ_WORK_HARD_IRQ Flags](#23-irq_work_lazy-and-irq_work_hard_irq-flags)
   4. [How IWI Is Raised: Self-IPI via IRQ_WORK_VECTOR](#24-how-iwi-is-raised-self-ipi-via-irq_work_vector)
   5. [The IWI Handler: sysvec_irq_work](#25-the-iwi-handler-sysvec_irq_work)
   6. [Lazy vs Non-Lazy Work: The Self-IPI Suppression Logic](#26-lazy-vs-non-lazy-work-the-self-ipi-suppression-logic)
   7. [irq_work_tick: Processing Lazy Work at Timer Tick](#27-irq_work_tick-processing-lazy-work-at-timer-tick)
   8. [irq_work_needs_cpu: Preventing Tick Stop](#28-irq_work_needs_cpu-preventing-tick-stop)
   9. [Subsystems That Use irq_work_queue](#29-subsystems-that-use-irq_work_queue)
   10. [Lazy irq_work on nohz_full CPUs: The Deferred Indefinitely Problem](#210-lazy-irq_work-on-nohz_full-cpus-the-deferred-indefinitely-problem)
   11. [IWI Counts in /proc/interrupts: What Drives Them](#211-iwi-counts-in-procinterrupts-what-drives-them)
3. [Cross-Cutting Interactions](#3-cross-cutting-interactions)
4. [Observability and Diagnosis](#4-observability-and-diagnosis)

---

## 1. LOC — Local Timer Interrupt

### 1.1 What Generates the LOC Interrupt

Every logical CPU on an x86-64 system has its own **Local APIC** (Advanced Programmable Interrupt Controller), a per-CPU interrupt controller integrated into the processor die. The Local APIC contains a **Local Timer** unit that can fire a periodic or one-shot interrupt directly to the CPU it belongs to, without involving the I/O APIC or any cross-CPU routing.

The LOC entry in `/proc/interrupts` counts these per-CPU local timer interrupt deliveries. Each CPU has an independent counter. The vector assigned to this interrupt is `LOCAL_TIMER_VECTOR = 0xec`, defined in `arch/x86/include/asm/irq_vectors.h`.

The interrupt entry point is defined via:

```c
// arch/x86/kernel/apic/apic.c
DEFINE_IDTENTRY_SYSVEC(sysvec_apic_timer_interrupt)
{
    struct pt_regs *old_regs = set_irq_regs(regs);
    apic_eoi();
    trace_local_timer_entry(LOCAL_TIMER_VECTOR);
    local_apic_timer_interrupt();
    trace_local_timer_exit(LOCAL_TIMER_VECTOR);
    set_irq_regs(old_regs);
}
```

`local_apic_timer_interrupt()` retrieves the per-CPU clock event device (`lapic_events`) and calls its registered `event_handler`:

```c
static void local_apic_timer_interrupt(void)
{
    struct clock_event_device *evt = this_cpu_ptr(&lapic_events);
    if (!evt->event_handler) {
        pr_warn("Spurious LAPIC timer interrupt on cpu %d\n",
                smp_processor_id());
        lapic_timer_shutdown(evt);
        return;
    }
    inc_irq_stat(apic_timer_irqs);
    evt->event_handler(evt);
}
```

The `apic_timer_irqs` per-CPU counter is what `/proc/interrupts` reads for the LOC row.

### 1.2 APIC Timer Programming Model

The APIC timer is programmed via two hardware registers and one LVTT (Local Vector Table Timer) register:

| Register | Purpose |
|---|---|
| `APIC_LVTT` | Delivery mode, vector, mask bit, timer mode bits |
| `APIC_TMICT` | Initial count (countdown start value) |
| `APIC_TDCR` | Timer divide configuration register (clock divisor) |
| `MSR_IA32_TSC_DEADLINE` | TSC deadline value (TSC-deadline mode only) |

Three operating modes:

**Periodic mode** (`APIC_LVT_TIMER_PERIODIC` set in APIC_LVTT): The timer counts down from `APIC_TMICT` to zero, fires the interrupt, and automatically reloads. Rate is fixed at boot. Used only on very old kernels or non-HPET systems. `lapic_timer_set_periodic()` calls `lapic_timer_set_periodic_oneshot(evt, false)` → `__setup_APIC_LVTT(lapic_timer_period, false, 1)`.

**One-shot mode** (neither periodic nor TSC-deadline bit set): The timer counts down once. After firing, it must be explicitly re-armed. `lapic_next_event()` programs the next event:

```c
static int lapic_next_event(unsigned long delta,
                            struct clock_event_device *evt)
{
    apic_write(APIC_TMICT, delta);
    return 0;
}
```

**TSC-deadline mode** (`APIC_LVT_TIMER_TSCDEADLINE` set): The APIC timer fires when the CPU's TSC reaches the value written to `MSR_IA32_TSC_DEADLINE`. This bypasses the countdown register entirely:

```c
static int lapic_next_deadline(unsigned long delta,
                               struct clock_event_device *evt)
{
    u64 tsc;
    weak_wrmsr_fence();
    tsc = rdtsc();
    wrmsrl(MSR_IA32_TSC_DEADLINE, tsc + (((u64) delta) * TSC_DIVISOR));
    return 0;
}
```

The `weak_wrmsr_fence()` is necessary because x2APIC mode requires an `mfence` between writing `APIC_LVTT` and `MSR_IA32_TSC_DEADLINE` to guarantee ordering. TSC-deadline mode is advertised via `X86_FEATURE_TSC_DEADLINE_TIMER` (CPUID leaf 1, ECX bit 24).

The `lapic_clockevent` structure registers with the clock event framework:

```c
static struct clock_event_device lapic_clockevent = {
    .name           = "lapic",
    .features       = CLOCK_EVT_FEAT_PERIODIC | CLOCK_EVT_FEAT_ONESHOT |
                      CLOCK_EVT_FEAT_C3STOP | CLOCK_EVT_FEAT_DUMMY,
    .set_state_shutdown  = lapic_timer_shutdown,
    .set_state_periodic  = lapic_timer_set_periodic,
    .set_state_oneshot   = lapic_timer_set_oneshot,
    .set_next_event      = lapic_next_event,   /* or lapic_next_deadline in TSC-DL mode */
    .rating         = 100,
    .irq            = -1,
};
```

**Calibration**: `calibrate_APIC_clock()` measures how many APIC timer ticks elapse across `LAPIC_CAL_LOOPS = HZ/10` jiffies and computes:

```c
lapic_timer_period = (delta * APIC_DIVISOR) / LAPIC_CAL_LOOPS;
```

This gives the APIC timer count needed for one jiffy (`1/HZ` seconds). The clockevent multiplier is then computed via `div_sc()` to convert nanoseconds to APIC ticks.

### 1.3 Interrupt Handling Call Chain

The `event_handler` pointer in `lapic_events` is set by the clockevent framework based on the system's timer configuration. There are three distinct paths:

**Path 1 — Periodic tick (legacy / no dyntick)**

```
sysvec_apic_timer_interrupt
  → local_apic_timer_interrupt
    → evt->event_handler = tick_handle_periodic
      → tick_periodic(cpu)
        → do_timer(1)           [updates jiffies, wall clock]
        → update_process_times  [accounting, hrtimers, irq_work, scheduler]
          → account_process_tick
          → run_local_timers
            → hrtimer_run_queues   [hrtimer expiry processing]
            → raise_softirq(TIMER_SOFTIRQ) [if timer wheel has work]
          → rcu_sched_clock_irq
          → irq_work_tick         [drain lazy irq_work]
          → sched_tick            [= scheduler_tick]
          → run_posix_cpu_timers
```

**Path 2 — Dynamic tick / hrtimer mode (most modern x86)**

When `CONFIG_HIGH_RES_TIMERS=y` and the clockevent is programmed in one-shot mode, the hrtimer subsystem takes over event programming:

```
sysvec_apic_timer_interrupt
  → local_apic_timer_interrupt
    → evt->event_handler = hrtimer_interrupt
      → __hrtimer_run_queues (HRTIMER_ACTIVE_HARD)
        → [executes expired hard hrtimers, including:]
          → tick_sched_timer callback (the "tick emulation" hrtimer)
            → tick_sched_do_timer  [jiffies update]
            → tick_sched_handle
              → update_process_times  [same chain as above]
      → [if softirq hrtimers expired: raise_softirq(HRTIMER_SOFTIRQ)]
      → hrtimer_update_next_event
      → tick_program_event        [re-arm APIC timer for next expiry]
```

**Path 3 — nohz handler (used when tick is running in tickless/dyntick)**

The `tick_nohz_handler` is set as the hrtimer callback for the sched_timer:

```c
static enum hrtimer_restart tick_nohz_handler(struct hrtimer *timer)
{
    struct tick_sched *ts = container_of(timer, struct tick_sched, sched_timer);
    struct pt_regs *regs = get_irq_regs();
    ktime_t now = ktime_get();

    tick_sched_do_timer(ts, now);
    if (regs)
        tick_sched_handle(ts, regs);
    else
        ts->next_tick = 0;

    if (unlikely(tick_sched_flag_test(ts, TS_FLAG_STOPPED)))
        return HRTIMER_NORESTART;   /* tick stopped → don't re-arm */

    hrtimer_forward(timer, now, TICK_NSEC);
    return HRTIMER_RESTART;
}
```

The return of `HRTIMER_NORESTART` is the mechanism by which tick stoppage is implemented: the APIC timer simply does not get re-armed after the last tick fires.

### 1.4 hrtimers and the LOC Handler

The `hrtimer` subsystem provides nanosecond-resolution timers layered on top of whatever the underlying clock event device supports. On modern x86:

- `hrtimer_interrupt()` is called from the LAPIC LOC handler (via `evt->event_handler`).
- It processes all expired "hard" hrtimers (in `HRTIMER_ACTIVE_HARD` state) directly in interrupt context via `__hrtimer_run_queues()`.
- "Soft" hrtimers (those registered with `HRTIMER_MODE_SOFT`) are delegated to `HRTIMER_SOFTIRQ` via `raise_softirq_irqoff()` to avoid running expensive callbacks in hard IRQ context.
- After processing, it calls `hrtimer_update_next_event()` to find the next expiry, then `tick_program_event()` to re-arm the APIC timer.

The "tick emulation" hrtimer (`ts->sched_timer`) is the specific hrtimer that fires the scheduler tick in dyntick mode. In periodic mode, no such hrtimer is needed because the APIC fires at `HZ` unconditionally. In one-shot/dyntick mode, the sched_timer hrtimer fires to emulate the tick and is re-armed at `TICK_NSEC` intervals unless the tick is stopped.

The key insight: **every LOC interrupt is a clock event device interrupt, but not every LOC interrupt results in a full scheduler tick**. In high-resolution timer mode, the APIC is re-armed for whatever the nearest hrtimer expiry is. If an hrtimer fires in 50 µs, the APIC fires in 50 µs — the LOC counter increments, but the scheduler tick hrtimer may not have expired. LOC counts hrtimer wakeups, not just HZ-rate scheduler ticks.

### 1.5 CONFIG_HZ and the Scheduler Tick

`CONFIG_HZ` (typically 250 or 1000 on servers, 100–300 on desktops) defines:

- `HZ`: interrupts per second for the periodic tick
- `TICK_NSEC = 1e9 / HZ`: nanoseconds between scheduler ticks
- `LAPIC_CAL_LOOPS = HZ/10`: calibration duration in jiffies

In periodic mode, the APIC fires exactly `HZ` times per second, and each interrupt runs `update_process_times()` → `sched_tick()` (internal name: `scheduler_tick()`).

`scheduler_tick()` performs:
- `update_rq_clock(rq)` — updates the runqueue clock
- `curr->sched_class->task_tick(rq, curr, 0)` — per-scheduling-class tick handler (e.g., CFS accounts vruntime, RT checks time slice)
- `trigger_load_balance(rq)` — schedules SCHED_SOFTIRQ for load balancing
- `perf_event_task_tick()` — notifies perf of a scheduler tick
- `calc_global_load_tick()` — contributes to global load average

In `CONFIG_NO_HZ_*` kernels, the APIC is in one-shot mode. The scheduler tick is emulated by an hrtimer that fires at `TICK_NSEC` intervals only when the tick is running. When the tick is stopped (idle or nohz_full single-task), no scheduler tick hrtimer fires.

### 1.6 NO_HZ_IDLE: Suppressing LOC on Idle CPUs

`CONFIG_NO_HZ_IDLE` (formerly `CONFIG_NO_HZ`) suppresses the periodic tick on CPUs that are in the idle loop. This is the baseline "dyntick idle" mode, enabled by default since Linux 2.6.21.

When the idle task runs:
1. `tick_nohz_idle_enter()` is called from the idle path (`do_idle()` in `kernel/sched/idle.c`).
2. `tick_nohz_idle_stop_tick()` evaluates whether the tick can be stopped via `can_stop_idle_tick()`.
3. If stoppable, `tick_nohz_stop_tick()` cancels the sched_timer hrtimer and records `TS_FLAG_STOPPED`.

`can_stop_idle_tick()` returns false (tick must keep running) if:
- `need_resched()` is set
- Softirq work is pending
- The CPU is the designated `do_timer_cpu` (timekeeping duty, selected by `tick_do_timer_cpu`) — unless another CPU can take over
- `rcu_needs_cpu()` — RCU has callbacks pending
- `irq_work_needs_cpu()` — lazy irq_work is pending
- `arch_needs_cpu()` — architecture-specific constraint
- `local_timer_softirq_pending()` — software timer softirq pending

When the CPU exits idle (interrupt, wake-up):
1. `tick_nohz_idle_restart_tick()` calls `tick_nohz_restart_sched_tick()`.
2. Jiffies are brought up to date via `tick_do_update_jiffies64(now)`.
3. `timer_clear_idle()` marks the timer base as non-idle.
4. The sched_timer hrtimer is re-armed.

Under `NO_HZ_IDLE`, idle CPUs generate zero LOC interrupts (the APIC timer is either disarmed or programmed to fire only at the next timer event, which may be much later than `TICK_NSEC`).

### 1.7 NO_HZ_FULL: Suppressing LOC on Single-Task CPUs

`CONFIG_NO_HZ_FULL` extends tick suppression to CPUs running a single userspace task — even non-idle CPUs. The set of "adaptive tick" CPUs is specified at boot via `nohz_full=<cpulist>` (the boot CPU is excluded automatically).

The key architectural requirement: at least one non-nohz_full CPU must remain to handle timekeeping (jiffies updates, global load). The `do_timer_cpu` is never a nohz_full CPU.

Tick evaluation for nohz_full happens at every kernel exit to userspace:
1. `tick_nohz_irq_exit()` → `__tick_nohz_full_update_tick()`:

```c
static void __tick_nohz_full_update_tick(struct tick_sched *ts, ktime_t now)
{
    if (can_stop_full_tick(cpu, ts))
        tick_nohz_full_stop_tick(ts, cpu);
    else if (tick_sched_flag_test(ts, TS_FLAG_STOPPED))
        tick_nohz_restart_sched_tick(ts, now);
}
```

2. `can_stop_full_tick()` is the gating function:

```c
static bool can_stop_full_tick(int cpu, struct tick_sched *ts)
{
    if (unlikely(!cpu_online(cpu)))
        return false;
    if (check_tick_dependency(&tick_dep_mask))        /* global deps */
        return false;
    if (check_tick_dependency(&ts->tick_dep_mask))    /* per-CPU deps */
        return false;
    if (check_tick_dependency(&current->tick_dep_mask)) /* per-task deps */
        return false;
    if (check_tick_dependency(&current->signal->tick_dep_mask)) /* per-process deps */
        return false;
    return true;
}
```

If `can_stop_full_tick()` returns true, the sched_timer hrtimer is cancelled and `TS_FLAG_STOPPED` is set. No further LOC interrupts will be generated on that CPU until a dependency is set.

Additionally, the scheduler's `sched_can_stop_tick()` is consulted. It returns false (tick required) if:
- `rq->dl.dl_nr_running > 0` — deadline tasks always need the tick for replenishment
- `rq->rt.rr_nr_running > 1` — multiple round-robin RT tasks need preemption
- `rq->cfs.nr_running > 1` — multiple CFS tasks need involuntary preemption
- A CFS task has CPU bandwidth constraints (`cfs_task_bw_constrained()`)
- SCX (sched_ext) disallows tick stopping

The `TICK_DEP_BIT_SCHED` dependency bit is set when `sched_can_stop_tick()` returns false. This feeds into `can_stop_full_tick()` via `check_tick_dependency()`.

### 1.8 The Residual ~1 Hz Tick

Despite the name "full tickless," nohz_full does not achieve zero interrupts. There is a **residual tick at approximately once per second**. This is a deliberate design choice rooted in correctness requirements, not a bug.

The mechanism that enforces this residual tick lives in `tick_sched_do_timer()`:

```c
if (ts->last_tick_jiffies != jiffies) {
    ts->stalled_jiffies = 0;
    ts->last_tick_jiffies = READ_ONCE(jiffies);
} else {
    if (++ts->stalled_jiffies == MAX_STALLED_JIFFIES) {
        tick_do_update_jiffies64(now);
        ts->stalled_jiffies = 0;
        ts->last_tick_jiffies = READ_ONCE(jiffies);
    }
}
```

Where `MAX_STALLED_JIFFIES = 5`. When jiffies have not advanced for 5 consecutive ticks on a given CPU, the kernel forces a `tick_do_update_jiffies64()` call. Since the `do_timer_cpu` (the CPU responsible for advancing jiffies globally) is not a nohz_full CPU, jiffies on nohz_full CPUs can fall behind. The stall detection forces a corrective update approximately once per second when nohz_full CPUs take their rare housekeeping tick.

**Why does the 1 Hz tick still fire?**

Even with all tick dependencies cleared, the kernel needs this interval for:

1. **Jiffies accounting and CPU usage statistics**: `update_process_times()` updates `p->stime`/`p->utime` via `account_process_tick()`. Without any tick, the process would not accumulate CPU time in kernel accounting, breaking `getrusage()`, `/proc/PID/stat`, and `top`.

2. **Scheduler load averages**: `calc_global_load_tick()` contributes the CPU's load to the exponential moving average used for `/proc/loadavg`. This requires periodic sampling.

3. **RCU quiescent states**: Even with `CONFIG_RCU_NOCB_CPU`, the RCU subsystem periodically needs to confirm quiescent states on user-mode CPUs to advance grace periods.

4. **Watchdog refresh**: `touch_softlockup_watchdog_sched()` is called in `tick_nohz_restart_sched_tick()` to prevent false soft-lockup reports.

5. **vmstat**: Memory management statistics are updated via a deferred work mechanism that fires approximately once per second.

The fundamental tension is that the kernel was designed around the assumption that CPUs receive periodic ticks. Removing the tick entirely requires auditing and patching every subsystem that implicitly relies on it — an ongoing, incomplete process.

### 1.9 Tick Dependency Tracking

Tick dependencies are tracked via a multi-level bitmask system defined in `include/linux/tick.h`:

```c
enum tick_dep_bits {
    TICK_DEP_BIT_POSIX_TIMER    = 0,
    TICK_DEP_BIT_PERF_EVENTS    = 1,
    TICK_DEP_BIT_SCHED          = 2,
    TICK_DEP_BIT_CLOCK_UNSTABLE = 3,
    TICK_DEP_BIT_RCU            = 4,
    TICK_DEP_BIT_RCU_EXP        = 5
};
```

Four dependency scopes, checked in `can_stop_full_tick()`:

| Scope | Variable | Set/cleared by |
|---|---|---|
| Global | `tick_dep_mask` | `tick_nohz_dep_set()` / `tick_nohz_dep_clear()` |
| Per-CPU | `ts->tick_dep_mask` | `tick_nohz_dep_set_cpu()` / `tick_nohz_dep_clear_cpu()` |
| Per-task | `current->tick_dep_mask` | `tick_nohz_dep_set_task()` / `tick_nohz_dep_clear_task()` |
| Per-process | `current->signal->tick_dep_mask` | `tick_nohz_dep_set_signal()` / `tick_nohz_dep_clear_signal()` |

The per-nohz_full-CPU kick mechanism uses an `irq_work` to force re-evaluation without requiring an existing tick:

```c
static DEFINE_PER_CPU(struct irq_work, nohz_full_kick_work) =
    IRQ_WORK_INIT_HARD(nohz_full_kick_func);
```

`tick_nohz_full_kick()` queues this work locally; `tick_nohz_full_kick_cpu(cpu)` sends an IPI to queue it on a remote CPU. Because it is `IRQ_WORK_HARD_IRQ`, it bypasses the lazy path and raises a self-IPI immediately, forcing the IWI handler to call `irq_work_run()` which runs `nohz_full_kick_func()`, which calls `tick_nohz_full_update_tick()` to re-evaluate tick necessity.

### 1.10 Conditions That Force Tick Re-enablement

A nohz_full CPU that had its tick stopped will have its tick re-enabled when any of the following `TICK_DEP_BIT_*` conditions arise:

**TICK_DEP_BIT_SCHED (bit 2)**
- More than one CFS task becomes runnable on the CPU (`rq->cfs.nr_running > 1`)
- A deadline task is enqueued
- `fork()` creates a second runnable task
- `sched_setaffinity()` migrates a task to the CPU
- `exec()` with a new process image (which briefly has the old task runnable)

**TICK_DEP_BIT_POSIX_TIMER (bit 0)**
Set in `arm_timer()` (`kernel/time/posix-cpu-timers.c`) when a POSIX CPU timer is armed:
```c
if (CPUCLOCK_PERTHREAD(timer->it_clock))
    tick_dep_set_task(p, TICK_DEP_BIT_POSIX_TIMER);
else
    tick_dep_set_signal(p, TICK_DEP_BIT_POSIX_TIMER);
```
Cleared when the timer is disarmed or expires.

**TICK_DEP_BIT_PERF_EVENTS (bit 1)**
Set when perf events exceed the hardware PMU counter capacity and require software multiplexing. Software-mode perf events always require the tick for sampling.

**TICK_DEP_BIT_CLOCK_UNSTABLE (bit 3)**
Set globally by `clocksource_verify_choose_delay()` or `mark_tsc_unstable()` when the TSC is found unreliable. Affects all nohz_full CPUs.

**TICK_DEP_BIT_RCU (bit 4)**
Set by RCU in `__rcu_irq_enter_check_tick()` when a nohz_full CPU has been in kernel mode long enough to be suspected of blocking an RCU grace period:
```c
WRITE_ONCE(rdp->rcu_forced_tick, true);
tick_dep_set_cpu(rdp->cpu, TICK_DEP_BIT_RCU);
```
Cleared by `rcu_disable_urgency_upon_qs()` after the CPU reports a quiescent state.

**TICK_DEP_BIT_RCU_EXP (bit 5)**
Set during expedited RCU grace periods, which require faster quiescent state reporting.

**Kernel thread wakeup on an isolated CPU**: If any non-idle kernel thread is scheduled on a nohz_full CPU (migration thread, kworker, etc.), `rq->cfs.nr_running` becomes > 1 (current userspace task + kernel thread), immediately triggering `TICK_DEP_BIT_SCHED` and re-enabling the tick.

**perf stat / perf record on the profiled CPU**: When `perf stat` attaches to a process on a nohz_full CPU, the perf event counts as an active perf event requiring the tick. See §1.5 for the perf dependency path.

### 1.11 tick_nohz_stop_sched_tick and tick_nohz_restart_sched_tick

**Stopping the tick** (`tick_nohz_stop_tick()` / `tick_nohz_full_stop_tick()`):

1. Evaluates the next timer event via `tick_nohz_next_event()`. This checks `rcu_needs_cpu()`, `arch_needs_cpu()`, `irq_work_needs_cpu()`, `local_timer_softirq_pending()` — if any return true, the tick must fire at the next jiffy boundary.
2. Calls `hrtimer_cancel(&ts->sched_timer)` to stop the scheduler tick hrtimer.
3. If there is a future timer event (not `KTIME_MAX`), reprograms the APIC via `tick_program_event()` to fire at that future time. If `KTIME_MAX`, the APIC timer is disarmed completely.
4. Sets `TS_FLAG_STOPPED` in `ts->flags`.

**Restarting the tick** (`tick_nohz_restart_sched_tick()`):

```c
static void tick_nohz_restart_sched_tick(struct tick_sched *ts, ktime_t now)
{
    tick_do_update_jiffies64(now);
    timer_clear_idle();
    calc_load_nohz_stop();
    touch_softlockup_watchdog_sched();
    tick_sched_flag_clear(ts, TS_FLAG_STOPPED);
    tick_nohz_restart(ts, now);
}
```

`tick_nohz_restart()` calls `hrtimer_start()` to re-arm the sched_timer at the next `TICK_NSEC` boundary from `now`. `tick_do_update_jiffies64(now)` catches up any missed jiffies from the tickless period, updating wall time, `xtime`, and load averages.

The historical function name `tick_nohz_stop_sched_tick()` appeared in older kernels (pre-4.x). In v6.12, the equivalent is `tick_nohz_idle_stop_tick()` for idle and `tick_nohz_full_stop_tick()` for nohz_full. The `tick_nohz_restart_sched_tick()` name is preserved.

### 1.12 TSC-Deadline Mode vs APIC Countdown Mode

**APIC countdown (one-shot) mode**:
- Programs `APIC_TMICT` with a bus-clock-derived count
- Accuracy limited by: bus clock stability, calibration precision, PICLK jitter
- Minimum reprogramming latency: several bus cycles for the write + APIC pipeline
- Subject to C-state timer freezing (`CLOCK_EVT_FEAT_C3STOP` flag) — deep sleep states may stop the APIC timer
- Rating: `lapic_clockevent.rating = 100`

**TSC-deadline mode** (`X86_FEATURE_TSC_DEADLINE_TIMER`):
- Programs `MSR_IA32_TSC_DEADLINE` with an absolute TSC value
- Accuracy tied to TSC frequency, which is invariant on modern Intel/AMD CPUs (`X86_FEATURE_CONSTANT_TSC`, `X86_FEATURE_NONSTOP_TSC`)
- The TSC does not stop in C-states when `NONSTOP_TSC` is set, so the timer fires correctly after deep sleep
- Nanosecond-level accuracy vs microsecond-level for bus-based countdown
- `lapic_next_deadline()` is registered as `set_next_event` instead of `lapic_next_event()`
- Eliminates one source of timer interrupt rate increase: bus-clock rounding means the countdown mode may fire slightly early and need re-arming more often

For nohz_full / HPC workloads, TSC-deadline mode is significantly better:
1. Fewer spurious wakeups (no early-fire due to bus-clock quantization)
2. Works correctly through C-states (no timer lost in C3 when `NONSTOP_TSC`)
3. Higher clock rating causes the kernel to prefer it over other clockevent sources

Check availability: `cat /sys/devices/system/clockevents/clockevent*/current_device` — look for "lapic-deadline". Or `grep -c tsc_deadline /proc/cpuinfo` on Intel.

### 1.13 The task_isolation Patchset

The `task_isolation` patchset (proposed by Chris Metcalf for Tilera/Mellanox systems, not merged into mainline as of kernel 6.x) aims to eliminate the residual ~1 Hz tick that persists even under `nohz_full`.

The patchset adds a `prctl(PR_SET_TASK_ISOLATION, flags, ...)` API allowing a task to declare itself fully isolated:

- `PR_TASK_ISOLATION_ENABLE`: marks the task as wanting isolation; the kernel ensures the tick is stopped when returning to user mode
- `PR_TASK_ISOLATION_STRICT`: terminates the task (SIGKILL) if any unexpected kernel entry occurs while isolated

**The busy-wait approach**: Rather than stopping the tick and hoping no timer fires, `task_isolation` adds a spin-wait loop in the return-to-userspace path (`kernel/entry/common.c`) that busy-waits until all near-term timers have expired. This is controversial because a misplaced one-year timer would spin for a year.

**Subsystems that need explicit fix-ups** before `task_isolation` can guarantee zero ticks:
- `vmstat`: the deferred worker is disabled on isolated CPUs
- `lru_add_drain()`: CPU-local page cache drain is triggered before isolation entry
- POSIX CPU timers: must not be armed on isolated CPUs
- RCU: requires `CONFIG_RCU_NOCB_CPU=y` and the CPU in the nocb set

**Why it was not merged**: Thomas Gleixner and others argued the correct solution is to surgically eliminate each source of unexpected kernel entry, rather than imposing a post-hoc enforcement mechanism. The patchset requires touching too many subsystems and the busy-wait approach is untenable. Work continues incrementally (each kernel release addresses one or two timer sources).

**Current state (6.x)**: The `nohz_full` infrastructure is the closest mainline approximation. For production HPC/RT isolation, `isolcpus=domain,managed_irq nohz_full=<cpus> rcu_nocbs=<cpus>` combined with `tuna` or `cset` for process isolation achieves near-isolation with the unavoidable ~1 Hz residual.

---

## 2. IWI — IRQ Work Interrupt

### 2.1 The irq_work Subsystem: Problem Statement

Several kernel subsystems need to perform work that requires:
1. Running in a well-defined interrupt context (not NMI, not softirq)
2. Being schedulable from NMI context (which cannot call `local_bh_disable()`, take spinlocks with BH, or raise softirqs safely)
3. Being schedulable from hard IRQ context without re-entering a higher-level handler

The classic use case is `perf_events`: the PMU overflow handler fires as an NMI. The NMI needs to wake up a userspace reader of the ring buffer. But `wake_up()` takes a spinlock — impermissible in NMI context. The solution: queue an `irq_work`, which delivers a self-IPI, which fires as a regular hardware interrupt (not NMI), where `wake_up()` is safe.

The `irq_work` subsystem (`kernel/irq_work.c`, `include/linux/irq_work.h`) provides this mechanism: **a way to run a function in hard-IRQ context from NMI or hard-IRQ context, using a per-CPU self-IPI**.

The IWI entry in `/proc/interrupts` counts these self-IPI deliveries per CPU.

### 2.2 struct irq_work and State Machine

```c
// include/linux/irq_work.h
struct irq_work {
    struct __call_single_node node;  /* llist node + flags */
    void (*func)(struct irq_work *);
    struct rcuwait irqwait;
};
```

The flags are stored in the low bits of `node.a_flags` (an atomic). The state machine has four states:

```
free (0)
  │  irq_work_claim() sets CLAIMED | CSD_TYPE_IRQ_WORK
  ↓
claimed (1)   [node acquired, not yet on list]
  │  added to per-CPU raised_list or lazy_list
  ↓
pending (3)   [on list, waiting to run]
  │  irq_work_run_list processes it
  ↓
busy (2)      [func() is executing]
  │  func() returns
  ↓
free (0)      [ready for reuse]
```

The state machine prevents double-queuing: if an `irq_work` is already pending/busy when `irq_work_queue()` is called again, the function returns false and the work is not re-queued.

### 2.3 IRQ_WORK_LAZY and IRQ_WORK_HARD_IRQ Flags

Three initialization helpers define the work type:

```c
IRQ_WORK_INIT(_func)       /* standard: goes to raised_list, self-IPI if tick stopped */
IRQ_WORK_INIT_LAZY(_func)  /* lazy: deferred to next tick, no self-IPI */
IRQ_WORK_INIT_HARD(_func)  /* hard: always raises self-IPI, even on PREEMPT_RT */
```

- **Standard work** (`raised_list`): causes a self-IPI unless already in IRQ context or the tick has just run
- **Lazy work** (`IRQ_WORK_LAZY` flag, `lazy_list`): deferred to the next timer tick. No self-IPI is generated. If the tick is stopped (nohz_full or idle), a self-IPI is generated anyway to avoid indefinite deferral.
- **Hard IRQ work** (`IRQ_WORK_HARD_IRQ` flag): always runs in hard-IRQ context via self-IPI. On `PREEMPT_RT` kernels, most work is soft-irq-deferred, but hard work bypasses that.

### 2.4 How IWI Is Raised: Self-IPI via IRQ_WORK_VECTOR

The x86 architecture-specific implementation (`arch/x86/kernel/irq_work.c`):

```c
#ifdef CONFIG_X86_LOCAL_APIC
void arch_irq_work_raise(void)
{
    __apic_send_IPI_self(IRQ_WORK_VECTOR);
    apic_wait_icr_idle();
}
#endif
```

`IRQ_WORK_VECTOR = 0xf6` (defined in `arch/x86/include/asm/irq_vectors.h`).

This sends an IPI to the CPU itself using the APIC's ICR (Interrupt Command Register). `apic_wait_icr_idle()` polls the ICR delivery-status bit until the IPI has been accepted by the local APIC, ensuring the interrupt is in-flight before the calling code returns.

In x2APIC mode (MSR-based APIC), `__apic_send_IPI_self()` uses `x2apic_send_IPI_self()` which writes `MSR_X2APIC_ICR` with the self-IPI shorthand, which is atomic and does not require the ICR idle wait.

The `SELF_IPI_VECTOR` (the value 0xf9 mentioned in older documentation and some Intel manuals) refers to a specific x2APIC self-IPI mechanism available in newer Intel CPUs. In Linux, `IRQ_WORK_VECTOR = 0xf6` is what the IWI handler uses, distinct from the `CALL_FUNCTION_SINGLE_VECTOR`.

### 2.5 The IWI Handler: sysvec_irq_work

```c
// arch/x86/kernel/irq_work.c
DEFINE_IDTENTRY_SYSVEC(sysvec_irq_work)
{
    apic_eoi();
    trace_irq_work_entry(IRQ_WORK_VECTOR);
    inc_irq_stat(apic_irq_work_irqs);   /* this is the IWI counter */
    irq_work_run();
    trace_irq_work_exit(IRQ_WORK_VECTOR);
}
```

`apic_irq_work_irqs` is the per-CPU counter shown as IWI in `/proc/interrupts`.

`irq_work_run()` calls `irq_work_run_list()` on both the `raised_list` and (on non-PREEMPT_RT) the `lazy_list`. Each work item's `func` callback is invoked sequentially.

### 2.6 Lazy vs Non-Lazy Work: The Self-IPI Suppression Logic

The decision of whether to raise a self-IPI or defer to the tick is made in `__irq_work_queue_local()`:

```c
static void __irq_work_queue_local(struct irq_work *work)
{
    struct llist_head *list;
    bool rt_lazy_work = false;
    bool lazy_work = false;
    int work_flags;

    work_flags = atomic_read(&work->node.a_flags);
    if (work_flags & IRQ_WORK_LAZY)
        lazy_work = true;
    else if (IS_ENABLED(CONFIG_PREEMPT_RT) &&
             !(work_flags & IRQ_WORK_HARD_IRQ))
        rt_lazy_work = true;

    if (lazy_work || rt_lazy_work)
        list = this_cpu_ptr(&lazy_list);
    else
        list = this_cpu_ptr(&raised_list);

    if (!llist_add(&work->node.llist, list))
        return;

    if (!lazy_work || tick_nohz_tick_stopped())
        irq_work_raise(work);
}
```

The critical line: `if (!lazy_work || tick_nohz_tick_stopped())` — lazy work is promoted to immediate self-IPI if the tick has been stopped. This prevents lazy work from being stranded indefinitely on a tickless CPU.

**Implications**:
- On a CPU with `HZ=1000` running in full periodic mode: lazy work waits up to 1 ms for the next tick. No self-IPI, no IWI count increment.
- On an idle CPU (NO_HZ_IDLE, tick stopped): lazy work immediately generates a self-IPI → IWI count increments.
- On a nohz_full CPU (tick stopped): lazy work generates a self-IPI → IWI count increments.
- On a busy periodic CPU: lazy work silently accumulates until `irq_work_tick()` drains it at the next tick.

**Performance implication**: On isolated nohz_full CPUs, queuing lazy irq_work (e.g., perf ring buffer wakeup when ring buffer is not full enough to trigger immediate wakeup) causes an IWI self-IPI. This is visible as a non-zero IWI count even on "isolated" CPUs.

### 2.7 irq_work_tick: Processing Lazy Work at Timer Tick

`irq_work_tick()` runs inside `update_process_times()` (conditionally, when in IRQ context):

```c
void irq_work_tick(void)
{
    struct llist_head *raised = this_cpu_ptr(&raised_list);

    if (!llist_empty(raised) && !arch_irq_work_has_interrupt())
        irq_work_run_list(raised);

    if (!IS_ENABLED(CONFIG_PREEMPT_RT))
        irq_work_run_list(this_cpu_ptr(&lazy_list));
    else
        wake_irq_workd();
}
```

On standard kernels: both lists are drained at each tick. On PREEMPT_RT: the lazy list is handled by the per-CPU `irq_workd` kthread to maintain deterministic latency.

The `arch_irq_work_has_interrupt()` check avoids running the raised list here if the architecture already has a dedicated interrupt handler (x86 does — the IWI vector). On architectures without a self-IPI capability, `irq_work_tick()` is the only drain path for both lists.

### 2.8 irq_work_needs_cpu: Preventing Tick Stop

`irq_work_needs_cpu()` is called from `tick_nohz_next_event()` to prevent the tick from being stopped when lazy work is pending:

```c
bool irq_work_needs_cpu(void)
{
    struct llist_head *raised, *lazy;

    raised = this_cpu_ptr(&raised_list);
    lazy = this_cpu_ptr(&lazy_list);

    if (llist_empty(raised) || arch_irq_work_has_interrupt())
        if (llist_empty(lazy))
            return false;

    WARN_ON_ONCE(cpu_is_offline(smp_processor_id()));
    return true;
}
```

The logic: if the raised list is non-empty AND the architecture has no interrupt-based drain (i.e., non-x86), or if the lazy list is non-empty, return true — tick must not stop.

On x86 (`arch_irq_work_has_interrupt()` returns true): the raised list check is bypassed (the IWI handler drains it), but the lazy list still gates tick stopping. This prevents lazy work from being stranded when transitioning to idle.

### 2.9 Subsystems That Use irq_work_queue

#### perf_events (primary IWI driver)

The perf event subsystem is the most significant source of IWI interrupts. `struct perf_event` contains:

```c
struct irq_work pending_irq;         /* ring buffer wakeup */
struct irq_work pending_disable_irq; /* event disable from NMI */
```

**Ring buffer wakeup path** (`kernel/events/ring_buffer.c`):

```c
static void perf_output_wakeup(struct perf_output_handle *handle)
{
    atomic_set(&handle->rb->poll, EPOLLIN);
    handle->event->pending_wakeup = 1;
    if (*perf_event_fasync(handle->event) && !handle->event->pending_kill)
        handle->event->pending_kill = POLL_IN;
    irq_work_queue(&handle->event->pending_irq);
}
```

`perf_output_wakeup()` is called from `perf_output_put_handle()` when the ring buffer reaches a watermark or from `perf_aux_output_begin()` on AUX buffer exhaustion. Since PMU overflow fires as NMI, and the NMI handler fills the ring buffer and then calls `perf_output_wakeup()`, the `irq_work_queue()` defers the `wake_up()` to a safe IRQ context.

The `pending_irq` function (set to `perf_pending_irq` at event init) ultimately calls:
1. `wake_up_all(&event->waitq)` — wakes readers of the mmap ring buffer
2. `kill_fasync()` — sends SIGIO if async notification is configured

**How `perf stat` drives IWI counts on profiled CPUs**: `perf stat` attaches software and hardware events. Even in counting mode (no ring buffer), `perf stat` causes:
- `TICK_DEP_BIT_PERF_EVENTS` to be set, re-enabling the tick on nohz_full CPUs
- On overflow (counter wrap), the NMI handler queues `pending_irq` → IWI
- Additionally, `perf_duration_work` (an irq_work warning about long-running perf callbacks) can fire

**`perf record` is far worse**: every sample writes to the ring buffer; when the buffer watermark is crossed (default: 1 wakeup per page), `perf_output_wakeup()` fires, queuing an irq_work per crossed watermark → potentially thousands of IWI interrupts per second on heavily sampled CPUs.

#### printk from NMI context (`kernel/printk/printk.c`)

```c
static DEFINE_IRQ_WORK(wake_up_klogd_work, wake_up_klogd_work_func);
```

`wake_up_klogd()` calls `irq_work_queue(&wake_up_klogd_work)` to defer the `wake_up_interruptible()` of the klogd waitqueue to IRQ context. This is NMI-safe because `irq_work_queue()` uses only atomic operations and llist manipulation.

Since `wake_up_klogd_work` is initialized as lazy (`IRQ_WORK_LAZY`), it will not generate an IWI self-IPI unless the tick is stopped.

#### RCU (`kernel/rcu/tree.c`)

RCU uses `irq_work` to force quiescent state detection on nohz_full CPUs:

```c
if (IS_ENABLED(CONFIG_IRQ_WORK) &&
    !rdp->rcu_iw_pending && rdp->rcu_iw_gp_seq != rnp->gp_seq &&
    (rnp->ffmask & rdp->grpmask)) {
    rdp->rcu_iw_pending = true;
    rdp->rcu_iw_gp_seq = rnp->gp_seq;
    irq_work_queue_on(&rdp->rcu_iw, rdp->cpu);
}
```

`irq_work_queue_on()` sends an IPI to the target CPU to force it to run `rcu_iw` callback, which reports a quiescent state. On a nohz_full CPU with a stopped tick, this generates a remote self-IPI → IWI increment on the target CPU.

#### Scheduler (`kernel/sched/`)

The `nohz_full_kick_work` per-CPU irq_work (see §1.9) is queued when tick dependencies change. On the local CPU, `tick_nohz_full_kick()` queues it directly; on remote CPUs, `tick_nohz_full_kick_cpu()` sends it via `irq_work_queue_on()`. Both paths increment IWI on the target CPU.

`sched_ttwu_pending()` (task wake-up pending) is delivered via `call_single_data`, not irq_work directly, but uses the same IPI vector infrastructure.

#### Other users

- `ftrace`: function tracer uses `irq_work` to flush trace buffers from NMI context
- `kprobes`: some kprobe handlers use `irq_work` to defer processing
- `watchdog`: hardlockup detector uses `irq_work` for NMI-safe reporting
- `thermal`: Intel RAPL and thermal monitoring use `irq_work` for threshold violations reported via NMI

### 2.10 Lazy irq_work on nohz_full CPUs: The Deferred Indefinitely Problem

On a nohz_full CPU with the tick stopped, lazy `irq_work` items queued by other CPUs (or by the CPU itself during a kernel entry) will not be processed until either:
1. The tick restarts (because a dependency was set), or
2. `irq_work_needs_cpu()` returns true and prevents the tick from stopping, or
3. The lazy work detects `tick_nohz_tick_stopped()` and promotes itself to immediate self-IPI

The kernel handles this via the `tick_nohz_tick_stopped()` check inside `__irq_work_queue_local()`:

```c
if (!lazy_work || tick_nohz_tick_stopped())
    irq_work_raise(work);
```

If a CPU has been in nohz_full with tick stopped, and then new lazy work is queued (e.g., from `wake_up_klogd()`), `tick_nohz_tick_stopped()` returns true, so `irq_work_raise()` is called immediately, generating a self-IPI. This causes an IWI interrupt, which runs `irq_work_run()`, which drains both lists.

**The remote queue case**: `irq_work_queue_on(work, cpu)` queues work on a remote CPU. If that CPU has its tick stopped, the local queueing still happens atomically, but the remote CPU must receive an IPI to process it. `irq_work_queue_on()` internally checks the remote CPU's nohz state and sends a dedicated IPI if needed. This generates an IWI on the remote CPU.

**PREEMPT_RT deferral**: On `CONFIG_PREEMPT_RT`, lazy work goes to `lazy_list` and is processed by the `irq_workd/$cpu` kthread rather than in interrupt context. This kthread runs at a specific priority and can be preempted, providing better latency guarantees. On nohz_full RT kernels, the interaction between tick suppression and `irq_workd` scheduling requires careful configuration.

### 2.11 IWI Counts in /proc/interrupts: What Drives Them

Reading `/proc/interrupts` for the IWI row:

```
           CPU0   CPU1   CPU2   CPU3
IWI:          5    128      0    892
```

**Interpretation**:
- `CPU0 = 5`: minimal activity, perhaps a few perf events or printk from NMI at boot
- `CPU1 = 128`: moderate activity — some perf monitoring, RCU kicks, scheduler wakeups
- `CPU2 = 0`: completely isolated CPU — no perf, no NMI printk, no RCU forced kicks, no scheduler tick dependency changes
- `CPU3 = 892`: actively monitored by `perf record` or has high-rate PMU overflow → each ring buffer wakeup = one IWI

**Factors that increase IWI on a CPU**:
1. `perf record` running with high sample rate (−F option) → wakeup per ring buffer watermark
2. `perf stat` with software events that overflow frequently
3. NMI-context printk (hardware errors, MCE handling)
4. RCU forcing quiescent states on nohz_full CPUs
5. `tick_nohz_full_kick_cpu()` from other CPUs when dependencies change
6. Lazy irq_work promotion on tickless CPUs

**Factors that keep IWI at zero**:
1. CPU is isolated (`isolcpus=`, `cpuset`) with no attached perf events
2. No RCU callbacks pending (`rcu_nocbs=` helps)
3. No NMI-context logging
4. No tick dependency changes being sent from other CPUs
5. No lazy irq_work being queued while tick is stopped

**The perf-stat-on-isolated-CPU problem**: Running `perf stat -C <isolated_cpu> -- sleep 10` will:
1. Set `TICK_DEP_BIT_PERF_EVENTS` on that CPU → tick re-enables
2. Each PMU counter overflow fires NMI → `perf_output_sample()` → `perf_output_wakeup()` → `irq_work_queue(&event->pending_irq)` → IWI self-IPI
3. Even in count-only mode (no sampling), some software events use irq_work for notification

Result: IWI goes from 0 to potentially hundreds per second on the "isolated" CPU.

---

## 3. Cross-Cutting Interactions

### LOC and IWI Are Not Independent

The tick (LOC) and irq_work (IWI) subsystems are deeply coupled:

1. **irq_work_tick()** runs inside `update_process_times()`, which is called from the LOC handler via `tick_sched_handle()`. Every tick drains the lazy irq_work list. Therefore: more LOC → more frequent lazy irq_work drain → fewer IWI self-IPIs needed.

2. **irq_work_needs_cpu()** prevents tick suppression when lazy irq_work is pending. A CPU with pending lazy work cannot stop its tick → LOC continues.

3. **tick_nohz_full_kick_work** is an `IRQ_WORK_HARD_IRQ` irq_work. When queued, it generates an IWI. The IWI handler runs `nohz_full_kick_func()` which re-evaluates tick necessity. If the tick re-enables, LOC resumes. IWI caused the LOC re-enablement.

4. **Cascading on perf**: perf sets `TICK_DEP_BIT_PERF_EVENTS` → `tick_nohz_full_kick_cpu()` sends IWI to the target CPU → IWI handler re-enables tick (LOC) → each LOC fires `perf_event_task_tick()` → potential further perf callbacks.

### nohz_full Steady State on an Ideal Isolated CPU

On a perfectly isolated nohz_full CPU running a single compute-bound userspace process with no perf, no POSIX timers, `rcu_nocbs=` configured:

- LOC rate: ~1/second (the residual jiffies-stall tick)
- IWI rate: 0 (no subsystem is queuing work)
- `perf stat -C <cpu>`: shows `apic_timer_irqs ≈ 1/s` and `apic_irq_work_irqs = 0`

Any deviation from zero IWI indicates a subsystem is queueing work on that CPU. The IWI count is a sensitive indicator of isolation quality.

---

## 4. Observability and Diagnosis

### /proc/interrupts

```bash
watch -n1 'grep -E "^LOC:|^IWI:" /proc/interrupts'
```

Rates (delta per second) are more useful than totals. Use:

```bash
awk 'BEGIN{cmd="cat /proc/interrupts"; while(1){
    while((cmd | getline line)>0) { ... }; close(cmd); sleep(1) }}'
```

Or `perf stat -e irq_vectors:local_timer_entry,irq_vectors:irq_work_entry -C <cpu> -I 1000`.

### perf tracepoints

```bash
# Count LOC and IWI per CPU per second
perf stat -e irq_vectors:local_timer_entry,irq_vectors:irq_work_entry \
          -C 3 -I 1000 -- sleep 10

# Identify which irq_work functions fire
perf record -e irq_work:irq_work_entry -C 3 -- sleep 5
perf report
```

### ftrace

```bash
# Trace IWI firing with function caller
echo 'irq_work_run' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace_pipe
```

### tick dependency inspection

```bash
# Show tick_dep_mask for a process (requires kernel debug patch or BPF)
bpftrace -e 'kprobe:can_stop_full_tick { printf("cpu=%d dep=%x\n",
    cpu, ((struct tick_sched *)arg1)->tick_dep_mask.bits[0]); }'
```

### Confirming TSC-deadline mode

```bash
cat /sys/devices/system/clockevents/clockevent0/current_device
# Output: lapic-deadline  (TSC-DL) or lapic (countdown mode)
```

---

## References

- `arch/x86/kernel/apic/apic.c` — LAPIC timer implementation
- `arch/x86/kernel/irq_work.c` — x86 arch_irq_work_raise, sysvec_irq_work
- `arch/x86/include/asm/irq_vectors.h` — `LOCAL_TIMER_VECTOR = 0xec`, `IRQ_WORK_VECTOR = 0xf6`
- `kernel/irq_work.c` — irq_work_queue, irq_work_run, irq_work_tick, irq_work_needs_cpu
- `include/linux/irq_work.h` — struct irq_work, IRQ_WORK_LAZY, IRQ_WORK_HARD_IRQ
- `kernel/time/tick-sched.c` — tick_nohz_*, can_stop_full_tick, tick_sched_do_timer, TICK_DEP_BIT_*
- `include/linux/tick.h` — enum tick_dep_bits, TICK_DEP_MASK_*
- `kernel/time/tick-common.c` — tick_handle_periodic, tick_periodic
- `kernel/time/hrtimer.c` — hrtimer_interrupt, __hrtimer_run_queues
- `kernel/time/timer.c` — run_local_timers, update_process_times
- `kernel/events/ring_buffer.c` — perf_output_wakeup
- `include/linux/perf_event.h` — struct perf_event.pending_irq
- `kernel/time/posix-cpu-timers.c` — arm_timer, TICK_DEP_BIT_POSIX_TIMER
- `kernel/rcu/tree.c` — __rcu_irq_enter_check_tick, TICK_DEP_BIT_RCU
- `kernel/sched/core.c` — sched_can_stop_tick
- `Documentation/timers/no_hz.rst` — NO_HZ_FULL operational requirements
