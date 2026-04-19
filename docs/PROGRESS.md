# Learning Progress Tracker

This file tracks what has been studied, understood, and noted.
Update it after each session. Claude reads this at the start of each conversation
to know where you are and what to build on.

**Last updated:** 2026-04-01

---

## How to Update

After a study session, add an entry under the relevant phase:
- `[x]` = understood, notes written
- `[~]` = partially read, needs revisit
- `[ ]` = not started

Add a one-line note for anything surprising or non-obvious — that's what's
worth remembering across sessions.

---

Detailed concrete checklist is in `docs/linux/kernel-foundations-study-plan.md`.
Each phase below links to the section. Tick a phase only when all items in the
detailed plan are done.

Primary practical plan is now also in `docs/linux/machine-first-study-plan.md`.
Use that plan to guide subsystem order for work on the hardware this machine
actually uses; use the foundations plan as just-in-time background/reference.

## Machine-First Practical Track

- [ ] **M0** Runtime inventory of this machine — NICs, GPU, storage, cpufreq, idle, and module map
- [ ] **M1** Wired networking (`r8169`) + generic RX/TX path
- [ ] **M2** Wi-Fi (`iwlwifi` / `iwlmvm`)
- [ ] **M3** GPU / display (`amdgpu` + DRM)
- [ ] **M4** Storage (`nvme` + page cache + `ext4`)
- [ ] **M5** Cross-cutting runtime behavior — IRQs, softirqs, workqueues, scheduling, power
- [ ] **M6** Real-hardware bug-fixing workflow and local kernel instrumentation

## Phase 1 — Core Data Structures

- [~] **1.1** Intrusive linked lists (`include/linux/list.h`) — 9/12 done; remaining: workqueue.c grep, kernel/pid.c hlist trace, llist.h lock-free add
- [ ] **1.2** Red-black trees (`include/linux/rbtree.h`, `lib/rbtree.c`) — 5 items
- [ ] **1.3** XArrays (`include/linux/xarray.h`) — 6 items
- [ ] **1.4** Hash tables (`include/linux/hashtable.h`) — 4 items

## Phase 2 — Concurrency Primitives

- [ ] **2.1** Spinlocks — 4 items
- [ ] **2.2** Mutexes — 4 items
- [ ] **2.3** RW semaphores — 3 items
- [ ] **2.4** Atomics — 4 items
- [ ] **2.5** Memory barriers (`Documentation/memory-barriers.txt`) — 5 items
- [ ] **2.6** RCU — 8 items

## Phase 3 — Memory Management

- [ ] **3.1** Page allocator (`mm/page_alloc.c`) — 6 items
- [ ] **3.2** SLUB (`mm/slub.c`) — 6 items
- [ ] **3.3** VMAs and page tables (`mm/mmap.c`, `mm/memory.c`) — 7 items
- [ ] **3.4** Page cache and folios (`mm/filemap.c`) — 6 items

## Phase 4 — Process Model and Scheduling

- [ ] **4.1** task_struct (`include/linux/sched.h`, `kernel/fork.c`) — 5 items
- [ ] **4.2** CFS scheduler (`kernel/sched/fair.c`) — 6 items
- [ ] **4.3** RT scheduling classes (`kernel/sched/rt.c`) — 3 items

## Phase 5 — Filesystems

- [ ] **5.1** VFS layer (`Documentation/filesystems/vfs.rst`, `fs/namei.c`) — 7 items
- [ ] **5.2** ramfs (`fs/ramfs/`) — 6 items — read every line
- [ ] **5.3** ext4 (`fs/ext4/`) — 5 items

## Phase 6 — IRQ Subsystem

- [x] **6.1 IRQ descriptor lifecycle** — `kernel/irq/irqdesc.c`, `handle.c`, `manage.c`
  - Notes: `docs/linux/interrupts/linux_interrupt_architecture.md`
  - Notes: `docs/linux/interrupts/INT3515_IRQ_issue_investigation.md`
  - Notes: `docs/linux/interrupts/loc_iwi_interrupt_deep_dive.md`
- [x] **6.2 MSI/MSI-X and APIC** — x86 vector allocation, per-CPU IRQ delivery
  - Covered in interrupt architecture notes above
- [x] **6.3 Softirqs and tasklets** — `kernel/softirq.c`
  - Covered in interrupt architecture notes above

## Phase 7 — Networking Stack

- [x] **7.1 SolarFlare EF100 RX path** — MSI-X → NAPI → `netif_receive_skb_list` → packet socket
  - Notes: `docs/solarflare/rx-interrupt-path.md`
- [ ] **7.2 sk_buff layout** — `include/linux/skbuff.h` in depth
- [ ] **7.3 Generic RX path** — `net/core/dev.c`, `net/ipv4/ip_input.c`, `net/ipv4/tcp_input.c`
- [ ] **7.4 AF_XDP** — `net/xdp/xsk.c`, `net/xdp/xdp_umem.c`

## Phase 8 — io_uring

- [ ] **8.1 Ring setup and core** — `io_uring/io_uring.c`
- [ ] **8.2 SQPOLL** — `io_uring/sqpoll.c`
- [ ] **8.3 Networking ops** — `io_uring/net.c`
- [ ] **8.4 Zero-copy RX** — `io_uring/zcrx.c`

## RT / Low-Latency Plan

Status: plan written, not yet worked through systematically.
Plan file: `docs/linux/realtime/realtime-lowlatency-study-plan.md`

- [ ] Phase 0 — Execution contexts and timer infrastructure
- [ ] Phase 1 — NMIs
- [ ] Phase 2 — SMIs
- [ ] Phase 3 — CPU isolation and IRQ affinity *(partial — covered via interrupt notes)*
- [ ] Phase 4 — RCU offloading
- [ ] Phase 5 — PREEMPT_RT kernel
- [ ] Phase 6 — Power management and C-states
- [ ] Phase 7 — Memory: pre-faulting and huge pages
- [ ] Phase 8 — Measurement tools

---

## Investigation Notes (real-world work)

These came from debugging real systems, not structured study.
They count as practical knowledge even though they skipped prerequisites.

| Date | Topic | Notes file |
|------|-------|------------|
| 2026-03-15 | AMD GPU flip_done timeout | `docs/linux/gpu/flip_done_timeout_amdgpu.md` |
| 2026-03-15 | Kernel crash analysis | `docs/linux/gpu/crash_analysis_20260315.md` |
| ongoing | INT3515 IRQ issue | `docs/linux/interrupts/INT3515_IRQ_issue_investigation.md` |
| ongoing | LOC/IWI interrupt deep dive | `docs/linux/interrupts/loc_iwi_interrupt_deep_dive.md` |
| ongoing | SolarFlare EF100 RX path | `docs/solarflare/rx-interrupt-path.md` |
| ongoing | vng/virtme-ng workflow | `docs/linux/tools/vng_workflow_summary.md` |

---

## Session Log

### 2026-04-01
- Created comprehensive kernel foundations study plan (`docs/linux/kernel-foundations-study-plan.md`)
- Assessed current progress: Phases 6–7 partially done via investigation work; Phases 1–5 not started
- Read `tools/include/linux/list.h` in full — covered list_head, container_of, INIT_LIST_HEAD, list_add/tail, list_del/del_init, list_for_each_entry, list_for_each_entry_safe, hlist with pprev trick
- **Next session:** finish 1.1 — grep workqueue.c for safe iteration, trace pid.c hlist lookup, read llist.h

### 2026-04-19
- Added a machine-first practical study plan (`docs/linux/machine-first-study-plan.md`)
- Reoriented study order toward the drivers and subsystems this machine actually uses: `r8169`, `iwlwifi`/`iwlmvm`, `amdgpu`, `nvme`, `ext4`, `acpi-cpufreq`, `acpi_idle`
- Foundations plan remains the reference plan; machine-first plan is now the default practical path
