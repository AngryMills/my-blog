# springcloud gateway 自定义 accesslog elk

大家好，我是烤鸭：

​	最近用 springcloud gateway 时，想使用类似 logback-access的功能，用来做数据统计和图表绘制等等，发现没有类似的功能，只能自己开发了。

环境：

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
```



## 整体思路

logback-access.jar 只需要在logback.xml 配置 LogstashAccessTcpSocketAppender 即可完成异步的日志上报。

如果采用相同的方式，考虑到同一个进程里异步上报占性能。(其实是开发太麻烦了)

这里采用的本地日志文件 + elk。

仿照 logback-access ，定义要收集的字段，开发过滤器收集字段，自定义 logstash.yml。

收集到的字段：
"User-Agent" ： 请求头字段
"server_ip" ：服务器ip
"Content-Length"  ： 请求参数长度
"request_uri" ：请求路径(网关转发的路径)
"host"  ：本机ip
"client_ip" ：请求ip 
"method" ：get/post
"Host" :  请求头字段
"params" ：请求参数
"request_url" ：请求全路径
"thread_name" ：当前线程
"level"  ：日志级别
"cost_time" ： 请求耗时
"logger_name" ：日志类
"Protocol"  ： 请求头字段



## 代码实现

LoggingFilter

```
package com.xxx.gateway.filter;

import com.alibaba.fastjson.JSONObject;
import com.google.common.collect.Maps;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;
import org.springframework.util.MultiValueMap;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.Map;


@Slf4j
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    private static final String UNKNOWN = "unknown";


    private static final String METHOD = "method";
    private static final String PARAMS = "params";
    private static final String REQUEST_URI = "request_uri";
    private static final String REQUEST_URL = "request_url";
    private static final String CLIENT_IP = "client_ip";
    private static final String SERVER_IP = "server_ip";
    private static final String HOST = "Host";
    private static final String COST_TIME = "cost_time";
    private static final String CID = "cid";
    private static final String CONTENT_LENGTH = "Content-Length";
    private static final String PROTOCOL = "Protocol";
    private static final String REQID = "reqid";
    private static final String USER_AGENT = "User-Agent";


    private static final String START_TIME = "gw_start_time";
    private static final String LOGINFOCOLLECTOR = "logInfoCollector";


    /**
     * Process the Web request and (optionally) delegate to the next {@code WebFilter}
     * through the given {@link GatewayFilterChain}.
     *
     * @param exchange the current server exchange
     * @param chain    provides a way to delegate to the next filter
     * @return {@code Mono<Void>} to indicate when request processing is complete
     */
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String requestUrl = request.getPath().toString();
        Map<String, Object> logInfoCollector = Maps.newLinkedHashMap();
        logInfoCollector.put(CLIENT_IP, getIpAddress(request));
        logInfoCollector.put(SERVER_IP, request.getURI().getHost());
        logInfoCollector.put(HOST, getHeaderValue(request, HOST));
        logInfoCollector.put(METHOD, request.getMethodValue());
        logInfoCollector.put(REQUEST_URI, request.getURI().getPath());
        logInfoCollector.put(REQUEST_URL, getRequestUrl(request));
        logInfoCollector.put(PARAMS, request.getURI().getQuery());
        logInfoCollector.put(CID, getHeaderValue(request, CID));
        logInfoCollector.put(CONTENT_LENGTH, request.getHeaders().getContentLength());
        logInfoCollector.put(PROTOCOL, getHeaderValue(request, PROTOCOL));
        logInfoCollector.put(REQID, getHeaderValue(request, REQID));
        logInfoCollector.put(USER_AGENT, getHeaderValue(request, USER_AGENT));

        exchange.getAttributes().put(START_TIME, System.currentTimeMillis());
        exchange.getAttributes().put(LOGINFOCOLLECTOR, logInfoCollector);
    
        String requestMethod = request.getMethodValue();
        String contentType = exchange.getRequest().getHeaders().getFirst(HttpHeaders.CONTENT_TYPE);
        String contentLength = exchange.getRequest().getHeaders().getFirst(HttpHeaders.CONTENT_LENGTH);
        if (HttpMethod.POST.toString().equals(requestMethod) || HttpMethod.PUT.toString().equals(requestMethod)) {
            // 根据请求头，用不同的方式解析Body
            if ((Character.DIRECTIONALITY_LEFT_TO_RIGHT + "").equals(contentLength) || StringUtils.isEmpty(contentType)) {
                MultiValueMap<String, String> getRequestParams = request.getQueryParams();
                log.info("\n 请求url:`{}` \n 请求类型：{} \n 请求参数：{}", requestUrl, requestMethod, getRequestParams);
                return chain.filter(exchange);
            }
            Mono<DataBuffer> bufferMono = DataBufferUtils.join(exchange.getRequest().getBody());
            return bufferMono.flatMap(dataBuffer -> {
                byte[] bytes = new byte[dataBuffer.readableByteCount()];
                dataBuffer.read(bytes);
                String postRequestBodyStr = new String(bytes, StandardCharsets.UTF_8);
                if (contentType.startsWith("multipart/form-data")) {
                    log.info("\n 请求url:`{}` \n 请求类型：{} \n 文件上传", requestMethod);
                } else {
                    log.info("\n 请求url:`{}` \n 请求类型：{} \n 请求参数：{}", requestMethod, postRequestBodyStr);
                }
                // 后续需要用到参数的可以从这个地方获取
                exchange.getAttributes().put("POST_BODY", postRequestBodyStr);
                logInfoCollector.put(PARAMS, postRequestBodyStr);
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
                return chain.filter(exchange.mutate().request(mutatedRequest).build()).then(Mono.fromRunnable(() -> {
                    Long startTime = exchange.getAttribute(START_TIME);
                    Map<String, Object> logInfo = exchange.getAttribute(LOGINFOCOLLECTOR);
                    if (startTime != null && !CollectionUtils.isEmpty(logInfo)) {
                        Long executeTime = (System.currentTimeMillis() - startTime);
                        logInfo.put(COST_TIME, executeTime);
                        log.info(JSONObject.toJSONString(logInfo));
                    }
                }));
            });
        } else if (HttpMethod.GET.toString().equals(requestMethod)
                || HttpMethod.DELETE.toString().equals(requestMethod)) {
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                Long startTime = exchange.getAttribute(START_TIME);
                Map<String, Object> logInfo = exchange.getAttribute(LOGINFOCOLLECTOR);
                if (startTime != null && !CollectionUtils.isEmpty(logInfo)) {
                    Long executeTime = (System.currentTimeMillis() - startTime);
                    logInfo.put(COST_TIME, executeTime);
                    log.info(JSONObject.toJSONString(logInfo));
                }
            }));
        }
        return chain.filter(exchange);
    }

    public String getIpAddress(ServerHttpRequest request) {
        HttpHeaders headers = request.getHeaders();
        String ip = headers.getFirst("x-forwarded-for");
        if (StringUtils.isNotBlank(ip) && !UNKNOWN.equalsIgnoreCase(ip)) {
            // 多次反向代理后会有多个ip值，第一个ip才是真实ip
            if (ip.indexOf(",") != -1) {
                ip = ip.split(",")[0];
            }
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = headers.getFirst("Proxy-Client-IP");
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = headers.getFirst("WL-Proxy-Client-IP");
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = headers.getFirst("HTTP_CLIENT_IP");
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = headers.getFirst("HTTP_X_FORWARDED_FOR");
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = headers.getFirst("X-Real-IP");
        }
        if (StringUtils.isBlank(ip) || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddress().getAddress().getHostAddress();
        }
        return ip;
    }

    private String getRequestUrl(ServerHttpRequest request) {
        String url = request.getURI().toString();
        if (url.contains("?")) {
            url = url.substring(0, url.indexOf("?"));
        }
        return url;
    }

    private String getHeaderValue(ServerHttpRequest request, String key) {
        if (StringUtils.isEmpty(key)) {
            return "";
        }
        HttpHeaders headers = request.getHeaders();
        if (headers.containsKey(key)) {
            return headers.get(key).get(0);
        }
        return "";
    }

    /**
     * Get the order value of this object.
     * <p>Higher values are interpreted as lower priority. As a consequence,
     * the object with the lowest value has the highest priority (somewhat
     * analogous to Servlet {@code load-on-startup} values).
     * <p>Same order values will result in arbitrary sort positions for the
     * affected objects.
     *
     * @return the order value
     * @see #HIGHEST_PRECEDENCE
     * @see #LOWEST_PRECEDENCE
     */
    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}

```

logstash.yml

```yml
input {
      file {
           path => "D:/data/logs/ccc-gateway/*.log"
           type => "ccc-gateway"
           codec => json {
                 charset => "UTF-8"
           }
      }
}

filter {
    json {
        source => "message"
        skip_on_invalid_json => true
        add_field => { "@accessmes" => "%{message}" } 
        remove_field => [ "@accessmes" ]
     }
}

output {
   elasticsearch {
       hosts => "localhost:9200" 
       index => "ccc-gateway_%{+YYYY.MM.dd}"
   }
}
```

上面的 logstash.yml 兼容 json和非json格式，loggingFilter 会保证数据打印为json格式，其他的地方log也可以是非json的。



## 效果如图

accesslog：

![1](.\1.png)

其他的log：

![1](.\2.png)

## 图表绘制

其实netty 作为容器本身也是有 acesslog的，可以开启。

```
-Dreactor.netty.http.server.accessLogEnabled=true
```

AccessLog的log方法直接通过logger输出日志，其日志格式为COMMON_LOG_FORMAT(`{} - {} [{}] "{} {} {}" {} {} {} {} ms`)，分别是address, user, zonedDateTime, method, uri, protocol, status, contentLength, port, duration

![1](.\5.png)

没有请求参数和自定义参数(一般链路id放在请求头里的)和响应参数(这次也没加)，所以算是对accesslog做了改进。下图是访问量和平均耗时，后续还可以加tp99，请求路径等等

访问量：

![1](.\3.png)

平均耗时：

![1](.\4.png)