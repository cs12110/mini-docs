# Flume 整合 Kafka

在 flume 应用中,根据业务需求,需要把数据存放在 kafka 上.flume 的 kafka sink 可以把数据输出到 kafka 中.问题是: 默认的 flume 的数据只会输出到 kafka 的一个分区中去,那么在客户端,就不能用多个 consumer 来消费消息,这不适用于生产环境.

---

## 1. 详细过程

### 1.1 修改 hosts 映射

```powershell
[dev@aya irs-flume]$ vim /etc/hosts
10.10.2.70 dev1
10.10.2.70 dev2
10.10.2.70 dev3
[dev@aya irs-flume]$
```

### 1.2 创建 topic

```powershell
[dev@aya kafka_2.12-1.0.0]$ bin/kafka-topics.sh  --zookeeper 10.10.2.70:2181 --create --replication-factor 1 --partitions 10 --topic flume-topic
```

### 1.3 配置文件

```properties
# agent组件名称
agent_kafka.sources=r1
agent_kafka.sinks=k1
agent_kafka.channels=c1


# 设置source
agent_kafka.sources.r1.type=spooldir
agent_kafka.sources.r1.spoolDir=/home/dev/flume1.6/irs-flume/test/

# 指定Flume sink
agent_kafka.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
agent_kafka.sinks.k1.topic = flume-topic
agent_kafka.sinks.k1.brokerList = dev1:9092,dev2:9093,dev3:9094
agent_kafka.sinks.k1.requiredAcks = 1
agent_kafka.sinks.k1.batchSize = 20

# 指定Flume channel
agent_kafka.channels.c1.type=memory
agent_kafka.channels.c1.capacity=1000
agent_kafka.channels.c1.transactionCapacity=100

# 指定拦截器,注意多个拦截器的使用 i1 i2
# i1: 自定义拦截器,对信息进行处理
# i2: 随机发布消息分区
agent_kafka.sources.r1.interceptors=i1 i2
agent_kafka.sources.r1.interceptors.i1.type=org.apache.flume.interceptor.MyInterceptor$Builder

# 随机发布到kafka分区
agent_kafka.sources.r1.interceptors.i2.type=org.apache.flume.sink.solr.morphline.UUIDInterceptor$Builder
agent_kafka.sources.r1.interceptors.i2.headerName=key
agent_kafka.sources.r1.interceptors.i2.preserveExisting=false

# 绑定source到sink
agent_kafka.sources.r1.channels =c1
agent_kafka.sinks.k1.channel=c1
```

---

## 2. 测试

### 2.1 Java 消费端

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>kafka-mq</groupId>
	<artifactId>kafka-mq</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<dependencies>
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
	</dependencies>
</project>
```

```java
package com.mq;

/**
 *
 * 常用配置文件
 * <p>
 *
 * @author huanghuapeng 2017年11月8日
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
	public static final String TOPIC0 = "java-demo";

	public static final String FLUME_KAFKA = "flume-topic";

	public static final String MANY_THREADS_TOPIC = "thread-demo4";

	/**
	 * kafka服务器地址
	 */
	public static final String KAFKA_SERVER = "10.10.2.70:9092";

}
```

```java
package com.mq.flume;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import com.mq.Conf;

import kafka.consumer.Consumer;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
import kafka.message.MessageAndMetadata;

public class KafkaConsumer implements Runnable {

	private final int threadNum = 10;
	private String topic;
	private ConsumerConnector consumer;

	public KafkaConsumer(String topic) {
		this.topic = topic;
		consumer = Consumer.createJavaConsumerConnector(buildConsumerConf());
	}

	private static ConsumerConfig buildConsumerConf() {
		Properties props = new Properties();

		props.put("zookeeper.connect", Conf.ZOOKEEPER_HOST);
		props.put("group.id", Conf.GROUP_ID);
		props.put("zookeeper.session.timeout.ms", "20000");
		props.put("zookeeper.sync.time.ms", "200");
		props.put("auto.commit.interval.ms", "500");
		props.put("rebalance.backoff.ms", "25000");
		props.put("rebalance.max.retries", "10");

		/**
		 * groupId不一致时,可能会导致同一条信息被多个consumer消费
		 */
		props.put("group.id", Conf.GROUP_ID); // group组的名字 （做group组区分）

		return new ConsumerConfig(props);

	}

	public void run() {

		Map<String, Integer> topicCountMap = new HashMap<String, Integer>();;
		topicCountMap.put(topic, threadNum);
		Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer
				.createMessageStreams(topicCountMap);

		List<KafkaStream<byte[], byte[]>> streamList = consumerMap.get(topic);
		ExecutorService executor = Executors.newFixedThreadPool(threadNum);

		for (KafkaStream<byte[], byte[]> stream : streamList) {
			executor.submit(new KafkaConsumerThread(stream));
		}

	}

	class KafkaConsumerThread implements Runnable {
		private KafkaStream<byte[], byte[]> stream;
		public KafkaConsumerThread(KafkaStream<byte[], byte[]> stream) {
			this.stream = stream;
		}

		public void run() {
			ConsumerIterator<byte[], byte[]> it = stream.iterator();
			while (it.hasNext()) {
				MessageAndMetadata<byte[], byte[]> mam = it.next();
				System.out.println(Thread.currentThread().getName()
						+ ": partition[" + mam.partition() + "]," + "offset["
						+ mam.offset() + "], " + new String(mam.message()));

			}
		}

	}

	public static void main(String[] args) {
		Thread consumer = new Thread(
				new KafkaConsumer(Conf.MANY_THREADS_TOPIC));

		consumer.start();
	}

}
```

```java
public class TestKafka {

	public static void main(String[] args) {
		Thread consumer = new Thread(new KafkaConsumer(Conf.FLUME_KAFKA));

		consumer.start();
	}

}
```

### 2.2 测试

> 先启动 KafkaConsumer.java

> 编辑服务器文件,放入 spoolDir 文件夹

```console
[root@aya irs-flume]# cat h
allow1
allow2
allow3
allow4
allow5
allow6
allow7
allow8
allow9
allow10
allow11
allow12
allow13
allow14
allow15
[root@aya irs-flume]#  mv h test/
```

### 3.3 测试结果

结果说明: append 前缀是在 MyInterceptor 上追加的字段,可以看出在自定义的拦截器里面实现自己的业务逻辑.

MyInterceptor 参考:[MyInterceptor.java 参考代码](https://github.com/cs12110/docs/blob/master/doc/flume/flume-interceptor.md)
MyInterceptor 不同之处代码为:

```java
@Override
public Event intercept(Event event) {
	logger.info("Process by MyInterceptor");
	try {
		String body = new String(event.getBody(), "UTF-8");
		if (body.startsWith("allow")) {
			event.setBody(("append " + body).getBytes());
			return event;
		}
	} catch (Exception e) {
		e.printStackTrace();
	}
	return null;
}
```

```java
pool-2-thread-9: partition[5],offset[31], append allow11
pool-2-thread-9: partition[5],offset[32], append allow15
pool-2-thread-2: partition[2],offset[3], append allow1
pool-2-thread-1: partition[7],offset[48], append allow4
pool-2-thread-1: partition[7],offset[49], append allow9
pool-2-thread-4: partition[4],offset[48], append allow3
pool-2-thread-4: partition[4],offset[49], append allow6
pool-2-thread-4: partition[4],offset[50], append allow7
pool-2-thread-4: partition[4],offset[51], append allow13
pool-2-thread-4: partition[4],offset[52], append allow14
pool-2-thread-5: partition[1],offset[0], append allow8
pool-2-thread-6: partition[3],offset[1], append allow2
pool-2-thread-6: partition[3],offset[2], append allow5
pool-2-thread-8: partition[9],offset[16], append allow10
pool-2-thread-8: partition[9],offset[17], append allow12
```

---

## 4. 参考资料

1. [flume 拦截器资料](https://github.com/cs12110/docs/blob/master/doc/flume/flume-interceptor.md)
2. [kafka 资料](https://github.com/cs12110/docs/blob/master/doc/kafka/kafka-doc.md)
