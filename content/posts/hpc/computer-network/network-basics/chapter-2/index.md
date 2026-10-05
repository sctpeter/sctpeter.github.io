---
date: '2026-10-05T02:15:00-07:00'
draft: false
categories:
  - 网络
title: '计算机网络学习笔记：第二章'
---

参考资料： https://gaia.cs.umass.edu/kurose_ross/videos/2/

### HTTP

- hypertext transfer protocol

- HTTP/1.1 和 HTTP/2 使用 TCP

  - 客户端建立连接，服务端接受连接
  - 交换消息
  - 关闭连接

- 是无状态的，不会记录过去的请求

- 分类

  - 非持久化的: 每次请求一个object就需要重新建立一次TCP连接

  {{< figure src="/posts/hpc/computer-network/network-basics/chapter-2/http-non-persistent.png" alt="非持久化 HTTP" width="240" >}}

  - 持久化： 请求多个object可以复用相同的TCP连接

HOL blocking 是 Head-of-Line Blocking，队头阻塞。意思是：排在队伍最前面的那个数据没处理完，导致后面的数据即使已经准备好了，也必须一起等。http/2和http/3都解决了HOL blocking的问题。

{{< figure src="/posts/hpc/computer-network/network-basics/chapter-2/http-hol-blocking.png" alt="HTTP/2 缓解队头阻塞" width="520" >}}

HTTP/1.1 HTTP/2   假设用TLS :  HTTP → TLS → TCP → IP
HTTP/3   HTTP/3 → QUIC → UDP → IP

http/3相比于http/2的改进

**HTTP/2 跑在一条 TCP 连接上。** TCP 只认识一个有序的字节流，它不知道上面有多个 HTTP 流。如果某个 TCP 报文段丢失，内核会把它之后收到的所有数据都扣在缓冲区里，等丢失的报文段重传到达后才按顺序交给应用。结果是：丢的哪怕只是流 A 的一个帧，流 B、C 的数据即使已经到了，也会被一起卡住。

**HTTP/3 跑在 QUIC 上，QUIC 跑在 UDP 上。** QUIC 在传输层就知道有多个独立的流，并为每个流分别做有序交付和丢包重传。流 A 丢了一个包，只有流 A 需要等重传，流 B、C 的数据可以直接交给应用。

### dns

![DNS 层级结构](dns-hierarchy.png)

**递归查询（recursive query）**

提问方说："请把最终答案给我。"被问的服务器必须承担起全部查询工作，自己去问其他服务器，最后只返回最终结果（IP 地址，或"该域名不存在"的错误）。

**迭代查询（iterated query）**

提问方说："你知道多少就告诉我多少。"被问的服务器只根据自己掌握的信息回答：如果知道答案就直接给出；如果不知道，就返回一个指引（referral），即"去问某某服务器，它的地址是……"。之后由提问方自己去问下一个服务器。

**典型的 DNS 解析中两者同时存在**

| 查询的双方                        | 类型 | 原因                                          |
| --------------------------------- | ---- | --------------------------------------------- |
| 主机 → local DNS                  | 递归 | 主机只想要结果，不想自己跑多趟                |
| local DNS → 根 / TLD / 权威服务器 | 迭代 | 上层服务器只给指引，由 local DNS 自己逐级去问 |

dns记录类型

- A: 域名的ipv4
- AAAA: 域名的ipv6
- NS: 域名的权威name server（主域名注册时，上一级域（如 `.com` TLD）的服务器会存储它的 NS 记录；NS 记录的值是一个域名而不是 IP，当这个域名位于该主域名之内时，TLD 服务器还需额外存储它的 A 记录，以避免循环依赖）
- CNAME: 查找别名域名对应的原始域名
- 其他略

### Socket programming

数据报(datagram)这个名称可能是以下两种情况之一

- 它可以是网络层的IP数据报(IP datagram)——对应网络层
  - 结构：IP 头 + 数据部分（数据部分装着一个 TCP 段或 UDP 数据报，未分片时）。
  - 由谁生成：内核的网络层，在传输层的单元前加上 IP 头得到。
  - 在这个课程里面，网络层不讲IP包，讲数据报(datagram)
- 它也可以是UDP数据报(UDP datagram)——对应传输层
  - 结构：UDP 头 + 应用的消息。
  - UDP头中没有IP，只有源端口与目的端口
  - 由谁生成：应用通过 `sendto()` 提供消息和目的地址，内核在传输层加上 UDP 头得到

**发送时的封装顺序**

| 步骤 | 在哪一层 | 做了什么                                                | 得到的单元   |
| ---- | -------- | ------------------------------------------------------- | ------------ |
| 1    | 应用层   | 应用调用 `send()` 或 `sendto()`，把数据写进 socket      | **message**  |
| 2    | 传输层   | 内核给 message 加上 TCP 头或 UDP 头（端口号、校验和等） | **segment**  |
| 3    | 网络层   | 内核给 segment 加上 IP 头（源 IP、目的 IP 等）          | **datagram** |
| 4    | 链路层   | 网卡驱动给 datagram 加上以太网帧头和帧尾                | **frame**    |

tcp

- 传输层协议
- 可靠传输，保证传输顺序，流量控制，拥塞控制，不保证安全
- 传输文件FTP, e-mail的SMTP
- 由系统内核实现
- 依靠TLS 实现安全 TLS transport layer security  应用 → TLS → TCP → IP。TLS运行在用户态，但是一般是调用库而不是用户自己实现。

udp

- 传输层协议
- 没有握手
- 不保证可靠传输，不保证顺序
- 由系统内核实现

socket套接字

- Socket 是操作系统提供给应用进程的一个通信端点。应用进程通过它把数据交给传输层（TCP/UDP），也通过它从传输层取回数据。从网络分层的角度看，socket 是应用层与传输层之间的接口：在它之上的部分由应用程序员控制，在它之下的部分由操作系统内核控制。
- **UDP socket**：由（本地 IP 地址，本地端口号）标识。所有发往这个二元组的数据报，无论来自哪个发送方，都会交给同一个 socket。socket是操作系统的抽象，可以和进程绑定，不一定对应原始哪个层。源端口在传输层写入 UDP 头，源 IP 在网络层写入 IP 头。
- **TCP socket**：一条已建立的连接由四元组（源 IP，源端口，目的 IP，目的端口）标识。因此服务器在同一个端口（如 80）上可以同时拥有多个连接 socket，每个对应一个不同的客户端。
  - 监听TCP socket服务器仅仅建一个，当每一个客户端的连接请求被接受的时候，服务器就新建一个TCP socket
- TCP 和 UDP 是由操作系统内核实现的，应用通过系统调用（`socket()`、`send()` 等）使用它们
