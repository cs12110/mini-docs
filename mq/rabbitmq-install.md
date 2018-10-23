# CentOS 安装 RabbitMQ

## 安装 RabbitMq

### 1. 安装依赖

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

### 2. 安装 Rabbit

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

### 3. 启动问题

```sh
# rabbitmq 启动异常
[root@team-2 sbin]# ERROR: epmd error for host team-2: address (cannot connect to host/port)

# 设置: host里面的名称和上面红色标志一致
[root@team-2 sbin]# vim /etc/hosts
127.0.0.1   team-2
```

---

## RabbitMq 与 Java

### 1. 消息的持久化

生产者代码

```java
package com.app.durable;

import com.app.util.RabbitUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.MessageProperties;

/**
 *
 * 测试:
 *
 * 当rabbitmq服务器down的时候,没有持久化的数据也会被抹除
 *
 * 所以使用持久化,可以保证数据被消费,也不是被抹除
 *
 * <p>
 *
 * @author huanghuapeng 2017年7月6日
 * @see
 * @since 1.0
 */
public class MsgProducer {

      private static String durableQueueName = "durableQueue";

      private static String normalQueueName = "normalQueue";

      public static void main(String[] args) {
            sendNormal();
            sendDurableMsg();

      }

      /**
       * 持久化mq
       */
      public static void sendDurableMsg() {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();
                  Channel ch = conn.createChannel();

                  /*
                   * 第二个参数设置持久化,设置durable为true,同样在消费端queue的声明也必须为true
                   *
                   * text格式持久化?
                   */
                  ch.queueDeclare(durableQueueName, true, false, false, null);
                  ch.basicPublish("", durableQueueName, MessageProperties.PERSISTENT_TEXT_PLAIN,
                              "This is durable message".getBytes());

                  ch.close();
                  conn.close();

            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
      /**
       * 发送简单的mq
       */
      public static void sendNormal() {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();
                  Channel ch = conn.createChannel();

                  ch.queueDeclare(normalQueueName, false, false, false, null);
                  ch.basicPublish("", normalQueueName, null, "This is durable message".getBytes());

                  ch.close();
                  conn.close();

            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
}
```

消费者代码

```java
package com.app.durable;

import java.io.IOException;

import com.app.util.RabbitUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;
import com.rabbitmq.client.AMQP.BasicProperties;

public class MsgConsumer {
      private static String queueName = "durableQueue";

      public static void main(String[] args) {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();

                  final Channel ch = conn.createChannel();
                  ch.queueDeclare(queueName, true, false, false, null);

                  Consumer consumer = new DefaultConsumer(ch) {

                        @Override
                        public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                                    byte[] body) throws IOException {

                              String msg = new String(body, "UTF-8");

                              System.out.println("Rev: " + msg);
                              ch.basicAck(envelope.getDeliveryTag(), false);
                        }
                  };

                  ch.basicConsume(queueName, false, consumer);

            } catch (Exception e) {
                  e.printStackTrace();
            }

      }
}
```

## 参考文档

a. [CSDN 博客](http://blog.csdn.net/a15134566493/article/details/51393955)
