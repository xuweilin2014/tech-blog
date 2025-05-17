# ReentrantLock 和读写锁解析

## 一、ReentrantLock 详解

### 1 前言

重入锁 ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。而 synchronized 关键字隐式的支持重进入，比如一个 synchronized 修饰的递归方法，在方法执行时，执行线程在获取了锁之后仍能连续多次地获得该锁。

ReentrantLock 虽然没能像 synchronized 关键字一样支持隐式的重进入，但是在调用 lock() 方法时，已经获取到锁的线程，能够再次调用 lock() 方法获取锁而不被阻塞。**<font color="red">这里提到一个锁获取的公平性问题，如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的</font>**。公平的获取锁，也就是等待时间最长的线程最优先获取锁，也可以说锁获取是顺序的。ReentrantLock 提供了一个构造函数，能够控制锁是否是公平的。

事实上，公平的锁机制往往没有非公平的效率高，但是，并不是任何场景都是以 TPS 作为唯一的指标，公平锁能够减少“饥饿”发生的概率，等待越久的请求越是能够得到优先满足。

### 2 核心属性

ReentrantLock 只有一个 sync 属性，别看只有一个属性，这个属性提供了所有的实现，我们上面介绍 ReentrantLock 对 Lock 接口的实现的时候就说到，它对所有的 Lock 方法的实现都调用了 sync 的方法，这个 sync 就是 ReentrantLock 的属性，它继承了 AQS。

```java{.line-numbers}
private final Sync sync; 

abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();
    //...
} 
```

在 Sync 类中，定义了一个抽象方法 lock，该方法应当由继承它的子类 FairSync 和 NonFairSync 来实现。

### 3.构造函数

ReentrantLock 共有两个构造函数：

```java{.line-numbers}
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
} 
```

**<font color="red">默认的构造函数使用了非公平锁，另外一个构造函数通过传入一个 boolean 类型的 fair 变量来决定使用公平锁还是非公平锁</font>**。其中，FairSync 和 NonfairSync 的定义如下：

```java{.line-numbers}
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    // lock 为定义在 Sync 类中的抽象方法
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        // 定义在 Sync 类中
        return nonfairTryAcquire(acquires);
    }
}

static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    // lock 为定义在 Sync 类中的抽象方法
    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
} 
```

这里为什么默认创建的是非公平锁呢？因为非公平锁的效率高呀，当一个线程请求非公平锁时，如果在发出请求的同时该锁变成可用状态，那么这个线程会跳过队列中所有的等待线程而获得锁。有的同学会说了，这不就是插队吗？没错，这就是插队！这也就是为什么它被称作非公平锁。之所以使用这种方式是因为：

**<font color="red">在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟</font>**

在公平锁模式下，大家讲究先来后到，如果当前线程A在请求锁，即使现在锁处于可用状态，它也得在队列的末尾排着，这时我们需要唤醒排在等待队列队首的线程 H (在 AQS 中其实是次头节点)，**<font color="red">由于恢复一个被挂起的线程并且让它真正运行起来需要较长时间，那么这段时间锁就处于空闲状态，时间和资源就白白浪费了</font>**，非公平锁的设计思想就是将这段白白浪费的时间利用起来——由于线程 A 在请求锁的时候本身就处于运行状态，因此如果我们此时把锁给它，它就会立即执行自己的任务，因此线程A有机会在线程H完全唤醒之前获得、使用以及释放锁。这样我们就可以把线程 H 恢复运行的这段时间给利用起来了，提高吞吐量。

当然，非公平锁仅仅是在当前线程请求锁，并且锁处于可用状态时有效，当请求锁时，锁已经被其他线程占有时，就只能还是老老实实的去排队了。无论是非公平锁的实现 NonfairSync 还是公平锁的实现 FairSync，它们都覆写了 lock 方法和 tryAcquire 方法，这两个方法都将用于获取一个锁。

### 4.Lock 接口实现

#### 4.1 lock()

ReentrantLock 中的 lock 方法实现如下：

```java{.line-numbers}
public void lock() {
     sync.lock();
} 
```

由于 Sync 类中的 lock 方法为抽象方法，具体的实现在 FairSync 和 NonFairSync 中，分为公平锁或者非公平锁。

#### 4.2 公平锁实现

关于 ReentrantLock 对于 lock 方法的公平锁的实现逻辑如下：

```java{.line-numbers}
// FairSync 中的 lock 方法
final void lock() {
    acquire(1);
} 
```

#### 4.3 非公平锁实现

接下来我们看看非公平锁的实现逻辑：

```java{.line-numbers}
// NonfairSync 中的 lock 方法
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
} 
```

对比公平锁中的 lock 方法，**<font color="red">可见，非公平锁在当前锁没有被占用时，可以直接尝试去获取锁，而不用排队</font>**，所以它在一开始就尝试使用 CAS 操作去抢锁，只有在该操作失败后，才会调用 AQS 的 acquire 方法。由于 acquire 方法中除了 tryAcquire 由子类实现外，其余都由 AQS 实现，我们在前面的文章中已经介绍的很详细了，这里不再赘述，我们仅仅看一下非公平锁的 tryAcquire 方法实现：

```java{.line-numbers}
// NonfairSync 中的 tryAcquire 方法实现
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
} 
```

它调用了 Sync 类的 nonfairTryAcquire 方法：

```java{.line-numbers}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 只有这一处和公平锁的实现不同，其它的完全一样。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
} 
```

我们可以拿它和公平锁的 tryAcquire 对比一下：

```java{.line-numbers}
// FairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
} 
```

看见没？这两个方法几乎一模一样，唯一的区别就是非公平锁在抢锁时不再需要调用 hasQueuedPredecessors 方法先去判断是否有线程排在自己前面，而是直接争锁，其它的完全和公平锁一致。

#### 4.4 lockInterruptibly()

前面的 lock 方法是非阻塞式的，抢到锁就返回 true，抢不到锁就返回 false，并且在抢锁的过程中是不响应中断的（关于不响应中断，见这篇文章末尾的分析），lockInterruptibly 提供了一种响应中断的方式，**<font color="red">在 ReentrantLock 中，无论是公平锁还是非公平锁，这个方法的实现都是一样的</font>**：

```java{.line-numbers}
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
} 
```

他们都调用了 AQS 的 acquireInterruptibly 方法：

```java{.line-numbers}
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
} 
```

该方法首先检查当前线程是否已经被中断过了，如果已经被中断了，则立即抛出 InterruptedException (这一点是 lockInterruptibly 要求的，参见上一篇 Lock 接口的介绍)。如果调用这个方法时，当前线程还没有被中断过，则接下来先尝试用普通的方法来获取锁（tryAcquire）。如果获取成功了，则万事大吉，直接就返回了；否则，与前面的 lock 方法一样，我们需要将当前线程包装成 Node 扔进等待队列，所不同的是，这次，在队列中尝试获取锁时，如果发生了中断，我们需要对它做出响应, 并抛出异常。

```java{.line-numbers}
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return; //与acquireQueued方法的不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException(); //与acquireQueued方法的不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
} 
```

如果你在上面分析 lock 方法的时候已经理解了 acquireQueued 方法，那么再看这个方法就很轻松了，我们把 lock 方法中的 acquireQueued 拿出来和上面对比一下：

```java{.line-numbers}
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; //不同之处
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; //不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; //不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
} 
```

通过代码对比可以看出，**`doAcquireInterruptibly`** 和 **`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`** 的调用本质上讲并无区别。只不过对于 **`addWaiter(Node.EXCLUSIVE)`**，一个是外部调用，通过参数传进来；一个是直接在方法内部调用。所以这两个方法的逻辑几乎是一样的，唯一的不同就是在 **`doAcquireInterruptibly`** 中，当我们检测到中断后，不再是简单的记录中断状态，而是直接抛出 InterruptedException，以达到响应中断的目的。

而在 **`acquireQueued`** 方法中，如果 **`parkAndCheckInterrupt`** 返回的为 true（表明发生了中断），则只是用变量 interrupted 记录一下，然后在获取到锁之后返回 interrupted，如果 interrupted 为 true，就会调用 **`selfInterrupt`** 进行中断。也就是说，在自旋获取锁的过程中不对线程中断进行响应，而只是在获取到锁之后，如果检测到之前发生中断，就会调用 **`selfInterrupt`** 补上中断。

#### 4.5 tryLock()

由于 tryLock 仅仅是用于检查锁在当前调用的时候是不是可获得的，所以即使现在使用的是公平锁，在调用这个方法时，当前线程也会直接尝试去获取锁，哪怕这个时候队列中还有在等待中的线程。**<font color="red">所以这一方法对于公平锁和非公平锁的实现是一样的，它被定义在 ReentrantLock 类中</font>**：

```java{.line-numbers}
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
} 
```

这个 nonfairTryAcquire 我们在上面分析非公平锁的 lock 方法时已经讲过了，这里只是简单的方法复用。该方法不存在任何和队列相关的操作，仅仅就是直接尝试去获锁，成功了就返回 true，失败了就返回 false。可能大家会觉得公平锁也使用这种方式去 tryLock 就丧失了公平性，但是这种方式在某些情况下是非常有用的，如果你还是想维持公平性，那应该使用带超时机制的 tryLock。

#### 4.6 tryLock(long timeout, TimeUnit unit)

与立即返回的 **`tryLock()`** 不同，**`tryLock(long timeout, TimeUnit unit)`** 带了超时时间，所以是阻塞式的，并且在获取锁的过程中可以响应中断异常：

```java{.line-numbers}
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
} 
```

Sync 继承于 AQS，因此会调用 AQS 中的 **`tryAcquireNanos`** 方法：

```java{.line-numbers}
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
} 
```

与 lockInterruptibly 方法一样，该方法首先检查当前线程是否已经被中断过了，如果已经被中断了，则立即抛出 InterruptedException。

随后我们通过调用 tryAcquire 和 **`doAcquireNanos(arg, nanosTimeout)`** 方法来尝试获取锁，注意，这时公平锁和非公平锁对于 tryAcquire 方法就有不同的实现了，公平锁首先会检查当前有没有别的线程在队列中排队，关于公平锁和非公平锁对 tryAcquire 的不同实现上文已经讲过了，这里不再赘述。我们直接来看 doAcquireNanos，这个方法其实和前面说的 **`doAcquireInterruptibly`** 方法很像，我们通过将相同的部分注释掉，直接看不同的部分：

```java{.line-numbers}
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    /*final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;*/
                return true; // doAcquireInterruptibly 中为 return
            /*}*/
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
       /* }
    } finally {
        if (failed)
            cancelAcquire(node);
    }*/
} 
```

可以看出，这两个方法的逻辑大差不差，只是 doAcquireNanos 多了对于截止时间的检查。

不过这里有两点需要注意，一个是 doAcquireInterruptibly 是没有返回值的，而 doAcquireNanos 是有返回值的。这是因为 doAcquireNanos 有可能因为获取到锁而返回，也有可能因为超时时间到了而返回，为了区分这两种情况，因为超时时间而返回时，我们将返回 false，代表并没有获取到锁。

另外一点值得注意的是，上面有一个 **`nanosTimeout > spinForTimeoutThreshold`** 的条件，在它满足的时候才会将当前线程挂起指定的时间，这个 spinForTimeoutThreshold 是个啥呢：

```java{.line-numbers}
/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices
 * to improve responsiveness with very short timeouts.
 */
static final long spinForTimeoutThreshold = 1000L; 
```

它就是个阈值，是为了提升性能用的。如果当前剩下的等待时间已经很短了，我们就直接使用自旋的形式等待，而不是将线程挂起，可见作者为了尽可能地优化 AQS 锁的性能费足了心思。

#### 4.7 unlock()

unlock 操作用于释放当前线程所占用的锁，这一点对于公平锁和非公平锁的实现是一样的，所以该方法被定义在 Sync 类中，由 FairSync 和 NonfairSync 直接继承使用：

```java{.line-numbers}
public void unlock() {
    sync.release(1);
} 
```

## 二、读写锁（ReentrantReadWriteLock）

### 1.读写锁简介

之前提到锁（如 ReentrantLock）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而读锁在同一时刻可以允许多个读线程访问，但是对于写锁而言，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

除了保证写操作对读操作的可见性以及并发性的提升之外，读写锁能够简化读写交互场景的编程方式。假设在程序中定义一个共享的用作缓存数据结构，它大部分时间提供读服务（例如查询和搜索），而写操作占有的时间很少，但是写操作完成之后的更新需要对后续的读服务可见。

在没有读写锁支持的（Java 5 之前）时候，如果需要完成上述工作就要使用 Java 的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并进行通知之后，所有等待的读操作才能继续执行（写操作之间依靠 synchronized 关键进行同步），这样做的目的是使读操作能读取到正确的数据，不会出现脏读。**<font color="red">改用读写锁实现上述功能，只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续（非当前写操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行</font>**，编程方式相对于使用等待通知机制的实现方式而言，变得简单明了。

**<font color="red">一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量</font>**。Java 并发包提供读写锁的实现是 ReentrantReadWriteLock，它提供的特性如下所示：

- 公平性选择：支持非公平性（默认）和公平的锁获取方式，吞吐量还是非公平优于公平；
- 重入性：支持重入，读锁获取后能再次获取，写锁获取之后能够再次获取写锁，同时也能够获取读锁；
- 锁降级：遵循获取写锁，获取读锁再释放写锁的次序，写锁能够降级成为读锁；

ReadWriteLock 仅定义了获取读锁和写锁的两个方法，即 readLock() 方法和 writeLock() 方法，而其实现——ReentrantReadWriteLock，除了接口方法之外，还提供了一些便于外界监控其内部工作状态的方法：

- **`getReadLockCount`**：它返回当前读锁被获取的次数。该次数不等于获取读锁的线程数，例如一个线程连续获取了 n 次读锁，则占据读锁的线程数为 1，但是该方法返回 10；
- **`getReadHoldCount`**：返回当前线程获取的读锁的次数；
- **`isWriteLocked`**：判断写锁是否被获取；
- **`getWriteHoldCount`**：返回当前线程写锁获取的次数；

下面是一个简单使用读写锁的示例，通过读写锁来实现一个缓存，在从缓存中读取数据的时候，加的是读锁，而写数据到缓存中或者清空缓存时，则加的是写锁，具体的代码如下：

```java{.line-numbers}
public class Cache {
    static Map<String, Object> map = new HashMap<String, Object>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();
    // 获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }
    // 设置key对应的value，并返回旧的value
    public static final Object put(String key, Object value) {
        w.lock();
        try {
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }
    // 清空所有的内容
    public static final void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
} 
```

### 2.读写锁的实现分析

读写锁同样依赖自定义同步器 AQS 来实现同步功能，而读写状态就是其同步器的同步状态。回想 ReentrantLock 中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。

如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高 16 位表示读，低 16 位表示写，划分方式如图所示：

<div align="center">
    <img src="ReentrantLock 和读写锁解析_static/1.png" width="450"/>
</div>

当前同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。假设当前同步状态值为 S，写状态等于 S&0x0000FFFF（将高16位全部抹去），读状态等于 S>>>16（无符号补 0 右移 16 位）。当写状态增加 1 时，等于 S+1，当读状态增加 1 时，等于 S+(1<<16)，也就是 S+0x00010000。

#### 2.1 写锁的获取与释放

写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为 0）或者说有其它线程获取到了写锁， 则当前线程进入等待状态，获取写锁的代码如代码清单如下所示：

```java{.line-numbers}
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 存在读锁或者当前获取线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires)) {
        return false;
    }
    setExclusiveOwnerThread(current);
    return true;
}
```

方法 **`w = exclusiveCount(c)`** 就是用来获取状态 state 中的低 16 位也就是写状态，如果 w == 0 而 c != 0，那么就说明当前还有读线程在进行读取操作，所以不能获取写锁。而当 w != 0 并且 c != 0 时，那么说明当前还有写锁，那么接下来判断获取到写锁的线程是否为当前线程，如果不是的话，也直接返回 false。

该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的 判断。如果存在读锁，则写锁不能被获取，原因在于：读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

写锁的释放与 ReentrantLock 的释放过程基本类似，每次释放均减少写状态，当写状态为 0 时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

#### 2.2 读锁的获取和释放

读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问 （即写状态为 0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。获取读锁的实现从 Java 5 到 Java 6 变得复杂许多，主要原因是新增了一 些功能，例如 **`getReadHoldCount()`** 方法，因此，这里将获取读锁的代码做了删减，保留必要的部分，如代码清单如下所示：

```java{.line-numbers}
protected final int tryAcquireShared(int unused) {
    for (;;) {
        int c = getState();
        int nextc = c + (1 << 16);
        if (nextc < c)
            throw new Error("Maximum lock count exceeded");
        if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
            return -1;
        if (compareAndSetState(c, nextc))
            return 1;
    }
}
```

在 **`tryAcquireShared(int unused)`** 方法中，**<font color="red">如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态</font>**。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠 CAS 保证）增加读状态，成功获取读锁。 读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的 值是（1<<16）。

