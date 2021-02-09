# Zuul 1.x 升级 springcloud gateway 2.x 遇到的一点问题



## 介绍

zuul 和springcloud gateway 都是比较优秀的网关，而 zuul 1.x 采用的是 servlet 模型，gate 采用的是 reactor模型，效率和资源上 gateway 要优秀一些。

zuul 和 springcloud 在 filter 架构上类似，都提供了基类 ZuulFilter 和 GlobalFilter，只要继承/实现基类就可以自定义拦截器，并且有排序。



## 升级场景

由于原来的网关项目并没有完全脱离业务场景(使用了部分zuul的拦截器和大量的自定义拦截器，白名单、签名校验等等)，所以改造起来是比较麻烦的。

在全局参数管理上，由于是单线程模型，zuul1 采用 TheadLocal 维护的 RequestContext，而 gateway 需要使用 exchange 传递上下文参数。



### zuul的拦截器改造

原网关的配置有 :

zuul.ignored-services=*
zuul.ignoredPatterns=/abc,/abd
zuul.sensitiveHeaders=key1

zuul.ignored-services=*， zuul有默认的隐射机制 * 表示禁用默认路由，外界无法访问未配置路由的服务
zuul.ignoredPatterns=/abc/aaa,/abd ，如果路由配置了 /abc/**，那么abc/aaa会被过滤，其他访问正常
zuul.sensitiveHeaders=key1，忽略某个敏感的请求头(不向下游传递)

- zuul.ignored-services

springcloud 默认是不会转发未配置的路由，所以第一个不需要考虑。

- zuul.ignoredPatterns=/abc/aaa,/abd 

​    过滤请求路径，gateway没有现成的filter，所以需要自己写一个，我用的 spring security

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

​    WebSecurityConfig

```
package test.gateway.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.stereotype.Component;


@RefreshScope
@Configuration
public class WebSecurityConfig {
    /**
     * 网关入口黑名单
     */
    @Value("${gateway.routes.black:/abc/aaa,/abd}")
    private String[] blacklist;

    /**
     * 网关入口白名单
     */
    @Value("${gateway.routes.white:/*}")
    private String[] whitelist;
   
   
    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        http.csrf().disable()
                .authorizeExchange()
                //需要授权使用的特殊接口,黑白名单顺序很重要，需要授权的放上面
                .pathMatchers(blacklist).authenticated()
                //无需进行权限过滤的请求路径
                .pathMatchers(whitelist).permitAll()
                .anyExchange().authenticated()
                .and().httpBasic()
                .and().formLogin();
        return http.build();
    }

}

```

- zuul.sensitiveHeaders
  这个其实gateway 有现成的filter，只要配置下就可以了，RemoveRequestHeaderGatewayFilterFactory

  调下游服务之前会把这个header去掉，但是我这个场景不符合，只能手写一个filter。

  原因是要过滤前端传过来的某个key(重要的头参数)，正常这个参数会从token中解密获得，如果解密成功，会覆盖前端传过来的(使用的是 RequestContext.addZuulRequestHeader，当然前端也可能没传)。怕的是接口被盗刷，随意传重要的头参数，而不会走解密(不是所以接口都需要解密)，所以需要在前端传了的情况下，remove掉。
  RemoveRequestHeaderGatewayFilterFactory 只是在调下游服务前过滤，会过滤正常的(解密成功覆盖的)。所以需要手写filter，顺序在 解密的filter 之前执行。

  SensitiveHeadersFilter

  ```
  package test.gateway.filters.pre;
  
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.cloud.gateway.filter.GatewayFilterChain;
  import org.springframework.cloud.gateway.filter.GlobalFilter;
  import org.springframework.core.Ordered;
  import org.springframework.http.HttpHeaders;
  import org.springframework.stereotype.Component;
  import org.springframework.web.server.ServerWebExchange;
  import reactor.core.publisher.Mono;
  
  
  @Component
  public class SensitiveHeadersFilter implements GlobalFilter, Ordered {
      
      private static Logger logger = LoggerFactory.getLogger(SensitiveHeadersFilter.class);
      
      /**
       * 过滤请求头
       */
      @Value("${gateway.sensitive.headers:key}")
      private String[] sensitiveHeaders;
      
      /**
       * @param
       * @return int
       * @Author 
       * @Description 只要在 DecryptFilter 之前执行就可以，目的是移除配置的请求头
       * @Date 2021/2/7 16:35
       **/
      @Override
      public int getOrder() {
          return HIGHEST_PRECEDENCE + 1000;
      }
      
      @Override
      public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
          HttpHeaders headers = exchange.getRequest().getHeaders();
          if (headers != null) {
              for (int i = 0; i < sensitiveHeaders.length; i++) {
                  if(headers.containsKey(sensitiveHeaders[i])){
                      headers.remove(sensitiveHeaders[i]);
                  }
              }
          }
          return chain.filter(exchange);
      }
  }
  
  ```

  ### 路由配置

  改造如下：

  ```
  #zuul.routes.aaa.path=/aaa
  #zuul.routes.aaa.url=http://www.test.com/
  spring.cloud.gateway.routes[0].id=aaa
  spring.cloud.gateway.routes[0].filters[0].name=StripPrefix
  spring.cloud.gateway.routes[0].filters[0].args.parts=1
  spring.cloud.gateway.routes[0].predicates[0].name=Path
  spring.cloud.gateway.routes[0].predicates[0].args[pattern]=/aaa
  spring.cloud.gateway.routes[0].uri=http://www.test.com/
  
  #zuul.routes.bbb.path=/bbb/**
  #zuul.routes.bbb.serviceId=bbb
  spring.cloud.gateway.routes[1].id=bbb
  spring.cloud.gateway.routes[1].filters[0].name=StripPrefix
  spring.cloud.gateway.routes[1].filters[0].args.parts=1
  spring.cloud.gateway.routes[1].predicates[0].name=Path
  spring.cloud.gateway.routes[1].predicates[0].args[pattern]=/bbb/**
  spring.cloud.gateway.routes[1].uri=lb://bbb
  ```

  ### 其他逻辑修改

  由于模型不同，获取参数的方式等等都不一样，除了代码上兼容，逻辑上还需要再测试。

  就比如如果zuul想获取post请求body体的参数：

  可以直接使用 (RequestContext)ctx.getRequest().getParameterMap()
  调用的是 com.netflix.zuul.http.HttpServletRequestWrapper.parseRequest() 可以直接解析参数后返回map，而 gateway 里没有类似的方法的，还需要考虑到流读取后再写回的问题。

  类似场景可以参考下面这个过滤器，在后续需要用到参数的地方 可以通过 exchange.getAttributes("POST_BODY") 获取

  HttpRequestGlobalFilter

  ```
  package test.filters.pre;
  
  import java.nio.charset.StandardCharsets;
  
  import test.ZuulFilterAdapter;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.cloud.gateway.filter.GatewayFilterChain;
  import org.springframework.cloud.gateway.filter.GlobalFilter;
  import org.springframework.core.Ordered;
  import org.springframework.core.annotation.Order;
  import org.springframework.core.io.buffer.DataBuffer;
  import org.springframework.core.io.buffer.DataBufferUtils;
  import org.springframework.http.HttpMethod;
  import org.springframework.http.server.reactive.ServerHttpRequest;
  import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
  import org.springframework.stereotype.Component;
  import org.springframework.util.MultiValueMap;
  import org.springframework.web.server.ServerWebExchange;
  
  import lombok.extern.slf4j.Slf4j;
  import reactor.core.publisher.Flux;
  import reactor.core.publisher.Mono;
  
  public class HttpRequestGlobalFilter extends ZuulFilterAdapter {
      
      private static Logger logger = LoggerFactory.getLogger(HttpRequestGlobalFilter.class);
      
      
      @Override
      public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
          ServerHttpRequest request = exchange.getRequest();
          String requestUrl = request.getPath().toString();
          String requestMethod = request.getMethodValue();
          if (HttpMethod.POST.toString().equals(requestMethod) || HttpMethod.PUT.toString().equals(requestMethod)) {
              return DataBufferUtils.join(exchange.getRequest().getBody()).flatMap(dataBuffer -> {
                  byte[] bytes = new byte[dataBuffer.readableByteCount()];
                  dataBuffer.read(bytes);
                  String postRequestBodyStr = new String(bytes, StandardCharsets.UTF_8);
                  String contentType = request.getHeaders().getFirst("Content-Type");
                  if (requestUrl.contains("/web/api/file") || contentType.startsWith("multipart/form-data")) {
                      logger.info("\n 请求url:`{}` \n 请求类型：{} \n 文件上传", requestUrl, requestMethod);
                  } else {
                      logger.info("\n 请求url:`{}` \n 请求类型：{} \n 请求参数：{}", requestUrl, requestMethod, postRequestBodyStr);
                  }
                  exchange.getAttributes().put("POST_BODY", postRequestBodyStr);
                  DataBufferUtils.release(dataBuffer);
                  Flux<DataBuffer> cachedFlux = Flux.defer(() -> {
                      DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(bytes);
                      return Mono.just(buffer);
                  });
                  // 下面的将请求体再次封装写回到request里，传到下一级，否则，由于请求体已被消费，后续的服务将取不到值
                  ServerHttpRequest mutatedRequest = new ServerHttpRequestDecorator(exchange.getRequest()) {
                      @Override
                      public Flux<DataBuffer> getBody() {
                          return cachedFlux;
                      }
                  };
                  // 封装request，传给下一级
                  return chain.filter(exchange.mutate().request(mutatedRequest).build());
              });
          } else if (HttpMethod.GET.toString().equals(requestMethod)
              || HttpMethod.DELETE.toString().equals(requestMethod))
  
          {
              MultiValueMap<String, String> getRequestParams = request.getQueryParams();
              logger.debug("\n 请求url:`{}` \n 请求类型：{} \n 请求参数：{}", requestUrl, requestMethod, getRequestParams);
              return chain.filter(exchange);
          }
          return chain.filter(exchange);
      }
  }
  
  ```

  

## 总结

目前升级还没有完成，坑比想象中的要多...
除了框架本身的问题，还有代码年久失修、业务逻辑等等一系列的问题
代码重构 任重道远啊

