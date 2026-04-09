# OSPF 数据库同步及泛洪机制

## 一、OSPF 报文结构

### 1.报文类型

OSPF 共有 5 种类型的协议报文：

- Hello 报文：周期性发送，用来发现和维持 OSPF 邻居关系。
- DD 报文（DataBase Description Packet）：**<font color="red">描述了本地 LSDB 的摘要信息，用于两台路由器进行数据库同步</font>**。
- LSR 报文（Link State Request Packet）：向对方请求所需的 LSA，只有在双方成功开始交换 DD 报文后才会向对方发出 LSR 报文。
- LSU 报文（Link State Update Packet）：向对方发送其所需要的 LSA 或者泛洪自己更新的 LSA。
- LSAck 报文（Link State Acknowledgment Packet）：用来对收到的 LSA 进行确认。

OSPF 工作在 IP 层，是个可靠的协议，协议内部包含确认机制。OSPF 报文中不需要确认的报文有 Hello 和 LSAck 报文。

### 2.OSPF 报文头

### 3.Hello 报文

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 1 Hello 报文格式</div>
    <img src="ospf_static//115.png" width="550"/>
</div>

Hello本报文本身并不承载 LSA 正文，RFC 2328 对 Hello 的总定义是：**`establish and maintain neighbor relationships`**，并且在支持广播/组播的网络上还能动态发现邻居。Hello 报文的作用可以总结为以下 5 点：

#### 3.1 发现邻居

Hello 会周期性从接口发出，在支持广播或组播能力的网络上，Hello 报文还能让路由器动态发现同网段里的其他 OSPF 路由器。Hello 报文能动态发现邻居依赖于 IP 组播技术，RFC 2328 规定，AllSPFRouters 组播地址为 **`224.0.0.5`**，**所有运行 OSPF 的路由器都应能够接收发往该地址的报文，且 Hello 报文始终发送到该地址**。当在路由器的某个接口上激活 OSPF 时，该接口会自动监听 AllSPFRouters 组播地址。因此路由器 A 不需要提前知道路由器 B 的 IP 地址，只要向 AllSPFRouters 发送报文，路由器 B 就能从中提取出路由器 A 的源 IP 和 Router ID，从而动态发现了 A 的存在。

另外，Hello 报文只能发现同网段内的路由器，根据 RFC 2328 文档，发送到 OSPF 组播地址的报文不应被转发，它们的设计目标就是只在单跳范围内传播；为确保这一点，这类报文的 IP TTL 必须设置为 1。因此，OSPF 的组播 Hello 只能在本地链路内传播，也就是当前二层广播域或当前 VLAN 内。

**<font color="red">在广播网络和物理点到点网络上，Hello 报文每隔 HelloInterval 秒发送一次，目的地址是 IP 组播地址 AllSPFRouters</font>**。在虚链路上，Hello 报文以单播方式发送，也就是直接发给虚链路另一端的路由器，发送周期同样是 HelloInterval。在 **`Point-to-MultiPoint`** 网络上，每隔 HelloInterval 秒，会分别向每一个直连邻居发送独立的 Hello 报文。

#### 3.2 维持邻居关系（保活）

Hello 报文要持续发送、持续接收。Hello 报文中携带 **`RouterDeadInterval`**，它定义了本路由器多长时间收不到对方报文，就不再认为对方活跃，则认为邻居失效，并从邻居表里移除该邻居，从路由表里撤销指向其的路由。同时 Hello 报文中还包含该接口发送 Hello 报文的时间间隔，即 **`HelloInterval`**。

#### 3.3 参数校验

Hello 报文还承担参数一致性检查的职责。RFC 10.5 规定，收到 Hello 后必须检查 **`Network Mask`**、**`HelloInterval`** 和 **`RouterDeadInterval`** 是否与本接口配置一致；不一致就停止处理并丢弃。对于区域能力，还要检查 Options 字段中的 E-bit 是否与该区域的 **`ExternalRoutingCapability`** 匹配，如果区域是 Stub，则 E-bit 必须清零，否则必须置位，错了也要丢弃。

>E-bit 置位表示该区域（普通区域或者骨干区域）支持引入 Type 5 LSA 外部路由，E-bit 清零表示该区域（Stub 区域）不支持引入 Type 5 LSA 外部路由。

#### 3.4 建立并确认双向通信

Hello 报文里有一个邻居列表，表示最近收到过哪些路由器的 Hello 报文，如果本路由器出现在对方 Hello 的邻居列表中，就触发 **`2-WayReceived`**，否则触发 **`1-WayReceived`**，并停止继续处理。

系统会尝试将 Hello 报文的发送方与接收接口的某个邻居对应起来。如果接收接口连接的是广播网络、Point-to-MultiPoint 网络或 NBMA 网络，则通过 Hello 报文 IP 首部中的源 IP 地址来识别发送方。如果接收接口连接的是 P2P 或虚链路，则通过 Hello 报文 OSPF 首部中的 Router ID 来识别发送方。 

接口当前的邻居列表保存在接口的数据结构中。**<font color="red">如果找不到匹配的邻居（也就是说，这是第一次检测到这个邻居），就创建一个新的邻居并添加到邻居列表中。新创建的邻居，其初始状态设为 Down</font>**。

#### 3.5 支持 DR/BDR 选举。

Hello 报文之所以能参与 DR/BDR 选举，是因为它已经携带了选举算法所需的关键信息：Router Priority、发送者当前认为的 Designated Router、发送者当前认为的 Backup Designated Router，以及邻居列表。根据 RFC 文档，The Designated Router is elected by the Hello Protocol. 

在上图 1 中的 **`Designated Router`** 表示本网段上 DR 路由器的 **<font color="red">接口</font>** IP 地址，**`Backup DesignatedRouter`** 表示本网段上 BDR 路由器的 **<font color="red">接口</font>** IP 地址（DR 和 BDR 都是接口的概念）。**`Neighbor`** 表示邻居列表，用 Router ID 标识，记录当前路由器已知的链路上所有邻居的 RID。**`Rtr Pri`** 表示 DR 优先级，默认为 1，如果为 0，则路由器不能参与 DR 或 BDR 的选举。Network Mask 表示发送 Hello 报文的接口所在网络的掩码。

### 4.DD 报文

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 2 DD 报文格式</div>
    <img src="ospf_static//116.png" width="550"/>
</div>

DD 报文（Database Description，Type 2）的主要作用，并不是直接传送完整的链路状态信息，而是让两台路由器先对彼此的 LSDB 进行比对。DD 报文先告诉对端我这里有哪些 LSA、版本，从而让对端能够判断哪些 LSA 是一致的，哪些 LSA 缺失，哪些 LSA 版本较旧。因此 DD 报文只在邻接建立的 exstart 和 exchange 阶段使用，而真正缺失或需要更新的完整 LSA，则由后续的 LSR/LSU 机制来请求和传送。

在 exstart 阶段，双方发送的是空的 DD 报文，此时 DD 报文承担了协商功能，双方通过 DD 报文中的 I、M、MS 标志位来确定主从关系和初始序列号，后续再进入 exchange 过程。

在 exchange 阶段，DD 报文传递的并不是完整的 LSA，而是 LSDB 的摘要。RFC 2328 规定，LSDB 中的每一条 LSA 在 DD 报文中都是通过 LSA Header 的形式来描述的，而不是携带完整内容，并且只有当前一个 DD 报文被确认之后，才继续发送下一部分。当路由器收到并接受邻居发来的 DD 报文后，会逐条检查其中列出的 LSA Header，并与本地 LSDB 中对应的条目进行比较。如果发现本地没有该 LSA，或者本地已有副本但版本较旧，那么这条 LSA 就会被加入请求列表。后续，路由器再通过 LSR 报文向邻居请求这些具体的 LSA，而邻居则通过 LSU 报文返回完整内容。

DD 报文之所以能可靠地完成链路状态数据库（LSDB）摘要的同步，根本原因在于 OSPF 协议为其设计了严密的序列号控制与隐式确认机制。在 ExStart 阶段完成主从（Master/Slave）选举后，Exchange 阶段的数据交互便确立了严格的轮询模型：**<font color="red">交互始终由 Master 节点主动发起，Slave 节点在收到报文后，必须被动回复一个携带相同序列号的 DD 报文以完成隐式确认；确认无误后，Master 才会递增序列号并发送下一个摘要报文</font>**。当主从双方均完成本地 LSA 摘要（LSA Headers）的交互后，邻居状态机将平滑过渡至 Loading 阶段，进而按需触发 LSR 与 LSU 报文的交互，以完成最终完成  LSDB 的精确同步。

在上图 2 中，**`Interface MTU`** 表示在不分片的情况下，此接口最大可发出的 IP 报文长度。华为默认不填充接口实际 MTU 值，所以值为 0；**`I（Initialization）`** 位表示当发送连续多个 DD 报文时，如果这是第一个 DD 报文，则置为 1，否则置为 0；**`M（More）`** **<font color="red">位表示当发送连续多个 DD 报文时，如果后续的 DD 报文中不再有 LSA 头，则置为 0</font>**；**`M/S（Master/Slave）`** 位表示当两台 OSPF 路由器交换 DD 报文时，首先需要确定双方的主从关系，RouterID 大的一方会成为 Master，当值为 1 时表示发送方为 Master；**`DD Sequence Number`** 表示 DD 报文序列号；**`LSA Header`** 表示本端 LSDB 中 LSA 的头。

### 5.LSR 报文

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 3 LSR 报文格式</div>
    <img src="ospf_static//117.png" width="550"/>
</div>

LSR 报文不参与 LSA 的常规网络泛洪过程，而是专用于 OSPF 数据库同步阶段的按需请求机制。其核心作用是在路由器交互 DD 报文并完成 LSDB 摘要比对后，向邻居路由器精确请求本地缺失或版本滞后 LSA 的完整数据载荷，从而确保双方链路状态数据库的一致性。LSR 的请求对象为特定的 LSA 实例，根据 RFC 2328 规定，**<font color="red">LSR 报文严格通过 LSA 头部的核心三元组——LSA 类型（LS Type）、链路状态 ID（Link State ID）和通告路由器（Advertising Router）——来唯一标识目标 LSA</font>**。RFC 的原文是：The LSA header contains the LS type, Link State ID and Advertising Router fields.  The combination of these three fields uniquely identifies the LSA. 收到 LSR 的一方，需要按请求内容返回相应的 LSU。

**`LS type`** 表示 LSA 的类型号；**`Link State ID`** 表示根据 LSA 中的 LS Type 和 LSA Description 在路由域中描述一个 LSA；**`Advertising Router`** 表示产生此 LSA 的路由器的 Router ID。

### 6.LSU 报文

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 4 LSU 报文格式</div>
    <img src="ospf_static//118.png" width="550"/>
</div>

### 7.LSAck 报文

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 5 LSAck 报文格式</div>
    <img src="ospf_static//119.png" width="550"/>
</div>

## 二、LSA 格式及类型

## 三、泛洪机制

