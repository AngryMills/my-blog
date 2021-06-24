# nginx 配置 http/2(h2) 和 http 在同一端口的问题

大家好，我是烤鸭：

​	这个完全是个采坑记录了。



## 场景说明
由于这边有个需求想加个支持 grpc 方式转发的域名。

正常的二级域名都是映射到80端口，所以也没想太多，按照这个方式加上了。

```
# 已有的配置
server {
	listen 80;
	server_name yyy.xx.com;
	# access_log logs/access.log main;
	location / {
    	proxy_pass http://127.0.0.1:10086; # 此处配置http服务的ip和端口
    }
}

# grpc 代理配置
server {
	listen 80 http2; # grpc方式对外暴露端口
	server_name xxx.xx.com;
	# access_log logs/access.log main;
	location / {
		grpc_pass grpc://127.0.0.1:11800; # 此处配置grpc服务的ip和端口
	}
}
```

生效之后，nginx 其他的域名访问不了了，what ？？？



## 问题排查

加的这块有什么问题呢？
有人说是因为 http2 必须建立在https之上，只能挂在 443端口，果然改成 443端口之后，其他域名就可以访问了。

事实是这样的么？

我们看下nginx 官网对 http/2 升级的说明。
https://www.nginx.com/blog/nginx-1-9-5/

![1](.\1.png)

如果您的应用程序还没有用SSL/TLS加密，那么现在正是采取这一行动的好时机。加密应用程序可保护您免受间谍和中间人攻击。一些搜索引擎甚至奖励加密网站在搜索结果中排名的提高。以下配置块将所有普通HTTP请求重定向到站点的加密版本。

要启用HTTP/2支持，只需将http2参数添加到所有监听端口中。还包括ssl参数，这是必需的，因为浏览器不支持没有加密的HTTP/2。
两句话：开启 ssl/tsl，配置http2 ，这里虽然官方demo给的端口是 443，但是说明没有说必须配置在443。

http/2 和https 到底有什么区别？浏览器如果兼容https协议，访问 http/2 就没问题。这个只是针对 client端的，如果通过程序语言，使用 http/2 的grpc 方式，仅配置 http2 就行。

![1](.\3.png)

## 问题原因

那如果不是 http/2 必须配置在 443端口的原因，实际原因是什么。

几年前就有人给nginx提过这个问题。
https://trac.nginx.org/nginx/ticket/808 和  https://trac.nginx.org/nginx/ticket/816

**http/2 和 http 可以同时存在，只要不是同一个端口就行。**

配置 http/2 也可以两个方式，

``listen port_num http2 ssl `` （支持HTTP/1.1和h2，通过ALPN选择协议）

```listen port_num http2``` （仅支持h2）

由于我是服务端连接，所以选择了下边的方式。

看下源码更清晰。

https://github.com/nginx/nginx/blob/9e07862d6e9d37041a42bbfa34dd1f56ed547505/src/http/ngx_http_request.c#L324-L331

![1](.\4.png)



## 解决方案

仅配置 http/2 的时候选择一个和 http不同的端口。

配置 http2 ssl 的时候选择支持 https的端口。(一般就是 443)

