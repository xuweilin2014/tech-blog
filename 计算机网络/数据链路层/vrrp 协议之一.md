# VRRP 协议

## 1.VRRP 概述

在下图 1 所示的网络中，多台 PC 连接在接入层交换机 SW 上，SW 通过单链路上联路由器 R1。这些 PC 属于相同的 IP 网段，并且均将默认网关地址配置为 R1 的 **`GE0/0/0`** 接口的 IP 地址 **`192.168.1.253`**。这个网络确实能够满足基本的通信需求，当 PC 发送数据到外部网络时，它们将数据包发给 R1，再由 R1 将数据包转发出去。然而该网络在可靠性上却存在极大的短板——PC 的默认网关没有冗余性，也就是说当 SW 与 R1 之间的互联链路发生故障时，或者 R1 发生故障时，PC 就丢失了与默认网关的连通性，它们与外部网络的通信也就断开了，业务自然会受到影响。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 1 网络的可靠性存在短板</div>
    <img src="vrrp_static//1.png" width="450"/>
</div>

可以在网络中增加一台路由器，作为冗余的网关，如图 2 所示。现在当网络正常时，PC 将默认网关设置为 **`192.168.1.253`**，而当 R1 发生故障时，则由用户更改 PC 网卡的配置，将默认网关修改为 **`192.168.1.252`**。显然这种方法是非常低效的，手工的配置变更增加了工作成本，当 PC 的数量特别多时，这部分工作量将变得非常大。**<font color="red">我们需要的解决方案是当故障发生时，网络要能够自动感知并且实现自动切换，网络对故障的响应过程对业务无影响，PC 端对此无感知</font>**。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 2 在网络中增加一台路由器</div>
    <img src="vrrp_static//2.png" width="450"/>
</div>

VRRP（Virtual Router Redundancy Protocol，虚拟路由器冗余协议）使得多台同属一个广播域的网络设备能够协同工作，实现设备冗余，从而提高网络的可靠性。**VRRP 目前有两个版本：VRRPv2 和 VRRPv3，其中 `VRRPv2` 仅适用于 IPv4 网络，而 `VRRPv3` 适用于 IPv4 和 IPv6 两种网络**。

如下图所示，在 R1 及 R2 的 **`GE0/0/0`** 接口上部署 VRRP，使得两者能够协同工作，可以实现网关冗余。当 VRRP 开始工作后，它将产生一台虚拟路由器，这台虚拟路由器的 IP 地址为 **`192.168.1.254`**（**<font color="red">该地址由网络管理员指定</font>**），PC 将自己的默认网关设置为该虚拟路由器的 IP 地址。如此一来，当 PC 向外部网络发送数据时，数据将被发送给虚拟路由器。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 3 在 R1 以及 R2 上部署 VRRP</div>
    <img src="vrrp_static//3.png" width="450"/>
</div>

值得注意的是，虚拟路由器是一台逻辑设备，它只是 VRRP 虚拟出来的一台路由器，当 VRRP 开始工作后，R1 及 R2 会进行选举，胜出的路由器成为 Master（主）路由器，其他的路由器则成为 Backup（备份）路由器。

Master 路由器承担虚拟路由器的具体工作，如此一来当 PC 需要发送数据包到外部网络时，数据包实际被发给 Master 路由器 R1（如下图 4 所示），**<font color="red">而当 R1 发生故障时，通过 VRRP 协议的运作，R2 能感知到当前的 Master 路由器发生了故障，从而将自己的状态自动地切换到 Master</font>**，接下来它将接替原来属于 R1 的工作（如下图 5 所示）。在整个 VRRP 的切换过程中，用户是完全无感知的，PC 的配置也不需要做任何变更。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 4 R1 作为 Master 路由器转发 PC 发往外部网络的数据</div>
    <img src="vrrp_static//4.png" width="450"/>
</div>
<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 5 VRRP 实现网关设备的平滑迁移</div>
    <img src="vrrp_static//5.png" width="450"/>
</div>

## 2.VRRP 基本概念

### 2.1 VRRP 路由器

我们将运行 VRRP 的路由器称为 VRRP 路由器。**<font color="red">实际上，VRRP 是配置在路由器的接口上的，而且也是基于接口来工作的</font>**。如图 6 所示，R1 及 R2 均在各自的 **`GE0/0/0`** 上配置了 VRRP，VRRP 一旦被激活，路由器的接口便开始发送及侦听 VRRP 协议报文。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">图 6 R1 及 R2 的接口上配置 VRRP 并加入相同的 VRRP 组</div>
    <img src="vrrp_static//6.png" width="450"/>
</div>

需要协同工作的 VRRP 路由器（的接口）必须属于同一个广播域，否则 VRRP 报文无法正常交互。在本例中，R1 的 **`GE0/0/0`** 接口与 R2 的 **`GE0/0/0`** 接口连接在同一台二层交换机上，**<font color="red">而且交换机连接这两台路由器的接口属于相同的 VLAN，因此 R1 及 R2 的这两个接口即属于相同的广播域</font>**。一旦交换机上的 VLAN 配置错误导致 R1 及 R2 属于不同 VLAN，那么 VRRP 的工作也将出现问题。

### 2.2 VRRP 组以及 VRID

