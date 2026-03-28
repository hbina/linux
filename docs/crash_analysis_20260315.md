# Crash Analysis — 2026-03-15

**Machine:** evox2 (GMKtec NucBox EVO-X2)
**Kernel:** 6.17.0-19-generic (Ubuntu)
**GPU:** AMD Radeon (amdgpu, 16 GB VRAM — device 0000:c6:00.0, IP Discovery 0x1586)
**Crash time:** ~22:01:06 (abrupt hang, no clean shutdown)

---

## Summary

The machine hard-locked at approximately **22:01:06**. The last log entry is Discord sending an RTC heartbeat at that timestamp with **no ACK ever received** — the journal just cuts off, which is characteristic of a sudden kernel freeze rather than a controlled shutdown.

---

## Timeline of Suspicious Events

| Time     | Event |
|----------|-------|
| 21:21:36 | VirtualBox modules loaded (vboxdrv + VBoxNetFlt attached to WiFi) |
| 21:26:14 | `vboxnetflt: 0 out of 8394 packets not sent (directed to host)` |
| 21:34:47 | VirtualBox modules reloaded (second VM session?) |
| 21:35:39 | `vboxnetflt: 0 out of 1428 packets not sent` |
| 21:41–21:54 | **tracker-miner-fs-3 repeatedly spawning new extract processes** (~every 60s, each failing immediately with `Error creating IO channel for /proc/self/mountinfo: Invalid argument`) — PIDs: 70855, 74256, 78183, 78604, 79451, 79901, 83334 |
| 21:48:32 | `vte-spawn-ba3e9488.scope: Consumed 2min 53.846s CPU time` — a terminal session ate ~3 minutes of CPU |
| **21:49:13** | **`libinput error: Keychron K10 Pro: event processing lagging behind by 35ms, your system is too slow`** |
| **21:49:19** | **`libinput error: Keychron K10 Pro: lagging behind by 48ms`** (worsening) |
| **21:49:51** | **`libinput error: Logitech G403 Mouse: lagging behind by 22ms`** — both input devices affected |
| **21:52:58** | **Last kernel message** (apparmor audit #2275) — kernel goes silent |
| 22:00:08 | `systemd: Started gnome-terminal via gsd-media-keys` — user opened a terminal 1 min before crash |
| **22:01:06** | **Hard lockup** — Discord heartbeat sent, no ACK, log ends |

---

## Key Indicators

### 1. Input Processing Lag (21:49) — Earliest Clear Signal
gnome-shell reported both the keyboard and mouse lagging, saying "your system is too slow." This is the first hard evidence that the system was under significant CPU/memory pressure ~12 minutes before the crash. The lag was progressively worsening (35ms → 48ms on keyboard alone).

### 2. tracker-miner Rapid Respawning — Possible OOM Symptom
`tracker-miner-fs-3` kept spawning new `tracker-extract-3` worker processes roughly every minute from 21:41 to 21:54, each one dying immediately with:
```
Error creating IO channel for /proc/self/mountinfo: Invalid argument
```
This error on `/proc/self/mountinfo` is unusual. Combined with the respawn frequency, it strongly suggests tracker's workers were being **killed by the OOM killer** and tracker-miner was restarting them. The OOM killer messages themselves may not have been flushed to disk before the crash.

### 3. Kernel Silent for 8+ Minutes Before Crash
The last kernel log message is at **21:52:58**. The system kept running (Discord heartbeats continued, systemd started a terminal at 22:00:08) but the kernel produced zero log output for over 8 minutes until the hard lockup. This gap suggests the kernel was either:
- In an uninterruptible wait / soft-lockup not yet expired
- Under severe memory pressure with the journal ring buffer unable to flush
- Already partially locked up but userspace processes with cached pages continued briefly

### 4. Active Load at Crash Time
The system was running concurrently:
- **VirtualBox VM** with KVM (loaded at 21:34, WiFi bridging active)
- **Discord** voice call (RTC socket with sequence #17 active throughout)
- **Firefox** (snap, multiple processes — pid 11598, 49742)
- **Steam**
- **GNOME Shell + Wayland compositor**

This combination easily saturates RAM on a mini-PC class machine.

### 5. Terminal Spawned 1 Minute Before Crash
At 22:00:08 the user opened a new GNOME Terminal via a keyboard shortcut (`gsd-media-keys`). This was 58 seconds before the last log entry. Whatever was run in that terminal may have been the final trigger — possibly a memory-intensive command that pushed an already-stressed system over the edge.

---

## Most Likely Cause

**Memory exhaustion leading to a kernel hard lockup**, with VirtualBox + KVM as the primary memory consumer. The sequence:

1. VirtualBox VM running for ~40 minutes consumed a large chunk of RAM
2. Discord voice call, Firefox, and Steam added further pressure
3. System started becoming unresponsive to input (~21:49, the libinput lag)
4. tracker-miner workers began getting OOM-killed and respawning (~21:41–21:54)
5. Kernel logging stalled (~21:53) as journald could no longer write
6. User opened a terminal at 22:00 and likely ran a command
7. Final hard lockup at 22:01:06 — no kernel panic, no OOM report, just silence

The lack of a kernel panic or OOM log is consistent with a **complete kernel freeze** (not a controlled OOM kill) under extreme memory pressure, which can happen when the OOM killer itself cannot run because all memory paths are blocked.

---

## Recommendations

1. **Check available RAM** when VirtualBox is running — `free -h` before starting a VM. Consider setting a hard memory limit on VMs.
2. **Monitor with `vmstat 1`** or `dmesg -w` in a terminal during heavy sessions to catch OOM events early.
3. Enable **`/proc/sys/vm/panic_on_oom = 1`** (or at least `2`) to force a kernel panic + reboot instead of a silent freeze — this also produces a crash dump.
4. Check `journalctl -b -1` on next boot (the *previous* boot's journal) for any OOM killer messages that may have been flushed post-crash.
5. Consider whether the VirtualBox session in use had swap enabled — adding a swapfile can absorb OOM pressure and prevent hard lockups.
