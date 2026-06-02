
# MRC: Multipath Reliable Connection

> This document assumes familiarity with RDMA, InfiniBand RC transport, and RoCEv2 packet format. For background, see the [RDMA Primer](https://github.com/ManiAm/RDMA-Primer) project. For the load balancing challenges that motivated MRC, see [RoCEv2 Load Balancing](./03_README_ROCE_LB.md).

## Overview

RoCEv2 transports InfiniBand packets over Ethernet by encapsulating them inside UDP/IP. The InfiniBand specification defines three primary transport services — RC, UC, and UD (see the [InfiniBand deep dive](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/02_README_INFINIBAND.md#transport-services)) — and RoCEv2 reuses them directly. **Multipath Reliable Connection (MRC)** adds multipath capabilities on top of RC, exclusively in the RoCEv2/Ethernet context. It does not define a new InfiniBand transport service, nor does it operate over native InfiniBand fabrics. MRC was released as an [open specification](https://www.opencompute.org/documents/ocp-mrc-1-0-pdf) (Revision 1.0) through the Open Compute Project (OCP) in March 2026.

MRC was developed collaboratively by AMD, Broadcom, Intel, Microsoft, NVIDIA, and OpenAI. The specification builds on concepts from the [UltraEthernet Transport Specification](https://ultraethernet.org/) (UET 1.01), referencing it for congestion control algorithms, packet trimming behavior, and several protocol details. MRC has been implemented in 400 and 800 Gb/s NICs (NVIDIA ConnectX-8, AMD Pollara/Vulcano, Broadcom Thor Ultra) and is deployed in production across OpenAI's largest training clusters, Microsoft's Fairwater data centers, and Oracle Cloud Infrastructure's (OCI) Abilene facility, where it has been used to train frontier large language models.

MRC eliminates the fundamental constraints that make standard RoCEv2 RC unsuitable for large-scale AI training. The following table summarizes these constraints and the corresponding MRC solutions:

| Problem                           | RC Constraint                                  | MRC Solution |
|-----------------------------------|------------------------------------------------|--------------|
| Single-path flow pinning          | All packets in a QP share one ECMP path        | Per-packet spraying across hundreds of paths via Entropy Values (EVs) |
| Out-of-order delivery is fatal    | Only the first packet carries the RETH address | Full RETH in every packet — immediate placement regardless of arrival order |
| Go-Back-N amplifies loss          | One lost packet retransmits half the window    | Selective retransmission via SACK bitmap — retransmit only what is missing |
| PFC causes collateral damage      | Lossless Ethernet required                     | PFC disabled; lossy Ethernet with packet trimming for fast loss notification |
| No path-level congestion response | DCQCN reduces rate globally                    | NSCC: ECN + RTT driven window control combined with per-EV path avoidance |
| Slow failure recovery             | Subnet Manager re-sweeps (seconds)             | Per-EV state machine with sub-millisecond failover and probe-based self-healing |

MRC supports only **RDMA Write** and **Write-with-Immediate (WriteIMM)** operations — Read, Send, and Atomic operations are not supported. This deliberate restriction reflects the dominant operations in AI collective communication libraries (such as NCCL), allowing significant transport simplification with no practical impact on AI training workloads.

In production at OpenAI and Microsoft, MRC has demonstrated link-flap tolerance during frontier model training, sub-second switch failure recovery, and near-line-rate throughput under 0.1% packet loss — scenarios where standard RoCEv2 performance collapses.


## Multi-Plane Topology

MRC was co-designed with a specific network topology that maximizes its strengths. Understanding this topology is essential context for the protocol mechanics that follow.

Instead of treating an 800 Gb/s NIC as a single high-speed link, the NIC is broken out into multiple smaller links (for example, 8 × 100 Gb/s). Each link connects to a different top-of-rack (T0) switch, creating **eight independent parallel networks** called planes. In MRC's device model, each panel port maps directly to a plane — every port is exposed as a separate PCIe physical function with its own netdev interface and management IP address.

Using 51.2 Tb/s switches, each switch at 100 Gb/s has 512 ports instead of 64 ports at 800 Gb/s. With 512-port switches, a two-tier Clos topology can connect over **131,000 GPUs** — a scale that would require three or four tiers with conventional 800 Gb/s links.

The multi-plane design delivers several advantages:

- **Lower latency**: The longest path traverses 3 switches instead of 5 or 7.
- **Higher redundancy**: Losing a single T0–T1 link reduces a node's capacity by only ~0.4% (in an 8-plane network) versus ~3% in a single-plane design.
- **Graceful NIC port failure**: If one of eight NIC ports fails, the node loses only 12.5% of bandwidth. MRC detects the failure, remaps EVs to avoid the failed plane, and sends a Port Status Update to notify remote peers — the training job continues without interruption.
- **Reduced cost and power**: A two-tier multi-plane network requires roughly 60% of the switches and 67% of the optics compared to an equivalent three-tier single-plane network.

MRC distributes its EV set equally across all planes. When the NIC scheduler is ready to send a packet, it calls `GetSendParams()` with a bitmap indicating which ports currently have space for transmission, and selects an EV from the corresponding plane — skipping EVs whose plane is not ready. This tight coupling between topology and transport ensures traffic is inherently balanced across planes.


## Routing Modes

With the multi-plane topology established, the next question is how MRC directs individual packets along specific paths. MRC supports three forwarding models that determine how Entropy Values are encoded into packets and how switches use them to select output ports. All three coexist in the specification; the choice depends on switch capabilities and deployment requirements.

### ECMP

In ECMP mode, the EV is embedded in the packet's UDP source port (and IPv6 flow label, if present). Switches apply their standard hash function to these fields to select an ECMP next-hop. The sender rotates EVs to distribute packets across different hash buckets, but the exact path each EV takes is determined by the switch's hash configuration and is not directly known to the sender.

ECMP mode requires no switch modifications beyond standard ECMP support. It provides the weakest path determinism — different EVs may collide on the same hash bucket, and the sender cannot precisely identify which physical path a given EV traverses — but it is the simplest to deploy.

### Structured Entropy Values

Structured EV provides source-routing-like determinism without full IPv6-in-IPv6 encapsulation. A 32-bit entropy field — the lower 16 bits of the IPv6 flow label concatenated with the 16-bit UDP source port — is partitioned into multiple hop-specific subfields. Each subfield encodes a forwarding decision for a specific network tier:

```text
Example: 3-hop topology with 10-bit, 8-bit, and 4-bit subfields

  ┌──────────┬────────┬──────┬──────────┐
  │  hop0    │  hop1  │ hop2 │ reserved │
  │  (10b)   │  (8b)  │ (4b) │          │
  └──────────┴────────┴──────┴──────────┘
  T0 switch    T1 switch  T2 switch

Each switch examines its designated hop subfield to select an egress port.
```

The number of subfields, their widths, and valid value ranges are configured at deployment time and known to all endpoints and switches. Each switch uses ACLs or static forwarding tables to map its hop subfield to an egress port. This gives the sender explicit control over path selection at each tier while using standard L3/L4 header fields with no encapsulation overhead.

### SRv6 Source Routing

For maximum path control and observability, MRC uses **IPv6 Segment Routing (SRv6)** with static forwarding tables in the switches. SRv6 encodes the complete path as a sequence of 16-bit micro-Segment IDs (uSIDs) packed into an IPv6 destination address:

1. At QP startup, the NIC (or MRC Controller) generates the EV set and maps each EV to a specific SRv6 destination address. This address encodes the full path as a stack of uSIDs, each identifying a specific switch. An optional Segment Routing Header (SRH) using Compressed Segment List Encoding (RFC 9800) can extend the uSID stack beyond a single 128-bit address or carry a copy of the original path for debugging.

2. When a packet is sent, the NIC encapsulates it as **IPv6-in-IPv6**: the outer header carries the SRv6 path, and the inner header carries the destination NIC's actual address for decapsulation.

3. At each hop, the switch identifies its own uSID, performs a **left-shift** of the address by 16 bits to expose the next hop's uSID, and forwards the packet out the corresponding port using a static forwarding table (configured once at installation and never changed by dynamic routing protocols).

Because the forwarding tables are static and the path is fully determined by the sender, there is no routing convergence delay, no ECMP hash ambiguity, and no switch-level computation. Failures are handled exclusively by MRC removing the corresponding EV from its active set.

This design provides excellent **observability**: because each EV deterministically maps to a specific physical path, when MRC reports a bad EV, operators can immediately identify the exact failed link or switch — something that is extremely difficult with hash-based ECMP, where the mapping from flow to path is opaque.

The decision to disable dynamic routing in SRv6 deployments is deliberate. Having two adaptive mechanisms — MRC at the endpoints and dynamic routing in the switches — creates unpredictable interactions: MRC avoids a failed path, then routing re-converges and changes the ECMP-to-port mappings, disturbing MRC's load balancing. With static routes, the control plane is simplified to a single adaptive layer at the transport.


## Traffic Classes

Before describing the protocol mechanics, it is important to understand the priority structure MRC uses to separate control feedback from data. MRC requires at least two traffic classes, separated by DSCP codepoints:

| Traffic Class | Packets | DSCP Codepoints |
|---------------|---------|-----------------|
| **High priority (Control)** | SACKs, NACKs, Transport ACKs, Trimmed packets | `DSCP_CONTROL`, `DSCP_TRIMMED`, `DSCP_TRIMMED_LASTHOP` |
| **Data** | Write requests, Reliability Probe requests, Retransmissions | `DSCP_TRIMMABLE`, `DSCP_NO_TRIM`, `DSCP_TRIMMABLE_RETX` |

Retransmitted data uses a distinct DSCP (`DSCP_TRIMMABLE_RETX`) so switches can apply different queuing or marking policies to retransmissions. The control class should be configured as high-priority best-effort without trimming or WRED, ensuring that SACK/NACK feedback reaches the sender promptly even during heavy data congestion. The `DSCP_NO_TRIM` codepoint is used when the network does not support trimming.


## How MRC Works

With the topology, routing, and traffic class context established, this section describes how the MRC transport protocol operates.

### Packet Spraying and Entropy Values

The most fundamental change in MRC is the elimination of single-path flow pinning. Instead of binding a QP to one ECMP path, MRC **sprays** packets from a single QP across hundreds of network paths simultaneously, spanning all planes in the multi-plane topology.

At QP startup, the sender is assigned an **Entropy Value (EV) Profile** — a set of EVs, each mapping to a unique path through the network. The sender rotates through the active EV set, embedding a different EV in each packet's header fields. Depending on the routing mode, the EV is encoded in the UDP source port and IPv6 flow label (for ECMP or Structured EV routing) or in an SRv6 outer destination address (for source-routed deployments). Each EV causes the packet to traverse a different physical path, transforming what was a single "elephant flow" into a fine-grained spray distributed evenly across the fabric.

Because the spray is per-packet rather than per-flow, load balancing operates at the finest possible granularity. The MRC specification recommends pseudo-random EV selection to avoid path synchronization between different QPs sharing the same EV profile. The aggregate effect across hundreds of QPs naturally distributes load without inter-sender coordination.

### Out-of-Order Delivery with Immediate Memory Placement

Packet spraying means packets traverse paths of different lengths and congestion states, so they inevitably arrive out of order. This is fatal for standard RC — the RETH with the destination address is only in the first packet, so subsequent packets depend on in-order arrival (see [The RC Ordering Constraint](./03_README_ROCE_LB.md#the-rc-ordering-constraint)).

MRC solves this by including the **full RDMA Extended Transport Header (RETH) in every data packet** — first, middle, last, and only. Each RETH carries the virtual address, `R_Key`, and DMA length. The virtual address is incremented by the PMTU for each successive packet, making every packet fully self-describing. The receiving NIC can write each packet directly to its correct position in the application's memory buffer the instant it arrives, regardless of arrival order. No reordering buffer is needed, and no packet is discarded simply because it arrived ahead of an earlier one.

For WriteIMM operations, MRC adds a **Message Extended Transport Header (METH)** carrying a Message Sequence Number (MSN) and a Receive Queue MSN (RQMSN) to track in-flight WriteIMM requests. Because Immediate Data values may arrive out of order, the responder stashes them until all preceding requests have been received and the completion can be delivered in send order. The maximum number of in-flight WriteIMM operations (`max_wimm_inflight`) is negotiated during QP setup — exceeding this limit transitions the QP to an error state. MRC does not support RNR-NAK flow control; the application (e.g., NCCL) is responsible for ensuring in-flight WriteIMM requests stay within the advertised limit.

### Selective Retransmission (SACK/NACK)

With out-of-order delivery handled, MRC replaces Go-Back-N with **selective retransmission**. MRC introduces **Reliability SACKs** — logically separate from standard RDMA Transport ACKs — that provide fine-grained delivery feedback. Each SACK contains:

- A **cumulative ACK PSN** (`cack_psn`): the highest sequence number before the first gap, confirming all prior packets have arrived.
- A **64-bit bitmap**: each bit indicates whether a specific packet (relative to a `sack_offset` from `cack_psn`) has been received, allowing the sender to identify exactly which packets are missing.
- **Reflected entropy**: the EV from the data packet that triggered the SACK, enabling the sender to correlate loss feedback to a specific path.
- **Congestion state** (`CC_STATE`): a reflected timestamp for RTT measurement, an out-of-order count, a receiver congestion penalty, and a received byte count — all inputs to the congestion control algorithm.

When a gap is detected, the sender retransmits only the specific lost packets (marked with a `rtx` flag in the BTH header) rather than the entire window. **Reliability NACKs** provide explicit negative acknowledgment with a reason code — `TRIMMED` (packet was trimmed by a switch), `NO_BITMAP` (responder tracking resources exhausted), `NO_PKT_BUFFER` (responder buffer full), `PSN_OOR_WINDOW` (PSN outside the responder's tracking window), or `UNEXP_EVENT` (fatal error) — so the sender can distinguish between network loss, responder resource exhaustion, and unrecoverable errors.

The bandwidth savings from selective retransmission are substantial:

```text
At 800 Gb/s with 9 KB jumbo frames:
  Packets in flight (BDP) = 800 Gbps × 10 µs RTT = 1 MB ≈ 108 packets

Go-Back-N (standard RC):
  1 lost packet → retransmit from lost PSN onward ≈ N/2 = ~54 packets
  Throughput loss per event ≈ 50%
  At 1% loss rate:  effective throughput ≈ 65%  → 35% bandwidth wasted

Selective Retransmission (MRC):
  1 lost packet → retransmit only that 1 packet
  At 1% loss rate:  effective throughput > 99%  → <1% bandwidth wasted
```

### Operating on Lossy Ethernet

#### Disabling PFC

Because MRC sprays a single QP's packets across hundreds of paths, a flow reaches the last-hop switch over many different ingress links. PFC, which pauses an entire ingress port, would indiscriminately throttle packets from many unrelated flows that happen to share those links. MRC therefore **disables PFC entirely** and runs Ethernet in best-effort (lossy) mode.

This is a deliberate trade-off: accepting occasional packet loss in exchange for eliminating PFC-induced [head-of-line blocking and congestion spreading](https://github.com/ManiAm/GNS-QOS/blob/master/docs/04_PFC.md#the-dangers-of-pfc-managing-the-lossless-safety-net). The selective retransmission mechanism handles the resulting losses efficiently, making the system more predictable under stress.

#### Packet Trimming

To accelerate loss recovery, especially during incast (many-to-one) traffic patterns, MRC supports **packet trimming** as defined in the UltraEthernet specification. When a switch would otherwise drop a packet due to buffer overflow, it instead strips the payload and priority-forwards only the header to the destination on the high-priority control traffic class (using a `DSCP_TRIMMED` codepoint). The receiving NIC recognizes the trimmed packet and can generate a NACK with reason code `TRIMMED`, triggering fast retransmission.

Packet trimming serves a dual purpose: it provides faster loss notification than timeout-based detection, and it helps MRC distinguish **congestion-induced loss** (trimmed packets arrive as headers) from **path failure** (packets vanish entirely). This distinction is critical for making correct EV state decisions — congestion triggers a temporary skip, while path failure triggers longer-term removal.

Generating trim NACKs is an **optional** responder capability, negotiated per-connection via the `MRC_DEVICE_CAP_TRIM_NACK` attribute during QP setup. When the responder does not support trim NACKs, the requestor may use alternative techniques — such as marking packets as non-trimmable or sending periodic reliability probes — to accelerate loss detection.

### Congestion Control (NSCC)

MRC adopts **NSCC (Network-Signalled Congestion Control)**, a sender-side, SACK-clocked, window-based congestion control algorithm defined in the UltraEthernet Transport Specification. NSCC dynamically adjusts a per-QP congestion window (`cwnd`) to bound the amount of outstanding data in-flight and keep queueing delay within a configurable target (`target_Qdelay`). A sender can only transmit when `cwnd > inflight`.

NSCC relies on two mandatory network signals:

- **Request RTT**: a lagging indicator that estimates queueing delay by measuring the round-trip time between a data packet and its corresponding SACK. MRC supports two mechanisms for RTT measurement: a **Timestamp Extended Header (TSETH)** that embeds a 16-bit transmit timestamp directly in data packets (reflected by the responder in the SACK's `CC_STATE`), or a local send-time database at the requestor.

- **ECN (Explicit Congestion Notification)**: a leading indicator where switches mark packets when queue depth exceeds a threshold. The responder echoes the ECN signal back to the sender via the SACK's `M` field, tagged with the specific EV that experienced congestion.

These two signals combine into a four-quadrant response:

| ECN | Request RTT | Inferred Network State | Window Adjustment |
|-----|-------------|------------------------|-------------------|
| Not set | Below target | Uncongested | Proportional increase |
| Not set | Above target | Recovering from congestion | Fair increase |
| Set | Above target | Actively congested | Multiplicative decrease |
| Set | Below target | Transient congestion | No change |

This dual-signal approach is more precise than [DCQCN](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/03_README_ROCE.md#dcqcn-and-congestion-control), which relies on ECN alone. NSCC only decreases the window when both ECN and elevated RTT confirm sustained congestion, avoiding unnecessary rate reduction from transient ECN marks. More importantly, NSCC **combines** rate control (adjusting `cwnd`) **with** path-level load balancing (temporarily skipping congested EVs, described in the next section). DCQCN can only reduce the sender's overall injection rate; NSCC can also redirect traffic to a better path while maintaining full throughput.

MRC also supports **responder flow control**: the responder can signal back-pressure via a `rcv_cwnd_pen` (receiver congestion-window penalty) field in SACKs, asking the sender to reduce its congestion window when the responder itself is becoming overloaded. The penalty ranges from 0 (no effect) to 127 (reduce to one packet per RTT). This handles receiver-side bottlenecks — a distinct concern from network congestion.

Each requestor maintains a **QP Congestion Controller (QPCC)**, functionally equivalent to the UltraEthernet Congestion Control Context (CCC). The NIC runs a scheduler with four QP states — **Idle** (no data), **Active** (data queued but window full), **Ready** (data queued and window permits sending), and **Pending** (data in flight but nothing new to send). The scheduler rotates among QPs in the Ready state. The EV for each packet is selected at send time via `GetSendParams()` so that path decisions reflect the most current congestion feedback — incoming SACKs or NACKs between a QP becoming Ready and the packet actually being sent can change the choice of EV and port.

### EV State Machine and Adaptive Load Balancing

MRC maintains per-EV state to track path health. Each EV in the profile exists in one of four states:

| EV State | Active/Inactive | Description |
|----------|----------------|-------------|
| **GOOD** | Active | Path is healthy; EV is available for packet transmission |
| **SKIP** | Inactive | Path recently experienced congestion (ECN-marked); temporarily bypassed, automatically returns to GOOD after being rotated past |
| **ASSUMED_BAD** | Inactive | Path failure detected (timeout, persistent loss); requires probe-based recovery before reuse |
| **DENIED** | Inactive | Administratively disabled by the MRC Controller; never auto-recovers |

The sender selects EVs only from the Active set. Transitions between states are driven by SACK and NACK feedback:

- When a SACK arrives with the `M` field set to **SKIP_ONCE** (0b01, indicating ECN marking on the forward path), the corresponding EV transitions from GOOD → **SKIP**. The sender bypasses it for the current rotation cycle and resets it to GOOD on the next pass. This provides short-lived congestion avoidance that naturally smooths out momentary queue buildup.
- When a SACK arrives with `M` set to **ALWAYS_SKIP** (0b10), the EV transitions to **ASSUMED_BAD**, indicating the responder has identified a persistently bad path.
- When a NACK with reason `TRIMMED` (but not `TRIMMED_LASTHOP`) arrives, the corresponding EV transitions to **SKIP** — the trim indicates congestion, not path failure.
- When a packet times out without any trim notification or SACK acknowledgment, the EV transitions to **ASSUMED_BAD** — the lack of any response suggests path failure.
- The MRC Controller can set any EV to **DENIED** to administratively exclude a known-bad path, or restore it to **GOOD** when the underlying issue is resolved.

This four-state model provides two distinct levels of load balancing response. Transient congestion triggers SKIP — a lightweight, self-recovering avoidance that redistributes one rotation's worth of traffic. Persistent failures trigger ASSUMED_BAD, which removes the EV from service until probing confirms recovery. Administrative control via DENIED allows operators to preemptively remove paths during maintenance.

### Path Failure Detection and Recovery

MRC defines two distinct probe mechanisms for detecting and recovering from path failures:

**Reliability Probes** operate at connection scope. They are addressed to the peer QP (using the standard QP number), do not consume PSNs, and elicit a SACK response containing the responder's current bitmap state and congestion information. The sender can target specific paths by varying the probe's EV. Reliability Probes are useful for two purposes: testing whether a specific path can still reach the peer, and obtaining fresh acknowledgment state when the sender suspects that SACKs have been lost.

**EV Probes** operate at node scope. They are **connectionless** Endpoint Operations (using the reserved QPN 0x2) and are not tied to any specific QP. The MRC Controller uses EV Probes to measure forward-path reachability and round-trip latency independently of active QP traffic. Each probe is tagged with a unique identifier that the responder echoes in the reply, enabling precise request-response matching. EV Probes allow the controller to test paths before re-enabling previously failed EVs — for example, verifying that a repaired link is actually forwarding before moving its EVs back to GOOD state.

For EVs in the ASSUMED_BAD state, the sender periodically sends probes on the retired path. If a probe elicits a SACK with the `M` field set to `NONE` (no congestion), the EV transitions back to GOOD. If `M` is `SKIP_ONCE`, it transitions to SKIP. This creates a self-healing loop: failures are bypassed almost instantly, and recovered paths are automatically brought back into service.

**Port Status Updates** complement probes with proactive NIC-to-NIC notification. When a NIC port fails or recovers, the NIC sends a Port Status Update — another connectionless Endpoint Operation — containing a `port_status_mask` bitmap where each bit represents the reachability state of a local port. This allows the peer to adjust its EV selection immediately rather than waiting for timeouts on the affected paths.


## Software and Control Plane

MRC exposes its functionality through **libmrc**, a library with two distinct APIs:

- **`mrc.h`** (Application API): Mirrors `libibverbs` semantics for creating, modifying, and destroying MRC QPs and CQs, posting work requests, and polling for completions. Applications continue to use `libibverbs` for device discovery and context management (`ibv_open_device`, `ibv_query_port`); `libmrc` handles MRC-specific QP configuration via `mrc_modify_qp()`.

- **`mrc_ctl.h`** (Controller API): A privileged interface (requires `CAP_NET_ADMIN`) for the **MRC Controller** — a per-node daemon or CLI tool responsible for:
  - **EV Profile management**: Creating profiles in EXPLICIT mode (controller programs each EV value), GENERATED mode (NIC generates EVs from parameters defining hop widths and valid value ranges per tier), or AUTO mode (vendor-defined default behavior, e.g., ECMP-style hashing).
  - **CC Profile management**: Configuring NSCC parameters (target queueing delay, initial congestion window) per destination or group of QPs.
  - **EV state management**: Setting individual EVs to DENIED or GOOD state, and receiving asynchronous EV events when the NIC detects a bad path — enabling the controller to correlate failures across QPs and identify the affected link.
  - **EV Probes**: Sending connectionless probes to test path reachability independently of active QP traffic.

Connection setup requires **out-of-band QP attribute exchange** — RDMA-CM is not supported. During QP setup, peers exchange capability attributes including the responder's WriteIMM in-flight limit (`MRC_QP_MAX_WIMM_DEST`), maximum PSN tracking range (`MRC_QP_MAX_MPR_DEST`), and optional feature flags for Dynamic MPR (allowing the responder to adjust the in-flight PSN window at runtime), Trim NACK generation, and Service Time reporting.


## MRC vs. InfiniBand RC: Key Differences

MRC and InfiniBand RC serve fundamentally different deployment contexts. The following table summarizes the core architectural differences:

| Aspect                       | InfiniBand RC                              | MRC (extends RoCEv2 RC)                         |
| ---------------------------- | ------------------------------------------ | ----------------------------------------------- |
| Network Type                 | Native InfiniBand (lossless fabric)        | Ethernet (best-effort / lossy)                  |
| Path Model                   | Single path per QP                         | Hundreds of paths per QP (packet spraying)      |
| Packet Ordering              | Strict in-order delivery required          | Out-of-order delivery with immediate placement  |
| Loss Recovery                | Go-Back-N (retransmit entire window)       | Selective retransmission (SACK/NACK with bitmap)|
| Flow Control                 | CBFC (per-VL credit-based, lossless)       | No PFC; lossy Ethernet with packet trimming     |
| Congestion Control           | FECN/BECN reduces sender injection rate    | NSCC: ECN + RTT window control + path avoidance |
| Failure Detection            | Subnet Manager re-sweeps (seconds)         | Per-EV state machine + probes (sub-millisecond) |
| Routing                      | SM-computed LFTs in switches               | ECMP, Structured EV, or SRv6 source routing     |
| Supported Operations         | All (Send/Recv, Write, Read, Atomics)      | RDMA Write and Write-with-Immediate only        |
| Target Scale                 | Thousands of nodes (single subnet)         | 100,000+ GPUs (multi-plane Ethernet)            |

MRC is not a replacement for InfiniBand RC in all scenarios. It targets a specific and critical use case: sustaining predictable, high-throughput collective communication across massive Ethernet-based AI training clusters where flow collisions, PFC storms, and slow failure recovery would otherwise cripple training efficiency.


## Production Impact

In production deployments at OpenAI and Microsoft, MRC has demonstrated several concrete operational improvements:

- **Link flap tolerance**: During training of frontier models, multiple link flaps per minute between T0 and T1 switches had no measurable impact on synchronous pretraining. Repair of these links became a low-priority maintenance activity rather than an urgent operational event.

- **Switch failure resilience**: When T1 switches had to be rebooted during active training, MRC progressively detected that the affected EVs were failing — each individual EV was moved to ASSUMED_BAD state and traffic redistributed across remaining paths. The full recovery completed within seconds. The training job continued without coordination between network operations and the training team.

- **NIC port failure survival**: Before MRC, a failed link between a GPU's NIC and its T0 switch would crash the training job. With MRC, the NIC sends a Port Status Update to peers, EVs for the failed plane are removed from active sets, and the job survives with reduced bandwidth (losing one of eight planes reduces capacity by 12.5%). Most such failures recover within a minute.

- **Performance under loss**: In controlled experiments comparing MRC against RoCEv2 on identical hardware, a single MRC QP spraying across 256 paths outperformed 16 RoCEv2 QPs in all-reduce collectives. Under 0.1% induced packet loss, MRC maintained near-line-rate throughput for large messages, while RoCEv2 performance collapsed due to Go-Back-N amplification and PFC-induced blocking.

- **Zero collateral damage**: In 7-to-1 incast experiments, MRC perfectly shared the bottleneck link among incast flows with zero impact on a concurrent "victim" flow on the same fabric. RoCEv2 with DCQCN degraded the victim flow's throughput by 25–75% depending on QP count.
