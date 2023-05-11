# 线程

### 1.介绍线程

线程是独立的处理流程，可以和系统的其他线程并行或并发地执行。多线程可以共享数据和资源，利用所谓的共享内存空间。线程和进程的具体实现取决于你要运行的操作系统，但是总体来讲，我们可以说线程是包含在进程中的，同一进程的多个不同的线程可以共享相同的资源。相比而言，进程之间不会共享资源。

每一个线程基本上包含3个元素：程序计数器，寄存器和栈。与同一进程的其他线程共享的资源基本上包括数据和系统资源。每一个线程也有自己的运行状态，可以和其他线程同步，这点和进程一样。线程的状态大体上可以分为 ready，running，blocked。**线程的典型应用是应用软件的并行化——为了充分利用现代的多核处理器，使每个核心可以运行单个线程**。相比于进程，使用线程的优势主要是性能。相比之下，在进程之间切换上下文要比在统一进程的多线程之间切换上下文要重的多。

**多线程编程一般使用共享内容空间进行线程间的通讯。这就使管理内容空间成为多线程编程的重点和难点**。

### 2.如何定义一个线程

使用线程最简单的一个方法是，用一个目标函数实例化一个 Thread 然后调用 start() 方法启动它。Python 的 threading 模块提供了 Thread() 方法在不同的线程中运行函数或处理过程等。

```python
class threading.Thread(group=None, target=None, name=None, args=(), kwargs={})
```

上面的代码中：

- group: 一般设置为 `None`，这是为以后的一些特性预留的
- target: 当线程启动的时候要执行的函数
- name: 线程的名字，默认会分配一个唯一名字 Thread-N
- args: 传递给 target 的参数，要使用tuple类型
- kwargs: 同上，使用字典类型dict

创建线程的方法非常实用，通过`target`参数、`arg`和`kwarg`告诉线程应该做什么。下面这个例子传递一个数字给线程（这个数字正好等于线程号码），目标函数会打印出这个数字。

让我们看一下如何通过 threading 模块创建线程，只需要几行代码就可以了：

```python{.line-numbers}
import threading

def function(i):
    print ("function called by thread %i\n" % i)
    return

threads = []

for i in range(5):
    t = threading.Thread(target=function , args=(i, ))
    threads.append(t)
    t.start()
    t.join()
```

因为写了 t.join() ，这意味着，t 线程结束之前并不会看到后续的线程，换句话说，主线程会调用 t 线程，然后等待 t 线程完成再执行 for 循环开启下一个 t 线程，事实上，这段代码是顺序运行的，实际运行顺序永远是 01234 顺序出现。

线程被创建之后并不会马上运行，需要手动调用 start() ， join() 让调用它的线程一直等待直到执行结束（即阻塞调用它的主线程， t 线程执行结束，主线程才会继续执行）。

```python
t.start()
t.join()
```

### 3 线程命名

每一个 Thread 实例创建的时候都有一个带默认值的名字，并且可以修改。在服务端通常一个服务进程都有多个线程服务，负责不同的操作，这时候命名线程是很实用的。为了演示如何确定正在运行的线程，我们创建了三个目标函数，并且引入了 time 在运行期间挂起 2s，让结果更明显。

```python{.line-numbers}
import threading
import time

def first_function():
    print(threading.currentThread().getName() + str(' is Starting '))
    time.sleep(2)
    print (threading.currentThread().getName() + str(' is Exiting '))
    return

def second_function():
    print(threading.currentThread().getName() + str(' is Starting '))
    time.sleep(2)
    print (threading.currentThread().getName() + str(' is Exiting '))
    return

def third_function():
    print(threading.currentThread().getName() + str(' is Starting '))
    time.sleep(2)
    print(threading.currentThread().getName() + str(' is Exiting '))
    return

if __name__ == "__main__":
    t1 = threading.Thread(name='first_function', target=first_function)
    t2 = threading.Thread(name='second_function', target=second_function)
    t3 = threading.Thread(name='third_function', target=third_function)
    t1.start()
    t2.start()
    t3.start()
```

输出如下图所示：

<div align="center">
    <img src="Python线程同步_static/1.png" width="500"/>
</div>

我们使用目标函数实例化线程。同时，我们传入 name 参数作为线程的名字，如果不传这个参数，将使用默认的参数：

```python
t1 = threading.Thread(name='first_function', target=first_function)
t2 = threading.Thread(name='second_function', target=second_function)
t3 = threading.Thread(target=third_function)
```

如果改为这里的代码，那么线程3将会输出的是 `Thread-1 is Starting` 以及 `Thread-1 is Exiting`

