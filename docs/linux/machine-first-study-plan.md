# Machine-First Kernel Study Plan

Goal: study the kernel through the hardware and drivers that this machine
actually uses, with an emphasis on debugging bugs, understanding performance,
and building enough subsystem knowledge to make targeted fixes.

This plan is intentionally practical. The existing
`docs/linux/kernel-foundations-study-plan.md` remains the background/reference
plan, but the default reading order should now be driven by the active hardware
on this machine:

- CPU / SoC: AMD Ryzen AI MAX+ 395 ("Strix Halo" APU, 32 threads)
- wired NIC: `r8169` driving Realtek RTL8125 2.5GbE (`eno1`, PCI `c1:00.0`)
- Wi-Fi: MediaTek MT7925 (`14c3:0717`, PCI `c3:00.0`) — kernel driver `mt7925e` / `mt76` (currently unbound on this kernel; no `wlan*` interface present)
- GPU / display: `amdgpu` + DRM core for integrated Radeon 8060S (RDNA 3.5, `1002:1586`); HDMI + 8 DisplayPort outputs exposed
- storage:
  - root: `nvme1n1p2` (Phison `5029`) `ext4` at `/`, `nvme1n1p1` is the EFI vfat at `/boot/efi`
  - secondary NVMe `nvme0n1` (Sandisk SN580) and SATA `sda`/`sdb` are ZFS (out-of-tree, **not** part of in-tree study)
- power / latency: `amd-pstate-epp` cpufreq driver, `acpi_idle` cpuidle driver
- audio: `snd_hda_intel` (AMD Rembrandt HDA + on-board codec)
- cross-cutting infrastructure: IRQs, softirqs, workqueues, scheduling

Use the foundations plan just in time: when a driver path depends on a data
structure or primitive you do not fully understand, jump to that topic there,
then return to the machine-first path.

Track high-level completion in `docs/PROGRESS.md`.

---

## Phase 0 — Machine Inventory and Runtime Map

Before changing code, know exactly what hardware and runtime stack you are
dealing with on this machine.

- [ ] Record the active runtime mapping in a notes file under `docs/linux/machine/`: network interfaces, GPU, block devices, root filesystem, cpufreq driver, idle driver, loaded modules. Inputs to capture: `lspci -nnk`, `lsmod`, `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT`, `findmnt /`, `/sys/devices/system/cpu/cpu0/cpufreq/scaling_driver`, `/sys/devices/system/cpu/cpuidle/current_driver`, `uname -r`.
- [ ] Confirm the wired NIC driver path from interface to driver: `/sys/class/net/eno1/device/driver` -> `r8169`; PCI device is `c1:00.0` (RTL8125 2.5GbE, `10ec:8125`).
- [ ] Confirm the Wi-Fi situation: PCI `c3:00.0` is MediaTek `14c3:0717` (MT7925) but no kernel driver is currently bound and no `/sys/class/ieee80211/` entry exists. Decide whether to enable `CONFIG_MT7925E` in the next build, or treat Wi-Fi as out-of-scope on this machine for now.
- [ ] Confirm the active DRM driver: `/sys/class/drm/card0/device/driver` -> `amdgpu`. Note this is an integrated APU (RDNA 3.5 / Strix Halo, `1002:1586`), not a discrete GPU — bring-up paths differ from dGPUs.
- [ ] Confirm the root storage path: `nvme1n1` -> `nvme1n1p2` -> `/` on `ext4` (Phison `5029` controller). Note that `nvme0n1`, `sda`, `sdb` are ZFS pool members and out of scope for in-tree study.
- [ ] Confirm cpufreq/idle: `scaling_driver` = `amd-pstate-epp`, `cpuidle/current_driver` = `acpi_idle`. Both differ from the legacy `acpi-cpufreq` assumption — relevant for Phase 5 reading.
- [ ] Answer: which parts of this machine are likely latency-sensitive or performance-sensitive for you personally? Rank networking, graphics/display, storage, and power management.

---

## Phase 1 — Wired Networking: `r8169` (RTL8125 2.5GbE) and the Generic RX/TX Path

Start here if you want the shortest path to practical driver work without
immediately taking on Wi-Fi firmware complexity. The active chip on this
machine is the RTL8125 (`10ec:8125`, rev 05) — within `r8169_main.c` look
for the `RTL_GIGA_MAC_VER_*` chip-version branches that cover the 8125
family rather than the older 8168/8169 paths.

- [ ] Read `drivers/net/ethernet/realtek/r8169_main.c`: find probe/remove, NAPI registration, interrupt setup, and RX/TX ring allocation. Note the chip-version dispatch — the RTL8125 takes a different code path from the older 8168/8169.
- [ ] Trace one RX packet end to end for `r8169`: MSI-X / NAPI entry -> RX descriptor handling -> `napi_gro_receive()` / `netif_receive_skb()` -> protocol demux. The 8125 uses MSI-X with multiple queues — confirm queue count via `ethtool -l eno1`.
- [ ] Read `include/linux/skbuff.h` at a high level only: understand `head`, `data`, `tail`, `end`, `len`, `data_len`, and header offsets well enough to follow the driver.
- [ ] Read `net/core/dev.c`: `netif_receive_skb_internal()` and identify where driver-specific work ends and generic networking starts.
- [ ] Read one TX path through `r8169`: queueing an skb, descriptor preparation, DMA doorbell, completion cleanup.
- [ ] Answer: which parts of the path are driver-specific bugs versus generic stack bugs? Give three examples of each.
- [ ] Measure something real on this machine: packet drops, RX/TX interrupt rate, coalescing, or throughput. Write a short note describing what you measured and which kernel files would matter if the result were bad.

---

## Phase 2 — Wi-Fi: `mt76` / `mt7925e` (MediaTek MT7925)

This machine has a MediaTek MT7925 (`14c3:0717`) at PCI `c3:00.0`, **not** an
Intel adapter. The relevant in-tree drivers live under
`drivers/net/wireless/mediatek/mt76/` (shared bus/HW abstraction) with the
device-specific code in `mt76/mt7925/`. Wi-Fi is less attractive as a first
modification target than `r8169`, but it is the only wireless device on this
machine.

Note: as of the current running kernel, no `mt7925e` module appears bound and
no `wlan*` interface exists. Phase 0 should decide whether to enable
`CONFIG_MT7925E` (and any prerequisite `CONFIG_MT792x_*` symbols) before this
phase becomes practical.

- [ ] Read the high-level split between `mt76/` (bus/HW abstraction shared across MediaTek chips) and `mt76/mt7925/` (device-specific PCIe/USB driver) and `mt76/mt792x_*` (shared 7921/7922/7925 logic).
- [ ] Identify probe, firmware load (look for `mt7925_load_firmware*`), and netdev/mac80211 registration entry points.
- [ ] Trace one receive path at a high level from hardware notification (MSI/MSI-X) into mac80211.
- [ ] Read just enough `net/mac80211/` to understand where device-specific logic hands off to common Wi-Fi code.
- [ ] Answer: why is Wi-Fi debugging generally harder than `r8169` debugging on the same machine? Be specific about firmware blobs (where `mt7925` firmware lives in `linux-firmware`), rate control, aggregation, regulatory state, and mac80211 layering.
- [ ] Pick one practical symptom you might care about on this machine: driver-bind failure (current state), reconnect failures, suspend/resume regression, throughput drop, or high interrupt/CPU cost. Identify the first 5 files you would inspect.

---

## Phase 3 — GPU / Display: `amdgpu` and DRM (Strix Halo APU, RDNA 3.5)

This overlaps directly with your existing investigation notes and is the best
fit if your practical goal is display bugs or graphics latency.

The GPU on this machine is the integrated Radeon 8060S in the Ryzen AI MAX+
395 (`1002:1586`, "Strix Halo"). Because it is an APU sharing system memory,
**TTM behavior, GTT vs VRAM placement, and the unified-memory paths are more
relevant than discrete-GPU VRAM management**. Display outputs include 1×
HDMI-A and 8× DisplayPort connectors plus a writeback connector — non-trivial
for atomic-commit and hotplug paths.

- [ ] Read `drivers/gpu/drm/amd/amdgpu/` at the top level: identify PCI probe, IP block bring-up (look for the `*_ip_block` arrays specific to the GFX11.5 / RDNA 3.5 IP version), interrupt handling, and scheduler submission layers.
- [ ] Read enough DRM core to place `amdgpu` in context: `drm_file`, `drm_ioctl`, GEM/TTM (especially the APU-relevant `TTM_PL_TT` vs `TTM_PL_VRAM` distinction), modesetting, vblank, and atomic commit.
- [ ] Revisit your existing note `docs/linux/gpu/flip_done_timeout_amdgpu.md` and map each important function in the note to the current source tree.
- [ ] Trace one atomic modeset/page-flip path from userspace ioctl to `amdgpu` display code and back to completion/vblank signaling.
- [ ] Trace one GPU job submission path at a high level: ioctl -> scheduler -> ring/queue submission -> fence completion.
- [ ] Answer: for the bugs you are most likely to care about on this machine, where is the boundary between DRM core and `amdgpu` driver code?
- [ ] Write a short “AMDGPU bug triage checklist” note: what logs, tracepoints, debugfs, and source files you would inspect first for display, hang, and performance issues.

---

## Phase 4 — Storage: `nvme`, Block Layer, Page Cache, `ext4`

This is the best path if you care about boot speed, I/O latency, page-cache
behavior, or filesystem correctness/performance on the actual root disk.

Layout on this machine: root is `nvme1n1p2` (`ext4`) on a Phison `5029`
controller; `nvme1n1p1` is the EFI vfat partition. The other disks
(`nvme0n1` Sandisk SN580, `sda`/`sdb` SATA) are ZFS pool members and out of
scope for in-tree filesystem study (ZFS is out-of-tree). Use the in-tree
nvme/block/ext4 path on the Phison NVMe for all measurements below.

- [ ] Read `drivers/nvme/host/`: identify probe, queue setup, request submission, completion, and timeout/error recovery entry points. Note the Phison controller is DRAM-equipped; the Sandisk SN580 is DRAM-less (relevant if you later compare them).
- [ ] Trace one read request from VFS/page cache miss -> block layer (`blk-mq`) -> NVMe queue -> completion -> folio uptodate.
- [ ] Read `mm/filemap.c:filemap_read()` and connect it to the `ext4` and NVMe path.
- [ ] Read `fs/ext4/` at a practical level: `ext4_file_read_iter`, `ext4_readahead`, extent lookup, and writeback/journal entry points.
- [ ] Answer: for a slow file read on `/`, how would you distinguish whether the bottleneck is page cache, ext4 metadata lookup, block layer queueing, or NVMe device behavior?
- [ ] Measure something real on this machine: page-cache hit/miss behavior, readahead, read latency, or writeback stalls on `nvme1n1p2`. Record the measurement and the code path it suggests.

---

## Phase 5 — IRQs, NAPI, Workqueues, Scheduling, and Power

Use this phase to tie together the cross-cutting runtime mechanisms that affect
every active device in this machine.

- [ ] For `r8169`, `mt7925e` (if enabled), `amdgpu`, and `nvme`, identify how each device gets from hardware event to IRQ handler to deferred work (threaded IRQ, softirq, NAPI, workqueue, tasklet, or kthread).
- [ ] Re-read `kernel/irq/`, `kernel/softirq.c`, and `workqueue` code only in the places needed to explain those active devices.
- [ ] Read `kernel/sched/` just enough to explain how driver-related kthreads, worker threads, and softirq load affect perceived performance on a 32-thread Strix Halo CPU.
- [ ] Read the runtime power side: `drivers/cpufreq/amd-pstate.c` (this machine uses `amd-pstate-epp`, **not** `acpi-cpufreq`), `drivers/cpuidle/` + `acpi_idle`, and the relevant parts of the RT/low-latency plan on C-states and housekeeping. Understand the EPP (Energy-Performance Preference) hint and how it is exposed via `/sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference`.
- [ ] Answer: which of this machine's likely performance complaints would be caused by the device driver itself versus CPU frequency scaling (amd-pstate EPP), CPU idle policy, IRQ placement, or scheduler behavior?
- [ ] Perform one practical experiment on this machine involving IRQ affinity, `amd-pstate` EPP / governor settings, or C-state limits, and write down what changed and why you think it changed.

---

## Phase 6 — Bug-Fixing Workflow on Real Hardware

The goal of this phase is to make you effective at changing and validating code
for the machine you actually own.

- [ ] Pick one subsystem to make your primary practical target for the next month: `r8169`, `amdgpu`, `nvme/ext4`, or `mt7925e` (note: `mt7925e` requires first enabling the driver in your build, since it is not currently bound).
- [ ] Write a subsystem-specific debug workflow note: reproduction, logs, tracepoints, dynamic debug, ftrace/perf/bpftrace hooks, and source files to inspect first.
- [ ] Build and boot a local kernel with one safe instrumentation-only change in that subsystem, and verify that you can observe the behavior you care about.
- [ ] Make one small code change that improves observability or fixes a local annoyance, and document exactly how you validated it.
- [ ] Answer: what class of bugs are you now prepared to work on in this subsystem, and what classes are still too opaque?

---

## Recommended Reading Order

Unless a concrete bug forces a different order, use this:

1. Phase 0 — inventory and runtime map
2. Phase 1 — `r8169` (RTL8125) and generic networking
3. Phase 3 — `amdgpu` / DRM (Strix Halo APU)
4. Phase 4 — `nvme` + page cache + `ext4` (root on Phison NVMe)
5. Phase 5 — power / IRQ / scheduling cross-cuts (`amd-pstate-epp`, `acpi_idle`)
6. Phase 2 — `mt7925e` / `mt76` (only after enabling the driver)
7. Phase 6 — bug-fixing workflow and code changes

Why this order:

- `r8169` is a simpler practical driver target than Wi-Fi.
- `amdgpu` directly overlaps with work you have already done, and the APU/unified-memory characteristics make it more interesting than a stock dGPU study.
- `nvme` + `ext4` covers the root filesystem you use every day.
- power/IRQ/scheduler knowledge becomes useful once you are looking at real regressions; `amd-pstate` EPP behavior is non-trivial on Ryzen.
- Wi-Fi (`mt7925e`) is important, but more layered and firmware-heavy, and currently requires enabling the driver before any practical work.
