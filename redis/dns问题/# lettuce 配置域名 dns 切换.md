# lettuce 配置域名 dns 切换

大家好，我是烤鸭：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果你也有类似的困扰，运维告诉你，redis连接配置域名，这样出问题了，直接改dns地址就行，不需要重启服务。。。梦想是美好的，现实是残酷的。如果你使用的是 lettuce连接池，那么恭喜你，必须重启服务才能生效。



## 测试一下

### 环境：

```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.2</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
    </dependencies>        
```

### 配置：

application.properties

```
server.port=8115

spring.redis.open=false
spring.redis.database=0
spring.redis.cluster.max-redirects=3
spring.redis.cluster.nodes=myredis1.test.com:6001,myredis1.test.com:6002,myredis2.test.com:6001,myredis2.test.com:6002,myredis3.test.com:6001,myredis3.test.com:6002
spring.redis.timeout=6000
spring.redis.pool.max-active=200
spring.redis.pool.max-wait=-1
spring.redis.pool.max-idle=10
spring.redis.pool.min-idle=5
spring.redis.password=8efS6Snt
```

### dns配置：

本地切换dns解析,可以通过 switchHosts 工具

https://pan.baidu.com/s/1eSgmYzS

修改完如图所示，右下角点 √  生效

![4](.\4.png)



使用jmeter 每2秒发送一次请求，都是成功的

![4](.\5.png)

接着down掉现有的redis集群，服务开始报错：

切换本地hosts，发现报错依旧，spring-boot-starter-data-redis 是不支持 域名dns切换的。

```
2021-07-26 10:42:54.796 ERROR 3712 --- [nio-8115-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: CLUSTERDOWN The cluster is down] with root cause

io.lettuce.core.RedisCommandExecutionException: CLUSTERDOWN The cluster is down
	at io.lettuce.core.internal.ExceptionFactory.createExecutionException(ExceptionFactory.java:137) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.internal.ExceptionFactory.createExecutionException(ExceptionFactory.java:110) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.AsyncCommand.completeResult(AsyncCommand.java:120) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.AsyncCommand.complete(AsyncCommand.java:111) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.CommandWrapper.complete(CommandWrapper.java:63) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.cluster.ClusterCommand.complete(ClusterCommand.java:65) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.complete(CommandHandler.java:746) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:681) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.channelRead(CommandHandler.java:598) ~[lettuce-core-6.1.3.RELEASE.jar:6.1.3.RELEASE]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:166) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:719) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:655) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:581) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:493) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_45]

```



## 源码分析

PooledClusterConnectionProvider

根据操作类型获取不同的 connection

```
@Override
public CompletableFuture<StatefulRedisConnection<K, V>> getConnectionAsync(Intent intent, int slot) {

    if (debugEnabled) {
        logger.debug("getConnection(" + intent + ", " + slot + ")");
    }

    if (intent == Intent.READ && readFrom != null && readFrom != ReadFrom.UPSTREAM) {
        return getReadConnection(slot);
    }

    return getWriteConnection(slot).toCompletableFuture();
}
```

以set操作为例，会发现根据slot获取节点的时候，一直从 partitions 获取，而 partitions 初始化后，如果集群扩容之类的操作会更新 partitions ,其他是不变的

```
private CompletableFuture<StatefulRedisConnection<K, V>> getWriteConnection(int slot) {

    CompletableFuture<StatefulRedisConnection<K, V>> writer;// avoid races when reconfiguring partitions.
    synchronized (stateLock) {
        writer = writers[slot];
    }

    if (writer == null) {
        RedisClusterNode partition = partitions.getPartitionBySlot(slot);
        if (partition == null) {
            clusterEventListener.onUncoveredSlot(slot);
            return Futures.failed(new PartitionSelectorException("Cannot determine a partition for slot " + slot + ".",
                    partitions.clone()));
        }

        // Use always host and port for slot-oriented operations. We don't want to get reconnected on a different
        // host because the nodeId can be handled by a different host.
        RedisURI uri = partition.getUri();
        ConnectionKey key = new ConnectionKey(Intent.WRITE, uri.getHost(), uri.getPort());

        ConnectionFuture<StatefulRedisConnection<K, V>> future = getConnectionAsync(key);

        return future.thenApply(connection -> {

            synchronized (stateLock) {
                if (writers[slot] == null) {
                    writers[slot] = CompletableFuture.completedFuture(connection);
                }
            }

            return connection;
        }).toCompletableFuture();
    }

    return writer;
}
```

再看下初始化连接的方法，连接初始化之后内部的实例对象不会再更新了

```
/**
 * Create a clustered pub/sub connection with command distributor.
 *
 * @param codec Use this codec to encode/decode keys and values, must not be {@code null}
 * @param <K> Key type
 * @param <V> Value type
 * @return a new connection
 */
private <K, V> CompletableFuture<StatefulRedisClusterConnection<K, V>> connectClusterAsync(RedisCodec<K, V> codec) {

    if (partitions == null) {
        return Futures.failed(new IllegalStateException(
                "Partitions not initialized. Initialize via RedisClusterClient.getPartitions()."));
    }

    topologyRefreshScheduler.activateTopologyRefreshIfNeeded();

    logger.debug("connectCluster(" + initialUris + ")");

    DefaultEndpoint endpoint = new DefaultEndpoint(getClusterClientOptions(), getResources());
    RedisChannelWriter writer = endpoint;

    if (CommandExpiryWriter.isSupported(getClusterClientOptions())) {
        writer = new CommandExpiryWriter(writer, getClusterClientOptions(), getResources());
    }

    if (CommandListenerWriter.isSupported(getCommandListeners())) {
        writer = new CommandListenerWriter(writer, getCommandListeners());
    }

    ClusterDistributionChannelWriter clusterWriter = new ClusterDistributionChannelWriter(getClusterClientOptions(), writer,
            topologyRefreshScheduler);
    PooledClusterConnectionProvider<K, V> pooledClusterConnectionProvider = new PooledClusterConnectionProvider<>(this,
            clusterWriter, codec, topologyRefreshScheduler);

    clusterWriter.setClusterConnectionProvider(pooledClusterConnectionProvider);

    StatefulRedisClusterConnectionImpl<K, V> connection = newStatefulRedisClusterConnection(clusterWriter,
            pooledClusterConnectionProvider, codec, getDefaultTimeout());

    connection.setReadFrom(ReadFrom.UPSTREAM);
    connection.setPartitions(partitions);

    Supplier<CommandHandler> commandHandlerSupplier = () -> new CommandHandler(getClusterClientOptions(), getResources(),
            endpoint);
    Mono<SocketAddress> socketAddressSupplier = getSocketAddressSupplier(connection::getPartitions,
            TopologyComparators::sortByClientCount);
    Mono<StatefulRedisClusterConnectionImpl<K, V>> connectionMono = Mono
            .defer(() -> connect(socketAddressSupplier, endpoint, connection, commandHandlerSupplier));

    for (int i = 1; i < getConnectionAttempts(); i++) {
        connectionMono = connectionMono
                .onErrorResume(t -> connect(socketAddressSupplier, endpoint, connection, commandHandlerSupplier));
    }

    return connectionMono.doOnNext(
                    c -> connection.registerCloseables(closeableResources, clusterWriter, pooledClusterConnectionProvider))
            .map(it -> (StatefulRedisClusterConnection<K, V>) it).toFuture();
}
```



![4](.\9.png)



## 解决方案

别闹了，年初就有人提过。

https://github.com/lettuce-io/lettuce-core/issues/1572

想更新dns，需要重建 channel。作者也说了，没什么好办法，除非用反射，但是反射也不是很好。

刚开始出现这个问题，我还以为是 netty的问题，劲劲的给提issue了，不知道是是我表达不清楚，还是什么其他原因，他们还真给源码改了...（其实是lettuce的问题）

https://github.com/netty/netty/issues/11519

**所以用域名，即便是dns切换了，也得重启服务才能生效。**

**对了，不止 lettuce，jedis 也是一样的。**

