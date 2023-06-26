## send 和 recv 函数详解

### 一、send 和 recv 函数详解

recv/send 函数类似于标准的 read/write 函数，不过需要一个额外的参数：

```c{.line-numbers}
// 若成功则为读入或写出的字节数，若出错则为 -1
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buff, size_t nbytes, int flags);
ssize_t send(int sockfd, const void *buff, size_t nbytes, int flags);
```

recv 和 send 的前 3 个参数等同于 read 和 write 的 3 个参数。flags 参数的值或为0，或为下面列出的一个或多个常值的逻辑或。

<div align="center">
    <img src="12_socket_send_recv_函数详解/1.png" width="450"/>
</div>

**1) MSG_DONTROUTE 标志位**

**本标志告知内核目的主机在某个直接连接的本地网络上，因而无需执行路由表查找**。本选项规定外出的分组将绕过底层协议的正常路由机制。举例来说，在 IPv4 情况下外出分组将被定向到适当的本地接口，也就是由其目的地址的网络和子网部分确定的本地接口。如果这样的本地接口无法由目的地址确定（譬如说目的地主机不在一个点对点链路的另一端，也不在一个共享的网上），那么返回 ENETUNREACH 错误。**这个既可以使用 MSG_DONTROUTE 标志针对单个输出操作开启，也可以使用 `SO_DONTROUTE` 套接字选项针对某个给定套接字上的所有输出操作开启**。

**2) MSG_DONIWAIT**

本标志在无需打开相应套接字的非阻塞标志的前提下，把单个 I/O 操作临时指定为非阻塞，接着执行 I/O 操作，然后关闭非阻塞标志。

**3) MSG_OOB**

对于 send，本标志指明即将发送带外数据。TCP 连接上只有一个字节可以作为带外数据发送。对于recv，本标志指明即将读入的是带外数据而不是普通数据。

**4) MSG_PEEK**

本标志适用于 recv 和 recvfrom，它允许我们查看已可读取的数据，而且系统不在 recv 或 recvfrom 返回后丢弃这些数据。

**5) MSG_WAITALL**

本标志随 4.3BSD Reno 引入。它告知内核不要在尚未读入请求数目的字节之前让一个读操作返回。如果系统支持本标志，我们就可以省掉 readn 函数，而替之以如下的宏：

```c
#define readn(fd, ptr, n) recv(fd, ptr, n, MSG_WAITALL)
```

即使指定了 `MSG_WAITALL`，如果发生下列情况之一：(a) 捕获一个信号，(b) 连接被终止，(c)套接字发生一个错误，相应的读函数仍有可能返回比所请求字节数要少的数据。

### 二、阻塞和非阻塞模式

#### 1.阻塞与非阻塞模式介绍

对 socket 在阻塞和非阻塞模式下的各个函数的行为差别深入的理解是掌握网络编程的基本要求之一，是重点也是难点。阻塞和非阻塞模式下，我们常讨论的具有不同行为表现的 socket 函数一般有如下几个，见下表：

- connect
- accept
- send (Linux 平台上对 socket 进行操作时也包括 write 函数，下文中对 send 函数的讨论也适用于 write 函数)
- recv (Linux 平台上对 socket 进行操作时也包括 read 函数，下文中对 recv 函数的讨论也适用于 read 函数)

限于文章篇幅，这里只讨论 send 和 recv 函数。在正式讨论之前，我们先解释一下阻塞模式和非阻塞模式的概念。

- **阻塞模式，就当某个函数"执行成功的条件"当前不能满足时，该函数会阻塞当前执行线程**，程序执行流在超时时间到达或"执行成功的条件"满足后恢复继续执行。
- **非阻塞模式，与阻塞模式相反，即使某个函数的"执行成功的条件"不当前不能满足，该函数也不会阻塞当前执行线程，而是立即返回**，继续运行执行程序流。

#### 2.如何将 socket 设置成非阻塞模式

无论是 Windows 还是 Linux 平台，默认创建的 socket 都是阻塞模式的。在 Linux 平台上，我们可以使用 fcntl() 函数或 ioctl() 函数给创建的 socket 增加 O_NONBLOCK 标志来将 socket 设置成非阻塞模式。示例代码如下：

```c{.line-numbers}
int oldSocketFlag = fcntl(sockfd, F_GETFL, 0);
int newSocketFlag = oldSocketFlag | O_NONBLOCK;
fcntl(sockfd, F_SETFL,  newSocketFlag);
```

ioctl() 函数 与 fcntl() 函数使用方式基本一致，这里就不再给出示例代码了。**当然，Linux 下的 socket() 创建函数也可以直接在创建时将 socket 设置为非阻塞模式**，socket() 函数的签名如下：

```c
int socket(int domain, int type, int protocol);
```

给 type 参数增加一个 SOCK_NONBLOCK 标志即可，例如：

```c
int s = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, IPPROTO_TCP);
```

不仅如此，Linux 系统下利用 accept() 函数返回的代表与客户端通信的 socket 也提供了一个扩展函数 accept4()，直接将 accept 函数返回的 socket 设置成非阻塞的。

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen); 
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
```

只要将 accept4() 函数最后一个参数 flags 设置成 SOCK_NONBLOCK 即可。也就是说以下代码是等价的：

```c{.line-numbers}
// 使用 accept4 函数设置非阻塞
socklen_t addrlen = sizeof(clientaddr);
int clientfd = accept4(listenfd, &clientaddr, &addrlen, SOCK_NONBLOCK);

// 常规方式设置函数为非阻塞
socklen_t addrlen = sizeof(clientaddr);
int clientfd = accept(listenfd, &clientaddr, &addrlen);
if (clientfd != -1) {
    int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
    int newSocketFlag = oldSocketFlag | O_NONBLOCK;
    fcntl(clientfd, F_SETFL,  newSocketFlag);
}
```

#### 3.send 和 recv 函数在阻塞和非阻塞模式下的行为

send 和 recv 函数其实名不符实。

send 函数本质上并不是往网络上发送数据，而是将应用层发送缓冲区的数据拷贝到内核缓冲区（下文为了叙述方便，我们以"网卡缓冲区"代指）中去，至于什么时候数据会从网卡缓冲区中真正地发到网络中去要根据 TCP/IP 协议栈的行为来确定，这种行为涉及到一个叫 `NAGLE` 算法和 `TCP_NODELAY` 的 socket 选项。

recv 函数本质上也并不是从网络上收取数据，而只是将内核缓冲区中的数据拷贝到应用程序的缓冲区中，当然拷贝完成以后会将内核缓冲区中该部分数据移除。

可以用下面一张图来描述上述事实：

<div align="center">
    <img src="12_socket_send_recv_函数详解/2.png" width="600"/>
</div>

通过上图我们知道，不同的程序进行网络通信时，发送的一方会将内核缓冲区的数据通过网络传输给接收方的内核缓冲区。在应用程序 A 与 应用程序 B 建立了 TCP 连接之后，假设应用程序 A 不断调用 send 函数，这样数据会不断拷贝至对应的内核缓冲区中，如果 B 那一端一直不调用 recv 函数，那么 B 的内核缓冲区被填满以后，A 的内核缓冲区也会被填满，此时 A 继续调用 send 函数会是什么结果呢？ 具体的结果取决于该 socket 是否是阻塞模式。我们这里先给出结论：

- 当 socket 是阻塞模式的，继续调用 send/recv 函数会导致程序阻塞在 send/recv 调用处
- 当 socket 是非阻塞模式，继续调用 send/recv 函数，send/recv 函数不会阻塞程序执行流，而是会立即出错返回，我们会得到一个相关的错误码，Linux 平台上该错误码为 EWOULDBLOCK 或EAGAIN（这两个错误码值相同）

我们实际来编写一下代码来验证一下以上说的两种情况。

#### 4.socket 阻塞模式下的 send 行为

服务端代码 (blocking_server.cpp) 如下所示：

```c{.line-numbers}
/**
 * 验证阻塞模式下 send 函数的行为，server 端
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

int main(int argc, char* argv[]) {

    // 1.创建一个侦听 socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        std::cout << "create listen socket error." << std::endl;
        return -1;
    }

    // 2.初始化服务器地址
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    bindaddr.sin_port = htons(3000);
    // bind 监听地址，绑定的地址为 0.0.0.0:3000
    if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1) {
        std::cout << "bind listen socket error." << std::endl;
        close(listenfd);
        return -1;
    }

    // 3.启动侦听
    if (listen(listenfd, SOMAXCONN) == -1) {
        std::cout << "listen error." << std::endl;
        close(listenfd);
        return -1;
    }

    while (true) {
        struct sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        // 4.接受客户端连接
        int clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
        if (clientfd != -1) {             
            // 只接受连接，不调用 recv 收取任何数据
            std:: cout << "accept a client connection." << std::endl;
        }
    }

    // 7.关闭侦听 socket
    close(listenfd);

    return 0;
}
```

客户端代码（blocking_client.cpp）如下：

```c{.line-numbers}
/**
 * 验证阻塞模式下send函数的行为，client端
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       "helloworld"

int main(int argc, char* argv[]) {

    // 1.创建一个 socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1) {
        std::cout << "create client socket error." << std::endl;
        close(clientfd);
        return -1;
    }

    // 2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1) {
        std::cout << "connect socket error." << std::endl;
        return -1;
    }

    // 3.不断向服务器发送数据，或者出错退出
    int count = 0;
    while (true) {
        int ret = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0);
        if (ret != strlen(SEND_DATA)) {
            std::cout << "send data error." << std::endl;
            break;
        } else {
            count ++;
            std::cout << "send data successfully, count = " << count << std::endl;
        }
    }

    // 5.关闭 socket
    close(clientfd);
    return 0;
}
```

在 shell 中分别编译这两个 cpp 文件得到两个可执行程序 blocking_server 和 blocking_client：

```shell
g++ -g -o blocking_server blocking_server.cpp
g++ -g -o blocking_client blocking_client.cpp
```

我们先启动 **`blocking_server`**，然后用 gdb 启动 **`blocking_client`**，输入 run 命令让 **`blocking_client`** 跑起来，**`blocking_client`** 会不断地向 **`blocking_server`** 发送"helloworld"字符串，每次 send 成功后，会将计数器 count 的值打印出来，计数器会不断增加，程序运行一段时间后，计数器 count 值不再增加且程序不再有输出。操作过程及输出结果如下。

**`blocking_server`** 端：

```vb
[root@localhost testsocket]# ./blocking_server 
accept a client connection.
[root@localhost testsocket]# gdb blocking_client
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-100.el7_4.1
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /root/testsocket/blocking_client...done.
(gdb) run
// 输出结果太多，省略部分...
send data successfully, count = 355384
send data successfully, count = 355385
send data successfully, count = 355386
send data successfully, count = 355387
send data successfully, count = 355388
send data successfully, count = 355389
send data successfully, count = 355390
```

此时程序不再有输出，说明我们的程序应该“卡在”某个地方，继续按 Ctrl + C 让 gdb 中断下来，输入 `bt` 命令查看此时的调用堆栈，我们发现我们的程序确实阻塞在 `send` 函数调用处：

```c
^C
Program received signal SIGINT, Interrupt.
0x00007ffff72f130d in send () from /lib64/libc.so.6
(gdb) bt
#0  0x00007ffff72f130d in send () from /lib64/libc.so.6
#1  0x0000000000400b46 in main (argc=1, argv=0x7fffffffe598) at blocking_client.cpp:41
(gdb) 
```

上面的示例验证了**如果一端一直发数据，而对端应用层一直不取数据（或收取数据的速度慢于发送速度），则很快两端的内核缓冲区很快就会被填满，导致发送端调用 send 函数被阻塞**。这里说的“内核缓冲区” 其实有个专门的名字，即 **`TCP 窗口`**。也就是说 socket 阻塞模式下， send 函数在 TCP 窗口太小时的行为是阻塞当前程序执行流（即阻塞 send 函数所在的线程的执行）。

说点题外话，上面的例子，我们每次发送一个 "helloworld"（10 个字节），一共发了 355390 次（每次测试的结果略有不同），我们可以粗略地算出 TCP 窗口的大小大约等于 1.7 M左右 （10 * 355390 / 2）。

让我们再深入一点，我们利用 Linux tcpdump 工具来动态看一下这种情形下 TCP 窗口大小的动态变化。需要注意的是，Linux 下使用 tcpdump 这个命令需要有 root 权限。我们开启三个 shell 窗口，在第一个窗口先启动 blocking_server 进程，在第二个窗口用 tcpdump 抓经过 TCP 端口 3000 上的数据包：

```vb
[root@localhost testsocket]# tcpdump -i any -nn -S 'tcp port 3000'    
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
```

接着在第三个 shell 窗口，启动 blocking_client。当 blocking_client 进程不再输出时，我们抓包的结果如下：

```shell
[root@localhost testsocket]# tcpdump -i any -nn -S 'tcp port 3000' 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
11:52:35.907381 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [S], seq 1394135076, win 43690, options [mss 65495,sackOK,TS val 78907688 ecr 0,nop,wscale 7], length 0
20:32:21.261484 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [S.], seq 1233000591, ack 1394135077, win 43690, options [mss 65495,sackOK,TS val 78907688 ecr 78907688,nop,wscale 7], length 0
11:52:35.907441 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 0
11:52:35.907615 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1394135077:1394135087, ack 1233000592, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 10
11:52:35.907626 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1394135087, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 0
11:52:35.907785 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1394135087:1394135097, ack 1233000592, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 10
11:52:35.907793 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1394135097, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 0
11:52:35.907809 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1394135097:1394135107, ack 1233000592, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 10
11:52:35.907814 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1394135107, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 0
...内容太长， 部分省略...
11:52:40.075794 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395013717, win 374, options [nop,nop,TS val 78911856 ecr 78911816], length 0
11:52:40.075829 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1395013717:1395030517, ack 1233000592, win 342, options [nop,nop,TS val 78911856 ecr 78911856], length 16800
11:52:40.115847 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395030517, win 305, options [nop,nop,TS val 78911896 ecr 78911856], length 0
11:52:40.115866 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1395030517:1395047317, ack 1233000592, win 342, options [nop,nop,TS val 78911896 ecr 78911896], length 16800
11:52:40.155703 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395047317, win 174, options [nop,nop,TS val 78911936 ecr 78911896], length 0
11:52:40.155752 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1395047317:1395064117, ack 1233000592, win 342, options [nop,nop,TS val 78911936 ecr 78911936], length 16800
11:52:40.195132 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395064117, win 43, options [nop,nop,TS val 78911976 ecr 78911936], length 0
11:52:40.435748 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [P.], seq 1395064117:1395069621, ack 1233000592, win 342, options [nop,nop,TS val 78912216 ecr 78911976], length 5504
11:52:40.435782 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78912216 ecr 78912216], length 0
11:52:40.670661 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78912451 ecr 78912216], length 0
11:52:40.670674 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78912451 ecr 78912216], length 0
11:52:41.141703 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78912922 ecr 78912451], length 0
11:52:42.083643 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78913864 ecr 78912451], length 0
11:52:42.083655 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78913864 ecr 78912216], length 0
11:52:43.967506 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78915748 ecr 78913864], length 0
11:52:43.967532 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78915748 ecr 78912216], length 0
11:52:47.739259 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78919520 ecr 78915748], length 0
11:52:47.739274 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78919520 ecr 78912216], length 0
11:52:55.275863 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78927056 ecr 78919520], length 0
11:52:55.275931 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [.], ack 1395069621, win 0, options [nop,nop,TS val 78927056 ecr 78912216], length 0
```

抓取到的前三个数据包是 blocking_client 与 blocking_server 建立三次握手的过程。

```shell
11:52:35.907381 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [S], seq 1394135076, win 43690, options [mss 65495,sackOK,TS val 78907688 ecr 0,nop,wscale 7], length 0
20:32:21.261484 IP 127.0.0.1.3000 > 127.0.0.1.40846: Flags [S.], seq 1233000591, ack 1394135077, win 43690, options [mss 65495,sackOK,TS val 78907688 ecr 78907688,nop,wscale 7], length 0
11:52:35.907441 IP 127.0.0.1.40846 > 127.0.0.1.3000: Flags [.], ack 1233000592, win 342, options [nop,nop,TS val 78907688 ecr 78907688], length 0
```

示意图如下：

<div align="center">
    <img src="12_socket_send_recv_函数详解/3.png" width="300"/>
</div>

当每次 **`blocking_client`** 给 **`blocking_server`** 发数据以后，**`blocking_server`** 会应答 **`blocking_server`**，在每次应答的数据包中会带上自己的当前可用 TCP 窗口大小（看上文中结果从 127.0.0.1.3000 > 127.0.0.1.40846方向的数据包的 win 字段大小变化），由于 TCP 流量控制和拥赛控制机制的存在，**`blocking_server`** 端的 TCP 窗口大小短期内会慢慢增加，后面随着接收缓冲区中数据积压越来越多， TCP 窗口会慢慢变小，最终变为 0。

另外，细心的读者如果实际去做一下这个实验会发现一个现象，即当 tcpdump 已经显示对端的 TCP 窗口是 0 时，**`blocking_client`** 仍然可以继续发送一段时间的数据，**此时的数据已经不是在发往对端，而是逐渐填满到本端的内核发送缓冲区中去了，这也验证了 send 函数实际上是往内核缓冲区中拷贝数据这一行为**。

#### 5.非阻塞模式下 send 的行为

我们再来验证一下非阻塞 socket 的 send 行为，server 端的代码不变，我们将 blocking_client.cpp 中 socket 设置成非阻塞的，修改后的代码如下：

```cpp{.line-numbers}
/**
 * 验证非阻塞模式下 send 函数的行为，client 端，nonblocking_client.cpp
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       "helloworld"

int main(int argc, char* argv[]) {

    // 1.创建一个 socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1) {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    // 2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1) {
        std::cout << "connect socket error." << std::endl;
        close(clientfd);
        return -1;
    }

    // 连接成功以后，我们再将 clientfd 设置成非阻塞模式，
    // 不能在创建时就设置，这样会影响到 connect 函数的行为
    int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
    int newSocketFlag = oldSocketFlag | O_NONBLOCK;
    if (fcntl(clientfd, F_SETFL,  newSocketFlag) == -1) {
        close(clientfd);
        std::cout << "set socket to nonblock error." << std::endl;
        return -1;
    }

    // 3.不断向服务器发送数据，或者出错退出
    int count = 0;
    while (true) {
        int ret = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0);
        if (ret == -1) {
            // 非阻塞模式下 send 函数由于 TCP 窗口太小发不出去数据，错误码是 EWOULDBLOCK
            if (errno == EWOULDBLOCK) {
                std::cout << "send data error as TCP Window size is too small." << std::endl;
                continue;
            } else if (errno == EINTR) {
                // 如果被信号中断，我们继续重试
                std::cout << "sending data interrupted by signal." << std::endl;
                continue;
            } else {
                std::cout << "send data error." << std::endl;
                break;
            }
        }
        if (ret == 0) {
            // 对端关闭了连接，我们也关闭
            std::cout << "send data error." << std::endl;
            break;
        } else {
            count ++;
            std::cout << "send data successfully, count = " << count << std::endl;
        }
    }

    // 5.关闭 socket
    close(clientfd);
    return 0;
}
```

编译 **`nonblocking_client.cpp`** 得到可执行程序 **`nonblocking_client`**：

```shell
g++ -g -o nonblocking_client nonblocking_client.cpp 
```

运行 **`nonblocking_client`**，运行一段时间后，由于对端和本端的 TCP 窗口已满，数据发不出去了，但是 send 函数不会阻塞，而是立即返回，返回值是 -1（Windows 系统上 返回 **`SOCKET_ERROR`**，这个宏的值也是 -1），此时得到错误码是 **`EWOULDBLOCK`**。执行结果如下：

<div align="center">
    <img src="12_socket_send_recv_函数详解/4.png" width="450"/>
</div>

#### 6.socket 阻塞模式下的 recv 行为

在了解了 send 函数的行为，我们再来看一下阻塞模式下的 recv 函数行为。服务器端代码不需要修改，我们修改一下客户端代码，如果服务器端不给客户端发数据，此时客户端调用 recv 函数执行流会阻塞在 recv 函数调用处。继续修改一下客户端代码：

```cpp{.line-numbers}
/**
 * 验证阻塞模式下recv函数的行为，client端，blocking_client_recv.cpp
 * zhangyl 2018.12.17
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       "helloworld"

int main(int argc, char* argv[]) {

    // 1.创建一个 socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1) {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    // 2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1) {
        std::cout << "connect socket error." << std::endl;
        close(clientfd);
        return -1;
    }

    // 直接调用 recv 函数，程序会阻塞在 recv 函数调用处
    char recvbuf[32] = {0};
    int ret = recv(clientfd, recvbuf, 32, 0);
    if (ret > 0) {
        std::cout << "recv successfully." << std::endl;
    } else {
        std::cout << "recv data error." << std::endl;
    }

    // 5.关闭 socket
    close(clientfd);
    return 0;
}
```

编译 **`blocking_client_recv.cpp`** 并启动，我们发现程序既没有打印 **`recv`** 调用成功的信息也没有调用失败的信息，将程序中断下来，使用 bt 命令查看此时的调用堆栈，发现程序确实阻塞在 **`recv`** 函数调用处。

```c{.line-numbers}
[root@localhost testsocket]# g++ -g -o blocking_client_recv blocking_client_recv.cpp 
[root@localhost testsocket]# gdb blocking_client_recv
Reading symbols from /root/testsocket/blocking_client_recv...done.
(gdb) r
Starting program: /root/testsocket/blocking_client_recv 
^C
Program received signal SIGINT, Interrupt.
0x00007ffff72f119d in recv () from /lib64/libc.so.6
Missing separate debuginfos, use: debuginfo-install glibc-2.17-196.el7_4.2.x86_64 libgcc-4.8.5-16.el7_4.2.x86_64 libstdc++-4.8.5-16.el7_4.2.x86_64
(gdb) bt
#0  0x00007ffff72f119d in recv () from /lib64/libc.so.6
#1  0x0000000000400b18 in main (argc=1, argv=0x7fffffffe588) at blocking_client_recv.cpp:40
```

#### 7.非阻塞模式下 recv 的行为

非阻塞模式下如果当前无数据可读，recv 函数将立即返回，返回值为 -1，错误码为 EWOULDBLOCK。将客户端代码修成一下：

```cpp{.line-numbers}
/**
 * 验证阻塞模式下recv函数的行为，client端，blocking_client_recv.cpp
 * zhangyl 2018.12.17
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       "helloworld"

int main(int argc, char* argv[]) {

    // 1.创建一个 socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1) {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    // 2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1) {
        std::cout << "connect socket error." << std::endl;
        close(clientfd);
        return -1;
    }

    // 连接成功以后，我们再将 clientfd 设置成非阻塞模式，
    // 不能在创建时就设置，这样会影响到 connect 函数的行为
    int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
    int newSocketFlag = oldSocketFlag | O_NONBLOCK;
    if (fcntl(clientfd, F_SETFL,  newSocketFlag) == -1) {
        close(clientfd);
        std::cout << "set socket to nonblock error." << std::endl;
        return -1;
    }

    // 直接调用 recv 函数，程序会阻塞在 recv 函数调用处
    while (true) {
        char recvbuf[32] = {0};
        int ret = recv(clientfd, recvbuf, 32, 0);
        if (ret > 0) {
            // 收到了数据
            std::cout << "recv successfully." << std::endl;
        } else if (ret == 0) {
            // 对端关闭了连接
            std::cout << "peer close the socket." << std::endl; 
            break;
        } else if (ret == -1) {
            if (errno == EWOULDBLOCK) {
                std::cout << "There is no data available now." << std::endl;
            } else if (errno == EINTR) {
                // 如果被信号中断了，则继续重试 recv 函数
                std::cout << "recv data interrupted by signal." << std::endl;               
            } else {
                // 真的出错了
                break;
            }
        }
    }

    // 5.关闭 socket
    close(clientfd);
    return 0;
}
```

执行结果与我们预期的一模一样， recv 函数在无数据可读的情况下并不会阻塞情绪，所以程序会一直有"There is no data available now."相关的输出。

<div align="center">
    <img src="12_socket_send_recv_函数详解/5.png" width="300"/>
</div>

### 三、总结

#### 1.非阻塞模式下 send 和 recv 返回值的含义

我们来总结一下 **`send`** 和 **`recv`** 函数的各种返回值意义：

<div align="center">
    <img src="12_socket_send_recv_函数详解/6.png" width="800"/>
</div>

我们来逐一介绍下这三种情况：

**1) 返回值大于 0**

对于 **`send`** 和 **`recv`** 函数返回值大于 0，表示发送或接收多少字节，需要注意的是，在这种情形下，我们一定要判断下 **`send`** 函数的返回值是不是我们期望发送的缓冲区长度，而不是简单判断其返回值大于 0。举个例子：

```cpp
int n = send(socket, buf, buf_length, 0)；
if (n > 0) {
    printf("send data successfully\n");
}
```

很多新手会写出上述代码，虽然返回值 n 大于 0，但是实际情形下，由于对端的 TCP 窗口可能因为缺少一部分字节就满了，所以返回值 n 的值可能在 (0, buf_length] 之间，当 0 < n < buf_length 时，**虽然此时 send 函数是调用成功了，但是业务上并不算正确，因为有部分数据并没发出去**。你可能在一次测试中测不出 n 不等于 buf_length 的情况，但是不代表实际中不存在。所以，建议要么认为返回值 n 等于 buf_length 才认为正确，要么在一个循环中调用 send 函数，如果数据一次性发不完，记录偏移量，下一次从偏移量处接着发，直到全部发送完为止。

```cpp
// 推荐的方式一
int n = send(socket, buf, buf_length, 0)；
if (n == buf_length) {
    printf("send data successfully\n");
}
```

```cpp
// 推荐的方式二：在一个循环里面根据偏移量发送数据
bool SendData(const char* buf , int buf_length) {
    // 已发送的字节数目
    int sent_bytes = 0;
    int ret = 0;
    while (true) {
        ret = send(m_hSocket, buf + sent_bytes, buf_length - sent_bytes, 0);
        if (ret == -1) {
            if (errno == EWOULDBLOCK) {
                // 严谨的做法，这里如果发不出去，应该缓存尚未发出去的数据，后面介绍
                break;
            } else if (errno == EINTR) {
                continue;
            } else {
                return false;
            }
        } else if (ret == 0) {
            return false;
        }

        sent_bytes += ret;
        if (sent_bytes == buf_length)
            break;

        // 稍稍降低 CPU 的使用率
        usleep(1);
    }

    return true;
}
```

**2) 返回值等于 0**

通常情况下，**如果 send 或者 recv 函数返回 0，我们就认为对端关闭了连接，我们这端也关闭连接即可**，这是实际开发时最常见的处理逻辑。

但是，现在还有一种情形就是，假设调用 send 函数传递的数据长度就是 0 呢？send 函数会是什么行为？对端会 recv 到一个 0 字节的数据吗？需要强调的是，在实际开发中，你不应该让你的程序有任何机会去 send 0 字节的数据，这是一种不好的做法。 这里仅仅用于实验性讨论，我们来通过一个例子，来看下 send 一个长度为 0 的数据，send 函数的返回值是什么？对端会 recv 到 0 字节的数据吗？

server 端代码：

```cpp{.line-numbers}
/**
 * 验证 recv 函数接受 0 字节的行为，server 端，server_recv_zero_bytes.cpp
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <vector>

int main(int argc, char* argv[]) {
    // 1.创建一个侦听 socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        std::cout << "create listen socket error." << std::endl;
        return -1;
    }

    // 2.初始化服务器地址
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    bindaddr.sin_port = htons(3000);
    // 服务器进程绑定监听地址：0.0.0.0:3000
    if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1) {
        std::cout << "bind listen socket error." << std::endl;
        close(listenfd);
        return -1;
    }

    // 3.启动侦听
    if (listen(listenfd, SOMAXCONN) == -1) {
        std::cout << "listen error." << std::endl;
        close(listenfd);
        return -1;
    }

    int clientfd;

    struct sockaddr_in clientaddr;
    socklen_t clientaddrlen = sizeof(clientaddr);
    // 4.接受客户端连接
    clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
    if (clientfd != -1) {           
        while (true) {
            char recvBuf[32] = {0};
            // 5.从客户端接受数据，客户端没有数据来的时候会在 recv 函数处阻塞
            int ret = recv(clientfd, recvBuf, 32, 0);
            if (ret > 0) {
                std::cout << "recv data from client, data: " << recvBuf << std::endl;               
            } else if (ret == 0) {
                std::cout << "recv 0 byte data." << std::endl;
                continue;
            } else {
                // 出错
                std::cout << "recv data error." << std::endl;
                break;
            }
        }               
    }


    // 关闭客户端 socket
    close(clientfd);
    // 7.关闭侦听 socket
    close(listenfd);

    return 0;
}
```

上述代码侦听端口号是 3000，代码 55 行调用了 recv 函数，如果客户端一直没有数据，程序会阻塞在这里。client 端代码如下所示：

```cpp{.line-numbers}
/**
 * 验证非阻塞模式下 send 函数发送 0 字节的行为，client nonblocking_client_send_zero_bytes.cpp
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       ""

int main(int argc, char* argv[]) {
    // 1.创建一个 socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1) {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    // 2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1) {
        std::cout << "connect socket error." << std::endl;
        close(clientfd);
        return -1;
    }

    // 连接成功以后，我们再将 clientfd 设置成非阻塞模式，
    // 不能在创建时就设置，这样会影响到 connect 函数的行为
    int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
    int newSocketFlag = oldSocketFlag | O_NONBLOCK;
    if (fcntl(clientfd, F_SETFL,  newSocketFlag) == -1) {
        close(clientfd);
        std::cout << "set socket to nonblock error." << std::endl;
        return -1;
    }

    // 3.不断向服务器发送数据，或者出错退出
    int count = 0;
    while (true) {
        // 发送 0 字节的数据
        int ret = send(clientfd, SEND_DATA, 0, 0);
        if (ret == -1) {
            // 非阻塞模式下 send 函数由于 TCP 窗口太小发不出去数据，错误码是 EWOULDBLOCK
            if (errno == EWOULDBLOCK) {
                // 等待 3s 后再次重试发送数据
                std::cout << "send data error as TCP Window size is too small." << std::endl;
            } else if (errno == EINTR) {
                // 如果被信号中断，我们继续重试
                std::cout << "sending data interrupted by signal." << std::endl;
                continue;
            } else {
                std::cout << "send data error." << std::endl;
                break;
            }
        } else if (ret == 0){
            // 对端关闭了连接，我们也关闭
            std::cout << "send 0 byte data." << std::endl;
            break;
        } else {
            count ++;
            std::cout << "send data successfully, count = " << count << std::endl;
        }

        // 每三秒发一次
        sleep(3);
    }

    // 5.关闭 socket
    close(clientfd);
    return 0;
}
```

client 端连接服务器成功以后，每隔 3 秒调用 send 一次发送一个 0 字节的数据。除了先启动 server 以外，我们使用 tcpdump 抓一下经过端口 3000 上的数据包，使用如下命令：

```shell
1tcpdump -i any 'tcp port 3000'
```

然后启动 client ，我们看下结果：

<div align="center">
    <img src="12_socket_send_recv_函数详解/7.png" width="400"/>
</div>

客户端确实是每隔 3 秒 send 一次数据。然后，tcpdump 抓包结果输出中，除了连接时的三次握手数据包，再也无其他数据包，也就是说，send 函数发送 0 字节数据，client 的协议栈并不会把这些数据发出去。

```shell
1[root@localhost ~]# tcpdump -i any 'tcp port 3000'
2tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
3listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
417:37:03.028449 IP localhost.48820 > localhost.hbci: Flags [S], seq 1632283330, win 43690, options [mss 65495,sackOK,TS val 201295556 ecr 0,nop,wscale 7], length 0
517:37:03.028479 IP localhost.hbci > localhost.48820: Flags [S.], seq 3669336158, ack 1632283331, win 43690, options [mss 65495,sackOK,TS val 201295556 ecr 201295556,nop,wscale 7], length 0
617:37:03.028488 IP localhost.48820 > localhost.hbci: Flags [.], ack 1, win 342, options [nop,nop,TS val 201295556 ecr 201295556], length 0
```

因此，server 端也会一直没有输出，如果你用的是 gdb 启动 server，此时中断下来会发现，server 端由于没有数据会一直阻塞在 recv 函数调用处（55 行）。

<div align="center">
    <img src="12_socket_send_recv_函数详解/8.png" width="600"/>
</div>

