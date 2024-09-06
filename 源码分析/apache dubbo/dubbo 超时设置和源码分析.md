# Dubbo 超时设置和源码分析

## 一、前言

在工作中碰到一个业务接口时间比较长，需要修改超时时间，不知道原理，在网上搜索，看到有人说如果你觉得自己了解了 dubbo 的超时机制，那么问问自己以下问题：

- 超时是针对消费端还是服务端？
- 超时在哪设置？
- 超时设置的优先级是什么？
- 超时解决的是什么问题 ？

如果连这些都回答不上了，那只能说明还没有完全掌握 dubbo 的超时机制。先来说一说结论：

- 超时是针对消费端的，消费端超时会抛出 TimeoutException 而服务器端调用服务超时仅仅会打印一个 warn 日志
- 超时在消费端、服务器端设置，dubbo 会合并这两个设置
- consumer 方法级别 > provider 方法级别 > consumer 接口级别 > provider 接口级别 > consumer 全局级别 > provider 全局级别。如果都没配置，那么就是 dubbo 默认的 1 秒
- 超时机制是为了防止线程一直阻塞，客户端的用户线程不能因为服务端超时而一直类似 wait， 导致无法正常响应其他业务。

超时的实现原理如下：

1. 对于客户端而言，在向服务器发送 Rpc 请求时，会获取到 URL 中配置好的超时时间 timeout，在创建 DefaultFuture 对象时传入。接着调用 DefaultFuture 的 get 方法，阻塞 timeout 时间，直到超时或者结果返回。在创建 DefaultFuture 对象时，会创建一个扫描线程，扫描 FUTURES 中的 future 对象，如果现在的时间减去 DefaultFuture 创建的时间大于超时时间 timeout 的话，就创建一个 Response，并且设置为超时状态（可根据请求是否真正发送分为客户端超时以及服务端超时），然后唤醒阻塞在 get 方法上的用户线程。
2. 对于服务端而言，在 Dubboprotocol$1#reply 方法中，调用 invoker.invoke 方法的过程中，会经过多个 Filter 才会调用执行服务器端的具体方法。其中有一个 Filter 是 TimeoutFilter，它会记录方法调用完成所花的具体时间，如果大于用户在服务端配置的超时时间，就会打印出警告信息。所以说，客户端超时会抛出异常，而服务器端超时只会记录警告信息。

## 二、超时时间优先级

Consumer 端全局超时时间配置：

```xml{.line-numbers}
<dubbo:consumer timeout="5000" /> 
```

Consumer 端指定接口以及特定方法超时配置：

```xml{.line-numbers}
<dubbo:service interface="me.kimi.samples.dubbo.facade.QuestionFacade" ref="questionFacade" timeout="6000">
    <dubbo:method name="getQuestionById" timeout="7000"/>
</dubbo:service> 
```

观察控制台打印的注册 URL：

```shell{.line-numbers}
consumer://172.16.71.30/me.kimi.samples.dubbo.facade.QuestionFacade?application=demo-consumer&category=providers,configurators,routers&check=false&default.proxy=jdk&default.timeout=5000&dubbo=2.6.2&getQuestionById.timeout=7000&interface=me.kimi.samples.dubbo.facade.QuestionFacade&logger=log4j&methods=getQuestionById&pid=13884&side=consumer&timeout=6000&timestamp=1536630294523 
```

从这里可以看出：

- Consumer 全局的超时设置，default.timeout = 5000
- Consumer 接口级别的超时设置，timeout = 6000
- Consumer 方法级别的超时设置，getQuestionById.timeout = 7000

省略了一部分调用链，最终会来到这里，DubboInvoker，读取超时时间：

```java{.line-numbers}
// DubboInvoker
protected Result doInvoke(final Invocation invocation) throws Throwable {

    // 省略代码

    try {
        // 获取异步配置
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        // isOneway 为 true，表明单向通信，也就是异步无返回值
        // isOneway 其实就是根据 url 中 return 参数的值，如果 return 为 false，则表明不关注返回值，因此在后面中不会在 RpcContext 中设置 future
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        // 读取超时时间，这里 dubbo 已经把服务端的 timeout 参数和消费端的 timeout 参数合并
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);

        // 省略代码
    
    // 在抛出 TimeoutException 和 RemotingException 之后，都会将其封装成 RpcException 对象，
    // 这个 RpcException 可以被上级（比如 FailoverClusterInvoker）捕获
    } catch (TimeoutException e) {
        throw new RpcException("Invoke remote method timeout. method: ");
    } catch (RemotingException e) {
        throw new RpcException("Failed to invoke remote method: ");
    }
} 
```

看一下获取参数的方法：

```java{.line-numbers}
public int getMethodParameter(String method, String key, int defaultValue) {
    // 首先查 getQuestionById.timeout
    String methodKey = method + "." + key;
    // 从数字缓存中先获取，不需要每次都 parseInt
    Number n = getNumbers().get(methodKey);
    if (n != null) {
        return n.intValue();
    }
    // 没得话，去取字符串值
    String value = getMethodParameter(method, key);
    if (value == null || value.length() == 0) {
        // 三个地方都没配置，返回默认值，默认是1秒
        return defaultValue;
    }
    // 放入缓存中
    int i = Integer.parseInt(value);
    getNumbers().put(methodKey, i);
    return i;
} 

public String getMethodParameter(String method, String key) {
    // 首先查 getQuestionById.timeout
    String value = parameters.get(method + "." + key);
    if (value == null || value.length() == 0) {
        // 没有设定方法级别的，去查接口级别或全局的
        return getParameter(key);
    }
    return value;
} 

public String getParameter(String key) {
    // 接口级别去查 timeout 
    String value = parameters.get(key);
    if (value == null || value.length() == 0) {
        // 没的话查询全局级别 default.timeout
        value = parameters.get(Constants.DEFAULT_KEY_PREFIX + key);
    }
    return value;
} 
```

从上面的代码可以得知，timeout 设置的优先级为：方法级别 > 接口级别 > 全局级别。这里要特殊提一点，就是 dubbo 会合并服务端客户端的设置。我们在这里将 provider 端的配置设置如下：

```xml{.line-numbers}
<dubbo:provider timeout="10000" accepts="500"/>
<!-- service implementation, as same as regular local bean -->
<bean id="questionFacade" class="me.kimi.samples.dubbo.provider.service.QuestionFacadeImpl"/>
<!-- declare the service interface to be exported -->
<dubbo:service interface="me.kimi.samples.dubbo.facade.QuestionFacade" ref="questionFacade" timeout="11000">
    <dubbo:method name="getQuestionById" timeout="12000"/>
</dubbo:service> 
```

最后在客户端调用的适合，会对客户端和服务端的参数进行合并，合并之后的 url 如下：

```xml{.line-numbers}
dubbo://172.16.71.30:20880/me.kimi.samples.dubbo.facade.QuestionFacade?anyhost=true&application=demo-provider&default.accepts=500&default.timeout=5000&dubbo=2.6.2&generic=false&interface=me.kimi.samples.dubbo.facade.QuestionFacade&logger=log4j&methods=getQuestionById&getQuestionById.timeout=7000&pid=17508&side=provider&timeout=6000&timestamp=1536660132286 
```

客户端与服务端合并 url 的代码如下所示：

```xml{.line-numbers}
private URL mergeUrl(URL providerUrl) {
    // 这里的 queryMap 是消费者端 consumerUrl 的 map，也就是 localMap
    providerUrl = ClusterUtils.mergeUrl(providerUrl, queryMap); // Merge the consumer side parameters

    // 省略代码
    return providerUrl;
} 

public static URL mergeUrl(URL remoteUrl, Map<String, String> localMap) {
    Map<String, String> map = new HashMap<String, String>();
    Map<String, String> remoteMap = remoteUrl.getParameters();

    if (remoteMap != null && remoteMap.size() > 0) {
        // remoteMap 表示的是 providerUrl 的参数配置
        map.putAll(remoteMap);
        // 省略代码
    }

    if (localMap != null && localMap.size() > 0) {
        // localMap 表示的是 localMap 的参数配置
        map.putAll(localMap);
    }

    if (remoteMap != null && remoteMap.size() > 0) {
        // 省略代码
    }

    return remoteUrl.clearParameters().addParameters(map);
} 
```

所以从上面的代码可以看出，localMap 中的参数值会覆盖掉 remoteMap 中的参数值。也就是 consumer 中的参数配置大于 provider 中的参数配置。所以综合，超时时间的优先级为：：

consumer 方法级别 > provider 方法级别 > consumer 接口级别 > provider 接口级别 > consumer 全局级别 > provider 全局级别

在 dubbo 的用户手册中，对配置有这样的推荐用法，在 Provider 上尽量多配置 Consumer 端属性，原因如下：

- 作为服务的提供者，比服务使用方更清楚服务性能参数，如调用的超时时间，合理的重试次数，等等
- 在 Provider 配置后，Consumer 不配置则会使用Provider的配置值，即 Provider 配置可以作为 Consumer 的缺省值。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider不可控的，并且往往是不合理的
- 另外，配置的覆盖规则：1) 方法级配置别优于接口级别，即小 Scope 优先 2) Consumer 端配置 优于 Provider 配置 优于 全局配置，最后是 Dubbo Hard Code 的配置值（见配置文档）

在 Provider 可以配置的 Consumer 端属性有：

- timeout，方法调用超时
- retries，失败重试次数，缺省是 2（表示加上第一次调用，会调用 3 次）
- loadbalance，负载均衡算法（有多个 Provider 时，如何挑选 Provider 调用），缺省是随机（random）。还可以有轮训 (roundrobin)、最不活跃优先（leastactive，指从 Consumer 端并发调用最好的 Provider，可以减少的反应慢的 Provider 的调用，因为反应更容易累积并发的调用）
- actives，消费者端，最大并发调用限制，即当 Consumer 对一个服务的并发调用到上限后，新调用会 Wait 直到超时。在方法上配置（dubbo:method ）则并发限制针对方法，在接口上配置（dubbo:service），则并发限制针对服务。

## 三、超时实现

有了超时时间，那么 dubbo 是怎么实现超时的呢？再看上面的 DubboInvoker，对于一般的有返回值的调用，最终调用：

```java{.line-numbers}
return (Result) currentClient.request(inv, timeout).get();
```

get 方法的代码如下：

```java{.line-numbers}
public Object get(int timeout) throws RemotingException {
    if (timeout <= 0) {
        timeout = Constants.DEFAULT_TIMEOUT;
    }
    // 检测服务提供方是否成功返回了调用结果
    if (!isDone()) {
        long start = System.currentTimeMillis();
        lock.lock();
        try {
            // 循环检测服务提供方是否成功返回了调用结果
            while (!isDone()) {
                // 如果调用结果尚未返回，这里等待一段时间
                done.await(timeout, TimeUnit.MILLISECONDS);
                // 如果调用结果成功返回，或等待超时，此时跳出 while 循环，执行后续的逻辑
                if (isDone() || System.currentTimeMillis() - start > timeout) {
                    break;
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            lock.unlock();
        }
        if (!isDone()) {
            throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
        }
    }
    return returnFromResponse();
} 
```

先看一下 request 方法，来到 com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeChannel 的 Request 方法：

```java{.line-numbers}
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException("Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // 创建 Request 对象
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    // 设置双向通信标志为 true
    req.setTwoWay(true);
    // 这里的 request 变量类型为 RpcInvocation
    req.setData(request);        
    // 创建 DefaultFuture 对象
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try {
        // 调用 NettyClient 的 send 方法发送请求，需要说明的是，NettyClient 中并未实现 send 方法，该方法继承自父类 AbstractPeer
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    // 返回 DefaultFuture 对象
    return future;
} 
```

重点是 DefaultFuture：

```java{.line-numbers}
static {
    Thread th = new Thread(new RemotingInvocationTimeoutScan(), "DubboResponseTimeoutScanTimer");
    th.setDaemon(true);
    th.start();
}
```

类加载的时候会启动一个超时扫描线程：

```java{.line-numbers}
private static class RemotingInvocationTimeoutScan implements Runnable {
    @Override
    public void run() {
        while (true) {
            try {
                // 扫描 FUTURES 集合
                for (DefaultFuture future : FUTURES.values()) {
                    if (future == null || future.isDone()) {
                        continue;
                    }
                    // 如果 future 未完成，且超时
                    if (System.currentTimeMillis() - future.getStartTimestamp() > future.getTimeout()) {
                        // 创建一个异常的 Response
                        Response timeoutResponse = new Response(future.getId());
                        // set timeout status.
                        // 如果消息已经被客户端发送出去，那么就是服务端时间超时
                        // 如果消息还没有被客户端发送出去，那么就是客户端超时
                        timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
                        timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
                        // 处理异常
                        DefaultFuture.received(future.getChannel(), timeoutResponse);
                    }
                }
                Thread.sleep(30);
            } catch (Throwable e) {
                logger.error("Exception when scan the timeout invocation of remoting.", e);
            }
        }
    }
} 
```

也就是在 DefaultFuture 中会创建一个超时扫描线程，FUTURES 集合表示在阻塞等待服务端响应的 DefaultFuture 对象，所以线程会遍历 FUTURES 集合，如果某个 DefaultFuture 等待的时间超过设定，那么就会创建一个异常 Response，并且将其异常设置为 TimeoutException。received 方法如下：

```java{.line-numbers}
public static void received(Channel channel, Response response) {
    try {
        // 当接收到服务端的返回值时，从 FUTURES 这个 map 中移除掉此 response 对应的 DefaultFuture 对象
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            // 真正接收 response 对象
            future.doReceived(response);
        } else {
            // 省略日志代码
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
} 

private void doReceived(Response res) {
    lock.lock();
    try {
        // 接收到返回值，向在 done 变量上阻塞的线程发送信号进行唤醒操作
        response = res;
        if (done != null) {
            done.signal();
        }
    } finally {
        lock.unlock();
    }
    // 对 callback 进行回调操作
    if (callback != null) {
        invokeCallback(callback);
    }
} 
```

显然这里扫描线程把用户请求线程恢复了。恢复以后，顺着刚才的 DefaultFuture 的 get 方法，来到 returnFromResponse 方法：

```java{.line-numbers}
private Object returnFromResponse() throws RemotingException {
    Response res = response;
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }

    // 如果调用结果的状态为 Response.OK，则表示调用过程正常，服务提供方成功返回了调用结果
    if (res.getStatus() == Response.OK) {
        return res.getResult();
    }

    // 消费者或者提供者出现超时情况的话，抛出异常
    if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        throw new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage());
    }
    throw new RemotingException(channel, res.getErrorMessage());
} 
```