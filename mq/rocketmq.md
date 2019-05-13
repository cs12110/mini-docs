# rocketmq

rocketmq 指南文档.仅用作入门,请知悉.

---

## 1. 安装 rocketmq

rocketmq 官方文档 [link](http://rocketmq.apache.org/docs/quick-start/)

### 1.1 依赖环境

| 依赖软件 | 版本 | 备注           |
| -------- | ---- | -------------- |
| jdk      | 1.8  | 需配置环境变量 |
| maven    | 3.6  | 需配置环境变量 |
| rocketmq | 4.4  | -              |

### 1.2 安装 rocketmq

```sh
[root@dev rocketmq-all-4.4.0]# mvn -Prelease-all -DskipTests clean install -U
# 在经历整一个世纪之后
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Apache RocketMQ 4.4.0 4.4.0:
[INFO]
[INFO] Apache RocketMQ 4.4.0 .............................. SUCCESS [03:55 min]
[INFO] rocketmq-logging 4.4.0 ............................. SUCCESS [ 22.315 s]
[INFO] rocketmq-remoting 4.4.0 ............................ SUCCESS [ 10.185 s]
[INFO] rocketmq-common 4.4.0 .............................. SUCCESS [  6.032 s]
[INFO] rocketmq-client 4.4.0 .............................. SUCCESS [ 10.901 s]
[INFO] rocketmq-store 4.4.0 ............................... SUCCESS [  6.126 s]
[INFO] rocketmq-srvutil 4.4.0 ............................. SUCCESS [  3.311 s]
[INFO] rocketmq-filter 4.4.0 .............................. SUCCESS [  2.393 s]
[INFO] rocketmq-acl 4.4.0 ................................. SUCCESS [  3.019 s]
[INFO] rocketmq-broker 4.4.0 .............................. SUCCESS [  6.887 s]
[INFO] rocketmq-tools 4.4.0 ............................... SUCCESS [  3.265 s]
[INFO] rocketmq-namesrv 4.4.0 ............................. SUCCESS [  1.572 s]
[INFO] rocketmq-logappender 4.4.0 ......................... SUCCESS [  2.571 s]
[INFO] rocketmq-openmessaging 4.4.0 ....................... SUCCESS [  2.747 s]
[INFO] rocketmq-example 4.4.0 ............................. SUCCESS [  1.834 s]
[INFO] rocketmq-test 4.4.0 ................................ SUCCESS [  6.574 s]
[INFO] rocketmq-distribution 4.4.0 ........................ SUCCESS [01:52 min]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  07:20 min
[INFO] Finished at: 2019-05-13T16:45:32+08:00
[INFO] ------------------------------------------------------------------------

[root@dev rocketmq-all-4.4.0]# cd distribution/target/apache-rocketmq/
[root@dev apache-rocketmq]# ls
benchmark  bin  conf  lib  LICENSE  NOTICE  README.md
```

### 1.3 启动 name server

```sh
[root@dev apache-rocketmq]# nohup sh bin/mqnamesrv &
[1] 16726
[root@dev apache-rocketmq]# tail -f ~/logs/rocketmqlogs/namesrv.log
2019-05-13 16:53:45 INFO main - tls.client.keyPath = null
2019-05-13 16:53:45 INFO main - tls.client.keyPassword = null
2019-05-13 16:53:45 INFO main - tls.client.certPath = null
2019-05-13 16:53:45 INFO main - tls.client.authServer = false
2019-05-13 16:53:45 INFO main - tls.client.trustCertPath = null
2019-05-13 16:53:46 INFO main - Using OpenSSL provider
2019-05-13 16:53:46 INFO main - SSLContext created for server
2019-05-13 16:53:46 INFO NettyEventExecutor - NettyEventExecutor service started
2019-05-13 16:53:46 INFO main - The Name Server boot success. serializeType=JSON
2019-05-13 16:53:46 INFO FileWatchService - FileWatchService service starte
```

### 1.4 启动 broker

```sh
[root@dev apache-rocketmq]# nohup sh bin/mqbroker -n localhost:9876 &
```

**由于虚拟机只有 2g 内存,但是默认使用内存为 8g,在不修改配置的前提下,出现如下异常**

```bash
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 8589934592 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /opt/soft/rocketmq/rocketmq-all-4.4.0/distribution/target/apache-rocketmq/hs_err_pid16823.log
Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x00000005c0000000, 8589934592, 0) failed; error='Cannot allocate memory' (errno=12)
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 8589934592 bytes for committing reserved memory.
# An error report file with more information is saved as:
# /opt/soft/rocketmq/rocketmq-all-4.4.0/distribution/target/apache-rocketmq/hs_err_pid16880.log
```

解决方案: 修改 bin 目录里面的 runbroker.sh 脚本配置参数如下,调小 broker 的运行所需内存.

```properties
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx512m -Xmn256m"
```

修改后重新启动

```sh
[root@dev apache-rocketmq]# nohup sh bin/mqbroker -n localhost:9876 &
[2] 17014
[root@dev apache-rocketmq]# jps -lm
17081 sun.tools.jps.Jps -lm
16732 org.apache.rocketmq.namesrv.NamesrvStartup
17021 org.apache.rocketmq.broker.BrokerStartup -n localhost:9876
```

### 1.5 测试使用

```sh
# 设置name sever的位置,这里使用export设置
[root@dev apache-rocketmq]# export NAMESRV_ADDR=localhost:9876

# 生产者生成消息,经过输出一段不明觉厉的东西之后,就停止了
[root@dev apache-rocketmq]# sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
...
SendResult [sendStatus=SEND_OK, msgId=C0A863CDAED07D4991AD417FF08303E7, offsetMsgId=C0A863CD00002A9F000000000002BDFE, messageQueue=MessageQueue [topic=TopicTest, brokerName=dev, queueId=2], queueOffset=249]
...

# 启动消费者
[root@dev apache-rocketmq]# sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
...
ConsumeMessageThread_20 Receive New Messages: [MessageExt [queueId=3, storeSize=180, queueOffset=106, sysFlag=0, bornTimestamp=1557738901467, bornHost=/192.168.99.205:54720, storeTimestamp=1557738901469, storeHost=/192.168.99.205:10911, msgId=C0A863CD00002A9F00000000000129B2, commitLogOffset=76210, bodyCRC=865372478, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=250, CONSUME_START_TIME=1557739065653, UNIQ_KEY=C0A863CDAED07D4991AD417FE7DB01A8, WAIT=true, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 52, 50, 52], transactionId='null'}]]
...
```

### 1.6 shutdown

```sh
[root@dev apache-rocketmq]# sh bin/mqshutdown broker

[root@dev apache-rocketmq]# sh bin/mqshutdown namesrv
```

---

## 2. Hello world

Q: 那么在 java 里面该怎么使用呢?

A: let do this!

### 2.1 maven 依赖

```xml
<dependency>
    <groupId>com.alibaba.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>3.2.6</version>
</dependency>
```

### 2.2 code

**公共配置**

```java
package com.pkg.common;

/**
 * rocketmq配置
 *
 * @author cs12110 create at 2019/5/13 17:35
 * @version 1.0.0
 */
public class MqSetting {

    /**
     * rocketmq name server host
     */
    public final static String NAME_SERVER_HOST = "192.168.99.205:9876";

    /**
     * message group
     */
    public final static String GROUP_NAME = "msg-group";

    /**
     * topic name
     */
    public final static String TOPIC_NAME = "4fun-topic";


    /**
     * tag name
     */
    public final static String TAG_NAME = "4fun";


    /**
     * keys
     */
    public final static String MESSAGE_KEYS = "cs12110";
}
```

**生产者**

```java
package com.pkg.producer;

import com.alibaba.rocketmq.client.producer.DefaultMQProducer;
import com.alibaba.rocketmq.common.message.Message;
import com.pkg.common.MqSetting;

import java.util.HashMap;
import java.util.Map;

/**
 * 消息提供者
 *
 * @author cs12110 create at 2019/5/13 17:35
 * @version 1.0.0
 */
public class MqProducer {

    public static void main(String[] args) {
        for (int index = 0; index < 10; index++) {
            Map<String, Object> map = new HashMap<>(2);
            map.put("index", index);
            map.put("value", index);

            sendMessage(map.toString());

            try {
                Thread.sleep(1000);
            } catch (Exception e) {
                //
            }
        }
    }

    private static void sendMessage(String message) {
        DefaultMQProducer producer = new DefaultMQProducer(MqSetting.GROUP_NAME);
        producer.setNamesrvAddr(MqSetting.NAME_SERVER_HOST);
        try {
            producer.start();

            Message msg = new Message();
            msg.setTopic(MqSetting.TOPIC_NAME);
            msg.setTags(MqSetting.TAG_NAME);
            msg.setKeys(MqSetting.MESSAGE_KEYS);
            msg.setBody(message.getBytes());

            producer.send(msg);

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.shutdown();
        }
    }
}
```

**消费者**

```java
package com.pkg.consumer;

import com.alibaba.rocketmq.client.consumer.DefaultMQPushConsumer;
import com.alibaba.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import com.alibaba.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import com.alibaba.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import com.alibaba.rocketmq.common.consumer.ConsumeFromWhere;
import com.alibaba.rocketmq.common.message.Message;
import com.alibaba.rocketmq.common.message.MessageExt;
import com.pkg.common.MqSetting;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;

/**
 * rocketmq消费者
 *
 * @author cs12110 create at 2019/5/13 17:35
 * @version 1.0.0
 */
public class MqConsumer {

    public static void main(String[] args) {
        consumer();
    }

    private static void consumer() {
        try {
            DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(MqSetting.GROUP_NAME);
            consumer.setNamesrvAddr(MqSetting.NAME_SERVER_HOST);

            // 订阅主题和标签
            consumer.subscribe(MqSetting.TOPIC_NAME, MqSetting.TAG_NAME);
            consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

            // 注册监听器
            consumer.registerMessageListener(new MsgListener());

            consumer.start();
            System.out.println("Consumer startup is success");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static class MsgListener implements MessageListenerConcurrently {

        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
            Message message = list.get(0);
            System.out.println(getTime() + " - " + new String(message.getBody()));
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }

        private static String getTime() {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            return sdf.format(new Date());
        }
    }
}
```

测试结果

```java
Consumer startup is success
2019-05-13 17:59:54 - {index=0, value=0}
2019-05-13 17:59:56 - {index=1, value=1}
2019-05-13 17:59:59 - {index=2, value=2}
2019-05-13 18:00:02 - {index=3, value=3}
2019-05-13 18:00:04 - {index=4, value=4}
2019-05-13 18:00:07 - {index=5, value=5}
2019-05-13 18:00:10 - {index=6, value=6}
2019-05-13 18:00:12 - {index=7, value=7}
2019-05-13 18:00:15 - {index=8, value=8}
2019-05-13 18:00:18 - {index=9, value=9}
```

---

## 3. 参考资料

a. [rocketmq 官方文档](http://rocketmq.apache.org/docs/quick-start/)

b. [CSDN 博客:java 使用 rocketMq](https://blog.csdn.net/zhangcongyi420/article/details/82593982)

c. [轻松搞定 RocketMQ 入门](https://segmentfault.com/a/1190000015951993)

d. [搭建 RocketMQ 踩的坑](https://blog.csdn.net/c_yang13/article/details/76836753)
