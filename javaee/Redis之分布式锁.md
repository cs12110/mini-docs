# Redis 之分布式锁

分布式锁是一个要掌握的关键知识点,特别应用在分布式系统的资源访问控制上面.

---

## 1. Redis 锁

在网上实现简单的分布式锁,几乎都是使用 setnx+超时这个命令来完成.

但这个简单的方法存在天然的缺陷

- 方法执行耗时长于锁的过时时间,在方法还没执行完就自动释放锁.
- 如果设置过时时间过长,执行过程 down 的话,会给其他没 down 导致很长的等待时间.

### 1.1 pom.xml

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

### 1.2 代码

```java
import java.util.Collections;

import redis.clients.jedis.Jedis;

/**
 * lock
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月26日
 * @see
 * @since 1.0
 */
public class RedisLockUtil {

	/**
	 * 过时时间,1s
	 */
	private static long expiredTime = 1000L;

	private static final String SET_IF_NOT_EXISTS = "NX";
	private static final String EXPX = "PX";
	private static final String LOCKED = "OK";
	private static final String UNLOCKED = "1";

	/**
	 * try lock,如果获得锁返回true,否则返回false
	 *
	 * @return boolean
	 */
	public static boolean trylock(String lockKey, String lockValue) {
		Jedis jedis = getRedisConnection();
		String result = jedis.set(lockKey, lockValue, SET_IF_NOT_EXISTS, EXPX, expiredTime);
		jedis.close();
		return LOCKED.equals(result);
	}

	/**
	 * 加锁,这个会出现问题.
	 *
	 * <pre>
	 *
	 * 1. 方法执行耗时长于锁的过时时间,会自动释放锁.
	 *
	 * 2. 如果设置过时时间过长,执行过程down的话,会给其他没down导致很长的等待时间.
	 *
	 * </pre>
	 *
	 *
	 * @return
	 */
	public static boolean lock(String lockKey, String lockValue) {
		Jedis jedis = getRedisConnection();
		while (true) {
			String result = jedis.set(lockKey, lockValue, SET_IF_NOT_EXISTS, EXPX, expiredTime);
			if (LOCKED.equals(result)) {
				break;
			}
			try {
				Thread.sleep(10);
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		jedis.close();
		return true;
	}

	/**
	 * 释放锁
	 *
	 * @return boolean
	 */
	public static boolean unlock(String lockKey, String lockValue) {
		Jedis jedis = getRedisConnection();
		Object result = jedis.eval(releaseLockSscript(), Collections.singletonList(lockKey),
				Collections.singletonList(lockValue));
		jedis.close();
		return UNLOCKED.equals(String.valueOf(result));
	}

	/**
	 * 获取redis连接
	 *
	 * @return Jedis
	 */
	private static Jedis getRedisConnection() {
		Jedis jedis = new Jedis("10.33.1.116", 6379);
		return jedis;
	}

	/**
	 * 释放锁lua脚本
	 *
	 * @return String
	 */
	private static String releaseLockScript() {
		String script = "if redis.call('get', KEYS[1]) == ARGV[1] then  return redis.call('del', KEYS[1]) else  return 0 end";
		return script;
	}
}
```

### 1.3 测试

测试代码

```java
package com.pkgs;

import java.text.SimpleDateFormat;
import java.util.Date;

import com.pkgs.lock.RedisLockUtil;

public class SetNxLock {

	private static final String KEY_NAME = "tickets-lock";
	private static final String RESOURCE_NAME = "tickets";

	public static void main(String[] args) {
		for (int index = 0; index < 5; index++) {
			new Thread(new Client("c" + index)).start();
		}
	}

	static class Client implements Runnable {
		private String threadName;

		public Client(String threadName) {
			this.threadName = threadName;
		}

		@Override
		public void run() {
			try {
				RedisLockUtil.lock(KEY_NAME, RESOURCE_NAME);
				System.out.println(getTime() + " " + threadName + " get the lock");
				// Thread.sleep(1000);
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				System.out.println(getTime() + " " + threadName + " release the lock");
				RedisLockUtil.unlock(KEY_NAME, RESOURCE_NAME);
			}
		}

		private static String getTime() {
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

			return sdf.format(new Date());
		}

	}
}
```

正常情况

```java
2018-11-26 09:49:00 c2 get the lock
2018-11-26 09:49:00 c2 release the lock
2018-11-26 09:49:00 c0 get the lock
2018-11-26 09:49:00 c0 release the lock
2018-11-26 09:49:00 c3 get the lock
2018-11-26 09:49:00 c3 release the lock
2018-11-26 09:49:00 c4 get the lock
2018-11-26 09:49:00 c4 release the lock
2018-11-26 09:49:00 c1 get the lock
2018-11-26 09:49:00 c1 release the lock
```

异常情况: 在`SetNxLock.Client#run`的执行方法里面添加`Thread.sleep(1100);`

结果如下所示,可以看出在方法还没执行完就释放锁了.

```java
2018-11-26 09:56:19 c3 get the lock
2018-11-26 09:56:20 c1 get the lock
2018-11-26 09:56:20 c3 release the lock
2018-11-26 09:56:20 c0 get the lock
2018-11-26 09:56:21 c1 release the lock
2018-11-26 09:56:21 c2 get the lock
2018-11-26 09:56:22 c0 release the lock
2018-11-26 09:56:22 c4 get the lock
2018-11-26 09:56:22 c2 release the lock
2018-11-26 09:56:23 c4 release the lock
```

---

## 2. Redission 锁

Redission 可以解决上面那些问题. 泪目

### 2.1 pom.xml

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>2.2.13</version>
</dependency>
```

### 2.2 代码

```java
package com.pkgs;

import java.text.SimpleDateFormat;
import java.util.Date;

import org.redisson.Config;
import org.redisson.Redisson;
import org.redisson.RedissonClient;
import org.redisson.core.RLock;

/**
 * redission分布式锁
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月26日
 * @see
 * @since 1.0
 */
public class RedissionLock {

	private static RedissonClient redisson = null;

	/**
	 * 锁名称,一般由:锁名称+资源组成
	 */
	private static String lockKey = "app.lock";
	private static String redisHost = "10.10.2.150:6379";

	public static void main(String[] args) {

		// 创建配置信息
		Config config = new Config();
		config.useSingleServer().setAddress(redisHost);
		redisson = Redisson.create(config);

		for (int index = 0; index < 5; index++) {
			new Thread(new Client("c" + index)).start();
		}

	}

	/**
	 * 测试线程
	 *
	 */
	static class Client implements Runnable {
		private String threadName;

		public Client(String threadName) {
			this.threadName = threadName;
		}

		@Override
		public void run() {
			RLock lock = redisson.getLock(lockKey);

			lock.lock();
			try {
				System.out.println(getTime() + " " + threadName + " get the lock");
				Thread.sleep(1000);
				System.out.println(getTime() + " " + threadName + " all is done");
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				lock.unlock();
				System.out.println(getTime() + " " + threadName + " release the lock");
			}

		}

		private static String getTime() {
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

			return sdf.format(new Date());
		}

	}
}
```

### 2.2 测试结果

```java
2018-11-26 09:54:23 c4 get the lock
2018-11-26 09:54:24 c4 all is done
2018-11-26 09:54:24 c4 release the lock
2018-11-26 09:54:24 c1 get the lock
2018-11-26 09:54:25 c1 all is done
2018-11-26 09:54:25 c1 release the lock
2018-11-26 09:54:25 c0 get the lock
2018-11-26 09:54:26 c0 all is done
2018-11-26 09:54:26 c0 release the lock
2018-11-26 09:54:26 c2 get the lock
2018-11-26 09:54:27 c2 all is done
2018-11-26 09:54:27 c2 release the lock
2018-11-26 09:54:27 c3 get the lock
2018-11-26 09:54:28 c3 all is done
2018-11-26 09:54:28 c3 release the lock
```

---

## 3. 结论

如果使用分布式锁,建议使用 redission 来做.
