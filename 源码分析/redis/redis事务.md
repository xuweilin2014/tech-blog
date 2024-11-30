# Redis 事务

## 1.Redis 中的事务

严格来讲，Redis 的事务和其它关系型数据库中的事务是不一样的。**<font color="red">Redis 中的事务（transaction）是一组命令的集合，事务同命令一样都是 Redis 的最小执行单位，事务的原理是先将属于一个事务的命令发送给 Redis，然后再让 Redis 依次执行这些命令</font>**。

Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行的期间，服务器不会打断事物而改去执行其它客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才会去处理其他客户端的命令请求。

## 2.Redis 事务命令与用法

### 2.1.Redis 事务的相关命令

- WATCH 命令：在 MULTI 命令执行之前，将给出的 Keys 标记为监测态，作为事务执行的条件。
- UNWATCH 命令：取消当前事务中指定监控的 Keys，如果执行了 EXEC 或 DISCARD 命令，则无需再手工执行该命令了，因为在此之后，事务中所有被监控的 Keys 都将自动取消。
- MULTI 命令：**<font color="red">用于标记事务的开始，其后执行的命令都将被存入命令队列，直到执行 EXEC 时，这些命令才会被原子的执行</font>**。
- EXEC 命令：执行在一个事务内命令队列中的所有命令，同时将当前连接的状态恢复为正常状态，即非事务状态。如果在事务中执行了 WATCH 命令，那么只有当 WATCH 所监控的 Keys 没有被修改的前提下，EXEC 命令才能执行事务队列中的所有命令，否则 EXEC 将放弃当前事务中的所有命令。
- DISCARD 命令：回滚事务队列中的所有命令，同时再将当前连接的状态恢复为正常状态，即非事务状态。如果 WATCH 命令被使用，该命令将 UNWATCH 所有的 Keys。

我们可以通过 MULTI 命令开启一个事务，有关系型数据库开发经验的人可以将其理解为 BEGIN TRANSACTION 语句。在该语句之后执行的命令都将被视为事务之内的操作，最后我们可以通过执行 EXEC/DISCARD 命令来提交/ 回滚该事务内的所有操作。这两个 Redis 命令可被视为等同于关系型数据库中的 COMMIT/ROLLBACK 语句。

### 2.2.用法

使用 MULTI 命令显式开启 Redis 事务。该命令总是以 OK 回应。此时用户可以发出多个命令，Redis 不会执行这些命令，而是将它们排队。EXEC 被调用后，所有的命令都会被执行。而调用 DISCARD 可以清除事务中的 commands 队列并退出事务。

以下示例以原子方式，递增键 foo 和 bar：

```java{.line-numbers}
>MULTI
OK
>INCR foo
QUEUED
>INCR bar
QUEUED
>EXEC
1）（整数）1
2）（整数）1 
```

从上面的命令执行中可以看出，EXEC 返回一个数组，其中每个元素都是事务中单个命令的返回结果，而且顺序与命令的发出顺序相同。当 Redis 连接处于 MULTI 请求的上下文中时，所有命令将以字符串 QUEUED 作为回复，并在命令队列中排队。只有 EXEC 被调用时，排队的命令才会被执行，此时才会有真正的返回结果。

## 3.Redis 事务的实现

**<font color="red">一个事务从开始到结束通常会经历以下三个阶段：事务开始、命令入队、事务执行</font>**。

### 3.1.事务开始

MULTI 命令的执行标志着事务的开始：

```java{.line-numbers}
redis> MULTI
OK 
```

MULTI 命令可以将执行该命令的客户端从非事务状态切换至事务状态，这一切换是通过在客户端状态的 flags 属性中打开 REDIS_MULTI 标识来完成的。

### 3.2.命令入队

当一个客户端处于非事务状态时，这个客户端发送的命令会立即被服务器执行，与此不同的是，当一个客户端切换到事务状态之后，服务器会根据这个客户端发来的不同命令执行不同的操作：

- 如果客户端发送的命令为 EXEC 、 DISCARD 、 WATCH 、 MULTI 四个命令的其中一个，那么服务器立即执行这个命令。
- 与此相反，如果客户端发送的是其它的命令，那么服务器并不立即执行这个命令，而是将这个命令放入一个事务队列里面，然后向客户端返回 QUEUED 回复。

每个 Redis 客户端都有自己的事务状态，这个事务状态保存在客户端状态的 mstate 属性里面。事务状态包含一个事务队列，以及一个已入队命令的计数器（也可以说是事务队列的长度）：

```c{.line-numbers}
typedef struct redisClient {
    // ...
    // 事务状态
    multiState mstate;      /* MULTI/EXEC state */
} redisClient;

typedef struct multiState {
    // 事务队列，FIFO 顺序
    multiCmd *commands;
    // 已入队命令计数
    int count;
} multiState;
```

事务队列是一个 multiCmd 类型的数组，数组中的每个 multiCmd 结构都保存了一个已入队命令的相关信息，包括指向命令实现函数的指针，命令的参数，以及参数的数量，并且事务队列以先进先出（FIFO）的方式保存入队的命令： 较先入队的命令会被放到数组的前面，而较后入队的命令则会被放到数组的后面。

```c{.line-numbers}
typedef struct multiCmd {
    // 参数
    robj **argv;
    // 参数数量
    int argc;
    // 命令指针
    struct redisCommand *cmd;
} multiCmd; 
```

### 3.3 执行事务

当一个处于事务状态的客户端向服务器发送 EXEC 命令时，这个 EXEC 命令将立即被服务器执行： 服务器会遍历这个客户端的事务队列，执行队列中保存的所有命令，最后将执行命令所得的结果全部返回给客户端。

```c{.line-numbers}
def EXEC():
    # 创建空白的回复队列
    reply_queue = []

    # 遍历事务队列中的每个项
    # 读取命令的参数，参数的个数，以及要执行的命令
    for argv, argc, cmd in client.mstate.commands:

        # 执行命令，并取得命令的返回值
        reply = execute_command(cmd, argv, argc)

        # 将返回值追加到回复队列末尾
        reply_queue.append(reply)

    # 移除 REDIS_MULTI 标识，让客户端回到非事务状态
    client.flags &= ~REDIS_MULTI

    # 清空客户端的事务状态，包括：
    # 1）清零入队命令计数器
    # 2）释放事务队列
    client.mstate.count = 0
    release_transaction_queue(client.mstate.commands)

    # 将事务的执行结果返回给客户端
    send_reply_to_client(client, reply_queue) 
```

## 4.Watch 命令

### 4.1.Watch 命令简介

大家可能知道 redis 提供了基于 incr 命令来操作一个整数型数值的原子递增，那么我们假设如果 redis 没有这个 incr 命令，我们该怎么实现这个 incr 的操作呢？那么我们下面的正主 watch 就要上场了。正常情况下我们想要对一个整形数值做修改是这么做的( 伪代码实现)：

```c{.line-numbers}
val = GET mykey
val = val + 1
SET mykey $val 
```

但是上述的代码会出现一个问题，因为上面吧正常的一个 incr( 原子递增操作) 分为了两部分，那么在多线程( 分布式) 环境中，这个操作就有可能不再具有原子性了。研究过 java 的 juc 包的人应该都知道 cas，那么 redis 也提供了这样的一个机制，就是利用 watch 命令来实现的。

WATCH 命令是一个乐观锁（optimistic locking），它可以监视任意数量的数据库键，一旦其中有一个键被修改（或删除），之后的事务就不会执行。监控一直持续到 EXEC 命令（**<font color="red">事务中的命令是在 EXEC 之后才执行的，所以在 MULTI 命令后可以修改 WATCH 监控的键值</font>**）。我们可以利用 watch 命令来实现 Incr 指令，具体的实现如下：

```c{.line-numbers}
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC 
```

和此前代码不同的是，新代码在获取 mykey 的值之前先通过 WATCH 命令监控了该键，此后又将 set 命令包围在事务中，这样就可以有效的保证每个连接在执行 EXEC 之前，如果当前连接获取的 mykey 的值被其它连接的客户端修改，那么当前连接的 EXEC 命令将执行失败。这样调用者在判断返回值后就可以获悉 val 是否被重新设置成功。**<font color="red">由于 WATCH 命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以在一般的情况下我们需要在 EXEC 执行失败后重新执行整个函数。执行 EXEC 命令后会取消对所有键的监控</font>**。

### 4.2 Watch 命令的实现原理

每个 Redis 数据库都保存着一个 watched_keys 字典，这个字典的键是某个被 WATCH 命令监视的数据库键，而字典的值则是一个链表，链表中记录了所有监视相应数据库键的客户端：

```c{.line-numbers}
typedef struct redisDb {
    // ...
    dict *watched_keys;
    // ...
} redisDb; 
```

通过 watched_keys 字典，服务器可以清楚地知道哪些数据库键正在被监视，以及哪些客户端正在监视这些数据库键。通过执行 WATCH 命令，客户端可以在 watched_keys 字典中与被监视的键进行关联。下图就是一个 watched_keys 字典实例：

```c{.line-numbers}
watched_keys
   "name"   -> c1 -> c2
    "age"   -> c3
  "address" -> c2 -> c4 
```

所有对数据库进行修改的命令，比如 SET、LPUSH、SADD、ZREM、DEL、FLUSHDB 等等，在执行之后都会调用 multi.c/touchWatchKey 函数对 watched_keys 字典进行检查，查看是否有客户端正在监视刚刚被命令修改过的数据库键，如果有的话，那么 touchWatchKey 函数就会将监视被修改键的客户端的 REDIS_DIRTY_CAS 标识打开，表示该客户端的事务安全性已经被破坏。touchWatchKey 函数的定义可以用以下伪代码来描述：

```java{.line-numbers}
def touchWatchKey(db, key):

    # 如果键 key 存在于数据库的 watched_keys 字典中
    # 那么说明至少有一个客户端在监视这个 key
    if key in db.watched_keys:
        # 遍历所有监视键 key 的客户端
        for client in db.watched_keys[key]:
            # 打开标识
            client.flags |= REDIS_DIRTY_CAS 
```

当服务器接收到一个客户端发来的 EXEC 命令时，服务器会根据这个客户端是否打开了 **<font color="red">REDIS_DIRTY_CAS</font>** 标识来决定是否执行事务：

- 如果客户端的 REDIS_DIRTY_CAS 标识已经被打开，那么说明客户端所监视的键当中，至少有 一个键已经被修改过了，在这种情况下，客户端提交的事务已经不再安全，所以服务器会拒绝执行客户端提交的事务。
- 如果客户端的 REDIS_DIRTY_CAS 标识没有被打开，那么说明客户端监视的所有键都没有被修改过（或者客户端没有监视任何键），事务仍然是安全的，服务器将执行客户端提交的这个事务。

## 5.Redis 事务的 ACID 性质

### 5.1 原子性

事务具有原子性指的是，数据库将事务中的多个操作当作一个整体来执行，服务器要么就执行事务中的所有操作，要么就一个操作也不执行。而在 Redis 中，由于 Redis 的应用场景明显不是为了数据存储的高可靠而设计的，而是为了数据访问的高性能而设计，设计者为了简单性和高性能而部分放弃了原子性，**<font color="red">比如和其它传统的关系型数据库相比，Redis 不支持事物的回滚机制</font>**，即使事务队列中的某个命令在执行期间出现了错误，整个事务也会继续执行下去，直到将事务队列中的所有命令都执行完毕为止。出于以上考虑，Redis 的事务执行有以下特点：

- 命令在发送 EXEC 命令前被放入队列中
- 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行
- 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中

在事务的执行期间，可能会遇到两种错误：

1.在调用 EXEC 命令之前出现错误（COMMAND 排队失败），命令可能存在语法错误（参数数量错误，错误的命令名称…）：

客户端会在 EXEC 调用之前检测第一种错误。 通过检查排队命令的状态回复，如果命令使用 QUEUED 进行响应，则它已正确排队，否则 Redis 将返回错误。**<font color="red">如果排队命令时发生错误，大多数客户端将中止该事务并清除命令队列</font>**。下面这个是由于 INCR 命令的语法错误，将在调用 EXEC 之前被检测出来，并终止事务（version2.6.5+）。

```c{.line-numbers}
>MULTI
+OK
>INCR a b c
-ERR wrong number of arguments for 'incr' command 
```

2.在调用 EXEC 命令之后出现错误，比如使用错误的值对某个 key 执行操作（如针对 String 值调用 List 操作），EXEC 命令执行之后发生的错误并不会被特殊对待：即使事务中的某些命令执行失败，其他命令仍会被正常执行。EXEC 返回一个包含两个元素的字符串数组，一个元素是 OK，另一个是-ERR，**<font color="red">需要注意的是，即使命令失败，队列中的所有其他命令也会被处理—-Redis 不会停止命令的处理。所以综上，Redis 事务不支持传统关系型数据库中的回滚操作（RollBack）</font>**。

```c{.line-numbers}
>MULTI
+OK
>SET a 3
+QUEUED
>LPOP a
+QUEUED
>EXEC
*2
+OK
-ERR Operation against a key holding the wrong kind of value 
```

### 5.2 一致性

Redis 通过谨慎的错误检测和简单的设计来保证事务一致性。

### 5.3 隔离性

事务的隔离性指的是，即使数据库中有多个事务并发地执行，各个事务之间也不会互相影响，并且在并发状态下执行的事务和串行执行的事务产生的结果完全 相同。因为 Redis 使用单线程的方式来执行事务（以及事务队列中的命令），并且服务器保证，在执行事务期间不会对事物进行中断，因此，Redis 的事务总是以串行的方式运行的，并且事务也总是具有隔离性的。

### 5.4 持久性

事务的耐久性指的是，当一个事务执行完毕时，执行这个事务所得的结果已经被保持到永久存储介质（比如硬盘）里面，即使服务器在事务执行完毕之后停机，执行事务所得的结果也不会丢失。

因为 Redis 事务不过是简单的用队列包裹起了一组 Redis 命令，Redis 并没有为事务提供任何额外的持久化功能，所以 Redis 事务的耐久性由 Redis 所使用的持久化模式决定：

- 当服务器在无持久化的内存模式下运作时，事务不具有耐久性：一旦服务器停机，包括事务数据在内的所有服务器数据都将丢失。
- 当服务器在 RDB 持久化模式下运作时，服务器只会在特定的保存条件被满足时，才会执行 BGSAVE 命令，对数据库进行保存操作，因此 RDB 持久化模式下的事务也不具有耐久性。
- 当服务器运行在 AOF 持久化模式下，并且 appedfsync 选项的值为 always 时，程序总会在执行命令之后调用同步（sync）函数，将命令数据真正地保存到硬盘里面，因此这种配置下的事务是具有耐久性的。
- 当服务器运行在 AOF 持久化模式下，并且 appedfsync 的选项的值为 everysec 时，程序会每秒同步一次命令数据到磁盘。因为停机可能会恰好发生在等待同步的那一秒钟之内，这可能会造成事务数据丢失，所以这种配置下的事务不具有耐久性。
- 当服务器运行在 AOF 持久化模式下，并且 appedfsync 的选项的值为 no 时，程序会交由操作系统来决定何时将命令数据同步到硬盘。因为事务数据可能在等待同步的过程中丢失，所以这种配置下的事务不具有耐久性。

