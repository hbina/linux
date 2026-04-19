# Machine-First Kernel Study Plan

Goal: study the kernel through the hardware and drivers that this machine
actually uses, with an emphasis on debugging bugs, understanding performance,
and building enough subsystem knowledge to make targeted fixes.

This plan is intentionally practical. The existing
`docs/linux/kernel-foundations-study-plan.md` remains the background/reference
plan, but the default reading order should now be driven by the active hardware
on this machine:

- wired NIC: `r8169`
- Wi-Fi: `iwlwifi` + `iwlmvm`
- GPU / display: `amdgpu` + DRM core
- storage: `nvme`, block layer, page cache, `ext4`
- power / latency: `acpi-cpufreq`, `acpi_idle`
- cross-cutting infrastructure: IRQs, softirqs, workqueues, scheduling

Use the foundations plan just in time: when a driver path depends on a data
structure or primitive you do not fully understand, jump to that topic there,
then return to the machine-first path.

Track high-level completion in `docs/PROGRESS.md`.

---

## Phase 0 — Machine Inventory and Runtime Map

Before changing code, know exactly what hardware and runtime stack you are
dealing with on this machine.

- [ ] Record the active runtime mapping in a notes file under `docs/linux/machine/`: network interfaces, GPU, block devices, root filesystem, cpufreq driver, idle driver, loaded modules.
- [ ] Confirm the wired NIC driver path from interface to driver: `/sys/class/net/enp2s0f0` -> `r8169`.
- [ ] Confirm the Wi-Fi driver path from interface to driver: `/sys/class/net/wlp3s0` -> `iwlwifi`.
- [ ] Confirm the active DRM driver from `/sys/class/drm/card*/device/driver` -> `amdgpu`.
- [ ] Confirm the root storage path: `nvme0n1` -> partition -> `/` on `ext4`.
- [ ] Answer: which parts of this machine are likely latency-sensitive or performance-sensitive for you personally? Rank networking, graphics/display, storage, and power management.

---

## Phase 1 — Wired Networking: `r8169` and the Generic RX/TX Path

Start here if you want the shortest path to practical driver work without
immediately taking on Wi-Fi firmware complexity.

- [ ] Read `drivers/net/ethernet/realtek/r8169_main.c`: find probe/remove, NAPI registration, interrupt setup, and RX/TX ring allocation.
- [ ] Trace one RX packet end to end for `r8169`: interrupt/NAPI entry -> RX descriptor handling -> `napi_gro_receive()` / `netif_receive_skb()` -> protocol demux.
- [ ] Read `include/linux/skbuff.h` at a high level only: understand `head`, `data`, `tail`, `end`, `len`, `data_len`, and header offsets well enough to follow the driver.
- [ ] Read `net/core/dev.c`: `netif_receive_skb_internal()` and identify where driver-specific work ends and generic networking starts.
- [ ] Read one TX path through `r8169`: queueing an skb, descriptor preparation, DMA doorbell, completion cleanup.
- [ ] Answer: which parts of the path are driver-specific bugs versus generic stack bugs? Give three examples of each.
- [ ] Measure something real on this machine: packet drops, RX/TX interrupt rate, coalescing, or throughput. Write a short note describing what you measured and which kernel files would matter if the result were bad.

---

## Phase 2 — Wi-Fi: `iwlwifi` / `iwlmvm`

Wi-Fi is less attractive as a first modification target than `r8169`, but it is
still valuable because the machine actually uses it.

- [ ] Read the high-level split between `iwlwifi` transport code and `iwlmvm` policy/mac80211 integration.
- [ ] Identify probe, firmware load, and netdev/mac80211 registration entry points in the Intel Wi-Fi stack.
- [ ] Trace one receive path at a high level from hardware notification into mac80211.
- [ ] Read just enough `net/mac80211/` to understand where device-specific logic hands off to common Wi-Fi code.
- [ ] Answer: why is Wi-Fi debugging generally harder than `r8169` debugging on the same machine? Be specific about firmware, rate control, aggregation, regulatory state, and mac80211 layering.
- [ ] Pick one practical symptom you might care about on this machine: reconnect failures, suspend/resume regression, throughput drop, or high interrupt/CPU cost. Identify the first 5 files you would inspect.

---

## Phase 3 — GPU / Display: `amdgpu` and DRM

This overlaps directly with your existing investigation notes and is the best
fit if your practical goal is display bugs or graphics latency.

- [ ] Read `drivers/gpu/drm/amd/amdgpu/` at the top level: identify PCI probe, IP block bring-up, interrupt handling, and scheduler submission layers.
- [ ] Read enough DRM core to place `amdgpu` in context: `drm_file`, `drm_ioctl`, GEM/TTM, modesetting, vblank, and atomic commit.
- [ ] Revisit your existing note `docs/linux/gpu/flip_done_timeout_amdgpu.md` and map each important function in the note to the current source tree.
- [ ] Trace one atomic modeset/page-flip path from userspace ioctl to `amdgpu` display code and back to completion/vblank signaling.
- [ ] Trace one GPU job submission path at a high level: ioctl -> scheduler -> ring/queue submission -> fence completion.
- [ ] Answer: for the bugs you are most likely to care about on this machine, where is the boundary between DRM core and `amdgpu` driver code?
- [ ] Write a short “AMDGPU bug triage checklist” note: what logs, tracepoints, debugfs, and source files you would inspect first for display, hang, and performance issues.

---

## Phase 4 — Storage: `nvme`, Block Layer, Page Cache, `ext4`

This is the best path if you care about boot speed, I/O latency, page-cache
behavior, or filesystem correctness/performance on the actual root disk.

- [ ] Read `drivers/nvme/host/`: identify probe, queue setup, request submission, completion, and timeout/error recovery entry points.
- [ ] Trace one read request from VFS/page cache miss -> block layer -> NVMe queue -> completion -> folio uptodate.
- [ ] Read `mm/filemap.c:filemap_read()` and connect it to the `ext4` and NVMe path.
- [ ] Read `fs/ext4/` at a practical level: `ext4_file_read_iter`, `ext4_readahead`, extent lookup, and writeback/journal entry points.
- [ ] Answer: for a slow file read on `/`, how would you distinguish whether the bottleneck is page cache, ext4 metadata lookup, block layer queueing, or NVMe device behavior?
- [ ] Measure something real on this machine: page-cache hit/miss behavior, readahead, read latency, or writeback stalls. Record the measurement and the code path it suggests.

---

## Phase 5 — IRQs, NAPI, Workqueues, Scheduling, and Power

Use this phase to tie together the cross-cutting runtime mechanisms that affect
every active device in this machine.

- [ ] For `r8169`, `iwlwifi`, `amdgpu`, and `nvme`, identify how each device gets from hardware event to IRQ handler to deferred work (threaded IRQ, softirq, NAPI, workqueue, tasklet, or kthread).
- [ ] Re-read `kernel/irq/`, `kernel/softirq.c`, and `workqueue` code only in the places needed to explain those active devices.
- [ ] Read `kernel/sched/` just enough to explain how driver-related kthreads, worker threads, and softirq load affect perceived performance.
- [ ] Read the runtime power side: `acpi-cpufreq`, `acpi_idle`, and the relevant parts of the RT/low-latency plan on C-states and housekeeping.
- [ ] Answer: which of this machine's likely performance complaints would be caused by the device driver itself versus CPU frequency scaling, CPU idle policy, IRQ placement, or scheduler behavior?
- [ ] Perform one practical experiment on this machine involving IRQ affinity, CPUFreq governor settings, or C-state limits, and write down what changed and why you think it changed.

---

## Phase 6 — Bug-Fixing Workflow on Real Hardware

The goal of this phase is to make you effective at changing and validating code
for the machine you actually own.

- [ ] Pick one subsystem to make your primary practical target for the next month: `r8169`, `amdgpu`, `nvme/ext4`, or `iwlwifi`.
- [ ] Write a subsystem-specific debug workflow note: reproduction, logs, tracepoints, dynamic debug, ftrace/perf/bpftrace hooks, and source files to inspect first.
- [ ] Build and boot a local kernel with one safe instrumentation-only change in that subsystem, and verify that you can observe the behavior you care about.
- [ ] Make one small code change that improves observability or fixes a local annoyance, and document exactly how you validated it.
- [ ] Answer: what class of bugs are you now prepared to work on in this subsystem, and what classes are still too opaque?

---

## Recommended Reading Order

Unless a concrete bug forces a different order, use this:

1. Phase 0 — inventory and runtime map
2. Phase 1 — `r8169` and generic networking
3. Phase 3 — `amdgpu` / DRM
4. Phase 4 — `nvme` + page cache + `ext4`
5. Phase 5 — power / IRQ / scheduling cross-cuts
6. Phase 2 — `iwlwifi` / `iwlmvm`
7. Phase 6 — bug-fixing workflow and code changes

Why this order:

- `r8169` is a simpler practical driver target than Wi-Fi.
- `amdgpu` directly overlaps with work you have already done.
- `nvme` + `ext4` covers the root filesystem you use every day.
- power/IRQ/scheduler knowledge becomes useful once you are looking at real regressions.
- Wi-Fi is important, but more layered and firmware-heavy.
