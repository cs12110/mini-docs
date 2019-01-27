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
import java.text.SimpleDateFormat;
import java.util.Date;

public class SetNxLock {

	private static final String KEY_NAME = "tickets-lock";
	// private static final String RESOURCE_NAME = "tickets";

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
			//使用 uuid 作为value,可以确定释放当前占用的锁
			String value = UUID.randomUUID().toString();
			try {
				RedisLockUtil.lock(KEY_NAME, value);
				System.out.println(getTime() + " " + threadName + " get the lock");
				Thread.sleep(100);
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				System.out.println(getTime() + " " + threadName + " release the lock");
				RedisLockUtil.unlock(KEY_NAME, value);
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

			/* 
			 * 要注意服务器异常导致的死锁
			 * 
			 * 建议采用: void lock(long leaseTime, TimeUnit unit);
			 */
			lock.lock();

			try {
				System.out.println(getTime() + " " + threadName + " get the lock");
				Thread.sleep(1000);
				System.out.println(getTime() + " " + threadName + " all is done");
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				System.out.println(getTime() + " " + threadName + " release the lock");
				lock.unlock();
			}
		}

		private static String getTime() {
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			return sdf.format(new Date());
		}
	}
}
```

### 2.3 测试结果

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

### 2.4 Redission 源码

RedissionLock 主要是使用 lua eval 执行脚本(保证原子性),使用 hset+发布/订阅实现的.

**加锁源码**

`lockInterruptibly`: 获取锁,不成功则订阅释放锁的消息,获得消息前阻塞.得到释放通知后再去循环获取锁.

```java
@Override
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
	// ttl: time to live
	Long ttl = tryAcquire(leaseTime, unit);
	// lock acquired
	if (ttl == null) {
		return;
	}

	// 异步订阅redis channel
	Future<RedissonLockEntry> future = subscribe();
	get(future);

	try {
		// 循环判断是否获取锁
		while (true) {
			ttl = tryAcquire(leaseTime, unit);
			// lock acquired
			if (ttl == null) {
				break;
			}

			// waiting for message
			RedissonLockEntry entry = getEntry();
			if (ttl >= 0) {
				entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
			} else {
				entry.getLatch().acquire();
			}
		}
	} finally {
		// 取消订阅
		unsubscribe(future);
	}
}
```

```java
Future<Long> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId) {
internalLockLeaseTime = unit.toMillis(leaseTime);

return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_LONG,
			"if (redis.call('exists', KEYS[1]) == 0) then " +
				"redis.call('hset', KEYS[1], ARGV[2], 1); " +
				"redis.call('pexpire', KEYS[1], ARGV[1]); " +
				"return nil; " +
			"end; " +
			"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
				"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
				"redis.call('pexpire', KEYS[1], ARGV[1]); " +
				"return nil; " +
			"end; " +
			"return redis.call('pttl', KEYS[1]);",
			Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```

**解锁源码**

```java
@Override
public void unlock() {
	Boolean opStatus = commandExecutor.evalWrite(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
					"if (redis.call('exists', KEYS[1]) == 0) then " +
						"redis.call('publish', KEYS[2], ARGV[1]); " +
						"return 1; " +
					"end;" +
					"if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
						"return nil;" +
					"end; " +
					"local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
					"if (counter > 0) then " +
						"redis.call('pexpire', KEYS[1], ARGV[2]); " +
						"return 0; " +
					"else " +
						"redis.call('del', KEYS[1]); " +
						"redis.call('publish', KEYS[2], ARGV[1]); " +
						"return 1; "+
					"end; " +
					"return nil;",
					Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(Thread.currentThread().getId()));
	if (opStatus == null) {
		throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
				+ id + " thread-id: " + Thread.currentThread().getId());
	}
	if (opStatus) {
		cancelExpirationRenewal();
	}
}
```

---

## 3. 结论

如果使用分布式锁,建议使用 `redission` 来做.

---

## 4. 参考资料

a. [Redission 分布式锁实现原理](https://www.cnblogs.com/diegodu/p/8185480.html)
