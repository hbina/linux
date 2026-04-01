# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is the Linux kernel source tree (currently v7.0-rc5, "Baby Opossum Posse"). The primary goal in this workspace is **read-only exploration and learning** — understanding kernel internals, not modifying the kernel itself. Learnings, investigation notes, and explorations should be written to `docs/` in this repository root.

## Learning Progress

**Always read `docs/PROGRESS.md` at the start of every conversation.**
It tracks which phases of the study plan have been completed, which are in
progress, and what was covered in the last session. Use it to:
- Know what the user already understands (don't re-explain it)
- Know what the next logical topic is
- Calibrate explanation depth to existing knowledge

The structured study plan is in `docs/linux/kernel-foundations-study-plan.md`.
The RT/low-latency plan is in `docs/linux/realtime/realtime-lowlatency-study-plan.md`.

**IMPORTANT — progress updates:**
- **Never mark a checklist item or phase as complete on the user's behalf.**
- Only tick off items (`[ ]` → `[x]`) or update phase status when the user
  **explicitly states** they have completed or understood that item/phase.
- Observing that the user has read a file is not sufficient — wait for the user
  to confirm understanding before marking anything done.
- You may use `[~]` (in progress) when the user is actively working through
  something but has not yet confirmed completion.

---

## Build & Navigation Commands

```bash
# Search kernel symbols
grep -r "function_name" --include="*.c" --include="*.h" .

# Find Kconfig options for a subsystem
grep -r "CONFIG_SFC" --include="Kconfig" .

# Generate cross-reference browsable HTML (requires ctags/cscope)
make tags     # generates ctags
make cscope   # generates cscope database

# Build kernel documentation
make htmldocs  # full HTML docs in Documentation/output/
make pdfdocs   # PDF format

# Build & run kernel self-tests (requires kernel headers)
make headers
make -C tools/testing/selftests
make -C tools/testing/selftests run_tests
# Run specific subsystem tests:
make -C tools/testing/selftests TARGETS="net" run_tests

# Check .config options relevant to a feature
grep CONFIG_SFC .config
```

---

## Repository Architecture

### Top-Level Layout

| Directory | Purpose |
|-----------|---------|
| `arch/` | Architecture-specific code (x86, arm64, riscv, etc.) |
| `kernel/` | Core kernel: scheduling, IRQs, locking, RCU, power |
| `mm/` | Memory management |
| `fs/` | Filesystems and VFS layer |
| `net/` | Networking stack |
| `io_uring/` | Async I/O subsystem |
| `drivers/` | Hardware drivers |
| `block/` | Block I/O layer |
| `include/linux/` | Kernel-wide headers |
| `Documentation/` | Authoritative documentation (RST format) |
| `tools/testing/selftests/` | Kernel self-tests |

---

## Area 1: Interrupts and CPU Isolation

### IRQ Subsystem Core
- `kernel/irq/` — the entire IRQ framework lives here
  - `manage.c` — `request_irq()`, `free_irq()`, IRQ thread setup
  - `handle.c` — dispatcher: `handle_level_irq()`, `handle_edge_irq()`, etc.
  - `chip.c` — `irq_chip` operations (mask, unmask, ack, eoi)
  - `irqdesc.c` — `struct irq_desc`, the per-IRQ descriptor
  - `irqdomain.c` — maps hardware IRQ numbers to Linux IRQ numbers
  - `affinity.c` — IRQ-to-CPU affinity, `irq_set_affinity()`
  - `msi.c` — MSI/MSI-X interrupt management
  - `ipi.c` — inter-processor interrupt abstraction
  - `spurious.c` — spurious interrupt detection and handling
  - `proc.c` — `/proc/irq/` interface

### Architecture IRQ Layer (x86 example)
- `arch/x86/kernel/irq.c` — x86 top-half IRQ entry
- `arch/x86/kernel/apic/` — APIC/x2APIC interrupt controller
- `arch/x86/include/asm/irq_vectors.h` — vector assignments

### CPU Isolation
- `kernel/sched/isolation.c` — `nohz_full` and `isolcpus` implementation
- `Documentation/admin-guide/kernel-parameters.txt` — `isolcpus=`, `nohz_full=`, `rcu_nocbs=`
- `kernel/rcu/` — RCU offloading (`rcu_nocbs`) essential for true isolation
- `/proc/irq/<N>/smp_affinity` — runtime IRQ-to-CPU mapping (bitmask)

### Key Data Structures
- `struct irq_desc` (`include/linux/irqdesc.h`) — per-IRQ state
- `struct irq_chip` (`include/linux/irq.h`) — hardware controller ops
- `struct irq_domain` (`include/linux/irqdomain.h`) — HW-to-Linux IRQ mapping

---

## Area 2: Networking Stack

### Packet Receive Path (top to bottom)
1. Driver DMA → ring buffer (`drivers/net/ethernet/<vendor>/`)
2. NAPI poll → `net/core/dev.c`: `napi_gro_receive()` → `netif_receive_skb()`
3. XDP hook (if attached) in `net/core/filter.c`
4. Protocol demux → `net/ipv4/ip_input.c` → `net/ipv4/tcp_input.c`

### Core Files
- `net/core/dev.c` — network device management, NAPI, `netif_receive_skb()`
- `net/core/skbuff.c` — `sk_buff` allocation, cloning, linearization
- `net/core/filter.c` — BPF/XDP program execution
- `net/core/xdp.c` — XDP memory model (umem frames)
- `net/ipv4/tcp_input.c` / `tcp_output.c` — TCP state machine

### AF_XDP (Kernel Bypass via XDP)
- `net/xdp/xdp_umem.c` — user-space memory registration
- `net/xdp/xsk.c` — AF_XDP socket implementation
- `include/uapi/linux/if_xdp.h` — userspace API
- `Documentation/networking/af_xdp.rst` — **start here**
- `tools/testing/selftests/bpf/xdpxceiver.c` — reference AF_XDP userspace example

### Key Data Structures
- `struct sk_buff` (`include/linux/skbuff.h`) — the universal packet descriptor
- `struct net_device` (`include/linux/netdevice.h`) — network device abstraction
- `struct napi_struct` (`include/linux/netdevice.h`) — NAPI poll context
- `struct xdp_buff` (`include/net/xdp.h`) — XDP frame descriptor (pre-skb)

---

## Area 3: SolarFlare NIC Drivers

### Location
`drivers/net/ethernet/sfc/`

### Driver Generations
| Generation | Hardware | Directory |
|------------|----------|-----------|
| Falcon | Old (SFC4000) | `sfc/falcon/` |
| Siena | SFC9000 | `sfc/siena/` |
| EF10 | Huntington/Medford (SFC9100/9200) | `sfc/*.c` (ef10_ prefix) |
| EF100 | Medford2 / X2 (SFC9250+) | `sfc/ef100_*.c` |

### EF100 Architecture (most current)
- `ef100.c` — probe/remove, PCI setup
- `ef100_nic.c` — EF100 NIC-specific hardware init
- `ef100_netdev.c` — `net_device_ops` registration
- `ef100_rx.c` — RX descriptor ring, DMA, NAPI completion
- `ef100_tx.c` — TX descriptor ring, TSO
- `ef100_ethtool.c` — ethtool stats, ring params, coalesce

### Management Interface
- `mcdi.c` / `mcdi.h` — MCDI (Management-Controller-to-Driver Interface): firmware RPC
- `mcdi_pcol.h` — MCDI protocol command/response definitions (auto-generated from firmware)
- `mcdi_functions.c` — wrappers for common MCDI calls

### Hardware Offload (MAE)
- `mae.c` / `mae.h` — Match-Action Engine: hardware flow tables
- `tc.c` — Linux TC (traffic control) offload integration
- `tc_conntrack.c` — conntrack offload to MAE
- `ef100_rep.c` — representor netdevs for SR-IOV VFs

### SR-IOV
- `ef100_sriov.c` — VF management, VNIC allocation
- `ef10_sriov.c` — EF10 generation SR-IOV

### Key Entry Points for Reading
1. Start with `ef100.c:ef100_probe()` — hardware init sequence
2. Follow to `ef100_nic.c` — register setup, interrupt allocation
3. `ef100_rx.c:ef100_rx_packet()` — per-packet RX processing
4. `mcdi.c:efx_mcdi_rpc()` — how the driver talks to firmware

---

## Area 4: Kernel Bypass

### Mechanisms Overview

| Mechanism | Use Case | Kernel Entry Point |
|-----------|----------|-------------------|
| AF_XDP | Zero-copy packet processing in userspace | `net/xdp/xsk.c` |
| io_uring | Async syscall batching (including networking) | `io_uring/` |
| VFIO | Full device passthrough to userspace/VMs | `drivers/vfio/` |
| UIO | Simple userspace driver framework | `drivers/uio/` |

### AF_XDP Deep Dive
- **Concept:** BPF program running in driver's NAPI poll redirects frames directly to a userspace-mapped ring (UMEM), bypassing kernel networking stack.
- `net/xdp/xsk.c` — socket operations, `bind()` attaches to a specific queue
- `net/xdp/xdp_umem.c` — `xdp_umem_reg()` pins user pages for zero-copy DMA
- `net/core/filter.c` — `xdp_do_redirect()` → `__xsk_map_redirect()`
- Driver integration: look for `XDP_SETUP_PROG` / `ndo_bpf` in SFC's `ef100_netdev.c`

### io_uring Networking
- `io_uring/net.c` — `IORING_OP_RECV`, `IORING_OP_SEND`, `IORING_OP_ACCEPT`
- `io_uring/zcrx.c` — zero-copy RX interface (maps NIC buffers into uring)
- `io_uring/napi.c` — busy-poll integration to reduce latency
- `io_uring/sqpoll.c` — kernel thread that polls submission queue (eliminates syscall)

### VFIO (Device Passthrough)
- `drivers/vfio/vfio.c` — container/group/device model
- `drivers/vfio/pci/` — PCI device passthrough (used for DPDK with physical NICs)
- Relies on IOMMU to isolate DMA from the rest of the system
- `Documentation/driver-api/vfio.rst` — architecture overview

---

## Area 5: Filesystems

### VFS (Virtual Filesystem Switch) — the abstraction layer
- `fs/namei.c` — path lookup: `vfs_open()`, `filename_lookup()`
- `fs/dcache.c` — dentry cache (negative and positive lookups)
- `fs/inode.c` — inode lifecycle
- `fs/buffer.c` — buffer cache (block layer interface)
- `fs/aio.c` — POSIX AIO implementation
- `fs/read_write.c` — `vfs_read()`, `vfs_write()`, `copy_file_range()`

### Key VFS Data Structures
- `struct super_block` (`include/linux/fs.h`) — mounted filesystem instance
- `struct inode` (`include/linux/fs.h`) — file metadata (shared across hard links)
- `struct dentry` (`include/linux/dcache.h`) — directory entry cache node
- `struct file` (`include/linux/fs.h`) — open file description (per fd)
- `struct address_space` (`include/linux/fs.h`) — page cache for a file

### Individual Filesystems
| Filesystem | Location | Key File |
|------------|----------|----------|
| ext4 | `fs/ext4/` | `inode.c`, `extents.c` |
| btrfs | `fs/btrfs/` | `ctree.c` (B-tree), `extent_io.c` |
| xfs | `fs/xfs/` | `xfs_inode.c`, `xfs_log.c` |
| NFS client | `fs/nfs/` | `file.c`, `nfs4proc.c` |
| overlayfs | `fs/overlayfs/` | `copy_up.c`, `super.c` |
| erofs | `fs/erofs/` | `super.c`, `decompressor.c` |

### Page Cache & I/O Path
- `mm/filemap.c` — `filemap_read()`, page cache lookup, `wait_on_page_locked()`
- `mm/readahead.c` — read-ahead logic
- `block/blk-core.c` — block I/O submission (`submit_bio()`)
- `block/blk-mq.c` — multi-queue block layer

---

## Documentation Index

Key docs for each learning area (all under `Documentation/`):

| Topic | Document |
|-------|----------|
| IRQ subsystem | `core-api/genericirq.rst` |
| CPU isolation | `admin-guide/kernel-parameters.txt` (`isolcpus`, `nohz_full`) |
| Networking overview | `networking/index.rst` |
| AF_XDP | `networking/af_xdp.rst` |
| XDP | `networking/xdp-rx-metadata.rst`, `bpf/prog_type_xdp.rst` |
| io_uring | `filesystems/io_uring.rst` |
| VFIO | `driver-api/vfio.rst` |
| Filesystem API | `filesystems/vfs.rst` |
| Writing drivers | `driver-api/driver-model/` |
| Coding style | `process/coding-style.rst` |

---

## Notes Directory

Investigations and learning notes are stored in `docs/` at the repository root. Each topic gets its own subdirectory (e.g., `docs/interrupts/`, `docs/networking/`, `docs/solarflare/`, `docs/kernel-bypass/`, `docs/filesystems/`).
