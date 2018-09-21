# kafka 基础教程

Apache kafka 是消息中间件的一种(如 RabbitMQ).

---

## 1. kafka 简介

关于`Message Queue`例子:

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

---

## 2. kafka 安装

1.启动 kafka 自带的 zookeeper

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/zookeeper-server-start.sh config/zookeeper.properties &
```

2.启动 kafka

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-server-start.sh  config/server.properties
```

3.创建 topic

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh  --create --zookeeper 10.10.2.70:2181 --replication-factor 1 --partitions 1 --topic test
Created topic "test".
[dev@aya kafka_2.12-1.0.0]$
```

4.查看创建的 topic

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh  --list --zookeeper 10.10.2.70:2181
test
[dev@aya kafka_2.12-1.0.0]$
```

5.发送消息

```sh
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-console-producer.sh --broker-list 10.10.2.70:9092 --topic test
>This is first message
>This is secondly message
>^C  --> ctrl+c退出
```

6.消费消息

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

创建消息队列: `adv_success_mq`

```sh
hadoop231:/opt/kafka/kafka_2.12-0.10.2.0/bin # ./kafka-topics.sh --create --zookeeper hadoop233:2181,hadoop234:2181,hadoop235:2181 --replication-factor 2 --partition 1 --topic adv_success_mq
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

---

## 5. Java 整合 kafka

Java 对 Kafka 说: 你喊呀,喊破喉咙都没人来救你的.

### 5.1 maven 依赖

```xml
<dependency>
	<groupId> org.apache.kafka</groupId>
	<artifactId> kafka_2.10</artifactId>
	<version> 0.8.0</version>
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
```

### 5.2 常量配置文件`Conf.java`

```java
package com.mq;

/**
 *
 * 常用配置文件
 * <p>
 *
 * @author 3306 2017年11月8日
 * @see
 * @since 1.0
 */
public class Conf {

	/**
	 * zookeeper地址
	 */
	public static final String ZOOKEEPER_HOST = "10.10.2.70:2181";

	/**
	 * 用户组Id
	 */
	public static final String GROUP_ID = "kafka";

	/**
	 * 使用topic
	 */
	public static final String TOPIC0 = "replication-topic";

	/**
	 * kafka服务器地址
	 */
	public static final String KAFKA_SERVER = "10.10.2.70:9092";

}
```

### 5.3 消息生产类

```java
package com.mq;

import java.util.Properties;

import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

/**
 * 消息生成类
 *
 * <p>
 * detailed comment
 *
 * @author huanghuapeng 2017年11月8日
 * @see
 * @since 1.0
 */
public class MqProducer implements Runnable {

	private String topic;
	private Producer<Integer, String> producer;

	public MqProducer(String topic) {
		Properties props = new Properties();
		props.put("serializer.class", "kafka.serializer.StringEncoder");
		props.put("metadata.broker.list", Conf.KAFKA_SERVER);

		this.topic = topic;

		producer = new Producer<Integer, String>(new ProducerConfig(props));
	}

	public void run() {
		int msgIndex = 1;
		try {
			while (true) {
				String msg = "sending message" + (msgIndex++);
				System.out.println("Producer:" + msg);

				producer.send(new KeyedMessage<Integer, String>(topic, msg));

				Thread.sleep(500);
			}
		} catch (Exception e) {
			e.printStackTrace();
			System.exit(1);
		}
	}

}
```

### 5.4 消息消费类

```java
package com.mq;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import kafka.consumer.Consumer;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;

/**
 * 消息生成类
 *
 * <p>
 * detailed comment
 *
 * @author huanghuapeng 2017年11月8日
 * @see
 * @since 1.0
 */
public class MqConsumer implements Runnable {

	private String topic;
	private ConsumerConnector consumer;

	public MqConsumer(String topic) {
		this.topic = topic;
		consumer = Consumer.createJavaConsumerConnector(buildConsumerConf());
	}

	private static ConsumerConfig buildConsumerConf() {
		Properties props = new Properties();

		props.put("zookeeper.connect", Conf.ZOOKEEPER_HOST);
		props.put("group.id", Conf.GROUP_ID);
		props.put("zookeeper.session.timeout.ms", "40000");
		props.put("zookeeper.sync.time.ms", "200");
		props.put("auto.commit.interval.ms", "1000");

		return new ConsumerConfig(props);

	}

	public void run() {

		Map<String, Integer> topicCountMap = new HashMap<String, Integer>();;

		topicCountMap.put(topic, 1);

		Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer
				.createMessageStreams(topicCountMap);

		KafkaStream<byte[], byte[]> stream = consumerMap.get(topic).get(0);

		ConsumerIterator<byte[], byte[]> iterator = stream.iterator();

		try {
			while (iterator.hasNext()) {
				System.out.println(
						"Consumer: " + new String(iterator.next().message()));;
				try {
					Thread.sleep(1000);
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
```

### 5.5 测试类

```java
package com.mq;

public class TestMq {

	public static void main(String[] args) {
		Thread producer = new Thread(new MqProducer(Conf.TOPIC0));
		Thread consumer = new Thread(new MqConsumer(Conf.TOPIC0));

		producer.start();
		consumer.start();
	}

}
```

---

## 6. 参考资料

a. [CSDN 博客](http://blog.csdn.net/hmsiwtv/article/details/46960053)
