# Machine Inventory and Runtime Map

Snapshot of the hardware and runtime stack on the development machine used for
this kernel study. Fulfills Phase 0 of `docs/linux/machine-first-study-plan.md`.

**Snapshot date:** 2026-05-03
**Hostname:** `evox2`
**System:** GMKtec NucBox EVO-X2 (mini-PC)

If anything below has drifted since this snapshot was written, regenerate by
re-running the commands listed under "Reproduce" at the bottom of this file.

---

## CPU / SoC

| Item | Value |
|------|-------|
| Model | AMD Ryzen AI MAX+ 395 ("Strix Halo" APU) |
| Architecture | x86_64 |
| Sockets / Cores / Threads | 1 / 16 / 32 |
| NUMA nodes | 1 (CPUs 0–31 on node 0) |
| Frequency range | 625 MHz – 5187.5 MHz |
| cpufreq driver | `amd-pstate-epp` |
| cpufreq governor | `powersave` |
| EPP (Energy/Performance Preference) | `performance` |
| EPP available | `default performance balance_performance balance_power power custom` |
| cpuidle driver | `acpi_idle` |
| cpuidle governor | `menu` |
| Memory | 32 GiB (32488440 kB), 8 GiB swap |
| Hugepages | none configured |

**Notes**
- `amd-pstate-epp` is the active path — **not** the legacy `acpi-cpufreq`. Phase 5 reading should target `drivers/cpufreq/amd-pstate.c`.
- Governor is `powersave` but EPP is `performance`: under `amd-pstate-epp`, the EPP hint controls the trade-off, so this is closer to "max performance with light power saving" than the name suggests.

---

## Kernel

| Item | Value |
|------|-------|
| Running kernel | `7.0.0-13096-gdd6c438c3e64` |
| Build date | Fri Apr 24 17:51:43 UTC 2026 |
| Preemption model | `PREEMPT_DYNAMIC` + `PREEMPT_LAZY` (runtime-tunable) |
| Config file | `/boot/config-7.0.0-13096-gdd6c438c3e64` |
| Other configs available | `6.17.0-23-generic` (Ubuntu stock), `7.1.0-rc1-ga787f13efe41` (×2) |

### Module loading model

`lsmod` shows only four loaded modules:

```
zfs spl configfs efivarfs
```

All in-tree subsystem drivers relevant to this study (`r8169`, `amdgpu`,
`nvme`, `ext4`, `xhci_hcd`, `snd_hda_intel`, etc.) are compiled **`=y`
(builtin)** in this kernel, confirmed by their presence in
`/lib/modules/$(uname -r)/modules.builtin` and the absence of corresponding
`.ko` files.

**Practical consequence:** there is no `rmmod`/`insmod` cycle for driver work
on this machine. Iterating on a driver change requires either:
- a full kernel rebuild + reboot, or
- `vng` (`virtme-ng`) for fast in-VM iteration — see `docs/linux/tools/vng_workflow_summary.md`.

---

## Networking

### Wired NIC — `eno1` (active)

| Item | Value |
|------|-------|
| PCI address | `c1:00.0` |
| PCI ID | `10ec:8125` |
| Chip | Realtek RTL8125 2.5 GbE Controller (rev 05) |
| Driver | `r8169` (builtin) |
| MAC | `84:47:09:6c:27:3f` |
| State | `up` |
| Source path | `drivers/net/ethernet/realtek/r8169_main.c` |

Within `r8169_main.c`, the RTL8125 path is selected by the
`RTL_GIGA_MAC_VER_*` 8125-family branches — distinct from the older 8168/8169
paths.

### Wi-Fi — MediaTek MT7925 (currently unusable)

| Item | Value |
|------|-------|
| PCI address | `c3:00.0` |
| PCI ID | `14c3:0717` |
| Chip | MediaTek MT7925 |
| Kernel driver bound | **none** |
| `/sys/class/ieee80211/` | empty |
| In-tree driver | `drivers/net/wireless/mediatek/mt76/mt7925/` (PCIe variant: `mt7925e`) |
| `CONFIG_MT7925E` in current kernel | `# not set` |
| Other MT76 family options in current kernel | all `# not set` |

The driver is in-tree but disabled in this kernel build. To make Wi-Fi
operational, rebuild with at least:

```
CONFIG_MT76_CORE=m   (or =y)
CONFIG_MT792X_LIB=m
CONFIG_MT7925_COMMON=m
CONFIG_MT7925E=m
```

(Plus matching firmware in `linux-firmware`.)

---

## GPU / Display — `amdgpu`

| Item | Value |
|------|-------|
| PCI address | `c6:00.0` |
| PCI ID | `1002:1586` (rev c1) |
| Chip | AMD Radeon 8060S (integrated, RDNA 3.5 / GFX11.5, "Strix Halo" APU) |
| Driver | `amdgpu` (builtin) |
| DRM card | `card0` |
| Render node | `renderD128` |
| Connectors | 1× HDMI-A, 8× DisplayPort, 1× Writeback |

Because this is an APU sharing system memory with the CPU, the relevant
`amdgpu` paths are GTT-/unified-memory-heavy (`TTM_PL_TT`) rather than
discrete-VRAM-centric (`TTM_PL_VRAM`). The 8 DP + 1 HDMI + writeback
connector layout makes atomic-commit and hotplug paths non-trivial.

Audio companion devices (also `amdgpu` neighbours, but bound to
`snd_hda_intel`):
- `c6:00.1` — Rembrandt Radeon HDA (`1002:1640`)
- `c6:00.6` — Family 17h/19h on-board HDA (`1022:15e3`)

---

## Storage

### Block device map

```
nvme1n1   931.5G   Phison 5029 (PCI c4:00.0, 1987:5029)
├─ p1       1.0G   vfat   /boot/efi
└─ p2     930.5G   ext4   /                                 <-- root
nvme0n1   1.9T     Sandisk WD Blue SN580 DRAM-less (PCI c5:00.0, 15b7:5041)
└─ p1     1.9T     zfs_member                               (out of scope)
sda       931.5G   SATA disk
└─ p1     931.5G   zfs_member                               (out of scope)
sdb       931.5G   SATA disk
└─ p1     931.5G   zfs_member                               (out of scope)
```

### Scope for in-tree study

| Path | In-tree? | Used in study? |
|------|----------|----------------|
| `nvme1n1p2` (root, ext4) | yes | **yes — primary target for Phase 4** |
| `nvme1n1p1` (EFI, vfat) | yes | incidental |
| `nvme0n1` (Sandisk, ZFS) | hardware yes; ZFS no | NVMe driver in-scope, FS layer out-of-scope |
| `sda`, `sdb` (SATA, ZFS) | hardware yes; ZFS no | possible AHCI/libata reading later, ZFS out-of-scope |

ZFS is the only out-of-tree module loaded (`zfs` + `spl`); it covers the
secondary NVMe and both SATA disks. Because it is out-of-tree, none of those
paths are part of this kernel study.

The Phison NVMe is DRAM-equipped (`5029`); the Sandisk SN580 is DRAM-less.
That contrast may be useful for later performance experiments.

---

## Other in-use devices

| PCI | Chip | Driver |
|-----|------|--------|
| `c2:00.0` | Genesys GL9755 SD Host Controller | (none bound; SD slot unused) |
| `c6:00.4`, `c8:00.0`, `c8:00.3`, `c8:00.4` | AMD USB controllers | `xhci_hcd` (via `xhci_pci`) |
| `c6:00.2` | AMD Encryption controller (CCP) | none currently bound |
| `c7:00.1` | AMD Signal processing controller | none currently bound |
| `c6:00.1`, `c6:00.6` | AMD HDA audio controllers | `snd_hda_intel` |
| `00:00.2` | AMD IOMMU | (kernel-internal) |

---

## Config vs Hardware Audit

Checked against the running kernel config
`/boot/config-7.0.0-13096-gdd6c438c3e64` and the repo `.config`
(`7.1.0-rc1-ga787f13efe41` lineage). As of the local config update on
2026-05-03, `.config` enables the hardware-support gaps listed below; the
running `/boot/config-7.0.0-13096-gdd6c438c3e64` still reflects the old booted
kernel until the next rebuild/install/reboot.

### Hardware with matching driver support enabled

| Hardware | PCI ID | Active driver | Required config | Status |
|----------|--------|---------------|-----------------|--------|
| Realtek RTL8125 2.5GbE | `10ec:8125` | `r8169` | `CONFIG_R8169=y`, `CONFIG_REALTEK_PHY=y` | OK |
| Phison NVMe | `1987:5029` | `nvme` | `CONFIG_NVME_CORE=y` | OK |
| Sandisk SN580 NVMe | `15b7:5041` | `nvme` | `CONFIG_NVME_CORE=y` | OK |
| AMD Radeon integrated GPU | `1002:1586` | `amdgpu` | `CONFIG_DRM_AMDGPU=y` | OK |
| AMD HDA audio | `1002:1640`, `1022:15e3` | `snd_hda_intel` | `CONFIG_SND_HDA_INTEL=y` | OK |
| AMD xHCI USB controllers | `1022:1587`, `1588`, `1589`, `158b` | `xhci_hcd` | `CONFIG_USB_XHCI_HCD=y`, `CONFIG_USB_XHCI_PCI=y` | OK |
| AMD IOMMU | `1022:1508` | kernel-internal | `CONFIG_AMD_IOMMU=y` | OK |

### Hardware present but not currently enabled in the running kernel

| Hardware | PCI ID | Driver / config | Current state | Practical impact |
|----------|--------|-----------------|---------------|------------------|
| MediaTek MT7925 Wi-Fi | `14c3:0717` | `mt7925e`, `CONFIG_MT7925E` | running: `# not set`; repo `.config`: `m` | Current boot has no Wi-Fi driver; next build should produce `mt7925e` |
| Genesys GL9755 SD host | `17a0:9755` | `sdhci-pci`, `CONFIG_MMC`, `CONFIG_MMC_SDHCI`, `CONFIG_MMC_SDHCI_PCI` | running: `# CONFIG_MMC is not set`; repo `.config`: `m` | Current boot lacks SD reader support; next build should produce MMC/SDHCI modules |
| AMD encryption / PSP / CCP | `1022:17e0` | `ccp`, `CONFIG_CRYPTO_DEV_CCP` / PSP-related CCP config | running: `# not set`; repo `.config`: enabled with driver pieces as modules | Current boot has no CCP/PSP driver; next build should produce `ccp` support |
| AMD sensor fusion / signal processing | `1022:17f0` | `amd_sfh`, `CONFIG_AMD_SFH_HID` | running: `# not set`; repo `.config`: `m` | Current boot lacks SFH support; next build should produce `amd-sfh` |
| AMD platform management | ACPI/platform | `amd_pmc`, `CONFIG_AMD_PMC` | running: `# not set`; repo `.config`: `m` | Current boot lacks AMD PMC driver; next build should produce `amd-pmc` |
| USB4 / Type-C support | USB/ACPI/platform | `CONFIG_USB4`, `CONFIG_TYPEC` | running: both `# not set`; repo `.config`: both `m` | Current boot lacks USB4/Type-C class support; next build should produce those modules |

### Why `CONFIG_MT7925E` is not set

The MT7925 PCIe driver is available in this tree and it explicitly matches the
machine's Wi-Fi PCI ID:

```c
{ PCI_DEVICE(PCI_VENDOR_ID_MEDIATEK, 0x0717),
	.driver_data = (kernel_ulong_t)MT7925_FIRMWARE_WM },
```

The Kconfig entry is:

```text
config MT7925E
	tristate "MediaTek MT7925E (PCIe) support"
	select MT7925_COMMON
	depends on MAC80211
	depends on PCI
```

The dependencies are already satisfied in the current config:

```text
CONFIG_MAC80211=y
CONFIG_PCI=y
CONFIG_WLAN=y
CONFIG_WLAN_VENDOR_MEDIATEK=y
CONFIG_FW_LOADER=y
```

So this is not a dependency-resolution problem. `CONFIG_MT7925E` is a normal
user-selectable tristate with no default `y`/`m`; this local config simply did
not opt into it. Since `CONFIG_MT7925E` is `# not set`, the build does not
produce either a builtin driver or an `mt7925e.ko`, so PCI autoload/binding has
nothing to attach to `14c3:0717`.

Firmware is already present:

```text
/lib/firmware/mediatek/mt7925/WIFI_RAM_CODE_MT7925_1_1.bin.zst
/lib/firmware/mediatek/mt7925/WIFI_MT7925_PATCH_MCU_1_1_hdr.bin.zst
```

The local repo `.config` now uses the smallest practical enablement:

```text
CONFIG_MT7925E=m
```

Kconfig will select the shared MT76/MT7925 support needed underneath
(`MT7925_COMMON`, `MT792x_LIB`, `MT76_CONNAC_LIB`, `MT76_CORE`). Using `=m`
keeps Wi-Fi reloadable during bring-up; using `=y` matches the current
mostly-builtin style of this kernel.

---

## Phase 0 status

Item-by-item against `machine-first-study-plan.md` Phase 0. All items here are
**recorded** — leaving the actual `[x]` ticking to the user per the project
convention.

- **0.1** Inventory recorded — this file.
- **0.2** Wired NIC: `/sys/class/net/eno1/device/driver` → `r8169`; PCI `c1:00.0` (RTL8125 2.5GbE, `10ec:8125`). **Confirmed.**
- **0.3** Wi-Fi: PCI `c3:00.0` is MediaTek MT7925 (`14c3:0717`), no driver bound, `CONFIG_MT7925E` is **explicitly `# not set`** in this kernel. **Decision pending — see "Open decisions" below.**
- **0.4** DRM: `card0` → `amdgpu`; integrated APU (RDNA 3.5 / Strix Halo, `1002:1586`). **Confirmed.**
- **0.5** Root storage: `nvme1n1p2` ext4 on Phison NVMe; ZFS disks out of scope. **Confirmed.**
- **0.6** cpufreq = `amd-pstate-epp` (governor `powersave`, EPP `performance`); cpuidle = `acpi_idle` (governor `menu`). **Confirmed — differs from legacy `acpi-cpufreq` assumption.**
- **0.7** Personal latency/performance ranking — **owner-only question, draft below.**

---

## Open decisions

### 0.3 — Wi-Fi enablement

Two reasonable paths:

1. **Enable `CONFIG_MT7925E=m`** in the next kernel rebuild. Adds Wi-Fi as a real device on the machine and unlocks Phase 2 of the study plan as practical (rather than theoretical). Modest extra build/firmware footprint.
2. **Defer.** Mark Wi-Fi out-of-scope until a concrete reason to enable it appears. Phase 2 then becomes a code-reading-only exercise with no on-machine validation.

The plan currently leans toward (1) since the whole point of the
machine-first track is to study what the hardware actually is. But this is a
choice the owner of the machine should make — both options are recorded.

### 0.7 — Latency/performance ranking (DRAFT — owner to confirm)

Drafted from context (existing investigation notes lean GPU + interrupts;
`HuggingFace/` in the workspace runs LLM inference on this APU, which is
GPU/memory-bandwidth heavy):

1. **Graphics / display** — APU shared memory + 8 DP + LLM inference workload makes this both the most-touched and the most-investigated subsystem.
2. **Storage** — root on `nvme1n1p2`; build/edit cycles, model loading, and page cache behaviour all hit this every day.
3. **Networking** — single 2.5 GbE link is rarely the bottleneck for this machine's workload, but the existing EF100/IRQ notes show the area is interesting independently.
4. **Power management** — last in priority since this is a mains-powered desktop, but `amd-pstate` EPP behaviour interacts with everything above.

Confirm or edit when ready.

---

## Reproduce

Commands used to gather everything in this file:

```bash
# Identity
uname -a
cat /sys/devices/virtual/dmi/id/{sys_vendor,product_name,board_name}

# CPU / power
lscpu
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences
cat /sys/devices/system/cpuidle/current_driver
cat /sys/devices/system/cpuidle/current_governor
free -h

# PCI / drivers
lspci -nnk

# Network interfaces
for iface in /sys/class/net/*; do
    name=$(basename "$iface")
    drv=$(readlink -f "$iface/device/driver" 2>/dev/null | xargs -I{} basename {})
    mac=$(cat "$iface/address" 2>/dev/null)
    state=$(cat "$iface/operstate" 2>/dev/null)
    echo "$name: driver=$drv mac=$mac state=$state"
done

# DRM
ls /sys/class/drm/
readlink -f /sys/class/drm/card0/device/driver

# Wi-Fi state
ls /sys/class/ieee80211/  # empty -> no wireless interfaces
grep MT7925 /boot/config-$(uname -r)

# Storage
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
findmnt /
findmnt /boot/efi

# Module model
lsmod
grep -E 'r8169|amdgpu|nvme|ext4' /lib/modules/$(uname -r)/modules.builtin
```
