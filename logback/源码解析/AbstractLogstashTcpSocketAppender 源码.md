# logback AbstractLogstashTcpSocketAppender 源码解析 

大家好，我是烤鸭：

​	今天分享下 logback  源码 ，版本是 6.5-SNAPSHOT。

1.  写这篇的目的

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

   

2. AbstractLogstashTcpSocketAppender 

   由于 这个用到 com.lmax.disruptor 这个包，推荐看一下美团的这篇 https://tech.meituan.com/2016/11/18/disruptor.html

   **AbstractLogstashTcpSocketAppender**  ，一般是用于发送日志内容，比如将日志内容发送到 logstash/flume 等。

   具体的配置可以参考下 https://www.cnblogs.com/zhyg/p/6994314.html

   **内部类**：

   **TcpSendingEventHandler implements EventHandler<LogEvent<Event>>, LifecycleAware**

   负责执行TCP传输的事件处理器，这个类内部还有3个线程内部类 分别是 

   **KeepAliveRunnable**(用于和socet 保持连接)、**ReaderCallable**(接收socket的流信息)、**WriteTimeoutRunnable**(检测写入超时,如果超时了就关闭连接)

   **UnconnectedConfigurableSSLSocketFactory extends ConfigurableSSLSocketFactory** (创建链接，使用自定义配置参数)

   **TcpSendingEventHandler** 重点看下这个类，处理tcp事务都处理些啥，方法如下：

   **onEvent** ，对 EventHandler.onEvent 的实现，有事件就去处理。代码不长，而且注释特别清晰。

   接受到事件后循环5次，判断socket的读取流的线程是否结束或者socket是否为空，调用 reopenSocket 方法，否则调用下面的writeEvent。如果 readerFuture.isDone() 是服务器关闭了连接，如果是 socket为空，是写入超时。

   ```
   if (readerFuture.isDone() || socket == null) {
       /*
        * If readerFuture.isDone(), then the destination has shut down its output (our input),
        * and the destination is probably no longer listening to its input (our output).
        * This will be the case for Amazon's Elastic Load Balancers (ELB)
        * when an instance behind the ELB becomes unhealthy while we're connected to it.
        *
        * If socket == null here, it means that a write timed out,
        * and the socket was closed by the WriteTimeoutRunnable.
        * 
        * Therefore, attempt reconnection.
        */
       addInfo(peerId + "destination terminated the connection. Reconnecting.");
       reopenSocket();
       try {
           readerFuture.get();
           sendFailureException = NOT_CONNECTED_EXCEPTION;
       } catch (Exception e) {
           sendFailureException = e;
       }
       continue;
   }
   ```

   **writeEvent**，tcp 往服务器写数据。由于keepalive 也会触发事件，但是event 为null。所以这时候判断是 keepalive还是其他事件。

   其他事件的写入还要 兼容 logbakck1.x版本，keepalive 写入的话，写入 换行符。还有个属性 endOfBatch，如果是的话，会执行

   outputStream.flush()

   **onStart** , 启动方法。

   初始化 destinationAttemptStartTimes 数组，目的是为了存放每个连接目标最后尝试连接的时间。openSocket(建立 socket连接)，scheduleKeepAlive (定时线程 触发keepAlive 事件)，scheduleWriteTimeout(定时检查写超时的话，就关闭连接(这个在5.x是没有的方法))

   **onShutdown** 就不说了

   **reopenSocket** 调了两个方法，关闭连接，打开连接。

   **openSocket**  ，是被 synchronized 修饰的。方法注释说的是反复打开socket，直到线程被打断了。

   ```
   /**
    * Repeatedly tries to open a socket until it is successful,
    * or the hander is stopped, or the handler thread is interrupted.
    *
    * If the socket is non-null when this method returns,
    * then it should be able to be used to send.
    */
   ```

   方法比较长，一点点看

   ```
   while (isStarted() && !Thread.currentThread().isInterrupted()) {
   	// 获取下一个连接的index,多个链接地址的时候多个<destination>标签，默认主从，还有轮询获取和随机
       destinationIndex = connectionStrategy.selectNextDestinationIndex(destinationIndex, destinations.size());
       long startWallTime = System.currentTimeMillis();
       Socket tempSocket = null;
       OutputStream tempOutputStream = null;
       
       
       /*
        * Choose next server
        */
       InetSocketAddress currentDestination = destinations.get(destinationIndex);
       try {
           /*
            * Update peerId (for status message)
            */
           peerId = "Log destination " + currentDestination + ": ";
           
           /*
            * Delay the connection attempt if the last attempt to the selected destination
            * was less than the reconnectionDelay.
            * 判断最后一次尝试连接的时间和延迟重连比较,如果上一次重试的时间小于30s,会提示并在30减去重试时间后，发起重连
            */
           final long millisSinceLastAttempt = startWallTime - destinationAttemptStartTimes[destinationIndex];
           if (millisSinceLastAttempt < reconnectionDelay.getMilliseconds()) {
               final long sleepTime = reconnectionDelay.getMilliseconds() - millisSinceLastAttempt;
               if (errorCount < MAX_REPEAT_CONNECTION_ERROR_LOG * destinations.size()) {
                   addWarn(peerId + "Waiting " + sleepTime + "ms before attempting reconnection.");
               }
               try {
                   shutdownLatch.await(sleepTime, TimeUnit.MILLISECONDS);
                   if (!isStarted()) {
                       return;
                   }
               } catch (InterruptedException ie) {
                   Thread.currentThread().interrupt();
                   addWarn(peerId + "connection interrupted. Will no longer attempt reconnection.");
                   return;
               }
               // reset the start time to be after the wait period.
               startWallTime = System.currentTimeMillis();
           }
           // 更新当前index的最后重试时间
           destinationAttemptStartTimes[destinationIndex] = startWallTime;
           
           /*
            * Set the SO_TIMEOUT so that SSL handshakes will timeout if they take too long.
            *
            * Note that SO_TIMEOUT only applies to reads (which occur during the handshake process).
            */
           tempSocket = socketFactory.createSocket();
           tempSocket.setSoTimeout(acceptConnectionTimeout);
           /*
            * currentDestination is unresolved, so a new InetSocketAddress
            * must be created to resolve the hostname.
            */
           tempSocket.connect(new InetSocketAddress(getHostString(currentDestination), currentDestination.getPort()), acceptConnectionTimeout);
           
           /*
            * Trigger SSL handshake immediately and declare the socket unconnected if it fails
            */
           if (tempSocket instanceof SSLSocket) {
               ((SSLSocket)tempSocket).startHandshake();
           }
           
           /*
            * Issue #218, make buffering the output stream optional.
            */
           tempOutputStream = writeBufferSize > 0
                   ? new BufferedOutputStream(tempSocket.getOutputStream(), writeBufferSize)
                   : tempSocket.getOutputStream();
           
           if (getLogback11Support().isLogback11OrBefore()) {
               getLogback11Support().init(encoder, tempOutputStream);
           }
           
           addInfo(peerId + "connection established.");
           
           this.socket = tempSocket;
           this.outputStream = tempOutputStream;
           
           boolean shouldUpdateThreadName = (destinationIndex != connectedDestinationIndex);
           connectedDestinationIndex = destinationIndex;
           connectedDestination = currentDestination;
           
           connectionStrategy.connectSuccess(startWallTime, destinationIndex, destinations.size());
           
           if (shouldUpdateThreadName) {
               /*
                * destination has changed, so update the thread name
                */
               updateCurrentThreadName();
           }
           // 默认的schedule线程池，每10s触发一次，读取server的返回
           this.readerFuture = scheduleReaderCallable(
                   new ReaderCallable(tempSocket.getInputStream()));
           
           fireConnectionOpened(this.socket);
           
           return;
           
       } catch (Exception e) {
           CloseUtil.closeQuietly(tempOutputStream);
           CloseUtil.closeQuietly(tempSocket);
           
           connectionStrategy.connectFailed(startWallTime, destinationIndex, destinations.size());
           fireConnectionFailed(currentDestination, e);
           
           /*
            * Avoid spamming status messages by checking the MAX_REPEAT_CONNECTION_ERROR_LOG.
            */
           if (errorCount++ < MAX_REPEAT_CONNECTION_ERROR_LOG * destinations.size()) {
               addWarn(peerId + "connection failed.", e);
           }
       }
   }
   ```

   **scheduleKeepAlive** 维持连接的，需要在xml中配置 <keepAliveDuration>，默认不触发这个方法

   **scheduleWriteTimeout** 监测写入超时的

   **ReaderCallable.call** 读取服务器的流，没有返回空。但是! 触发定时线程池往 Disruptor 中触发一个空事件。

   其实作者也说了触发空事件就是为了 keepalive，触发的时候会判断 future是否结束，结束的话重新打开socket。

   如果没有这个方法，会在下次触发 onEvent时重新连接，所以为了尽快打开socket，作者加了这个折中的方案。

   ```
   @Override
   public Void call() throws Exception {
       updateCurrentThreadName();
       try {
           while (true) {
               try {
                   if (inputStream.read() == -1) {
                       /*
                        * End of stream reached, so we're done.
                        */
                       return null;
                   }
               } catch (SocketTimeoutException e) {
                   /*
                    * ignore, and try again
                    */
               } catch (Exception e) {
                   /*
                    * Something else bad happened, so we're done.
                    */
                   throw e;
               }
           }
       } finally {
           if (!Thread.currentThread().isInterrupted()) {
               getExecutorService().submit(() -> {
                   /*
                    * https://github.com/logstash/logstash-logback-encoder/issues/341
                    *
                    * Pro-actively trigger the event handler's onEvent method in the handler thread
                    * by publishing a null event (which usually means a keepAlive event).
                    *
                    * When onEvent handles the event in the handler thread,
                    * it will detect that readerFuture.isDone() and reopen the socket.
                    *
                    * Without this, onEvent would not be called until the next event,
                    * which might not occur for a while.
                    * So, this is really just an optimization to reopen the socket as soon as possible.
                    *
                    * We can't reopen the socket from this thread,
                    * since all socket open/close must be done from the event handler thread.
                    *
                    * There is a potential race condition here as well, since
                    * onEvent could be triggered before the readerFuture completes.
                    * We reduce (but not eliminate) the chance of that happening by
                    * scheduling this task on the executorService.
                    */
                   getDisruptor().getRingBuffer().tryPublishEvent(getEventTranslator(), null);
               });
           }
       }
   }
   ```

   其实看完这块代码，我的问题就破案了。

   每隔10s发起重连是来源于这个地方触发的空事件，也是正常的。期间很有可能是服务器断开了连接，之后发起了重连。

3. 对上面的流程梳理下

   启动的时候：创建连接、定时心跳维护连接(默认关闭)、定时监测写入超时(默认100ms)

   创建连接：看上次尝试连接时间是否超过30s，没超过的话，等待20s后重连，超过的话立即重连。定时10s的单个线程读取socket的输入流，读取完毕后触发 Disruptor 一个空事件。

   触发事件的时候：循环5次，判断当前的连接状态(线程状态和socket状态)，关闭：调用关闭连接和创建连接。开启：调用写入方法。

   写入方法：判断是心跳维护还是正常事件，心跳维护写换行符，正常事件写入事件值。如果是批量终结，调用 flush ，刷新流。

4. 解决方案

   虽然问题找到了，由于猜测是服务端释放连接导致的这个问题，所以没什么好的解决方案，粗暴一点，直接改了 logback-encoder 的日志级别，改为ERROR，看不到WARN日志了，有点骗自己的意思...

   当我写完整个的时候发现了真正的问题所在...











