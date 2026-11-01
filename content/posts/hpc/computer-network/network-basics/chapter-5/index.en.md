---
date: '2026-11-01T05:00:00-07:00'
draft: false
categories:
  - Networking
title: 'Computer Networking Notes: Chapter 5'
---

Reference: https://gaia.cs.umass.edu/kurose_ross/videos/5/

These notes are based on the course by J.F. Kurose & K.W. Ross; copyright of the course material belongs to the original authors (https://gaia.cs.umass.edu/kurose_ross/ppt.php).

### Classification

![Per-router control plane vs. SDN control plane](control-plane.png)

- Routing algorithms can be classified as global or decentralized (whether they need complete cost information for every link in the network), and as static or dynamic.
- A global (centralized) routing algorithm computes shortest paths using complete, global information about the network. The computation can run in one place (e.g. a logically centralized controller), or be replicated on every router. Example: Dijkstra.
- A decentralized (distributed) routing algorithm computes shortest paths iteratively and in a distributed way across the routers. No node has complete cost information for all links: each node starts out knowing only the costs of its directly attached links, then repeatedly exchanges information with its neighbors and iterates, gradually working out the shortest paths to each destination. Example: the distance vector (DV) algorithm, i.e. distributed Bellman-Ford.

### Link State (LS) Algorithm: Dijkstra

- The algorithm itself should be well known.
- Either every router first broadcasts its directly attached links and their costs and then each router runs Dijkstra on its own, or a central controller does the computation for everyone.
- When link costs depend on the current traffic load, routing can oscillate: traffic keeps switching back and forth between paths.

### Distance Vector (DV) Algorithm: Bellman-Ford

- Distance vector: a table held by a router, listing "my current estimate of the shortest distance to every destination in the network".

$$D_x(y) = \min_v \left\{ c_{x,v} + D_v(y) \right\}$$

This is the core update formula: the shortest distance from x to y is the minimum, over every neighbor v of x, of the cost from x to v plus v's shortest distance to y.

- The computation works as shown below.

{{< figure src="/posts/hpc/computer-network/network-basics/chapter-5/dv-algorithm.png" alt="What each node does in the distance vector algorithm" width="349" >}}

- Dynamic topology is the real weakness of distance vector. Take a line A–B–C where the B–C link goes down. B should learn that C is unreachable, but A is still advertising "my distance to C is 2". B can't see the topology and doesn't know that A's route actually goes through B itself, so it wrongly concludes "I can reach C via A, distance 3". A then updates to 4 based on B, B updates to 5, and so on. The two keep pushing each other up, forming a routing loop between A and B.
- If a misconfigured router claims a tiny distance to every node in the network, it can cause routing problems across the network.

### Routing on the Internet

- In practice, scalability and administrative autonomy have to be considered. Autonomy means every network on the Internet has its own operator, who decides which routing protocol runs inside it, which paths its outgoing traffic takes, and how other people's traffic is forwarded once it arrives.
- Routing splits into two cases: routing within an autonomous system (intra-domain routing) and routing between autonomous systems (inter-domain routing). For intra-domain routing, different ASes can use different protocols.
- A gateway router (also called a border router) is a router in an AS that has a link directly to another AS. It does two jobs at once:
  - Externally: it runs BGP with the gateways of neighboring ASes, exchanging information about "which prefixes are reachable";
  - Internally: it is also an ordinary router inside the AS.

### Intra-domain Routing

- Routing Information Protocol (RIP): a distance vector protocol.
- OSPF (Open Shortest Path First): uses Dijkstra's algorithm.
- EIGRP (Enhanced Interior Gateway Routing Protocol).

### OSPF

- Every router first broadcasts its directly attached links and their costs, then each router runs a Dijkstra-like computation on its own.

### Hierarchical OSPF

![Hierarchical OSPF](hierarchical-ospf.png)

### BGP: the Border Gateway Protocol

- It is a path vector protocol.
- BGP lets an autonomous system:
  - Obtain network reachability information from neighboring ASes (eBGP)
  - Decide routes to other networks based on reachability and policy
  - Propagate reachability information to routers inside the AS (iBGP)
  - Tell neighboring ASes which networks it can reach (eBGP)
- Border (gateway) routers run both iBGP and eBGP.
- BGP messages
  - OPEN: the first message sent after the TCP connection is established; authenticates the BGP peer sending the OPEN.
  - UPDATE: adds a new route or withdraws an old one.
  - KEEPALIVE
    - During session setup, after receiving the peer's OPEN, a router replies with a KEEPALIVE to acknowledge it; only then does the session enter the Established state.
    - Between two peers, if there is no UPDATE to send, KEEPALIVEs should be sent periodically. If neither an UPDATE nor a KEEPALIVE arrives from the peer within the set time, the peer is considered dead: the connection is closed and the routes learned from that peer are withdrawn.
  - NOTIFICATION: reports errors, or closes the connection.
- BGP path advertisement
  - Advertises a destination IP prefix
  - With attributes attached
    - AS-PATH: the sequence of ASes a packet passes through from the sender's AS to the prefix.
    - NEXT-HOP: the IP address of the interface, on the gateway router of the next-hop AS along this route, that connects to this AS.
  - An ISP can choose not to advertise certain routes according to its own policy.
- Hot potato routing
  - When a host inside an AS sends to another AS, among the gateways that can reach the destination prefix, choose the one with the lowest intra-AS routing cost.
  - Note that hot potato routing is not the only rule; there may be rules with higher priority.
- ICMP
  - Also carried in the payload of an IP packet, but we don't consider it a transport-layer protocol.
  - When the TTL drops to 0, a router may reply with ICMP to report it. When UDP reaches the destination and the port isn't open, the destination host also replies with ICMP.

---

*This post was translated from the Chinese original by Claude.*
