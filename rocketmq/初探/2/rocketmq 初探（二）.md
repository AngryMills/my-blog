# rocketmq 初探（二）

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;上一篇简单介绍和rocketmq，这一篇看下源码之注册中心。



## namesrv

先看两个初始化方法
NamesrvController.initialize() 和 NettyRemotingServer.start();

```
public boolean initialize() {
	// 加载配置文件
    this.kvConfigManager.load();
	// 创建 NettyRemotingServer 并初始化参数
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
	
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
	// 将刚才的线程池和netty server 绑定
    this.registerProcessor();
	// 每隔10s检测最近120s不活跃的broker并移除
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
	// 每隔10分钟输出一下配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);
	// 如果想用tls,ssl协议的话,需要证书构造 sslContext
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        // Register a listener to reload SslContext
        try {
            fileWatchService = new FileWatchService(
                new String[] {
                    TlsSystemConfig.tlsServerCertPath,
                    TlsSystemConfig.tlsServerKeyPath,
                    TlsSystemConfig.tlsServerTrustCertPath
                },
                new FileWatchService.Listener() {
                    boolean certChanged, keyChanged = false;
                    @Override
                    public void onChanged(String path) {
                        if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                            log.info("The trust certificate changed, reload the ssl context");
                            reloadServerSslContext();
                        }
                        if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                            certChanged = true;
                        }
                        if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                            keyChanged = true;
                        }
                        if (certChanged && keyChanged) {
                            log.info("The certificate and private key changed, reload the ssl context");
                            certChanged = keyChanged = false;
                            reloadServerSslContext();
                        }
                    }
                    private void reloadServerSslContext() {
                        ((NettyRemotingServer) remotingServer).loadSslContext();
                    }
                });
        } catch (Exception e) {
            log.warn("FileWatchService created error, can't load the certificate dynamically");
        }
    }

    return true;
}
```

NettyRemotingServer.start()

```
public void start() {
	// 用刚才初始化的线程池创建线程
    this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
        nettyServerConfig.getServerWorkerThreads(),
        new ThreadFactory() {

            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
            }
        });
	// 构建netty 相关的handler,包含连接、读数据、解码、请求和响应处理
    prepareSharableHandlers();
	// 创建netty server,使用初始化的参数和刚才的handler初始化channel
    ServerBootstrap childHandler =
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
            .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 1024)
            .option(ChannelOption.SO_REUSEADDR, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
            .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
            .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                        .addLast(defaultEventExecutorGroup,
                            encoder,
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            connectionManageHandler,
                            serverHandler
                        );
                }
            });

    if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
        childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    }

    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
        throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
    }
	// 启动channel的监听器,针对channel的连接、关闭、异常、空闲(后面其他的实现都是关闭逻辑)
    if (this.channelEventListener != null) {
        this.nettyEventExecutor.start();
    }
	// 每秒处理过时的响应,如果超时时间+1秒没响应,就移除该请求并手动回调(由于注册中心没有对外发请求,所以没用到,client和server用到了)
    this.timer.scheduleAtFixedRate(new TimerTask() {从

        @Override
        public void run() {
            try {
                NettyRemotingServer.this.scanResponseTable();
            } catch (Throwable e) {
                log.error("scanResponseTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

再看下 NettyClientHandler，对请求和响应指令进行处理

```
/**
 * Entry of incoming command processing.
 *
 * <p>
 * <strong>Note:</strong>
 * The incoming remoting command may be
 * <ul>
 * <li>An inquiry request from a remote peer component;</li>
 * <li>A response to a previous request issued by this very participant.</li>
 * </ul>
 * </p>
 *
 * @param ctx Channel handler context.
 * @param msg incoming remoting command.
 * @throws Exception if there were any error while processing the incoming command.
 */
public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
    final RemotingCommand cmd = msg;
    if (cmd != null) {
        switch (cmd.getType()) {
            case REQUEST_COMMAND:
            	// 接收请求并处理
                processRequestCommand(ctx, cmd);
                break;
            case RESPONSE_COMMAND:
            	// 接收响应,维护responseTable(注册中心用不到)
                processResponseCommand(ctx, cmd);
                break;
            default:
                break;
        }
    }
}
```

由于注册中心没有发起 request，看下 processRequestCommand(接收request)

```
/**
 * Process incoming request command issued by remote peer.
 *
 * @param ctx channel handler context.
 * @param cmd request command.
 */
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
	// request的code在 RequestCode 类维护,包括 发送、拉取等等
    final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
    final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
    // 自增计数
    final int opaque = cmd.getOpaque();

    if (pair != null) {
        Runnable run = new Runnable() {
            @Override
            public void run() {
                try {
                	// ACL鉴权 (client端和broker使用)
                    doBeforeRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                    final RemotingResponseCallback callback = new RemotingResponseCallback() {
                        @Override
                        public void callback(RemotingCommand response) {
                            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                            if (!cmd.isOnewayRPC()) {
                                if (response != null) {
                                    response.setOpaque(opaque);
                                    response.markResponseType();
                                    try {
                                        ctx.writeAndFlush(response);
                                    } catch (Throwable e) {
                                        log.error("process request over, but response failed", e);
                                        log.error(cmd.toString());
                                        log.error(response.toString());
                                    }
                                } else {
                                }
                            }
                        }
                    };
                    // 异步 or 同步
                    if (pair.getObject1() instanceof AsyncNettyRequestProcessor) {
                        AsyncNettyRequestProcessor processor = (AsyncNettyRequestProcessor)pair.getObject1();
                        processor.asyncProcessRequest(ctx, cmd, callback);
                    } else {
                        NettyRequestProcessor processor = pair.getObject1();
                        // 比较重要的地方,单独分析
                        RemotingCommand response = processor.processRequest(ctx, cmd);
                        callback.callback(response);
                    }
                } catch (Throwable e) {
                    log.error("process request exception", e);
                    log.error(cmd.toString());

                    if (!cmd.isOnewayRPC()) {
                        final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                            RemotingHelper.exceptionSimpleDesc(e));
                        response.setOpaque(opaque);
                        ctx.writeAndFlush(response);
                    }
                }
            }
        };
		// 系统繁忙,注册中心不会提示这个(broker 刷盘不及时会报这个)
        if (pair.getObject1().rejectRequest()) {
            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                "[REJECTREQUEST]system busy, start flow control for a while");
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            return;
        }

        try {
            final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
            pair.getObject2().submit(requestTask);
        } catch (RejectedExecutionException e) {
        	// 避免日志打印的太多
            if ((System.currentTimeMillis() % 10000) == 0) {
                log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                    + ", too many requests and system thread pool busy, RejectedExecutionException "
                    + pair.getObject2().toString()
                    + " request code: " + cmd.getCode());
            }
			// 不是单向请求(onewayRPC,线程池满的话,直接返回系统繁忙)
            if (!cmd.isOnewayRPC()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                    "[OVERLOAD]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
            }
        }
    } else {
        String error = " request type " + cmd.getCode() + " not supported";
        final RemotingCommand response =
            RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
        response.setOpaque(opaque);
        ctx.writeAndFlush(response);
        log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
    }
}
```

我们先看一下 NettyRequestProcessor.processRequest  实现

![1](.\1.png)

 DefaultRequestProcessor.processRequest 

其实看名字就能看出来 注册中心的操作了

```
public RemotingCommand processRequest(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {

    if (ctx != null) {
        log.debug("receive request, {} {} {}",
            request.getCode(),
            RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
            request);
    }


    switch (request.getCode()) {
        case RequestCode.PUT_KV_CONFIG:
        	// admin调用,配置添加到 configTable,定期打印
            return this.putKVConfig(ctx, request);
        case RequestCode.GET_KV_CONFIG:
        	// admin调用,获取配置
            return this.getKVConfig(ctx, request);
        case RequestCode.DELETE_KV_CONFIG:
        	// admin调用,删除配置
            return this.deleteKVConfig(ctx, request);
        case RequestCode.QUERY_DATA_VERSION:
        	// broker 获取topic配置
            return queryBrokerTopicConfig(ctx, request);
        case RequestCode.REGISTER_BROKER:
        	// 注册broker,版本不同处理逻辑有些不一样(topic配置信息封装不同)
            Version brokerVersion = MQVersion.value2Version(request.getVersion());
            if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                return this.registerBrokerWithFilterServer(ctx, request);
            } else {
                return this.registerBroker(ctx, request);
            }
        case RequestCode.UNREGISTER_BROKER:
        	// 下线 broker
            return this.unregisterBroker(ctx, request);
        case RequestCode.GET_ROUTEINFO_BY_TOPIC:
        	// 根据topic获取路由信息，获取的key是 ORDER_TOPIC_CONFIG+topicid
            return this.getRouteInfoByTopic(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_INFO:
        	// 获取broker 集群信息
            return this.getBrokerClusterInfo(ctx, request);
        case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
        	// 废除broker的写入权限
            return this.wipeWritePermOfBroker(ctx, request);
        case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
        	// 获取所有的topic
            return getAllTopicListFromNameserver(ctx, request);
        case RequestCode.DELETE_TOPIC_IN_NAMESRV:
        	// 删除topic
            return deleteTopicInNamesrv(ctx, request);
        case RequestCode.GET_KVLIST_BY_NAMESPACE:
        	// 根据namespace获取配置
            return this.getKVListByNamespace(ctx, request);
        case RequestCode.GET_TOPICS_BY_CLUSTER:
        	// 根据cluster下的broker获取topic
            return this.getTopicsByCluster(ctx, request);
        case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
        	// 获取cluster、broker和关联信息
            return this.getSystemTopicListFromNs(ctx, request);
        case RequestCode.GET_UNIT_TOPIC_LIST:
        	// 设置unit_mode true && 非重试的时候,这个配置好像没用啊(https://github.com/apache/rocketmq/issues/639)
            return this.getUnitTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
        	// 设置unit_mode true(校验消息和心跳的时候),获取topic
            return this.getHasUnitSubTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
        	// !getUnitTopicList && getHasUnitSubTopicList
            return this.getHasUnitSubUnUnitTopicList(ctx, request);
        case RequestCode.UPDATE_NAMESRV_CONFIG:
        	// 更新注册中心配置
            return this.updateConfig(ctx, request);
        case RequestCode.GET_NAMESRV_CONFIG:
        	// 获取注册中心配置
            return this.getConfig(ctx, request);
        default:
            break;
    }
    return null;
}
```

## 小结 

注册中心的作用：

存了 cluster、broker、topic的信息。

提供了一些接口，可以broker注册和下线，修改配置等。

检测和维护broker是否活跃。

