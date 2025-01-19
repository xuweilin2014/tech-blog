## Redis 之延时队列

延时任务有别于定式任务，定式任务往往是固定周期的，有明确的触发时间。而延时任务的发生时间不是确定的，并且这个任务触发之后，延后一段时间才会被处理，比如订单的产生，订单产生的具体时间是不确定的，通常订单生成后，会有 30min 的时间让用户支付，如果用户没有支付的话，订单会被取消。也就是说，任务事件生成时并不想让消费者立即拿到，而是延迟一定时间后才接收到该事件进行消费。

延迟任务相关的业务场景如下：

- 场景一：在订单系统中，一个用户某个时刻下单之后通常有 30 分钟的时间进行支付，如果 30 分钟之内没有支付成功，那么这个订单将自动进行过期处理。
- 场景二：用户某个时刻通过手机远程遥控家里的智能设备在指定的时间进行工作。这时就可以将用户指令发送到延时队列，当指令设定的时间到了再将指令推送到只能设备。

下面，我们介绍一下各种方案来实现延时队列，包括：定时器轮询遍历数据库记录、JDK 的 DelayQueue、JDK ScheduledExecutorService、Redis 的 ZSet 实现以及 RabbitMQ 实现延时队列。

### 1.定时器轮询遍历数据库记录

这是比较常见的一种方式，所有的订单或者所有的命令一般都会存储在数据库中。我们会起一个线程定时去扫数据库或者一个数据库定时 Job，找到那些超时的数据，直接更新状态，或者拿出来执行一些操作。这种方式很简单，不会引入其他的技术，开发周期短。

如果数据量比较大，千万级甚至更多，插入频率很高的话，上面的方式在性能上会出现一些问题，查找和更新对会占用很多时间，轮询频率高的话甚至会影响数据入库。一种可以尝试的方式就是使用类似 TBSchedule 或 Elastic-Job 这样的分布式的任务调度加上数据分片功能，把需要判断的数据分到不同的机器上执行。

如果数据量进一步增大，那扫数据库肯定就不行了。另一方面，对于订单这类数据，我们也许会遇到分库分表，那上述方案就会变得过于复杂，得不偿失。

### 2.JDK 的 DelayQueue

Java 中的 DelayQueue 位于 java.util.concurrent 包下，作为单机实现，它很好的实现了延迟一段时间后触发事件的需求。由于是线程安全的它可以有多个消费者和多个生产者，从而在某些情况下可以提升性能。DelayQueue 本质是封装了一个 PriorityQueue，使之线程安全，加上 Delay 功能，也就是说，消费者线程只能在队列中的消息“过期”之后才能返回数据获取到消息，不然只能获取到 null。

之所以要用到 PriorityQueue，主要是需要排序。也许后插入的消息需要比队列中的其他消息提前触发，那么这个后插入的消息就需要最先被消费者获取，这就需要排序功能。PriorityQueue 内部使用最小堆来实现排序队列。队首的，最先被消费者拿到的就是最小的那个。使用最小堆让队列在数据量较大的时候比较有优势。使用最小堆来实现优先级队列主要是因为最小堆在插入和获取时，时间复杂度相对都比较好，都是 O(logN)。

下面例子实现了未来某个时间要触发的消息。我把这些消息放在 DelayQueue 中，当消息的触发时间到，消费者就能拿到消息，并且消费，实现处理方法。示例代码：

```java{.line-numbers}
/**
 * 定义放在延时队列中的对象，需要实现Delayed接口
 */
public class DelayedTask implements Delayed {

    private long _expireInSecond = 0;

    public DelayedTask(int delaySecond) {
        Calendar cal = Calendar.getInstance();
        cal.add(Calendar.SECOND, delaySecond);
        System.out.println("expire time: " + cal.getTime());
        this._expireInSecond = (cal.getTimeInMillis());
    }

    public long getDelay(TimeUnit unit) {
        Calendar calendar = Calendar.getInstance();

        //距离消息的expire_time(过期时间)的剩余时间
        return _expireInSecond - calendar.getTimeInMillis();
    }

    //DelayQueue在jdk中是使用优先队列来实现的，排序的依据就是
    //expire_time的大小，expire_time越小，则排序越靠前
    public int compareTo(Delayed o) {
        long d = (getDelay(TimeUnit.SECONDS) - o.getDelay(TimeUnit.SECONDS));
        return (d == 0 ? 0 : (d < 0 ? -1 : 1));
    }
} 
```

下面定义了三个延迟任务，分别是 10 秒，5 秒和 15 秒。依次入队列，期望 5 秒钟后，5 秒的消息先被获取到，然后每个 5 秒钟，依次获取到 10 秒数据和 15 秒的那个数据。

```java{.line-numbers}
public class DelayedQueueTest {
    public static void main(String[] args) throws InterruptedException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

        DelayQueue<DelayedTask> delayQueue = new DelayQueue<DelayedTask>();

        DelayedTask task1 = new DelayedTask(10);
        DelayedTask task2 = new DelayedTask(5);
        DelayedTask task3 = new DelayedTask(15);

        System.out.println();

        delayQueue.offer(task1);
        delayQueue.offer(task2);
        delayQueue.offer(task3);

        System.out.println(sdf.format(new Date()) + " start");

        while (delayQueue.size() != 0){
            DelayedTask task = delayQueue.poll();

            //如果队列中没有任务到expire_time(过期时间)，
            //则取出的task值为null
            if (task != null){
                System.out.println(sdf.format(new Date()));
            }

            Thread.sleep(1000);
        }
    }
} 
```

DelayQueue 是一种很好的实现方式，虽然是单机，但是可以多线程生产和消费，提高效率。拿到消息后也可以使用异步线程去执行下一步的任务。如果有分布式的需求可以使用 Redis 来实现消息的分发，如果对消息的可靠性有非常高的要求可以使用消息中间件。

### 3.JDK ScheduledExecutorService

JDK 自带的一种线程池，它能调度一些命令在一段时间之后执行，或者周期性的执行。文章开头的一些业务场景主要使用第一种方式，即，在一段时间之后执行某个操作。代码例子如下：

```java{.line-numbers}
public static void main(String[] args) {
    // TODO Auto-generated method stub
    ScheduledExecutorService executor = Executors.newScheduledThreadPool(100);
    
    for (int i = 10; i > 0; i--) {
        executor.schedule(new Runnable() {
    
            public void run() {
                // TODO Auto-generated method stub
                System.out.println(
                        "Work start, thread id:" + Thread.currentThread().getId() + " " + sdf.format(new Date()));
            }
    
        }, i, TimeUnit.SECONDS);
    }
} 
```

ScheduledExecutorService 的实现类 ScheduledThreadPoolExecutor 提供了一种并行处理的模型，简化了线程的调度。ScheduledExecutorService 比上面一种 DelayQueue 更加实用。因为，一般来说，使用 DelayQueue 获取消息后触发事件都会实用多线程的方式执行，以保证其他事件能准时进行。而 ScheduledThreadPoolExecutor 就是对这个过程进行了封装，让大家更加方便的使用。同时在加强了部分功能，比如定时触发命令。

### 4.Redis Zset

Redis 中的 ZSet 是一个有序的 Set，内部使用 HashMap 和跳表 (SkipList) 来保证数据的存储和有序，**<font color="red">HashMap 里放的是成员到 score 的映射，而跳跃表里存放的是所有的成员，排序依据是 HashMap 里存的 score</font>**，使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。

延时队列可以通过 Redis 的 zset( 有序列表) 来实现。我们将消息序列化成一个字符串作为 zset 的 value，这个消息的到期处理时间作为 score，然后用多个线程轮询 zset 获取到期的任务进行处理，多个线程是为了保障可用性，万一挂了一个线程还有其它线程可以继续处理。因为有多个线程，所以需要考虑并发争抢任务，确保任务不能被多次执行。RedisDelayingQueue 的代码如下：

```java{.line-numbers}
//redis的java客户端jedis不是线程安全的，所以通过JedisPool来给每一个线程
//分配一个jedis，向redis服务器发送命令
private static JedisPool jedisPool = new JedisPool();
private String queueName;

// fastjson 序列化对象中存在 generic 类型时，需要使用 TypeReference
private Type taskType = new TypeReference<Task<T>>(){}.getType();

public RedisDelayingQueue(String queueName){
    this.queueName = queueName;
}

public void addTask(Task<T> task, Jedis jedis){
    String strTask = JSON.toJSONString(task);
    //加上5s，表示这个消息至少过5s之后再做处理
    jedis.zadd(queueName, System.currentTimeMillis() + 5000, strTask);
}

public void loop(Jedis jedis){
    while (!Thread.interrupted()){
        Set<String> tasks = jedis.zrangeByScore(queueName, 0, System.currentTimeMillis(), 0, 1);
        if (tasks.isEmpty()){
            try {
                //在线程睡眠的时候，如果发生了中断，则退出循环
                Thread.sleep(500);
            } catch (InterruptedException e) {
                break;
            }
            continue;
        }
        String task = tasks.iterator().next();

        // zrem 方法是多线程多进程争抢任务的关键，它的返回值决定了当前实例有没有抢到任务，
        // 因为 loop 方法可能会被多个线程、多个进程调用，同一个任务可能会被多个进程线程抢到，
        // 通过 zrem 来决定唯一的属主。
        if (jedis.zrem(queueName, task) > 0){
            handle_task(task);
        }
    }
}

private void handle_task(String task) {
    Task<T> newTask = JSON.parseObject(task, taskType);
    System.out.println(newTask.getData());
} 
```

在上面的 loop 代码中，Redis 的 zrem 方法是多线程多进程争抢任务的关键，它的返回值决定了当前实例有没有抢到任务，因为 loop 方法可能会被多个线程、多个进程调用，同一个任务可能会被多个进程线程抢到，通过 zrem 来决定唯一的属主。同时，我们要注意一定要对 handle_task 进行异常捕获，避免因为个别任务处理问题导致循环异常退出。另外由于要使用到 Json 序列化，所以还需要 fastjson 库的支持。

```java{.line-numbers}
public static void main(String[] args){
    final RedisDelayingQueue<String> redisDelayingQueue =
            new RedisDelayingQueue<String>( "q-demo");
    
    //消息生产者
    Thread producer = new Thread(new Runnable() {
        public void run() {
            Jedis jedis = jedisPool.getResource();
            try{
                for (int i = 0; i < 10; i++){
                    String uuid = UUID.randomUUID().toString().substring(0,6);
                    redisDelayingQueue.addTask(new Task<String>(uuid, "task-" + uuid), jedis);
                }
            }finally {
                jedis.close();
            }
    
        }
    }, "producer");
    
    //消息消费者1
    Thread consumer1 = new Thread(new Runnable() {
        public void run() {
            Jedis jedis = jedisPool.getResource();
            try{
                redisDelayingQueue.loop(jedis);
            }finally {
                jedis.close();
            }
        }
    }, "consumer1");
    
    //消息消费者2
    Thread consumer2 = new Thread(new Runnable() {
        public void run() {
            Jedis jedis = jedisPool.getResource();
            try{
                redisDelayingQueue.loop(jedis);
            }finally {
                jedis.close();
            }
        }
    }, "consumer2");
    
    producer.start();
    consumer1.start();
    consumer2.start();
    
    try {
        producer.join();
        Thread.sleep(6000);
        consumer1.interrupt();
        consumer2.interrupt();
        consumer1.join();
        consumer2.join();
    } catch (InterruptedException e) {
    }finally {
        redisDelayingQueue.closeJedis();
    }
} 
```

在使用 zadd 添加延迟任务的时候，把 score 写成未来某个时刻的 unix 时间戳（在上面的代码中，所有的任务均一致设定为 5s 以后执行）。消费者使用 zrangeWithScores 获取优先级最高的（最早开始的的）任务。注意，zrangeWithScores 并不是取出来，只是看一下并不删除，类似于 Queue 的 peek 方法。程序对最早的这个消息进行验证，看是否到达要运行的时间，如果是则执行，然后删除 zset 中的数据。如果不是，则继续等待。

### 5.RabbitMQ 的延时任务

RabbitMQ 本身没有直接支持延迟队列功能，但是可以通过以下特性模拟出延迟队列的功能：

RabbitMQ 可以针对 Queue 和 Message 设置 x-message-tt，来控制消息的生存时间，如果超时，则消息变为 dead letter。RabbitMQ 针对队列中的消息过期时间有两种方法可以设置。

- 通过队列属性设置，队列中所有消息都有相同的过期时间。
- 对消息进行单独设置，每条消息 TTL 可以不同。