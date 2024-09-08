# RocketMQ 消息消费详解之一

## 0 前言

在前面，已经简略地介绍过 RocketMQ 中消息消费时 Pull 消息的长轮询机制了，其主要的思路是：Consumer 如果第一次尝试 Pull 消息失败（比如：Broker 端没有可以消费的消息），并不立即给消费者客户端返回 Response 的响应，而是先 hold 住并且挂起请求。然后在 Broker 端，通过后台独立线程— PullRequestHoldService 重复尝试执行 Pull 消息请求来取消息。同时，另外一个 ReputMessageService 线程不断地构建 ConsumeQueue/IndexFile 数据，当有新消息到达时，就通知阻塞的长轮询线程，有新消息到达。通过这种长轮询机制，即可解决 Consumer 端需要通过不断地发送无效的轮询 Pull 请求，而导致整个 RocketMQ 集群中 Broker 端负载很高的问题。

