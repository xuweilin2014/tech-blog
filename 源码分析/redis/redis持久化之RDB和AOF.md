# Redis 持久化之 RDB 和 AOF 

## 一、Redis 持久化

Redis 作为一个键值对内存数据库(NoSQL)，数据都存储在内存当中，在处理客户端请求时，所有操作都在内存当中进行。

这样做有什么问题呢？其实，只要稍微有点计算机基础知识的人都知道，存储在内存当中的数据，只要服务器关机( 各种原因引起的)，内存中的数据就会消失了，不仅服务器关机会造成数据消失，Redis 服务器守护进程退出，内存中的数据也一样会消失。

对于只把 Redis 当缓存来用的项目来说，数据消失或许问题不大，重新从数据源把数据加载进来就可以了，但如果直接把用户提交的业务数据存储在 Redis 当中，把 Redis 作为数据库来使用，在其放存储重要业务数据，那么 Redis 的内存数据丢失所造成的影响也许是毁灭性。为了避免内存中数据丢失，Redis 提供了对持久化的支持，我们可以选择不同的方式将数据从内存中保存到硬盘当中，使数据可以持久化保存。

Redis 提供了 RDB 和 AOF 两种不同的数据持久化方式，下面我们就来详细介绍一下这种不同的持久化方式吧。

## 二、RDB 数据快照

### 1 RDB 文件的创建和载入

Redis 是一个键值对数据库服务器，服务器中通常包含着任意个非空数据库，而每个非空数据库中又可以包含任意个键值对。为了方便起见，我们将服务器中的非空数据库以及它们的键值对统称为数据库状态。RDB，提供一个某个时间点的数据的 Snapshot，保存在 RDB 文件中。它可以通过 SAVE/BGSAVE 命令手动执行，把数据 Snapshot 写到 RDB 文件，也可以通过配置，定时执行。该功能可以将某个时间点上的数据库状态保存到一个 RDB 文件，生成的 RDB 文件是一个经过压缩的二进制文件。Redis 也可以通过加载 RDB 文件，把数据从磁盘加载读取到 Redis 中。

#### 1.1 save 命令

```sql{.line-numbers}
# 同步数据到磁盘上
> save 
```

当客户端向服务器发送 save 命令请求进行持久化时，服务器会阻塞 save 命令之后的其他客户端的请求，直到数据同步完成。如果数据量太大，同步数据会执行很久，而这期间 Redis 服务器也无法接收其他请求，所以，最好不要在生产环境使用 save 命令。

#### 1.2 bgsave 命令

与 save 命令不同，bgsave 命令是一个异步操作。

```sql{.line-numbers}
# 异步保存数据集到磁盘上
> bgsave 
```

当客户端发服务发出 bgsave 命令时，Redis 服务器主进程会 forks 一个子进程来数据同步问题，在将数据保存到 rdb 文件之后，子进程会退出。所以，与 save 命令相比，Redis 服务器在处理 bgsave 采用子进程进行 IO 写入，而主进程仍然可以接收其他请求，**<font color="red">但 forks 子进程是同步的，所以 forks 子进程时，一样不能接收其他请求</font>**，这意味着，如果 forks 一个子进程花费的时间太久( 一般是很快的)，bgsave 命令仍然有阻塞其他客户端的请求的情况发生。

创建 RDB 文件实际上是由 rdb.c/rdbSave 函数完成的，SAVE 命令与 BGSAVE 命令会以不同的方式调用这个函数，以下伪代码可以看出这两个命令的区别：

```java{.line-numbers}
def SAVE():
    #创建rdb文件
    rdbSave()

def BGSAVE():
    #创建子进程
    pid = fork()
    
    if pid == 0:
        #子进程负责创建rdb文件
        rdbSave()
        #完成之后向父进程发送信号
        signal_parent()
    elif pid > 0:
        #父进程继续处理命令请求，并且通过轮询子进程的信号
        handle_request_and_wait_signal()
    else:
        #处理出错情况
        handle_fork_error() 
```

RDB 的载入工作是在 Redis 服务器启动时自动完成的，所以 Redis 并没有专门用于载入 RDB 的命令，只要 Redis 启动时检测到 RDB 文件的存在，它就会自动载入 RDB 文件。另外，由于 AOF 文件的更新频率要大于 RDB 文件，所以：

- 如果服务器开启了 AOF 持久化功能，那么服务器会优先使用 AOF 文件来还原数据库的状态
- 只有在 AOF 持久化关闭的时候，服务器才会使用 RDB 文件来还原数据库状态

载入 RDB 由 rdb.c/rdbLoad 函数完成。

#### 1.3 SAVE 命令执行时的服务器状态

正如前面所说的，当 SAVE 命令执行时，Redis 服务器会被阻塞，所以当 SAVE 命令正在执行时，客户端发送的所有命令请求都会被拒绝。只有在服务器执行完 SAVE 命令，客户端发送的命令才会被处理。

#### 1.4 BGSAVE 命令执行时的服务器状态

**<font color="red">在 BGSAVE 命令执行期间，客户端发送的 SAVE 命令会被拒绝</font>**，禁止 SAVE 命令和 BGSAVE 命令同时执行是因为可以避免父进程( 服务器进程，执行 SAVE 操作) 和子进程( 执行 BGSAVE 操作) 同时调用 rdbSave 函数，产生竞争条件

**<font color="red">在 BGSAVE 命令执行期间，客户端发送的 BGSAVE 命令会被拒绝</font>**。这是因为如果允许的话，服务器会产生另外一个子进程，这样就有两个子进程调用 rdbSave 函数，也会产生竞争条件。

**<font color="red">另外，BGREWRITEAOF 和 AOF 命令也不能同时执行</font>**：

- 如果 BGREWRITEAOF 命令正在执行，那么客户端发送的 BGSAVE 命令会被拒绝
- 如果 BGSAVE 命令正在进行，那么客户端发送的 BGREWRITEAOF 命令会被延迟到 BGSAVE 执行完毕后执行

BGREWRITEAOF 和 AOF 命令也不能同时执行实际上是出于性能方面的考虑，并发两个子进程，并且这两个子进程都同时执行大量的磁盘写入 IO 操作，这并不是一个好主意。

#### 1.5 RDB 载入时服务器的状态

服务器在载入 RDB 文件时，会一直处于阻塞状态，直到载入工作完成为止。

### 2 服务器配置自动触发

#### 2.1 设置保存条件

save 命令和 bgsave 命令是手动执行的过程，但在生产过程中我们很少手动的登上服务去执行操作，所以更多的时候是依赖 Redis 的配置，让服务器每个一段时间自动执行一次，即在 Redis 配置文件中的 save 指定到达触发 RDB 持久化的条件，比如【多少秒内至少达到多少写操作】就开启 RDB 数据同步。打开 redis.conf 配置文件，找到 SNAPSHOTTING 的配置，Save Point 的设置。

<div align="center">
    <img src="redis_static//1.png" width="450"/>
</div>

save < 指定时间间隔> < 执行指定次数更新操作>，满足条件就将内存中的数据同步到硬盘中。用户可以通过 save 选项配置多个保存条件，但只要其中一个条件被满足，服务器就会执行 BGSAVE 命令。上图所示为服务器中默认配置，那么只要满足以下三个条件中的任意一个，BGSAVE 命令就会被执行：

- 服务器在 900s 内，对数据库做了至少 1 次修改
-  服务器在 300s 内，对数据库做了至少 10 次修改
-  服务器在 60s 内，对数据库做了至少 10000 次修改
  
#### 2.2 dirty 计数器和 lastsave 属性

当设置好 save 选项时，服务器程序会根据 save 选项所设置的保存条件，设置服务器状态 redisServer 结构中的 saveparams 属性：

```c{.line-numbers}
//redis.h
struct redisServer {
    //....
    //记录了save条件的数组
    struct saveparam *saveparams;
    //....
} 
```

saveparams 是一个数组，数组中的每个元素都是一个 saveparam 结构，每个 saveparam 结构都保存了一个 save 选项设置的保存条件：

```c{.line-numbers}
//redis.h
// 服务器的保存条件（BGSAVE 自动执行的条件）
struct saveparam {
    // 多少秒之内
    time_t seconds;
    // 发生多少次修改
    int changes;
}; 
```

如果说，save 想选的值为上图中的条件，那么服务器状态中的 saveparams 数组将会是下图所示：

<div align="center">
    <img src="redis_static//2.png" width="450"/>
</div>

除了 saveparams 数组外，redisServer 中还保存着一个 dirty 计数器，以及一个 lastsave 属性：

- dirty 计数器记录距离上一次 SAVE 命令或者 BGSAVE 命令之外，服务器对数据库状态( 数据库中的所有数据) 进行了多少次修改( 包括写入、删除与更新)；
- lastsave 属性是一个 UNIX 时间戳，记录了服务器上一次成功执行 SAVE 命令或者 BGSAVE 命令的时间；

```c{.line-numbers}
//redis.h
struct redisServer {
    //....
    //记录了save条件的数组
    struct saveparam *saveparams;
    // 自从上次 SAVE 执行以来，数据库被修改的次数
    long long dirty;              
    // 最后一次完成 SAVE 的时间
    time_t lastsave;
    //....
} 
```

当服务器成功执行一个数据库修改命令之后，程序就会对 dirty 计数器进行更新：命令修改了多少次数据库，dirty 计数器的值就增加多少。比如，我们向一个集合中增加三个新元素：

```c{.line-numbers}
redis>SADD database Redis MongoDB MariaDB
(integer) 3 
```

那么程序会将 dirty 计数器的值增加 3。

#### 2.3 检查保存条件是否满足

Redis 服务器的周期性函数操作函数 serverCron 默认每隔 100 毫秒就会执行一次，该函数用于对正在运行的服务器进行维护，它的其中一项工作就是检查 save 选项所设置的保存条件是否满足，如果满足的话，就执行 BGSAVE 命令。

```c{.line-numbers}
//redis.c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    int j;
    //......
    /* Update the time cache. */
    // 更新redisServer中的unixtime和mstime，也就是每秒更新redisServer.hz次
    updateCachedTime();
    // 记录服务器执行命令的次数
    run_with_period(100) trackOperationsPerSecond();
    /* 
     * LRU 时间的精度可以通过修改 REDIS_LRU_CLOCK_RESOLUTION 常量来改变
     * 更新redisServer中的lruclock，也就是每秒更新redisServer.hz次
     */
    server.lruclock = getLRUClock();
    // 增加 loop 计数器
    server.cronloops++;

    /* If there is not a background saving/rewrite in progress check if
         * we have to save/rewrite now */
        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们

        // 遍历所有保存条件，看是否需要执行 BGSAVE 命令
        for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * REDIS_BGSAVE_RETRY_DELAY seconds already elapsed. */
            // 检查是否有某个保存条件已经满足了
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 REDIS_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == REDIS_OK))
            {
                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
                break;
            }
        }
    //.......
} 
```

上面代码即为 serverCron 函数中关于 bgsave 相关的部分。也就是遍历 saveparams 数组，只要其中有一个条件满足，同时上一次 bgsave 的结果是成功的，或者上一次 bgsave 的结果是失败的，但是距离上一次 bgsave 的时间也超过了 REDIS_BGSAVE_RETRY_DELAY 的限制，那么就可以执行 BGSAVE，将数据库的状态保存到 rdb 文件中。最终，redis 服务器和 BGSAVE 有关的状态如下：

<div align="center">
    <img src="redis_static//3.png" width="450"/>
</div>

### 3 Copy-On-Write 机制

本节只是简略的介绍一下，详细的请看专门的文章。

在 Redis 进行持久化的同时，内存数据结构还在改变，比如一个大型的 hash 字典正在持久化，结果一个请求过来把它给删掉了，还没持久化完。那该怎么办呢？事实上 Redis 使用操作系统的多进程 COW(Copy On Write) 机制来实现快照持久化，这个机制很有意思，也很少人知道。

Redis 在持久化时会调用 glibc 的函数 fork 产生一个子进程，快照持久化完全交给子进程来处理，父进程继续处理客户端请求。子进程刚刚产生时，它和父进程共享内存里面的代码段和数据段。这时你可以将父子进程想像成一个连体婴儿，共享身体。这是 Linux 操作系统的机制，为了节约内存资源，所以尽可能让它们共享起来。在进程分离的一瞬间，内存的增长几乎没有明显变化。

用 Python 语言描述进程分离的逻辑如下。fork 函数会在父子进程同时返回，在父进程里返回子进程的 pid，在子进程里返回零。如果操作系统内存资源不足，pid 就会是负数，表示 fork 失败。

```c{.line-numbers}
pid = os.fork()
if pid > 0:
    handle_client_requests()  # 父进程继续处理客户端请求
if pid == 0:
    handle_snapshot_write()  # 子进程处理快照写磁盘
if pid < 0:
    # fork error 
```

子进程做数据持久化，它不会修改现有的内存数据结构，它只是对数据结构进行遍历读取，然后序列化写到磁盘中。但是父进程不一样，它必须持续服务客户端请求，然后对内存数据结构进行不间断的修改。这个时候就会使用操作系统的 COW 机制来进行数据段页面的分离。数据段是由很多操作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。

<div align="center">
    <img src="redis_static//4.png" width="380"/>
</div>

随着父进程修改操作的持续进行，越来越多的共享页面被分离出来，内存就会持续增长。但是也不会超过原有数据内存的 2 倍大小。另外一个 Redis 实例里冷数据占的比例往往是比较高的，所以很少会出现所有的页面都会被分离，被分离的往往只有其中一部分页面。每个页面的大小只有 4K，一个 Redis 实例里面一般都会有成千上万的页面。

子进程因为数据没有变化，它能看到的内存里的数据在进程产生的一瞬间就凝固了，再也不会改变，这也是为什么 Redis 的持久化叫「快照」的原因。接下来子进程就可以非常安心的遍历数据了进行序列化写磁盘了。

### 4 默认数据压缩

```java{.line-numbers}
rdbcompression yes
```

配置存储至本地数据库时是否压缩数据，默认为 yes。Redis 采用 LZF 压缩方式，但占用了一点 CPU 的时间。若关闭该选项，但会导致数据库文件变的巨大。建议开启。

### 5 触发 RDB 快照的时机

综上所述，触发 RDB 快照的时机如下所示：

- 在指定的时间间隔内，执行指定次数的写操作
- 执行 save（阻塞， 只管保存快照，其他的等待） 或者是 bgsave （异步）命令
- 执行 flushall 命令：当我们使用了则表明我们需要对数据进行清空，那 redis 当然需要对快照文件也进行清空，所以会触发 bgsave
- 执行 shutdown 命令：redis 在关闭前处于安全角度将所有数据全部保存下来，以便下次启动会恢复。
- 主从复制时，从库全量复制同步主库数据，此时主库会执行 bgsave 命令进行快照；

### 6 RDB 总结

对 RDB 持久化的总结为：用户可以通过手动的使用参数 SAVE 或者 BGSAVE 命令来主动对 Redis 进行持久化，生成 RDB 文件；同时，Redis 也可以根据用户对保存条件的配置，比如在多少秒的时间里面对数据库做了多少改动，如果达到了用户设置条件中的任意一个，就会触发 BGSAVE 命令，生成一个子进程生成 RDB 文件；这些 save 条件保存在服务器状态 redisServer 中的 saveparams 属性中，也就是 saveparam 数组，serverCron 函数默认每隔 100ms 被调用一次，在 serverCron 函数中会遍历 saveparams 数组，检查其中的条件是否满足，如果有一个满足的话，就会调用 rdbSaveBackground 函数进行 RDB 持久化。

在使用 BGSAVE 命令进行 RDB 持久化的时候，会有可能子进程在生成 RDB 文件的时候，主进程接收客户端的命令又继续对数据库进行修改，这时，会用到 Copy-on-Write 技术，也就是刚开始父子进程共享同一块物理内存，直到父进程对内存中的某一页进行修改，那么操作系统会对被修改的这一页进行复制，新的副本分配给子进程，父子进程对这一页分别持有一份副本。不过物理内存中的其它页任然是两个进程共享的。这样就可以使得子进程观察到的数据库状态保持不变。同时，当 Redis 在上一次进行 RDB 持久化之后，如果运行了一段时间，没有达到 save 条件，而服务器突然宕机了，那么这段时间内的数据就丢失了。

同时，Redis 服务器在 shutdown 数据库时，或者执行 flushall 命令都会触发 RDB 快照。

## 三、AOF—日志追加

与 RDB 存储某个时刻的快照不同，AOF 持久化方式会记录客户端对服务器的每一次写操作命令，并将这些写操作以 Redis 协议追加保存到以后缀为 aof 文件末尾，在 Redis 服务器重启时，如果开启了 AOF 持久化功能，则会加载并运行 aof 文件，以达到恢复数据的目的。

### 1.AOF 启用

Redis 默认不开启 AOF 持久化方式，打开 redis.conf 配置文件，找到 appendonly no 改成 appendonly yes。

<div align="center">
    <img src="redis_static//5.png" width="450"/>
</div>

### 2.AOF 持久化的实现

#### 2.1 命令追加

当 AOF 功能处于打开状态时，那么服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的 aof_buf 缓冲区的末尾：

```c{.line-numbers}
//redis.h
struct redisServer {
    //....
    //记录了save条件的数组
    struct saveparam *saveparams;
    // 自从上次 SAVE 执行以来，数据库被修改的次数
    long long dirty;              
    // 最后一次完成 SAVE 的时间
    time_t lastsave;
    //AOF缓冲区
    sds aof_buf;
    //....
} 
```

#### 2.2 AOF 文件的写入与同步

Redis 服务器进程就是一个事件循环，这个循环中的文件事件负责接收客户端的命令请求，以及向客户端发送命令回复。而时间事件则负责执行像 serverCron 函数这样要定时运行的函数。因为服务器在处理文件事件时可能会执行写命令，使得一些内容被追加到 aof_buf 缓冲区里面，所以在服务器在结束一个事件循环之前，它都会调用 flushAppendOnlyFile 函数，考虑是否需要将 aof_buf 缓冲区中的内容写入和保存到 AOF 文件里面。过程如下伪代码所示：

```c{.line-numbers}
def eventLoop():
    while True:
        #处理文件事件，接收命令请求以及发送命令回复
        #处理命令请求时可能会有新内容被追加到aof_buf缓冲区中
        processFileEvents()

        #处理时间事件
        processTimeEvents()

        #考虑是否要将aof_buf缓冲区中的内容写入和保存到AOF文件里面
        flushAppendOnlyFile()
```

flushAppendOnlyFile 函数的行为由服务器配置的 appendfsync 选项的值来决定，各个不同值行为如下所示：

- **`Always`**: 将 aof_buf 缓冲区中的所有内容写入并且同步到 AOF 文件
- **`Everysec`**: 将 aof_buf 缓冲区中的所有内容写入到 AOF 文件，如果上次同步 AOF 文件的时间距离现在超过一秒钟，那么再次对 AOF 文件进行同步。
- **`No`**: 将 aof_buf 缓冲区中的所有内容写入到 AOF 文件，但是并不对 AOF 文件进行同步，何时同步由操作系统来决定。

这里，要讲解一下操作系统中的 **<font color="red">写入与同步(将保存在缓冲区中的数据强制刷新到磁盘中) 概念</font>**。为了提高文件的写入效率，在现代操作系统中，当用户调用 write 函数，将一些数据写入到文件的时候，操作系统会暂时将写入数据保存在一个内存缓冲区中，等到缓存区被填满，或者超过有限的时间之后，才真正的将缓冲区中的数据写入到磁盘里面。

这种做法虽然提高了效率，但也为写入数据带来了安全问题，如果计算机发生了停机，那么保存在内存缓冲区里面的写入数据将会丢失。为此，系统提供了 fsync 和 fdatasync 两个同步函数，他们可以强制让操作系统立即将缓冲区中的数据写入到硬盘里面，从而确保写入数据的安全性。

### 3.AOF 文件的载入与还原

Redis 读取 AOF 文件并且还原数据库状态的详细步骤如下：

1. 创建一个不带网络连接的伪客户端：服务器使用没有网络连接的无客户端来执行 AOF 文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样
2. 从 AOF 文件中分析并且读出一条写命令
3. 使用伪客户端执行被读出的写命令

一直执行步骤 2 和步骤 3，直到 AOF 文件中的所有写命令都被处理完毕为止。

### 4.AOF 文件格式解析

在客户端上执行一些命令后，打开 AOF 文件，可以观察到有对应的操作的记录日志。

<div align="center">
    <img src="redis_static//6.png" width="260"/>
</div>

文件解析说明：

- **`*，表示命令的参数个数，例如 set a 1 是三个参数，所以是 *3`**
- **`$，表示参数的字节数，例如 set 这个参数是三字节，所以是 $3，key 值 a 是一个字节，所以是$1`**
- **`无符号，表示是参数的数据，例如 set,a,1 就是具体的数据`**

### 5.AOF 文件的重写

#### 5.1 AOF 文件重写的实现

AOF 虽然比 RDB 更可靠，但缺点也是比较明显的，就是每次写操作都要把操作日志写到文件上，这样会导致文件非常冗余。

假若你要自增一个计数器 100 次，如果不重写，AOF 文件就就会有这 100 次的自增记录，如 INCR a。如果执行了日志重写，那么文件只会保留 set a 100 而不是 100 条 INCR a。除了这个例子之外，其他所有类型的键都可以用同样的方法去减少 AOF 文件中命令的数量，首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令。这就是 AOF 实现的原理，因为 aof_rewrite 函数新生成的 AOF 文件只包含还原当前数据库状态所必须的命令，所以 AOF 文件不会浪费任何硬盘空间。

这样拥有相同的结果，但可以大大减少 AOF 的文件大小，并且可以让 AOF 载入的时候提升载入的效率。为了处理这种情况，Redis 支持一种有趣的特性： 可以在不打断服务客户端的情况下， 对 AOF 文件进行重建（rebuild）。执行 **<font color="red">BGREWRITEAOF</font>** 命令， Redis 将生成一个新的 AOF 文件， 这个文件包含重建当前数据集所需的最少命令。

另外，在实际中，为了避免在后来使用 AOF 文件恢复数据库时，执行 AOF 中的命令时造成客户端输入缓冲区溢出，重写程序在处理列表时、哈希表、集合、有序集合这四种可能会带有多个元素的键时，会先检查键所包含的元素数量，如果元素的数量超过了 REDIS_AOF_REWRITE_ITEMS_PER_CMD 常量的值，那么重写程序将使用多条命令来记录键的值，而不是单单使用一条命令。

看回 redis.conf 配置，有两项控制 rewrite 的选项。

<div align="center">
    <img src="redis_static///7.png" width="450"/>
</div>

- auto-aof-rewrite-percentage 100，当文件增长 100%（一倍）时候，自动重写。
- auto-aof-rewrite-min-size 64mb，日志重写最小文件大小，如果小于该大小，不会自动重写。

值得注意的是，重写 AOF 文件的操作，并没有读取旧的 aof 文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的 aof 文件，这点和快照有点类似。AOF 重写并不需要对原有 AOF 文件进行任何的读取，写入，分析等操作，这个功能是通过读取服务器当前的数据库状态来实现的。

```java{.line-numbers}
# 假设服务器对键 list 执行了以下命令 s;
127.0.0.1:6379> RPUSH list "A" "B"
(integer) 2
127.0.0.1:6379> RPUSH list "C"
(integer) 3
127.0.0.1:6379> RPUSH list "D" "E"
(integer) 5
127.0.0.1:6379> LPOP list
"A"
127.0.0.1:6379> LPOP list
"B"
127.0.0.1:6379> RPUSH list "F" "G"
(integer) 5
127.0.0.1:6379> LRANGE list 0 -1
1) "C"
2) "D"
3) "E"
4) "F"
5) "G"
127.0.0.1:6379> 
```

当前列表键 list 在数据库中的值就为 ["C", "D", "E", "F", "G"]。要使用尽量少的命令来记录 list 键的状态，最简单的方式不是去读取和分析现有 AOF 文件的内容，而是直接读取 list 键在数据库中的当前值，然后用一条 RPUSH list "C" "D" "E" "F" "G" 代替前面的 6 条命令。

#### 5.2 AOF 后台重写

AOF 重写程序 aof_rewrite 函数可以很好地完成创建一个新 AOF 文件的任务，但是，因为这个函数会进行大量的写入操作，所以调用这个函数的线程将被长时间阻塞，在此期间服务器将无法处理客户端发来的命令请求。所以，考虑到这种情况，Redis 将 AOF 重写程序放到子进程中执行。不过，当子进程在重写 AOF 文件的时候，可能父进程收到客户端的命令对数据库进行修改，造成子进程重写的 AOF 文件与数据库状态不一致的情况。

为了解决这个情况，Redis 服务器设置了一个 AOF 重写缓冲区。AOF 执行重写的步骤如下：

- 当 Redis 服务器执行完一个写命令之后，它会同时将这个写命令发送给 AOF 缓冲区以及 AOF 重写缓冲区。这样可以保证：
  - AOF 缓冲区中的内容可以被定期写入和同步到 AOF 文件，对现有 AOF 文件的处理工作会如常进行
  - **<font color="red">从创建子进程开始，服务器执行的所有写命令都会被记录到 AOF 重写缓冲区中</font>**
- 子进程完成 AOF 重写工作之后，向父进程发送一个信号
- 父进程接收到信号后，首先会将 AOF 重写缓冲区中的所有内容写入到新 AOF 文件中，这时新 AOF 文件所保存的数据库状态和服务器当前的数据库状态一致
- 父进程原子地使用新的 AOF 文件替换旧的 AOF 文件

接下来，父进程就可以向往常一样接收命令请求了。

### 6.AOF 文件损坏

在写入 aof 日志文件时，如果 Redis 服务器宕机，则 aof 日志文件文件会出格式错误，在重启 Redis 服务器时，Redis 服务器会拒绝载入这个 aof 文件，可以通过以下步骤修复 aof 并恢复数据。

1. 备份现在 aof 文件，以防万一。
2. 使用 redis-check-aof 命令修复 aof 文件，该命令格式如下：

```java{.line-numbers}
# 修复aof日志文件
$ redis-check-aof -fix file.aof 
```

3. 重启 Redis 服务器，加载已经修复的 aof 文件，恢复数据。

## 四、RDB 与 AOF 混合持久化

细细想来 **<font color="red">aofrewrite 时也是先写一份全量数据到新 AOF 文件中再追加增量( 也就是前面说的子进程先重写数据库状态，再由父进程将 AOF 重写缓冲区中的内容写入到新 AOF 文件)</font>**，只不过全量数据是以 redis 命令的格式写入。那么是否可以先以 RDB 格式写入全量数据再追加增量日志呢这样既可以提高 aofrewrite 和恢复速度也可以减少文件大小还可以保证数据的完毕性整合 RDB 和 AOF 的优点那么现在 4.0 实现了这一特性——RDB-AOF 混合持久化。

### 1.aofrewrite

RDB-AOF 混合持久化体现在 aofrewrite 时那么我们就从这里开始来看 4.0 是如何实现的。回忆下 aofrewrite 的过程：

无论是 serverCron 触发或者执行 BGREWRITEAOF 命令最终 redis 都会走到 rewriteAppendOnlyFileBackground()。rewriteAppendOnlyFileBackground 函数会 fork 子进程子进程进入 rewriteAppendOnlyFile 函数来生成新的 AOF 文件混合持久化就从这里开始。

```java{.line-numbers}
int rewriteAppendOnlyFile(char *filename) {
    ...
    if (server.aof_use_rdb_preamble) {
        int error;
        if (rdbSaveRio(&aof,&error,RDB_SAVE_AOF_PREAMBLE,NULL) == C_ERR) {
            errno = error;
            goto werr;
        }
    } else {
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
    }
    ...
} 
```

可以看到当混合持久化开关 **<font color="red">aof_use_rdb_preamble<</font>** 打开时就会进入 rdbSaveRio 函数先以 RDB 格式来保存全量数据，前文说道子进程在做 aofrewrite 时会通过管道从父进程读取增量数据并缓存下来，那么在以 RDB 格式保存全量数据时也会从管道读取数据并不会造成管道阻塞。

在 rdbSaveRio 函数中，首先把 RDB 的版本 (注意不是 redis 的版本) 和辅助域写入文件。

```java{.line-numbers}
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi) {
    ...
    snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
    if (rdbSaveInfoAuxFields(rdb,flags,rsi) == -1) goto werr; 
```

然后遍历 DB 先把 SELECTDB 操作码、被选中数据库的 dbnum、db_size 以及 expires_size 写入文件。

```java{.line-numbers}
for (j = 0; j < server.dbnum; j++) {
    redisDb *db = server.db+j;
    dict *d = db->dict;
    if (dictSize(d) == 0) continue;
    di = dictGetSafeIterator(d);
    if (!di) return C_ERR;

    /* Write the SELECT DB opcode */
    if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr;
    if (rdbSaveLen(rdb,j) == -1) goto werr;

    /* Write the RESIZE DB opcode. We trim the size to UINT32_MAX, which
     * is currently the largest type we are able to represent in RDB sizes.
     * However this does not limit the actual size of the DB to load since
     * these sizes are just hints to resize the hash tables. */
    uint32_t db_size, expires_size;
    db_size = (dictSize(db->dict) <= UINT32_MAX) ?
                            dictSize(db->dict) :
                            UINT32_MAX;
    expires_size = (dictSize(db->expires) <= UINT32_MAX) ?
                            dictSize(db->expires) :
                            UINT32_MAX;
    if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;
    if (rdbSaveLen(rdb,db_size) == -1) goto werr;
    if (rdbSaveLen(rdb,expires_size) == -1) goto werr; 
```

在当前 DB 中遍历所有的 key 把 key-value 对及过期时间如果有设置的话写入文件。这里提一下在 rdbSaveKeyValuePair 函数中会判断 expire 是否已经到了过期时间如果已经过期就不会写入文件。同时如果 flags 标记了 RDB_SAVE_AOF_PREAMBLE 的话说明是在 aofrewrite 且开启了 RDB-AOF 混合开关此时就会从父进程去读取增量数据了。

```java{.line-numbers}

while((de = dictNext(di)) != NULL) {
           sds keystr = dictGetKey(de);
           robj key, *o = dictGetVal(de);
           long long expire;

           initStaticStringObject(key,keystr);
           expire = getExpire(db,&key);
           if (rdbSaveKeyValuePair(rdb,&key,o,expire,now) == -1) goto werr;

           /* When this RDB is produced as part of an AOF rewrite, move
            * accumulated diff from parent to child while rewriting in
            * order to have a smaller final write. */
           if (flags & RDB_SAVE_AOF_PREAMBLE &&
               rdb->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES)
           {
               processed = rdb->processed_bytes;
               aofReadDiffFromParent();
           }
       }
       dictReleaseIterator(di);
   }
   di = NULL; /* So that we don't release it again on error. */ 
```

### 2.数据恢复

- 当 appendonly 配置项为 no 时 redis 启动后会去加载 RDB 文件以 RDB 格式来解析 RDB 文件自然没有问题。
- 当 appendonly 配置项为 yes 时 redis 启动后会加载 AOF 文件来恢复数据，如果持久化时开启了 RDB-AOF 混合开关那么 AOF 文件的前半段就是 RDB 格式，此时要如何正确加载数据呢

一切数据都逃不过协议二字，不以正确的协议存储和解析，那就是乱码。既然允许 RDB-AOF 混合持久化就要能够识别并恢复数据这一节我们来介绍如何以正确的姿势来恢复数据。加载 AOF 文件的入口为 loadAppendOnlyFile。

```java{.line-numbers}
int loadAppendOnlyFile(char *filename) {
    ...
    /* Check if this AOF file has an RDB preamble. In that case we need to
     * load the RDB file and later continue loading the AOF tail. */
    char sig[5]; /* "REDIS" */
    if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
        /* No RDB preamble, seek back at 0 offset. */
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    } else {
        /* RDB preamble. Pass loading the RDB functions. */
        rio rdb;

        serverLog(LL_NOTICE,"Reading RDB preamble from AOF file...");
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
        rioInitWithFile(&rdb,fp);
        if (rdbLoadRio(&rdb,NULL) != C_OK) {
            serverLog(LL_WARNING,"Error reading the RDB preamble of the AOF file, AOF loading aborted");
            goto readerr;
        } else {
            serverLog(LL_NOTICE,"Reading the remaining AOF tail...");
        }
    }
    ...
} 
```

打开 AOF 文件之后首先读取 5 个字符如果是"REDIS"，那么就说明这是一个混合持久化的 AOF 文件。正确的 RDB 格式一定是以"REDIS" 开头，而纯 AOF 格式则一定以"*" 开头，此时就会进入 rdbLoadRio 函数来加载数据。rdbLoadRio 函数此处就不详细展开了就是以约定好的协议解析文件内容，直至遇到 RDB_OPCODE_EOF 结束标记，返回 loadAppendOnlyFile 函数继续以 AOF 格式解析文件直到结束，整个加载过程完成。

### 3.总结

在过去， Redis 用户通常会因为 RDB 持久化和 AOF 持久化之间不同的优缺点而陷入两难的选择当中：

- RDB 持久化能够快速地储存和回复数据，但是在服务器停机时却会丢失大量数据；
- AOF 持久化能够有效地提高数据的安全性，但是在储存和恢复数据方面却要耗费大量的时间。

为了让用户能够同时拥有上述两种持久化的优点， Redis 4.0 推出了一个能够"鱼和熊掌兼得"的持久化方案 —— RDB-AOF 混合持久化：这种持久化能够通过 AOF 重写操作创建出一个同时包含 RDB 数据和 AOF 数据的 AOF 文件， 其中 RDB 数据位于 AOF 文件的开头， 它们储存了服务器开始执行AOF重写操作时的数据库状态，至于那些在重写操作执行之后执行的 Redis 命令， 则会继续以 AOF 格式追加到 AOF 文件的末尾，也即是 RDB 数据之后。