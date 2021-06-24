# skywalking oap-server 域名配置

大家好，我是烤鸭：

​	由于skywalking 的 -Dskywalking.collector.backend_service 的后端服务过多，想通过配置域名的方式简化上报端agent配置，也更灵活。

1.  报错了，先看代码

   报错信息：

   ```
   org.apache.skywalking.apm.dependencies.io.grpc.StatusRuntimeException: INTERNAL: http2 exception
   at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:262)
   at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:243)
   at org.apache.skywalking.apm.dependencies.io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:156)
   at org.apache.skywalking.apm.network.language.agent.v3.JVMMetricReportServiceGrpc$JVMMetricReportServiceBlockingStub.collect(JVMMetricReportServiceGrpc.java:181)
   at org.apache.skywalking.apm.agent.core.jvm.JVMMetricsSender.run(JVMMetricsSender.java:82)
   at org.apache.skywalking.apm.util.RunnableWithExceptionProtection.run(RunnableWithExceptionProtection.java:33)
   at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
   at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308)
   at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
   at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
   at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
   at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   at java.lang.Thread.run(Thread.java:748)
   Caused by: org.apache.skywalking.apm.dependencies.io.netty.handler.codec.http2.Http2Exception: First received frame was not SETTINGS. Hex dump for first 5 bytes: 485454502f
   at org.apache.skywalking.apm.dependencies.io.netty.handler.codec.http2.Http2Exception.connectionError(Http2Exception.java:103)
   at org.apache.skywalking.apm.dependencies.io.netty.handler.codec.http2.Http2ConnectionHandler$PrefaceDecoder.verifyFirstFrameIsSettings(Http2ConnectionHandler.java:338)
   at org.apache.skywalking.apm.dependencies.io.netty.handler.codec.http2.Http2ConnectionHandler$PrefaceDecoder.decode(Http2ConnectionHandler.java:239)
   ```

   因为这个问题，有个人还专门提了issue和提交。

   https://github.com/apache/skywalking/issues/6040
   想使用grpc的话(还没有证书)，开启配置。

   agent.config 中设置 agent.force_tls 为 true 或者脚本中增加 -Dskywalking.agent.force_tls=true

   agent.force_tls=${SW_AGENT_FORCE_TLS:true}

   代码片段：


   GRPCChannelManager.run 方法里的初始化 channel 里的 new TLSChannelBuilder()

   ```
   managedChannel = GRPCChannel.newBuilder(ipAndPort[0], Integer.parseInt(ipAndPort[1]))
                               .addManagedChannelBuilder(new StandardChannelBuilder())
                               .addManagedChannelBuilder(new TLSChannelBuilder())
                               .addChannelDecorator(new AgentIDDecorator())
                               .addChannelDecorator(new AuthenticationDecorator())
                               .build(); 
   ```

2. nginx 配置

   按照以前的 http_proxy的方式是不行的，skywalking 是通过 grpc 方式通信的。

   ```
   # grpc 代理配置
   server {
   	listen 11800 http2; # grpc方式对外暴露端口
   	server_name xxx.xx.com;
   	# access_log logs/access.log main;
   	location / {
   		grpc_pass grpc://127.0.0.1:11800; # 此处配置grpc服务的ip和端口
   	}
   }
   
   # http 代理配置
   server {
   	listen 80; # http方式对外暴露端口
   	server_name localhost;
   	# access_log logs/access.log main;
   	location / {
   		proxy_pass http://127.0.0.1:8210; # 此处配置http服务的ip和端口
   	}
   }
   ```

   重点注意！！！

   **http2 和 http的端口不能重复，除了443**。

   修改配置 -Dskywalking.collector.backend_service=xxx.xx.com:11800