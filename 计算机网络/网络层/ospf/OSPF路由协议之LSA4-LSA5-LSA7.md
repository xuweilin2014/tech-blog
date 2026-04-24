# OSPF 路由协议之 LSA4 和 LSA5

## 1.LSA4

LSA4（ABR summary）像 LSA3 一样都是由 ABR 产生的，并在 Area 内泛洪的一类 LSA。LSA4 和 LSA3 使用相同的报文格式，区别是 Type 字域是 4，**<font color="red">Link State ID 字域是 ASBR 路由器的 Router ID，LSA4 的内容是 ASBR 到 ABR 的成本，在 LSA4 中，AdvRouter 为 ABR 的 Router ID，并且会随着 ABR 的不同而发生变化</font>**。

我们以下面的 topo 图来分析 LSA4，其中 R1 会引入外部路由 **`10.1.6.0/24`**。

<div align="center">
    <img src="ospf_static/108.png" width="800"/>
</div>

上述拓扑图中的 AR6 的配置如下所示：

```java{.line-numbers}
#
 sysname AR6
#
interface GigabitEthernet0/0/0
 ip address 10.1.16.6 255.255.255.0 
#
interface LoopBack0
 ip address 10.1.6.6 255.255.255.0 
#
rip 1
 undo summary
 version 2
 peer 10.1.16.1
 network 10.0.0.0
```

AR1 的配置如下所示：

```java{.line-numbers}
#
 sysname AR1
#
firewall zone Local
 priority 15
#
interface GigabitEthernet0/0/0
 ip address 10.1.12.1 255.255.255.0 
 ospf cost 2
#
interface GigabitEthernet0/0/1
 ip address 10.1.16.1 255.255.255.0 
#
ospf 1 router-id 1.1.1.1 
 import-route rip 1 cost 10 type 2 route-policy RIP_TO_OSPF
 area 0.0.0.1 
  network 10.1.12.1 0.0.0.0 
  network 192.168.12.0 0.0.0.255 
#
rip 1
 undo summary
 version 2
 peer 10.1.16.6
 network 10.0.0.0
#
route-policy RIP_TO_OSPF permit node 10 
 if-match ip-prefix RIP_TO_OSPF 
#
ip ip-prefix RIP_TO_OSPF index 20 permit 10.1.6.0 24
```

AR4 在 Area2 上收到的 LSA4 和 AR5 在 Area0 上收到的 LSA4 如下所示。AR4 收到的 LSA4 中 AdvRouter 是 ABR R3 的 Router ID，而 AR5 收到的 LSA4 中 AdvRouter 是 ABR R2 的 Router ID，**<font color="red">同一条 ASBR 路由在不同 Area 内的 LSA4 中 AdvRouter 是不同的</font>**。但是 AR4 和 AR5 收到的 LSA4 中 Link State ID 都是 ASBR R1 的 Router ID，**<font color="red">同一条 ASBR 路由在不同 Area 内的 LSA4 中 Link State ID 是相同的</font>**。**<font color="blue">LSA4 仅当网络中有 ASBR 时，才在区域间由 ABR 产生并泛洪，每个区域可通过 LSA4 计算出到 ASBR 的距离</font>**。

```java{.line-numbers}
<AR4>display ospf lsdb asbr

	 OSPF Process 1 with Router ID 4.4.4.4
		         Area: 0.0.0.2
		 Link State Database 

  Type      : Sum-Asbr
  Ls id     : 1.1.1.1
  Adv rtr   : 3.3.3.3  
  Ls age    : 1038 
  Len       : 28 
  Options   :  E  
  seq#      : 80000005 
  chksum    : 0xf44f
  Tos 0  metric: 4

<AR5>display ospf lsdb asbr

	 OSPF Process 1 with Router ID 5.5.5.5
		         Area: 0.0.0.0
		 Link State Database 

  Type      : Sum-Asbr
  Ls id     : 1.1.1.1
  Adv rtr   : 2.2.2.2  
  Ls age    : 435 
  Len       : 28 
  Options   :  E  
  seq#      : 80000004 
  chksum    : 0xf655
  Tos 0  metric: 1
```

上述两个 LSA4 中的 metric 分别是 4 和 1，**<font color="red">LSA4 中的 metric 是 ABR 到 ASBR 的成本</font>**。在 OSPF 的路由计算中，路由器在执行 Dijkstra（SPF）算法时，会基于全网同步的 LSDB 生成一张有向图，并严格以自身为根节点（Root）构建最短路径树。路径 cost 的累加方向与本路由器到目的地的数据转发方向一致。在每一跳上，使用的是当前节点在其 Router-LSA 中通告的链路输出 cost，也就是该节点把报文发往下一跳时所使用的出接口 cost。**因此，OSPF 计算 A 到 B 的 cost 时，累加的是 `A->...->B` 方向上各节点的出接口 cost，而不是反方向或接收端入接口 cost**。

因此 AR4 收到的 LSA4 中的 metric 是 4，表示 AR3 告诉 Area 2 内部路由器：如果你们要到 ASBR AR1，从我 AR3 这里看过去，cost 是 4。这是因为 AR3 到 AR1 的最优路径是 **`AR3->AR2->AR1`**，因此 cost 值累加为 4。对于 AR5 来说同理。

## 2.LSA5

### 2.1 LSA5 的报文格式

LSA5 报文中各个字段的含义如下所示：

- Ls id：引入的外部路由的网络号；
- Adv rtr：**<font color="red">Advertising Router，产生的 LSA5 的路由器 RouterID（在各个 OSPF 区域中都保持一致）</font>**；
- Net mask：引入的外部路由的掩码；
- Forwarding Address：可以是 0.0.0.0，也可以是非 0；**<font color="red">如果是 0.0.0.0，访问外部网络的报文转发给 ASBR，如果是非 0，报文转发给该非 0 地址</font>**；
- Tag：用于标记外部路由的标签，在路由引入时配置给外部路由，默认值是 1；
- Metric：**ASBR 到外部网络的成本**；
- Etype：Metric-type 可以是 1，也可以是 2，默认是 2。**Type1 和 Type2 的区别在路由表中可以看出来，Type2 路由仅考虑外部成本，Type1 路由考虑的是端到端的成本（内外成本之和）**。Type1 和 Type2 的另外一个区别是在外部路由的选路上的差别，详见下节；

AR5 和 AR4 收到的 LSA5 如下所示：

```java{.line-numbers}
<AR5>display ospf lsdb ase

	 OSPF Process 1 with Router ID 5.5.5.5
		 Link State Database

  Type      : External
  Ls id     : 10.1.6.0
  Adv rtr   : 1.1.1.1  
  Ls age    : 568 
  Len       : 36 
  Options   :  E  
  seq#      : 80000007 
  chksum    : 0x5e4e
  Net mask  : 255.255.255.0 
  TOS 0  Metric: 10 
  E type    : 2
  Forwarding Address : 0.0.0.0 
  Tag       : 1 
  Priority  : Low

<AR4>display ospf lsdb ase

	 OSPF Process 1 with Router ID 4.4.4.4
		 Link State Database

  Type      : External
  Ls id     : 10.1.6.0
  Adv rtr   : 1.1.1.1  
  Ls age    : 640 
  Len       : 36 
  Options   :  E  
  seq#      : 80000007 
  chksum    : 0x5e4e
  Net mask  : 255.255.255.0 
  TOS 0  Metric: 10 
  E type    : 2
  Forwarding Address : 0.0.0.0 
  Tag       : 1 
  Priority  : Low
```

在 LSA5 中，metric 值永远表示的是 ASBR 到外部网络的 cost，即外部 cost。因此 AR4 和 AR5 收到的 LSA5 中 metric 都是 10，因为在 AR1 上引入外部 RIP 路由时，配置的 cost 就是 10。

```java{.line-numbers}
ospf 1 router-id 1.1.1.1 
 import-route rip 1 cost 10 type 2 route-policy RIP_TO_OSPF
```

但是在路由表中，如果 Metric-type 为 Type2，那么路由表中显示的 cost 就是外部 cost。AR5 上显示的到外部 RIP 网段的 cost 就是 10，也就是外部 cost 为 10。

```java{.line-numbers}
<AR4>display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 12       Routes : 12       

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

       10.1.6.0/24  O_ASE   150  10          D   10.1.34.3       GigabitEthernet0/0/0
```

如果 Metric-type 为 Type1，那么路由表中显示的 cost 就是外部 cost 与内部 cost 之和。首先改变 AR1 上引入外部路由的 Metric-type 为 Type1，接着查看 AR5 的路由表，**<font color="red">如果 Forwarding Address 是 **`0.0.0.0`**，那么 AR4 要先算到 ASBR AR1 的内部 cost，否则如果 Forwarding Address 是非 0 的地址，那么 AR4 就要算到 FA 地址的内部 cost</font>**。这里 AR4 收到的 LSA5 中 Forwarding Address 是 0，因此 AR4 到 ASBR AR1 的内部 cost 是 **`1+3+1=5`**，外部 cost 是 10，因此 AR4 到外部网络的总 cost 就是 15。

```java{.line-numbers}
[AR1-ospf-1]import-route rip 1 cost 10 type 1 route-policy RIP_TO_OSPF
<AR4>display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 12       Routes : 12       

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

       10.1.6.0/24  O_ASE   150  15          D   10.1.34.3       GigabitEthernet0/0/0
```

### 2.2 LSA5 的作用

LSA5 区别于 LSA3/LSA4，LSA5 仅负责通告 OSPF 路由域外其他协议的路由，如 RIP、BGP 等。引入到 OSPF 后，这些外部路由靠 LSA5 将其泛洪到整个 OSPF 路由域。LSA5 具有其他 LSA 所没有的泛洪范围，**<font color="red">LSA5 能够泛洪到所有 Area，除了特殊类型区域（Stub 及 NSSA）</font>**。在上面的拓扑图中，LSA5 由 AR1 产生，在区域间泛洪至 Area2，泛洪期间仅 Age 会增加，其他都没有变化。

LSA5 的作用是除了向路由域中路由器通告外部路由外，还告知其他路由器如何访问该外部网络。**根据 LSA5 中的 FA（Forwarding Address）地址决定访问外部网络是经过 ASBR 还是经过拥有 FA 地址（非 0）的路由器**。

## 3.Forwarding Address 的作用

Forwarding-Address，简称 FA，仅出现在 LSA5 或 LSA7 中，**它是数据包访问外部网络时，在数据报文离开 OSPF 路由域时必须经过的设备地址**。这里仅介绍 LSA5 中的 FA，LSA5 携带外部路由，该外部路由一定要出现在路由表中，数据包才能访问到该外部目的地。**<font color="red">而外部路由能否出现在路由表中，则要依赖于 LSA5 的 FA 的可达性，如果 FA 不可达，则 LSA5 所通告的外部路由不进路由表</font>**（FA 不可达，LSA5 路由进路由表没有意义）。FA 地址可以是全 0，也可以是非 0。

若 **`FA=0`**，数据包要经过 ASBR 访问外部网络。如果 **`FA!=0`**，数据包要转发至拥有 FA 地址的路由设备，再由其转发到外部网络。

华为实现中，如果 **`FA=0`**，LSA5 要判断如何到 ASBR，继而决定该外部路由能否进 IP 路由表。**如果 ASBR 在其他区域，则依赖于 LSA4 来决定如何到达 ASBR。如果 ABSR 在当前区域，则依赖于 LSA1/LSA2 计算到 ASBR 的路径**。

如果 **`FA!=0`**，则要根据 OSPF 路由表（**`display ospf routing`**）中是否有 FA 地址所对应的路由来判断可达性。若不可达，则该外部路由不进 IP 路由表。

### 3.1 FA 是非 0 地址的场景

<div align="center">
    <img src="ospf_static/108.png" width="750"/>
</div>

在上述拓扑图中，AR1 和 AR6 处在两个 AS 中，AR1 和 AR6 之间运行 RIP 协议，AR1 向 OSPF 区域中通告 **`10.1.6.0/24`** 的 RIP 路由，AR1 收到该 BGP 路由，并把它放到全局路由表中。

```java{.line-numbers}
<AR2>display ospf lsdb 

	 OSPF Process 1 with Router ID 2.2.2.2
		 Link State Database 

		         Area: 0.0.0.0
 Type      LinkState ID    AdvRouter          Age  Len   Sequence   Metric
 Router    2.2.2.2         2.2.2.2            747  48    80000013       7
 Router    5.5.5.5         5.5.5.5           1307  48    80000010       9
 Router    3.3.3.3         3.3.3.3           1327  48    80000010       3
 Network   10.1.23.2       2.2.2.2            520  32    8000000A       0
 Network   10.1.35.5       5.5.5.5            515  32    8000000A       0
 Network   10.1.25.2       2.2.2.2            630  32    8000000A       0
 Sum-Net   10.1.34.0       3.3.3.3            502  28    80000009       3
 Sum-Net   10.1.12.0       2.2.2.2            738  28    80000009       1
 Sum-Net   10.1.16.0       2.2.2.2            107  28    80000001       2
 Sum-Asbr  1.1.1.1         2.2.2.2            738  28    80000009       1
 
		         Area: 0.0.0.1
 Type      LinkState ID    AdvRouter          Age  Len   Sequence   Metric
 Router    2.2.2.2         2.2.2.2            744  36    8000000E       1
 Router    1.1.1.1         1.1.1.1             66  48    80000012       1
 Network   10.1.12.1       1.1.1.1            742  32    8000000A       0
 Sum-Net   10.1.23.0       2.2.2.2            717  28    8000000A       7
 Sum-Net   10.1.35.0       2.2.2.2            747  28    80000009       8
 Sum-Net   10.1.25.0       2.2.2.2            747  28    80000009      11
 Sum-Net   10.1.34.0       2.2.2.2            747  28    80000009      10

		 AS External Database
 Type      LinkState ID    AdvRouter          Age  Len   Sequence   Metric
 External  10.1.6.0        1.1.1.1            108  36    8000000D      10
<AR2>display ospf lsdb ase

	 OSPF Process 1 with Router ID 2.2.2.2
		 Link State Database

  Type      : External
  Ls id     : 10.1.6.0
  Adv rtr   : 1.1.1.1  
  Ls age    : 17 
  Len       : 36 
  Options   :  E  
  seq#      : 8000000d 
  chksum    : 0xf193
  Net mask  : 255.255.255.0 
  TOS 0  Metric: 10 
  E type    : 2
  Forwarding Address : 10.1.16.6        // AR2 收到的 LSA5 中 FA 地址是外部 RIP 路由的下一跳地址 
  Tag       : 1 
  Priority  : Low
<AR2>display ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 17       Routes : 17       

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

       10.1.6.0/24  O_ASE   150  10          D   10.1.12.1       GigabitEthernet0/0/0      // 去往外部 RIP 路由的下一跳等于去往 FA 地址的下一跳
      10.1.16.0/24  OSPF    10   2           D   10.1.12.1       GigabitEthernet0/0/0
```

从上面的显示可以看出，AR2 收到的 LSA5 中 FA 地址是 ASBR 上外部 RIP 路由的下一跳地址，**<font color="red">并且在 AR2 的路由表中去往外部 RIP 路由的下一跳等于去往 FA 地址的下一跳</font>**。ASBR 上的接口如果满足以下四个规则，则 ASBR 上外部路由的下一跳地址就是该外部路由 LSA5 的 FA，否则该外部路由 LSA5 中的 FA 为 0。

