---
date: '2026-10-29T23:00:00-07:00'
draft: false
categories:
  - 网络
title: '计算机网络学习笔记：第四章'
---

参考资料： https://gaia.cs.umass.edu/kurose_ross/videos/4/

本笔记参考 J.F. Kurose & K.W. Ross 的课程，课程内容版权归原作者所有(https://gaia.cs.umass.edu/kurose_ross/ppt.php)

### 分类

数据平面

- 是相对于单个路由器来说的，例如决定一个到达的IP数据报从路由器的哪个出接口出去。通常是用硬件实现的。

控制平面

- 相对于整体的网络来说的，数据如何传输等。通常是用软件实现的。

### 路由器

#### 路由器的结构

![路由器结构](router-architecture.png)

![输入端口的处理](input-port.png)

路由器转发

- destination forwarding
  - 目的 IP，用最长前缀匹配得到出接口，动作也只有"转发"一种。

  - 最长前缀匹配
  
    - 查找forwarding table的时候的匹配方式，匹配到多条时，前缀长的优先。
  
    - TCAM 是 Ternary Content-Addressable Memory，三态内容寻址存储器，是路由器用来极快地查转发表的一种硬件。
  
      - 普通的RAM是输入地址查找地址上的内容
  
      - CAM是输入地址上的内容，快速查找地址，它把输入同时与所有表项进行比较
  
      - Ternary是三态，意思是每一位可以是0、1或者X，这里X表示任意匹配。
  
- Generalized forwarding
  - 可以根据多个首部字段做匹配，并执行转发、丢弃、修改首部等多种动作。

#### switching fabric

- 确定了输出端口之后，packet才会被送到switching fabric中。
- switching fabric把包从输入端口传输到输出端口的最大可达速度(所有端口一起算)叫做switching rate。

  - non-blocking switch ： 包在输入端口不用排队，switching rate 足够大让所有包都不在输入端口排队堆积。
  - input port queuing: 与non-blocking switch相对应，输入端口需要排队。
- 分类
  - switching via memory
    - cpu控制，先拷贝到内存，再传到输出端口，会受到内存带宽速度限制。
  - switching via bus
    - 不经过内存，所以一个包只需要占用总线一次；使用共享总线传输，所以传输速率受到总线带宽制约。
  - switching via interconnection network
    - 方式一： 有些交叉开关（crossbar switch）通过 N² 个交叉连接点，把 N 个输入端口和 N 个输出端口直接相连。
    - 方式二：multistage switching networks。例如，下图中，有4个4 $\times$ 4口的交换结构, 还有4个2 $\times$ 2的交换结构。然后从一个输入Port到一个输出Port之间可以有多条路径可以并行。 


  {{< figure src="/posts/hpc/computer-network/network-basics/chapter-4/multistage-switch.png" alt="由小交换结构组成的 8×8 多级交换网络" width="281" >}}

- 骨干网络的交换设备可以有多个fabric plane并行。

#### 队列

##### 输入Port

- Datagram传入的速度超过switching fabric可以处理的速度造成队列等待。
- Head-of-the-Line(HOL) blocking: 多个输入端口的队首 datagram 争用同一个输出端口时，同一时刻只有一个能通过，其余的只能留在各自输入队列的队首等待；排在它们后面的 datagram 即使目的输出端口空闲，也会因 FIFO 顺序而无法被转发。

##### 输出Port队列

- switching传输的速度超过了output port可以输出的速度，packet在output port的buffer上会排队。此时会在包上设置congestion 的信号。

#### buffer

- buffer用于暂时存储因为拥塞而不能立即传输的包。
  - 缓冲区有两个作用：
    - 吸收突发：短时间内涌来一批包，输出链路发不过来，先存进缓冲区，不至于直接丢掉。
    - 让链路保持忙碌：突发过后，缓冲区里还存着包，链路可以继续发，不会闲着。没有缓冲，链路就会出现"一会儿丢包、一会儿空转"的情况，利用率下降。
- 太多buffer的坏处
  - 延迟很长，对延迟敏感的应用不友好
  - TCP拥塞控制会变迟钝
- 缓冲区从很大减小到一个临界值，可以降低延迟而不损失吞吐量；再继续减小，就开始用吞吐量换延迟了。
- buffer管理
  - tail drop:队列满了，最后新来的包丢弃。
  - Priority: 队列满的时候，也可以把最后新来的包留下，依据优先级去除前面的包

#### 路由器缓冲区优先级调度

- First in,First Serve
- Priority scheduling
  - 优先传输最高优先级且队列非空的类别中的数据包。
- Round Robin scheduling(轮询调度)
  1. 把到达的包按某种规则分类（比如按来源、按应用类型），每一类放进各自的队列
  2. 调度器按固定顺序轮流访问各个队列，每轮从每个队列发送一个完整的包
  3. 某个队列为空就直接跳过，轮到下一个队列
- weighted fair queuing
  - 泛化的轮询调度，每一类有特定的权重，不同的最小保证带宽。
- ISP决定优先级策略，例如可能依据端口判断服务性质，或者依据源IP(付费公司可能优先)，然后决定优先级。

### Ipv4 header

![IPv4 数据报格式](ipv4-datagram.png)

- ipv4的header是可变长度的，因此这里会有header len
- header checksum在ipv6中被删除了，因为ttl每变化一次，checksum会重算，带来开销

### DHCP

- 使用UDP,客户端使用68端口，服务端使用67端口
- 分配ip步骤
  - DHCP discover(optional)
    - source ip 0.0.0.0(客户端还没有ip), destination ip 255.255.255.255(广播ip地址)。
    - 还有一个transaction ID:用于确认接下来的DHCP offer是不是给自己的。
  - DHCP offer(optional)
    - 目的ip是255.255.255.255，携带上之前的transaction ID。
    - 在yiaddr field中，含有给那个请求DHCP服务的设备分配的ip
  - DHCP request
    - 主机有自己想要的ip，可能是之前DHCP offer给的，也可能是自己之前用过的ip。然后向DHCP服务器请求使用这个ip.
  - DHCP ACK
- 除了分配ip外，DHCP还可以，给出第一跳路由器的ip，给出dns服务器的ip，还有给出子网掩码。

### 公网路由

- IP在公网上路由的时候，也有最长前缀匹配，如下图。

![公网路由中的最长前缀匹配](longest-prefix-routing.png)

### Ipv6

- 相比ipv4，更难耗尽

- 固定长度的header，处理起来更快
- Flow label:用来标记这个包属于哪个流。一个典型用途：当到达目的地有多条等价路径时，路由器会把不同的流分散到不同路径上做负载均衡，但同一个流的包要走同一条路径，否则包可能乱序到达。但是具体如何用应该是路由器决定的。

![IPv6 数据报格式](ipv6-datagram.png)

### tunneling

- 一些古老的设备可能不支持ipv6
- tunneling让Ipv6的datagram成为ipv4的payload，从而让被ipv4包裹的ipv6 packet可以在ipv4路由器上路由，最终在一个同时支持ipv4与ipv6的路由器上转化回原本的ipv6  packet然后到达终点。

### Generalized forwarding

- 可以根据多个首部字段做匹配，并执行转发、丢弃、修改首部等多种动作。
