# Redis 发布/订阅

在有些生产场景里面,要实现简单的发布/订阅场景.

如果不想用 Mq(大材小用)的话,可以简单粗暴的使用 redis 来实现.

---

## 1. 代码实现

### 1.1 pom.xml

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

### 1.2 发布者

```java

import redis.clients.jedis.Jedis;

/**
 * 发布者
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class Publisher implements Runnable {

	public static final String CHANNEL_NAME = "pubsub-logs";
	private static LogUtil logger = new LogUtil(Publisher.class);

	@Override
	public void run() {
		// 发送3条消息,每条间隔1s
		publish(3, 1);
	}

	/**
	 * 发布消息
	 *
	 * @param msgNum
	 *            消息条数
	 * @param gap
	 *            sleep秒数
	 */
	private void publish(int msgNum, int gap) {
		Jedis jedis = new Jedis("10.10.1.142", 6379);
		try {
			for (int index = 0; index < msgNum; index++) {
				jedis.publish(CHANNEL_NAME, "msg:value" + index);
				logger.mark("send: " + index);
				Thread.sleep(gap * 1000);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			jedis.close();
		}
	}
}
```

### 1.3 订阅者

**监听器**

```java

import redis.clients.jedis.JedisPubSub;

/**
 * 订阅者监听器
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class SubscribeHandler extends JedisPubSub {

	private LogUtil logger = new LogUtil(this.getClass());

	@Override
	public void onMessage(String channel, String message) {
		String log = "t" + getThreadId() + " [" + channel + "] get->" + message;
		logger.mark(log);

		// 取消订阅,只消费一条消息,就会取消订阅了.
		// this.unsubscribe();
	}

	@Override
	public void onSubscribe(String channel, int subscribedChannels) {
		String log = "t" + getThreadId() + " [" + channel + "] is been subscribed:" + subscribedChannels;
		logger.mark(log);
	}

	@Override
	public void onUnsubscribe(String channel, int subscribedChannels) {
		String log = "t" + getThreadId() + " [" + channel + "] is been unsubscribed:" + subscribedChannels;
		logger.mark(log);
	}

	private long getThreadId() {
		return Thread.currentThread().getId();
	}
}
```

**订阅者**

```java
import redis.clients.jedis.Jedis;

/**
 * 订阅者
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class Subscriber implements Runnable {

	@Override
	public void run() {
		subscribe();
	}

	private void subscribe() {
		Jedis jedis = new Jedis("10.10.1.142", 6379);
		try {
			SubscribeHandler listener = new SubscribeHandler();
			jedis.subscribe(listener, Publisher.CHANNEL_NAME);
		} finally {
			jedis.close();
		}
	}
}
```

### 1.4 日志工具类

```java

import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * 日志工具类
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class LogUtil {

	private Class<?> clazz;

	public LogUtil(Class<?> clazz) {
		this.clazz = clazz;
	}

	/**
	 * 打印日志
	 *
	 * @param log
	 *            日志内容
	 */
	public void mark(String log) {
		synchronized (LogUtil.class) {
			String className = simplify(clazz);
			System.out.println(getTime() + " - " + className + " " + log);
		}
	}

	private String simplify(Class<?> clazz) {
		if (null == clazz) {
			return "N/A";
		}
		String className = clazz.getName();
		StringBuilder simple = new StringBuilder(className.length());

		int index = 0;
		int next = 0;

		while (true) {
			next = className.indexOf(".", index);
			if (next == -1) {
				break;
			}
			simple.append(className.charAt(index));
			simple.append(".");
			index = next + 1;
		}
		simple.append(className.substring(index));
		return simple.toString();
	}

	private String getTime() {
		return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
	}
}
```

---

## 2. 测试代码

### 2.1 单个订阅者

```java
/**
 * 测试
 *
 *
 * <p>
 *
 * 要首先启动订阅者,然后才能启动发布者
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class PubSubApp {
	public static void main(String[] args) {
		new Thread(new Subscriber()).start();
		try {
			// 确保订阅者首先开启
			Thread.sleep(1000);
		} catch (Exception e) {
		}
		new Thread(new Publisher()).start();
	}
}
```

测试结果

```java
2018-12-04 09:59:35 - c.p.SubscribeHandler t13 [pubsub-logs] is been subscribed:1
2018-12-04 09:59:36 - c.p.Publisher send: 0
2018-12-04 09:59:36 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value0
2018-12-04 09:59:37 - c.p.Publisher send: 1
2018-12-04 09:59:37 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value1
2018-12-04 09:59:38 - c.p.Publisher send: 2
2018-12-04 09:59:38 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value2
```

### 2.2 多个订阅者

```java
/**
 * 测试
 *
 *
 * <p>
 *
 * 要首先启动订阅者,然后才能启动发布者
 *
 * @author cs12110 2018年12月4日
 * @see
 * @since 1.0
 */
public class PubSubApp {
	public static void main(String[] args) {
		new Thread(new Subscriber()).start();
		new Thread(new Subscriber()).start();
		try {
			// 确保订阅者首先开启
			Thread.sleep(1000);
		} catch (Exception e) {
		}
		new Thread(new Publisher()).start();
	}
}
```

测试结果

```java
2018-12-04 10:01:29 - c.p.SubscribeHandler t13 [pubsub-logs] is been subscribed:1
2018-12-04 10:01:29 - c.p.SubscribeHandler t14 [pubsub-logs] is been subscribed:1
2018-12-04 10:01:30 - c.p.Publisher send: 0
2018-12-04 10:01:30 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value0
2018-12-04 10:01:30 - c.p.SubscribeHandler t14 [pubsub-logs] get->msg:value0
2018-12-04 10:01:31 - c.p.Publisher send: 1
2018-12-04 10:01:31 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value1
2018-12-04 10:01:31 - c.p.SubscribeHandler t14 [pubsub-logs] get->msg:value1
2018-12-04 10:01:32 - c.p.Publisher send: 2
2018-12-04 10:01:32 - c.p.SubscribeHandler t14 [pubsub-logs] get->msg:value2
2018-12-04 10:01:32 - c.p.SubscribeHandler t13 [pubsub-logs] get->msg:value2
```

---

## 3. 总结

- 要首先开启订阅者,不然会出现消息丢失.
- 发布消息会给多个订阅者消费.
