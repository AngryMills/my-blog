# springcloud gateway 使用nacos 作为配置中心

大家好，我是烤鸭：

​     今天分享下 springcloud gateway 使用nacos作为配置中心和注册中心，主要是还是配置中心。

​     源码下载：

 https://gitee.com/fireduck_admin/springcloud-gateway-nacos-demo

1.  本地部署nacos

下载 https://github.com/alibaba/nacos/releases/tag/1.3.2

本地新建nacos数据库，执行 conf/nacos-mysql.sql

修改 conf/application.properties 关于数据库的配置

```
spring.datasource.platform=mysql

### Count of DB:
db.num=1

### Connect URL of DB:
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=root

```

启动 startup.cmd，访问 localhost:8848 如图

![2](E:\my\blog\springcloud\cloud_gateway\2.png)

2. 网关项目搭建

   这里需要注意的是普通项目和 gateway项目 有些不一样

   比如普通项目的 bootstrap.yml，这里不需要写nacos 地址，指定加载的配置文件 -Dspring.profiles.active=dev，

   ```
   spring:
     cloud:
       nacos:
         config:
           file-extension: yml
           group: demo-dick
           prefix: application
     profiles:
       active:
         - @profiles.active@
   ```

   在 bootstrap-dev.yml 里再写地址

   ```
   spring:
     cloud:
       nacos:
         config:
           server-addr: localhost:8848
   ```

   但是 gatewway 项目不行，加载顺序的问题，必须在 bootstrap.yml 指定地址。

   ${} 可以读取启动参数，需要在启动时加上 -Dnacos-server-addr=123.1.1.2:8848，不写的话就默认localhost:8848

   ```
   spring:
     cloud:
       nacos:
         config:
           file-extension: yml
           group: gateway
           prefix: application
           server-addr: ${nacos-server-addr:localhost:8848}
   ```

   

3. nacos集成

   gateway 项目nacos 配置，lb://后面的是其他服务注册在nacos上的名称，也就是spring.applicaiton.name

   ```
   management:
     endpoints:
       web:
         exposure:
           include: '*'
   server:
     port: 8081
     servlet:
       context-path: /
   spring:
     application:
       name: gateway
     cloud:
       gateway:
         routes:
           - id: tick-route
             filters:
               - StripPrefix=1
             predicates:
               - name: Path
                 args[pattern]: /tick/**
             uri: lb://demo-tick1
           - id: tick-route
             filters:
               - StripPrefix=1
             predicates:
               - name: Path
                 args[pattern]: /dick/**
             uri: lb://demo-dick
       nacos:
         discovery:
           server-addr: localhost:8848
         password: nacos
         username: nacos
   ```

   启动成功拉取nacos配置(端口 8081 生效)

   ![1](E:\my\blog\springcloud\cloud_gateway\1.png)

   另外两个项目就不贴了，源码地址在文章开始。

4. 注册中心

   可以看到3个服务都注册成功了。

   ![7](E:\my\blog\springcloud\cloud_gateway\7.png)

   正常情况下访问 http://localhost:8081/dick/dick/abc 和 http://localhost:8081/tick/tick/abc 都可以返回。

   ![8](E:\my\blog\springcloud\cloud_gateway\8.png)

   动态修改网关路由：

   更新gateway nacos 配置后，lb://demo-tick 改为 demo-tick1 立即生效，无需重启。由于找不到 demo-tick1 所以报错。

   ![6](E:\my\blog\springcloud\cloud_gateway\6.jpg)



5.  最后说一下

   关于上面地址/dick/dick 第一个是网关转发的路由，第二个是服务本身的 context-path。

   而如果网关项目用的是域名/gateway 转发的话，需要为网关项目加 context-path，具体可以参考 

   https://blog.csdn.net/Angry_Mills/article/details/108132203。