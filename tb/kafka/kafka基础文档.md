# kafka 基础教程

Apache kafka 是消息中间件的一种(如 RabbitMQ).

---

## 1. kafka 简介

基础概念你我他.

### 1.1 Mq

假设消费者在处理鸡蛋时噎住了(消费者系统 down),生产者还在生产鸡蛋,那新生产的鸡蛋就丢失了.

再比如生产者很强劲(大交易量的情况),生产者每秒生产 100 个鸡蛋,消费者只能每秒消费 50 个,不用一会消费者就吃不消了(消息堵塞,导致系统超时),消费者拒绝再吃,鸡蛋又丢失.

为了应对这种情况,我们把一个篮子放在消费者和生产者之间,生产出来的鸡蛋放到篮子里面,消费者从篮子里面取出鸡蛋,这样鸡蛋就不会丢失,都放在篮子里面了.这个篮子就是`kafka`

> producer: 生产者(生产鸡蛋)
>
> consumer: 消费者(消费鸡蛋)
>
> topic: 标签,生产者为生产出来的每一鸡蛋贴上标签(topic),不同消费者根据标签(topic)消费响应的鸡蛋
>
> broker: kafka 的篮子

### 1.2 Kafka

那么 kafka 具有哪些优势呢?

- 时间复杂度为 O(1)的方式提供消息持久化能力,即使对 TB 级以上数据也能保证常数时间的访问性能
- 高吞吐率,即使在非常廉价的商用机器上也能做到单机支持每秒 100K 条消息的传输
- 支持 Kafka Server 间的消息分区,及分布式消费,同时保证每个 partition 内的消息顺序传输
- 同时支持离线数据处理和实时数据处理

---

## 2. kafka 安装

## 2.1 启动 kafka 自带的 zookeeper

注意: **如果使用另外的 zookeeper 集群,请忽略这一步,同时请把配置文件里面的 zookeeper 连接地址修改为自己的 zookeeper 集群地址即可.**

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/zookeeper-server-start.sh config/zookeeper.properties &
```

### 2.2 启动 kafka

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-server-start.sh  config/server.properties
```

### 2.3 创建 topic

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh  --create --zookeeper 10.10.2.70:2181 --replication-factor 1 --partitions 1 --topic test
Created topic "test".
```

### 2.4 查看创建的 topic

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh  --list --zookeeper 10.10.2.70:2181
test
```

### 2.5 发送消息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-producer.sh --broker-list 10.10.2.70:9092 --topic test
>This is first message
>This is secondly message
>^C  --> ctrl+c退出
```

### 2.6 消费消息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-consumer.sh  --zookeeper 10.10.2.70:2181 --topic test --from-beginning
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
This is first message
This is secondly message
```

---

## 3. kafka 集群

上面只是启动了单个 broker,现在启动三个 broker,每个 broker 都有自己的 properties 文件设置,分别修改里面的三个参数`broker.id`,`port`,`log.dir`,修改成不同的值.

### 3.1 复制配置文件 server.properties

```sh
[dev@aya config]$ cp server.properties  server-1.properties
[dev@aya config]$ cp server.properties  server-2.properties
```

### 3.2 修改配置文件`server-x.properties`

`server-1.properties`

```sh
broker-id=1
port= 9093
log.dir=/tmp/kafka-logs-1

#特别注意
host.name=10.10.2.70 #服务器ip
zookeeper.connect=10.10.2.70:2181 #服务器ip地址
```

`server-2.properties`

```sh
broker-id=2
port= 9094
log.dir=/tmp/kafka-logs-2

#特别注意
host.name=10.10.2.70 #服务器ip
zookeeper.connect=10.10.2.70:2181 #服务器ip地址
```

### 3.3 启动 server1 和 server2 两个节点

```sh
[dev@aya kafka_2.12-1.0.0]$bin/kafka-server-start.sh config/server-1.properties &

[dev@aya kafka_2.12-1.0.0]$bin/kafka-server-start.sh config/server-2.properties &
```

### 3.4 创建拥有 3 个副本的 topic

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh --create --zookeeper 10.10.2.70:2181 --replication-factor 3 --partitions 1 --topic replication-topic

Created topic "replication-topic".
```

### 3.5 查看每一个节点的信息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh --describe --zookeeper 10.10.2.70:2181 --topic replication-topic
Topic:replication-topic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: replication-topic	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
[dev@aya kafka_2.12-1.0.0]$
```

第一行是对所有分区进行描述,然后对每个分区对应一行.

- leader:负责处理消息的读写,leader 是从所有的节点随机选取的
- replicas:列出所有的副本节点,不管节点是否在服务中
- isr:正在服务的节点

在上面例子中,leader 为节点 1

### 3.6 往集群发送消息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic replication-topic
>This is replication message
>Hello world
>^C
```

### 3.7 消费集群消息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic replication-topic --from-beginning
This is replication message
Hello world
```

### 3.8 测试容错能力

现在节点 1 位`leader`,那先`kill`掉节点 1

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh --describe --zookeeper localhost:2181  --topic replication-topic
Topic:replication-topic	PartitionCount:1	ReplicationFactor:3	Configs:
	Topic: replication-topic	Partition: 0	Leader: 2	Replicas: 1,2,0	Isr: 2,0
[dev@aya kafka_2.12-1.0.0]$
```

leader 切换为节点 2,isr 为 2,0

再次测试获取信息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-consumer.sh  --zookeeper localhost:2181 --from-beginning --topic replication-topic
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
This is replication message
Hello world
```

可以看出 kafka 成功切换.

---

## 4. 常用命令

### 4.1 创建队列

```sh
hadoop231:/opt/kafka/kafka_2.12-0.10.2.0/bin # ./kafka-topics.sh --create --zookeeper hadoop233:2181,hadoop234:2181,hadoop235:2181 --replication-factor 2 --partitions 1 --topic adv_success_mq
```

### 4.2 显示队列

```sh
hadoop231:/opt/kafka/kafka_2.12-0.10.2.0/bin # ./kafka-topics.sh   --zookeeper hadoop233:2181 --list
...
adv_failure_mq
adv_success_mq
...
```

### 4.3 删除队列

删除的时候,需要把 kafka 和 zookeeper 里面的都清理掉

删除 kafka 的消息队列

```sh
hadoop231:/opt/kafka/kafka_2.12-0.10.2.0/bin # ./kafka-topics.sh  --delete --zookeeper hadoop233:2181 --topic adv_success_mq
```

删除 zookeeper[/brokers/topic/yourMqName]文件夹(Mq 文件可能不存在)

```sh
hadoop233:/opt/hadoop/zookeeper-3.4.9/bin # ./zkCli.sh
[zk: localhost:2181(CONNECTED) 1] rmr /brokers/topics/adv_success_mq
```

删除 zookeeper[/admin/delete_topic/]文件夹(Mq 文件可能不存在)

```sh
[zk: localhost:2181(CONNECTED) 1] rmr /admin/delete_topics
```

### 4.4 修改 partitions

注意: **partitions 的数量只能单调递增.**

```sh
^C[root@bi141 kafka_2.11-2.1.0]# bin/kafka-topics.sh --describe --zookeeper 10.10.1.141:2181 --topic streaming_topic
Topic:streaming_topic	PartitionCount:2	ReplicationFactor:2	Configs:
	Topic: streaming_topic	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: streaming_topic	Partition: 1	Leader: 2	Replicas: 2,0	Isr: 2,0
[root@bi141 kafka_2.11-2.1.0]# bin/kafka-topics.sh --zookeeper 10.10.1.141 --alter --partitions 5 --topic streaming_topic
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected
Adding partitions succeeded!
[root@bi141 kafka_2.11-2.1.0]# bin/kafka-topics.sh --describe --zookeeper 10.10.1.141:2181 --topic streaming_topic
Topic:streaming_topic	PartitionCount:5	ReplicationFactor:2	Configs:
	Topic: streaming_topic	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: streaming_topic	Partition: 1	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: streaming_topic	Partition: 2	Leader: 0	Replicas: 0,2	Isr: 0,2
	Topic: streaming_topic	Partition: 3	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: streaming_topic	Partition: 4	Leader: 2	Replicas: 2,0	Isr: 2,0
[root@bi141 kafka_2.11-2.1.0]#
```

---

## 5. Java 整合 kafka

WARNING: **注意创建消费者队列时 partitions 和客户端消费的情况**.

该程序实现: 一个生产者/多个消费者的生产模式,请知悉.

### 5.1 创建消息队列

创建测试的消息队列: <u>请必须注意队列的 topic partitions 是否符合创建的数量</u>.

```sh
[root@bi141 kafka_2.11-2.1.0]# bin/kafka-topics.sh --create --zookeeper 10.10.1.141:2181,10.10.1.142:2181,10.10.1.143:2181 --replication-factor 2 --partitions 2 --topic streaming_topic
[root@bi141 kafka_2.11-2.1.0]# bin/kafka-topics.sh --describe --zookeeper 10.10.1.141:2181 --topic streaming_topic
Topic:streaming_topic	PartitionCount:2	ReplicationFactor:2	Configs:
	Topic: streaming_topic	Partition: 0	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: streaming_topic	Partition: 1	Leader: 2	Replicas: 2,0	Isr: 2,0
```

### 5.2 maven 依赖

```xml
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka_2.10</artifactId>
	<version>0.8.0</version>
</dependency>

<dependency>
	<groupId>log4j</groupId>
	<artifactId>log4j</artifactId>
	<version>1.2.15</version>
	<exclusions>
		<exclusion>
			<groupId>javax.jms</groupId>
			<artifactId>jms</artifactId>
		</exclusion>
		<exclusion>
			<groupId>com.sun.jdmk</groupId>
			<artifactId>jmxtools</artifactId>
		</exclusion>
		<exclusion>
			<groupId>com.sun.jmx</groupId>
			<artifactId>jmxri</artifactId>
		</exclusion>
	</exclusions>
</dependency>

<dependency>
	<groupId>org.projectlombok</groupId>
	<artifactId>lombok</artifactId>
	<version>1.18.4</version>
</dependency>

<!-- fastjson -->
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.30</version>
</dependency>
```

### 5.3 配置类

```java
package com.pkgs.conf;

/**
 * 配置类
 * <p>
 * <p/>
 *
 * @author cs12110 created at: 2019/2/14 13:40
 * <p>
 * since: 1.0.0
 */
public class KafkaConf {

    public static final String ZOOKEEPER_URL = "10.10.1.141:2181,10.10.1.142:2181,10.10.1.143:2181";

    public static final String GROUP_ID = "streaming";

    public static final String STREAMING_TOPIC_NAME = "streaming_topic";

    public static final String KAFKA_SERVER_URL = "10.10.1.141:20000,10.10.1.141:20001,20.10.1.141:20002";

}
```

### 5.4 实体类

```java
package com.pkgs.entity;

import com.alibaba.fastjson.JSON;
import lombok.Data;

/**
 * student
 * <p/>
 *
 * @author cs12110 created at: 2019/2/14 14:22
 * <p>
 * since: 1.0.0
 */

@Data
public class StudentEntity {

    private Integer id;

    private String name;

    private String date;

    private float score;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

### 5.5 线程工具类

```java
package com.pkgs.util;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Thread util
 * <p/>
 *
 * @author cs12110 created at: 2019/2/15 9:19
 * <p>
 * since: 1.0.0
 */
public class ThreadUtil {


    public static ExecutorService buildConsumerExecutor(int threadNum) {
        return new ThreadPoolExecutor(
                threadNum,
                threadNum,
                0,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(threadNum),
                new MyThreadFactory("consumer-pool")
        );
    }

    public static ExecutorService buildAppExecutor() {
        int threadNum = 4;

        return new ThreadPoolExecutor(
                threadNum,
                threadNum,
                0,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(threadNum),
                new MyThreadFactory("mq-pool")
        );
    }

    public static class MyThreadFactory implements ThreadFactory {

        private String prefixName;
        private AtomicInteger counter = new AtomicInteger(1);

        MyThreadFactory(String prefixName) {
            this.prefixName = prefixName;
        }

        @Override
        public Thread newThread(Runnable r) {
            String name = prefixName + "-" + counter.getAndIncrement();
            return new Thread(r, name);
        }
    }

}
```

### 5.6 消息生产者

```java
package com.pkgs.worker;

import com.pkgs.conf.KafkaConf;
import com.pkgs.entity.StudentEntity;
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Properties;
import java.util.function.Supplier;

/**
 * 消息提供者
 * <p/>
 *
 * @author cs12110 created at: 2019/2/14 13:47
 * <p>
 * since: 1.0.0
 */
public class MqProvider implements Runnable {

    private String topicName;

    public MqProvider(String topicName) {
        this.topicName = topicName;
    }

    @Override
    public void run() {
        Properties props = new Properties();
        props.put("serializer.class", "kafka.serializer.StringEncoder");
        props.put("metadata.broker.list", KafkaConf.KAFKA_SERVER_URL);
        Producer<String, String> producer = new Producer<>(new ProducerConfig(props));

        int index = 0;
        int times = 5;
        while (index++ < times) {

            StudentEntity stu = buildStu(index);

            // 指定key,发布到不同的分区里面去.
            KeyedMessage<String, String> message = new KeyedMessage<>(topicName, "" + index, stu.toString());
            producer.send(message);
        }
    }

    private static Supplier<String> dateSupplier = () -> {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmmss,SSS");
        return sdf.format(new Date());
    };

    private static StudentEntity buildStu(int id) {
        // 构建消息体
        StudentEntity stu = new StudentEntity();
        stu.setId(id);
        stu.setName("name" + id);
        stu.setDate(dateSupplier.get());
        return stu;
    }
}
```

### 5.7 消息消费者

```java
package com.pkgs.worker;

import com.pkgs.conf.KafkaConf;
import com.pkgs.util.ThreadUtil;
import kafka.consumer.Consumer;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
import kafka.message.MessageAndMetadata;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;

/**
 * 消费者
 * <p/>
 *
 * @author cs12110 created at: 2019/2/14 13:58
 * <p>
 * since: 1.0.0
 */
public class MqConsumer implements Runnable {

    private String topicName;

    /**
     * topic的分区数量
     */
    private static final int PARTITION_NUM = 2;

    public MqConsumer(String topicName) {
        this.topicName = topicName;
        consumer = Consumer.createJavaConsumerConnector(buildConsumerConf());
    }


    private ConsumerConnector consumer;


    private ConsumerConfig buildConsumerConf() {
        Properties props = new Properties();

        props.put("zookeeper.connect", KafkaConf.ZOOKEEPER_URL);
        props.put("group.id", KafkaConf.GROUP_ID);
        props.put("group.name", "streaming-consumer");
        props.put("zookeeper.session.timeout.ms", "40000");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");

        return new ConsumerConfig(props);
    }

    @Override
    public void run() {
        // 注意数量为partition的数量
        Map<String, Integer> topicCountMap = new HashMap<>(1);
        topicCountMap.put(topicName, PARTITION_NUM);

        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topicName);

        // 创建消费线程池
        ExecutorService executor = ThreadUtil.buildConsumerExecutor(PARTITION_NUM);
        for (final KafkaStream<byte[], byte[]> stream : streams) {
            executor.submit(new ConsumerThread(stream));
        }
    }


    static class ConsumerThread implements Runnable {
        private KafkaStream<byte[], byte[]> stream;

        ConsumerThread(KafkaStream<byte[], byte[]> stream) {
            this.stream = stream;
        }

        @Override
        public void run() {
            ConsumerIterator<byte[], byte[]> it = stream.iterator();
            while (it.hasNext()) {
                MessageAndMetadata<byte[], byte[]> mam = it.next();
                System.out.println(Thread.currentThread().getName() + ": partition[" + mam.partition() + "],"
                        + "offset[" + mam.offset() + "], " + new String(mam.message()));
            }
        }
    }
}
```

### 5.7 测试类

```java
package com.pkgs;

import com.pkgs.conf.KafkaConf;
import com.pkgs.util.ThreadUtil;
import com.pkgs.worker.MqConsumer;
import com.pkgs.worker.MqProvider;

import java.util.concurrent.ExecutorService;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/14 13:40
 * <p>
 * since: 1.0.0
 */
public class KafkaApp {

    public static void main(String[] args) {
        ExecutorService service = ThreadUtil.buildAppExecutor();
        service.submit(new MqProvider(KafkaConf.STREAMING_TOPIC_NAME));
        service.submit(new MqConsumer(KafkaConf.STREAMING_TOPIC_NAME));
    }
}
```

测试结果

```java
consumer-pool-2: partition[0],offset[2], {"date":"20190215102634,707","id":2,"name":"name2","score":0.0}
consumer-pool-1: partition[1],offset[3], {"date":"20190215102613,528","id":1,"name":"name1","score":0.0}
consumer-pool-2: partition[0],offset[3], {"date":"20190215102634,713","id":4,"name":"name4","score":0.0}
consumer-pool-1: partition[1],offset[4], {"date":"20190215102634,711","id":3,"name":"name3","score":0.0}
consumer-pool-1: partition[1],offset[5], {"date":"20190215102634,714","id":5,"name":"name5","score":0.0}
```

总结: 从测试结果可以看出,有两个线程消费队列里面的消息.

---

## 6. 参考资料

a. [Kafka 基础教程](http://blog.csdn.net/hmsiwtv/article/details/46960053)

b. [Kafka 队列 partitions 与 Consumer 的关系](https://www.jianshu.com/p/6233d5341dfe)
