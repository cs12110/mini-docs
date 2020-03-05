# docker map

**该文档操作系统为:`centos7`,请知悉.**

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

# 安装完成之后可以看到版本号,同时docker的配置文件在/etc/docker/里面
root@team-2 ~]# docker --version
Docker version 19.03.6, build 369ce74a3c
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

**文件操作命令**

```sh
# 将主机/www/runoob目录拷贝到容器a27f9a637d62的/www目录下.
[root@team-2 soft]# docker cp /www/runoob a27f9a637d62:/www/

# 将容器a27f9a637d62的/www目录拷贝到主机的/tmp目录中.
[root@team-2 soft]# docker cp  a27f9a637d62:/www /tmp/
```

---

## 2. docker 使用

### 2.1 获取镜像

Q: 那么问题来了,该怎么找镜像?

A: 可以在这个网站找到镜像,然后根据名称下载. [hub link](https://hub.docker.com/)

```sh
[root@team-2 ~]# cd /etc/docker/

# 使用国内的镜像会比较快
# 如果daemon.json不存在,则新建并写入如下json内容
[root@team-2 docker]# cat daemon.json
{"registry-mirrors":["https://registry.docker-cn.com"]}

# 重启docker
[root@team-2 docker]# systemctl restart docker
```

下载新的镜像到本地的 docker

```sh
# 查找符合条件的镜像
[root@team-2 docker]# docker search centos

# 下载centos到本地
[root@team-2 docker]# docker pull centos:7
7: Pulling from library/centos
ab5ef0e58194: Downloading [=====>                                             ]  8.068MB/75.78MB

[root@team-2 docker]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              7                   5e35e350aded        3 months ago        203MB
hello-world         latest              fce289e99eb9        14 months ago       1.84kB
```

### 2.2 创建容器

首次创建容器并运行

```sh
# -i: 以交互模式运行容器,通常与 -t 同时使用
# -t: 为容器重新分配一个伪输入终端,通常与 -i 同时使用
# -p: 指定端口映射,格式为:`主机(宿主)端口:容器端口`
# -v: 绑定一个卷
# --name: 指定容器的名称
#
[root@team-2 ~]# docker run -it --name centos7-box centos:7

# 这个终端已经是容器里面
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

Q: 要是想安装 jdk 环境该怎么安装?

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

Q: <font color="red">可是为什么每次进入容器,都要执行`source /etc/profile`,jdk 环境才能生效?</font>

A: 解决方法是在容器里面的`/etc/bashrc`文件末尾处添加如下命令,以后每次登陆就不用刷新. :"}

小知识点:`/etc/bashrc:为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取.`

```sh
# set refresh /etc/profile
source /etc/profile
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

启动容器里面的 tomcat 之后,就可以通过浏览器访问了.

### 2.3 搬家

Q: 好的,我现在已经安装好整一个容器的环境,那我该怎么移植到一台服务器上面去?

A: 将容器导出来,然后搬到另一台服务器,导进去.

流程: `47.98.104.252导出容器` -> `scp到118.89.113.147` -> `118.89.113.147导入目标容器`.

#### 导出容器

```sh
[root@team-2 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
8753e1d3e736        8e0001d8989c        "/bin/bash"         7 hours ago         Up 7 hours          0.0.0.0:9500->8080/tcp   centos7-v2

# 根据容器id导出容器
[root@team-2 ~]# docker export 8753e1d3e736 > /opt/soft/pkgs/centos7-jdk.tar.gz

# 导出成功之后,可以在目标文件夹看到压缩包
[root@team-2 ~]# ls -alh /opt/soft/pkgs/ |grep centos
-rw-r--r--  1 root root 724M Mar  4 21:50 centos7-jdk.tar.gz

# scp到另一台服务器
[root@team-2 pkgs]# scp centos7-jdk.tar.gz root@118.89.113.147:/opt/soft
```

#### 导入容器

```sh
# 系统原有镜像
[root@team3 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE

# 导入目标镜像
[root@team3 soft]# docker import centos7-jdk.tar.gz centos7-jdk:centos7-jdk
sha256:dfd317699f590bf4e0c6ecf9db93e0aa930c6b47e0aef37013de1182de6c99a2

[root@team3 soft]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos7-jdk         centos7-jdk         dfd317699f59        31 seconds ago      749 MB

# 启动目标镜像容器
root@team3 soft]# docker run -it --name centos7-jdk -p 9500:8080 dfd317699f59 /bin/bash

# 进入容器并且启动tomcat
[root@9816335bf0b1 /]# source /etc/profile
[root@9816335bf0b1 /]# cd /opt/soft/tomcat8.5/
[root@9816335bf0b1 tomcat8.5]# bin/startup.sh
Using CATALINA_BASE:   /opt/soft/tomcat8.5
Using CATALINA_HOME:   /opt/soft/tomcat8.5
Using CATALINA_TMPDIR: /opt/soft/tomcat8.5/temp
Using JRE_HOME:        /opt/soft/jdk/jdk1.8/
Using CLASSPATH:       /opt/soft/tomcat8.5/bin/bootstrap.jar:/opt/soft/tomcat8.5/bin/tomcat-juli.jar
Tomcat started.

# 检查是否正常启动
[root@9816335bf0b1 tomcat8.5]# curl http://127.0.0.1:8080
[root@9816335bf0b1 tomcat8.5]# exit

# 因为不是守护进程的开启,所以又要重新进去容器开启tomcat
[root@team3 soft]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
9816335bf0b1        dfd317699f59        "/bin/bash"         6 minutes ago       Up 2 minutes        0.0.0.0:9500->8080/tcp   centos7-jdk

# 检查能否联通容器内部的tomcat
[root@team3 soft]# curl  http://127.0.0.1:9500
```

---

## 3. 参考资料

a. [菜鸟教程 docker](https://www.runoob.com/docker/centos-docker-install.html)

b. [菜鸟教程 dockerfile](https://www.runoob.com/docker/docker-dockerfile.html)
