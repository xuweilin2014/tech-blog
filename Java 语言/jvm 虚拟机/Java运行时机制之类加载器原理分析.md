# 类加载器原理分析

本文分析了双亲委派模型的实现原理，并通过代码示例说明了什么时候需要实现自己的类加载器以及如何实现自己的类加载器。本文的内容和代码都基于 jdk1.8。

## 1.ClassLoader 的作用

ClassLoader 用于将 class 文件加载到 JVM 中，另外一个作用是确认每个类应该哪一个类加载器加载。第二个作用也用于判断 JVM 运行时的两个类是否相等。影响的判断方法有 **`equals()`**、**`isAssignableFrom()`**、**`isInstance()`** 以及 instanceof 关键字，这一点在后文中会举例说明。类加载的触发可以分为隐式加载和显式加载，其中，隐式加载的情况分为以下 4 种：

1. 遇到 **`new`**、**`getstatic`**、**`putstatic`**、**`invokestatic`** 这 4 条字节码指令时；
2. 对类进行反射调用时；
3. 当初始化一个类时，如果其父类还没有初始化，优先加载其父类并初始化；
4. 虚拟机启动时，需指定一个包含 main 函数的主类，优先加载并初始化这个主类；

显式加载包括以下 3 种情况：

1. 通过 ClassLoader 的 loadClass 方法
2. 通过 Class.forName
3. 通过 ClassLoader 的 findClass 方法

并且，在 JDK1.8 之前加载的类存放在方法区中，而从 JDK8 到现在为止，会加载到元数据区。

## 2.ClassLoader 的种类

整个 JVM 平台提供三类 ClassLoader。分别为：Bootstrap ClassLoader、ExtClassLoader 和 AppClassLoader。

- Bootstrap ClassLoader：加载 JVM 自身工作需要的类，它由 JVM 自己实现。它会加载 **`$JAVA_HOME/jre/lib`** 下的文件
- ExtClassLoader：它是 JVM 的一部分，由 **`sun.misc.LauncherExtClassLoader`** 实现，他会加载 **`JAVA_HOME/jre/lib/ext`** 目录中的文件（或由 **`System.getProperty("java.ext.dirs")`** 所指定的文件）
- AppClassLoader：系统类加载器，我们工作中接触最多的也是这个类加载器，它由 **`sun.misc.Launcher$AppClassLoader`** 实现。它加载 **`System.getProperty("java.class.path")`** 指定目录下的文件，也就是我们通常说的 classpath 路径。

## 3.双亲委派模型

### 3.1 双亲委派模型的原理

从 JDK1.2 之后，类加载器引入了双亲委派模型，其模型图如下：

<div align="center">
    <img src="Java运行时机制_static//13.png" width="450"/>
</div>

其中，两个用户自定义类加载器的父加载器是 AppClassLoader，AppClassLoader 的父加载器是 ExtClassLoader，ExtClassLoader 是没有父类加载器的，在代码中，ExtClassLoader 的父类加载器为 null。BootstrapClassLoader 也并没有子类，因为他完全由 JVM 实现。

双亲委派模型的原理是：当一个类加载器接收到类加载请求时，首先会请求其父类加载器加载，每一层都是如此，当父类加载器无法找到这个类时（根据类的全限定名称），子类加载器才会尝试自己去加载。为了说明这个继承关系，我这里实现了一个自己的类加载器，名为 TestClassLoader，在类加载器中，用 parent 字段来表示当前加载器的父类加载器，其定义如下：

```java{.line-numbers}
public abstract class ClassLoader {
...
    // The parent class loader for delegation
    // Note: VM hardcoded the offset of this field, thus all new fields
    // must be added *after* it.
    private final ClassLoader parent;
...
} 
```

然后通过 debug 来看一下这个结构，如下图：

<div align="center">
    <img src="Java运行时机制_static//14.png" width="550"/>
</div>

这里的第一个红框是我自己定义的类加载器，对应上图的最下层部分；第二个框是自定义类加载器的父类加载器，可以看到是 AppClassLoader；第三个框是 AppClassLoader 的父类加载器，是 ExtClassLaoder；第四个框是 ExtClassLoader 的父类加载器，是 null。

### 3.2 双亲委派模型解决的问题

双亲委派模型是 JDK1.2 之后引入的。根据双亲委派模型原理，可以试想，没有双亲委派模型时，如果用户自己写了一个全限定名为 **`java.lang.Object`** 的类，并用自己的类加载器去加载，同时 BootstrapClassLoader 加载了 **`rt.jar`** 包中的 JDK 本身的 **`java.lang.Object`**，这样内存中就存在两份 Object 类了，此时就会出现很多问题，例如根据全限定名无法定位到具体的类。

有了双亲委派模型后，所有的类加载操作都会优先委派给父类加载器，这样一来，即使用户自定义了一个 **`java.lang.Object`**，但由于 BootstrapClassLoader 已经检测到自己加载了这个类，用户自定义的类加载器就不会再重复加载了。

所以，双亲委派模型能够保证类在内存中的唯一性。

### 3.3 双亲委派模型实现原理

下面从源码的角度看一下双亲委派模型的实现。JVM 在加载一个 class 时会先调用 classloader 的 loadClassInternal 方法，该方法源码如下：

```java{.line-numbers}
// This method is invoked by the virtual machine to load a class.
private Class<?> loadClassInternal(String name)
    throws ClassNotFoundException
{
    // For backward compatibility, explicitly lock on 'this' when
    // the current class loader is not parallel capable.
    if (parallelLockMap == null) {
        synchronized (this) {
             return loadClass(name);
        }
    } else {
        return loadClass(name);
    }
} 
```

该方法里面做的事儿就是调用了 loadClass 方法，loadClass 方法的实现如下：

```java{.line-numbers}
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 先查看这个类是否已经被自己加载了
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 如果有父类加载器，先委派给父类加载器来加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 如果父类加载器为 null，说明 ExtClassLoader 也没有找到目标类，则调用 BootstrapClassLoader 来查找
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            // 如果都没有找到，调用 findClass 方法，尝试自己加载这个类
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
} 
```

源码中已经给出了几个关键步骤的说明。代码中调用 BootstrapClassLoader 的地方实际是调用的 native 方法。由此可见，双亲委派模型实现的核心就是这个 loadClass 方法。

## 4.实现自己的类加载器

为了能够完全掌控类的加载过程，我们的定制类加载器需要直接从 ClassLoader 继承。首先我们来介绍一下 ClassLoader 类中和热替换有关的的一些重要方法。

- **`findLoadedClass`**: 每个类加载器都维护有自己的一份已加载类名字空间，其中不能出现两个同名的类。凡是通过该类加载器加载的类，无论是直接的还是间接的，都保存在自己的名字空间中，该方法就是在该名字空间中寻找指定的类是否已存在，如果存在就返回给类的引用，否则就返回 null。这里的直接是指，存在于该类加载器的加载路径上并由该加载器完成加载，间接是指，由该类加载器把类的加载工作委托给其他类加载器完成类的实际加载。
- **`getSystemClassLoader`**: Java2 中新增的方法。该方法返回系统使用的 ClassLoader。可以在自己定制的类加载器中通过该方法把一部分工作转交给系统类加载器去处理。
- **`defineClass`**: 该方法是 ClassLoader 中非常重要的一个方法，它接收以字节数组表示的类字节码，并把它转换成 Class 实例，该方法转换一个类的同时，会先要求装载该类的父类以及实现的接口类。
- **`loadClass`**: 加载类的入口方法，调用该方法完成类的显式加载。通过对该方法的重新实现，我们可以完全控制和管理类的加载过程。
- **`resolveClass`**: 链接一个指定的类。这是一个在某些情况下确保类可用的必要方法。

