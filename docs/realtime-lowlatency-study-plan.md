# Real-Time / Low-Latency Linux Kernel Study Plan

Goal: understand the kernel internals behind each source of latency jitter on a
Linux system, how to measure it, and how the kernel can be configured or patched
to eliminate it.

The quote that motivates this plan names the real culprits:
NMIs, SMIs, stray IRQs, RCU housekeeping, page faults, C-states, and "system
software doing things you didn't ask for."  The plan is structured around those
culprits, bottom-up, so every concept builds on the previous one.

---

## Phase 0 — Foundations (do this first)

Before diving into RT-specific material, make sure you have a clear mental model
of how the kernel handles execution flow and time.

### 0.1 Execution contexts

The kernel runs in several distinct contexts, and understanding which context
can be preempted by which other context is the basis of everything that follows.

| Context | Can preempt | Can be preempted by |
|---------|-------------|---------------------|
| Hard IRQ | process, softirq | NMI (always), higher-priority IRQ |
| Soft IRQ / tasklet | process | hard IRQ, NMI |
| Process (kernel) | nothing (in non-RT) | IRQ, NMI, higher-priority process |
| NMI | everything | nothing (almost) |

Key files:
- `kernel/softirq.c` — softirq raise/dispatch, `ksoftirqd` thread
- `kernel/irq/handle.c` — `handle_irq_event()`, flow through `irq_desc`
- `arch/x86/kernel/irq.c` — x86 hard IRQ entry path
- `arch/x86/kernel/nmi.c` — `do_nmi()`, registered NMI handlers

### 0.2 Timer infrastructure

Timer resolution and tick behaviour are central to latency.

| Component | File | Purpose |
|-----------|------|---------|
| hrtimers | `kernel/time/hrtimer.c` | nanosecond-resolution one-shot timers |
| tick device | `kernel/time/tick-common.c` | per-CPU clock event device |
| NO_HZ_FULL | `kernel/time/tick-sched.c` | suppress periodic tick on idle/isolated CPUs |
| timekeeping | `kernel/time/timekeeping.c` | wall clock, CLOCK_MONOTONIC |

Read: `Documentation/timers/hrtimers.rst`, `Documentation/timers/no_hz.rst`

---

## Phase 1 — NMIs (Non-Maskable Interrupts)

NMIs cannot be masked by `cli`/`sti`.  They arrive at any point, including in
the middle of a lock-protected critical section.  On RT systems they are a
primary source of unbounded latency.

### 1.1 What generates NMIs?

| Source | Kernel hook | How to disable |
|--------|------------|----------------|
| NMI watchdog (perf-based) | `arch/x86/kernel/apic/hw_nmi.c` | `nmi_watchdog=0` on cmdline |
| Performance counters (PMU) | `arch/x86/events/core.c` | `perf_event_disable_all()` or avoid perf |
| PCIe AER / SERR | `drivers/pci/pcie/aer.c` | `pci=noaer` cmdline, or BIOS AER disable |
| MCE (machine-check) | `arch/x86/kernel/cpu/mce/core.c` | cannot fully disable; reduce polling |
| IPMI/BMC watchdog | `drivers/char/ipmi/` | disable in BIOS or `ipmi_watchdog` module |

### 1.2 NMI dispatch in the kernel

Read `arch/x86/kernel/nmi.c` carefully:

```
do_nmi()
  → nmi_handle()         # walks registered handlers via nmi_desc[]
  → unknown_nmi_error()  # fallback if nothing claimed it
```

Handlers register via `register_nmi_handler()` (in `include/linux/nmi.h`).
Each handler returns `NMI_HANDLED` or `NMI_DONE`.

### 1.3 Measuring NMI latency

Tool: `hwlatdetect` (part of `rt-tests` package) — polls in a tight loop and
measures gaps that can only be explained by NMIs or SMIs.

```bash
hwlatdetect --threshold=10 --duration=60   # report gaps > 10 µs for 60 s
```

Also: `rtla hwnoise` — kernel-side equivalent, uses `osnoise` tracer with
hardware noise isolation. See `Documentation/tools/rtla/rtla-hwnoise.rst`.

---

## Phase 2 — SMIs (System Management Interrupts)

SMIs are even worse than NMIs: they are invisible to the OS.  The CPU switches
to System Management Mode (SMM), runs firmware code, and returns.  The OS sees
only a gap in time.

### 2.1 Sources

- BIOS thermal management
- Corrected memory errors (patrol scrubbing)
- Legacy USB emulation
- ACPI embedded controller events
- Platform security features (Intel TXT, TPM)

### 2.2 Detection

Because SMIs are invisible, detection is indirect:

```bash
# hwlatdetect catches both NMIs and SMIs as unexplained gaps
hwlatdetect --threshold=5

# Intel-specific: read MSR 0x34 (SMI counter)
rdmsr -p <cpu> 0x34    # increment = one SMI occurred
```

There is no kernel source you can read to understand SMIs — they live entirely
in firmware.  The relevant kernel interfaces are:

- `arch/x86/kernel/cpu/mce/` — MCE handler (some SMI triggers overlap)
- `drivers/acpi/ec.c` — ACPI embedded controller, major SMI source
- `drivers/platform/x86/` — many platform drivers trigger SMIs

### 2.3 Mitigation

- Disable legacy USB emulation in BIOS (biggest SMI source on servers)
- Disable ACPI-based thermal management; use passive cooling or fixed fan curves
- Disable patrol scrub / memory training in BIOS
- On Intel: `intel_idle.max_cstate=1` reduces C-state SMI activity

---

## Phase 3 — CPU Isolation and IRQ Affinity

The goal is a "tickless isolated" CPU that runs only your RT task.

### 3.1 Kernel cmdline parameters

```
isolcpus=domain,managed_irq,4-7   # isolate CPUs 4-7 from the scheduler
nohz_full=4-7                     # suppress scheduling clock tick
rcu_nocbs=4-7                     # move RCU callbacks off isolated CPUs
irqaffinity=0-3                   # default IRQ affinity (keeps IRQs off 4-7)
```

Read `Documentation/admin-guide/kernel-parameters.txt` entries for each of
these.  They interact in non-obvious ways.

### 3.2 Isolation implementation

Key source file: `kernel/sched/isolation.c`

```c
// housekeeping_cpumask() returns which CPUs do "housekeeping"
// isolated CPUs are excluded from:
//   - load balancer
//   - RCU callbacks (with rcu_nocbs)
//   - tick (with nohz_full)
//   - workqueue (with WQ_UNBOUND + housekeeping affinity)
```

Follow the call graph:
1. `housekeeping_init()` — parses `isolcpus=`, builds masks
2. `sched_init_granularity()` in `kernel/sched/fair.c` — load balancer skips isolated CPUs
3. `tick_nohz_full_setup()` in `kernel/time/tick-sched.c` — disables periodic tick

### 3.3 IRQ affinity

After `isolcpus`, verify no IRQs land on isolated CPUs:

```bash
# Check all IRQ affinities
for i in /proc/irq/*/smp_affinity_list; do echo "$i: $(cat $i)"; done

# Move a specific IRQ off CPU 4
echo 0-3 > /proc/irq/<N>/smp_affinity_list
```

The kernel runtime path:
- `kernel/irq/affinity.c` — `irq_set_affinity()`, affinity spreading for vectors
- `kernel/irq/manage.c` — `irq_thread()`, threaded IRQ handlers

For MSI/MSI-X vectors (most modern NICs), affinity is set per-queue:
```bash
ethtool -L eth0 combined 4   # limit NIC to 4 queues
# then set each queue's IRQ to a housekeeping CPU
```

---

## Phase 4 — RCU Offloading (rcu_nocbs)

RCU (Read-Copy-Update) is a kernel synchronization mechanism.  By default,
each CPU runs RCU callbacks (grace period processing) on itself.  On isolated
CPUs this creates unpredictable latency spikes.

### 4.1 What RCU does on a CPU

Without `rcu_nocbs`:
- Each CPU periodically calls `rcu_check_callbacks()` from the scheduler tick
- When a grace period ends, the CPU runs `rcu_do_batch()` — potentially many callbacks
- This happens in softirq context, interrupting your RT task

### 4.2 rcu_nocbs mechanism

Source: `kernel/rcu/tree_nocb.h` (included into `kernel/rcu/tree.c`)

With `rcu_nocbs=4-7`:
1. CPUs 4-7 still participate in grace-period detection
2. But callbacks are enqueued to a `rcuo` kthread running on a housekeeping CPU
3. The RT CPU never runs `rcu_do_batch()`

Key functions to read:
```c
rcu_nocb_cpu_needs_barrier()   // fast-path check on isolated CPU
__call_rcu_nocb_enqueue()      // sends callback to rcuo kthread
rcu_nocb_kthread()             // the rcuo kthread body
```

### 4.3 RCU stall warnings

If your isolated task holds an RCU read lock too long, you get a stall warning.
See `Documentation/RCU/stallwarn.rst`.  On RT kernels this is often a sign of
priority inversion — an RT task waiting on a lock held by a lower-priority task
that can't run because your RT task is spinning.

---

## Phase 5 — PREEMPT_RT Kernel

`PREEMPT_RT` (now fully merged as of Linux 6.12) converts most in-kernel
spinlocks to sleeping rt-mutexes, making the kernel fully preemptible.

### 5.1 The core idea

In a non-RT kernel:
- Spinlocks disable preemption → a high-priority task can't preempt a low-priority task holding a spinlock
- This creates unbounded priority inversion

With `PREEMPT_RT`:
- `spin_lock()` becomes `rt_mutex_lock()` — the holder can be preempted
- Priority inheritance: if high-priority task blocks on a lock, it donates its priority to the holder

### 5.2 Key documents

- `Documentation/core-api/real-time/theory.rst` — motivation and design
- `Documentation/core-api/real-time/differences.rst` — what changes vs non-RT
- `Documentation/core-api/real-time/architecture-porting.rst` — porting drivers
- `Documentation/locking/locktypes.rst` — full table of lock types and RT behaviour
- `Documentation/locking/rt-mutex-design.rst` — pi-chain algorithm

### 5.3 Lock type changes under PREEMPT_RT

| Non-RT lock | RT replacement | Sleepable? |
|-------------|---------------|-----------|
| `spinlock_t` | `rt_mutex` (via `rtmutex.c`) | yes |
| `rwlock_t` | `rwbase_rt` | yes |
| `raw_spinlock_t` | stays a spinlock | no (use sparingly) |
| `local_lock_t` | per-CPU sleeping lock | yes |

Key source: `kernel/locking/rtmutex.c` — `rt_mutex_slowlock()`, PI chain walk

### 5.4 Enabling

```
CONFIG_PREEMPT_RT=y        # fully preemptible kernel
CONFIG_HZ_1000=y           # 1000 Hz tick for finer timer resolution
CONFIG_NO_HZ_FULL=y        # tickless isolated CPUs
CONFIG_RCU_NOCB_CPU=y      # rcu_nocbs= support
```

---

## Phase 6 — Power Management and C-states

Deep C-states (C6, C7, C8+) cause re-initialization latency on wake-up that
can be hundreds of microseconds.

### 6.1 C-state latency model

```
C0 (running) → C1 (halt) → C1E → C3 → C6 → C7 → C10
               ~1 µs exit    ~10 µs    ~100 µs    ~1 ms
```

Deeper states save more power but have higher exit latency.

### 6.2 Kernel interfaces

- `drivers/cpuidle/` — cpuidle framework, governs C-state selection
- `drivers/cpuidle/governors/menu.c` — default governor, predicts idle duration
- `drivers/cpuidle/governors/latency_req.c` — respects latency QoS requests
- `kernel/power/qos.c` — `cpu_latency_qos_add_request()` — userspace/driver API to cap exit latency

```bash
# Disable all C-states deeper than C1 via cpuidle
for cpu in /sys/devices/system/cpu/cpu*/cpuidle/state*/; do
    name=$(cat $cpu/name)
    [[ "$name" == "C1" || "$name" == "POLL" ]] && continue
    echo 1 > $cpu/disable
done

# Or: set a latency QoS limit from userspace
# writing to /dev/cpu_dma_latency (0 = no deep sleep)
exec 3>/dev/cpu_dma_latency && echo -ne '\x00\x00\x00\x00' >&3
# keep fd 3 open for duration of RT task
```

### 6.3 P-states / frequency scaling

Frequency transitions also cause latency.

```bash
# Pin to max frequency, disable boosting
cpupower frequency-set -g performance
echo 0 > /sys/devices/system/cpu/cpufreq/boost

# Or in kernel: CONFIG_CPU_FREQ_GOV_PERFORMANCE=y, set as default
```

Source: `drivers/cpufreq/` — `cpufreq_governor.c`, `intel_pstate.c`

---

## Phase 7 — Memory: Pre-faulting and Huge Pages

Page faults on an RT task cause it to block while the kernel allocates and maps
a page — potentially triggering reclaim, compaction, or I/O.

### 7.1 mlockall

```c
mlockall(MCL_CURRENT | MCL_FUTURE | MCL_ONFAULT);
```

- `MCL_CURRENT` — faults in all current mappings
- `MCL_FUTURE` — locks future mappings as they are created
- `MCL_ONFAULT` (Linux 4.4+) — locks pages when faulted, not at mlock time

Source path: `mm/mlock.c` → `__mlock_vma_pages_range()` → `get_user_pages()`

### 7.2 Pre-fault stack and heap

```c
// Pre-fault the stack by touching pages from the bottom
void prefault_stack(size_t stack_size) {
    volatile char *p = alloca(stack_size);
    memset((void*)p, 0, stack_size);
}

// Pre-fault heap allocations
ptr = malloc(size);
memset(ptr, 0, size);   // touch every page
```

### 7.3 Huge pages

TLB misses add latency.  Huge pages (2 MB) reduce TLB pressure for large
working sets.

```bash
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
# then in code:
madvise(ptr, size, MADV_HUGEPAGE);
```

Source: `mm/huge_memory.c` — `khugepaged` daemon, THP collapse logic

---

## Phase 8 — Measurement Tools

You cannot optimize what you cannot measure.  Use these in this order.

### 8.1 hwlatdetect / rtla hwnoise

Detects hardware-level latency (NMIs, SMIs) before the OS can do anything.

```bash
hwlatdetect --threshold=10 --window=1000000 --width=500000 --duration=120
# window = 1ms period, width = 500µs poll window per period
```

Kernel tracer backing this: `kernel/trace/trace_hwlat.c`
See also: `Documentation/tools/rtla/rtla-hwnoise.rst`

### 8.2 rtla osnoise

Measures OS noise — latency caused by the kernel on a CPU running an RT task.
Breaks down noise by source (IRQ, softirq, thread, NMI).

```bash
rtla osnoise top -c 4 -d 60s      # monitor CPU 4 for 60s
rtla osnoise hist -c 4 -d 60s     # histogram of noise events
```

Source: `kernel/trace/trace_osnoise.c`
Doc: `Documentation/tools/rtla/rtla-osnoise.rst`

### 8.3 rtla timerlat

Measures timer wakeup latency end-to-end: from when the timer fires to when the
task actually runs.  Decomposes into IRQ latency + thread latency.

```bash
rtla timerlat top -c 4 -d 60s
rtla timerlat hist -c 4 -p 1000   # 1 ms period
```

Source: `kernel/trace/trace_timerlat.c`
Doc: `Documentation/tools/rtla/rtla-timerlat.rst`

### 8.4 cyclictest

The classic RT latency benchmark — measures round-trip timer latency from
userspace.

```bash
cyclictest -m -p99 -t1 -a 4 -h 100 -i 1000 -D 60
# -m: mlockall, -p99: SCHED_FIFO priority 99
# -a 4: affine to CPU 4, -i 1000: 1ms interval
```

### 8.5 ftrace / tracers

For debugging specific latency spikes:

```bash
# Trace IRQ-off latency
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 100 > /sys/kernel/debug/tracing/tracing_thresh  # threshold in µs
cat /sys/kernel/debug/tracing/trace

# Trace preempt-off latency
echo preemptoff > /sys/kernel/debug/tracing/current_tracer
```

Source: `kernel/trace/trace_irqsoff.c`

---

## Phase 9 — Putting It Together: System Configuration Checklist

This maps directly to the checklist implied by the quote:

### BIOS/UEFI
- [ ] Disable NMI watchdog in BIOS
- [ ] Disable PCIe AER / SERR if not needed
- [ ] Disable legacy USB emulation (huge SMI source)
- [ ] Disable Intel Hyperthreading if sharing physical cores creates noise
- [ ] Disable patrol scrub / memory error correction polling
- [ ] Disable power management features (SpeedStep, TurboBoost) or pin to max freq
- [ ] Disable C-states deeper than C1 in BIOS

### Boot cmdline
```
isolcpus=domain,managed_irq,4-7
nohz_full=4-7
rcu_nocbs=4-7
irqaffinity=0-3
nmi_watchdog=0
intel_idle.max_cstate=1
processor.max_cstate=1
idle=poll             # extreme: busy-wait instead of any sleep (high power cost)
nosoftlockup
```

### Runtime
```bash
# IRQ affinity: move everything off isolated CPUs
service irqbalance stop
for i in /proc/irq/*/smp_affinity_list; do echo 0-3 > $i 2>/dev/null; done

# CPU frequency
cpupower frequency-set -g performance

# C-state: hold /dev/cpu_dma_latency open with value 0
```

### Application
```c
// Set SCHED_FIFO before going RT
struct sched_param sp = { .sched_priority = 99 };
sched_setscheduler(0, SCHED_FIFO, &sp);

// Lock memory
mlockall(MCL_CURRENT | MCL_FUTURE);

// Pre-fault stack and all buffers
// Pin to isolated CPU
cpu_set_t cpuset; CPU_ZERO(&cpuset); CPU_SET(4, &cpuset);
sched_setaffinity(0, sizeof(cpuset), &cpuset);
```

---

## Suggested Reading Order in the Kernel Source

1. `Documentation/core-api/real-time/theory.rst` — motivation
2. `Documentation/core-api/real-time/differences.rst` — what PREEMPT_RT changes
3. `Documentation/locking/locktypes.rst` — lock taxonomy
4. `Documentation/timers/no_hz.rst` — tickless operation
5. `kernel/sched/isolation.c` — CPU isolation implementation
6. `kernel/rcu/tree_nocb.h` — RCU offloading
7. `arch/x86/kernel/nmi.c` — NMI handling
8. `kernel/locking/rtmutex.c` — RT mutex + priority inheritance
9. `kernel/trace/trace_osnoise.c` — noise measurement framework
10. `kernel/trace/trace_timerlat.c` — timer latency tracer
11. `kernel/time/tick-sched.c` — NO_HZ_FULL tick suppression
12. `drivers/cpuidle/governors/menu.c` — idle state selection
13. `mm/mlock.c` — memory locking

---

## Further Study Topics (after the above)

- **SCHED_DEADLINE** (`kernel/sched/deadline.c`) — CBS-based real-time scheduling
  with bandwidth reservations; better than SCHED_FIFO for bounded tasks
- **membarrier** (`kernel/sched/membarrier.c`) — expedited memory barriers for
  lock-free RT/non-RT communication without stopping CPUs
- **io_uring + SQPOLL** (`io_uring/sqpoll.c`) — kernel thread polls SQ, eliminates
  syscall latency for I/O
- **AF_XDP** — zero-copy packet I/O bypassing the network stack entirely
- **VFIO + DPDK** — full device passthrough for when AF_XDP is not enough
