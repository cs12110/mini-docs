# RabbitMq

在现实分布式环境里面,不同的模块之间的数据共享和通知可以使用 mq,那么 rabbitmq 是一个不错的选择.

下面我们来熟悉一下 rabbitmq 的各个模式的使用.

<!-- TOC -->

- [RabbitMq](#rabbitmq)
  - [1. 共用代码](#1-共用代码)
    - [1.1 pom.xml](#11-pomxml)
    - [1.2 RabbitUtil](#12-rabbitutil)
    - [1.3 SysUtil](#13-sysutil)
    - [1.4 关于 Connection 与 Channel 的关闭问题](#14-关于-connection-与-channel-的关闭问题)
  - [2. 简单队列](#2-简单队列)
    - [2.1 生产者](#21-生产者)
    - [2.2 消费者](#22-消费者)
    - [2.3 测试类](#23-测试类)
  - [3. worker 模式](#3-worker-模式)
    - [3.1 生产者](#31-生产者)
    - [3.2 消费者](#32-消费者)
    - [3.3 测试类](#33-测试类)
  - [4. 发布订阅模式](#4-发布订阅模式)
    - [4.1 生产者](#41-生产者)
    - [4.2 消费者](#42-消费者)
    - [4.3 测试类](#43-测试类)
  - [5. 路由模式](#5-路由模式)
    - [5.1 生产者](#51-生产者)
    - [5.2 消费者](#52-消费者)
    - [5.3 测试类](#53-测试类)
  - [6. Topic](#6-topic)
    - [6.1 生产者](#61-生产者)
    - [6.2 消费者](#62-消费者)
    - [6.3 测试类](#63-测试类)
  - [7. 消息持久化](#7-消息持久化)
  - [8. 异常](#8-异常)
  - [9. 参考资料](#9-参考资料)

<!-- /TOC -->

---

## 1. 共用代码

### 1.1 pom.xml

```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>4.1.0</version>
</dependency>

<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <scope>test</scope>
    <version>4.12</version>
</dependency>
```

### 1.2 RabbitUtil

```java
package com.pkgs.util;

import java.util.Optional;

import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**
 *
 * Rabbit util
 *
 * <p>
 *
 * @author huanghuapeng 2017年6月20日
 * @see
 * @since 1.0
 */
public class RabbitUtil {

	private static ConnectionFactory factory = new ConnectionFactory();
	static {
		factory.setHost(SysUtil.RabbitConfig.RABBIT_HOST);
		factory.setPort(SysUtil.RabbitConfig.RABBIT_PORT);
		factory.setVirtualHost(SysUtil.RabbitConfig.RABBIT_VHOST);
		factory.setUsername(SysUtil.RabbitConfig.RABBIT_USER);
		factory.setPassword(SysUtil.RabbitConfig.RABBIT_PASSWORD);
	}

	/**
	 * 获取Rabbit连接
	 *
	 * @return Connection
	 */
	public static Connection getDefRabbitConnection() {
		Connection rabbitConn = null;
		try {
			rabbitConn = factory.newConnection();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return rabbitConn;
	}
}
```

### 1.3 SysUtil

```java
package com.pkgs.util;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.function.Supplier;

/**
 * 系统工具类
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午8:38:18
 * @see
 * @since 1.0
 */
public class SysUtil {

	/**
	 * Rabbit配置
	 */
	public static class RabbitConfig {
		public final static String RABBIT_HOST = "10.33.1.115";
		public final static Integer RABBIT_PORT = 5672;
		public final static String RABBIT_VHOST = "/";
		public final static String RABBIT_USER = "root";
		public final static String RABBIT_PASSWORD = "root";

		public final static String SIMPLE_QUEUE = "simpleQueue";
		public final static String WORKER_QUEUE = "workQueue";
		public final static String FANOUT_QUEUE = "fanoutQueue";
		public final static String DIRECT_QUEUE = "directQueue";

		public final static String FANOUT_EXCHANGE = "fanoutExchange";
		public final static String ROUTING_EXCHANGE = "routingExchange";
	}

	/**
	 * 打印日志
	 *
	 * @param log
	 *            日志
	 */
	public static void log(String log) {
		StringBuilder body = new StringBuilder();
		body.append(getTime());
		body.append(" ");
		body.append(getStackTrackInfo());
		body.append(" - ");
		body.append(log);

		System.out.println(body);
	}

	/**
	 * 获取堆栈信息
	 *
	 * @return String
	 */
	private static String getStackTrackInfo() {
		Exception trace = new Exception();
		StackTraceElement[] stackTraceArr = trace.getStackTrace();
		String selfName = SysUtil.class.getName();
		for (StackTraceElement e : stackTraceArr) {
			if (!e.getClassName().equals(selfName)) {
				return simplify(e.getClassName()) + ":" + e.getLineNumber();
			}
		}
		return "N/A";
	}

	/**
	 * 简化类名
	 *
	 * @param clazzName
	 *            类名称
	 * @return String
	 */
	private static String simplify(String clazzName) {
		int index = clazzName.lastIndexOf(".");
		if (-1 == index) {
			return clazzName;
		}
		StringBuilder name = new StringBuilder();
		int left = 0;
		while (true) {
			int next = clazzName.indexOf(".", left);
			if (-1 == next) {
				break;
			}
			name.append(clazzName.charAt(left));
			name.append(".");
			left = next + 1;
		}
		name.append(clazzName.substring(left));
		return name.toString();
	}

	/**
	 * 获取当前时间
	 *
	 * @return
	 */
	private static String getTime() {
		Supplier<String> supplier = () -> {
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			return sdf.format(new Date());
		};
		return supplier.get();
	}

	/**
	 * 休眠
	 *
	 * @param millSeconds
	 */
	public static void sleep(long millSeconds) {
		try {
			Thread.sleep(millSeconds);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

### 1.4 关于 Connection 与 Channel 的关闭问题

> Next we create a channel, which is where most of the API for getting things done resides. Note we can use a try-with-resources statement because both Connection and Channel implement java.io.Closeable. This way we don't need to close them explicitly in our code. [link](http://next.rabbitmq.com/tutorials/tutorial-one-java.html)

---

## 2. 简单队列

在最简单的消息队列里面,一个生产者/一个消费者. [官网 link](http://next.rabbitmq.com/tutorials/tutorial-one-java.html)

这里面设置了 Qos 和手动去确认消息被消费,而不是自动,请知悉.

### 2.1 生产者

```java
package com.pkgs.simple.provider;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:44:49
 * @see
 * @since 1.0
 */
public class MsgProducer {

	/**
	 * 消息队列名称
	 */
	private static final String QUEUE_NAME = SysUtil.RabbitConfig.SIMPLE_QUEUE;

	/**
	 * 标识
	 */
	private String name;

	public MsgProducer(String name) {
		this.name = name;
	}

	/**
	 * 发送消息
	 *
	 * @param times
	 *            条数
	 * @throws Exception
	 */
	public void sendMsgToQueue(int times) throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		Channel channel = connection.createChannel();
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);
		for (int i = 0; i < times; i++) {
			String message = "NO." + i;
			channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));

			SysUtil.log(name + "->" + message);
		}
	}
}
```

### 2.2 消费者

```java
package com.pkgs.simple.consumer;
import java.io.IOException;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:45:17
 * @see
 * @since 1.0
 */
public class MyConsumer {

	/**
	 * 消息队列名称
	 */
	private static String QUEUE_NAME = SysUtil.RabbitConfig.SIMPLE_QUEUE;

	/**
	 * 名称
	 */
	private String name;

	/**
	 * 休眠时间
	 */
	private int sleepTime;

	public MyConsumer(String name, int sleepTime) {
		this.name = name;
		this.sleepTime = sleepTime;
	}

	/**
	 * 处理消息
	 *
	 * @throws Exception
	 */
	public void takeMsg() throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		final Channel channel = connection.createChannel();

		/**
		 * 设置Qos: 消费能力强的消费更多消息,能者多劳.
		 *
		 * 不设置Qos,默认为多个消费者平均消费消息.
		 */
		channel.basicQos(1);
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);

		// 处理消息
		Consumer consumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {

				// 获取消息
				String message = new String(body, "UTF-8");
				SysUtil.log(name + "(" + QUEUE_NAME + "):" + message);
				SysUtil.sleep(sleepTime);

				// 设置消费成功
				channel.basicAck(envelope.getDeliveryTag(), false);
			}
		};

		// 设置autoAck为false,手动确认消息消费
		channel.basicConsume(QUEUE_NAME, false, consumer);
	}

}
```

### 2.3 测试类

```java
package com.test;

import com.pkgs.simple.consumer.MyConsumer;
import com.pkgs.simple.provider.MsgProducer;

public class SimpleQueue {
	public static void main(String[] args) throws Exception {
		MsgProducer sender = new MsgProducer("Boss");
		sender.sendMsgToQueue(3);

		MyConsumer recv1 = new MyConsumer("Consumer1", 200);
		recv1.takeMsg();
	}
}
```

测试结果

```java
2018-12-21 10:46:12 c.p.s.p.MsgProducer:50 - Boss->NO.0
2018-12-21 10:46:12 c.p.s.p.MsgProducer:50 - Boss->NO.1
2018-12-21 10:46:12 c.p.s.p.MsgProducer:50 - Boss->NO.2
2018-12-21 10:46:13 c.p.s.c.MyConsumer$1:60 - Consumer1(simpleQueue):NO.0
2018-12-21 10:46:13 c.p.s.c.MyConsumer$1:60 - Consumer1(simpleQueue):NO.1
2018-12-21 10:46:13 c.p.s.c.MyConsumer$1:60 - Consumer1(simpleQueue):NO.2
```

---

## 3. worker 模式

生产者生产的消息发至消息队列,然后多个消费者来消费.[官网 link](http://next.rabbitmq.com/tutorials/tutorial-two-java.html)

和 fanout 的区别是: **队列的消息只能给其中一个消费者消费,而不是全部消费者.**

### 3.1 生产者

```java
package com.pkgs.worker.provider;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:44:49
 * @see
 * @since 1.0
 */
public class WorkerProducer {

	/**
	 * 消息队列名称
	 */
	private static final String QUEUE_NAME = SysUtil.RabbitConfig.WORKER_QUEUE;

	/**
	 * 标识消息名称
	 */
	private String name;

	public WorkerProducer(String name) {
		this.name = name;
	}

	/**
	 * 发送消息
	 *
	 * @param times
	 *            条数
	 * @throws Exception
	 */
	public void sendMsgToQueue(int times) throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		Channel channel = connection.createChannel();
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);
		for (int i = 0; i < times; i++) {
			String message = "NO." + i;
			channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));

			SysUtil.log(name + "->" + message);
		}
	}
}
```

### 3.2 消费者

```java
package com.pkgs.worker.consumer;
import java.io.IOException;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:45:17
 * @see
 * @since 1.0
 */
public class WorkerConsumer {

	/**
	 * 消息队列名称
	 */
	private static String QUEUE_NAME = SysUtil.RabbitConfig.WORKER_QUEUE;

	/**
	 * 名称
	 */
	private String name;

	/**
	 * 休眠时间
	 */
	private int sleepTime;

	public WorkerConsumer(String name, int sleepTime) {
		this.name = name;
		this.sleepTime = sleepTime;
	}

	/**
	 * 处理消息
	 *
	 * @throws Exception
	 */
	public void takeMsg() throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		final Channel channel = connection.createChannel();

		/**
		 * 设置Qos: 消费能力强的消费更多消息,能者多劳.
		 *
		 * 不设置Qos,默认为多个消费者平均消费消息.
		 */
		channel.basicQos(1);
		channel.queueDeclare(QUEUE_NAME, false, false, false, null);

		// 处理消息
		Consumer consumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {

				// 获取消息
				String message = new String(body, "UTF-8");
				SysUtil.log(name + "(" + QUEUE_NAME + "):" + message);
				SysUtil.sleep(sleepTime);

				// 设置消费成功
				channel.basicAck(envelope.getDeliveryTag(), false);
			}
		};

		// 设置autoAck为false,手动确认消息消费
		channel.basicConsume(QUEUE_NAME, false, consumer);
	}

}
```

### 3.3 测试类

```java
package com.test;

import com.pkgs.worker.consumer.WorkerConsumer;
import com.pkgs.worker.provider.WorkerProducer;

public class WorkerQueue {
	public static void main(String[] args) throws Exception {
		WorkerProducer sender = new WorkerProducer("worker-provider");
		sender.sendMsgToQueue(7);

		WorkerConsumer consumer1 = new WorkerConsumer("worker-consumer1", 1000);
		consumer1.takeMsg();

		WorkerConsumer consumer2 = new WorkerConsumer("worker-consumer2", 2000);
		consumer2.takeMsg();
	}
}
```

测试结果

```java
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.0
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.1
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.2
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.3
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.4
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.5
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.6
2018-12-21 11:01:42 c.p.w.p.WorkerProducer:50 - worker-provider->NO.7
2018-12-21 11:01:42 c.p.w.c.WorkerConsumer$1:71 - worker-consumer1(workQueue):NO.0
2018-12-21 11:01:42 c.p.w.c.WorkerConsumer$1:71 - worker-consumer2(workQueue):NO.1
2018-12-21 11:01:43 c.p.w.c.WorkerConsumer$1:71 - worker-consumer1(workQueue):NO.2
2018-12-21 11:01:44 c.p.w.c.WorkerConsumer$1:71 - worker-consumer1(workQueue):NO.3
2018-12-21 11:01:44 c.p.w.c.WorkerConsumer$1:71 - worker-consumer2(workQueue):NO.4
2018-12-21 11:01:45 c.p.w.c.WorkerConsumer$1:71 - worker-consumer1(workQueue):NO.5
2018-12-21 11:01:46 c.p.w.c.WorkerConsumer$1:71 - worker-consumer1(workQueue):NO.6
2018-12-21 11:01:46 c.p.w.c.WorkerConsumer$1:71 - worker-consumer2(workQueue):NO.7
```

结论: **设置不同的消费耗时,消费者 1 消费能力比消费者 2 的强,所以消费者 1 比消费者 2 消费更多的消息.**

---

## 4. 发布订阅模式

发布订阅模式相当于 q 群里面的信息群发,每条消息都会给所有消费者消费. [官网 link](http://next.rabbitmq.com/tutorials/tutorial-three-java.html)

注意: **订阅者要比发布者先启动**.

### 4.1 生产者

```java
package com.pkgs.pubsub.provider;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:44:49
 * @see
 * @since 1.0
 */
public class PubSubProducer {

	/**
	 * 交换机名称
	 */
	private static String FANOUT_EXCHANGE = SysUtil.RabbitConfig.FANOUT_EXCHANGE;

	/**
	 * 标识消息名称
	 */
	private String name;

	public PubSubProducer(String name) {
		this.name = name;
	}

	/**
	 * 发送消息
	 *
	 * @param times
	 *            条数
	 * @throws Exception
	 */
	public void sendMsgToQueue(int times) throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		Channel channel = connection.createChannel();
		channel.exchangeDeclare(FANOUT_EXCHANGE, BuiltinExchangeType.FANOUT.getType());
		for (int i = 0; i < times; i++) {
			String message = "NO." + i;
			// queue设置为空
			channel.basicPublish(FANOUT_EXCHANGE, "", null, message.getBytes("UTF-8"));

			SysUtil.log(name + "->" + message);
		}
	}
}
```

### 4.2 消费者

```java
package com.pkgs.pubsub.consumer;
import java.io.IOException;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:45:17
 * @see
 * @since 1.0
 */
public class PubSubConsumer {

	/**
	 * 交换机名称
	 */
	private static String FANOUT_EXCHANGE = SysUtil.RabbitConfig.FANOUT_EXCHANGE;

	/**
	 * 名称
	 */
	private String queueName;

	/**
	 * 休眠时间
	 */
	private int sleepTime;

	public PubSubConsumer(String queueName, int sleepTime) {
		this.queueName = queueName;
		this.sleepTime = sleepTime;
	}

	/**
	 * 处理消息
	 *
	 * @throws Exception
	 */
	public void takeMsg() throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		final Channel channel = connection.createChannel();

		// 声明exchange和queue
		channel.exchangeDeclare(FANOUT_EXCHANGE, BuiltinExchangeType.FANOUT.getType());
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, FANOUT_EXCHANGE, "");

		// 处理消息
		Consumer consumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {

				// 获取消息
				String message = new String(body, "UTF-8");
				SysUtil.log(queueName + "(" + FANOUT_EXCHANGE + "):" + message);
				SysUtil.sleep(sleepTime);

				// 设置消费成功
				channel.basicAck(envelope.getDeliveryTag(), false);
			}
		};

		// 设置autoAck为false,手动确认消息消费
		channel.basicConsume(queueName, false, consumer);
	}

}
```

### 4.3 测试类

```java
package com.test;

import com.pkgs.pubsub.consumer.PubSubConsumer;
import com.pkgs.pubsub.provider.PubSubProducer;

public class PubSubQueue {
	public static void main(String[] args) throws Exception {

		// 订阅者必须比发布者先启动,或者订阅不到消息.
		PubSubConsumer consumer1 = new PubSubConsumer("pubsub-consumer1", 0);
		consumer1.takeMsg();

		PubSubConsumer consumer2 = new PubSubConsumer("pubsub-consumer2", 0);
		consumer2.takeMsg();

		PubSubProducer sender = new PubSubProducer("pubsub-provider");
		sender.sendMsgToQueue(2);
	}
}
```

测试结果

```java
2018-12-21 11:44:08 c.p.p.p.PubSubProducer:51 - pubsub-provider->NO.0
2018-12-21 11:44:08 c.p.p.p.PubSubProducer:51 - pubsub-provider->NO.1
2018-12-21 11:44:08 c.p.p.c.PubSubConsumer$1:69 - pubsub-consumer2(fanoutExchange):NO.0
2018-12-21 11:44:08 c.p.p.c.PubSubConsumer$1:69 - pubsub-consumer1(fanoutExchange):NO.0
2018-12-21 11:44:08 c.p.p.c.PubSubConsumer$1:69 - pubsub-consumer1(fanoutExchange):NO.1
2018-12-21 11:44:08 c.p.p.c.PubSubConsumer$1:69 - pubsub-consumer2(fanoutExchange):NO.1
```

---

## 5. 路由模式

如果只想同一个交换机下面的某几个队列收到消息该怎么办呢?

路由模式,可以做到这个功能. [官网 link](http://next.rabbitmq.com/tutorials/tutorial-four-java.html)

### 5.1 生产者

```java
package com.pkgs.routing.provider;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:44:49
 * @see
 * @since 1.0
 */
public class RoutingProducer {

	/**
	 * 交换机名称
	 */
	private static String EXCHANGE_NAME = SysUtil.RabbitConfig.ROUTING_EXCHANGE;

	// 名称
	private String name;

	public RoutingProducer(String name) {
		this.name = name;
	}

	/**
	 * 发送消息
	 *
	 * @param routeKey
	 *            路由key
	 * @param times
	 *            条数
	 * @throws Exception
	 */
	public void sendMsgToQueue(String routeKey, int times) throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		Channel channel = connection.createChannel();

		// 修改为DIRECT
		channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT.getType());
		for (int i = 0; i < times; i++) {
			String message = "NO." + i;
			// queue设置为空
			channel.basicPublish(EXCHANGE_NAME, routeKey, null, message.getBytes("UTF-8"));

			SysUtil.log(name + "->[" + routeKey + "]" + message);
		}
	}
}
```

### 5.2 消费者

```java
package com.pkgs.routing.consumer;
import java.io.IOException;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

/**
 * 消费者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:45:17
 * @see
 * @since 1.0
 */
public class RoutingConsumer {

	/**
	 * 交换机名称
	 */
	private static String EXCHANGE_NAME = SysUtil.RabbitConfig.ROUTING_EXCHANGE;

	/**
	 * 名称
	 */
	private String queueName;

	/**
	 * 路由key
	 */
	private String routeKey;

	/**
	 * 休眠时间
	 */
	private int sleepTime;

	public RoutingConsumer(String queueName, String routeKey, int sleepTime) {
		this.queueName = queueName;
		this.routeKey = routeKey;
		this.sleepTime = sleepTime;
	}

	/**
	 * 处理消息
	 *
	 * @throws Exception
	 */
	public void takeMsg() throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		final Channel channel = connection.createChannel();

		// 声明exchange和queue
		channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT.getType());
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, EXCHANGE_NAME, routeKey);

		// 处理消息
		Consumer consumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {

				// 获取消息
				String message = new String(body, "UTF-8");
				SysUtil.log(queueName + "(" + EXCHANGE_NAME + "):" + message);
				SysUtil.sleep(sleepTime);

				// 设置消费成功
				channel.basicAck(envelope.getDeliveryTag(), false);
			}
		};

		// 设置autoAck为false,手动确认消息消费
		channel.basicConsume(queueName, false, consumer);
	}
}
```

### 5.3 测试类

```java
package com.test;

import java.util.HashMap;
import java.util.Map;

import com.pkgs.routing.consumer.RoutingConsumer;
import com.pkgs.routing.provider.RoutingProducer;

public class RoutingQueue {
	public static void main(String[] args) throws Exception {

		Map<String, Integer> map = new HashMap<String, Integer>(2);
		map.put("color.black", 2);
		map.put("color.orange", 3);

		RoutingProducer sender = new RoutingProducer("routing-provider1");

		map.forEach((routeKey, num) -> {
			try {
				sender.sendMsgToQueue(routeKey, num);
			} catch (Exception e) {
				e.printStackTrace();
			}
		});

		RoutingConsumer consumer1 = new RoutingConsumer("routing-consumer1", "color.black", 0);
		consumer1.takeMsg();

		RoutingConsumer consumer2 = new RoutingConsumer("routing-consumer2", "color.orange", 0);
		consumer2.takeMsg();
	}
}
```

测试结果

```java
2018-12-21 13:01:04 c.p.r.p.RoutingProducer:54 - routing-provider1->[color.orange]NO.0
2018-12-21 13:01:04 c.p.r.p.RoutingProducer:54 - routing-provider1->[color.orange]NO.1
2018-12-21 13:01:04 c.p.r.p.RoutingProducer:54 - routing-provider1->[color.orange]NO.2
2018-12-21 13:01:05 c.p.r.p.RoutingProducer:54 - routing-provider1->[color.black]NO.0
2018-12-21 13:01:05 c.p.r.p.RoutingProducer:54 - routing-provider1->[color.black]NO.1
2018-12-21 13:01:05 c.p.r.c.RoutingConsumer$1:75 - routing-consumer1(routingExchange:color.black):NO.0
2018-12-21 13:01:05 c.p.r.c.RoutingConsumer$1:75 - routing-consumer1(routingExchange:color.black):NO.1
2018-12-21 13:01:05 c.p.r.c.RoutingConsumer$1:75 - routing-consumer2(routingExchange:color.orange):NO.0
2018-12-21 13:01:05 c.p.r.c.RoutingConsumer$1:75 - routing-consumer2(routingExchange:color.orange):NO.1
2018-12-21 13:01:05 c.p.r.c.RoutingConsumer$1:75 - routing-consumer2(routingExchange:color.orange):NO.2
```

---

## 6. Topic

在交换机里面实现更灵活的队列绑定消费相关消息.

### 6.1 生产者

```java
package com.pkgs.topic.provider;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产者
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月21日上午10:44:49
 * @see
 * @since 1.0
 */
public class TopicProducer {

	/**
	 * 交换机名称
	 */
	private static String EXCHANGE_NAME = SysUtil.RabbitConfig.TOPIC_EXCHANGE;

	// 名称
	private String name;

	public TopicProducer(String name) {
		this.name = name;
	}

	/**
	 * 发送消息
	 *
	 * @param routeKey
	 *            路由key
	 * @param times
	 *            条数
	 * @throws Exception
	 */
	public void sendMsgToQueue(String routeKey, int times) throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		Channel channel = connection.createChannel();

		// 修改为DIRECT
		channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC.getType());
		for (int i = 0; i < times; i++) {
			String message = "NO." + i;
			// queue设置为空
			channel.basicPublish(EXCHANGE_NAME, routeKey, null, message.getBytes("UTF-8"));

			SysUtil.log(name + "->[" + routeKey + "]" + message);
		}
	}
}
```

### 6.2 消费者

```java
package com.pkgs.topic.consumer;

import java.io.IOException;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class TopicConsumer {

	/**
	 * 交换机名称
	 */
	private static String EXCHANGE_NAME = SysUtil.RabbitConfig.TOPIC_EXCHANGE;

	/**
	 * 名称
	 */
	private String queueName;

	/**
	 * 路由key
	 */
	private String routeKey;

	/**
	 * 休眠时间
	 */
	private int sleepTime;

	public TopicConsumer(String queueName, String routeKey, int sleepTime) {
		this.queueName = queueName;
		this.routeKey = routeKey;
		this.sleepTime = sleepTime;
	}

	/**
	 * 处理消息
	 *
	 * @throws Exception
	 */
	public void takeMsg() throws Exception {
		Connection connection = RabbitUtil.getDefRabbitConnection();
		final Channel channel = connection.createChannel();

		// 设置topic模式
		channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.TOPIC.getType());
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, EXCHANGE_NAME, routeKey);

		// 处理消息
		Consumer consumer = new DefaultConsumer(channel) {
			@Override
			public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
					byte[] body) throws IOException {

				// 获取消息
				String message = new String(body, "UTF-8");
				SysUtil.log(queueName + "(" + EXCHANGE_NAME + ":" + routeKey + "):" + message);
				SysUtil.sleep(sleepTime);

				// 设置消费成功
				channel.basicAck(envelope.getDeliveryTag(), false);
			}
		};

		// 设置autoAck为false,手动确认消息消费
		channel.basicConsume(queueName, false, consumer);
	}
}
```

### 6.3 测试类

```java
package com.test;

import java.util.HashMap;
import java.util.Map;

import com.pkgs.topic.consumer.TopicConsumer;
import com.pkgs.topic.provider.TopicProducer;

public class TopicQueue {
	public static void main(String[] args) throws Exception {

		Map<String, Integer> map = new HashMap<String, Integer>(2);
		map.put("color.black", 1);
		map.put("color.orange", 1);
		map.put("color.pink", 1);
		map.put("other.shape", 1);

		TopicProducer sender = new TopicProducer("topic-provider1");

		map.forEach((routeKey, num) -> {
			try {
				sender.sendMsgToQueue(routeKey, num);
			} catch (Exception e) {
				e.printStackTrace();
			}
		});

		// 只获取color.black
		TopicConsumer consumer1 = new TopicConsumer("topic-consumer1", "color.black", 0);
		consumer1.takeMsg();

		// 获取color.black,color.orange,color.pink的消息
		TopicConsumer consumer2 = new TopicConsumer("topic-consumer2", "color.*", 0);
		consumer2.takeMsg();

		// 获取全部
		TopicConsumer consumer3 = new TopicConsumer("topic-consumer3", "#", 0);
		consumer3.takeMsg();
	}
}
```

测试结果

```java
2018-12-21 15:56:44 c.p.t.p.TopicProducer:54 - topic-provider1->[color.black]NO.0
2018-12-21 15:56:44 c.p.t.p.TopicProducer:54 - topic-provider1->[color.pink]NO.0
2018-12-21 15:56:44 c.p.t.p.TopicProducer:54 - topic-provider1->[other.shape]NO.0
2018-12-21 15:56:44 c.p.t.p.TopicProducer:54 - topic-provider1->[color.orange]NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer1(topicExchange:color.black):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer2(topicExchange:color.*):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer2(topicExchange:color.*):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer2(topicExchange:color.*):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer3(topicExchange:#):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer3(topicExchange:#):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer3(topicExchange:#):NO.0
2018-12-21 15:56:44 c.p.t.c.TopicConsumer$1:65 - topic-consumer3(topicExchange:#):NO.0
```

---

## 7. 消息持久化

如果消息不要进行就持久化的话,在 rabbitmq 服务器 down 之后,消息会丢失.

所以对重要的消息进行持久化是很必要的一件事.

Q: 那么该怎么进行数据持久化呢?

A: 在生产者和消费者声明队列的时候,把`durable`参数设置为 true.

```java
/*
 * 第二个参数设置持久化,设置durable为true,同样在消费端queue的声明也必须为true
 *
 * text格式持久化?
 */
ch.queueDeclare(durableQueueName, true, false, false, null);
```

---

## 8. 异常

重现操作: 只要关闭然后重启快的话,就会出现如下异常.

测试代码

```java
package com.test;

import com.pkgs.util.RabbitUtil;
import com.pkgs.util.SysUtil;
import com.rabbitmq.client.Connection;

public class QuickGetConnectionTest {

	public static void main(String[] args) {
		for (int index = 0; index < 100; index++) {
			Connection conn = RabbitUtil.getDefRabbitConnection();
			if (conn == null) {
				SysUtil.log("Error on get connection: " + index);
			}
			try {
				conn.close();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		SysUtil.log("All is done");
	}
}
```

出现异常如下:

```java
java.util.concurrent.TimeoutException
	at com.rabbitmq.utility.BlockingCell.get(BlockingCell.java:77)
	at com.rabbitmq.utility.BlockingCell.uninterruptibleGet(BlockingCell.java:120)
	at com.rabbitmq.utility.BlockingValueOrException.uninterruptibleGetValue(BlockingValueOrException.java:36)
	at com.rabbitmq.client.impl.AMQChannel$BlockingRpcContinuation.getReply(AMQChannel.java:398)
	at com.rabbitmq.client.impl.AMQConnection.start(AMQConnection.java:304)
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:920)
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:870)
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:828)
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:966)
	at com.pkgs.util.RabbitUtil.getDefRabbitConnection(RabbitUtil.java:40)
	at com.pkgs.simple.consumer.MyConsumer.takeMsg(MyConsumer.java:52)
	at com.test.SimpleQueue.main(SimpleQueue.java:12)
Exception in thread "main" java.lang.NullPointerException
	at com.pkgs.simple.consumer.MyConsumer.takeMsg(MyConsumer.java:53)
	at com.test.SimpleQueue.main(SimpleQueue.java:12)
2018-12-21 17:06:46,024 ERROR ForgivingExceptionHandler.java:124 - An unexpected connection driver error occured
java.net.SocketException: Socket Closed
	at java.net.SocketInputStream.socketRead0(Native Method)
	at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
	at java.net.SocketInputStream.read(SocketInputStream.java:170)
	at java.net.SocketInputStream.read(SocketInputStream.java:141)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:265)
	at java.io.DataInputStream.readUnsignedByte(DataInputStream.java:288)
	at com.rabbitmq.client.impl.Frame.readFrom(Frame.java:91)
	at com.rabbitmq.client.impl.SocketFrameHandler.readFrame(SocketFrameHandler.java:164)
	at com.rabbitmq.client.impl.AMQConnection$MainLoop.run(AMQConnection.java:578)
	at java.lang.Thread.run(Thread.java:745)
```

现在还没有救,窝草.

即使是官网的推荐方法,在频繁获取 connection 的时候,都还是会出错的,惨不忍睹.

```java
ConnectionFactory factory = new ConnectionFactory();
// configure various connection settings

try {
  Connection conn = factory.newConnection();
} catch (java.net.ConnectException e) {
  Thread.sleep(5000);
  // apply retry logic
}
```

---

## 9. 参考资料

a. [RabbitMq 官网](http://next.rabbitmq.com/tutorials/tutorial-one-java.html)
