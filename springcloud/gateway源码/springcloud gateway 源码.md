# springcloud gateway 源码



## 官方介绍

官方文档：

看的是 **2.2.5.RELEASE** 版本的

https://docs.spring.io/spring-cloud-gateway/docs/2.2.5.RELEASE/reference/html/

![31](E:\my\blog\springcloud\gateway源码\31.png)

看一下官方这段说明，gateway 使用的是 webflux 和 reactor，有一些同步的包(data,security 可能不支持)。

还有就是需要netty作为服务器，传统的 servlet 模型和 war 包不支持。

工作流程：

![32](E:\my\blog\springcloud\gateway源码\32.png)

简单说就是网关收到客户端的 request 之后经过一系列请求过滤器到代理的服务，收到 response之后，再经过一系列的响应过滤器返回到网关->客户端。

## 源码分析

## reactor 介绍

由于 gateway 是基于 reactor 的，先看下 reactor 的 Flux 和 Mono

reactor 是响应式编程，相对的传统模式 遍历-> 拉取 ->TODO ，而响应式 遍历（发布） -> 订阅者 -> TODO，是基于 发布 —— 订阅的模式。

而 Flux 和 Mono 是 Reactor 中的两个基本概念。都实现 CorePublisher 接口。(用于发布的元素)

Flux 表示的是包含 0 到 N 个元素的异步序列。在该序列中可以包含三种不同类型的消息通知：正常的包含元素的消息、序列结束的消息和序列出错的消息。当消息通知产生时，订阅者中对应的方法 onNext(), onComplete()和 onError()会被调用。Mono 表示的是包含 0 或者 1 个元素的异步序列。该序列中同样可以包含与 Flux 相同的三种类型的消息通知。Flux 和 Mono 之间可以进行转换。对一个 Flux 序列进行计数操作，得到的结果是一个 Mono对象。把两个 Mono 序列合并在一起，得到的是一个 Flux 对象。

## handler 和 filter

看了网上一些资料，从 ReactorHttpHandlerAdapter.apply 开始，debug 看看从哪发起的。

![14](E:\my\blog\springcloud\gateway源码\14.png)

HttpServerHandle.onStateChange 这个是监听连接状态的方法，可以看到请求刚进来的时候是 request_received

![1](E:\my\blog\springcloud\gateway源码\1.png)

HttpWebHandlerAdapter.apply ，包装模式，request 和 response，继续执行 handle方法，生成 exchange 对象，将 request ,response 还有其他的 attribute 封装到一个对象

![15](E:\my\blog\springcloud\gateway源码\15.png)

之后进去 Dispatcherhandler.handle ，这里边会有3个操作，获取路由(getHandler)、请求路由(invokeHandler)、解析结果(handleResult)

![16](E:\my\blog\springcloud\gateway源码\16.png)

获取路由：AbstractHandlerMapping.getHandler (代码没贴,有个判断 是否跨域配置的判断) 中间调用RoutePredicateHandlerMapping.getHandlerInternal，从RouteLocator 获取 路由信息

![3](E:\my\blog\springcloud\gateway源码\3.png)

请求路由：默认走的是  SimpleHandlerAdapter.handle

![4](E:\my\blog\springcloud\gateway源码\4.png)

FilteringWebHandler.handle，获取路由对象和对应的 filter，经过 过滤器链，默认是9个全局过滤器，主要看下 NettyRoutingFilter 和 NettyWriteResponseFilter。

![11](E:\my\blog\springcloud\gateway源码\11.png)

NettyRoutingFilter.filter 就是发送http请求，调用第三方服务，对response 请求头过滤处理，将response 封装到 Mono

![20](E:\my\blog\springcloud\gateway源码\20.png)

NettyWriteResponseFilter.filter

可以看出来这时候netty 已经响应结果，并且往 exchange的reponse 写入

![17](E:\my\blog\springcloud\gateway源码\17.png)

会判断响应的 content-type 是流还是其他的，如果是流的话，写入并刷新。这个地方看一下  wrap 方法，我找了半天都没有找到 gateway 在哪保存的第三方服务的返回值，就在这个byteBuf，封装 DataBuffer，设置到 DataBufferFactory，和 response 绑定。

![18](E:\my\blog\springcloud\gateway源码\18.png)

看一下 wrap 方法

![22](E:\my\blog\springcloud\gateway源码\22.png)

之后的方法调用 ReactorServerHttpResponse.writeWithInternal ，感兴趣可以看出，代码就不贴了。

由于 SimpleHandlerAdapter 返回的是 `Mono.empty()` ，所以不会触发 handleResult 方法。

其中还有HttpServerHandle.onStateChange 监听到其他的状态，也不贴了。

# 总结

以上就是 gateway 收到请求后，到返回一系列的流程，内容也不是特别全。

监听到请求进入：HttpServerHandle.onStateChange(newState = request_received) -> 

包装 request和response：ReactorHttpHandlerAdapter.apply ->

生成 exchange(包含 request和 response)，用于netty请求 :  HttpWebHandlerAdapter.apply ->

获取路由、请求路由、解析结果：Dispatcherhandler.handle ->

​	获取路由 AbstractHandlerMapping.getHandler：-> RoutePredicateHandlerMapping.getHandlerInternal

​	请求路由：SimpleHandlerAdapter.handle：-> FilteringWebHandler.handle

​		默认9个全局过滤器：

​			发送http请求，调用第三方服务，对response 请求头过滤处理，将response 封装到 Mono：NettyRoutingFilter.filter

​			获取netty 响应结果，并且往 exchange的reponse 写入：NettyWriteResponseFilter.filter

​	结果解析：ResponseBodyResultHandler.handleResult （由于 SimpleHandlerAdapter 返回的是 `Mono.empty()`，所以不会触发 handleResult 方法）

写入response并返回：NettyWriteResponseFilter.wrap （byteBuf，封装 DataBuffer，设置到 DataBufferFactory，和 response 绑定）



所以看到有一些想在网关层面修改 response 的需求，只需要增加一个filter 修改 response绑定的 DataBufferFactory 中的 DataBuffer 即可。



还有一些不错的文章推荐：

https://developer.ibm.com/zh/articles/j-cn-with-reactor-response-encode/

https://blog.csdn.net/chengqiuming/article/details/103394337

https://www.jianshu.com/p/c40a757fad01