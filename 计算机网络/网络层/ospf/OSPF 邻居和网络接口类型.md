# OSPF 邻居和网络接口类型

## 一、OSPF 邻居关系的建立和握手过程

OSPF ( Open Shortest Path First ) 是由 IETF 开发的链路状态路由协议，广泛应用于各种网络环境中。OSPF 使用 Dijkstra 算法计算最短路径，收敛速度快，具有层次化的多区域设计，常用于大型园区网、企业网或城域网。当前版本为 v2，主要标准为 RFC1583 和 RFC2328。OSPFv3 主要用于 IPv6，参考标准为 RFC5340。

### 1.邻居发现

OSPF 通过 Hello 报文发现并维护邻居关系。邻居关系不同于邻接关系，只有处于 2-Way 状态的路由器才能建立双向邻居关系。

OSPF 通过 Hello 协议发现、维护邻居关系，并验证双方是否具备双向通信能力。严格来说，路由器仅仅收到对端的 Hello 报文，只能说明发现了对方；**<font color="red">只有当本路由器在对端 Hello 报文的邻居列表中看到了自己的 Router ID，才说明双向通信已经建立，邻居状态进入 2-Way</font>**。与此同时，Hello 协议在广播网络和 NBMA 网络上还承担 DR/BDR 选举职责。

在 OSPF 中，邻居关系（neighbor relationship）与邻接关系（adjacency）必须严格区分。邻居关系解决的是双方是否通过 Hello 相互发现并确认双向可达的问题；邻接关系则是在邻居的基础上进一步建立的数据库同步关系，其目的是交换路由信息并同步链路状态数据库（LSDB）。因此，并不是所有邻居都会成为邻接，RFC 2328 明确规定：**_Not every two neighboring routers will become adjacency_**，并不是所有的邻居关系都会变成邻接关系。

从状态机角度看，Init 状态表示已经收到了邻居的 Hello，但尚未确认双向通信；2-Way 状态表示双向通信已经建立。后续需要进一步建立邻接时，**<font color="red">状态才会从 2-Way 继续进入 `ExStart`、`Exchange`、`Loading`，最终达到 Full。在这个过程中，双方先通过 Database Description（DBD）报文交换各自数据库的摘要信息，再通过 Link State Request（LSR）补齐缺失的 LSA，此时双方的数据库才被视为已经同步，路由器才算真正完全邻接（adjacent）</font>**。如果没有建立邻接的必要，邻居状态就会停留在 2-Way，而不会继续进入 Full。

是否需要从 2-Way 继续建立邻接，取决于网络类型以及 DR/BDR 角色。在点到点网络、P2MP 网络和虚链路上，只要邻居可达，通常都会进一步建立邻接；而在广播网络和 NBMA 网络上，并不是所有邻居两两之间都要建立邻接，通常只有与 DR、BDR 之间才建立邻接，其他普通路由器之间往往只保持 2-Way。

所有启用 OSPF 的接口都会发送 Hello 报文，发送间隔和目的地址根据网络类型不同而不同。

- 在广播和点到点网络中: Hello 报文每 10 秒发送一次。
- 在 NBMA 和 P2MP 网络中: Hello 报文每 30 秒发送一次。

在广播、点到点和点到多点网络中，OSPF 使用组播（**`224.0.0.5`** — 所有 OSPF 路由器）自动发现邻居，在 NBMA 网络中，邻居必须手动配置。

>大多数现代网络使用以太网，默认 OSPF 网络类型为 Broadcast。**`PPP/HDLC/FrameRelay`** 点到点链路被视为 OSPF P2P；非广播 FrameRelay（如物理或多点子接口）、X.25、ATM 可配置为 OSPF NBMA 或 P2MP。

### 2.邻居建立条件

路由器必须在 Hello 报文中同意某些参数才能建立邻居关系。参数包括：

- **`Hello/Dead Interval`**: 时间一致性要求。如果 Hello 间隔为 10s，Dead 间隔默认是 4×该值（即 40s）。
- **`Area ID`**: 邻居必须位于同一区域。Area ID 出现在所有 OSPF 报文头部，而不仅仅是 Hello 报文。
- **`Area Type`**: 必须匹配。由 Hello 报文中的 Option 字段确定；**`bits E`** 和 **`N/P`** 有不同含义取决于其位置，具体如下所示：

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 1 OSPF bits E 和 N/P 置位含义</div>
    <img src="ospf_static//110.png" width="650"/>
</div>

>Stub 和 Totally Stub 区域类型具有相同的区域位置；同样，NSSA 和 Totally NSSA 具有相同的区域位置。

- 认证类型和密钥必须匹配: 只有当认证通过时，才能建立 OSPF 邻居关系；
- **`Router ID`** 冲突: **<font color="red">OSPF Router ID 可以手动配置或由系统自动选择；优先选择 Loopback 接口</font>**。当直连路由器时，其 Router ID 不能相同；

>说明：Router ID 和 Area ID 认证信息出现在 OSPF 头部；Hello 报文间隔、Dead 间隔和网络掩码出现在 Hello 报文中。

影响 OSPF 邻居关系建立的原因包括：Hello/Dead 间隔不一致、直连路由器 Router ID 冲突、认证失败、区域类型不匹配或 Area ID 不一致。

### 3.邻居关系建立过程

OSPF 邻居的建立过程：

- 三步握手：即 OSPF 用三次 Hello 完成双方双向确认的报文过程；
- **`Down`**、**`Init`**、**`Two-way`** 状态；

邻居之间发送 hello 报文的状态机变化如下所示：

```c{.line-numbers}
                                   +----+
                                   |Down|
                                   +----+
                                     |\
                                     | \Start
                                     |  \      +-------+
                             Hello   |   +---->|Attempt|
                            Received |         +-------+
                                     |             |
                             +----+<-+             |HelloReceived
                             |Init|<---------------+
                             +----+<--------+
                                |           |
                                |2-Way      |1-Way
                                |Received   |Received
                                |           |
              +-------+         |        +-----+
              |ExStart|<--------+------->|2-Way|
              +-------+                  +-----+
```

上图中 **`2-Way Received`** 代表收到含自己 Router ID 的 Hello 报文，根据 RFC 文档，**<font color="red">Bidirectional communication has been realized between the two neighboring routers. This is indicated by the router seeing itself in the neighbor's Hello packet</font>**；**`1-Way Received`** 代表收到不含自己 Router ID 的 Hello 报文，根据 RFC 文档，**<font color="red">An Hello packet has been received from the neighbor, in which the router is not mentioned.  This indicates that communication with the neighbor is not bidirectional.</font>**。

场景：A—B 两台路由器，初始化邻居关系建立过程，A 接收 B 的 Hello 报文过程中，状态变化过程如下。

<div align="center">
    <img src="ospf_static//111.png" width="450"/>
</div>

- **`Down`** 状态：邻居的初始状态。但初始时，邻居的 **`Router ID 2.2.2.2`** 还没有出现在 OSPF 邻居列表里；
- **`Init`** 状态：当路由器 A 收到邻居的 Hello 报文，且该 Hello 报文中的 **`Active Neighbor`** 字域中没有包含 A 自己的 **`Router ID (1.1.1.1)`** 时，A 为该邻居设置的状态为 Init；
- **`2-Way`** 状态：当路由器 A 再次收到同一邻居的 Hello 报文，且该 Hello 报文中 **`Active Neighbor`** 字域中包含了 A 自己的 **`Router ID (1.1.1.1)`** 时，A 为该邻居设置的状态为 2-Way。只有当 A、B 双方邻居都进入 2-Way 状态时，才表明 A、B 之间建立了双向邻居关系；

>邻居表中邻居处于 Init 状态仅代表单向邻居建立起来，**`2-Way`** 代表双向邻居已建立起来。

### 4.邻接关系建立过程

**<font color="red">OSPF 路由器在双向邻居关系建立完成后，开始建立邻接关系</font>**。在广播和非广播（NBMA）网络中，邻接关系发生在 DR 和 BDR 选举之后，在其他网络类型中没有 DR/BDR 选举过程，邻居关系建立完成后即开始建立邻接关系。邻接建立过程如下图所示。

根据 RFC 的文档，Adjacencies are established with some subset of the router's neighbors. **<font color="red">Routers connected by point-to-point networks,Point-to-MultiPoint networks and virtual links always become adjacent</font>**. On broadcast and NBMA networks, all routers become adjacent to both the Designated Router and the Backup Designated Router.

对于点对点、点对多点以及虚链路网络类型，邻居关系建立完成后，路由器之间会自动建立邻接关系；对于广播和 NBMA 网络类型，因为这两类网络是多接入网络，一条二层网络上可能挂很多台路由器。如果所有路由器两两都建 Full 邻接，控制平面开销会迅速膨胀。**RFC 2328 规定，在广播和 NBMA 网络上，所有路由器都与 DR、BDR 建立邻接，而不是彼此全部建邻接**。

```c{.line-numbers}
                                  +-------+
                                  |ExStart|
                                  +-------+
                                    |
                     NegotiationDone|
                                    +->+--------+
                                       |Exchange|
                                    +--+--------+
                                    |
                            Exchange|
                              Done  |
                    +----+          |      +-------+
                    |Full|<---------+----->|Loading|
                    +----+<-+              +-------+
                            |  LoadingDone     |
                            +------------------+
```

邻接关系是指邻居路由器之间完成 LSDB 同步的过程，也称为 LSA 泛洪过程。同步完成后，双方邻居进入 FULL 状态。**<font color="red">在广播或非广播（NBMA）网络中，DRother 路由器之间保持 2-Way 状态，而与 DR/BDR 建立 FULL 邻接关系。在其他网络类型中，邻居关系建立为 2-Way 状态后直接开始建立邻接关系，无需 DR/BDR 选举过程</font>**。

邻接关系的状态迁移过程如下所示：

#### 4.1 信息交换初始状态（ExStart）

**<font color="red">首先是信息交换初始状态（ExStart）</font>**：在这一状态下，双方首先交换空的 DD 报文，用于确定：
  - Master/Slave 关系；
  - 初始 DD 的初始序列号；
  - 比较接口 MTU（可选）；

在 ExStart 状态下，路由器互相发送的空 DD 报文中置 I（Initialize），M（More）及 MS（Master/Slave）位。

- I 位：初始化位，仅头两份 DD 报文中置该位，代表同步过程开始；
- M 位：如果 M 位=0，则代表后续 DD 报文中没有 LSA Summary 要传。**<font color="red">任何一方 M 位不为 0，Master 就要继续发送 DD 报文，Slave 收到之后，不论是否还有 LSA Summary 要传递，一定要回应 DD 报文</font>**；
- M/S：初始双方均认定自己是 Master，所以 M/S 均置位。双方收到对方的 DD，**Router ID 高的一方为 Master，其后续 DD 报文中，M/S 会一直置位**。Master 会一直发送 DD 报文，Slave 回应 DD 报文，Slave 回应的 DD 报文是对 Master 发送的 DD 报文的确认。此过程持续到双方的 LSA 头都交换完成。

下面就是空的 DD 报文，I 位、M 位及 M/S 位均置位。图中不协商 MTU，所以 MTU 默认为 0；

```c{.line-numbers}
Open Shortest Path First
    OSPF Header
    OSPF DB Description
        Interface MTU: 0
        Options: 0x02 (E)
            0... .... = DN: Not set
            .0.. .... = O: Not set
            ..0. .... = DC: Demand Circuits are NOT supported
            ...0 .... = L: The packet does NOT contain LLS data block
            .... 0... = NP: NSSA is NOT supported
            .... .0.. = MC: NOT Multicast Capable
            .... ..1. = E: External Routing Capability
            .... ...0 = MT: NO Multi-Topology Routing
        DB Description: 0x07 (I, M, MS)
            .... 0... = R: OOBResync bit is NOT set
            .... .1.. = I: Init bit is SET
            .... ..1. = M: More bit is SET
            .... ...1 = MS: Master/Slave bit is SET
        DD Sequence: 510
```

#### 4.2 信息交换状态（Exchange）

选举出 Master 后，**<font color="red">Slave 路由器向 Master 回送 DD 报文，其中包含 LSDB 中的 LSA 头（LSA summary，LSA 摘要）列表，并使用 Master 的序列号。Master 也把自己的 LSA 头列表用 DD 发送，序列号增加 1，同时，Slave 收到后，会回应相同序列号的 DD 报文</font>**。任何一侧只要还有未传递完的 LSA 头，Master 就一定要产生 DD 报文并由 Slave 回应。Exchange 阶段通过这种可靠的 DD 交互，完成快速交换 LSA 头。

Master 和 Slave 的角色分工不同，Master 是由 RID 高的路由器充当，负责发送序列号递增的 DD 报文，如果 Master 没能收到回应，则 Master 间隔 5s 重传该 DD 报文，直至收到 Slave 的 DD 报文。

下面举例说明 R1 和 R2 之间的 exstart 和 exchange 过程。假设 R1 的 Router ID 相对 R2 较小，并且 R1 最终成为 slave，R2 最终成为 master。R1 和 R2 之间的 DD 交换过程遵守前面所述的 4 条规则：

- ExStart 阶段双方发送的是空 DD 报文，并且 **`I=1、M=1、MS=1`**，目的是协商主从，而不是正式交换 LSA Summary，根据 RFC 文档，**<font color="red">This is the first step in creating an adjacency between the two neighboring routers.  The goal of this step is to decide which router is the master, and to decide upon the initial DD sequence number.</font>**；
- 主从关系一旦确定，后续整个 Exchange 过程以 Master 选定的 DD sequence number 为唯一基准；未当选一方在 ExStart 中自行提出的序列号不再生效。
- 在 Exchange 状态下，Master 负责推进 DD 序列并驱动交换节奏，Slave 负责按 Master 的节奏进行确认性应答，即用相同的序列号回包；
- DD 交换的结束条件是双向收尾完成，而不是某一方已经没有更多摘要可发，根据 RFC 文档，**<font color="red">The Database Exchange Process is over when a router has received and sent Database Description Packets with the M-bit off.</font>**。

**（1）Master 和 Slave 都还有多份摘要要发送**

```c{.line-numbers}
// R1 (RID较小，最终成为 Slave)，R2 (RID较大，最终成为 Master)

ExStart
D-D (Seq=x, I=1, M=1, MS=1, empty)  ----------------------->
<-----------------------  D-D (Seq=y, I=1, M=1, MS=1, empty)


Exchange
D-D (Seq=y, I=0, M=1, MS=0)  ------------->
<------------  D-D (Seq=y+1, I=0, M=1, MS=1)

D-D (Seq=y+1, I=0, M=1, MS=0)  ----------->
<------------  D-D (Seq=y+2, I=0, M=1, MS=1)

D-D (Seq=y+2, I=0, M=1, MS=1) ------------>
<------------ D-D (Seq=y+3, I=0, M=1, MS=1)

D-D (Seq=y+3, I=0, M=1, MS=0, last) ------->

......

<------------ D-D (Seq=y+n, I=0, M=1, MS=1)
D-D (Seq=y+n, I=0, M=1, MS=0) ------------->
```

**（2）Master 还有多份摘要要发送，Slave 没有了**

```c{.line-numbers}
ExStart
RT1 -> RT2 : D-D (Seq=x, I=1,M=1,MS=1, empty)
RT1 <- RT2 : D-D (Seq=y, I=1,M=1,MS=1, empty)

Exchange
RT1 -> RT2 : D-D (Seq=y,   M=0, MS=0, S1-last)     // RT1 发送 LSA 摘要完毕 
RT1 <- RT2 : D-D (Seq=y+1, M=1, MS=1, M1)

RT1 -> RT2 : D-D (Seq=y+1, M=0, MS=0, empty/ack)
RT1 <- RT2 : D-D (Seq=y+2, M=1, MS=1, M2)

RT1 -> RT2 : D-D (Seq=y+2, M=0, MS=0, empty/ack)
RT1 <- RT2 : D-D (Seq=y+3, M=1, MS=1, M3)

RT1 -> RT2 : D-D (Seq=y+3, M=0, MS=0, empty/ack)
RT1 <- RT2 : D-D (Seq=y+4, M=0, MS=1, M4-last)

RT1 -> RT2 : D-D (Seq=y+4, M=0, MS=0, empty/ack)
```

这里从 **`Seq=y`** 开始，slave 自己其实已经没更多 LSA 摘要需要发送了，所以 slave 很早就把 **`M=0`** 置出来了。但交换并不能结束，因为 master 连续发来的 **`y+1/y+2/y+3`** 都还是 **`M=1`**，说明 Master 这边还没发完。所以 slave 还是必须继续回 **`y+1/y+2/y+3`**。因此 slave 回复 DD，不只是为了发自己的 LSA 摘要，还为了确认 master 的上一份 DD，并维持整个主从序列继续前进。

**（3）Master 没有了，Slave 还有多份摘要要发送**

```c{.line-numbers}
ExStart
RT1 -> RT2 : D-D (Seq=x, I=1,M=1,MS=1, empty)
RT1 <- RT2 : D-D (Seq=y, I=1,M=1,MS=1, empty)

Exchange
RT1 -> RT2 : D-D (Seq=y,   M=1, MS=0)
RT1 <- RT2 : D-D (Seq=y+1, M=0, MS=1, last)

RT1 -> RT2 : D-D (Seq=y+1, M=1, MS=0)
RT1 <- RT2 : D-D (Seq=y+2, M=0, MS=1, empty)

RT1 -> RT2 : D-D (Seq=y+2, M=1, MS=0, S3)
RT1 <- RT2 : D-D (Seq=y+3, M=0, MS=1, empty)

RT1 -> RT2 : D-D (Seq=y+3, M=0, MS=0, last)
```

虽然 Master 在 **`Seq=y+1`** 就已经把自己的摘要发完了，但只要它收到 Slave 的报文里 **`M=1`**，就说明 Slave 还没结束，因此 Master 仍要继续发新的 DD 来推进节奏，直到 Slave 也发出自己的最后一个 **`M=0`**。

根据 RFC 文档，Master 和 Slave 的行为如下所示：

- Master：Increments the DD sequence number in the neighbor data structure. If the router has already sent its entire sequence of Database Description Packets, and the just accepted packet has the more bit (M) set to 0, the neighbor event ExchangeDone is generated. Otherwise, it should send a new Database Description to the slave.
- Slave：Sets the DD sequence number in the neighbor data structure to the DD sequence number appearing in the received packet. The slave must send a Database Description Packet in reply. If the received packet has the more bit (M) set to 0, and the packet to be sent by the slave will also have the M-bit set to 0, the neighbor event ExchangeDone is generated.Note that the slave always generates this event before the master.

>ExchangeDone：Both routers have successfully transmitted a full sequence of Database Description packets. Each router now knows what parts of its link state database are out of date.

上述 RT1 和 RT2，都必须在本端确认双方完整的 DBD 交换已经结束之后，才会离开 Exchange；离开后如果还有 LSA 要请求，就进入 Loading，否则直接进入 Full。

**根据上述的讨论可知 Exchange 过程中，DD 交互过程是可靠的。Master 发送 **`seq=y+n`** 的报文，Slave 回应 **`seq=y+n`** 的报文**，Master 和 Slave 的 DD 报文中 M 都不置位，Exchange 过程才结束。

#### 4.3 加载状态（Loading）

在这一状态下，**<font color="red">本地路由器将会向它的邻居路由器发送链路状态请求数据包 LSReq，以请求本地 LSDB 中没有的 LSA</font>**。收到 LSReq 的报文，路由器会用包含完整的被请求的 LSA 的 LSU 做回应。请求方收到 LSU 后，如果无误，则 LSAck 确认该 LSU。一份 LSAck 可同时为多份 LSUpdate 做确认。

#### 4.4 完整状态（Full）

在 FULL 状态下，邻居路由器之间已完成同步过程，建立起完全邻接关系。