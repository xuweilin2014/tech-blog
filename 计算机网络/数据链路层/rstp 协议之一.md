# RSTP 协议

<div align="center">
    <img src="rstp_static//1.png" width="550"/>
</div>

Port Role Transitions 状态机为了让一个Designated Port（指定端口）从非转发快速进入 Forwarding，会用到 **`Proposal/Agreement`** 的消息交换，并依赖下面这些布尔变量：

- **`proposing`**：当一个端口是指定端口但还没处于 Forwarding 时，它可以置位 proposing，并在发出的 RST BPDU 里把 Proposal 位带出去，相当于表达我想快速转发，你能不能配合？对上图中从 **`BRIDGE A -> BRIDGE B`** 的竖向下箭头 Proposal，上方标着 proposing；
- **`proposed`**：当本端（BRIDGE B）收到一个 BPDU，里面声明对端这个口是 Designated role，并且 Proposal 位=1，就把本端对应端口的 proposed 置位，表示我收到了对端的提议。注意，如果当前还没准备好同意（agree 还没能置位），那么 proposed 会触发把本桥的其他端口都置 sync，也就是让全桥 B 先同步到安全状态。对应上图中在 BRIDGE B 顶部，靠近端口的虚线框里有 proposed；并且从那里指向 **`setSyncTree()`**，再分叉到 B 的其他端口；
- **`sync`**：sync 是把端口拉回安全态的请求位。只要 sync=1，端口就会被驱动去 Discarding——除非它是 Edge Port，或者它已经是 synced=1（已经处在安全/已同步状态）。对应上图中 BRIDGE B 内部每个端口小状态图里都有 **`synced -> sync -> Discard -> Discarding`** 的路径。
- **`synced`** ：synced 表示这个端口已经处在不会形成环路的安全状态。标准给的两个典型来源：端口已经在 Discarding 或这个端口已经拿到了 agreed。对应上图中看到每个端口里有一个 OR 汇合 Discarding 或 agreed 都能导向 synced。

>这里解释一下第 2 个来源，这里最关键的点在于 agreed 并不是一个随便的标志位，它代表的是对端已经完成了必要的同步动作（例如把其余端口同步到安全态、满足 allSynced 等条件），并且通过 Agreement 明确授权本端端口可以无额外延迟进入转发。如果不把 agreed 视作 synced 的一种来源，就会出现逻辑上的反直觉：**<font color="red">端口刚刚通过 `Proposal/Agreement` 握手完成了我可以安全快速转发的证明，结果下一次 sync 触发的全桥同步又会把它强行拉回 Discarding，等于是把刚刚建立的安全确认与快速收敛成果全部推翻</font>**，不但浪费握手过程，还会让收敛退化回更慢、更保守的路径。

- **`agree`**：当且仅当本桥除了收到 Proposal 的那个端口之外，其他所有端口都 synced=1（即 allSynced），本桥才会在该端口置位 agree，并发送带 Agreement 位的 RST BPDU 给对端。agree 第一次置位时会触发发送 Agreement，并把 proposed 清掉（表示这个提议已经处理完/进入下一阶段）。对应上图中 BRIDGE B 中间有 allSynced 指向 agree，并且从 **`BRIDGE B -> BRIDGE A`** 有竖向上箭头 Agreement。
- **`agreed`**：当本端收到带 Agreement 位的 RST BPDU，且对端端口角色是 **`Root/Alternate/Backup`**，并且该 BPDU 的优先级不比本端更好（即不会因为更优 BPDU 导致角色又要重新选），就置位 agreed。

任何一个指定端口一旦转换到 Discarding 状态，就会向其相邻网桥请求许可，以便随后能够转换到 Forwarding 状态。其结果是在当前活动拓扑中形成的一个切口（cut）（可以理解为转发路径被临时切断/阻塞的边界）会从根桥方向向网络外缘逐步传播，直到它在最终稳定的活动拓扑中到达其应处的位置，或一直传播到网络边缘为止。