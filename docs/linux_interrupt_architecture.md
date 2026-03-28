# Linux Kernel Interrupt Architecture: A Comprehensive Technical Reference

**Date**: 2026-03-15
**Scope**: x86-64 unless otherwise noted

---

## Table of Contents

1. [Interrupt Taxonomy](#1-interrupt-taxonomy)
2. [Interrupt Routing Mechanisms](#2-interrupt-routing-mechanisms)
3. [Core Isolation and Interrupt Bypass](#3-core-isolation-and-interrupt-bypass)
4. [Interrupt Sources That Pierce Core Isolation](#4-interrupt-sources-that-pierce-core-isolation)
5. [CachyOS and Clear Linux Tuning Approaches](#5-cachyos-and-clear-linux-tuning-approaches)
6. [Tools to Observe and Diagnose Interrupts](#6-tools-to-observe-and-diagnose-interrupts)

---

## 1. Interrupt Taxonomy

The Linux kernel deals with several fundamentally different categories of interrupt, each with distinct delivery paths, handling contexts, and latency characteristics.

### 1.1 Hardware IRQs

Hardware interrupts are asynchronous signals from peripherals that cause the CPU to suspend normal execution and transfer control to an interrupt service routine (ISR). The kernel represents each interrupt source with an `irq_desc` structure (defined in `include/linux/irqdesc.h`) containing status flags, flow-handler pointers, chip descriptor, and a chain of `irqaction` structures (one per registered handler).

#### Legacy 8259A / XT-PIC

The original PC interrupt controller (Intel 8259A Programmable Interrupt Controller) provides 15 usable IRQ lines (IRQ 0–15, with IRQ 2 cascaded). These are level- or edge-triggered, not per-CPU, and must be shared across all CPUs via the BSP (Boot Strap Processor). The kernel still emulates the 8259 for legacy compatibility but real hardware has used the APIC model for decades.

Key limitation: IRQ sharing is unavoidable, and the single-destination routing model creates contention on SMP systems.

#### MSI (Message Signaled Interrupts)

MSI is a PCIe mechanism where devices write a small message (address + data) to a specially-mapped memory address instead of asserting a physical IRQ pin. Benefits over legacy IRQ:

- No IRQ sharing (each MSI message is unique)
- Lower latency (measured ~7x faster than XT-PIC baseline in Intel benchmarks)
- Up to 32 interrupt vectors per device (powers of 2: 1, 2, 4, … 32)
- Triggered by a PCI write, so ordering is guaranteed with respect to DMA data

Enabled via `pci_enable_msi()` or `pci_alloc_irq_vectors()` in drivers. Requires `CONFIG_PCI_MSI=y`.

#### MSI-X (Extended MSI)

MSI-X extends MSI to allow up to 2048 interrupt vectors per device, each independently maskable and each targetable to a different CPU. This is the mechanism used by modern high-performance NICs (e.g., Intel ixgbe, mlx5) to achieve per-CPU interrupt queues.

Each MSI-X entry has its own entry in the device's MSI-X table (a BAR-mapped structure), with an individual address/data pair and a per-vector mask bit. The kernel maps these through the IRQ domain hierarchy (see §2.4).

Enabled via `pci_enable_msix_range()`. Requires `CONFIG_PCI_MSI=y`.

### 1.2 Software Interrupts (Softirqs and Tasklets)

Software interrupts are deferred processing mechanisms run in a special non-preemptible (on non-RT kernels), non-sleepable context called "softirq context" or "BH context" (Bottom Half).

#### Softirqs

Softirqs are a fixed, statically-allocated set of deferred work types defined in `include/linux/interrupt.h`. They are designed for high-frequency, low-latency deferred processing. The full enum as of current kernels:

```c
enum {
    HI_SOFTIRQ        = 0,   /* High-priority tasklets */
    TIMER_SOFTIRQ     = 1,   /* Timer callbacks */
    NET_TX_SOFTIRQ    = 2,   /* Network transmit */
    NET_RX_SOFTIRQ    = 3,   /* Network receive */
    BLOCK_SOFTIRQ     = 4,   /* Block device completion */
    IRQ_POLL_SOFTIRQ  = 5,   /* IRQ polling (formerly BLOCK_IOPOLL) */
    TASKLET_SOFTIRQ   = 6,   /* Normal-priority tasklets */
    SCHED_SOFTIRQ     = 7,   /* Scheduler load balancing */
    HRTIMER_SOFTIRQ   = 8,   /* High-resolution timer callbacks */
    RCU_SOFTIRQ       = 9,   /* RCU callback processing */
    NR_SOFTIRQS       = 10
};
```

Softirqs are raised by calling `raise_softirq()` or `raise_softirq_irqoff()`, which sets a per-CPU bitmask. They are processed at `irq_exit()` time (after returning from a hardware interrupt), in `ksoftirqd` if the load is high, and at various `local_bh_enable()` call sites in the kernel.

The per-CPU thread `ksoftirqd/%u` is the fallback: if softirqs keep being re-raised (e.g., under heavy network load), the kernel eventually defers remaining processing to the `ksoftirqd` kthread to avoid starving user processes. Source: `kernel/softirq.c`.

#### Tasklets

Tasklets are built on top of the `HI_SOFTIRQ` (for `tasklet_hi_schedule()`) and `TASKLET_SOFTIRQ` (for `tasklet_schedule()`) vectors. Unlike raw softirqs:

- They can be registered dynamically at runtime (no fixed limit)
- A given tasklet instance runs on at most one CPU at a time (serialized)
- Different tasklet instances can run concurrently on different CPUs

Tasklets are being deprecated in the mainline kernel in favor of threaded IRQs and workqueues, as they cannot sleep and can hold off other softirq processing.

### 1.3 Inter-Processor Interrupts (IPIs)

IPIs are interrupts sent from one CPU to one or more other CPUs via the Local APIC. On x86, they use vectors in the range `0xf0–0xff` (defined in `arch/x86/include/asm/irq_vectors.h`). Key IPI types:

| Vector Name | Value | Purpose |
|---|---|---|
| `RESCHEDULE_VECTOR` | 0xfd | Tell a CPU to call `schedule()` |
| `CALL_FUNCTION_VECTOR` | 0xfc | Execute a function on multiple CPUs (`smp_call_function`) |
| `CALL_FUNCTION_SINGLE_VECTOR` | 0xfb | Execute a function on one specific CPU |
| `REBOOT_VECTOR` | 0xf8 | Initiate CPU reboot/shutdown |
| `IRQ_MOVE_CLEANUP_VECTOR` | 0xf7 | Clean up after IRQ migration |
| TLB flush vectors | 0xf0–0xf6 | TLB invalidation (multiple vectors spread load) |

**What triggers IPIs in practice:**

- **TLB shootdown**: When one CPU unmaps a page that may be cached in TLBs of other CPUs, it sends an IPI to those CPUs instructing them to invalidate their TLB entries. Triggered by `munmap()`, `mprotect()`, `madvise(MADV_FREE)`, COW page splits, and NUMA page migration. Implementation: `arch/x86/mm/tlb.c`.
- **Reschedule**: When a task is woken up and the scheduler decides it should run on a different CPU (via `ttwu_queue`), a reschedule IPI is sent. Triggered by any `wake_up*()` call where the target CPU differs from the current.
- **RCU expedited grace periods**: `synchronize_rcu_expedited()` sends IPIs to all CPUs to force a quiescent state immediately rather than waiting for a natural one.
- **CPU frequency scaling**: Some cpufreq governors send IPIs to coordinate frequency changes across cores in the same clock domain.
- **Workqueue management**: `kick_all_cpus_sync()` wakes per-CPU kworker threads.
- **perf events**: When a sampling counter overflows on one CPU, it may need to notify another.

### 1.4 NMI (Non-Maskable Interrupts)

NMIs are interrupts that cannot be blocked by the CPU's interrupt flag (`IF`). On x86, NMIs arrive on a dedicated pin and use vector 2. Because they are non-maskable, they can interrupt any kernel context including spin-locked critical sections, making NMI handlers extremely constrained (they cannot take locks).

Key NMI sources in Linux:

- **NMI watchdog (hardlockup detector)**: Uses a performance counter (`CONFIG_LOCKUP_DETECTOR=y`, `CONFIG_HARDLOCKUP_DETECTOR=y`) to fire an NMI at a rate of once per `watchdog_thresh` seconds (default: 10). If a CPU has not seen a softirq in that window, a hardlockup is reported. Configured via `/proc/sys/kernel/watchdog` and `/proc/sys/kernel/watchdog_thresh`. On `NO_HZ_FULL` systems, the watchdog runs only on housekeeping CPUs by default.
- **MCE (Machine Check Exception)**: Hardware error reporting (ECC errors, bus faults, thermal events). Handled in `arch/x86/kernel/cpu/mce/`. MCEs use a special MCE vector (vector 18 / `#MC`), technically a fault/exception rather than a true NMI but with similar non-maskable delivery semantics. Cannot be disabled on most hardware.
- **IOCK NMI**: I/O bus parity errors (legacy).
- **Debugging NMIs**: `perf` and `oprofile` use NMI-based performance monitoring interrupts (PMIs) to implement hardware sampling.
- **IPMI/BMC NMIs**: Out-of-band management hardware can inject NMIs.

NMI handling in Linux uses a dedicated NMI stack on x86 (IST-based). Source: `arch/x86/kernel/nmi.c`.

### 1.5 Timer Interrupts

#### The Periodic Tick (HZ tick)

The traditional Linux timer tick is a periodic interrupt generated by a per-CPU clock event device (Local APIC timer on modern x86) at `CONFIG_HZ` frequency. This tick drives:

- Scheduler accounting (`scheduler_tick()`)
- `jiffies` increment (only on one CPU in modern kernels)
- `TIMER_SOFTIRQ` processing (low-resolution timer wheel)
- Load average calculation
- CPU time accounting

`CONFIG_HZ` choices: 100, 250, 300, 1000. Higher values reduce scheduling latency but increase interrupt overhead. Desktop distributions typically use 250 or 1000; server distributions may use 100 or 250.

#### High-Resolution Timers (hrtimers)

`CONFIG_HIGH_RES_TIMERS=y` enables hrtimers, which are nanosecond-resolution timers backed by hardware clocksources (HPET, TSC deadline timer, Local APIC timer in TSC-deadline mode). When active, the periodic tick is replaced by a per-CPU hrtimer that fires only when the next timer actually expires—this is the foundation of dynamic ticks.

Source: `kernel/time/hrtimer.c`, `kernel/time/tick-sched.c`.

#### NO_HZ_IDLE (Dyntick Idle)

`CONFIG_NO_HZ_IDLE=y` (the default in most distributions): When a CPU is idle, the tick is stopped entirely. The CPU programs a one-shot wakeup at the time of the next timer event and then enters a sleep state. This dramatically reduces power consumption and improves performance on virtualized systems where spurious timer interrupts are expensive.

Boot parameter: `nohz=on` (default when `CONFIG_NO_HZ_IDLE=y`).

#### NO_HZ_FULL (Adaptive Ticks)

`CONFIG_NO_HZ_FULL=y`: When a CPU has exactly one runnable task (not counting the idle task), the scheduler tick is stopped even while the CPU is busy. This eliminates the primary source of periodic jitter for latency-sensitive applications.

Conditions that re-enable the tick on a `nohz_full` CPU:
- More than one runnable task on the CPU
- Active POSIX CPU timers (`clock_gettime(CLOCK_PROCESS_CPUTIME_ID)`, `setitimer`)
- Active `perf` events sampling CPU time
- RCU callbacks queued locally (mitigated by `rcu_nocbs=`)
- Certain scheduler load-balancing operations
- Reliable TSC not available (x86 requirement for `NO_HZ_FULL`)

A residual tick at approximately 1 Hz is maintained for scheduler statistics (jiffies-based load average, CFS vruntime). Kernel documentation notes: "Some process-handling operations still require the occasional scheduling-clock tick...They are currently accommodated by scheduling-clock tick every second or so."

Boot parameter: `nohz_full=<cpulist>` (e.g., `nohz_full=1-7`).

Requires: `CONFIG_NO_HZ_FULL=y`, `CONFIG_RCU_NOCB_CPU=y` (mandatory co-dependency).

### 1.6 Exceptions and Fault Interrupts

CPU exceptions are synchronous interrupts triggered by the executing instruction itself. On x86, these use vectors 0–31. Key ones:

| Vector | Name | Common Linux trigger |
|---|---|---|
| 0 | `#DE` Divide Error | Integer divide by zero |
| 6 | `#UD` Invalid Opcode | Illegal instruction |
| 8 | `#DF` Double Fault | Stack overflow in kernel |
| 13 | `#GP` General Protection | Unaligned access, privilege violation |
| 14 | `#PF` Page Fault | Every `malloc`/page-in, CoW, swap-in |
| 18 | `#MC` Machine Check | Hardware error reporting |
| 19 | `#XF` SIMD Exception | FPU/SSE fault |

Page faults (`#PF`) are by far the most common exception in normal operation. They are handled in `arch/x86/mm/fault.c` → `do_page_fault()` → `handle_mm_fault()`. On isolated CPUs running latency-sensitive code, page faults are a significant source of jitter; `mlockall(MCL_CURRENT | MCL_FUTURE)` is used to pre-fault all pages.

---

## 2. Interrupt Routing Mechanisms

### 2.1 APIC Architecture

#### Local APIC (LAPIC)

Every CPU has its own Local APIC, typically integrated into the processor die. The LAPIC:

- Receives interrupts from the I/O APIC, other LAPICs (IPIs), and local sources
- Manages the Local Vector Table (LVT) for per-CPU interrupt sources: timer, thermal, performance counter, LINT0/LINT1
- Delivers interrupts to the CPU core at the appropriate priority
- Implements interrupt priority registers (TPR/PPR) for masking lower-priority vectors
- Provides the `EOI` (End-of-Interrupt) register that clears the ISR (In-Service Register)

On x86-64, the LAPIC is memory-mapped at `0xFEE00000` (or via MSR in x2APIC mode).

#### I/O APIC

The I/O APIC (typically one per PCH/chipset, possibly more) receives interrupt signals from external devices and routes them to LAPICs. Its Interrupt Redirection Table (IOREDTBL) contains one 64-bit entry per external IRQ line, specifying:

- **Destination**: Which CPU(s) to target (physical mode: single LAPIC ID; logical mode: bitmask)
- **Delivery mode**: Fixed, lowest-priority, SMI, NMI, INIT, ExtINT
- **Trigger mode**: Edge or level
- **Vector**: The interrupt vector number (32–255)
- **Mask bit**: Whether the interrupt is currently masked

The Linux kernel programs the IOREDTBL during boot via ACPI/MP table parsing. Source: `arch/x86/kernel/apic/io_apic.c`.

**Performance note**: Intel benchmarks show I/O APIC reduces interrupt latency ~3x vs. 8259 emulation; MSI reduces it ~7x vs. baseline.

### 2.2 IRQ Affinity

Each IRQ has affinity masks controlling which CPUs may handle it:

```
/proc/irq/<N>/smp_affinity        # hex bitmask (CPU 0 = bit 0)
/proc/irq/<N>/smp_affinity_list   # human-readable CPU list (e.g., "0-3")
/proc/irq/<N>/effective_affinity  # actual effective mask after constraints
/proc/irq/<N>/node                # NUMA node affinity
```

Writing to `smp_affinity` or `smp_affinity_list` reprograms the I/O APIC redirection table entry (for I/O APIC interrupts) or the MSI message address/data (for MSI/MSI-X interrupts) to target the specified CPU(s).

To redirect all hardware IRQs away from an isolated CPU set (e.g., CPUs 1-7):

```bash
for irq in /proc/irq/*/smp_affinity_list; do
    echo 0 > "$irq" 2>/dev/null  # CPU 0 only
done
```

Note: The per-CPU timer vector (vector 0 on x86, `LOCAL_TIMER_VECTOR`) cannot be redirected—it will fail with `EINVAL`. This is expected and handled by `nohz_full`.

#### irqbalance Daemon

`irqbalance` is a userspace daemon that monitors `/proc/interrupts` and dynamically redistributes IRQ affinity across CPUs to balance load and respect NUMA topology. It reads the IRQ counts periodically and rewrites `smp_affinity`.

Important: `irqbalance` respects CPUs listed in the kernel `isolcpus=` boot parameter by default—it will not assign interrupts to isolated CPUs. This can be overridden with `--banirq` and `--banscript` options, or disabled entirely for fully manual control.

### 2.3 MSI/MSI-X Per-CPU Vectors

With MSI-X, a device like a NIC can have one interrupt vector per CPU (or per TX/RX queue), with each vector's affinity pinned to a specific CPU. This eliminates inter-CPU interrupt delivery overhead and allows interrupt processing to happen on the same CPU where the associated data was prepared (cache locality).

The typical setup for a multi-queue NIC:

```bash
# Set queue N's IRQ to CPU N
ethtool -L eth0 combined 8     # 8 combined queues
for i in $(seq 0 7); do
    irq=$(cat /proc/interrupts | grep "eth0-TxRx-$i" | awk '{print $1}' | tr -d ':')
    echo $i > /proc/irq/$irq/smp_affinity_list
done
```

### 2.4 IRQ Domain Subsystem

The IRQ domain library (`kernel/irq/irqdomain.c`, `include/linux/irq_domain.h`) solves the mapping problem between hardware-specific IRQ numbers (hwirq) and kernel-internal Linux IRQ numbers.

**Mapping types:**

- **Linear**: Fixed-size table indexed by hwirq, O(1) lookup. Preferred for small hwirq spaces (≤256).
- **Tree (radix tree)**: For sparse or large hwirq spaces.
- **No-map** (`CONFIG_IRQ_DOMAIN_NOMAP`): Driver programs Linux IRQ number directly into hardware.
- **Legacy**: Pre-allocated descriptor ranges; deprecated, avoid in new code.

**Hierarchical domains**: For cascaded interrupt controllers (the common case on x86), irq_domain instances form a hierarchy matching hardware topology:

```
Device → IOAPIC domain → Remapping domain → LAPIC domain → CPU
```

Each domain only manages its own hardware layer. MSI domains sit at the device level and translate MSI messages into hwirq numbers for the IOAPIC/remapping domain above them. Requires `CONFIG_IRQ_DOMAIN_HIERARCHY=y` (selected by `CONFIG_X86`).

The `irq_create_mapping()` function creates a hwirq→Linux IRQ mapping; `irq_find_mapping()` performs reverse lookup. These are used internally by interrupt controller drivers.

---

## 3. Core Isolation and Interrupt Bypass

### 3.1 `isolcpus=` Kernel Parameter

`isolcpus=<flags>,<cpulist>` removes listed CPUs from the general-purpose scheduler domain at boot time.

**What it does:**
- Removes isolated CPUs from the load-balancing domain (no tasks migrate to/from them automatically)
- Prevents the scheduler from assigning tasks to them via normal scheduling
- Sets `isolcpus` bit in `/sys/devices/system/cpu/cpuN/topology/` (visible to tools)
- `irqbalance` respects these CPUs and avoids assigning IRQs to them

**What it does NOT do:**
- Does not stop the periodic scheduler tick
- Does not prevent hardware IRQs from being delivered
- Does not prevent IPIs
- Does not offload RCU callbacks

**Flags** (prefixed to cpulist since kernel 4.15):
- `isolcpus=domain,<cpulist>`: Remove from scheduling domain (legacy behavior; now explicit)
- `isolcpus=nohz,<cpulist>`: Also enable `nohz_full` for listed CPUs
- `isolcpus=managed_irq,<cpulist>`: Exclude from managed IRQ affinity masks

The `isolcpus=` functionality was unified with the housekeeping subsystem in `kernel/sched/isolation.c`. The parameter now calls `housekeeping_setup()` with `HK_TYPE_DOMAIN` (and optionally `HK_TYPE_TICK`).

### 3.2 cgroups cpuset Isolation

The `cpuset` cgroup controller provides a complementary mechanism for restricting task placement, usable at runtime without a reboot:

```bash
# Create a cpuset cgroup
mkdir /sys/fs/cgroup/cpuset/isolated
echo 4-7 > /sys/fs/cgroup/cpuset/isolated/cpuset.cpus
echo 0   > /sys/fs/cgroup/cpuset/isolated/cpuset.sched_load_balance
# Move task to isolated set
echo $PID > /sys/fs/cgroup/cpuset/isolated/cgroup.procs
```

Setting `cpuset.sched_load_balance=0` disables cross-CPU load balancing within the cpuset, similar to `isolcpus=domain`. However, cpusets do not affect hardware IRQ routing—that still requires manual `smp_affinity` manipulation.

### 3.3 `nohz_full=` (Adaptive Ticks / NO_HZ_FULL)

`nohz_full=<cpulist>` activates full dynticks on the listed CPUs. This is the primary mechanism for tick-noise reduction.

**What it does beyond `isolcpus=`:**
- Stops the periodic scheduler tick when exactly one runnable task is on the CPU
- Automatically implies `rcu_nocbs=<cpulist>` (see §3.4)
- Moves unbound workqueue work to housekeeping CPUs
- Defers `vmstat` per-CPU accounting updates to housekeeping CPUs (patch in mainline: `vmstat: skip periodic vmstat update for isolated CPUs`)
- CPUs enter RCU extended quiescent state on return to userspace

**Kernel config requirements:**
```
CONFIG_NO_HZ_FULL=y
CONFIG_RCU_NOCB_CPU=y
CONFIG_HZ_PERIODIC=n
```

**Tick re-enablement conditions** (tick comes back even on `nohz_full` CPUs):
1. More than one runnable task on the CPU
2. `POSIX CPU-time timers` active on the running task (`CLOCK_PROCESS_CPUTIME_ID`, `CLOCK_THREAD_CPUTIME_ID`)
3. `perf` event sampling that uses scheduler-clock ticks
4. RCU callbacks pending locally (requires `rcu_nocbs=` to mitigate)
5. 1 Hz residual tick for scheduler statistics (cannot be fully eliminated in current kernels without `task_isolation` patches)
6. Kernel subsystems that pin timers or workqueues to specific CPUs

### 3.4 `rcu_nocbs=` (RCU Callback Offloading)

`rcu_nocbs=<cpulist>` moves RCU callback processing off the listed CPUs. RCU callbacks (i.e., the deferred work registered via `call_rcu()`) are executed by per-CPU kthreads `rcuop/%d` (preemptible RCU) running on unbound kthreads that the scheduler places on non-`nocbs` CPUs.

This is automatically implied by `nohz_full=` but can be specified independently. Requires `CONFIG_RCU_NOCB_CPU=y`.

Without `rcu_nocbs=`, RCU callbacks queued on a CPU cause the tick to re-enable on that CPU (to process them), which defeats `nohz_full`.

Related RCU tuning:
- `rcupdate.rcu_normal=1`: Disables expedited RCU grace periods (prevents expedited-grace-period IPIs)
- `rcupdate.rcu_expedited=0`: Same as above (older parameter name)
- `CONFIG_RCU_BOOST=y`: Priority-boosts RCU readers to prevent grace-period stalls, at the cost of running `rcub/%d` kthreads

### 3.5 The Housekeeping Subsystem

The housekeeping subsystem (`kernel/sched/isolation.c`, `include/linux/sched/isolation.h`) is the kernel-internal framework underpinning `isolcpus=` and `nohz_full=`. It maintains per-feature cpumasks of "housekeeping" CPUs (those that perform kernel bookkeeping work on behalf of isolated CPUs).

The `HK_TYPE_*` enum (from `include/linux/sched/isolation.h`):

```c
enum hk_type {
    HK_TYPE_TIMER,       /* timers routed to housekeeping CPUs */
    HK_TYPE_RCU,         /* RCU grace-period processing */
    HK_TYPE_MISC,        /* miscellaneous kernel work */
    HK_TYPE_SCHED,       /* scheduler tick and load balancing */
    HK_TYPE_TICK,        /* timer tick (nohz_full) */
    HK_TYPE_DOMAIN,      /* scheduling domain (isolcpus=domain) */
    HK_TYPE_WQ,          /* unbound workqueue placement */
    HK_TYPE_MANAGED_IRQ, /* managed IRQ affinity */
    HK_TYPE_KTHREAD,     /* per-CPU kthread placement */
    HK_NR_TYPES,
};
```

The function `housekeeping_cpumask(type)` returns the cpumask of CPUs that handle work of the given type. Various kernel subsystems call `housekeeping_any_cpu(type)` to find a housekeeping CPU to delegate work to.

**`isolcpus=` vs. `nohz_full=` — key differences:**

| Aspect | `isolcpus=domain` | `nohz_full=` |
|---|---|---|
| Scheduler tick suppressed | No | Yes (when single task) |
| Load balancing disabled | Yes | Partial |
| RCU offloaded | No | Yes (implied) |
| Runtime configurable | No (boot-time only) | No (boot-time only) |
| Workqueues moved | Partial | Yes (unbound WQs) |
| vmstat deferred | No | Yes |
| IRQ affinity changed | No (still manual) | No (still manual) |
| HK_TYPE affected | `HK_TYPE_DOMAIN` | `HK_TYPE_TICK`, `HK_TYPE_RCU`, `HK_TYPE_WQ`, etc. |

In practice, full isolation requires **both** `isolcpus=domain` **and** `nohz_full=` targeting the same CPU set, plus manual IRQ affinity management.

---

## 4. Interrupt Sources That Pierce Core Isolation

Even with `isolcpus=`, `nohz_full=`, `rcu_nocbs=`, and fully steered IRQ affinity, the following interrupt sources can still reach isolated CPUs. This section documents each source, the conditions that trigger it, and available mitigations.

### 4.1 Scheduler Tick (Residual 1 Hz)

**Pierces**: `nohz_full=`
**Trigger**: Process scheduling statistics maintenance, load average computation, CFS vruntime normalization
**Frequency**: ~1 Hz even on fully-tickless CPUs in current mainline
**Mitigation**: No complete mitigation in mainline. The experimental `task_isolation` patchset (proposed via `prctl(PR_SET_TASK_ISOLATION)`) addresses this but has not been merged as of kernel 6.12. With `rcupdate.rcu_normal=1`, some tick re-arming from RCU is avoided.

### 4.2 RCU Grace Period IPIs (Expedited)

**Pierces**: `isolcpus=`, `nohz_full=`
**Trigger**: `synchronize_rcu_expedited()` calls from any CPU. Common callers: `synchronize_rcu_expedited()` is used by module loading, device hotplug, certain locking primitives.
**Frequency**: Triggered by system events, not periodic. Can be frequent on busy systems.
**Mitigation**: `rcupdate.rcu_normal=1` (boot param) forces all expedited grace periods to use normal (non-IPI) grace periods. Cost: higher grace period latency. `rcu_nocbs=` prevents RCU callbacks from queuing locally but does not prevent expedited IPIs.

### 4.3 TLB Shootdown IPIs

**Pierces**: `isolcpus=`, `nohz_full=`
**Trigger**: Any CPU modifying the virtual-to-physical mapping of pages that may be cached in the TLB of the isolated CPU. This occurs on:
  - `munmap()` of mapped memory
  - `mprotect()` changing page permissions
  - `madvise(MADV_FREE)` or `madvise(MADV_DONTNEED)`
  - `mremap()` moving mappings
  - CoW page splits (forking)
  - NUMA automatic page migration (`CONFIG_NUMA_BALANCING`)
  - Kernel page table modifications (e.g., KPTI flushes)

**Frequency**: Application-dependent. A process that frequently maps/unmaps memory, or a kernel doing NUMA balancing, can generate dozens to thousands of TLB shootdowns per second.
**Mitigation**:
  - `mlockall(MCL_CURRENT | MCL_FUTURE)` to pre-fault and lock all pages
  - `echo 0 > /proc/sys/kernel/numa_balancing` to disable NUMA auto-balancing
  - Avoid `munmap()` / `madvise(MADV_FREE)` in the hot path; use huge pages (2MB/1GB) to reduce TLB entries
  - Disable Transparent Huge Pages (THP) auto-promotion to avoid promotions triggering shootdowns: `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
  - In userspace: never release memory during latency-sensitive operation

### 4.4 Reschedule IPIs

**Pierces**: `isolcpus=` (partially), `nohz_full=`
**Trigger**: `wake_up*()` calls that select an isolated CPU as the target for a waking task (via `ttwu_queue`). Also triggered by `smp_send_reschedule()`.
**Frequency**: Every task wakeup targeting the isolated CPU.
**Mitigation**: On `isolcpus=domain` CPUs, the scheduler does not select them for new tasks via load balancing, but a task pinned to an isolated CPU (`sched_setaffinity()`) will still receive a reschedule IPI when woken from another CPU. Ensuring only one task is affined to the isolated CPU minimizes this.

### 4.5 POSIX Timer Expiry

**Pierces**: `nohz_full=`
**Trigger**: Any active `ITIMER_*` (from `setitimer()`), `timer_create()` with `CLOCK_PROCESS_CPUTIME_ID` or `CLOCK_THREAD_CPUTIME_ID`, or `clock_nanosleep()` with CPU clocks.
**Effect**: Re-enables the scheduler tick on the isolated CPU.
**Mitigation**: Do not use POSIX CPU timers in isolated workloads. Use wall-clock `CLOCK_MONOTONIC` timers instead, which can be handled without a per-CPU tick.

### 4.6 `perf_events` Sampling

**Pierces**: `nohz_full=`, potentially also isolated CPUs via PMU overflow NMIs
**Trigger**: `perf_event_open()` with `CLOCK_MONOTONIC`-based sampling, or PMU hardware counter overflow on the isolated CPU.
**Effect**: PMU overflow generates an NMI (non-maskable, cannot be prevented). CPU-time sampling (`PERF_CLOCK_PROCESS`) re-enables the tick.
**Mitigation**: Do not run `perf` sampling on isolated CPUs during latency-sensitive operation. Disable system-wide perf sampling (`/proc/sys/kernel/perf_event_paranoid=3`, or disable `CONFIG_PERF_EVENTS=n` if extreme isolation is needed).

### 4.7 NMI Watchdog

**Pierces**: `nohz_full=`, `isolcpus=` — NMIs are non-maskable by definition
**Trigger**: Periodic hardware performance counter overflow, configured by `CONFIG_HARDLOCKUP_DETECTOR=y`
**Frequency**: Once per `watchdog_thresh` seconds (default 10 s), but reconfigured by `/proc/sys/kernel/watchdog_thresh`
**Behavior with `nohz_full`**: By default, the watchdog runs only on housekeeping CPUs when `nohz_full=` is active. The kernel source in `kernel/watchdog.c` checks `housekeeping_cpumask(HK_TYPE_TIMER)` to determine watchdog CPU placement.
**Mitigation**:
  - Explicitly disable: `echo 0 > /proc/sys/kernel/watchdog`
  - Boot parameter: `nosoftlockup nmi_watchdog=0`
  - Kernel build: `CONFIG_LOCKUP_DETECTOR=n` (removes entirely)

### 4.8 MCE (Machine Check Exception)

**Pierces**: Everything — MCEs are hardware-generated faults that cannot be masked
**Trigger**: Hardware errors: ECC DRAM errors, CPU internal errors, PCIe bus errors, thermal events, DIMM failures
**Frequency**: Rare in healthy systems; can become frequent on failing hardware
**Mitigation**: None (disabling MCE would hide hardware failures). On production systems, ensure ECC memory and monitor MCE logs via `mcelog` or `rasdaemon`.

### 4.9 `vmstat` Deferred Work

**Pierces**: `nohz_full=` (historically)
**Mechanism**: The VM statistics subsystem maintains per-CPU counters and periodically flushes them. The flush was previously triggered by a per-CPU delayed work item (`vmstat_work`) that fired at `CONFIG_HZ`-based intervals.
**Status**: Mitigated in mainline. The kernel now skips periodic `vmstat_update` flushes for CPUs in the `nohz_full` set (patch: "vmstat: skip periodic vmstat update for isolated CPUs"). The `vm.stat_interval` sysctl controls flush interval on non-isolated CPUs.

### 4.10 `kworker` and Workqueue Items

**Pierces**: `isolcpus=` (partially)
**Trigger**: Kernel subsystems that schedule work on per-CPU workqueues (block completions, networking, filesystem, etc.)
**Mitigation**: `nohz_full=` moves unbound workqueues to housekeeping CPUs. Per-CPU (bound) workqueues pinned to an isolated CPU still run there. Mitigate with:
  ```bash
  # Move unbound workqueues away from CPU 4
  echo 0-3 > /sys/devices/virtual/workqueue/*/cpumask
  ```

### 4.11 Hypervisor Interrupts (VMs only)

**Pierces**: All isolation mechanisms
**Trigger**: The VMM/hypervisor can inject virtual interrupts or reschedule virtual CPUs at any time, causing the guest to see spurious IPIs or LAPIC timer fires.
**Mitigation**: Use CPU pinning (`taskset` / VM CPU pinning), dedicated physical CPUs for VMs, and disable overcommitment. Paravirtualized systems (`KVM` with `virtio`) can negotiate interrupt coalescing.

---

## 5. CachyOS and Clear Linux Tuning Approaches

### 5.1 CachyOS Kernel

CachyOS (an Arch-based distribution) ships several kernel variants, each a patchset applied to mainline or LTS kernels. As of early 2026, the default kernel is based on Linux 6.13.

**Kernel variants:**

| Package | Scheduler | Preemption | Notes |
|---|---|---|---|
| `linux-cachyos` | BORE | PREEMPT | Default |
| `linux-cachyos-bore` | BORE | PREEMPT | Explicit BORE |
| `linux-cachyos-eevdf` | EEVDF (mainline) | PREEMPT | Upstream scheduler |
| `linux-cachyos-bmq` | BMQ | PREEMPT | Alfred Chen's scheduler |
| `linux-cachyos-rt-bore` | BORE | PREEMPT_RT | Full real-time |
| `linux-cachyos-lts` | BORE | PREEMPT | LTS base |
| `linux-cachyos-server` | EEVDF | NONE/VOLUNTARY | Server optimization |

**Key CONFIG options set by CachyOS:**

```
# Timer frequency
CONFIG_HZ_1000=y          # Default; 1000 Hz for responsiveness
CONFIG_HZ_300=y           # Server variant
# Also supports: 500, 600, 750 Hz (user-selectable)

# Tick mode
CONFIG_NO_HZ_FULL=y       # Adaptive ticks on designated CPUs
# (CONFIG_NO_HZ_IDLE=y is always enabled as well)

# RCU
CONFIG_RCU_NOCB_CPU=y     # Enable RCU callback offloading
CONFIG_RCU_BOOST=y        # Priority-boost preempted RCU readers

# Preemption (non-RT variants)
CONFIG_PREEMPT=y           # Full preemption (not PREEMPT_RT)
CONFIG_PREEMPT_RT=y        # Only in linux-cachyos-rt-bore

# Compilation
CONFIG_LTO_CLANG_THIN=y   # Thin LTO with Clang
CONFIG_CFI_CLANG=y        # kCFI (kernel Control Flow Integrity)

# Memory management patches
# le9uo patch: adjusts vm.watermark_scale_factor to prevent OOM thrashing
# Zen-kernel MM tweaks: adjusted compaction aggressiveness, watermark scale
```

**BORE Scheduler (Burst-Oriented Response Enhancer):**

BORE modifies the CFS/EEVDF scheduler to track a "burstiness" score per task. Tasks that use CPU in short bursts (typical of interactive applications, games) get a small priority boost relative to CPU-bound tasks. This reduces scheduling latency for interactive workloads without dramatically affecting throughput.

BORE is a patchset on top of mainline EEVDF (Linux 6.6+). It does not fundamentally change interrupt handling but improves scheduling fairness under mixed workloads.

**Compilation flags** (interrupt-relevant):
- `-fivopts`: Induction variable optimization, can reduce loop overhead in hot interrupt paths
- `-fmodulo-sched`: Software pipelining, improves instruction-level parallelism
- AutoFDO (Automatic Feedback-Directed Optimization): Profile-guided optimization using perf data—can optimize interrupt handler code paths based on real workload profiles

### 5.2 Clear Linux Kernel

Intel's Clear Linux uses a different approach: highly optimized for Intel hardware, prioritizing throughput and benchmark performance over low latency.

**Key characteristics:**

```
# Preemption model
CONFIG_PREEMPT_VOLUNTARY=y   # Voluntary preemption (not full PREEMPT)
                              # Favors throughput over latency

# Timer frequency
CONFIG_HZ_250=y              # Lower frequency, reduces timer overhead

# Tick mode
CONFIG_NO_HZ_IDLE=y          # Tickless idle only (not NO_HZ_FULL)

# Compiler
Clang + ThinLTO + PGO        # Profile-guided optimization
x86_64-v3 target             # AVX2-capable machines (Haswell+)
```

Clear Linux does not aim for real-time latency; it aims for maximum IPC throughput on Intel hardware. Its interrupt-related optimizations are:

1. **Reduced timer overhead** (250 Hz vs. 1000 Hz)
2. **Voluntary preemption** reduces lock contention in kernel paths
3. **PGO**: Hot interrupt handler paths are compiled with actual profile data
4. **Intel-specific optimizations**: IOMMU passthrough modes, PCIe optimizations that reduce interrupt overhead for Intel NICs

### 5.3 PREEMPT_RT (Real-Time Preemption)

Fully merged into mainline with Linux 6.12 (September 2024). Previously a separate patchset maintained by Thomas Gleixner, Ingo Molnár, and others.

**What PREEMPT_RT changes:**

1. **Forced interrupt threading**: With `CONFIG_PREEMPT_RT=y` (or `threadirqs` boot param + `CONFIG_IRQ_FORCED_THREADING=y`), all interrupt handlers run in kernel threads (`irq/<N>-<name>`) rather than hard-IRQ context. Exceptions: handlers marked `IRQF_NO_THREAD` (clocksource event interrupts, perf, cascaded IRQ controllers).

2. **Priority inheritance mutexes**: `spinlock_t` becomes a sleeping lock with priority inheritance under `PREEMPT_RT`, preventing priority inversion.

3. **Preemptible RCU read-side critical sections**: Under `PREEMPT_RT`, RCU read-side sections can be preempted, enabling lower worst-case latency.

4. **High-resolution timer migration**: Timer handling unified with hrtimer infrastructure.

**Interrupt threading impact:**
Each hardware IRQ gets a kernel thread `irq/<N>-<name>` at priority `SCHED_FIFO:50` by default. The thread can be reprioritized:
```bash
chrt -f -p 90 $(pgrep "irq/24-eth0")
```
This allows IRQ handler priority to be explicitly managed alongside application priorities.

**Relevant config options:**

```
CONFIG_PREEMPT_RT=y                    # Full PREEMPT_RT
CONFIG_IRQ_FORCED_THREADING=y          # Force all IRQs to threads
CONFIG_GENERIC_IRQ_FORCED_THREADING=y  # Architecture-independent forced threading
CONFIG_PREEMPT_LAZY=y                  # "PREEMPT_AUTO" — adaptive between voluntary/full
```

Note: `CONFIG_PREEMPT_RT=y` and `CONFIG_HZ_1000=y` together give the best worst-case latency but lower throughput than voluntary preemption.

### 5.4 CONFIG_HZ Choices

| Value | Use case | Scheduling granularity | Timer overhead |
|---|---|---|---|
| 100 Hz | Servers, batch workloads | 10 ms | Lowest |
| 250 Hz | General purpose (Debian/Ubuntu default) | 4 ms | Low |
| 300 Hz | Divisible by 60 (LCD refresh) and 100 | 3.33 ms | Low-medium |
| 1000 Hz | Desktop, gaming, low-latency | 1 ms | Higher |

With `NO_HZ_FULL=y`, the periodic tick only fires for CPUs with multiple runnable tasks, so the `CONFIG_HZ` value mainly affects those CPUs and latency when the tick fires.

### 5.5 Additional Interrupt-Reduction Kernel Configs

```
# Softirq / BH processing
CONFIG_PREEMPT_RT=y           # Converts ksoftirqd to RT thread

# RCU
CONFIG_RCU_NOCB_CPU=y         # Offload RCU callbacks
CONFIG_RCU_NOCB_CPU_DEFAULT_ALL=y  # All CPUs offload by default (no boot param needed)
CONFIG_RCU_BOOST=y            # Boost preempted RCU readers

# Watchdog
CONFIG_LOCKUP_DETECTOR=y      # Soft/hard lockup detection (disable for isolation)
CONFIG_HARDLOCKUP_DETECTOR=y  # NMI-based hardlockup detection

# Timer
CONFIG_HZ_PERIODIC=n          # Must be n for NO_HZ to work
CONFIG_HIGH_RES_TIMERS=y      # Required for NO_HZ_FULL

# IRQ threading
CONFIG_IRQ_FORCED_THREADING=y # Available without full PREEMPT_RT

# Tracing (disable for production isolation)
CONFIG_TRACING=n               # Eliminates tracepoint overhead
CONFIG_FTRACE=n

# CPU isolation
CONFIG_CPU_ISOLATION=y         # Housekeeping subsystem
CONFIG_CPUSETS=y               # cgroup cpuset controller
```

---

## 6. Tools to Observe and Diagnose Interrupts

### 6.1 `/proc/interrupts`

The primary per-CPU interrupt counter file. Updated atomically in the interrupt handler. Format:

```
           CPU0       CPU1       CPU2       CPU3
  0:         28          0          0          0  IR-IO-APIC    2-edge      timer
  1:          0          0         83          0  IR-IO-APIC    1-edge      i8042
 ...
 24:          0      12483          0          0  IR-PCI-MSI 524288-edge   xhci_hcd
NMI:        123        120        118        119   Non-maskable interrupts
LOC:     125400     125134     124892     125210   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
PMI:        123        120        118        119   Performance monitoring interrupts
IWI:          0          0          0          0   IRQ work interrupts
RTR:          0          0          0          0   APIC ICR read retries
RES:       4823       5102       4931       5287   Rescheduling interrupts
CAL:        892        823        901        844   Function call interrupts
TLB:       3241       2987       3102       3198   TLB shootdowns
TRM:          0          0          0          0   Thermal event interrupts
THR:          0          0          0          0   Threshold APIC interrupts
MCE:          0          0          0          0   Machine check exceptions
MCP:          1          1          1          1   Machine check polls
...
```

The non-numeric rows at the bottom (NMI, LOC, RES, CAL, TLB, etc.) are particularly useful for isolation diagnostics—`RES` (reschedule IPIs), `CAL` (function call IPIs), and `TLB` (TLB shootdowns) should be near-zero on properly isolated CPUs.

**Monitoring changes over time:**
```bash
watch -n1 -d cat /proc/interrupts
# Or for a specific CPU column (e.g., CPU3):
awk 'NR>1 {print $1, $4}' /proc/interrupts | sort -k2 -rn | head -20
```

### 6.2 `/proc/softirqs`

Analogous to `/proc/interrupts` but for softirq processing counts per CPU:

```bash
cat /proc/softirqs
```

Useful for identifying which softirq types are most active. On an isolated CPU, `SCHED`, `RCU`, and `TIMER` should be minimal with proper configuration.

### 6.3 `perf stat` and `perf record` (IRQ Tracepoints)

`perf` can record both hardware PMU events and kernel tracepoints:

```bash
# List available IRQ tracepoints
perf list 'irq:*'
perf list 'irq_vectors:*'   # x86 APIC vector events

# Count all IRQ handler invocations system-wide for 5 seconds
perf stat -e 'irq:irq_handler_entry' -a sleep 5

# Count specific softirq types
perf stat -e 'irq:softirq_entry,irq:softirq_exit' -a sleep 5

# Count TLB shootdowns
perf stat -e 'tlb:tlb_flush' -a sleep 5

# Record IRQ handler entry/exit with stacks for 10 seconds
perf record -e 'irq:irq_handler_entry,irq:irq_handler_exit' -ag -o irq.perf.data sleep 10
perf report -i irq.perf.data

# Count reschedule and call-function IPIs on CPU 4
perf stat -e 'irq_vectors:reschedule_entry,irq_vectors:call_function_entry' -C 4 sleep 5

# New 'perf irq' subcommand (added ~5.12):
perf irq record -- sleep 5
perf irq report
```

**Key tracepoints for isolation diagnosis:**
- `irq:irq_handler_entry/exit` — Hardware IRQ entry/exit
- `irq:softirq_entry/exit/raise` — Softirq lifecycle
- `irq_vectors:reschedule_entry` — Reschedule IPIs received
- `irq_vectors:call_function_entry` — Function-call IPIs (smp_call_function)
- `irq_vectors:tlb_flush` — TLB flush IPIs
- `irq_vectors:local_timer_entry` — Local APIC timer fires
- `tlb:tlb_flush` — TLB flush events

### 6.4 `ftrace` (Function Tracer)

ftrace provides fine-grained kernel tracing without external tools:

```bash
cd /sys/kernel/debug/tracing

# Enable IRQ handler tracing
echo irq_handler_entry >> set_event
echo irq_handler_exit >> set_event
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace | head -100

# Trace only CPU 4
echo 4 > tracing_cpumask

# Use irqsoff tracer to find longest IRQ-disabled section
echo irqsoff > current_tracer
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace

# Use preemptirqsoff for combined preempt+IRQ latency
echo preemptirqsoff > current_tracer

# Trace TLB flush functions
echo flush_tlb_mm_range > set_ftrace_filter
echo function > current_tracer
```

`trace-cmd` wraps ftrace for convenience:
```bash
trace-cmd record -e irq:irq_handler_entry -e irq:irq_handler_exit sleep 5
trace-cmd report
```

The Real-Time Linux Analysis (RTLA) tool (merged in 5.17, part of kernel-tools):
```bash
rtla timerlat hist -c 4   # Measure timer latency on CPU 4
rtla osnoise top          # OS noise (interrupt + scheduling overhead) per CPU
```

### 6.5 `turbostat`

`turbostat` (part of `linux-tools` / `kernel-tools`) reports per-CPU power and frequency statistics, including interrupt counts:

```bash
# Show IRQ count per CPU, refresh every second
turbostat --show IRQ --interval 1

# Full output with C-states, frequencies, and IRQs
turbostat --interval 5
```

The `IRQ` column shows total interrupts per CPU per interval—a quick way to confirm that isolated CPUs have low interrupt rates.

### 6.6 `cyclictest`

`cyclictest` (from `rt-tests`) measures scheduling latency by measuring the deviation between requested and actual wakeup times of a high-priority task:

```bash
# Basic SMP test, all CPUs, SCHED_FIFO:80, 100 µs interval
cyclictest --mlockall --smp --priority=80 --interval=100 --histogram=200 --duration=60s

# Measure latency on isolated CPU 4 only
cyclictest --mlockall --affinity=4 --priority=99 --interval=100 --duration=60s --quiet

# With nanosecond output
cyclictest -p 99 -t 1 -i 100 -l 100000 -n  # -n = use nanosecond clock
```

High latency values (>100 µs on an untuned system, >10 µs on a tuned system, >1 µs with isolation + RT) indicate interrupt or scheduling noise from the sources described in §4.

### 6.7 `irqtop` and `irqstat`

```bash
# irqtop: top-like view of interrupts (from util-linux ≥2.38)
irqtop

# irqstat: per-CPU IRQ rates (from sysstat)
irqstat 1     # 1-second refresh

# watch /proc/interrupts with diff highlighting
watch -n0.5 -d 'cat /proc/interrupts'
```

### 6.8 `tuna`

`tuna` (Red Hat tuning assistant) provides a graphical/CLI interface for managing IRQ affinity and thread priorities:

```bash
# Show current IRQ and thread placement
tuna --show_irqs
tuna --show_threads

# Move IRQ 24 to CPU 0
tuna --irqs=24 --cpus=0 --move

# Move all IRQs away from CPUs 4-7
tuna --irqs='*' --cpus=!4-7 --move
```

### 6.9 `sar` / `mpstat` (sysstat)

```bash
# Per-CPU interrupt rate (1-second samples)
mpstat -P ALL 1

# interrupt/s column shows hardware + software IRQ rate per CPU
sar -I ALL 1
```

### 6.10 Summary Diagnostic Workflow

For investigating interrupt noise on an isolated CPU (example: CPU 4):

```bash
# Step 1: Baseline interrupt counts
cat /proc/interrupts | awk '{print $1, $5}' > before.txt
sleep 10
cat /proc/interrupts | awk '{print $1, $5}' > after.txt
diff before.txt after.txt

# Step 2: Check for TLB, reschedule, call-function IPIs on CPU 4
grep -E "TLB|RES|CAL" /proc/interrupts

# Step 3: Trace what is hitting CPU 4
perf stat -e 'irq:*,irq_vectors:*' -C 4 sleep 5

# Step 4: If noise found, trace which kernel function triggered it
trace-cmd record -e irq:irq_handler_entry -C 4 sleep 5
trace-cmd report | head -50

# Step 5: Measure actual scheduling latency
cyclictest --affinity=4 --priority=99 --interval=100 --duration=30s

# Step 6: Check IRQ affinity
for d in /proc/irq/*/smp_affinity_list; do
    echo "$d: $(cat $d)"
done | grep -v "^$" | awk -F: '{if ($2 ~ /4/) print $1}'
```

---

## References and Sources

- [The irq_domain Interrupt Number Mapping Library — Linux Kernel Docs](https://docs.kernel.org/core-api/irq/irq-domain.html)
- [Linux generic IRQ handling — Linux Kernel Docs](https://docs.kernel.org/core-api/genericirq.html)
- [NO_HZ: Reducing Scheduling-Clock Ticks — Linux Kernel Docs](https://docs.kernel.org/timers/no_hz.html)
- [Reducing OS jitter due to per-cpu kthreads — Linux Kernel Docs](https://www.kernel.org/doc/html/v5.7/admin-guide/kernel-per-CPU-kthreads.html)
- [Softlockup and hardlockup detector — Linux Kernel Docs](https://docs.kernel.org/admin-guide/lockup-watchdogs.html)
- [A full task-isolation mode for the kernel — LWN.net](https://lwn.net/Articles/816298/)
- [CPU Isolation – Nohz_full – by SUSE Labs (part 3)](https://www.suse.com/c/cpu-isolation-nohz_full-part-3/)
- [CPU Isolation – A practical example – by SUSE Labs (part 5)](https://www.suse.com/c/cpu-isolation-practical-example-part-5/)
- [Low Latency Tuning Guide — Erik Rigtorp](https://rigtorp.se/low-latency-guide/)
- [Linux kernel sched/isolation.c — torvalds/linux on GitHub](https://github.com/torvalds/linux/blob/master/kernel/sched/isolation.c)
- [Linux kernel include/linux/sched/isolation.h — torvalds/linux on GitHub](https://github.com/torvalds/linux/blob/master/include/linux/sched/isolation.h)
- [arch/x86/include/asm/irq_vectors.h — torvalds/linux on GitHub](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irq_vectors.h)
- [Softirq, Tasklets and Workqueues — linux-insides](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)
- [Advanced Programmable Interrupt Controller — Wikipedia](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
- [CachyOS Kernel Documentation](https://wiki.cachyos.org/features/kernel/)
- [CachyOS linux-cachyos GitHub Repository](https://github.com/CachyOS/linux-cachyos)
- [Towards Deterministic Sub-0.5 µs Response on Linux through Interrupt Isolation — arxiv.org](https://arxiv.org/html/2509.03855)
- [IRQ, ACPI and APIC and the Linux kernel — rigacci.org](https://www.rigacci.org/wiki/doku.php/doc/appunti/linux/sa/irq_acpi_apic)
- [IRQs: the Hard, the Soft, the Threaded and the Preemptible — Alison Chaiken, ELCE 2016](https://events.static.linuxfound.org/sites/events/files/slides/Chaiken_ELCE2016.pdf)
- [Understanding Interrupts, Softirqs, and Softnet in Linux — Netdata](https://www.netdata.cloud/blog/understanding-interrupts-softirqs-and-softnet-in-linux/)
- [Real-Time Linux Analysis (RTLA) Tool — kernel.org](https://docs.kernel.org/tools/rtla/rtla.html)
- [MSI-X – the right way to spread interrupt load — Alex on Linux](http://www.alexonlinux.com/msi-x-the-right-way-to-spread-interrupt-load)
- [TLB Shootdowns: How To Deter or Disarm Them — JabPerf Corp](https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/)
- [PREEMPT_RT — Wikipedia](https://en.wikipedia.org/wiki/PREEMPT_RT)
- [Linux Kernel Tracepoint API](https://docs.kernel.org/core-api/tracepoint.html)
- [vmstat: skip periodic vmstat update for isolated CPUs — Patchew](https://patchew.org/linux/ZIDoV._2FzxFKVmQl7W@tpad/)
