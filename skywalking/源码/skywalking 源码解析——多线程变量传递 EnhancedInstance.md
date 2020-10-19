# skywalking 源码解析——多线程变量传递 EnhancedInstance

大家好，我是烤鸭：

​	今天分享下 skywalking源码，正好自己用到相关的内容了。

1.  拦截点

   三个主要的拦截器、构造方法、静态方法和示例方法，每个切面里都可以重写这些方法，并且指定进入的拦截器。

   ![1](E:\my\blog\skywalking\源码\1.png)

2.  trace 相关内容

   建议观看这篇博客，写的很详细了。

   http://www.iocoder.cn/SkyWalking/agent-collect-trace/?vip&guanfang#

   我也简单写下吧，其实 skywalking 也是借鉴了 open-trace的思路，这是源码 https://github.com/opentrace-community 

   一条链路可能会有多个span(记录当前节点的信息比如 ip、端口、请求时间、当前span的链路id，以及需要上下文传递的信息  全局链路id、header中的参数)

   span之间需要关联，会有 parent和child的概念，在多线程的场景下，可能会有 一个parent多个child的情况。

   CarrierItem 用于上下文传递参数，结构是 key、value，还有自身节点，类似链表的结构

   ContextManager 全局 span管理器，用于创建不同类型 、销毁、复制 span

   ContextSnapshot span的快照版，用于不同线程间使用

   SW8CarrierItem skywalking 8.x 版本特有的，会随着版本升级类名会改变，对  CarrierItem  包装，和当前作用域以及上下文绑定

   TracingContext 链路管理器，看一下类的注释吧。通过 ContextCarrier 和 ContextSnapshot 传递参数生成 TraceSegmentRef，利用的方法是 inject  和 extract （从前一个元素中获取参数）

   ```
    * In skywalking core concept, FOLLOW_OF is an abstract concept when cross-process MQ or cross-thread async/batch tasks
    * happen, we used {@link TraceSegmentRef} for these scenarios. Check {@link TraceSegmentRef} which is from {@link
    * ContextCarrier} or {@link ContextSnapshot}.
   ```

   ![2](E:\my\blog\skywalking\源码\2.png)

   

3. 说下 EnhancedInstance

   这个类出现在所有切面方法的请求参数，这就是作者做的优化。跟上面一样，构造器拦截模板、实例方法和静态方法。

   ![3](E:\my\blog\skywalking\源码\3.png)

   这几个类是被放到启动类加载器加载。 InstanceMethodsAroundInterceptor INTERCEPTOR 这个变量是在 prepareJREInstrumentation 进行的初始化。至此这几个拦截模板会拦截所有的方法，如果是有拦截器的再按拦截的处理，否则直接放行。除了更好的异常处理、还可以传递参数就是 EnhancedInstance

   ![4](E:\my\blog\skywalking\源码\4.png)

   

4.  以 org.apache.skywalking.apm.plugin.hystrix.v1 插件为例说一下  EnhancedInstance 的作用

   简单介绍下 hystrix ，是一个很好的管理降级和熔断的插件，可以自定义接口超时 走熔断或 超时逻辑，并且可以重写 回滚时的方法，避免了由于下游服务不可用拖垮了上游服务。

   看下 HystrixCommandInstrumentation 拦截的是 com.netflix.hystrix.HystrixCommand 类的 run 和 getFallback 方法。

   看下 run的拦截器。

   从 EnhancedInstance 中获取 enhanceRequireObjectCache 自定义的传递变量。为了避免异步线程，将当前span的快照版放到自定义传递变量中。

   ```
   // Because of `fall back` method running in other thread. so we need capture concurrent span for tracing.
   enhanceRequireObjectCache.setContextSnapshot(ContextManager.capture()); 
   ```

   ![6](E:\my\blog\skywalking\源码\6.png)

   再看下 getFallback 的拦截器

   ![7](E:\my\blog\skywalking\源码\7.png)

    continue 方法就是 从父线程获取context ，创建子节点

   ![5](E:\my\blog\skywalking\源码\5.png)

5.  最后说几句

   其实不管是跨进程还是垮线程，只要拦截了进程或线程转换的入口，将  EnhancedInstance 传入当前快照对象，这样子线程/进程 是可以获取到东西的。就比如 new Thread的时候，拦截了构造器，EnhancedInstance .set(snapshot) 下游就可以获取到了。

