# Linux 共享库的组织

这里先澄清一个说法，即共享库（Shared Library）的概念。其实 **<font color="red">从文件结构上来讲，共享库和共享对象没什么区别，Linux 下的共享库就是普通的 ELF 共享对象</font>**。由于共享对象可以被各个程序之间共享，所以它也就成为了库的很好的存在形式，很多库的开发者都以共享对象的形式让程序来使用，久而久之，共享对象和共享库这两个概念已经很模糊了，所以广义上我们可以将它们看作是同一个概念。

## 1.共享库版本

### 1.1 共享库兼容性

共享库的开发者会不停地更新共享库的版本，以修正原有的 Bug、增加新的功能或改进性能等。由于动态链接的灵活性，使得程序本身和程序所依赖的共享库可以分别独立开发和更新，比如当有程序 A 依赖于 **`libfoo.so`**，当 **`libfoo.so`** 的开发者宣布新版本开发完成之后，理论上我们只需要用新的 **`libfoo.so`** 将旧版本的替换掉即可享用新版 **`libfoo.so`** 提供的一切好处。**但是共享库版本的更新可能会导致接口的更改或删除，这可能导致依赖于该共享库的程序无法正常运行**。最简单的情况下，共享库的更新可以被分为两类。

- **兼容更新**：所有的更新只是在原有的共享库基础上增加一些内容，所有原有的接口都保持不变。
- **不兼容更新**：共享库更新改变了原有的接口，使用该共享库原有接口的程序可能不能运行或运行不正常。

这里讨论的接口是二进制接口，即 ABI（Application Binary Interface）。ABI 对于不同的语言来说，主要包括一些诸如函数调用的堆栈结构、符号命名、参数规则、数据结构的内存分布等方面的规则。下面列举了几种常见的共享库更改类型，以及修改后的兼容性：

- 往共享库 **`libfoo.so`** 里面添加一个导出符号 foo2，兼容；
- 删除共享库 **`libfoo.so`** 里面一个原有的导出符号 foo，不兼容；
- 将 **`libfoo.so`** 给一个导出函数添加一个参数，比如原来的 **`foo(int a)`** 变成了 **`foo(int a, int b)`**，不兼容；
- 删除一个导出函数中的一个参数，如原来的 **`foo(int a, int b)`** 变成了 **`foo(int a)`**，不兼容；
- 如果一个结构类型被用于一个导出函数或导出全局变量，那么改变结构类型的长度、内容、成员类型，如 **`libfoo.so`** 有导出函数 **`foo(struct bar b)`**，而 bar 的结构被改变，不兼容；
- 修正一个导出函数中的 Bug，或者改进某个导出函数的性能，但是不改变导出函数的语义、功能、行为和接口类型，兼容；
- 修正一个导出函数中的 Bug，或者改进某个导出函数的性能，但是同时改变了导出函数的语义、功能、行为或接口类型，不兼容；

导致 C 语言的共享库 ABI 改变的行为主要有如下 4 个：

- **导出函数的行为发生改变**，也就是说调用这个函数以后产生的结果与以前不一样，不再满足旧版本规定的函数行为准则；
- **导出函数被删除**。
- **导出数据的结构发生变化**，比如共享库定义的结构体变量的结构发生改变：结构成员删除、顺序改变或其他引起结构体内存布局变化的行为。
- **导出函数的接口发生变化**，如函数返回值、参数被更改。

如果能够保证上述 4 种情况不发生，那么绝大部分情况下，C 语言的共享库会保持 ABI 兼容。**<font color="red">注意，仅仅是绝大部分情况，要破坏一个共享库的 ABI 十分容易，要保持 ABI 的兼容却十分困难</font>**。很多因素会导致 ABI 的不兼容，比如不同版本的编译器、操作系统和硬件平台等，使得 ABI 兼容尤为困难。

### 1.2 共享库版本命名

既然共享库存在这样那样的兼容性问题，那么保持共享库在系统中的兼容性，保证依赖于它们的应用程序能够正常运行是必须要解决的问题。**其中最有效的办法之一就是使用共享库版本的方法**。Linux 有一套规则来命名系统中的每一个共享库，它规定共享库的文件名规则必须为：**`libname.so.x.y.z`**。

最前面使用前缀 lib，中间是库的名字和后缀 **`.so`**，最后面跟着的是三个数字组成的版本号。**<font color="red">x 表示主版本号（Major Version Number），y 表示次版本号（Minor Version Number），z 表示发布版本号（Release Version Number）</font>**。三个版本号的含义不一样。

**（1）主版本号**

**主版本号表示库的重大升级，不同主版本号的库之间是不兼容的**，依赖于旧的主版本号的程序需要改动相应的部分，并且重新编译，才可以在新版的共享库中运行；或者，系统必须保留旧版的共享库，使得那些依赖于旧版共享库的程序能够正常运行。

**（2）次版本号**

**次版本号表示库的增量升级，即增加一些新的接口符号，且保持原来的符号不变。在主版本号相同的情况下，高的次版本号的库应向后兼容低的次版本号的库**。一个依赖于旧的次版本号共享库的程序，可以在新的次版本号共享库中运行，因为新版中保留了原来所有的接口，并且不改变它们的定义和含义。比如系统中有一个共享库 **`libfoo.so.1.2.x`**，后来在升级过程中增加了一个函数，版本号变成了 **`1.3.x`**。因为 **`1.2.x`** 的所有接口都被保留到 **`1.3.x`** 中了，所以那些依赖于 **`1.1.x`** 或 **`1.2.x`** 的程序都可以在 **`1.3.x`** 中正常运行。

**（3）发布版本号**

发布版本号表示库的一些错误的修正、性能的改进等，并不添加任何新的接口，也不对接口进行更改。**相同主版本号、次版本号的共享库，不同的发布版本号之间完全兼容**，依赖于某个发布版本号的程序可以在任何一个其他发布版本号中正常运行，而无须做任何修改。

### 1.3 SO-NAME

可以这么说，共享库的主版本号和次版本号决定了一个共享库的接口。我们假设程序中有一个它所依赖的共享库的列表，其中每一项对应于它所依赖的一个共享库。**<font color="red">可以肯定的是，程序中必须包含被依赖的共享库的名字和主版本号，因为我们知道不同主版本号之间的共享库是完全不兼容的</font>**。所以程序中保存在一个诸如 **`libfoo.so.2`** 的记录，以防止动态链接器在运行时意外地将程序与 **`libfoo.so.1`** 或 **`libfoo.so.3`** 链接到一起。通过这个可以发现，如果在系统中运行旧的应用程序，就需要在系统中保留旧应用程序所需要的旧的主版本号的共享库。

对于新的系统来说，包括 Solaris 和 Linux，普遍采用一种叫做 SO-NAME 的命名机制来记录共享库的依赖关系。**<font color="red">每个共享库都有一个对应的 SO-NAME，这个 SO-NAME 即共享库的文件名去掉次版本号和发布版本号，保留主版本号</font>**。比如一个共享库叫做 **`libfoo.so.2.6.1`**，那么它的 SO-NAME 即 **`libfoo.so.2`**。很明显，SO-NAME 规定了共享库的接口，**SO-NAME 相同的两个共享库，次版本号大的兼容次版本号小的**。

在 Linux 系统中，**系统会为每个共享库在它所在的目录创建一个跟 SO-NAME 相同的并且指向它的软链接（Symbol Link）**。比如系统中存在一个共享库 **`/lib/libfoo.so.2.6.1`**，那么 Linux 中的共享库管理程序就会为它产生一个软链接 **`/lib/libfoo.so.2`** 指向它。示例如下所示：

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders$ ls -l /lib32/lib*
lrwxrwxrwx 1 root root      17  5月 13  2023 /lib32/libubsan.so.1 -> libubsan.so.1.0.0
-rw-r--r-- 1 root root 2549604  5月 13  2023 /lib32/libubsan.so.1.0.0
lrwxrwxrwx 1 root root      19  5月 13  2023 /lib32/libstdc++.so.6 -> libstdc++.so.6.0.30
-rw-r--r-- 1 root root 2295720  5月 13  2023 /lib32/libstdc++.so.6.0.30
lrwxrwxrwx 1 root root      20  5月 13  2023 /lib32/libquadmath.so.0 -> libquadmath.so.0.0.0
-rw-r--r-- 1 root root  689956  5月 13  2023 /lib32/libquadmath.so.0.0.0
lrwxrwxrwx 1 root root      15  5月 13  2023 /lib32/libitm.so.1 -> libitm.so.1.0.0
-rw-r--r-- 1 root root  100028  5月 13  2023 /lib32/libitm.so.1.0.0
lrwxrwxrwx 1 root root      16  5月 13  2023 /lib32/libasan.so.6 -> libasan.so.6.0.0
-rw-r--r-- 1 root root 6813784  5月 13  2023 /lib32/libasan.so.6.0.0
lrwxrwxrwx 1 root root      18  5月 13  2023 /lib32/libatomic.so.1 -> libatomic.so.1.2.0
-rw-r--r-- 1 root root   26096  5月 13  2023 /lib32/libatomic.so.1.2.0
```

那么以 SO-NAME 为名建立软链接有什么用处呢？**其实这个软链接会指向目录中主版本号相同、次版本号和发布版本号最新的共享库**。也就是说，比如目录中有两个共享库版本分别为：**`/lib/libfoo.so.2.6.1`** 和 **`/lib/libfoo.so.2.5.3`**，那么软链接 **`libfoo.so.2`** 会指向 **`/lib/libfoo.so.2.6.1`**。这样保证了所有的以 SO-NAME 为名的软链接都指向系统中最新版本的共享库。

**<font color="red">建立以 `SO-NAME` 为名的软链接的目的，是使得所有依赖某个共享库的模块，在编译链接和运行时，都使用共享库的 `SO-NAME`，而不使用详细的版本号</font>**。我们在前面介绍动态链接文件中的 **`.dynamic`** 段时已经提到过，如果某文件 A 依赖于某文件 B，那么 A 的 **`.dynamic`** 段中会有 **`DT_NEEDED`** 类型的字段，字段的值就是 B。

现在有一个问题，这个字段该如何表示 B 这个文件呢？如果依赖的是 B 的文件名，即包含次版本号和发布版本号，那么会有什么问题呢？很直接的问题是，这个文件 A 能够依赖某个特定版本的 B。比如程序 A 依赖于 C 语言库，它在编译时，系统中存在的 C 语言库版本是 **`lib/c/libc-2.6.1.so`**，而 A 的动态链接器会解析出对应的 **`.dynamic`** 中的 **`DT_NEEDED`** 类型，系统将通过它指向新的库版本 **`lib/c/libc-2.6.1.so`**。当系统将 C 语言库版本升级至 **`2.6.2`** 或 **`2.7.1`** 时，系统必须保留原来的 **`2.6.1`** 的共享库，否则这个程序 A 就无法运行。

但是我们知道，因为根据 Linux 的共享库规范，实际上 **`2.6.2`** 或 **`2.7.1`** 版的共享库是兼容 **`2.6.1`** 的，我们不需要继续保留原来的 **`2.6.1`**，而系统中将遗留大量的各种版本的共享库，大大浪费了磁盘和内存空间。**所以一个可行的方法就是编译输出 ELF 文件时，将被依赖的共享库的 SO-NAME 保存在 `.dynamic` 中**，这样当动态链接器进行共享库依赖文件查找时，就会根据系统中各个共享库目录中的 SO-NAME 软链接自动指向到最新版本的共享库。示例如下所示：

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders$ readelf -d /lib32/libstdc++.so.6 | grep SONAME
 0x0000000e (SONAME)                     Library soname: [libstdc++.so.6]
monica@monica-virtual-machine:~/linkers_loaders$ readelf -d /lib32/libatomic.so.1  | grep SONAME
 0x0000000e (SONAME)                     Library soname: [libatomic.so.1]
```

根据 gABI（ELF 规范），**<font color="red">应用程序依赖列表中的名称（`DT_NEEDED`），要么是被依赖库的 `DT_SONAME` 字符串，要么就是构建时使用的库文件路径名</font>**。比如，我们把 **`libfoo.so.1.2`**（**`SONAME=libfoo.so.1`**）拿来链接可执行程序 ab，最终可执行程序 ab 的依赖列表中 **`DT_NEEDED`** 写的是 **`libfoo.so.1`**，而不是 **`libfoo.so.1.2`**。如果我们链接了一个没有 SONAME 的共享库 **`/usr/lib/libbar.so`**，那 **`DT_NEEDED`** 里就会出现这个带路径的名字，运行期加载器也按这个路径找它。

注意，只要库有 **`SONAME`**，链接器就把 **`SONAME`** 写进可执行文件的 **`DT_NEEDED`**，无论你是用 **`-lXXX`** 还是直接用绝对路径链接。只有当库里没有 **`DT_SONAME`** 时，链接器才会把用于链接的名字（可能是带斜杠的路径名）写进 **`DT_NEEDED`**。

>Names in the dependency list are copies either of the **`DT_SONAME`** strings or the path names of the shared objects used to build the object file. For example, if the link editor builds an executable file using one shared object with a **`DT_SONAME`** entry of lib1 and another shared object library with the path name **`/usr/lib/lib2`**, the executable file will contain lib1 and **`/usr/lib/lib2`** in its dependency list.

当共享库进行升级的时候，如果只是增量升级，即保持主版本号不变，只改变次版本号或发布版本号，那么我们可以直接将新版的共享库替换旧版，并且修改 SO-NAME 的软链接指向新版共享库，即实现升级；当共享库的主版本号升级时，系统中就会存在多个 SO-NAME，由于这些 SO-NAME 并不相同，所以已有的程序并不会受到影响。

### 1.4 链接名

当我们在编译器里面使用共享库的时候（比如使用 **`gcc -l`** 参数链接某个共享库），我们使用了更为简洁的方式，比如需要链接 **`libXXX.so.2.6.1`** 的共享库，只需要在编译器命令行里指定 **`-lXXX`** 即可，省略所有其他部分。编译器会根据当前环境，在系统中的相关路径（往往由 **`-L`** 参数指定）查找最新版本的 XXX 库。这个 XXX 又被称为共享库的链接名（Link Name）。

## 2.SONAME 总结

### 2.1 独立库的 SONAME 机制

在 Linux 下，动态库通常有三种名字：

- 真实文件名：如 **`libubsan.so.1.0.0`**；
- SONAME（共享对象名）：如 **`libubsan.so.1`**；
- 链接名（编译期用的）：如 **`libubsan.so`**；

编译和链接时，编译器通过链接名 **`-lubsan`** 找到 **`libubsan.so`** 真实共享库，然后读取其中的 **`DT_SONAME`**，然后把这个 **`SONAME`** 记录进可执行程序的 **`NEEDED`** 条目里即可。动态链接时，程序的 ELF 文件记录下依赖的 SONAME（比如 **`NEEDED libubsan.so.1`**）。运行时，动态链接器只会在系统库路径下寻找 同名的 SONAME 文件（比如 **`libubsan.so.1`**），找到就认为匹配。

下面我们以 **`libstdc++.so`** 和 **`libubsan.so`** 这两个共享库为例。

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders$ ls -l /lib32/lib*
lrwxrwxrwx 1 root root      17  5月 13  2023 /lib32/libubsan.so.1 -> libubsan.so.1.0.0
-rw-r--r-- 1 root root 2549604  5月 13  2023 /lib32/libubsan.so.1.0.0
lrwxrwxrwx 1 root root      19  5月 13  2023 /lib32/libstdc++.so.6 -> libstdc++.so.6.0.30
-rw-r--r-- 1 root root 2295720  5月 13  2023 /lib32/libstdc++.so.6.0.30
lrwxrwxrwx 1 root root      20  5月 13  2023 /lib32/libquadmath.so.0 -> libquadmath.so.0.0.0
-rw-r--r-- 1 root root  689956  5月 13  2023 /lib32/libquadmath.so.0.0.0
lrwxrwxrwx 1 root root      15  5月 13  2023 /lib32/libitm.so.1 -> libitm.so.1.0.0
-rw-r--r-- 1 root root  100028  5月 13  2023 /lib32/libitm.so.1.0.0
lrwxrwxrwx 1 root root      16  5月 13  2023 /lib32/libasan.so.6 -> libasan.so.6.0.0
-rw-r--r-- 1 root root 6813784  5月 13  2023 /lib32/libasan.so.6.0.0
lrwxrwxrwx 1 root root      18  5月 13  2023 /lib32/libatomic.so.1 -> libatomic.so.1.2.0
-rw-r--r-- 1 root root   26096  5月 13  2023 /lib32/libatomic.so.1.2.0
monica@monica-virtual-machine:~/linkers_loaders$ file /lib32/libstdc++.so.6
/lib32/libstdc++.so.6: symbolic link to libstdc++.so.6.0.30
```

在上面的目录中，可以看到只有编译时的 SONAME 软链接 **`libstdc++.so.6 -> libstdc++.so.6.0.30`**，却没有编译期的 **`libstdc++.so -> libstdc++.so.6`**，这是因为在 Ubuntu/Debian 等发行版里，运行期库和开发包是分开的。

**<font color="red">开发包（`*-dev`）才提供无版本号的 **`libxxx.so`**（给链接器在编译/链接时使用）</font>**。The development package should contain a symlink for the associated shared library without a version number. For example, the libgdbm-dev package should include a symlink from **`/usr/lib/libgdbm.so`** to **`libgdbm.so.3.0.0`**. This symlink is needed by the linker (ld) when compiling packages, as it will only look for **`libgdbm.so`** when compiling dynamically.

开发包中的 32 位 **`libstdc++.so`** 在 GCC 目录下，如下所示，**`/usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so`** 作为一个软链接指向 **`/lib32/libstdc++.so.6`**，这个 **`libstdc++.so.6`** 就是此共享库的 SONAME。

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -L lib32stdc++-11-dev | grep stdc++
/usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so
monica@monica-virtual-machine:~/linkers_loaders$ ls -l /usr/lib/gcc/x86_64-linux-gnu/11/32/ | grep stdc++
-rw-r--r-- 1 root root 5036114  5月 13  2023 libstdc++.a
lrwxrwxrwx 1 root root      35  5月 13  2023 libstdc++.so -> ../../../../../lib32/libstdc++.so.6
monica@monica-virtual-machine:~/linkers_loaders$ file /usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so
/usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so: symbolic link to ../../../../../lib32/libstdc++.so.6
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -S /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30 
libstdc++6:amd64: /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30
```

下面是 32/64 位的 **`libstdc++`** 开发包和运行包的不同目录路径，开发期 **`.so`** 软链接（给链接器用）和运行时 **`.so.N`** SONAME 软链接（给动态装载器用）不在同一目录；而且32 位与 64 位也会放在不同前缀目录，这是 Ubuntu 的多架构规范。

>有的发行版/版本会把运行库放在 **`/usr/lib/x86_64-linux-gnu`**，有的在 **`/lib/x86_64-linux-gnu`**；两处都在默认搜索路径里，不影响使用。

```c{.line-numbers}
// 32 位的 libstdc++ 的开发包路径 /usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -S /usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so
lib32stdc++-11-dev: /usr/lib/gcc/x86_64-linux-gnu/11/32/libstdc++.so
// 32 位的 libstdc++ 的运行包路径 /usr/lib32/libstdc++.so.6
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -S /usr/lib32/libstdc++.so.6
lib32stdc++6: /usr/lib32/libstdc++.so.6
// 64 位的 libstdc++ 的开发包路径 /usr/lib/gcc/x86_64-linux-gnu/11/libstdc++.so
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -S /usr/lib/gcc/x86_64-linux-gnu/11/libstdc++.so
libstdc++-11-dev:amd64: /usr/lib/gcc/x86_64-linux-gnu/11/libstdc++.so
// 64 位的 libstdc++ 的运行包路径 /usr/lib/x86_64-linux-gnu/libstdc++.so.6
monica@monica-virtual-machine:~/linkers_loaders$ dpkg -S /usr/lib/x86_64-linux-gnu/libstdc++.so.6
libstdc++6:amd64: /usr/lib/x86_64-linux-gnu/libstdc++.so.6
```

运行期包只提供真实库文件（如 **`libstdc++.so.6.0.30`**）和 SONAME 链接（如 **`libstdc++.so.6`**），供运行时动态装载器使用。A shared library is identified by the SONAME attribute stored in its dynamic section. **<font color="red">When a binary is linked against a shared library, the `SONAME` of the shared library is recorded in the binary’s `NEEDED` section</font>** so that the dynamic linker knows that library must be loaded at runtime. The shared library file’s full name (which usually contains additional version information not needed in the SONAME) is therefore normally not referenced directly. Instead, the shared library is loaded by its SONAME, which exists on the file system as a symlink pointing to the full name of the shared library. This symlink must be provided by the package.

Shared libraries are normally split into several binary packages. The SONAME symlink is installed by the runtime shared library package, and the bare .so symlink is installed in the development package since it’s only used when linking binaries or shared libraries. **<font color="red">总结来说就是 **`SONAME`** 软链接由运行时库包安装；而裸的 **`.so`** 链接只在开发包里出现，因为它仅在编译/链接阶段需要</font>**。

### 2.2 独立库的 SONAME 机制—实例

下面用一个自定义 C 共享库做最小可复现示例，清楚展示以下三层关系，并且验证最后可执行文件里的 **`DT_NEEDED`** 就是 **`SONAME`**。

```c{.line-numbers}
编译期名（link name）：libhello.so
-> 运行期 SONAME：libhello.so.2
-> 真实库文件（realname）：libhello.so.2.3.4
```

#### 2.2.1 创建独立库

下面我们创建一个共享库 **`libhello.so.2.3.4`**，并且在创建时指定了共享库的 SONAME 为 **`libhello.so.2`**，随后创建编译期名到运行期 SONAME 的软链接，运行期 SONAME 到真实库文件的名字。**<font color="red">只要库有 SONAME，链接器就把共享库的 SONAME 写进可执行文件的 `DT_NEEDED`，只有当库里没有 `DT_SONAME` 时，链接器才会把用于链接的名字（可能是带斜杠的路径名）写进 `DT_NEEDED`</font>**，后面会进行验证。

```shell{.line-numbers}
# 0) 准备干净目录
monica@monica-virtual-machine:~/linkers_loaders$ mkdir -p ./tmp/soname-demo && cd ./tmp/soname-demo

# 1) 写库与可执行的源码
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ cat > hello.c << 'EOF'
> #include <stdio.h>
> void hello(void) { printf("hello from libhello\n"); }
> EOF
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ cat > main.c << 'EOF'
> void hello(void);
> int main() {hello();return 0;}
> EOF

# 2) 构建共享库：给它设置 SONAME=libhello.so.2，真实文件名带完整版本号
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -fPIC -c hello.c
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared -Wl,-soname,libhello.so.2 -o libhello.so.2.3.4 hello.o

# 3) 建立两级符号链接：编译期名 -> SONAME -> 真实文件
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ln -s libhello.so.2.3.4 libhello.so.2
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ln -s libhello.so.2 libhello.so
# 看看链条：
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l libhello.so libhello.so.2 libhello.so.2.3.4
lrwxrwxrwx 1 monica monica    13  9月 25 21:41 libhello.so -> libhello.so.2
lrwxrwxrwx 1 monica monica    17  9月 25 21:41 libhello.so.2 -> libhello.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 25 21:40 libhello.so.2.3.4

# 4) 生成可执行文件：让链接器在当前目录找库，并把运行路径设为 $ORIGIN（可执行所在目录）
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello -Wl,-rpath,'$ORIGIN' -o app
```

接下来运行 app，成功运行并输出字符串。并且 **`libhello.so.2.3.4`** 的 SONAME 就是我们设置的 **`libhello.so.2`**，app 可执行程序的 **`DT_NEEDED`** 值就是共享库的 SONAME，也就是 **`libhello.so.2`**。

```c{.line-numbers}
# 1) 运行并验证
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ./app
hello from libhello

# 2) 验证库与可执行的元数据
#   2.1 库的 SONAME（来自真实文件的动态段）
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libhello.so.2.3.4 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libhello.so.2]
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libhello.so.2 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libhello.so.2]

#   2.2 可执行文件的 NEEDED
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d app | grep NEEDED
 0x0000000000000001 (NEEDED)             共享库：[libhello.so.2]
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]

#   2.3 运行期解析（ldd 会把 SONAME 解析到实际路径）
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ldd ./app | grep libhello
	libhello.so.2 => /home/monica/linkers_loaders/tmp/soname-demo/./libhello.so.2 (0x000073ec55a58000)
```

下面分别是链接时，链接器将共享库的 **`SONAME`** 写入可执行程序 **`DT_NEEDED`** 的过程，以及运行时动态加载器根据可执行程序中的 SONAME 找到真实共享库并加载的过程。

```c{.line-numbers}
编译期：
  -lhello
    └─ 在当前目录 ./tmp/soname-demo 找到 libhello.so
        └─ 通过链接找到实际共享对象：libhello.so.2.3.4
           链接器读取 SONAME = libhello.so.2
            └─ 在可执行程序中写入 NEEDED: libhello.so.2

运行期：
  动态装载器读取 NEEDED: libhello.so.2
    └─ ./tmp/soname-demo/libhello.so.2 -> libhello.so.2.3.4
       └─ 加载成功
```

#### 2.2.2 独立库—不使用 SONAME

由于只要库有 SONAME，链接器就把共享库的 SONAME 写进可执行文件的 **`DT_NEEDED`**，只有当库里没有 **`DT_SONAME`** 时，链接器才会把用于链接的名字（可能是带斜杠的路径名）写进 **`DT_NEEDED`**。

```shell{.line-numbers}
# 1) 构建共享库：不设置 SONAME
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -fPIC -c hello.c
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared -o libhello.so.2.3.4 hello.o

# 2) 建立两级符号链接：编译期名 -> SONAME -> 真实文件
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ln -s libhello.so.2.3.4 libhello.so.2
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ln -s libhello.so.2 libhello.so
# 看看链条：
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l libhello.so libhello.so.2 libhello.so.2.3.4
lrwxrwxrwx 1 monica monica    13  9月 25 21:41 libhello.so -> libhello.so.2
lrwxrwxrwx 1 monica monica    17  9月 25 21:41 libhello.so.2 -> libhello.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 25 21:40 libhello.so.2.3.4

# 3) 生成可执行文件：让链接器在当前目录找库，并把运行路径设为 $ORIGIN（可执行所在目录）
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello -Wl,-rpath,'$ORIGIN' -o app

# 4) 运行并验证
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ./app
hello from libhello

# 5) 共享库 libhello.so.2.3.4 没有 SONAME
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libhello.so.2.3.4 | grep SONAME

# 6) 可执行文件的 NEEDED 就是编译链接时直接指定的共享库路径
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d app | grep NEEDED
 0x0000000000000001 (NEEDED)             共享库：[./libhello.so.2.3.4]
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]
```

如上所示，在生成共享库时，没有指定 SONAME，因此在链接时，链接器才会将链接中出现的路径名 **`./libhello.so.2.3.4`** 直接写到可执行程序的 **`DT_NEEDED`** 字段中，并且指定了运行时库的搜索路径除了默认的目录外，还有当前目录。因此加载器根据 app 中 **`DT_NEEDED`** 的值去当前目录中找到 **`libhello.so.2.3.4`** 共享库并加载到内存中。

#### 2.2.3 独立库—搜索路径

在以下命令中，**`-L.`** 只影响链接阶段，让链接器在当前目录找到 **`libhello.so`**，并把库的 SONAME（即 **`libhello.so.2`**）写进可执行文件的 **`DT_NEEDED`**。对运行时的加载路径没有作用。

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello -Wl,-rpath,'$ORIGIN' -o app
```

**`-Wl,-rpath,'$ORIGIN'`** 选项会把可执行文件的 **`RUNPATH/RPATH`** 设为 **`$ORIGIN`**（可执行所在目录），于是动态装载器能在当前目录找到 **`libhello.so.2`**。**<font color="red">在多数发行版上，`-Wl,-rpath` 默认生成 RUNPATH</font>**。去掉此选项后，动态加载器在默认的目录下（**`LD_LIBRARY_PATH`**、**`ld.so.cache`**、**`/lib`**、**`/usr/lib`** 等）无法找到 **`libhello.so.2.3.4`** 共享库，因此会报错。

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello -o app
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ./app
./app: error while loading shared libraries: libhello.so: cannot open shared object file: No such file or directory
```

解决办法除了 **`-Wl,-rpath,'$ORIGIN'`** 外还有如下：

**（1）使用 `LD_LIBRARY_PATH`**

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello  -o app
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ./app
./app: error while loading shared libraries: libhello.so: cannot open shared object file: No such file or directory
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ LD_LIBRARY_PATH=. ./app
hello from libhello
```

运行时临时指定查找共享库的路径除了默认的目录外，还有当前目录。

**（2）使用 ldconfig**

ldconfig 这个程序的作用是为共享库目录下的各个共享库创建、删除或更新相应的 SO-NAME（即相应的符号链接），这样每个共享库的 SO-NAME 就能够指向正确的共享库文件；并且这个程序还会将这些 SO-NAME 收集起来，集中存放到 **`/etc/ld.so.cache`** 文件里面，并建立一个 SO-NAME 的缓存。当动态链接器要查找共享库时，它可以直接从 **`/etc/ld.so.cache`** 里面查找。

我们可以修改 **`/etc/ld.so.conf`** 文件，将 **`/home/monica/linkers_loaders/tmp/soname-demo`** 目录加入其中，这样 ldconfig 命令就会将 **`/home/monica/linkers_loaders/tmp/soname-demo`** 目录中的映射关系保存到 **`/etc/ld.so.cache`** 中。最后运行 app 程序时，可以根据 **`DT_NEEDED`** 值（**`libhello.so`**）在 **`/etc/ld.so.cache`** 成功找到真实共享库路径，然后成功运行。

```c{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d app | grep NEEDED
 0x0000000000000001 (NEEDED)             共享库：[libhello.so]
 0x0000000000000001 (NEEDED)             共享库：[libc.so.6]

monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ sudo vi /etc/ld.so.conf
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ sudo ldconfig
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ldconfig -p | grep hello
	libhello.so.2.3.4 (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libhello.so.2.3.4
	libhello.so.2 (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libhello.so.2
	libhello.so (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libhello.so
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ./app
hello from libhello
```

### 2.3 glibc 的 SONAME 机制

**<font color="red">glibc 大量使用符号版本（symbol versioning）来保持 ABI 兼容，因此很少变更 SONAME；打包时常直接把实际文件就命名为 SONAME（没有 **`.X.Y.Z`** 的更长文件名）</font>**。下面以 **`libanl.so`** 为例来进行解释，可以看到只有 **`libanl.so`**、**`libanl.so.1`**，并没有看到 **`libanl.so.1.Y.Z`** 这个包，**`libanl.so.1`** 就是实际的共享库，而不是软链接。

```shell{.line-numbers}
# 可以看到 libanl.so.1 在 libc6 运行包中
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ dpkg -L libc6 | grep anl
/lib/x86_64-linux-gnu/libanl.so.1
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ dpkg -S /lib/x86_64-linux-gnu/libanl.so.1
libc6:amd64: /lib/x86_64-linux-gnu/libanl.so.1

# 可以看到 libanl.so 在 libc6 开发包中
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ dpkg -L libc6-dev | grep anl
/usr/lib/x86_64-linux-gnu/libanl.a
/usr/lib/x86_64-linux-gnu/libanl.so
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ dpkg -S /usr/lib/x86_64-linux-gnu/libanl.so
libc6-dev:amd64: /usr/lib/x86_64-linux-gnu/libanl.so

# 可以看到 libanl.so.1 本身就是真实的库文件，而不是软链接，并且大小为 14432 字节
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ file /lib/x86_64-linux-gnu/libanl.so.1
/lib/x86_64-linux-gnu/libanl.so.1: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=7fe8b780dd1e4c5e714edf3f03f3a640450069bd, for GNU/Linux 3.2.0, stripped
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l /lib/x86_64-linux-gnu/libanl.so.1
-rw-r--r-- 1 root root 14432  1月 29  2025 /lib/x86_64-linux-gnu/libanl.so.1

monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l /usr/lib/x86_64-linux-gnu/ | grep anl
-rw-r--r--  1 root root        8  1月 29  2025 libanl.a
lrwxrwxrwx  1 root root       33  1月 29  2025 libanl.so -> /lib/x86_64-linux-gnu/libanl.so.1
-rw-r--r--  1 root root    14432  1月 29  2025 libanl.so.1
```

## 3.共享库的查找过程

### 3.1 共享库的系统路径


