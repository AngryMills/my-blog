# Gateway Sentinel 做网关降级，转发header和cookie

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;Springcloud Gateway Sentinel 做网关降级。



## 环境

springcloud-gateway的网关应用，springboot的服务，nacos作为注册中心

sentinel-dashboard-1.8.2 

## 目标

在网关层根据qps对指定路由降级到其他接口。



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

先试一下，默认的快速失败

