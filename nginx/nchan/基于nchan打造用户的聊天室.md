# 基于nchan打造百万用户的聊天室

大家好，我是烤鸭：

&nbsp;&nbsp;&nbsp;这次介绍下nchan，nginx的一个module。

## nchan 

源码: https://github.com/slact/nchan
官网: https://nchan.io/
nginx 配置说明文档: https://nchan.io/documents/nginxconf2016-slides.pdf



## 测试环境搭建

4 台linux centos 7，都安装了nginx和nchan。

安装可以参考下这篇文章。 

https://www.cnblogs.com/rongfengliang/p/7866122.html

目前使用的是4台nginx做测试,1台模拟上游的转发服务器(类似 keepalived),后3台是安装了nchan的nginx,用来 pub/sub
看下配置。

nginx_master.conf

```
#master
upstream ws {
   server 192.168.1.1:8080 weight=1 max_fails=2;
   server 192.168.1.2:8080 weight=1 max_fails=2;
   server 192.168.1.3:8080 weight=1 max_fails=2;
}

server {
  listen       80;
  server_name test.xxx.xxx.com;
  root  /usr/local/nginx/chat;

  #error_page 404 /404.html;
  error_page 500 502 503 504 /50x.html;
  location = 50x.html {
    root /usr/local/nginx/html;
  }

  #master
  location / {
    proxy_pass http://ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  access_log  logs/barrage.access.log;
  error_log  logs/nchan_error.log;
}
```

nginx_salve.conf

```
nchan_shared_memory_size 256M;

upstream redis_cluster {
    nchan_redis_server redis://:password@10.168.1.2:3001;
    nchan_redis_server redis://:password@10.168.1.2:3002;
    nchan_redis_server redis://:password@10.168.1.3:3001;
    nchan_redis_server redis://:password@10.168.1.3:3002;
    nchan_redis_server redis://:password@10.168.1.1:3001;
    nchan_redis_server redis://:password@10.168.1.1:3002;
    # you don't need to specify all the nodes, they will be autodiscovered
    # however, it's recommended that you do specify at least a few master nodes.
  }

server {
  listen       8080;
  server_name localhost;
  root  /usr/local/nginx/chat;

  location = /sub {
    add_header x-hit-from 127 always;
    nchan_subscriber;
    nchan_channel_id $arg_vid;
    nchan_use_redis on;
    nchan_redis_pass redis_cluster;
  }

  location = /pub{
     nchan_publisher;
     nchan_channel_id $arg_id;
     nchan_redis_pass redis_cluster;
     nchan_message_timeout 5m;
  }

  #..
  location = /private/status {
    nchan_stub_status;
  }
}
```

websocket测试网站：
http://www.websocket-test.com/

ws 建联 sub接口用来接收消息，vid是nginx配置里的channel_id

![1](.\2.png)

postman模拟消息发送到 nchan, /pub 接口

![1](.\1.png)

redis 存消息数据

![1](.\3.png)

如果自己写前端页面的话：可以是创建原生的websocket，也可以使用官方提供的NchanSubscriber.js。

正常 websocket:

```
var ws =new WebScoket("ws://xxx.yyy.com/sub?id=demo")
ws.onMessage=funciton(data){
   console.log(data)

}
```

引入 NchanSubscriber.js:

```
var sub = new NchanSubscriber('http://xxx.yyy.com/sub?id=demo', 'websocket');
sub.on("message",
function(message, message_metadata) {
    alert(message);
});
```

## 架构

```
# 共享内存,单机的时候取决于单机的内存,redis集群下取决于 单个节点的内存
nchan_shared_memory_size 32000M;
```

单机nginx的worker通过channel的hash把数据同步到当前worker的内存，再同步到共享内存，同时刷到第二个worker。（关于内存操作其实调用的是nginx的api）

右边的redis模拟的场景是不同的nchan节点，共享层用redis来实现。

![1](.\7.png)



如果量小的话，简单的做个im或者聊天室，单机就足够了，使用的是nginx的机器内存。(和 springboot 直接加个 @WebSocket 注解差不多)

![1](.\11.png)

支持水平扩展的集群架构：(理论上支持不限数量的横向扩展，瓶颈在redis集群，得做好容灾方案。)

一般单台机器的连接数在 65535(可以改大)，所以即便存储使用了redis，单机还是有瓶颈的，当然一般看消息体的大小，可能redis会先崩。所以需要多台nginx机器和一个超大的redis集群，来扛得住百万用户。nginx集群至少20台才能维持这么多人同时在线，如果要考虑消息的话，得看消息内容，预估redis集群大小。

![1](.\4.png)



## nchan 和 netty

今天有同事问我，关于分发消息的。nchan和netty有什么区别。

简单说下netty的实现。

用 ConcurrentMap 维护 key(聊天室id)和channelGroup(每一个用户连接成功，就会增加一个channel)。

当有消息需要通知的时候需要调用方法即可。

```
channelGroup.writeAndFlush(new TextWebSocketFrame(message));
```

再看下 nchan的源码：

slact/nchan/blob/master/src/subscribers/common.c

```
ngx_int_t nchan_subscriber_receive_notice(subscriber_t *self, ngx_int_t code, void *data) {
  if(code == NCHAN_NOTICE_SUBSCRIBER_INFO_REQUEST) {
  	// 构建需要通知的内容...
    // 获取需要通知的 channel_id,从这个函数 nchan_get_subscriber_info_response_channel_id
    ngx_str_t *response_channel_id = nchan_get_subscriber_info_response_channel_id(self->request, response_id);
    // 构建msg对象
    cf->storage_engine->publish(response_channel_id, &msg, cf, NULL, NULL);
    
    if(result_allocd) {
      ngx_http_complex_value_free(&result);
    }
  }
  return NGX_OK;
}
```

/src/util/nchan_channel_id.c

```
ngx_str_t *nchan_get_subscriber_info_response_channel_id(ngx_http_request_t *r, uintptr_t request_id) {
  // 全局的 request_ctx 对象,调用nginx api获得,https://www.nginx.com/resources/wiki/extending/api/http/#ngx-http-get-module-ctx
  nchan_request_ctx_t    *ctx = ngx_http_get_module_ctx(r, ngx_nchan_module);
  
  ngx_str_t *chid = ctx->subscriber_info_response_channel_id;
  // child 是空的,就新分配内存,并给ctx的subscriber_info_response_channel_id赋值
  if(!chid) {
  	// nginx 分配内存的函数,https://www.nginx.com/resources/wiki/extending/api/alloc/
    chid = ngx_palloc(r->pool, sizeof(ngx_str_t));
    if(chid == NULL) {
      return NULL;
    }
    ctx->subscriber_info_response_channel_id = chid;
    
    chid->data = ngx_palloc(r->pool, NCHAN_SUBSCRIBER_INFO_CHANNEL_ID_BUFFER_SIZE);
    if(chid->data == NULL) {
      ctx->subscriber_info_response_channel_id = NULL;
      return NULL;
    }
  }
    
  u_char *end = ngx_snprintf(chid->data, NCHAN_SUBSCRIBER_INFO_CHANNEL_ID_BUFFER_SIZE, "meta/sr%d", (ngx_int_t )request_id);
  chid->len = end - chid->data;
  
  return chid;
  
}
```

src/nchan_types.h

看一下 nchan_request_ctx_t 的结构,订阅者的id和channel_id 都存了。

```
#define NCHAN_MULTITAG_REQUEST_CTX_MAX 4
typedef struct {
  subscriber_t                  *sub;
  nchan_reuse_queue_t           *output_str_queue;
  nchan_reuse_queue_t           *reserved_msg_queue;
  nchan_bufchain_pool_t         *bcp; //bufchainpool maybe?
  
  ngx_str_t                     *subscriber_type;
  nchan_msg_id_t                 msg_id;
  nchan_msg_id_t                 prev_msg_id;
  ngx_str_t                     *publisher_type;
  ngx_str_t                     *multipart_boundary;
  ngx_str_t                     *channel_event_name;
  ngx_str_t                      channel_id[NCHAN_MULTITAG_REQUEST_CTX_MAX];
  int                            channel_id_count;
  time_t                         channel_subscriber_last_seen;
  int                            channel_subscriber_count;
  int                            channel_message_count;
  ngx_str_t                     *channel_group_name;
  
  ngx_str_t                     *request_origin_header;
  ngx_str_t                     *allow_origin;
  
  ngx_int_t                      subscriber_info_response_id;
  ngx_str_t                     *subscriber_info_response_channel_id;
  
  unsigned                       sent_unsubscribe_request:1;
  unsigned                       request_ran_content_handler:1;
  
} nchan_request_ctx_t;
```

其实是从nginx的全局对象获取channel_id的。

## 总结

### 优点:

- 服务不需要考虑和维护链接等,只需要专注处理业务相关逻辑。

- nchan由于直接在nginx层,性能更好(比起自己搭建pub/sub服务器)。

### 缺点: 

- 非应用层的，出现问题不好排查。
- 默认的消息存储时间过长(nchan_message_timeout:1h),消息大量情况下容易拖垮集群。
- 以channelid为key的话,可能导致redis分配不均匀,单个节点压力过大,影响不止当前的channelid。
- 集群模式依赖redis，需要考虑容灾方案。
- 扩展性差，由于不支持多个redis集群，没法根据特定条件分片。(比如某个聊天室人特别多，单独一套redis集群，用netty的话可能比较好实现)（特意给作者留言确认了一下，https://github.com/slact/nchan/issues/619）