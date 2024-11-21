# Netty 服务端启动流程

## 一、简介

在详细介绍 Netty 服务端的启动流程之前，我们先进行一个简单概括。和使用 JDK NIO 模型进行编程一样，首先服务端初始化一个 NioServerSocketChannel，并将它注册到 bossGroup 中的某一个 NioEventLoop 上（其实是 Selector 上），并且将监听事件设置为 OP_ACCPET，用来监听客户端的连接。当客户端连接上服务端以后，初始化一个 NioSocketChannel，并且将它传递给一个特殊的 ChannelHandler（ServerBootstrapAcceptor）。这个 ChannelHandler 会把 NioSocketChannel 注册到 workerGroup 中的某一个 NioEventLoop 上，并且设置监听事件为 OP_READ。以后，这个 NioEventLoop 负责此 channel 上的 I/O 事件处理。

## 二、示例

本篇文章以下面这个服务端代码为例：

```java{.line-numbers}
public final class SimpleServer {

    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new SimpleServerHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                        }
                    });

            ChannelFuture f = b.bind(8888).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    private static class SimpleServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelActive");
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelRegistered");
        }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            System.out.println("handlerAdded");
        }
    }
} 
```

简单的几行代码就能开启一个服务端，端口绑定在 8888，使用 nio 模式，下面讲下每一个步骤的处理细节。EventLoopGroup 已经是一个死循环，主要包括三个步骤：检测 channel 上的 I/O 事件，处理 I/O 事件，执行任务队列 taskQueue 中的任务。ServerBootstrap 是服务端的一个启动辅助类，通过给他设置一系列参数来绑定端口启动服务。**`group(bossGroup, workerGroup)`** 我们需要两种类型的人干活，一个是老板，一个是工人，老板负责从外面接活，接到的活分配给工人干，放到这里，bossGroup 的作用就是不断地 accept 到新的连接，将新的连接丢给 workerGroup 来处理。

**`.channel(NioServerSocketChannel.class)`** 表示服务端启动的是 nio 相关的 channel，channel 在 netty 里面是一大核心概念，可以理解为一条 channel 就是一个连接或者一个服务端 bind 动作。**`childHandler(new ChannelInitializer<SocketChannel>)...`** 表示一条新的连接进来之后，该怎么处理，也就是上面所说的，老板如何给工人配活。**`ChannelFuture f = b.bind(8888).sync();`** 这里就是真正的启动过程了，绑定 8888 端口，等待服务器启动完毕，才会进入下行代码。

**`f.channel().closeFuture().sync();`** 等待服务端关闭 socket。**`bossGroup.shutdownGracefully(); workerGroup.shutdownGracefully();`** 关闭两组死循环。上述代码可以很轻松地再本地跑起来，最终控制台的输出为：

```java{.line-numbers}
handlerAdded
channelRegistered
channelActive
```

## 三、源代码详解

ServerBootstrap 一系列的参数配置其实就是使用 method chaining（方法链）的方式将启动服务器需要的参数保存到 filed。我们的重点落入到下面这段代码：

```java{.line-numbers}
b.bind(8888).sync();
```

bind 方法是最终启动服务器的入口，接下来跟进去：

```java{.line-numbers}
//class:AbstractBootstrap
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}  
```

通过端口号创建一个 InetSocketAddress，然后继续 bind：

```java{.line-numbers}
//class:AbstractBootstrap
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
} 
```

validate() 验证服务启动需要的必要参数，然后调用 doBind()：

```java{.line-numbers}
/**
 * class:AbstractBootstrap
 * 
 */
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 初始化 NioServerSocketChannel，并将其中的 ServerSocketChannel 注册到NioEventLoop中的 Selector 上
    // 并且如果线程没有启动的话，创建线程并启动
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        //绑定 channel 到 localAddress 上
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        //.......
    }
} 
```

这里，我去掉了细枝末节，让我们专注于核心方法，**<font color="red">其实就两大核心一个是 initAndRegister()，以及 doBind0()</font>**。其实，从方法名上面我们已经可以略窥一二， init 主要负责 channel 的初始化，register 负责 channel 的注册，那么到底要注册到什么呢？联系到 nio 里面轮询器的注册，可能是把某个东西初始化好了之后注册到 selector 上面去，最后 bind，像是在本地绑定端口号，带着这些猜测，我们深入下去：

```java{.line-numbers}
//class:AbstractBootstrap
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        /**
         * 此处的channelFactory实际的类型为ReflectiveChannelFactory，而其newChannel的代码如下：
         * return clazz.getConstructor().newInstance();
         * 也就通过反射，调用channel的构造函数，生成一个新的Channel
         */
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        //......
    }

    // 将创建好的AbstractNioChannel中的ch属性(类型为Selectable Channel)
    // 注册到NioEventLoop中的Selector上，并把AbstractNioChannel作为attachment
    // 另外，如果NioEventLoop没有启动的话，则创建线程并且启动
    ChannelFuture regFuture = config().group().register(channel);
    //......
    return regFuture;
} 
```

我们还是专注于核心代码，抛开边角料，我们看到 initAndRegister() 做了几件事情：

- new 一个 channel
- init 这个 channel
- 将这个 channel register 到某个对象

我们逐步分析这三件事情：

### 1.new 一个 channel

我们首先要搞懂 channel 的定义，netty 官方对 channel 的描述为：A nexus to a network socket or a component which is capable of I/O operations such as read, write, connect, and bind。

这里的 channel，由于是在服务启动的时候创建，我们可以和普通 Socket 编程中的 ServerSocket 对应上，表示服务端绑定的时候经过的一条流水线。我们发现这条 channel 是通过一个 channelFactory new 出来的。这里，我们的 demo 程序调用 channel(channelClass) 方法的时候，将 channelClass 作为 ReflectiveChannelFactory 的构造函数创建出一个 ReflectiveChannelFactory。

```java{.line-numbers}
//class: AbstractBootstrap
//这个函数在b.channel(NioServerSocketChannel.class)的时候被调用，传入一个Class类型的对象
//以这个Class对象为参数，构造一个ReflectiveChannelFactory，然后调用下面的channelFactory方法
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
} 
```

channelFactory 的代码如下：

```java{.line-numbers}
//class: AbstractBootstrap
//传入的参数为ReflectiveChannelFactory，并且把这个ReflectiveChannelFactory赋值给这个
//AbstractBootstrap对象中的channelFactory属性
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    if (channelFactory == null) {
        throw new NullPointerException("channelFactory");
    }
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return (B) this;
} 
```

然后回到本节最开始：

```java{.line-numbers}
channelFactory.newChannel();
```

我们就可以推断出，最终是调用到 ReflectiveChannelFactory.newChannel() 方法，跟进：

```java{.line-numbers}
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
} 
```

看到 clazz.newInstance();，我们明白了，原来是通过反射的方式来创建一个对象，而这个 class 就是我们在 ServerBootstrap 中传入的 NioServerSocketChannel.class。结果，绕了一圈，最终创建 channel 相当于调用默认构造函数 new 出一个 NioServerSocketChannel 对象。接下来我们将重心放到 NioServerSocketChannel 的构造函数：

```java{.line-numbers}
/**
 * class:NioServerSocketChannel
 * 通过newChannel()方法，使用反射来调用NioServerSocketChannel的构造函数，生成channel对象
 * 在这个构造函数中，会继续调用另外一个构造函数，参数为SelectorProvider通过openServerSocketChannel方法
 * 返回的ServerSocketChannel对象
 */
public NioServerSocketChannel() {
    //newSocket(DEFAULT_SELECTOR_PROVIDER)返回一个ServerSocketChannel
    this(newSocket(DEFAULT_SELECTOR_PROVIDER)); 
} 

private static ServerSocketChannel newSocket(SelectorProvider provider) {
    //...
    return provider.openServerSocketChannel();
} 
```

通过 SelectorProvider.openServerSocketChannel() 创建一条 server 端 channel，然后进入到以下方法：

```java{.line-numbers}
//class:NioServerSocketChannel
/**
 * NioServerSocketChannel的继承层次如下：
 * NioServerSocketChannel --> AbstractNioMessageChannel --> AbstractNioChannel。
 * 在这个构造函数中，会调用NioServerSocketChannel父类AbstractNioChannel的构造函数
 * 具体来说，就是把新创建好的ServerSocketChannel赋值给AbstractNioChannel的ch属性，以及在readInterestOp
 * 属性中，保存ServerSocketChannel感兴趣的监听事件，此处为OP_ACCEPT事件
 */
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
} 
```

这里第一行代码就跑到父类里面去了，第二行，new 出来一个 NioServerSocketChannelConfig，其顶层接口为 ChannelConfig，基本可以判定，ChannelConfig 也是 netty 里面的一大核心模块，初次看源码，看到这里，我们大可不必深挖这个对象，而是在用到的时候再回来深究，只要记住，**<font color="red">这个对象在创建 NioServerSocketChannel 对象的时候被创建即可</font>**。我们继续追踪到 NioServerSocketChannel 的父类 AbstractNioChannel：

```java{.line-numbers}
/**
 * class:AbstractNioChannel
 * AbstractNioChannel类的构造函数，这个函数将参数中的ch(SelectableChannel类型)保存到ch属性中，同样的，
 * 调用ch.configureBlocking(false); 设置该channel为非阻塞模式，标准的jdk nio编程的玩法
 */
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // 这个super调用AbstractChannel的构造函数，就是下面那个
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Failed to close a partially initialized socket.", e2);
            }
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
} 
```

这里，简单地将前面 provider.openServerSocketChannel(); 创建出来的 ServerSocketChannel 保存到成员变量，然后调用 ch.configureBlocking(false); 设置该 channel 为非阻塞模式，标准的 jdk nio 编程的玩法。这里的 readInterestOp 即前面层层传入的 SelectionKey.OP_ACCEPT，接下来重点分析 super(parent);(这里的 parent 其实是 null，由前面写死传入)：

```java{.line-numbers}
/**
 * class:AbstractChannel
 * 初始化三大组件：id(为每一个Channel的标识)、unsafe、pipeline
 */
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
} 
```

在这里，又创建了属于这个 channel 的三个组件：id（netty 中每条 channel 的唯一标识），unsafe 和 pipeline。AbstractChannel 是 AbstractNioChannel 的父类，所以 AbstractNioChannel 保存了和 NIO 相关的属性，比如 NioServerSocketChannel 保存到 ch 属性中，readInterestOps 保存 channel 感兴趣的事件以及对ch设置非阻塞模式。而 AbstractChannel 保存了和 channel 相关的更加基本的组件，比如 pipeline、unsafe 等，这些在 NIO 或者 OIO 类型的 channel 中都能够用到，所以保存在 AbstractChannel。

pipeline 的创建代码如下：

```java{.line-numbers}
pipeline = newChannelPipeline();

protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}

protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
} 
```

初次看这段代码，可能并不知道 DefaultChannelPipeline 是干嘛用的，我们仍然使用上面的方式，查看顶层接口ChannelPipeline的定义：A list of ChannelHandlers which handles or intercepts inbound events and outbound operations of a Channel。

到了这里，我们总算把一个服务端channel创建完毕了，将这些细节串起来的时候，我们顺带提取出netty的几大基本组件，先总结如下:Channel、ChannelConfig、ChannelId、Unsafe
Pipeline、ChannelHander。总结一下，用户调用方法 Bootstrap.bind(port) 第一步就是通过反射的方式 new 一个 NioServerSocketChannel 对象，并且在 new 的过程中创建了一系列的核心组件，仅此而已，并无他，真正的启动我们还需要继续跟。

### 2.init 这个 Channel

第一步 newChannel 完毕，这里就对这个 channel 做 init，init 方法具体干啥，我们深入：

```java{.line-numbers}
/**
 * class:AbstractBootstrap
 */
@Override
void init(Channel channel) throws Exception {
    // 获取启动器 启动时配置的option参数，主要是TCP的一些属性
    final Map<ChannelOption<?>, Object> options = options0();
    // 将获得到 options 配置到 ChannelConfig 中去
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    // 获取 ServerBootstrap 启动时配置的 attr 参数
    final Map<AttributeKey<?>, Object> attrs = attrs0();
    // 配置 Channel attr，主要是设置用户自定义的一些参数
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    // 获取channel中的 pipeline，这个pipeline使我们前面在channel创建过程中设置的 pipeline
    ChannelPipeline p = channel.pipeline();

    // 将启动器中配置的 childGroup 保存到局部变量 currentChildGroup
    final EventLoopGroup currentChildGroup = childGroup;
    // 将启动器中配置的 childHandler 保存到局部变量 currentChildHandler
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    // 保存用户设置的 childOptions 到局部变量 currentChildOptions
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    // 保存用户设置的 childAttrs 到局部变量 currentChildAttrs
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    // p.addLast()向serverChannel的流水线处理器中加入了一个 ServerBootstrapAcceptor
    // 这是一个接入器，专门接受新请求，把新的请求扔给某个事件循环器
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
} 
```

总结以上代码的步骤如下：

1.设置 option 和 attr
2.设置新接入 channel 的 option 和 attr
3.加入新连接的处理器，到了最后一步，p.addLast() 向 serverChannel 的流水线处理器中加入了一个 ServerBootstrapAcceptor，从名字上就可以看出来，这是一个接入器，专门接受新请求，把新的请求注册到某个事件循环器上，由这个事件循环器来处理这个请求连接上的 I/O 事件。

总结一下，我们发现其实 init 也没有启动服务，只是初始化了一些基本的配置和属性，以及在 pipeline 上加入了一个接入器，用来专门接受新连接，我们还得继续往下跟。
 
### 3.将这个 Channel 注册到 NioEventLoop 上

在这一步，我们分析如下的方法：

```java{.line-numbers}
ChannelFuture regFuture = config().group().register(channel);
```

**`group().register(channel)`** 这段代码会通过选择器，从 bossGroup 中（也就是 NioEventLoopGroup 中）选择一个 NioEventLoop，从而把获取到的 NioServerSocketChannel 注册到选择到的 NioEventLoop 调用到 NioEventLoop 中的 register：

```java{.line-numbers}
//class:SingleThreadEventLoop
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
} 

//class:SingleThreadEventLoop
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
} 

//class:AbstractChannel$AbstractUnsafe
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    //省略代码......
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            //省略代码......
        }
    }
} 
```

上面调用 AbstractChannel 中的内部类 AbstractUnsafe 的 register 方法，它会将 NioServerSocketChannel 注册到的 NioEventLoop 赋值给 NioServerSocketChannel 中的 eventLoop 属性，从而可以通过 channel 来获取到 NioEventLoop，接着调用 register0 方法。

```java{.line-numbers}
// class:AbstractChannel$AbstractUnsafe
private void register0(ChannelPromise promise) {
    try {
        // 省略代码......
        boolean firstRegistration = neverRegistered;
        // 将NioSocketChannel中的ch属性(也就是SocketChannel)真正注册到selector上
        doRegister();
        neverRegistered = false;
        registered = true;

        // 前面doRegister注册完之后，调用invokeHandlerAddedIfNeeded，它最终会调用到下面 ChannelInitializer
        // 的 handlerAdded 方法，并最终会调用用户自己创建ChannelInitializer类中initChannel方法，调用之后
        // 再把ChannelInitializer从pipeline中删除。
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);

        // 调用一下业务pipeline中每个处理器的 channelRegistered 方法
        pipeline.fireChannelRegistered();

        // isActive在连接已经建立的条件下，返回true
        if (isActive()) {
            if (firstRegistration) {
                // fireChannelActive会将调用DefaultChannelPipeline中的fireChannelActive方法，
                // 如果用户往pipeline中添加过ChannelHandler，并且自己重写了其中的channelActive方法，
                // 那么此channelActive方法就会被调用
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // 省略代码......
    }
} 
```

上面的 doRegister 是真正地将 NioServerSocketChannel 注册到 NioEventLoop 中的 Selector 上（实际上是 NioServerSocketChannel 中的 SelectableChannel）。而最重要的是下面的代码段：

```java{.line-numbers}
pipeline.invokeHandlerAddedIfNeeded();
```

首先，在 init(channel) 方法中，有如下的代码：

```java{.line-numbers}
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
}); 
```

在上面的代码中，将 ChannelInitializer（一个特殊的 ChannelHandler）添加到 NioServerSocketChannel 的 pipeline 中。在 pipeline.addLast(ChannelHandler) 方法中，当把 ChannelHandler 添加到 pipeline 后，一般会回调 ChannelHandler 中的 handlerAdded 方法，不过前提是此时 NioServerSocketChannel 已经注册到 Selector 上，如果没有，则会将 ctx 包装成一个 PendingHandlerAddedTask，然后当 NioServerSocketChannel 在 register0 中完成注册后被执行（也就是 channel 注册之后再回调 ChannelHandler 的 handlerAdded）。

所以当把 ChannelInitializer 添加到 pipeline 后没有回调 handlerAdded 方法，而 NioServerSocketChannel 通过 register0 被注册到 selector 上之后，通过 invokeHandlerAddedIfNeeded 方法才调用了 ChannelInitializer 中的 handlerAdded 方法，最后调用 initChannel 方法，把 handler（如果 handler 通过 .handler 方法往里面赋值了的）和 ServerBootstrapAcceptor 添加到 pipeline 中（此时 NioServerSocketChannel 中的 registered 为 true，所以这些 handler 中的 handlerAdded 方法会被直接回调），同时把自己从 pipeline 中移除。ServerBootstrapAcceptor 专门用来处理连接到服务期端的客户端连接。

接着回到上面的 register0 方法，这个方法的逻辑也很清晰，先调用 doRegister();，使用 jdk 的 nio 方法把 channel 真正注册到 selector 上，然后调用 invokeHandlerAddedIfNeeded(), 于是乎，控制台第一行打印出来的就是：

```java{.line-numbers}
handlerAdded
```

具体的原因上面解释过，就是 NioServerSocketChannel 注册到 Selector 上之后，会调用 initChannel 方法把 .handler 方法中添加的 SimpleServerHandler 增加到 pipeline 中，并且由于 NioServerSocketChannel 已经注册了，所以会直接回调 SimpleServerHandler 的 handlerAdded 方法。

然后调用 pipeline.fireChannelRegistered(); 调用之后，控制台的显示为：

```java{.line-numbers}
handlerAdded
channelRegistered 
```

继续往下跟：

```java{.line-numbers}
if (isActive()) {
    if (firstRegistration) {
        pipeline.fireChannelActive();
    } else if (config().isAutoRead()) {
        beginRead();
    }
} 
```

读到这，你可能会想当然地以为，控制台最后一行是由以下代码触发打印的：

```java{.line-numbers}
pipeline.fireChannelActive();
```

由这行代码输出，我们不妨先看一下 isActive() 方法：

```java{.line-numbers}
@Override
public boolean isActive() {
    return javaChannel().socket().isBound();
}
```

这里 isBound() 返回 false，这是因为从目前我们跟下来的流程看，我们并没有将一个 NioServerSocketChannel 绑定到一个 address，所以 isActive() 返回 false，我们没有成功进入到 pipeline.fireChannelActive() 方法。那么 pipeline.fireChannelActive() 是从哪儿发起调用的？其实是从 doBind0 方法开始：

```java{.line-numbers}
//class:AbstractBootstrap
private static void doBind0(
        final ChannelFuture regFuture, final ChannelFuture channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                //调用下面AbstractChannel中的bind方法，将channel绑定到localAddress上面
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
} 
```

我们发现，在调用 doBind0(...) 方法的时候，是通过包装一个 Runnable 进行异步化的，好，接下来我们进入到 channel.bind() 方法：

```java{.line-numbers}
//class:AbstractChannel
@Override
public ChannelFuture bind(SocketAddress localAddress) {
    return pipeline.bind(localAddress);
} 
```

发现是调用 pipeline 的 bind 方法：

```java{.line-numbers}
@Override
public final ChannelFuture bind(SocketAddress localAddress) {
    return tail.bind(localAddress);
} 
```

最后一直调用到 AbstractUnsafe, 中的 bind 方法，准确点，应该是 NioMessageUnsafe。我们进入到它的 bind 方法：

```java{.line-numbers}
//class:AbstractChannel
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();
    //......
    boolean wasActive = isActive();
    try {
        //NioSevrerSocketChannel#doBind方法会调用jdk的api来将ServerSocketChannel真正绑定到localAddress上
        doBind(localAddress);
    } catch (Throwable t) {
        //.....
    }

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
} 
```

显然按照正常流程，我们前面已经分析到 isActive(); 方法返回 false，进入到 doBind() 之后，如果 channel 被正常绑定到某个端口，isActive() 返回 true。接着发起 **`pipeline.fireChannelActive();`** 调用，最终调用到用户方法，在控制台打印出了最后一行，所以到了这里，你应该清楚为什么最终会在控制台按顺序打印出那三行字了吧。doBind() 方法也很简单：

```java{.line-numbers}
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        //noinspection Since15
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
} 
```


最终调到了 jdk 里面的 bind 方法，这行代码过后，正常情况下，就真正进行了端口的绑定。另外，通过自顶向下的方式分析，在调用 **`pipeline.fireChannelActive();`** 方法的时候，会调用到如下方法：

```java{.line-numbers}
//class:DefaultChannelPipeline$HeadContext
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    ctx.fireChannelActive();
    readIfIsAutoRead();
} 

   //class:DefaultChannelPipeline$HeadContext
private void readIfIsAutoRead() {
    if (channel.config().isAutoRead()) {
        channel.read();
    }
} 
```

isAutoRead 方法默认返回 true，于是进入到以下方法：

```java{.line-numbers}
public Channel read() {
    pipeline.read();
    return this;
} 
```

最终调用到：

```java{.line-numbers}
//class:AbstractNioChannel.AbstractNioUnsafe
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    // 这里的interestOps在我们先前调用doRegister对NioServerSocketChannel进行注册时设置的为0。
    // 而 readInterestOp 设置为 OP_ACCEPT，具体的是在channelFactory通过反射调用NioServerSocketChannel构造函数时赋值，
    // public NioServerSocketChannel(ServerSocketChannel channel) {
    // super(null, channel, SelectionKey.OP_ACCEPT);
    // config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    // }
    // 也就是将 NioServerSocketChannel 的 interestedOps 设置为 OP_ACCEPT，接收新的连接

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
} 
```

所以对 NioServerSocketChannel 添加一个 OP_ACCEPT 事件，让 NioEventLoop 能够监听客户端与服务器端的连接（这是因为最开始 doRegister 注册的时候，将 interestOps 设置为 0，即不监听任何事件）。

#### 4.总结

首先，服务端会先创建一个 NioServerSocketChannel，然后从 bossGroup 中选择一个 NioEventLoop，将 NioServerSocketChannel 注册到 NioEventLoop 上，同时会将 ServerBootstrapAcceptor 添加到 NioServerSocketChannel 中的 pipeline 中，这是一个特殊的 ChannelHandler，它主要用来处理新连接到服务端的连接请求。同时，如果在启动代码中调用了 .handler 添加了 ChannelHandler，那么它也会被添加进 pipeline 中（不过很少使用）。

NioServerSocketChannel 注册完成之后，会将其绑定到程序所设定的端口上，同时会设置 Selector 监听 NioServerSocketChannel 上的 OP_ACCEPT 事件（最开始注册的时候设置的为 0，也就是不监听任何事件）。