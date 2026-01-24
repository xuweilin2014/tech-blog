# STP 协议

```c{.line-numbers}
/* called under bridge lock */
static inline int br_is_root_bridge(const struct net_bridge *br)
{
    return !memcmp(&br->bridge_id, &br->designated_root, 8);
}

/**
  * 当端口 p 收到一帧 STP Configuration BPDU 时，它负责比较该新 BPDU 与端口 p 当前缓存的最优 BPDU 配置之间的优劣，
  * 判断这条新到的 BPDU 是否应当取代（supersede）端口内现有的 designated_root/designated_cost/designated_bridge/designated_port 信息。
  * 如果返回 1，说明新 BPDU 更优，会调用 br_record_config_information() 将该 BPDU 携带的关键信息保存到端口 P 中进行更新；
  * 如果返回 0，说明新 BPDU 不够优，不会更新端口 p 的 BPDU 缓存；
  */
static int br_supersedes_port_info(const struct net_bridge_port *p, const struct br_config_bpdu *bpdu)
{
    int t;
    // 在同一个 STP 实例里，STP 的收敛目标不是先找最小的根桥 cost，而是先让全网对同一个根桥达成一致，根桥由 Bridge ID 最小的设备当选
    // 如果根桥都不一样，那么比较到根桥的路径 cost 没有意义
    // 因此如果收到的 BPDU 宣告的根桥 ID 更小，说明该根桥更优，需要更新
    t = memcmp(&bpdu->root, &p->designated_root, 8);
    if (t < 0)
        return 1;
    else if (t > 0)
        return 0;
    
    // 根桥 ID 一致时，需要比较到根桥的路径代价大小
    // bpdu->root_path_cost 表示发送这条 BPDU 的桥/交换机到根桥的路径大小
    // p->designated_cost 表示端口 p 所在网段上，目前被认为最优的那一侧到根桥的路径大小，注意它不表示端口 p 到根桥的路径大小
    if (bpdu->root_path_cost < p->designated_cost)
        return 1;
    else if (bpdu->root_path_cost > p->designated_cost)
        return 0;
    
    // p->designated_bridge 表示在端口 p 所连接的那个网段上，目前被认为最优的那台桥的 Bridge ID
    // 接下来再比较发送 BPDU 的桥的 Bridge ID 和目前端口 p 认为所连接的网段上最优的那台桥的 Bridge ID
    t = memcmp(&bpdu->bridge_id, &p->designated_bridge, 8);
    if (t < 0)
        return 1;
    else if (t > 0)
        return 0;
    
    // 执行到这里，意味着这个新的 BPDU 和端口保存的最优 BPDU 信息的 root（根桥）、root_path_cost（到根桥最优路径大小）、bridge_id（桥 ID）均相同
    // 因此该 BPDU 来自同一个发送桥，这里返回 1 的主要目的是更新计时器，避免超时引发不必要的重算
    // 如果长期不接受来自同样发送桥的相同配置的 BPDU，计时器会过期，端口会认为对端信息失效，触发重新计算、甚至引起拓扑变化
    if (memcmp(&bpdu->bridge_id, &p->br->bridge_id, 8))
        return 1;

    if (bpdu->port_id <= p->designated_port)
        return 1;

    return 0;
}

/* called under bridge lock */
static void br_record_config_information(struct net_bridge_port *p, const struct br_config_bpdu *bpdu)
{
    p->designated_root = bpdu->root;
    p->designated_cost = bpdu->root_path_cost;
    p->designated_bridge = bpdu->bridge_id;
    p->designated_port = bpdu->port_id;
    p->designated_age = jiffies - bpdu->message_age;

    mod_timer(&p->message_age_timer, jiffies + (bpdu->max_age - bpdu->message_age));
}

/* called under bridge lock */
void br_configuration_update(struct net_bridge *br)
{
    // 在本桥内部选出 root port，并算出本桥 root port 到根的代价
    br_root_selection(br);
    // 遍历本桥内部的所有端口，对每个端口判断是否应该成为 designated port（指定端口）
    br_designated_port_selection(br);
}

/* called under bridge lock */
static void br_reply(struct net_bridge_port *p)
{
    br_transmit_config(p);
}

/* called under bridge lock */
void br_received_config_bpdu(struct net_bridge_port *p, const struct br_config_bpdu *bpdu)
{
    struct net_bridge *br;
    int was_root;

    p->stp_xstats.rx_bpdu++;

    br = p->br;
    // 判断本桥是否是根桥
    was_root = br_is_root_bridge(br);

    // 负责比较该新 BPDU 与端口 p 当前缓存的最优 BPDU 配置之间的优劣
    // 返回 1 表示新的 BPDU 更优，需要把这个更优 BPDU 的关键信息保存到端口 p 上，返回 0 则不需要更新
    if (br_supersedes_port_info(p, bpdu)) {
        // 将新的更优的 BPDU 的信息保存到端口 p 中
        br_record_config_information(p, bpdu);
        // br_configuration_update() 在本桥所有端口中选举出唯一的根端口（Root Port），并确定到根路径代价；
        // 随后再逐端口判断并选举各网段上的指定端口（Designated Port）
        br_configuration_update(br);
        br_port_state_selection(br);

        if (!br_is_root_bridge(br) && was_root) {
            timer_delete(&br->hello_timer);
            if (br->topology_change_detected) {
                timer_delete(&br->topology_change_timer);
                br_transmit_tcn(br);
                mod_timer(&br->tcn_timer, jiffies + br->bridge_hello_time);
            }
        }

        // 如果这条 BPDU 从本桥的 root_port 收到，要向下游扩散
        // 构造一帧 802.1D 的 Configuration BPDU 并通过本桥的指定端口发送扩散出去
        if (p->port_no == br->root_port) {
            br_record_config_timeout_values(br, bpdu);
            br_config_bpdu_generation(br);
            if (bpdu->topology_change_ack)
                br_topology_change_acknowledged(br);
        }
    // 如果端口 p 接收到的 BPDU 更差，则将本端口更优的 BPDU 从端口 P 扩散发送出去
    } else if (br_is_designated_port(p)) {
        br_reply(p);
    }
}

/* called under bridge lock */
static void br_root_selection(struct net_bridge *br)
{
    struct net_bridge_port *p;
    u16 root_port = 0;

    // list_for_each_entry 遍历 br->port_list 上的所有端口，root_port 保存遍历过程中选出的最佳 root port 候选，
    // 但是要将这个 root port 候选和后续遍历的 br->port_list 上的端口 p 进行比较（使用 br_should_become_root_port 方法），直到选出最佳 root port
    list_for_each_entry(p, &br->port_list, list) {
        if (!br_should_become_root_port(p, root_port))
            continue;

        if (p->flags & BR_ROOT_BLOCK)
            br_root_port_block(br, p);
        else
            root_port = p->port_no;
    }

    // 此时 root_port 变量保存的就是本桥中最佳的根端口
    br->root_port = root_port;

    // 本桥视自己为根桥
    if (!root_port) {
        // 本桥认为根桥就是自己，因此到根的路径代价为 0
        br->designated_root = br->bridge_id;
        br->root_path_cost = 0;
    } else {
        p = br_get_port(br, root_port);
        // 将本桥认定的 Root ID 更新为 Root Port 上最优 BPDU 宣告的根桥 ID
        br->designated_root = p->designated_root;
        // 计算并更新本桥到根的总代价
        br->root_path_cost = p->designated_cost + p->path_cost;
    }
}

// 在一台桥（bridge）内部，从所有可能通向更优根桥的端口里，挑出最该成为 Root Port（根端口）的那个端口
// 参数 p：表示正在被评估的端口（候选端口）
// 参数 root_port：当前已经选出来的最佳根端口的端口号（port_no）。如果还没选任何端口，root_port 是 0
// 端口 p 比当前的 root_port 更合适当根端口，则返回 1；端口 p 不如当前的 root_port，或者根本不具备候选资格，则返回 0
static int br_should_become_root_port(const struct net_bridge_port *p, u16 root_port)
{
    struct net_bridge *br;
    struct net_bridge_port *rp;
    int t;

    /*
     * p->designated_root：端口 p 所连接网段上，当前被判定为最优 BPDU 所宣告的根桥 ID
     * p->designated_cost：端口 p 所连接网段上，当前被判定为最优 BPDU 中所携带的到根桥的路径代价
     * p->designated_bridge：产生这条最优 BPDU 的发送桥 ID
     * p->designated_port：上述发送桥上用于发送该最优 BPDU 的端口
     */

    br = p->br;
    // 如果候选端口 p 被禁用或者端口 p 已经是指定端口，那么该端口不能作为根端口
    if (p->state == BR_STATE_DISABLED || br_is_designated_port(p))
        return 0;
    // 比较本桥 ID 和端口 p 所在网段最优 BPDU 里宣告的根桥 ID
    // 如果 br->bridge_id <= p->designated_root，也就是本桥 ID 比对方宣告的 Root ID 更小或相同，由于在 stp 里，根桥是 Bridge ID 最小者，
    // 因此当出现这种情况时，说明 p 端口学到的根并不比本桥更优（甚至根就是本桥自己），所以直接 return 0
    if (memcmp(&br->bridge_id, &p->designated_root, 8) <= 0)
        return 0;
    // 如果这是第一次找到有资格的端口时，先暂定 p 端口为 root_port 候选
    if (!root_port)
        return 1;
    
    rp = br_get_port(br, root_port);
    // 比较两端口 p 和 rp 所指向的根桥 ID，root id 更小者代表更优的根桥
    t = memcmp(&p->designated_root, &rp->designated_root, 8);
    if (t < 0)
        return 1;
    else if (t > 0)
        return 0;
    // 端口 p 和 rp 指向的根桥 id 相同时，比较本桥分别经由端口 p 和 rp 到根的总代价
    // p->path_cost 表示本端口链路代价，因此 p->designated_cost + p->path_cost 表示本桥通过端口 p 到达根桥的累计路径代价
    // 累计代价更小的端口更适合作为 Root Port。
    if (p->designated_cost + p->path_cost < rp->designated_cost + rp->path_cost)
        return 1;
    else if (p->designated_cost + p->path_cost > rp->designated_cost + rp->path_cost)
        return 0;
    // 到根桥的路径代价 rpc 相同，继续比较上行设备的 bid 大小，所连上行设备更小的端口作为根端口
    t = memcmp(&p->designated_bridge, &rp->designated_bridge, 8);
    if (t < 0)
        return 1;
    else if (t > 0)
        return 0;
    // 如果上行设备的 BID 相同，那么比较上行设备的 PID，较小的座位根端口
    if (p->designated_port < rp->designated_port)
        return 1;
    else if (p->designated_port > rp->designated_port)
        return 0;

    if (p->port_id < rp->port_id)
        return 1;

    return 0;
}

/* called under bridge lock */
static void br_designated_port_selection(struct net_bridge *br)
{
    struct net_bridge_port *p;

    // 遍历本桥中的所有端口，对每个未禁用的端口执行判定，若判定为真，则把该端口切换/收敛到 Designated Port 状态
    list_for_each_entry(p, &br->port_list, list) {
        if (p->state != BR_STATE_DISABLED && br_should_become_designated_port(p))
            br_become_designated_port(p);
    }
}

/* called under bridge lock */
// 判断本桥在这个端口所属的网段上，应该不应该将端口 p 切换成为 Designated Port（指定端口）
// 如果返回 1，表明这个端口应该成为 DP；如果返回 0，表明这个端口不应该成为 DP
static int br_should_become_designated_port(const struct net_bridge_port *p)
{
    struct net_bridge *br;
    int t;

    br = p->br;
    // 如果端口 p 已经是指定端口了，直接返回 1
    if (br_is_designated_port(p))
        return 1;
 
    if (memcmp(&p->designated_root, &br->designated_root, 8))
        return 1;
    // 如果根桥的 root id 保持一致之后，比较谁更靠近根桥
    // br->root_path_cost 表示本桥到根桥的代价，p->designated_cost 表示端口 p 所连接网段上，当前被判定为最优 BPDU 中所携带的到根桥的路径代价
    // 如果本桥到根桥的路径代价更小，说明本桥更靠近根，因此本桥的端口 p 就该是 DP
    // 注意这里没有把 p->path_cost 加进去。原因是指定端口的比较比的是哪个桥更靠近根，也就是比较桥本身到根的代价；path_cost 是从本端口跨到对端的一跳代价，那是在选 root port 时才加的。
    if (br->root_path_cost < p->designated_cost)
        return 1;
    else if (br->root_path_cost > p->designated_cost)
        return 0;

    t = memcmp(&br->bridge_id, &p->designated_bridge, 8);
    if (t < 0)
        return 1;
    else if (t > 0)
        return 0;

    if (p->port_id < p->designated_port)
        return 1;

    return 0;
}

/* called under bridge lock */
void br_become_designated_port(struct net_bridge_port *p)
{
    struct net_bridge *br;

    br = p->br;
    p->designated_root = br->designated_root;
    p->designated_cost = br->root_path_cost;
    p->designated_bridge = br->bridge_id;
    p->designated_port = p->port_id;
}

/* called under bridge lock */
void br_config_bpdu_generation(struct net_bridge *br)
{
    struct net_bridge_port *p;
    // 遍历 br 上的所有端口，如果端口 p 没有被禁用并且是指定端口，就构造一帧 802.1D 的 Configuration BPDU 并发送出去
    list_for_each_entry(p, &br->port_list, list) {
        if (p->state != BR_STATE_DISABLED && br_is_designated_port(p))
            br_transmit_config(p);
    }
}

/* called under bridge lock */
// 在端口 p 上构造一帧 802.1D 的 Configuration BPDU 并发送。它把本桥当前认定的根是谁、到根的路径大小、本桥 ID、该 BPDU 哪个端口发以及 STP 的计时参数打包进 BPDU 并发送
void br_transmit_config(struct net_bridge_port *p)
{
    struct br_config_bpdu bpdu;
    struct net_bridge *br;

    if (timer_pending(&p->hold_timer)) {
        p->config_pending = 1;
        return;
    }

    br = p->br;

    bpdu.topology_change = br->topology_change;
    bpdu.topology_change_ack = p->topology_change_ack;
    bpdu.root = br->designated_root;
    bpdu.root_path_cost = br->root_path_cost;
    bpdu.bridge_id = br->bridge_id;
    bpdu.port_id = p->port_id;
    if (br_is_root_bridge(br))
        bpdu.message_age = 0;
    else {
        struct net_bridge_port *root = br_get_port(br, br->root_port);
        bpdu.message_age = (jiffies - root->designated_age) + MESSAGE_AGE_INCR;
    }
    bpdu.max_age = br->max_age;
    bpdu.hello_time = br->hello_time;
    bpdu.forward_delay = br->forward_delay;

    if (bpdu.message_age < br->max_age) {
        br_send_config_bpdu(p, &bpdu);
        p->topology_change_ack = 0;
        p->config_pending = 0;
        if (p->br->stp_enabled == BR_KERNEL_STP)
            mod_timer(&p->hold_timer, round_jiffies(jiffies + BR_HOLD_TIME));
    }
}

/**
  * 处理端口 p 收到的 TCN BPDU 报文，802.1D 里，TCN BPDU 由检测到拓扑变化的非根桥沿着生成树向根桥方向逐跳上报，
  * 通常由非根桥的 root port 发送出去（朝根桥方向），经过每一级桥的指定端口 DR 接收、确认（返回 TCA 报文），并继续向根桥上报
  */
void br_received_tcn_bpdu(struct net_bridge_port *p)
{
    p->stp_xstats.rx_tcn++;

    // 仅当该端口是 DR 时，才处理接收到的 TCN 报文
    if (br_is_designated_port(p)) {
        br_info(p->br, "port %u(%s) received tcn bpdu\n", (unsigned int) p->port_no, p->dev->name);
        // 1.如果本桥就是根桥，设置本桥的 TC 标志位（topology_change）为 1
        // 2.如果本桥是非根桥，通过本桥的 root port 发送 TCN，向根桥上报
        br_topology_change_detection(p->br);
        // 将端口 p 的 topology_change_ack 标志位设置为 1，然后立刻发送 Config BPDU，用来携带 TCA 标志位，并且将 topology_change_ack 标志位清零
        br_topology_change_acknowledge(p);
    }
}

void br_topology_change_detection(struct net_bridge *br)
{
    int isroot = br_is_root_bridge(br);

    if (br->stp_enabled != BR_KERNEL_STP)
        return;
    br_info(br, "topology change detected, %s\n", isroot ? "propagating" : "sending tcn bpdu");

    // 如果本桥市根桥的话
    if (isroot) {
        // 将本桥的 TC 标志位设置为 1，也就是 br->topology_change 设置为 1，后续发出的 config BPDU 中的 TC 位设置为 1，用于通知全网刷新转发表
        // 1.本桥调用 __br_set_topology_change 立刻会缩短转发表的老化时间，并且根桥周期性在 hello timer 到期之后，调用 br_config_bpdu_generation，
        // 然后调用 br_transmit_config，构造 TC 位置为 1 的 config BPDU 报文从所有指定端口扩散发送出去 
        // 2.其它桥接收到 TC 位置为 1 的 config BPDU 报文后，在 br_received_config_bpdu 函数中，调用 br_record_config_timeout_values 将本桥的
        // TC 标志位设置为 1，并且继续构造 TC 位置为 1 的 config BPDU 报文从所有指定端口扩散发送出去
        __br_set_topology_change(br, 1);
        mod_timer(&br->topology_change_timer, jiffies + br->bridge_forward_delay + br->bridge_max_age);
    // 如果本桥不是根桥的话    
    } else if (!br->topology_change_detected) {
        // 通过本桥的 root port 向根桥继续发送 TCN 报文
        br_transmit_tcn(br);
        mod_timer(&br->tcn_timer, jiffies + br->bridge_hello_time);
    }

    br->topology_change_detected = 1;
}

// 设置 br->topology_change 标志位（TC 状态），并在状态变化时调整 MAC 地址老化时间 ageing_time
void __br_set_topology_change(struct net_bridge *br, unsigned char val)
{
    unsigned long t;
    int err;

    if (br->stp_enabled == BR_KERNEL_STP && br->topology_change != val) {
        /* On topology change, set the bridge ageing time to twice the
         * forward delay. Otherwise, restore its default ageing time.
         */
        if (val) {
            t = 2 * br->forward_delay;
            br_debug(br, "decreasing ageing time to %lu\n", t);
        } else {
            t = br->bridge_ageing_time;
            br_debug(br, "restoring ageing time to %lu\n", t);
        }

        err = __set_ageing_time(br->dev, t);
        if (err)
            br_warn(br, "error offloading ageing time\n");
        else
            br->ageing_time = t;
    }

    br->topology_change = val;
}

static void br_record_config_timeout_values(struct net_bridge *br, const struct br_config_bpdu *bpdu)
{
    br->max_age = bpdu->max_age;
    br->hello_time = bpdu->hello_time;
    br->forward_delay = bpdu->forward_delay;
    // 设置 br->topology_change 标志位（TC 状态），并在状态变化时调整 MAC 地址老化时间 ageing_time
    __br_set_topology_change(br, bpdu->topology_change);
}

// 非根桥向根桥方向上报拓扑变化从 root port 发送 TCN BPDU
void br_transmit_tcn(struct net_bridge *br)
{
    struct net_bridge_port *p;
    p = br_get_port(br, br->root_port);
    if (p)
        br_send_tcn_bpdu(p);
    else
        br_notice(br, "root port %u not found for topology notice\n", br->root_port);
}

void br_send_tcn_bpdu(struct net_bridge_port *p)
{
    unsigned char buf[4];

    if (p->br->stp_enabled != BR_KERNEL_STP)
        return;

    buf[0] = 0;
    buf[1] = 0;
    buf[2] = 0;
    buf[3] = BPDU_TYPE_TCN;
    br_send_bpdu(p, buf, 4);

    p->stp_xstats.tx_tcn++;
}

static void br_topology_change_acknowledge(struct net_bridge_port *p)
{
    // 将本端口的 TCA 标志位（topology_change_ack）设置为 1
	p->topology_change_ack = 1;
    // 构造 bpdu 报文时把 bpdu.topology_change_ack 设置为 1，并且把此 config BPDU 报文从端口 p 发送出去，并且将 topology_change_ack 清零，只应答一次
	br_transmit_config(p);
}
```

STP 接收 Config BPDU 的主流程如下所示：

<div style="display:flex; justify-content:center;">
  <div style="zoom:0.9;">
      <div style="text-align:center; color:#d00; font-weight:750; margin:0 0 8px 0;">STP 接收 Config BPDU 主流程</div>
    <pre class="mermaid">
%%{init: {'flowchart': {'useMaxWidth': false, 'nodeSpacing': 15, 'rankSpacing': 30, 'useMaxHeight': false}, 'themeVariables': {'fontSize': '12px'}}}%%
flowchart TD
A[端口 p 收到 config BPDU] --> B{调用 br_supersedes_port_info 函数判断接收到的 BPDU 报文是否比端口 p 缓存的 BPDU 报文更优}
B -- 是 --> C[br_record_config_information 保存更优的 BPDU 信息到端口 p 中并刷新计时器]
C --> D[调用 br_configuration_update 函数]
D --> D1[br_root_selection 选 root port 并更新本桥根桥和  root_path_cost]
D --> D2[br_designated_port_selection 遍历本桥的所有端口判断并选 designated port]
D2 --> E[br_port_state_selection 更新端口状态，将根端口 RP 以及指定端口 DR 设置为转发状态，其他的端口设置为阻塞状态]
E -->  G{该 BPDU 是否从 root_port 收到}
G -- 是 --> H[br_record_config_timeout_values 同步定时参数]
H --> I[调用 br_config_bpdu_generation 函数]
I --> J[对本桥所有 designated port 调用 br_transmit_config 发送扩散 BPDU]
J --> K[结束]
G -- 否 --> K
B -- 否 --> L{端口 p 是否为指定端口}

L -- 是 --> M[调用 br_reply 函数从本端口 p 发送本桥更优 BPDU 报文]
M --> N[调用 br_transmit_config 函数构造并从端口 p 发送 BPDU 报文]
L -- 否 --> K
    </pre>
  </div>
</div>

在收到 BPDU 报文之后，在本桥中选出新的根端口的流程图如下：

<div style="display:flex; justify-content:center;">
    <div style="zoom:0.9;">
        <div style="text-align:center; color:#d00; font-weight:750; margin:0 0 8px 0;">本桥中选出新的根端口的流程</div>
            <pre class="mermaid">
%%{init: {'flowchart': {'useMaxWidth': false, 'nodeSpacing': 15, 'rankSpacing': 30, 'useMaxHeight': false}, 'themeVariables': {'fontSize': '12px'}}}%%
flowchart TD
S[开始 br_root_selection] --> I[root_port 设为 0]
I --> L[遍历本桥的所有端口 port_list]
L --> C[调用 br_should_become_root_port 比较候选端口 p 和 root_port 端口，从而判断候选端口 p 是否可以为根端口]
C --> G[比较候选端口 p 和 root_port 指向的根桥 ID、经由本端口到根桥的路径代价、上行设备的桥 ID 等]
G --> U[遍历结束]
U --> E0[root_port 更新为最终比较胜出的端口 p 的值]
E0 --> SET[本桥的根端口设置为 root_port]
SET --> Z{root_port 是否为 0}
Z -- 是 --> R0[1.说明本桥视自己为根桥;<br/>2.设置本桥的根桥字段设为本桥的 bridge_id;<br/>3.设置本桥到根桥的路径代价大小为 0]
R0 --> END[结束]
Z -- 否 --> GP[1.更新本桥的根桥 ID 为胜出根端口 p 的根桥字段;<br/>2.计算本桥到根桥的路径大小 = 端口 p 到根桥的路径 + 本端口到对端一跳的代价;]
GP --> END
        </pre>
    </div>
</div>

在收到 BPDU 报文之后，在本桥中遍历所有端口，设置指定端口的流程图如下所示：

<div style="display:flex; justify-content:center;">
    <div style="zoom:1;">
        <div style="text-align:center; color:#d00; font-weight:750; margin:0 0 8px 0;">本桥中选出新的根端口的流程</div>
            <pre class="mermaid">
%%{init: {'flowchart': {'useMaxWidth': false, 'nodeSpacing': 20, 'rankSpacing': 30}, 'themeVariables': {'fontSize': '12px'}}}%%
flowchart TD
S[开始执行 br_designated_port_selection 函数设置指定端口] --> L[遍历本桥的所有端口 port_list]
L --> C[如果遍历候选端口 p 没有被禁用，调用 br_should_become_designated_port 判断端口 p 是否应该成为 DP]
C --> G[1.比较根桥 ID 是否一致<br/>2.比较本桥到根桥的路径代价和端口 p 所连接网段上当前被判定为最优 BPDU 中所携带的到根桥的路径代价<br/>3.再比较桥 ID<br/>4.最后比较端口 ID]
G --> R{是否应该成为 DP}
R -- 是 --> U[调用 br_become_designated_port 将端口 p 设为指定端口并写入本桥的 designated_* 字段信息到端口 p 中]
U --> L
            </pre>
    </div>
</div>