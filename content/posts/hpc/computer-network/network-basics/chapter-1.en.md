---
date: '2026-10-04T08:45:10-07:00'
draft: false
categories:
  - Networking
title: 'Computer Networking Notes: Chapter 1'
---

Reference: https://gaia.cs.umass.edu/kurose_ross/videos/1/

### Basic Concepts

ISP: Internet Service Provider.

Packet switching: the data to be sent is split into individual packets. Each packet carries a header containing the destination address and is forwarded hop by hop through the network independently. No resources are reserved before communication; links are shared among all packets on demand.

Circuit switching is a way of transmitting data across a network: before communication begins, an end-to-end path is set up between the source and destination, and resources are reserved for this communication along the way. During the communication these resources are used exclusively by it, and they are released only when it ends.

- FDM (Frequency Division Multiplexing): one physical link can carry communication for multiple users; multiplexing means dividing a link's resources so that multiple users can share it. FDM splits the link's available frequency range into several non-overlapping frequency bands, and each user (each connection) is permanently assigned one band.
- TDM (Time Division Multiplexing): TDM divides time into fixed-length frames, and each frame is further divided into several time slots. Each user always occupies the same slot in every frame; during its own slot, the user can use the link's full rate.

### Network Models

| OSI Seven Layers | Five-Layer Model         |
| ---------------- | ------------------------ |
| Application      | Application              |
| Presentation     | ↑ Merged into Application |
| Session          | ↑ Merged into Application |
| Transport        | Transport                |
| Network          | Network                  |
| Data Link        | Link                     |
| Physical         | Physical                 |

The five-layer model is the one that corresponds to real network protocols.

### Data Units

| Layer                 | Data Unit           | Example                     |
| --------------------- | ------------------- | --------------------------- |
| Layer 4: Transport    | Segment / Datagram  | TCP segment, UDP datagram   |
| Layer 3: Network      | Packet              | IP packet                   |
| Layer 2: Data Link    | Frame               | Ethernet frame              |
| Layer 1: Physical     | Bit                 | Electrical/optical signals on the wire |

- The data portion of an Ethernet frame holds exactly one IP packet
- The data portion of an IP packet holds exactly one TCP segment or UDP datagram

### Routers

A router maintains two tables internally:

- **RIB (Routing Information Base, the routing table)**: built from directly connected routes, routing protocols (OSPF, BGP, etc.), and static configuration. It records "all known routes", so a single destination prefix may have several candidate paths. It updates slowly and is not used directly for forwarding. Most routers have a default route; a few core routers have no default route and instead record routes for aggregated prefixes, so they don't need to record every IP.
- **FIB (Forwarding Information Base, the forwarding table)**: the best route for each prefix is picked out of the RIB and organized into a format suited for fast lookup. Every packet has to look it up.

Control plane

- The control plane decides which way to go: it generates directly connected routes from the prefixes configured on its own interfaces, reads the administrator's static configuration, exchanges routing information with neighboring routers, and computes which next hop should be used to reach each prefix. This work doesn't happen often, but the logic is complex (e.g., running shortest-path algorithms). The RIB belongs to the control plane.

Data plane

- Every time a packet arrives, the router looks up the FIB by the destination IP, chooses the longest of the matching prefixes of different lengths, and gets the outgoing interface and next hop. It then rewrites the two MAC addresses in the frame header (the frame must be rewritten for each different layer-2 network crossed at every hop), decrements the TTL in the IP header by 1, and sends the packet out of the corresponding interface. This happens extremely frequently, and the logic is simple. The FIB belongs to the data plane.

### Packet Delay

- Nodal processing: check for bit errors, read the packet header, determine the outgoing interface
- Queuing delay: waiting in the outgoing interface's buffer for the link to become free
- Transmission delay: the time needed to push all of a packet's bits onto the link
- Propagation delay: the time it takes for one bit to propagate from one end of the link to the other

---

*This post was translated from the Chinese original by Claude.*
