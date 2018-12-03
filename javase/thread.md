# Java 之线程

多线程是绕不过去的,ta 是一条你要解决温饱的必经之路.

线程又绕不开 JMM(Java Memory Model),这里提供两个很优秀的 blog 作为参考.

a. [全面理解 Java 内存模型(JMM)及 volatile 关键字](https://blog.csdn.net/javazejian/article/details/72772461)

b. [深入理解 Java 并发之 synchronized 实现原理](https://blog.csdn.net/javazejian/article/details/72828483)

---

## 1. 死锁

死锁在在多线程里面是一个经常被面试的问题.

Q: **什么场景下会产生死锁呀???**

A: **thread1 持有 lock1 想去持有 lock2,而 thread2 持有 lock2 想去持有 lock1.**

在这种相互竞争的前提下,就产生死锁.

### 1.1 代码

```java
package test;

public class DeadLock {

	private static Object lock1 = new Object();
	private static Object lock2 = new Object();

	public static void main(String[] args) {
		new Thread(new MyRun1()).start();
		new Thread(new MyRun2()).start();
	}

	static class MyRun1 implements Runnable {
		@Override
		public void run() {
			try {
				synchronized (lock1) {
					// sleep 一下,确保线程2 获得lock2的锁
					System.out.println(this + " get lock1");
					Thread.sleep(1000);
					synchronized (lock2) {
						System.out.println(this + " is going well");
					}
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	static class MyRun2 implements Runnable {
		@Override
		public void run() {
			try {
				System.out.println(this + " get lock2");
				synchronized (lock2) {
					// sleep 一下,确保线程1 获得lock1的锁
					Thread.sleep(1000);
					synchronized (lock1) {
						System.out.println(this + " is going well");
					}
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}
}
```

测试结果

```java
test.DeadLock$MyRun1@4818f0c4 get lock1
test.DeadLock$MyRun2@2f7f9bcc get lock2
```

可以看出:`going well`那里面的没有一句话是打印出来的.

### 1.2 分析

下面我们使用 jstack 来看看线程情况.

```sh
$ jps -lm |grep DeadLock
13008 test.DeadLock

$ jstack -l 13008

....

Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00000000576f7998 (object 0x00000000d65ac450, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00000000576f6398 (object 0x00000000d65ac460, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
        at test.DeadLock$MyRun2.run(DeadLock.java:40)
        - waiting to lock <0x00000000d65ac450> (a java.lang.Object)
        - locked <0x00000000d65ac460> (a java.lang.Object)
        at java.lang.Thread.run(Thread.java:745)
"Thread-0":
        at test.DeadLock$MyRun1.run(DeadLock.java:22)
        - waiting to lock <0x00000000d65ac460> (a java.lang.Object)
        - locked <0x00000000d65ac450> (a java.lang.Object)
        at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.
```

在堆栈信息的后面有死锁存在.

thread-1 持有锁`0x00000000d65ac460`,在等待`0x00000000d65ac450`.

thread-2 持有锁`0x00000000d65ac450`,在等待`0x00000000d65ac450`.

---

## 2. 生产者/消费者模式

面试中经常遇到的`生产者/消费者`模式,经典的`wait/notify`的使用场景.

### 2.1 基础知识

重点: **wait 会释放锁,进入 blocked 状态,当被 notify 的时候再重新进行 ready-to-run 里面.**

gotMsg=false

消费者获得锁 -> if(!gotMsg){Lock.class.wait()} do something -> done.

生产者获取消费者释放的锁 -> 生产消息 -> gotMsg=ture ->Lock.class.notify().

### 2.2 测试代码

```java
package test;
/**
 * 生产者/消费者模式
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月15日
 * @see
 * @since 1.0
 */
public class Factory {

	private static String msg = "";

	private static boolean isHaveWorkToDo = false;

	public static void main(String[] args) {
		new Thread(new MyConsumer()).start();
		new Thread(new MyProvider()).start();
	}

	/**
	 * 消费者
	 */
	static class MyConsumer implements Runnable {
		@Override
		public void run() {
			try {
				synchronized (Factory.class) {
					System.out.println(this + " get the lock");
					//这里面,为什么使用while而不是if?请参考下面的QA
					while(!isHaveWorkToDo) {
						System.out.println(this + " got to wait");
						Factory.class.wait();
					}
					System.out.println(this + " consumer: " + msg);
				}

			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	/**
	 * 生产者
	 */
	static class MyProvider implements Runnable {
		@Override
		public void run() {
			try {
				synchronized (Factory.class) {
					System.out.println(this + " get the lock");
					msg = "timestamp:" + System.currentTimeMillis();
					System.out.println(this + " provider: " + msg);
					isHaveWorkToDo = false;

					Factory.class.notify();
				}

			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}
}
```

测试结果

```java
test.Factory$MyConsumer@64f7047 get the lock
test.Factory$MyConsumer@64f7047 got to wait
test.Factory$MyProvider@612327cd get the lock
test.Factory$MyProvider@612327cd provider: timestamp:1542247858340
test.Factory$MyConsumer@64f7047 consumer: timestamp:1542247858340
```

Q: 为什么使用`while`而不是`if`?

A: 在没有被通知,中断或超时的情况下,线程还可以唤醒一个所谓的虚假唤醒(spurious wakeup,唤醒时,条件仍然不满足).虽然这种情况在实践中很少发生,但是应用程序必须通过以下方式防止其发生,即对应该导致该线程被提醒的条件进行测试,如果不满足该条件,则继续等待. [link](https://www.cnblogs.com/nulisaonian/p/6076674.html)

---

## 3. CountdownLatch

适用于: 在主线程需要等待所有子线程执行完,再往下执行的场景.

最简单的就是使用 `join` 来实现,但是 `join` 并不能完全实现多线程的作用.

### 3.1 测试线程

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.CountDownLatch;

/**
 * 测试线程
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月28日
 * @see
 * @since 1.0
 */
public class MyRun implements Runnable {

	/**
	 * 线程名称
	 */
	private String threadName;
	/**
	 * {@link CountDownLatch}
	 */
	private CountDownLatch latch;

	public MyRun(String threadName) {
		this(threadName, null);
	}

	public MyRun(String threadName, CountDownLatch latch) {
		this.threadName = threadName;
		this.latch = latch;
	}

	@Override
	public void run() {
		System.out.println(getTime() + " - " + threadName + " start running");
		try {
			Thread.sleep(1000);
		} catch (Exception e) {
			e.printStackTrace();
		}
		System.out.println(getTime() + " - " + threadName + " all done");
		// latch count down
		if (latch != null) {
			latch.countDown();
		}

	}

	/**
	 * 获取当前时间
	 *
	 * @return
	 */
	public static String getTime() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss,SSS");
		return sdf.format(new Date());
	}
}
```

### 3.2 join

```java
/**
 * 测试Join的使用
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月28日
 * @see
 * @since 1.0
 */
public class MyJoin {

	public static void main(String[] args) {
		// 线程数
		int threadNum = 3;

		// 启动线程
		for (int index = 0; index < threadNum; index++) {
			Thread t = new Thread(new MyRun("t" + index));
			t.start();
			try {
				t.join();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		System.out.println(MyRun.getTime() + " - main all is done");
	}

}
```

测试结果

```java
2018-11-28 14:10:01,728 - t0 start running
2018-11-28 14:10:02,729 - t0 all done
2018-11-28 14:10:02,731 - t1 start running
2018-11-28 14:10:03,731 - t1 all done
2018-11-28 14:10:03,733 - t2 start running
2018-11-28 14:10:04,733 - t2 all done
2018-11-28 14:10:04,734 - main all is done
```

### 3.3 countdown

```java
import java.util.concurrent.CountDownLatch;
/**
 * 测试{@link CountDownLatch}的使用
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月28日
 * @see
 * @since 1.0
 */
public class MyLatch {

	/**
	 * CountdownLacth对象
	 */
	private static CountDownLatch latch;

	public static void main(String[] args) {

		// 线程数
		int threadNum = 3;

		// 实例化CountdownLatch
		latch = new CountDownLatch(threadNum);

		// 启动线程
		for (int index = 0; index < threadNum; index++) {
			new Thread(new MyRun("t" + index, latch)).start();
		}

		try {
			// 注意是await,而不是wait
			latch.await();
		} catch (Exception e) {
			e.printStackTrace();
		}

		System.out.println(MyRun.getTime() + " - main all is done");
	}

}
```

测试结果

```java
2018-11-28 14:10:52,699 - t2 start running
2018-11-28 14:10:52,699 - t0 start running
2018-11-28 14:10:52,699 - t1 start running
2018-11-28 14:10:53,700 - t2 all done
2018-11-28 14:10:53,700 - t1 all done
2018-11-28 14:10:53,700 - t0 all done
2018-11-28 14:10:53,701 - main all is done
```

### 3.4 结论

可以看出`join`相当于单线程执行,`CountdownLatch`更能发回多线程的作用.

所以遇到相似的场景时,应选用`CountdownLatch`来代替`join`.

---
