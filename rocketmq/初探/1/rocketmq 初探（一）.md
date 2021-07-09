# rocketmq 初探（一）

大家好，我是烤鸭：

​	今天看下rocketmq。

## 项目架构

感兴趣的可以下载源码看下。

https://github.com/apache/rocketmq

项目结构图。

![3](.\3.png)

rocketmq-acl: acl 秘钥方式的鉴权，用在broker端。
rocketmq-broker：整个mq的核心，他能够接受producer和consumer的请求，并调用store层服务对消息进行处理。HA服务的基本单元，支持同步双写，异步双写等模式。
rocketmq-client：mq客户端实现，目前官方仅仅开源了java版本的mq客户端，c++，go客户端有社区开源贡献。
rocketmq-common：一些模块间通用的功能类，比如一些配置文件、常量。
rocketmq-distribution：脚本、配置模块。
rocketmq-example：官方提供的例子。
rocketmq-filtersrv：消息过滤服务，相当于在broker和consumer中间加入了一个filter代理。
rocketmq-logappender：日志
rocketmq-logging：日志
rocketmq-namesrv：NameServer，类似服务注册中心，broker在这里注册，consumer和producer在这里找到broker地址
rocketmq-openmessaging：RocketMQ支持openmessaging，详见：https://rocketmq.apache.org/docs/openmessaging-example/
rocketmq-remoting：基于netty的底层通信实现，所有服务间的交互都基于此模块。
rocketmq-srvut：解析命令行的工具类。
rocketmq-store：存储层实现，同时包括了索引服务，高可用HA服务实现。
rocketmq-tools：命令行工具，提供了消息查询等功能。

下面重点说一下几个模块：

注册中心 namesrv、broker、client 和 store，先看一下关系。

![3](.\4.png)

看这个图是不是有点相似，没错，跟 dubbo 很像，除了多了 broker。

nameserver 是注册中心，用来记录broker信息、broker和topic关系。

producer 从nameserver 获取broker信息，进行消息发送。

consumer 从nameserver 获取broker信息，进行消息消费。

## idea 导入源码，本地调试

设置 rocketmq _home 目录，后边的namesrv和broker会用到。新建conf目录，并将 rocket-distribution 的conf里的broker.conf、logback_broker.xml、

logback_namesrv.xml、logback_tools.xml 复制到新建的conf目录中。我这里设置的目录是 E:\my\rocketmq

![3](.\6.png)

我这里修改了日志目录，方便查看日志。

### 启动 NamesrvStartup

```
Connected to the target VM, address: '127.0.0.1:58819', transport: 'socket'
Please set the ROCKETMQ_HOME variable in your environment to match the location of the RocketMQ installation
Disconnected from the target VM, address: '127.0.0.1:58819', transport: 'socket'
```

启动参数配置 rocketmq_home 的环境变量，ROCKETMQ_HOME=E:\my\rocketmq

![3](.\5.png)

启动成功：

```
Connected to the target VM, address: '127.0.0.1:50261', transport: 'socket'
The Name Server boot success. serializeType=JSON
```

会发现 rocketmq_home 目录下生成了 logs/rocketmqlogs 目录，存放的是日志文件。

### 启动broker

设置启动参数和  rocketmq_home 的环境变量 ：

autoCreateTopicEnable=true 是为了测试的时候可以发送时创建topic，默认是 false(不建议开启，避免并发发送时，topic重复问题)

```
-c E:\my\rocketmq\conf\broker.conf -n localhost:9876 autoCreateTopicEnable=true
```

```
ROCKETMQ_HOME=E:\my\rocketmq
```

![3](.\7.png)

会发现 rocketmq_home 目录下生成了 store 目录，存放的是broker维护的信息，像消费者的偏移量、延迟队列的偏移量、topic。

### 启动consumer

rocketmq-example 项目下，example\src\main\java\org\apache\rocketmq\example\quickstart\Consumer.java

指定broker地址：

```
consumer.setNamesrvAddr("localhost:9876");
```

![3](.\8.png)

### 启动producer并发送消息

rocketmq-example 项目下，example\src\main\java\org\apache\rocketmq\example\quickstart\Producer.java

指定broker地址，修改循环次数为2次：

```
producer.setNamesrvAddr("localhost:9876");
```

发送成功：

```
SendResult [sendStatus=SEND_OK, msgId=7F000001395C18B4AAC22C7A99940000, offsetMsgId=0AA80D1200002A9F0000000000000000, messageQueue=MessageQueue [topic=TopicTest, brokerName=broker-a, queueId=3], queueOffset=0]
SendResult [sendStatus=SEND_OK, msgId=7F000001395C18B4AAC22C7A99D80001, offsetMsgId=0AA80D1200002A9F00000000000000C9, messageQueue=MessageQueue [topic=TopicTest, brokerName=broker-a, queueId=0], queueOffset=0]
```

消费端接收成功：

```
ConsumeMessageThread_1 Receive_1 New Messages: [MessageExt [brokerName=broker-a, queueId=3, storeSize=201, queueOffset=0, sysFlag=0, bornTimestamp=1625815032213, bornHost=/10.168.13.18:57729, storeTimestamp=1625815032241, storeHost=/10.168.13.18:10911, msgId=0AA80D1200002A9F0000000000000000, commitLogOffset=0, bodyCRC=613185359, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, CONSUME_START_TIME=1625815055025, UNIQ_KEY=7F000001395C18B4AAC22C7A99940000, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 48], transactionId='null'}]] 
ConsumeMessageThread_2 Receive_1 New Messages: [MessageExt [brokerName=broker-a, queueId=0, storeSize=201, queueOffset=0, sysFlag=0, bornTimestamp=1625815032280, bornHost=/10.168.13.18:57729, storeTimestamp=1625815032282, storeHost=/10.168.13.18:10911, msgId=0AA80D1200002A9F00000000000000C9, commitLogOffset=201, bodyCRC=1401636825, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, CONSUME_START_TIME=1625815056025, UNIQ_KEY=7F000001395C18B4AAC22C7A99D80001, CLUSTER=DefaultCluster, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 49], transactionId='null'}]] 
```

当发送的时候 store目录下会生成 commitLog 目录(消息内容)和consumequeue目录(存的是topic和queueId)

默认上来生成两个文件，2个G。

![3](.\9.png)

