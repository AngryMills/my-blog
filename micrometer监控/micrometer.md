# 从源码角度分析下 micrometer

官方文档：

http://micrometer.io/docs

github：

https://github.com/micrometer-metrics/micrometer

springboot集成：
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-metrics

## 监测信息

jvm、memory、cpu、tomcat 等等。

网上关于 springboot 集成和使用，也有很多文章，这里就不赘述了。

本篇文章主要是源码分析和简单场景使用。

# 源码分析

说几个核心类：

```
MeterRegistry
```

注册表中心用来管理应用的注册表，再遍历注册表获取指标。

```
public abstract class MeterRegistry {
    protected final Clock clock;
    private final Object meterMapLock = new Object();
    private volatile MeterFilter[] filters = new MeterFilter[0];
    private final List<Consumer<Meter>> meterAddedListeners = new CopyOnWriteArrayList<>();
    private final List<Consumer<Meter>> meterRemovedListeners = new CopyOnWriteArrayList<>();
    private final Config config = new Config();
    private final More more = new More();
    //...
}
```

```
MeterBinder
```

绑定容器内部测量的父类接口。(所有需要测量类的重写这个接口就行)

```
/**
 * Binders register one or more metrics to provide information about the state
 * of some aspect of the application or its container.
 * <p>
 * Binders are enabled by default if they source data for an alert
 * that is recommended for a production ready app.
 */
public interface MeterBinder {
    void bindTo(@NonNull MeterRegistry registry);
}
```

```
Gauge
```

Meter的子类，Meter是测量指标（可以理解为值对象），而Gauge是指标的瞬时值（普通的对象）。

```
/**
 * A gauge tracks a value that may go up or down. The value that is published for gauges is
 * an instantaneous sample of the gauge at publishing time.
 *
 * @author Jon Schneider
 */
public interface Gauge extends Meter {
    /**
     * @param name The gauge's name.
     * @param obj  An object with some state or function which the gauge's instantaneous value
     *             is determined from.
     * @param f    A function that yields a double value for the gauge, based on the state of
     *             {@code obj}.
     * @param <T>  The type of object to gauge.
     * @return A new gauge builder.
     */
    static <T> Builder<T> builder(String name, @Nullable T obj, ToDoubleFunction<T> f) {
        return new Builder<>(name, obj, f);
    }
    //...
}
```

我们以其中某个类分析下：

```
JvmThreadMetrics
```

监测 JVM 线程变化的

```
@Override
    public void bindTo(MeterRegistry registry) {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();

        Gauge.builder("jvm.threads.peak", threadBean, ThreadMXBean::getPeakThreadCount)
                .tags(tags)
                .description("The peak live thread count since the Java virtual machine started or peak was reset")
                .baseUnit(BaseUnits.THREADS)
                .register(registry);

        Gauge.builder("jvm.threads.daemon", threadBean, ThreadMXBean::getDaemonThreadCount)
                .tags(tags)
                .description("The current number of live daemon threads")
                .baseUnit(BaseUnits.THREADS)
                .register(registry);

        Gauge.builder("jvm.threads.live", threadBean, ThreadMXBean::getThreadCount)
                .tags(tags)
                .description("The current number of live threads including both daemon and non-daemon threads")
                .baseUnit(BaseUnits.THREADS)
                .register(registry);

        for (Thread.State state : Thread.State.values()) {
            Gauge.builder("jvm.threads.states", threadBean, (bean) -> getThreadStateCount(bean, state))
                    .tags(Tags.concat(tags, "state", getStateTagValue(state)))
                    .description("The current number of threads having " + state + " state")
                    .baseUnit(BaseUnits.THREADS)
                    .register(registry);
        }
    }
```

创建 Gauge 内部类builder 和 当前的 registry 绑定，我们看下方法注释怎么说的。

对单例的注册表添加一个指标测量对象，或者返回一个已存在的。返回的是当前注册表唯一的，每个注册表保证相同名字和标签只创建一个指标测量对象。

```
 		/**
         * Add the gauge to a single registry, or return an existing gauge in that registry. The returned
         * gauge will be unique for each registry, but each registry is guaranteed to only create one gauge
         * for the same combination of name and tags.
         *
         * @param registry A registry to add the gauge to, if it doesn't already exist.
         * @return A new or existing gauge.
         */
        public Gauge register(MeterRegistry registry) {
            return registry.gauge(new Meter.Id(name, tags, baseUnit, description, Type.GAUGE, syntheticAssociation), obj,
                    strongReference ? new StrongReferenceGaugeFunction<>(obj, f) : f);
        }
```

上面就是一个收集信息的过程，简单来说 收集到的信息放到注册表，需要的时候来取。看一下springboot的actuator的源码。

先说一下 endpoint 这个关键的包。

其中一个 Endpoint 注解（执行器断点），带有这个注解的执行器会被公开。

```
/**
 * Identifies a type as being an actuator endpoint that provides information about the
 * running application. Endpoints can be exposed over a variety of technologies including
 * JMX and HTTP.
 *
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Endpoint {

	/**
	 * The id of the endpoint (must follow {@link EndpointId} rules).
	 * @return the id
	 * @see EndpointId
	 */
	String id() default "";

	/**
	 * If the endpoint should be enabled or disabled by default.
	 * @return {@code true} if the endpoint is enabled by default
	 */
	boolean enableByDefault() default true;

}
```

简单来说：

带有这个注解的类，会被增加到servlet，路径是 basePath+注解的id属性。源码是这个类。

```
EndpointDiscoverer.createEndpointBeans
```

```
private Collection<EndpointBean> createEndpointBeans() {
   Map<EndpointId, EndpointBean> byId = new LinkedHashMap<>();
   String[] beanNames = BeanFactoryUtils.beanNamesForAnnotationIncludingAncestors(this.applicationContext,
         Endpoint.class);
   for (String beanName : beanNames) {
      if (!ScopedProxyUtils.isScopedTarget(beanName)) {
         EndpointBean endpointBean = createEndpointBean(beanName);
         EndpointBean previous = byId.putIfAbsent(endpointBean.getId(), endpointBean);
         Assert.state(previous == null, () -> "Found two endpoints with the id '" + endpointBean.getId() + "': '"
               + endpointBean.getBeanName() + "' and '" + previous.getBeanName() + "'");
      }
   }
   return byId.values();
}
```

找到带有Endpoint注解的类，比如 MetricsEndpoint.class，metric 就是请求 /actuator/metrics/jvm.gc.max.data.size 调用的方法。

```
@Endpoint(id = "metrics")
public class MetricsEndpoint {
	//...
   @ReadOperation
   public MetricResponse metric(@Selector String requiredMetricName, @Nullable List<String> tag) {
      List<Tag> tags = parseTags(tag);
      Collection<Meter> meters = findFirstMatchingMeters(this.registry, requiredMetricName, tags);
      if (meters.isEmpty()) {
         return null;
      }
      Map<Statistic, Double> samples = getSamples(meters);
      Map<String, Set<String>> availableTags = getAvailableTags(meters);
      tags.forEach((t) -> availableTags.remove(t.getKey()));
      Meter.Id meterId = meters.iterator().next().getId();
      return new MetricResponse(requiredMetricName, meterId.getDescription(), meterId.getBaseUnit(),
            asList(samples, Sample::new), asList(availableTags, AvailableTag::new));
   }

  //...

}
```

知道Endpoint，尝试写自己的监控指标。 

# 实现自定义micrometer

简单点的方式：

自定义 RedisMetric 重写 bindTo方法，访问 /metric/redis.get.count 就可以看到指标了

```
package com.maggie.measure.micrometer.metric;

import java.util.concurrent.atomic.AtomicInteger;

import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.stereotype.Component;

@Component
public class RedisMetric implements MeterBinder {

    public static AtomicInteger atomicInteger = new AtomicInteger(0);

    @Override
    public void bindTo(MeterRegistry meterRegistry) {
        Gauge.builder("redis.get.count", atomicInteger, c -> c.get())
                .tags("host", "localhost")
                .description("demo of custom meter binder")
                .register(meterRegistry);
    }

}
```

实现一个监控redis get/set 方法的次数统计。

访问 http://localhost:8081/get 9次

结果如图。

稍微复杂点，实现拦截 redis get/set 方法，统计get/set 方法 的key以及每个key 的调用次数。

自定义 endponit 实现。

```
/**
 * @program: micrometer-demo
 * @description: redis监控断点
 */
@Component
@Endpoint(id = "redis")
public class RedisRegistryEndpoint {
    private final MeterRegistry registry;

    public RedisRegistryEndpoint(MeterRegistry registry) {
        this.registry = registry;
    }

    @ReadOperation
    public String home() {
        Set<String> set = new HashSet<>();
        set.add("redis.get.info");
        set.add("redis.set.info");
        return JSONObject.toJSONString(set);
    }

    @ReadOperation
    public String metric(@Selector String tagName) {
        tagName = tagName.replaceAll("\\.", "")
                .replaceAll("redis", "").replaceAll("info", "");
        return JSONObject.toJSONString(RedisMetric.param.get(tagName));
    }
}
```

统计次数和key的是通过aop实现的。

```
package com.maggie.measure.micrometer.aspect;

import com.maggie.measure.micrometer.metric.RedisMetric;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Aspect
@SuppressWarnings("all")
@Component("redisApiAspect")
public class RedisApiAspect {

    public static Map incrMap = new HashMap<>();

    @Pointcut("execution(public * com.maggie.measure.micrometer.service.RedisOpsValService.*(..))")
    private void redisApi() {
    }

    @Around("redisApi()")
    public Object doProfiling(ProceedingJoinPoint point) throws Throwable {

        long initTime = System.currentTimeMillis();
        long sTime = initTime, eTime = initTime;


        MethodSignature methodSignature = null;
        Object proceed = null;
        try {
            methodSignature = (MethodSignature) point.getSignature();
        } finally {
            String met = methodSignature.getName(); // 拦截方法名称
            Object[] args = point.getArgs(); // 拦截的方法参数
            proceed = point.proceed();
            if ("get".equals(met)) {
                RedisMetric.atomicGetInteger.getAndIncrement();
            }
            if ("set".equals(met)) {
                RedisMetric.atomicSetInteger.getAndIncrement();
            }
            if (RedisMetric.param.get(met) != null) {
                Map<String, Object> metMap = RedisMetric.param.get(met);
                incrMap.put(met + "incr", Double.valueOf((Integer) metMap.getOrDefault(met + "incr", 0) + 1));
                int incr = (Integer) incrMap.getOrDefault(met + args[0] + "incr", 0) + 1;
                incrMap.put(met + args[0] + "incr", incr);
                if (args != null && args[0] instanceof String) {
                    metMap.put((String) args[0], incr);
                }
            } else {
                Map<String, Object> metMap = new HashMap<>();
                incrMap.put(met + "incr", Double.valueOf((Integer) metMap.getOrDefault(met + "incr", 0) + 1));
                int incr = (Integer) incrMap.getOrDefault(met + args[0] + "incr", 0) + 1;
                if (incrMap.get(met + args[0] + "incr") == null) {
                    incrMap.put(met + args[0] + "incr", incr);
                }
                if (args != null && args[0] instanceof String) {
                    metMap.put((String) args[0], incr);
                }
                RedisMetric.param.put(met, metMap);
            }
        }
        return proceed;
    }
}
```

结果如图：

可以看出 查哪些参数(get.info,set.info)，以及 get/set 的key和单个key的调用次数。



源代码地址：

https://gitee.com/fireduck_admin/micrometer-demo/tree/master

# 总结

实现系统监控有很多方式，micrometer-metrics 是个不错的开源框架，而且springboot 扩展不错。

关于拉式(提供接口，外部调用)还是推式(上报，http/socket 等等)方案的选择，还是看自己的业务场景。

量大(服务器数量多且服务多)的时候无论采用哪种都不太好，不仅对性能损耗，而且维护麻烦，不易升级。

这里只是看了 metrics 源码， 做了一个简单场景的尝试，其实可做的方向还很多。

其实关于方式的选择，留着以后说吧。

