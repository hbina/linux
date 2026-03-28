# SolarFlare EF100: RX Path from MSI-X Interrupt to tcpdump

## Overview

Two CPUs are involved. The IRQ CPU services the NIC and feeds the kernel's packet
socket. The tcpdump CPU (CPU 3 on this host) drains the packet socket and writes
the pcap file. The interrupt never reaches tcpdump directly.

---

## Step-by-step with kernel source locations

### 1. NIC DMA-writes the packet into host memory

The NIC uses a descriptor pre-posted by the driver in `ef100_rx_write()`:

```
drivers/net/ethernet/sfc/ef100_rx.c:192  ef100_rx_write()
```

The driver writes the DMA address into the RX ring descriptor
(`ESF_GZ_RX_BUF_ADDR`), then rings the doorbell (`ER_GZ_RX_RING_DOORBELL`).
The NIC reads this to know where to DMA incoming frames.

### 2. NIC raises the MSI-X interrupt

The NIC signals one MSI-X vector per channel. The vector was allocated
during probe and bound to a specific CPU via `irq_set_affinity_hint()`:

```
drivers/net/ethernet/sfc/efx_channels.c:363  efx_set_interrupt_affinity()
drivers/net/ethernet/sfc/efx_channels.c:378      irq_set_affinity_hint(channel->irq, cpumask_of(cpu))
```

On this host the RX IRQs are on CPUs 0, 1, and 23.

### 3. Hard interrupt handler — `ef100_msi_interrupt()`

```c
// drivers/net/ethernet/sfc/ef100_nic.c:328
static irqreturn_t ef100_msi_interrupt(int irq, void *dev_id)
{
    struct efx_msi_context *context = dev_id;
    struct efx_nic *efx = context->efx;

    if (likely(READ_ONCE(efx->irq_soft_enabled))) {
        if (context->index == efx->irq_level)
            efx->last_irq_cpu = raw_smp_processor_id();

        /* Schedule processing of the channel */
        efx_schedule_channel_irq(efx->channel[context->index]);
    }
    return IRQ_HANDLED;
}
```

This is the entire hard IRQ handler. It does almost nothing except schedule
NAPI polling. It does NOT touch any packet data.

### 4. `efx_schedule_channel_irq()` → `napi_schedule()`

```c
// drivers/net/ethernet/sfc/efx_common.h:71
static inline void efx_schedule_channel(struct efx_channel *channel)
{
    napi_schedule(&channel->napi_str);
}

static inline void efx_schedule_channel_irq(struct efx_channel *channel)
{
    channel->event_test_cpu = raw_smp_processor_id();
    efx_schedule_channel(channel);
}
```

`napi_schedule()` raises the `NET_RX_SOFTIRQ` on the current CPU. The
hard IRQ returns and the CPU runs the softirq shortly after.

### 5. NAPI poll — `efx_poll()`

```c
// drivers/net/ethernet/sfc/efx_channels.c:1252
static int efx_poll(struct napi_struct *napi, int budget)
{
    struct efx_channel *channel = container_of(napi, ...);
    int spent;

    spent = efx_process_channel(channel, budget);

    if (budget)
        xdp_do_flush();    // flush any XDP redirects

    if (spent < budget) {
        // IRQ rate adaptation
        if (napi_complete_done(napi, spent))
            efx_nic_eventq_read_ack(channel);  // re-arm the event queue
    }
    return spent;
}
```

Registered with `netif_napi_add(channel->napi_dev, &channel->napi_str, efx_poll)`
at `efx_channels.c:1303`.

### 6. `efx_process_channel()` — drain the event queue

```c
// drivers/net/ethernet/sfc/efx_channels.c:1180
static int efx_process_channel(struct efx_channel *channel, int budget)
{
    spent = efx_nic_process_eventq(channel, budget);  // reads HW event ring
    if (spent && efx_channel_has_rx_queue(channel)) {
        efx_rx_flush_packet(channel);
        efx_fast_push_rx_descriptors(rx_queue, true);  // replenish descriptors!
    }
    // ...
    netif_receive_skb_list(channel->rx_list);  // hand up to protocol stack
    return spent;
}
```

The critical call is `efx_fast_push_rx_descriptors()`. This refills the RX
ring. If the ring empties before this runs, the NIC has nowhere to put new
frames → `port_rx_nodesc_drops` increments (see below).

### 7. Event processing — `ef100_ev_process()` → `efx_ef100_ev_rx()`

```c
// drivers/net/ethernet/sfc/ef100_nic.c:258
static int ef100_ev_process(struct efx_channel *channel, int quota)
{
    while (spent < quota) {
        p_event = efx_event(channel, read_ptr);
        ev_type = EFX_QWORD_FIELD(*p_event, ESF_GZ_E_TYPE);

        switch (ev_type) {
        case ESE_GZ_EF100_EV_RX_PKTS:
            efx_ef100_ev_rx(channel, p_event);  // ← RX event
            ++spent;
            break;
        case ESE_GZ_EF100_EV_TX_COMPLETION:
            ef100_ev_tx(channel, p_event);
            break;
        // ...
        }
        ++read_ptr;
    }
    return spent;
}
```

One event can cover multiple packets (`ESF_GZ_EV_RXPKTS_NUM_PKT` field).
That's how the NIC batches: one interrupt wakes NAPI, NAPI sees one event
covering N packets.

```c
// drivers/net/ethernet/sfc/ef100_rx.c:172
void efx_ef100_ev_rx(struct efx_channel *channel, const efx_qword_t *p_event)
{
    unsigned int n_packets = EFX_QWORD_FIELD(*p_event, ESF_GZ_EV_RXPKTS_NUM_PKT);

    if (n_packets > 1)
        ++channel->n_rx_merge_events;   // batched events counter

    channel->irq_mod_score += 2 * n_packets;

    for (i = 0; i < n_packets; ++i) {
        ef100_rx_packet(rx_queue, rx_queue->removed_count & rx_queue->ptr_mask);
        ++rx_queue->removed_count;
    }
}
```

### 8. Per-packet processing — `__ef100_rx_packet()` → GRO

```c
// drivers/net/ethernet/sfc/ef100_rx.c:56
void __ef100_rx_packet(struct efx_channel *channel)
{
    // Read RX prefix (length, checksum, RSS hash, ingress mport)
    rx_buf->len = PREFIX_FIELD(prefix, LENGTH);

    // SR-IOV representor routing (if applicable)
    if (ing_port != nic_data->base_mport) { ... }

    // Checksum offload
    csum = PREFIX_FIELD(prefix, CSUM_FRAME);

    // Feed into GRO
    efx_rx_packet_gro(channel, rx_buf, channel->rx_pkt_n_frags, eh, csum);
}
```

GRO path (`rx_common.c:509`):

```c
void efx_rx_packet_gro(...)
{
    skb = napi_get_frags(napi);
    // fill skb page fragments from DMA buffer (zero-copy page flip)
    napi_gro_frags(napi);   // → eventually netif_receive_skb_list()
}
```

### 9. `netif_receive_skb_list()` → packet socket

`netif_receive_skb_list()` is called at `efx_channels.c:1220` with all packets
accumulated during the NAPI poll. The packet socket layer is a registered
protocol handler (`ETH_P_ALL`). It intercepts every packet:

```c
// net/packet/af_packet.c:2114  packet_rcv()  (non-mmap path)
static int packet_rcv(struct sk_buff *skb, ...)
{
    // BPF filter (tcpdump's compiled filter expression)
    // ...
    __skb_queue_tail(&sk->sk_receive_queue, skb);
    sk->sk_data_ready(sk);    // ← wakes tcpdump
    return 0;
}
```

For the packet_mmap path (TPACKET_V3, which libpcap uses for performance):

```c
// net/packet/af_packet.c:816
sk->sk_data_ready(sk);   // called when a block is sealed
```

`sk_data_ready` is `sock_def_readable`, which calls `wake_up_interruptible`
on the socket's wait queue. This makes tcpdump **runnable** — it was blocked
in `poll()`/`recvmsg()`.

### 10. tcpdump wakes on CPU 3

Because tcpdump is pinned to CPU 3 via `sched_setaffinity()`, the scheduler
will only dispatch it there. Once the IRQ CPU finishes the softirq and the
scheduler runs on CPU 3, tcpdump reads from its TPACKET ring and writes to
the pcap file.

---

## The `port_rx_nodesc_drops` counter

```
drivers/net/ethernet/sfc/ef100_nic.h:60   EF100_STAT_port_rx_nodesc_drops
drivers/net/ethernet/sfc/ef100_nic.c:581  EF100_DMA_STAT(port_rx_nodesc_drops, RX_NODESC_DROPS)
drivers/net/ethernet/sfc/ef100_nic.c:621  core_stats->rx_dropped = stats[EF100_STAT_port_rx_nodesc_drops] + ...
```

This counter is incremented by the **NIC firmware** (via MCDI stats DMA),
not the driver. It counts frames the NIC received but had no RX descriptor
to place them into. This happens when:

- The RX ring is full (driver hasn't replenished descriptors fast enough)
- NAPI is falling behind under high load

The descriptor refill happens in step 6 (`efx_fast_push_rx_descriptors()`).
If the IRQ CPU is under pressure, refill is delayed, the ring drains, and
the NIC starts dropping before the packet ever reaches host memory.
**tcpdump cannot recover these drops** — they happen before the `sk_buff`
is ever created.

---

## Kernel bypass (solar_capture) comparison

Under solar_capture / AF_XDP / DPDK+VFIO, the path diverges at step 3:

| Stage | tcpdump (normal) | solar_capture / kernel bypass |
|-------|-----------------|-------------------------------|
| DMA buffer | Kernel-allocated pages | User-registered UMEM / pinned hugepages |
| IRQ handler | `ef100_msi_interrupt` → NAPI | Same IRQ handler, or polling loop |
| Packet delivery | `netif_receive_skb_list` → packet socket | Zero-copy via shared ring (UMEM/AF_XDP) or full device passthrough (VFIO) |
| tcpdump/capture wakeup | `sk_data_ready` → scheduler | Direct ring consumer, no kernel wakeup needed |
| `port_rx_nodesc_drops` | Possible if NAPI falls behind | Still possible at NIC level if consumer ring is full |

For AF_XDP specifically, the XDP hook fires **inside the NAPI poll** before
`napi_gro_frags()` would be called. A BPF program can redirect to the AF_XDP
socket's umem ring instead of the normal stack — bypassing GRO, sk_buff
allocation, and the packet socket layer entirely.

```
net/core/filter.c   xdp_do_redirect()
net/xdp/xsk.c      __xsk_map_redirect()
```

---

## Summary of the two CPUs

```
CPU 0/1/23 (IRQ CPU)            CPU 3 (tcpdump CPU)
────────────────────────────    ──────────────────────────
ef100_msi_interrupt()           blocked in poll()
  └─ napi_schedule()
NET_RX_SOFTIRQ runs
  └─ efx_poll()
      └─ efx_process_channel()
          └─ ef100_ev_process()
              └─ efx_ef100_ev_rx()
                  └─ __ef100_rx_packet()
                      └─ efx_rx_packet_gro()
                          └─ napi_gro_frags()
                              └─ netif_receive_skb_list()
                                  └─ packet_rcv() / tpacket_rcv()
                                      └─ sk_data_ready()  ──────→ woken
                                                                   reads ring
                                                                   writes pcap
```
