# Netty 之内存释放

之前的章节我们提过，堆外内存是不受 JVM 垃圾回收机制控制的，所以我们分配一块堆外内存进行 ByteBuf 操作时，使用完毕要对对象进行回收, 这一小节, 就以 PooledUnsafeDirectByteBuf 为例讲解有关内存分配的相关逻辑。PooledUnsafeDirectByteBuf 中内存释放的入口方法是其父类 AbstractReferenceCountedByteBuf 中的 release 方法:

```java{.line-numbers}
//class:AbstractReferenceCountedByteBuf
public boolean release() {
    return release0(1);
}
```

这里调用了 release0，跟进去：

```java{.line-numbers}
//class:AbstractReferenceCountedByteBuf
private boolean release0(int decrement) {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt < decrement) {
            throw new IllegalReferenceCountException(refCnt, -decrement);
        }

        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement)) {
            if (refCnt == decrement) {
                deallocate();
                return true;
            }
            return false;
        }
    }
} 
```

if (refCnt == decrement) 中判断当前 byteBuf 是否没有被引用了，如果没有被引用，则通过 deallocate() 方法进行释放。因为我们是以 PooledUnsafeDirectByteBuf 为例，所以这里会调用其父类 PooledByteBuf 的 deallocate 方法：

```java{.line-numbers}
//class:PooledByteBuf
protected final void deallocate() {
    if (handle >= 0) {
        final long handle = this.handle;
        this.handle = -1;
        memory = null;
        tmpNioBuf = null;
        chunk.arena.free(chunk, handle, maxLength, cache);
        chunk = null;
        recycle();
    }
} 
```

this.handle = -1 表示当前的 ByteBuf 不再指向任何一块内存。chunk.arena.free(chunk, handle, maxLength, cache) 这一步是将 ByteBuf 的内存进行释放。recycle() 是将对象放入的对象回收站, 循环利用。我们首先分析 free 方法：

```java{.line-numbers}
//class:PoolArena
void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
    //是否为unpooled
    if (chunk.unpooled) {
        int size = chunk.chunkSize();
        destroyChunk(chunk);
        activeBytesHuge.add(-size);
        deallocationsHuge.increment();
    } else {
        //那种级别的Size
        SizeClass sizeClass = sizeClass(normCapacity);
        //加到缓存里
        if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
            return;
        }
        //将缓存对象标记为未使用
        freeChunk(chunk, handle, sizeClass);
    }
} 
```

首先判断是不是 Unpooled，我们这里是 Pooled，所以会走到 else 块中。sizeClass(normCapacity) 计算是哪种级别的 size，我们按照 tiny 级别进行分析。cache.add(this, chunk, handle, normCapacity, sizeClass) 是将当前当前 ByteBuf 进行缓存。我们之前讲过, 再分配 ByteBuf 时首先在缓存上分配, 而这步, 就是将其缓存的过程, 跟进去：

```java{.line-numbers}
//class:PoolThreadCache
boolean add(PoolArena<?> area, PoolChunk chunk, long handle, int normCapacity, SizeClass sizeClass) {
    //拿到MemoryRegionCache节点
    MemoryRegionCache<?> cache = cache(area, normCapacity, sizeClass);
    if (cache == null) {
        return false;
    }
    //将chunk, 和handle封装成实体加到queue里面
    return cache.add(chunk, handle);
} 
```

首先根据根据类型拿到相关类型缓存节点, 这里会根据不同的内存规格去找不同的对象, 我们简单回顾一下, 每个缓存对象都包含一个 queue, queue 中每个节点是 entry, 每一个 entry 中包含一个 chunk 和 handle, 可以指向唯一的连续的内存。我们跟到 cache 中：

```java{.line-numbers}
private MemoryRegionCache<?> cache(PoolArena<?> area, int normCapacity, SizeClass sizeClass) {
    switch (sizeClass) {
    case Normal:
        return cacheForNormal(area, normCapacity);
    case Small:
        return cacheForSmall(area, normCapacity);
    case Tiny:
        return cacheForTiny(area, normCapacity);
    default:
        throw new Error();
    }
} 
```

回到 add 方法中，这里的 cache 对象调用了一个 add 方法, 这个方法就是将 chunk 和 handle 封装成一个 entry 加到 queue 里面：

```java{.line-numbers}
public final boolean add(PoolChunk<T> chunk, long handle) {
    Entry<T> entry = newEntry(chunk, handle); 
    boolean queued = queue.offer(entry);
    if (!queued) {
        entry.recycle();
    }
    return queued;
} 
```

我们之前介绍过，从在缓存中分配的时候从 queue 弹出一个 entry，**<font color="red">会放到一个对象池里面</font>**, 而这里 **`Entry<T> entry = newEntry(chunk, handle)`** 就是 **<font color="red">从对象池里去取一个 entry 对象</font>**, 然后将 chunk 和 handle 进行赋值。然后通过 queue.offer(entry) 加到 queue 中。我们回到最开始的 free 方法，**<font color="red">在 free 方法中，将 PoolChunk 加到缓存之后, 如果成功, 就会 return；如果不成功, 就会调用 **`freeChunk(chunk, handle, sizeClass)`** 方法, 这个方法的意义是, 将原先给 ByteBuf 分配的内存区段标记为未使用</font>**。跟进 freeChunk 简单分析下:

```java{.line-numbers}
//class:PoolArena
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
    final boolean destroyChunk;
    synchronized (this) {
        switch (sizeClass) {
            case Normal:
                ++deallocationsNormal;
                break;
            case Small:
                ++deallocationsSmall;
                break;
            case Tiny:
                ++deallocationsTiny;
                break;
            default:
                throw new Error();
        }
        destroyChunk = !chunk.parent.free(chunk, handle);
    }
    if (destroyChunk) {
        destroyChunk(chunk);
    }
}
```

我们再跟到 free 方法中:

```java{.line-numbers}
//class:PoolChunkList
boolean free(PoolChunk<T> chunk, long handle) {
    chunk.free(handle);
    if (chunk.usage() < minUsage) {
        remove(chunk);
        // Move the PoolChunk down the PoolChunkList linked-list.
        return move0(chunk);
    }
    return true;
} 
```

chunk.free(handle) 的意思是通过 chunk 释放一段连续的内存，我们再进入到 free 方法：

```java{.line-numbers}
void free(long handle) {
    int memoryMapIdx = memoryMapIdx(handle);
    int bitmapIdx = bitmapIdx(handle);

    if (bitmapIdx != 0) { 
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage != null && subpage.doNotDestroy;
        PoolSubpage<T> head = arena.findSubpagePoolHead(subpage.elemSize);
        synchronized (head) {
            if (subpage.free(head, bitmapIdx & 0x3FFFFFFF)) {
                return;
            }
        }
    }
    freeBytes += runLength(memoryMapIdx);
    setValue(memoryMapIdx, depth(memoryMapIdx));
    updateParentsFree(memoryMapIdx);
} 
```

if (bitmapIdx != 0) 这里判断是当前缓冲区分配的级别是 normal 还是 tiny/small，如果是 tiny/small，则会找到相关的 PoolSubpage，将其位图标记为 0。如果不是 PoolSubpage, 这里通过分配内存的反向标记, 将该内存标记为未使用。回到 PooledByteBuf 的 deallocate 方法中，最后, 通过 recycle() 将释放的 ByteBuf 放入对象回收站。