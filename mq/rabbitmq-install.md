# CentOS 安装 RabbitMQ

## 1. 安装依赖

安装编译软件

```sh
[root@team-2 ~]# yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel
```

安装`erlang`

```sh
# 解压
[root@team-2 ~]# tar -xvf otp_src_18.3.tar.gz cd otp_src_18.3

# 配置 '--prefix'指定的安装目录
[root@team-2 ~]# ./configure --prefix=/usr/local/erlang --without-javac

# 安装
[root@team-2 ~]# make && make install
```

配置`erlang`环境

```sh
[root@team-2 ~] vim /etc/profile
ERLANG_HOME=/usr/local/erlang
export PATH=$ERLANG_HOME/bin:$PATH
[root@team-2 ~] source /etc/profile
[root@alice erlang]# erl
Erlang/OTP 18 [erts-7.3] [source] [async-threads:10] [hipe] [kernel-poll:false]
Eshell V7.3  (abort with ^G)
1>
```

---

## 2. 安装 Rabbit

需要开启防火墙`15672`和`5672`端口.

```sh
[root@team-2 ~] wget
http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.10/rabbitmq-server-generic-unix-3.6.10.tar.xz

# 开启管理页面插件
[root@team-2 ~] cd rabbitmq-3.6.10/sbin/

# 启动rabbitmq服务
[root@team-2 ~] ./rabbitmq-server start &
[root@team-2 ~] ./rabbitmq-plugins enable rabbitmq_management
```

网页端第一次使用 guest 用户登录不成功,解决方法:

```sh
# 新增用户:root,登录密码:root
[root@alice team-2]# ./rabbitmqctl add_user root root
Creating user "root"

# 赋予用户角色,设置root用户为administrator
[root@team-2 sbin]# ./rabbitmqctl set_user_tags root administrator
Setting tags for user "root" to [administrator]

# 设置权限
[root@team-2 sbin]# ./rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
Setting permissions for user "root" in vhost "/"
```

关闭服务

```sh
[root@team-2 sbin]# rabbitmqctl stop

# 强制关闭
[root@team-2 sbin]# netstat -lnp|grep 5672
[root@team-2 sbin]# kill -9 thePidOfUsingPort5672
```

---

## 3. 启动问题

```sh
# rabbitmq 启动异常
[root@team-2 sbin]# ERROR: epmd error for host team-2: address (cannot connect to host/port)

# 设置: host里面的名称和上面红色标志一致
[root@team-2 sbin]# vim /etc/hosts
127.0.0.1   team-2
```

---

## 参考文档

a. [CSDN 博客](http://blog.csdn.net/a15134566493/article/details/51393955)
