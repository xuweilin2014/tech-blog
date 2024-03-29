# 服务器编程示例与难点

## 一、简介

我们将在本章使用 socket 的基本函数编写一个完整的 TCP 客户/服务器程序示例。这个简单的例子是执行如下步骤的一个回射服务器（Echo Server）：

（1）客户从标准输入读入一行文本，并写给服务器；
（2）服务器从网络输入读入这行文本，并回射给客户；
（3）客户从网络输入读入这行回射文本，并显示在标准输出上。

其结构如下图所示：

<div align="center">
    <img src="Socket_服务器编程难点/1.png" width="500"/>
</div>

实现任何客户/服务器网络应用所需的所有基本步骤都可以通过 Echo Server 阐明，如果想要把这个例子扩充成其他的应用程度，只需要修改服务器对来自客户的输入处理过程。

### 1.TCP 回射服务器程序：main 函数

```c{.line-numbers}
int main(int argc, char **argv){
    int listenfd, connfd;
    pid_t childpid;
    socklen_t clilen;
    struct sockaddr_in cliaddr, servaddr;
    // Returns a file descriptor for the new socket, or -1 for errors.
    listenfd = Socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd==-1){
        exit(0);
    }
    bzero(&servaddr, sizeof(servaddr));

    // 设置服务协议为 IPv4
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr= htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);
    Bind(listenfd, (SA *) &servaddr, sizeof (servaddr));
    Listen(listenfd, LISTENQ);

    for(;;){
        clilen = sizeof(cliaddr);

        // 获取已连接连接队列(已完成三次握手的连接)获取队头的连接,如果已连接链接队列为空,程序进入睡眠(如果监听套接字为默认阻塞方式)
         connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
        if (connfd<0){
            // 因为在子进程发送信号,在处理信号函数返回时可能会出现系统中断,所以在这检测重启
            if (errno==EINTR){  
                continue;
            } else{
                err_sys("serv: accept failed");
            }
        }
        if ((childpid=Fork()) == 0) { 
            Close(listenfd);
            // 如果是子进程,执行业务函数
            str_echo(connfd);
            // 关闭描述符,其实不关闭也可以,因为exit函数本身在内核中会将全部描述符关掉
            Close(connfd);

            // 关闭进程
            exit(0);
        }
        Close(connfd);
    }
}
```

在第 15 行，创建一个 TCP 套接字。在待捆绑到该 TCP 套接字的网际网套接字地址结构中填入通配地址（INADDR_ANY），捆绑通配地址是在告知系统:要是系统是多网卡的主机，我们将接受目的地址为任何本地接口的连接。

第 33 行，**判断是父进程还是子进程，如果是父进程，此处为子进程的 pid；如果是子进程，此处为 0**。fork 函数会返回两次，一次在父进程中，一次在子进程中。fork 有两种用法：一种是创建一个父进程的副本进程，进行某些操作；一种是在创建一个副本进程(子进程)后，在子进程中执行 exec 函数，这样这个子进程映像就会被替换为被 exec 的程序文件，而且新的程序通常从 main 函数执行

fork 为每个客户派生一个处理它们的子进程。并且子进程关闭监听套接字（进程之间的地址空间完全独立，子进程的 listenfd 和父进程不一样，并且对子进程没有意义），**同理父进程关闭已连接套接字，然后父进程又会阻塞在 accept 函数上，等待下一次连接**。子进程接着调用 str_echo 处理客户。

### 2.TCP 回射服务器程序：str_echo 函数

```c{.line-numbers}
#include "unp.h"

void str_echo(int sockfd) {
    ssize_t n;
    char buf[MAXLINE];

again:
    while((n = read(sockfd, buf, MAXLINE)) > 0) {
        Writen(sockfd, buf, n);
    }
    if (n < 0 && errno == EINTR)
        goto again;
    else if (n < 0)
        err_sys("str_echo: read error");
}
```

read 函数从套接字读入数据，writen 函数把其中内容回射给客户。**如果客户关闭连接（这是正常情况），那么接收到客户的 FIN 将导致服务器子进程的 read 函数返回 0**，这又导致 str_echo 函数的返回，从而终止子进程。

在第 11 行，当 n < 0 且 errno == EINTER 时，说明操作被中断，可以 goto again 继续读取。
**如果 sockfd 被设置为非阻塞，那么 read 函数的返回值小于 0 时，还有 EWOULDBLOCK 和 EAGAIN 这两个 error，表示现在 socket 的接收缓冲区中还没有数据，需要等待重试（EWOULDBLOCK = EAGAIN）**。 但是当 sockfd 默认为阻塞时，不会出现 EWOULDBLOCK 和 EAGAIN，因此 n < 0 说明可能出现了其它错误（比如 BrokenPipe error 或者 connection reset by peer）

### 3.TCP 回射客户程序：main 函数

```c{.line-numbers}
#include "unp. h"
int main(int argc, char **argv){
    int sockfd;
    struct sockaddr_in servaddr;
    if (argc != 2)
        err__quit ("usage: tcpcli <IPaddress>");
    
    sockfd =Socket (AF_INET, SOCK_STREAM, 0);
    bzero (&servaddr, sizeof (servaddr));
    servaddr. sin_family = AF_INET;
    servaddr. sin_port = htons (SERV_PORT);
    Inet_pton (AF_INET, argv[1], &servaddr. sin_addr);
    Connect (sockfd, (SA *) &servaddr, sizeof (servaddr));
    /* do it all*/
    str_cli (stdin, sockfd);
    exit (0);
}
```

connect 建立与服务器的连接。str_cli 函数完成剩余部分的客户处理工作。

### 4.TCP 回射客户程序：str_cli 函数

```c{.line-numbers}
#include "unp. h"

void str_cli (FILE *fp, int sockfd) {
    char sendline[MAXLINE], recvline[MAXLINE];
    while (Fgets(sendline, MAXLINE, fp)!= NULL) {
        Writen(sockfd, sendline, strlen(sendline));
        if (Readline(sockfd, recvline, MAXLINE) == 0) 
            err_quit("str_cli: server terminated prematurely");
        Fputs(recvline, stdout);
    }
}
```

对于第 5 行，当遇到文件结束符或错误时，fgets 将返回一个空指针，于是客户处理循环终止。然后阻塞在 Readline 函数中，等待 echo server 返回的数据，然后通过 fputs 显示在控制台上。如果与服务器的连接关闭了，那么服务器会发送 EOF 给客户端，Readline 会直接返回 0，打印出 `str_cli: server terminated prematurely`。

## 二、正常启动

服务器启动后，它调用 socket、bind、listen 和 accept，并阻塞于 accept 调用。（我们还没有启动客户）。在启动客户之前，我们运行 netstat 程序来检查服务器监听套接字的状态。

```shell
linux % netstat -a
Active Internet connections (servers and established) 
Proto Recv-Q    Send-Q     Local Address   Foreign Address     State 
tcp   0             0           *:9877           *:*           LISTEN
```

这个输出正是我们所期望的：有一个套接字处于 LISTEN 状态，它有通配的本地 IP 地址，本地端口为 9877。netstat 用星号 * 来表示一个为 0 的 IP 地址（INADDR_ANY，通配地址）或为 0 的端口号。

我们接着在同一个主机上启动客户，并指定服务器主机的 P 地址为 127.0.0.1（环回地址）：

```shell
linux % tcpcli01 127.0.0.1
```

客户调用 socket 和 connect，后者引起 TCP 的三路握手过程。当三路握手完成后，客户中的 connect 和服务器中的 accept 均返回，连接于是建立。接着发生的步骤如下：

1. 客户调用 str_cli 函数，该函数将阻塞于 fgets 调用，因为我们还未曾键入过一行文本。
2. 当服务器中的 accept 返回时，服务器调用 fork，再由子进程调用 str_echo。该函数调用 readline，readline 调用 read,而 read 在等待客户送入一行文本期间阻塞。
3. 另一方面，服务器父进程再次调用 accept 并阻塞，等待下一个客户连接。

至此，我们有 3 个都在睡眠（即已阻塞）的进程：客户进程、服务器父进程和服务器子进程。使用 netstat 给出的输出如下：

```shell
linux % netstat -a
Active Internet connections (servers and established) 
Proto Recv-Q  Send-Q    Local Address       Foreign Address     State 
tcp     0      0        localhost:9877      localhost:42758   ESTABLISHED 
tcp     0      0        localhost:42758     localhost:9877    ESTABLISHED 
tcp     0      0        *：9877                *:*               LISTEN
```

## 三、正常终止

至此连接已经建立，不论我们在客户的标准输入中键入什么，都会回射到它的标准输出中。

```shell
linux % tcpcli01 127.0.0.1    我们已经给出过本行 
hello world                   现在键入这一行 
hello, world                  这一行被回射回来 
good bye 
good bye
^D                           <CtrH+D> 是我们的终端 EOF 字符
```

我们键入两行，每行都得到回射，我们接着键入终端 EOF 字符 ( Control-D) 以终止客户。接下来我们可以总结出正常终止客户和服务器的步骤：

1. 当我们键入 EOF 字符时，fgets 返回一个空指针，于是 str_cli 函数返回。
2. 当 str_cli 返回到客户的 main 函数时，main 通过调用 exit 终止。
3. 进程终止处理的部分工作是关闭所有打开的描述符，因此客户打开的套接字由内核关闭。这导致客户 TCP 发送一个 FIN 给服务器，服务器 TCP 则以 ACK 响应，这就是 TCP 连接终止序列的前半部分。至此，服务器套接字处于 CLOSE_WAIT 状态，客户套接字则处于 FIN_WAIT2 状态。
4. 当服务器 TCP 接收 FIN 时，服务器子进程阻塞于 readline 调用，于是 readline 返回 0。这导致 str_echo 函数返回服务器子进程的 main 函数。
5. 服务器子进程通过调用 exit 来终止。
6. 服务器子进程中打开的所有描述符随之关闭。由子进程来关闭已连接套接字会引发 TCP 连接终止序列的最后两个分节：一个从服务器到客户的 FIN 和一个从客户到服务器的 ACK。至此，连接完全终止，客户套接字进入 TIME_WAIT 状态。
7. 进程终止处理的另一部分内容是：在服务器子进程终止时，给父进程发送一个 SIGCHLD 信号。这一点在本例中发生了，但是我们没有在代码中捕获该信号，而该信号的默认行为是被忽略。既然父进程未加处理，子进程于是进入僵死状态。

## 四、客户端与服务器异常情况

### 1.accept 函数

#### 1.1 accept 返回前连接中止

这里，三路握手完成从而连接建立之后，客户 TCP 却发送了一个 RST（复位）。**在服务器端看来，就在该连接已由 TCP 排队，等着服务器进程调用 accept 的时候 RST 到达。稍后，服务器进程调用 accept**。

模拟这种情形的一个简单方法就是：启动服务器，让它调用 socket、bind 和 listen，然后在调用 accept 之前睡眠一小段时间。在服务器进程睡眠时，启动客户，让它调用 socket 和 connect。**一旦 connect 返回，就设置 SO＿LINGER 套接字选项以产生这个 RST，然后终止**。

<div align="center">
    <img src="Socket_服务器编程难点/2.png" width="500"/>
</div>

但是，如何处理这种中止的连接依赖于不同的实现。**源自 Berkeley 的实现完全在内核中处理中止的连接，服务器进程根本看不到**。然而大多数 SVR4 实现返回一个错误给服务器进程，作为 accept 的返回结果，不过错误本身取决于实现。POSIX 指出返回的 **errno 值必须是 ECONNABORTED（"software caused connection abort"，软件引起的连接中止）**。服务器可以直接忽略 **`ECONNABORTED`** 错误，再次调用 accept 就行。

accept(2) man page 写道 `[ECONNABORTED] A connection arrived, but it was closed while waiting on the listen queue.`，即连接已到达，但在侦听队列中等待时已被关闭。

下面我们使用 Linux 来测试上述理论，使用的 Linux 内核版本是：**`Linux version 5.19.0-43-generic`**，客户端的代码如下所示，开启了 **`SO_LINGER`** 选项，这样调用 close 函数时，不会发送 FIN 报文，取而代之发送 RST 报文：

```c{.line-numbers}
int main() {

    int cfd;
    int counter = 10;
    char buf[BUFSIZ];
    unsigned int ip = 0;
    struct linger lgr;

    lgr.l_onoff = 1;
    lgr.l_linger = 0;

    // 服务器地址结构
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &ip);
    serv_addr.sin_addr.s_addr = ip;

    cfd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(cfd, SOL_SOCKET, SO_LINGER, &lgr, sizeof(lgr));
    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));

    printf("connect successfully, now existing....\n");
    close(cfd);

    while (counter--) {
        write(cfd, "hello\n", 6);
        int ret = read(cfd, buf, sizeof buf);
        write(STDOUT_FILENO, buf, ret);
    }

    return 0;
}
```

服务端的部分代码如下所示：

```c{.line-numbers}
    clit_addr_len = sizeof(clit_addr);
    // 模拟故障，阻塞 10s
    sleep(10);
    cfd = accept(lfd, (struct sockaddr *) &clit_addr, &clit_addr_len);

    if (cfd < 0) {
        if (errno == ECONNABORTED) {
            printf("accept: connect reset by peer\n");
        }
        return 1;
    }

    while (1) {
        int ret = read(cfd, buf, sizeof(buf));
        if (errno == ECONNRESET) {
            printf("read: connection reset by peer\n");
            return 1;
        }
        write(STDOUT_FILENO, buf, ret);
        for(int i = 0; i < ret; i++) {
            buf[i] = toupper(buf[i]);
        }
        write(cfd, buf, ret);
    }
```

最后服务端的运行结果如下所示，对于客户端连接的提前终止（发出 **`RST`** 报文），**`accept`** 函数没有抛出异常，而是正常取出连接，到了 **`read`** 函数才出现 **`ECONNABORTED`** 错误。

```shell
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/server
read: connection reset by peer

Process finished with exit code 1
```

使用 **`tcpdump`** 命令查看网络数据包的接受和发送情况如下所示：

```shell
xuweilin@xuweilin-virtual-machine:~/桌面$ sudo tcpdump -i 3 -nt
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
IP 127.0.0.1.48378 > 127.0.0.1.9523: Flags [S], seq 2316230539, win 65495, options [mss 65495,sackOK,TS val 3266194383 ecr 0,nop,wscale 7], length 0
IP 127.0.0.1.9523 > 127.0.0.1.48378: Flags [S.], seq 3778378464, ack 2316230540, win 65483, options [mss 65495,sackOK,TS val 3266194383 ecr 3266194383,nop,wscale 7], length 0
IP 127.0.0.1.48378 > 127.0.0.1.9523: Flags [.], ack 1, win 512, options [nop,nop,TS val 3266194383 ecr 3266194383], length 0
IP 127.0.0.1.48378 > 127.0.0.1.9523: Flags [R.], seq 1, ack 1, win 512, options [nop,nop,TS val 3266194383 ecr 3266194383], length 0
```

由此可见，对于 Linux 内核而言，如果客户端连接在 **`accept`** 函数之前终止，accept 函数不会感觉出异常，还是会从监听队列中取出连接，而不论连接处于何种状态，更不关心任何网络状况的变化。

#### 1.2 非阻塞式 accept

当有一个已完成的连接准备好被 accept 时，select 将作为可读描述符返回该连接的监听套接字。因此，如果我们使用 select 在某个监听套接字上等待一个外来连接，那就没有必要把该监听套接字设置为非阻塞，**这是因为如果 select 告诉我们该套接字上已有连接就绪，那么随后的 accept 调用不会阻塞**。

不幸的是，这里存在一个可能让我们掉入陷阱的定时问题。为了查看这个问题，我们首先把 TCP 回射客户程序改写成建立连接后发送一个 RST 到服务器。下面给出了这个新版本。

```c{.line-numbers}
#include "unp.h"

int main(int argc, char **argv) {
    int sockfd;
    struct linger ling;
    struct sockaddr_in servaddr;

    if (argc != 2) 
        err_quit("usage: tcpcli <IPaddress>");
    
    sockfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);

    Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));

    /* cause RST to be sent on close() */
    ling.l_onoff = 1;
    ling.l_linger = 0;
    Setsockopt(sockfd, SOL_SOCKET, SO_LINGER, &ling, sizeof(ling));
    Close(sockfd);

    exit(0);
}
```

一旦连接建立，我们设置 SO_LINGER 套接字选项，把 l_onoff 标志设置为 1，把 l_linger 时间设置为 0。正如前面所述，**这样的设置导致连接被关闭时在 TCP 套接字上发送一个 RST**。我们随后关闭该套接字。

我们接着修改 TCP 回射服务器程序，在 select 返回监听套接字的可读条件之后但在调用 accept 之前暂停。在下面这段代码中，以加号打头的那两行是新加的。

```c{.line-numbers}
    if (FD_ISSET(listenfd, &rset)) {
        /* new client connection */
+       printf("listening socket readable\n");
+       sleep(5);
        clilen = sizeof(cliaddr);
        connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
    }
```

这里我们是在模拟一个繁忙的服务器，它无法在 select 返回监听套接字的可读条件后就马上调用 accpet。通常情况下服务器的这种迟钝不成问题（实际上这就是要维护一个已完成连接队列的原因），但是结合上连接建立之后到达的来自客户的 RST，问题就出现了。

我们在上一节指出，当客户在服务器调用 accept 之前中止某个连接时，**源自 Berkeley 的实现不把这个中止的连接返回给服务器（内核直接处理掉）**，而其他实现有些会返回 ECONNABORTED 错误，有些却返回 EPROTO 错误。考虑一个源自 Berkeley 的实现上的如下例子。

- 客户建立一个连接并随后中止它；
- select 向服务器进程返回可读条件，不过服务器要过一小段时间才调用 accept；
- 在服务器从 select 返回到调用 accept 期间，服务器 TCP 收到来自客户的 RST；
- **这个已完成的连接被服务器 TCP 驱除出队列**，我们假设队列中没有其他已完成的连接；
- 服务器调用 accept，但是由于没有任何已完成的连接，服务器于是阻塞。

服务器会一直阻塞在 accept 调用上，直到其他某个客户建立一个连接为止。但是在此期间，就以上面给出的服务器程序为例，服务器单纯阻塞在 accept 调用上，无法处理任何其他已就绪的描述符。本问题和拒绝服务攻击（DoS）多少有些类似，不过对于这个新的缺陷，一旦另有客户建立一个连接，服务器就会脱出阻塞中的 accept。

本问题的解决办法如下。

- 当使用 select 获悉某个监听套接字上何时有已完成连接准备好被 accept 时，**总是把这个监听套接字设置为非阻塞**。
- 在后续的 accept 调用中忽略以下错误：EWOULDBLOCK（源自 Berkeley 的实现，客户中止连接时）、ECONNABORTED（POSIX 实现，客户中止连接时）、EPROTO（SVR4 实现，客户中止连接时）和 EINTR（如果有信号被捕获）。

### 2.服务器进程终止/崩溃

现在启动我们的客户/服务器对，然后杀死服务器子进程。这是在模拟服务器进程崩溃的情形，我们可从中查看客户将发生什么。**(我们必须小心区别即将讨论的服务器进程崩溃与将在之后讨论的服务器主机崩溃)** 所发生的步骤如下所述：

1. 我们在同一个主机上启动服务器和客户，并在客户上键入一行文本，以验证一切正常。正常情况下该行文本由服务器子进程回射给客户。
2. 找到服务器子进程的进程 ID，并执行 kill 命令杀死它。作为进程终止处理的部分工作，服务器子进程中所有打开着的描述符都被关闭。这就导致向客户发送一个 FIN，而客户 TCP 则响应以一个 ACK。这就是 TCP 连接终止工作的前半部分。
3. SIGCHLD 信号被发送给服务器父进程，并得到正确处理。
4. 客户上没有发生任何特殊之事。客户 TCP 接收来自服务器 TCP 的 FIN 并响应以一个 ACK，然而问题是客户进程阻塞在 fgets 调用上，等待从终端接收一行文本，无法对 FIN 进行响应，及时关闭掉连接。
5. 此时，在另外一个窗口上运行 netstat 命令，以观察套接字的状态。

```shell
linux % netstat -a | grep 9877
tcp     0   0   *：9877          *:*                LISTEN
tcp     0   0   localhost:9877  localhost:43604     FIN_WAIT2
tcp     1   0   localhost:43604 localhost:9877      CLOSE_WAIT
```

6. 我们可以在客户上再键入一行文本。以下是从第一步开始发生在客户之事：

```shell
linux % tcpcli01 127.0.0.1              启动客户
hello                                   键入第一行文本
hello                                   它被正确回射
                                        在这儿杀死服务器子进程
another line                            然后键入下一行文本
str_cli: server terminated prematurely
```

当我们键入 "another line" 时，str_cli 调用 writen，客户 TCP 接着把数据发送给服务器。TCP 允许这么做，因为客户 TCP 接收到 FIN 只是表示服务器进程已关闭了连接的服务器端，从而不再往其中发送任何数据而己。FIN 的接收并没有告知客户 TCP 服务器进程已经终止（虽然在本例子中它确实是终止了）。当服务器 TCP 接收到来自客户的数据时，既然先前打开那个套接字的进程已经终止，于是响应以一个 RST。

7. 然而客户进程看不到这个 RST，因为它在调用 writen 后立即调用 read1line，并且由于第 2 步中接收的 FIN，所调用的 readline 立即返回 0（表示 EOF）。我们的客户端此时收到 EOF，于是以出错信息 "server terminated prematurely"（服务器过早终止）退出。
8. 当客户终止时，它所有打开着的描述符都被关闭

我们的上述讨论还取决于本例子的时序，客户调用 readline 既可能发生在服务器的 RST 被客户收到之前，也可能发生在收到之后。**如果 readline 发生在收到 RST 之前（如本例子所示），那么结果是客户得到一个未预期的 EOF；否则结果是由 readline：返回一个 ECONNRESET ("connection reset by peer"，对方复位连接错误)**，表示服务端的进程被终止。

本例子的问题在于：**当 FIN 到达套接字时，客户正阻塞在 fgets 调用上，无法及时对 FIN 报文进行响应，关闭掉客户端连接，导致后续又向服务端发送报文，导致服务器发送 RST 重置为报文**。

客户实际上在应对两个描述符-套接字和用户输入，它不能单纯阻塞在这两个源中某个特定源的输入上（正如目前编写的 str_cli 函数所为），而是应该阻塞在其中任何一个源的输入上。事实上这正是 select 和 po11 这两个函数的目的之一，我们可以使用 I/O 多路复用函数重新编写 str_cli 函数之后，一旦杀死服务器子进程，客户就会立即被告知已收到 FIN。

### 3.SIGPIPE 信号

要是客户不理会 readline 函数返回的错误，反而写入更多的数据到服务器上，那又会发生什么呢？这种情况是可能发生的，举例来说，客户可能在读回任何数据之前执行两次针对服务器的写操作，而 RST 是由其中第一次写操作引发的。

适用于此的规则是：**当一个进程向某个已收到 RST 的套接字执行写操作时，内核向该进程发送一个 SIGPIPE 信号。该信号的默认行为是终止进程，因此进程必须捕获它以免不情愿地被终止**。

不论该进程是捕获了该信号并从其信号处理函数返回，还是简单地忽略该信号，**写操作都将返回 EPIPE 错误**。一个在 Usenet 上经常问及的问题 ( frequently asked question, FAQ) 是如何在第一次写操作时而不是在第二次写操作时捕获该信号。这是不可能的。遵照上述讨论，第一次写操作引发 RST，第二次写引发 SIGPIPE 信号。**写一个已接收了 FIN 的套接字不成问题，但是写一个已接收了 RST 的套接宇则是一个错误**。

接下来介绍一下如何处理 SIGPIPE 信号，处理 SIGPIPE 的建议方法取决于它发生时应用进程想做什么。如果没有特殊的事情要做，那么将信号处理办法直接设置为 SIG_IGN，并假设后续的输出操作将捕捉 EPIPE 错误并终止。如下所示：

```c
int main()
{
    signal(SIGPIPE, SIG_IGN);  // 忽略 SIGPIPE 信号
    // ...
}
```

如果信号出现时需采取特殊措施（可能需在日志文件中登记），那么就必须捕获该信号，以便在信号处理函数中执行所有期望的动作。**但是必须意识到，如果使用了多个套接字，该信号的递交无法告诉我们是哪个套接字出的错**。

### 4.服务器主机崩溃

我们来看看**当服务器主机崩溃**或者**当客户端发送数据时服务器主机不可达的情形（即建立连接后某些中间路由器不工作）**。为了模拟这种情况，我们可以先启动服务器，再启动客户，接着在客户上键入一行文本来确认连接正常工作，然后从网络上断开服务器主机，这也模拟了我们所说的第二种情况，即主机因为网络不可达。**在这两种情况下，服务器不会对客户端的请求作出任何响应（不会发送 FIN 报文，ACK 以及其它响应），可以归为一类**。

步骤如下：

- 当服务器主机崩溃时，来不及在连接上发出 FIN 报文。这里我们假设的是主机崩溃，而不是由操作员执行命令关机
- 我们在客户上键入一行文本，它由 writen 写入内核，再由客户 TCP 作为一个数据分节送出。客户随后阻塞于 readline 调用，等待回射的应答。
- 如果我们用 tcpdump 观察网络就会发现，客户 TCP 持续重传数据分节，试图从服务器上接收一个 ACK。源自 Berkeley 的实现重传该数据分节 12 次，共等待约 9 分钟才放弃重传。当客户 TCP 最后终于放弃时 (假设在这段时间内，服务器主机没有重新启动，或者如果是服务器主机未崩溃但是从网络上不可达，那么假设主机仍然不可达) ，给客户进程返回一个错误。
  - **假设服务器主机已崩溃，从而对客户的数据根本没有响应，那么所返回的错误 ETIMEDOUT**。
  - **假设某个中间路由器判定服务器主机已不可达，从而响应以一个 "destination unreachable"(目的地不可达) ICMP 消息， 那么所返回的错误是 EHOSTUNREACH 或 ENETUNREACH**。

### 5.服务器主机崩溃后重启

前一节中，当我们发送数据时，服务器主机仍然处于崩溃状态，在这一节中，在客户端发送数据前，虽然服务器之前已经崩溃了，但是现在已经重启。我们模拟这种情况最简单的办法就是：**客户端与服务器端先建立连接，再从网络上断开服务器主机，将它关机后再重新启动，最后把它重新连接到网络中**。

在模拟时，先断开网络，再对服务器关机，服务器不会给客户端发送一个 FIN 报文，这样客户端就不会知道服务器已经被关闭了。所发生的步骤如下所示：

1. 我们启动服务器和客户，并在客户键入一行文本以确认连接已经建立
2. 服务器之前主机崩溃，但是现在被重启
3. 在客户上键入一行文本，它将作为一个 TCP 数据分节发送到服务器主机
4. 当服务器主机崩溃后重启时，**它的 TCP 丢失了崩溃前的所有连接信息，因此服务器 TCP 对于所收到的来自客户的数据分节响应以一个 RST**
5. 当客户 TCP 收到该 RST 时，客户正阻塞于 readline 调用，导致该调用返回 ECONNRESET 错误

### 6.服务器主机关机

Unix 系统关机时，init 进程通常先给所有进程发送 SIGTERM 信号（该信号可被捕获），等待一段固定的时间（往往在 5 到 20 秒之间），然后给所有仍在运行的进程发送 SIGKILL 信号（该信号不能被捕获）。这么做留给所有运行的进程一小段时间来清除和终止。

如果我们不捕获 SIGTERM 信号并终止，我们的服务器将由 SIGKILL 信号终止。当服务器子进程终止时，它的所有打开着的描述符都被关闭，随后发生的步骤与**服务器进程终止/崩溃**中讨论过的一样。正如那一节所述，**我们必须在客户中使用 select 或 poll 函数，使得服务器进程的终止一经发生，客户就能检测到**。

### 7.总结

- 服务器 accept 前，客户端与服务器端的连接被关闭，这时服务器端会抛出 __`ECONNABORTED`__，此 errno 可以直接被忽略，服务器端再次调用 accept 函数即可
- 服务器端进程终止/崩溃，或者服务器被关机时，服务器中对应被关闭的套接字会发送 FIN 报文给客户端，客户端接收到 FIN 报文后（read 函数返回 0），可以继续向服务端发送数据，第一次发送，服务端会响应 RST 报文，客户端可能会产生 __`ECONNRESET`__ 错误；第二次发送，客户端内核可能直接产生 __`SIGPIPE`__ 错误。
- 服务器崩溃、崩溃后重启或者网络不可达时，对应套接字不会发送 FIN 报文（或者发送但是客户端接收不到），客户端不知道服务器已经崩溃，客户端向服务器发送消息然后调用阻塞在 read 函数上读取响应，这时会有以下 3 种情况：
    - 如果是因为服务器已经崩溃，对客户端没有任何响应，则产生 __`ETIMEDOUT`__ 错误；
    - 如果因为网络不可达，中间路由器向客户端产生目标不可达的 ICMP 报文，那么产生 __`ENETUNREACH`__ 或者 __`EHOSTUNREACH`__ 错误；
    - 如果服务器崩溃后重启之后，客户端向其发送数据，服务器会返回 RST 响应给客户端，产生 __`ECONNRESET`__ 错误；

## 五、实例分析

### 1.SIGPIPE 信号讲解

#### 1.1 进程间通信

在计算机科学中，进程间通信 (IPC) 特指操作系统提供的允许进程共享数据的机制。进程间通信有多种方式，比如文件、信号、管道、共享内存和 Socket 等。这篇文章涉及到的进程间通信方式主要有信号，管道和 Socket 三种。

#### 1.2 信号（Signal）

信号就是向一个进程或同一进程中的特定线程发送的异步通知，以通知它一个事件。大多数信号可以被忽略、阻止或处理（通过指定的代码），SIGSTOP（暂停）和 SIGKILL（立即终止）是两个例外。__信号常量是有整数值的，比如 SIGKILL 的整数值为 9__。下面介绍常用的信号：

- SIGINT：__当用户按下了<Ctrl+C>组合键时__，用户终端向正在运行中的由该终端启动的程序发出此信号。默认动作为终止进程。
- SIGQUIT：__当用户按下<ctrl+\\>组合键时产生该信号__，用户终端向正在运行中的由该终端启动的程序发出些信号。默认动作为终止进程。
- SIGKILL：无条件终止进程。__本信号不能被忽略，处理和阻塞__。默认动作为终止进程。它向系统管理员提供了可以杀死任何进程的方法。
- SIGTERM：程序结束信号，与 SIGKILL 不同的是，__该信号可以被阻塞和终止__。通常用来要示程序正常退出。执行 shell 命令 Kill 时，缺省产生这个信号。默认动作为终止进程。
- SIGSTOP：停止进程的执行。__信号不能被忽略，处理和阻塞__。默认动作为暂停进程。

再比如我们经常用 kill 系统函数来向指定进程发送信号：

```c
int kill(pid_t pid, int signum); /* declaration */
```

kill 命令的第二个参数就是一个标准的信号，如 SIGTERM 或 SIGKILL。当然也可以指定其他信号，比如 SIGUSR1 和 SIGUSR2，**这两个信号被用来告诉进程有一个用户定义的事件发生了，具体做什么取决于进程如何处理发送过来的信号，不一定是杀掉进程**，比如在 MongoDB 中，运行下边的命令就是告诉 MongoDB 该进行 log rotation 了。

```shell
kill -SIGUSR1 MONGODPID
```

当信号发出时，操作系统会中断目标进程的正常执行流程进而完成信号传递。进程可以在任何非原子指令期间被中断，如果进程之前已经注册了一个对这个信号的处理程序，则执行该程序，否则就执行默认的信号处理程序。

信号处理程序可以通过 signal(2) 或 sigaction(2) 这两个系统调用来设置，现在 sigaction()函数取代了 signal() 函数，应优先使用。在设置信号处理程序的时候还可以用这两个特殊的值：

- __SIG_IGN: 忽略信号（ignore）__
- __SIG_DFL: 和使用默认信号处理程序（default）__

#### 1.3 管道（Pipe）

管道是基于消息传递实现的，将一组进程的标准流链在一起，这样每个进程的输出（stdout）直接作为输入（stdin）传递给下一个进程。它们是并发执行的，也就是说后边的进程可以在前一个进程running 的时候被启动，直观来说：

```shell
process1 | process2 | process3
```

我们经常用到管道，比如要列出当前目录下的文件(ls)，只保留 ls 输出中包含字符串 "key" 的行(grep)，并在滚动页面中查看结果 (less)：

<div align="center">
    <img src="Socket_服务器编程难点/3.png" width="300"/>
</div>

```shell
ls -l | grep key | less
```

#### 1.4 SIGPIPE 信号

说了这么多前置知识，我们现在来看看本文的主角—— SIGPIPE 信号：

> The SIGPIPE signal is sent to a process when it attempts to write to a pipe without a process connected to the other end.

当一个进程试图向一个管道写入时，如果没有一个进程连接到另一端，则会向该尝试写入的进程发送 SIGPIPE 信号。__也就是说当一个进程试图向一个读端已经关闭的管道写入时，就会收到这个信号__，而收到这个信号的默认操作是终止进程。

假设有一个管道：

```shell
process1 | process2
```

如果 process2 已经死掉了，理应通知 process1 一下，不能让他在那一直做无用功，至于收到信号process1 怎么处理就是它自己的事了。举个例子：

```shell
yes | head -n 1
```

yes 命令的作用是将无限的 "y" 序列写入 STDOUT，而 head 则将其从管道的另一端读为 STDIN。head 读取第一行，然后退出，只要 head 终止，管道的接收端就关闭了。因此，Linux 内核会向 yes 进程发送 SIGPIPE 信号，表示没有 reader 了，然后 yes 进程就会终止。

不只是对于管道，Socket 也有这个机制：

> When a process writes to a socket that has received an RST, the SIGPIPE signal is sent to the process. The default action of this signal is to terminate the process, so the process must catch the signal to avoid being involuntarily terminated.

如果向一个收到 RST 的 Socket 继续发送数据，则会收到 SIGPIPE 信号。

先来回忆一下 RST 数据包，当一个意外的 TCP 数据包到达主机时，该主机通常会通过在同一连接上发送一个 RST 数据包来进行响应。RST 数据包就是一个没有有效载荷，并且在 TCP 头标志中设置了RST位的数据包。常见的意外的 TCP 数据包有：

- __SYN 数据包，试图建立连接到一个没有进程监听的服务器端口__
- 数据包到达之前建立的 TCP 连接上，但本地应用程序已经关闭了它的套接字或退出，操作系统关闭了套接字

### 2.管道的 SIGPIPE

#### 2.1 管道的读端全部关闭

如果所有指向管道读端的文件描述符都关闭了（管道读端引用计数为 0），这时有进程向管道的写端 write，那么该进程会收到信号 SIGPIPE，通常会导致进程异常终止。当然也可以对 SIGPIPE 信号实施捕捉，不终止进程。具体代码如下所示：

```c{.line-numbers}
int main() {
    int ret;
    int fd[2];
    pid_t pid;

    char *str = "hello pipe\n";
    char buf[1024];

    ret = pipe(fd);
    if (ret == -1) {
        perror("pipe error");
        exit(1);
    }

    pid = fork();
    if (pid > 0) {
        // 关闭父进程管道的读端 fd[0]
        close(fd[0]);
        sleep(3);
        write(fd[1], str, strlen(str));
        if (errno == EPIPE) {
            printf("epipe\n");
        }
        close(fd[1]);
    } else if (pid == 0){
        // 关闭子进程管道的读端 fd[0]
        close(fd[0]);
    }

    return 0;
}
```

当父进程 fork 出子进程时，父子进程与管道读写端的描述符如下所示：

<div align="center">
    <img src="Socket_服务器编程难点/4.png" width="500"/>
</div>

当使用 close 关闭管道的读端时，只有当管道的读端引用计数为 0 时，才会真正关闭读端的描述符，因此需要同时关闭父子进程的读端描述符。这时，父进程继续往管道中写时，会产生 SIGPIPE 信号，如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/linux_programming

Process finished with exit code 141 (interrupted by signal 13: SIGPIPE)
```

#### 2.2 管道的写端全部关闭

```c{.line-numbers}
int main() {
    int ret;
    int fd[2];
    pid_t pid;

    char *str = "hello pipe\n";
    char buf[1024];

    ret = pipe(fd);
    if (ret == -1) {
        perror("pipe error");
        exit(1);
    }

    pid = fork();
    if (pid > 0) {
        // 关闭管道的写端 fd[1]
        close(fd[1]);
        sleep(3);
        write(fd[1], str, strlen(str));
        close(fd[0]);
    } else if (pid == 0){
        // 关闭管道的写端 fd[1]
        close(fd[1]);
        ret = read(fd[0], buf, sizeof(buf));
        write(STDOUT_FILENO, buf, ret);
        close(fd[0]);
    }

    return 0;
}
```

当我们把父子进程管道的写端描述符都关闭时，父进程向写端发送数据时，不会报错，直接向后执行。而当子进程执行 read 操作时，由于写端的描述符均被关闭，这时 read 直接返回 0（类似于读到文件结尾），ret = 0，因此控制台上不会显示字符串。如下所示：

```shell
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/linux_programming

Process finished with exit code 0
```

总结管道的读写会发生的情况：

读管道：
1. 管道中有数据，read 返回实际读到的字节数。
2. 管道中无数据：
(1) **管道写端被全部关闭，read 返回 0 (好像读到文件结尾)**
(2) 写端没有全部被关闭，read 阻塞等待 (不久的将来可能有数据递达，此时会让出 CPU)

写管道：
1. **管道读端全部被关闭， 进程异常终止 (也可使用捕捉 SIGPIPE 信号，使进程不终止)**
2. 管道读端没有全部关闭：
(1) 管道已满，write 阻塞。
(2) 管道未满，write 将数据写入，并返回实际写入的字节数。

### 3.Socket 的 SIGPIPE

#### 3.1 Socket 中两次 write 产生 SIGPIPE 信号

我们使用以下的 client.c 和 server.c 代码来模拟出现 SIGPIPE 的情况。服务器端的代码如下所示，服务端通过 read(fd) 读取 socket 获取客户端数据，然后将小写转换为大写 toupper()，然后发送给客户端。

```c{.line-numbers}
// server.c
int main() {

    int lfd = 0, cfd = 0;

    struct sockaddr_in serv_addr, clit_addr;
    socklen_t clit_addr_len;
    char buf[BUFSIZ], client_ip[1024];
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    lfd = socket(AF_INET, SOCK_STREAM, 0);
    if (lfd == -1) {
        sys_err("socket error");
    }

    bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr));
    listen(lfd, 256);
    clit_addr_len = sizeof(clit_addr);
    cfd = accept(lfd, (struct sockaddr *) &clit_addr, &clit_addr_len);

    printf("client ip:%s, port:%d\n",
           inet_ntop(AF_INET, &clit_addr.sin_addr.s_addr, client_ip, sizeof(clit_addr)),
           ntohs(clit_addr.sin_port));

    if (cfd == -1) {
        sys_err("accept error");
    }

    while (1) {
        int ret = read(cfd, buf, sizeof(buf));
        if (ret < 0) {
            if (errno == ECONNRESET ){
                write(STDOUT_FILENO, 'reset\n', 6);
            }
        }
        write(STDOUT_FILENO, buf, ret);

        for(int i = 0; i < ret; i++) {
            buf[i] = toupper(buf[i]);
        }

        write(cfd, buf, ret);
        // 服务器睡眠 5s，等待客户端进程执行完毕后退出，客户端进程退出时，会发送 FIN 报文
        sleep(5);
    }

    close(lfd);
    close(cfd);

    return 0;

}
```

客户端的代码如下所示，向服务端发送 10 次 'hello' 字符串，然后进程退出，在代码中没有显式关闭 cfd 套接字描述符，但是客户端进程退出后，Init 进程会自动关闭客户端进程所属的所有描述符，描述符被关闭时，会发出 FIN 报文。

```c{.line-numbers}
// client.c
int main() {

    int cfd;
    int counter = 10;
    char buf[BUFSIZ];
    unsigned int ip = 0;

    // 服务器地址结构
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &ip);
    serv_addr.sin_addr.s_addr = ip;

    cfd = socket(AF_INET, SOCK_STREAM, 0);

    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));

    while (counter--) {
        int ret = write(cfd, "hello\n", 6);
        printf("%d\n", counter);
    }

    return 0;
}
```

最后服务器端 server.c 运行的结果为：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/server
client ip:127.0.0.1, port:40398
hello
hello

Process finished with exit code 141 (interrupted by signal 13: SIGPIPE)
```

client 向 server 发送 10 次 hello 字符串，然后进程退出，cfd 套接字描述符被关闭，并且向服务器端发送 FIN 报文。server 首先读取第一个 hello 字符串，然后转换成大写，再调用 write 写会给 client，而 client 此时进程已经退出，会返回一个 RST 报文给 server。但是此时 server 阻塞在 sleep 函数上。随后 server 读取第二个 hello 字符串，然后再次写回给 client，这时出现 SIGPIPE 错误。

#### 3.2 Socket 的 SHUT_WR 产生 SIGPIPE

我们对上述 client.c 中的代码进行了一些修改，在客户端发送数据之前，调用 shutdown 关闭了连接的写部分，如下所示：

```c{.line-numbers}
// client.c

int main() {

    int cfd;
    int counter = 10;
    char buf[BUFSIZ];
    unsigned int ip = 0;

    // 服务器地址结构
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &ip);
    serv_addr.sin_addr.s_addr = ip;

    cfd = socket(AF_INET, SOCK_STREAM, 0);

    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));

    while (counter--) {
        // 在发送数据之前，关闭此 socket 的写部分
        shutdown(cfd, SHUT_WR);
        int ret = write(cfd, "hello\n", 6);
        printf("%d\n", counter);
        sleep(60);
    }

    return 0;
}
```

最后 client.c 运行产生的结果如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/my_client
Signal: SIGPIPE (Broken pipe)

Process finished with exit code 1
```

所以，对 A -> B 之间的 socket 使用 shutdown 函数关闭 A 的写部分，A 再往 socket 中写入数据时，会产生 SIGPIPE 错误。

#### 3.3 Socket 的 SHUT_RD 不产生 SIGPIPE

接下来我们测试一下关闭 B 端的读部分，A 继续往 socket 写入数据的情况。客户端的代码如下：

```c{.line-numbers}
    // 前面代码与 3.1 的 client 类似
    // client.c 专门发送数据

    cfd = socket(AF_INET, SOCK_STREAM, 0);

    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));

    while (counter--) {
        int ret = write(cfd, "hello\n", 6);
        sleep(10);
    }

    return 0;
```

服务端的代码如下所示：

```c{.line-numbers}
    // 前面代码与 3.1 的 server 类似
    // server.c 专门接收数据

    while (1) {
        shutdown(cfd, SHUT_RD);
        int ret = read(cfd, buf, sizeof(buf));
        if (ret < 0) {
            if (errno == ECONNRESET ){
                write(STDOUT_FILENO, 'reset\n', 6);
            }
        }
        write(STDOUT_FILENO, buf, ret);

        for(int i = 0; i < ret; i++) {
            buf[i] = toupper(buf[i]);
        }

        sleep(5);
    }
```

server 端关闭 socket 的读部分，client 继续往 socket 中写入数据，最后 client 运行得到的结果如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/my_client

```

server 运行得到的结果（只显示了一部分）如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/server
client ip:127.0.0.1, port:49994
hello
hello
hello
hello
hello

```

可以看出，对 A -> B 之间的 socket 使用 shutdown 函数（参数为 SHUT_RD）关闭 B 的读部分，B 可以继续从 socket 中读取数据，__因此只 SHUT_RD 对 socket 套接字没有影响__。

#### 3.5 Socket 的 SHUT_RDWR 和 SHUT_RD/SHUT_WR 会产生 SIGPIPE

当 A -> B 发送数据时，虽然只关闭 B 的读部分（SHUT_RD）对 B 接收数据以及 A 发送数据没有影响，但是当我们在 B 端同时关闭读/写部分（SHUT_RDWR）或者先关闭读部分再关闭写部分（SHUT_RD/SHUT_WR）时，情况就变得不一样了。

client 端的代码如下所示：

```c{.line-numbers}
    // 前面代码和 3.1 类似
    // client.c client 专门发送数据

    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));

    while (counter--) {
        int ret = write(cfd, "hello\n", 6);
        sleep(10);
    }
```

server 端的代码如下所示：

```c{.line-numbers}
    // 前面代码和 3.1 类似
    // server.c server 专门接收数据

    cfd = accept(lfd, (struct sockaddr *) &clit_addr, &clit_addr_len);

    printf("client ip:%s, port:%d\n",
           inet_ntop(AF_INET, &clit_addr.sin_addr.s_addr, client_ip, sizeof(clit_addr)),
           ntohs(clit_addr.sin_port));

    if (cfd == -1) {
        sys_err("accept error");
    }

    while (1) {
        shutdown(cfd, SHUT_RD);
        shutdown(cfd, SHUT_WR);
        int ret = read(cfd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, ret);

        for(int i = 0; i < ret; i++) {
            buf[i] = toupper(buf[i]);
        }

        sleep(5);
    }
```

最后 client 端运行的结果如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/my_client
Signal: SIGPIPE (Broken pipe)
Signal: SIGPIPE (Broken pipe)
Signal: SIGPIPE (Broken pipe)
Signal: SIGPIPE (Broken pipe)

```

server 端的运行结果如下所示：

```c
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/server
client ip:127.0.0.1, port:39386
hello

```

当 server 既关闭了读部分，又关闭了写部分，会发送一个 **`FIN`** 报文给 client，而 client 收到 **`FIN`** 之后只能表明 server 关闭了发送通道，至于是使用 close(fd) 还是 **`shutdown(fd, SHUT_WR)`** 或者 **`shutdown(fd, SHUT_RDWR)`** 对端是不知道的。client 也并不知道 server 的读通道是否关闭，因此可以向 server 继续发送数据。因此 server 端会显示 hello 字符串（有可能会显示多个，取决于 client 在 server 调用 shutdown 之前发送了多少个 hello 字符串），并返回 client RST 报文。

当 client 继续向 server 发送数据时，内核会产生 **`SIGPIPE`** 错误。此实验结果对于 **`SHUT_RDWR`** 也同样适用。

## 六、主机字节序和网络字节序

现代 CPU 的累加器一次都能装载（至少）4 字节（这里考虑 32 位机，下同），即一个整数。那么这 4 字节在内存中排列的顺序将影响它被累加器装载成的整数的值。这就是字节序问题。字节序分为大端字节序（big endian）和小端字节序（little endian）。大端字节序是指一个整数的高位字节存储在内存的低地址处，低位字节存储在内存的高地址处。**小端字节序则是指整数的高位字节存储在内存的高地址处，而低位字节则存储在内存的低地址处**。我们可以使用如下代码进行判断：

```c{.line-numbers}
void byteorder() {
    union {
        short value;
        char union_bytes[sizeof(short )];
    } test;

    test.value = 0x0102;

    if ((test.union_bytes[0] == 2) && (test.union_bytes[1] == 1)) {
        printf("little endian\n");
    } else if ((test.union_bytes[0] == 1) && (test.union_bytes[1] == 2)) {
        printf("big endian\n");
    } else {
        printf("unknown...\n");
    }
}
```

**现代 PC 大多采用小端字节序，因此小端字节序又被称为主机字节序**。当格式化的数据（比如 32bit 整型数和 16bit 短整型数）在两台使用不同字节序的主机之间直接传递时，接收端必然错误地解释之。解决问题的方法是：**发送端总是把要发送的数据转化成大端字节序数据后再发送，而接收端知道对方传送过来的数据总是采用大端字节序**，所以接收端可以根据自身采用的字节序决定是否对接收到的数据进行转换（小端机转换，大端机不转换）。因此大端字节序也称为网络字节序，它给所有接收数据的主机提供了一个正确解释收到的格式化数据的保证。

## 七、socket 函数

下面的 socket 系统调用可以创建一个 socket：

```c{.line-numbers}
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

值得指出的是，自 Linux 内核版本 2.6.17 起，type 参数可以接受上述服务类型与下面两个重要的标志相与的值：**`SOCK_NONBLOCK`** 和 **`SOCK_CLOEXEC`**。它们分别表示将新创建的 socket 设为非阻塞的，以及用 fork 调用创建子进程时在子进程中关闭该 socket。接下来详细解释 **`SOCK_CLOEXEC`** 选项的作用。

在实际生产中遇到一个问题，有一个进程 A，它是一个全局监控进程，监控进程 B。进程 B 是一个局部监控进程，监控 C，C 是由 B fork 出来的子进程。C 向 B 汇报，B 向 A 汇报。

因为进程 A 和其他进程在不同机器上，所以所有的操作都是通过 json rpc 的远程调用执行的。假设 B 监听 11111 端口，A 通过这个端口与其通信。现在我手动 kill B，理论上的现象应该11111 端口此时无人监听，A 发 rpc call 的时候会报一个异常，rpc 连接会断掉。实际的情况是 A 会在 rpc call 上阻塞，观察 11111 端口的情况没，发现被 C 监听了。

分析的结果如下：

**`Linux`** 下 **`socket`** 也是文件描述符的一种，**当 B fork 进程 C 的时候，C 也会继承 B 的 11111 端口 socket 文件描述符，当 B 挂了的时候，C 就会占领监听权**。子进程以写时复制（COW，Copy-On-Write）方式获得父进程的数据空间、堆和栈副本，这其中也包括文件描述符。刚刚 fork 成功时，**父子进程中相同的文件描述符指向系统文件表中的同一项**（这也意味着他们共享同一文件偏移量）。

接着，一般我们会调用 exec 执行另一个程序，此时会用全新的程序替换子进程的正文，数据，堆和栈等。此时保存文件描述符的变量当然也不存在了，我们就无法关闭无用的文件描述符了。所以通常我们会 fork 子进程后在子进程中直接执行 close 关掉无用的文件描述符，然后再执行 exec。

但是在复杂系统中，有时我们 fork 子进程时已经不知道打开了多少个文件描述符（包括 socket 句柄等），这此时进行逐一清理确实有很大难度。我们期望的是能在 fork 子进程前打开某个文件句柄时就指定好："这个句柄我在 fork 子进程后执行 exec 时就关闭"。其实时有这样的方法的：即所谓的 close-on-exec。

**当父进程打开文件时，只需要应用程序设置 `FD_CLOSEXEC` 标志位，则当 fork 后 exec 其他程序的时候，内核自动会将其继承的父进程 FD 关闭**。

close_on_exec 另外的一大意义就是安全。比如父进程打开了某些文件，父进程 fork 了子进程，但是子进程就会默认有这些文件的读取权限，但是很多时候我们并不想让子进程有这么多的权限。试想一下这样的场景：在 Webserver 中，首先会使用 root 权限启动，以此打开 root 权限才能打开的端口、日志等文件。然后降权到普通用户，fork 出一些 worker 进程，这些进程中再进行解析脚本、写日志、输出结果等进一步操作。

然而这里，就会发现隐含一个安全问题：子进程中既然继承了父进程的 FD，那么子进程中运行的脚本只需要继续操作这些 FD，就能够使用普通权限“越权”操作 root 用户才能操作的文件。系统提供的 close_on_exec 就可以有效解决这个问题。

## 八、监听 socket

socket 被命名之后，还不能马上接受客户端连接，**我们需要使用如下系统调用来创建一个监听队列以存放待处理的客户连接**。

```c{.line-numbers}
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

sockfd 参数指定被监听的 socket。**`backlog`** 参数提示内核监听队列的最大长度。**监听队列的长度如果超过 backlog，服务器将不受理新的客户连接，客户端也将收到 `ECONNREFUSED` 错误信息**。在内核版本 2.2 之前的 Linux 中，backlog 参数是指所有处于半连接状态（**`SYN_RCVD`**）和完全连接状态（**`ESTABLISHED`**）的 socket 的上限。但自内核版本 2.2 之后，它只表示处于完全连接状态的 socket 的上限。**`backlog`** 参数的典型值是 5。

我们使用如下代码进行验证：

```c{.line-numbers}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/wait.h>
#include <stdbool.h>

#define SERV_PORT 9523
static bool stop = false;

static void handle_term(int sig) {
    stop = true;
}

int main() {

    signal(SIGTERM, handle_term);

    int lfd = 0;
    struct sockaddr_in serv_addr;

    lfd = socket(PF_INET, SOCK_STREAM, 0);
    if (lfd == -1) {
        sys_err("socket error");
    }

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr));
    listen(lfd, 5);

    while (!stop) {
        sleep(1);
    }

    close(lfd);
    return 0;
}
```

运行上述 server 程序之后，在 Linux 上也同时运行多个客户端使用 **`telnet`** 连接 server 程序，最后使用 **`netstat`** 命令查看网络的连接状态：

```shell
# 重复多次 telnet 命令
xuweilin@xuweilin-virtual-machine:~$ telnet 192.168.17.128 9523
# 使用 netstat 命令查看网络状态
xuweilin@xuweilin-virtual-machine:~$ netstat -nt | grep 9523
```

最后的运行结果如下所示：

```c{.line-numbers}
tcp        0      0 192.168.17.128:34968    192.168.17.128:9523     ESTABLISHED
tcp        0      1 192.168.17.128:60908    192.168.17.128:9523     SYN_SENT   
tcp        0      0 192.168.17.128:54288    192.168.17.128:9523     ESTABLISHED
tcp        0      0 192.168.17.128:41828    192.168.17.128:9523     ESTABLISHED
tcp        0      0 192.168.17.128:60894    192.168.17.128:9523     ESTABLISHED
tcp        0      0 192.168.17.128:57410    192.168.17.128:9523     ESTABLISHED
tcp        0      1 192.168.17.128:57288    192.168.17.128:9523     SYN_SENT   
tcp        0      0 192.168.17.128:57908    192.168.17.128:9523     ESTABLISHED
```

在监听队列中，处于 **`ESTABLISHED`** 状态的连接只有 6 个（**`backlog + 1`**），其余连接处于 **`SYN_SENT`** 状态。

## 九、发送/接收缓冲区大小选项

当使用 setsockopt 来设置 TCP 的接收缓冲区和发送缓冲区的大小时，**系统都会将其值加倍，并且不得小于某个最小值**。TCP 接收缓冲区的最小值是 256 字节（不同系统会有不同的值，比如我们测试的最小值为 2304 字节），而发送缓冲区的最小值是 2048 字节（不同系统会有不同的值）。系统这样做的目的主要是：确保一个 TCP 连接拥有足够的空闲缓冲区来处理拥塞（比如快速重传算法就期望 TCP 接收缓冲区能至少容纳 4 个大小为 SMSS 的 TCP 报文段）。

客户端发送数据给服务端，并且设置了发送缓冲区的大小（大小由程序参数传入），设置完之后打印一下设置的缓冲区大小：

```c{.line-numbers}
int main() {

    int cfd;
    int counter = 10;
    char buf[BUFFER_SIZE];
    unsigned int ip = 0;
    int sendbuf = 2400;
    int len = sizeof(sendbuf);

    // 服务器地址结构
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &ip);
    serv_addr.sin_addr.s_addr = ip;

    cfd = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(cfd, SOL_SOCKET, SO_SNDBUF, &sendbuf, sizeof(sendbuf));
    getsockopt(cfd, SOL_SOCKET, SO_SNDBUF, &sendbuf, (socklen_t *) &len);
    printf("the tcp send buffer size after setting is %d\n", sendbuf);

    connect(cfd, (struct sockaddr*) &serv_addr, sizeof(serv_addr));
    memset(buf, 'a', BUFFER_SIZE);
    send(cfd, buf, BUFFER_SIZE, 0);

    close(cfd);

    return 0;
}
```

服务端接收客户端的数据，并且设置了接收缓冲区的大小（大小由程序参数传入），设置完之后打印一下设置的缓冲区大小。

```c{.line-numbers}
int main() {

    int cfd, lfd;

    struct sockaddr_in serv_addr, clit_addr;
    socklen_t clit_addr_len;
    char buf[BUFFER_SIZE];
    int recvbuf = 50;
    int len = sizeof(recvbuf);

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    lfd = socket(AF_INET, SOCK_STREAM, 0);
    if (lfd == -1) {
        sys_err("socket error");
    }

    setsockopt(lfd, SOL_SOCKET, SO_RCVBUF, &recvbuf, sizeof(recvbuf));
    getsockopt(lfd, SOL_SOCKET, SO_RCVBUF, &recvbuf, (socklen_t *) &len);
    printf("the tcp receive buffer size after setting is %d\n", recvbuf);

    bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr));
    listen(lfd, 256);

    clit_addr_len = sizeof(clit_addr);
    // 模拟故障，阻塞 10s
    sleep(10);
    cfd = accept(lfd, (struct sockaddr *) &clit_addr, &clit_addr_len);

    if (cfd < 0) {
        if (errno == ECONNABORTED) {
            printf("accept: connect reset by peer\n");
        }
        return 1;
    }

    memset(buf, '\0', BUFFER_SIZE);
    while (recv(cfd, buf, BUFFER_SIZE, 0) > 0) {
    }

    close(lfd);
    return 0;
}
```

运行服务端程序，将其接收缓冲区大小设置为 50 字节（但是输出的大小却是 2304 字节，说明此系统的 TCP 接收端缓冲区最小值为 2304）：

```shell
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/server
the tcp receive buffer size after setting is 2304

Process finished with exit code 0
```

运行客户端程序，将其发送缓冲区大小设置为 2400 字节（但是输出的大小却是 4800，这与缓冲区会被系统设置加倍的概念相符合）：

```shell
/home/xuweilin/CLionProjects/linux_programming/cmake-build-debug/my_client
the tcp send buffer size after setting is 4800

Process finished with exit code 0
```

首先注意第 2 个 TCP 报文段，它指出服务器的接收通告窗口大小为 1152 字节。该值小于 2304 字节，显然是在情理之中。同时，该同步报文段还指出服务器采用的窗口扩大因子是 7。因此客户端可以一次发送 512 字节的数据。

```shell
xuweilin@xuweilin-virtual-machine:~/桌面$ sudo tcpdump -i 3 -nt
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
<------------------------- SYN 三次握手 ----------------------------------->
IP 127.0.0.1.54702 > 127.0.0.1.9523: Flags [S], seq 3369076222, win 65495, options [mss 65495,sackOK,TS val 3326519652 ecr 0,nop,wscale 7], length 0
IP 127.0.0.1.9523 > 127.0.0.1.54702: Flags [S.], seq 4207978142, ack 3369076223, win 1152, options [mss 65495,sackOK,TS val 3326519652 ecr 3326519652,nop,wscale 0], length 0
IP 127.0.0.1.54702 > 127.0.0.1.9523: Flags [.], ack 1, win 512, options [nop,nop,TS val 3326519652 ecr 3326519652], length 0

<------------------------- 发送普通数据 ----------------------  ------------>
IP 127.0.0.1.54702 > 127.0.0.1.9523: Flags [P.], seq 1:513, ack 1, win 512, options [nop,nop,TS val 3326519652 ecr 3326519652], length 512
IP 127.0.0.1.9523 > 127.0.0.1.54702: Flags [.], ack 513, win 640, options [nop,nop,TS val 3326519652 ecr 3326519652], length 0

<--------------------------  FIN 四次挥手 ------------------ --------------->
IP 127.0.0.1.54702 > 127.0.0.1.9523: Flags [F.], seq 513, ack 1, win 512, options [nop,nop,TS val 3326519652 ecr 3326519652], length 0
IP 127.0.0.1.9523 > 127.0.0.1.54702: Flags [.], ack 514, win 639, options [nop,nop,TS val 3326519696 ecr 3326519652], length 0
IP 127.0.0.1.9523 > 127.0.0.1.54702: Flags [F.], seq 1, ack 514, win 639, options [nop,nop,TS val 3326528013 ecr 3326519652], length 0
IP 127.0.0.1.54702 > 127.0.0.1.9523: Flags [.], ack 2, win 512, options [nop,nop,TS val 3326528013 ecr 3326528013], length 0
```

