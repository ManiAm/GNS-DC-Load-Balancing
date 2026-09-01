
# Data Center Fabric Load Balancing

In a Leaf-Spine fabric, every leaf switch has multiple equal-cost uplinks to the spine tier, creating inherent path diversity. How the network distributes traffic across those paths — and how it adapts when conditions change — determines fabric throughput, latency, and resilience. This project covers the full load-balancing stack, from static ECMP hashing through congestion-aware adaptive routing to the transport-level innovations designed for large-scale AI training.

## Documentation and Learning Path

The following documents are structured progressively. The first covers foundational ECMP mechanics, the second builds on that with congestion-aware adaptive routing, the third applies both to the specific constraints of RDMA transport in AI training fabrics, and the fourth is a protocol-level deep dive into MRC.

- [Fabric Load Balancing](01_README_LB.md): ECMP fundamentals — path discovery, hash functions, flow pinning, hash polarization, resilient hashing, the elephant flow problem, weighted ECMP, and centralized traffic engineering.
- [Adaptive Routing](02_README_ARS.md): How modern switch ASICs overcome ECMP's congestion blindness — port grading, quality scores, flowlet switching, packet spraying, global adaptive routing with upstream notifications (ARN), and the interplay with centralized TE.
- [Load Balancing for RoCEv2](03_README_ROCE_LB.md): Why RDMA's single-path Reliable Connection breaks standard load balancing at AI scale — the RC ordering constraint, QP scaling, adaptive routing for RoCE, measuring balance with CoV, and cell-based switching.
- [MRC: Multipath Reliable Connection](04_README_MRC.md): The OCP multipath transport protocol for AI-scale Ethernet — multi-plane topology, entropy values and per-packet spraying, routing modes (ECMP, Structured EV, SRv6), lossy Ethernet with packet trimming, out-of-order placement, selective retransmission (SACK/NACK), the EV state machine and probe-based failure recovery, NSCC congestion control, the software API, and production impact.
