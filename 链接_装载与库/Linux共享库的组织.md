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

下面分别是链接时，链接器将共享库的 **`SONAME`** 写入可执行程序 **`DT_NEEDED`** 的过程，以及运行时动态加载器根据可执行程序中的 SONAME 找到真实共享库并加载的过程。

```c{.line-numbers}
编译期：
  -lanl
    └─ 找到开发期名：/usr/lib/i386-linux-gnu/libanl.so 或 /lib32/libanl.so
        └─ 链接器读取 SONAME = "libanl.so.1"
            └─ 在产物中写入 NEEDED: libanl.so.1

运行期：
  动态装载器读取 NEEDED: libanl.so.1
    └─ 在 /lib32/ 或 /lib/x86_64-linux-gnu/ 中找到 libanl.so.1（真实文件）
        └─ 加载成功
```

## 3.共享库的查找过程

### 3.1 ldconfig 命令

ldconfig creates the necessary links and cache to the most recent shared libraries found **<font color="red">in the directories specified on the command line, in the file `/etc/ld.so.conf`, and in the trusted directories, `/lib` and `/usr/lib`</font>**. On some 64-bit architectures such as x86-64, **`/lib`** and **`/usr/lib`** are the trusted directories for 32-bit libraries, while **`/lib64`** and **`/usr/lib64`** are used for 64-bit libraries.

扫描命令行给出的目录、**`/etc/ld.so.conf`**（含其 **`*.conf`** 包含项）以及受信任目录（常见是 **`/lib`**、**`/usr/lib`**；在 64 位系统上，32 位/64 位分别对应 **`/lib`**、**`/usr/lib`** 与 **`/lib64`**、**`/usr/lib64`**），据此重建 **`/etc/ld.so.cache`**。这个缓存是运行时动态链接器 **`ld.so/ld-linux.so`** 查库时用的"索引"，能显著加速查找。

The cache is used by the run-time linker, ld.so or ld-linux.so. ldconfig checks the header and filenames of the libraries it encounters when determining which versions should have their links updated. ldconfig should normally be run by the superuser as it may require write permission on some root owned directories and files.

只处理名字形如 **`lib*.so*`**（普通共享库）和 **`ld-*.so*`**（动态链接器本体）。ldconfig 会据此把"同一 SONAME 中最新"的版本设为目标，必要时更新软链接。

ldconfig will look only at files that are named lib*.so* (for regular shared objects) or ld-*.so* (for the dynamic loader itself).  Other files will be ignored.  Also, ldconfig expects a certain pattern to how the symbolic links are set up, like this example, where the middle file (libfoo.so.1 here) is the SONAME for the library:

```c{.line-numbers}
libfoo.so -> libfoo.so.1 -> libfoo.so.1.12
```

ldconfig 影响的是运行时找库，它不改变编译/链接期的 **`-L/-l`** 行为，运行期搜索顺序与 **`ld.so`** 规则相关，**`/etc/ld.so.cache`** 就是其中关键一环。如果希望安装/升级/移动共享库之后，希望程序无需设置 **`LD_LIBRARY_PATH`** 就能在新路径找到库，可以把共享库所在目录的路径写到 **`/etc/ld.so.conf`** 中，然后 **`sudo ldconfig`** 即可。可以使用 **`ldconfig -p`** 来查看当前缓存里有哪些库与目录。

**`ldconfig -n DIR`** 命令表示只处理命令行上给出的目录 DIR 来更新 SONAME 链，不扫描受信任目录（如 **`/lib`**、**`/usr/lib`**），也不读取 **`/etc/ld.so.conf`**，并且隐含 -N，即不重建全局缓存 **`/etc/ld.so.cache`**。

#### 3.1.1 ldconfig 遇到多个相同 SONAME 的库

ldconfig 在遇到多个提供相同 SONAME 的库时，会在该目录内选择版本号更高的那个，并建立/更新 SONAME 链接。版本号更低的共享库会保留在目录中，但不再是 SONAME 的指向目标。

```shell{.line-numbers}
# 1.创建一个共享库 libfoo.so.1.2.3，其 SONAME 为 libfoo.so.1
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared hello.o -o libfoo.so.1.2.3 -Wl,-soname,libfoo.so.1 
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libfoo.so.1.2.3 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libfoo.so.1]

# 2.创建一个共享库 libfoo.so.2.3.4，其 SONAME 也为 libfoo.so.1
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared hello.o -o libfoo.so.2.3.4 -Wl,-soname,libfoo.so.1 
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libfoo.so.2.3.4 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libfoo.so.1]

# 3.当前目录 ~/linkers_loaders/tmp/soname-demo 也在 /etc/ld.so.conf 文件中
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ cat /etc/ld.so.conf
include /etc/ld.so.conf.d/*.conf
/home/monica/software/linux_dir/dynamic_lib/lib
/usr/local/lib
/home/monica/linkers_loaders/tmp/soname-demo

# 4.使用命令 ldconfig 来更新扫描 /etc/ld.so.conf 以及受信任目录中的共享库 SONAME 链接，并且更新 /etc/ld.so.cache
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ sudo ldconfig

# 5.ls -l 命令的结果显示 ldconfig 为版本号更高的那个共享库建立 SONAME 链接
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l
总用量 76
-rwxrwxr-x 1 monica monica 15952  9月 25 23:27 app
-rw-rw-r-- 1 monica monica    73  9月 25 21:34 hello.c
-rw-rw-r-- 1 monica monica  1504  9月 25 22:58 hello.o
lrwxrwxrwx 1 root     root    15  9月 26 22:28 libfoo.so.1 -> libfoo.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 26 22:27 libfoo.so.1.2.3
-rwxrwxr-x 1 monica monica 15568  9月 26 22:28 libfoo.so.2.3.4
lrwxrwxrwx 1 monica monica    13  9月 25 22:58 libhello.so -> libhello.so.2
lrwxrwxrwx 1 monica monica    17  9月 25 22:58 libhello.so.2 -> libhello.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 25 22:58 libhello.so.2.3.4
-rw-rw-r-- 1 monica monica    49  9月 25 21:36 main.c

# 6.查看 ldconfig 缓存，发现 SONAME -> 此 SONAME 软链接的真实路径
# 左边是可被查找的库名（通常是 SONAME），括号里是这条记录适用的 ABI/架构信息，右边是实际文件路径
# 动态加载器（ld.so）运行时就是按这个缓存把名字解析到具体文件的。
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ldconfig -p | grep foo
	libfoo.so.1 (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libfoo.so.1
```

#### 3.1.2 ldconfig 遇到多个不同 SONAME 的库

对于两个共享库 SONAME 不同的情形，ldconfig 会按各自的 SONAME 建立和更新链接，比如若 **`libfoo.so.1.Y.Z`** 这个共享库族存在多个版本，则 ldconfig 选取最高版本建立 SONAME 软链接 **`libfoo.so.1 -> libfoo.so.1.Y`**；若 **`libfoo.so.2.Y.Z`** 这个共享库族存在多个版本，则 ldconfig 选取最高版本建立 SONAME 软链接 **`libfoo.so.2 -> libfoo.so.2.Y`**。

在 **`/etc/ld.so.cache`** 缓存中会有两条独立的键，运行时需要 **`libfoo.so.1`** 的程序只会装载此软链接指向的共享库；需要 **`libfoo.so.2`** 的程序只会装载此软链接指向的共享库，互不影响。

```shell{.line-numbers}
# 1.创建一个共享库 libfoo.so.1.2.3，其 SONAME 为 libfoo.so.1
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared hello.o -o libfoo.so.1.2.3 -Wl,-soname,libfoo.so.1
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libfoo.so.1.2.3 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libfoo.so.1]

# 2.创建一个共享库 libfoo.so.2.3.4，其 SONAME 也为 libfoo.so.2
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc -shared hello.o -o libfoo.so.2.3.4 -Wl,-soname,libfoo.so.2
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d libfoo.so.2.3.4 | grep soname
 0x000000000000000e (SONAME)             Library soname: [libfoo.so.2]

# 3.使用命令 ldconfig 来更新扫描 /etc/ld.so.conf 以及受信任目录中的共享库 SONAME 链接，并且更新 /etc/ld.so.cache
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ sudo ldconfig

# 4.ls -l 命令的结果显示 ldconfig 按各自的 SONAME 建立和更新链接
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ls -l
总用量 76
-rwxrwxr-x 1 monica monica 15952  9月 25 23:27 app
-rw-rw-r-- 1 monica monica    73  9月 25 21:34 hello.c
-rw-rw-r-- 1 monica monica  1504  9月 25 22:58 hello.o
lrwxrwxrwx 1 root     root        15  9月 26 23:13 libfoo.so.1 -> libfoo.so.1.2.3
-rwxrwxr-x 1 monica monica 15568  9月 26 23:12 libfoo.so.1.2.3
lrwxrwxrwx 1 root     root        15  9月 26 23:13 libfoo.so.2 -> libfoo.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 26 23:12 libfoo.so.2.3.4
lrwxrwxrwx 1 monica monica    13  9月 25 22:58 libhello.so -> libhello.so.2
lrwxrwxrwx 1 monica monica    17  9月 25 22:58 libhello.so.2 -> libhello.so.2.3.4
-rwxrwxr-x 1 monica monica 15568  9月 25 22:58 libhello.so.2.3.4
-rw-rw-r-- 1 monica monica    49  9月 25 21:36 main.c

# 5.查看 ldconfig 缓存，发现有 2 条链
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ ldconfig -p | grep foo
	libfoo.so.2 (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libfoo.so.2
	libfoo.so.1 (libc6,x86-64) => /home/monica/linkers_loaders/tmp/soname-demo/libfoo.so.1
```

### 3.2 共享库的系统路径

目前大多数包括 Linux 在内的开源操作系统都遵守一个叫做 FHS（File Hierarchy Standard）的标准，这个标准规定了一个系统中的系统文件应该如何存放，包括各个目录的结构、组织和作用。FHS 规定，一个系统中主要有两个存放共享库的位置，它们分别如下：

- **`/lib`**：**这个位置主要存放系统最关键和基础的共享库**，比如动态链接器、C 语言运行库、数学库等，这些库主要是那些 **`/bin`** 和 **`/sbin`** 下的程序所需要到的库，还有系统启动时需要的库。
- **`/usr/lib`**：**这个目录下主要保存的是一些非系统运行时所需要的关键性的共享库**，主要是一些开发时用到的共享库，这个目录下面还包含了开发时可能会用到的静态库、目标文件等。
- **`/usr/local/lib`**：这个目录用来放置一些跟操作系统本身并不十分相关的库，主要是一些第三方的应用程序的库。GNU 的标准推荐第三方的程序应该默认将库安装到 **`/usr/local/lib`** 下。

### 3.3 共享库的查找过程

动态链接的 ELF 可执行文件在启动时同时会启动动态链接器。在 Linux 系统中，动态链接器是 **`/lib/ld-linux.so.X`**（X 是版本号），程序所依赖的共享对象全部由动态链接器负责装载和初始化。我们知道任何一个动态链接的模块所依赖的模块路径保存在 **`.dynamic`** 段里面，由 **`DT_NEED`** 类型的项表示。

动态链接器对于模块的查找有一定的规则：如果 **`DT_NEED`** 里面保存的是绝对路径，那么动态链接器就按照这个路径去查找；如果 **`DT_NEED`** 里面保存的是相对路径，那么动态链接器会在 **`/lib`**、**`/usr/lib`** 和由 **`/etc/ld.so.conf`** 配置文件指定的目录中查找共享库。为了程序的可移植性和兼容性，共享库的路径往往是相对的。

**`ld.so.conf`** 是一个文本配置文件，它可能包含其他的配置文件，这些配置文件中存放着目录信息。在我的机器中，**`ld.so.conf`** 指定的目录是：

```shell{.line-numbers}
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ cat /etc/ld.so.conf
include /etc/ld.so.conf.d/*.conf
/home/monica/software/linux_dir/dynamic_lib/lib
/usr/local/lib
```

**`include /etc/ld.so.conf.d/*.conf`** 这一行配置的意思是在解析 **`/etc/ld.so.conf`** 时，**<font color="red">把 **`/etc/ld.so.conf.d/`** 目录里所有匹配 **`*.conf`** 的文件一并纳入（就像把那些文件的内容原地插入到这行位置一样）</font>**。这样做的好处是让系统或软件包把自己的库搜索目录放到独立的"片段文件"里，你安装/卸载包时只需增删对应的 **`.conf`** 文件即可，不必直接改主配置文件。

如果动态链接器在每次查找共享库时都去遍历这些目录，那将会非常耗费时间。所以 Linux 系统中都有一个叫做 ldconfig 的程序，**<font color="red">这个程序的作用是为共享库目录下的各个共享库创建、删除或更新相应的 SO-NAME（即相应的符号链接）</font>**，这样每个共享库的 SO-NAME 就能够指向正确的共享库文件；**<font color="red">并且这个程序还会将这些 SO-NAME 收集起来，集中存放到 **`/etc/ld.so.cache`** 文件里面，并建立一个 SO-NAME 的缓存</font>**。当动态链接器要查找共享库时，它可以直接从 **`/etc/ld.so.cache`** 里面查找。而 **`/etc/ld.so.cache`** 的结构是经过特殊设计的，非常适合查找，所以这个设计大大加快了共享库的查找过程。

如果动态链接器在 **`/etc/ld.so.cache`** 里面没有找到所需要的共享库，那么它还会遍历 **`/lib`** 和 **`/usr/lib`** 这两个目录，如果还是没找到，就宣告失败。

所以理论上讲，如果我们在系统指定的共享库目录下添加、删除或更新任何一个共享库，或者我们更改了 **`/etc/ld.so.conf`** 的配置，都应该运行 ldconfig 这个程序，以便调整 SO-NAME 和 **`/etc/ld.so.cache`**。很多软件包的安装程序在安装共享库以后都会自动调用 ldconfig。

### 3.4 环境变量

当动态链接器加载这个程序时，它会按照如下的顺序去搜索程序依赖的共享库：

- 若二进制含 **`DT_RPATH`** 且没有 **`DT_RUNPATH`**，则先查 RPATH (在可执行文件或调用库中定义的路径)；
- 查找 **`LD_LIBRARY_PATH`** 路径；
- 若含 **`DT_RUNPATH`**，按 **`RUNPATH`** 查找；
- 查找 **`/etc/ld.so.cache`** 缓存；
- 查默认目录，**`/lib*`**，**`/usr/lib*`** 等；

>1).Using the directories specified in the **`DT_RPATH`** dynamic section attribute of the binary if present and **`DT_RUNPATH`** attribute does not exist.
>2).Using the environment variable **`LD_LIBRARY_PATH`**, unless the executable is being run in secure-execution mode (see below), in which case this variable is ignored.
>3).Using the directories specified in the **`DT_RUNPATH`** dynamic section attribute of the binary if present. Such directories are searched only to find those objects required by **`DT_NEEDED`** (direct dependencies) entries and do not apply to those objects' children, which must themselves have their own **`DT_RUNPATH`** entries. This is unlike **`DT_RPATH`**, which is applied to searches for all children in the dependency tree.
>4).From the cache file **`/etc/ld.so.cache`**, which contains a compiled list of candidate shared objects previously found in the augmented library path.
>5).In the default path /lib, and then **`/usr/lib`**. (On some 64-bit architectures, the default paths for 64-bit shared objects are **`/lib64`**, and then **`/usr/lib64`**.).

#### 3.4.1 RPATH/RUNPATH

如果设置了 **`RPATH/RUNPATH`**，在编译链接程序的时候，开发者就直接把库的路径信息内置到了可执行文件 ELF 头部中（**`DT_RPATH/DT_RUNPATH`**）。程序启动时，程序自己就知道到哪个路径中去搜索库，这是一种永久的、内部的配置。

**`RPATH（DT_RPATH）`** 的优先级高于 **`LD_LIBRARY_PATH`**。这意味着，一旦一个程序的 RPATH 被设置，用户就无法使用 **`LD_LIBRARY_PATH`** 来覆盖它，让程序去加载另一个位置的同名库。RPATH 在可执行/库上设置后，既影响本对象的直接依赖，也会传递影响间接依赖。现代系统上若同时存在 RUNPATH，则 RPATH 会被忽略。使用如下命令强制生成 RPATH。**`$ORIGIN`** 变量表示在运行时，**<font color="red">动态链接器会将 **`$ORIGIN`** 动态地替换为可执行文件本身所在的绝对路径</font>**。

```c{.line-numbers}
-Wl,--disable-new-dtags,-rpath,'$ORIGIN/lib'
```

**`RUNPATH（DT_RUNPATH）`** 的优先级低于 **`LD_LIBRARY_PATH`**，只对直接依赖生效（不传递给子依赖）。现代 GNU 链接器默认更倾向生成 **`RUNPATH`**，可以使用如下命令生成 **`RUNPATH`**。

```c{.line-numbers}
// 必要时加 -Wl,--enable-new-dtags 明确开启
-Wl,-rpath,'$ORIGIN/lib'
```

举例如下所示：

```shell{.line-numbers}
# 1.在可执行文件的 ELF 头部条目生成 RPATH
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello  -o app -Wl,--disable-new-dtags,-rpath,'$ORIGIN'
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d app | grep PATH
 0x000000000000000f (RPATH)              Library rpath: [$ORIGIN]

# 2.在可执行文件的 ELF 头部条目生成 RUNPATH
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ gcc main.c -L. -lhello  -o app -Wl,--enable-new-dtags,-rpath,'$ORIGIN'
monica@monica-virtual-machine:~/linkers_loaders/tmp/soname-demo$ readelf -d app | grep PATH
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
```

#### 3.4.2 LD_LIBRARY_PATH

Linux 系统提供了很多方法来改变动态链接器装载共享库路径的方法，其中改变共享库查找路径最简单的方法是使用 **`LD_LIBRARY_PATH`** 环境变量，这个方法可以临时地、优先地告诉系统上的动态链接器/加载器 (ld.so) 在哪些额外的目录中去查找共享库，而不会影响系统中的其他程序。

在 Linux 系统中，**`LD_LIBRARY_PATH`** 是一个由若干个路径组成的环境变量，每个路径之间由冒号隔开。默认情况下，**`LD_LIBRARY_PATH`** 为空。如果我们为某个进程设置了 **`LD_LIBRARY_PATH`**，那么进程在启动时，动态链接器在查找共享库时，会首先查找由 **`LD_LIBRARY_PATH`** 指定的目录。这个环境变量可以很方便地让我们测试新的共享库或使用非标准的共享库。比如我们希望使用修改过的 **`libc.so.6`**，可以将这个新版的 libc 放到我们的目录 **`/home/user`** 中，然后指定 **`LD_LIBRARY_PATH`**：

```c{.line-numbers}
$ LD_LIBRARY_PATH=/home/user /bin/ls
```

#### 3.4.3 LD_PRELOAD

系统中另外还有一个环境变量叫做 **`LD_PRELOAD`**，这个文件中我们可以指定预先装载的一些共享库甚至是目标文件。在 **`LD_PRELOAD`** 里面指定的文件会在动态链接器按照固定规则搜索共享库之前装载。**<font color="red">它比 **`LD_LIBRARY_PATH`** 里面所指定的目录中的共享库还要优先。无论程序是否依赖于它们， **`LD_PRELOAD`** 里面指定的共享库或目标文件都会被装载</font>**。

由于全局符号介入这个机制的存在，**`LD_PRELOAD`** 里面指定的共享库或目标文件中的全局符号会覆盖后面加载的同名全局符号，这使得我们可以很方便地做到改写标准 C 库中的某个或某几个函数而不影响其他函数，对于程序的调试或测试非常有用。与 **`LD_LIBRARY_PATH`** 一样，正常情况下应该尽量避免使用 **`LD_PRELOAD`**，比如一个发布版本的程序运行不应该依赖于 **`LD_PRELOAD`**。

**`LD_LIBRARY_PATH`** 的作用是告诉链接器"去这些目录里找库"，只改变搜索路径。**`LD_PRELOAD`** 的作用是告诉链接器"先把这些库加载进来"（在一切库前预先装入指定 **`.so`**，其导出符号在全局查找时优先匹配），改变装载顺序与符号优先级，可实现函数拦截/替换（interpose）。

## 4.共享库的创建和安装

### 4.1 共享库的创建

