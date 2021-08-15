# rocketmq 初探（三）

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;上一篇介绍了注册中心，这一篇看下broker。



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
		// 初始化,一会看
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
    boolean result = this.topicConfigManager.load();

    result = result && this.consumerOffsetManager.load();
    result = result && this.subscriptionGroupManager.load();
    result = result && this.consumerFilterManager.load();

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

    result = result && this.messageStore.load();

    if (result) {
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
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

        this.pullMessageExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getPullMessageThreadPoolNums(),
            this.brokerConfig.getPullMessageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.pullThreadPoolQueue,
            new ThreadFactoryImpl("PullMessageThread_"));

        this.replyMessageExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getProcessReplyMessageThreadPoolNums(),
            this.brokerConfig.getProcessReplyMessageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.replyThreadPoolQueue,
            new ThreadFactoryImpl("ProcessReplyMessageThread_"));

        this.queryMessageExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getQueryMessageThreadPoolNums(),
            this.brokerConfig.getQueryMessageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.queryThreadPoolQueue,
            new ThreadFactoryImpl("QueryMessageThread_"));

        this.adminBrokerExecutor =
            Executors.newFixedThreadPool(this.brokerConfig.getAdminBrokerThreadPoolNums(), new ThreadFactoryImpl(
                "AdminBrokerThread_"));

        this.clientManageExecutor = new ThreadPoolExecutor(
            this.brokerConfig.getClientManageThreadPoolNums(),
            this.brokerConfig.getClientManageThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.clientManagerThreadPoolQueue,
            new ThreadFactoryImpl("ClientManageThread_"));

        this.heartbeatExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getHeartbeatThreadPoolNums(),
            this.brokerConfig.getHeartbeatThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.heartbeatThreadPoolQueue,
            new ThreadFactoryImpl("HeartbeatThread_", true));

        this.endTransactionExecutor = new BrokerFixedThreadPoolExecutor(
            this.brokerConfig.getEndTransactionThreadPoolNums(),
            this.brokerConfig.getEndTransactionThreadPoolNums(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.endTransactionThreadPoolQueue,
            new ThreadFactoryImpl("EndTransactionThread_"));

        this.consumerManageExecutor =
            Executors.newFixedThreadPool(this.brokerConfig.getConsumerManageThreadPoolNums(), new ThreadFactoryImpl(
                "ConsumerManageThread_"));

        this.registerProcessor();

        final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
        final long period = 1000 * 60 * 60 * 24;
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

        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
                if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
                    this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
                    this.updateMasterHAServerAddrPeriodically = false;
                } else {
                    this.updateMasterHAServerAddrPeriodically = true;
                }
            } else {
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
        initialTransaction();
        initialAcl();
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

注册中心的作用：

存了 cluster、broker、topic的信息。

提供了一些接口，可以broker注册和下线，修改配置等。

检测和维护broker是否活跃。

