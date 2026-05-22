
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

The most common software-level approach to improving RoCEv2 load balancing is **QP scaling**, also known as **Multiple Queue Pairs (MQP)**. The core idea is simple: instead of sending a transfer over a single QP, the application (or collective communication library such as NCCL) opens multiple QPs to the same destination. Each QP is assigned a different UDP source port, so ECMP treats each one as a separate flow and hashes them independently across different spine links — turning one elephant flow into several smaller, well-distributed flows.

For example, if a single 400 Gb/s transfer is split across 4 QPs, each QP carries approximately 100 Gb/s. Because ECMP hashes each QP independently, the probability that all four collide on the same link drops significantly compared to a single QP.

There are two common MQP strategies:

| Strategy        | How it works | Best for |
|-----------------|-------------|----------|
| **Split**       | A single large message is divided across multiple QPs simultaneously | Latency-sensitive large transfers |
| **Round-robin** | Successive messages are posted to different QPs in rotation | Sustained throughput workloads |

Meta deployed QP scaling across their production AI training clusters and measured its impact on AllReduce bandwidth. The figure below (from [Meta's SIGCOMM 2024 paper](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/)) compares these two strategies at QP counts of 1, 4, and 16. Key findings:

- **Round-robin with 16 QPs** achieves near-100% normalized bandwidth for large messages (2 GB).
- **Single QP** (the "out of box" baseline) reaches only ~60%.

The improvement is most pronounced for large messages where elephant flows dominate; for small messages (2–8 MB), even 16 QPs provide limited benefit because the transfers are too short to create sustained congestion.

<img src="./pics/qp-scaling-meta.png" width="550"/>

However, QP scaling has practical limits:

- **Diminishing returns**: Experiments show negligible improvement beyond 8–16 QPs. With more QPs, the probability of *some* collision remains high (birthday paradox with more draws), and the overhead of managing additional QPs increases.

- **Application complexity**: The sending and receiving applications must coordinate multiple QPs, manage multiple completion queues, and ensure correct data reassembly. Collective communication libraries absorb this complexity, but it adds software overhead.

- **Does not eliminate collisions**: QP scaling reduces the probability of worst-case collisions but does not prevent them. At large scale (hundreds of thousands of QPs across thousands of nodes), collisions are still statistically inevitable.

QP scaling is a pragmatic improvement within the constraints of single-path RC, but it does not solve the fundamental problem. True resolution requires either the network or the transport protocol to support per-packet multi-path operation.


## Adaptive Routing for RoCEv2

The [Adaptive Routing](./02_README_ARS.md) documentation describes how modern switch ASICs overcome ECMP's congestion blindness by dynamically steering traffic based on real-time port congestion. The two primary algorithms (flowlet switching and packet spraying) apply to RoCEv2 traffic, but the RC ordering constraint changes the calculus significantly.

### Flowlet Switching

In flowlet mode, the switch monitors each flow for idle gaps (periods where no packets are in transit). When a gap exceeds the configured idle time (which must be greater than the maximum path latency skew), the switch knows all previous packets have been delivered. It can safely reassign the flow to a less-congested spine link without causing reordering.

For RoCEv2, flowlet switching is the safe, protocol-compatible choice because it preserves in-order delivery. However, AI training collectives often generate continuous, high-bandwidth streams with minimal idle gaps. A GPU performing an all-reduce at line rate produces a near-continuous packet stream — the switch may rarely or never observe a gap long enough to trigger a reroute. In this scenario, a massive elephant flow remains pinned to a congested link for its entire duration, and flowlet switching degrades to behaving like static ECMP.

### Packet Spraying

In packet-spray mode, the switch abandons flow affinity entirely. Every packet is independently routed to the least-congested spine link. This shatters elephant flows across all available paths, achieving near-perfect link utilization.

The problem is that packets from the same QP take different paths with different latencies and arrive out of order. Standard RoCEv2 NICs interpret this as packet loss and trigger Go-Back-N retransmission, destroying throughput. Packet spraying therefore cannot be used with standard RC unless the receiving NIC can handle reordering.

Two approaches have emerged, each removing a constraint that limited the previous one:

**Approach 1: Hardware reorder buffer (tolerate limited reordering)**

For packet spraying to work, the receiving NIC must handle out-of-order arrival. Modern HCAs (such as NVIDIA ConnectX-7) include dedicated **hardware reorder buffers** — on-chip SRAM that holds early-arriving packets, waits for the missing earlier packets to arrive, and then writes the reassembled, correctly ordered data to host memory. The application never sees out-of-order data.

However, reorder buffers have a fundamental limitation: they are **finite**. The buffer can only hold a limited number of out-of-order packets at a time. At 400–800 Gb/s line rates, the bandwidth-delay product across a multi-hop fabric can exceed the buffer capacity. When too many packets arrive out of order — or the gap between the earliest and latest packet grows too large — the buffer overflows. Overflowed packets must be dropped, triggering Go-Back-N retransmission and negating the throughput benefit of spraying. This limits the degree of path diversity that reorder-buffer NICs can tolerate.

**Approach 2: True out-of-order placement (eliminate reordering entirely)**

Rather than buffering and reassembling, [MRC](#mrc-multipath-reliable-connection) makes every packet self-describing — each carries the full RDMA virtual address, so the receiving NIC writes it directly to the correct memory position on arrival, regardless of order. No reorder buffer is needed, no packet is dropped for arriving early, and path diversity is no longer constrained by buffer size.



## Measuring Load Balance: Coefficient of Variation (CoV)

The preceding sections describe several load balancing strategies — static ECMP, MQP, flowlet switching, and packet spraying — each offering progressively better traffic distribution. But how do we *quantify* "better"? Saying one approach is "more balanced" than another is meaningless without a metric. The industry-standard metric for this is the **Coefficient of Variation (CoV)** of per-link load.

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



## MRC: Multipath Reliable Connection

The preceding sections traced a progression of increasingly severe problems: [single-path RC pinning](#the-rc-ordering-constraint) causes [elephant flow collisions](#why-this-breaks-at-ai-scale) that ECMP cannot resolve, [QP scaling](#qp-scaling-the-software-workaround) provides diminishing returns, [flowlet switching](#flowlet-switching) cannot reroute continuous streams, and [packet spraying](#packet-spraying) violates RC's ordering guarantees. Each workaround addresses a symptom while leaving the root constraints intact: single-path delivery, Go-Back-N retransmission, and PFC-enforced losslessness.

RoCEv2 transports InfiniBand packets over Ethernet by encapsulating them inside UDP/IP. The InfiniBand specification defines three primary transport services — RC, UC, and UD (see the [InfiniBand deep dive](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/02_README_INFINIBAND.md#transport-services)) — and RoCEv2 reuses them directly. **Multipath Reliable Connection (MRC)** adds multipath capabilities on top of RC, exclusively in the RoCEv2/Ethernet context. It does not define a new InfiniBand transport service, nor does it operate over native InfiniBand fabrics. MRC was released as an open specification through the Open Compute Project (OCP) in May 2026. It eliminates the root constraints identified above.

MRC was developed collaboratively by OpenAI, Microsoft, NVIDIA, AMD, Broadcom, and Intel. It has been implemented in 400 and 800 Gb/s NICs (NVIDIA ConnectX-8, AMD Pollara/Vulcano, Broadcom Thor Ultra) and is deployed in production across OpenAI's largest training clusters, Microsoft's Fairwater data centers, and Oracle Cloud Infrastructure's (OCI) Abilene facility, where it has been used to train frontier large language models for ChatGPT and Codex.

### How MRC Works

Rather than incremental patches to these problems, MRC rethinks the interaction between the transport protocol, the network topology, and the routing layer.

#### Packet Spraying Across Hundreds of Paths

The most fundamental change in MRC is the elimination of single-path flow pinning. Instead of binding a QP to one ECMP path, MRC **sprays** packets from a single QP across hundreds of network paths simultaneously, spanning all planes in a multi-plane topology.

At QP startup, the sender generates a set of **Entropy Values (EVs)** — typically 128 to 256 entries — each mapping to a unique path through the network. In a standard ECMP deployment, the EV is embedded in the packet's UDP source port and IPv6 flow label, causing each packet to hash to a different ECMP path. In production deployments, MRC replaces ECMP with SRv6 static source routing (described in the [SRv6 section](#static-source-routing-with-srv6) below), where each EV maps directly to a specific SRv6-encoded path — but the spraying principle is the same. The sender rotates through the EV set, sending consecutive packets over different paths. This transforms what was a single "elephant flow" into a fine-grained spray distributed evenly across the fabric.

Because the spray is per-packet rather than per-flow, load balancing operates at the finest possible granularity. Different senders do not coordinate their EV sets — randomized selection is sufficient because the aggregate effect across hundreds of QPs naturally distributes load.

#### Out-of-Order Delivery with Immediate Memory Placement

Packet spraying means packets traverse paths of different lengths and congestion states, so they inevitably arrive out of order. As established in [The RC Ordering Constraint](#the-rc-ordering-constraint), this is fatal for standard RC — the RETH with the destination address is only in the first packet, so subsequent packets depend on in-order arrival.

MRC solves this by including the **full RDMA virtual address and `R_Key` in every data packet**. This allows the receiving NIC to write each packet directly to its correct position in the application's memory buffer the instant it arrives, regardless of arrival order. No reordering buffer is needed, and no packet is discarded simply because it arrived ahead of an earlier one.

> At the transport level, MRC only supports RDMA Write and Write-with-Immediate operations. These are the dominant operations in AI collective communication libraries (such as NCCL), so this restriction has no practical impact on AI training workloads.

#### Selective Retransmission (SACK/NACK)

With out-of-order delivery handled, MRC replaces Go-Back-N with **selective retransmission**. Instead of cumulative ACKs that only confirm the highest in-order sequence number, MRC uses Selective ACK (SACK) packets that report exactly which packets have arrived and which are missing. When a gap is detected, the sender retransmits only the specific lost packets rather than the entire window.

This dramatically reduces the bandwidth wasted on unnecessary retransmissions, especially at 400–800 Gb/s line rates where a Go-Back-N window can contain thousands of packets. The difference is quantifiable:

```text
At 800 Gb/s with 9 KB jumbo frames:
  Packets in flight (BDP) = 800 Gbps × 10 µs RTT = 1 MB ≈ 108 packets

Go-Back-N (standard RC):
  1 lost packet → retransmit from lost PSN onward ≈ N/2 = ~54 packets
  Throughput loss per event = 54/108 ≈ 50%
  At 1% packet loss rate:  throughput ≈ 1/(1 + N×p/2) = 1/1.54 ≈ 65%  → 35% wasted

Selective Retransmission (MRC):
  1 lost packet → retransmit only that 1 packet
  At 1% packet loss rate:  throughput > 99%  → <1% wasted
```

At the same 1% loss rate, Go-Back-N wastes 35% of bandwidth on redundant retransmissions while selective retransmission wastes less than 1% — a difference that determines whether a training job survives a transient loss event or stalls.

#### Disabling PFC: Operating on Lossy Ethernet

Because MRC sprays a single QP's packets across hundreds of paths, a flow reaches the last-hop switch over many different ingress links. PFC, which pauses an entire ingress port, would indiscriminately throttle packets from many unrelated flows that happen to share those links. MRC therefore **disables PFC entirely** and runs Ethernet in best-effort (lossy) mode.

This is a deliberate trade-off: accepting occasional packet loss in exchange for eliminating PFC-induced [head-of-line blocking and congestion spreading](https://github.com/ManiAm/GNS-QOS/blob/master/docs/04_PFC.md#the-dangers-of-pfc-managing-the-lossless-safety-net). The selective retransmission mechanism handles the resulting losses efficiently, making the system more predictable under stress.

#### Packet Trimming for Incast

To accelerate loss recovery, especially during incast (many-to-one) traffic patterns, MRC uses **packet trimming**. When a switch would otherwise drop a packet due to buffer overflow, it instead strips the payload and priority-forwards only the header to the destination. The receiving NIC recognizes the trimmed packet and immediately generates a NACK, triggering fast retransmission on an alternate path.

Packet trimming serves a dual purpose: it provides faster loss notification than timeout-based detection, and it helps MRC distinguish **congestion-induced loss** (trimmed packets arrive as headers) from **path failure** (packets vanish entirely). This distinction is critical for making correct routing decisions.

#### ECN-Based Adaptive Load Balancing

MRC keeps a small amount of per-EV state to track path health. Switches along each path mark packets with ECN (Explicit Congestion Notification) when queues begin to build. The receiver echoes the ECN signal back to the sender in SACK packets, tagged with the specific EV that experienced congestion. The sender then temporarily avoids that EV, redistributing traffic to less-congested paths.

In a network with full bisection bandwidth, sustained congestion in the core indicates an imbalance rather than true oversubscription. ECN-based load balancing smooths out this unevenness, preventing internal queues from growing enough to cause congestive loss. This is fundamentally different from [DCQCN](https://github.com/ManiAm/RDMA-Primer/blob/master/docs/03_README_ROCE.md#dcqcn-and-congestion-control), which reduces the sender's overall injection rate in response to ECN — MRC instead shifts traffic to a better path while maintaining full throughput.

#### Path Failure Detection and Recovery

When a packet is genuinely lost (not trimmed), MRC assumes the corresponding path has failed and immediately removes that EV from the active set, replacing it with a backup EV from the same network plane. This reaction happens within tens of microseconds — orders of magnitude faster than switch-based dynamic routing convergence, which can take seconds.

To prevent permanently retiring paths that suffered a transient error (such as a bit flip from a cosmic ray), MRC sends background **probe packets** on retired EVs. If enough consecutive probes succeed, the EV is restored to the active set. This creates a self-healing loop: failures are bypassed almost instantly, and recovered paths are automatically brought back into service.



### Multi-Plane Topology Co-Design

MRC was co-designed with a specific network topology that maximizes its strengths. Instead of treating an 800 Gb/s NIC as a single high-speed link, the NIC is broken out into multiple smaller links (for example, 8 × 100 Gb/s). Each link connects to a different top-of-rack (T0) switch, creating **eight independent parallel networks** called planes.

Using 51.2 Tb/s switches (the current fastest Ethernet switching silicon), each switch at 100 Gb/s has 512 ports instead of 64 ports at 800 Gb/s. With 512-port switches, a two-tier Clos topology can connect over **131,000 GPUs** — a scale that would require three or four tiers with conventional 800 Gb/s links.

The multi-plane design delivers several advantages:

- **Lower latency**: The longest path traverses 3 switches instead of 5 or 7.
- **Higher redundancy**: Losing a single T0–T1 link reduces a node's capacity by only ~0.4% (in an 8-plane network) versus ~3% in a single-plane design.
- **Graceful NIC port failure**: If one of eight NIC ports fails, the node loses only 12.5% of bandwidth. MRC detects the failure, remaps EVs to avoid the failed plane, and notifies remote peers — the training job continues without interruption.
- **Reduced cost and power**: A two-tier multi-plane network requires roughly 60% of the switches and 67% of the optics compared to an equivalent three-tier single-plane network.

MRC distributes its EV set equally across all planes, ensuring that traffic is inherently balanced between them. This tight coupling between topology and transport is what enables the protocol to fully utilize the available path diversity.



### Static Source Routing with SRv6

Conventionally, switches run dynamic routing protocols (such as BGP) to compute paths and react to failures. However, MRC already handles failure detection and avoidance at the transport layer, making dynamic routing redundant. Worse, having two adaptive mechanisms (MRC at the endpoints and dynamic routing in the switches) interact creates unpredictable behavior: MRC avoids a failed path, then dynamic routing re-converges and changes ECMP mappings, disturbing MRC's load balancing.

MRC addresses this by **disabling dynamic routing entirely** and using **IPv6 Segment Routing (SRv6)** with static forwarding tables in the switches. SRv6 works as follows:

1. At QP startup, the NIC generates the EV set and maps each EV to a specific SRv6 destination address. This address encodes the complete path through the network as a sequence of 16-bit micro-Segment IDs (uSIDs), each identifying a specific switch.

2. When a packet is sent, its IPv6 destination address contains the full path: the first uSID identifies the first-hop switch, the second identifies the next switch, and so on.

3. At each hop, the switch checks if its own uSID is present, then **left-shifts** the address by 16 bits to expose the next hop's uSID. It looks up this new address in a static forwarding table (configured once at installation and never changed) and forwards the packet out the corresponding port.

4. The MRC NIC encapsulates packets as IPv6-in-IPv6, with the outer address being the SRv6 path and the inner address being the destination NIC's actual address for decapsulation.

Because the forwarding tables are static and the path is fully determined by the sender, there is no routing convergence delay, no ECMP ambiguity, and no switch-level computation. Failures are handled exclusively by MRC removing the corresponding EV from its active set.

This design also provides excellent **observability**: because each EV deterministically maps to a specific physical path, when MRC reports a bad EV, operators can immediately identify the exact failed link or switch — something that is extremely difficult with hash-based ECMP, where the mapping from flow to path is opaque.



### MRC vs. InfiniBand RC: Key Differences

MRC and InfiniBand RC serve fundamentally different deployment contexts. The following table summarizes the core architectural differences:

| Aspect                       | InfiniBand RC                              | MRC (extends RoCEv2 RC)                         |
| ---------------------------- | ------------------------------------------ | ----------------------------------------------- |
| Network Type                 | Native InfiniBand (lossless fabric)        | Ethernet (best-effort / lossy)                  |
| Path Model                   | Single path per QP                         | Hundreds of paths per QP (packet spraying)      |
| Packet Ordering              | Strict in-order delivery required          | Out-of-order delivery with immediate placement  |
| Loss Recovery                | Go-Back-N (retransmit entire window)       | Selective retransmission (SACK/NACK)            |
| Flow Control                 | CBFC (per-VL credit-based, lossless)       | No PFC; lossy Ethernet with packet trimming     |
| Congestion Response          | FECN/BECN reduces sender injection rate    | ECN steers traffic to less-congested paths      |
| Failure Detection            | Subnet Manager re-sweeps (seconds)         | Per-path EV removal (microseconds)              |
| Routing                      | SM-computed LFTs in switches               | SRv6 static source routing from NIC             |
| Supported Operations         | All (Send/Recv, Write, Read, Atomics)      | RDMA Write and Write-with-Immediate only        |
| Target Scale                 | Thousands of nodes (single subnet)         | 100,000+ GPUs (multi-plane Ethernet)            |

MRC is not a replacement for InfiniBand RC in all scenarios. It targets a specific and critical use case: sustaining predictable, high-throughput collective communication across massive Ethernet-based AI training clusters where flow collisions, PFC storms, and slow failure recovery would otherwise cripple training efficiency.



### Production Impact

In production deployments at OpenAI and Microsoft, MRC has demonstrated several concrete operational improvements:

- **Link flap tolerance**: During training of frontier models, multiple link flaps per minute between T0 and T1 switches had no measurable impact on synchronous pretraining. Repair of these links became a low-priority maintenance activity rather than an urgent operational event.

- **Switch failure resilience**: When T1 switches had to be rebooted during active training, MRC progressively detected that the affected EVs were failing (each individual EV is retired within tens of microseconds) and redistributed traffic across remaining paths. The full recovery — retiring all EVs traversing the failed switch and rebalancing — completed within seconds. The training job continued without coordination between network operations and the team running the training job.

- **NIC port failure survival**: Before MRC, a failed link between a GPU's NIC and its T0 switch would crash the training job. With MRC, the job survives with reduced bandwidth (losing one of eight planes reduces capacity by 12.5%), and most such failures recover within a minute.

- **Performance under loss**: In controlled experiments comparing MRC against RoCEv2 on identical hardware, a single MRC QP spraying across 256 paths outperformed 16 RoCEv2 QPs in all-reduce collectives. Under 0.1% induced packet loss, MRC maintained near-line-rate throughput for large messages, while RoCEv2 performance collapsed due to Go-Back-N amplification and PFC-induced blocking.

- **Zero collateral damage**: In 7-to-1 incast experiments, MRC perfectly shared the bottleneck link among incast flows with zero impact on a concurrent "victim" flow on the same fabric. RoCEv2 with DCQCN degraded the victim flow's throughput by 25–75% depending on QP count.


## Cell-Based Switching (Distributed Scheduled Fabric)

The [Packet Spraying](#packet-spraying) section above described two endpoint-side approaches to handling out-of-order arrival: hardware reorder buffers and true OOO placement. Both assume a standard Ethernet fabric where packets can arrive out of order. A fundamentally different architecture solves the problem inside the **network itself**, so the NIC needs no reordering hardware at all.

A **Distributed Scheduled Fabric (DSF)** makes the entire multi-switch fabric behave as a **single Ethernet node**. Leaf switches act as line cards and Spine switches act as the backplane of what appears to be one enormous crossbar switch. The mechanism works in three steps:

1. **Segmentation**: The ingress Leaf chops each Ethernet frame into fixed-size **cells** (typically 64–256 bytes), each tagged with a sequence number. Cells are placed into **Virtual Output Queues (VOQs)** — one queue per destination Leaf — to prevent head-of-line blocking.

2. **Deterministic Spraying**: Cells are distributed across all Spines using a mathematically precise schedule (not a hash), guaranteeing exactly equal load with no hash collisions.

3. **Reassembly**: The egress Leaf collects cells from all Spines, reassembles them into the original frame using the sequence numbers, and delivers a perfectly ordered stream to the NIC.

From the server's perspective, the multi-hop fabric is invisible — it behaves as if every server is plugged into the same single switch. This makes DSF compatible with any NIC and any transport protocol (including standard RoCE) with no endpoint modifications.

DSF was designed as a **high-performance Ethernet alternative to InfiniBand**, achieving comparable lossless guarantees and load balancing while remaining on standard Ethernet at the server-facing interface. This allows operators to connect GPUs and NICs from any vendor that supports standard Ethernet or RoCE, avoiding InfiniBand's single-vendor ecosystem.

The trade-offs are significant. Both ingress and egress Leaf switches must perform segmentation, scheduling, and reassembly at line rate, which demands specialized ASICs and higher switch cost. DSF requires a **homogeneous fabric** — all switches in the path must run the same vendor's cell-based protocol, and mixing DSF switches with standard Ethernet switches is not possible. For this reason, DSF is often considered a **closed vendor solution**, architecturally similar to InfiniBand in that it relies on proprietary hardware and tightly coupled traffic management. The primary deployment today is in AI training backends at hyperscale operators like Meta.
