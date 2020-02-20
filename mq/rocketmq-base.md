# Hello Rocketmq

多年以后,站在敬老院的门前,我将会回想起瑟瑟发抖的使用 rocketmq 那个遥远的下午.

---

## 1. 基础知识

[RocketMq 基础知识官方文档 link](http://rocketmq.apache.org/docs/core-concept/)

### 1.1 Why RocketMq?

Q: 为什么使用 rocketmq,而不使用其他的 mq 呢?

A: 官方给出的回答.

```bash
# Based on our research, with increased queues and virtual topics in use, ActiveMQ IO module reaches a bottleneck. We tried our best to solve this problem through throttling, circuit breaker or degradation, but it did not work well. So we begin to focus on the popular messaging solution Kafka at that time. Unfortunately, Kafka can not meet our requirements especially in terms of low latency and high reliability, see here for details.

大意: 随着队列和topic的增长,ActiveMq遇到了瓶颈.kafka的低延迟和高可靠不达标.

# In this context, we decided to invent a new messaging engine to handle a broader set of use cases, ranging from traditional pub/sub scenarios to high volume real-time zero-loss tolerance transaction system.

大意: 在这样子的前提下,决定去弄一个新的消息引擎来处理消息,从传统的发布/订阅场景到`高容量`,`实时`,`零丢失`事务系统.
```

下面是 RocketMq 和 Kafka 以及 ActiveMq 的对比(注意: 站在偏向 RocketMq 的角度的对比)

![compare](imgs/rocketmq-compare.png)

- ordered message: FIFO(First in first out).
- scheduled message: 定时发送消息.
- batched message: 批量消息(相当 jdbc 的批处理).
- broadcast message: 广播消息.

### 1.2 rocketmq 的结构

Q: 那么一般高可用的 rocketmq 的架构是怎样子呢?

A: 请看下面高可用的 rocketmq 架构图.

![](imgs/rocketmq.png)

名称解释

| 名称                | 说明                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Producer`          | 消息生产者,位于用户的进程内,Producer 通过 NameServer 获取所有 Broker 的路由信息,根据负载均衡策略选择将消息发到哪个 Broker,然后调用 Broker 接口提交消息.                                                                                                                                                                                      |
| `Producer Group`    | 生产者组,简单来说就是多个发送同一类消息的生产者称之为一个生产者组.                                                                                                                                                                                                                                                                           |
| `Consumer`          | 消息消费者,位于用户进程内.Consumer 通过 NameServer 获取所有 broker 的路由信息后,向 Broker 发送 Pull 请求来获取消息数据.Consumer 可以以两种模式启动,广播（Broadcast）和集群（Cluster）,广播模式下,一条消息会发送给所有 Consumer,集群模式下消息只会发送给一个 Consumer.                                                                        |
| `Consumer Group`    | 消费者组,和生产者类似,消费同一类消息的多个 Consumer 实例组成一个消费者组.<span style="color:red">(记住 ta,这个超重要的!!!)<span>                                                                                                                                                                                                             |
| `Topic`             | Topic 用于将消息按主题做划分,Producer 将消息发往指定的 Topic,Consumer 订阅该 Topic 就可以收到这条消息.Topic 跟发送方和消费方都没有强关联关系,发送方可以同时往多个 Topic 投放消息,消费方也可以订阅多个 Topic 的消息.在 RocketMQ 中,Topic 是一个上逻辑概念.消息存储不会按 Topic 分开.                                                          |
| `Message`           | 代表一条消息,使用 MessageId 唯一识别,用户在发送时可以设置 messageKey,便于之后查询和跟踪.一个 Message 必须指定 Topic,相当于寄信的地址.Message 还有一个可选的 Tag 设置,以便消费端可以基于 Tag 进行过滤消息.也可以添加额外的键值对,例如你需要一个业务 key 来查找 Broker 上的消息,方便在开发过程中诊断问题.                                      |
| `Tag`               | 标签可以被认为是对 Topic 进一步细化.一般在相同业务模块中通过引入标签来标记不同用途的消息.                                                                                                                                                                                                                                                    |
| `Broker`            | Broker 是 RocketMQ 的核心模块,负责接收并存储消息,同时提供 Push/Pull 接口来将消息发送给 Consumer.Consumer 可选择从 Master 或者 Slave 读取数据.多个主/从组成 Broker 集群,集群内的 Master 节点之间不做数据交互.Broker 同时提供消息查询的功能,可以通过 MessageID 和 MessageKey 来查询消息.Borker 会将自己的 Topic 配置信息实时同步到 NameServer. |
| `Queue`             | Topic 和 Queue 是 1 对多的关系,一个 Topic 下可以包含多个 Queue,主要用于负载均衡.发送消息时,用户只指定 Topic,Producer 会根据 Topic 的路由信息选择具体发到哪个 Queue 上.Consumer 订阅消息时,会根据负载均衡策略决定订阅哪些 Queue 的消息.                                                                                                       |
| `Offset`            | RocketMQ 在存储消息时会为每个 Topic 下的每个 Queue 生成一个消息的索引文件,每个 Queue 都对应一个 Offset 记录当前 Queue 中消息条数.                                                                                                                                                                                                            |
| `NameServer`        | NameServer 可以看作是 RocketMQ 的注册中心,它管理两部分数据:集群的 Topic-Queue 的路由配置;Broker 的实时配置信息.其它模块通过 Nameserv 提供的接口获取最新的 Topic 配置和路由信息.                                                                                                                                                              |
| `Producer/Consumer` | 通过查询接口获取 Topic 对应的 Broker 的地址信息                                                                                                                                                                                                                                                                                              |
| `Broker`            | 注册配置信息到 NameServer, 实时更新 Topic 信息到 NameServer                                                                                                                                                                                                                                                                                  |

### 1.3 pull 与 push 模式

消息队列消息有两种方式

| 模式   | 说明                                                                                                                                    |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `Push` | 由 MQ 收到消息后主动调用消费者的新消息通知接口,需要消耗服务器 MQ 宝贵的线程资源,同时消费者只能被动等待消息通知(适合实时性要求高的场景). |
| `Pull` | 由消费者轮询调用 API 去获取消息,不消耗服务器 MQ 线程,消费者更加主动,虽然消费者的处理逻辑变得稍稍复杂.                                   |

两种方式的根本区别在于线程消耗问题,由于 MQ 服务器的线程资源相对客户端更加宝贵,Push 方式会占用服务器过多的线程从而难以适应高并发的消息场景.同时当某一消费者离线一段时间再次上线后,大量积压消息处理会消耗大量 MQ 线程从而拖累其它消费者的消息处理,所以 Pull 方式相对来说更好.

---

## 2. 安装 rocketmq

rocketmq 官方文档 [link](http://rocketmq.apache.org/docs/quick-start/)

### 2.1 依赖环境

| 依赖软件 | 版本 | 备注           |
| -------- | ---- | -------------- |
| jdk      | 1.8  | 需配置环境变量 |
| maven    | 3.6  | 需配置环境变量 |
| rocketmq | 4.4  | -              |

### 2.2 安装 rocketmq

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

### 2.3 启动 name server

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

### 2.4 启动 broker

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

### 2.5 测试使用

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

### 2.6 shutdown

```sh
[root@dev apache-rocketmq]# sh bin/mqshutdown broker

[root@dev apache-rocketmq]# sh bin/mqshutdown namesrv
```

---

## 3. 使用设计

### 3.1 topic-queue 设计

Q: 在 rocketmq 里面,一个 topic 可以对应多个 queue,那么我们该怎么有效的设置 topic 与 queue 的数量呢?(queue 与 tag 的关系是什么???)

![](imgs/topic-queue.jpg)

A: TOPIC_A 在一个 Broker 上的 Topic 分片有 4 个 Queue,一个 Consumer Group 内有 2 个 Consumer 按照集群消费的方式消费消息,按照平均分配策略进行负载均衡得到的结果是:第一个 Consumer 消费 3 个 Queue,第二个 Consumer 消费 1 个 Queue.如果增加 Consumer,每个 Consumer 分配到的 Queue 会相应减少.Rocket MQ 的负载均衡策略规定:**<u style='color:#e74c3c'>Consumer 数量应该小于等于 Queue 数量,如果 Consumer 超过 Queue 数量,那么多余的 Consumer 将不能消费消息</u>**.在一个 Consumer Group 内,Queue 和 Consumer 之间的对应关系是一对多的关系:`一个 Queue 最多只能分配给一个 Consumer,一个 Cosumer 可以分配得到多个 Queue`.这样的分配规则,每个 Queue 只有一个消费者,可以避免消费过程中的多线程处理和资源锁定,有效提高各 Consumer 消费的并行度和处理效率.[origin link](https://mp.weixin.qq.com/s/1pFddUuf_j9Xjl58MBnvTQ)

### 3.2 消费模式

MQ 说: 不要堆积,于是便有了消费者.

消费流程: `服务端获取订阅关系,得到tag的hash集合codeSet` -> `遍历ConsumerQueue记录,判断标签hashCode是否在codeSet中` -> `客户度过滤:tag的字符串值做对比`.

RocketMQ 有两种消费模式:`BROADCASTING(广播模式)`和`CLUSTERING(集群模式)`,默认的是 `集群消费模式`.

源码: `com.alibaba.rocketmq.client.consumer.DefaultMQPushConsumer`(都是使用 pull 方式拉取消息)

```java
 public DefaultMQPushConsumer(String consumerGroup, RPCHook rpcHook, AllocateMessageQueueStrategy allocateMessageQueueStrategy) {
    // MessageModel.CLUSTERING为集群消费模式
    this.messageModel = MessageModel.CLUSTERING;
    this.consumeFromWhere = ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET;
    this.consumeTimestamp = UtilAll.timeMillisToHumanString3(System.currentTimeMillis() - 1800000L);
    this.subscription = new HashMap();
    this.consumeThreadMin = 20;
    this.consumeThreadMax = 64;
    this.adjustThreadPoolNumsThreshold = 100000L;
    this.consumeConcurrentlyMaxSpan = 2000;
    this.pullThresholdForQueue = 1000;
    this.pullInterval = 0L;
    this.consumeMessageBatchMaxSize = 1;
    this.pullBatchSize = 32;
    this.postSubscriptionWhenPull = false;
    this.unitMode = false;
    this.consumerGroup = consumerGroup;
    this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;
    this.defaultMQPushConsumerImpl = new DefaultMQPushConsumerImpl(this, rpcHook);
}
```

代码设置消费模式,示例如下

```java
 try {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(String.valueOf(id));
    consumer.setNamesrvAddr(MqSetting.NAME_SERVER_HOST);
    // 设置消费类型,集群还是广播
    consumer.setMessageModel(MessageModel.BROADCASTING);


    // 订阅主题和标签
    consumer.subscribe(MqSetting.TOPIC_NAME, "*");
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

    // 调节pull时间间隔,5s
    consumer.setPullInterval(5 * 1000);

    // 控制每次pull次消息数量
    consumer.setPullBatchSize(1);

    // 注册监听器
    consumer.registerMessageListener(new MsgListener("consumer-" + id));

    consumer.start();
} catch (Exception e) {
    e.printStackTrace();
}
```

#### 3.2.1 广播消费模式

广播消费模式:<u>topic 下的同一条消息将被集群内的所有消费者消费一次.</u>

![](imgs/consume-brocast.jpg)

适用场景: `req -> ehcache -> redis -> db,使用 mq 来做 ehcache 缓存的数据清理`.

#### 3.2.2 集群消费模式

集群消费模式:<u>topic 下的同一条消息只允许被其中一个消费者消费</u>

<span style='color:#e74c3c'>FBI WARNING</span>: **在集群消费模式下,一个 Topic 的消息被多个 Consumer Group 消费的行为比较特殊.每个 Consumer Group 会分别将该 Topic 的消息消费一遍;在每一个 Consumer Group 内,各 Consumer 通过负载均衡的方式消费该 Topic 的消息.**

![](imgs/consume-cluster.jpg)

适用场景: `如 lians 的手机通知短信的发送(存在多个手机短信 mq 消费端)`.

### 3.3 消息的 ack

[rocketmq ack 消费确认机制 link](https://zhuanlan.zhihu.com/p/25265380)

#### 3.3.1 消费策略

```java
//默认策略,从该队列最尾开始消费,即跳过历史消息
CONSUME_FROM_LAST_OFFSET

//从队列最开始开始消费,即历史消息（还储存在broker的）全部消费一遍
CONSUME_FROM_FIRST_OFFSET

//从某个时间点开始消费,和setConsumeTimestamp()配合使用,默认是半个小时以前
CONSUME_FROM_TIMESTAMP
```

#### 3.3.2 ack 代码示例

```java
/**
* 消费者
*/
static class MsgListener implements MessageListenerConcurrently {
/**
 * 线程名称
 */
private String threadName;

private static Supplier<String> dateSupplier = () -> {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    return sdf.format(new Date());
};

MsgListener(String threadName) {
    this.threadName = threadName;
}

@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
    try {
        Message message = list.get(0);
        System.out.println(dateSupplier.get() + " - " + threadName + " - " + message.getTopic() + "." + message.getTags() + ":" + new String(message.getBody()));

        // 消费成功
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (Exception e) {
        // 消费失败,重试
        return ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }
}
}
```

---

## 4. 注意事项

### 4.1 topic 与 tag 的设计

Q: 到底什么时候该用 Topic,什么时候该用 Tag?[详情 link](https://help.aliyun.com/document_detail/95837.html?spm=a2c4g.11186623.6.613.405c1da9JMnX5O)

A: 设计参考如下:

- 消息类型是否一致:如普通消息,事务消息,定时消息,顺序消息,不同的消息类型使用不同的 Topic,无法通过 Tag 进行区分.

- 业务是否相关联:没有直接关联的消息,如淘宝交易消息,京东物流消息使用不同的 Topic 进行区分；而同样是天猫交易消息,电器类订单、女装类订单、化妆品类订单的消息可以用 Tag 进行区分.

- 消息优先级是否一致:如同样是物流消息,盒马必须小时内送达,天猫超市 24 小时内送达,淘宝物流则相对会会慢一些,不同优先级的消息用不同的 Topic 进行区分.

- 消息量级是否相当:有些业务消息虽然量小但是实时性要求高,如果跟某些万亿量级的消息使用同一个 Topic,则有可能会因为过长的等待时间而『饿死』,此时需要将不同量级的消息进行拆分,使用不同的 Topic.

场景: 比如会员成长值的增长,使用 topic:`sys_xyz_topic`,tag 有如下类型值:

```json
{
  "SCAN_MATERIAL": "浏览内容",
  "FIRST_UPLOAD_POLICY": "首次完成上传保单",
  "DAILY_SIGNED": "每日签到",
  "REGISTRATION_SUCCESS": "完成付费（支付成功），挂号",
  "BIND_WX_CODE": "绑定微信号",
  "SUBSCRIBE": "关注公众号",
  "SHARE_MATERIAL": "分享内容",
  "FIRST_REGISTRATION": "首次挂号",
  "POLICY_SUCCESS": "完成付费（支付成功），投保",
  "BIND_PHONE": "绑定手机号",
  "INVITE_USER": "邀请用户",
  "FIRST_BUY_POLICY": "首次投保",
  "FIRST_PRODUCT_EVALUATE": "首次查看某个产品分析或上传产品",
  "FIRST_DIAGNOSTIC": "首次完成智诊",
  "BIRTHDAY_INSURE": "生日投保"
}
```

Q: 那我可不可以把 tag 设置为 topic 呀?

A: 江湖有一句老话,`可以,但是没必要`. 微笑.jpg

### 4.2 消费幂等

消费幂等[详情 link](https://help.aliyun.com/document_detail/44397.html),保证消息的唯一性:

- 发送时消息重复
- 投递时消息重复
- 负载均衡时消息重复（包括但不限于网络抖动、Broker 重启以及订阅方应用重启）

Fun fact: `rocketmq确认消息肯定会被消费>=1次`.

所以在一些被消费一次的消息里面,做幂等校验,相当重要.如转账之类的业务场景.

**解决方法**

生产者

```java
Message message = new Message();
message.setKey("ORDERID_100");
SendResult sendResult = producer.send(message);
```

消费者

```java
consumer.subscribe("ons_test", "*", new MessageListener() {
    public Action consume(Message message, ConsumeContext context) {
        String key = message.getKey()
        // 根据业务唯一标识的 key 做幂等处理
        if(isUniqueOrderKey(key)){
            // continue do anything you want.
        }
    }
});
```

场景: lians 手机发送消息消费端,如果使用`msg_id`来做 mq 消息去重的处理.(`t_template_sms.mq_message_id`,如果没猜错的话)

### 4.3 订阅关系一致

**订阅关系一致** [link](https://help.aliyun.com/document_detail/43523.html?spm=a2c4g.11186623.6.605.2a381da95V6B1X)

订阅关系一致指的是同一个消费者 Group ID 下所有 Consumer 实例的处理逻辑必须完全一致.<u style='color:#e74c3c'>一旦订阅关系不一致,消息消费的逻辑就会混乱,甚至导致消息丢失</u>.

由于消息队列 RocketMQ 的订阅关系主要由 Topic + Tag 共同组成,因此,保持订阅关系一致意味着同一个消费者 Group ID 下所有的实例需在以下两方面均保持一致:

- 订阅的 Topic 必须一致

- 订阅的 Topic 中的 Tag 必须一致

所以,订阅关系的一致性几乎是整一个 rocketmq 正常运行的前提,不要浪,不要浪,不要浪,over!!!

Q: 为什么要保持消费组的订阅一致性呢?

A: 在 broker 里面消费者都是以 group 来划分的,管理关系如下,如果不一致的话,后面注册订阅信息会覆盖原来的注册订阅信息.

`org.apache.rocketmq.broker.client.ConsumerManager`:

```java
private final ConcurrentMap<String/* Group */, ConsumerGroupInfo> consumerTable =
  new ConcurrentHashMap<String, ConsumerGroupInfo>(1024);
```

---

## 5. 使用案例

mq 的使用场景.

### 5.1 lians 的短信消息

改造消息发送前

![](imgs/lians-sms-before.jpg)

改造消息发送后

![](imgs/lians-sms-after.jpg)

- 解耦,确定消息服务出问题的情况下,不影响主流程.
- 保证消息发送.

### 5.2 爬虫优化

知乎的一个小爬虫,现在获取到的话题数量.

```sql
mysql> select count(1) from t_zhihu_topic
+----------+
| count(1) |
+----------+
|     7058 |
+----------+
1 row in set (0.00 sec)
```

如果每一个话题下面有 10(如生活下面那种话题,远远大这个数量) 个回答,那么总回答大概在: 7058x10 个.因为知乎存在反爬,同一个 ip 要请求间隔要>10s 才不会被禁用(另一个方案是使用 ip 代理池).这个可以看出一台服务器的话,爬取这个知乎答案,简直就是锻炼耐性.是的,你没猜错,现在就是一台服务器在爬取. 泪流满面.gif

Q:那么该怎么改进这个爬取的玩意呢?

A:把那些知乎话题下面的回答的 url,放到 mq 里面,在多个服务器上面,开启多个消费端来消费,这样子就能加快爬取的速度了.

**version1 架构**

![](imgs/zhihu-v1.png)

**version2 架构**

![](imgs/zhihu-v2.png)

- 易于横向扩展

---

## 6. 参考资料

| name                                     | link                                                                                               |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------- |
| RocketMq 官方文档                        | [link](http://rocketmq.apache.org/docs/quick-start/)                                               |
| RocketMq 博客                            | [link](https://www.cnblogs.com/qdhxhz/p/11094624.html)                                             |
| RocketMQ 消息发送的高可用设计            | [link](http://objcoding.com/2019/04/06/rocketmq-fault-strategy/)                                   |
| 分布式开放消息系统(RocketMQ)的原理与实践 | [link](https://www.cnblogs.com/xuwc/p/9034352.html)                                                |
| RocketMq 分布式事务                      | [link](https://www.jianshu.com/p/cc5c10221aa1)                                                     |
| RocketMq 优秀样例                        | [link](https://github.com/apache/rocketmq/blob/master/docs/cn/RocketMQ_Example.md)                 |
| CSDN 博客:java 使用 rocketMq             | [link](https://blog.csdn.net/zhangcongyi420/article/details/82593982)                              |
| 轻松搞定 RocketMQ 入门                   | [link](https://segmentfault.com/a/1190000015951993)                                                |
| 搭建 RocketMQ 踩的坑                     | [link](https://blog.csdn.net/c_yang13/article/details/76836753)                                    |
| RocketMq 订阅关系                        | [link](https://help.aliyun.com/document_detail/43523.html?spm=a2c4g.11186623.6.605.2a381da95V6B1X) |
