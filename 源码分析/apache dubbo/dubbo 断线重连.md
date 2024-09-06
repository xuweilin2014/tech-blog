# Dubbo 断线重连

在 dubbo 2.6.0 版本中，客户端还具有一个断线重连的功能，在前面心跳机制的代码中，客户端与服务器通过是否接收到对方发送的心跳包或者其它数据，来判断连接有否有效或者存活，如果超过一定时间没有接收到的话，就会进行重连操作。

在创建 NettyClient，并且连接服务器的过程中，还会创建一个重连线程（reconnect thread）。它会定时检测 channel 是否出于连接状态（代码层面，通过 SocketChannelImpl 中 state 变量的值来判断是否连接）。如果连接出于断开状态，则会调用 doConnect 进行重连操作，当重连次数达到一定值还没有成功时，就会打印警告和错误日志。

心跳和重连线程可以看作是相互补充，其中重连线程通过代码层面的判断 channel 是否出于连接状态（不过比如直接拔网线等行为，应用层是接收不到这个信号的，所以 Netty 框架还会认为这个链接是正常的，从而产生错误），而心跳则是通过是否真正接收到对方发来的数据来判断。并且重连线程进行连接状态判断和重连操作的时间间隔为 2 s。

断线重连是客户端的功能，具体是在 AbstractClient 和 NettyClient 中实现，AbstractClient 的代码如下：

```java{.line-numbers}
// AbstractClient 的子类有 GrizzlyClient，MinaClient，NettyClient，NettyClient（分别是 Netty3 和 Netty4）
public abstract class AbstractClient extends AbstractEndpoint implements Client {

    private final boolean send_reconnect;

    private final AtomicInteger reconnect_count = new AtomicInteger(0);

    // Reconnection error log has been called before?
    // 重连警告是否已经被写入到日志中
    private final AtomicBoolean reconnect_error_log_flag = new AtomicBoolean(false);

    private final long shutdown_timeout;

    private final int reconnect_warning_period;

    private long lastConnectedTime = System.currentTimeMillis();

    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);

        /**
         * 客户端需要提供重连机制，所以初始化的几个参数都和重连有关：
         */

        // send_reconnect 表示在发送消息时如果发现连接已经断开是否发起重连，默认为 false
        send_reconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, false);
        // shutdown_timeout 表示连接服务器一直连接不上的超时时间
        shutdown_timeout = url.getParameter(Constants.SHUTDOWN_TIMEOUT_KEY, Constants.DEFAULT_SHUTDOWN_TIMEOUT);
        // The default reconnection interval is 2s, 1800 means warning interval is 1 hour.
        // reconnect_warning_period 表示经过多少次重连尝试之后报一次重连警告
        reconnect_warning_period = url.getParameter("reconnect.waring.period", 1800);

        try {
            doOpen();
        } catch (Throwable t) {
            // 省略代码....
        }
        try {
            // connect.
            connect();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " " + NetUtils.getLocalAddress() + " connect to the server " + getRemoteAddress());
            }
        } catch (RemotingException t) {
            // 省略代码....
        } catch (Throwable t) {
            // 省略代码....
        }

        executor = (ExecutorService) ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension().get(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
        ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension().remove(Constants.CONSUMER_SIDE, Integer.toString(url.getPort()));
    }

    protected void connect() throws RemotingException {
        connectLock.lock();
        try {
            // 判断是否已经连接
            if (isConnected()) {
                return;
            }
            // 初始化连接状态检查器，定期检查 channel 是否连接，连接断开会进行重连操作
            initConnectStatusCheckCommand();
            doConnect();
            if (!isConnected()) {
                throw new RemotingException(this, "Failed connect to server ....");
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info("Successed connect to server ...." );
                }
            }
            // 连接建立好或者重新建立好之后，将 reconnect_count（表示重连次数）以及 reconnect_error_log_flag（是否已经记录了重连警告信息）
            // 重置为初始值 0 和 false
            reconnect_count.set(0);
            reconnect_error_log_flag.set(false);
        } catch (RemotingException e) {
            throw e;
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed connect to server " + getRemoteAddress() + " from " + getClass().getSimpleName() + " "
                    + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion()
                    + ", cause: " + e.getMessage(), e);
        } finally {
            connectLock.unlock();
        }
    }

    // 创建了一个Runnable，用来检测是否连接，如果连接断开，调用connect方法；定时调度交给ScheduledThreadPoolExecutor来执行
    private synchronized void initConnectStatusCheckCommand() {
        // reconnect=false to close reconnect
        // 获取到<dubbo:reference/>中的reconnect参数。参数的值可以为true/false，也可以为进行重连接操作的间隔。reconnect参数如果为false，将其设置为0，
        // 表示进行重连接操作，reconnect如果为true，就将其设置为 2000ms，或者用户自己设置的时间间隔数
        int reconnect = getReconnectParam(getUrl());

        // reconnectExecutorFuture == null || reconnectExecutorFuture.isCancelled() 这个判断条件保证了当再一次调用 connect 方法时，
        // 如果定时任务已经开始了并且没有被取消，那么就不再重复创建定时任务
        if (reconnect > 0 && (reconnectExecutorFuture == null || reconnectExecutorFuture.isCancelled())) {
            Runnable connectStatusCheckCommand = new Runnable() {
                public void run() {
                    try {
                        if (!isConnected()) {
                            connect();
                        } else {
                            lastConnectedTime = System.currentTimeMillis();
                        }
                    } catch (Throwable t) {
                        String errorMsg = "client reconnect to " + getUrl().getAddress() + " find error . url: " + getUrl();
                        // wait registry sync provider list
                        // shutdown_timeout 表示服务器一直连不上的超时时间，如果距离上次连上的时间间隔（lastConnectedTime）超过
                        // 了 shutdown_timeout，且还没有在日志中记录重连警告，那么就在日志里面进行记录
                        if (System.currentTimeMillis() - lastConnectedTime > shutdown_timeout) {
                            if (!reconnect_error_log_flag.get()) {
                                reconnect_error_log_flag.set(true);
                                logger.error(errorMsg, t);
                                return;
                            }
                        }
                        // 重连次数达到 reconnect_warning_period 或者其整数倍之后，才会报一次重连警告
                        if (reconnect_count.getAndIncrement() % reconnect_warning_period == 0) {
                            logger.warn(errorMsg, t);
                        }
                    }
                }
            };
            // 创建好定时任务之后，就交给 ScheduledThreadPoolExecutor 来执行
            reconnectExecutorFuture = reconnectExecutorService.scheduleWithFixedDelay(connectStatusCheckCommand, reconnect, reconnect, TimeUnit.MILLISECONDS);
        }
    }

    public void send(Object message, boolean sent) throws RemotingException {
        // 在发送时，如果没有连接，那么就会调用AbstractClient中的connect方法进行连接
        if (send_reconnect && !isConnected()) {
            connect();
        }

        // 获取的并不是网络通讯框架的channel，比如Netty中的原生channel，而是Dubbo对这个channel的封装类对象，比如NettyChannel
        Channel channel = getChannel();
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }

        // 调用 NettyChannel 的 send 方法
        channel.send(message, sent);
    }

    protected abstract Channel getChannel();

} 
```

NettyClient 的代码如下：

```java{.line-numbers}
public class NettyClient extends AbstractClient {

    private volatile Channel channel;

    private static final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup(Constants.DEFAULT_IO_THREADS, new DefaultThreadFactory("NettyClientWorker", true));

    private Bootstrap bootstrap;

    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        super(url, wrapChannelHandler(url, handler));
    }

    // 设置Netty客户端的一些启动参数
    @Override
    protected void doOpen() throws Throwable {
        NettyHelper.setNettyLoggerFactory();
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        if (getTimeout() < 3000) {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
        } else {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout());
        }

        bootstrap.handler(new ChannelInitializer() {
            protected void initChannel(Channel ch) throws Exception {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline().addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("handler", nettyClientHandler);
            }
        });
    }

    // 和服务端建立新连接，并且替换掉旧的连接
    // 替换掉就得连接的原因是在 AbstractClient 的 connect 代码中，会添加一个定时任务主动检测客户端到服务端的连接是否已经断开，如果断开则会进行重连
    // 所以，doConnect 有可能会被反复调用，因此在进行重连的时候，必须要替换掉之前的 channel
    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis();
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        try {
            boolean ret = future.awaitUninterruptibly(3000, TimeUnit.MILLISECONDS);
            // 如果连接建立成功的话，若存在旧的连接，那么就将旧的连接关闭掉。
            // 但是若客户端已经关闭了的话，就把新建立的连接也关闭掉，并且把 NettyClient.this.channel 置为null
            // 若客户端仍然正常，那么就将新连接保存到 NettyClient.this.channel 中
            if (ret && future.isSuccess()) {
                Channel newChannel = future.channel();
                try {
                    // close old channel
                    Channel oldChannel = NettyClient.this.channel; // copy reference
                    if (oldChannel != null) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close old netty channel " + oldChannel + " on create new netty channel " + newChannel);
                            }
                            oldChannel.close();
                        } finally {
                            NettyChannel.removeChannelIfDisconnected(oldChannel);
                        }
                    }
                } finally {
                    if (NettyClient.this.isClosed()) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close new netty channel " + newChannel + ", because the client closed.");
                            }
                            newChannel.close();
                        } finally {
                            NettyClient.this.channel = null;
                            NettyChannel.removeChannelIfDisconnected(newChannel);
                        }
                    } else {
                        NettyClient.this.channel = newChannel;
                    }
                }
            } else if (future.cause() != null) {
                throw new RemotingException("client(url: " + getUrl() + ") failed to connect to server ");
            } else {
                throw new RemotingException("client(url: " + getUrl() + ") failed to connect to server ");
            }
        } finally {
            if (!isConnected()) {
                //future.cancel(true);
            }
        }
    }

    @Override
    protected com.alibaba.dubbo.remoting.Channel getChannel() {
        // 这里的channel就是上面doConnect方法中创建并且保存的，并且是Netty中的原生channel
        Channel c = channel;
        if (c == null || !c.isConnected())
            return null;

        // 获取/创建一个 NettyChannel 类型对象
        return NettyChannel.getOrAddChannel(c, getUrl(), this);
    }

}
```
