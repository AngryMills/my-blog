# jenkins发布docker项目

大家好，我是烤鸭：

​     jenkins 部署k8s 项目还是比较流畅的，本身建立多流水线项目，在项目中添加jenkinsfile就好了，镜像需要额外的参数，还可以添加dokcerfile文件。由于我现在的问题是不能够修改原有的项目，还想利用项目中的Dockerfile打包后发布到私服仓库(Harbor)中。



## 构建普通maven项目

这种构建有个劣势就是只能单分支的。

![1](.\1.png)

## docker 安装

```
yum install docker
docker -v 
[]: Docker version 19.03.4, build 9013bf583a
```

这里有个小坑就是docker 默认使用https链接，而局域网内ip都是http的。

```
vi /etc/docker/daemon.json
```

registry-mirrors 是下载镜像的备用镜像地址、insecure-registries 是可以使用http链接的地址。

```
{
  "registry-mirrors": ["https://dhq9bx4f.mirror.aliyuncs.com",
                      "https://registry.docker-cn.com",
                      "http://hub-mirror.c.163.com",
                      "https://tnxkcso1.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.1.1：80"],
   "bip": "192.168.2.1/24"
}

```

registry-mirrors 也没啥用，后来构建的时候死活拉不下来包(无论怎么改都会从 docker.io 拉包)。

![image-20210308135453500](.\3.png)

只能手动拉下来再重命名了。

之前有个包拉不下来，frolvlad/alpine-oraclejdk8。只能先从别的镜像地址拉。

```
docker pull docker.mirrors.ustc.edu.cn/frolvlad/alpine-oraclejdk8
```

拉完了再重命名，要不每次还会从 docker.io 拉取

```
docker tag tnxkcso1.mirror.aliyuncs.com/frolvlad/alpine-oraclejdk docker.io/frolvlad/alpine-oraclejdk
```

虽然image id 一样，但是包是ok的

![image-20210308135453500](.\4.png)

## 利用脚本发布

Post Steps

Execute Shell

```
#项目所在jenkins目录
cd /var/lib/jenkins/workspace/xxx/
#复制到指定目录
rm -rf /data/apps/xxx/*
cp ./target/*.jar /data/apps/xxx
cp ./Dockerfile /data/apps/xxx
#进入目录执行docker命令
cd /data/apps/xxx
#docker生成镜像并推送到仓库,build-arg非必填,需要看dockerfile是否有环境变量引用
docker build -t 192.168.1.1:80/xxx/xxx:v1 --build-arg "JAR_NAME=./xxx-1.0-SNAPSHOT.jar" -f ./Dockerfile .
docker login -u=admin -p=admin 192.168.1.1:80
docker push 192.168.1.1:80/xxx/xxx:v1

```

push到harbor这块还有个小坑，需要先在 harbor 建立项目。

比如项目名称是 AAA，你的镜像+tag 是 xxx:v1。

那么push的时候要写上全路径，不建项目是不行的！

```
docker push 192.168.1.1:80/xxx/xxx:v1
```



## 发布到harbor

![1](.\2.png)

