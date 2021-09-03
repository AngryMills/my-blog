# rocketmq 初探（三）

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;上一篇介绍了注册中心，这一篇看下broker。基于 rocketmq 4.9 版本。



## BrokerStartup#BrokerController

按照代码的先后顺序撸源码：

BrokerController.createBrokerController

```
public static BrokerController createBrokerController(String[] args) {
    // ...

    try {
        // ...
        final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();
		// salve节点的messageMmeory 比例由40% 降低至 30%
        if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
            int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
            messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
        }
		// ...

        switch (messageStoreConfig.getBrokerRole()) {
            case ASYNC_MASTER:
            case SYNC_MASTER:
            	// 0 是master
                brokerConfig.setBrokerId(MixAll.MASTER_ID);
                break;
            case SLAVE:
                if (brokerConfig.getBrokerId() <= 0) {
                    System.out.printf("Slave's brokerId must be > 0");
                    System.exit(-3);
                }

                break;
            default:
                break;
        }
		// DLeger模式,brokerId 为 -1
        if (messageStoreConfig.isEnableDLegerCommitLog()) {
            brokerConfig.setBrokerId(-1);
        }
        
		// ...
        final BrokerController controller = new BrokerController(
            brokerConfig,
            nettyServerConfig,
            nettyClientConfig,
            messageStoreConfig);
        // remember all configs to prevent discard
        controller.getConfiguration().registerConfig(properties);
		// 初始化
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
		// 注册 shutdown 事件
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            private volatile boolean hasShutdown = false;
            private AtomicInteger shutdownTimes = new AtomicInteger(0);

            @Override
            public void run() {
                synchronized (this) {
                    log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
                    if (!this.hasShutdown) {
                        this.hasShutdown = true;
                        long beginTime = System.currentTimeMillis();
                        controller.shutdown();
                        long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                        log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
                    }
                }
            }
        }, "ShutdownHook"));

        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

BrokerController.initialize()

```
public boolean initialize() throws CloneNotSupportedException {
	// 加载本地配置文件(topic,consumerOffset等信息),加载不到加载 .bak文件
    boolean result = this.topicConfigManager.load();

    result = result && this.consumerOffsetManager.load();
    result = result && this.subscriptionGroupManager.load();
    result = result && this.consumerFilterManager.load();
	// 初始化 messageStore(持久化)
    if (result) {
        try {
            this.messageStore =
                new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                    this.brokerConfig);
            if (messageStoreConfig.isEnableDLegerCommitLog()) {
                DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
                ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
            }
            this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
            //load plugin
            MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
            this.messageStore = MessageStoreFactory.build(context, this.messageStore);
            this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
        } catch (IOException e) {
            result = false;
            log.error("Failed to initialize", e);
        }
    }
	// commitlog、consumer和topic关系、索引恢复(临时文件存在的话)
    result = result && this.messageStore.load();

    if (result) {
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
        // netty 配置
        NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
        fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
        this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);
        this.sendMessageExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getSendMessageThreadPoolNums(),
            this.brokerConfig.getSendMessageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.sendThreadPoolQueue,
            new ThreadFactoryImpl("SendMessageThread_"));
		// 不同的线程池，注册到对应的processor
        this.pullMessageExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getPullMessageThreadPoolNums(),
            this.brokerConfig.getPullMessageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.pullThreadPoolQueue,
            new ThreadFactoryImpl("PullMessageThread_"));
        // ...
		// 线程池注册到processor,后续仔细说下
        this.registerProcessor();

        final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
        final long period = 1000 * 60 * 60 * 24;
        // 延迟1天,每天记录昨天存取的消息数量
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.getBrokerStats().record();
                } catch (Throwable e) {
                    log.error("schedule record error.", e);
                }
            }
        }, initialDelay, period, TimeUnit.MILLISECONDS);
		// 每隔5s检查是否更新consumer和offset数据
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.consumerOffsetManager.persist();
                } catch (Throwable e) {
                    log.error("schedule persist consumerOffset error.", e);
                }
            }
        }, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
		// 每隔10s检查是否更新consumer过滤规则数据
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.consumerFilterManager.persist();
                } catch (Throwable e) {
                    log.error("schedule persist consumer filter error.", e);
                }
            }
        }, 1000 * 10, 1000 * 10, TimeUnit.MILLISECONDS);
		// 每隔3分钟检查,开启consumer消费过慢后移除该consumer(默认关闭)
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.protectBroker();
                } catch (Throwable e) {
                    log.error("protectBroker error.", e);
                }
            }
        }, 3, 3, TimeUnit.MINUTES);
		// 每秒打印send\pull\query\transaction队列大小和的最慢的消费耗时
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.printWaterMark();
                } catch (Throwable e) {
                    log.error("printWaterMark error.", e);
                }
            }
        }, 10, 1, TimeUnit.SECONDS);
		// 主broker同步从broker的时候重试
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    log.info("dispatch behind commit log {} bytes", BrokerController.this.getMessageStore().dispatchBehindBytes());
                } catch (Throwable e) {
                    log.error("schedule dispatchBehindBytes error.", e);
                }
            }
        }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
		// 没配置注册中心的话,会每隔2分钟去拉取(url为默认读取系统变量 rocketmq.namesrv.domain:8080/rocketmq)
        if (this.brokerConfig.getNamesrvAddr() != null) {
            this.brokerOuterAPI.updateNameServerAddressList(this.brokerConfig.getNamesrvAddr());
            log.info("Set user specified name server address: {}", this.brokerConfig.getNamesrvAddr());
        } else if (this.brokerConfig.isFetchNamesrvAddrByAddressServer()) {
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

                @Override
                public void run() {
                    try {
                        BrokerController.this.brokerOuterAPI.fetchNameServerAddr();
                    } catch (Throwable e) {
                        log.error("ScheduledTask fetchNameServerAddr exception", e);
                    }
                }
            }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
        }
		// DLeger模式,从节点不需要定期更新 HA主节点地址
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
                if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
                    this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
                    this.updateMasterHAServerAddrPeriodically = false;
                } else {
                    this.updateMasterHAServerAddrPeriodically = true;
                }
            } else {
            	// 每分钟打印主节点和从节点的offset的不同
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.printMasterAndSlaveDiff();
                        } catch (Throwable e) {
                            log.error("schedule printMasterAndSlaveDiff error.", e);
                        }
                    }
                }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
            }
        }
		// tls模式需要证书建立ssl
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
                            ((NettyRemotingServer) fastRemotingServer).loadSslContext();
                        }
                    });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }
        // 事务初始化
        initialTransaction();
        // acl鉴权初始化
        initialAcl();
        // rpc钩子初始化(acl和匿名的)
        initialRpcHooks();
    }
    return result;
}
```

BrokerStartup.start() 

```
public void start() throws Exception {
		// 消息存储,包含 commitLog 和 集群模式下消息同步
        if (this.messageStore != null) {
            this.messageStore.start();
        }
		// nettyServer 的初始化,初探(二)有详细的
        if (this.remotingServer != null) {
            this.remotingServer.start();
        }
		// 同上
        if (this.fastRemotingServer != null) {
            this.fastRemotingServer.start();
        }
		// 每500ms监测 tls的3个证书,如果变了就重新加载(tls.server.certPath...)
        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }
		// nettyRemotingClient.satrt,一会单独看下
        if (this.brokerOuterAPI != null) {
            this.brokerOuterAPI.start();
        }
		// 需要hold的拉取请求统一处理,每隔5s或1s检测消息是否到达
        if (this.pullRequestHoldService != null) {
            this.pullRequestHoldService.start();
        }
		// 每10s扫描 producer、consumer、broker 的在线情况	
        if (this.clientHousekeepingService != null) {
            this.clientHousekeepingService.start();
        }
		// 过滤服务器,在broker机器启动多个filter进程用来进程consumer消息过滤
        if (this.filterServerManager != null) {
            this.filterServerManager.start();
        }
		// 启用DLedger,slave 每隔10s更新配置，broker注册到nameserver
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            startProcessorByHa(messageStoreConfig.getBrokerRole());
            handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
            this.registerBrokerAll(true, false, true);
        }
		// 随机 30-60s,定时 broker注册到nameserver
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

        if (this.brokerStatsManager != null) {
            this.brokerStatsManager.start();
        }
		// 快速失败,如果是刷盘过慢,返回 system busy，清理队列(发送、拉取、心跳、请求)
        if (this.brokerFastFailure != null) {
            this.brokerFastFailure.start();
        }

    }
```



## 小结 

这篇主要是介绍了broker的init和start。

DLegerCommit开启模式(替代原有的commitLog，可以直接读取CommitLog的API)

初始化：

- 消费持久化 messageStore

- commitlog、consumer和topic关系、索引恢复(临时文件存在的话)
- netty配置(remotingServer 和 fastRemotingServer)
- processor(PullMessage、SendMessage 等)
- 定时器(记录消费总量、更新consumer和offset数据 等)
- acl鉴权、事务、rpchook

启动：

- 消息持久化 messageStore
- netty server启动(remotingServer 和 fastRemotingServer)
- tls模式检测证书变化
- pullRequestHold模式下，启动监听消息
- 每10s扫描 producer、consumer、broker 的在线情况
- 启用DLedger,slave 每隔10s更新配置，broker注册到nameserver
- 随机 30-60s,定时 broker注册到nameserver
- 快速失败,如果是刷盘过慢,返回 system busy，清理队列(发送、拉取、心跳、请求)