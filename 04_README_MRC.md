# Multipath Reliable Connection (MRC)

> **Prerequisites**: This document builds on RDMA, InfiniBand RC transport, and the RoCEv2 packet format. If you are new to these topics, start with the [RDMA Primer](https://github.com/ManiAm/RDMA-Primer) project, which covers the fundamentals. For the load balancing problems that motivated MRC, see [Load Balancing for RoCEv2](./03_README_ROCE_LB.md).



## Overview

RoCEv2 carries InfiniBand packets over Ethernet by encapsulating them inside UDP/IP. The InfiniBand specification defines three primary transport services — RC, UC, and UD (see the [InfiniBand deep dive](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/02_README_INFINIBAND.md#transport-services)) — and RoCEv2 reuses them unchanged.

**Multipath Reliable Connection (MRC)** extends the Reliable Connection (RC) service so that a single connection can use many network paths at once. It is defined exclusively for the RoCEv2/Ethernet context: it is not a new InfiniBand transport service, and it does not run over native InfiniBand fabrics.

MRC was initiated by OpenAI in late 2023 and developed over two years in collaboration with AMD, Broadcom, Intel, Microsoft, and NVIDIA. OpenAI led the protocol design — building on Mark Handley's earlier research on packet spraying and trimming in [NDP](https://dl.acm.org/doi/10.1145/3098822.3098825) — and drove the production deployment, while AMD, Broadcom, and NVIDIA implemented MRC in their respective NIC hardware, Microsoft contributed deployment requirements and architectural feedback from hyperscale AI infrastructure, and Intel participated in the specification process.

MRC builds on the [UltraEthernet Transport Specification](https://ultraethernet.org/) (UET 1.01), which it references for congestion control, packet trimming behavior, and several protocol details. MRC is implemented in 400 and 800 Gb/s NICs — NVIDIA ConnectX-8 and ConnectX-9, AMD Pollara and Vulcano, and Broadcom Thor Ultra — and has been deployed in production across OpenAI's largest training clusters, Microsoft's Fairwater data centers, and Oracle Cloud Infrastructure's Abilene facility, where it has been used to train frontier large language models. That deployment experience was formalized as an [open specification](https://www.opencompute.org/documents/ocp-mrc-1-0-pdf) (Revision 1.0) through the Open Compute Project (OCP) in March 2026. The companion paper [*Resilient AI Supercomputer Networking using MRC and SRv6*](https://arxiv.org/abs/2605.04333) followed in May 2026 with a detailed account of the protocol design and production results.

## Why Standard RoCEv2 Breaks at AI Scale — and How MRC Fixes It

Standard RoCEv2 RC was designed for a small number of well-behaved connections on a lossless fabric. AI training violates every one of those assumptions. This section traces the chain of problems, each leading to the next, and the corresponding MRC solution. Later sections expand on each mechanism in detail.

### 1. The Lossless Fabric Problem

RoCEv2 inherits InfiniBand's assumption of a **lossless** fabric — one where credit-based flow control ensures a sender never transmits a packet the receiver cannot buffer. Ethernet, however, is inherently **lossy**: switches drop packets when buffers overflow.

To bridge this gap, the industry developed [Priority Flow Control (PFC)](https://github.com/ManiAm/GNS-QOS/blob/master/docs/04_PFC.md), part of the Data Center Bridging (DCB) Ethernet extensions. PFC allows a switch to send a pause frame to an upstream port, telling it to stop transmitting until the buffer drains. This prevents drops and gives RoCEv2 the lossless environment it expects.

**The complication**: PFC pauses an *entire priority group* on an ingress port, not a specific flow. In a typical RoCEv2 deployment all RDMA traffic shares a single lossless priority group, so when one flow causes congestion, every other RDMA flow in the same group on that port is paused too — even if those flows are headed to completely different destinations. In large fabrics this causes [head-of-line blocking and congestion spreading](https://github.com/ManiAm/GNS-QOS/blob/master/docs/04_PFC.md#the-dangers-of-pfc-managing-the-lossless-safety-net) — a single hotspot cascades pause frames across the network, stalling unrelated traffic. At AI training scale, with thousands of flows converging on shared links, PFC storms become a serious source of **tail latency** that dictates overall job speed because every node must wait for the slowest.

**MRC's fix**: Disable PFC entirely and run Ethernet in its natural best-effort (lossy) mode. MRC uses **packet trimming** and **selective retransmission** to handle losses cheaply, as described in [Running on Lossy Ethernet](#running-on-lossy-ethernet) and [Reliable Delivery Without Ordering](#reliable-delivery-without-ordering).

### 2. The Load Balancing Problem

In a leaf-spine fabric, **Equal-Cost Multi-Path (ECMP)** routing distributes traffic by hashing each packet's **5-tuple** (source IP, destination IP, source port, destination port, and protocol number) to select an output port. RoCEv2 generates its entropy through the **UDP source port**, which is derived once per Queue Pair (QP) at connection setup. Every packet in that QP carries the same 5-tuple, so every packet follows the exact same path (see [RoCEv2 Entropy: The UDP Source Port](./03_README_ROCE_LB.md#rocev2-entropy-the-udp-source-port)).

This per-QP path pinning is adequate when the fabric carries thousands of small, short-lived flows — the statistical diversity naturally balances the load. AI training traffic is the opposite: a small number of massive, long-lived connections (**elephant flows**) that individually consume an entire link's capacity (see [Why This Breaks at AI Scale](./03_README_ROCE_LB.md#why-this-breaks-at-ai-scale)). When two elephant flows hash to the same spine link, that link becomes a bottleneck while neighboring links sit idle. ECMP has no feedback loop and no ability to rebalance, and the conventional workaround — [QP scaling](./03_README_ROCE_LB.md#qp-scaling-mqp-the-software-workaround) (splitting a transfer across multiple QPs) — has diminishing returns beyond 8 QPs.

**MRC's fix**: Replace per-QP entropy with **per-packet entropy**. Each packet carries a different **Entropy Value (EV)** so that a single QP's traffic is sprayed across hundreds of paths. The mechanism is detailed in [Entropy Values and Packet Spraying](#entropy-values-and-packet-spraying).

### 3. The Ordering Problem

Spraying solves load balancing, but it creates a new problem: packets now take paths of different lengths and congestion levels, so they arrive **out of order**. Standard RC cannot tolerate this. The RDMA Extended Transport Header (RETH) — which carries the destination memory address — appears only in the **first** packet of a message. Every subsequent packet is a bare payload fragment that depends on arriving in sequence so the receiver can write it to the correct memory offset (see [The RC Ordering Constraint](./03_README_ROCE_LB.md#the-rc-ordering-constraint)). An out-of-order packet looks identical to a lost one, triggering Go-Back-N retransmission and collapsing throughput.

**MRC's fix**: Include the **full RETH in every data packet**, so each packet is self-describing and can be written to the correct memory offset the instant it arrives, regardless of order. The full mechanism is described in [Reliable Delivery Without Ordering](#reliable-delivery-without-ordering).

### 4. The Retransmission Problem

Running on lossy Ethernet (because MRC disabled PFC) means packets *will* occasionally be dropped — by congested switches, by bit errors, by link flaps. Standard RC recovers from loss using **Go-Back-N**: when the sender detects a gap, it retransmits *everything* from the lost packet onward, wasting substantial bandwidth at high line rates.

**MRC's fix**: **Selective retransmission** using SACK (Selective Acknowledgment) bitmaps. The receiver tells the sender exactly which packets arrived and which are missing, and the sender retransmits only the missing ones. This is the single largest performance difference between MRC and standard RC, detailed in [Selective Retransmission (SACK and NACK)](#selective-retransmission-sack-and-nack).

### 5. The Congestion Control Problem

RoCEv2 uses [DCQCN](https://github.com/ManiAm/GNS-QOS/blob/master/docs/05_DCQCN.md) (Data Center Quantized Congestion Notification) for congestion control. DCQCN reacts to **Explicit Congestion Notification (ECN)** marks from switches by reducing the sender's overall injection rate. It has two limitations at AI scale: first, it can only slow the sender down globally — it cannot redirect traffic away from a congested path to a healthy one. Second, it is notoriously difficult to tune, because optimal parameters are traffic-pattern-specific; some hyperscalers have disabled it in production entirely.

**MRC's fix**: **NSCC (Network-Signalled Congestion Control)**, a window-based algorithm that combines ECN marks and measured RTT (round-trip time) to adjust a per-QP congestion window. Unlike DCQCN, NSCC can both reduce the sending rate *and* steer traffic away from congested paths. The algorithm is described in [Congestion Control (NSCC)](#congestion-control-nscc).

### 6. The Failure Recovery Problem

When a link fails in a standard RoCEv2 network, recovery depends on the routing protocol (typically BGP — Border Gateway Protocol, the standard inter-switch routing protocol in data center Ethernet) detecting the failure, withdrawing affected routes, and reprogramming forwarding tables across the fabric. This takes seconds — an eternity for a synchronous training job where the slowest node dictates overall performance. During reconvergence, affected QPs stall and may time out entirely, crashing the job.

**MRC's fix**: A **per-EV state machine** that tracks the health of every path individually. Failover happens in tens of microseconds, and recovered paths rejoin automatically — no routing reconvergence is needed because MRC uses static source routes. The state machine and recovery mechanisms are covered in [Adaptive Path Selection](#adaptive-path-selection).



## Deliberate Scope Restriction

MRC supports only **RDMA Write** and **Write-with-Immediate (WriteIMM)** operations. RDMA Write places data directly into a specified region of the remote node's memory without involving its CPU. WriteIMM does the same but additionally delivers a small immediate-data value that generates a completion notification on the receiver, signaling that new data has arrived. Read, Send, and Atomic operations are not supported. This is intentional: Write and WriteIMM are the operations that dominate collective communication — the coordinated data exchanges (such as all-reduce and all-gather) among all participants in a distributed training job — through libraries such as NVIDIA's **NCCL** (NVIDIA Collective Communications Library), so dropping the rest allows substantial transport simplification with no practical impact on training workloads.



## Multi-Plane Topology

MRC was co-designed with a specific physical topology that maximizes the benefit of packet spraying. Understanding this topology is essential context for the sections that follow.

Instead of treating an 800 Gb/s NIC as a single high-speed link, the NIC is *broken out* into several smaller links. Two breakout configurations are deployed in production: **8 × 100 Gb/s** and **4 × 200 Gb/s**. Each link connects to a different tier-0 (T0) switch, the top-of-rack tier. The result is multiple **independent parallel networks**, called **planes**, that share no switches with one another. In MRC's device model, each physical NIC port (called a *panel port* in the specification) maps directly to one plane, and every port is exposed as a separate PCIe (Peripheral Component Interconnect Express) physical function with its own network interface and management IP address.

Breaking out the ports also multiplies the usable **radix** — the total number of ports available on a switch — of each switch. A 51.2 Tb/s switch offers 64 ports at 800 Gb/s but 512 ports at 100 Gb/s. With 512-port switches and an 8-plane breakout, a two-tier **Clos topology** — a scalable, non-blocking network architecture built from identical switch stages, where every input can reach every output at full bandwidth — can connect over **131,000 GPUs**. That scale would otherwise require three or four tiers.

<img src="./pics/mrc-planes.jpg" width="700"/>

The diagram walks through the 8-plane topology from bottom to top. At the bottom, each GPU node has a single 800 Gb/s NIC broken out into **8 × 100 Gb/s ports** (one per plane). Each port connects to a different Tier 0 (T0) switch, shown as a stack of 8 in the middle row — one switch per plane, so each stack represents 8 independent T0 switches that share no hardware with one another. Each T0 switch has **512 ports**: 256 downlinks to GPU nodes and 256 uplinks to Tier 1 (T1) switches above. At the top, T1 switches are also stacked 8 deep (one per plane), each with 512 ports connecting down to T0 switches. The shaded bands between the tiers represent the 8 separate planes — each is a physically independent network. The bottom line confirms the resulting scale: 512 T0 switches × 256 GPUs per T0 = **131,072 GPUs** reachable in a two-tier fabric.

That reduction in tiers is where most of the benefits come from:

- **Lower latency**: The longest path traverses 3 switches instead of the 5 or 7 needed by a deeper topology.

- **Locality**: Many more nodes are reachable in one hop (256 under a single T0 switch, versus 32 at 800 Gb/s), making it easier to exploit locality in job placement and reducing load on T0 uplinks.

- **Reduced cost and power**: A two-tier multi-plane network needs roughly 60% of the switches and 67% of the optics of an equivalent three-tier single-plane network.

- **Higher redundancy**: Losing one T0–T1 link reduces a node's capacity by only about 0.4% in an 8-plane network, versus about 3% in a single-plane design.

- **Graceful NIC port failure**: If one of eight NIC ports fails, the node loses only 12.5% of its bandwidth rather than all connectivity. With a 4-plane breakout the loss is 25%, still far better than losing the entire connection. MRC detects the failure and keeps the job running, as described in [Failure Detection and Recovery](#failure-detection-and-recovery).



## Entropy Values and Packet Spraying

The most fundamental change in MRC is the elimination of single-path flow pinning.

In networking, **entropy** refers to the variable header bits a switch feeds into its load balancing hash. Standard RoCEv2 derives those bits once per QP, which is exactly why every packet of a QP follows the same route (see [RoCEv2 Entropy: The UDP Source Port](./03_README_ROCE_LB.md#rocev2-entropy-the-udp-source-port)). MRC makes entropy a per-packet property instead.

An **Entropy Value (EV)** is a 32-bit identifier that selects one specific path through the fabric. The 32 bits are striped across two standard header fields — the lower 16 bits of the IPv6 flow label and the 16-bit UDP source port — so switches can act on them without protocol-specific logic. At QP startup the sender is assigned an **EV Profile**: a set of EVs, each mapping to a different path. The set is typically 128 to 256 entries, distributed equally across all network planes so that traffic is balanced at the plane level from the start. Profiles are programmed either by the NIC itself or by a privileged host daemon called the **MRC Controller**, described in [Software and Control Plane](#software-and-control-plane).

In addition to the active EV set, the sender maintains a **backup EV set** — reserve EVs that are not currently in use. When MRC detects that a path has failed and removes an EV from the active set, it replaces it with a backup EV from the same plane, keeping the plane distribution balanced. This mechanism is what allows QPs to start without advance knowledge of which paths are down: the QP begins spraying across all its EVs, quickly discovers failed paths through loss detection, swaps in backups, and settles on a set of healthy paths within seconds.

When transmitting, the sender rotates through its active EV set, stamping a different EV into each packet. This is called **spraying**: what was a single elephant flow pinned to one link becomes a fine-grained stream distributed evenly across the whole fabric. Because the granularity is a single packet rather than a whole flow, load balancing is as fine as it can possibly be. Different senders do not coordinate when choosing their EV sets, and the specification recommends pseudo-random EV selection so that different QPs sharing an EV profile do not synchronize onto the same paths at the same time. The aggregate effect across hundreds of QPs distributes load naturally.

> How an EV is physically encoded into a packet — and therefore how switches act on it — depends on the [Routing Modes](#routing-modes).

Spraying is what makes MRC effective, but it also breaks two assumptions that standard RC relies on. Packets now arrive out of order, and a single connection now touches many links, so congestion and failure become per-path conditions rather than per-connection ones. Most of the remaining sections exist to handle those two consequences.

<img src="./pics/mrc-multipath.png" width="400"/>

The diagram above walks through MRC's core loop using a simplified example. On the left, the **EV Profile** holds four EVs (numbered 1 through 4), each initially in the GOOD state. A message arrives at the MRC QP, and the **Path Selection** stage assigns a different EV to each outgoing packet — the first packet gets EV 1, the second gets EV 2, the third gets EV 3. Because each EV maps to a different network path, the three packets are sprayed across three separate paths (EV 4 is still available in the profile for subsequent packets).

Once the packets reach the receiver, feedback flows back along each path independently:

- **Path 1** (EV 1): The receiver returns a SACK confirming the data arrived, but the SACK carries an **ECN** mark — a switch along this path flagged its queue as building up. This is mild congestion: the data got through, but the path is getting busy.

- **Path 2** (EV 2): The receiver returns a **NACK** with reason `TRIMMED` — a switch along this path ran out of buffer space, stripped the packet's payload, and forwarded only the header. The data must be retransmitted.

- **Path 3** (EV 3): The receiver returns a clean SACK with no congestion signals. This path is healthy.

Both congestion signals feed back into the EV Profile (the dashed "Congestion!" arrow): EV 1 and EV 2 are moved from GOOD to **SKIP**, so subsequent packets bypass those two paths for the current rotation. EVs 3 and 4 remain GOOD, and traffic shifts to them. On the next rotation the SKIP EVs automatically return to GOOD and are tried again. The mechanisms behind each feedback signal — trimming, SACKs, NACKs, ECN, and the full EV state machine — are covered in the sections that follow.



## Routing Modes

With planes in place, the next question is how an EV actually steers a packet. MRC defines three forwarding models that differ in how EVs are encoded and how much control the sender has over the resulting path. All three coexist in the specification; the choice depends on switch capability and how much path determinism the operator needs.

In all three models the switch stays **stateless** with respect to individual connections. It forwards purely on the entropy or segment information the NIC embedded in the packet, with no per-flow tracking and no connection awareness.

### ECMP

Here the EV is placed in the packet's UDP source port, and in the IPv6 flow label when one is present. Switches apply their normal ECMP hash to those fields to pick a next hop. Rotating EVs therefore spreads packets across different hash buckets.

This mode needs no switch changes beyond standard ECMP support, which makes it the simplest to deploy. It also offers the weakest determinism: two different EVs may land in the same hash bucket, and the sender cannot tell which physical path a given EV traverses.

### Structured Entropy Values

Structured EV gives source-routing-like determinism without any encapsulation. A 32-bit entropy field — the lower 16 bits of the IPv6 flow label concatenated with the 16-bit UDP source port — is partitioned into per-hop subfields, one for each network tier:

```text
Example: 3-hop topology with 10-bit, 8-bit, and 4-bit subfields

  ┌──────────┬────────┬──────┬──────────┐
  │  hop0    │  hop1  │ hop2 │ reserved │
  │  (10b)   │  (8b)  │ (4b) │          │
  └──────────┴────────┴──────┴──────────┘
  T0 switch    T1 switch  T2 switch

Each switch examines its designated hop subfield to select an egress port.
```

The number of subfields, their widths, and their valid value ranges are fixed at deployment time and known to all endpoints and switches. Each switch uses Access Control Lists (ACLs) or static forwarding tables to map its own subfield to an egress port. The sender therefore controls path selection at every tier while using only standard L3/L4 header fields, with no encapsulation overhead.

### SRv6 Source Routing

For maximum path control and observability, MRC uses **IPv6 Segment Routing (SRv6)**, a source-routing architecture in which the sending node encodes the complete forwarding path into the IPv6 header so that each intermediate switch simply follows the embedded instructions rather than computing next hops independently. In MRC's SRv6 mode, paths are represented as sequences of 16-bit micro-Segment IDs (uSIDs) packed into the IPv6 destination address:

1. At QP startup, the NIC or MRC Controller generates the EV set and maps each EV to a specific SRv6 destination address encoding the full path as a stack of uSIDs, one per switch. An optional Segment Routing Header using Compressed Segment List Encoding (RFC 9800) can extend the stack beyond a single 128-bit address, or carry a copy of the original path for debugging.

2. When a packet is sent, the NIC encapsulates it as **IPv6-in-IPv6**. The outer header carries the SRv6 path; the inner header carries the destination NIC's real address for decapsulation.

3. At each hop the switch performs the **uN (micro-node)** behavior in three steps. First, it matches its **Locator** prefix together with its own uSID at the leading position of the destination address. Second, it **left-shifts** all uSIDs by 16 bits — the next hop's uSID moves into the leading position and a zero fills the vacated slot at the end. Third, it performs a **/48 static route lookup** on the updated address (Locator + the now-leading uSID) and forwards the packet out the corresponding port. Each subsequent switch repeats the same process until the packet reaches its destination. Some deployments also use **uA (micro-argument)** behavior, where a uSID encodes both a target node and an argument such as a specific egress interface, allowing finer-grained decisions within a single hop.

   The following diagram illustrates these three steps at a single switch, showing how the destination address is transformed as the packet passes through.

   <img src="./pics/srv6-mrc.png" width="550"/>

#### Mapping EVs to SRv6 Addresses

The mapping from EV to SRv6 destination address is **algorithmic**, so the NIC only needs to store per-path EV state, not a full 128-bit address per path.

Switch uSIDs are allocated according to the network structure, allowing the EV to serve as a compressed representation of the bits that vary between SRv6 paths to a given destination. On QP startup, the NIC looks up the destination address prefix in a node-specific configuration file to obtain a generic SRv6 address **template** for nodes in that destination's row. The destination uSID in this template is then specialized by copying in the last-hop downlink number. This template is shared by all packets sent by the QP.

Each time a packet is sent, a new EV is selected. The template is further specialized by copying the **plane number** from the EV into all uSIDs and the **T0 uplink number** into the T1 uSID, producing the final destination address. Because the mapping is deterministic and reversible, the NIC does not need to store a separate SRv6 address for each EV — it derives the address on the fly from the EV and the template.

Because the tables are static and the path is fully determined by the sender, there is no routing convergence delay, no hash ambiguity, and no switch-level path computation. Link failures are handled entirely by MRC removing the affected EV from its active set.

This yields excellent **observability**. Since each EV maps deterministically to one physical path, an EV reported as bad identifies the exact failed link or switch. That is extremely difficult with hash-based ECMP, where the flow-to-path mapping is opaque.

Disabling dynamic routing here is deliberate. Two adaptive mechanisms — MRC at the endpoints and dynamic routing in the switches — interact unpredictably: MRC steers away from a failed path, then routing reconverges and remaps ECMP groups, disturbing MRC's balance. Static routes reduce the control plane to a single adaptive layer at the transport.



## Running on Lossy Ethernet

Standard RDMA deployments make Ethernet lossless with Priority Flow Control (PFC). MRC does the opposite, and the reason is a direct consequence of spraying.

### Disabling PFC

As described in [the overview](#1-the-lossless-fabric-problem), PFC pauses an entire priority group on an ingress port, causing head-of-line blocking and congestion spreading across unrelated flows. Spraying makes this incompatibility worse: a single MRC QP's packets now arrive at the last-hop switch over many different ingress links, so PFC pauses intended for that one QP would throttle every other RDMA flow sharing those ports.

MRC therefore **disables PFC entirely** and runs Ethernet in best-effort (lossy) mode. Occasional packet loss is a worthwhile trade for eliminating PFC-induced congestion spreading. Selective retransmission absorbs the resulting losses cheaply, making overall behavior under stress more predictable than PFC's.

### Packet Trimming

[Packet trimming](https://github.com/ManiAm/GNS-QOS/blob/master/docs/02a_AQM.md#packet-trimming--the-third-congestion-action) is a general switch-level congestion action defined in the UltraEthernet specification. Traditional switches have two responses to a full egress queue: **drop** the packet (RED/WRED), which silently destroys data and forces the sender to wait for a retransmission timeout, or **mark** it with ECN, which warns the sender to slow down but offers no recovery once the queue overflows and packets must be discarded. Trimming is a third option: instead of dropping a congested packet, the switch strips its payload and forwards only the headers on a high-priority queue. Because the transport headers — including sequence numbers — survive, the receiver knows exactly which data was lost and can request retransmission immediately, recovering in a single round-trip rather than waiting for a timeout.

MRC uses trimming in two ways. First, when the receiving NIC detects a trimmed packet it returns a NACK (Negative Acknowledgment) with reason `TRIMMED`, triggering **selective retransmission** of only the missing payload — far faster than a timeout-driven recovery. Second, trimming lets MRC distinguish **congestion loss**, where trimmed headers still arrive, from **path failure**, where packets vanish completely. That distinction is what allows the [EV state machine](#the-ev-state-machine) to react proportionately: congestion moves an EV to SKIP (temporary avoidance), while total silence moves it to ASSUMED_BAD (withdrawn until probed healthy).

Trimming matters most during **incast** — a many-to-one traffic pattern where multiple senders transmit to the same receiver simultaneously, overwhelming the last-hop link or switch buffer. This is a common pattern in AI collective communication, where many GPUs converge their results on a single node.

Generating trim NACKs is an **optional** responder capability, negotiated per connection via the `MRC_DEVICE_CAP_TRIM_NACK` attribute at QP setup. When the responder cannot generate them, the requestor falls back to other techniques, such as marking packets non-trimmable or sending periodic reliability probes.

### Traffic Classes

Trimmed headers, SACKs (Selective Acknowledgments), and NACKs are only useful if they survive the congestion that produced them. MRC therefore requires at least two traffic classes, separated by DSCP (Differentiated Services Code Point) values in the IP header:

| Traffic class | Packets carried | DSCP codepoints |
|---------------|-----------------|-----------------|
| **High priority (control)** | SACKs, NACKs, Transport ACKs, trimmed packets | `DSCP_CONTROL`, `DSCP_TRIMMED`, `DSCP_TRIMMED_LASTHOP` |
| **Data** | Write requests, reliability probes, retransmissions | `DSCP_TRIMMABLE`, `DSCP_NO_TRIM`, `DSCP_TRIMMABLE_RETX` |

The control class should be configured as high-priority best-effort with neither trimming nor WRED, so feedback reaches the sender promptly even under heavy data congestion. Retransmitted data carries its own codepoint (`DSCP_TRIMMABLE_RETX`) so switches can apply different queuing or marking policy to retransmissions than to first transmissions. `DSCP_NO_TRIM` is used when the network does not support trimming.



## Reliable Delivery Without Ordering

Spraying distributes packets across paths of different lengths and congestion levels, so they arrive out of order. Running without PFC, as described in the preceding section, means some packets will also be dropped. This section covers the mechanisms that handle both realities: placing out-of-order packets correctly and retransmitting lost ones efficiently.

### Out-of-Order Delivery with Immediate Memory Placement

As introduced in the [overview](#3-the-ordering-problem), out-of-order arrival is fatal for standard RC because the RETH appears only in the first packet, making every subsequent packet depend on in-order delivery (see also [The RC Ordering Constraint](./03_README_ROCE_LB.md#the-rc-ordering-constraint)).

MRC solves this by including the **full RETH in every data packet** — first, middle, last, and only. Each RETH carries the destination virtual address, the **R_Key** (a remote access key that authorizes the receiver's NIC to write into the specified memory region), and the **DMA length** (the number of bytes to transfer via Direct Memory Access), with the virtual address advanced by one **Path MTU** (Maximum Transmission Unit) for each successive packet. Every packet is therefore self-describing: the receiving NIC can write it straight to its correct offset in the application's memory buffer the instant it arrives. No reordering buffer is required, and no packet is discarded merely for arriving early.

WriteIMM needs a little more, because it delivers a completion to the receiving application and completions must appear in send order. MRC adds a **Message Extended Transport Header (METH)** carrying a Message Sequence Number (MSN) and a Receive Queue MSN (RQMSN) to track in-flight WriteIMM requests. Immediate Data values that arrive early are stashed until all preceding requests have been received, at which point the completion is delivered in the correct order. The maximum number of in-flight WriteIMM operations (`max_wimm_inflight`) is negotiated during QP setup; exceeding it moves the QP to an error state. MRC has no RNR-NAK (Receiver Not Ready — the standard RC mechanism that pauses a sender when the receiver's work queue is temporarily full) flow control, so the application — NCCL, typically — is responsible for staying within the advertised limit.

### Selective Retransmission (SACK and NACK)

MRC's reliability layer uses three types of acknowledgment. The first is inherited from standard RC; the other two are added by MRC to support selective retransmission. All three use the same 24-bit **Packet Sequence Number (PSN)** from the InfiniBand Base Transport Header (BTH).

- **Transport ACK**: The standard RC acknowledgment, carried in the ACK Extended Transport Header (AETH). In standard RC, Transport ACKs are the *only* feedback: the receiver confirms the highest in-order PSN, and any gap triggers Go-Back-N retransmission of everything from the gap onward. MRC still generates Transport ACKs for compatibility, but they are no longer the primary reliability mechanism.

- **Reliability SACK (Selective Acknowledgment)**: Instead of confirming just one PSN, a SACK carries a cumulative ACK *plus* a bitmap that tells the sender exactly which packets arrived and which are missing. The sender retransmits only the missing ones.

- **Reliability NACK (Negative Acknowledgment)**: An explicit signal that something went wrong, carrying a reason code so the sender can distinguish congestion from receiver exhaustion from fatal errors.

The bandwidth savings are substantial. The following comparison illustrates why:

```text
At 800 Gb/s with 9 KB jumbo frames:
  BDP (Bandwidth-Delay Product: how much data is in the network
       between sender and receiver at any moment)
      = 800 Gb/s × 10 µs RTT = 1 MB ≈ 108 packets in flight

Go-Back-N (standard RC):
  1 lost packet → retransmit from the lost PSN onward ≈ N/2 ≈ 54 packets
  Throughput loss per event ≈ 50%
  At 1% loss rate:  effective throughput ≈ 65%  → ~35% of bandwidth wasted

Selective retransmission (MRC):
  1 lost packet → retransmit exactly 1 packet
  At 1% loss rate:  effective throughput > 99%  → <1% of bandwidth wasted
```

#### Reliability SACKs

As data packets arrive (potentially out of order), the **responder's NIC** tracks which PSNs it has received. It encodes that receive state into a Reliability SACK and sends it back to the **requestor**, which reads the bitmap and retransmits only the missing packets. Each SACK carries:

- A **cumulative ACK PSN** (`cack_psn`): the highest PSN before the first gap, confirming everything before it arrived.

- A **64-bit bitmap**: one bit per packet, relative to a `sack_offset` from `cack_psn`, telling the sender exactly which packets are present and which are missing.

- **Reflected entropy**: the EV of the data packet that triggered this SACK, so the sender can attribute loss to a specific path.

- **Congestion state** (`CC_STATE`): a reflected timestamp for round-trip time (RTT) measurement, an out-of-order count, a receiver congestion penalty, and a received byte count — the inputs to congestion control.

The following example shows how the cumulative ACK and bitmap work together. The sender transmitted PSNs 0 through 11. PSNs 5 and 8 were lost in the network. The receiver builds a SACK reporting everything it knows:

```text
Sent by requestor:
  PSN:  0  1  2  3  4  5  6  7  8  9  10  11
                       ✗       ✗          ← lost in network

Receiver's SACK:
  cack_psn = 4            (everything up to PSN 4 arrived)
  sack_offset = 1         (bitmap starts 1 position after cack_psn, i.e. at PSN 5)
  bitmap = 0 1 1 0 1 1 1  (PSN 5=missing, 6=ok, 7=ok, 8=missing, 9=ok, 10=ok, 11=ok)
           ↑     ↑
         PSN 5  PSN 8   ← sender retransmits only these two packets
```

When a gap appears, the sender retransmits only the missing packets (the zeros in the bitmap), marked with an `rtx` flag in the BTH.

#### Reliability NACKs

While SACKs let the sender *infer* loss from bitmap gaps, NACKs provide an explicit signal with a reason code, so the sender can tell network loss apart from receiver exhaustion and from unrecoverable errors:

| Reason code      | Meaning |
|------------------|---------|
| `TRIMMED`        | A switch stripped the packet's payload due to buffer congestion and forwarded only the header (see [Packet Trimming](#packet-trimming)) |
| `NO_BITMAP`      | The responder ran out of PSN tracking resources |
| `NO_PKT_BUFFER`  | The responder's packet buffer is full |
| `PSN_OOR_WINDOW` | The PSN fell outside the responder's tracking window |
| `UNEXP_EVENT`    | Fatal error |


### Reverse Path Handling

SACKs, NACKs, and Transport ACKs travel on the **reverse path** from responder to requestor. These control packets need EVs too — they must be forwarded through the fabric — but the forward-path EV set cannot simply be reused. Each EV encodes a specific directional path (a sequence of switch ports from sender to receiver), so an EV that routes correctly from A to B will not route correctly from B to A.

With bidirectional traffic, control packets can piggyback on any EV from the responder's own active set. However, many collective QPs are unidirectional at any given instant: one side is only receiving, so it has no outbound data EVs of its own.

MRC solves this with a small **reverse EV set** dedicated to control packets, maintaining at least one EV per plane. Each RTT that the QP is active inbound but generates no outbound data traffic, the receiver sends an **EV probe** using a randomly chosen EV. If the probe is acknowledged, the reverse EV for that plane is updated to the probe's EV. When bidirectional data traffic is present, the reverse EVs are updated from data SACKs instead, avoiding the need for probes. This ensures the reverse EV set always contains EVs known to be on working paths. MRC's performance is not highly sensitive to reverse-path loss, but maintaining healthy reverse paths reduces tail latency by ensuring timely delivery of congestion and loss feedback.



## Congestion Control (NSCC)

MRC adopts **NSCC (Network-Signalled Congestion Control)**, a sender-side, window-based algorithm defined in the UltraEthernet Transport Specification whose transmit pacing is driven by returning SACKs. NSCC maintains a per-QP *congestion window* (`cwnd`) — a limit, measured in bytes, on how much data the sender may have in flight (sent but not yet acknowledged). The window is continuously adjusted to keep queueing delay near a configurable target (`target_Qdelay`). A sender may transmit only while `cwnd > inflight`.

### The Two Network Signals

NSCC requires two signals, which complement each other in timing:

- **Request RTT** is a lagging indicator. It estimates queueing delay from the round-trip time between a data packet and its SACK. MRC measures it either with a **Timestamp Extended Header (TSETH)** that embeds a 16-bit transmit timestamp in data packets and is reflected in the SACK's `CC_STATE`, or from a local send-time database at the requestor.

- **ECN (Explicit Congestion Notification)** is a leading indicator. Switches mark packets once queue depth crosses a threshold, and the responder echoes the mark back in the SACK's `M` field, tagged with the EV that experienced the congestion.

### ECN as a Load-Balancing Signal

In a multi-plane network with full **bisection bandwidth** — meaning the network can simultaneously carry the maximum traffic that all nodes could generate, with no bottleneck between any two halves of the fabric — the aggregate traffic should not experience congestion except from incast on the last hop to the receiver. MRC exploits this property by **disabling ECN marking on last-hop switches**. With this configuration, ECN marks signal only mid-fabric congestion — queuing at T0 uplinks or T1 switches — which in a properly provisioned network reflects load imbalance rather than true oversubscription.

This turns ECN into a **load-balancing signal**. When the responder echoes an ECN mark for a specific EV, the sender knows that path is carrying more traffic than its neighbors and temporarily avoids it (via the SKIP mechanism described in [The EV State Machine](#the-ev-state-machine)). Different senders react independently, and the aggregate effect smooths out the small statistical imbalances that arise from uncoordinated EV selection.

### Combining the Two Signals

Using a leading and a lagging signal together produces a four-quadrant response:

| ECN     | Request RTT  | Inferred network state     | Window adjustment |
|---------|--------------|----------------------------|-------------------|
| Not set | Below target | Uncongested                | Proportional increase |
| Not set | Above target | Recovering from congestion | Fair increase |
| Set     | Above target | Actively congested         | Multiplicative decrease |
| Set     | Below target | Transient congestion       | No change |

This is more precise than [DCQCN](https://github.com/ManiAm/GNS-QOS/blob/master/docs/05_DCQCN.md), which relies on ECN alone. NSCC shrinks the window only when ECN *and* elevated RTT agree that congestion is sustained, so transient marks do not needlessly cut throughput. More importantly, NSCC pairs rate control with path selection: DCQCN can only reduce a sender's overall injection rate, whereas NSCC can also move traffic to a healthier path and keep full throughput. That path-level half of the response is the subject of the next section.

### Responder Flow Control

Network congestion is not the only possible bottleneck; the receiver itself may fall behind. A responder can signal back-pressure through the `rcv_cwnd_pen` (receiver congestion-window penalty) field in its SACKs, asking the sender to shrink its window. The penalty ranges from 0 (no effect) to 127 (reduce to one packet per RTT).

### The QP Scheduler

Each requestor maintains a **QP Congestion Controller (QPCC)**, functionally equivalent to the UltraEthernet Congestion Control Context. The NIC scheduler tracks four states per QP:

| State       | Meaning |
|-------------|---------|
| **Idle**    | No data to send |
| **Active**  | Data queued, but the window is full |
| **Ready**   | Data queued and the window permits sending |
| **Pending** | Data in flight, nothing new to send |

<img src="./pics/qp-scheduler.jpg" width="600"/>

The scheduler rotates among QPs in the Ready state. Crucially, the EV for a packet is chosen at send time, via `GetSendParams()`, rather than when the QP became Ready. The call takes a bitmap of which ports currently have transmit space and returns an EV drawn from a ready plane. Deferring the choice this way means SACKs or NACKs arriving in the interim can still change both the EV and the port, so path decisions always reflect the freshest congestion feedback.



## Adaptive Path Selection

Congestion control decides *how much* to send. This section covers the other half of MRC's response: deciding *where* to send it, and recovering when a path breaks.

### The EV State Machine

MRC tracks the health of every EV in the profile using four states:

| EV state        | Active?  | Meaning |
|-----------------|----------|---------|
| **GOOD**        | Active   | Path is healthy and available for transmission |
| **SKIP**        | Inactive | Path recently showed congestion; bypassed for one rotation, then automatically returns to GOOD |
| **ASSUMED_BAD** | Inactive | Path failure detected; requires probe-based recovery before reuse |
| **DENIED**      | Inactive | Administratively disabled by the MRC Controller; never auto-recovers |

<img src="./pics/EV_states.jpg" width="600"/>

The sender draws EVs only from the active set. Transitions are driven by the feedback signals introduced earlier:

- A SACK whose `M` field is **SKIP_ONCE** (`0b01`, meaning the forward path was ECN-marked) moves the EV from GOOD to **SKIP**. The sender bypasses it for the current rotation and restores it on the next pass, smoothing out momentary queue buildup.

- A SACK whose `M` field is **ALWAYS_SKIP** (`0b10`) moves the EV to **ASSUMED_BAD**: the responder has concluded the path is persistently bad.

- A NACK with reason `TRIMMED` (but not `TRIMMED_LASTHOP`) moves the EV to **SKIP**, because a trim indicates congestion rather than failure.

- A packet that times out with no trim notification and no SACK moves the EV to **ASSUMED_BAD**. Total silence suggests the path is gone, not merely busy.

- The MRC Controller can set any EV to **DENIED** to exclude a known-bad path, and back to **GOOD** once the underlying issue is fixed.

The value of four states rather than two is that MRC can respond in proportion to the problem. Transient congestion triggers SKIP, a lightweight self-recovering avoidance that redistributes one rotation's worth of traffic. Persistent failure triggers ASSUMED_BAD, which withdraws the path until probing proves it healthy. DENIED gives operators a way to drain paths ahead of planned maintenance.

### Failure Detection and Recovery

MRC defines two probe mechanisms, distinguished by scope.

**Reliability Probes** operate at connection scope. They are addressed to the peer QP using its normal QP number, consume no PSNs, and elicit a SACK carrying the responder's current bitmap and congestion state. By varying the probe's EV, the sender can test a specific path. They serve two purposes: checking whether a given path still reaches the peer, and refreshing acknowledgment state when the sender suspects SACKs have been lost.

**EV Probes** operate at node scope. They are **connectionless** Endpoint Operations using the reserved QP number `0x2` and are not tied to any QP. The MRC Controller uses them to measure forward reachability and round-trip latency independently of active traffic. Each probe carries a unique identifier that the responder echoes, allowing exact request-response matching. Their main use is verifying that a repaired link genuinely forwards traffic before its EVs are returned to service.

For EVs in ASSUMED_BAD, the sender periodically probes the retired path. If a probe returns a SACK with `M` set to `NONE`, the EV returns to GOOD; if `M` is `SKIP_ONCE`, it becomes SKIP instead. The result is a self-healing loop: failures are bypassed almost immediately, and recovered paths rejoin the rotation without operator action.

**Port Status Updates** complement probes with proactive notification. When a local NIC port fails or recovers, the NIC sends a Port Status Update — another connectionless Endpoint Operation — carrying a `port_status_mask` bitmap with one bit per local port. Peers adjust their EV selection immediately instead of waiting for timeouts on the affected paths, which is what allows a node to survive the loss of a plane without disrupting the job.

### Startup Path Discovery

At QP startup, the sender does not need advance knowledge of which paths are down. It populates a large active EV set (typically over 100 entries) plus a backup set, and begins spraying across all corresponding paths. Some paths may be down; the resulting packet losses trigger retransmissions and EV swaps within seconds. Production data from a 75,000-GPU pretraining job shows that the per-NIC loss rate drops well below one packet per second — a loss rate of roughly 1 in 25 million at 800 Gb/s — within the first couple of minutes, even without pre-populated denylists. Given that large training jobs must ramp up slowly to avoid destabilizing the power grid, this startup transient has minimal impact on training time.

MRC does support an explicit **denylist** mechanism, allowing paths known to traverse failed links to be excluded at startup. A monitoring infrastructure called **Clustermapper** (described in [Software and Control Plane](#software-and-control-plane)) can populate these denylists. In practice, however, denylists have not proved necessary for pretraining — MRC's self-discovery is fast enough.



## Inter-Plane Loading

MRC makes a deliberate design choice to keep traffic **equally distributed across all planes** at all times. When an EV is removed from the active set (because its path failed or showed congestion), it is always replaced with a backup EV **from the same plane**. This invariant ensures that as long as all NIC ports are operational, every plane carries the same amount of traffic.

Equal plane loading is valuable for two reasons. First, it prevents **false incast** at the destination. If senders reacted to mild T0-uplink congestion by shifting traffic to other planes, multiple senders could independently pile onto the same less-congested plane, creating hotspots worse than the original congestion. Second, it simplifies operational monitoring — all planes normally look identical in network statistics once MRC has avoided bad links, so if one plane looks worse than the others, it generally points to a network problem that needs attention.

This design has two trade-offs:

- **Single-path traffic interference**: If any non-MRC single-path traffic is present on the back-end network, MRC is constrained by the most congested plane and loses capacity. This is not a problem in practice when the back-end network runs only MRC traffic.

- **Gradual link degradation**: If a NIC–T0 link does not fail completely but instead develops a high packet loss rate, MRC cannot rebalance away from the degraded plane on its own, because it cannot reliably determine whether the problem is at its end or the remote end. Clustermapper (see [Software and Control Plane](#software-and-control-plane)) fills this gap: its local probes can detect degraded NIC–T0 links and add a denylist entry to avoid the affected plane.



## Software and Control Plane

### The MRC API (libmrc)

MRC is exposed through **libmrc**, which presents two distinct APIs.

**`mrc.h`** is the application API. It mirrors the semantics of `libibverbs` (the standard RDMA user-space library) for creating, modifying, and destroying MRC QPs and Completion Queues, posting work requests, and polling completions. Applications continue to use `libibverbs` for device discovery and context management (`ibv_open_device`, `ibv_query_port`), while `libmrc` handles MRC-specific configuration through `mrc_modify_qp()`.

**`mrc_ctl.h`** is the controller API. It is privileged, requiring `CAP_NET_ADMIN`, and is used by the **MRC Controller**, a per-node daemon or CLI tool responsible for:

- **EV profile management**: creating profiles in EXPLICIT mode (the controller programs each EV value), GENERATED mode (the NIC derives EVs from parameters defining per-tier hop widths and valid ranges), or AUTO mode (vendor-defined default, typically ECMP-style hashing).

- **CC profile management**: configuring NSCC parameters such as target queueing delay and initial congestion window, per destination or per group of QPs.

- **EV state management**: forcing individual EVs to DENIED or GOOD, and receiving asynchronous EV events when the NIC detects a bad path. Because SRv6 makes the EV-to-path mapping deterministic, the controller can correlate these events across QPs to pinpoint the specific failing link.

- **EV probes**: testing path reachability independently of active QP traffic.

Connection setup requires **out-of-band exchange of QP attributes**: peers must share connection parameters through their own mechanism (for example, TCP sockets or a key-value store) before establishing a QP. RDMA-CM (RDMA Communication Manager), which automates this exchange in standard RDMA deployments, is not supported. During setup, peers exchange capability attributes including the responder's WriteIMM in-flight limit (`MRC_QP_MAX_WIMM_DEST`), its maximum PSN tracking range (`MRC_QP_MAX_MPR_DEST`), and optional feature flags for Dynamic MPR (letting the responder adjust the in-flight PSN window at runtime), trim NACK generation, and service time reporting.

### Clustermapper

**Clustermapper** is a distributed monitoring infrastructure, built by OpenAI, that provides ground-truth data about network health, independently of active MRC traffic. It is not part of the open MRC specification published through OCP; it is internal operational tooling described in the companion paper for completeness. It consists of an agent running on every node in the cluster, and the agents collectively probe every link in the network at high frequency (every millisecond).

Each Clustermapper agent probes its directly connected T0 switches by sending SRv6 source-routed packets to the T0 and back to itself (one probe per port, covering all planes). It also probes a subset of T1 switches in the same way, routing to the T1 and back, so that all T0–T1 links are probed at high frequency across all agents.

The combination of T0 and T1 self-probes enables **precise failure localization**: if T0 probes succeed but T1 probes fail, the issue is on the T0–T1 link, not the NIC–T0 link. Static SRv6 routing makes this particularly reliable: unlike with ECMP hashing, there is no ambiguity about which path a probe packet takes, there is no dynamic routing changing forwarding paths underneath, and probe packets take the same paths as MRC data packets. And unlike switch ICMP pinging, SRv6 probes are handled in the data plane, enabling high-frequency probing without loading the switch control plane.

Clustermapper serves several operational roles:

- **Denylist management**: detecting and excluding links with high loss rates or persistent failures.
- **Failure correlation**: providing a cluster-wide map of link health for root-cause analysis.
- **Pre-job validation**: verifying network health before launching a training job.
- **Continuous monitoring**: detecting issues even when no workload is running.



## Hardware Implementations

How much of MRC runs in hardware depends on the NIC generation.

### Native MRC

In native mode the NIC implements the complete protocol — spraying, SACK/NACK generation, selective retransmission, EV state tracking, and NSCC — entirely in firmware and on-chip logic, with the host CPU absent from all data-path decisions. NVIDIA's **ConnectX-9 (CX9) SuperNIC** is the first NIC to support native MRC. *SuperNIC* is NVIDIA's term for Data Processing Unit (DPU)-class adapters that integrate programmable congestion control and advanced transport offloads directly in hardware, which is what allows full line-rate MRC with no host software assistance.

### Proxy MRC

In proxy mode the protocol is split between host software and the NIC. The NIC handles packet transmission and reception, while higher-level functions such as EV selection, retransmission scheduling, and congestion window management run in a host-side library or driver. Proxy mode is primarily a path to validating and deploying MRC before native hardware is available.

Three NICs support MRC through proxy mode:

- **NVIDIA ConnectX-8 (CX8)**: An 800 Gb/s NIC already widely deployed. CX8 lacks CX9's dedicated transport engines but can execute the protocol through cooperation between firmware and host-resident software. CX8 is the NIC used in the largest production MRC deployments to date.

- **AMD Pollara and Vulcano**: 400 Gb/s NICs with MRC support. AMD's implementation was validated on a 64-GPU cluster using a 4-plane (4 × 100 Gb/s) configuration with SRv6 routing.

- **Broadcom Thor Ultra**: An NIC that supports both MRC and conventional RoCEv2, allowing direct performance comparisons on the same hardware.

### Switch Platforms

On the switch side, MRC has been validated on multiple platforms:

- **NVIDIA Spectrum-4 and Spectrum-5**: 51.2 Tb/s ASICs that support standard ECMP, Structured EV forwarding via ACLs, and SRv6 uSID forwarding with static tables. Both Cumulus and SONiC are supported as the network operating system, providing configuration and management interfaces for routing, traffic classes, ECN marking thresholds, and trimming behavior.

- **Broadcom Tomahawk 5**: Used in conjunction with Arista EOS for SRv6 forwarding.



## Performance Evaluation

MRC is evaluated with **NCCL benchmarks**: standardized collective communication tests (all-reduce, all-gather, reduce-scatter, send/recv) that reproduce the traffic patterns of real distributed training. The headline metric is **JCT (Job Completion Time)**, the wall-clock duration of a full training job or benchmark run. JCT is used because it captures throughput, tail latency, and failure recovery together, as a single end-to-end number.

### Point-to-Point Performance

Point-to-point measurements on an 8-plane (8 × 100 Gb/s) cluster with CX8 NICs show:

| Topology | Message size | Metric | Result |
|----------|-------------|--------|--------|
| T0-local (within one T0 switch) | 2 B | Latency | 5.09 µs |
| T0-local | 32 KB | Bandwidth | ~770 Gb/s (96% of peak) |
| Cross-T1 (traversing T1 switches) | 2 B | Latency | 6.54 µs |
| Cross-T1 | 32 KB | Bandwidth | ~770 Gb/s (96% of peak) |

Both T0-local and cross-T1 configurations achieve near-identical bandwidth, demonstrating that steady-state throughput is not constrained by path length. The 1.45 µs latency difference for short messages reflects the additional switch hops when crossing T0 boundaries, with negligible effect on large-message performance.

### NCCL Collective Performance at Scale

Using the NCCL sendrecv benchmark at 42,000 GPUs on an 8-plane cluster, MRC achieves up to **92 GB/s per NIC** for large message sizes (1–2 GB). This validates that MRC's spraying and congestion control scale to production cluster sizes.

### MRC versus RoCEv2

A direct comparison on identical hardware (64 AMD Pollara 400 Gb/s NICs on Broadcom Tomahawk 5 switches) isolates MRC's benefits. RoCEv2 uses a single-plane 400 Gb/s configuration with PFC and DCQCN; MRC uses a four-plane (4 × 100 Gb/s) configuration with SRv6 and no PFC.

**Ring all-reduce (64 nodes)**: Each node sends to one peer and receives from another; any stall creates a bubble that hurts overall performance.

- Without loss: a single MRC QP spraying across 256 paths outperforms 16 RoCEv2 QPs across all large message sizes, primarily because MRC eliminates ECMP hash collisions. For RoCEv2, QP scaling beyond 8 QPs yields little further gain.
- At 0.1% induced loss: MRC holds near line rate for large messages. RoCEv2 collapses under Go-Back-N amplification and PFC-induced blocking.
- At 1% induced loss: Even MRC achieves only about a third of intended throughput, but RoCEv2 is essentially unusable. This loss rate is high enough that MRC would detect the lossy paths and avoid them in production, limiting the damage.

**All-to-all (64 nodes)**: Every node sends to and receives from multiple peers simultaneously, stressing load balancing and incast handling.

- Without loss: MRC outperforms RoCEv2 at all message sizes. RoCEv2 QP scaling does not help here — 16 QPs per destination actually perform slightly *worse* than 1 QP, because with many simultaneous QPs the additional path diversity offers diminishing returns while adding overhead.
- With loss: RoCEv2 suffers especially at small message sizes, where there is not enough traffic volume to mask a retransmission pause. MRC's SACK-based retransmission helps greatly regardless of message size.

### Collateral Damage (Incast)

Multiple collectives performing different axes of training parallelism run simultaneously and can interfere with each other. A cross-spine 7-to-1 incast experiment with a concurrent "victim" flow to a different node in the same rack reveals:

- **MRC (1 QP)**: The bottleneck link is shared evenly among the incast flows with **no measurable impact** on the victim flow.
- **RoCEv2 with DCQCN (1 QP)**: Victim flow throughput degrades by ~25%.
- **RoCEv2 with DCQCN (10 QPs)**: Average victim impact is smaller, but one-second intervals show the victim dropping to 100 Gb/s (a 75% drop from optimal).
- **RoCEv2 with PFC only (no DCQCN)**: Victim flow drops to 30–100 Gb/s.

DCQCN parameter tuning can reduce this issue in theory, but in practice it is very difficult because optimal settings are traffic-pattern-specific. Some hyperscalers have disabled DCQCN in production for this reason.

### The Comparison Baseline

The benchmark baseline in the [companion paper](https://arxiv.org/abs/2605.04333) is **ZTR-RTT CC (Zero-Tolerance Retransmission, RTT-based Congestion Control)**, NVIDIA's proprietary algorithm for CX9 SuperNICs. ZTR-RTT CC drives rate adjustment from RTT measurements and relies on **switch-side** packet spraying, where the switch distributes a flow's packets across equal-cost links. Measuring MRC against it isolates the value of MRC's specific approach — endpoint-controlled path selection and selective retransmission — versus the conventional method of delegating both spraying and congestion response to the network.



## Production Impact

Deployments at OpenAI and Microsoft have produced several concrete operational results that go beyond what controlled benchmarks (see [Performance Evaluation](#performance-evaluation)) can measure, demonstrating MRC's resilience under real-world fault conditions.

- **Link flap tolerance**: During frontier model training on a 75,000-GPU cluster, multiple T0–T1 link flaps per minute had no measurable effect on synchronous pretraining. These flappy links are left in service: MRC maps them out when they drop and only brings them back when enough probes succeed over time. Repairing such links became routine low-priority maintenance rather than an urgent incident. This approach is also robust to the inevitable disruption caused when a technician repairing one link disturbs neighboring links.

- **Switch failure resilience**: When T1 switches were rebooted during active training (four such events during a single 75,000-GPU job), MRC progressively detected the failing EVs, moved each to ASSUMED_BAD, and redistributed traffic across the remaining paths. Full recovery completed within seconds, and the job continued with no coordination between network operations and the training team. With static SRv6 routing, the switch reboot itself had no impact — there was no routing convergence to wait for.

- **NIC port failure survival**: Before MRC, losing the link between a GPU's NIC and its T0 switch would crash the job. With MRC, the NIC detects the link failure, remaps EVs to avoid the failed port, and sends a Port Status Update to notify peers. Remote endpoints also remap their EVs to avoid the failed plane. In an 8-plane network the job continues at 87.5% of its bandwidth; in a 4-plane network at 75%. Most such failures are transient and recover within a minute, after which full bandwidth returns. When a port stays down permanently, the node is banned and scheduled for repair.

- **Transceiver glitch recovery**: On a 50,000-GPU cluster, an optical transceiver on a T0 switch glitched and flapped all four of its links in rapid succession, affecting three active training nodes. Job throughput dropped approximately 25% during the minute of flaps, then recovered to full speed immediately afterward. No QP failed and no nodes needed to be removed from the job. One remaining single point of failure: in a 4-plane deployment, the 800 Gb/s NIC optics are split into 4 × 200 Gb/s links that share a single transceiver. If the NIC transceiver itself flaps (rather than a switch transceiver), all ports on that NIC fail simultaneously and QPs cannot survive. These events are rare but do occur at scale.

- **Operational simplicity**: Static SRv6 routing eliminates the need for dynamic routing protocols and their associated complexity. When a switch stops forwarding packets (a failure mode that is difficult for dynamic routing to detect, since the control plane remains up), MRC simply removes the affected paths. The switch can be rebooted without coordinating with active training jobs. Clustermapper provides continuous ground-truth health data that is unambiguous — each probe takes a known path and returns to the same agent — unlike switch-level telemetry, which can mask data-plane failures behind healthy control-plane state.
