## Socket 中 read 和 write 函数

### 1.read/write 的语义：为什么会阻塞？

先从 write 说起：

```c
#include <unistd.h>
ssize_t write(int fd, const void *buf, size_t count);
```

首先，write 成功返回，**只是 buf 中的数据被复制到了 kernel 中的 TCP 发送缓冲区**。至于数据什么时候被发往网络，什么时候被对方主机接收，什么时候被对方进程读取，系统调用层面不会给予任何保证和通知。

write 在什么情况下会阻塞？当 kernel 的该 socket 的发送缓冲区已满时。对于每个 socket，拥有自己的 send buffer 和 receive buffer。从 Linux 2.6 开始，两个缓冲区大小都由系统来自动调节（autotuning），但一般在 default 和 max 之间浮动。

```c
# 获取 socket 的发送/接受缓冲区的大小：（后面的值是在我在 Linux 2.6.38 x86_64 上测试的结果）
sysctl net.core.wmem_default       #126976
sysctl net.core.wmem_max　　　　    #131071
sysctl net.core.wmem_default       #126976
sysctl net.core.wmem_max           #131071
```

已经发送到网络的数据依然需要暂存在 send buffer 中，只有收到对方的 ack 后，kernel 才从buffer 中清除这一部分数据，为后续发送数据腾出空间。接收端将收到的数据暂存在 receive buffer 中，自动进行确认。但如果 socket 所在的进程不及时将数据从 receive buffer 中取出，最终导致 receive buffer 填满，由于 TCP 的滑动窗口和拥塞控制，接收端会阻止发送端向其发送数据。这些控制皆发生在 TCP/IP 栈中，对应用程序是透明的，应用程序继续发送数据，最终导致send buffer 填满，write 调用阻塞。

一般来说，**由于接收端进程从 socket 读数据的速度跟不上发送端进程向 socket 写数据的速度，最终导致发送端 write 调用阻塞**。

而 read 调用的行为相对容易理解，从 socket 的 receive buffer 中拷贝数据到应用程序的buffer 中。**read 调用阻塞，通常是发送端的数据没有到达**。

### 2.blocking（默认）和 nonblock 模式下 read/write 行为的区别

将 socket fd 设置为 nonblock（非阻塞）是在服务器编程中常见的做法，采用 blocking I/O 并为每一个 client 创建一个线程的模式开销巨大且可扩展性不佳（带来大量的切换开销），**更为通用的做法是采用线程池 + Nonblock I/O + Multiplexing（select/poll，以及 Linux 上特有的 epoll）**。因为 select/poll/epoll 这些多路 IO 复用函数是阻塞的，即程序会阻塞在这些函数上等待描述符上的读写事件准备就绪，而在描述符就绪之后，可以直接调用非阻塞的 read/write 方法，直接返回结果。

```c{.line-numbers}
// 设置一个文件描述符为 nonblock
int set_nonblocking(int fd)
{
    int flags;
    if ((flags = fcntl(fd, F_GETFL, 0)) == -1)
        flags = 0;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

几个重要的结论：

1. **read 总是在接收缓冲区有数据时立即返回，而不是等到给定的 read buffer 填满时返回**。只有当 receive buffer 为空时，blocking 模式才会等待，而 nonblock 模式下会立即返回 -1（errno = EAGAIN 或 EWOULDBLOCK）

2. **blocking 的 write 只有在缓冲区足以放下整个 buffer 时才返回（与 blocking read 并不相同）**。nonblock write 则是返回能够放下的字节数，之后调用如果缓冲区已满则返回 -1 (errno = EAGAIN 或 EWOULDBLOCK)。

#### 3.read/write 对连接异常的反馈行为

对应用程序来说，与另一进程的 TCP 通信其实是完全异步的过程：

1. 我并不知道对面什么时候、能否收到我的数据
2. 我不知道什么时候能够收到对面的数据
3. 我不知道什么时候通信结束（主动退出或是异常退出、机器故障、网络故障等等）

对于 1 和 2，采用 write() -> read() -> write() -> read() ->...的序列，通过 blocking read或者 nonblock read + 轮询的方式，应用程序基于可以保证正确的处理流程。对于 3，kernel 将这些事件的"通知"通过 read/write 的结果返回给应用层。

假设 A 机器上的一个进程 a 正在和 B 机器上的进程 b 通信：某一时刻 a 正阻塞在 socket 的 read 调用上（或者在 nonblock 下轮询 socket）。当 b 进程终止时，无论应用程序是否显式关闭了 socket（OS 会负责在进程结束时关闭所有的文件描述符，对于 socket，则会发送一个 FIN 包到对面）。

"同步通知"：进程 a 对已经收到 FIN 的 socket 调用 read，如果已经读完了 receive buffer 的剩余字节，则会返回 EOF:0

"异步通知"：如果进程 a 正阻塞在 read 调用上（前面已经提到，此时 receive buffer 一定为空，因为 read 在 receive buffer 有内容时就会返回），则 read 调用立即返回 EOF，进程 a 被唤醒。

socket 在收到 FIN 后，虽然调用 read 会返回 EOF，但进程 a 依然可以其调用 write，因为根据 TCP 协议，收到对方的 FIN 包只意味着对方不会再发送任何消息。 在一个双方正常关闭的流程中，收到FIN 包的一端将剩余数据发送给对面（通过一次或多次 write），然后关闭 socket。

但是事情远远没有想象中简单。优雅地（gracefully）关闭一个 TCP 连接，不仅仅需要双方的应用程序遵守约定，中间还不能出任何差错。

假如 b 进程是异常终止的，发送 FIN 包是 OS 代劳的，b 进程已经不复存在，当机器再次收到该 socket 的消息时，会回应 RST（因为拥有该 socket 的进程已经终止）。a 进程对收到 RST 的 socket再次调用 write 时，操作系统会给 a 进程发送 SIGPIPE，默认处理动作是终止进程，知道你的进程为什么毫无征兆地死亡了。

from 《Unix Network programming, vol1》 3rd Edition：

> "It is okay to write to a socket that has received a FIN, but it is an error to write to a socket that has received an RST."

通过以上的叙述，内核通过 socket 的 read/write 将双方的连接异常通知到应用层，虽然很不直观，似乎也够用。这里说一句题外话：

不知道有没有同学会和我有一样的感慨：在写 TCP/IP 通信时，似乎没怎么考虑连接的终止或错误，只是在 read/write 错误返回时关闭 socket，程序似乎也能正常运行，但某些情况下总是会出奇怪的问题。想完美处理各种错误，却发现怎么也做不对。

原因之一是：**socket（或者说 TCP/IP 栈本身）对错误的反馈能力是有限的**。考虑这样的错误情况：

不同于 b 进程退出（此时 OS 会负责为所有打开的 socket 发送 FIN 包），当 B 机器的 OS 崩溃（注意不同于人为关机，因为关机时所有进程的退出动作依然能够得到保证）/主机断电/网络不可达时，a 进程根本不会收到 FIN 包作为连接终止的提示。

如果 a 进程阻塞在 read 上，那么结果只能是永远的等待。

如果 a 进程先 write 然后阻塞在 read，由于收不到 B 机器 TCP/IP 栈的 ack，TCP 会持续重传 12 次（时间跨度大约为 9 分钟），然后在阻塞的 read 调用上返回错误：ETIMEDOUT/EHOSTUNREACH/ENETUNREACH

假如 B 机器恰好在某个时候恢复和 A 机器的通路，并收到 a 某个重传的 packet，因为不能识别所以会返回一个 RST，此时 a 进程上阻塞的 read 调用会返回错误 ECONNREST

socket 对这些错误还是有一定的反馈能力的，前提是在对面不可达时你依然做了一次 write 调用，而不是轮询或是阻塞在 read 上，那么总是会在重传的周期内检测出错误。如果没有那次 write 调用，应用层永远不会收到连接错误的通知。

#### 4. write 的附加说明

write 函数的返回值：**如果顺利 write() 会返回实际写入的字节数。当有错误发生时则返回 -1，错误代码存入 errno 中**。

1. write() 函数返回值一般无 0，只有当如下情况发生时才会返回 0：write(fp, p1+len, (strlen(p1)-len)) 中第三参数为 0，此时 write() 什么也不做，只返回 0。
2. write() 函数从 buf 写数据到 fd 中时，若 buf 中数据无法一次性读完，那么第二次读 buf 中数据时，其读位置指针（也就是第二个参数 buf）不会自动移动，需要程序员编程控制，而不是简单的将 buf 首地址填入第二参数即可。如可按如下格式实现读位置移动：write(fp, p1+len, (strlen(p1)-len))。 这样 write 第二次循环时变会从 p1 + len 处写数据到 fp，之后的也由此类推，直至(strlen(p1)-len) 变为 0。
3. 在 write 一次可以写的最大数据范围内（貌似是 BUFSIZ，8192），第三参数 count 大小最好为 buf 中数据的大小，以免出现错误。举例如下所示：

```c{.line-numbers}
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
int main()
{
  char *p1 = "This is a c test code";
  volatile int len = 0;
 
  int fp = open("/home/test.txt", O_RDWR|O_CREAT);
  for(;;)
  {
     int n;
 
     if((n = write(fp, p1+len, (strlen(p1)-len))) == 0)   // if((n=write(fp, p1+len, 3)) == 0) 
     {                                                 //strlen(p1) = 21
         printf("n = %d \n", n);
         break;
     }
     len += n;
  }
  return 0;
}
```

此程序中的字符串 "This is a c test code" 有 21 个字符，经笔者亲自试验，若 write 时每次写 3个字节，虽然可以将 p1 中数据写到 fp 中，但文件 test.txt 中会带有很多乱码。唯一正确的做法还是将第三参数设为 (strlen(p1) - len)，这样当 write 到 p1 末尾时 (strlen(p1) - len) 将会变为0，此时符合附加说明（1）中所说情况，write 返回 0，write 结束。

写的本质也不是进行发送操作，而是把用户态的数据 copy 到系统底层去，然后再由系统进行发送操作，send，write 返回成功，只表示数据已经 copy 到底层缓冲，而不表示数据已经发出，更不能表示对方端口已经接收到数据。

**阻塞情况下**            

阻塞情况下，write 会将数据发送完。(不过可能被中断)，在阻塞的情况下，是会一直等待，直到 write 完，全部的数据再返回，这点行为上与读操作有所不同。正如前面所说，**blocking 的 write 只有在缓冲区足以放下整个 buffer 时才返回**。

读，究其原因主要是读数据的时候我们并不知道对端到底有没有数据，数据是在什么时候结束发送的，如果一直等待就可能会造成死循环，所以并没有去进行这方面的处理，只要接受缓冲区中有数据，就可以返回。

写，而对于 write，由于需要写的长度是已知的，所以可以一直再写，直到写完。不过问题是 write 是可能被打断，造成 write 一次只 write 一部分数据，**所以 write 的过程还是需要考虑循环 write**，只不过多数情况下次 write 调用就可能成功。

**非阻塞情况下**

非阻塞写的情况下，是采用可以写多少就写多少的策略。与读不一样的地方在于，有多少读多少是由网络发送的那一端是否有数据传输到为标准，但是对于可以写多少是由本地的网络堵塞情况为标准的。在网络阻塞严重的时候，网络层没有足够的内存来进行写操作，这时候就会出现写不成功的情况（只能写出一部分），阻塞情况下会尽可能(有可能被中断) 等待到数据全部发送完毕，**对于非阻塞的情况就是一次写多少算多少，没有中断的情况下也还是会出现 write 到一部分的情况**。

#### 5.read 附加说明

返回值：**返回值为实际读取到的字节数，如果返回 0，表示已到达文件尾或是无可读取的数据。若参数count 为 0，则 read() 不会有作用并返回 0**。

**阻塞情况下：**

- 如果没有发现数据在网络缓冲中会一直等待，
- 当发现有数据的时候会把数据读到用户指定的缓冲区，但是如果这个时候读到的数据量比较少，比参数中指定的长度要小，read 并不会一直等待下去，而是立刻返回非阻塞情况下

**在非阻塞的情况下：**

- 如果发现没有数据就直接返回，
- 如果发现有数据那么也是采用有多少读多少的进行处理．

read 的原则为：数据在不超过指定的长度的时候有多少读多少，没有数据就会一直等待（对于阻塞情况而言）。**所以一般情况下我们读取数据都需要采用循环读的方式读取数据，因为一次 read 完毕不能保证读到我们需要长度的数据**，read 完一次需要判断读到的数据长度再决定是否还需要再次读取。

#### 6.非阻塞 I/O 的概念

如果在 open 一个设备时指定了 O_NONBLOCK 标志，read/write 就不会阻塞。以 read 为例，如果设备暂时没有数据可读就返回 -1，同时置 errno 为 EWOULDBLOCK（或者 EAGAIN，这两个宏定义的值相同），表示本来应该阻塞在这里（would block，虚拟语气），事实上并没有阻塞而是直接返回错误，调用者应该试着再读一次（again）。这种行为方式称为轮询（Poll），调用者只是查询一下，而不是阻塞在这里死等，这样可以同时监视多个设备：

```c
while(true)
{
    非阻塞read(设备1);
    if(设备1有数据到达)
        处理数据; 
        
    非阻塞read(设备2);
    if(设备2有数据到达)
        处理数据; 

    ...
}
```

果 read(设备1) 是阻塞的，那么只要设备 1 没有数据到达就会一直阻塞在设备 1 的 read 调用上，即使设备 2 有数据到达也不能处理，使用非阻塞 I/O 就可以避免设备 2 得不到及时处理（或者使用多路 I/O 复用）。

非阻塞 I/O 有一个缺点，如果所有设备都一直没有数据到达，调用者需要反复查询做无用功，如果阻塞在那里，操作系统可以调度别的进程执行，就不会做无用功了。在使用非阻塞 I/O 时，通常不会在一个while 循环中一直不停地查询（这称为 Tight Loop），而是每延迟等待一会儿来查询一下，以免做太多无用功，在延迟等待的时候可以调度其它进程执行。

```c
while(true)
{
    非阻塞read(设备1);
    if(设备1有数据到达)
        处理数据; 
        
    非阻塞read(设备2);
    if(设备2有数据到达)
        处理数据; 

    ...
  sleep(n);  
}
```

这样做的问题是，设备 1 有数据到达时可能不能及时处理，最长需延迟 n 秒才能处理，而且反复查询还是做了很多无用功。而 select/poll/epoll 等函数可以阻塞地同时监视多个设备，还可以设定阻塞等待的超时时间，从而圆满地解决了这个问题。