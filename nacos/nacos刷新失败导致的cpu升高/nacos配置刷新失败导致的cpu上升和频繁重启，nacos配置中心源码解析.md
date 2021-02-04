# nacos配置刷新失败导致的cpu上升和频繁重启，nacos配置中心源码解析

大家好，我是烤鸭：
	nacos 版本 1.3.2，先说下结论，频繁重启的原因确实没有找到，跟nacos有关，日志没有保留多少，只能从源码找下头绪(出问题的版本 server用的是 nacos 1.1，nacos-client 1.0)



## nacos 拉取配置原理

有两个核心的类  ClientWorker 和 ServerHttpAgent，先从头捋一下。

NacosConfigBootstrapConfiguration 初始化 NacosConfigManager

```
@Bean
@ConditionalOnMissingBean
public NacosConfigManager nacosConfigManager(
		NacosConfigProperties nacosConfigProperties) {
	return new NacosConfigManager(nacosConfigProperties);
}

```

NacosConfigManager

```
static ConfigService createConfigService(
		NacosConfigProperties nacosConfigProperties) {
	if (Objects.isNull(service)) {
		synchronized (NacosConfigManager.class) {
			try {
				if (Objects.isNull(service)) {
					service = NacosFactory.createConfigService(
							nacosConfigProperties.assembleConfigServiceProperties());
				}
			}
			catch (NacosException e) {
				log.error(e.getMessage());
				throw new NacosConnectionFailureException(
						nacosConfigProperties.getServerAddr(), e.getMessage(), e);
			}
		}
	}
	return service;
}
```

NacosFactory -> ConfigFactory 初始化  NacosConfigService

```
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable e) {
        throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
    }
}
```

executor 每10ms 检查配置信息, executorService 执行 LongPollingRunnable 的线程池(单线程)

```
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager, final Properties properties) {
    this.agent = agent;
    this.configFilterChainManager = configFilterChainManager;

    // 设置超时时间(默认30s)
    init(properties);

    //...
	// 每10ms 检查配置信息
    executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}
```

ClientWorker.checkConfigInfo
拉取任务的执行条件，cacheMap 是维护在内存里的任务数据(Map的key是groupid+dataid 分别 urlencode、value是CacheData)

```
public void checkConfigInfo() {
        // 获取cache的数据数量
        int listenerSize = cacheMap.get().size();
        // 向上取整
        int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
        if (longingTaskCount > currentLongingTaskCount) {
            for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
                // 要判断任务是否在执行 这块需要好好想想。 任务列表现在是无序的。变化过程可能有问题
                executorService.execute(new LongPollingRunnable(i));
            }
            currentLongingTaskCount = longingTaskCount;
        }
    }
```

ClientWorker$LongPollingRunnable

```
 class LongPollingRunnable implements Runnable {
        private int taskId;

        public LongPollingRunnable(int taskId) {
            this.taskId = taskId;
        }

        @Override
        public void run() {

            List<CacheData> cacheDatas = new ArrayList<CacheData>();
            List<String> inInitializingCacheList = new ArrayList<String>();
            try {
                // 校验本地数据
                for (CacheData cacheData : cacheMap.get().values()) {
                    if (cacheData.getTaskId() == taskId) {
                        cacheDatas.add(cacheData);
                        try {
                        	// 看下面具体方法
                            checkLocalConfig(cacheData);
                            if (cacheData.isUseLocalConfigInfo()) {
                            	// 是否有属性变更(变更会通过notify改变监听器的md5值),变更的话更新cacheData
                                cacheData.checkListenerMd5();
                            }
                        } catch (Exception e) {
                            LOGGER.error("get local config info error", e);
                        }
                    }
                }

                // 获取变更属性,更多看下面
                List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
                LOGGER.info("get changedGroupKeys:" + changedGroupKeys);
				// 解析变更属性
                for (String groupKey : changedGroupKeys) {
                    String[] key = GroupKey.parseKey(groupKey);
                    String dataId = key[0];
                    String group = key[1];
                    String tenant = null;
                    if (key.length == 3) {
                        tenant = key[2];
                    }
                    try {
                        String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                        CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                        cache.setContent(ct[0]);
                        if (null != ct[1]) {
                            cache.setType(ct[1]);
                        }
                        LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}, type={}", agent.getName(), dataId, group, tenant, cache.getMd5(), ContentUtils.truncateContent(ct[0]), ct[1]);
                    } catch (NacosException ioe) {
                        String message = String.format( "[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s", agent.getName(), dataId, group, tenant);
                        LOGGER.error(message, ioe);
                    }
                }
                // 更新缓存cacheData
                for (CacheData cacheData : cacheDatas) {
                    if (!cacheData.isInitializing() || inInitializingCacheList.contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                        cacheData.checkListenerMd5();
                        cacheData.setInitializing(false);
                    }
                }
                inInitializingCacheList.clear();
				// 由于从server端拉取配置是 30s一次(同步操作), 这里可以理解为每30s执行的定时线程
                executorService.execute(this);

            } catch (Throwable e) {

                // If the rotation training task is abnormal, the next execution time of the task will be punished
                LOGGER.error("longPolling error : ", e);
                executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
            }
        }
    }
```

ClientWorker.checkLocalConfig

```
private void checkLocalConfig(CacheData cacheData) {
        final String dataId = cacheData.dataId;
        final String group = cacheData.group;
        final String tenant = cacheData.tenant;
        File path = LocalConfigInfoProcessor.getFailoverFile(agent.getName(), dataId, group, tenant);
		// 开启本地配置并且本地文件存在,以本地文件为准,更新缓存
        if (!cacheData.isUseLocalConfigInfo() && path.exists()) {
            String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
            final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
            cacheData.setUseLocalConfigInfo(true);
            cacheData.setLocalConfigInfoVersion(path.lastModified());
            cacheData.setContent(content);
            LOGGER.warn("[{}] [failover-change] failover file created. dataId={}, group={}, tenant={}, md5={}, content={}", agent.getName(), dataId, group, tenant, md5, ContentUtils.truncateContent(content));
            return;
        }
		
        // 开启本地配置并且本地文件不存在,以服务器拉取为准,同时关闭本地配置
        if (cacheData.isUseLocalConfigInfo() && !path.exists()) {
            cacheData.setUseLocalConfigInfo(false);
            LOGGER.warn("[{}] [failover-change] failover file deleted. dataId={}, group={}, tenant={}", agent.getName(), dataId, group, tenant);
            return;
        }

        // 开启本地配置并且本地文件存在并且缓存中版本和本地文件版本不一样,以本地文件为准,更新缓存
        if (cacheData.isUseLocalConfigInfo() && path.exists() && cacheData.getLocalConfigInfoVersion() != path.lastModified()) {
            String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
            final String md5 = MD5Utils.md5Hex(content, Constants.ENCODE);
            cacheData.setUseLocalConfigInfo(true);
            cacheData.setLocalConfigInfoVersion(path.lastModified());
            cacheData.setContent(content);
            LOGGER.warn("[{}] [failover-change] failover file changed. dataId={}, group={}, tenant={}, md5={}, content={}", agent.getName(), dataId, group, tenant, md5, ContentUtils.truncateContent(content));
        }
    }
```

ClientWorker.checkUpdateDataIds —> ClientWorker.checkUpdateConfigStr —> (ServerHttpAgent)HttpAgent.httpPost
组装请求头和参数，重试，异常处理的代码都删掉了，这里只保留了这块，这就是为什么每次请求server端会本地阻塞30s。

```
public HttpRestResult<String> httpPost(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encode, long readTimeoutMs) throws Exception {
        // ...
        do {
            
                // ...
                HttpRestResult<String> result = NACOS_RESTTEMPLATE
                        .postForm(getUrl(currentServerAddr, path), httpConfig, newHeaders,
                                new HashMap<String, String>(0), paramValues, String.class);
            // ...
            
        } while (System.currentTimeMillis() <= endTime);
        
    }
```

CacheData.checkListenerMd5 ，判断内存中的和server端的md5 值

```
void checkListenerMd5() {
        for (ManagerListenerWrap wrap : listeners) {
            if (!md5.equals(wrap.lastCallMd5)) {
                safeNotifyListener(dataId, group, content, type, md5, wrap);
            }
        }
    }
```

CacheData.safeNotifyListener ，可以看到  listener.receiveConfigChange 是执行具体的刷新操作(热加载)，最后会更新内存的 lastCallMd5 值

```
Runnable job = new Runnable() {
            @Override
            public void run() {
                //...
                // 执行回调之前先将线程classloader设置为具体webapp的classloader，以免回调方法中调用spi接口是出现异常或错用（多应用部署才会有该问题）。
                    Thread.currentThread().setContextClassLoader(appClassLoader);
                    // ...
                    // compare lastContent and content
                    if (listener instanceof AbstractConfigChangeListener) {
                        Map data = ConfigChangeHandler.getInstance()
                                .parseChangeData(listenerWrap.lastContent, content, type);
                        ConfigChangeEvent event = new ConfigChangeEvent(data);
                        // 执行属性变更操作
                        ((AbstractConfigChangeListener) listener).receiveConfigChange(event);
                        listenerWrap.lastContent = content;
                    }
                    
                    listenerWrap.lastCallMd5 = md5;
                    LOGGER.info("[{}] [notify-ok] dataId={}, group={}, md5={}, listener={} ", name, dataId, group, md5,
                            listener);
                    // ...
            }
        };
```



## 问题排查

先看下当时的日志：
能看出来服务在不停的刷新应用(热重启，间隔时间30s)，不停热重启引起cpu升高（又是另一个问题了）

![1](.\1.png)

再看一下机器的cpu，3台不同的机器都几乎是同时出现了这个问题，cpu在24h之内缓慢上升

![1](.\2.png)

日志内容很少，结合刚才看的源码，获取服务端配置确实是30s一次，也就是找到触发点就知道了。



## 假设猜想

1.0 的代码和 1.3.2 还是有点区别，这是 1.0 的 CacheData.safeNotifyListener

```
 private void safeNotifyListener(final String dataId, final String group, final String content,
                                    final String md5, final ManagerListenerWrap listenerWrap) {
        final Listener listener = listenerWrap.listener;

        Runnable job = new Runnable() {
            public void run() {
                // ...
                    Thread.currentThread().setContextClassLoader(appClassLoader);

                    ConfigResponse cr = new ConfigResponse();
                    cr.setDataId(dataId);
                    cr.setGroup(group);
                    cr.setContent(content);
                    configFilterChainManager.doFilter(null, cr);
                    String contentTmp = cr.getContent();
                    listener.receiveConfigInfo(contentTmp);
                    listenerWrap.lastCallMd5 = md5;
                    LOGGER.info("[{}] [notify-ok] dataId={}, group={}, md5={}, listener={} ", name, dataId, group, md5,
                        listener);
            }
        };
```

结合日志猜测是 listenerWrap.lastCallMd5 = md5; 没有执行，也就是执行 listener.receiveConfigInfo(contentTmp); 报错了。

listener.receiveConfigInfo 都干啥了

![1](.\3.png)

简单来说就是调用了 applicationContext.publishEvent，传入的是刷新事件 RefreshEvent

 RefreshEventListener 会接收刷新事件执行刷新操作，继续发布事件  RefreshScopeRefreshedEvent

![1](.\4.png)

对 @RefreshScope 的 Bean 完成热加载。

看一段nacos正常刷新属性（相同的项目）的日志。

后续是对注册中心再次注册和发现服务的过程 (DiscoveryClient)，目前来看是这个地方报错了，但是具体原因还是不知道。

![1](.\5.png)

## 总结

虽然是nacos 引起的，但是问题原因并没有找到。

注册中心使用的是eurka。

服务是新服务，暂时没流量，重启之后也没再发现类似的问题，等下次出现了一定要保留日志...