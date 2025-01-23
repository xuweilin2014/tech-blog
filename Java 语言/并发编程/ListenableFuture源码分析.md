# ListenableFuture 源码分析

## 1.简介

JDK 原生的 Future 已经提供了异步操作，但是不能直接回调。guava 对 Future 进行了增强，核心接口就是 ListenableFuture。如果已经开始使用了 jdk8，可以直接学习使用原生的 CompletableFuture，这是 jdk 从 guava 中吸收了精华新增的类。

ListenableFuture 继承了 Future，额外新增了一个方法，Listener 是任务结束后的回调方法，executor 是执行回调方法的执行器( 通常是线程池)。guava 中对 Future 的增强就是在 addListener 这个方法上进行了各种各样的封装，所以 addListener 是核心方法。

```java{.line-numbers}
public interface ListenableFuture<V> extends Future<V> {
    void addListener(Runnable listener, Executor executor);
} 
```

所以说，ListenableFuture 相比于普通的 Future 来说，本质上只是增加了回调函数。

## 2.使用示例

```java{.line-numbers}
public class ListenableFutureTest {
    public static void main(String[] args) {
        testListenFuture();
    }

    public static void testListenFuture() {
        System.out.println("主线程start");
        ListeningExecutorService pool = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(5));

        Task task1 = new Task();
        task1.args = "task1";
        Task task2 = new Task();
        task2.args = "task2";
        ListenableFuture<String> future = pool.submit(task1);
        ListenableFuture<String> future2 = pool.submit(task2);

        future2.addListener(() -> System.out.println("addListener 不能带返回值"), pool);

        /**
         * FutureCallBack接口可以对每个任务的成功或失败单独做出响应
         */
        FutureCallback<String> futureCallback = new FutureCallback<String>() {
            @Override
            public void onSuccess(String result) {
                System.out.println("Futures.addCallback 能带返回值：" + result);
            }
            @Override
            public void onFailure(Throwable t) {
                System.out.println("出错,业务回滚或补偿");
            }
        };

        //为任务绑定回调接口
        Futures.addCallback(future, futureCallback, pool);
        System.out.println("主线程end");
    }
}

class Task implements Callable<String> {
    String args;
    @Override
    public String call() throws Exception {
        Thread.sleep(1000);
        System.out.println("任务：" + args);
        return "dong";
    }
} 
```

在上面的代码中，future2.addListener 和 Futures.addCallback 都是注册回调的方法，在本质上是一样的，对于 Futures.addCallback，也是把 FutureCallback 对象封装成一个 Listener，然后调用 addListener 方法，从而添加到 future 中的 Listener 链表里面。

