# docker 相关命令

### 查看镜像

```
docker images
```

![1](E:\my\blog\my-blog\docker\常用命令\1.png)

### 运行镜像（挂载目录）

```
docker run --name python -uroot --privileged=true -p 1111:2222 -v d\dev\docker\python:\opt\myscript -it python:v1 /bin/bash
```

--name: 容器名称

-v：本地磁盘目录 D:\dev\docker\python 映射到容器 \opt\myscript

python:v1 指的是 镜像名称:版本号

/bin/bash : 在容器内执行/bin/bash命令

--privileged=true : 无视权限

-p 1111:2222 : 1111 是服务器ip，2222是容器内ip

### 查看运行中的镜像

```
docker ps
```

![2](E:\my\blog\my-blog\docker\常用命令\2.png)

### 镜像环境修改后保存

```
docker commit -a "python@123.com" -m "install python" 300e315adb2f python:v1 
```

将上面的centos镜像安装python生成新的镜像

-a : 提交人

-m : 提交message

300e315adb2f : centos 的 container_id

python:v1 新的镜像名称和版本

## 进入正在运行的容器

```
docker exec -it containerId /bin/bash
```

## 删除镜像

```
docker rm containerId
```

## 启动/停止/重启镜像

```
docker start containerId
docker stop containerId 
docker restart containerId
```

