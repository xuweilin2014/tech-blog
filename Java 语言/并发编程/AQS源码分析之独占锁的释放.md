# AQS 源码分析之独占锁的释放

## 1.前言

上一篇文章我们逐行分析了独占锁的获取操作，本篇文章我们来看看独占锁的释放。如果前面的锁的获取流程你已经趟过一遍了，那锁的释放部分就很简单了，这篇文章我们直接开始看源码。开始之前先提一句，JAVA 的内置锁在退出临界区之后是会自动释放锁的，但是 ReentrantLock 这样的显式锁是需要自己显式的释放的，所以在加锁之后一定不要忘记在 finally 块中进行显式的锁释放:

```java{.line-numbers}
Lock lock = new ReentrantLock();
...
lock.lock();
try {
        // 更新对象
        // 捕获异常
} finally {
        lock.unlock();
}
```

## 2.ReentrantLock 的释放

由于锁的释放操作对于公平锁和非公平锁都是一样的，所以，unlock 的逻辑并没有放在 FairSync 或 NonfairSync 里面，而是直接定义在 ReentrantLock类中:

```java{.line-numbers}
public void unlock() {
    sync.release(1);
}
```

由于释放锁的逻辑很简单，这里就不画流程图了，我们直接看源码:

### 2.1 release

release 方法定义在 AQS 类中，描述了释放锁的流程：

```java{.line-numbers}
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
        return true;
    }
    return false;
}
```

可以看出，相比获取锁的 acquire 方法，释放锁的过程要简单很多，它只涉及到两个子函数的调用:

- **`tryRelease(arg)`**：该方法由继承 AQS 的子类实现， 为释放锁的具体逻辑
- **`unparkSuccessor(h)`**：唤醒后继线程

下面我们分别分析这两个子函数。

### 2.2 tryRelease

tryRelease 方法由 ReentrantLock 的静态类 Sync 实现。多嘴提醒一下，能执行到释放锁的线程，一定是已经获取了锁的线程。另外，相比获取锁的操作，这里并没有使用任何 CAS 操作，也是因为当前线程已经持有了锁，所以可以直接安全的操作，不会产生竞争。

```java{.line-numbers}
protected final boolean tryRelease(int releases) {       
    // 首先将当前持有锁的线程个数减 1 (回溯到调用源头 sync.release(1) 可知, releases 的值为 1)
    // 这里的操作主要是针对可重入锁的情况下, c 可能大于 1
    int c = getState() - releases; 
    
    // 释放锁的线程当前必须是持有锁的线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
    
    // 如果c为0了, 说明锁已经完全释放了
    boolean free = false;
    if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### 2.3 unparkSuccessor

```java{.line-numbers}
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

锁成功释放之后，接下来就是唤醒后继节点了，这个方法同样定义在 AQS 中。值得注意的是，在成功释放锁之后(tryRelease 返回 true 之后)，唤醒后继节点只是一个 "附加操作"，无论该操作结果怎样，最后 release操作都会返回 true。在 AQS 的 Node 类中，结点的状态如下所示：

- **`CANCELLED`** = 1
- **`SIGNAL`** = -1
- **`CONDITION`** = -2
- **`PROPGATE`** = -3

接下来我们就看看 unparkSuccessor 的源码：

```java{.line-numbers}
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;

    // 如果 head 节点的 ws 比 0 小, 则直接将它设为 0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 通常情况下, 要唤醒的节点就是自己的后继节点
    // 如果后继节点存在且也在等待锁, 那就直接唤醒它
    // 但是有可能存在 后继节点取消等待锁 的情况
    // 此时从尾节点开始向前找起, 直到找到距离 head 节点最近的 ws<=0 的节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t; // 注意! 这里找到了之并有 return, 而是继续向前找
    }
    // 如果找到了还在等待锁的节点,则唤醒它
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

在上一篇文章分析 **`shouldParkAfterFailedAcquire`** 方法的时候，我们重点提到了当前节点的前驱节点的 waitStatus 属性, 该属性决定了我们是否要挂起当前线程, 并且我们知道, 如果一个线程被挂起了, 它的前驱节点的 waitStatus 值必然是 **`Node.SIGNAL`**。在唤醒后继节点的操作中, 我们也需要依赖于节点的 waitStatus 值。

### 2.4 interrupt

最后, 在调用了 **`LockSupport.unpark(s.thread)`** 也就是唤醒了线程之后, 会发生什么呢?当然是回到最初的原点啦, 从哪里跌倒 (被挂起) 就从哪里站起来 (唤醒) 呗:

```java{.line-numbers}
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 喏, 就是在这里被挂起了, 唤醒之后就能继续往下执行了
    return Thread.interrupted();
}
```

那接下来做什么呢？还记得我们上一篇在讲“锁的获取”的时候留的问题吗？ 如果线程从这里唤醒了，它将接着往下执行。注意，这里有两个线程：

- 一个是我们这篇讲的线程，它正在释放锁，并调用了 **`LockSupport.unpark(s.thread)`** 唤醒了另外一个线程； 
- 另外一个线程，就是我们上一节讲的因为抢锁失败而被阻塞在 **`LockSupport.park(this)`** 处的线程；

我们再倒回上一篇结束的地方，看看这个被阻塞的线程被唤醒后，会发生什么。从上面的代码可以看出，他将调用 **`Thread.interrupted()`** 并返回。我们知道，**`Thread.interrupted()`** 这个函数将返回当前正在执行的线程的中断状态，并清除它。接着，我们再返回到 parkAndCheckInterrupt 被调用的地方:

```java{.line-numbers}
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 我们在这里！在这里！！在这里！！！
            // 我们在这里！在这里！！在这里！！！
            // 我们在这里！在这里！！在这里！！！
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

具体来说，就是这个 if 语句:

```java{.line-numbers}
if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
    interrupted = true;
```

可见，如果 **`Thread.interrupted()`** 返回 true，则 **`parkAndCheckInterrupt()`** 就返回 true，if 条件成立，interrupted 状态将设为 true; 如果 **`Thread.interrupted()`** 返回 false, 则 interrupted 仍为 false。再接下来我们又回到了 **`for (;;)`** 死循环的开头，进行新一轮的抢锁。假设这次我们抢到了，我们将从 return interrupted 处返回，返回到哪里呢？ 当然是 acquireQueued 的调用处啦:

```java{.line-numbers}
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

我们看到，如果 acquireQueued 的返回值为 true, 我们将执行 selfInterrupt():

```java{.line-numbers}
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

而它的作用，就是中断当前线程。绕了这么一大圈，到最后还是中断了当前线程。为什么要这么做呢？从上面的代码中我们知道，即使线程在等待资源的过程中被中断唤醒，它还是会不依不饶的再抢锁，直到它抢到锁为止。也就是说，它是不响应这个中断的，仅仅是记录下自己被人中断过。最后，当它抢到锁返回了，如果它发现自己曾经被中断过，它就再中断自己一次，将这个中断补上。

## 3.总结

AQS 释放独占锁的步骤是：首先调用子类的 tryRelease 方法，如果返回 true，表明成功释放了锁，接下来就会尝试去唤醒 head 节点的后面节点中的线程。而且在 unparkSuccesor 方法中，寻找后序需要唤醒的节点都是从 tail 开始遍历（避免尾分叉），并且跳过 waitStatus 为 **`CANCELLED(1)`** 的节点。如果找到，则进行唤醒，使其再尝试去获取锁。

并且，在 AQS 中，当一个线程尝试去获取锁但是没有获取到时，它会被包装成 Node 结点放入到等待队列中等待唤醒，并且在它抢到锁之前，这个线程是不会响应中断的，如果确实发生了中断，那么就会将其记录在 interrupted 变量中（interrupted = true），表示发生了中断。然后当获取到锁之后，会检查 interrputed 变量的值，如果为 true，那么调用 **`Thread.currentThread().interrupt()`** 方法，来对当前线程进行中断。

