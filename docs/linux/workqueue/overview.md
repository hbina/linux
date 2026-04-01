# Linux Kernel Workqueue Subsystem

The workqueue subsystem (`kernel/workqueue.c`) is the kernel's generic mechanism
for deferring work to be executed later in **process context**. It is one of the
most widely used subsystems in the kernel — nearly every driver and subsystem
relies on it.

---

## 1. The Problem It Solves

Many parts of the kernel need to perform work that:

- **Cannot happen right now** — e.g. you're inside a hard IRQ handler and cannot
  sleep, allocate memory with `GFP_KERNEL`, or take a mutex.
- **Must happen in process context** — not in softirq or hard IRQ context.

The solution is to package the work as a `struct work_struct` with a function
pointer and hand it off to a kernel worker thread to execute later.

```
Hard IRQ fires
  → driver ISR runs (cannot sleep)
      → queue_work(wq, &my_work)   ← just enqueues, returns immediately

Later, a kworker thread wakes up
  → my_work_fn(&my_work)           ← runs in process context, can sleep
```

---

## 2. Execution Contexts

The kernel has several distinct execution contexts. Understanding them is
essential for knowing when workqueue is the right tool.

| Context | Can sleep? | `current` valid? | Example |
|---------|-----------|-----------------|---------|
| Process context | Yes | Yes | syscall, kernel thread |
| Softirq context | No | No (meaningless) | `net_rx_action`, timers |
| Hard IRQ context | No | No (meaningless) | Hardware interrupt handler |
| NMI context | No | No | Watchdog, perf sampling |

"Process context" means a real kernel thread (or a task running a syscall) is
executing. The CPU is running on behalf of that task — it has a kernel stack,
`current` is valid, and calling `schedule()` is legal.

A hard IRQ handler runs on top of whatever task happened to be interrupted. There
is no task to put to sleep. Calling `mutex_lock()` or `kmalloc(GFP_KERNEL)` from
hard IRQ context will either BUG immediately or corrupt state.

---

## 3. Worker Pools

The workqueue subsystem manages a set of kernel threads called **workers**
(`kworker/N:M`). These are grouped into **pools**.

```c
// kernel/workqueue.c
static cpumask_var_t wq_unbound_cpumask;   // where unbound workers may run
static cpumask_var_t wq_isolated_cpumask;  // isolated CPUs excluded from above
```

There are two kinds of pools:

### Bound pools (per-CPU)

Each CPU has two bound pools — one normal priority, one high priority. Workers in
a bound pool are pinned to their CPU. They are named `kworker/N:M` where `N` is
the CPU number.

```
CPU 0: [kworker/0:0] [kworker/0:1H]   ← normal and high-priority
CPU 1: [kworker/1:0] [kworker/1:1H]
...
```

### Unbound pools

Unbound workers are not tied to a specific CPU (`kworker/uN:M`). These are used
by workqueues created with `WQ_UNBOUND`. The pool grows and shrinks dynamically
based on demand.

---

## 4. Key Data Structures

| Structure | Purpose |
|-----------|---------|
| `struct work_struct` | The unit of deferred work — function pointer + flags |
| `struct delayed_work` | A `work_struct` with an attached timer |
| `struct worker_pool` | A set of worker threads + pending work list for one CPU/priority |
| `struct workqueue_struct` | A named queue that routes submitted work to a pool |
| `struct worker` | One kworker thread; picks items off the pool list and runs them |

The relationship:

```
workqueue_struct  →  pool_workqueue  →  worker_pool  →  worker threads
     (API handle)      (per-cpu glue)    (shared pool)    (kworker/N:M)
```

---

## 5. Common API

```c
// Static declaration
DECLARE_WORK(my_work, my_work_fn);

// Dynamic declaration
struct work_struct my_work;
INIT_WORK(&my_work, my_work_fn);

// Queue onto the global system_wq (most common)
schedule_work(&my_work);

// Queue with a delay
schedule_delayed_work(&my_dwork, msecs_to_jiffies(100));

// Queue onto a specific workqueue
queue_work(my_wq, &my_work);

// Wait for all currently queued work to complete
flush_work(&my_work);
flush_workqueue(my_wq);
```

The global `system_wq` is suitable for short, non-blocking work items. For work
that may sleep for a long time or is CPU-intensive, a dedicated workqueue with
appropriate flags (`WQ_UNBOUND`, `WQ_CPU_INTENSIVE`) is preferred.

---

## 6. CPU Isolation and Workqueues

When a system is configured for real-time or low-latency use, CPUs are isolated
with `isolcpus=` or `nohz_full=` kernel parameters. The workqueue subsystem
has explicit support for this.

### Bound workers on isolated CPUs

Bound kworker threads **always exist** on every CPU, including isolated ones.
They cannot be prevented from existing. However, on an isolated CPU they sit idle
— nothing queues work to them unless a driver explicitly targets that CPU.

The kernel is aware of this tension. From `kernel/workqueue.c:2958`:

```c
/*
 * We don't want to disturb isolated CPUs because of a pcpu kworker being
 * culled, so this also resets worker affinity.
 */
```

### Unbound workers are automatically excluded

`kernel/sched/isolation.c` describes its own purpose:

> Manage the targets for routine code that can run on any CPU: unbound
> workqueues, timers, kthreads and any offloadable work.

Non-isolated CPUs are called **housekeeping CPUs**. When isolation parameters
change, `workqueue_unbound_housekeeping_update()` recalculates:

```
wq_unbound_cpumask = requested_cpumask ∩ housekeeping_cpumask
```

Unbound workers are automatically confined to non-isolated CPUs. This is
observable at runtime:

```bash
cat /sys/devices/virtual/workqueue/cpumask           # where unbound workers run
cat /sys/devices/virtual/workqueue/cpumask_isolated  # isolated CPUs excluded
```

### Summary

| Worker type | Behaviour with `isolcpus=` |
|-------------|---------------------------|
| Bound (`kworker/N:M`) | Exists on isolated CPU but sits idle |
| Unbound (`kworker/uN:M`) | Automatically excluded via `wq_unbound_cpumask` |
| Your own `kthread` | Runs wherever `kthread_bind()` places it — fully under your control |

For true RT isolation, `nohz_full=` is also needed — it stops the scheduler tick
on isolated CPUs, preventing even brief bound-worker wakeups.

---

## 7. Connection to `list_for_each_entry_safe`

The workqueue internally maintains lists of workers and pending work items. When
idle workers are reaped, the kernel must delete entries from a list while
iterating over it — exactly the scenario `list_for_each_entry_safe` exists for:

```c
// kernel/workqueue.c
static void reap_dying_workers(struct list_head *cull_list)
{
    struct worker *worker, *tmp;

    list_for_each_entry_safe(worker, tmp, cull_list, entry) {
        list_del_init(&worker->entry);
        kthread_stop_put(worker->task);
        kfree(worker);
    }
}
```

`tmp` pre-saves the next `worker` before the loop body runs. After
`list_del_init(&worker->entry)` the `entry` field is reset to a self-loop, so
the iterator cannot follow it. `tmp` provides the escape route to the real next
element.

---

## References

- `kernel/workqueue.c`
- `kernel/workqueue_internal.h`
- `kernel/sched/isolation.c`
- `include/linux/workqueue.h`
- `Documentation/core-api/workqueue.rst`
