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
                // 3.端口正要学习或转发数据帧，必须拉回 discarding 状态以阻断潜在环路，如果本来就在 Discarding，一般 learn/forward 已经是 FALSE，再切状态没意义
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

void rstpPrtAlternatePortFsm(RstpBridgePort *port) {
    // 端口角色转换（PRT）状态机：持续评估当前状态的所有转移条件，满足任一条件即发生状态迁移
    switch (port->prtState) {
    case RSTP_PRT_STATE_ALTERNATE_PROPOSED:
    case RSTP_PRT_STATE_ALTERNATE_AGREED:
    case RSTP_PRT_STATE_BACKUP_PORT:
        rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PORT);
        break;

    // 处于 BLOCK_PORT（阻塞端口）状态时
    case RSTP_PRT_STATE_BLOCK_PORT:
        if (port->selected && !port->updtInfo) {
            // 只有当端口既不学习（learning=0）也不转发（forwarding=0）时，才认为确实处于丢弃/阻塞语义，进而开始设置 Alternate/Backup 端口的状态机
            if (!port->learning && !port->forwarding) {
                // 切到 ALTERNATE_PORT（替代端口稳定态）
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PORT);
            }
        }

        break;

    // 处于 ALTERNATE_PORT（替代端口稳定态）时，按握手/同步/计时器等条件决定是否进入其它子状态
    case RSTP_PRT_STATE_ALTERNATE_PORT:
        if (port->selected && !port->updtInfo) {
            // Proposal/Agreement 握手：对端提出 proposal，而本端还未 agree，则进入 ALTERNATE_PROPOSED 状态，触发全树 sync
            if (port->proposed && !port->agree) {
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PROPOSED);
            // 已满足同步条件但尚未 agree，或者既 proposed 又 agree——进入已同意子状态
            } else if ((rstpAllSynced(port->context) && !port->agree) || (port->proposed && port->agree)) {
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_AGREED);
            // 备份端口判定，当端口角色为 BACKUP 且 Recent Backup 计时器不等于 2*HelloTime，这里用 rbWhile != 2*HelloTime 作为触发条件之一
            } else if (port->rbWhile != (2 * rstpHelloTime(port)) && port->role == STP_PORT_ROLE_BACKUP) {
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_BACKUP_PORT);
            // 同步/重选根相关的触发：fdWhile、sync、reRoot、synced 等变化时，重新进入 ALTERNATE_PORT
            // 注意：这里虽然还是切到同一状态，但会执行该状态的 entry 动作（用于重置变量/计时器等）
            } else if (port->fdWhile != rstpForwardDelay(port) || port->sync || port->reRoot || !port->synced) {
                rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PORT);
            } else {
                //Just for sanity
            }
        }

        break;

    default:
        rstpFsmError(port->context);
        break;
    }
}

void rstpPrtAlternatePortChangeState(RstpBridgePort *port, RstpPrtState newState) {
    // 切换到新的 PRT（Port Role Transition）状态：用于 Alternate/Backup 端口角色的状态入口处理
    port->prtState = newState;

    switch (port->prtState) {
    // ALTERNATE_PROPOSED：对端发送 Proposal，本端口进入提议已收到/待同步阶段
    case RSTP_PRT_STATE_ALTERNATE_PROPOSED:
        // Proposal 处理：
        // 1) 对整棵树置 sync=TRUE，要求全桥端口进入同步流程（同步树）
        // 2) 清除本端口 proposed 标志，避免重复处理同一提议
        rstpSetSyncTree(port->context);
        port->proposed = FALSE;
        break;

    //ALTERNATE_AGREED state?
    case RSTP_PRT_STATE_ALTERNATE_AGREED:
        // Agreement 处理：
        // 1) 清除 proposed
        // 2) 置 agree=TRUE，表示该端口达成 Agreement
        // 3) 置 newInfo=TRUE，提示发送状态机需要发送 BPDU（例如携带 Agreement 信息）
        port->proposed = FALSE;
        port->agree = TRUE;
        port->newInfo = TRUE;
        break;

    // BLOCK_PORT：进入阻塞相关状态（确保端口不学习/不转发）
    case RSTP_PRT_STATE_BLOCK_PORT:
        // 1) 用计算得到的 selectedRole 更新当前 role
        // 2) 清除 learn/forward 请求，防止 PST 状态机把端口推进到 Learning/Forwarding
        port->role = port->selectedRole;
        port->learn = FALSE;
        port->forward = FALSE;
        break;

    //BACKUP_PORT state?
    case RSTP_PRT_STATE_BACKUP_PORT:
        port->rbWhile = 2 * rstpHelloTime(port);
        break;

    // ALTERNATE_PORT：标准 Alternate 端口状态
    case RSTP_PRT_STATE_ALTERNATE_PORT:
        // 勘误处理：按 802.1Q-2018 13.37 的建议，在进入 Alternate 时重装 fdWhile
        port->fdWhile = rstpForwardDelay(port);
        // 进入 Alternate 后，认为端口处于已同步状态，并清理同步/重根控制变量
        port->synced = TRUE;
        port->rrWhile = 0;
        port->sync = FALSE;
        port->reRoot = FALSE;
        break;

    //Invalid state?
    default:
        //Just for sanity
        break;
    }

    //The RSTP state machine is busy
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

void rstpPtxInit(RstpBridgePort *port) {
    // PTX（Port Transmit）状态机初始化：直接进入 TRANSMIT_INIT
    // 该状态的入口动作会初始化发送相关变量（newInfo、txCount），随后在 PTX FSM 中无条件回到 IDLE
    rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_INIT);
}

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

void rstpSetTcFlags(RstpBridgePort *port) {
    const RstpBpdu *bpdu;
    bpdu = &port->context->bpdu;

    //Check BPDU type
    if(bpdu->bpduType == RSTP_BPDU_TYPE_CONFIG || bpdu->bpduType == RSTP_BPDU_TYPE_RST) {
        // Sets rcvdTc and/or rcvdTcAck if the Topology Change and/or Topology Change Acknowledgment flags, respectively, are set in a Configuration BPDU or RST BPDU
        if((bpdu->flags & RSTP_BPDU_FLAG_TC) != 0) {
            port->rcvdTc = TRUE;
        }
        if((bpdu->flags & RSTP_BPDU_FLAG_TC_ACK) != 0) {
            port->rcvdTcAck = TRUE;
        }
    } else if(bpdu->bpduType == RSTP_BPDU_TYPE_TCN) {
        //Set rcvdTcn TRUE if the BPDU is a TCN BPDU
        port->rcvdTcn = TRUE;
    } else {
        //Just for sanity
    }
}


typedef enum {
   RSTP_BPDU_TYPE_CONFIG = 0x00,
   RSTP_BPDU_TYPE_TCN    = 0x80,
   RSTP_BPDU_TYPE_RST    = 0x02,
} RstpBpduTypes;

RstpRcvdInfo rstpRcvInfo(RstpBridgePort *port) {
    int_t res;
    uint8_t role;
    RstpRcvdInfo portInfo;
    const RstpBpdu *bpdu;

    bpdu = &port->context->bpdu;

    // 在 rstp_bpdu.h 里，CycloneSTP 把 BPDU 类型定义为:
    // RSTP_BPDU_TYPE_CONFIG = 0x00: 传统 STP 的 Configuration BPDU
    // RSTP_BPDU_TYPE_TCN    = 0x80: 传统 STP 的 TCN
    // RSTP_BPDU_TYPE_RST    = 0x02: RSTP 的 Rapid Spanning Tree BPDU，也就是 RST BPDU
    if (bpdu->bpduType == RSTP_BPDU_TYPE_TCN) {
        port->rcvdTcn = TRUE;
    }

    port->msgPriority.rootBridgeId.priority = ntohs(bpdu->rootId.priority);
    port->msgPriority.rootBridgeId.addr = bpdu->rootId.addr;
    port->msgPriority.rootPathCost = ntohl(bpdu->rootPathCost);
    port->msgPriority.designatedBridgeId.priority = ntohs(bpdu->bridgeId.priority);
    port->msgPriority.designatedBridgeId.addr = bpdu->bridgeId.addr;
    port->msgPriority.designatedPortId = ntohs(bpdu->portId);

    port->msgPriority.bridgePortId = port->portId;

    port->msgTimes.messageAge = ntohs(bpdu->messageAge) / 256;
    port->msgTimes.maxAge = ntohs(bpdu->maxAge) / 256;
    port->msgTimes.forwardDelay = ntohs(bpdu->forwardDelay) / 256;
    port->msgTimes.helloTime = ntohs(bpdu->helloTime) / 256;

    // 如果 BPDU 类型是 RST BPDU，根据 RST BPDU 的报文格式，则端口角色由 Flags 中的 bit2-3 位指定
    // 把 flags 里除 bit2~bit3 之外的其它标志位（TC、Proposal、Learning、Forwarding、Agreement、TCA 等）都清掉，只保留端口角色那两位
    // UNKNOWN = 0x00、ALT_BACKUP = 0x04、ROOT = 0x08、DESIGNATED = 0x0C
    if (bpdu->bpduType == RSTP_BPDU_TYPE_RST) {
        role = bpdu->flags & RSTP_BPDU_FLAG_PORT_ROLE;
    // CycloneSTP 这里做了一个兼容性处理，把收到的 Configuration BPDU 视为显式携带 Designated port role
    } else if (bpdu->bpduType == RSTP_BPDU_TYPE_CONFIG) {
        role = RSTP_BPDU_FLAG_PORT_ROLE_DESIGNATED;
    } else {
        role = RSTP_BPDU_FLAG_PORT_ROLE_UNKNOWN;
    }

    if (role == RSTP_BPDU_FLAG_PORT_ROLE_DESIGNATED) {
        // 比较本端口收到的 BPDU 里解出来的消息优先级向量 msgPriority，以及该端口当前保存/持有的优先级向量 portPriority
        // rstpComparePriority 返回大于 0 表示 msgPriority 更优先，小于 0 表示 portPriority 更优先，等于 0 表示两者相等
        res = rstpComparePriority(&port->msgPriority, &port->portPriority);

        // 消息向量 superior 的条件：要么消息向量更好；要么 Designated Bridge 的 Bridge Address 分量 + Designated Port 的 Port Number 分量相同
        // 因此即使这次 BPDU 的完整优先级向量看起来更差，只要它仍来自同一个 designated bridge + 同一个 designated port（端口号一致），Cyclone 仍把它归为 Superior，从而刷新/重记收到的信息
        if (res < 0) {
            if (rstpCompareBridgeAddr(&port->msgPriority.designatedBridgeId.addr,
                    &port->portPriority.designatedBridgeId.addr) == 0 &&
                rstpComparePortNum(port->msgPriority.designatedPortId,
                    port->portPriority.designatedPortId) == 0) {
                res = 1;
            }
        }

        if (res > 0) {
            portInfo = RSTP_RCVD_INFO_SUPERIOR_DESIGNATED;
        } else if (res == 0) {
            // 当 msgPriority == portPriority 时，拓扑优先级没变，但 times 不同 表示协议时序参数更新了，需要采取 Superior 分支的动作来刷新本端口持有的 times 参数
            if (rstpCompareTimes(&port->msgTimes, &port->portTimes) != 0) {
                portInfo = RSTP_RCVD_INFO_SUPERIOR_DESIGNATED;
            } else {
                portInfo = RSTP_RCVD_INFO_REPEATED_DESIGNATED;
            }
        } else {
            portInfo = RSTP_RCVD_INFO_INFERIOR_DESIGNATED;
        }
    // 在 RSTP 里，非 Designated 端口（Root/Alternate/Backup）也会发 RST BPDU，并且可能携带 Agreement 位，用于快速同步（proposal/agreement 机制）
    // 因此标准/实现会把它归为一个特殊类别：InferiorRootAlternateInfo
    } else if (role == RSTP_BPDU_FLAG_PORT_ROLE_ROOT || role == RSTP_BPDU_FLAG_PORT_ROLE_ALT_BACKUP) {
        res = rstpComparePriority(&port->msgPriority, &port->portPriority);
        if (res <= 0) {
            portInfo = RSTP_RCVD_INFO_INFERIOR_ROOT_ALTERNATE;
        } else {
            portInfo = RSTP_RCVD_INFO_OTHER;
        }
    } else {
        portInfo = RSTP_RCVD_INFO_OTHER;
    }

    return portInfo;
}

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

void rstpSetSyncTree(RstpBridgeContext *context) {
    uint_t i;

    // 作用：把整桥所有端口的 sync 变量置为 TRUE，用于触发同步（sync）流程。
    // - 当 Root/Alternate 端口处于 Proposal/Agreement 握手流程时，需要先让相关端口进入同步状态，以避免在拓扑快速切换时形成临时二层环路。
    // - 各端口的状态机会根据 sync 标志，去做相应的阻塞/等待/状态收敛动作，当同步完成后，再由状态机清除 sync 或设置 synced 等标志。
    /* 遍历桥上的所有端口，将 sync 置位 */
    for (i = 0; i < context->numPorts; i++) {
        context->ports[i].sync = TRUE;
    }
}


void rstpSetReRootTree(RstpBridgeContext *context) {
    uint_t i;

    // 作用：把整桥所有端口的 reRoot 变量置为 TRUE，用于触发重新定根（re-root）流程。
    // 当 Root 端口进入 REROOT 状态时，会调用该过程，把 reRoot 置位扩散到所有端口。
    /* 遍历桥上的所有端口，将 reRoot 置位 */
    for (i = 0; i < context->numPorts; i++) {
        context->ports[i].reRoot = TRUE;
    }
}

typedef __packed_struct {
    uint16_t priority; //0-1
    MacAddr addr;      //2-7
} StpBridgeId;

typedef struct {
    StpBridgeId rootBridgeId;           // 当前收到的 BPDU 中所宣称的根桥 ID
    uint32_t rootPathCost;              // 到根桥的到根桥的累计路径开销
    StpBridgeId designatedBridgeId;     // 指定桥 ID，这个到根最好路径信息是谁代表这个网段发出来的（谁是这个网段的指定桥）
    uint16_t designatedPortId;          // 指定端口 ID，指定桥用它的哪个端口在这个网段宣告这份信息
    uint16_t bridgePortId;              // 桥端口 ID，本桥用于保存/接收该向量的端口 ID，也就是接收端口的 Port ID
} RstpPriority;

typedef struct {
    uint_t messageAge;
    uint_t maxAge;
    uint_t forwardDelay;
    uint_t helloTime;
} RstpTimes;

typedef enum {
    RSTP_INFO_IS_DISABLED = 0,
    RSTP_INFO_IS_RECEIVED = 1,
    RSTP_INFO_IS_MINE     = 2,
    RSTP_INFO_IS_AGED     = 3
} RstpInfoIs;

void rstpClearReselectTree(RstpBridgeContext *context) {
    uint_t i;
    // 清空所有端口的 reselect 变量
    for(i = 0; i < context->numPorts; i++) {
        //Clear the reselect variable
        context->ports[i].reselect = FALSE;
    }
}

void rstpUpdtRolesTree(RstpBridgeContext *context) {
    uint_t i;
    RstpBridgePort *port;
    RstpBridgePort *rootPort;
    RstpPriority rootPathPriority;

    // 假设本桥就是根桥，然后再去遍历本桥的各个端口的信息，查看是否有比本桥作为根桥更优的信息
    // rootPriority 和 bridgePriority 都属于生成树优先级向量，先假设本桥就是根，把 rootPriority 初始化为 bridgePriority
    context->rootPriority = context->bridgePriority;
    // 在本桥是根桥的假设下，根端口并不存在，这里用一个本桥自己的 bridgePortId 字段当占位/默认值
    context->rootPortId = context->bridgePriority.bridgePortId;
    // bridgeTimes 的类型是 RstpTimes，表示本桥应当遵循的根计时参数合集，根桥用自己的 BridgeTimes 作为全网计时基准
    context->rootTimes = context->bridgeTimes;
    //The port the root priority vector is derived from
    rootPort = NULL;

    // 遍历本桥的所有端口，选出本桥的最佳根信息（rootPriority/rootTimes）与 RootPort
    for(i = 0; i < context->numPorts; i++) {
        port = &context->ports[i];
        // infoIs 表示每个端口保存的生成树信息来源/状态标志，它是一个枚举，有四种可能的取值：RSTP_INFO_IS_DISABLED、RSTP_INFO_IS_RECEIVED、RSTP_INFO_IS_MINE、RSTP_INFO_IS_AGED，可以理解为端口保存的优先级向量到底是从哪儿来的以及是否还有效
        // 当 infoIs 等于 RSTP_INFO_IS_RECEIVED 时，表示该端口保存的优先级向量是从对端 BPDU 中收到的，并且该信息仍然有效（没有过期）
        // PIM 状态机会在收到更优（Superior）的指定信息时：
        // 1.先调用 rstpRecordPriority(port);/rstpRecordTimes(port); 把收到的 BPDU 携带的信息记录到端口变量里
        // 2.再更新 rcvdInfoWhile 为 3*helloTime，表示该信息的有效期（rcvdInfoWhile = 0 表示信息过期）
        // 3.最后把 infoIs 置为 RSTP_INFO_IS_RECEIVED
        // PIM 状态机的 CURRENT 状态处理过程中：
        // 1.如果 infoIs == RECEIVED，并且 rcvdInfoWhile == 0
        // 就把 infoIs 置为 RSTP_INFO_IS_AGED，表示该信息已过期
        if(port->infoIs == RSTP_INFO_IS_RECEIVED) {
            // 先把该端口收到的 portPriority 复制出来
            rootPathPriority = port->portPriority;
            // 再把本端口 path cost 加到 rootPathCost 上，得到经由该端口到根的代价
            rootPathPriority.rootPathCost += port->portPathCost;

            // 一个桥有多个转发端口，每个端口有一个 MAC 地址，通常我们把端口号最小的那个端口的 MAC 地址作为整个桥的 MAC 地址
            // rootPathPriority.designatedBridgeId.addr 表示从该端口收到并记录的 BPDU（portPriority）里带来的 Designated Bridge ID 的桥 MAC
            // context->bridgePriority.designatedBridgeId.addr 表示桥自身的 bridgePriority 里的 Designated Bridge ID 的桥 MAC
            // 这个判断条件是为了避免把本桥自己发出的/源自本桥的信息当成外部候选来参与根选择，否则在某些拓扑下会产生不合理的根路径候选，甚至导致角色选择不稳定
            // 本桥的两个端口被外部线路/Hub/错误接线形成回环，导致本桥从端口 A 发出的 BPDU 被端口 B 收到了
            if(rstpCompareBridgeAddr(&rootPathPriority.designatedBridgeId.addr, &context->bridgePriority.designatedBridgeId.addr) != 0) {
                // 比较从该端口收到的优先级向量与当前桥的 rootPriority，比较顺序也是：Root ID -> Path Cost -> Designated Bridge ID -> Designated Port ID -> Bridge Port ID
                // 如果从该端口收到的优先级向量更优（rstpComparePriority 返回值 > 0），就更新桥的 rootPriority
                if(rstpComparePriority(&rootPathPriority, &context->rootPriority) > 0) {
                    //Save current root path priority vector
                    context->rootPriority = rootPathPriority;
                    context->rootPortId = rootPathPriority.bridgePortId;

                    rootPort = port;

                    context->rootTimes = port->portTimes;
                    // 从邻居收到的计时参数转为本桥 rootTimes 时，消息年龄加 1
                    context->rootTimes.messageAge++;
                }
            }
        }
    }

    // 为每个端口计算如果我在该段成为 Designated 时应该通告出去的 designatedPriority/designatedTimes
    for(i = 0; i < context->numPorts; i++) {
        port = &context->ports[i];

        port->designatedPriority.rootBridgeId = context->rootPriority.rootBridgeId;
        port->designatedPriority.rootPathCost = context->rootPriority.rootPathCost;
        port->designatedPriority.designatedBridgeId = context->bridgeId;
        port->designatedPriority.designatedPortId = port->portId;
        port->designatedPriority.bridgePortId = port->portId;
        port->designatedTimes = context->rootTimes;
        port->designatedTimes.helloTime = context->bridgeTimes.helloTime;
    }

    // The port role for each port is assigned, and its port priority vector and Spanning Tree timer information are updated
    for(i = 0; i < context->numPorts; i++) {
        //Point to the current bridge port
        port = &context->ports[i];

        // RSTP_INFO_IS_DISABLED 的含义是：端口的生成树信息处于禁用状态，也就是该端口当前不参与生成树计算
        // 当 PIM 会根据 port->portEnabled = port->macOperState && port->params.adminPortState; 来判断端口是否可以收发帧
        // 只有该端口当前链路工作状态正常并且管理员配置为启用 STP/RSTP 时，该端口才算 enabled
        // 因此当 infoIs 等于 RSTP_INFO_IS_DISABLED 时，协议状态机（PIM）判定该端口不参与生成树，于是在角色选择时直接给 Disabled 角色
        if(port->infoIs == RSTP_INFO_IS_DISABLED) {
            // If the Port is Disabled, selectedRole is set to DisabledPort
            port->selectedRole = STP_PORT_ROLE_DISABLED;
 
        // 当 infoIs 等于 RSTP_INFO_IS_AGED 时，表示该端口保存的生成树信息已过期
        // 因此既然该端口已经没有任何有效的外部（received）优先级向量可参考，那在这个网段上，本桥就按规则先把自己当成指定端口（Designated Port）候选
        // 同时把 updtInfo 置 TRUE，是为了驱动 PIM 去把端口的 portPriority/portTimes 更新成 designated 值，并准备发 BPDU
        // 在 PIM 状态机中，在 AGED 状态如果 port->selected && port->updtInfo，就切到 UPDATE 状态；在 UPDATE 状态入状态动作中，会把 portPriority/portTimes 更新成 designatedPriority/designatedTimes，将 infoIs 设置为 RSTP_INFO_IS_MINE，并且将 newInfo 设置为 TRUE，准备发 BPDU
        } else if(port->infoIs == RSTP_INFO_IS_AGED) {
            port->selectedRole = STP_PORT_ROLE_DESIGNATED;
            port->updtInfo = TRUE;
        
        // RSTP_INFO_IS_MINE 表示端口当前持有的信息来自本桥自身/本桥其他端口推导
        // 因此此时 port->portPriority 保存的不是从 BPDU 里记录下来的对端宣告的优先级向量
        // 而是本桥根据当前选出来的 rootPriority/rootTimes，为该端口计算得到的 designatedPriority/designatedTimes
        // Mine 说明这个端口当前采用的是我方要在该网段上宣告的 designated 信息，那它的角色自然就是 Designated Port（该段上由我负责发通告的一侧）
        } else if(port->infoIs == RSTP_INFO_IS_MINE) {
            port->selectedRole = STP_PORT_ROLE_DESIGNATED;

            // 如果 portPriority/portTimes 变量已经不一致了，就标记 updtInfo，让 PIM 去 UPDATE 一次，把端口信息刷新到最新
            if(rstpComparePriority(&port->portPriority, &port->designatedPriority) != 0 || rstpCompareTimes(&port->portTimes, &context->rootTimes) != 0) {
                port->updtInfo = TRUE;
            }

        // RSTP_INFO_IS_RECEIVED 表示端口当前持有的信息是从对端收到的，并且该信息仍然有效    
        } else if(port->infoIs == RSTP_INFO_IS_RECEIVED) {
            if(port == rootPort) {
                port->selectedRole = STP_PORT_ROLE_ROOT;
                port->updtInfo = FALSE;
            } else {
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
                        //selectedRole is set to AlternatePort and updtInfo is reset
                        port->selectedRole = STP_PORT_ROLE_ALTERNATE;
                        port->updtInfo = FALSE;
                    }
                }
            }
        } else {
            //Just for sanity
        }
    }

    //Loop through the ports of the bridge
    for(i = 0; i < context->numPorts; i++) {
        port = &context->ports[i];
    }
}

void rstpSetSelectedTree(RstpBridgeContext *context) {
    uint_t i;
    bool_t reselect;

    // 1.汇总是否有端口需要重选
    for(reselect = FALSE, i = 0; i < context->numPorts; i++) {
        reselect |= context->ports[i].reselect;
    }
    // 2.只有当没有端口需要重选时，才把所有端口的 selected 变量置为 TRUE
    // 如果所有端口的 port->reselect 都是 FALSE，说明这次角色选择没有被任何端口打断/推翻，那么就把所有端口的 selected 置为 TRUE
    if(!reselect) {
        //Loop through the ports of the bridge
        for(i = 0; i < context->numPorts; i++) {
            context->ports[i].selected = TRUE;
        }
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

## 七、rstp_pti.c

```c{.line-numbers}
// rstp_pti.c
void rstpPtiFsm(RstpBridgePort *port) {
    // PTI（Port Timers state machine）：负责每秒一次的节拍处理
    // 状态机持续评估当前状态的迁移条件，满足条件就切换状态
    switch (port->ptiState) {
        // ONE_SECOND：等待 1 秒 tick 信号（tick 通常由外部 1s 定时器置位）
        case RSTP_PTI_STATE_ONE_SECOND:
            // 每来一次 tick，就进入 TICK 状态做每秒钟应做的事
            if (port->tick) {
                rstpPtiChangeState(port, RSTP_PTI_STATE_TICK);
            }
            break;
        // TICK：完成一次每秒处理后，立刻回到 ONE_SECOND
        case RSTP_PTI_STATE_TICK:
            // 无条件回到 ONE_SECOND：用于清掉 tick，并等待下一次 tick
            rstpPtiChangeState(port, RSTP_PTI_STATE_ONE_SECOND);
            break;
        default:
            // 非法状态：触发统一的状态机错误处理
            rstpFsmError(port->context);
            break;
    }
}

void rstpPtiChangeState(RstpBridgePort *port, RstpPtiState newState) {
    // 切换到新状态
    port->ptiState = newState;

    // 进入某状态时，该状态的入口动作只执行一次
    switch (port->ptiState) {
        case RSTP_PTI_STATE_ONE_SECOND:
            // ONE_SECOND 入口动作：清除 tick，表示已消费本轮 1s 节拍
            port->tick = FALSE;
            break;

        case RSTP_PTI_STATE_TICK:
            // TICK 入口动作：将所有非 0 的计时器减 1（单位：秒）
            // 这些计时器对应 RSTP 中的各类 per-port timer
            rstpDecrementTimer(&port->helloWhen);       // Hello timer：控制 BPDU 周期发送节奏
            rstpDecrementTimer(&port->tcWhile);         // Topology Change timer：TC 传播/持续时间窗口
            rstpDecrementTimer(&port->fdWhile);         // Forward Delay timer：与状态迁移/延迟相关
            rstpDecrementTimer(&port->rcvdInfoWhile);   // Received Info timer：接收信息有效期/超时相关
            rstpDecrementTimer(&port->rrWhile);         // Recent Root timer：近期根相关计时
            rstpDecrementTimer(&port->rbWhile);         // Recent Backup timer：近期备份相关计时
            rstpDecrementTimer(&port->mdelayWhile);     // Migration Delay timer：STP/RSTP 迁移期计时
            rstpDecrementTimer(&port->edgeDelayWhile);  // Edge Delay timer：边缘端口延时相关计时
            // txCount：每发送一个 BPDU 会递增；这里每秒递减，用于发包保持计数/限速窗口随时间衰减
            rstpDecrementTimer(&port->txCount);
            break;

        default:
            // 非法状态：不做处理（保持健壮性）
            break;
    }

    // 标记上下文 busy：表示本轮有状态机执行了入口动作/发生了状态变化，驱动调度器继续迭代直至收敛
    port->context->busy = TRUE;
}
```

## 八、rstp_conditions.c

```c{.line-numbers}
/**
 * 在 CycloneSTP Open 中，rrWhile 是 Recent Root timer
 * - 当端口成为 Root Port 时，PRT 状态机会把 rrWhile 重新装载为 ForwardDelay
 * - 当端口进入 DESIGNATED_SYNCED 等同步完成/退休相关状态时，会把 rrWhile 清零
 *
 * 因此 rrWhile != 0 表示该端口仍处于最近 Root 端口的窗口期（重选根/同步流程尚未完全收敛）。本条件用于判断 reRoot 流程是否已经在整桥范围完成：只要除了本端口之外还有任意端口 rrWhile != 0，就认为尚未 reRooted；只有当所有其它端口 rrWhile 都为 0 时，才返回 TRUE。
 */
bool_t rstpReRooted(RstpBridgePort *port) {
    uint_t i;
    RstpBridgeContext *context;
    // 获取桥上下文（包含端口数组与端口数量）
    context = port->context;
    // 遍历桥上的所有端口，检查其它端口是否仍处于 Recent Root 窗口期
    // 注意：需要跳过本端口自身；本端口的 rrWhile 在流程中允许为非 0
    for (i = 0; i < context->numPorts; i++) {
        if (&context->ports[i] != port && context->ports[i].rrWhile != 0) {
            // 发现至少一个其它端口仍未完成 reRoot（Recent Root 计时器仍在运行）
            return FALSE;
        }
    }
    // 除本端口外，所有端口 rrWhile 均为 0，认为整桥 reRoot 流程已完成
    return TRUE;
}

bool_t rstpAllSynced(RstpBridgeContext *context) {
    uint_t i;
    bool_t res;
    RstpBridgePort *port;

    // allSynced 条件（IEEE 802.1D-2004 17.20.3）：
    // 对给定树（given tree）上的所有端口都必须满足：
    //   1) selected == TRUE、role == selectedRole
    //   2) updtInfo == FALSE
    //   3) 并且满足二选一：
    //      a) synced == TRUE
    //      b) 该端口是 Root port（role == STP_PORT_ROLE_ROOT）
    //
    // 语义直观解释：
    // - selected/role==selectedRole/!updtInfo 表示端口角色与信息已经稳定、可用于一致性判断；
    // - 对非 Root 端口，必须已完成同步（synced==TRUE）；
    // - Root 端口在 allSynced 判定中允许例外（即使 synced==FALSE 也不阻塞 allSynced），这样 Root Port 状态机才能以 allSynced 作为其他端口已同步完成的门槛。
    // 注意：Cyclone 这里不会在 res 变为 FALSE 后提前 break，而是继续遍历所有端口，这不影响最终结果（仍返回 res），只是实现风格上选择不短路。

    // 初始化结果为 TRUE：只要有任一端口不满足条件，后续就会把 res 置为 FALSE
    res = TRUE;

    for (i = 0; i < context->numPorts; i++) {
        port = &context->ports[i];
        // 条件 1：端口必须已被选中（参与本轮计算/收敛）
        if (!port->selected) {
            res = FALSE;
        // 条件 2：端口当前 role 必须与计算得到的 selectedRole 一致（角色尚未稳定则不算 allSynced）
        } else if (port->role != port->selectedRole) {
            res = FALSE;
        // 条件 3：updtInfo 必须为 FALSE（端口信息正在更新时，不能认为同步完成）
        } else if (port->updtInfo) {
            res = FALSE;
        // 条件 4：对非 Root 端口，必须 synced==TRUE；Root 端口允许例外
        } else if (!port->synced && port->role != STP_PORT_ROLE_ROOT) {
            res = FALSE;
        } else {
            // 满足当前端口的 allSynced 判定条件
        }
    }
    // 只有当所有端口都满足上述条件时，res 才会保持 TRUE
    return res;
}
```

## 九、rstp_misc.c

```c{.line-numbers}
int_t rstpComparePriority(const RstpPriority *p1, const RstpPriority *p2) {

    int_t res;

    /* Compare priority vectors according to 17.6 rules */
    if (rstpCompareBridgeId(&p1->rootBridgeId, &p2->rootBridgeId) < 0) {
        res = 1;
    } else if (rstpCompareBridgeId(&p1->rootBridgeId, &p2->rootBridgeId) > 0) {
        res = -1;
    } else if (p1->rootPathCost < p2->rootPathCost) {
        res = 1;
    } else if (p1->rootPathCost > p2->rootPathCost) {
        res = -1;
    } else if (rstpCompareBridgeId(&p1->designatedBridgeId, &p2->designatedBridgeId) < 0) {
        res = 1;
    } else if (rstpCompareBridgeId(&p1->designatedBridgeId, &p2->designatedBridgeId) > 0) {
        res = -1;
    } else if (p1->designatedPortId < p2->designatedPortId) {
        res = 1;
    } else if (p1->designatedPortId > p2->designatedPortId) {
        res = -1;
    } else if (p1->bridgePortId < p2->bridgePortId) {
        res = 1;
    } else if (p1->bridgePortId > p2->bridgePortId) {
        res = -1;
    } else {
        res = 0;
    }
    /* Return comparison result */
    return res;
}

```

## 十、rstp_pim.c

```c{.line-numbers}
void rstpPimInit(RstpBridgePort *port) {
    rstpPimChangeState(port, RSTP_PIM_STATE_DISABLED);
}

// PIM 它负责维护每个端口当前持有的生成树信息
void rstpPimFsm(RstpBridgePort *port) {
    /**
      * portEnabled:   端口是否启用，链路状态 UP 且管理员启用 STP/RSTP
      * rcvdMsg:       是否收到过 BPDU 消息
      * rcvdInfoWhile: 接收信息的保鲜计时器，当它归零时，信息会被判定为 aged out
      * updtInfo:      端口信息是否需要用本桥计算出的 designated 信息进行更新，portPriority/portTimes 需要刷新为 designatedPriority/designatedTimes
      * selected/reselect: 桥的端口角色选择流程用的握手标志；PIM 在很多入场动作里把 reselect=TRUE, selected=FALSE，强制外层重新跑一次角色选择树（roles tree）
      */
    // 只要端口不再可用，无论端口 infoIs 当前在哪个 PIM 状态，都立即强制切到 DISABLED
    if (!port->portEnabled && port->infoIs != RSTP_INFO_IS_DISABLED) {
        rstpPimChangeState(port, RSTP_PIM_STATE_DISABLED);
    } else {

        switch (port->pimState) {
        case RSTP_PIM_STATE_DISABLED:
            // 禁用态下如果还收到了 BPDU（rcvdMsg=TRUE），就重新进入一次 DISABLED 以触发 entry action，即禁用口收到 BPDU 也不处理，只是清掉我收到了消息这个标志，保持禁用态干净。
            if (port->rcvdMsg) {
                rstpPimChangeState(port, RSTP_PIM_STATE_DISABLED);
            // 如果端口恢复可用（portEnabled=TRUE），就从 DISABLED 切到 AGED，也就是端口刚恢复，先当作没有任何可信的接收信息（AGED 起点）
            } else if (port->portEnabled) {
                rstpPimChangeState(port, RSTP_PIM_STATE_AGED);
            } else {
            }
            break;

        case RSTP_PIM_STATE_AGED:
            // AGED 只是声明端口原先接收来的信息已失效，并不代表立刻就该用本桥信息覆盖并对外宣告
            // 因此它要等到 selected && updtInfo 成立后才进入 UPDATE，把端口信息切换为 MINE 并置 newInfo=TRUE
            // 先让全桥的角色/根信息计算结果稳定，再在确实需要同步端口信息时（updtInfo 为 true 表示不一致） 才更新和触发发送，避免在角色未定时发出自相矛盾的 BPDU、减少无意义的覆盖与抖动，同时防止把最终应为 Alternate/Backup/Disabled 的端口误写成可宣告的 designated 信息
            if (port->selected && port->updtInfo) {
                rstpPimChangeState(port, RSTP_PIM_STATE_UPDATE);
            }
            break;

        case RSTP_PIM_STATE_UPDATE:
        case RSTP_PIM_STATE_SUPERIOR_DESIGNATED:
        case RSTP_PIM_STATE_REPEATED_DESIGNATED:
        case RSTP_PIM_STATE_INFERIOR_DESIGNATED:
        case RSTP_PIM_STATE_NOT_DESIGNATED:
        case RSTP_PIM_STATE_OTHER:
            rstpPimChangeState(port, RSTP_PIM_STATE_CURRENT);
            break;

        case RSTP_PIM_STATE_CURRENT:
            // 本轮全桥的角色/优先级计算已经提交完成 并且 Cyclone 里注释是提示需要更新 portPriority 和 portTimes
            // UPDATE 就是把端口信息更新为本桥计算出的 designated 信息
            if (port->selected && port->updtInfo) {
                rstpPimChangeState(port, RSTP_PIM_STATE_UPDATE);

            // 只有在真的没有新报文、也没有待提交更新的情况下，才把 RECEIVED 判定为过期，避免出现计时器刚好到 0 但新 BPDU 同时来了时的无谓 AGED 之间的 RECEIVE 来回震荡。
            } else if (port->infoIs == RSTP_INFO_IS_RECEIVED && port->rcvdInfoWhile == 0 && !port->updtInfo && !port->rcvdMsg) {
                //Switch to AGED state
                rstpPimChangeState(port, RSTP_PIM_STATE_AGED);

            // 进入 UPDATE 后，会把 portPriority/portTimes 覆盖成 designatedPriority/designatedTimes，把 infoIs 设为 MINE，并置 newInfo=TRUE，这相当于把本端口要对外宣告/持有的信息正式提交并准备发出去
            // 必须要 !updtInfo 是因为 RECEIVE 的分类结果依赖当前端口持有的信息，rstpRcvInfo 会把收到的 msgPriority/msgTimes 和端口当前的 portPriority/portTimes 做比较，从而判定 Superior/Repeated/Inferior/Other。若 updtInfo==TRUE 说明端口当前信息马上就要被 UPDATE 覆盖，那么此时用旧的 portPriority/Times 去分类，结果可能马上失效
            } else if (port->rcvdMsg && !port->updtInfo) {
                //Switch to RECEIVE state
                rstpPimChangeState(port, RSTP_PIM_STATE_RECEIVE);
            } else {
                //Just for sanity
            }
            break;

        case RSTP_PIM_STATE_RECEIVE:
            if (port->rcvdInfo == RSTP_RCVD_INFO_SUPERIOR_DESIGNATED) {
                rstpPimChangeState(port, RSTP_PIM_STATE_SUPERIOR_DESIGNATED);
            } else if (port->rcvdInfo == RSTP_RCVD_INFO_REPEATED_DESIGNATED) {
                rstpPimChangeState(port, RSTP_PIM_STATE_REPEATED_DESIGNATED);
            } else if (port->rcvdInfo == RSTP_RCVD_INFO_INFERIOR_DESIGNATED) {
                rstpPimChangeState(port, RSTP_PIM_STATE_INFERIOR_DESIGNATED);
            } else if (port->rcvdInfo == RSTP_RCVD_INFO_INFERIOR_ROOT_ALTERNATE) {
                rstpPimChangeState(port, RSTP_PIM_STATE_NOT_DESIGNATED);
            } else if (port->rcvdInfo == RSTP_RCVD_INFO_OTHER) {
                rstpPimChangeState(port, RSTP_PIM_STATE_OTHER);
            } else {
            }
            break;

        default:
            rstpFsmError(port->context);
            break;
        }
    }
}

void rstpPimChangeState(RstpBridgePort *port, RstpPimState newState) {
    port->pimState = newState;

    switch (port->pimState) {
    case RSTP_PIM_STATE_DISABLED:
        port->rcvdMsg = FALSE;
        port->proposing = FALSE;
        port->proposed = FALSE;
        port->agree = FALSE;
        port->agreed = FALSE;
        port->rcvdInfoWhile = 0;
        port->infoIs = RSTP_INFO_IS_DISABLED;
        port->reselect = TRUE;
        port->selected = FALSE;
        break;

    case RSTP_PIM_STATE_AGED:
        port->infoIs = RSTP_INFO_IS_AGED;
        port->reselect = TRUE;
        port->selected = FALSE;
        break;

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
        port->infoIs = RSTP_INFO_IS_MINE;
        port->newInfo = TRUE;
        break;

    case RSTP_PIM_STATE_SUPERIOR_DESIGNATED:
        port->agreed = FALSE;
        port->proposing = FALSE;
        rstpRecordProposal(port);
        rstpSetTcFlags(port);

        port->agree = port->agree && rstpBetterOrSameInfo(port, RSTP_INFO_IS_RECEIVED);

        rstpRecordPriority(port);
        rstpRecordTimes(port);
        rstpUpdtRcvdInfoWhile(port);
        port->infoIs = RSTP_INFO_IS_RECEIVED;
        port->reselect = TRUE;
        port->selected = FALSE;
        port->rcvdMsg = FALSE;
        break;

    case RSTP_PIM_STATE_REPEATED_DESIGNATED:
        rstpRecordProposal(port);
        rstpSetTcFlags(port);
        rstpUpdtRcvdInfoWhile(port);
        port->rcvdMsg = FALSE;
        break;

    case RSTP_PIM_STATE_INFERIOR_DESIGNATED:
        rstpRecordDispute(port);
        port->rcvdMsg = FALSE;
        break;

    case RSTP_PIM_STATE_NOT_DESIGNATED:
        rstpRecordAgreement(port);
        // 
        rstpSetTcFlags(port);
        port->rcvdMsg = FALSE;
        break;

    case RSTP_PIM_STATE_OTHER:
        port->rcvdMsg = FALSE;
        break;

    case RSTP_PIM_STATE_CURRENT:
        break;

    case RSTP_PIM_STATE_RECEIVE:
        port->rcvdInfo = rstpRcvInfo(port);
        break;

    default:
        break;
    }

    port->context->busy = TRUE;
}
```

## 十一、rstp_tcm.c

```c{.line-numbers}
void rstpTcmInit(RstpBridgePort *port) {
    rstpTcmChangeState(port, RSTP_TCM_STATE_INACTIVE);
}

/**
  * rcvdTc:收到 RSTP BPDU/STP Config BPDU 时，BPDU 的 flags 里 TC bit=1，表明对端拓扑变化正在发生/刚发生过，对端处于 TC 宣告窗口（tcWhile 期间）
  * rcvdTcn:收到 TCN BPDU（这是一种老 STP 里的专用报文类型）
  * rcvdTcAck:收到 STP Config BPDU 时，flags 里 TCA bit=1，注意 RSTP BPDU 里这个位不使用/置 0，Cyclone 发送 RSTP BPDU 时也注明了 TC_ACK 不用
  * rstpNewTcWhile():用于启动/更新 tcWhile，并在需要时更新拓扑变化计数等。
  * rstpSetTcPropTree():用于把 tcProp 设置到除自身端口外的其它端口上，以触发它们后续发送带 TC 的 BPDU，实现拓扑变化传播
  */
void rstpTcmFsm(RstpBridgePort *port) {

    switch (port->tcmState) {
    case RSTP_TCM_STATE_INACTIVE:
        // 在 rstpTcmInit 函数中，将 tcmState 设置为 RSTP_TCM_STATE_INACTIVE
        // 在 rstpTcmChangeState 函数中，一进入 INACTIVE，TCM 就把 fdbFlush 置 TRUE，然后在 rstpFsm 函数中调用 rstpRemoveFdbEntries 函数对 FDB/桥转发表模块真正执行 flush 后，把 fdbFlush 设置为 FALSE
        // 所以 !fdbFlush 的意义是：避免在旧的转发表项还没清干净时就开始学习新 MAC，否则容易出现拓扑变化后短时间内错误转发/错误学习的副作用
        if (port->learn && !port->fdbFlush) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_LEARNING);
        }
        break;

    case RSTP_TCM_STATE_LEARNING:
        // LEARNING：这个状态的一个关键作用是吸收/清空事件标志，进入该状态时会把 rcvdTc/rcvdTcn/rcvdTcAck/tcProp 清零
        // 如果检测到任一 TC 相关事件标志为 TRUE，则再次切换到 LEARNING，通过重新进入状态的入口动作把这些标志清掉，表示事件已被消费
        if (port->rcvdTc || port->rcvdTcn || port->rcvdTcAck || port->tcProp) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_LEARNING);
        // 如果端口角色是 Root/Designated，且端口允许转发 (forward=TRUE)，并且不是边缘端口 (operEdge=FALSE)，
        // 说明一个树端口进入了可以转发的阶段——这在 RSTP 中被视为一次拓扑变化触发点，转到 DETECTED 去启动 tcWhile、触发传播等动作
        // In RSTP, only non-edge ports that move to the forwarding state cause a topology change. This means that a loss of connectivity is not considered as a topology change any more, contrary to 802.1D (that is, a port that moves to blocking no longer generates a TC)
        // 只有非边缘端口切换到转发状态时才被定义为拓扑变动，非边缘端口丢失连接不会触发拓扑变化通知
        } else if ((port->role == STP_PORT_ROLE_ROOT ||
                    port->role == STP_PORT_ROLE_DESIGNATED) &&
                   port->forward && !port->operEdge) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_DETECTED);
        // 若端口既不是 Root/Designated，且也不处于学习允许/学习中状态，并且没有任何 TC 相关事件，则可以回到 INACTIVE
        } else if (port->role != STP_PORT_ROLE_ROOT &&
                   port->role != STP_PORT_ROLE_DESIGNATED &&
                   !(port->learn || port->learning) &&
                   !(port->rcvdTc || port->rcvdTcn || port->rcvdTcAck || port->tcProp)) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_INACTIVE);
        } else {
        }
        break;

    case RSTP_TCM_STATE_NOTIFIED_TCN:
        rstpTcmChangeState(port, RSTP_TCM_STATE_NOTIFIED_TC);
        break;

    // 这些状态都是一次性动作状态：无条件进入 ACTIVE 作为稳定运行状态。
    case RSTP_TCM_STATE_DETECTED:
    case RSTP_TCM_STATE_NOTIFIED_TC:
    case RSTP_TCM_STATE_PROPAGATING:
    case RSTP_TCM_STATE_ACKNOWLEDGED:
        rstpTcmChangeState(port, RSTP_TCM_STATE_ACTIVE);
        break;

    case RSTP_TCM_STATE_ACTIVE:
        // 如果端口不再是 Root/Designated，或者端口成为边缘端口 (operEdge=TRUE)，就不应继续作为 TC 的树端口处理，回到 LEARNING
        if ((port->role != STP_PORT_ROLE_ROOT && port->role != STP_PORT_ROLE_DESIGNATED) || port->operEdge) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_LEARNING);
        } else if (port->rcvdTcn) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_NOTIFIED_TCN);
        // 收到 TC 标志，进入 NOTIFIED_TC，会调用 rstpSetTcPropTree 函数，将 tcProp 设置为 true，然后发送带 TC 标志位的 BPDU，
        // 然后其他桥收到 TC 标志位的 BPDU 时，也会将 tcProp 设置为 true，然后继续扩散发送 BPDU（TC=1）到全树
        } else if (port->rcvdTc) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_NOTIFIED_TC);
        // 需要传播 TC：进入 PROPAGATING，tcProp 由 rstpSetTcPropTree() 在其他端口上置位，这里处理时要求该端口不是边缘端口
        } else if (port->tcProp && !port->operEdge) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_PROPAGATING);
        } else if (port->rcvdTcAck) {
            rstpTcmChangeState(port, RSTP_TCM_STATE_ACKNOWLEDGED);
        } else {
        }
        break;

    default:
        rstpFsmError(port->context);
        break;
    }
}

void rstpTcmChangeState(RstpBridgePort *port, RstpTcmState newState) {
    port->tcmState = newState;

    switch (port->tcmState) {
    case RSTP_TCM_STATE_INACTIVE:
        port->fdbFlush = TRUE;
        port->tcWhile = 0;
        port->tcAck = FALSE;
        break;

    case RSTP_TCM_STATE_LEARNING:
        port->rcvdTc = FALSE;
        port->rcvdTcn = FALSE;
        port->rcvdTcAck = FALSE;
        port->tcProp = FALSE;
        break;

    case RSTP_TCM_STATE_DETECTED:
        // 根据 ieee 802.1d-2004 规范，If the value of tcWhile is zero and sendRstp is true, this procedure sets the value of tcWhile to HelloTime plus one second and sets newInfo true.
        rstpNewTcWhile(port);
        rstpSetTcPropTree(port);
        port->newInfo = TRUE;
        break;

    case RSTP_TCM_STATE_NOTIFIED_TCN:
        rstpNewTcWhile(port);
        break;

    case RSTP_TCM_STATE_NOTIFIED_TC:
        port->rcvdTcn = FALSE;
        port->rcvdTc = FALSE;

        if (port->role == STP_PORT_ROLE_DESIGNATED) {
            port->tcAck = TRUE;
        }

        rstpSetTcPropTree(port);
        break;

    case RSTP_TCM_STATE_PROPAGATING:
        rstpNewTcWhile(port);
        port->fdbFlush = TRUE;
        port->tcProp = FALSE;
        break;

    case RSTP_TCM_STATE_ACKNOWLEDGED:
        port->tcWhile = 0;
        port->rcvdTcAck = FALSE;
        break;

    case RSTP_TCM_STATE_ACTIVE:
        break;

    default:
        break;
    }

    port->context->busy = TRUE;
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

如果一个端口此时并非处于转发状态（FORWARDING），当它转换到根端口状态时，它将调用 **`setReRootTree()`** 过程，为本桥的所有端口（Bridge Ports）设置 reRoot，从而指示所有近期根端口（recent roots）切换到丢弃状态。**<font color="red">根端口（Root Port）会将 rrWhile 计时器维持在 FwdDelay 这一数值；一旦该端口不再是根端口，便允许 rrWhile 自然递减直至到期；并且如果该端口进入丢弃状态，rrWhile 将被置为 0</font>**。当所有近期根端口都已被退役/解除（即满足 reRooted）之后，**<font color="red">根端口即可转换到学习（Learning）状态，并进一步转换到转发（Forwarding）状态</font>**。基本上 reRoot 机制是为了确保在根端口切换时，新的根端口能够安全地进入转发状态，而不会因为旧根端口的存在而引发临时的二层环路，只会对旧根端口有影响。

假设一个 AP 端口不是根端口，但是如果在 **`rstpFsm()`** 函数的状态机循环中，调用 **`rstpPrsFsm()`** 函数，重新对端口 P 进行角色选举，发现端口 P 现在变成了根端口，那么它在调用 **`rstpPrtFsm()`** 函数时，由于 **`if(port->role != port->selectedRole)`** 初次判断不通过，所以会调用 **`rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PORT);`** 函数，将 **`port->role`** 设置为 **`STP_PORT_ROLE_ROOT`**，并刷新 rrWhile 计时器为 rstpFwdDelay。后续第二次调用 **`rstpPrtFsm()`** 函数时，由于此时 **`port->role`** 和 **`port->selectedRole`** 已经相等了，所以会进入 **`rstpPrtRootPortFsm()`** 函数的 **`RSTP_PRT_STATE_ROOT_PORT`** 状态分支，由于 **`else if(!port->forward && !port->reRoot)`** 判断通过，因为此时只是将 AP 端口设置为根端口，没有进行数据转发，也没有进入 reRoot 流程。

>后续在 **`rstpPrtRootPortFsm()`** 状态机中，会判断 **`port->rrWhile != rstpFwdDelay(port)`**，如果根端口的 rrWhile 计时器不等于 rstpFwdDelay，则会刷新 rrWhile 计时器为 rstpFwdDelay。也就是前面所说的根端口会将 rrWhile 计时器维持在 FwdDelay 这一数值。AP 端口在初始化之后，rrWhile 始终为 0，reRoot 始终为 false。DP 端口在 P/A 协商过程中，rrWhile 也会被清 0（fsm 第二个 if 分支），reRoot 也会被清 false（fsm 第三个 if 分支）。因此只有初始根端口的 rrWhile 不为 0，本桥中初始为其它端口的 rrWhile 都为 0。

后续调用 **`rstpSetReRootTree(port->context);`** 函数，把全桥所有端口 reRoot 置 True。后续此新根端口 RP 想要进入转发状态必须满足两个条件，要么 **`port->fdWhile == 0`** 为真，要么 **`rstpReRooted(port) && port->rbWhile == 0 && rstpVersion(port->context)`** 为真。第一个条件是让计时器的时间走完，第二个条件主要是 **`rstpReRooted(port)`** 为真，表示全桥所有其它端口的 rrWhile 都已经归零，表示所有近期根端口都已被退役/解除，根端口 RP 才能进入学习状态，进而进入转发状态。

此时本桥之前根端口 RP 变为 AP 端口，其状态机函数 **`rstpPrtAlternatePortFsm()`** 会进入 **`RSTP_PRT_STATE_ALTERNATE_PORT`** 状态分支，由于 **`if(port->reRoot)`** 判断通过，所以会调用 **`rstpPrtAlternatePortChangeState(port, RSTP_PRT_STATE_ALTERNATE_PORT);`** 函数，将端口的 learn 和 forward 都设置为 false，确保端口不会进入转发状态。同时 rrWhile 计时器会被清 0，reBoot 设置为 false，表示端口已经稳定，退出 reRoot 流程，并且 synced 设置为 True。

本桥之前根端口 RP 变为 DP 端口，其状态机函数 **`rstpPrtDesignatedPortFsm()`** 会进入 **`RSTP_PRT_STATE_DESIGNATED_PORT`** 状态分支，由于 **`port->reRoot && port->rrWhile != 0`** 判断通过，所以会调用 **`rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_DISCARD);`** 函数，将端口的 learn 和 forward 都设置为 false，之后此 DP 端口会开始进行 P/A 协商，DP 发送 Proposal，然后接收 Agreement，并且将 agreed 设置为 true。DP 之前的角色 RP 的 agreed/synced 一直为 false。因此在变为 DP 端口后会进入 **`rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_SYNCED);`** 函数将 rrWhile 清零。

所以新的根端口 RP 会通过 **`rstpReRooted(port) && port->rbWhile == 0 && rstpVersion(port->context)`** 此判断条件，然后进入 **`LEARNING`** 状态，进而进入 **`FORWARDING`** 状态，实现快速收敛转发数据。

### 3.P/A 协商机制

Port Role Transitions 状态机为了让一个 Designated Port（指定端口）从非转发快速进入 Forwarding，会用到 **`Proposal/Agreement`** 的消息交换。

首先调用 **`rstpPrtFsm()`** 函数，如果端口目前的 **`port->role`** 是 Designated 和 **`port->selectedRole`** 相等，那么会进入 **`rstpPrtDesignatedPortFsm()`** 状态机函数的 **`RSTP_PRT_STATE_DESIGNATED_PORT`** 状态分支。由于此时端口 port 处于非转发状态，proposing 为 false，agreed 为 false，forward 为 false，所以会进入第 1 个 if 分支，调用 **`rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_PROPOSE);`** 函数，将 proposing 置为 true，触发 P/A 协商流程。同时将 newInfo 置为 true，接下来由 PTX 看到 newInfo 条件为真，就去发送 Proposal RSTP BPDU 报文。

最开始在状态机初始化函数 **`rstpFsmInit()`** 中，会调用 **`rstpPtxInit()`** 函数，进而调用 **`rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_INIT);`** 函数，将 ptx 的状态设置为 **`RSTP_PTX_STATE_TRANSMIT_INIT`**。后面继续调用 **`rstpPtxFsm()`** 函数时，由于 ptx 状态是 **`RSTP_PTX_STATE_TRANSMIT_INIT`**，所以会调用 **`rstpPtxChangeState(port, RSTP_PTX_STATE_IDLE);`** 函数，继续将 ptxState 状态设置为 **`RSTP_PTX_STATE_IDLE`**。接下来再次调用 **`rstpPtxFsm()`** 函数时，由于 ptx 状态是 **`RSTP_PTX_STATE_IDLE`**，并且端口的 newInfo 为 true（之前在 PRT 状态机中设置的），所以会调用 **`rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_RSTP);`** 函数，将 ptxState 状态设置为 **`RSTP_PTX_STATE_TRANSMIT_RSTP`**。接着调用 **`rstpPtxChangeState(port, RSTP_PTX_STATE_TRANSMIT_RSTP);`** 函数，将 newInfo 清零，表示这次的新消息已经发送完，然后调用 **`rstpTxRstp(port);`** 函数发送 RSTP BPDU 报文，并且如果 **`port->proposing`** 为 True，**`bpdu.flags |= RSTP_BPDU_FLAG_PROPOSAL;`**，将 Proposal 位设置为 1，表示这是一个 Proposal 报文，最后发送出去。

当根端口收到这个 Proposal 报文后，会调用 **`rstpRecordProposal(port);`** 函数，只有当收到的 BPDU 端口角色=Designated，且 Proposal 位=1 时，才把本根端口的 **`port->proposed = TRUE`**。随后调用 **`rstpPrtFsm()`** 函数时，由于端口的 role 是 Root，selectedRole 也是 Root，所以会进入 **`rstpPrtRootPortFsm()`** 函数的 **`RSTP_PRT_STATE_ROOT_PORT`** 状态分支。由于 proposed 为 true，agree 为 false，所以会进入第 1 个 if 分支，调用 **`rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_PROPOSED);`** 函数，将 proposed 设置为 false，并且继续调用 **`rstpSetSyncTree(port->context)`** 函数，将本桥所有端口的 sync 变量设置为 true。即 Root Port 收到 Proposal 后，先要求本桥所有相关端口进入同步流程，确保不会瞬间形成环路。

随后根端口想要进入 **`ROOT_AGREED`** 状态，需要 **`rstpAllSynced(context) && !port->agree`** 为真，或 **`proposed && agree`** 为真，前者表示所有端口都已经同步完成，后者表示本端口已经同意对端的 Proposal，后面又收到对端的 Proposal 报文。而 **`rstpAllSynced()`** 的判定非常关键，它主要要求 **`(port->synced == TRUE) || (port->role == ROOT)`**，也就是根端口发 Agreement 之前，必须等其他端口都把 synced 变成 TRUE（根端口自己 role==ROOT 例外）。

此时本桥上其他 DP 端口如果已经处于同步/安全状态，比如已经处于 agreed 状态（处于可以安全快速转发的证明），已经处于 synced 状态（处于同步完成的证明，因为本桥上可能同时有 2 个正在 P/A 协商的端口），或者本端口是边缘端口（edge port），或者本端口本身既不处于学习状态，也不处于转发状态。如果 DP 端口处于上述状态时，就没有必要进入丢弃状态，直接执行 **`rstpPrtDesignatedPortChangeState(port, RSTP_PRT_STATE_DESIGNATED_SYNCED);`** 函数，将 synced 置为 true，sync 设置为 false，rrWhile 设置为 0，表示本端口已经同步完成，可以安全转发数据。否则如果当 **`sync == 1 and synced == 0`**，并且这个 DP 当前还在 learn 或 forward（而且不是 edge 口）时，就会将该端口进入 **`RSTP_PRT_STATE_DESIGNATED_DISCARD`** 状态，确保端口不会转发数据，等待同步完成。

假设所有端口都已经同步完成，那么会调用 **`rstpPrtRootPortChangeState(port, RSTP_PRT_STATE_ROOT_AGREED);`** 函数，将根端口的 agree 置为 true，proposed 设置为 false，sync 设置为 false，触发 Agreement 流程。同时将 newInfo 置为 true，接下来由 PTX 看到 newInfo 条件为真，就去发送 Agreement RSTP BPDU 报文。

>这里解释为什么要将 proposed 设置为 false，因为根端口在本桥同步完成（agree 已经设置为 true）后，如果又收到对端的 Proposal 报文，那么会再次进入 **`RSTP_PRT_STATE_ROOT_PROPOSED`** 状态，将 proposed 设置为 false，避免再次调用 **`rstpSetSyncTree`** 函数，将本桥的所有端口重新进入 sync 状态。

DP 端口在收到对端发回的 Agreement 后，PIM 会在合适的分支里调用 **`rstpRecordAgreement()`** 函数，只有当运行在 RSTP 版本，且端口被判定为点到点链路，且收到的 BPDU Agreement 位=1 时，才把本 DP 端口的 **`port->agreed = TRUE`**，并清掉 **`port->proposing`**。随后调用 **`rstpPrtFsm()`** 函数时，由于端口的 role 是 Designated，selectedRole 也是 Designated，所以会进入 **`rstpPrtDesignatedPortFsm()`** 函数。接着由于 **`((port->fdWhile == 0 || port->agreed || port->operEdge) && (port->rrWhile == 0 || !port->reRoot) && !port->sync)`**，所以会快速进入 LEARNING 状态，进而进入 FORWARDING 状态，实现快速收敛转发数据。

>注意，如果不是 RP 端口而是 AP 端口收到 proposal 报文，那么也会发送 Agreement 报文给对端，从而让对端 DP 端口快速进入转发状态。但是由于调用 **`rstpSetSyncTree`** 函数将 AP 端口本身的 sync 也设置为 true，因此 **`(port->fdWhile != rstpForwardDelay(port) || port->sync || port->reRoot || !port->synced)`** 判断通过，会进入 **`RSTP_PRT_STATE_ALTERNATE_PORT`** 状态分支，将 sync 设置为 false，synced 设置为 true，表示已经同步，并且 AP 端口的 learn 和 forward 一直设置为 false，确保此 AP 端口不会进入转发状态。

### 4.Topology Change 通知

首先，TC 相关的收到事件入口是 **`rstpSetTcFlags()`**，当端口收到 RST BPDU 报文且 flags 里带 TC 时，就置 rcvdTC。这个 TC 标志位并不是直接发出去，而是交给 TCM 状态机去消费和驱动后续动作。接着，TCM 状态机在 **`LEARNING/ACTIVE`** 等状态里会持续观察 **`rcvdTc/rcvdTcn/rcvdTcAck/tcProp`** 等条件。

比如在 LEARNING 里，只要有任意一个置位（含 tcProp），就会再次进入 LEARNING，相当于消化/清理收到的事件标志；而当端口是 **`Root/Designated`**、处于转发且不是 edge 口时，TCM 会进入 **`DETECTED`**，把拓扑变化正式转换成要对外传播的动作序列。一旦进入 **`DETECTED`**（本端口检测到/需要触发 TC），CycloneSTP 做了三件非常关键的事：

- 调 **`rstpNewTcWhile(port)`**：如果 **`tcWhile==0`** 才启动一次 TC 窗口，并且在 **`sendRstp==TRUE`** 时，把 tcWhile 设为 **`HelloTime+1`** 且置 **`newInfo=TRUE`** 表示立刻触发发送；
- 调 **`rstpSetTcPropTree(port)`**：把除本端口之外的所有端口的任务广播给整桥其它端口；
- 再额外把本端口 **`newInfo=TRUE`**，确保本端口马上进入要发 BPDU 的路径；

然后是发 **`TC-bit`**，CycloneSTP 在构造 Config BPDU 时明确规定，只要端口 **`tcWhile != 0`** 就把 BPDU 的 TC 标志位置 1，所以 tcWhile 本质上就是在一段时间窗口内持续携带 TC 位的开关，而 newInfo 是现在就该发一帧出去的触发器。

最后是把事件扩散到全树并触发 fdbFlush，当其它端口因为 **`rstpSetTcPropTree()`** 被置了 **`tcProp=TRUE`** 后，在 TCM 的 ACTIVE 状态里会命中 **`tcProp && !operEdge`** 的条件并转到 PROPAGATING。进入 PROPAGATING 时，CycloneSTP 会再次 **`rstpNewTcWhile(port)`**，让该端口也进入 TC 窗口并且后续发出的 BPDU 也带 TC 位，同时把 **`fdbFlush=TRUE`** 并清掉 tcProp（避免重复传播）。这就是 tcProp 把事件扩散到全树以及 fdbFlush 刷新转发表的落地实现。

虽然 **`rstpSetTcPropTree()`** 会把需要传播拓扑变化的标记（tcProp）置到本桥除本端口外的所有端口上，但在 Cyclone 的 TCM 状态机里，并不是所有端口都会真正参与传播。在 ACTIVE 状态下，TCM 会先做筛选：如果端口不是 RP/DP，或者该端口是边缘口（operEdge 为真），就会被切回 LEARNING，相当于被排除在 TC 传播之外；只有满足参与条件的端口，才会在检测到 **`tcProp && !operEdge`** 时进入 PROPAGATING 状态。即 **`tcPropTree`** 先把要传播 TC 的事件扩散到本桥所有端口；但只有非边缘并且角色为 DP/RP（并处在 ACTIVE/转发相关路径上）的端口会进入 PROPAGATING，从而启动自己的 tcWhile，最终让它们发出去的 BPDU 带 TC 位。