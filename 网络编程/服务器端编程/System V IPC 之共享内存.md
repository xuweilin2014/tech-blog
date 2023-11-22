# System V IPC 之共享内存

### 1.共享内存概述

现在将介绍 System V 共享内存。**共享内存允许两个或多个进程共享物理内存的同一块区域（通常被称为段）**。由于一个共享内存段会成为一个进程用户地址空间内存的一部分，**因此这种 IPC 机制无需内核介入**，只需要添加一些页表项，进程便可像访问普通物理内存那样访问共享内存。所有需要做的就是让一个进程将数据复制进共享内存中，并且这部分数据会对其他所有共享同一个段的进程可用。

> 与管道或消息队列要求发送进程将数据从用户空间的缓冲区复制进内核内存和接收进程将数据从内核内存复制进用户空间的缓冲区的做法相比，这种 IPC 技术的速度更快。但是，**共享内存机制通常需要通过某些同步方法使得进程不会出现同时访问共享内存的情况**。

使用共享内存的概念如下所示：

- 调用 **`shmget()`** 创建一个新共享内存段或取得一个既有共享内存段的标识符（即由其他进程创建的共享内存段）；
- 使用 **`shmat()`** 来附上共享内存段，即使该段成为调用进程的虚拟内存的一部分，为引用这块共享内存，程序需要使用由 **`shmat()`** 调用返回的 addr 值，它是一个指向进程的**虚拟地址空间中该共享内存段的起点的指针**；
- 调用 **`shmdt()`** 来分离共享内存段。在这个调用之后，进程就无法再引用这块共享内存了（**在进程终止时会自动完成这一步，分离共享内存段**）；
- 调用 **`shmctl()`** 来删除共享内存段。只有当当前所有附加内存段的进程都与之分离之后，内存段才会真正被销毁，**shmctl 使用 **`IPC_RMID`** 标志更多地只是标记此共享内存段需要被删除**，而不是真正执行。**只有一个进程需要执行这一步**；

### 2.创建或者打开一个共享内存段

**`shmget()`** 系统调用创建一个新共享内存段或获取一个既有段的标识符。新创建的内存段中的内容会被初始化为 0。

```c{.line-numbers}
#include <sys/shm.h>
/* returns shared memory segment identifier on success, or -1 on error */
int shmget(key_t key, size_t size, int shmflg);
```

当使用 **`shmget()`** 创建一个新共享内存段时，size 则是一个正整数，它表示需分配的共享内存的字节数。内核是以系统分页大小的整数倍来分配共享内存的，因此实际上 size 会被提升到最近的系统分页大小的整数倍。**如果使用 shmget() 来获取一个既有段的标识符，那么 size 对段不会产生任何效果**，但它必须要小于或等于段的大小。

shmflg 指定施加于新共享内存段上的权限或需检查的既有内存段的权限：

- **`IPC_CREAT`**：如果不存在与指定的 key 对应的段，那么就创建一个新段；
- **`IPC_EXCL`**：如果同时指定了 **`IPC_CREAT`** 并且与指定的 key 对应的段已经存在，那么返回 **`EEXIST`** 错误；
- **`SHM_HUGETLB`**：特权（CAP_IPC_LOCK）进程能够使用这个标记创建一个使用巨页（huge page）的共享内存段（如 x86-32 允许使用 4MB 大小的巨页代替 4KB 大小的巨页），使用巨页可以降低硬件内存管理单元的超前转换缓冲器（TLB）中的条目数量（TLB 中的条目是一个稀缺资源）；

### 3.使用共享内存

**`shmat()`** 系统调用将 shmid 标识的共享内存段附加到调用进程的虚拟地址空间中。

```c{.line-numbers}
#include <sys/shm.h>
/* returns address at which shared memory is attached on success, or (void*) -1 on error */
void* shmat(int shmid, const void* shmaddr, int shmflg);
```

shmaddr 参数有如下几种参数：

- 如果 **`shmaddr`** 是 NULL，那么段会被附加到内核所选择的一个合适的地址处；
- 如果 **`shmaddr`** 不为 NULL 并且没有设置 **`SHM_RND`**，那么段会被附加到由 **`shmaddr`** 指定的地址处，它必须是系统分页大小的一个倍数（否则会发生 **`EINVAL`** 错误）；
- 如果 shmaddr 不为 NULL 并且设置了 **`SHM_RND`**，那么段会被映射到的地址为在 shmaddr 中提供的地址被舍入到最近的常量 **`SHMLBA`** 的倍数，即 [shm_addr - (shm_addr % SHMLBA)]，**`SHM_RND`** 标志的含义是圆整（round）。即将共享内存被关联的地址向下圆整到离 shm_addr 最近的 **`SHMLBA`** 的整数倍地址处。这个常量等于系统分页大小的某个倍数。**在 x86 架构上，`SHMLBA` 的值与系统分页大小是一样的**；

不推荐将 shmaddr 设置为一个非 NULL 值，因为首先它降低了一个应用程序的可移植性（在一个 UNIX 系统上可用的地址在另外一个 UNIX 系统上不一定可用）；其次将一个共享内存段附加到一个正在使用中的特定地址处的操作会失败（比如有另外一个程序在该地址处附加了共享内存段或者创建了内存映射）。

shmat() 的函数结果是返回附加共享内存段的地址。开发人员可以像对待普通的 C 指针那样对待这个值，**通常会将 shmat() 的返回值赋给一个指向某个由程序员定义的结构的指针以便在该段上设定该结构**。

如果给 shmflg 指定 **`SHM_RDONLY`** 标记，那么该共享内存段必须以只读访问，试图更新只读段中的内容会导致段错误（**`SIGSEGV`** 信号）的发生。**如果没有指定 **`SHM_RDONLY`**，那么就既可以读取内存又可以修改内存**。

> **在一个进程中可以多次附加同一个共享内存段**（即将此共享内存段多次添加到进程的虚拟地址空间中，每次附加的虚拟地址不同），即使一个附加操作是只读的而另一个是读写的也没有关系。但是每个附加点上内存中的内容都是一样的，因为进程虚拟内存页表中的不同条目将其映射到同样的内存物理页面。

最后讲解 **`SHM_REMAP`** 标志位，在指定了这个标记之后 shmaddr 的值必须为非 NULL。这个标记要求 shmat()调用替换起点在 shmaddr 处长度为共享内存段的长度的任何既有共享内存段或内存映射。一般来讲，如果试图将一个共享内存段附加到一个已经在用的地址范围时将会导致 **`EINVAL`** 错误的发生。

### 3.分离一个共享内存段

当一个进程不再需要访问一个共享内存段时就可以调用 shmdt() 来将该段分离出其虚拟地址空间了。shmaddr 参数标识出了待分离的段，它应该是由之前的 shmat() 调用返回的一个值。

```c{.line-numbers}
#include <sys/shm.h>
/* returns 0 on success, or -1 on error */
int shmdt(const void* shmaddr);
```

分离一个共享内存段与删除它是不同的。通过 fork() 创建的子进程会继承其父进程附加的共享内存段。因此，共享内存为父进程和子进程之间的通信提供了一种简单的 IPC 方法。即子进程也可以使用父进程调用 shmat() 函数返回的共享内存段地址，读取/写入数据，实现父子进程间通信。

在一个 exec()中，所有附加的共享内存段都会被分离。在进程终止之后共享内存段也会自动被分离。

下面介绍一个共享内存的使用实例，实现一个单生产者和单消费者模式，并且通过二元信号量机制来控制读写进程对共享内存的并发访问。

首先初始化一个包含两个信号量（**`WRITE_SEM/READ_SEM`**）的信号量集，其中 **`WRITE_SEM`** 的值为 1，因为需要写进程首先往共享内存中写入数据，而 **`READ_SEM`** 的值为 0，读进程直接阻塞。当写进程写完数据后，会将 **`READ_SEM`** 的值增加 1，唤醒读进程。读进程被唤醒后，从共享内存中读取数据，然后将 **`WRITE_SEM`** 的值也增加 1，唤醒写进程，这样不断重复。

但是当写进程写完数据后，会写入一个 0 标志到共享内存中，然后将 **`READ_SEM`** 信号量的值增加 1，跳出循环并再次阻塞在 **`WRITE_SEM`** 信号量上，这是为了等待读进程收到标志后，将共享内存进行分离以及处理可能的其它善后操作。然后读进程再次将 **`WRITE_SEM`** 的值增加 1，唤醒写进程。此时，写进程会删除共享内存段，删除信号量集，然后退出。

<div align="center">
    <img src="System_V_IPC_之共享内存_static/4.png" width="500"/>
</div>

下面是写进程与读进程共享的头文件，其中最重要的是定义了 shmseg 结构，程序使用了这个结构来声明指向共享内存段的指针，这样就能给共享内存段中的字节规定一种结构。

```c{.line-numbers}
#ifndef HTTP_PARSER_SVSHM_XFR_H
#define HTTP_PARSER_SVSHM_XFR_H

#include <sys/types.h>
#include <sys/stat.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include "../semp/binary_sems.h"
#include "../get_num.h"

/* key for shared memory segment */
#define SHM_KEY 0x1234
/* key for semaphore set */
#define SEM_KEY 0x5678
/* permissions for our IPC objects */
#define OBJ_PERMS (S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP)

/* writer has access to shared memory */
#define WRITE_SEM 0
/* reader has access to shared memory */
#define READ_SEM 1

#ifndef BUF_SIZE
#define BUF_SIZE 1024
#endif

/* defines structure of shared memory segment */
struct shmseg {
    /* number of bytes used in 'buf' */
    int cnt;
    /* data being transferred */
    char buf[BUF_SIZE];
};

#endif //HTTP_PARSER_SVSHM_XFR_H
```

下面是写进程的代码：

```c{.line-numbers}
int main(int argc, char* argv[]) {

    int semid, shmid, bytes, xfrs;
    struct shmseg* shmp;
    union semun dummy;

    semid = semget(SEM_KEY, 2, IPC_CREAT | OBJ_PERMS);
    if (semid == -1)
        err_exit("semget-SEM_KEY");
    /* 初始化写者的信号量为 1 */
    if (init_sem_available(semid, WRITE_SEM) == -1)
        err_exit("init_sem_available-WRITE_SEM");
    /* 初始化读者的信号量为 0 */
    if (init_sem_in_use(semid, READ_SEM) == -1)
        err_exit("init_sem_in_use-READ_SEM");

    shmid = shmget(SHM_KEY, sizeof(struct shmseg), IPC_CREAT | OBJ_PERMS);
    if (shmid == -1)
        err_exit("shmget-SHM_KEY");

    shmp = shmat(shmid, NULL, 0);
    if (shmp == (void*) -1)
        err_exit("shmat");

    /* transfer blocks of data from stdin to shared memory */
    for (xfrs = 0, bytes = 0;; xfrs++, bytes += shmp->cnt) {
        /* wait for our turn */
        if (reserve_sem(semid, WRITE_SEM) == -1)
            err_exit("reserve semaphore");

        shmp->cnt = read(STDIN_FILENO, shmp->buf, BUF_SIZE);
        if (shmp->cnt == -1)
            err_exit("read-STDIN_FILENO");

        /* give reader a turn */
        if (release_sem(semid, READ_SEM) == -1)
            err_exit("release semaphore");

        /* have we reached EOF? we test this after giving the reader a turn so that it
         * can see the 0 value in shmp->cnt
         * 这个 shmp->cnt 的判断必须要放在 release_sem 之后，因为如果判断 shmp->cnt 为 0 后，直接 break 跳出循环，
         * 那么读进程将会一直阻塞在 READ_SEM 信号量上，而写进程也将一直阻塞在 WRITE_SEM 信号量上，形成死锁
         */
        if (shmp->cnt == 0)
            break;
    }

    /* wait until reader has let us have one more turn. we then know
     * reader has finished, and so we can delete the IPC objects
     * 一直等待读进程增加 WRITE_SEM 信号量的值，唤醒写进程
     */
    if (reserve_sem(semid, WRITE_SEM) == -1)
        err_exit("reserve semaphore");
    if (semctl(semid, 0, IPC_RMID, dummy) == -1)
        err_exit("semctl-IPC_RMID");
    if (shmdt(shmp) == -1)
        err_exit("shmdt");
    if (shmctl(shmid, IPC_RMID, 0) == -1)
        err_exit("shmctl-IPC_RMID");

    fprintf(stderr, "sent %d bytes, (%d xfrs)\n", bytes, xfrs);
    exit(EXIT_SUCCESS);
}
```

读进程的代码如下：

```c{.line-numbers}
int main(int argc, char* argv[]) {

    int semid, shmid, xfrs, bytes;
    struct shmseg* shmp;

    /* get ids for semaphore set and shared memory created by writer */
    semid = semget(SEM_KEY, 0, 0);
    if (semid == -1)
        err_exit("semget-SEM_KEY");

    shmid = shmget(SHM_KEY, 0, 0);
    if (shmid == -1)
        err_exit("shmget-SHM_KEY");

    shmp = shmat(shmid, NULL, SHM_RDONLY);
    if (shmp == (void*) -1)
        err_exit("shmat");

    /* transfer blocks of data from shared memory to stdout */
    for (xfrs = 0, bytes = 0;; xfrs++) {
        /* wait for our turn */
        if (reserve_sem(semid, READ_SEM) == -1)
            err_exit("reserve_sem-READ_SEM");

        if (shmp->cnt == 0)
            break;
        bytes += shmp->cnt;

        if (write(STDOUT_FILENO, shmp->buf, shmp->cnt) != shmp->cnt)
            err_exit("partial/failed write");

        /* give writer a turn */
        if (release_sem(semid, WRITE_SEM) == -1)
            err_exit("release semaphore");
    }

    if (shmdt(shmp) == -1)
        err_exit("shmdt");

    /* give writer one more turn, so it can clean up */
    if (release_sem(semid, WRITE_SEM) == -1)
        err_exit("release_semaphore-WRITE_SEM");

    fprintf(stderr, "received %d bytes (%d xfrs)\n", bytes, xfrs);
    exit(EXIT_SUCCESS);
}
```

上述程序运行效果如下：

```shell
# writer 进程
xuweilin@xvm:~/CLionProjects/http_parser/cmake-build-debug$ wc -c /etc/services 
12813 /etc/services
xuweilin@xvm:~/CLionProjects/http_parser/cmake-build-debug$ ./writer < /etc/services 
sent 12813 bytes, (13 xfrs)

# reader 进程
xuweilin@xvm:~/CLionProjects/http_parser/cmake-build-debug$ ./reader > out.txt 
received 12813 bytes (13 xfrs)
xuweilin@xvm:~/CLionProjects/http_parser/cmake-build-debug$ diff /etc/services out.txt 
```

### 4.共享内存在虚拟内存中的位置

如果遵循所推荐的方法，即让内核自行选择在何处附加共享内存段，那么（在 x86-32 架构上）内存布局就会像下图中所示的那样，段被附加在向上增长的堆和向下增长的栈之间未被分配的空间中，并且内存映射与共享库也是被放置在这个区域中。

<div align="center">
    <img src="System_V_IPC_之共享内存_static/5.png" width="350"/>
</div>

地址 0x40000000 被定义成了内核常量 **`TASK_UNMAPPED_BASE`**。通过将这个常量定义成一个不同的值并且重建内核可以修改这个地址的值。如果在调用 **`shmat()`**（或 **`mmap()`**）时采用了不推荐的方法，即显式地指定一个地址，那么一个共享内存段（或内存映射）可以被放置在低于 **`TASK_UNMAPPED_BASE`** 的地址处。

### 5.在共享内存中存储指针

每个进程都可能会用到不同的共享库和内存映射，并且可能会附加不同的共享内存段集。因此如果遵循推荐的做法，让内核来选择将共享内存段附加到何处，那么一个段在各个进程中可能会被附加到不同的地址上。**因此在共享内存段中存储指向段中其他地址的引用时应该使用（相对）偏移量，而不是（绝对）指针**。

<div align="center">
    <img src="System_V_IPC_之共享内存_static/6.png" width="300"/>
</div>

例如，假设一个共享内存段的起始地址为 baseaddr（即 baseaddr 的值为 shmat() 的返回值）。再假设需要在 p 指向的位置处存储一个指针，该指针指向的位置与 target 指向的位置相同。在 C 中，设置 *p 的传统做法如下所示：

```c
*p = target; /* place pointer in *p (WRONG) */
```

假设在进程 A 中，baseaddr 的值为 A，p 和 target 的绝对地址为 A + N、A + M，因此在 p 指向的共享内存位置处存储的值为 A + M（target 的绝对地址）。现在在进程 B 中，baseaddr 的值为 B，那么 *p 引用的地址（A + M）在进程 B 的地址空间中实际上是未知的。

上面这段代码存在的问题是当共享内存段被附加到另一个进程中时 target 指向的位置可能会位于一个不同的虚拟地址处，这意味着在那个进程中那个策划中存储在 *p 中的值是是无意义的。正确的做法是在 *p 中存储一个偏移量，如下所示：

```c
/* place offset in *p */
*p = (target - baseaddr);
/* 在解引用这种指针时需要颠倒上面的步骤 */
target = baseaddr + *p;
```

这里假设在各个进程中 baseaddr 指向共享内存段的起始位置（即各个进程中 shmat() 的返回值）。

### 6.共享内存的操作

shmctl() 系统调用在 shmid 标识的共享内存段上执行一组控制操作：

```c{.line-numbers}
#include <sys/shm.h>
/* returns 0 on success, or -1 on error */
int shmctl(int shmid, int cmd, struct shmid_ds* buf);
```

cmd 可以取如下的值：

**1).`IPC_RMID`**

标记这个共享内存段及其关联 **`shmid_ds`** 数据结构以便删除。如果当前没有进程附加该段，那么就会执行删除操作，否则就在所有进程都已经与该段分离（即当 **`shmid_ds`** 数据结构中 **`shm_nattch`** 字段的值为 0 时）之后再执行删除操作。

在一些应用程序中可以通过在所有进程使用 **`shmat()`** 将共享内存段附加到其虚拟地址空间之后立即将共享内存段标记为删除来确保在应用程序退出时干净地清除共享内存段。根据 shmctl 的 man page：

> *__If a segment has been marked for destruction, then the (nonstandard) SHM_DEST flag of the shm_perm.mode field in the associated data structure retrieved by IPC_STAT will be set.__*
> **_The caller must ensure that a segment is eventually destroyed; otherwise its pages that were faulted in will remain in memory or swap_**.

上面第二段话的意思是：The purpose of the second fragment is to remind you, that in a solution using shared memory you need to have some process mark it for destruction or it will remain in memory/swap forever. It might be good idea to use IPC_RMID immediately after creating segment. 假如有两个进程 A/B 附加了一个共享内存段 M，那么当 A/B 两个进程都终止时，会自动分离（detach）内存段 M，但是由于 M 没有被标记为删除，所以在 A/B 进程终止后，M 会继续保留在内存或者交换空间中。因此为了避免这种情况，A/B 中任一进程需要通过 shmctl 函数将内存段 M 标记为删除，在进程都退出之后（M 被两个进程分离），M 就会直接被删除掉。现在来解释 **`SHM_DEST`** 的含义，如果内存段 M 被一个进程标记为待删除，那么 **`SHM_DEST`** 字段就会被设置值。

**2).`IPC_STAT`**

将与这个共享内存段关联的 shmid_ds 数据结构的一个副本防止到 buf 指向的缓冲区中。

**3).`IPC_SET`**

使用 buf 指向的缓冲区中的值来更新与这个共享内存段相关联的 shmid_ds 数据结构中被选中的字段。

**4).`SHM_LOCK/SHM_UNLOCK`**

一个共享内存段可以被锁进 RAM 中，这样它就永远不会被交换出去了。这种做法能够带来性能上的提升，因为一旦段中的所有分页都驻留在内存中，就能够确保一个应用程序在访问分页时永远不会因发生缺页故障而被延迟。

- **`SHM_LOCK`**：操作将一个共享内存段锁进内存；
- **`SHM_UNLOCK`**：操作为共享内存段解锁以允许它被交换出去；

锁住一个共享内存段无法确保在 shmctl() 调用结束时段的所有分页都驻留在内存中。非驻留分页会在附加该共享内存段的进程引用这些分页时因分页故障而一个一个地被锁进内存。一旦分页因分页故障而被锁进了内存，那么分页就会一直驻留在内存中直到被解锁为止，即使所有进程都与该段分离之后也不会发生改变。换句话说，**`SHM_LOCK`** 操作为共享内存段设置了一个属性，而不是为调用进程设置了一个属性，某一个调用进程退出之后并不影响此共享内存段的性质。

### 7.共享内存关联的数据结构

每个共享内存段都有一个关联的 shmid_ds 数据结构：

```c{.line-numbers}
struct shmid_ds {
    struct ipc_perm shm_perm;        /* operation perms */
    int     shm_segsz;               /* size of segment (bytes) */
    time_t  shm_atime;               /* last shmat() time */
    time_t  shm_dtime;               /* last shmdt() time */
    time_t  shm_ctime;               /* last change time */
    unsigned short  shm_cpid;        /* pid of creator */
    unsigned short  shm_lpid;        /* pid of last shmat()/shmdt() */
    short   shm_nattch;              /* no. of currently attached processes */
};
```

除了常规的权限位之外，**`shm_perm.mode`** 字段还有两个只读位掩码标记。其中第一个是 **`SHM_DEST`**（销毁），它表示当所有进程的地址空间都与该段分离之后是否将该段标记为删除（通过 shmctl() **`IPC_RMID`** 操作）。另一个标记是 **`SHM_LOCKED`**，它表示是否将段锁进物理内存中（通过 shmctl() **`SHM_LOCK`** 操作）。

### 8.共享内存实例

接下来，我们使用共享内存来实现一个多人聊天程序（群聊），大致的思想是一个服务器子进程在接收到客户端发送过来的数据时，会将这些数据保存到共享内存中，然后通知服务器其它子进程读取共享内存中的数据，并将这些数据发送给子进程监听的客户端，这样就实现了多人群聊功能。整个多人聊天程序的架构图如下所示：

<div align="center">
    <img src="System_V_IPC_之共享内存_static/1.png" width="650"/>
</div>

在主进程的事件循环中，Main 进程不断循环处理三种事件：新客户端的连接事件、信号事件（**`SIGCHLD/SIGTERM/SIGINT`**）、子进程的写入数据事件。Main 进程的代码如下所示：

```c{.line-numbers}
int main() {

    const char* ip = "127.0.0.1";
    int port = 9523;

    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);

    listenfd = socket(PF_INET, SOCK_STREAM, 0);
    int option = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &option, sizeof(option));
    ret = bind(listenfd, (struct sockaddr*) &address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);

    /* 客户连接的编号 */
    user_count = 0;
    /* user_count -> client_data */
    users = calloc(USER_LIMIT, sizeof(struct client_data));
    /* pid -> user_count（客户编号）*/
    sub_process = calloc(PROCESS_LIMIT, sizeof(int));
    for (int i = 0; i < PROCESS_LIMIT; i++) {
        sub_process[i] = -1;
    }

    struct epoll_event events[5];
    epollfd = epoll_create(5);
    assert(epollfd != -1);
    /* epoll 事件循环监听 listenfd 上的事件 */
    addfd(epollfd, listenfd);

    ret = socketpair(PF_UNIX, SOCK_STREAM, 0, sig_pipefd);
    assert(ret != -1);
    setnonblocking(sig_pipefd[1]);
    /* epoll 事件循环监听 sig_pipefd[0] 上的事件，也就是信号处理器上的事件 */
    addfd(epollfd, sig_pipefd[0]);

    addsig(SIGCHLD, sig_handler, true);
    addsig(SIGTERM, sig_handler, true);
    addsig(SIGINT, sig_handler, true);
    addsig(SIGPIPE, SIG_IGN, true);

    bool stop_server = false;
    bool terminate = false;
    struct shmid_ds dummy;

    /* 创建共享内存，作为所有客户 socket 连接的读缓存 */
    shm_id = shmget(IPC_PRIVATE, USER_LIMIT * BUFFER_SIZE, IPC_CREAT | S_IRUSR | S_IWUSR);
    shmp = shmat(shm_id, NULL, 0);
    printf("addr: %p\n", shmp);

    /* 先预先删除共享内存，当所有使用此共享内存的进程与其分离之后，操作系统就会自动删除共享内存 */
    shmctl(shm_id, IPC_RMID, &dummy);

    while (!stop_server) {
        /* epoll_wait 会监听 sig_pipefd[0] 信号处理器、listenfd 监听套接字、以及 pipefd[0] 子进程的管道描述符这三类描述符 */
        int number = epoll_wait(epollfd, events, 5, -1);
        if (number < 0 && errno != EINTR) {
            printf("epoll failure\n");
            break;
        }

        for (int i = 0; i < number; i++) {
            int sockfd = events[i].data.fd;
            /********************** 新的客户连接到来 **********************/
            if (sockfd == listenfd) {
                struct sockaddr_in client_address;
                socklen_t client_addr_len = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr*) &client_address, &client_addr_len);
                if (connfd < 0) {
                    printf("errno is: %d (%s)\n", errno, strerror(errno));
                    break;
                }

                if (user_count >= USER_LIMIT) {
                    const char* info = "too many users\n";
                    printf("%s", info);
                    send(connfd, info, strlen(info), 0);
                    close(connfd);
                    continue;
                }

                /* 保存第 user_count 个客户连接的相关数据 */
                users[user_count].address = client_address;
                users[user_count].connfd = connfd;
                /* 在主进程和子进程之间建立管道，以传递必要的数据 */
                ret = socketpair(PF_UNIX, SOCK_STREAM, 0, users[user_count].pipefd);
                assert(ret != -1);

                pid_t pid = fork();
                if (pid < 0) {
                    close(connfd);
                    continue;
                } else if (pid == 0) {
                    close(epollfd);
                    close(listenfd);
                    close(sig_pipefd[0]);
                    close(sig_pipefd[1]);
                    close(users[user_count].pipefd[0]);
                    run_child(user_count, users, shmp);
                    shmdt(shmp);
                    _exit(EXIT_SUCCESS);
                } else {
                    close(connfd);
                    close(users[user_count].pipefd[1]);
                    addfd(epollfd, users[user_count].pipefd[0]);
                    users[user_count].pid = pid;
                    /* 记录新的客户连接在数组 users 中的索引值，建立进程 pid 和该索引值之间的映射关系 */
                    sub_process[pid] = user_count;
                    user_count++;
                }
            }
            /********************** 处理信号事件 **********************/
            else if ((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN)) {
                int sig;
                char signals[1024];
                while ((ret = recv(sig_pipefd[0], signals, sizeof(signals), 0)) > 0) {
                    for (int j = 0; j < ret; ++j) {
                        switch (signals[j]) {
                            /* 子进程退出，表示有某个客户端关闭了连接 */
                            case SIGCHLD: {
                                pid_t pid;
                                int stat;
                                while ((pid = waitpid(-1, &stat, WNOHANG)) > 0) {
                                    /* 用子进程的 pid 取得被关闭的客户连接的编号 */
                                    int del_user = sub_process[pid];
                                    sub_process[pid] = -1;
                                    if (del_user < 0 || del_user >= USER_LIMIT) {
                                        continue;
                                    }

                                    /* 清除第 del_user 个客户连接使用的相关数据 */
                                    epoll_ctl(epollfd, EPOLL_CTL_DEL, users[del_user].pipefd[0], 0);
                                    close(users[del_user].pipefd[0]);
                                    users[del_user] = users[--user_count];
                                    sub_process[users[del_user].pid] = del_user;
                                }

                                if (terminate && user_count == 0) {
                                    stop_server = true;
                                }

                                break;
                            }
                            case SIGINT:
                            case SIGTERM: {
                                /* 结束服务器程序 */
                                printf("kill all the child now\n");
                                /* 当外部要终止服务器程序时，如果子进程为 0，那么直接设置 stop_server */
                                if (user_count == 0) {
                                    stop_server = true;
                                    break;
                                }

                                for (int k = 0; k < user_count; k++) {
                                    int pid = users[k].pid;
                                    kill(pid, SIGTERM);
                                }

                                /* 说明服务器主进程将要被关闭退出 */
                                terminate = true;
                                break;
                            }
                            default: {
                                break;
                            }
                        }
                    }
                }

                if (ret == -1 || ret == 0) {
                    continue;
                }
            /********************** 某个子进程向父进程写入了数据 **********************/
            } else if (events[i].events & EPOLLIN) {
                int child = 0;
                /* 读取管道数据，child 变量记录了是哪个客户连接有数据到达 */
                ret = recv(sockfd, (char*)&child, sizeof(child), 0);
                printf("read data from child across pipe %d\n", ret);

                if (ret == -1) {
                    continue;
                } else if (ret == 0) {
                    continue;
                } else {
                    /* 向除负责处理第 child 个客户连接的子进程之外的其他子进程发送消息，通知它们有客户数据要写 */
                    for (int j = 0; j < user_count; j++) {
                        if (users[j].pipefd[0] != sockfd) {
                            printf("send data to child accross pipe\n");
                            send(users[j].pipefd[0], (char*) &child, sizeof(child), 0);
                        }
                    }
                }
            }
        }
    }

    del_resouce();

    return 0;
}
```

对于新客户端的连接事件（服务器最多只能与 5 个客户端建立连接），**会创建一个新的子进程来处理此客户端上的各种读/写事件，同时创建一个 **`socketpair`** 类型的 **`pipefd[2]`** 描述符，此描述符用于主/子进程之间的通信**，注意 **`socketpair`** 函数创建的 **`pipefd[2]`** 描述符是全双工的，**`pipefd[0]/pipefd[1]`** 既可以用来发送数据，也可以用来接收数据。最后会把子进程的信息保存到 **`client_data`** 数组中。

对于信号事件，主进程在启动时，注册了 **`SIGCHLD/SIGINT/SIGTERM`** 三个信号的处理函数。主进程程序中有一个全局变量 **`sig_pipefd[2]`**，由 **`socketpair`** 函数创建，也具备全双工通信的能力，此描述符用于信号处理器和主进程之间的全双工通信。当信号产生并触发信号处理器时，处理器会将此信号通过 **`sigpipefd[1]`** 写入到管道中，然后在管道中依次读取。

当产生 **`SIGCHLD`** 信号时，说明有子进程退出，由于信号处理器不具备排队机制，即在信号处理器被触发时，如果同时又有多个 **`SIGCHLD`** 信号产生，那么这些 **`SIGCHLD`** 信号都会被屏蔽处于等待状态，**待屏蔽状态被解除后，只有一个 **`SIGCHLD`** 信号会被传递给主进程**，因此在 Main 进程中，需要循环调用 **`waitpid`** 函数来避免退出子进程处于僵尸状态。同时，主进程不再监听用于和此子进程通信的 **`pipefd[0]`** 描述符，并且将子进程监听的客户连接数据也清除掉。

这里重点关注下面两行代码：

```c{.line-numbers}
users[del_user] = users[--user_count];
sub_process[users[del_user].pid] = del_user;
```

假设现在要删除的连接序号 **`del_user`** 为 2，而下一个连接序号 **`user_count`** 为 3（需注意，根据代码，在 **`user_count`** 序号之前的连接数据永远都是正常填充的），第 1 行代码将 **`user_count=2`** 的数据赋值给 **`del_user=2`** 处，看似好像没有进行删除，**但是 **`user_count`** 表示数组中最小的没有连接数据的下标值**，下一次有新的连接建立，就会将连接对应的 **`client_data`** 数据结构保存到 **`user_count`** 处，覆盖掉原有的连接数据。

<div align="center">
    <img src="System_V_IPC_之共享内存_static/2.png" width="400"/>
</div>

假设现在要删除的连接序号 **`del_user`** 为 1，而下一个连接序号 **`user_count`** 为 4，第 1 行代码将 **`user_count=3`** 处的数据赋值给 **`del_user=1`** 处，下次有新的连接建立，会将连接对应的 **`client_data`** 数据保存到 **`user_count=3`** 处，对原有的数据进行覆盖。

<div align="center">
    <img src="System_V_IPC_之共享内存_static/3.png" width="400"/>
</div>

当产生 **`SIGTERM/SIGINT`** 信号时，若 **`user_count`** 为 0，即没有任何连接了，那么直接设置 **`stop_server=true`**，否则就对剩余的每个子进程发送 **`SIGTERM`** 信号，强行关闭子进程，**`terminate`** 变量被设置为 true。当服务器处理子进程退出的 **`SIGCHLD`** 信号时，会检测 **`terminate`** 和 **`user_count`** 的值，只有当 terminate 为 true 并且 **`user_count`** 为 0 时，那么才会停止主进程的事件循环并退出。

因为 terminate 表示外部需要终止服务器的主进程，只有这个值为 true，当没有子进程时，才会终止主进程的事件循环并退出，否则单纯的子进程为 0，只能说明现在没有新的连接建立，不会终止服务器。

子进程的事件循环代码如下所示：

```c{.line-numbers}
/* 
 * 子进程运行的函数，参数 idx 指出该子进程处理的客户连接的编号，users 是保存所有客户连接数据的数组，
 * 参数 shmp 指出共享内存的起始位置
 */
void run_child(int idx, struct client_data* users, char* shmp) {
    struct epoll_event events[5];
    /* 子进程使用 I/O 复用技术来同时监听两个文件描述符：客户连接 socket、与父进程通信的管道文件描述符 */
    int child_epollfd = epoll_create(5);
    assert(child_epollfd != -1);
    /* 子进程监听的客户端套接字描述符 connfd */
    int connfd = users[idx].connfd;
    addfd(child_epollfd, connfd);
    /* pipefd[1] 用于此子进程和主进程之间的通信 */
    int pipefd = users[idx].pipefd[1];
    addfd(child_epollfd, pipefd);

    int ret;
    /* 子进程需要设置自己的信号处理函数，当收到 SIGTERM 信号时，会停止子进程的事件监听循环 */
    addsig(SIGTERM, child_term_handler, false);

    while (!stop_child) {
        int number = epoll_wait(child_epollfd, events, 5, -1);
        if ((number < 0) && (errno != EINTR)) {
            printf("epoll failure\n");
            break;
        }

        for (int i = 0; i < number; i++) {
            int sockfd = events[i].data.fd;
            /* 本子进程负责的客户连接有数据到达 */
            if ((sockfd == connfd) && (events[i].events & EPOLLIN)) {
                memset(shmp + idx * BUFFER_SIZE, '\0', BUFFER_SIZE);
                /* 将客户数据读取到对应的读缓存中。该读缓存是共享内存的一段，它开始于 idx * BUFFER_SIZE 处，长度为 BUFFER_SIZE 字节，
                 * 因此，各个客户连接的读缓存是共享的
                 */
                ret = recv(connfd, shmp + idx * BUFFER_SIZE, BUFFER_SIZE - 1, 0);

                if (ret < 0) {
                    if (errno != EAGAIN) {
                        stop_child = true;
                    }
                } else if (ret == 0) {
                    stop_child = true;
                } else {
                    /* 成功读取客户数据后就通知主进程（通过管道）来处理 */
                    send(pipefd, (char*)&idx, sizeof(idx), 0);
                }
            /* 主进程通知本进程（通过管道）将第 client 个客户的数据发送到本进程负责的客户端 */
            } else if((sockfd == pipefd) && (events[i].events & EPOLLIN)) {
                int client = 0;
                ret = recv(sockfd, (char *)&client, sizeof(client), 0);
                if (ret < 0) {
                    if (errno != EAGAIN) {
                        stop_child = true;
                    }
                } else if (ret == 0) {
                    stop_child = true;
                } else {
                    send(connfd, shmp + client * BUFFER_SIZE, BUFFER_SIZE, 0);
                }
            } else {
                continue;
            }
        }
    }

    close(connfd);
    close(pipefd);
    close(child_epollfd);
}
```

子进程需要处理客户端发送数据事件、主进程写入数据事件。

当客户端有数据发送过来时，子进程会将数据保存到共享内存中（地址为 **`shmp + idx * BUFFER_SIZE`**），然后会将客户端的编号（idx）发送给主进程。主进程会将 idx 编号转发给其余所有的子进程，这些子进程会从 **`shmp + idx * BUFFER_SIZE`** 处读取保存的数据，然后发送给客户端。这样就实现了群聊的功能。

下面程序的一些公共部分：

```c{.line-numbers}
#define USER_LIMIT 5
#define BUFFER_SIZE 1024
#define FD_LIMIT 65535
#define MAX_EVENT_NUMBER 1024
#define PROCESS_LIMIT 65536
/* key for shared memory segment */
#define SHM_KEY 0x1234

char *shmp;

/* 处理一个客户端连接必要的数据 */
struct client_data {
    /* 子进程处理连接的客户端地址 */
    struct sockaddr_in address;
    /* 子进程处理连接的套接字描述符 connfd */
    int connfd;
    /* 处理这个连接的子进程的 pid */
    pid_t pid;
    /* pipefd[2] 中的 pipefd[0] 属于主进程，而 pipefd[1] 属于子进程，pipefd[2] 是由 socketpair 函数创建的，具有双向通信功能，
     * 即每个描述符 pipefd[0]/pipefd[1] 既可以发送数据，也可以接收数据；而一般的管道 pfd[0] 用于读取数据，而 pfd[1] 用于写入数据，
     * pipefd[2] 用于主进程和子进程之间的通信使用
     */
    int pipefd[2];
};

/* 专用于主进程与信号处理器之间通信的全双工通道，由 socketpair 创建完成 */
int sig_pipefd[2];
/* 主进程的监听 epoll 描述符 */
int epollfd;
int listenfd;
int shm_id;
/* 客户连接数组，进程用客户连接的编号 user_count 来索引这个数组，即可取得相关的客户连接数据 client_data */
struct client_data* users = 0;
/* 子进程和客户连接的映射关系表，用进程的 PID 来索引这个数组，即可取得该进程所处理的客户连接的编号 user_count */
int* sub_process = 0;
/* 当前客户的数量 and 客户连接的编号 */
int user_count = 0;
bool stop_child = false;

int setnonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

/* 将文件描述符添加到 epollfd 中进行监控 */
void addfd(int epollfd, int fd) {
    struct epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}

void sig_handler(int sig) {
    int save_errno = errno;
    char msg = sig;
    /* 向 sig_pipefd[1] 管道中写入 msg 即产生的信号值 sig，统一事件源 */
    send(sig_pipefd[1], &msg, 1, 0);
    errno = save_errno;
}

void addsig(int sig, void(*handler)(int), bool restart) {
    struct sigaction sa;
    memset(&sa, '\0', sizeof(sa));
    sa.sa_handler = handler;
    if (restart) {
        sa.sa_flags |= SA_RESTART;
    }
    /* 在信号处理函数被调用时，为了防止同时接收其他信号的干扰，通常会将一些信号加入到屏蔽信号集中。
     * 屏蔽信号集用于指定在当前信号处理函数执行期间应该被阻塞的信号
     * 下面会屏蔽掉所有的信号
     */
    sigfillset(&sa.sa_mask);
    assert(sigaction(sig, &sa, NULL) != -1);
}

void del_resouce() {
    close(sig_pipefd[0]);
    close(sig_pipefd[1]);
    close(listenfd);
    close(epollfd);
    free(users);
    free(sub_process);
    // 将共享内存进行分离 shmdt
    shmdt(shmp);
}

/* 停止一个子进程 */
/* 每个子进程的虚拟地址空间不同，故一个子进程中的 stop_child 变量发生变化之后，其它子进程中同样的变量不受影响 */
void child_term_handler(int sig) {
    stop_child = true;
}
```