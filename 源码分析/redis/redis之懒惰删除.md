# Redis 之懒惰删除

## 1.LazyFree 简介

使用 redis 的用户应该都遇到过使用 DEL 命令删除体积较大的键， 又或者在使用 FLUSHDB 和 FLUSHALL 删除包含大量键的数据库时，造成 redis 阻塞的情况；**<font color="red">另外 redis 在清理过期数据和淘汰内存超限的数据时，如果碰巧撞到了大体积的键也会造成服务器阻塞</font>**。

为了解决以上问题， redis 4.0 引入了 lazyfree 的机制，它可以将删除键或数据库的操作放在后台线程里执行， 从而尽可能地避免服务器阻塞。本文将介绍使用 lazyFree 机制的 3 个场景，包括 DEL 命令、FLUSHDB 与 FLUSHALL 命令以及 Redis 清理过期数据或者淘汰内存中的数据。

## 2.LazyFree 机制

lazyfree 的原理不难想象，就是在删除对象时只是进行逻辑删除，然后把对象丢给后台，让后台线程去执行真正的 destruct，避免由于对象体积过大而造成阻塞。redis 的 lazyfree 实现即是如此，下面我们由几个命令来介绍下 lazyfree 的实现。

### 2.1 UNLINK 命令

首先我们来看下新增的 unlink 命令：

```c{.line-numbers}
void unlinkCommand(client *c) {
    delGenericCommand(c, 1);
} 
```

入口很简单，就是调用 delGenericCommand，第二个参数为 1 表示需要异步删除。

```c{.line-numbers}
/* This command implements DEL and LAZYDEL. */
void delGenericCommand(client *c, int lazy) {
    int numdel = 0, j;

    for (j = 1; j < c->argc; j++) {
        expireIfNeeded(c->db,c->argv[j]);
        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) :
                              dbSyncDelete(c->db,c->argv[j]);
        if (deleted) {
            signalModifiedKey(c->db,c->argv[j]);
            notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,
                "del",c->argv[j],c->db->id);
            server.dirty++;
            numdel++;
        }
    }
    addReplyLongLong(c,numdel);
} 
```

delGenericCommand 函数根据 lazy 参数来决定是同步删除还是异步删除，同步删除的逻辑没有什么变化就不细讲了，我们重点看下新增的异步删除的实现。

```c{.line-numbers}
#define LAZYFREE_THRESHOLD 64
// 首先定义了启用后台删除的阈值，对象中的元素大于该阈值时才真正丢给后台线程去删除，如果对象中包含的元素太少就没有必要丢给后台线程，因为线程同步也要一定的消耗。
int dbAsyncDelete(redisDb *db, robj *key) {
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    //清除待删除key的过期时间

    dictEntry *de = dictUnlink(db->dict,key->ptr);
    //dictUnlink返回数据库字典中包含key的条目指针，并从数据库字典中摘除该条目（并不会释放资源）
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);
        //lazyfreeGetFreeEffort来获取val对象所包含的元素个数

        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            //原子操作给lazyfree_objects加1，以备info命令查看有多少对象待后台线程删除
            bioCreateBackgroundJob(BIO_LAZY_FREE ,val,NULL,NULL);
            //此时真正把对象val丢到后台线程的任务队列中
            dictSetVal(db->dict,de,NULL);
            //把条目里的val指针设置为NULL，防止删除数据库字典条目时重复删除val对象
        }
    }

    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        //删除数据库字典条目，释放资源
        return 1;
    } else {
        return 0;
    }
} 
```

以上便是异步删除的逻辑，首先会清除过期时间，然后调用 dictUnlink 把要删除的对象从数据库字典摘除，再判断下对象的大小（太小就没必要后台删除），如果足够大就丢给后台线程，最后清理下数据库字典的条目信息。由以上的逻辑可以看出，当 unlink 一个体积较大的键时，实际的删除是交给后台线程完成的，所以并不会阻塞 redis。同时，也可以看出，dbAsyncDelete (由主线程调用) 并不会自己执行对键的删除操作，而是调用 bioCreateBackgroundJob 将其包装成一个 job 对象，等待 lazyFree 线程去获取并且真正地执行删除操作。

### 2.2 FLUSHALL、FLUSHDB 命令

4.0 给 flush 类命令新加了 option——async，当 flush 类命令后面跟上 async 选项时，就会进入后台删除逻辑，代码如下：

```c{.line-numbers}
/* FLUSHDB [ASYNC]
 *
 * Flushes the currently SELECTed Redis DB. */
void flushdbCommand(client *c) {
    int flags;

    if (getFlushCommandFlags(c,&flags) == C_ERR) return;
    signalFlushedDb(c->db->id);
    server.dirty += emptyDb(c->db->id,flags,NULL);
    addReply(c,shared.ok);

    sds client = catClientInfoString(sdsempty(),c);
    serverLog(LL_NOTICE, "flushdb called by client %s", client);
    sdsfree(client);
}

/* FLUSHALL [ASYNC]
 *
 * Flushes the whole server data set. */
void flushallCommand(client *c) {
    int flags;

    if (getFlushCommandFlags(c,&flags) == C_ERR) return;
    signalFlushedDb(-1);
    server.dirty += emptyDb(-1,flags,NULL);
    addReply(c,shared.ok);
    ...
} 
```

flushdb 和 flushall 逻辑基本一致，都是先调用 getFlushCommandFlags 来获取 flags（其用来标识是否采用异步删除），然后调用 emptyDb 来清空数据库，第一个参数为-1 时说明要清空所有数据库。

```c{.line-numbers}
long long emptyDb(int dbnum, int flags, void(callback)(void*)) {
    int j, async = (flags & EMPTYDB_ASYNC);
    long long removed = 0;

    if (dbnum < -1 || dbnum >= server.dbnum) {
        errno = EINVAL;
        return -1;
    }

    for (j = 0; j < server.dbnum; j++) {
        if (dbnum != -1 && dbnum != j) continue;
        removed += dictSize(server.db[j].dict);
        if (async) {
            emptyDbAsync(&server.db[j]);
        } else {
            dictEmpty(server.db[j].dict,callback);
            dictEmpty(server.db[j].expires,callback);
        }
    }
    return removed;
} 
```

进入 emptyDb 后首先是一些校验步骤，校验通过后开始执行清空数据库，同步删除就是调用 dictEmpty 循环遍历数据库的所有对象并删除（这时就容易阻塞 redis），今天的核心在异步删除 emptyDbAsync 函数。

```c{.line-numbers}
/* Empty a Redis DB asynchronously. What the function does actually is to
 * create a new empty set of hash tables and scheduling the old ones for
 * lazy freeing. */
void emptyDbAsync(redisDb *db) {
    dict *oldht1 = db->dict, *oldht2 = db->expires;
    db->dict = dictCreate(&dbDictType,NULL);
    db->expires = dictCreate(&keyptrDictType,NULL);
    atomicIncr(lazyfree_objects,dictSize(oldht1));
    bioCreateBackgroundJob(BIO_LAZY_FREE,NULL,oldht1,oldht2);
} 
```

这里直接把 db->dict 和 db->expires 指向了新创建的两个空字典，然后把原来两个字典丢到后台线程的任务队列就好了，简单高效，再也不怕阻塞 redis 了。

### 2.3 过期与逐出

当 Redis 中内存的使用量达到了 maxmemory 阈值时，会按照一定的算法对内存中的键进行删除，同时当 Redis 中的键过期时，也必须执行删除操作。而在此期间的删除动作也可能会阻塞 redis。所以 redis 4.0 这次除了显示增加 unlink、flushdb async、flushall async 命令之外，还增加了 4 个后台删除配置项，分别为：

- slave-lazy-flush：slave 接收完 RDB 文件后清空数据选项
- lazyfree-lazy-eviction：内存满逐出选项
- lazyfree-lazy-expire：过期 key 删除选项
- lazyfree-lazy-server-del：内部删除选项，比如 rename srckey destkey 时，如果 destkey 存在需要先删除 destkey

以上 4 个选项默认为同步删除，可以通过 config set [parameter] yes 打开后台删除功能。后台删除的功能无甚修改，只是在原先同步删除的地方根据以上 4 个配置项来选择是否调用 dbAsyncDelete 或者 emptyDbAsync 进行异步删除，具体代码可见：

#### 2.3.1 slave-lazy-flush

```c{.line-numbers}
void readSyncBulkPayload(aeEventLoop *el, int fd, void *privdata, int mask) {
    ...
    if (eof_reached) {
        ...
        emptyDb(
            -1,
            server.repl_slave_lazy_flush ? EMPTYDB_ASYNC : EMPTYDB_NO_FLAGS,
            replicationEmptyDbCallback);
        ...
    }
    ...
} 
```

#### 2.3.2 lazyfree-lazy-eviction

```c{.line-numbers}
int freeMemoryIfNeeded(long long timelimit) {
    ...
            /* Finally remove the selected key. */
            if (bestkey) {
                ...
                propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
                if (server.lazyfree_lazy_eviction)
                    dbAsyncDelete(db,keyobj);
                else
                    dbSyncDelete(db,keyobj);
                ...
            }
    ...
} 
```

#### 2.3.3 lazyfree-lazy-expire

```c{.line-numbers}
int activeExpireCycleTryExpire(redisDb *db, struct dictEntry *de, long long now) {
    ...
    if (now > t) {
        ...
        propagateExpire(db,keyobj,server.lazyfree_lazy_expire);
        if (server.lazyfree_lazy_expire)
            dbAsyncDelete(db,keyobj);
        else
            dbSyncDelete(db,keyobj);
        ...
    }
    ...
} 
```

## 3.LazyFree 线程

### 3.1 LazyFree 线程的实现

接下来，我们介绍一下真正在后台工作的 lazyFree 线程。

首先要澄清一个误区，很多人提到 redis 时都会讲这是一个单线程的内存数据库，其实不然。虽然 redis 把处理网络收发和执行命令这些操作都放在了主工作线程，但是除此之外还有许多 bio 后台线程也在兢兢业业的工作着。比如说，当 Redis 同步 AOF 缓冲区中的内容到磁盘上时，需要调用 Sync 函数，这也是一个耗时的操作，所以 Redis 也将这个操作移到异步线程来完成。执行 AOF Sync 操作的线程是一个独立的异步线程，和前面的懒惰删除线程不是一个线程。这次 bio 家族又加入了新的小伙伴——lazyfree 线程。

主线程需要将删除任务传递给异步线程，它是通过一个普通的双向链表 (或者说任务队列) 来传递的。因为链表需要支持多线程并发操作，所以它需要有锁来保护。执行懒惰删除时，Redis 将删除操作的相关参数封装成一个 bio_job 结构，然后追加到链表尾部。异步线程通过遍历链表摘取 job 元素来挨个执行异步任务。

```c{.line-numbers}
struct bio_job {
    time_t time;  // 时间字段暂时没有使用，应该是预留的
    void *arg1, *arg2, *arg3;
}; 
```

我们注意到这个 job 结构有三个参数，为什么删除对象需要三个参数呢？我们继续看代码：

```c{.line-numbers}
void *bioProcessBackgroundJobs(void *arg) {
    ...
        if (type == BIO_LAZY_FREE) {
            /* What we free changes depending on what arguments are set:
             * arg1 -> free the object at pointer.
             * arg2 & arg3 -> free two dictionaries (a Redis DB).
             * only arg3 -> free the skiplist. */
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2, job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        }
    ...
} 
```

**<font color="red">redis 给新加入的 lazyfree 线程起了个名字叫 `BIO_LAZY_FREE`，后台线程根据 type 判断出自己是 lazyfree 线程</font>**，然后再通过三个参数的组合实现不同的释放逻辑。

**1.lazyfreeFreeObjectFromBioThread**：释放一个普通对象，string/set/zset/hash 等等，用于普通对象的异步删除。调用 decrRefCount 来减少对象的引用计数，引用计数为 0 时会真正的释放资源：

```c{.line-numbers}
void lazyfreeFreeObjectFromBioThread(robj *o) {
    decrRefCount(o);
    atomicDecr(lazyfree_objects,1);
} 
```

**2.lazyfreeFreeDatabaseFromBioThread**：释放全局 redisDb 对象的 dict 字典和 expires 字典，用于 flushdb。后台清空数据库字典，调用 dictRelease 循环遍历数据库字典删除所有对象。

```c{.line-numbers}
void lazyfreeFreeDatabaseFromBioThread(dict *ht1, dict *ht2) {
    size_t numkeys = dictSize(ht1);
    dictRelease(ht1);
    dictRelease(ht2);
    atomicDecr(lazyfree_objects,numkeys);
} 
```

**3.lazyfreeFreeSlotsMapFromBioThread**：释放 Cluster 的 slots_to_keys 对象，后台删除 key-slots 映射表，原生 redis 如果运行在集群模式下会用。

```c{.line-numbers}
void lazyfreeFreeSlotsMapFromBioThread(rax *rt) {
    size_t len = rt->numele;
    raxFree(rt);
    atomicDecr(lazyfree_objects,len);
} 
```

接下来我们继续追踪普通对象的异步删除 lazyfreeFreeObjectFromBioThread 是如何进行的，请仔细阅读代码注释。

```c{.line-numbers}
void lazyfreeFreeObjectFromBioThread(robj *o) {
    decrRefCount(o); // 降低对象的引用计数，如果为零，就释放
    atomicDecr(lazyfree_objects,1); // lazyfree_objects 为待释放对象的数量，用于统计
}

// 减少引用计数
void decrRefCount(robj *o) {
    if (o->refcount == 1) {
        // 该释放对象了
        switch(o->type) {
        case OBJ_STRING: freeStringObject(o); break;
        case OBJ_LIST: freeListObject(o); break;
        case OBJ_SET: freeSetObject(o); break;
        case OBJ_ZSET: freeZsetObject(o); break;
        case OBJ_HASH: freeHashObject(o); break;  // 释放 hash 对象，继续追踪
        case OBJ_MODULE: freeModuleObject(o); break;
        case OBJ_STREAM: freeStreamObject(o); break;
        default: serverPanic("Unknown object type"); break;
        }
        zfree(o);
    } else {
        if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
        if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount--; // 引用计数减 1
    }
}

// 释放 hash 对象
void freeHashObject(robj *o) {
    switch (o->encoding) {
    case OBJ_ENCODING_HT:
        // 释放字典，我们继续追踪
        dictRelease((dict*) o->ptr);
        break;
    case OBJ_ENCODING_ZIPLIST:
        // 如果是压缩列表可以直接释放
        // 因为压缩列表是一整块字节数组
        zfree(o->ptr);
        break;
    default:
        serverPanic("Unknown hash encoding type");
        break;
    }
}

// 释放字典，如果字典正在迁移中，ht[0] 和 ht[1] 分别存储旧字典和新字典
void dictRelease(dict *d)
{
    _dictClear(d,&d->ht[0],NULL); // 继续追踪
    _dictClear(d,&d->ht[1],NULL);
    zfree(d);
}

// 这里要释放 hashtable 了
// 需要遍历第一维数组，然后继续遍历第二维链表，双重循环
int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    unsigned long i;

    /* Free all the elements */
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        dictEntry *he, *nextHe;

        if (callback && (i & 65535) == 0) callback(d->privdata);

        if ((he = ht->table[i]) == NULL) continue;
        while(he) {
            nextHe = he->next;
            dictFreeKey(d, he); // 先释放 key
            dictFreeVal(d, he); // 再释放 value
            zfree(he); // 最后释放 entry
            ht->used--;
            he = nextHe;
        }
    }
    /* Free the table and the allocated cache structure */
    zfree(ht->table); // 可以回收第一维数组了
    /* Re-initialize the table */
    _dictReset(ht);
    return DICT_OK; /* never fails */
} 
```

### 3.2 队列安全

前面提到任务队列是一个不安全的双向链表，需要使用锁来保护它。当主线程将任务追加到队列之前它需要加锁，追加完毕后，再释放锁，还需要唤醒异步线程，如果它在休眠的话。

```c{.line-numbers}
void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));

    job->time = time(NULL);
    job->arg1 = arg1;
    job->arg2 = arg2;
    job->arg3 = arg3;
    pthread_mutex_lock(&bio_mutex[type]); // 加锁
    listAddNodeTail(bio_jobs[type],job); // 追加任务
    bio_pending[type]++; // 计数
    pthread_cond_signal(&bio_newjob_cond[type]); // 唤醒异步线程
    pthread_mutex_unlock(&bio_mutex[type]); // 释放锁
} 
```

在 Redis 中有多种类型异步队列，比如说专门处理 AOF 同步任务的队列，以及 **`BIO_LAZY_FREE`** 线程处理任务的队列等。异步线程需要对任务队列进行轮训处理，依次从链表表头摘取元素逐个处理。摘取元素的时候也需要加锁，摘出来之后再解锁。如果一个元素都没有，它需要等待，直到主线程来唤醒它继续工作。

```c{.line-numbers}
// 异步线程执行逻辑
void *bioProcessBackgroundJobs(void *arg) {
...
    pthread_mutex_lock(&bio_mutex[type]); // 先加锁
    ...
    // 循环处理
    while(1) {
        listNode *ln;

        /* The loop always starts with the lock hold. */
        if (listLength(bio_jobs[type]) == 0) {
            // 对列空，那就睡觉吧
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* Pop the job from the queue. */
        ln = listFirst(bio_jobs[type]); // 获取队列头元素
        job = ln->value;
        /* It is now possible to unlock the background system as we know have
         * a stand alone job structure to process.*/
        pthread_mutex_unlock(&bio_mutex[type]); // 释放锁

        // 这里是处理过程，为了省纸，就略去了
        ...
        
        // 释放任务对象
        zfree(job);

        ...
        
        // 再次加锁继续处理下一个元素
        pthread_mutex_lock(&bio_mutex[type]);
        // 因为任务已经处理完了，可以放心从链表中删除节点了
        listDelNode(bio_jobs[type],ln);
        bio_pending[type]--; // 计数减 1
    } 
```

## 4. 总结

懒惰删除的技术主要用于三个地方：通过 DEL 命令删除键数据，通过 FLUSHALL、FLUSHDB 清空数据库、内存满了时，淘汰内存中的键以及删除过期了的键。懒惰删除实现就是在删除的时候，执行一个逻辑删除，把要删除的键或者数据从 Redis 中摘除掉，使得主线程不能再对键进行访问，同时，将这些删除操作包装成 job 对象，加入到对应类型的异步队列中。lazyFree 的线程 BIO_LAZY_FREE 来不断从异步队列中取出 job 任务，真正地执行删除操作。

在上面三个场景中，异步清理数据库中的键或者清空数据库时，分别调用 dbAsyncDelete 函数和 emptyDbAsync 函数实现的。而这两个函数并不是直接对数据库做删除操作，而是将其包装成一个 job 放入到队列中 (bioCreateBackgroundJob 函数)，等待 lazyFree 线程取出 job，对每一个 job 执行真正的删除操作 (bioProcessBackgroundJobs 函数)。