# 修改logback  logstash-logback-encoder 框架本身的日志级别

大家好，我是烤鸭：

​     最近遇到一个问题，想把logback框架本身的日志级别修改，需要 logstash-logback-encoder 6.1 以上的版本才可以。

## 直接上代码

这里修改的不是业务日志级别，是 logback 框架本身(确切地说是 logstash-logback-encoder)这个包的日志级别，源码默认的是 WARN 级别，现在想改成只有ERROR的日志输出。

![image-20201028160401394](C:\Users\wangguangmin\AppData\Roaming\Typora\typora-user-images\image-20201028160401394.png)

初始化加载类：

```
package com.xxx.reporter.flume;

import ch.qos.logback.classic.LoggerContext;
import ch.qos.logback.classic.joran.JoranConfigurator;
import ch.qos.logback.core.joran.spi.JoranException;
import ch.qos.logback.core.status.OnConsoleStatusListener;
import ch.qos.logback.core.status.Status;
import ch.qos.logback.core.util.StatusPrinter;
import com.xxx.log.LogUtil;
import net.logstash.logback.status.LevelFilteringStatusListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.URL;

/**

 * 自定义加载logback配置。
   *

 * @author 

 * @date 2020/7/23
   */
   public class LogbackConfigLoader {

   private static final String LOGBACK_CONFIG = "logback.xml";

   /**

    * 重新加载logback 配置文件
      */
      public static void load() {
      LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
      try {
          // assume SLF4J is bound to logback in the current environment
          URL url = LogbackConfigLoader.class.getClassLoader().getResource(LOGBACK_CONFIG);
          JoranConfigurator configurator = new JoranConfigurator();
          configurator.setContext(context);
          // Call context.reset() to clear any previous configuration, e.g. default
          // configuration. For multi-step configuration, omit calling context.reset().
          context.reset();
          // 初始化filter,并设置级别 ERROR
          addDefaultFilter(context);
          configurator.doConfigure(url);
      } catch (JoranException je) {
          // StatusPrinter will handle this
          LogUtil.log("LogbackConfigLoader error=" + je);
      }
      StatusPrinter.printIfErrorsOccured(context);

   }

   private static void addDefaultFilter(LoggerContext context) {
       context.getStatusManager().getCopyOfStatusListenerList();
       LevelFilteringStatusListener statusListener = new LevelFilteringStatusListener();
       statusListener.setLevelValue(Status.ERROR);
       statusListener.setDelegate(new OnConsoleStatusListener());
       statusListener.setContext(context);
       statusListener.start();
       context.getStatusManager().add(statusListener);
   }
   }
```

在项目启动的时候调一下这个方法就好了。

修改前以下的warn日志会打印，修改后就没了：

11:17:06,337 |-WARN in net.logstash.logback.appender.LogstashTcpSocketAppender[LOGSTASH] - Log destination xxx.com : 1234: Waiting 27476ms before attempting reconnection.
11:17:13,302 |-WARN in net.logstash.logback.appender.LogstashAccessTcpSocketAppender[logstash] - Log destination xxx.com : 1234: connection failed. java.net.ConnectException: Connection refused: connect
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
11:17:13,303 |-WARN in net.logstash.logback.appender.LogstashAccessTcpSocketAppender[logstash] - Log destination xxx.com :  1234: Waiting 27662ms before attempting reconnection.

根据自己的实际业务场景使用，有些 warn 还是有必要的 。