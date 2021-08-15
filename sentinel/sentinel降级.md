# Gateway Sentinel 做网关降级/流控，转发header和cookie

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;Springcloud Gateway Sentinel 做网关降级。



## 环境

springcloud-gateway的网关应用，springboot的服务，nacos作为注册中心

sentinel-dashboard-1.8.2 

最新版下载地址：
https://github.com/alibaba/Sentinel/releases

## 目标

在网关层根据qps对指定路由降级到其他接口。

sentinel 接入的官方wiki：

https://github.com/alibaba/Sentinel/wiki/%E7%BD%91%E5%85%B3%E9%99%90%E6%B5%81

网关代码：
https://gitee.com/fireduck_admin/scg-sentinel

业务代码就不贴了，就是普通的springboot服务，集成nacos注册中心。



## dashboard

dashboard 需要有访问才能显示,访问几次。

![1](.\1.png)

由于没配置网关类型，sentinel 默认是服务，是没有API管理的。

![2](.\2.png)

网关服务启动参数中添加 -Dcsp.sentinel.app.type=1，这回有了。

![2](.\3.png)



## API 管理和流控规则配置

API 添加的地址我这边选的是精确匹配路由，路由就是上图实时监控的地址。

![2](.\4.png)

流程规则选按API分组和QPS进行降级，模式选快速失败。（在网关代码里实现具体快速失败的逻辑，比如调第三方接口降级）

![2](.\5.png)

## scg 代码开发

### 默认流程：

GatewayConfiguration，配置降级时触发异常

```
package com.maggie.demo.scgsentinel.config;

import com.alibaba.csp.sentinel.adapter.gateway.sc.SentinelGatewayFilter;
import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.BlockRequestHandler;
import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.GatewayCallbackManager;
import com.maggie.demo.scgsentinel.handler.SentinelBlockRequestHandler;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.cloud.gateway.handler.FilteringWebHandler;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.codec.ServerCodecConfigurer;
import org.springframework.web.reactive.result.view.ViewResolver;

import javax.annotation.PostConstruct;
import java.util.Collections;
import java.util.List;

@Order(1)
@Configuration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.REACTIVE)
public class GatewayConfiguration implements ApplicationContextAware {
    
    @Autowired
    protected FilteringWebHandler filteringWebHandler;
    
    @Value("${spring.application.name}")
    private String PJ_NAME;
    
    public String getPJ_NAME() {
        return PJ_NAME;
    }
    
    public void setPJ_NAME(String PJ_NAME) {
        this.PJ_NAME = PJ_NAME;
    }
    
    private final List<ViewResolver> viewResolvers;
    private final ServerCodecConfigurer serverCodecConfigurer;
    private final BlockRequestHandler blockRequestHandler;
    private ApplicationContext applicationContext;
    
    
    public GatewayConfiguration(ObjectProvider<List<ViewResolver>> viewResolversProvider,
                                ServerCodecConfigurer serverCodecConfigurer) {
        this.viewResolvers = viewResolversProvider.getIfAvailable(Collections::emptyList);
        this.serverCodecConfigurer = serverCodecConfigurer;
        this.blockRequestHandler = new SentinelBlockRequestHandler();
    }
    // 不配置异常处理,采用默认的提示
//    @Bean
//    @Order(Ordered.HIGHEST_PRECEDENCE)
//    public SentinelGatewayBlockExceptionHandler sentinelGatewayBlockExceptionHandler() {
//        // Register the block exception handler for Spring Cloud Gateway.
//        return new GatewayBlockExceptionHandler(viewResolvers, serverCodecConfigurer);
//    }
    
    @Bean
    @Order(-1)
    public GlobalFilter sentinelGatewayFilter() {
        return new SentinelGatewayFilter();
    }
    
    @PostConstruct
    public void doInit() throws Exception {
        
        initBlockStrategy();
        
    }
    
    private void initBlockStrategy() {
        String[] beanNamesForType = applicationContext.getBeanNamesForType(BlockRequestHandler.class);
        if (beanNamesForType != null && beanNamesForType.length > 0) {
            GatewayCallbackManager.setBlockHandler(applicationContext.getBean(beanNamesForType[0], BlockRequestHandler.class));
        } else {
            GatewayCallbackManager.setBlockHandler(blockRequestHandler);
        }
    }
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
    
    /**
     * @Description 加这个注解可以直接通过注册中心的application.name直接调用服务
     * @Date 2021/8/12 14:28
     **/
    @Bean
    @LoadBalanced
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}
```

下图是触发降级和默认的限流异常提示

![2](.\8.png)

![](.\9.png)

### 自定义流程：
增加异常处理和降级逻辑

GatewayBlockExceptionHandler

```
package com.maggie.demo.scgsentinel.handler;

import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.GatewayCallbackManager;
import com.alibaba.csp.sentinel.adapter.gateway.sc.exception.SentinelGatewayBlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.util.function.Supplier;
import org.springframework.http.HttpStatus;
import org.springframework.http.codec.HttpMessageWriter;
import org.springframework.http.codec.ServerCodecConfigurer;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.reactive.result.view.ViewResolver;
import org.springframework.web.server.ResponseStatusException;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;


public class GatewayBlockExceptionHandler extends SentinelGatewayBlockExceptionHandler {

    private List<ViewResolver> viewResolvers;
    private List<HttpMessageWriter<?>> messageWriters;

    public GatewayBlockExceptionHandler(List<ViewResolver> viewResolvers, ServerCodecConfigurer serverCodecConfigurer) {
        super(viewResolvers, serverCodecConfigurer);
        this.viewResolvers = viewResolvers;
        this.messageWriters = serverCodecConfigurer.getWriters();
    }

    private Mono<Void> writeResponse(ServerResponse response, ServerWebExchange exchange) {
        return response.writeTo(exchange, contextSupplier.get());
    }

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        /**
         * 处理一下504
         */
        if (ex != null && ex instanceof ResponseStatusException) {
            ResponseStatusException responseStatusException = (ResponseStatusException) ex;
            //只处理超时的情况
            if (HttpStatus.GATEWAY_TIMEOUT.equals(responseStatusException.getStatus())) {
                return handleBlockedRequest(exchange, ex).flatMap(response -> writeResponse(response, exchange));
            }
        }

        if (exchange.getResponse().isCommitted()) {
            return Mono.error(ex);
        }
        // This exception handler only handles rejection by Sentinel.
        if (!BlockException.isBlockException(ex)) {
            return Mono.error(ex);
        }

        return handleBlockedRequest(exchange, ex)
                .flatMap(response -> writeResponse(response, exchange));
    }


    private Mono<ServerResponse> handleBlockedRequest(ServerWebExchange exchange, Throwable throwable) {
        return GatewayCallbackManager.getBlockHandler().handleRequest(exchange, throwable);
    }

    private final Supplier<ServerResponse.Context> contextSupplier = () -> new ServerResponse.Context() {
        @Override
        public List<HttpMessageWriter<?>> messageWriters() {
            return GatewayBlockExceptionHandler.this.messageWriters;
        }

        @Override
        public List<ViewResolver> viewResolvers() {
            return GatewayBlockExceptionHandler.this.viewResolvers;
        }
    };
}
```

SentinelBlockRequestHandler

```
package com.maggie.demo.scgsentinel.handler;

import com.alibaba.csp.sentinel.adapter.gateway.sc.callback.BlockRequestHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.context.ApplicationContext;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.util.MultiValueMap;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.server.ServerResponse;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.HashSet;
import java.util.Set;

import static org.springframework.web.reactive.function.BodyInserters.fromValue;

@Component
@Slf4j
public class SentinelBlockRequestHandler implements BlockRequestHandler, Ordered {
  
  public static final Set<String> FLOW_LIMIT_API = new HashSet<>(32);
  
  static {
    FLOW_LIMIT_API.add("/test/api/tab/test");
  }

  public static final String CODE_SYSTEM_BUSY = "11004";

  public static final String MSG_SYSTEM_BUSY = "网络开小差了,请稍后重试.";

  @Override
  public int getOrder() {
    return LOWEST_PRECEDENCE;
  }
  
  @Autowired
  private WebClient.Builder webClientBuilder;
  
  @Autowired
  private ApplicationContext applicationContext;
  
  @Override
  public Mono<ServerResponse> handleRequest(ServerWebExchange exchange, Throwable ex) {

    String requestUrl = exchange.getRequest().getPath().toString();
    Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
    if (FLOW_LIMIT_API.contains(requestUrl)) {
      return handleRequestDetail(exchange,ex);
    }

    // JSON result by default.
    return ServerResponse.status(HttpStatus.OK)
        .contentType(MediaType.APPLICATION_JSON)
        .body(fromValue(buildErrorResult(ex)));
  }

  private ErrorResult buildErrorResult(Throwable ex) {
    return new ErrorResult(CODE_SYSTEM_BUSY, MSG_SYSTEM_BUSY);
  }

  private static class ErrorResult {

    private final String status;
    private final String message;

    public String getStatus() {
      return status;
    }

    ErrorResult(String status, String message) {
      this.status = status;
      this.message = message;
    }

    public String getMessage() {
      return message;
    }
  }
  
  
  public Mono<ServerResponse> handleRequestDetail(ServerWebExchange exchange, Throwable ex) {
    String requestUrl = exchange.getRequest().getURI().getRawPath();
    System.out.println("requestUrl = " + requestUrl);
    Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
    // post请求的参数需要自己写filter 获取,这里只是获取的get请求参数
    MultiValueMap<String, String> getParams = exchange.getRequest().getQueryParams();
    
    // 转发到第三方接口,这里是配置的时候route-id和注册中心服务的application.name(lb://data-ballast-api) 一致了
    // 原地址 localhost:8083/test/api/tab/test, 降级地址 http://data-ballast-api/api/tab/test/sentinel
    StringBuilder newUri = new StringBuilder("http://")
            .append(route.getId())
            .append(requestUrl+"/sentinel");
    
    log.info("限流uri: {}, query参数: {}, 降级至: uri {}", requestUrl, getParams, newUri.toString());
    //
    return webClientBuilder.build()
            .post() //
            .uri(newUri.toString())
            // 传递 request header
            .headers(newHeaders -> newHeaders.putAll(exchange.getRequest().getHeaders()))
            .bodyValue(getParams)
            .exchangeToMono(response -> {
              return response.bodyToMono(String.class)
                      .defaultIfEmpty("")
                      .flatMap(body -> {
                        return ServerResponse.status(response.statusCode())
                                // 传递 response header,避免cookie在网关层丢失
                                .headers(it -> {
                                  it.addAll(response.headers().asHttpHeaders());
                                })
                                .bodyValue(body);
                      });
            });
  }
}
```

自定义三方降级接口：

![2](.\10.png)

降级成功：
![2](.\12.png)

![2](.\11.png)

## 待优化

上面已经基本实现了网关层面进行流控。

还有几个地方可以优化：

- **流控规则等配置的持久化**：sentinel 存的规则是默认存到内存里的，一旦重启了服务(网关或者普通的业务服务)，规则需要重新配置。持久化可以选择 apollo或者 nacos。(一般的配置中心)
- 针对不同方式的参数获取待完善（POST请求、文件上传 等等）
- 针对不同的route走不同的降级策略（代码优化，可以使用策略模式）
- 不同异常的处理，比如通用的 GatewayConfiguration 是处理了所有异常进行的 handleRequest 处理，可能有些请求不适合按降级处理(比如超时之类的)

