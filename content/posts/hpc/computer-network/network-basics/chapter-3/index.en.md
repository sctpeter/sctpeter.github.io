---
date: '2026-10-06T09:00:00-07:00'
draft: false
categories:
  - Networking
title: 'Computer Networking Notes: Chapter 3'
---

Reference: https://gaia.cs.umass.edu/kurose_ross/videos/3/

### Basic Concepts

- The network layer provides logical communication between hosts.
- The transport layer provides logical communication between processes running on different hosts. The transport layer itself is not responsible for moving data from one host to another; the actual cross-host delivery is done by the network layer and the layers below it. The transport layer runs only on the two end hosts, and what it does is extend the network layer's "host-to-host" delivery into "process-to-process" delivery, which is why it is described as logical communication.

### Multiplexing and Demultiplexing

- A socket can be thought of as a communication interface attached to a process; it is not tied to any particular layer. Except for a TCP listening socket, which can only receive, a socket can both send and receive.
- Multiplexing: the sender's transport layer gathers data from multiple sockets, adds a transport-layer header containing port numbers to each piece of data, encapsulates it into a segment, and hands it to the same network layer for sending.
- Demultiplexing: the receiver's transport layer uses the port numbers in the segment header (along with the addresses in the IP header) to dispatch the segments handed up by the same network layer to the correct socket.
- Sockets are distinguished by the following identifiers
  - **UDP socket**: (local IP, local port)
  - **TCP listening socket**: (local IP, local port)
  - **TCP connection socket**: (local IP, local port, remote IP, remote port)

### UDP

- User Datagram Protocol
- The UDP segment looks like this

{{< figure src="/posts/hpc/computer-network/network-basics/chapter-3/udp-segment.png" alt="UDP segment format" width="245" >}}

- UDP implements only the basic functions, such as multiplexing, demultiplexing and checksumming.
- It is message-oriented and does not guarantee ordering; how to handle that afterwards is up to the application layer.

### RDT

- Reliable data transfer protocol, a protocol defined in the lectures. What is implemented here is a simplified version, meaning it can only transfer information in one direction. The receiver can send messages back to the sender, but those messages are not the information we want to deliver.
- Version 1
  - The sender sends, and the receiver replies (ACK/NAK; ACK stands for acknowledgement, meaning the packet was received and passed the checksum). If the sender does not see an ACK, it retransmits.
  - The problem with version 1 is that if the ACK is corrupted, the sender retransmits the message, and the receiver treats it as a second, independent message from the sender, causing an error.
- Version 2
  - The sender attaches a 1-bit sequence number to each message
  - When the receiver is waiting for the message with sequence number 1: if it receives an intact message with sequence number 1, it switches to the state of waiting for sequence number 0 and returns an ACK; if it receives a message with sequence number 0, it returns an ACK and stays in the same state.
  - The problems with version 2: first, if the ACK is lost rather than corrupted, the sender may keep waiting for the receiver's response instead of retransmitting; also, an ACK that arrives too slowly may be mistaken for the ACK of another packet.
- Version 3
  - On top of version 2, the sender retransmits if no ACK arrives before a timeout, and the ACK also carries that 1-bit sequence number

Performance analysis

- For a link, the number of bits pushed into it can be approximated as proportional to time
- **RTT (Round-Trip Time)** is the total time it takes for a small packet to travel from the sender to the receiver, and for the receiver's response to travel back to the sender.

![Sender utilization of stop-and-wait](stop-and-wait.png)

Solution: pipelining

![Pipelining increases utilization](pipelining.png)

- To know which packet is which, packets obviously have to be numbered. But the numbering wraps around rather than being infinite, so both the sender and the receiver need a window; sequence numbers within this range are the ones that may be sent or received.

Below are two concrete implementations of pipelining, assuming a window size of N.

One is Go-Back-N:

- The sender sends all the packets in the window, and moves the window forward when ACKs arrive. If the earliest packet in the window times out without an ACK, it resends every packet in the window that has already been sent.
- The receiver accepts packets in order and returns an ACK carrying the highest sequence number received correctly in order so far
- The sender's window may lag the receiver's window by N, so the design must make sure sequence numbers don't repeat in a way that causes confusion. Whether the receiver buffers or discards out-of-order packets leads to different conclusions about the required size of the sequence number space.

The other is Selective Repeat:

- The receiver ACKs every packet individually, and when a single packet times out, the sender retransmits just that packet.
- Likewise, because the sender's window may lag the receiver's window by N, these 2N sequence numbers must all be representable as distinct sequence numbers.

### TCP

- Basic properties of TCP
  - Point-to-point
  - Reliable and in-order
  - A single connection can carry data in both directions
  - Cumulative ACKs
  - Connection-oriented
  - Flow controlled
- TCP segment

{{< figure src="/posts/hpc/computer-network/network-basics/chapter-3/tcp-segment.png" alt="TCP segment format" width="385" >}}

- TCP treats the data the application hands it as one continuous byte stream, not as separate messages. Data written by multiple `send()` calls is treated as successive parts of the same byte stream. TCP then cuts this stream into segments as it sees fit, and the boundaries of the original `send()` calls are not preserved.
- Definition of the sequence number: the sequence number of a TCP segment is the number, within the byte stream, of the first byte of data carried by that segment. All numbers are taken modulo 2³².
- Acknowledgement number: when the receiver sends an ACK, it indicates the number of the next **byte** the receiver expects to receive.
- How TCP achieves bidirectional (full-duplex) transfer: a segment can carry an acknowledgement for data in the opposite direction while also sending data (piggybacking)
- Setting the timeout
  - The round-trip time (RTT) of a packet must be estimated
  - EstimatedRTT = (1-a)EstimatedRTT + a(SampleRTT)
  - The actual timeout is EstimatedRTT plus a safety margin of 4DevRTT, which is estimated as follows.
  - DevRTT = (1-b)DevRTT + b|SampleRTT-EstimatedRTT|
- TCP fast retransmit
  - If the sender receives the same ACK 4 times in a row (3 duplicate ACKs), it concludes that a packet was lost: the receiver received packets with higher sequence numbers but is still ACKing the earlier one. So the sender retransmits that particular packet before the timeout.
- TCP flow control
  - Definition of the receive window (rwnd): how many free bytes are currently left in the receiver's receive buffer, i.e. how many more bytes it is willing to accept right now.
  - The sender keeps the amount of unACKed data within rwnd.
- TCP: connection-oriented
  - Two-way handshake
    - The client sends a message saying it wants to open a connection, the server replies that it accepts, and the server sets aside a socket resource.
    - The problem is that if a client's connection request is delayed and only arrives after the connection has ended, the server will wrongly set aside connection resources, and may wrongly act on other messages that belonged to an earlier, already accepted connection.
  - Three-way handshake
    - Here Seq is a random number generated each time based on the time and other inputs. After receiving the other side's confirmation, each side checks whether it echoes the random number it sent for this connection; only if it does is the other side really trying to connect this time. This seq is also the sequence number in the segment header.

![TCP three-way handshake](tcp-3way-handshake.png)

- Closing a TCP connection
  - When sending a TCP segment, setting the FIN bit to 1 means "I will no longer send anything to you in this direction." It also needs to be ACKed. Once both directions have notified each other with a FIN, the connection is closed.

### Congestion

- Congestion control is different from flow control
  - The former is about routers getting congested when faced with packets from many different sources
  - The latter is about the receiver not being able to keep up with how much data the sender sends; it can't process it fast enough.
- Causes of congestion
  - Suppose the router's buffer is infinite, its maximum input bandwidth is R, and its maximum output bandwidth is also R. If the sending machine sends at a bandwidth of R, then because the network is not perfectly steady, the rate is sometimes above R and sometimes below it, which can cause the router's buffer to build up and delay to grow.
  - Lost packets can be retransmitted, but these retransmitted packets consume link bandwidth while ultimately counting as only one packet in the throughput, so they reduce overall throughput.
  - Sometimes a path from source to destination crosses several routers. Suppose there are routers A and B; if one flow's retransmissions fill up A's bandwidth, another path that uses router A as its second hop may be unable to transmit.
- Congestion control
  - The sender observes delay or loss and slows down
  - The router explicitly tells the sender there is congestion, so it slows down.

### TCP Congestion Control

| Window                 | Decided by                                                                              | Prevents                                     |
| ---------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------- |
| Receive window rwnd    | The receiver, based on how much buffer space it has left, told to the sender via ACKs  | Overwhelming the **receiver** (flow control) |
| Congestion window cwnd | The sender, estimated by itself based on whether there is loss                          | Clogging the **network** (congestion control) |

The window the sender can actually use is the smaller of the two:

**Effective window = min(cwnd, rwnd)**

**Sending rate ≈ window size / RTT**

From here on rwnd is ignored and only cwnd is considered

- AIMD (additive increase, multiplicative decrease): every RTT the sender increases the congestion window by a fixed amount until a loss occurs; on a loss or timeout it multiplies the window by a factor (multiplicative decrease, e.g. halving) or resets it straight to 1 MSS, depending on the implementation.
  - On an ordinary link AIMD produces a sawtooth, but on ordinary links the router buffers and the superposition of many flows hide this overhead

- Slow start
  - Initially cwnd is 1 MSS (maximum segment size)
  - Then cwnd doubles every RTT
- Slow start makes cwnd grow until it reaches half of the cwnd at the last loss, after which growth switches to linear
- TCP CUBIC
  - Grows cwnd following a cubic function instead of a linear one, getting closer to the critical loss point faster. CUBIC grows fast when it is far from the window at the last loss, slows down as it approaches it, and speeds up again to probe once it has passed it.

![cwnd of TCP CUBIC vs. classic TCP](tcp-cubic.png)

- Delay-based congestion control
  - If the RTT is much longer than without congestion, it is taken as congestion, and the sender slows down.
- Explicit Congestion Notification (ECN)
  - When a router experiences congestion, it marks ECN in the IP header
  - When the receiver returns an ACK, it sets ECE in the segment to signal congestion
  - Finally the sender adjusts its window size
- Fairness
  - Two TCP connections share the same path; assume they have the same RTT, a fixed number of connections, and are in congestion avoidance (cwnd growing linearly, not slow start). The two connections may start with different cwnd, but since AIMD halves cwnd after congestion, the one with the larger cwnd drops more, while afterwards, without congestion, both grow at the same rate. So eventually the two connections converge to roughly the same rate.
  - TCP traffic can indeed be crowded out by UDP
  - Opening several TCP connections at once can indeed effectively increase transfer bandwidth

### QUIC

![QUIC vs. TCP+TLS handshakes](quic-handshake.png)

- In the figure above, TCP's third handshake message and TLS's first handshake message are the same arrow.
- QUIC's handshake already takes care of security and encryption, so it doesn't need an extra handshake for encryption the way TLS does.
- TCP only knows one ordered byte stream; it doesn't know which object the data in the stream belongs to, and can only hand it to the application strictly in byte order. For example, suppose segments of objects A, B and C are interleaved on the same TCP connection (as in HTTP/2): `[A-1][B-1][C-1][A-2][B-2]`. If `C-1` is lost, the `A-2` and `B-2` that have already arrived must also wait in the receive buffer until `C-1` is successfully retransmitted before they can be delivered to the application in order. Loss in one object holds up all objects; this is head-of-line blocking (HOL blocking).
- QUIC can also run multiple independent streams within one connection. They can transfer data in parallel and each handles its own reliable delivery: if a packet is lost, that stream takes care of it itself, but congestion control is shared. It has no HOL blocking, because as long as a stream's data is contiguous within that stream, it can be delivered to the application without caring whether other streams have gaps.

---

*This post was translated from the Chinese original by Claude.*
