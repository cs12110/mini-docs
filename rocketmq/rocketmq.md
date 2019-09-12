# Rocketmq 的结构以及高可用设计

Create at: 2019-08-11 12:00:00

Author: huanghuapeng@ingbaobei.com

---

## 1. RocketMq 基础知识

[RocketMq 基础知识 link](http://rocketmq.apache.org/docs/core-concept/)

### 1.1 Why RocketMq?

Q: 为什么使用 rocketmq,而不使用其他的 mq 呢?

A: 官方给出的回答.

```bash
# Based on our research, with increased queues and virtual topics in use, ActiveMQ IO module reaches a bottleneck. We tried our best to solve this problem through throttling, circuit breaker or degradation, but it did not work well. So we begin to focus on the popular messaging solution Kafka at that time. Unfortunately, Kafka can not meet our requirements especially in terms of low latency and high reliability, see here for details.

大意: 随着队列和topic的增长,ActiveMq遇到了瓶颈.kafka的低延迟和高可靠不达标.

# In this context, we decided to invent a new messaging engine to handle a broader set of use cases, ranging from traditional pub/sub scenarios to high volume real-time zero-loss tolerance transaction system.

大意: 在这样子的前提下,决定去弄一个新的消息引擎来处理消息,从传统的发布/订阅场景到`高容量`,`实时`,`零丢失`事务系统.
```

所以, ta 们自己一拍脑袋,那为什么不弄一个自己的 mq 出来?(cv 工程师已哭晕在厕所.)

下面是 RocketMq 和 Kafka 以及 ActiveMq 的对比(注意: 站在偏向 RocketMq 的角度的对比)

![compare](imgs/rocketmq-compare.png)

- ordered message: FIFO(First in first out).
- scheduled message: 定时发送消息.
- batched message: 批量消息(相当 jdbc 的批处理).
- broadcast message: 广播消息.

### 1.2 rocketmq 的结构

Q: 那么一般高可用的 rocketmq 的架构是怎样子呢?

A: 请看下面这个被复制烂的架构图.[rocketmq 架构 link](http://rocketmq.apache.org/docs/rmq-arc/)

![](imgs/rocketmq-server.png)

名称解释

| 名称                | 说明                                                                                                                                                                                                                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Producer`          | 消息生产者,位于用户的进程内,Producer 通过 NameServer 获取所有 Broker 的路由信息,根据负载均衡策略选择将消息发到哪个 Broker,然后调用 Broker 接口提交消息.                                                                                                                                                                                      |
| `Producer Group`    | 生产者组,简单来说就是多个发送同一类消息的生产者称之为一个生产者组.                                                                                                                                                                                                                                                                           |
| `Consumer`          | 消息消费者,位于用户进程内.Consumer 通过 NameServer 获取所有 broker 的路由信息后,向 Broker 发送 Pull 请求来获取消息数据.Consumer 可以以两种模式启动,广播（Broadcast）和集群（Cluster）,广播模式下,一条消息会发送给所有 Consumer,集群模式下消息只会发送给一个 Consumer.                                                                        |
| `Consumer Group`    | 消费者组,和生产者类似,消费同一类消息的多个 Consumer 实例组成一个消费者组.                                                                                                                                                                                                                                                                    |
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

## 2. 高可用设计

### 2.1 消息的存储结构

rocketmq 消息存储结构如下所示(origin from `RocketMQ技术内幕`)

![](imgs/rocketmq-data-struct.jpg)

- CommitLog: 消息存储文件,所有消息主体的消息都存在 CommitLog 文件里面.
- ConsumeQueue: 消息消费队列.消息到达 CommitLog 文件后,将异步转发消息到消费队列,供消费者消费.
- IndexFile: 消息索引文件.存储消息的 key 和 offset 对应关系.
- 事务状态: 存储每条消息的事务的状态
- 定时消息服务: 每一个延迟级别对应一个消息消费队列,并存储延迟队列的消息拉取进度.

### 2.2 消息的 ack

[rocketmq ack](https://zhuanlan.zhihu.com/p/25265380)

### 2.3 消费模式

#### 2.3.1 广播

Q: 多个消费者监听同一个 topic,不监听 tag

生产者生成两条消息到 topic:`4fun-topic`

```jvaa
Send: {index=0, value=0} to:4fun-topic.*
Send: {index=1, value=1} to:4fun-topic.*
```

开启两个消费者,监听 topic:`4fun-topic`

```java
Consumer startup is success
Consumer startup is success
2019-08-13 20:25:16 - consumer-11 - 4fun-topic.*:{index=0, value=0}
2019-08-13 20:25:16 - consumer-12 - 4fun-topic.*:{index=0, value=0}
2019-08-13 20:25:18 - consumer-12 - 4fun-topic.*:{index=1, value=1}
2019-08-13 20:25:18 - consumer-11 - 4fun-topic.*:{index=1, value=1}
```

#### 2.3.2

Q: 多个消费者监听同一个 topic 的同一个 tag

生产者往: `4fun-topic`的 tag:`tag`发送两条消息

```java
Send: {index=0, value=0} to:4fun-topic.tag
Send: {index=1, value=1} to:4fun-topic.tag
```

开启两个消费者,监听 topic:`4fun-topic`,tag:`tag`

```java
Consumer startup is success
Consumer startup is success
2019-08-13 20:24:00 - consumer-12 - 4fun-topic.tag:{index=0, value=0}
2019-08-13 20:24:00 - consumer-11 - 4fun-topic.tag:{index=0, value=0}
2019-08-13 20:24:02 - consumer-12 - 4fun-topic.tag:{index=1, value=1}
2019-08-13 20:24:02 - consumer-11 - 4fun-topic.tag:{index=1, value=1}
```

#### 2.3.3

Q: 多个消费者监听同一个 topic 的不同的 tag

生产者发送一条消息到 topic:`4fun-topic`,tag:`tag0`,发送一条消息到 topic:`4fun-topic`,tag:`tag1`

```java
Send: {index=0, value=0} to:4fun-topic.tag0
Send: {index=1, value=1} to:4fun-topic.tag1
```

消费者 1 监听 topic:`4fun-topic`,tag:`tag0`,消费者 2 监听 topic:`4fun-topic`,tag:`tag1`

```java
Consumer startup is success
Consumer startup is success
2019-08-13 20:19:32 - consumer-11 - 4fun-topic.tag0:{index=0, value=0}
2019-08-13 20:19:35 - consumer-12 - 4fun-topic.tag1:{index=1, value=1}
```

---

## 3. 实际使用案例

### 3.1 lians 的短信消息

改造消息发送前

![](imgs/lians-sms-before.jpg)

改造消息发送后

![](imgs/lians-sms-after.jpg)

- 解耦,确定消息服务出问题的情况下,不影响主流程.
- 保证消息发送.

### 3.2 爬虫优化

偶尔喜欢刷知乎,所以弄了一个爬虫去爬取知乎的高赞回答.

现在获取到的话题数量

```sql
mysql> select count(1) from t_zhihu_topic;
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

version1 架构

![](imgs/zhihu-v1.png)

version2 架构

![](imgs/zhihu-v2.png)

注: 爬虫服务器数量和银行卡余额成正比关系.

---

## 4. 注意事项

### 4.1 topic 与 tag 的设计

Q: 到底什么时候该用 Topic,什么时候该用 Tag？[详情 link](https://help.aliyun.com/document_detail/95837.html?spm=a2c4g.11186623.6.613.405c1da9JMnX5O)

A: 设计参考如下:

- 消息类型是否一致:如普通消息,事务消息,定时消息,顺序消息,不同的消息类型使用不同的 Topic,无法通过 Tag 进行区分.

- 业务是否相关联:没有直接关联的消息,如淘宝交易消息,京东物流消息使用不同的 Topic 进行区分；而同样是天猫交易消息,电器类订单、女装类订单、化妆品类订单的消息可以用 Tag 进行区分.

- 消息优先级是否一致:如同样是物流消息,盒马必须小时内送达,天猫超市 24 小时内送达,淘宝物流则相对会会慢一些,不同优先级的消息用不同的 Topic 进行区分.

- 消息量级是否相当:有些业务消息虽然量小但是实时性要求高,如果跟某些万亿量级的消息使用同一个 Topic,则有可能会因为过长的等待时间而『饿死』,此时需要将不同量级的消息进行拆分,使用不同的 Topic.

### 4.2 消费幂等

消费幂等[详情 link](https://help.aliyun.com/document_detail/44397.html),保证消息的唯一性:

- 发送时消息重复
- 投递时消息重复
- 负载均衡时消息重复（包括但不限于网络抖动、Broker 重启以及订阅方应用重启）

解决方法

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
    }
});
```

### 4.3 订阅关系一致

**订阅关系一致** [link](https://help.aliyun.com/document_detail/43523.html?spm=a2c4g.11186623.6.605.2a381da95V6B1X)

订阅关系一致指的是同一个消费者 Group ID 下所有 Consumer 实例的处理逻辑必须完全一致.一旦订阅关系不一致,消息消费的逻辑就会混乱,甚至导致消息丢失.消息队列 RocketMQ 里的一个消费者 Group ID 代表一个 Consumer 实例群组.对于大多数分布式应用来说,一个消费者 Group ID 下通常会挂载多个 Consumer 实例.

由于消息队列 RocketMQ 的订阅关系主要由 Topic + Tag 共同组成,因此,保持订阅关系一致意味着同一个消费者 Group ID 下所有的实例需在以下两方面均保持一致:

- 订阅的 Topic 必须一致

- 订阅的 Topic 中的 Tag 必须一致

---

## 5. The end

<u>**每一个不写代码的日子,都是对生命的辜负** </u>. by `弗里德里希·这不是我说的·尼采`

---

## 6. 参考资料

| 文档名称                                 | 连接地址                                                         |
| ---------------------------------------- | ---------------------------------------------------------------- |
| RocketMq 官方文档                        | [link](http://rocketmq.apache.org/docs/quick-start/)             |
| RocketMq 博客                            | [link](https://www.cnblogs.com/qdhxhz/p/11094624.html)           |
| RocketMQ 消息发送的高可用设计            | [link](http://objcoding.com/2019/04/06/rocketmq-fault-strategy/) |
| 分布式开放消息系统(RocketMQ)的原理与实践 | [link](https://www.jianshu.com/p/453c6e7ff81c)                   |
