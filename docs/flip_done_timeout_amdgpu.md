# AMD GPU `flip_done` Timeout: Root Cause and Fix

## Table of Contents

1. [Background: DRM Atomic Commit Model](#1-background-drm-atomic-commit-model)
2. [Background: Vblank Interrupts and Refcounting](#2-background-vblank-interrupts-and-refcounting)
3. [The AMD Display Manager Event Delivery Model](#3-the-amd-display-manager-event-delivery-model)
4. [The Two Commit Paths](#4-the-two-commit-paths)
5. [The Bug: Shared Event Slot and the Race](#5-the-bug-shared-event-slot-and-the-race)
6. [What Changed in Kernel 6.12](#6-what-changed-in-kernel-612)
7. [The Failing Sequence Step by Step](#7-the-failing-sequence-step-by-step)
8. [The Proposed Fix](#8-the-proposed-fix)
9. [Open Questions and Ongoing Discussion](#9-open-questions-and-ongoing-discussion)

---

## 1. Background: DRM Atomic Commit Model

Modern DRM drivers use an *atomic commit* model where all display state changes
(planes, CRTCs, connectors) are bundled into a single `drm_atomic_state` object
and committed to hardware atomically. The important invariant is:

> After the state is programmed to hardware, userspace must be notified **exactly
> once** that the hardware has latched the new frame.

This notification is the **`flip_done` completion**. It is represented as a
`struct drm_crtc_commit` containing a kernel `struct completion`:

```c
// drivers/gpu/drm/drm_atomic_helper.c:2563
new_crtc_state->event->base.completion = &commit->flip_done;
```

The KMS commit worker calls `drm_atomic_helper_wait_for_flip_done()` after
programming hardware. It blocks for up to **10 seconds**:

```c
// drivers/gpu/drm/drm_atomic_helper.c:1959
ret = wait_for_completion_timeout(&commit->flip_done, 10 * HZ);
if (ret == 0)
    drm_err(dev, "[CRTC:%d:%s] flip_done timed out\n",
            crtc->base.id, crtc->name);
```

The `flip_done` completion is triggered automatically when the associated
`drm_pending_vblank_event` is **sent** via `drm_crtc_send_vblank_event()`. This
is wired up in `drm_atomic_helper_setup_commit()`:

```c
// drivers/gpu/drm/drm_atomic_helper.c:2563-2564
new_crtc_state->event->base.completion = &commit->flip_done;
new_crtc_state->event->base.completion_release = release_crtc_commit;
```

So the critical invariant is: **every commit that has an event must eventually
call `drm_crtc_send_vblank_event()` exactly once for that event**. If it is
never called, the wait times out. If it is called twice, the second call fires
for the wrong commit.

---

## 2. Background: Vblank Interrupts and Refcounting

The DRM vblank subsystem maintains a **reference count** per CRTC. As long as
the refcount is non-zero, the vblank interrupt is guaranteed to be enabled:

```c
// drivers/gpu/drm/drm_vblank.c:1248
int drm_crtc_vblank_get(struct drm_crtc *crtc)
{
    return drm_vblank_get(crtc->dev, drm_crtc_index(crtc));
}
```

When the last reference is dropped via `drm_crtc_vblank_put()`, a timer is
scheduled to disable the interrupt after `offdelay_ms` milliseconds:

```c
// drivers/gpu/drm/drm_vblank.c:1266-1273
if (atomic_dec_and_test(&vblank->refcount)) {
    if (!vblank_offdelay)
        return;
    else if (vblank_offdelay < 0)
        vblank_disable_fn(&vblank->disable_timer);
    else if (!vblank->config.disable_immediate)
        mod_timer(&vblank->disable_timer,
                  jiffies + ((vblank_offdelay * HZ) / 1000));
}
```

The purpose of `drm_crtc_vblank_get()` before storing a pending event is
therefore: **keep the interrupt alive long enough for it to fire and deliver
the event**. The paired `drm_crtc_vblank_put()` is called inside
`drm_crtc_send_vblank_event()` via `dm_pflip_high_irq` or
`amdgpu_dm_crtc_handle_vblank()`.

---

## 3. The AMD Display Manager Event Delivery Model

`amdgpu_dm` (the Display Manager, the AMD-specific DRM driver layer) owns the
`struct amdgpu_crtc`, which extends `struct drm_crtc` with AMD-specific state:

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_mode.h:460-513
struct amdgpu_crtc {
    struct drm_crtc base;
    // ...
    struct amdgpu_flip_work *pflip_works;
    enum amdgpu_flip_status pflip_status;   // NONE / PENDING / SUBMITTED
    // ...
    struct drm_pending_vblank_event *event; // the pending completion event
    // ...
};
```

The `pflip_status` field acts as a state machine:

```c
// drivers/gpu/drm/amd/amdgpu/amdgpu_mode.h:120-124
enum amdgpu_flip_status {
    AMDGPU_FLIP_NONE,       // 0 — idle, no flip in progress
    AMDGPU_FLIP_PENDING,    // 1 — scheduled but not yet submitted to hardware
    AMDGPU_FLIP_SUBMITTED   // 2 — submitted to hardware, waiting for IRQ
};
```

There are **two distinct interrupt handlers** responsible for delivering events:

### `dm_pflip_high_irq` — for plane (framebuffer) flips

This fires when the display hardware physically latches a new framebuffer
address. It is the **authoritative** completion signal for plane flips.

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:453-460
if (amdgpu_crtc->pflip_status != AMDGPU_FLIP_SUBMITTED) {
    drm_dbg_state(dev, "pflip_status = %d != SUBMITTED ...\n", ...);
    spin_unlock_irqrestore(...);
    return;  // bail out — not our event to deliver
}
```

If `pflip_status == SUBMITTED`, it:
1. Grabs `acrtc->event` (the pending completion event)
2. Clears `acrtc->event = NULL`
3. Calls `drm_crtc_send_vblank_event()`, which fires the `flip_done` completion
4. Calls `drm_crtc_vblank_put()` to release the vblank reference
5. Resets `pflip_status = AMDGPU_FLIP_NONE`

### `amdgpu_dm_crtc_handle_vblank` — for cursor-only commits

This is called from the vblank interrupt handler (`dm_crtc_high_irq` /
`dm_vupdate_high_irq`). For cursor-only commits there is no page flip interrupt
(the hardware does not fire `dm_pflip_high_irq` for cursor position changes), so
event delivery must happen here instead.

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_crtc.c:41-59
void amdgpu_dm_crtc_handle_vblank(struct amdgpu_crtc *acrtc)
{
    struct drm_crtc *crtc = &acrtc->base;
    struct drm_device *dev = crtc->dev;
    unsigned long flags;

    drm_crtc_handle_vblank(crtc);

    spin_lock_irqsave(&dev->event_lock, flags);

    /* Send completion event for cursor-only commits */
    if (acrtc->event && acrtc->pflip_status != AMDGPU_FLIP_SUBMITTED) {
        drm_crtc_send_vblank_event(crtc, acrtc->event);
        drm_crtc_vblank_put(crtc);
        acrtc->event = NULL;
    }

    spin_unlock_irqrestore(&dev->event_lock, flags);
}
```

The guard condition `pflip_status != AMDGPU_FLIP_SUBMITTED` is intended to
mean: *"only deliver if there is no plane flip in flight — plane flips are
handled by `dm_pflip_high_irq`."*

---

## 4. The Two Commit Paths

Inside `amdgpu_dm_commit_planes()`, AMD's display manager chooses one of two
paths depending on what changed in the commit:

### Path A: Plane flip (framebuffer changed)

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:10179-10208
if (acrtc_attach->base.state->event &&
    acrtc_state->active_planes > 0) {
    drm_crtc_vblank_get(pcrtc);           // hold vblank interrupt alive

    spin_lock_irqsave(&pcrtc->dev->event_lock, flags);
    WARN_ON(acrtc_attach->pflip_status != AMDGPU_FLIP_NONE);
    prepare_flip_isr(acrtc_attach);       // arm for IRQ delivery
    spin_unlock_irqrestore(&pcrtc->dev->event_lock, flags);
}
```

`prepare_flip_isr()` stores the event and marks `pflip_status = SUBMITTED`:

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:9567-9584
static void prepare_flip_isr(struct amdgpu_crtc *acrtc)
{
    assert_spin_locked(&acrtc->base.dev->event_lock);
    WARN_ON(acrtc->event);

    acrtc->event = acrtc->base.state->event;  // store the event

    /* Set the flip status */
    acrtc->pflip_status = AMDGPU_FLIP_SUBMITTED;  // mark in-flight

    /* Mark this event as consumed */
    acrtc->base.state->event = NULL;
}
```

**Delivery:** `dm_pflip_high_irq` fires when hardware latches the new buffer.
It checks `pflip_status == SUBMITTED`, delivers the event, and resets to `NONE`.

### Path B: Cursor-only update (cursor position changed, no framebuffer change)

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:10216-10203
} else if (cursor_update && acrtc_state->active_planes > 0) {
    spin_lock_irqsave(&pcrtc->dev->event_lock, flags);
    if (acrtc_attach->base.state->event) {
        drm_crtc_vblank_get(pcrtc);                        // hold vblank alive
        acrtc_attach->event = acrtc_attach->base.state->event;  // store event
        acrtc_attach->base.state->event = NULL;
        // NOTE: pflip_status is NOT changed — stays AMDGPU_FLIP_NONE
    }
    spin_unlock_irqrestore(&pcrtc->dev->event_lock, flags);
}
```

**Delivery:** `amdgpu_dm_crtc_handle_vblank()` fires on the next vblank
interrupt. It checks `event != NULL && pflip_status != SUBMITTED`, delivers the
event, and releases the vblank reference.

---

## 5. The Bug: Shared Event Slot and the Race

The fundamental flaw is that **both paths store their event in the same
`acrtc->event` field**, and the guard in `amdgpu_dm_crtc_handle_vblank` has a
logic error.

The guard is:
```c
if (acrtc->event && acrtc->pflip_status != AMDGPU_FLIP_SUBMITTED)
```

This is meant to exclude events belonging to in-flight plane flips. But look at
`enum amdgpu_flip_status` again:

```
AMDGPU_FLIP_NONE      = 0   ← idle
AMDGPU_FLIP_PENDING   = 1
AMDGPU_FLIP_SUBMITTED = 2   ← plane flip in-flight
```

`AMDGPU_FLIP_NONE == 0` satisfies `!= SUBMITTED`. So the guard actually means:

> "Deliver any event in `acrtc->event` as long as there is no plane flip
> *currently* in `SUBMITTED` state."

This creates a **race window** when:

1. A cursor-only commit stores its event in `acrtc->event` (pflip_status=NONE)
2. A plane flip commit races in and calls `prepare_flip_isr()`, which
   **overwrites** `acrtc->event` with the flip's event and sets
   `pflip_status = SUBMITTED`
3. `dm_pflip_high_irq` fires, sees `pflip_status == SUBMITTED`, delivers the
   event from `acrtc->event` (which is now the flip's event — the cursor event
   was overwritten), clears `acrtc->event = NULL`, resets to `NONE`
4. The **actual** flip commit's event was delivered prematurely for the wrong
   flip. The next flip commit has no event stored.

OR: the second race (starvation path):

1. Cursor-only commit stores event; `drm_crtc_vblank_get()` holds refcount
2. `drm_crtc_vblank_put()` is called elsewhere before the vblank fires
3. The off-delay timer fires and disables the interrupt
4. `amdgpu_dm_crtc_handle_vblank()` never runs
5. The cursor event is never delivered → 10-second timeout

The bpftrace analysis in the bug report captured Race 1 exactly:

```
9931771 drm_vblank_disable_and_save          ← vblank off-delay fires
9931771 drm_crtc_send_vblank_event           ← cursor event delivered by handle_vblank
9931771 drm_vblank_put
9931771 drm_atomic_helper_commit_hw_done
9931771 drm_atomic_helper_wait_for_flip_done ENTER [tid=35929]
9931771 drm_atomic_helper_wait_for_flip_done EXIT 0ms  ← instant: event was pre-delivered
9931773 drm_atomic_helper_commit_hw_done
9931773 drm_atomic_helper_wait_for_flip_done ENTER [tid=36929]  ← new commit
9931777 dm_pflip_high_irq                    ← flip fires, delivers the "wrong" one
9931777 drm_crtc_send_vblank_event
9931777 drm_vblank_put
9931777 drm_atomic_helper_wait_for_flip_done EXIT 3ms [tid=36929]
9931781 drm_atomic_helper_commit_hw_done
9931781 drm_atomic_helper_wait_for_flip_done ENTER [tid=36929]  ← THIS HANGS
... 10328ms silence ...
9942110 drm_atomic_helper_wait_for_flip_done TIMEOUT
```

The `dmesg` at timeout confirms the symptom:

```
pflip_status=0 (AMDGPU_FLIP_NONE) but event is still non-NULL
```

`pflip_status` is `NONE` but `event != NULL`. The event was armed for a flip
that hardware already processed, but the counter was zeroed by a cursor event
racing through `amdgpu_dm_crtc_handle_vblank()`. The real flip's event slot was
consumed by the cursor handler before `dm_pflip_high_irq` could deliver it.

---

## 6. What Changed in Kernel 6.12

This race existed in theory since commit `473683a03495` ("drm/amd/display: Create
a file dedicated for CRTC", 2022), which moved the shared `acrtc->event` logic
from `amdgpu_dm.c` into `amdgpu_dm_crtc.c`. However it was virtually impossible
to trigger because the default `drm_vblank_offdelay` was **5000 ms** — the
vblank interrupt almost never turned off between commits.

Commit `58a261bfc967` ("drm/amd/display: use a more lax vblank enable policy for
older ASICs", kernel 6.12) changed **all** ASICs to call
`drm_crtc_vblank_on_config()` with a computed off-delay:

```c
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c:9352-9368
} else if (amdgpu_ip_version(adev, DCE_HWIP, 0) <
           IP_VERSION(3, 5, 0) ||
           !(adev->flags & AMD_IS_APU)) {
    /*
     * Older HW and DGPU have issues with instant off;
     * use a 2 frame offdelay.
     */
    offdelay = DIV64_U64_ROUND_UP((u64)20 *
                                  timing->v_total *
                                  timing->h_total,
                                  timing->pix_clk_100hz);
    config.offdelay_ms = offdelay ?: 30;
} else {
    /* offdelay_ms = 0 will never disable vblank */
    config.offdelay_ms = 1;
    config.disable_immediate = true;
}
drm_crtc_vblank_on_config(&acrtc->base, &config);
```

At 144 Hz: `20 * v_total * h_total / pix_clk ≈ 2 frame periods ≈ 14 ms`.

This reduced the off-delay from 5000 ms to **~14 ms**, causing
`drm_vblank_disable_and_save` to fire **hundreds of times more often** during
normal desktop use. The bpftrace logs show it interleaved constantly between
commits, turning the theoretical race into a reliably reproducible one.

---

## 7. The Failing Sequence Step by Step

Here is the complete failing race reconstructed from the bpftrace output,
annotated with the exact code paths involved:

```
T=0ms  Cursor commit begins for CRTC 0
       amdgpu_dm_commit_planes() → Path B (cursor_update && active_planes > 0)
         drm_crtc_vblank_get(crtc0)     // refcount: 0→1
         acrtc->event = cursor_event_A  // store in shared slot
         pflip_status stays NONE (0)
         state->event = NULL

T=0ms  drm_atomic_helper_commit_hw_done() signals hw_done
       drm_atomic_helper_wait_for_flip_done() ENTER on tid=36929
       Waiting for commit->flip_done (which fires when cursor_event_A is sent)

T=1ms  Off-delay timer fires (14ms window expired from a prior vblank_put)
       drm_vblank_disable_and_save()
       Before disabling, flushes pending vblank events from vblank_event_list
       → dm_crtc_high_irq / amdgpu_dm_crtc_handle_vblank() is called

       Inside amdgpu_dm_crtc_handle_vblank():
         acrtc->event   = cursor_event_A  (non-NULL)
         pflip_status   = AMDGPU_FLIP_NONE (0) ← satisfies != SUBMITTED
         → drm_crtc_send_vblank_event(crtc, cursor_event_A)
           ↳ fires event->base.completion = &commit_A->flip_done  ✓
         drm_crtc_vblank_put(crtc)      // refcount: 1→0
         acrtc->event = NULL

       drm_atomic_helper_wait_for_flip_done() EXIT 0ms  ← instant ✓

T=2ms  New plane flip commit begins for CRTC 0 (framebuffer change)
       amdgpu_dm_commit_planes() → Path A (plane flip)
         drm_crtc_vblank_get(crtc0)       // refcount: 0→1
         prepare_flip_isr():
           acrtc->event = flip_event_B    // store in shared slot
           pflip_status = SUBMITTED

T=3ms  dm_pflip_high_irq fires for CRTC 0
       pflip_status == SUBMITTED ✓
       e = acrtc->event (= flip_event_B)
       acrtc->event = NULL
       drm_crtc_send_vblank_event(crtc, flip_event_B)
         ↳ fires event->base.completion = &commit_B->flip_done  ✓
       drm_crtc_vblank_put()     // refcount: 1→0
       pflip_status = NONE

       drm_atomic_helper_wait_for_flip_done() EXIT 1ms  ← correct ✓

T=4ms  ANOTHER plane flip commit for CRTC 0 (this is the one that will hang)
       amdgpu_dm_commit_planes() → Path A
         drm_crtc_vblank_get(crtc0)       // refcount: 0→1
         prepare_flip_isr():
           acrtc->event = flip_event_C    // store in shared slot
           pflip_status = SUBMITTED

       drm_atomic_helper_commit_hw_done()
       drm_atomic_helper_wait_for_flip_done() ENTER for commit_C

       *** HERE IS THE RACE ***
       At T=3ms, the off-delay timer fires AGAIN before dm_pflip_high_irq
       drm_vblank_disable_and_save() runs

       amdgpu_dm_crtc_handle_vblank():
         acrtc->event   = flip_event_C  (non-NULL)
         pflip_status   = AMDGPU_FLIP_SUBMITTED ← NOT NONE
         → condition (pflip_status != SUBMITTED) is FALSE
         → event NOT delivered (correct — this path would be wrong)

       But dm_pflip_high_irq was already queued BEFORE the refcount hit 0...
       Actually: with disable_immediate=false, vblank gets disabled by the timer.
       The hardware may already have latched the new buffer, but the IRQ is now
       suppressed or missed.

       OR: the prior cursor event race played out as in the bpftrace:
       cursor event at T=1ms consumed the shared slot early, then the flip at
       T=2ms stored its event, dm_pflip_high_irq at T=3ms delivered it — but
       that was commit_B not commit_C.  commit_C's event was overwritten and
       dm_pflip_high_irq has no event to deliver for commit_C.

...10 seconds of silence...
T=10004ms  drm_atomic_helper_wait_for_flip_done() TIMEOUT
           [CRTC:283:crtc-0] flip_done timed out
```

---

## 8. The Proposed Fix

The patch from Michele Palazzi modifies the cursor-only path in
`amdgpu_dm_commit_planes()`:

```diff
// drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm.c
 } else if (cursor_update && acrtc_state->active_planes > 0) {
     spin_lock_irqsave(&pcrtc->dev->event_lock, flags);
     if (acrtc_attach->base.state->event) {
-        drm_crtc_vblank_get(pcrtc);
-        acrtc_attach->event = acrtc_attach->base.state->event;
+        drm_crtc_send_vblank_event(pcrtc, acrtc_attach->base.state->event);
         acrtc_attach->base.state->event = NULL;
     }
     spin_unlock_irqrestore(&pcrtc->dev->event_lock, flags);
```

Instead of storing the event in the shared `acrtc->event` slot and deferring
delivery to `amdgpu_dm_crtc_handle_vblank()`, the event is **sent immediately**
via `drm_crtc_send_vblank_event()`, which:

1. Reads the current vblank sequence number and timestamp
2. Fills `e->event.vbl.sequence` and the timestamp fields
3. Calls `drm_send_event_locked()` → wakes up userspace
4. Fires `event->base.completion` → unblocks `flip_done`

`drm_crtc_send_vblank_event()` itself (from `drm_vblank.c:1138`):
```c
void drm_crtc_send_vblank_event(struct drm_crtc *crtc,
                                struct drm_pending_vblank_event *e)
{
    struct drm_device *dev = crtc->dev;
    u64 seq;
    unsigned int pipe = drm_crtc_index(crtc);
    ktime_t now;

    if (drm_dev_has_vblank(dev)) {
        seq = drm_vblank_count_and_time(dev, pipe, &now);
    } else {
        seq = 0;
        now = ktime_get();
    }
    e->pipe = pipe;
    send_vblank_event(dev, e, seq, now);  // fires flip_done completion
}
```

### Why this works

- The cursor update is **committed to hardware immediately** (AMD hardware
  latches cursor position on the next scanline, not the next vblank)
- The event slot `acrtc->event` is never populated for cursor commits
- `amdgpu_dm_crtc_handle_vblank()` cannot steal a cursor event, because there
  is no cursor event pending in the shared slot
- No `drm_crtc_vblank_get()` is needed, so the refcount is not held
- Race window for both race conditions is eliminated

### The trade-off

`drm_crtc_send_vblank_event()` uses the **last recorded vblank timestamp**,
which may be slightly stale (not from the exact vblank that follows the cursor
update). Michel Dänzer noted this in the review:

> "Can this code run before start of vblank? If yes, the event would have the
> wrong sequence number and timestamp. Compositors actually make use of the
> timestamp for frame scheduling."

The ideal fix (acknowledged by Leo Li of AMD) would be to add a **separate
`cursor_event` field** in `struct amdgpu_crtc`, distinct from the `event` field
used by plane flips, and deliver it from `dm_crtc_high_irq()` when hardware
latches the new cursor. This would:

- Eliminate the shared-slot race entirely (cursor and flip events never
  compete for the same field)
- Deliver the event with the correct vblank timestamp from the real hardware
  latch interrupt
- Mirror how `dm_pflip_high_irq` handles plane flips with proper timestamps

---

## 9. Open Questions and Ongoing Discussion

The mailing list thread (February–March 2026) raised several unresolved points:

### 1. Is the starvation race actually possible?

Michel Dänzer argued that if `drm_crtc_vblank_get()` is called correctly, the
off-delay timer cannot disable the interrupt before the handler runs, because
the refcount prevents it. Michele Palazzi and later testing showed that the
timeline can still produce hangs even when the refcount logic appears sound —
suggesting the race may be more subtle than a pure refcount failure.

### 2. The later bpftrace confirms a different manifestation

The March 2026 trace showed:

```
ARM cursor event=ffff...ce00 acrtc=ffff...7000 [CRTC 0]
commit_hw_done
WAIT_FLIP ENTER [tid=203071]
... 10252ms silence ...
no dm_crtc_high_irq on CRTC 0 during the wait
692 dm_crtc_high_irq fired, all on CRTC 1
WAIT_FLIP TIMEOUT
```

This indicates `dm_crtc_high_irq` stopped firing entirely on CRTC 0 — not just
a race between cursor and flip. Leo Li speculated this could be caused by
**DGPU idle power optimizations** hanging the timing generator, which is a
separate issue from the original race.

### 3. Status

- The `drm_crtc_send_vblank_event()` patch has `Reviewed-by: Leo Li` and
  `Reviewed-by: Alex Deucher` (implied by Alex's question about merging)
- Leo Li sent a reworked version using a separate `cursor_event` field, which
  Michele reports still causes timeouts in some cases, pointing to additional
  root causes
- The investigation is ongoing; the immediate fix reliably prevents the original
  race even if it doesn't address all possible hang paths
