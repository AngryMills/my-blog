# springcloud gateway 性能调优

大家好，我是烤鸭：
		springboot 2.2.6.RELEASE	

​        springcloud Hoxton.SR6



## 业务场景

有一个关于红包的业务场景需要2w qps的支持，压测过程中发现网关的qps一直没法上去(7台网关，单台1000，cpu未耗尽)，而下游服务单独测试的时候单台qps 5000不止。

机器配置是 8c 16g，单个服务用了8g，就以往的经验看，这种配置单台qps 5k是没问题的(不考虑中间件、网速、流量等)，为什么网关上不去，而cpu还未打满呢。





