
# Load Balancing for RoCEv2

> This document assumes familiarity with RDMA, InfiniBand transport services (RC, QP, Go-Back-N), and the RoCEv2 packet format. For background on these topics, see the [RDMA Primer](https://github.com/ManiAm/RDMA-Primer) project.

## ECMP Load Balancing for RoCEv2

In a Leaf-Spine fabric, ECMP distributes aggregate traffic across multiple equal-cost spine links by hashing each packet's 5-tuple to a deterministic output port. For general-purpose TCP workloads - thousands of short-lived, small flows (**mice flows**) - this statistical distribution works well. The sheer volume of independent flows naturally balances the load across all available paths.

RoCEv2 traffic, however, has characteristics that fundamentally conflict with this model. This section examines why standard load balancing breaks down for RDMA traffic and how the industry has evolved to address it.

### RoCEv2 Entropy: The UDP Source Port

For ECMP to distribute traffic, packets within different flows must produce different hash values. In the RoCEv2 5-tuple, four fields are static:

| Field            | Value                                            | Variability         |
| ---------------- | ------------------------------------------------ | ------------------- |
| Source IP        | Sender's NIC address                             | Fixed per node      |
| Destination IP   | Receiver's NIC address                           | Fixed per node      |
| Protocol         | UDP (17)                                         | Always fixed        |
| Destination Port | 4791 (IANA-assigned RoCEv2 well-known port)      | Always fixed        |
| **Source Port**  | **Hash of QP number and connection identifiers** | **Variable per QP** |

If the UDP source port were also fixed, all RoCEv2 traffic between any two servers would produce the same hash and be pinned to a single spine link — leaving every other path idle. To prevent this, the sender's NIC computes a per-QP hash (typically derived from the Queue Pair number and other internal connection state) and places it in the UDP source port field. To intermediate switches, each QP appears as a distinct UDP conversation with a unique 5-tuple, allowing ECMP to stripe different QPs across different spine links.

This mechanism is sufficient for separating different connections onto different paths. The challenge lies in what happens within a single connection.


### The RC Ordering Constraint

As covered in the [InfiniBand deep dive](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/02_README_INFINIBAND.md#segmentation-and-reassembly-sar), when an application message exceeds the MTU, the sender's NIC segments it into multiple packets. The first packet carries the RDMA Extended Transport Header (RETH) containing the remote virtual address, `R_Key`, and total DMA length. All subsequent packets carry raw payload fragments with no addressing metadata — they rely entirely on arriving in sequence so the receiver can write each fragment to the correct memory offset.

This creates a hard constraint: **all packets within a single QP must follow the exact same network path**. If packets from the same QP were spread across different spine links, path latency differences would cause out-of-order arrival. The receiving NIC would interpret this as packet loss, discard the out-of-order packets, and trigger Go-Back-N retransmission — exactly the catastrophic throughput collapse described in the [RoCE overview](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/03_README_ROCE.md#the-foundation-preparing-ethernet-for-rdma).

The consequence is that RoCEv2 + RC locks every QP to a single physical path. The per-QP UDP source port provides inter-QP entropy for ECMP, but intra-QP traffic is strictly single-path. No standard ECMP mechanism can split a single QP across multiple links.


### Why This Breaks at AI Scale

The combination of single-path pinning and the nature of AI training traffic creates a severe load balancing problem that standard ECMP cannot solve. RoCEv2 traffic is dominated by **elephant flows** — a small number of massive, long-lived connections that individually consume an entire link's capacity (see the [elephant flow analysis](./01_README_LB.md#the-elephant-flow-problem) for details).

AI training collectives (all-reduce, all-to-all, all-gather) generate textbook elephant flows. A single GPU may open an RDMA connection to another GPU and stream data at full 400 or 800 Gb/s line rate for seconds or minutes.

Standard ECMP is a purely static mechanism. It assigns a flow to a path at connection setup and never reconsiders. If a link becomes congested due to hash collisions, ECMP continues forwarding new packets into the bottleneck while neighboring links remain empty. It has no feedback loop, no congestion awareness, and no ability to rebalance.

This is the fundamental mismatch: ECMP was designed for a world of many small, statistically diverse flows. RoCEv2 AI traffic produces a small number of massive, long-lived flows where the statistical assumptions collapse.


### QP Scaling (MQP): The Software Workaround

The most common software-level approach to improving RoCEv2 load balancing is **QP scaling**, also known as **Multiple Queue Pairs (MQP)**. The core idea is simple: instead of sending a transfer over a single QP, the application (or collective communication library such as NCCL) opens multiple QPs to the same destination. Since each QP produces a distinct [UDP source port](#rocev2-entropy-the-udp-source-port), ECMP hashes them to different spine links — turning one elephant flow into multiple independent flows distributed across different paths.

Because each RC QP is [confined to a single forwarding path](#the-rc-ordering-constraint), a single QP can only utilize the bandwidth of one physical route. By splitting a large transfer across N QPs, the application creates N independent flows that the fabric can route over N different paths, potentially achieving N× the single-path throughput. This is particularly impactful in leaf-spine topologies where many equal-cost paths exist between endpoints.

There are two common MQP strategies:

| Strategy        | How it works | Best for |
|-----------------|--------------|----------|
| **Split**       | A single large message is divided across multiple QPs simultaneously | Latency-sensitive large transfers |
| **Round-robin** | Successive messages are posted to different QPs in rotation | Sustained throughput workloads |

The following examples illustrate the difference. In both cases, a node has 4 QPs to the same destination, each routed over a separate spine link at 100 Gb/s per QP:

```text
Split (one 8 MB message):
  The 8 MB buffer is divided into 4 × 2 MB chunks.
  All 4 QPs transmit their chunk simultaneously.

  QP 1 ──► 2 MB ──► spine A ─┐
  QP 2 ──► 2 MB ──► spine B ─┼──► receiver reassembles 8 MB
  QP 3 ──► 2 MB ──► spine C ─┤
  QP 4 ──► 2 MB ──► spine D ─┘

  Completion time ≈ 2 MB / 100 Gb/s ≈ 160 µs  (4× faster than a single QP)
  The application must wait for all 4 QPs to finish and reassemble the buffer.

Round-robin (four successive 8 MB messages):
  Each message is posted to the next QP in rotation.

  Message 1 ──► QP 1 ──► spine A   (t = 0)
  Message 2 ──► QP 2 ──► spine B   (t = 0)
  Message 3 ──► QP 3 ──► spine C   (t = 0)
  Message 4 ──► QP 4 ──► spine D   (t = 0)

  Each message takes ≈ 640 µs individually (8 MB / 100 Gb/s).
  But all 4 are in flight concurrently on different paths.
  Aggregate throughput = 4 × 100 Gb/s = 400 Gb/s (full bisection bandwidth).
  No cross-QP reassembly needed — each message is self-contained.
```

In short, split trades reassembly complexity for lower completion time on a single large transfer, while round-robin achieves higher sustained throughput by keeping all paths busy with independent, self-contained messages that require no cross-QP coordination.

Meta deployed QP scaling across their production AI training clusters and measured its impact on AllReduce bandwidth. The figure below (from [Meta's SIGCOMM 2024 paper](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/)) compares these two strategies at QP counts of 1, 4, and 16.

Note that "Message Size" on the x-axis is the **AllReduce buffer size**, not a single RDMA send. A collective operation like AllReduce internally decomposes into many individual RDMA transfers — for example, NCCL's ring-based AllReduce with N GPUs generates at least 2×(N−1) separate RDMA sends. This is why round-robin remains effective even at small buffer sizes — the collective always produces many individual transfers to distribute across QPs. Key findings:

- **Round-robin with 16 QPs** achieves near-100% normalized bandwidth for large buffers (2 GB).
- **Single QP** (the "out of box" baseline) reaches only ~60%.

The improvement is most pronounced for large buffers where individual RDMA transfers are large enough to become elephant flows that saturate a single link for extended periods. For small buffers (2–8 MB), each transfer completes quickly regardless of path assignment, so distributing them across additional QPs yields little additional throughput.

<img src="./pics/qp-scaling-meta.png" width="550"/>

QP scaling introduces significant trade-offs:

- **HCA resource consumption**: Each QP occupies hardware context memory on the HCA (connection state, sequence counters, retry timers). Creating hundreds of QPs to a single peer can exhaust finite hardware resources and increase cache pressure within the NIC, degrading performance for all connections.

- **Application-level reassembly**: When data is split across multiple QPs, each QP delivers its portion independently. The application (or middleware) must track which segments have completed and reassemble the logical buffer before consuming it. The hardware provides no cross-QP ordering guarantees.

- **Completion complexity**: Polling or waiting on multiple Completion Queues (or a shared CQ serving many QPs) increases software overhead and CPU utilization that partially offsets the throughput benefit.

- **Diminishing returns**: Beyond a certain QP count (typically 8–16), the probability that all QPs land on distinct paths decreases — hash collisions become likely (a birthday-paradox effect) — and the management overhead of additional QPs outweighs the bandwidth gain.

- **Cluster-wide collisions remain inevitable**: Even with optimal per-node QP counts, a large cluster with hundreds of thousands of QPs across thousands of nodes will still produce hash collisions somewhere in the fabric — a consequence of mapping a finite entropy space (16-bit UDP source port) to a limited number of spine paths.

In practice, the optimal QP count is a performance-engineering trade-off shaped by the fabric topology (number of equal-cost paths), the HCA's context memory capacity, and the workload's transfer pattern. Libraries such as NCCL tune this parameter internally based on the target platform.

QP scaling is a pragmatic improvement within the constraints of single-path RC, but it does not solve the fundamental problem. True resolution requires either the network or the transport protocol to support per-packet multi-path operation.


## Adaptive Routing for RoCEv2

The [Adaptive Routing](./02_README_ARS.md) documentation describes how modern switch ASICs overcome ECMP's congestion blindness by dynamically steering traffic based on real-time port congestion. The two primary algorithms (flowlet switching and packet spraying) apply to RoCEv2 traffic, but the RC ordering constraint changes the calculus significantly.

### Flowlet Switching

In flowlet mode, the switch monitors each flow for idle gaps (periods where no packets are in transit). When a gap exceeds the configured idle time (which must be greater than the maximum path latency skew), the switch knows all previous packets have been delivered. It can safely reassign the flow to a less-congested spine link without causing reordering.

For RoCEv2, flowlet switching is the safe, protocol-compatible choice because it preserves in-order delivery. However, AI training collectives often generate continuous, high-bandwidth streams with minimal idle gaps. A GPU performing an all-reduce at line rate produces a near-continuous packet stream — the switch may rarely or never observe a gap long enough to trigger a reroute. In this scenario, a massive elephant flow remains pinned to a congested link for its entire duration, and flowlet switching degrades to behaving like static ECMP.

### Packet Spraying

In packet-spray mode, the switch abandons flow affinity entirely. Every packet is independently routed to the least-congested spine link. This shatters elephant flows across all available paths, achieving near-perfect link utilization.

The problem is that packets from the same QP take different paths with different latencies and arrive out of order — triggering the [Go-Back-N retransmission collapse](#the-rc-ordering-constraint) described earlier. Packet spraying therefore cannot be used with standard RC unless the receiving NIC can handle reordering.

Two approaches have emerged, each removing a constraint that limited the previous one:

**Approach 1: Hardware reorder buffer (tolerate limited reordering)**

For packet spraying to work, the receiving NIC must handle out-of-order arrival. Modern HCAs (such as NVIDIA ConnectX-7) include dedicated **hardware reorder buffers** — on-chip SRAM that holds early-arriving packets, waits for the missing earlier packets to arrive, and then writes the reassembled, correctly ordered data to host memory. The application never sees out-of-order data.

However, reorder buffers have a fundamental limitation: they are **finite**. The buffer can only hold a limited number of out-of-order packets at a time. At 400–800 Gb/s line rates, the bandwidth-delay product across a multi-hop fabric can exceed the buffer capacity. When too many packets arrive out of order — or the gap between the earliest and latest packet grows too large — the buffer overflows. Overflowed packets must be dropped, triggering Go-Back-N retransmission and negating the throughput benefit of spraying. This limits the degree of path diversity that reorder-buffer NICs can tolerate.

**Approach 2: True out-of-order placement (eliminate reordering entirely)**

Rather than buffering and reassembling, [MRC](./04_README_MRC.md) makes every packet self-describing — each carries the full RDMA virtual address, so the receiving NIC writes it directly to the correct memory position on arrival, regardless of order. No reorder buffer is needed, no packet is dropped for arriving early, and path diversity is no longer constrained by buffer size.

### AR Eligibility Marking (The AR Bit)

The previous section established that Packet-Spray requires destination NICs with hardware reorder capability. But this creates a practical deployment problem: **how does a switch know which packets are safe to adaptively route?**

A real data center does not upgrade every NIC at once. During migration, the fabric will contain a mix of modern NICs (e.g., ConnectX-7) that handle out-of-order RoCE packets and legacy NICs (e.g., ConnectX-5 or third-party adapters) that require strict in-order delivery. Without a signaling mechanism, the operator faces a binary choice: enable AR globally (breaking legacy NICs) or disable it globally (wasting the capability of modern NICs). Neither option is acceptable.

#### The Solution: Per-Packet AR Eligibility

To solve this, the **source NIC** marks every RoCE packet with an **AR eligibility indication** before it enters the network. This indication tells every switch in the path whether the packet is permitted to undergo Adaptive Routing or must be forwarded using static ECMP.

The marking is a single bit in the RoCE **Base Transport Header (BTH)** — the InfiniBand transport header encapsulated inside the Ethernet frame. The BTH specification includes a 7-bit "Reserved" field, and the Most Significant Bit (MSB) of this field is repurposed as the AR eligibility flag:

- **Bit = 1**: The packet is **eligible** for AR. The switch may use Flowlet or Packet-Spray to route it.
- **Bit = 0**: The packet is **ineligible** for AR. The switch must use static ECMP hash-based routing.

```
RoCE Packet Structure:

┌──────────┬──────────┬──────────┬─────────────────────────────────┐
│ Ethernet │   IP     │   UDP    │     InfiniBand BTH              │
│ Header   │ Header   │ Header   │  ┌────────────────────────────┐ │
│  (L2)    │  (L3)    │  (L4)    │  │ OpCode │ ... │ Reserved(7) │ │
│          │          │          │  │              │  ↑ AR bit   │ │
│          │          │          │  └────────────────────────────┘ │
└──────────┴──────────┴──────────┴─────────────────────────────────┘
                                           ▲
                                    Switch reads this
                                    L4 bit to decide
                                    AR vs. static ECMP
```

Note that Ethernet switches are Layer-2 devices, yet they are reading a bit from the Layer-4 InfiniBand header. This deliberate cross-layer inspection is necessary because the BTH is the only header the source NIC fully controls that carries the transport-session context needed to make this decision.

#### How the Decision Is Made

The source NIC does not blindly guess — it uses an **auto-negotiation** handshake with the destination NIC when establishing a RoCE connection. During this exchange, each NIC reports its capabilities, including whether it can process packets that arrive out of order. The source NIC records this capability per peer and uses it to stamp every outgoing packet:

1. **Connection Setup**: Source NIC initiates a connection to a destination NIC.
2. **Capability Exchange**: The destination NIC reports whether it supports OOO (out-of-order) packet processing.
3. **Per-Packet Stamping**: For every subsequent packet in this connection, the source NIC sets the AR bit based on the negotiated result — `1` if the destination supports OOO, `0` if it does not.
4. **Switch Action**: Every switch along the path extracts the bit and applies AR or static routing accordingly.

This means the network automatically adapts to the capabilities of each endpoint. Traffic destined for a modern ConnectX-7 with reorder hardware gets full Packet-Spray benefits, while traffic destined for a legacy NIC is safely pinned to static ECMP — all within the same fabric, at the same time, with zero operator intervention per flow.

#### Override Mechanisms

Two override mechanisms provide additional control:

- **Host Override**: The server's host software (OS or driver) can override the NIC's auto-negotiation result. Even if the destination NIC advertised OOO support, the host can force AR = 0 for specific connections. This handles edge cases such as a destination NIC with known reorder bugs, or an application that requires strict ordering regardless of hardware capability.

- **Per-Packet Exception**: The source NIC can force static routing for individual packets within a flow that is otherwise AR-eligible. Certain control messages (e.g., connection management packets) within an RDMA session may require strict ordering even when the bulk data transfer can tolerate reordering.

> **Reference:** This mechanism is described in Mellanox/NVIDIA [US Patent 12,328,251 B2](https://patents.google.com/patent/US12328251B2/en), *"Marking of RDMA-over-Converged-Ethernet (RoCE) Traffic Eligible for Adaptive Routing,"* granted June 2025. The patent covers the NIC-side marking, the switch-side extraction, and the auto-negotiation protocol. Juniper Networks has since adopted a similar approach using the BTH for load-balancing decisions ([US 12,526,234 B1](https://patents.google.com/patent/US12526234B1/en), granted January 2026).


## Measuring Load Balance: Coefficient of Variation (CoV)

The preceding sections describe several approaches to improving RoCEv2 traffic distribution. Static ECMP, flowlet switching, and packet spraying are network-level load balancing strategies (decisions made by switches), while MQP is an endpoint software technique that creates more flows for ECMP to distribute. Each offers progressively better traffic distribution. But how do we *quantify* "better"? Saying one approach is "more balanced" than another is meaningless without a metric. The industry-standard metric for this is the **Coefficient of Variation (CoV)** of per-link load.

### What Is CoV?

CoV is the ratio of the standard deviation to the mean of a set of measurements:

```
CoV = σ / μ
```

Where:
- **σ** (standard deviation) measures how much individual link loads deviate from the average.
- **μ** (mean) is the average load across all links.

The result is a dimensionless number that captures how *uniformly* traffic is spread across available paths, regardless of total traffic volume or the number of links.

### Interpreting CoV

- **CoV = 0**: Every link carries exactly the same load. Perfect balance.
- **CoV > 0**: Some links carry more traffic than others. The higher the value, the worse the imbalance.

Because CoV is normalized by the mean, it allows fair comparison across different fabric sizes and traffic volumes. A CoV of 0.25 means the same thing whether you have 4 spine links or 64 — roughly 25% variation around the average load.

### Calculating CoV: A Simple Example

Suppose a Leaf-Spine fabric has 4 spine links. During an AllReduce collective, we measure the throughput on each link over a 1-second window:

```
Link 1:  350 Gb/s
Link 2:  280 Gb/s
Link 3:  310 Gb/s
Link 4:  260 Gb/s
```

**Step 1 — Compute the mean (μ):**

```
μ = (350 + 280 + 310 + 260) / 4 = 1200 / 4 = 300 Gb/s
```

**Step 2 — Compute each link's deviation from the mean:**

```
Link 1:  350 - 300 = +50
Link 2:  280 - 300 = -20
Link 3:  310 - 300 = +10
Link 4:  260 - 300 = -40
```

**Step 3 — Square the deviations and compute the variance:**

```
Variance = (50² + 20² + 10² + 40²) / 4
         = (2500 + 400 + 100 + 1600) / 4
         = 4600 / 4
         = 1150
```

**Step 4 — Take the square root to get the standard deviation (σ):**

```
σ = √1150 ≈ 33.9 Gb/s
```

**Step 5 — Divide by the mean to get CoV:**

```
CoV = σ / μ = 33.9 / 300 ≈ 0.113
```

A CoV of ~0.11 tells us there is moderate imbalance. For comparison, if all four links carried exactly 300 Gb/s, CoV would be 0 (perfect balance). If one link carried all 1,200 Gb/s and the rest carried nothing, CoV would be 1.73 (catastrophic imbalance).

### Why CoV Matters for AI Training

Consider a Leaf-Spine fabric with 8 spine links, each rated at 400 Gb/s (3,200 Gb/s aggregate bisection bandwidth). If traffic is perfectly balanced (CoV = 0), every link carries 400 Gb/s and the fabric delivers full bisection bandwidth. But if traffic is skewed (CoV > 0), some links saturate while others sit idle:

```
Example: 8 spine links, 2,400 Gb/s total traffic, CoV = 0.25

  Mean load per link:  μ = 2400 / 8 = 300 Gb/s
  Standard deviation:  σ = CoV × μ = 0.25 × 300 = 75 Gb/s

  Hottest links:       300 + 75 = 375 Gb/s  (near saturation at 400 Gb/s)
  Coldest links:       300 - 75 = 225 Gb/s  (significant spare capacity)
```

The fabric has 800 Gb/s of unused capacity (on the cold links), yet the hot links are approaching congestion. One more flow hashing to a hot link triggers packet drops, PFC pauses, or ECN marking — even though aggregate utilization is only 75%. With perfect balance, the fabric could absorb 33% more traffic before any link hits capacity.

This is why CoV directly translates to **effective fabric throughput** for AI collectives. AllReduce, AllGather, and AllToAll are *synchronized* — the collective completes at the speed of the *slowest* transfer. A single overloaded link (caused by hash collision) bottlenecks the entire operation across all GPUs, while the underloaded links waste capacity that cannot be reclaimed.

### CoV by Load Balancing Strategy

The following table summarizes typical CoV values for each approach discussed in this document, measured on production-scale Leaf-Spine fabrics under AI training workloads:

| Strategy                      | Typical CoV | Why |
|-------------------------------|-------------|-----|
| **Static ECMP (single QP)**   | ≈ 0.25      | One hash per flow → elephant flows pile on random links |
| **Flowlet switching**         | ≈ 0.10      | Reroutes during idle gaps, but continuous streams still pin |
| **MQP (16 QPs)**              | ≈ 0.07      | More flows improve statistical distribution, but collisions remain |
| **Packet spraying**           | ≈ 0.03      | Per-packet distribution, near-ideal but limited by reorder buffers |
| **SRv6 source routing (MRC)** | ≤ 0.02      | Deterministic path selection across all planes, no hash collisions |

The progression is clear: each strategy reduces CoV by attacking a different source of imbalance. Static ECMP suffers from hash collisions and flow pinning. MQP reduces the impact of individual collisions by creating more flows. Flowlet and packet spraying introduce dynamic path selection. MRC with SRv6 eliminates hash-based randomness entirely, achieving near-zero CoV by deterministically distributing packets across all available paths.

### CoV as a Design Target

In practice, network architects use CoV thresholds to guide design decisions:

- **CoV > 0.20**: Unacceptable for large-scale AI training. Frequent hotspots cause PFC storms or packet loss. Requires intervention (MQP at minimum).
- **CoV 0.10–0.20**: Tolerable for smaller clusters or less bandwidth-intensive workloads. Flowlet switching typically achieves this range.
- **CoV 0.05–0.10**: Good. MQP with sufficient QP count, or flowlet with cooperative traffic patterns.
- **CoV < 0.05**: Excellent. Requires per-packet load balancing (spraying or MRC). The fabric operates near its theoretical maximum throughput.

These thresholds explain why the industry has progressively moved from ECMP toward packet-level load balancing for AI workloads — the performance cost of imbalance is too high when thousands of GPUs synchronize on every collective operation.



## Cell-Based Switching (Distributed Scheduled Fabric)

The [Packet Spraying](#packet-spraying) section above described two endpoint-side approaches to handling out-of-order arrival: hardware reorder buffers and true OOO placement. Both assume a standard Ethernet fabric where packets can arrive out of order. A fundamentally different architecture solves the problem inside the **network itself**, so the NIC needs no reordering hardware at all.

A **Distributed Scheduled Fabric (DSF)** makes the entire multi-switch fabric behave as a **single Ethernet node**. Leaf switches act as line cards and Spine switches act as the backplane of what appears to be one enormous crossbar switch. The mechanism works in three steps:

1. **Segmentation**: The ingress Leaf chops each Ethernet frame into fixed-size **cells** (typically 64–256 bytes), each tagged with a sequence number. Cells are placed into **Virtual Output Queues (VOQs)** — one queue per destination Leaf — to prevent head-of-line blocking.

2. **Deterministic Spraying**: Cells are distributed across all Spines using a mathematically precise schedule (not a hash), guaranteeing exactly equal load with no hash collisions.

3. **Reassembly**: The egress Leaf collects cells from all Spines, reassembles them into the original frame using the sequence numbers, and delivers a perfectly ordered stream to the NIC.

From the server's perspective, the multi-hop fabric is invisible — it behaves as if every server is plugged into the same single switch. This makes DSF compatible with any NIC and any transport protocol (including standard RoCE) with no endpoint modifications.

DSF was designed as a **high-performance Ethernet alternative to InfiniBand**, achieving comparable lossless guarantees and load balancing while remaining on standard Ethernet at the server-facing interface. This allows operators to connect GPUs and NICs from any vendor that supports standard Ethernet or RoCE, avoiding InfiniBand's single-vendor ecosystem.

The trade-offs are significant. Both ingress and egress Leaf switches must perform segmentation, scheduling, and reassembly at line rate, which demands specialized ASICs and higher switch cost. DSF requires a **homogeneous fabric** — all switches in the path must run the same vendor's cell-based protocol, and mixing DSF switches with standard Ethernet switches is not possible. For this reason, DSF is often considered a **closed vendor solution**, architecturally similar to InfiniBand in that it relies on proprietary hardware and tightly coupled traffic management. The primary deployment today is in AI training backends at hyperscale operators like Meta.
