# RSTP 协议实验

## 1.RSTP P/A 机制详解

<div align="center">
    <div align="center" style="color: #F14; font-size:13px; font-weight:bold">P/A机制实验</div>
    <img src="rstp_static//16.png" width="450"/>
</div>

SW1 的配置如下所示：

```java{.line-numbers}
sysname SW1
#
stp mode rstp
stp instance 0 priority 0
interface Vlanif1
#
interface MEth0/0/1
#
interface GigabitEthernet0/0/1
 shutdown
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
```

SW2 的配置如下所示：

```java{.line-numbers}
sysname SW2
#
stp mode rstp
stp instance 0 priority 4096
#
interface Vlanif1
#
interface GigabitEthernet0/0/1
 shutdown
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
#
interface GigabitEthernet0/0/2
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
#
interface GigabitEthernet0/0/3
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
```

SW3 的配置如下所示：

```java{.line-numbers}
#
sysname SW3
#
stp mode rstp
stp instance 0 priority 8192
#
interface Vlanif1
#
interface GigabitEthernet0/0/1
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
#
interface GigabitEthernet0/0/2
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
```

SW4 的配置如下所示：

```java{.line-numbers}
#
sysname SW4
#
stp mode rstp
stp instance 0 priority 12288
#
interface Vlanif1
#
interface GigabitEthernet0/0/1
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
#
interface GigabitEthernet0/0/2
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
```

SW5 的配置如下所示：

```java{.line-numbers}
#
sysname SW5
#
stp mode rstp
stp instance 0 priority 16384
#
interface Vlanif1
#
interface GigabitEthernet0/0/1
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
#
interface GigabitEthernet0/0/2
 port link-type access
 stp edged-port disable
 stp point-to-point force-true
```

### 1.1 SW1-SW2 之间的 P/A 机制

RSTP 的 P/A 机制用来缩短上游端口进入 Forwarding 的等待时间，且只适用于两台交换设备之间的 P2P 全双工链路。在上面首先将 SW1 的 **`G0/0/1`** 端口配置 shutdown，这样 SW2 变成临时根桥。等到网络稳定后再把 SW1 的 **`G0/0/1`** 端口设置为 **`undo shutdown`**，重新启动，SW1（优先级更低为 0）接入，触发拓扑变更，此时 **`SW1—SW2`** 之间先触发 P/A 机制。下图是对 SW2 的 **`G0/0/1`** 端口进行抓包的结果。

<div align="center">
    <img src="rstp_static//17.png" width="800"/>
</div>

在 0-13 秒的时候，SW1 的 **`G0/0/1`** 端口处于正常转发状态，在第 13 秒的时候，SW1 的 **`G0/0/1`** 端口被配置为 shutdown，SW2 变成临时根桥。在第 42 秒的时候，SW1 的 **`G0/0/1`** 端口被重新启动（设置为 **`undo shutdown`**），触发拓扑变更，**<font color="red">首先 SW1 和 SW2 之间相互发送 Proposal 消息，并且 SW1 和 SW2 的 **`G0/0/1`** 端口此时都是指定端口</font>**。

在第 42 秒的时候，此时 SW2 的 **`G0/0/1`** 端口变为 DP 端口，并且发送 Proposal 消息，消息的具体内容如下所示。可以看出，SW2 的 **`G0/0/1`** 端口在发送 Proposal 消息时，状态为 Discarding（Forwarding 和 Learning 状态都为 no），且 Proposal 消息的 **`Flags`** 字段中包含 **`Proposal`** 标志位，表示这是一个 Proposal 消息。SW2 发送的 Proposal 消息的 **`Port Role`** 字段为 Designated Port，表示 SW2 的 **`G0/0/1`** 端口目前的状态是指定端口。**<font color="red">并且发送的 Proposal 消息以 SW2 的 Bridge ID 作为 Root ID，也就是 SW2 认为自己是根桥，因此到根桥的路径成本为 0</font>**。

<div align="center">
    <img src="rstp_static//19.png" width="800"/>
</div>

在第 43 秒的时候，此时 SW1 的 **`G0/0/1`** 端口变为 DP 端口，并且发送 Proposal 消息，消息的具体内容如下所示。可以看出，SW1 的 **`G0/0/1`** 端口在发送 Proposal 消息时，状态为 Discarding（Forwarding 和 Learning 状态都为 no），且 Proposal 消息的 **`Flags`** 字段中包含 **`Proposal`** 标志位，表示这是一个 Proposal 消息。SW1 发送的 Proposal 消息的 **`Port Role`** 字段为 Designated Port，表示 SW1 的 **`G0/0/1`** 端口目前的状态是指定端口。并且发送的 Proposal 消息以 SW1 的 Bridge ID 作为 Root ID，也就是 SW1 认为自己是根桥，因此到根桥的路径成本为 0。

<div align="center">
    <img src="rstp_static//18.png" width="800"/>
</div>

根据 **`SW1/SW2`** 之间互相发送的 Proposal 消息中 **`RootBridgeId`**、**`RootPathCost`** 等字段的比较结果，SW2 将 SW1 确认为根桥，因此 SW2 的 **`G0/0/1`** 端口变为 RP 端口，并且发送 Agreement 消息，消息的具体内容如下所示。并且可以看到发送的 Agreement 消息以 SW1 的 Bridge ID 作为 Root ID，也就是 SW2 认为 SW1 是根桥，因此到根桥的路径成本变为 20000，此时 SW2 的 **`G0/0/1`** 端口进入 Forwarding 转发状态。

<div align="center">
    <img src="rstp_static//20.png" width="800"/>
</div>

此时，SW1 和 SW2 之间的 P/A 机制已经协商完成，SW1 的 **`G0/0/1`** 端口变为 DP 端口，SW2 的 **`G0/0/1`** 端口都变为 RP 端口，并且都进入 Forwarding 状态。在随后 SW1 发送的 RST 消息内容如下所示。

<div align="center">
    <img src="rstp_static//21.png" width="800"/>
</div>

### 1.2 SW2-SW4 之间的 P/A 机制

据前所述，首先将 SW1 的 **`G0/0/1`** 端口配置 shutdown，这样 SW2 变成临时根桥。等到网络稳定后再把 SW1 的 **`G0/0/1`** 端口设置为 **`undo shutdown`**，重新启动，SW1（优先级更低为 0）接入，触发拓扑变更。下图是对 SW2 的 **`G0/0/3`** 端口进行抓包的结果。

<div align="center">
    <img src="rstp_static//22.png" width="800"/>
</div>

在第 0-22 秒的时候，SW1 的 **`G0/0/1`** 端口处于正常转发状态，在第 24 秒的时候，SW1 的 **`G0/0/1`** 端口被配置为 shutdown，SW2 变成临时根桥。根据前面的结论，**<font color="red">任何一台交换机的根端口消失，同时没有 AP 端口，此时交换机会置其他所有端口为 DP 端口角色，并产生自己的 BPDU</font>**。因此当 SW1-SW2 之间的链路断开时，SW2 的 **`G0/0/3`** 端口会发送 Proposal 消息，消息的具体内容如下所示。SW2 发送的 Proposal 消息的 **`Port Role`** 字段为 Designated Port，表示 SW2 的 **`G0/0/3`** 端口目前的状态是指定端口，并且发送的 Proposal 消息以 SW2 的 Bridge ID 作为 Root ID，也就是 SW2 认为自己是根桥，因此到根桥的路径成本为 0（也就是产生自己的 BPDU）。

<div align="center">
    <img src="rstp_static//23.png" width="800"/>
</div>

随后，SW4 的 **`G0/0/1`** 端口（此端口的状态为 RP）收到 SW2 发送的 Proposal 消息后，发送 Agreement 消息，消息的具体内容如下所示。并且可以看到发送的 Agreement 消息以 SW2 的 Bridge ID 作为 Root ID，也就是 SW4 认为 SW2 是根桥，因此到根桥的路径成本变为 20000，此时 SW4 的 **`G0/0/1`** 端口进入 Forwarding 转发状态，此时 SW2-SW4 之间的 P/A 机制协商完成，SW2 的 **`G0/0/3`** 端口进入 **`agreed`** 状态。

<div align="center">
    <img src="rstp_static//24.png" width="800"/>
</div>

当本端（BRIDGE）收到一个 BPDU，里面声明对端这个口是 Designated role，并且 Proposal 位=1，就把本端对应端口的 proposed 置位，表示我收到了对端的提议。然后就进入同步阶段，**<font color="red">已经处于丢弃状态或者处于 agreed 状态的接口缺省即已完成同步，而边缘接口则不参与该过程</font>**，除此之外，交换机处于转发状态的指定接口需切换到丢弃状态以便完成同步。因此由于 SW2 的 **`G0/0/3`** 端口经过 P/A 协商后进入 **`agreed`** 状态，当后续再恢复 SW1-SW2 之间的链路时，SW2 的 **`G0/0/3`** 端口不需要再进行 P/A 协商，一直保持 Forwarding 状态。下图是对 SW2 的 **`G0/0/3`** 端口进行抓包的结果。

