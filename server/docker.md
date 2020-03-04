# docker

我: Hi,docker,can I call you dog???

Docker: You can call me anything you want. Also fuck ya!!!.

---

## 1. 安装 docker

docker 基础知识,安装和常用命令.

### 1.1 基础知识

名词解释

| 名词         | 备注                         |
| ------------ | ---------------------------- |
| image        | 镜像,相当于 iso 文件         |
| container    | 容器, 相当于安装好的操作系统 |
| docker run   | 创建容器,每次都重新创建      |
| docker start | 开启已经存在的容器           |

### 1.2 big bang

先删除封建残余

```sh
[root@team-2 ~]# cat remove-docker.sh
#!/bin/bash

# 删除旧版本的docker
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

安装 docker,首先要设置仓库

```sh
# 安装必要的插件
[root@team-2 ~]# yum install -y yum-utils \
>   device-mapper-persistent-data \
>   lvm2

# 设置稳定仓库
[root@team-2 ~]# yum-config-manager \
>     --add-repo \
>     https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker
[root@team-2 ~]# yum -y install docker-ce docker-ce-cli containerd.io
```

如果上面都完成了,那么我们该 `Hello world`.

```sh
# 开启docker服务
[root@team-2 ~]# sytemctl start docker

# 获取docker里面的镜像
[root@team-2 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              fce289e99eb9        14 months ago       1.84kB

# 测试hello-world
[root@team-2 ~]# docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### 1.3 常用命令

**docker 的启动与关闭**

```sh
# 开启docker
[root@team-2 ~]# systemctl start docker

# 关闭docker
[root@team-2 ~]# systemctl stop docker

# 重启docker
[root@team-2 ~]# systemctl restart docker
```

**镜像和容器管理**

```sh
#删除镜像,首先要删除该镜像关联的容器
[root@team-2 ~]# docker rm containerId

#删除镜像
[root@team-2 ~]# docker rmi -f imageId
```

**容器重命名**

```sh
[root@team-2 soft]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a27f9a637d62        5e35e350aded        "bin/bash"          28 minutes ago      Up 27 minutes                           serene_mcnulty
[root@team-2 soft]# docker rename serene_mcnulty centos7-v1
[root@team-2 soft]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a27f9a637d62        5e35e350aded        "bin/bash"          29 minutes ago      Up 28 minutes                           centos7-v1
```

---

## 2. docker 使用

docker 要使用才叫 docker 呀.

### 2.1 获取镜像

使用国内的镜像会比较快

Q: 那么问题来了,该怎么找镜像?

A: 可以在这个网站找到镜像,然后根据名称下载. [hub link](https://hub.docker.com/)

```sh
[root@team-2 ~]# cd /etc/docker/

# 如果daemon.json不存在,则新建并写入如下json内容
[root@team-2 docker]# cat daemon.json
{"registry-mirrors":["https://registry.docker-cn.com"]}

# 重启docker
[root@team-2 docker]# systemctl restart docker
```

下载新的镜像到本地的 docker

```sh
# 查找符合条件的镜像
[root@team-2 docker]# systemctl restart docker

# 下载centos到本地
[root@team-2 docker]# docker pull centos:7
7: Pulling from library/centos
ab5ef0e58194: Downloading [=====>                                             ]  8.068MB/75.78MB

[root@team-2 docker]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              7                   5e35e350aded        3 months ago        203MB
hello-world         latest              fce289e99eb9        14 months ago       1.84kB
```

首次创建容器并运行

```sh
#
# -i: 以交互模式运行容器,通常与 -t 同时使用
# -t: 为容器重新分配一个伪输入终端,通常与 -i 同时使用
# -p: 指定端口映射,格式为:`主机(宿主)端口:容器端口`
# -v: 绑定一个卷
# --name: 指定容器的名称
#
[root@team-2 ~]# docker run -it --name centos7-box centos:7

# 这个终端已经是容器里面的了
[root@0695454bfe8e /]# exit

# 查看所有的容器
# docker ps: 查看正在运行的容器
[root@team-2 ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
0695454bfe8e        centos:7            "/bin/bash"         5 minutes ago       Exited (0) 4 minutes ago                       centos7-box
0aee31e4df9d        hello-world         "/hello"            10 hours ago        Exited (0) 10 hours ago                        hungry_einstein

# 关闭容器
[root@team-2 ~]# docker stop centos7-box
[root@team-2 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

# 重新启动该容器
[root@team-2 ~]# docker start 0695454bfe8e
0695454bfe8e
[root@team-2 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0695454bfe8e        centos:7            "/bin/bash"         10 minutes ago      Up 7 seconds                            centos7-box
# 进入容器并进行相关操作
[root@team-2 ~]# docker exec -it 0695454bfe8e /bin/bash
[root@0695454bfe8e /]#
```

Q: 要是我想安装 jdk 环境该怎么安装?

A: 把本地的 jdk 安装包传到容器里面,该怎么配置就怎么配置.

```sh
[root@team-2 soft]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
a27f9a637d62        5e35e350aded        "bin/bash"          9 minutes ago       Up 9 minutes                            serene_mcnulty

# a27f9a637d62: 容器id
# /opt/soft/jdk: 保存容器的文件夹首先存在
[root@team-2 soft]# docker cp /opt/soft/docker/jdk1.8.tar.gz a27f9a637d62:/opt/soft/jdk

# 进入容器
[root@team-2 soft]# docker exec -it a27f9a637d62 /bin/bash
[root@a27f9a637d62 /]# cd /opt/soft/jdk/
[root@a27f9a637d62 jdk]# ls
jdk1.8.tar.gz
```

将当前容器提交成镜像

```sh
# commit 容器id 
[root@team-2 soft]# docker commit a27f9a637d62 centos7-with-jdk-tomcat
[root@team-2 soft]# docker  images
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
centos7-with-jdk-tomcat   latest              8e0001d8989c        6 minutes ago       749MB
centos                    7                   5e35e350aded        3 months ago        203MB
hello-world               latest              fce289e99eb9        14 months ago       1.84kB

# 使用新的镜像运行的容器,并映射端口
[root@team-2 soft]# docker run -it -p 9500:8080 8e0001d8989c  /bin/bash
```

### 2.2 搬家

```sh

```

---

## 参考资料

a. [菜鸟教程 docker link](https://www.runoob.com/docker/centos-docker-install.html)

b. [个人之前笔记 link](https://app.yinxiang.com/shard/s51/nl/10635036/daa5e83e-d01f-4050-9e6d-5bc335f6d217?title=Docker%2BCentOS7)
