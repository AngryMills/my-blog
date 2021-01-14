# redis出现过多command 慢查询slowlog出现command命令

大家好，我是烤鸭：

​	今天分享一个问题，一个关于redis slowlog，执行过多 command命令的问题。



## 问题来源

所有走redis 的接口tp99和平均耗时是原来的两倍不止，运维说redis 的qps也翻倍了。查了下slowlog，发现command 命令占大多数，除了两条业务请求，其他都是command。

![1](.\1.png)

command命令返回的是当前redis所能执行的命令和对应的账号权限，也就不超过200条数据吧。(但频繁执行会增加qps，影响集群性能)

这个command命令是确实由业务服务发起的(ip符合)，但是代码里也没有显示的调用(第一时间感觉是sdk有问题)。由于问题最开始出现在5天前，让运维查了一周前的记录，发现command命令也存在，只是没那么明显,比例大概1/20(调20次get命令，会出现一次command)。

![1](.\2.png)

## 源码查看，找蛛丝马迹

项目框架用的是 springboot的版本2.0.4.RELEASE，对应 spring-boot-starter-data-redis的版本2.0.4.RELEASE

对应 lettuce-core 版本 5.0.4.RELEASE。

分别从这几个包搜 command  命令，搜不到。

代码里用的比较多的有 redisTemplate.executePipelined 方法，跟这个代码进去看一下。

以get方法为例，由于默认调用的是 lettuce，RedisStringCommands.get ——> LettuceStringCommands.getConnection(getAsyncConnection) —> 

LettuceConnection.getConnection()—>

而 LettuceConnection中的变量 asyncSharedConn 会是null，所以每次都得尝试重新连接

![1](.\12.png)

LettuceConnection.getDedicatedConnection()—> ClusterConnectionProvider.getConnection (配置连接池的话走的是 LettucePoolingConnectionProvider.getConnection，所以每次不需要再次创建连接 )

![1](.\9.png)

而LettucePoolingConnectionProvider.getConnection会先判断连接池是否存在，是否不存在，会先去创建。

ClusterConnectionProvider.getConnection 和 LettucePoolingConnectionProvider.getConnection 创建连接池时调用的都是 LettuceConnectionProvider.getConnection

![1](.\11.png)

LettuceConnectionProvider.getConnection(每次创建连接都会执行 command命令)

![1](.\4.png)

RedisClusterClient.connect—> RedisClusterClient.connectClusterImpl

这个方法里的调用了 connection.inspectRedisState();

command() 方法进入到了 ClusterFutureSyncInvocationHandler.handleInvocation

![1](.\6.png)

这里能看到每一次调用的命令，打开debug日志也能看到。

![1](.\10.png)

原因清楚了，就是每次执行命令都会检查 LettuceConnection asyncDedicatedConn是否为空， 如果为空，就会再次连接，再次连接就会执行command命令。为啥呢？连接没有池化。看下配置类。

```
package com.my.maggie.demo.config;


import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.data.redis.RedisProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.redis.connection.RedisClusterConfiguration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisNode;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.util.StringUtils;

import java.util.HashSet;
import java.util.List;
import java.util.Set;


@Configuration
public class RedisConfigration {

    @Bean
    @ConfigurationProperties(prefix="spring.redis-one")
    @Primary
    public RedisProperties redisPropertiesCache() {
        return new RedisProperties();
    }


    @Bean
    @Primary
    public LettuceConnectionFactory redisConnectionFactoryOne() {
        List<String> clusterNodes = redisPropertiesCache().getCluster().getNodes();
        Set<RedisNode> nodes = new HashSet<RedisNode>();
        clusterNodes.forEach(address -> nodes.add(new RedisNode(address.split(":")[0].trim(), Integer.parseInt(address.split(":")[1]))));
        RedisClusterConfiguration clusterConfiguration = new RedisClusterConfiguration();
        clusterConfiguration.setClusterNodes(nodes);
        if (!StringUtils.isEmpty(RedisPassword.of(redisPropertiesCache().getPassword()))) {
            clusterConfiguration.setPassword(RedisPassword.of(redisPropertiesCache().getPassword()));
        }
        return new LettuceConnectionFactory(clusterConfiguration);
    }

    @Bean(name="redisTemplateOne")
    public StringRedisTemplate redisTemplateCache(@Qualifier("redisConnectionFactoryOne") RedisConnectionFactory redisConnectionFactory){
        return new StringRedisTemplate(redisConnectionFactory);
    }

}

```



## 解决方案

1.  配置类增加池化配置

```
@Bean
    @Primary
    public LettuceConnectionFactory redisConnectionFactoryOne() {
        // 连接池配置
        GenericObjectPoolConfig genericObjectPoolConfig = new GenericObjectPoolConfig();
        genericObjectPoolConfig.setMaxIdle(100);
        genericObjectPoolConfig.setMinIdle(10);
        genericObjectPoolConfig.setMaxTotal(5);
        genericObjectPoolConfig.setMaxWaitMillis(-1);
        genericObjectPoolConfig.setTimeBetweenEvictionRunsMillis(100);
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .commandTimeout(Duration.ofMillis(1000))
                .shutdownTimeout(Duration.ofMillis(1000))
                .poolConfig(genericObjectPoolConfig)
                .build();
        // redis 配置
        List<String> clusterNodes = redisPropertiesCache().getCluster().getNodes();
        Set<RedisNode> nodes = new HashSet<RedisNode>();
        clusterNodes.forEach(address -> nodes.add(new RedisNode(address.split(":")[0].trim(), Integer.parseInt(address.split(":")[1]))));
        RedisClusterConfiguration clusterConfiguration = new RedisClusterConfiguration();
        clusterConfiguration.setClusterNodes(nodes);
        if (!StringUtils.isEmpty(RedisPassword.of(redisPropertiesCache().getPassword()))) {
            clusterConfiguration.setPassword(RedisPassword.of(redisPropertiesCache().getPassword()));
        }
        return new LettuceConnectionFactory(clusterConfiguration,clientConfig);
    }
```

2.  使用springboot自带的 StringRedisTemplate (2.0 以上版本默认 lettuce ,会自动创建连接池)
3.  升级springboot 版本(2.1.0.RELEASE 及以上)

## 总结

个人觉得这应该是算是 spring-data-redis 的一个bug吧。 升级版本确实可以解决这个问题。看下新版本怎么解决的。

2.1.0.RELEASE 以前

LettuceConnectionFactory，构造参数第一个是 asyncSharedConn，直接传的是 null

```
@Override
	public RedisClusterConnection getClusterConnection() {

		if (!isClusterAware()) {
			throw new InvalidDataAccessApiUsageException("Cluster is not configured!");
		}

		return new LettuceClusterConnection(connectionProvider, clusterCommandExecutor,
				clientConfiguration.getCommandTimeout());
	}
```

LettuceClusterConnection

```
public LettuceClusterConnection(LettuceConnectionProvider connectionProvider, ClusterCommandExecutor executor,
			Duration timeout) {

		super(null, connectionProvider, timeout.toMillis(), 0);

		Assert.notNull(executor, "ClusterCommandExecutor must not be null.");

		this.topologyProvider = new LettuceClusterTopologyProvider(getClient());
		this.clusterCommandExecutor = executor;
		this.disposeClusterCommandExecutorOnClose = false;
	}
```

2.1.0.RELEASE 以后

LettuceConnectionFactory，asyncSharedConn 这个值是传入的，getOrCreateSharedConnection()，没有就创建，所以不会有null的情况

```
public RedisClusterConnection getClusterConnection() {

		if (!isClusterAware()) {
			throw new InvalidDataAccessApiUsageException("Cluster is not configured!");
		}

		RedisClusterClient clusterClient = (RedisClusterClient) client;

		return getShareNativeConnection()
				? new LettuceClusterConnection(
						(StatefulRedisClusterConnection<byte[], byte[]>) getOrCreateSharedConnection().getConnection(),
						connectionProvider, clusterClient, clusterCommandExecutor, clientConfiguration.getCommandTimeout())
				: new LettuceClusterConnection(null, connectionProvider, clusterClient, clusterCommandExecutor,
						clientConfiguration.getCommandTimeout());
	}
```



 