# Redis 之分布式锁

分布式锁是一个要掌握的关键知识点,特别应用在分布式系统的资源访问控制上面.

---

## 1. Redis 锁

网上实现简单的分布式锁,几乎都是使用 `setnx+超时`命令来完成.

这个简单的方法存在天然的缺陷

- <u>方法执行耗时长于锁的过时时间,在方法还没执行完就自动释放锁.</u>
- <u>如果设置过时时间过长,执行过程 down 的话,会给其他没 down 导致很长的等待时间.</u>

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
			/*
			 * 使用 uuid 作为value,可以确定释放当前占用的锁
			 *
			 * 如果使用统一名称(如tickets),可以没获得锁,出现异常在finally里面释放掉了,那就操蛋了.
			 */
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
			 * 如果本jvm获得锁,在释放锁前down掉了,那么锁一直没能被释放,其他获取不到,就相当于死锁了!!!
			 *
			 * 所以,建议采用: void lock(long leaseTime, TimeUnit unit);
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

KEYS[1]代表的是你加锁的那个 key,比如说:`RLock lock = redisson.getLock("myLock");`这里你自己设置了加锁的那个锁 key 就是“myLock”.

ARGV[1]代表的就是锁 key 的默认生存时间,默认 30 秒.

ARGV[2]代表的是加锁的客户端的 ID,类似于下面这样:8743c9c0-0795-4907-87fd-6c719a6b4586:1

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

### 2.5 问题

Q: 如果获取锁的线程在运行过程中挂掉了,没有释放锁,会不会出现死锁情况?

A: 不会的,因为在`tryLock`这种没有明显设置超时时间的情况,其实也是默认设置了一个 30s 的超时时间.

```java
 private Future<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
	// leaseTime设置的情况
	if (leaseTime != -1) {
		return tryLockInnerAsync(leaseTime, unit, threadId);
	}
	// leaseTime没设置的情况
	return tryLockInnerAsync(threadId);
}

private Future<Long> tryLockInnerAsync(final long threadId) {
	// 这里面设置30秒的超时时间
	Future<Long> ttlRemaining = tryLockInnerAsync(LOCK_EXPIRATION_INTERVAL_SECONDS, TimeUnit.SECONDS, threadId);
	ttlRemaining.addListener(new FutureListener<Long>() {
		@Override
		public void operationComplete(Future<Long> future) throws Exception {
			if (!future.isSuccess()) {
				return;
			}

			Long ttlRemaining = future.getNow();
			// lock acquired
			if (ttlRemaining == null) {
				// 自动续约逻辑
				scheduleExpirationRenewal();
			}
		}
	});
	return ttlRemaining;
}
```

Q: 那线程在没设置超时时间的话,只有 30s,运行时间超过了 30s 的情况,是怎么做到自动续约?

A: Let check this out.

在使用 redisson 分布式锁的时候,有一块`很重要,很重要,很重要的代码`.特别,特别,特别要注意`leaseTime`的情况.

```java
/**
* reidsson分布式方法
*
* @param leaseTime 锁持有时间
* @param unit      时间类型
* @param threadId  线程id
* @return RFuture
*/
private RFuture<Boolean> tryAcquireOnceAsync(long leaseTime, TimeUnit unit, final long threadId) {

	/*
	*如果在使用锁时候,声明了锁的持有时间,则不走自动续约的流程
	*
	* 比如声明leaseTime=10s,但是程序执行需要15s,那么也会在10s的释放锁.
	*
	* 声明leaseTime适合用在那种明确知道程序执行耗时的边界值的场景,不然就不要骚了.
	*/
	if (leaseTime != -1L) {
		return this.tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
	} else {
	   /*
		* watchdog 自动续约逻辑
		*/
		RFuture<Boolean> ttlRemainingFuture = this.tryLockInnerAsync(this.commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
		ttlRemainingFuture.addListener(new FutureListener<Boolean>() {
			public void operationComplete(Future<Boolean> future) throws Exception {
				if (future.isSuccess()) {
					Boolean ttlRemaining = (Boolean) future.getNow();
					if (ttlRemaining) {
						// 自动续约逻辑
						RedissonLock.this.scheduleExpirationRenewal(threadId);
					}

				}
			}
		});
		return ttlRemainingFuture;
   }
}
```

现在看看自动续约的逻辑

```java
/**
* 自动续约
*
* @param threadId 线程id
*/
private void scheduleExpirationRenewal(final long threadId) {
if (!expirationRenewalMap.containsKey(this.getEntryName())) {
	/*
	*  构建定时器,每`this.internalLockLeaseTime / 3L`定时去续约`internalLockLeaseTime`时长,这样子也就解决了死锁的问题
	*
	*  因为不会一直持有,如果你设置的时间和合理的话,默认为30s.
	*/
	Timeout task = this.commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
		public void run(Timeout timeout) throws Exception {
			RFuture<Boolean> future = RedissonLock.this.commandExecutor.evalWriteAsync(RedissonLock.this.getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN, "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then redis.call('pexpire', KEYS[1], ARGV[1]); return 1; end; return 0;", Collections.singletonList(RedissonLock.this.getName()), new Object[]{RedissonLock.this.internalLockLeaseTime, RedissonLock.this.getLockName(threadId)});
			future.addListener(new FutureListener<Boolean>() {
				public void operationComplete(Future<Boolean> future) throws Exception {
					RedissonLock.expirationRenewalMap.remove(RedissonLock.this.getEntryName());
					if (!future.isSuccess()) {
						RedissonLock.log.error("Can't update lock " + RedissonLock.this.getName() + " expiration", future.cause());
					} else {
						if ((Boolean) future.getNow()) {
							RedissonLock.this.scheduleExpirationRenewal(threadId);
						}

					}
				}
			});
		}
	}, this.internalLockLeaseTime / 3L, TimeUnit.MILLISECONDS);
	if (expirationRenewalMap.putIfAbsent(this.getEntryName(), task) != null) {
		task.cancel();
	}

}
}
```

---

## 3. 结论

如果使用分布式锁,建议使用 `redission` 来做.

Q: 看 redisson 里面的加锁/释放都是使用 lua 脚本,那么这些脚本代表是什么意思呀?

例如:`org.redisson.RedissonLock#tryLockInnerAsync`加锁的代码

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
	internalLockLeaseTime = unit.toMillis(leaseTime);

	return evalWriteAsync(getName(), LongCodec.INSTANCE, command,
			"if (redis.call('exists', KEYS[1]) == 0) then " +
					"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
					"redis.call('pexpire', KEYS[1], ARGV[1]); " +
					"return nil; " +
					"end; " +
					"if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
					"redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
					"redis.call('pexpire', KEYS[1], ARGV[1]); " +
					"return nil; " +
					"end; " +
					"return redis.call('pttl', KEYS[1]);",
			Collections.singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```

A: 首先`lua数组下标默认是从1开始`,惊不惊喜意不意外.

那么看看: `org.redisson.command.CommandAsyncService#evalAsync`是怎么解析的

```java
/**
 * List<Object> keys: 为脚本里面的KEYS[number]的取值数组,下标从1开始
 *
 * Object... params: 为脚本ARGV[number]的取值数据,下标从1开始
 *
 */
private <T, R> RFuture<R> evalAsync(NodeSource nodeSource, boolean readOnlyMode, Codec codec, RedisCommand<T> evalCommandType, String script, List<Object> keys, Object... params) {

	RPromise<R> promise = new RedissonPromise<R>();
	String sha1 = calcSHA(script);
	RedisCommand cmd = new RedisCommand(evalCommandType, "EVALSHA");

	// 组装请求数据
	List<Object> args = new ArrayList<Object>(2 + keys.size() + params.length);
	args.add(sha1);
	args.add(keys.size());
	args.addAll(keys);
	args.addAll(Arrays.asList(params));

    // 发送脚本和参数,交给redis服务器解析
	RedisExecutor<T, R> executor = new RedisExecutor<>(readOnlyMode, nodeSource, codec, cmd,
												args.toArray(), promise, false, connectionManager, objectBuilder);
	executor.execute();
}
```

Redis 中使用 EVAL 命令来直接执行指定的 Lua 脚本。

```sh
EVAL luascript numkeys key [key ...] arg [arg ...]
- EVAL 命令的关键字
- luascript Lua 脚本
- numkeys 脚本需要处理键的数量,key数组的长度
- key 零到多个键,空格隔开,通过KEYS[index]来获取对应的值,其中1 <= index <= keys.length
- arg 零到多个附加参数,空格隔开,通过ARGV[index]来获取对应的值，其中1 <= index <= args.length
```

---

## 4. 参考资料

a. [Redission 分布式锁实现原理](https://www.cnblogs.com/diegodu/p/8185480.html)

b. [Redisson 分布式锁自动续约]((https://www.jianshu.com/p/2a90bba8922f)
