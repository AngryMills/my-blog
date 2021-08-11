# redisson 大量ping操作，导致 tps过高

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;&nbsp;这个问题有点奇怪，新服务上线，redis tps居高不下，还都是ping命令。



## 环境：
服务 ： 280台，redis集群：12主24从

## 问题

由于服务刚上线，还没有访问，发现ping命令的qps 7K，就很纳闷。运维帮忙看了下，确认这些命令的发起ip是业务服务。

![2](.\2.png)

## 问题排查

项目中用到了 lettuce和redisson，在测试环境测试，尝试把redisson去掉后，没有大量ping了。

加上之后，又有了，频率大概是 每分钟 26次。

![2](.\1.png)

## 看下源码

跟了源码发现是 PingConnectionHandler.sendPing 发起的ping操作。

如果触发了 channelActive 就会定时执行ping，检测channel 是否还保持连接。

```
protected void sendPing(final ChannelHandlerContext ctx) {
    final RedisConnection connection = RedisConnection.getFrom(ctx.channel());
    final RFuture<String> future = connection.async(StringCodec.INSTANCE, RedisCommands.PING);
    
    config.getTimer().newTimeout(new TimerTask() {
        @Override
        public void run(Timeout timeout) throws Exception {
            CommandData<?, ?> commandData = connection.getCurrentCommand();
            if ((commandData == null || !commandData.isBlockingCommand()) 
                    && (future.cancel(false) || !future.isSuccess())) {
                ctx.channel().close();
                log.debug("channel: {} closed due to PING response timeout set in {} ms", ctx.channel(), config.getPingConnectionInterval());
            } else {
                sendPing(ctx);
            }
        }
        // 决定ping的频率,为0表示不再ping了，默认是0
    }, config.getPingConnectionInterval(), TimeUnit.MILLISECONDS);
}
```

![](.\3.png)

大部分人都不会有这个问题，因为 redisson默认的 pingConnectionInterval 就是0...

![](.\4.png)

 主要是写公共组件那哥们把这个值默认写成了60s....

![](.\5.png)

## 结论

这个值改了之后就没有这个问题了。不过ping tps: 7k 确实有点诡异。

这个7k 只是部分client发起的，再平均到redis 实例，每个实例150 tps，也还可以接受吧。

最终发现是发现不同的grafana统计数据有差异，估计150 tps差不多。



