# Callable 和 Future 接口简介

## 1.Callable 和 Future 出现的原因

创建线程的2种方式，一种是直接继承 Thread，重写 run 方法；另外一种就是实现 Runnable 接口。这 2 种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。自从 Java 1.5 开始，就提供了 Callable 和 Future，通过它们联合使用，就可以在任务执行完毕之后得到任务执行结果。

Callable 接口代表一段可以调用并返回结果的代码; Future 接口表示异步任务，表示还没有完成的任务。所以说 Callable 用于产生结果，Future 用于获取结果。Callable 接口使用泛型去定义它的返回类型。Executors 类提供了一些有用的方法在线程池中执行 Callable 内的任务。由于 Callable 任务是并行的（并行就是整体看上去是并行的，其实在某个时间点只有一个线程在执行），我们必须等待它返回的结果。**`java.util.concurrent.Future`** 对象为我们解决了这个问题。**<font color="red">在线程池提交 Callable 任务后返回了一个 Future 对象，使用它可以知道 Callable 任务的状态和得到 Callable 返回的执行结果</font>**。Future 提供了 get() 方法让我们可以等待 Callable 结束并获取它的执行结果。

## 2.Callable 和 Runnable

**`java.lang.Runnable`** 它是一个接口，在它里面只声明了一个 **`run()`** 方法：

```java{.line-numbers}
public interface Runnable {
    public abstract void run();
} 
```

由于 **`run()`** 方法返回值为 void 类型，所以在执行完任务之后无法返回任何结果。Callable 位于 **`java.util.concurrent`** 包下，它也是一个接口，在它里面也只声明了一个方法，只不过这个方法叫做 **`call()`**：

```java{.line-numbers}
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
} 
```

这是一个泛型接口，**`call()`** 函数返回的类型就是传递进来的V类型。

## 3.Callable 的使用

Runnable 对象可以传入到 Thread 类的构造方法中，通过 Thread 来运行 Runnable 任务，而 Callable 接口则不能直接传入到 Thread 中来运行，Callable 接口通常结合线程池来使用。线程池 ThreadPoolExecutor 中除了提供 **`execute(Runnable)`** 方法来提交任务以外（**`execute(Runnable)`** 方法的返回值为 void），还提供了 **`submit()`** 的三个重载方法来提交任务，这三个方法均有返回值。 ThreadPoolExecutor 类继承了抽象类 AbstractExecutorService，在 AbstractExecutorService 中定义了 **`submit()`** 重载的三个方法。

```java{.line-numbers}
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task); 
```

可以看到，submit() 的三个重载方法的返回值均是Future类型的对象。第一个 submit 方法里面的参数类型就是 Callable。暂时只需要知道 Callable 一般是和线程池配合来使用的，具体的使用方法讲在后面讲述。**<font color="red">一般情况下我们使用第一个 submit 方法和第三个 submit 方法，第二个 submit 方法很少使用</font>**。由于 Callable 接口是在 JDK1.5 才引入的，而 Runnable 接口在 JDK1.0 就有了，我们有时候需要将一个已经存在 Runnable 对象转换成 Callable 对象，Executors 工具类为我们提供了这一实现：

```java{.line-numbers}
public class Executors {
    /**
     * Returns a {@link Callable} object that, when
     * called, runs the given task and returns the given result.  This
     * can be useful when applying methods requiring a
     * {@code Callable} to an otherwise resultless action.
     * @param task the task to run
     * @param result the result to return
     * @param <T> the type of the result
     * @return a callable object
     * @throws NullPointerException if task null
     */
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    
    /**
     * A callable that runs given task and returns given result
     */
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
} 
```

可以明显看出来，这个方法采用了设计模式中的适配器模式，将一个 Runnable 类型对象适配成 Callable 类型。因为 Runnable 接口没有返回值, 所以为了与 Callable 兼容, 我们额外传入了一个 result 参数, 使得返回的 Callable 对象的 call 方法直接执行 Runnable 的 run 方法, 然后返回传入的 result 参数。

## 4.Future 

Future 就是对于具体的 Runnable 或者 Callable 任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过 get 方法获取执行结果，该方法会阻塞直到任务返回结果。Future 类位于 **`java.util.concurrent`** 包下，它是一个接口：

```java{.line-numbers}
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
} 
```

在 Future 接口中声明了 5 个方法，下面依次解释每个方法的作用:

- **`cancel`** 方法用来取消一个任务的执行，它的返回值是 boolean 类型，表示取消操作是否成功。
- **`isCancelled`** 方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- **`isDone`** 方法表示任务是否已经完成，若任务完成，则返回 true。注意, 这里的任务结束包含了以下三种情况：
  - 任务正常执行完毕
  - 任务抛出了异常
  - 任务已经被取消
- **`get()`** 方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回。
- **`get(long timeout, TimeUnit unit)`** 用来获取执行结果，如果在指定时间内，还没获取到结果，就则会抛出 TimeoutException 异常。

Future 提供了三种功能：

1. 判断任务是否完成；
2. 能够中断任务；
3. 能够获取任务执行结果。

因为 Future 只是一个接口，所以是无法直接用来创建对象使用的，因此就有了下面的 FutureTask。

## 5.使用示例

### 5.1.使用 Callable+Future 获取执行结果

```java{.line-numbers}
public class Test {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        Future<Integer> result = executor.submit(task);
        executor.shutdown();
         
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
         
        System.out.println("主线程在执行任务");
         
        try {
            System.out.println("task运行结果"+result.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
         
        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
} 
```

执行结果如下：

```java{.line-numbers}
子线程在进行计算
主线程在执行任务
task 运行结果 4950
所有任务执行完毕 
```

### 5.2 使用 Callable+FutureTask 获取执行结果

```java{.line-numbers}
public class Test {
    public static void main(String[] args) {
        // 第一种方式
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executor.submit(futureTask);
        executor.shutdown();
         
        // 第二种方式，注意这种方式和第一种方式效果是类似的，只不过一个使用的是 ExecutorService，一个使用的是 Thread
        /*Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        Thread thread = new Thread(futureTask);
        thread.start();*/
         
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
         
        System.out.println("主线程在执行任务");
         
        try {
            System.out.println("task运行结果"+futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
         
        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
} 
```