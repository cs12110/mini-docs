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

### 2.2

---

## 参考资料

a. [菜鸟教程 docker link](https://www.runoob.com/docker/centos-docker-install.html)

b. [个人之前笔记 link](https://app.yinxiang.com/shard/s51/nl/10635036/daa5e83e-d01f-4050-9e6d-5bc335f6d217?title=Docker%2BCentOS7)
