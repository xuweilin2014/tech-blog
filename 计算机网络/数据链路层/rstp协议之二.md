# RSTP 协议

## 一、rstp_prt.c

```c{.line-numbers}
// rstp_prt.c
void rstpPrtInit(RstpBridgePort *port) {
    //Enter initial state
    rstpPrtDisabledPortChangeState(port, RSTP_PRT_STATE_INIT_PORT);
}

void rstpPrtDisabledPortChangeState(RstpBridgePort *port, RstpPrtState newState) {
    //Switch to the new state
    port->prtState = newState;

    //On entry to a state, the procedures defined for the state are executed
    //exactly once (refer to IEEE Std 802.1D-2004, section 17.16)
    switch(port->prtState) {
    case RSTP_PRT_STATE_INIT_PORT:
        //Initialize variables
        port->role = STP_PORT_ROLE_DISABLED;
        port->learn = FALSE;
        port->forward = FALSE;
        port->synced = FALSE;
        port->sync = TRUE;
        port->reRoot = TRUE;
        port->rrWhile = rstpFwdDelay(port);
        port->fdWhile = rstpMaxAge(port);
        port->rbWhile = 0;
        break;

    case RSTP_PRT_STATE_DISABLE_PORT:
        //Update port role
        port->role = port->selectedRole;
        port->learn = FALSE;
        port->forward = FALSE;
        break;

    //DISABLED_PORT state?
    case RSTP_PRT_STATE_DISABLED_PORT:
        //Reload the Forward Delay timer
        port->fdWhile = rstpMaxAge(port);
        port->synced = TRUE;
        port->rrWhile = 0;
        port->sync = FALSE;
        port->reRoot = FALSE;
        break;

    //Invalid state?
    default:
        break;
    }

    port->context->busy = TRUE;
}

// PRT（Port Role Transition） 状态机 - Designated Port 状态处理函数
void rstpPrtDesignatedPortFsm(RstpBridgePort *port) {

    // All conditions for the current state are evaluated continuously until one
    // of the conditions is met (refer to IEEE Std 802.1D-2004, section 17.16)
    // 该状态机是 RSTP 端口角色转换状态机，这里的 port->prtState 并不是端口转发状态（Discard/Learn/Forward），端口真正的转发状态由端口状态转换状态机（Port State Transition FSM）负责。
    // 
    switch (port->prtState) {
        // DESIGNATED_PROPOSE, DESIGNATED_SYNCED, DESIGNATED_RETIRED,
        // DESIGNATED_FORWARD, DESIGNATED_LEARN or DESIGNATED_DISCARD state?
        case RSTP_PRT_STATE_DESIGNATED_PROPOSE:
        case RSTP_PRT_STATE_DESIGNATED_SYNCED:
        case RSTP_PRT_STATE_DESIGNATED_RETIRED:
        case RSTP_PRT_STATE_DESIGNATED_FORWARD:
        case RSTP_PRT_STATE_DESIGNATED_LEARN:
        case RSTP_PRT_STATE_DESIGNATED_DISCARD:
            // Unconditional transition (UCT) to DESIGNATED_PORT state
            rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_PORT);
            break;

        // DESIGNATED_PORT state?
        case RSTP_PRT_STATE_DESIGNATED_PORT:
            // All transitions, except UCT, are qualified by the following condition
            if (port->selected && !port->updtInfo) {
                // Evaluate conditions for the current state
                /*
                 * 1) DESIGNATED_PORT -> DESIGNATED_PROPOSE
                 * 若该端口不是边缘端口，并且当前还未转发，且尚未与对端达成 agreement，同时自己也不处于 proposing 状态则启动 Proposal/Agreement 过程（快速收敛握手）。
                 */
                if (!port->forward && !port->agreed && !port->proposing && !port->operEdge) {
                    // Switch to DESIGNATED_PROPOSE state
                    rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_PROPOSE);
                }
                // 1.端口既不在 Learning，也不在 Forwarding，也就是说实际端口状态是 Discarding，可以将 synced 置位，处于 Discarding 的端口不会转发数据帧，本身就是不会形成临时环路的安全状态，因此可以把它标记为 synced，供快速收敛逻辑使用
                // 2.这个端口已经收到了邻桥对它的 Agreement（agreed=1）
                // 3.这个端口是边缘端口（operEdge=1），边缘端口本身不会形成环路，因此可以直接置位 synced
                // 4.端口收到了 sync 请求，并且已经处于 synced 状态，这时进入 synced 状态会把 sync 清掉，否则后面很多进入 Learning/Forwarding 的迁移条件都会被 !sync 卡住
                else if ((!port->learning && !port->forwarding && !port->synced) || (port->agreed && !port->synced) ||
                         (port->operEdge && !port->synced) ||
                         (port->sync && port->synced)) {
                    // Switch to DESIGNATED_SYNCED state
                    rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_SYNCED);
                }
                else if (port->rrWhile == 0 && port->reRoot) {
                    // Switch to DESIGNATED_RETIRED state
                    rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_RETIRED);
                }
                // 1.上层要求同步（比如调用了 rstpSetSyncTree() 函数将本桥的所有端口设置为 sync=1），并且当前端口还没处于 synced 状态，即未同步完成
                // 2.端口为非边缘口
                // 3.端口正要学习或转发数据帧，必须拉回 discarding 状态以阻断潜在环路
                // 只要出现必须先确保拓扑安全的事件（sync 未完成），并且该端口不是 edge 口（edge 口不存在形成环路的可能），而且端口正要学习或转发，那么必须先进入丢弃（Discarding）以阻断潜在环路。
                else if (((port->sync && !port->synced) || (port->reRoot && port->rrWhile != 0) || port->disputed) && !port->operEdge &&
                         (port->learn || port->forward)) {
                    // Switch to DESIGNATED_DISCARD state
                    rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_DISCARD);
                }
                // 1.fdWhile 表示 Forward Delay 计时器，在经典 STP 里，端口需要等 ForwardDelay（15s）来完成，fdWhile == 0 表示定时器等待已结束，可以推进到学习/转发
                // 2.agreed 表示已经通过 P/A 握手，可以快速进入转发
                // 3.如果 port 是边缘端口，则可以直接进入学习/转发，这里3  个条件表示要么该等的时间已等完（fdWhile==0），要么通过 RSTP 机制可以不等（agreed / operEdge）
                // 4.!port->sync，表示当前没有同步请求，端口可以安全地进入学习/转发状态（在 DESIGNATED_SYNCED 状态入口动作里，Cyclone 会把 synced=TRUE 且 sync=FALSE，以避免反复被拉回 Discarding）
                else if ((port->fdWhile == 0 || port->agreed || port->operEdge) && (port->rrWhile == 0 || !port->reRoot) && !port->sync) {
                    // The Designated port can transition to Learning and to Forwarding
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
                else {
                    // Just for sanity
                }
            }
            break;
        // Invalid state?
        default:
            // Just for sanity
            rstpFsmError(port->context);
            break;
    }
}

void rstpPrtDesignatedPortChangeState(RstpBridgePort *port, RstpPrtState newState) {

    // 切换到新状态（注意：这是 PRT 状态机的状态，不是端口的转发状态）
    port->prtState = newState;

    // 进入某个状态时，需要执行该状态定义的入口动作（entry actions），并且只执行一次。
    switch (port->prtState)
    {
        case RSTP_PRT_STATE_DESIGNATED_PROPOSE:
            // DESIGNATED_PROPOSE：
            // 进入提议（Proposal）阶段：置 proposing，使后续发送的 RST BPDU 带 Proposal 语义；同时置 newInfo，推动 Port Transmit 状态机尽快发送 BPDU。
            port->proposing = TRUE;
            port->edgeDelayWhile = rstpEdgeDelay(port);
            port->newInfo = TRUE;
            break;

        case RSTP_PRT_STATE_DESIGNATED_SYNCED:
            // DESIGNATED_SYNCED：
            // 标记本端口达到"已同步/安全"的语义：
            // synced=TRUE，满足同步条件，并且将 sync 设置为 FALSE，清掉同步请求，避免反复被拉回 discard
            port->rrWhile = 0;
            port->synced = TRUE;
            port->sync = FALSE;
            break;

        case RSTP_PRT_STATE_DESIGNATED_RETIRED:
            port->reRoot = FALSE;
            break;

        case RSTP_PRT_STATE_DESIGNATED_FORWARD:
            // DESIGNATED_FORWARD：请求端口尽快进入 Forwarding，forward=TRUE 通知端口状态转换（PST）状态机进入转发，agreed=sendRstp 表示只有在发送 RSTP（具备快速握手语义）时才认为 agreed 可用。
            port->forward = TRUE;
            port->fdWhile = 0;
            port->agreed = port->sendRstp;
            break;

        case RSTP_PRT_STATE_DESIGNATED_LEARN:
            // DESIGNATED_LEARN：请求端口进入 Learning，并加载 forwardDelay 计时（传统先学后转发阶段）。
            port->learn = TRUE;
            port->fdWhile = rstpForwardDelay(port);
            break;

        case RSTP_PRT_STATE_DESIGNATED_DISCARD:
            // DESIGNATED_DISCARD：强制端口回到 Discarding（安全优先，避免临时环路），将 learn/forward 置 FALSE，明确不要学习/不要转发。
            port->learn = FALSE;
            port->forward = FALSE;
            port->disputed = FALSE;
            port->fdWhile = rstpForwardDelay(port);
            break;

        case RSTP_PRT_STATE_DESIGNATED_PORT:
            port->role = STP_PORT_ROLE_DESIGNATED;
            break;

        default:
            // 其他状态不在本函数负责的范围内（按实现习惯：这里不处理）
            break;
    }

    // 标记：RSTP 状态机发生过动作/很忙，上层调度通常会尽快再跑一轮以收敛
    port->context->busy = TRUE;
}

void rstpPrtFsm(RstpBridgePort *port) {
    // PRT（Port Role Transition）状态机的入口分发器
    // selectedRole 由 PRS（Port Role Selection）状态机计算更新
    // role 表示当前已经生效的端口角色（由 PRT 在进入某些状态时更新）

    // 如果 PRS 计算出的角色与当前生效角色不同，说明拓扑/角色发生了变化
    if (port->role != port->selectedRole) {
        // 根据新计算的角色选择要进入的角色起始状态
        switch (port->selectedRole) {
            case STP_PORT_ROLE_DISABLED:
                // 端口被禁用：进入 DISABLE_PORT 路径（不参与拓扑，数据面不转发）
                rstpPrtDisabledPortChangeState(port, RSTP_PRT_STATE_DISABLE_PORT);
                break;
            case STP_PORT_ROLE_ROOT:
                // 端口成为 Root Port：进入 ROOT_PORT 路径（后续可推进到 learn/forward）
                rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PORT);
                break;
            case STP_PORT_ROLE_DESIGNATED:
                // 端口成为 Designated Port：进入 DESIGNATED_PORT 路径（后续可 propose/sync/learn/forward）
                rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_PORT);
                break;
            case STP_PORT_ROLE_ALTERNATE:
            case STP_PORT_ROLE_BACKUP:
                // 端口成为 Alternate/Backup：这是非转发角色
                // 进入 BLOCK_PORT 路径（阻塞/丢弃），避免形成环路
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_BLOCK_PORT);
                break;
            default:
                // 防御式编程：出现未知角色，认为状态机进入异常
                rstpFsmError(port->context);
                break;
        }
    } else {
        // 角色未发生变化：继续执行当前角色对应的子状态机
        // 子状态机内部才会根据各种条件推进 prtState（例如 learn/forward/sync 等）
        switch (port->role) {
            case STP_PORT_ROLE_DISABLED:
                // Disabled 角色子状态机（例如 INIT_PORT/DISABLE_PORT 等）
                rstpPrtDisabledPortFsm(port);
                break;
            case STP_PORT_ROLE_ROOT:
                // Root 角色子状态机（例如 ROOT_PORT/ROOT_LEARN/ROOT_FORWARD 等）
                rstpPrtRootPortFsm(port);
                break;
            case STP_PORT_ROLE_DESIGNATED:
                // Designated 角色子状态机（例如 DESIGNATED_PORT/PROPOSE/SYNCED/LEARN/FORWARD 等）
                rstpPrtDesignatedPortFsm(port);
                break;
            case STP_PORT_ROLE_ALTERNATE:
            case STP_PORT_ROLE_BACKUP:
                // Alternate/Backup 角色子状态机（例如 BLOCK/ALTERNATE_PORT 等）
                rstpPrtAlternatePortFsm(port);
                break;
            default:
                // 防御式编程：出现未知角色，认为状态机进入异常
                rstpFsmError(port->context);
                break;
        }
    }
}

void rstpPrtRootPortFsm(RstpBridgePort *port) {
    // Root Port 角色的 PRT（Port Role Transition）状态机
    switch (port->prtState) {
        case RSTP_PRT_STATE_ROOT_PROPOSED:
        case RSTP_PRT_STATE_ROOT_AGREED:
        case RSTP_PRT_STATE_REROOT:
        case RSTP_PRT_STATE_ROOT_FORWARD:
        case RSTP_PRT_STATE_ROOT_LEARN:
        case RSTP_PRT_STATE_REROOTED:
            // UCT: Unconditional Transition
            // 无条件切回 ROOT_PORT（让 ROOT_PORT 做主判断）
            rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PORT);
            break;

        case RSTP_PRT_STATE_ROOT_PORT:
            // 除 UCT 之外的所有迁移，都要求端口已经被选择(selected)且没有处于更新信息(updtInfo)阶段
            if (port->selected && !port->updtInfo) {
                // 收到 Proposal（proposed=1），但还没有准备发 Agreement（agree=0），进入 ROOT_PROPOSED，触发全树 sync（通常会先阻断其它可能成环端口）
                if (port->proposed && !port->agree) {
                    rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PROPOSED);
                }
                // 两种情况可以进入 ROOT_AGREED（准备/需要发 Agreement）：
                // 1) 全树 allSynced 成立 && 我还没 agree
                // 2) proposed=1 且 agree=1，这是重复 proposal 的情况
                // 最终会导致 ROOT_AGREED entry 会置 agree=1 并置 newInfo=1 触发发 BPDU
                else if ((rstpAllSynced(port->context) && !port->agree) || (port->proposed && port->agree)) {
                    rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_AGREED);
                } else if (!port->forward && !port->reRoot) {
                    rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_REROOT);
                } else if (port->rrWhile != rstpFwdDelay(port)) {
                    rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PORT);
                } else if (port->reRoot && port->forward) {
                    rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_REROOTED);
                }
                // 满足可以推进到 Learning/Forwarding 的条件
                //  a) 常规路径：fdWhile==0（计时器到）
                //  b) 快速路径：reRooted 成立 && rbWhile==0 && 运行在 RSTP 版本
                //  允许 Root 口按 learn -> forward 的顺序推进
                else if (port->fdWhile == 0 || (rstpReRooted(port) && port->rbWhile == 0 && rstpVersion(port->context))) {
                    // The Root port can transition to Learning and to Forwarding
                    // 还没请求进入 Learning：先切 ROOT_LEARN（entry: learn=1, fdWhile=forwardDelay）
                    if (!port->learn) {
                        rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_LEARN);
                    }
                    // 已经 learn 了但还没请求进入 Forwarding：切 ROOT_FORWARD（entry: forward=1, fdWhile=0）
                    else if (port->learn && !port->forward) {
                        rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_FORWARD);
                    } else {
                        // Just for sanity
                    }
                } else {
                    // Just for sanity
                }
            }

            break;

        default:
            // 非法状态：状态机一致性保护
            rstpFsmError(port->context);
            break;
    }
}

void rstpPrtRootPortChangeState(RstpBridgePort *port, RstpPrtState newState) {
    // Root Port 角色的 PRT（Port Role Transition）状态机：切换状态并执行 entry action

    // 切换到新状态
    port->prtState = newState;
    // 进入新状态时执行一次 entry action
    switch (port->prtState) {
    case RSTP_PRT_STATE_ROOT_PROPOSED:
        // 收到 Proposal（同步请求）后：对全桥下发需要同步的命令，目的把除 Root Port 以外的端口同步到安全状态（常见是转 Discarding），再回 Agreement
        rstpSetSyncTree(port->context);
        // 消费 proposed 事件
        port->proposed = FALSE;     
        break;

    case RSTP_PRT_STATE_ROOT_AGREED:
        // 准备发送 Agreement：
        // - 清 proposed（proposal 已处理）
        // - 清 sync（本端口同步流程可结束）
        // - 置 agree（Agreement 语义）
        // - 置 newInfo（触发 PTX 去发送 RSTP BPDU）
        port->proposed = FALSE;
        port->sync = FALSE;
        port->agree = TRUE;
        port->newInfo = TRUE;
        break;

    case RSTP_PRT_STATE_REROOT:
        // 启动 reroot：对全桥端口置 reRoot 标志，触发重收敛流程
        rstpSetReRootTree(port->context);
        break;

    case RSTP_PRT_STATE_ROOT_FORWARD:
        // 请求 PST（Port State Transition 状态机）把端口推进到 Forwarding
        // fdWhile=0 表示不需要再等待 forwardDelay
        port->fdWhile = 0;
        port->forward = TRUE;
        break;

    case RSTP_PRT_STATE_ROOT_LEARN:
        // 请求 PST 把端口推进到 Learning
        // 装载 ForwardDelay 计时器，用于 Learning 阶段的延时控制
        port->fdWhile = rstpForwardDelay(port);
        port->learn = TRUE;
        break;

    case RSTP_PRT_STATE_REROOTED:
        // reroot 完成：清 reRoot 标志
        port->reRoot = FALSE;
        break;

    case RSTP_PRT_STATE_ROOT_PORT:
        // Root Port 稳态：更新端口角色，并刷新 rrWhile（Recent Root timer）
        port->role = STP_PORT_ROLE_ROOT;
        port->rrWhile = rstpFwdDelay(port);
        break;

    default:
        // 非法/未覆盖状态：保持“无动作”
        break;
    }

    // 通知全局调度器：本轮发生了状态变化，需要继续迭代推进其它状态机
    port->context->busy = TRUE;
}
```

## 二、rstp_fsm.c

```c{.line-numbers}
// rstp_fsm.c
void rstpFsmInit(RstpBridgeContext *context) {
    uint_t i;
    RstpBridgePort *port;

    //The first (RootBridgeID) and third (DesignatedBridgeID) components of
    //the bridge priority vector are both equal to the value of the Bridge
    //Identifier. The other components are zero
    context->bridgePriority.rootBridgeId = context->bridgeId;
    context->bridgePriority.rootPathCost = 0;
    context->bridgePriority.designatedBridgeId = context->bridgeId;
    context->bridgePriority.designatedPortId = 0;
    context->bridgePriority.bridgePortId = 0;

    //BridgeTimes comprises four components (the current values of Bridge Forward
    //Delay, Bridge Hello Time, and Bridge Max Age, and a Message Age of zero)
    context->bridgeTimes.forwardDelay = context->params.bridgeForwardDelay;
    context->bridgeTimes.helloTime = context->params.bridgeHelloTime;
    context->bridgeTimes.maxAge = context->params.bridgeMaxAge;
    context->bridgeTimes.messageAge = 0;

    //Initialize bridge's root priority vector
    context->rootPriority = context->bridgePriority;
    //Initialize bridge's rootTimes parameter
    context->rootTimes = context->bridgeTimes;

    //The value of the ageingTime parameter is normally Ageing Time
    context->ageingTime = context->params.ageingTime;
    //Reset rapid ageing timer
    context->rapidAgeingWhile = 0;

    //Restore default ageing time
    rstpUpdateAgeingTime(context, context->params.ageingTime);

    //Clear BPDU
    osMemset(&context->bpdu, 0, sizeof(RstpBpdu));

    //Loop through the ports of the bridge
    for(i = 0; i < context->numPorts; i++) {
        //Point to the current bridge port
        port = &context->ports[i];

        //The designatedTimes for each port is set equal to the value of rootTimes
        //except for the Hello Time component, which is set equal to BridgeTimes' Hello Time
        port->designatedTimes = context->rootTimes;
        port->designatedTimes.helloTime = context->bridgeTimes.helloTime;

        //Initialize msgPriority and msgTimes for each port
        memset(&port->msgPriority, 0, sizeof(RstpPriority));
        memset(&port->msgTimes, 0, sizeof(RstpTimes));

        //Reset parameters
        port->disputed = FALSE;
        port->rcvdInfo = RSTP_RCVD_INFO_OTHER;
        port->rcvdTc = FALSE;
        port->rcvdTcAck = FALSE;
        port->rcvdTcn = FALSE;
        port->tcProp = FALSE;
        port->updtInfo = FALSE;
    }

    //One instance of the Port Role Selection state machine is implemented for the bridge
    rstpPrsInit(context);

    //One instance of each of the other state machines is implemented per port
    for(i = 0; i < context->numPorts; i++) {
        //Point to the current bridge port
        port = &context->ports[i];

        //Initialize Port Timers state machine
        rstpPtiInit(port);
        //Initialize Port Receive state machine
        rstpPrxInit(port);
        //Initialize Port Protocol Migration state machine
        rstpPpmInit(port);
        //Initialize Bridge Detection state machine
        rstpBdmInit(port);
        //Initialize Port Transmit state machine
        rstpPtxInit(port);
        //Initialize Port Information state machine
        rstpPimInit(port);
        //Initialize Port Role Transition state machine
        rstpPrtInit(port);
        //Initialize Port State Transition state machine
        rstpPstInit(port);
        //Initialize Topology Change state machine
        rstpTcmInit(port);
    }

    //Update RSTP state machine
    rstpFsm(context);
}

/**
  * @brief RSTP 状态机主调度函数
  * @param context 指向 RSTP 桥接上下文结构体的指针
  * rstpFsm() 是整个 RSTP（802.1w/802.1D-2004）协议栈的总调度器/主循环：它按照 IEEE 802.1D-2004 规定的多个协作状态机模型，把每个状态机依次跑一遍；如果任何状态机在本轮执行中改变了变量（触发了状态迁移/入口动作），就把 context->busy 置为 TRUE，于是主循环会再跑一轮，直到全桥进入稳定（busy==FALSE）为止
  * 标准明确说明：RSTP 的行为由多个协作状态机组成；其中 Port Role Selection（PRS）每桥一份，其它状态机，每端口一份
  */
void rstpFsm(RstpBridgeContext *context)
{
    uint_t i;
    RstpBridgePort *port;
    // RSTP 的行为由多个协作状态机共同定义：
    // 标准在 17.15 明确了这种组织方式 
    do {
        // 每一轮开始先假设本轮不会再有变化
        // 如果任意状态机发生状态迁移/入口动作，就会把 context->busy 置 TRUE
        context->busy = FALSE;
        // 1) 每桥状态机：端口角色选择（为所有端口选 Root/Designated/Alternate/Backup/Disabled 等）
        // 其它 per-port 状态机大量依赖 PRS 的输出（selectedRole/selected/reselect 等）
        rstpPrsFsm(context);
        // 2) 每端口状态机：逐个端口跑一遍
        for (i = 0; i < context->numPorts; i++) {
            port = &context->ports[i];
            // (a) Port Timers：递减各类计时器（fdWhile/rrWhile/helloWhen/...）
            rstpPtiFsm(port);
            // (b) Port Receive：接收并解析 BPDU，置位 rcvdMsg/rcvdBpdu/rcvdRSTP 等
            rstpPrxFsm(port);
            // (c) Port Protocol Migration：STP/RSTP 兼容迁移，更新 sendRSTP
            // sendRSTP 用来告诉 Port Transmit 应该发哪种 BPDU
            rstpPpmFsm(port);
            // (d) Bridge Detection：边缘端口/点到点等判定，更新 operEdge 等
            rstpBdmFsm(port);
            // (e) Port Information：维护端口“我是谁/对端是谁”的信息与优先向量比较
            rstpPimFsm(port);
            // (f) Port Role Transition：根据 selectedRole/sync/synced/agreed 等驱动角色转换动作
            // 它通过 learn/forward 等请求变量去驱动 PST
            rstpPrtFsm(port);
            // (g) Port State Transition：把端口真实转发状态切到 Discarding/Learning/Forwarding
            // learning/forwarding 等实际状态变量在这里更新
            rstpPstFsm(port);
            // (h) Topology Change：检测/传播拓扑变化，并要求刷新过滤数据库（动态 MAC 表）
            rstpTcmFsm(port);
            // 如果 TCM 要求刷新该端口相关的动态过滤表项，则执行实际的 FDB 清理
            // 标准描述：Topology Change state machine 会指示 Filtering Database 删除某些端口的动态项
            if (port->fdbFlush) {
                rstpRemoveFdbEntries(port);
            }
        }

        // 3) 只有当这一轮所有状态机都不再产生变化（busy==FALSE）时，才进行发送侧处理
        // Port Transmit 会周期性发送 BPDU，或在 newInfo 置位时立即发送，并受 txCount/TXHoldCount 限速
        if (!context->busy) {
            for (i = 0; i < context->numPorts; i++) {
                port = &context->ports[i];
                // 更新每端口发送状态机：决定是否发送 RSTP/STP/TCN 等 BPDU
                rstpPtxFsm(port);
            }
        }
    // 标准的语义是：状态块执行完后，退出条件会被“连续评估直到满足某个条件”
    // 这里用“只要 busy 被置回 TRUE 就再跑一轮”来模拟这种持续评估，直到系统稳定
    } while (context->busy);
}
```

## 三、rstp_ptx.c

```c{.line-numbers}
// rtsp_ptx.c
// IEEE 802.1D-2004（含 RSTP 802.1w）里的 Port Transmit state machine（端口发送状态机），它决定什么时候在某个端口上发 BPDU，以及发哪一种 BPDU
// 定时发送 + 事件触发发送（newInfo）+ 发送速率限制（txCount/TxHoldCount）+ 由 sendRSTP 决定 BPDU 类型
void rstpPtxFsm(RstpBridgePort *port)
{
    RstpBridgeContext *context;

    // 取所属桥上下文（包含 TxHoldCount 等全局/桥级参数）
    context = port->context;

    if (!port->portEnabled) {
        rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_INIT);
    } else {
        switch (port->ptxState)
        {
            case RSTP_PTX_STATE_TRANSMIT_INIT:
            case RSTP_PTX_STATE_TRANSMIT_PERIODIC:
            case RSTP_PTX_STATE_TRANSMIT_CONFIG:
            case RSTP_PTX_STATE_TRANSMIT_TCN:
            case RSTP_PTX_STATE_TRANSMIT_RSTP:
                // UCT：无条件跳转到 IDLE
                rstpPtxChangeState(port, RSTP_PTX_STATE_IDLE);
                break;

            case RSTP_PTX_STATE_IDLE:
                // 门禁条件：除 UCT 外，所有转移都必须满足 selected && !updtInfo
                // selected : 端口角色/选择已稳定
                // updtInfo : 角色选择要求端口信息更新时，暂缓发送，避免发出未对齐信息
                if (port->selected && !port->updtInfo) {
                    // 周期性发送触发：helloWhen 倒计时到 0
                    // helloWhen 是 HelloTime 的倒计时。当它归零，表示到了至少该发一帧 BPDU 的周期点。标准明确要求至少每个 HelloTime 要发 1 个 BPDU。
                    // 进入 TRANSMIT_PERIODIC 通常会装载 helloWhen=HelloTime，并可能置位 newInfo
                    if (port->helloWhen == 0) {
                        rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_PERIODIC);
                    } else {
                        // 事件触发发送 newInfo 表示需要发送 BPDU
                        if (port->newInfo) {
                            // 发送速率限制：
                            // txCount 每发一个 BPDU 自增，达到 TxHoldCount 时必须延迟发送，避免 1 秒内发太多帧 BPDU。
                            if (port->txCount < rstpTxHoldCount(context)) {
                                // sendRstp 决定发送 BPDU 类型：
                                // TRUE  -> 发送 RST BPDU（RSTP）
                                // FALSE -> 走 STP 兼容：根端口发 TCN，指定端口发 Config
                                if (port->sendRstp) {
                                    // 发送 RSTP BPDU
                                    rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_RSTP);
                                } else {
                                    // STP 兼容模式下，根据端口角色选择 BPDU 类型
                                    if (port->role == STP_PORT_ROLE_ROOT) {
                                        // 根端口：向根方向发送 TCN（拓扑变化通知）
                                        rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_TCN);
                                    } else if (port->role == STP_PORT_ROLE_DESIGNATED) {
                                        // 指定端口：发送 Configuration BPDU（配置 BPDU）
                                        rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_CONFIG);
                                    } else {
                                        // 其它角色（Alternate/Backup 等）一般不应主动发这些 BPDU
                                    }
                                }
                            } else {
                                // 被 TxHoldCount 限速：本轮不发，等待 txCount 被计时器递减
                            }
                        } else {
                            // 没有新信息，不需要发送
                        }
                    }
                } else {
                    // 未 selected 或 updtInfo 仍为真：暂不发送
                }
                break;

            default:
                // 非法状态：健壮性处理
                rstpFsmError(port->context);
                break;
        }
    }
}

void rstpPtxChangeState(RstpBridgePort *port, RstpPtxState newState) {

    // 切换到新状态
    port->ptxState = newState;

    // 按 IEEE 802.1 状态机风格：进入状态时执行一次该状态的 entry action
    switch (port->ptxState) {
        case RSTP_PTX_STATE_TRANSMIT_INIT:
            // 初始化发送机：触发一次发送机会，并清零本周期发送计数
            port->newInfo = TRUE;
            port->txCount = 0;
            break;

        case RSTP_PTX_STATE_TRANSMIT_PERIODIC:
            // 周期性发送触发：
            // 1.Designated 口：总是需要周期性维持 BPDU
            // 2.Root 口：仅在 tcWhile 运行（拓扑变化传播窗口）时才触发
            if (port->role == STP_PORT_ROLE_DESIGNATED) {
                // 触发发送（后续在 IDLE 状态里决定发 Config/RSTP）
                port->newInfo = TRUE;
            } else if (port->role == STP_PORT_ROLE_ROOT) {
                if (port->tcWhile != 0) {
                    // 拓扑变化期间触发发送（可能走 TCN 或带 TC 的 BPDU）
                    port->newInfo = TRUE;
                }
            } else {
                // 其它角色一般不在这里触发周期发送
            }
            break;

        case RSTP_PTX_STATE_TRANSMIT_CONFIG:
            // 发送传统 STP Configuration BPDU
            // 发送前清 newInfo，避免重复触发；发送后计数 +1；清 tcAck 防止确认语义粘连
            port->newInfo = FALSE;
            rstpTxConfig(port);
            port->txCount++;
            port->tcAck = FALSE;
            break;

        case RSTP_PTX_STATE_TRANSMIT_TCN:
            // 发送 TCN BPDU（Topology Change Notification）
            port->newInfo = FALSE;
            rstpTxTcn(port);
            port->txCount++;
            break;

        case RSTP_PTX_STATE_TRANSMIT_RSTP:
            // 发送 RSTP BPDU（携带 RSTP flags/role 等信息）
            // 同样：发送后计数 +1，并清 tcAck
            port->newInfo = FALSE;
            rstpTxRstp(port);
            port->txCount++;
            port->tcAck = FALSE;
            break;

        case RSTP_PTX_STATE_IDLE:
            // 回到空闲态：重装 Hello 计时器，等待 tick 递减到 0 再触发 periodic
            port->helloWhen = rstpHelloTime(port);
            break;

        default:
            // 非法状态：通常不应发生
            break;
    }

    // 只要端口启用，PTX 有动作就应驱动整个 RSTP 多状态机迭代继续运行
    if (port->portEnabled) {
        port->context->busy = TRUE;
    }
}
```

## 四、rstp_procedures.c

```c{.line-numbers}
// rstp_procedures.c
void rstpTxRstp(RstpBridgePort *port) {

    RstpBpdu bpdu;
    // ===== 1.组 RST BPDU 头部 =====
    // Protocol Identifier: STP 家族固定为 0（网络字节序）
    bpdu.protocolId = HTONS(STP_PROTOCOL_ID);
    // Protocol Version Identifier: STP 为 0、RSTP 为 2、MSTP 为 3
    bpdu.protocolVersionId = RSTP_PROTOCOL_VERSION;
    // BPDU Type：0x00 表示配置 BPDU，0x02 表示 RST BPDU（RSTP/MSTP 使用的类型），0x80 表示 TCN BPDU
    bpdu.bpduType = RSTP_BPDU_TYPE_RST;
    // Flags 先清零，后续按位 OR 进去
    bpdu.flags = 0;

    // ===== 2.填 Flags：端口角色(Port Role) =====
    // RSTP Flags 中 Bit3/Bit2 用于端口角色编码：
    // 00 Unknown, 01 Alternate/Backup, 10 Root, 11 Designated
    if (port->role == STP_PORT_ROLE_ALTERNATE || port->role == STP_PORT_ROLE_BACKUP) {
        // Alternate / Backup
        bpdu.flags |= RSTP_BPDU_FLAG_PORT_ROLE_ALT_BACKUP;
    } else if (port->role == STP_PORT_ROLE_ROOT) {
        // Root
        bpdu.flags |= RSTP_BPDU_FLAG_PORT_ROLE_ROOT;
    } else if (port->role == STP_PORT_ROLE_DESIGNATED) {
        // Designated
        bpdu.flags |= RSTP_BPDU_FLAG_PORT_ROLE_DESIGNATED;
    } else {
        // 正常实现不应产生 Unknown，这里属于健壮性兜底
        bpdu.flags |= RSTP_BPDU_FLAG_PORT_ROLE_UNKNOWN;
    }

    // ===== 3.填 Flags：Proposal/Agreement（P/A 快速收敛握手相关） =====
    // Agreement 标志位：表示我同意对端的 Proposal，且我已确保自己侧面已同步/安全
    // 注意这里用的是 port->agree（用于发包携带 Agreement 位），而不是 port->agreed（常用于状态机判定）
    if (port->agree) {
        bpdu.flags |= RSTP_BPDU_FLAG_AGREEMENT;
    }
    // Proposal 标志位：Designated 端口在需要发起 P/A 握手时置位
    if (port->proposing) {
        bpdu.flags |= RSTP_BPDU_FLAG_PROPOSAL;
    }

    // ===== 4.填 Flags：Topology Change (TC) =====
    // tcWhile 非 0 通常表示拓扑变化流程/计时器在进行，应在 BPDU 中携带 TC 通知全网
    // TCA（Topology Change Ack）在 RSTP 里一般不使用/不靠这个位来做确认（实现里常为 0）
    if (port->tcWhile != 0) {
        bpdu.flags |= RSTP_BPDU_FLAG_TC;
    }

    // ===== 5.填 Flags：Learning/Forwarding（反映本端口当前实际状态） =====
    // learning/forwarding 通常由端口状态机(PST)决定，这里只是把结果带进 BPDU 供邻居参考
    if (port->learning) {
        bpdu.flags |= RSTP_BPDU_FLAG_LEARNING;
    }
    if (port->forwarding) {
        bpdu.flags |= RSTP_BPDU_FLAG_FORWARDING;
    }

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

    // ===== 7.填计时器（designatedTimes）：并转换为 1/256 秒单位写入 =====
    // 标准里这些时间字段用：unsigned_value * (1/256秒) 表示
    // 所以实现常用 seconds * 256 再写入两字节字段
    bpdu.messageAge   = htons(port->designatedTimes.messageAge * 256);
    bpdu.maxAge       = htons(port->designatedTimes.maxAge * 256);
    bpdu.helloTime    = htons(port->designatedTimes.helloTime * 256);
    bpdu.forwardDelay = htons(port->designatedTimes.forwardDelay * 256);

    // ===== 8.Version 1 Length =====
    // RSTP BPDU 该字段为 0，表示不携带 Version 1 协议信息
    bpdu.version1Length = 0;
    
    // ===== 9.发送 BPDU =====
    // RSTP_RST_BPDU_SIZE 应为 RST BPDU 的固定长度（包含 Version1Length 字段）
    rstpSendBpdu(port, &bpdu, RSTP_RST_BPDU_SIZE);
}

void rstpUpdtRoleDisabledTree(RstpBridgeContext *context) {
    uint_t i;

    //Loop through the ports of the bridge
    for(i = 0; i < context->numPorts; i++) {
        //Set the selectedRole variable to DisabledPort
        context->ports[i].selectedRole = STP_PORT_ROLE_DISABLED;
    }
}

void rstpRecordProposal(RstpBridgePort *port) {
    uint8_t role;
    const RstpBpdu *bpdu;

    // bpdu 指向端口 p 接收到的 BPDU 报文
    bpdu = &port->context->bpdu;

    // Decode port role
    role = bpdu->flags & RSTP_BPDU_FLAG_PORT_ROLE;

    // If the received Configuration Message conveys a Designated Port Role, and has the Proposal flag set, the proposed flag is set. 
    // 只有当收到的 BPDU 端口角色=Designated，且 Proposal 位=1 时，才把 port->proposed = TRUE
    if (role == RSTP_BPDU_FLAG_PORT_ROLE_DESIGNATED) {
        if ((bpdu->flags & RSTP_BPDU_FLAG_PROPOSAL) != 0) {
            port->proposed = TRUE;
        }
    }
}

void rstpRecordAgreement(RstpBridgePort *port) {
    const RstpBpdu *bpdu;

    // Point to the received BPDU
    bpdu = &port->context->bpdu;

    // If rstpVersion is TRUE, operPointToPointMAC is TRUE, and the received Configuration Message has the Agreement flag set, the agreed flag is set
    // and the proposing flag is cleared. Otherwise, the agreed flag is cleared
    // 只有当运行在 RSTP 版本，且端口被判定为点到点链路，且收到的 BPDU Agreement 位=1 时，才把 port->agreed 置 TRUE，并清掉 port->proposing
    if (rstpVersion(port->context) && port->operPointToPointMac && (bpdu->flags & RSTP_BPDU_FLAG_AGREEMENT) != 0) {
        port->agreed = TRUE;
        port->proposing = FALSE;
    } else {
        port->agreed = FALSE;
    }
}
```

## 五、rstp_pst.c

```c{.line-numbers}
// rstp_pst.c
void rstpPstFsm(RstpBridgePort *port) {
    // PST（Port State Transition）状态机主循环
    // 作用是根据请求变量 port->learn/port->forward（其它机器请求让端口进入 LEARNING/FORWARDING 状态），推进端口的三态：DISCARDING/LEARNING/FORWARDING
    // 注意这里不直接操作硬件/驱动，真正 enable/disable 动作在 rstpPstChangeState() 的 entry action 里做

    // port->pstState 表示 PST 状态机当前的状态，即端口当前的状态；port->learn/port->forward 是请求变量，希望接口进入 LEARNING/FORWARDING 状态
    switch (port->pstState) {
        case RSTP_PST_STATE_DISCARDING:
            // DISCARDING：不学习、不转发（数据面阻断）
            // 只要控制面请求 learn=1，就允许进入 LEARNING
            if (port->learn) {
                // Enable learning（进入 LEARNING 时会在 changeState 里调用 rstpEnableLearning）
                rstpPstChangeState(port, RSTP_PST_STATE_LEARNING);
            }
            break;

        case RSTP_PST_STATE_LEARNING:
            // LEARNING：允许学习 MAC，但仍不转发数据帧
            // forward=1 的优先级更高，一旦允许 forward，就进入 FORWARDING
            if (port->forward) {
                // Enable forwarding（进入 FORWARDING 时会在 changeState 里调用 rstpEnableForwarding）
                rstpPstChangeState(port, RSTP_PST_STATE_FORWARDING);
            // forward=0，learn=0
            // learn 被撤回：退回 DISCARDING（进入 DISCARDING 会关学习和转发）
            } else if (!port->learn) {
                rstpPstChangeState(port, RSTP_PST_STATE_DISCARDING);
            // learn=1 且 forward=0：保持 LEARNING    
            } else {
            }
            break;

        case RSTP_PST_STATE_FORWARDING:
            // FORWARDING：允许转发（也意味着学习通常已允许或保持）
            // 一旦控制面撤回 forward=0，就回到 DISCARDING（并在 entry action 里关学习/转发）
            if (!port->forward) {
                rstpPstChangeState(port, RSTP_PST_STATE_DISCARDING);
            }
            break;

        default:
            // 非法状态保护
            rstpFsmError(port->context);
            break;
    }
}

void rstpPstChangeState(RstpBridgePort *port, RstpPstState newState) {
    // PST 状态切换函数的作用：更新 port->pstState、执行新状态的 entry action（只执行一次）、置 context->busy，触发全局调度器继续迭代直到收敛

    // Switch to the new state
    port->pstState = newState;

    // entry action：进入状态时执行一次该状态的动作
    switch (port->pstState) {
    case RSTP_PST_STATE_DISCARDING:
        // 进入 DISCARDING 状态，确保学习 + 转发功能都关闭
        rstpDisableLearning(port);
        port->learning = FALSE;
        rstpDisableForwarding(port);
        port->forwarding = FALSE;
        break;

    case RSTP_PST_STATE_LEARNING:
        // 进入 LEARNING 状态开启学习（但仍不转发）
        rstpEnableLearning(port);
        port->learning = TRUE;
        break;

    case RSTP_PST_STATE_FORWARDING:
        // 进入 FORWARDING 状态开启转发（数据面放行）
        rstpEnableForwarding(port);
        port->forwarding = TRUE;
        break;

    default:
        // 未覆盖状态：不做动作（防御式）
        break;
    }

    // 标记状态机忙：全局调度器需要继续跑，直到所有状态机不再产生变化
    port->context->busy = TRUE;
}
```

## 六、rstp_prs.c

```c{.line-numbers}
// rstp_prs.c
void rstpPrsInit(RstpBridgeContext *context) {
    //Enter initial state
    rstpPrsChangeState(context, RSTP_PRS_STATE_INIT_BRIDGE);
}

void rstpPrsChangeState(RstpBridgeContext *context, RstpPrsState newState) {
    //Switch to the new state
    context->prsState = newState;

    //On entry to a state, the procedures defined for the state are executed exactly once
    switch(context->prsState) {
        //INIT_BRIDGE state?
        case RSTP_PRS_STATE_INIT_BRIDGE:
            //On initialization all ports are assigned the Disabled port role
            rstpUpdtRoleDisabledTree(context);
            break;
        //ROLE_SELECTION state?
        case RSTP_PRS_STATE_ROLE_SELECTION:
            //Update spanning tree information and port roles
            rstpClearReselectTree(context);
            rstpUpdtRolesTree(context);
            rstpSetSelectedTree(context);
            break;
        //Invalid state?
        default:
            //Just for sanity
            break;
    }
    context->busy = TRUE;
}

void rstpPrsFsm(RstpBridgeContext *context)
{
    uint_t i;
    bool_t reselect;

    switch(context->prsState) {
        //INIT_BRIDGE state?
        case RSTP_PRS_STATE_INIT_BRIDGE:
            rstpPrsChangeState(context, RSTP_PRS_STATE_ROLE_SELECTION);
            break;
        case RSTP_PRS_STATE_ROLE_SELECTION:
            for(reselect = FALSE, i = 0; i < context->numPorts; i++) {
                reselect |= context->ports[i].reselect;
            }
            if(reselect) {
                rstpPrsChangeState(context, RSTP_PRS_STATE_ROLE_SELECTION);
            }
            break;
        default:
            rstpFsmError(context);
            break;
    }
}
```

## 解析

### 1.初始化 

RSTP 协议初始化入口是 **`rstpFsmInit()`**，它做完端口基础字段清零和设置后，会调用 **`rstpPrsInit(context);`** 先初始化每一个桥的 PRS（每个桥只有 1 个 PRS 状态机），接着调用 **`rstpPrsChangeState(context, RSTP_PRS_STATE_INIT_BRIDGE)`** 切换到 **`INIT_BRIDGE`** 状态，并在 entry action（**`rstpUpdtRoleDisabledTree`** 函数）里把所有端口的 **`selectedRole`** 设为 **`STP_PORT_ROLE_DISABLED`**，并且最重要的是将 **`context->prsState`** 设置为 **`RSTP_PRS_STATE_INIT_BRIDGE`**。随后，**`rstpFsmInit()`** 会循环调用每个端口的各个状态机的初始化函数，包括 **`rstpPrtInit(port);`** 初始化每个端口的 PRT 状态机，接着调用 **`rstpPrtDisabledPortChangeState(port, RSTP_PRT_STATE_INIT_PORT);`** 函数将某个端口 port 的 role 设置为 **`STP_PORT_ROLE_DISABLED`**。此时 role 和 selectedRole 都是 Disabled，端口处于禁用状态。

随后 **`rstpFsmInit()`** 会调用 **`rstpFsm()`**，**`rstpFsm()`** 会调用 **`rstpPrsFsm()`** 进入到 **`RSTP_PRS_STATE_INIT_BRIDGE`** 状态，并调用 **`rstpPrsChangeState(context, RSTP_PRS_STATE_ROLE_SELECTION);`** 切换到 **`ROLE_SELECTION`** 状态，在 entry action 里调用 **`rstpUpdtRolesTree(context);`** 函数更新端口角色，此时所有端口的 selectedRole 都被更新为 **`Alternate/Backup/Root/Designated`** 中的某个角色，端口进入选举后的稳定状态。

所以后续继续调用 **`rstpPrtFsm()`** 函数时，由于此时端口的 selectedRole 和  role 已经不相等，**`if(port->role != port->selectedRole)`** 判断不会通过，会根据 PRS 新设置的 selectedRole 进入相应的状态处理逻辑。比如 selectedRole 是 **`RSTP_PRT_STATE_ROOT_PORT`**，那么就会调用 **`rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PORT)`** 函数，将 port->role 设置为 **`STP_PORT_ROLE_ROOT`**，并刷新 **`rrWhile`** 计时器为 rstpFwdDelay。

特别如果 selectedRole 是 **`STP_PORT_ROLE_ALTERNATE`** 时，就会调用 **`rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_BLOCK_PORT)`** 函数，将 **`port->role`** 设置为 **`STP_PORT_ROLE_ALTERNATE`**，将 **`context->prsState`** 设置为 **`RSTP_PRT_STATE_BLOCK_PORT`**，并将 port 的 learn 和 forward 状态都设置为 false，确保此 AP 端口不会进入转发状态。后续 **`port->role`** 和 **`port->selectedRole`** 都是 Alternate，会调用 **`rstpPrtAlternatePortFsm()`** 函数的 **`RSTP_PRT_STATE_BLOCK_PORT`** 状态分支，由于 learn 和 forward 都是 false，所以不会进入 LEARNING 或 FORWARDING 状态，端口保持阻塞状态。并且继续调用 **`rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PORT)`** 函数时，将 rrWhile 清 0，reBoot 设置为 false，表示端口已经稳定，退出 reRoot 流程，并且 synced 设置为 True。

>Two variables, learn and forward, are used by this state machine to request the Port State Transitions machine to change the Port State. Two further variables, learning and forwarding, indicate when the Port State transition has actually occurred. 这两个变量主要由 PRT 状态机在进入某些状态时置位，用来请求 PST 把端口推进到 Learning 或 Forwarding。learning/forwarding 表示 PST 实际已经执行到哪儿的状态位。

### 2.reRoot 机制



### 3.RP 切换


