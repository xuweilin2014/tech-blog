# OSPF 拓扑描述及路由计算

## 1.实验案例

我们以下面这个 OSPF 实验案例来分析一下 LSA1 和 LSA2 的内容，拓扑图如下所示：

<div align="center">
    <img src="ospf_static/123.png" width="450"/>
</div>

按图中的网段规划，Area1 的网段为 **`10.1.45.0/24`**，Area2 的网段为 **`10.1.123.0/24`**，Area0 的网段为 **`10.1.234.0/24`**。并且不同的区域中网络类型不同，Area1 中的网络类型是点对点，Area2 中的网络类型是 NBMA，Area0 中的网络类型是广播。

AR1 的配置信息如下所示，AR1 的 dr-priority 设置为 100，为了让实验结果稳定，将 AR1 设置为 AR1-AR3 中 dr-priority 最高的路由器。

```java{.line-numbers}
#
 sysname AR1
#
interface GigabitEthernet0/0/0
 ip address 10.1.123.1 255.255.255.0 
 ospf network-type nbma
 ospf dr-priority 100
#
interface LoopBack0
 ip address 10.1.1.1 255.255.255.0 
#
ospf 1 router-id 1.1.1.1 
 peer 10.1.123.2
 peer 10.1.123.3
 area 0.0.0.2 
  network 10.1.1.1 0.0.0.0 
  network 10.1.123.1 0.0.0.0 
```

AR2 的配置信息如下所示，AR2 有 2 个 loopback 接口，分别是 LoopBack0 和 LoopBack1，LoopBack0 位于 Area2 中，LoopBack1 位于 Area0 中。并且由于 Area2 的网络类型是 NBMA，而且由于 NBMA 网络无法靠组播 hello 报文来发现邻居，所以我们在 AR2 和 AR3 上都配置了 peer 命令来手动指定邻居，**<font color="red">同时由于 NBMA/Broadcast 网络都是全互联网络，因此 AR2 上需要使用 peer 命令来指定 AR1 和 AR3 这两个邻居，而 AR3 上也同理</font>**。

```java{.line-numbers}
#
 sysname AR2
#
interface GigabitEthernet0/0/0
 ip address 10.1.123.2 255.255.255.0 
 ospf network-type nbma
 ospf dr-priority 50
#
interface GigabitEthernet0/0/1
 ip address 10.1.234.2 255.255.255.0
 ospf network-type broadcast
 ospf dr-priority 50
#
interface LoopBack0
 ip address 10.1.2.2 255.255.255.0 
#
interface LoopBack1
 ip address 10.1.22.22 255.255.255.0
#
ospf 1 router-id 2.2.2.2 
 peer 10.1.123.1
 peer 10.1.123.3
 area 0.0.0.0 
  network 10.1.234.2 0.0.0.0
  network 10.1.22.22 0.0.0.0
 area 0.0.0.2 
  network 10.1.2.2 0.0.0.0 
  network 10.1.123.2 0.0.0.0 
```

AR3 的配置信息如下所示，AR3 也有 2 个 loopback 接口，分别是 LoopBack0 和 LoopBack1，LoopBack0 位于 Area2 中，LoopBack1 位于 Area0 中。

```java{.line-numbers}
#
 sysname AR3
#
interface GigabitEthernet0/0/0
 ip address 10.1.123.3 255.255.255.0 
 ospf network-type nbma
 ospf dr-priority 15
#
interface GigabitEthernet0/0/1
 ip address 10.1.234.3 255.255.255.0
 ospf network-type broadcast
 ospf dr-priority 15
#
interface LoopBack0
 ip address 10.1.3.3 255.255.255.0 
#
interface LoopBack1
 ip address 10.1.33.33 255.255.255.0 
#
ospf 1 router-id 3.3.3.3 
 peer 10.1.123.1
 peer 10.1.123.2
 area 0.0.0.0 
  network 10.1.234.3 0.0.0.0 
  network 10.1.33.33 0.0.0.0
 area 0.0.0.2 
  network 10.1.3.3 0.0.0.0 
  network 10.1.123.3 0.0.0.0 
```

AR4 的配置信息如下所示：

```java{.line-numbers}
#
sysname AR4
#
interface Serial0/0/0
 link-protocol ppp
 ip address 10.1.45.4 255.255.255.0 
#
interface GigabitEthernet0/0/0
 ip address 10.1.234.4 255.255.255.0
 ospf network-type broadcast
 ospf dr-priority 100
#
interface LoopBack0
 ip address 10.1.4.4 255.255.255.0
#
interface LoopBack1
 ip address 10.1.44.44 255.255.255.0
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.0 
  network 10.1.234.4 0.0.0.0
  network 10.1.44.44 0.0.0.0
 area 0.0.0.1 
  network 10.1.45.4 0.0.0.0
  network 10.1.4.4 0.0.0.0
```

AR5 的配置信息如下所示：

```java{.line-numbers}
#
sysname AR5
#
interface Serial0/0/1
 link-protocol ppp
 ip address 10.1.45.5 255.255.255.0 
#
interface LoopBack0
 ip address 10.1.5.5 255.255.255.0 
#
ospf 1 router-id 5.5.5.5 
 area 0.0.0.1 
  network 10.1.45.5 0.0.0.0 
  network 10.1.5.5 0.0.0.0 
```

## 2.LSA1 和 LSA2

### 2.1 LSA1 和 LSA2 报文内容

每台路由器会为自己所处的每个区域产生一份 Router LSA，该 Router LSA 包含属于该区域的所有链路的链路状态信息。在上述拓扑的 AREA2 中 AR2 收到 AR3 发送的  Router-LSA 报文格式如下所示。

```java{.line-numbers}
Number of LSAs: 1
LSA-type 1 (Router-LSA), len 48
    .000 0000 0000 0001 = LS Age (seconds): 1
    0... .... .... .... = Do Not Age Flag: 0
    Options: 0x02, (E) External Routing
        0... .... = DN: Not set
        .0.. .... = O: Not set
        ..0. .... = (DC) Demand Circuits: Not supported
        ...0 .... = (L) LLS Data block: Not Present
        .... 0... = (N) NSSA: Not supported
        .... .0.. = (MC) Multicast: Not capable
        .... ..1. = (E) External Routing: Capable
        .... ...0 = (MT) Multi-Topology Routing: No
    LS Type: Router-LSA (1)
    Link State ID: 3.3.3.3
    Advertising Router: 3.3.3.3
    Sequence Number: 0x80000007
    Checksum: 0x3dc4
    Length: 48
    Flags: 0x01, (B) Area border router
    Number of Links: 2
    Type: Transit   ID: 10.1.123.1       Data: 10.1.123.3        Metric: 1
    Type: Stub      ID: 10.1.3.3         Data: 255.255.255.255   Metric: 0
```

Router LSA 包含以下几项：

- **`LS Age`**：16 位数，后 15 位数用来表示 age，LSA 初始产生时，age 数值为 0；**<font color="red">最高位有特殊含义，置位则代表该 LSA 在 LSDB 中年龄不老化过期（DoNotAge）</font>**；若没有置位，则 age 正常老化，即在 LSDB 中年龄老化。
- **`LS Type`**：LSA 的类型，Type=1。
- **`Link State ID`**：产生该 LSA 的路由器的 RouterID。
- **`Advertising Router`**：产生该 LSA 的路由器 RouterID。
- **`SequenceNumber`**：线性的序列号，初始值从 **`0x80000001`** 开始递增。新的 LSA 序列号会增加。
- **`Checksum`**：对整个 LSA 做 CheckSum（除去 Age 字域）。
- **`Flags`**：**<font color="red">V 若置位，代表是 Vlink endpoint；B 若置位，代表是 ABR；E 若置位代表是 ASBR</font>**。
- **`Number of Links`**：Link 的数量，代表 OSPF 画出的有向图上的 Link 的数量，而非物理路由器接口的数量。

Network LSA，简称为 LSA2，用来描述多点网络上的拓扑关系。**`Network LSA 仅会出现在网络类型是 Broadcast 或 NBMA 的网络`** 上。Broadcast 和 NBMA 网络上会选举 DR，选举出来的 DR 除用于数据库同步外，DR 也负责产生 LSA2。LSA2 用于描述虚节点周边的连接关系。这个 LSA2 用 DR 路由器的接口 IP 来标识。

```java{.line-numbers}
v LS Update Packet
    Number of LSAs: 2
  > LSA-type 1 (Router-LSA), len 48
  v LSA-type 2 (Network-LSA), len 36
        .000 0000 0000 0001 = LS Age (seconds): 1
        0... .... .... .... = Do Not Age Flag: 0
      v Options: 0x02, (E) External Routing
            0... .... = DN: Not set
            .0.. .... = O: Not set
            ..0. .... = (DC) Demand Circuits: Not supported
            ...0 .... = (L) LLS Data block: Not Present
            .... 0... = (N) NSSA: Not supported
            .... .0.. = (MC) Multicast: Not capable
            .... ..1. = (E) External Routing: Capable
            .... ...0 = (MT) Multi-Topology Routing: No
        LS Type: Network-LSA (2)
        Link State ID: 10.1.234.4
        Advertising Router: 4.4.4.4
        Sequence Number: 0x8000000a
        Checksum: 0xe33b
        Length: 36
        Netmask: 255.255.255.0
        Attached Router: 4.4.4.4
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3
```

Network LSA 包含内容如下：

- **`LS Age`**：同 LSA1；
- **`LS Type`**：type=2；
- **`Link State ID`**：**`DR 的接口 IP 地址`**；
- **`Advertising Router`**：产生 LSA2 的通告路由器。
- **`SequenceNumber`**：第一份 **`SequenceNumber`** 为 **`0x80000001`**，每次更新 Sequence Number 增 1；
- **`Checksum`**：对除 Age 外的 LSA 内容做计算；
- **`Netmask`**：和 Link State ID 执行与运算，得出 LSA2 所代表的网络号；
- **`Attached Router`**：连接到本网络的所有邻居路由器的 RouterID；

根据上面报文中的 LinkStateID 和 Netmask，可计算出 LSA2 代表网络 **`10.1.234.0/24`**，并根据 Attached Router 的表述，得知该网络同时连接 3 台路由器——R2、R3、R4。**<font color="red">所以 LSA2 作用是通告 TransNet 网络号及描述该网络和路由器的连接关系</font>**。因为该 LSA2 中既包含拓扑连接的信息，又包含网络信息。所以，当网络变化时，即使物理拓扑没有变化，同样会触发重新产生 LSA2，并致每台路由器都重新发生 SPF 计算。

LSA2 是由 MA 网络上的 DR 路由器产生的，使用 DR 接口 IP 地址作为 LSA2 的 Link State ID。相比于 LSA1 是由实节点产生并描述实节点的周边的连接关系和网络，LSA2 是由 DR 为虚节点产生，描述虚节点周边的连接关系和网络信息。

每个节点（除虚节点外）外出方向的链路成本为接口成本（可使用 OSPF Cost 修改），而虚节点是代表中间网络虚拟出来的节点，处在拓扑中间，不能引入额外成本，所以由其向外延伸的链路成本为 0。

### 2.2 Link 类型及描述

Router LSA 中的 Link 类型有以下几种：

OSPF 定义了 4 种类型 Link，路由器接口的 OSPF 网络类型不同，产生的 Link 也不同，路由器把所有接口的 Link 放到 RouterLSA 中在区域内泛洪。RFC2328 定义 OSPF Router LSA 只用 4 种类型的 Link 来描述各种类型的网络，每种 Link 的含义及构成见下表所示。

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">表 1 Router-LSA 所定义的四种 Link 类型</div>
    <img src="ospf_static/124.png" width="700"/>
</div>

### 2.3 Link 类型 1：P2P 类型

OSPF 节点间为点到点链路，如 PPP 或 HDLC 链路，OSPF 默认的网络类型为 **`ospf network point-to-point`**，则节点在表述拓扑关系时，使用 P2P 类型 Link。在上面的拓扑图中，AR4 和 AR5 之间的链路就是点对点链路，因此在 AR4 和 AR5 的 Router LSA 中都会有一个 Link 类型为 P2P 的 Link 来描述这条链路。

AR4 上 ospf lsdb 中的 router lsa 如下所示：

```java{.line-numbers}
        Area: 0.0.0.1
		Link State Database 

  Type      : Router
  Ls id     : 4.4.4.4
  Adv rtr   : 4.4.4.4  
  Ls age    : 639 
  Len       : 60 
  Options   :  ABR  E  
  seq#      : 80000007 
  chksum    : 0x42e
  Link count: 3
   * Link ID: 5.5.5.5      
     Data   : 10.1.45.4    
     Link Type: P-2-P        
     Metric : 1562
   * Link ID: 10.1.45.0    
     Data   : 255.255.255.0 
     Link Type: StubNet      
     Metric : 1562 
     Priority : Low
   * Link ID: 10.1.4.4     
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium

  Type      : Router
  Ls id     : 5.5.5.5
  Adv rtr   : 5.5.5.5  
  Ls age    : 642 
  Len       : 60 
  Options   :  E  
  seq#      : 80000007 
  chksum    : 0xe348
  Link count: 3
   * Link ID: 4.4.4.4      
     Data   : 10.1.45.5    
     Link Type: P-2-P        
     Metric : 1562
   * Link ID: 10.1.45.0    
     Data   : 255.255.255.0 
     Link Type: StubNet      
     Metric : 1562 
     Priority : Low
   * Link ID: 10.1.5.5     
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium
```

第一条 Router LSA 的 **`Adv rtr`** 和 **`Ls id`** 都是 **`4.4.4.4`**，代表该 Router LSA 是由 AR4 产生的。该 Router LSA 中第一个 Link 类型为 P2P，用来描述 AR4 和 AR5 之间的链路关系，Link ID 是邻居路由器的 RID，也就是 AR5 的 RID，Data 是 AR4 上该接口（**`S0/0/0`**）的 IP 地址 **`10.1.45.4`**；第二个 Link 类型为 StubNet，用来描述 AR4 上 **`S0/0/0`** 接口所在的网段 **`10.1.45.4/24`**，Link ID 是网段的网络地址，Data 是该网段的子网掩码；第三个 Link 类型也是 StubNet，用来描述 AR4 上 **`loopback0`** 接口所在的网段 **`10.1.4.4/32`**。第二条 Router LSA 同理。

根据上面的 router lsa 的内容，我们可以画出 AR4 和 AR5 之间的点到点逻辑拓扑图如下所示：

<div align="center">
    <img src="ospf_static/125.png" width="650"/>
</div>

### 2.4 Link 类型 2：StubNet 类型

**<font color="red">StubNet 代表一个网络，用末节节点来表示，附着（挂在）在实节点上，不表示任何连接关系，其实是实节点上的网络</font>**。在 OSPF 逻辑图上，StubNet 类型 Link 可以表示挂在实节点上的叶子节点。上面的 AR4 和 AR5 之间的 P2P 逻辑拓扑图表示 AR4 上有 2 个 **`StubNet`** 网络，也就有 2 个 **`StubNet`** 网络的叶子节点，挂在 AR4 上。

### 2.5 Link 类型 3：TransNet 类型

**<font color="red">在 OSPF 中，多路访问网络上如果有多个 OSPF 节点，彼此之间会形成全互联的邻居关系</font>**。

#### 2.5.1 Area0 中 Broadcast 网络

在上面的拓扑图中，Area0 中的 AR2、AR3、AR4 之间的网络类型是 Broadcast，因此在 AR2、AR3、AR4 的 Router LSA 中都会有一个 Link 类型为 TransNet 的 Link 来描述该网络。

```java{.line-numbers}
<AR2>display ospf lsdb router

	 OSPF Process 1 with Router ID 2.2.2.2
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Router
  Ls id     : 4.4.4.4
  Adv rtr   : 4.4.4.4  
  Ls age    : 508 
  Len       : 48 
  Options   :  ABR  E  
  seq#      : 80000010 
  chksum    : 0x8833
  Link count: 2
   * Link ID: 10.1.234.4   
     Data   : 10.1.234.4   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.44.44   
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium

  Type      : Router
  Ls id     : 2.2.2.2
  Adv rtr   : 2.2.2.2  
  Ls age    : 511 
  Len       : 48 
  Options   :  ABR  E  
  seq#      : 8000000e 
  chksum    : 0x53a8
  Link count: 2
   * Link ID: 10.1.234.4   
     Data   : 10.1.234.2   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.22.22   
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium

  Type      : Router
  Ls id     : 3.3.3.3
  Adv rtr   : 3.3.3.3  
  Ls age    : 510 
  Len       : 48 
  Options   :  ABR  E  
  seq#      : 8000000d 
  chksum    : 0xf1eb
  Link count: 2
   * Link ID: 10.1.234.4   
     Data   : 10.1.234.3   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.33.33   
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium
```

以 AR2 的 Router LSA 为例。第一条 Router LSA 的 **`Adv rtr`** 和 **`Ls id`** 都是 **`4.4.4.4`**，表示由 AR4 产生的 Router LSA，该 Router LSA 中有一个 Link 类型为 TransNet 的 Link 来描述 AR4 和 AR2 之间的 Broadcast 网络，Link ID 是 DR 的接口 IP 地址，也就是 **`10.1.234.4`**，Data 是 AR4 自己的接口 IP 地址。第二个 Link 类型为 StubNet，用来描述 AR2 上 loopback1 接口所在的网段。

第二条 Router LSA 的 **`Adv rtr`** 和 **`Ls id`** 都是 **`2.2.2.2`**，表示由 AR2 产生的 Router LSA。该 Router LSA 中的类型为 TransNet 的 Link 用来描述 AR2 和 AR4 之间的 Broadcast 网络，Link ID 是 DR 的接口即 **`10.1.234.4`** 的 IP 地址，Data 是 AR2 自己的接口 IP 地址 **`10.1.234.2`**。第二个 Link 类型为 StubNet，用来描述 AR2 上 loopback1 接口所在的网段。

AR2 上的 network lsa 如下所示：

```java{.line-numbers}
<AR2>display ospf lsdb network 

	 OSPF Process 1 with Router ID 2.2.2.2
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Network
  Ls id     : 10.1.234.4
  Adv rtr   : 4.4.4.4  
  Ls age    : 1712 
  Len       : 36 
  Options   :  E  
  seq#      : 8000000b 
  chksum    : 0xe13c
  Net mask  : 255.255.255.0
  Priority  : Low
     Attached Router    4.4.4.4
     Attached Router    2.2.2.2
     Attached Router    3.3.3.3
```

根据上面 AR2 的 Router LSA 和 Network LSA 的内容，我们可以画出 AR2、AR3、AR4 之间的 Broadcast 逻辑拓扑图如下所示：

<div align="center">
    <img src="ospf_static/126.png" width="550"/>
</div>

#### 2.5.2 Area2 中 NBMA 网络

在上面的拓扑图中，Area0 中的 AR2、AR3、AR4 之间的网络类型是 NBMA，因此在 AR2、AR3、AR4 的 Router LSA 中都会有一个 Link 类型为 TransNet 的 Link 来描述该网络。

```java{.line-numbers}
        Area: 0.0.0.2
    Link State Database 

  Type      : Router
  Ls id     : 2.2.2.2
  Adv rtr   : 2.2.2.2  
  Ls age    : 1173 
  Len       : 48 
  Options   :  ABR  E  
  seq#      : 8000000e 
  chksum    : 0x42c3
  Link count: 2
   * Link ID: 10.1.123.1   
     Data   : 10.1.123.2   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.2.2     
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium
 
  Type      : Router
  Ls id     : 1.1.1.1
  Adv rtr   : 1.1.1.1  
  Ls age    : 1179 
  Len       : 48 
  Options   :  E  
  seq#      : 8000000f 
  chksum    : 0x50c0
  Link count: 2
   * Link ID: 10.1.123.1   
     Data   : 10.1.123.1   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.1.1     
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium

  Type      : Router
  Ls id     : 3.3.3.3
  Adv rtr   : 3.3.3.3  
  Ls age    : 1181 
  Len       : 48 
  Options   :  ABR  E  
  seq#      : 8000000e 
  chksum    : 0x2fcb
  Link count: 2
   * Link ID: 10.1.123.1   
     Data   : 10.1.123.3   
     Link Type: TransNet     
     Metric : 1
   * Link ID: 10.1.3.3     
     Data   : 255.255.255.255 
     Link Type: StubNet      
     Metric : 0 
     Priority : Medium
```

AR2 上的 network lsa 如下所示：

```java{.line-numbers}
        Area: 0.0.0.2
    Link State Database 

  Type      : Network
  Ls id     : 10.1.123.1
  Adv rtr   : 1.1.1.1  
  Ls age    : 1498 
  Len       : 36 
  Options   :  E  
  seq#      : 8000000a 
  chksum    : 0xbfe9
  Net mask  : 255.255.255.0
  Priority  : Low
     Attached Router    1.1.1.1
     Attached Router    2.2.2.2
     Attached Router    3.3.3.3
```

根据上面 AR2 的 Router LSA 和 Network LSA 的内容，我们可以画出区域 2 中 AR1、AR2、AR3 之间的 NBMA 逻辑拓扑图如下所示：

<div align="center">
    <img src="ospf_static/127.png" width="550"/>
</div>

总结来说，OSPF 网络接口类型的区别如下所示：

- P2P 网络类型适用于广域网点对点网络的场景，不选举 DR/BDR，会主动发送组播 hello 报文；
- Broadcast 网络类型适用于广播网络的全互联网络场景，选举 DR/BDR，会主动发送组播 hello 报文；
- NBMA 网络类型适用于非广播网络的全互联网络的场景，选举 DR/BDR，不会主动发送组播 hello 报文；
- P2MP 适用于部分互联的网络场景，不会选举 DR/BDR，会主动发送组播 hello 报文，会产生 32 位 OSPF 主机路由；

在 OSPF 的逻辑拓扑中，表达拓扑的点有代表路由器的实节点和代表 TransNet 网络的虚节点。如果把代表路由器的节点叫作实节点（Node），则把代表 TransNet 网络的节点用虚节点（Pseduonode）代表。

网络分 TransNet 网络和 Stub 网络，二者的区别是 TransNet 网络上有至少两个节点，流量可以经过 TransNet 网络在节点间传递，而 Stub 网络仅代表连在一个节点（路由器）上的网络，流量只能进或者出该网络。Stub 网络包含如下类型：Loopback 接口的网络、无邻居的以太网接口网络及点到点链路上每个节点（路由器）所连的网络。

TransNet 网络要表达 4 个节点间全互联可达的关系，在数学模型的描述中，把中间的 Network 表述成一个节点，并连接每个节点。

## 3.SPF 路由计算

### 3.1 OSPF 路由计算

执行 SPF 计算包含三步：

- 路由器根据 LSDB 中的 LSA 画出网络图，**<font color="red">在这个 Graph 上包含 OSPF 网络拓扑中所有点和边，画图需要使用 LSA1 中 `Point-to-Point`、`TransNet` 和 `Vlink` 类型的 link 及 LSA2 来描述拓扑</font>**。

>由于每台路由器的 LSDB 内容一样，所以在每台路由器上根据 LSDB 中 LSA 画出的区域内的 Graph 也是一样的。区别是最先画起的路由器（起点路由器）不一样，**根据 RFC 2328 文档，每台路由器都是以自己为根来构造最短路径树**。

- 路由器以自己为树根，对图（Graph）执行 SPF 计算。画出一棵由树根到图中每个节点的最短成本路径树，从树根到树上任何其他节点的成本是最小的。
- 在树的节点上添加网络信息，并计算由树根到这些网络的成本及下一跳，并把计算结果加入到路由表中。把 SPF 树上添加的网络，称为叶子节点。SPF 已计算出树根到任何节点的最小距离，再把代表网络的叶子节点挂在 SPF 树的相应网络节点上。**`树根到网络的路由成本 = 树根到网络节点的距离 + 网络节点到叶子节点的成本`**。叶子节点可以是 LSA1 中的 StubNet、LSA2 中的网络、LSA3 的网络、LSA5/7 的网络。

>Stub 网络挂在路由器节点（实节点）上，LSA2 中网络挂在虚节点上，LSA3 的网络挂在 ABR 节点上，LSA7 挂在 ASBR 节点上，而 LSA5 则可能挂在 ASBR 或 ABR 上，这要依 ASBR 路由器是否在当前区域内而定。

因此，OSPF 的区域内最短路径计算更准确地说是一个两阶段过程。第一阶段先构造网络主干，也就是仅依据 routers 与 transit networks 之间的连接关系，建立最短路径树的主干结构。RFC 2328 的原文是：At first the tree is constructed by only considering those links between routers and transit networks. Then the stub networks are incorporated into the tree。在主干树建立完成后，第二阶段才把各路由器 Router-LSA 中描述的 stub networks 挂接到对应的树节点上。这样做的原因在于 stub network 本身不会影响到其他中转节点之间的最短路径结构。

### 3.2 iSPF

在路由计算方面，优化网络中的 SPF 计算过程，可降低计算负荷和收敛时间。iSPF 其主要思想就是增量计算（即只计算变化的部分，而不是全部计算）。SPF 算法将整个网络信息分为两个部分，如下图所示，**<font color="red">一个部分是网络的节点（对应于网络中的路由器、共享网段）和边（路由器以及共享网段之间的链路）组成的网络拓扑；另一个部分是挂在节点上的叶子（网段路由、主机路由等）</font>**。执行路由计算的路由器为树根（Root）。

<div align="center">
    <img src="ospf_static/128.png" width="450"/>
</div>

当网络链路状态发生变化时，iSPF 会判断出哪部分网络拓扑受到了影响，从而只计算那些受到了影响的部分，而不是全部网络拓扑。

网络拓扑变化的位置不同，受到影响的范围就不同，iSPF 计算所消耗的时间就不同，所以，iSPF 计算所消耗的时间是不确定的，即使是在相同的网络结构中。当然，如果发生变化的是根节点的边，那么受影响的范围就包括了整个拓扑，在这种情况下，iSPF 相当于进行了全部重新计算（Full SPF）。

### 3.3 PRC

任何一条路由都是网络节点上的一片树叶，即叶子节点，**<font color="red">从根节点看，只要到树中任何实（或虚）节点的最短路径确定了，到叶子节点的最短路径也就确定了</font>**，那么到节点发布的路由的最短路径也就确定了。因此，PRC 就是在 iSPF 计算出的最短路径树基础上再来计算叶子节点代表的路由。当有路由信息改变，PRC 直接判断在哪条链路上的节点的哪个叶子出现变化，之后直接进行路由的计算与更新。

### 3.4 路由计算示例

**（1）案例一**

物理网络如下图所示，计算 R4 到图中所有网络的路由。图中，Loop0 接口的 OSPF 成本设为 1。

<div align="center">
    <img src="ospf_static/129.png" width="500"/>
</div>

逻辑拓扑图如下所示，图是有向图，任何路由器上画出的图都一样。

<div align="center">
    <img src="ospf_static/130.png" width="350"/>
</div>

计算以 R4 为树根的 SPF 树，如下所示：

<div align="center">
    <img src="ospf_static/131.png" width="300"/>
</div>

添加叶子节点并计算路由，如下所示：

<div align="center">
    <img src="ospf_static/132.png" width="450"/>
</div>

从上图中可以看出，R4 路由器到网络 **`30.1.1.0/24`**，下一跳是 R1，路径成本是 **`40+11+0+10=61`**。这里解释一下，cost 的计算方式。

根据 RFC 2328 文档，Arcs are labelled with the cost of the corresponding router output interface. 除此之外，Interface output cost is the cost of sending a data packet on the interface, expressed in the link state metric. This is advertised as the link cost for this interface in the router-LSA. The cost of an interface must be greater than zero. **<font color="red">也就是说，OSPF 会给每个接口分配一个 cost，这个 cost 表示从本路由器经这个接口把包发出去的代价</font>**。这个代价随后会被写进本路由器发布的 Router-LSA 里，供全网其他 OSPF 路由器计算最短路径时使用。这里强调的是输出方向（output cost），所以同一条链路两端的 cost 可以不同。RFC 对点到点接口描述内容，Router-LSA 里的 cost 要设置为 the output cost of the point-to-point interface。另外，网络到路由器的代码都是 0，这是因为 RFC 规定了 arcs leading from networks to routers always have cost 0。

到 **`14.1.1.0/24`**，R4 在树上有两条路都能到达 **`14.1.1.0/24`**，选择成本最小的一条放到路由表里，在 OSPF 路由表（Display Ospf Routing）中，到 **`14.1.1.0/24`** 成本是 40。但 IP 路由表中，由于有直连路由 **`14.1.1.0/24`** 存在，所以路由表中看不到 OSPF 的 **`14.1.1.0/24`** 路由。

**`123.1.1.0/24`** 这条路由是图中的多点网络，由虚节点 N 所代表，可把 **`123.1.1.0/24`** 表示成挂在虚节点后面的叶子节点，虚节点不引入额外成本，故向外延伸的方向的成本都是 0，成本是 **`40+11+0=51`**。

**（2）案例二**

根据前面的分析，我们可以将前面的实验拓扑图简化成如下所示的逻辑拓扑图（含接口的 cost）：

<div align="center">
    <img src="ospf_static/133.png" width="800"/>
</div>

计算以 AR1 为树根的 SPF 树，R1 到拓扑图中各网段的 cost 代价如下所示，注意计算从 AR1 到其余网段路径代价 cost 时均使用出接口的 cost 来计算。

```java{.line-numbers}
<AR1>display ospf routing

	 OSPF Process 1 with Router ID 1.1.1.1
		  Routing Tables 

 Routing for Network 
 Destination        Cost  Type       NextHop         AdvRouter       Area
 10.1.1.1/32        0     Stub       10.1.1.1        1.1.1.1         0.0.0.2
 10.1.123.0/24      1     Transit    10.1.123.1      1.1.1.1         0.0.0.2
 10.1.2.2/32        1     Stub       10.1.123.2      2.2.2.2         0.0.0.2
 10.1.3.3/32        1     Stub       10.1.123.3      3.3.3.3         0.0.0.2
 10.1.4.4/32        14    Inter-area 10.1.123.2      2.2.2.2         0.0.0.2
 10.1.5.5/32        1576  Inter-area 10.1.123.2      2.2.2.2         0.0.0.2
 10.1.22.22/32      1     Inter-area 10.1.123.2      2.2.2.2         0.0.0.2
 10.1.33.33/32      1     Inter-area 10.1.123.3      3.3.3.3         0.0.0.2
 10.1.44.44/32      14    Inter-area 10.1.123.2      2.2.2.2         0.0.0.2
 10.1.45.0/24       1576  Inter-area 10.1.123.2      2.2.2.2         0.0.0.2
 10.1.234.0/24      14    Inter-area 10.1.123.2      2.2.2.2         0.0.0.2

 Total Nets: 11 
 Intra Area: 4  Inter Area: 7  ASE: 0  NSSA: 0 
```

## 4.DR 和 BDR 选举

只要网络类型是 Broadcast 或 NBMA，且这个网络上有多于一台路由器，则此网络需要且必定要选举 DR。DR 的作用一是用于数据库同步，二是为代表网络的虚节点产生 LSA2。DR 是 MA 网络上负责数据库同步的路由器，网络上所有路由器都和 DR 有邻接关系。路由器优先级在 1～255 之间都有资格成为 DR。每台路由器根据收到的 Hello 看是否有路由器已声称为 DR。

在 Wait 时间后，开始选举 DR，优先级高的路由器成为 DR，否则 RouterID 高的路由器成为 DR。DR 不能被抢占，一旦选举结束，即使有更高优先级的路由器接到网络中来，也不能抢占成为 DR，同样也不会抢占成为 BDR。这是为了保持网络稳定，避免震荡。当 DR 选举完成后，如果当前 DR 失效，则 BDR 成为 DR，并重新选举 BDR。