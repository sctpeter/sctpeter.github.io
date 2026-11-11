---
date: '2026-11-11T06:34:26-08:00'
draft: false
categories:
  - Networking
title: 'Notes on Datacenter Ethernet and RDMA: Issues at Hyperscale'
---

> Some figures and content in this post are adapted from T. Hoefler et al., "Datacenter Ethernet and RDMA: Issues at Hyperscale", arXiv:2302.03337 (https://arxiv.org/abs/2302.03337 ), licensed under CC BY 4.0 (https://creativecommons.org/licenses/by/4.0/ ). What follows are my study notes, which translate, rewrite, and reorganize the original paper.

### Some basic concepts

- ECMP (Equal-Cost Multi-Path routing): when a switch or router has several equal-cost paths to the same destination, it spreads traffic across those paths instead of using only one of them.

### Converged Ethernet

- Traffic classes that used to run on separate dedicated networks are merged onto a single Ethernet network.
- For example, ordinary TCP/IP application traffic, storage networking, and HPC networking all share the same Ethernet.
- A VLAN tag is a 4-byte field added to an Ethernet frame in certain cases to indicate which VLAN the frame belongs to. The VLAN tag contains a 3-bit field called PCP (Priority Code Point), with values 0–7. It is essentially a traffic-class number: the switch uses a mapping table to place the frame into the corresponding queue based on this number. Each queue has its own small dedicated buffer and can also borrow a certain amount from a shared buffer pool. Different queues can be given different bandwidth shares, and PFC (priority flow control) can be enabled or disabled independently per queue.
- PFC (priority flow control)
  - When a buffer queue grows to the pause threshold, a queue with PFC enabled sends a PAUSE frame upstream, while a queue without PFC simply drops frames.
- The usual practice is to put RoCE traffic in a lossless priority with PFC enabled, and leave ordinary TCP traffic in a lossy priority without PFC. Lossy and lossless traffic run side by side on the same Ethernet, and every frame, lossy or lossless, falls into one of these 8 classes (0–7).

### RDMA

- Remote direct memory access.
- RDMA is the ability of a NIC to read and write remote memory directly, without going through the remote CPU.
- There are several implementations, such as InfiniBand, RoCE, and iWARP.

### RoCE

- RoCEv1
  - Places the InfiniBand transport-layer header directly on top of the Ethernet data-link-layer L2 headers. It has no IP header, so it cannot be routed across subnets.
- RoCEv2
  - Adds IP/UDP L3 headers. The IP header supports routing within a datacenter and between datacenters; the UDP header provides compatibility with existing Ethernet equipment. For example, a switch can read the source port at the UDP position and use it for ECMP load splitting (the destination port for RoCE is always 4791, while source ports can differ).
- Because RoCE uses the InfiniBand transport layer, it requires in-order packet delivery and uses go-back-N retransmission (if one packet fails, the packets after it that did arrive are discarded too, and retransmission starts from the failed packet). The in-order delivery requirement makes multipath load balancing difficult.
- Congestion control
  - RoCEv2 congestion control relies on IP's Explicit Congestion Notification (ECN) signal. When a switch becomes congested, it marks packets; the receiver then relays the congestion information back to the sender, which slows down.
- Link-layer flow control
  - RoCE uses PFC, which operates per hop between adjacent devices, as described above.
  - InfiniBand differs from RoCE in that it uses credits: the receiver tells the sender in advance how much space it has, and the sender only transmits when it holds credits. This is proactive and fundamentally cannot overflow. (Credits held by the sender + data in flight on the link + data in the receiver's buffer not yet forwarded + credits in flight back on the link = the total allocated buffer.)

### PFC vs. ECN

- PFC: when a queue reaches the pause threshold, the switch sends a PAUSE to the previous hop as a no-drop safety net; the buffer reserved above the threshold absorbs packets still in flight before the PAUSE takes effect. ECN: as soon as a switch starts getting congested it marks packets, and the receiver relays the congestion information back to the sender so the sender reduces its rate.

### Communication patterns

- Incast (IN)

  - Many senders send packets to the same process, in a random, uncoordinated way. This communication pattern can cause congestion.

  - When the amount of data being communicated is smaller than the bandwidth-delay product, congestion control cannot notify the senders before the communication finishes.

- Oblivious bulk synchronous (OBS)

  - "Oblivious" means the communication pattern does not depend on the actual data, only on a few small hyperparameters.
  - Bulk synchronous: communication happens in synchronized bulk phases.

- Latency-sensitive (LS)

### Problems with RoCE

- Priority Flow Control (PFC) requires a lot of buffer
  - Because of the propagation delay of PAUSE messages, a switch must have enough buffer to accept the packets the sender transmits during that delay, which wastes buffer space. Freeing up this buffer space could lower the price of switch chips, or the extra buffer could be used to absorb transient bursts.
- Victim flows, congestion trees, PFC storms, and deadlocks
  - Victim flows: among different flows assigned to the same traffic-class number, as soon as one of them fills the buffer, none of the others can get through.
  - Congestion trees / PFC storms: congestion propagates hop by hop upstream against the direction of the links.
  - Deadlocks: if routing causes the wait-for relationships between switch buffers to form a cycle, then once all the buffers on the cycle are full, each switch must wait for the next hop to free up space before sending, and the next hop is likewise waiting for its own next hop. Every sender on the cycle stops sending, and no buffer can ever drain.
- Go-back-N retransmission
  - Delivery is in order; after a loss, retransmission starts from the lost packet, and all later packets are discarded even if they arrived, so everything is sent again. This wastes resources.
  - No support for multipath or out-of-order delivery.
- Congestion control
  - ECN provides only a binary "congested or not" mark. The information is too coarse, and the sender needs many RTTs to converge to an appropriate rate.
  - Ordinary datacenter switches have relatively large buffers to reduce the probability of packet loss. Because end-to-end latency in this kind of transport is long, it is hard to tell senders to slow down quickly and hard to make the network lossless, so the only option is to make buffers somewhat larger to reduce drops.
  - HPC machine rooms typically use shallow buffers plus a lossless network to reduce queuing delay and the cost of loss and retransmission. Losslessness is guaranteed by link-level flow control (such as InfiniBand credits). At the same time, cables in the machine room are short, so each hop needs only a small reserved buffer; the topology has a small diameter, congestion control reacts quickly, and queues do not build up easily. Shallow buffers are therefore sufficient, which also lowers switch cost.
  - Buffer waste in RoCE
    - The lossless requirement locks up part of the buffer in advance so that no packets are dropped when PFC is triggered. This portion normally sits unused.
    - To avoid triggering PFC, congestion control deliberately keeps queues very short, so the large buffers that were paid for are left idle on purpose.
    - Either go the HPC route and use shallow buffers to make the network fully lossless, or switch to a lossy design so the buffers are actually used to absorb bursts. RoCE datacenters are stuck in between and get the full benefit of neither.
- Headers
  - The headers are complex and carry considerable overhead.
- Weak support for user-defined protocols
  - With UDP, users can build custom protocols such as QUIC on top of it, tailoring the protocol to the characteristics of particular message flows. RoCE does not allow this: for example, even if the arrival order of messages does not matter, RoCE still delivers them in order.
- Security
- Link-layer reliability
  - This should be a general problem, not just a RoCE one.
  - As link speeds increase, so does the error rate. Using FEC (forward error correction) adds some waiting overhead and computation overhead. The waiting overhead comes from having to receive a full block of data before error correction can start, and forwarding can only begin after the correction is finished.
- System
  - Overheads that raise latency also increase the latency of congestion-control messages, which leads to more buffer usage and in turn further increases link latency.

---

*This post was translated from the Chinese original by Claude.*
