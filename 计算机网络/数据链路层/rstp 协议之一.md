# RSTP 协议

## 1.RSTP 介绍

IEEE 802.1w 中定义的 RSTP（Rapid Spanning Tree Protocol，快速生成树协议）可以视为 STP 的改进版本，RSTP 在许多方面对 STP 进行了优化，它的收敛速度更快，而且能够兼容 STP。

RSTP 引入了新的接口角色，其中替代接口的引入使得交换机在根接口失效时，能够立即获得新的路径到达根桥。**<font color="red">RSTP 引入了 P/A 机制，使得指定接口被选举产生后能够快速地进入转发状态，而不用像 STP 那样经历转发延迟时间</font>**。另外，RSTP 还引入了边缘接口的概念，这使得交换机连接终端设备的接口在初始化之后能够立即进入转发状态，提高了工作效率。

## 2.RSTP 接口角色

RSTP 在 STP 的基础上，增加了两种接口角色，它们是替代（Alternate）接口和备份（Backup）接口。因此，在 RSTP 中，共有 4 种接口角色：根接口、指定接口、替代接口和备份接口。

### 2.1 替代接口

替代接口可以简单地理解为根接口的备份，它是一台设备上，由于收到了其他设备所发送的 BPDU 从而被阻塞的接口。如果设备的根接口发生故障，那么替代接口可以成为新的根接口，这加快了网络的收敛过程。

在下图，SW1 是网络中的根桥，对于 SW3 而言，它有两个接口接入了该网络，由于从 **`GE0/0/22`** 接口到达根桥的 RPC 更小，因此该接口成为该设备的根接口。而 **`GE0/0/23`** 则由于收到了 SW2 所发送的 BPDU，并且经 SW3 计算后决定阻塞，成为该设备的替代接口。

<div align="center">
    <img src="rstp_static//2.png" width="350"/>
</div>

>AP 可以理解为端口收到对方更优质的 BPDU，但是本端口不能成为 RP，本端口就是 AP。这条端口所在的链路上，对端给出的 BPDU 是更优，说明在这条链路/这个网段上，对端是 Designated（指定端口那一侧），这端不是 Designated。但在这台桥上做 Root Port 选择时，还有另一条端口提供了更好的到 Root 的路径，所以这条端口不会被选成 Root Port（RP），就变成 Alternate（备选的根端口），通常处于 Discarding（阻塞）状态，随时待命。

此时在 SW3 上执行 **`display stp brief`** 命令能看到如下输出：

```java{.line-numbers}
<SW3>display stp brief
MSTID  Port                     Role   STP State   Protection
0      GigabitEthernet0/0/22    ROOT   FORWARDING  NONE
0      GigabitEthernet0/0/23    ALTE   DISCARDING  NONE
```

**<font color="red">一台设备如果是非根桥，那么它有且只能有一个根接口，但是该设备可以没有替代接口，也可以有</font>**，当存在替代接口时，可存在一个或多个。当设备的根接口发生故障时，最优的替代接口将成为新的根接口。

### 2.2 备份接口

备份接口是一台设备上由于收到了自己所发送的 BPDU 从而被阻塞的接口。**<font color="red">如果一台交换机拥有多个接口接入同一个网段，并且在这些接口中有一个被选举为该网段的指定接口，那么这些接口中的其他接口将被选举为备份接口</font>**。备份接口将作为该网段到达根桥的冗余接口。通常情况下，备份接口处于丢弃状态。

```c{.line-numbers}
if(rstpCompareBridgeAddr(addr, &context->bridgeId.addr) == 0 && rstpGetBridgePort(context, portId) != NULL) {
    //selectedRole is set to BackupPort and updtInfo is reset
    port->selectedRole = STP_PORT_ROLE_BACKUP;
    port->updtInfo = FALSE;
// 否则本端口就是 Alternate Port 了    
}
```

从上面的代码可以看出，如果收到的 **`portPriority/BPDU`** 里宣告的 **`designated bridge`** 地址等于本桥自己的桥地址，也就是 **`designatedBridge`** 是我自己；如果收到的 **`portPriority/BPDU`** 里宣告的 **`designatedPortId`**，确实对应本桥上的某个端口，即 **`designatedPort`** 是我桥上的另一个端口。**<font color="red">这正是 RSTP 里 Backup Port 的语义，同一台桥在同一个 LAN 段上有两个端口，其中一个端口是 DP 端口，另一个端口作为 DP 的备份而阻塞</font>**。

如下图所示，SW1 是根桥。对于 SW2，**`GE0/0/20`** 和 **`GE0/0/21`** 两个接口构成了环路。RSTP 能够检测到该环路，并在两个接口中选择一个将其阻塞。在没有其他配置的情况下，接口 ID 较小的接口将成为指定接口。在该例中，**`GE0/0/20`** 的接口 ID 小于 **`GE0/0/21`**，因此 **`GE0/0/20`** 成为指定接口，而 **`GE0/0/21`** 将成为备份端口，即被阻塞。

<div align="center">
    <img src="rstp_static//3.png" width="350"/>
</div>

此时，在 SW2 上执行 **`display stp brief`** 命令应该能看出如下状态：

```java{.line-numbers}
[SW2]display stp brief
 MSTID  Port                        Role  STP State     Protection
     0  GigabitEthernet0/0/24       ROOT  FORWARDING      NONE
     0  GigabitEthernet0/0/20       DESI  FORWARDING      NONE
     0  GigabitEthernet0/0/21       BACK  DISCARDING      NONE
```

在 Cyclone RSTP 的端口角色选择过程中，当某个端口当前的信息状态是 **`infoIs == RECEIVED`** 并且该端口又不是 RP 时，Cyclone RSTP 协议实现中会先把如果我把自己当作该链路上的指定端口时，我应当宣称出来的那套优先级向量构造成 **`designatedPriority`**，同时该端口也已经保存了一份从对端 BPDU 记录下来的 **`portPriority`**。

接下来 Cyclone 通过 **`rstpComparePriority(designatedPriority, portPriority)`** 对这两套优先级向量做严格比较：如果比较结果大于 0，含义就是我这套（以我的端口 ID 作为 **`designatedPortId`** 的宣称）整体更优，那么端口就会被直接赋予 **`selectedRole = DESIGNATED`**，即由该端口承担指定端口职责、向该段发送更优 BPDU。反之，如果比较结果不大于 0，说明本端口提出的 **`designatedPriority`** 并不优于它所收到并记录的那套 **`portPriority`**，此时 Cyclone 还不会立刻把它定为普通的替代/阻塞角色，而是会继续做同桥绕回。

Cyclone RSTP 协议会检查收到的 **`portPriority.designatedBridgeId`** 是否等于本桥的 bridgeId，并进一步用 **`rstpGetBridgePort(context, designatedPortId)`** 去本桥的端口表中查询，如果这个 **`designatedPortId`** 在本桥确实能找到对应端口，就意味着对端 BPDU 里宣称的指定端口其实就是我这台桥上的另一个端口，换句话说，当前端口所面对的更优宣称并不是来自另一台交换设备，而是来自同一台交换机的另一个端口绕回来的信息。

因此在这种情况下，Cyclone 会把该端口明确选为 **`selectedRole = BACKUP`**（备份端口），从机制上保证：**<font color="red">当同一冲突域里存在两个来自同一桥的候选指定端口时，只允许优先级更优的一侧继续转发并担当指定端口，而另一侧必须退化为备份端口进入阻塞状态，从而切断环路并维持无环拓扑</font>**。具体源码如下：

```java{.line-numbers}
// 比较本端口应该宣告的 designatedPriority 与从对端收到并记录下来的 portPriority
// 如果本端应宣告的 designatedPriority 更优，就把该端口选为 Designated Port，updtInfo=TRUE，让 PIM 进入 UPDATE
// 把 portPriority/Times 刷新为 designatedPriority/Times，并准备发出新的 BPDU
if(rstpComparePriority(&port->designatedPriority, &port->portPriority) > 0) {
    port->selectedRole = STP_PORT_ROLE_DESIGNATED;
    port->updtInfo = TRUE;
} else {
    // 否则对端（或该段上其他桥）给出的信息不比我差，本端口不能成为 Designated，需要在 Backup/Alternate 之间做区分
    MacAddr *addr;
    uint16_t portId;

    // Retrieve the designated bridge and designated port components of the port priority vector
    addr = &port->portPriority.designatedBridgeId.addr;
    portId = port->portPriority.designatedPortId;

    // 如果收到的 portPriority 里宣告的 designated bridge 地址 == 本桥自己的桥地址/designatedBridge 是我自己
    // 收到的 portPriority 里宣告的 designatedPortId，确实对应本桥上的某个端口/designatedPort 是我桥上的另一个端口
    // 这正是 RSTP 里 Backup Port 的语义，同一台桥在同一个 LAN 段上有两个端口，其中一个端口是 designated，另一个端口作为 designated 的备份而阻塞
    if(rstpCompareBridgeAddr(addr, &context->bridgeId.addr) == 0 && rstpGetBridgePort(context, portId) != NULL) {
        //selectedRole is set to BackupPort and updtInfo is reset
        port->selectedRole = STP_PORT_ROLE_BACKUP;
        port->updtInfo = FALSE;
    // 否则本端口就是 Alternate Port 了    
    } else {
        // selectedRole is set to AlternatePort and updtInfo is reset
        port->selectedRole = STP_PORT_ROLE_ALTERNATE;
        port->updtInfo = FALSE;
    }
}
```

上面这个场景看起来并没有实际意义，毕竟通过一条链路进行自环看起来确实没有什么必要，这可能是由于人为的疏忽误接线造成的，更有意义的备份场景如下图所示。如下图所示，SW2 通过两个接口连接到了一台 Hub（集线器）。由于 Hub 不运行 STP 或 RSTP，它会把收到的所有数据都泛洪给除入接口以外的其他所有接口，因此，SW2 从 **`GE0/0/20`** 接口发出的 BPDU 会被集线器接收并发往 SW2 的 **`GE0/0/21`** 接口，反之亦然。

<div align="center">
    <img src="rstp_static//4.png" width="350"/>
</div>

### 2.3 RSTP 接口状态

STP 定义了 5 种接口状态：**`Disabled`**、**`Blocking`**、**`Listening`**、**`Learning`** 和 **`Forwarding`**。RSTP 把原来 STP 的 **`Disabled`**、**`Blocking`** 和 **`Listening`** 状态合并为 **`Discarding`** 状态。接口状态的对应关系如下表所示。
 
<div align="center">
    <img src="rstp_static//5.png" width="650"/>
</div>

从上表可以看出，RSTP 的接口状态规范把 STP 接口状态中的 **`Disabled`**、**`Blocking`** 和 **`Listening`** 状态都归结为 **`Discarding`** 状态。处于 **`Discarding`** 状态的接口，既不转发用户流量，也不学习 MAC 地址。

### 2.4 BPDU

RSTP 的配置 BPDU 被称为 RST BPDU，其格式与 STP 的配置 BPDU 格式非常相似，但 RSTP 对配置 BPDU 中的部分字段进行了修改。RST BPDU 的 **`Protocol Version ID`** 字段值为 **`0x02`**，**`BPDU Type`** 字段值为 **`0x02`**。RSTP 对配置 BPDU 修改最大的字段是 Flags 字段。

<div align="center">
    <img src="rstp_static//6.png" width="450"/>
</div>

**<font color="red">在 STP 的配置 BPDU 中，Flags 字段虽然定义了 8 位，但是只使用了最低位和最高位。在 RSTP 的 RST BPDU 中，其余的 6 位也被利用起来了，并在 RSTP 协议计算中发挥着重要的作用</font>**。RST BPDU 的格式如上图所示。

STP 只使用了该字段的最高及最低比特位，在 RST BPDU 中这两个比特位的定义及作用不变。另外，Agreement（同意）及 Proposal（提议）比特位用于 RSTP 的 P/A（**`Proposal/Agreement`**）机制，该机制大大地提升了 RSTP 的收敛速度。Port Role（接口角色）比特位的长度为 2bit，它用于标识该 RST BPDU 发送接口的接口角色：

- **`01`**:表示根接口
- **`10`**:表示替代接口
- **`11`**:表示指定接口
- **`00`**:则被保留使用

最后，Forwarding（转发）及 Learning（学习）比特位用于表示该 RST BPDU 发送接口的接口状态。RSTP 与 STP 不同，在网络稳定后，无论是根桥还是非根桥，都将周期性地发送 RST BPDU。也就是说对于非根桥而言，它们不用在根接口上收到 BPDU 之后，才被触发而产生自己的配置 BPDU，而是自主地、周期性发送 BPDU。在后续的内容中，除非特别强调，BPDU 指的都是 RST BPDU。

在端口侧，**<font color="red">PTI 计时器状态机会在 tick 时把所有非 0 的计时变量统一递减，其中就包括负责 BPDU 周期节拍的 helloWhen</font>**。当某端口的 helloWhen 被递减到 0 后，PTX 发送状态机在 IDLE 状态里会检测到 **`port->helloWhen == 0`**，从而切到 **`TRANSMIT_PERIODIC`**。而在 **`TRANSMIT_PERIODIC`** 的入状态动作中，只要该端口当前角色是 **`DESIGNATED`**，就会把 **`port->newInfo = TRUE`**，含义就是需要周期性发送一次 RST BPDU 信息。随后状态机回到 IDLE 时会用 **`port->helloWhen = rstpHelloTime(port)`** 重新装载 Hello 计时器，为下一轮周期做准备。而一旦 newInfo 被置位，IDLE 分支还会在发送限速条件满足时真正进入发送动作。因此综上所述，在 RSTP 协议中，无论根桥还是非根桥，都会周期性（helloWhen）发送 RST BPDU。

```c{.line-numbers}
switch (port->ptxState) {
    case RSTP_PTX_STATE_TRANSMIT_INIT:
    case RSTP_PTX_STATE_TRANSMIT_PERIODIC:
    case RSTP_PTX_STATE_TRANSMIT_CONFIG:
    case RSTP_PTX_STATE_TRANSMIT_TCN:
    case RSTP_PTX_STATE_TRANSMIT_RSTP:
        rstpPtxChangeState(port, RSTP_PTX_STATE_IDLE);
        break;

    case RSTP_PTX_STATE_IDLE:
        // selected : 端口角色/选择已稳定，updtInfo : 角色选择要求端口信息更新时，暂缓发送，避免发出未对齐信息
        if (port->selected && !port->updtInfo) {
            // 周期性发送触发：helloWhen 倒计时到 0
            // helloWhen 是 HelloTime 的倒计时。当它归零，表示到了至少该发一帧 BPDU 的周期点。标准明确要求至少每个 HelloTime 要发 1 个 BPDU。
            // 进入 TRANSMIT_PERIODIC 通常会装载 helloWhen=HelloTime，并可能置位 newInfo
            if (port->helloWhen == 0) {
                rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_PERIODIC);
            } else {
                // 事件触发发送 newInfo 表示需要发送 BPDU
                if (port->newInfo) {
                    // 发送速率限制：txCount 每发一个 BPDU 自增，达到 TxHoldCount 时必须延迟发送，避免 1 秒内发太多帧 BPDU。
                    if (port->txCount < rstpTxHoldCount(context)) {
                        // sendRstp 决定发送 BPDU 类型：
                        // TRUE  -> 发送 RST BPDU（RSTP）
                        // FALSE -> 走 STP 兼容：根端口发 TCN，指定端口发 Config
                        if (port->sendRstp) {
                            // 发送 RSTP BPDU
                            rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_RSTP);
                        } else {
                            // STP 兼容模式下，根据端口角色选择 BPDU 类型
                            // 代码省略....
                        }
                    } else {
                    }
                } else {
                }
            }
        } 
        break;
}
```

RSTP 在 BPDU 的处理上的另一点改进是对于次优（Inferior）BPDU 的处理。运行 STP 的交换机在每个接口上保存一份 BPDU，**<font color="red">对于根接口及非指定接口而言，交换机保存的是发送自上游交换机的 BPDU，而对于指定接口而言，交换机保存的是自己根据根接口的 BPDU 所计算出的 BPDU</font>**。

**<font color="red">如果接口收到一份 BPDU，而且该接口当前所保存的 BPDU 比接收的 BPDU 更优，那么后者对于前者而言，就是次优 BPDU</font>**。在 STP 中，当指定接口收到次优 BPDU 时，它将立即发送自己的 BPDU；而对于非指定接口，当其收到次优 BPDU 时，它将等待接口所保存的 BPDU 老化之后，再重新计算新的 BPDU，并将新的 BPDU 发送出去，这将导致非指定接口需要最长约 20s 的时间才能启动状态迁移。在 RSTP 中，无论接口的角色如何，只要接口收到次优 BPDU，便立即发送自己的 BPDU，这个变化使得 RSTP 的收敛更快。

在 **`rstpUpdtRolesTree()`** 函数里，当端口的 **`infoIs==RECEIVED`** 时，并且该端口又不是 RP 时，Cyclone RSTP 协议实现中会先把如果我把自己当作该链路上的指定端口时，我应当宣称出来的那套优先级向量构造成 **`designatedPriority`**，同时该端口也已经保存了一份从对端 BPDU 记录下来的 **`portPriority`**。接下来 Cyclone 通过 **`rstpComparePriority(designatedPriority, portPriority)`** 对这两套优先级向量做严格比较。

一旦 designatedPriority 更优，就会选择 **`selectedRole=DESIGNATED`**，并置 **`updtInfo=TRUE`**。随后进入 PIM，**`updtInfo=TRUE`** 条件会被消化为真正可发送的标志，当端口满足 **`selected && updtInfo`** 条件时，PIM 会转入 UPDATE 状态，并在入状态动作里完成一组关键赋值，用本端的 designatedPriority 覆盖 portPriority，用 designatedTimes 覆盖 portTimes，然后清掉 updtInfo、并把 **`newInfo=TRUE`** 置位。随后本桥就将本端口存储的更优的 BPDU 信息发送出去。Cyclone RSTP 协议的相关源代码如下所示：

```c{.line-numbers}
// rstpUpdtRolesTree 函数
if(rstpComparePriority(&port->designatedPriority, &port->portPriority) > 0) {
    port->selectedRole = STP_PORT_ROLE_DESIGNATED;
    port->updtInfo = TRUE;
}

// rstpPimFsm 函数
// 本轮全桥的角色/优先级计算已经提交完成 并且需要更新 portPriority 和 portTimes，UPDATE 就是把端口信息更新为本桥计算出的 designated 信息
// 第一步必须先检测 updtInfo，然后有必要的话及时更新 port 的 portPriority 和 portTimes，因为后续 rstpRcvInfo 函数需要根据 portPriority 和 portTimes 来判定类型
if (port->selected && port->updtInfo) {
    rstpPimChangeState(port, RSTP_PIM_STATE_UPDATE);
}

// rstpPimChangeState 函数
void rstpPimChangeState(RstpBridgePort *port, RstpPimState newState) {
    case RSTP_PIM_STATE_UPDATE:
        port->proposing = FALSE;
        port->proposed = FALSE;
        port->agreed = port->agreed && rstpBetterOrSameInfo(port, RSTP_INFO_IS_MINE);

#if defined(RSTP_PIM_WORKAROUND_1)
        //Errata
        if (port->forward) {
            port->agreed = port->sendRstp;
        }
#endif

        port->synced = port->synced && port->agreed;
        port->portPriority = port->designatedPriority;
        port->portTimes = port->designatedTimes;
        port->updtInfo = FALSE;
        // 只有 updtInfo 为 true 时，才需要将 infoIs 设置为 RSTP_INFO_IS_MINE
        // 因为这说明端口的 portPriority 是使用本桥的信息进行更新的，所以将 infoIs 设置为 RSTP_INFO_IS_MINE
        port->infoIs = RSTP_INFO_IS_MINE;
        port->newInfo = TRUE;
        break;
}

// rstpTxRstp 函数
void rstpTxRstp(RstpBridgePort *port) {
    // ===== 6.填优先级向量（designatedPriority）：我准备在这个端口上宣告的根与路径信息 =====
    // Root Identifier（Root Bridge ID = priority + MAC）
    bpdu.rootId.priority = htons(port->designatedPriority.rootBridgeId.priority);
    bpdu.rootId.addr = port->designatedPriority.rootBridgeId.addr;
    // Root Path Cost：到根的累计代价
    bpdu.rootPathCost = htonl(port->designatedPriority.rootPathCost);
    // Bridge Identifier：Designated Bridge ID
    bpdu.bridgeId.priority = htons(port->designatedPriority.designatedBridgeId.priority);
    bpdu.bridgeId.addr = port->designatedPriority.designatedBridgeId.addr;
    // Port Identifier：Designated Port ID
    bpdu.portId = htons(port->designatedPriority.designatedPortId);
}
```

### 2.5 边缘接口

我们已经知道，运行了 STP 的交换机，其接口在初始启动之后，首先会进入阻塞状态。如果该接口被选举为根接口或指定接口，那么它还需要经历侦听及学习状态，最终才能进入转发状态，也就是说，一个接口从初始启动之后到进入转发状态至少需要花费 约 30s 的时间。

对于交换机上连接到交换网络的接口而言，经历上述过程是必要的，毕竟该接口存在产生环路的风险，然而有些接口引发环路的风险是非常低的，例如交换机连接终端设备（PC 或服务器等）的接口，这些接口如果启动之后依然需要经历上述过程就太低效了，而且用户当然希望将一台 PC 接入交换机后 PC 能够立即连接到网络，而不用耗费时间去等待。

<div align="center">
    <img src="rstp_static//7.png" width="350"/>
</div>

可以将交换机的接口配置为边缘接口（Edge Port）来解决上述问题。如上图所示，交换机 SW2 的 **`GE0/0/1`**、**`GE0/0/2`** 及 **`GE0/0/3`** 均可被配置为边缘接口。**<font color="red">边缘接口既不参与生成树计算，当边缘接口被激活之后，它可以立即切换到转发状态并开始收发业务流量，而不用经历转发延迟时间</font>**，因此工作效率大大提升了。

在 Cyclone RSTP 的实现中，边缘接口之所以能够激活后立即进入转发，因为 PRT 状态机的 **`rstpPrtDesignatedPortFsm()`** 函数在推进 designated 端口角色时，把 operEdge 当作一种快速放行条件，从而绕过原本需要等待的定时器与握手约束。具体来说，当端口处于 designated 相关状态，准备进一步从同步/协商阶段推进到学习/转发阶段时，PRT 的判断条件为 **`(fdWhile == 0 || agreed || operEdge)`**。

这意味着在普通非边缘端口场景下，端口通常要么等待 fdWhile 归零，要么等待 agreed（P/A 协商完成）成立，才能进入学习或转发。但只要该端口是边缘端口（**`operEdge == TRUE`**），PRT 会立即允许端口直接进入 **`DESIGNATED_LEARN`**，从逻辑上跳过必须等待这一段。相关源代码如下：

```c{.line-numbers}
else if ((port->fdWhile == 0 || port->agreed || port->operEdge) && (port->rrWhile == 0 || !port->reRoot) && !port->sync) {
    // Cyclone 实现中区分了 learn/learning、forward/forwarding 两组变量，前者是端口请求进入该状态的标志，后者是端口实际处于该状态的标志
    // 1.当前端口还没有发起进入学习的请求，因此先切到 DESIGNATED_LEARN 去发请求并启动必要的 delay
    // 2.当前端口已经发起了学习请求，但还没有发起转发请求，因此切到 DESIGNATED_FORWARD 去发起转发请求
    if (!port->learn) {
        // Switch to DESIGNATED_LEARN state
        rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_LEARN);
    }
    else if (port->learn && !port->forward) {
        // Switch to DESIGNATED_FORWARD state
        rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_FORWARD);
    }
    else {
        // Just for sanity
    }
}
```

另外，边缘接口的关闭或激活并不会触发 RSTP 拓扑变更。在实际项目中，我们通常会把用于连接终端设备的接口配置为边缘接口。以 SW2 为例，将其 **`GE0/0/1`** 接口配置为边缘接口的命令如下：

```c{.line-numbers}
[SW2]interface GigabitEthernet 0/0/1
[SW2-GigabitEthernet0/0/1]stp edged-port enable
```

值得注意的是，由于人为疏忽，边缘接口也可能会被误接交换设备，一旦交换设备连接到边缘接口，那么便引入了环路隐患。**<font color="red">因此如果边缘接口连接了交换设备并且收到了 BPDU，则该接口立即变成一个普通的生成树接口</font>**，在这个过程中，可能引发网络中的 RSTP 重计算，从而对网络造成影响。

当端口收到一份合法 BPDU 后，协议栈 **`rstpPrxFsm()`** 函数会把 **`port->rcvdBpdu`** 置位，PRX 状态机在 **`DISCARD/RECEIVE`** 的条件判断里检测到 **`rcvdBpdu && portEnabled`** 就会进入 RECEIVE 状态。进入 RECEIVE 状态时，Cyclone 会执行一组收到 BPDU 后的标准动作，其中最关键的就是 **`port->operEdge = FALSE`**。同时还会清掉 rcvdBpdu、置 **`rcvdMsg = TRUE`** 并把 edgeDelayWhile 重新装载为 **`rstpMigrateTime()`**。这一步在语义上就等价于只要收到 BPDU，就不再把该端口当作边缘端口对待。相关源代码如下：

```c{.line-numbers}
void rstpPrxFsm(RstpBridgePort *port) {
    case RSTP_PRX_STATE_RECEIVE:
        if(port->rcvdBpdu && port->portEnabled && !port->rcvdMsg) {
          rstpPrxChangeState(port, RSTP_PRX_STATE_RECEIVE);
        }
        break;
}

void rstpPrxChangeState(RstpBridgePort *port, RstpPrxState newState) {
    case RSTP_PRX_STATE_RECEIVE:
        rstpUpdtBpduVersion(port);
        // 将边缘端口设置为 false，此时不再将该端口作为边缘端口
        port->operEdge = FALSE;
        port->rcvdBpdu = FALSE;
        port->rcvdMsg = TRUE;
        port->edgeDelayWhile = rstpMigrateTime(port->context);
        break;
}
```

一个接口被配置为边缘接口后，该接口依然会周期性地发送 BPDU，然而正如上文所述，边缘接口通常用于连接终端设备，BPDU 对于这些设备而言其实是多余的，此时可以在接口的配置视图下增加 **`stp bpdu-filter enable`** 命令，这条命令用于激活该接口的 BPDU 过滤功能。在交换机的接口上激活 BPDU 过滤功能 后，该接口将不再发送 BPDU，而当其收到 BPDU 时，也会直接忽略。

## 3.P/A 机制

### 3.1 ieee-802.1d 规范介绍

Port Role Transitions 状态机为了让一个 Designated Port（指定端口）从非转发快速进入 Forwarding，会用到 **`Proposal/Agreement`** 的消息交换，并依赖下面这些布尔变量：

- **`proposing`**：当一个端口是指定端口但还没处于 Forwarding 时，它可以置位 proposing，并在发出的 RST BPDU 里把 Proposal 位带出去，相当于表达我想快速转发，你能不能配合？对上图中从 **`BRIDGE A -> BRIDGE B`** 的竖向下箭头 Proposal，上方标着 proposing；
- **`proposed`**：当本端（BRIDGE B）收到一个 BPDU，里面声明对端这个口是 Designated role，并且 Proposal 位=1，就把本端对应端口的 proposed 置位，表示我收到了对端的提议。注意，如果当前还没准备好同意（agree 还没能置位），那么 proposed 会触发把本桥的其他端口都置 sync，也就是让全桥 B 先同步到安全状态。对应上图中在 BRIDGE B 顶部，靠近端口的虚线框里有 proposed；并且从那里指向 **`setSyncTree()`**，再分叉到 B 的其他端口；
- **`sync`**：sync 是把端口拉回安全态的请求位。只要 sync=1，端口就会被驱动去 Discarding——除非它是 Edge Port，或者它已经是 synced=1（已经处在安全/已同步状态）。对应上图中 BRIDGE B 内部每个端口小状态图里都有 **`synced -> sync -> Discard -> Discarding`** 的路径。
- **`synced`** ：synced 表示这个端口已经处在不会形成环路的安全状态。标准给的两个典型来源：端口已经在 Discarding 或这个端口已经拿到了 agreed。对应上图中看到每个端口里有一个 OR 汇合 Discarding 或 agreed 都能导向 synced。

<div align="center">
    <img src="rstp_static//1.png" width="550"/>
</div>

>这里解释一下第 2 个来源，这里最关键的点在于 agreed 并不是一个随便的标志位，它代表的是对端已经完成了必要的同步动作（例如把其余端口同步到安全态、满足 allSynced 等条件），并且通过 Agreement 明确授权本端端口可以无额外延迟进入转发。如果不把 agreed 视作 synced 的一种来源，就会出现逻辑上的反直觉：**<font color="red">端口刚刚通过 `Proposal/Agreement` 握手完成了我可以安全快速转发的证明，结果下一次 sync 触发的全桥同步又会把它强行拉回 Discarding，等于是把刚刚建立的安全确认与快速收敛成果全部推翻</font>**，不但浪费握手过程，还会让收敛退化回更慢、更保守的路径。

- **`agree`**：当且仅当本桥除了收到 Proposal 的那个端口之外，其他所有端口都 synced=1（即 allSynced），本桥才会在该端口置位 agree，并发送带 Agreement 位的 RST BPDU 给对端。agree 第一次置位时会触发发送 Agreement，并把 proposed 清掉（表示这个提议已经处理完/进入下一阶段）。对应上图中 BRIDGE B 中间有 allSynced 指向 agree，并且从 **`BRIDGE B -> BRIDGE A`** 有竖向上箭头 Agreement。
- **`agreed`**：当本端收到带 Agreement 位的 RST BPDU，且对端端口角色是 **`Root/Alternate/Backup`**，并且该 BPDU 的优先级不比本端更好（即不会因为更优 BPDU 导致角色又要重新选），就置位 agreed。

任何一个指定端口一旦转换到 Discarding 状态，就会向其相邻网桥请求许可，以便随后能够转换到 Forwarding 状态。其结果是在当前活动拓扑中形成的一个切口（cut）（可以理解为转发路径被临时切断/阻塞的边界）会从根桥方向向网络外缘逐步传播，直到它在最终稳定的活动拓扑中到达其应处的位置，或一直传播到网络边缘为止。

把端口切到 Discarding 不会带来数据环路风险；但切到 Forwarding 必须与该端口所处局部区域内其它端口的角色/状态保持一致——这个区域的边界由非 Forwarding 的端口，以及连接到没有其他桥的 LAN 的端口共同限定。当一个端口因为生成树信息变化而被分配为 Root 或 Designated 时，桥只有在满足以下条件之一时，才知道它可以进入 Forwarding：

- 已经过去了足够长的时间，使得新信息能传播到该区域内所有桥，并且任何矛盾信息也来得及返回/被收到；
- 该端口现在是 Root Port，并且本桥上任何曾经是 Root Port 的端口现在都不是、且在生成树信息传播到网络其它桥之前也不会成为 Forwarding 状态；
- 该端口是 Designated Port，它所连接的 LAN 上至多还有一台其它桥；且那台桥的端口状态要么与生成树信息一致、要么处于 Discarding；并且通过它们的 Forwarding 端口再连接出去的更远端桥，也同样满足“一致或 Discarding”；
- 该端口是 Edge Port；

### 3.2 