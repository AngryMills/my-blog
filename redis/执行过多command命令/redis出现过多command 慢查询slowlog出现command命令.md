# redis出现过多command 慢查询slowlog出现command命令

大家好，我是烤鸭：

​	今天分享一个问题，一个关于redis slowlog的问题。



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

以get方法为例，由于默认调用的是 lettuce，RedisStringCommands.get ——> LettuceStringCommands.get

![1](.\3.png)

getAsyncConnection —> LettuceConnection.getAsyncConnection