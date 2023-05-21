# 服务器编程示例与难点

## 一、简介

我们将在本章使用 socket 的基本函数编写一个完整的 TCP 客户/服务器程序示例。这个简单的例子是执行如下步骤的一个回射服务器（Echo Server）：

（1）客户从标准输入读入一行文本，并写给服务器；
（2）服务器从网络输入读入这行文本，并回射给客户；
（3）客户从网络输入读入这行回射文本，并显示在标准输出上。

其结构如下图所示：

<div align="center">
    <img src="6_Socket_服务器编程难点/1.png" width="500"/>
</div>

实现任何客户/服务器网络应用所需的所有基本步骤都可以通过 Echo Server 阐明，如果想要把这个例子扩充成其他的应用程度，只需要修改服务器对来自客户的输入处理过程。

### 1. TCP 回射服务器程序：main 函数

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

### 1.accept 返回前连接中止

这里，三路握手完成从而连接建立之后，客户 TCP 却发送了一个 RST（复位）。**在服务器端看来，就在该连接已由 TCP 排队，等着服务器进程调用 accept 的时候 RST 到达。稍后，服务器进程调用 accept**。

模拟这种情形的一个简单方法就是：启动服务器，让它调用 socket、bind 和 listen，然后在调用 accept 之前睡眠一小段时间。在服务器进程睡眠时，启动客户，让它调用 socket 和 connect。一旦 connect 返回，就设置 SO＿LINGER 套接字选项以产生这个 RST，然后终止。

<div align="center">
    <img src="6_Socket_服务器编程难点/2.png" width="500"/>
</div>

但是，如何处理这种中止的连接依赖于不同的实现。源自 Berkeley 的实现完全在内核中处理中止的连接，服务器进程根本看不到。然而大多数 SVR4 实现返回一个错误给服务器进程，作为 accept 的返回结果，不过错误本身取决于实现。这些 SVR4 实现返回一个 EPROTO（“protocol error”，协议错误）errno 值。

而 POSIX 指出返回的 **errno 值必须是 ECONNABORTED（"software caused connection abort"，软件引起的连接中止）**。POSIX 作出修改的理由在于：流子系统（streams subsystem）中发生某些致命的协议相关事件时，也会返回 EPROTO。要是对于由客户引起的一个已建立连接的非致命中止也返回同样的错误，那么服务器就不知道该再次调用 accept 还是不该了。换成 ECONNABORTED 错误，服务器就可以忽略它，再次调用 accept 就行。

accept(2) man page 写道 `[ECONNABORTED] A connection arrived, but it was closed while waiting on the listen queue.`，即连接已到达，但在等待侦听队列时已关闭。

### 2.服务器进程终止

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

当我们键入 "another line" 时，str_cli 调用 writen，客户 TCP 接着把数据发送给服务器。
TCP 允许这么做，因为客户 TCP 接收到 FIN 只是表示服务器进程已关闭了连接的服务器端，从而不再往其中发送任何数据而己。FIN 的接收并没有告知客户 TCP 服务器进程已经终止（虽然在本例子中它确实是终止了）。当服务器 TCP 接收到来自客户的数据时，既然先前打开那个套接字的进程已经终止，于是响应以一个 RST。

7. 然而客户进程看不到这个 RST，因为它在调用 writen 后立即调用 read1line，并且由于第 2 步中接收的 FIN，所调用的 readline 立即返回 0（表示 EOF）。我们的客户端此时收到 EOF，于是以出错信息 "server terminated prematurely"（服务器过早终止）退出。
8. 当客户终止时，它所有打开着的描述符都被关闭

我们的上述讨论还取决于本例子的时序，客户调用 readline 既可能发生在服务器的 RST 被客户收到之前，也可能发生在收到之后。如果 readline 发生在收到 RST 之前（如本例子所示），那么结果是客户得到一个未预期的 EOF；否则结果是由 readline：返回一个 ECONNRESET ("connection reset by peer"，对方复位连接错误)，表示服务端的进程被终止。

本例子的问题在于：当 FIN 到达套接字时，客户正阻塞在 fgets 调用上，无法及时对 FIN 报文进行响应，关闭掉客户端连接，导致后续又向服务端发送报文，导致服务器发送 RST 重置为报文。

客户实际上在应对两个描述符-套接字和用户输入，它不能单纯阻塞在这两个源中某个特定源的输入上（正如目前编写的 str＿cli 函数所为），而是应该阻塞在其中任何一个源的输入上。事实上这正是 select 和 po11 这两个函数的目的之一，我们可以使用 I/O 多路复用函数重新编写 str＿cli 函数之后，一旦杀死服务器子进程，客户就会立即被告知已收到 FIN。

