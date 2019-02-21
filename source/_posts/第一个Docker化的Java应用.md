---
title: 第一个Docker化的Java应用
date: 2017-4-12 01:00:00
tags: [docker]
categories:
- Bigdata
- docker
---

# Why Docker
## Docker简介
首先我们要知道LXC（Linux Container），即Linux容器工具。linux原生支持的容器，可以追溯到2009年，源于cgroup和namespaces在Linux内核方面的发展。它是一种轻量级的容器虚拟化技术，可以最大效率地隔离进程和资源。LXC可以把传统虚拟技术以及后来的Xen、KVM的VM进程像HOST进程一样运行管理, 所以创建和销毁都非常轻。

简单的说，Docker就是一个基于LXC技术的高级容器引擎，你可以在一个容器里开发完应用之后，装箱打包成一个镜像，然后轻松部署在不同的运行环境里。Docker解决了运行环境依赖问题解决了企业的痛点，提供了快速的持续集成，服务的弹性伸缩，简单的部署方式，不但解放了运维，还为企业节省了大量的机器资源。因此，在虚拟化、云计算、敏捷开发和高频度弹性伸缩需求的今天，Docker的流行也是自然的。

<!-- more -->

## Docker思想
Docker的LOGO由鲸鱼和集装箱组成，体现了其核心思想：
- 集装箱式的封装
- 准化的迁移、存储和API接口包装方式
- OS隔离性

Docker重点解决了操作系统、运行依赖、配置文件和代码版本的不一致问题。并且Docker可以在应用启动时设置其运行的最大CPU占用、内存、硬盘等资源，如果超过就杀掉它，保证了其他应用程序的资源不受影响。

# Docker核心技术
## 镜像
Docker镜像其实就是一系列的文件，包括程序文件、运行环境文件等。Docker将这些文件以联合文件格式（UnionFS）格式保存在本地，以实现文件的分层存放。

下面这张图片展示了Docker文件的存储格式：

{% asset_img docker-image.png docker-image %}


最底层是操作系统引导，Base是一个OS的系统层，往上则是应用运行所依赖的环境。除了最上方的容器层，其他文件层都是只读的。UnionFS将这些文件挂载到同一目录中。

## 容器
Docker容器的本质相当于一个进程。最上层的容器层是可读写的，当执行一个程序时，可能会有日志的输出，或者是保存文件。但是底下的镜像是只读的，因此如果我们需要修改镜像层中的文件（如配置），Docker会将该文件拷贝至最顶层的读写层，供程序做读写操作。

## 仓库
Docker仓库提供了镜像存储功能（类似Git），允许我们上传和拉取镜像到本地。

Docker官方提供的镜像仓库：https://hub.docker.com/
网易蜂巢镜像仓库：https://c.163.com/

docker也支持私有镜像仓库的搭建。

# Docker使用简介
## Docker安装
我们可以使用发行版自带的包管理软件来安装Docker：
```bash
$ sudo pacman -S docker
```
安装完成后，我们先启动Docker服务：
```bash
$ sudo systemctl start docker
```
然后查看本机Docker版本：
```bash
$ sudo docker version
# Client:
#  Version:      17.05.0-ce
#  API version:  1.29
#  Go version:   go1.8.1
#  Git commit:   89658bed64
#  Built:        Fri May  5 22:40:58 2017
#  OS/Arch:      linux/amd64
#
# Server:
#  Version:      17.05.0-ce
#  API version:  1.29 (minimum version 1.12)
#  Go version:   go1.8.1
#  Git commit:   89658bed64
#  Built:        Fri May  5 22:40:58 2017
#  OS/Arch:      linux/amd64
#  Experimental: false
```

## Docker命令
Docker常用命令如下：
```bash
# 拉取镜像，默认从hub.docker.com拉取TAG=latest
$ docker pull [OPTIONS] NAME[:TAG]
# 查看本机Docker镜像
$ docker images [OPTIONS] [REPOSITORY[:TAG]]
```
例如我们拉取`hello-world`镜像：
```bash
$ docker pull hello-world
```

镜像拉取完成后，我们可以使用`docker run`命令运行Docker镜像：
```bash
$ docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```
```bash
$ docker run hello-world
# Hello from Docker!
# This message shows that your installation appears to be working correctly.
#
# To generate this message, Docker took the following steps:
#  1. The Docker client contacted the Docker daemon.
#  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
#  3. The Docker daemon created a new container from that image which runs the
#     executable that produces the output you are currently reading.
#  4. The Docker daemon streamed that output to the Docker client, which sent it
#     to your terminal.
#
# To try something more ambitious, you can run an Ubuntu container with:
#  $ docker run -it ubuntu bash
#
# Share images, automate workflows, and more with a free Docker ID:
#  https://cloud.docker.com/
#
# For more examples and ideas, visit:
#  https://docs.docker.com/engine/userguide/
```

## Docker Nginx
下面来尝试在后台运行一个Nginx的Docker镜像：
```bash
$ docker pull hub.c.163.com/library/nginx
$ docker run -d hub.c.163.com/library/nginx
```
使用`docker ps`查看正在运行的Docker进程：
```bash
$ sudo docker ps
# CONTAINER ID        IMAGE                         COMMAND                  CREATED              STATUS              PORTS               NAMES
# f9ac8a5eead7        hub.c.163.com/library/nginx   "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              compassionate_curie
```
使用`docker exec`进入容器：
```bash
$ docker exec -it hub.c.163.com/library/nginx bash
```

## Docker网络
我们要想访问容器中的Nginx，需要用到Docker网络相关的命令。

由于Docker的隔离性，Docker进程与主机的网络也是依靠`network namespace`隔离的。Docker容器支持Bridge（虚拟独立的网卡，分配ip等）、Host和None三种模式。在Bridge模式下，我们需要配置端口映射：
```bash
# -p 将容器指定端口映射到指定的主机端口
$ docker run -d -p 8080:80 hub.c.163.com/library/nginx
$ netstat -na | grep 8080

# -P 将容器所有端口映射到随机的主机端口
$ docker run -d -P hub.c.163.com/library/nginx
# 查看随机端口分配
$ docker ps
# CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                   NAMES
# 39da72060ccf        hub.c.163.com/library/nginx   "nginx -g 'daemon ..."   8 seconds ago       Up 7 seconds        0.0.0.0:32768->80/tcp   amazing_liskov
$ netstat -na | grep 32768
```

# 制作镜像
现在我们基于tomcat镜像制作一个Jpress的Docker镜像，首先拉取tomcat镜像到本地：
```bash
$ docker pull hub.c.163.com/library/tomcat:latest
```
然后我们编写Dockerfile规则：
```bash
FROM hub.c.163.com/library/tomcat:latest

MAINTAINER SilverLining minmin3772@gmail.com

COPY jpress.war /usr/local/tomcat/webapps
```
使用`docker build <dockerfile_dir>`命令来构建Docker镜像：
```bash
$ docker build MyDocker -t jpress:latest
```

# 运行自定义的镜像
使用`docker run`来运行我们自制的镜像：
```bash
$ docker run -d -p 8888:8080 jpress
```
现在我们已经可以通过`http://localhost:8888/jpress`正常访问我们的应用了。

但是要想应用能够正常运行，我们还需要在Docker镜像内启动数据库服务。首先拉取MySQL Docker镜像：
```bash
$ docker pull hub.c.163.com/library/mysql:latest
```
MySQL Docker页面上有如下使用说明：
> Starting a MySQL instance:
> ```bash
> $ docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
> ```
> ... where some-mysql is the name you want to assign to your container, my-secret-pw is the password to be set for the MySQL root user and tag is the tag specifying the MySQL version you want. See the list above for relevant tags.
>
> Connect to MySQL from an application in another Docker container
> ```bash
> $ docker run --name some-app --link some-mysql:mysql -d application-that-uses-mysql
> ```
>
>Connect to MySQL from the MySQL command line client
> ```bash
> $ docker run -it --link some-mysql:mysql --rm mysql sh -c 'exec mysql > -h\"$MYSQL_PORT_3306_TCP_ADDR\" -P\"$MYSQL_PORT_3306_TCP_PORT\" -uroot > -p\"$MYSQL_ENV_MYSQL_ROOT_PASSWORD\"'
>```
> ... where some-mysql is the name of your original mysql container.
>```bash
> $ docker run -it --rm mysql mysql -hsome.mysql.host -usome-mysql-user -p
>```
我们启动MySQL的Docker镜像（映射到3308端口与本地MySQL区分）：
```bash
$ sudo docker run -d -p 3308:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=jpress hub.c.163.com/library/mysql:latest
```
最后只需要使用`ifconfig`命令查看本机ip地址并配置连接即可。注意这里不能直接使用localhost，因为对于Jpress来说localhost访问的是Docker网卡而不是本机网卡。
