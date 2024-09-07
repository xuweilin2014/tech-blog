# rocketmq 顺序消息详解

## 一、顺序消息简介

### 1 场景分析

**<font color="red">顺序消息是指消息的产生顺序和消费顺序是相同的</font>**。假设有个下单场景，每个阶段需要发邮件通知用户订单状态变化。首先用户付款完成时系统给用户发送订单已付款邮件，接着订单已发货时给用户发送订单已发货邮件，最后订单完成时给用户发送订单已完成邮件。

发送邮件的操作为了不阻塞订单主流程，可以通过消息队列来解耦，下游邮件服务器收到 mq 消息后发送具体邮件，已付款邮件、已发货邮件、订单已完成邮件这三个消息。下游的邮件服务器需要顺序消费这3个消息并且顺序发送邮件才有意义。否则就会出现已发货邮件先发出，已付款邮件后发出的情况。

实现上面所说的先进先出（FIFO）的队列看似很简单，但是放到分布式队列里，实现起来是有点复杂的，尤其是不可能为了一个顺序消息的功能而破坏了原来的架构。我们先看一下有哪些问题：

- 根据前面讲过的 RocketMQ 的架构，一条消息从 Producer 发到 Broker 的过程其实是有发送到一个 Broker 的集群，消息会分布到多个 Broker 的多个 Queue 上面。即使发送的时候是一条一条按顺序发的，也保证不了消息到达 Broker 的时间也是按照发送的顺序来的（可能是因为不同的 Broker 网络条件不同）。
- Broker 之间是没有数据交互的，也就是说 Broker A 收到一条 Producer 提交的消息，它并不知道之前那条消息被发到了哪个 Broker 的哪个 Queue 上，更别提知道之前那条消息是否消费成功了，所以依赖 Broker 来控制消息的顺序是很困难的。
- 为了提高并发能力，同一个 Group 下会有多个 Consumer，每个 Consumer 消费一部分 queue 的消息。所以，如果两条有顺序关系的消息分布在两个 queue 上，就有可能被 push 到两个 consumer 上，而consumer 之间也没有数据交互，依赖 consumer 做排序也是很难实现的。

顺序消费又分两种，全局顺序消费和局部顺序消费，不过 rocketmq 只实现了局部的顺序消费。

#### 1.1 全局顺序消费

什么是全局顺序消费？所有发到 RocketMQ 的消息都被按照其发送的顺序进行消费，类似数据库中的 binlog，需要严格保证全局操作的顺序性。

那么RocketMQ中如何做才能保证全局顺序消费呢？这就需要设置 topic 下读写队列数量为 1 。为什么要设置读写队列数量为 1 呢？假设读写队列有多个，消息就会存储在多个队列中，消费者负载时可能会分配到多个消费队列同时进行消费，多队列并发消费时，无法保证消息消费顺序性

那么全局顺序消费有必要么？A、B都下了单，B用户订单的邮件先发送，A的后发送，不行么？其实，大多数场景下，mq 下只需要保证局部消息顺序即可，即A的付款消息先于 A 的发货消息即可，A 的消息和 B 的消息可以打乱，这样系统的吞吐量会更好，**<font color="red">将队列数量置为 1，极大的降低了系统的吞吐量，不符合 mq 的设计初衷</font>**。

举个例子来说明局部顺序消费。假设订单 A 的消息为 A1，A2，A3，发送顺序也如此。订单 B 的消息为 B1，B2，B3，A 订单消息先发送，B 订单消息后发送。

消费顺序如下:
A1，A2，A3，B1，B2，B3 是全局顺序消息，严重降低了系统的并发度
A1，B1，A2，A3，B2，B3 是局部顺序消息，可以被接受
A2，B1，A1，B2，A3，B3 不可接收，因为 A2 出现在了 A1 的前面

#### 1.2 局部顺序消费

那么在 RocketMQ 里局部顺序消息又是如何怎么实现的呢？

要保证消息的顺序消费，有三个关键点：

- 消息顺序发送
- 消息顺序存储
- 消息顺序消费

第一点，消息顺序发送，多线程发送的消息无法保证有序性，因此，需要业务方在发送时，**<font color="red">针对同一个业务编号(如同一笔订单)的消息需要保证在一个线程内顺序发送，在上一个消息发送成功后，在进行下一个消息的发送</font>**。对应到mq中，消息发送方法就得使用同步发送，异步发送无法保证顺序性

第二点，消息顺序存储，mq 的 topic 下会存在多个 queue，**<font color="red">要保证消息的顺序存储，同一个业务编号的消息需要被发送到一个 queue 中</font>**。对应到 mq 中，需要使用 MessageQueueSelector 来选择要发送的 queue，即对业务编号进行 hash，然后根据队列数量对hash值取余，将消息发送到一个 queue 中

第三点，消息顺序消费，**<font color="red">要保证消息顺序消费，同一个 queue 就只能被一个消费者所消费，因此对 broker 中消费队列加锁是无法避免的</font>**。同一时刻，一个消费队列只能被一个消费者消费，消费者内部，也只能有一个消费线程来消费该队列。即，同一时刻，一个消费队列只能被一个消费者中的一个线程消费。

上面第一、第二点中提到，要保证消息顺序发送和消息顺序存储需要使用 mq 的同步发送和 MessageQueueSelector 保证，具体 Demo 会有体现。

### 2 消息顺序发送和消费示例

```java{.line-numbers}
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        //1、初始化一个 producer，使用 group name.
        MQProducer producer = new DefaultMQProducer("example_group_name");
        //2、启动 producer
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //3、新建一条消息，指定 topic，tag、key和body
            Message msg = new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            //4、提交消息，制定 queue 选择器和排序参数
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId);

            System.out.printf("%s%n", sendResult);
        }
        //5、关闭Producer
        producer.shutdown();
    }
}
```

从这个例子中我们发现，除了第 4 步之外，和发送普通消息没有任何区别。**<font color="red">也就是说在发送端，初始化一个 Producer 后，既可以发送普通消息，也可以用来发送顺序消息</font>**，只是调用的 send 方法的参数不同。这里还有一点要说明下，这个 topic 也没有任何不同的地方，我们回想下 topic 的创建参数就知道，topic 的属性只有名称、queue 的数量和 flag。没有字段标识 topic 是否是用来发 order 消息的。**<font color="red">这也从侧面说明了 Broker 无法知道消息是否是顺序消息</font>**。

第4步的参数，第2个参数是用户自定义 queue 选择器，第3个是排序参数。Demo 中的实现是直接使用排序参数对总的队列数取模，这样可以保证相同 orderId 的消息肯定会走同一个 queue。比如 mq 所有 broker 上总共有4个队列，订单号为1的消息走q1，同时订单号为5的也会走q1，这些消息都会按发送顺序被 consumer 处理。下面看下接口实现：

```java{.line-numbers}
public SendResult send(Message msg, MessageQueueSelector selector, Object arg)
        throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return send(msg, selector, arg, this.defaultMQProducer.getSendMsgTimeout());
}

public SendResult send(Message msg, MessageQueueSelector selector, Object arg, long timeout)
    throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendSelectImpl(msg, selector, arg, CommunicationMode.SYNC, null, timeout);
}

private SendResult sendSelectImpl(Message msg, MessageQueueSelector selector, Object arg, final CommunicationMode communicationMode,
    final SendCallback sendCallback, final long timeout ) throws Exception {
    long beginStartTime = System.currentTimeMillis();
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer);

    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        try {
            //使用用户自定义MessageSelector来选择Queue
            mq = selector.select(topicPublishInfo.getMessageQueueList(), msg, arg);
        } catch (Throwable e) {
            throw new MQClientException("select message queue throwed exception.", e);
        }

        long costTime = System.currentTimeMillis() - beginStartTime;
        if (timeout < costTime) {
            throw new RemotingTooMuchRequestException("sendSelectImpl call timeout");
        }
        if (mq != null) {
            return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, null, timeout - costTime);
        } else {
            throw new MQClientException("select message queue return null.", null);
        }
    }

    throw new MQClientException("No route info for this topic, " + msg.getTopic(), null);
} 
```

**<font color="red">在普通消息的发送逻辑中，queue 的选择会采用系统的负载均衡策略，默认是采用轮询的方式</font>**（同时会将 broker 的延时参数计算进去），然后再调用 sendKernel 方法。**<font color="red">而从上面的代码可以看出，queue 的选择直接就是回调的用户的实现 selector，获取到要发送到的消息队列 mq 之后，后面的逻辑就跟普通消息一模一样了</font>**，也是调用 sendKernel 方法真正将消息发送到 Broker 端。

### 3.RocketMQ 顺序消息的实现原理

在源码分析之前，先来思考下几个问题。前面已经提到实现消息顺序消费的关键点有三个，其中前两点已经明确了解决思路：

- 第一点，消息顺序顺序发送，可以由业务方在单线程使用同步发送消息的方式来保证
- 第二点，消息顺序存储，可以由业务方将同一个业务编号的消息发送到一个队列中来实现

还剩下第三点，消息顺序消费，实现消息顺序消费的关键点又是什么呢？举个例子，假设业务方针对某个订单发送了N个顺序消息，这N个消息都发送到了 Broker 服务端的一个队列中，假设消费者集群中有3个消费者，每个消费者中又是开了 N 个线程进行消费。

第一种情形，假设 3 个消费者同时拉取一个队列的消息进行消费，结果会怎么样？N 个消息可能会分配在 3 个消费者中进行消费，多机并行的情况下，消费能力的不同，无法保证这 N 个消息被顺序消费，所以得保证一个消费队列同一个时刻只能被一个消费者消费。

假设又已经保证了一个队列同一个时刻只能被一个消费者消费，那就能保证顺序消费了？同一个消费者多线程进行消费，同样会使得的 N 个消费被分配到 N 个线程中，一样无法保证消息顺序消费，所以还得保证一个队列同一个时刻只能被一个消费者中一个线程消费。

下面顺序消息的源码分析中就针对这两点来进行分析，即

- 如何保证一个队列只被一个消费者消费
- 如何保证一个消费者中只有一个线程能进行消费

## 二、顺序消息实现源码分析

### 1 锁定 MessageQueue

首先对于第一个问题，如何保证一个队列只被一个消费者消费，先明确一点，同一个消费组中的消费者共同承担 topic 下所有消费者队列的消费，因此每个消费者需要定时重新负载并分配其对应的消费队列。消费队列存在于 broker 端，如果想保证一个队列被一个消费者消费，那么消费者在进行消息拉取消费时就必须先从 Broker 服务器申请队列锁。加锁的地方有两个：

- RebalanceImpl#lock：当有新的 MessageQueue 分配给 Consumer 时，就会调用在lock 方法，向 Broker 发送锁定这个 mq 的请求。只有加锁成功之后，才会创建这个 MessageQueue 的 PullRequest 请求，开始拉取消息。等待其他消费者释放该消息队列的锁，然后在下一次队列重新负载均衡的时候再尝试重新加锁。不过注意这里并没有将 ProcessQueue 的 locked 属性设置为 true。
- RebalanceImpl#lockAll: 在 ConsumeMessageOrderlyService 线程中会开启一个定时任务，每隔 20s 执行一次锁定分配给自己的消息消费队列。

#### 1.1 消息拉取

RocketMQ 消息拉取由 PullMessageService 线程负责，根据消息拉取任务循环拉取消息。

```java{.line-numbers}
// DefaultMQPushConsumerImpl#start
if (!this.consumeOrderly) {
    // ProcessQueue 中队列最大偏移量与最小偏离量的间距，不能超 consumeConcurrentlyMaxSpan = 2000，否则触发流控，
    if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
            // 打印警告日志
        }
        return;
    }
} else {
    // 如果 processQueue 被锁定的话
    // processQueue 被锁定（设置其 locked）属性是在 RebalanceImpl#lock/lockAll 方法中，在 consumer 向 Broker 发送锁定消息队列请求之后，
    // Broker 会返回已经锁定好的消息队列集合，接着就会依次遍历这些消息队列 mq，并且从缓存 processQueueTable 中获取到和 mq 对应的 ProcessQueue，
    // 并且将这些 ProcessQueue 的 locked 属性设置为 true
    // RebalanceImpl#lock 是对新分配给 consumer 的 mq 进行加锁，而 RebalanceImpl#lockAll 则是周期性的进行加锁
    if (processQueue.isLocked()) {
        // 该处理队列是第一次拉取任务，则首先计算拉取偏移量，然后向消息服务端拉取消息
        // pullRequest 第一次被处理的时候，lockedFirst 属性为 false，之后都为 true
        if (!pullRequest.isLockedFirst()) {
            final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
            // 如果 Broker 的 consume offset 小于 pullRequest 的 offset，则说明 Broker 可能比较繁忙，来不及更新消息队列的 offset
            boolean brokerBusy = offset < pullRequest.getNextOffset();
            log.info("the first time to pull message, so fix offset from broker");
            if (brokerBusy) {
                log.info("the first time to pull message, but pull request offset larger than broker consume offset");
            }

            pullRequest.setLockedFirst(true);
            // 拉取的偏移量以 Broker 为准
            pullRequest.setNextOffset(offset);
        }
    
    // 如果消息处理队列未被锁定，则延迟 3s 后再将 PullRequest 对象放入到拉取任务中
    } else {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
        log.info("pull message later because not locked in broker, {}", pullRequest);
        return;
    }
} 
```

如果消息处理队列 ProcessQueue 未被锁定，则延迟 3s 之后再将 PullRequest 对象放入到拉取任务中，如果该处理队列时第一次拉取任务，则首先计算拉取偏移量，然后向消息服务端拉取消息。**<font color="red">所以，对于顺序消息消费而言，如果要消费消息队列 mq 没有被锁定，那就不能从 Broker 端拉取消息</font>**。

#### 1.2 消息队列负载

```java{.line-numbers}
// RebalanceImpl#updateProcessQueueTableInRebalance
private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet, final boolean isOrder) {
    boolean changed = false;

    Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<MessageQueue, ProcessQueue> next = it.next();
        MessageQueue mq = next.getKey();
        ProcessQueue pq = next.getValue();

        if (mq.getTopic().equals(topic)) {
            // 不再消费这个 MessageQueue 的消息，也就是说经过负载均衡的分配策略之后，分配给这个 consumer 的消息队列发生了变化
            if (!mqSet.contains(mq)) {
                pq.setDropped(true);
                if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                    it.remove();
                    changed = true;
                    log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
                }
            // 超过max idle时间    
            } else if (pq.isPullExpired()) {
                // ignore code
            }
        }
    }

    List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
    for (MessageQueue mq : mqSet) {
        // 如果是新加入的 MessageQueue，也就是说新分配给这个 consumer 的 MessageQueue
        if (!this.processQueueTable.containsKey(mq)) {
            // 如果是顺序消息，对于新分配的消息队列，首先尝试向 Broker 发起锁定该消息队列的请求
            // 如果返回加锁成功则创建该消息队列的拉取请求，否则直接跳过。等待其他消费者释放该消息队列的锁，然后在下一次队列重新负载均衡的时候
            // 再尝试重新加锁
            if (isOrder && !this.lock(mq)) {
                log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
                continue;
            }

            // 从内存中移除该消息队列的消费进度
            this.removeDirtyOffset(mq);
            // 为新的 MessageQueue 初始化一个 ProcessQueue，用来缓存收到的消息
            ProcessQueue pq = new ProcessQueue();
            // 从磁盘中读取该消息队列的消费进度
            long nextOffset = this.computePullFromWhere(mq);
            if (nextOffset >= 0) {
                ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
                if (pre != null) {
                    log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
                } else {
                    // 对于新加的 MessageQueue，初始化一个 PullRequest，并且将其加入到 pullRequestList 中
                    // 在一个 JVM 进程中，同一个消费组中同一个队列只会存在一个 PullRequest 对象
                    log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                    PullRequest pullRequest = new PullRequest();
                    pullRequest.setConsumerGroup(consumerGroup);
                    pullRequest.setNextOffset(nextOffset);
                    pullRequest.setMessageQueue(mq);
                    pullRequest.setProcessQueue(pq);
                    pullRequestList.add(pullRequest);
                    changed = true;
                }
            } else {
                log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
            }
        }
    }

    // 分发 Pull Request 到 PullMessageService 中的 pullRequestQueue 中以便唤醒 PullMessageService 线程
    this.dispatchPullRequest(pullRequestList);

    return changed;
} 
```

RocketMQ 首先需要通过 RebalanceService 线程首先对消息队列进行负载，集群模式下同一个消费组内的消费者共同承担起订阅主题下消息队列的消费，同一个消息消费队列在同一时刻只会被消费组内的一个消费者消费，而一个消费者同一时刻可以分配多个消费队列。

从上面的代码可以看出，如果经过消息队列重新负载（分配）之后，如果当前 Consumer 分配到新的消息队列，那么就会首先尝试向 Broker 发起锁定该消息队列的请求，如果锁定成功，则创建一个 PullRequest 开始拉取消息。如果锁定失败，则会等待其他消费者释放该消息队列的锁，然后在下一次队列重新负载均衡的时候，再尝试加锁。不过这里需要注意的是，如果向 Broker 发起锁定请求成功之后，并没有设置 ProcessQueue 的 locked 属性为 true。lock 方法的具体代码如下：

```java{.line-numbers}
// RebalanceImpl#lock
public boolean lock(final MessageQueue mq) {
    // 根据 brokerName 找到 broker 集群中 master 结点
    FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(), MixAll.MASTER_ID, true);
    if (findBrokerResult != null) {
        LockBatchRequestBody requestBody = new LockBatchRequestBody();
        requestBody.setConsumerGroup(this.consumerGroup);
        requestBody.setClientId(this.mQClientFactory.getClientId());
        // 要加锁的 MessageQueue，类型是 set
        requestBody.getMqSet().add(mq);

        try {
            // 返回加锁成功的 MessageQueue 集合
            Set<MessageQueue> lockedMq = this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ(findBrokerResult.getBrokerAddr(), requestBody, 1000);
            // 设置消息处理队列锁定成功。锁定消息队列成功，可能本地没有消息处理队列，设置锁定成功会在 lockAll()方法
            for (MessageQueue mmqq : lockedMq) {
                ProcessQueue processQueue = this.processQueueTable.get(mmqq);
                if (processQueue != null) {
                    processQueue.setLocked(true);
                    processQueue.setLastLockTimestamp(System.currentTimeMillis());
                }
            }
            // 根据加锁成功的 mqSet 中是否包含传入的 mq 来判断是否加锁成功
            boolean lockOK = lockedMq.contains(mq);
            log.info("the message queue lock {}, {} {}",lockOK ? "OK" : "Failed", this.consumerGroup, mq);
            return lockOK;
        } catch (Exception e) {
            log.error("lockBatchMQ exception, " + mq, e);
        }
    }

    return false;
} 
```

通过 MQClientAPIImpl 向 Broker 发送锁定请求。如果锁定新增的消息队列 mq 成功，可是本地没有对应的消息处理队列 ProcessQueue，也就是上面所说的并没有将 ProcessQueue 的 locked 属性设置为 true。

### 2 消息消费

顺序消息消费的核心类是 ConsumeMessageOrderlyService，我们来看一下 ConsumeMessageOrderlyService 的 start 方法：

```java{.line-numbers}
// ConsumeMessageOrderlyService#start
public void start() {
    if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())) {
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                ConsumeMessageOrderlyService.this.lockMQPeriodically();
            }
        }, 1000 * 1, ProcessQueue.REBALANCE_LOCK_INTERVAL, TimeUnit.MILLISECONDS);
    }
} 
```

如果消费模式为集群模式，启动定时任务，默认每隔 20s 执行一次锁定分配给自己的消息消费队列。从之前 RebalanceImpl#updateProcessQueueTableInRebalance 方法中，集群模式下顺序消息消费对于新分配到的 mq，在创建拉取任务和锁定消息队列 mq 时并未将 ProcessQueue 的 locked 状态设置为 true，如果未锁定 ProcessQueue 的话，就不能执行消息拉取任务，会延时 3s 再去执行消息拉取。ConsumeMessageOrderlyService 以每秒 20s 频率对分配给自己的消息队列进行自动锁操作，从而可以消费加锁成功的消息队列。接下来分析一下加锁的具体实现：

```java{.line-numbers}
// RebalanceImpl#buildProcessQueueTableByBrokerName
private HashMap<String/* brokerName */, Set<MessageQueue>> buildProcessQueueTableByBrokerName() {
    HashMap<String, Set<MessageQueue>> result = new HashMap<String, Set<MessageQueue>>();
    for (MessageQueue mq : this.processQueueTable.keySet()) {
        Set<MessageQueue> mqs = result.get(mq.getBrokerName());
        if (null == mqs) {
            mqs = new HashSet<MessageQueue>();
            result.put(mq.getBrokerName(), mqs);
        }
        mqs.add(mq);
    }
    return result;
} 
```

将消息队列按照 Broker 组织成 Map<String/*brokerName */, Set<MessageQueue>>，方便下一步向 Broker 发送锁定消息队列的请求。

```java{.line-numbers}
// RebalanceImpl#lockAll
public void lockAll() {
    // 将消息队列按照 Broker 组织成 Map<String, Set<MessageQueue>>，方便下一步向 Broker 发送锁定消息队列的请求
    HashMap<String, Set<MessageQueue>> brokerMqs = this.buildProcessQueueTableByBrokerName();

    Iterator<Entry<String, Set<MessageQueue>>> it = brokerMqs.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, Set<MessageQueue>> entry = it.next();
        final String brokerName = entry.getKey();
        final Set<MessageQueue> mqs = entry.getValue();

        if (mqs.isEmpty())
            continue;

        FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(brokerName, MixAll.MASTER_ID, true);
        if (findBrokerResult != null) {
            LockBatchRequestBody requestBody = new LockBatchRequestBody();
            requestBody.setConsumerGroup(this.consumerGroup);
            requestBody.setClientId(this.mQClientFactory.getClientId());
            requestBody.setMqSet(mqs);
            try {
                // 向 Broker ( Master 主节点) 发送锁定消息队列的请求，该方法返回成功被当前消费者锁定的消息消费队列
                Set<MessageQueue> lockOKMQSet = this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ(findBrokerResult.getBrokerAddr(), requestBody, 1000);

                // 将成功锁定的消息消费队列相对应的处理队列 ProcessQueue 设置为锁定状态，同时更新加锁时间
                for (MessageQueue mq : lockOKMQSet) {
                    ProcessQueue processQueue = this.processQueueTable.get(mq);
                    if (processQueue != null) {
                        if (!processQueue.isLocked()) {
                            log.info("the message queue locked OK, Group: {} {}", this.consumerGroup, mq);
                        }
                        // 将 pq 设定成锁定状态
                        processQueue.setLocked(true);
                        // 更新 pq 的锁定时间
                        processQueue.setLastLockTimestamp(System.currentTimeMillis());
                    }
                }

                // 遍历当前处理队列中的消息消费队列，如果当前消费者不持有该消息队列的锁将处理队列锁状态设置为 false ，
                // 暂停该消息消费队列的消息拉取与消息消费
                for (MessageQueue mq : mqs) {
                    if (!lockOKMQSet.contains(mq)) {
                        ProcessQueue processQueue = this.processQueueTable.get(mq);
                        if (processQueue != null) {
                            processQueue.setLocked(false);
                            log.warn("the message queue locked Failed, Group: {} {}", this.consumerGroup, mq);
                        }
                    }
                }
            } catch (Exception e) {
                log.error("lockBatchMQ exception, " + mqs, e);
            }
        }
    }
} 
```

上面的 lockAll 代码主要完成以下三个功能：

- 向 Broker 的 Master 节点发送锁定消息队列的请求，该方法返回成功被当前消费者锁定的消息消费队列
- 遍历成功锁定的消息队列对应的 ProcessQueue，设置其 locked 属性为 false，并且更新 ProcessQueue 的锁定时间
- 遍历当前消费者在某个 Broker 上所分配到的消息队列，如果某个消息队列 mq 不在前面返回的成功锁定的消息消费队列中，那么就会将这个 mq 对应的 ProcessQueue 的 lcoked 属性设置为 false，暂停从该消息队列拉取消息

接下来，我们看看消息是如何被提交到 ConsumeMessageOrderlyService 线程中的。代码如下：

```java{.line-numbers}
// ConsumeMessageOrderlyService#submitConsumeRequest
public void submitConsumeRequest(final List<MessageExt> msgs, final ProcessQueue processQueue, final MessageQueue messageQueue, final boolean dispathToConsume) {
    // 构建消费任务，并且提交到消费线程池中
    if (dispathToConsume) {
        ConsumeRequest consumeRequest = new ConsumeRequest(processQueue, messageQueue);
        this.consumeExecutor.submit(consumeRequest);
    }
} 
```

从这里可以看出，顺序消息的 ConsumeRequest 消费任务不会直接消费本次拉取的消息 msgs，也就是构建 ConsumeRequest 对象时，msgs 完全被忽略了。而事实上，是在消息消费时从处理队列 ProcessQueue 中拉取的消息。接下来详细分析一下 ConsumeRequest 的 run 方法：

```java{.line-numbers}
// ConsumeRequest#run
public void run() {
    if (this.processQueue.isDropped()) {
        log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
        return;
    }

    // 根据消息队列 MessageQueue 获取一个对象 objLock，然后消息消费时先独占 objLock。顺序消息消费者的并发度为消息队列，
    // 也就是一个消息队列在同一时间，只会被一个消费线程池中的线程消费
    final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
    synchronized (objLock) {
        // 如果是广播模式的话，直接进入消费，无须锁定处理队列，因为相互直接无竞争;
        // 如果是集群模式，进入消息消费逻辑的前提条件 proceessQueue 已被锁定并且锁未超时
        if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
            || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {

            final long beginTime = System.currentTimeMillis();
            for (boolean continueConsume = true; continueConsume; ) {

                // 再次检查 processQueue 是否被锁定，如果没有，则延迟该消息队列的消费
                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && !this.processQueue.isLocked()) {
                    log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                // 同样，再次检查 processQueue 的锁是否超时，如果超时，也延迟该消息队列的消费
                if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    && this.processQueue.isLockExpired()) {
                    log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                    break;
                }

                long interval = System.currentTimeMillis() - beginTime;
                // 顺序消息的消费的处理逻辑，每一个 ConsumeRequest 消费任务不是以消费消息条数来计算的，而是根据消费时间，
                // 默认当消费时长超过 MAX_TIME_CONSUME_CONTINUOUSLY 之后，会延迟 10ms 再进行消息队列的消费，默认情况下，每消费 1 分钟休息 10ms
                if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
                    // 延迟 10ms 之后再消费
                    ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
                    break;
                }

                // 每次从处理队列中按顺序取出 consumeBatchSize 消息，如果未取到消息，也就是 msgs 为空，则设置 continueConsume 为 false ，本次消费任务结束。
                // 顺序消息消费时，从 ProceessQueue 取出的消息，会临时存储在 ProceeQueue 的 consumingMsgOrderlyTreeMap 属性中
                final int consumeBatchSize = ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();

                // 注意这里和并发处理消息不同，并发消费请求在 ConsumeRequest 创建时，已经设置好消费哪些消息
                List<MessageExt> msgs = this.processQueue.takeMessags(consumeBatchSize);

                if (!msgs.isEmpty()) {
                    final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);
                    ConsumeOrderlyStatus status = null;
                    ConsumeMessageContext consumeMessageContext = null;

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        // ignore code
                        // 执行消息消费钩子函数（消息消费之前 before 方法）
                        ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
                    }

                    long beginTimestamp = System.currentTimeMillis();
                    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
                    boolean hasException = false;
                    try {
                        // 申请消息消费锁，也就是一个 ProcessQueue 在同一时刻只能被一个消费线程处理
                        // 加锁完毕之后执行消息消费监听器，调用业务方具体消息监听器执行真正的消息消费处理逻辑，并通知 RocketMQ 消息消费结果
                        this.processQueue.getLockConsume().lock();
                        if (this.processQueue.isDropped()) {
                            log.warn("consumeMessage, the message queue not be able to consume, because it's dropped");
                            break;
                        }
                        status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
                    } catch (Throwable e) {
                        hasException = true;
                    } finally {
                        this.processQueue.getLockConsume().unlock();
                    }

                    if (null == status || ConsumeOrderlyStatus.ROLLBACK == status || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                        log.warn("consumeMessage Orderly return not OK");
                    }

                    // ignore code

                    if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
                    }

                    if (null == status) {
                        status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                    }

                    ConsumeMessageOrderlyService.this.getConsumerStatsManager().incConsumeRT(consumerGroup, messageQueue.getTopic(), consumeRT);
                    // 如果消息消费结果为 ConsumeOrderlyStatus.SUCCESS，执行 ProceeQueue 的 commit 方法，并返回待更新的消息消费进度
                    continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
                } else {
                    continueConsume = false;
                }
            }

        // 集群模式 CLUSTERING 下，如果未锁定处理队列，则延迟该队列的消息消费
        } else {
            if (this.processQueue.isDropped()) {
                log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                return;
            }
            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
        }
    }
}
```

ConsumeMessageOrderlyService 中 ConsumeRequest 的 run 方法主要完成的工作如下：

- 检查 ProcessQueue 是否已经被丢弃，是的话，直接返回
- 获取到一个消息队列 MessageQueue 的锁，保证一个消息队列 MessageQueue 在同一时刻只会被线程池中的一个线程所消费
- 检查 ProcessQueue 是否被锁定以及锁是否超时，如果是的话，延迟该消息队列的消费
- 当消费时长超过 **`MAX_TIME_CONSUME_CONTINUOUSLY`** 之后，延迟 10ms 再进行消息队列的消费，每消费 1 分钟休息 10ms
- 从处理队列中按顺序取出 consumeBatchSize 消息，如果未取到消息，也就是 msgs 为空，则设置 continueConsume 为 false ，本次消费任务结束
- 申请消费锁 lockConsume
- 加锁完毕之后执行消息消费监听器，调用业务方具体消息监听器执行真正的消息消费处理逻辑，并通知 RocketMQ 消息消费结果
- 处理消息消费的结果

### 3.顺序消息的重试机制

我们先来看看处理顺序消息消费结果的代码：

```java{.line-numbers}
// ConsumeMessageOrderlyService#processConsumeResult
public boolean processConsumeResult(List<MessageExt> msgs, final ConsumeOrderlyStatus status, final ConsumeOrderlyContext context, final ConsumeRequest consumeRequest) {
    boolean continueConsume = true;
    long commitOffset = -1L;
    if (context.isAutoCommit()) {
        switch (status) {
            case COMMIT:
            case ROLLBACK:
                log.warn("the message queue consume result is illegal, we think you want to ack these message");
            case SUCCESS:
                // 将这批消息从 processQueue 中移除，同时维护 processQueue 中的状态信息
                commitOffset = consumeRequest.getProcessQueue().commit();
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                break;
            case SUSPEND_CURRENT_QUEUE_A_MOMENT:
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
                if (checkReconsumeTimes(msgs)) {
                    // 消息消费重试，先将这批消息重新放入到 processQueue 的 msgTree 中，同时从 consumingMsgOrderlyTreeMap 中移除掉
                    consumeRequest.getProcessQueue().makeMessageToCosumeAgain(msgs);
                    // 延后一段时间再进行顺序消息消费
                    this.submitConsumeRequestLater(consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue(), context.getSuspendCurrentQueueTimeMillis());
                    continueConsume = false;
                } else {
                    commitOffset = consumeRequest.getProcessQueue().commit();
                }   
                break;
            default:
                break;
        }
    } else {
        // ignore code
    }

    if (commitOffset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
        this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), commitOffset, false);
    }

    return continueConsume;
} 

// ConsumeMessageOrderlyService#checkReconsumeTimes
private boolean checkReconsumeTimes(List<MessageExt> msgs) {
    boolean suspend = false;
    if (msgs != null && !msgs.isEmpty()) {
        for (MessageExt msg : msgs) {
            // 检查消息 msg 的最大重试次数，如果大于或者等于允许的最大重试次数，那么就将该消息发送到 Broker 端，
            // 该消息在消息服务端最终会进入到 DLQ 队列。
            if (msg.getReconsumeTimes() >= getMaxReconsumeTimes()) {
                MessageAccessor.setReconsumeTime(msg, String.valueOf(msg.getReconsumeTimes()));
                // 如果消息成功进入到 DLQ 队列，sendMessageBack 返回 true
                if (!sendMessageBack(msg)) {
                    suspend = true;
                    msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                }
            } else {
                suspend = true;
                msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
            }
        }
    }
    return suspend;
} 

// ProcessQueue#commit
public long commit() {
    try {
        this.lockTreeMap.writeLock().lockInterruptibly();
        try {
            Long offset = this.consumingMsgOrderlyTreeMap.lastKey();
            // 将 consumingMsgOrderlyTreeMap 中的该批消息从 ProcessQueue 中移除，
            // 更新 msgCount 和 msgSize 这两个变量
            msgCount.addAndGet(0 - this.consumingMsgOrderlyTreeMap.size());
            for (MessageExt msg : this.consumingMsgOrderlyTreeMap.values()) {
                msgSize.addAndGet(0 - msg.getBody().length);
            }
            this.consumingMsgOrderlyTreeMap.clear();
            // 从这里可以看出，offset 表示消息消费队列的逻辑偏移量，类似于数组的下标，代表第 n 个 ConsumeQueue 条目
            if (offset != null) {
                return offset + 1;
            }
        } finally {
            this.lockTreeMap.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("commit exception", e);
    }

    return -1;
} 

// ProcessQueue#makeMessageToCosumeAgain
public void makeMessageToCosumeAgain(List<MessageExt> msgs) {
    try {
        this.lockTreeMap.writeLock().lockInterruptibly();
        try {
            for (MessageExt msg : msgs) {
                // 将消息从 consumingMsgOrderlyTreeMap 中移除掉
                this.consumingMsgOrderlyTreeMap.remove(msg.getQueueOffset());
                // 将消息重新保存到 msgTreeMap 中
                this.msgTreeMap.put(msg.getQueueOffset(), msg);
            }
        } finally {
            this.lockTreeMap.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("makeMessageToCosumeAgain exception", e);
    }
} 

// ProcessQueue#takeMessags
public List<MessageExt> takeMessags(final int batchSize) {
    List<MessageExt> result = new ArrayList<MessageExt>(batchSize);
    final long now = System.currentTimeMillis();
    try {
        this.lockTreeMap.writeLock().lockInterruptibly();
        this.lastConsumeTimestamp = now;
        try {
            if (!this.msgTreeMap.isEmpty()) {
                for (int i = 0; i < batchSize; i++) {
                    // 从 msgTreeMap 上读取一条消息，并且在读取的同时，会将这条消息从 msgTreeMap 中移除掉
                    Map.Entry<Long, MessageExt> entry = this.msgTreeMap.pollFirstEntry();
                    if (entry != null) {
                        result.add(entry.getValue());
                        // 将从 msgTreeMap 中获取到的消息保存到 consuimgMsgOrderlyTreeMap 中
                        consumingMsgOrderlyTreeMap.put(entry.getKey(), entry.getValue());
                    } else {
                        break;
                    }
                }
            }
            if (result.isEmpty()) {
                consuming = false;
            }
        } finally {
            this.lockTreeMap.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("take Messages exception", e);
    }
    return result;
} 
```

首先，顺序消费消息结果 (ConsumeOrderlyStatus) 有四种情况:

- SUCCESS：消费成功并且提交。
- ROLLBACK：消费失败，消费回滚。
- COMMIT：消费成功提交并且提交。
- SUSPEND_CURRENT_QUEUE_A_MOMENT：消费失败，挂起消费队列一会，稍后继续消费。

考虑到 ROLLBACK 、COMMIT 暂时只使用在 MySQL binlog 场景，官方将这两状态标记为 @Deprecated。当然，相应的实现逻辑依然保留。在并发消费场景时，如果消费失败，Consumer 会将消费失败消息发回到 Broker 重试队列，跳过当前消息，等待下次拉取该消息再进行消费。但是在完全严格顺序消费消费时，这样做显然不行，有可能会破坏掉消息消费的顺序性。也因此，消费失败的消息，会挂起队列一会，稍后继续消费。不过消费失败的消息一直失败，也不可能一直消费。当超过消费重试上限时，Consumer 会将消费失败超过上限的消息发回到 Broker 死信队列。

另外，在 ProcessQueue 中有 2 个属性，msgTreeMap 和 consumingMsgOrderlyTreeMap。这两个属性的类型都为 TreeMap<Long, MessageExt>，键为消息在 ConsumeQueue 中的偏移量，而 MessageExt 为消息实体。其中 msgTreeMap 是消息存储的容器，而 consumingMsgOrderlyTreeMap 则是 msgTreeMap 的一个子集，只用于顺序消费的过程。比如，在 ConsumeRequest 的 run 方法中，由于从 Broker 端拉取的消息会保存在  ProcessQueue 中，所以调用 ProcessQueue 的 takeMessages 方法获取要处理的消息，在获取的过程中，从 msgTreeMap 中读取一条消息（同时会将这条消息从 msgTreeMap 中移除掉），就会将这条消息保存到 consumingMsgOrderlyTreeMap 中。

**1 顺序消息消费成功（SUCCESS）**

当消息消费成功时（返回 SUCCESS），就会直接提交被用户成功处理过的消息集合 msgs，就是将这批消息从 ProcessQueue 中移除（确切地说是从前面提到过的 consumingMsgOrderlyTreeMap 容器中移除掉），并且还会维护 msgCount 和 msgSize 这两个变量。

**2 顺序消息消费失败（SUSPEND_CURRENT_QUEUE_A_MOMENT）**

ConsumeRequest#run 方法每次默认会从 ProcessQueue 中读取最多 32 个顺序消息进行消费，如果这些消息 msgs 消费失败，也就是结果为 **`SUSPEND_CURRENT_QUEUE_A_MOMENT`**，那么只要 msgs 中的一个消息还可以继续重试，也就是可以继续处理，checkReconsumeTimes 方法都会返回 true，也就是对这批消息都再进行一次重新消费。除非 msgs 中所有消息的消费重试次数都达到了上限，那么才会将 msgs 发送到 Broker 端的死信队列。**<font color="red">这也就说明，对顺序消息进行重新消费的时候，是一个批次读取的消息集合 msgs 一起进行重新消费而不是对某一个消息进行重新消费</font>**。

### 4.移除消息队列

集群模式下，Consumer 移除自己的消息队列时，会向 Broker 解锁该消息队列（广播模式下不需要）。核心代码如下：

```java{.line-numbers}
// RebalancePushImpl#removeUnnecessaryMessageQueue
public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq) {
    // 持久化消费进度，并且移除
    this.defaultMQPushConsumerImpl.getOffsetStore().persist(mq);
    this.defaultMQPushConsumerImpl.getOffsetStore().removeOffset(mq);
    // 顺序消费&集群模式，解锁该队列的锁定
    if (this.defaultMQPushConsumerImpl.isConsumeOrderly()
        && MessageModel.CLUSTERING.equals(this.defaultMQPushConsumerImpl.messageModel())) {
        try {
            if (pq.getLockConsume().tryLock(1000, TimeUnit.MILLISECONDS)) {
                try {
                    return this.unlockDelay(mq, pq);
                } finally {
                    pq.getLockConsume().unlock();
                }
            } else {
                log.warn("[WRONG]mq is consuming, so can not unlock it, {}. maybe hanged for a while, {}");
                pq.incTryUnlockTimes();
            }
        } catch (Exception e) {
            log.error("removeUnnecessaryMessageQueue Exception", e);
        }

        return false;
    }
    return true;
} 
```

在解除消息队列的锁定之前，会先获取到消息消费锁，避免和消息队列消费冲突。如果获取锁失败，则移除消息队列失败，等待下次重新分配消费队列时，再进行移除。如果未获得锁而进行移除，则可能出现另外的 Consumer 和当前 Consumer 同时消费该消息队列，导致消息无法严格顺序消费。从这里也可以看出，前面 ConsumeRequest 的 run 方法中设置两个锁：消息队列 MessageQueue 的 objLock 和消息消费锁 lockConsume，就是为了使得锁的粒度更小，也就是说，只要消息队列没有线程正在消费，就可以向 Broker 申请解除这个消息队列的锁定，即使这个时候另外一个线程可能仍然占有消息队列的 objLock 锁。

```java{.line-numbers}
// RebalancePushImpl#unlockDelay
private boolean unlockDelay(final MessageQueue mq, final ProcessQueue pq) {
    // 当 ProcessQueue 中还有消息时，延迟解锁 Broker 端中这个 ProcessQueue 中的消息队列锁
    if (pq.hasTempMessage()) {
        log.info("[{}]unlockDelay, begin {} ", mq.hashCode(), mq);
        this.defaultMQPushConsumerImpl.getmQClientFactory().getScheduledExecutorService().schedule(new Runnable() {
            @Override
            public void run() {
                log.info("[{}]unlockDelay, execute at once {}", mq.hashCode(), mq);
                RebalancePushImpl.this.unlock(mq, true);
            }
        }, UNLOCK_DELAY_TIME_MILLS, TimeUnit.MILLISECONDS);
    // 如果不存在消息，则直接解锁
    } else {
        this.unlock(mq, true);
    }
    return true;
} 
```

解锁 Broker 消息队列锁。如果消息处理队列存在剩余消息，则延迟解锁 Broker 消息队列锁；如果不存在消息，则直接解锁。