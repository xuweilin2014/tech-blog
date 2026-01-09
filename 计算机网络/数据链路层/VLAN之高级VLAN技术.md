# VLAN 高级技术

## 一、MUX VLAN

在企业建立企业的数据网络时，我们可能经常会遇到各种流量隔离的需求。网络中有时候需要对不同用户进行网络隔离，以增强网络的安全。比如服务器商的 IDC，针对不同的客户托管的服务器需要进行网络隔离，传统的做法是为每个客户分配一个 VLAN。但这样做有很多缺点，比如交换机的 VLAN 数量会受到挑战（华为交换机最多支持 4094 个 VLAN），VLAN 越多，配置和管理也越复杂。

例如下图所示，该企业通过一台二层交换机连接着三个用户群体，分别是部门 A 的用户、部门 B 的用户以及访客，除此之外交换机上还连接着公共服务器（Server）。现在企业的需求是：

- A 部门内的用户之间能够进行二层通信，B 部门同理。但是 A、B 部门的用户之间不能进行二层通信。这个需求可以通过将 A、B 部门划到两个不同的 VLAN 来实现。
- 要求部门 A 以及部门 B 的用户都能访问公共服务器 Server。
- 要求访客区的任意一台 PC 除了能访问 Server 外，不能访问任何其他设备，包括其他访客 PC。

<div align="center">
    <img src="VLAN_static//33.png" width="480"/>
</div>

这是需面对的第一个疑难点。因为部门 A 及部门 B 如果被规划到了不同的 VLAN 中，而这两个 VLAN 现在都要访问 Server，那么 Server 究竟该放置在哪一个 VLAN 呢？如果将 Server 放置在与部门 A 相同的 VLAN 中，那么部门 B 的用户将无法访问到 Server，此时就不得不借助三层设备，例如由路由器来实现 VLAN 间的通信，这就增加了经济成本。

MUX VLAN (Multiplex VLAN) 可以实现上述需求，先介绍 MUX VLAN 的几个基本概念。

- 主 VLAN (Principal VLAN)：加入 Principal VLAN 的接口也即 Principal Port，它们可以和 MUX VLAN 内的所有接口进行通信；
- 从 VLAN (Subordinate VLAN)：Subordinate VLAN 分为两种，一种是互通型 Subordinate VLAN (Group VLAN)，另一种是隔离型 Subordinate VLAN(Separate VLAN)。**<font color="red">每个 Group VLAN 及 Separate VLAN 必须与一个 Principal VLAN 绑定</font>**；

加入 Separate VLAN 的接口也即 Separate Port，Separate Port 只能与 Principal Port 通信，**<font color="red">而无法与其他类型的接口通信，包括同属一个 Separate VLAN 的其他 Separate Port</font>**。

加入 Group VLAN 的接口也即 Group Port，Group Port 可以和 Principal Port 通信，属于同一个 Group VLAN 的用户之间能够进行二层通信，而属于不同 Group VLAN 的用户之间就无法通信了，另外 Group Port 也无法与 Separate Port 通信。

**<font color="red">一个主 VLAN 可关联多个从 VLAN，在关联的从 VLAN 中可以包含多个互通型 VLAN，但只能有一个隔离型 VLAN</font>**。

主 VLAN 和从 VLAN 都能拥有自己的端口，主 VLAN 中的端口称为 Principal Port（主端口），隔离型 VLAN 中的端口称为 Separate Port（隔离端口），互通型 VLAN 中的端口称为 Group Port（组端口）。**<font color="blue">要注意的是，这些端口只能拥有一个 VLAN，而且必须是 `Access` 或 `Hybrid untagged` 类型的，如果某 Trunk 端口或端口上有多个 VLAN，是不能启用 MUX VLAN 功能的</font>**。

>禁止接口 MAC 地址学习功能或限制接口 MAC 地址学习数量将会影响 MUX VLAN 功能的正常使用。
>可以为 Principal VLAN（主 VLAN）创建 VLANIF 接口，不能为 Group VLAN（互通型从 VLAN）和 Separate VLAN（隔离型从 VLAN）创建 VLANIF 接口。
>使能 MUX VLAN 功能的接口不能加入同一个 MUX VLAN 组中的其他 VLAN。
>Access 类型接口只能加入一个 MUX VLAN 组，Trunk、Hybrid 类型接口可以加入多个 MUX VLAN 组，最大可以加入 32 个 MUX VLAN 组。

回到最开始的案例，在该网络中部署 MUX VLAN 即可实现相关需求。如下图所示，交换机创建了四个 VLAN，它们分别是 100、101、102 以及 109，这四个 VLAN 分别给 Server、部门 A、部门 B 以及访客使用。

<div align="center">
    <img src="VLAN_static/34.png" width="480"/>
</div>

现在 VLAN100 被配置为 Principal VLAN，VLAN101 以及 VLAN102 被配置为 Principal VLAN 100 的 Group VLAN，如此一来，A 部门内的用户之间能够进行二层通信，B 部门同理，而这两个部门的用户之间则无法通信，同时由于 VLAN101 及 VLAN102 都是 VLAN100 的 Group VLAN，因此两个部门的用户都能与处于 VLAN100 的 Server 通信。接下来将 VLAN109 配置为 Principal VLAN 100 的 Separate VLAN，如此一来，属于 VLAN109 的访客只能与 Server 通信，而无法与其他任何接口通信，包括 VLAN109 中的其他访客。

## 二、MUX VLAN 实验配置

### 1.MUX VLAN 基础配置

### 2.MUX VLAN 使用上级路由器/三层接口作为网关

在传统的 VLAN 技术实现中，VLAN 用户通过 VLANIF 接口来实现和其他 VLAN 以及外网之间的三层通信，而华为 Sx7 系列交换在 v2v3 版本之后才新增 Mux VLAN 支持 VLANIF 接口的功能，之前的版本不支持。那么在旧版本交换机上如何让 MUX VLAN 用户能访问到其他网段或外网？这里提供两种网关设置方法。首先是使用上级路由器/三层接口作为网关。

如下图所示，VLAN2 和 VLAN3 作为 MUX VLAN 的两个二级 VLAN，使用相同的子网。这里的交换机只提供二层交换功能，上面的路由器充当三层交换和网关的功能。根据 MUX VLAN 中端口的访问规则，隔离型 VLAN 和互通型 VLAN 中的端口都可以访问主 VLAN 中的端口，所以只需要将交换机的上联端口加入 MUX VLAN 中的主 VLAN（VLAN 4）成为主端口后，两部门的用户都能访问到网关了。

<div align="center">
    <img src="VLAN_static/35.png" width="300"/>
</div>

上图可以理解为：用 MUX-VLAN 在接入交换机上把“用户之间的二层邻接”切断，同时又让 VLAN2 与 VLAN3 这两个二级 VLAN 采用同一个 IP 网段、同一个默认网关，通过主 VLAN（VLAN4）的主端口把所有“去网关”的流量汇聚到上级路由器。也就是说，A 部门（VLAN2）和 B 部门（VLAN3）在二层上被刻意分在不同的 VLAN、不同的端口角色里，从而达到隔离；但在三层地址规划上，它们仍然被当作“同一网段的一群终端”，都用同一个子网、同一个网关地址（例如都在 192.168.10.0/24，默认网关都是 192.168.10.1）。这种组合的目的通常是：既要隔离用户（防二层互访/互嗅/广播互扰），又要节省网段/网关数量（两个部门共享一套地址规划）。

在 VLAN 与端口角色上，这个方案的分工非常明确：VLAN4 是 Principal VLAN（主 VLAN），交换机连接上级路由器的上联口被定义为 principal port（主端口），也就是“允许所有二级 VLAN 去访问的出口端口”；而 VLAN2 是 Group VLAN，A 部门用户接入口作为 group port，通常允许同组内互通；VLAN3 是 Separate VLAN，B 部门用户接入口作为 separate port，通常相互隔离（哪怕同为 VLAN3 的 separate 端口也互相不通）。从二层视角看，这套规则要实现的是：用户彼此不要形成直接的二层邻接关系，但用户必须能访问一个共同的“上联/网关/服务器”出口，而这个出口就被集中放在主 VLAN（VLAN4）的主端口上。

关键机制就在“不同 VLAN 怎么还能共用同一个网关”这一步。正常的 VLAN 行为是：主机发 ARP 广播找网关 MAC，这个广播只能在本 VLAN 内扩散，根本到不了另一个 VLAN，更到不了 VLAN4 上的主端口。但 MUX-VLAN 的访问规则打破了这个限制：二级 VLAN（VLAN2/VLAN3）的端口被允许访问主 VLAN（VLAN4）的主端口，于是当 VLAN2 或 VLAN3 的主机为了出网而 ARP 询问 192.168.10.1 时，交换机并不会把这个“找网关的 ARP”只限制在 VLAN2 或 VLAN3 内部，而是会按照 MUX-VLAN 的规则把与“去往主端口/主 VLAN”相关的流量引导到 principal port 上，让路由器能够收到并回复。路由器回一个 ARP Reply（单播），交换机再把这个回复准确地送回发起请求的那台主机。之后主机学到网关 MAC，就把所有需要交给网关处理的流量都发往网关 MAC；交换机继续按 MUX-VLAN 规则把这些帧送到主端口，上级路由器再完成三层转发。总结成一句话就是：共用网关的关键不是“VLAN2/VLAN3 在链路上能把 VLAN4 跑起来”，而是“交换机在内部允许二级 VLAN 的终端把去网关的二层流量送达主端口”。

最后要回答“隔离还有什么意义”，答案恰恰是这方案最想实现的效果：MUX-VLAN隔离的是用户之间的二层邻接（L2 adjacency），而不是隔离用户到网关的能力。因为 VLAN2/VLAN3 在同一网段时，主机之间互访的第一步仍然是 ARP（广播问“谁是对方 IP”）。如果 VLAN2 想访问 VLAN3，VLAN2 主机就会广播 ARP 找 VLAN3 主机的 MAC；但 MUX-VLAN 不允许二级 VLAN 之间形成二层邻接（广播不会被送到对方用户口、对方也不会回 ARP），因此 VLAN2 主机拿不到 VLAN3 主机的 MAC，哪怕它们在同一网段，也仍然“看不见、连不上”。与此同时，上级路由器通常也不会替同一网段内的主机去做三层转发（它会认为同网段应该二层直达），所以也不会“帮你把 VLAN2 的包路由到 VLAN3”。于是你得到一个很有用的结果：两拨用户都能访问共同的网关/外网/主端口资源，但用户之间仍旧保持隔离——这就是方案一把“地址规划节省”和“二层隔离”同时兼顾的核心价值。

### 3.MUX VLAN 使用 VLAN 聚合技术

你明明想做“两拨用户（VLAN2/VLAN3）地址规划上看起来是同一个网段、默认网关也是同一个 IP”，但网络设备的能力把你常见的落地路径一条条堵死了，于是才不得不绕一圈，用 VLAN 聚合（Super/Sub VLAN）“造”出一个既能承载网关、又能服务 VLAN2/VLAN3 的三层锚点。具体因果链是这样的：首先，你的目标是让 VLAN2 和 VLAN3 的终端都使用同一个子网（比如都在 192.168.10.0/24）并指向同一个默认网关（比如 192.168.10.1），这通常意味着必须有一个三层接口持有这个网关 IP，因为网关本质上就是一个三层转发点：终端把发往外部或需要三层处理的流量交给它，它再负责路由转发。所以“网关”必须配置在某个能配置 IP 的三层接口上。接着问题来了：在 MUX-VLAN 场景里，VLAN2/VLAN3 只是从 VLAN（Group/Separate），它们的作用是做二层隔离/互通规则（端口角色决定谁能跟谁二层通信），而华为的实现里从 VLAN 不允许创建 VLANIF，也就不能直接在 VLAN2 或 VLAN3 上建 Vlanif2/Vlanif3 来配 192.168.10.1 这样的网关 IP；这条路被第一把锁卡死。你可能会说那我把网关放到上级三层交换机 SW1 的某个物理口上不就行了——像路由器接口那样，物理口直接配 IP，当成三层口，把下联链路当作一个三层点到点或三层接入来做；但文中又给了第二把锁：SW1 虽然叫“三层交换机”，但它的物理接口不支持三层路由口形态（不能把物理口切成 routed port，也不能在物理口上直接 ip address），它只能以二层 switchport 的方式工作（access/trunk/hybrid），三层能力只能通过 VLANIF（SVI）来体现。

于是你发现：你既不能在 VLAN2/VLAN3 上建 VLANIF 来当网关（因为它们是 MUX-VLAN 从 VLAN），也不能在 SW1 的物理口上直接配 IP 来当网关（因为物理口不能三层化），那么“网关到底落在哪里”就成了死结。为了解开这个结，就需要一个“既能建 VLANIF、又能让 VLAN2/VLAN3 的用户共享它”的承载体：这就是 VLAN 聚合（Super/Sub VLAN）出现的原因。VLAN 聚合的思路是把一个 VLAN（这里选 VLAN4）配置为 Super VLAN，允许在它上面创建 Vlanif4 并配置网关 IP（例如 192.168.10.1/24），同时把 VLAN2 和 VLAN3 配置为 Sub VLAN，让它们在二层仍然保持各自独立的 VLAN 身份（用户接入口依旧分在 VLAN2 或 VLAN3，MUX-VLAN 的二层隔离规则仍在 SW2 上生效），但在三层意义上“挂靠”到 Super VLAN 的网关接口上使用同一个网关和同一网段。这样做的结果是：SW1 通过 Vlanif4 成为 VLAN2/VLAN3 共同的三层出口点；SW1↔SW2 的上联链路必须以 trunk/hybrid 的形式放行 VLAN2 与 VLAN3（让这两类用户的二层报文能到达 SW1，进而被 Super VLAN 的三层网关处理），而 Super VLAN（VLAN4）本身通常不作为普通二层业务 VLAN 去在链路上“承载用户流量”，它更多是一个“承载 VLANIF、提供网关”的逻辑容器。

