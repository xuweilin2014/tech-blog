```c{.line-numbers}
// 
void rstpRecordProposal(RstpBridgePort *port)
{
    uint8_t role;
    const RstpBpdu *bpdu;

    //Point to the received BPDU
    bpdu = &port->context->bpdu;

    //Decode port role
    role = bpdu->flags & RSTP_BPDU_FLAG_PORT_ROLE;

    //If the received Configuration Message conveys a Designated Port Role,
    //and has the Proposal flag is set, the proposed flag is set. Otherwise,
    //the proposed flag is not changed
    if(role == RSTP_BPDU_FLAG_PORT_ROLE_DESIGNATED)
    {
        if((bpdu->flags & RSTP_BPDU_FLAG_PROPOSAL) != 0)
        {
            port->proposed = TRUE;
        }
    }
}
```

```c{.line-numbers}
// rtsp_prt.c
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
```

```c{.line-numbers}
// rstp_fsm.c
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
```