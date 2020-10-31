# # Log destination xxx.com:4562: Waiting 19995ms before attempting reconnection

大家好，我是烤鸭：

​	今天分享一个问题 Log destination xxxx.com:1111: Waiting 19999ms before attempting reconnection，正常来说就是一个超时重连，建议先确认服务端和网络没问题。

1.  问题出现的场景

   场景是，jar包本身有 logback.xml ，以 -javaagent 方式启动，单独加载 logback ，跟业务的分开。

   加载的代码如下，LOGBACK_CONFIG 路径是jar内的logback.xml

   ```
   public static void load() {
           LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
           try {
               // assume SLF4J is bound to logback in the current environment
               URL url = LogbackConfigLoader.class.getClassLoader().getResource(LOGBACK_CONFIG);
               JoranConfigurator configurator = new JoranConfigurator();
               configurator.setContext(context);
               // Call context.reset() to clear any previous configuration, e.g. default
               // configuration. For multi-step configuration, omit calling context.reset().
               // context.reset();
               configurator.doConfigure(url);
           } catch (JoranException je) {
               // StatusPrinter will handle this
               LogUtil.log("LogbackConfigLoader error=" + je);
           }
           StatusPrinter.printIfErrorsOccured(context);
   
       }
   ```

   如果是普通的jar加在pom里的话，其实和当前服务共用一个 logback.xml 就好了，或者也可以自定义名字，但没必要在jar包内自定义。

2. 异常日志

   由于最近项目中一直出现这个日志，而且基本每20秒就会打印一次，也没法关掉，百度上资料也很少，只能自己来了。

   10:04:01,393 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[SLEUTH-INFO] - Log destination xxxx.com:1111: Waiting 19999ms before attempting reconnection.

   正常来说，这个提示就是简单提示下，socket连接断开，可能是网络或者是服务端的原因，然后重连。比如下边这个日志。

   ```
   11:17:06,337 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[LOGSTASH] - Log destination xxx.com:1234: Waiting 27476ms before attempting reconnection.
   11:17:13,302 |-WARN in net.logstash.logback.appender.LogstashAccessTcpSocketAppender[logstash] - Log destination xxx.com:1234: connection failed. java.net.ConnectException: Connection refused: connect
   	at java.net.ConnectException: Connection refused: connect
   	at 	at java.net.DualStackPlainSocketImpl.waitForConnect(Native Method)
   	at 	at java.net.DualStackPlainSocketImpl.socketConnect(DualStackPlainSocketImpl.java:81)
   	at 	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:476)
   	at 	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:218)
   	at 	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:200)
   	at 	at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:162)
   	at 	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:394)
   	at 	at java.net.Socket.connect(Socket.java:606)
   	at 	at net.logstash.logback.appender.AbstractLogstashTcpSocketAppender$TcpSendingEventHandler.openSocket(AbstractLogstashTcpSocketAppender.java:721)
   	at 	at net.logstash.logback.appender.AbstractLogstashTcpSocketAppender$TcpSendingEventHandler.onStart(AbstractLogstashTcpSocketAppender.java:640)
   	at 	at net.logstash.logback.appender.AsyncDisruptorAppender$EventClearingEventHandler.onStart(AsyncDisruptorAppender.java:351)
   	at 	at com.xxx.arch.encoder.com.lmax.disruptor.BatchEventProcessor.notifyStart(BatchEventProcessor.java:224)
   	at 	at com.xxx.arch.encoder.com.lmax.disruptor.BatchEventProcessor.run(BatchEventProcessor.java:120)
   	at 	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
   	at 	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
   	at 	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180)
   	at 	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293)
   	at 	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
   	at 	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at 	at java.lang.Thread.run(Thread.java:748)
   11:17:13,303 |-WARN in net.logstash.logback.appender.LogstashAccessTcpSocketAppender[logstash] - Log destination xxx.com:1234: Waiting 27662ms before attempting reconnection.
   ```

   异常的日志，连接成功后，每10s断开连接，然后过20s重试后连接成功，一直反复，乐此不疲...

   ```
   11:48:54,484 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[SLEUTH-INFO] - Log destination xxx.com:4562: connection established.
   11:49:04,524 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[SLEUTH-INFO] - Log destination xxx.com:4562: Waiting 19949ms before attempting reconnection.
   11:49:24,477 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[SLEUTH-INFO] - Log destination xxx.com:4562: connection established.
   11:49:34,478 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[SLEUTH-INFO] - Log destination xxx.com:4562: Waiting 19995ms before attempting reconnection.
   ```

   

3. AbstractLogstashTcpSocketAppender  源码解析

   看这篇就好了。

   // TODO

   后来我改了源码加了日志，发现一直在报

   

4. 解决方案

   虽然问题找到了，由于猜测是服务端释放连接导致的这个问题，所以没什么好的解决方案，粗暴一点，直接改了 logback-encoder 的日志级别，改为ERROR，看不到WARN日志了，有点骗自己的意思...

   关于调整日志级别，看这篇。

   // TODO

   当我写完整个的时候发现了真正的问题所在...

   agent的jar的logstash-logback-encoder的版本和服务jar的版本不一样...或者说 6.4 版本的有bug？

   当我都改成 5.3 版本后，问题消失了。都改成6.4 ，问题依旧。

   也给作者提了 issue。

   https://github.com/logstash/logstash-logback-encoder/issues/447

   













