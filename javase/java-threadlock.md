# 线程与锁

这里主要记录一下线程与锁的使用.

你绕不开的,线程与为锁,只有一点一点的熟悉了. `orz`

---

## 1. 基础知识

### 1.1 Java 内存模型

Java 的线程模型(`Java Memory Model`),在 Java 里面所有的变量都存在主存里面(这个主存和 `GC` 里面的内存划分有区别,请注意).线程开启的时候,会从存储里面 copy 一份到自己的工作内存里面.每一个线程之间的工作内存相互独立的(不可见性).只有等线程操作完成了,才会刷新到主存里面去.

因为各个线程之间的工作空间是不可见的,所以线程 1 的改变数据还没来得及刷回主存,而线程 2 获取到的数据还是没改变之前的,就存在了线程安全问题.

在 Java 里面为了解决这种问题,就涉及到锁了. 微笑脸.jpg

### 1.2 简要说明

在 Java 里面:`volatile`,`synchronized`

- volatile:对数据的主要作用是每一次操作都强制要求线程使用主存里面的数据,每一次操作完立刻刷新到主存.如果该操作不是原子性的(如 i++这种),依然会有线程安全问题.

- synchronized:加锁前,强制同步主内存的数据到工作空间.解锁的前,必须把工作空间里面的刷回主内存.

You see, 最简单的概念也不是很难,对不对?

---

## 2. volatile 的使用

哔哔了那么多,你还没给我看看该怎么使用`volatile`呢!!!

ok,假设有这么一个案例,就是在线程 1 修改一个标志位,而线程 2 根据这个标志位,是否继续运行.

### 2.1 示例代码

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class VolatileTest {

	private static boolean isStop = false;

	public static void main(String[] args) {
		new Thread(new Commander()).start();
		new Thread(new Worker()).start();
	}

	static class Commander implements Runnable {
		@Override
		public void run() {
			System.out.println(this.getClass().getSimpleName() + " - " + getTime() + " set isStop=true");
			isStop = true;
		}
	}

	static class Worker implements Runnable {
		@Override
		public void run() {
			while (!isStop) {
			}
			System.out.println(this.getClass().getSimpleName() + " - " + getTime() + " has been stop");
		}
	}

	private static String getTime() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:sss");

		return sdf.format(new Date());
	}
```

测试结果

```java
Commander - 2018-10-28 16:59:41:041 set isStop=true
```

可以看出 Worker 的线程一直在`while(isStop){}`里面循环.

Q: 为什么`Commander`设置了`isStop=true`之后 ,`Worker`没有停止呢?

A: 因为`Commander`和`Worker`运行的时候,都 copy 一份`isStop=false`的变量到自己的工作内存.`Commander`操作完成了,刷回了主存,但是`Worker`并没使用主存里面最新的值,而还是自己的工作空间里面的值.

那么我们可以给`isStop`加上`volatile`关键字.保证 `Worker` 每一次获取都是主存里面的值,而不是自己工作空间里面的.

### 2.2 正确范例

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class VolatileTest {

	/**
	 * 加上volatile关键字
	 */
	private static volatile boolean isStop = false;

	public static void main(String[] args) {
		new Thread(new Commander()).start();
		new Thread(new Worker()).start();
	}

	static class Commander implements Runnable {
		@Override
		public void run() {
			System.out.println(this.getClass().getSimpleName() + " - " + getTime() + " set isStop=true");
			isStop = true;
		}
	}

	static class Worker implements Runnable {
		@Override
		public void run() {
			while (!isStop) {
			}
			System.out.println(this.getClass().getSimpleName() + " - " + getTime() + " has been stop");
		}
	}

	private static String getTime() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss:sss");

		return sdf.format(new Date());
	}
}
```

测试结果

```java
Commander - 2018-10-28 17:05:38:038 set isStop=true
Worker - 2018-10-28 17:05:38:038 has been stop
```

---

## 3. synchronized 的使用

`synchronized`: **悲观并发策略**.

在 Java 里面`volatile`是轻量级的锁.但是你以为 `volatile` 就可以解决一切问题的话,不好意思,`volatile`表示要打产品经理一顿.`volatile`对于原子性的操作是可以保证数据安全的,但是非原子性的操作(如 i++)就只有上`synchronized`了.

i++的非原子性是: 首先要从主存加载数据-> 自增 -> 将结果刷回主存.

synchronized 关键字 3 种应用方式

- 修饰实例方法,作用于当前实例加锁,进入同步代码前要获得当前实例的锁

- 修饰静态方法,作用于当前类对象(class)加锁,进入同步代码前要获得当前类对象的锁

- 修饰代码块,指定加锁对象,对给定对象加锁,进入同步代码库前要获得给定对象的锁

### 3.1 示例代码

```java
import java.util.concurrent.CountDownLatch;

public class SyncTest {

	private static volatile long num = 0;

	public static void main(String[] args) {
		int time = 20000;
		CountDownLatch latch = new CountDownLatch(time);
		for (int index = 0; index < time; index++) {
			Runnable r = () -> {
				increase();
				latch.countDown();
			};
			new Thread(r).start();
		}
		try {
			// 是await,而不是wait
			latch.await();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(new Worker()).start();
	}

	private static void increase() {
		num++;
	}

	static class Worker implements Runnable {
		@Override
		public void run() {
			System.out.println(this.getClass().getSimpleName() + ", the num is: " + num);
		}
	}
}
```

运行多次测试结果,其中有一次计算出现问题,正确结果应该是 7 才对.

```java
Worker, the num is: 19998
```

### 3.2 正确范例

```java
import java.util.concurrent.CountDownLatch;

public class SyncTest {

	private static volatile long num = 0;

	public static void main(String[] args) {
		int time = 20000;
		CountDownLatch latch = new CountDownLatch(time);
		for (int index = 0; index < time; index++) {
			Runnable r = () -> {
				increase();
				latch.countDown();
			};
			new Thread(r).start();
		}
		try {
			// 是await,而不是wait
			latch.await();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		new Thread(new Worker()).start();
	}

	private synchronized static void increase() {
		num++;
	}

	static class Worker implements Runnable {
		@Override
		public void run() {
			System.out.println(this.getClass().getSimpleName() + ", the num is: " + num);
		}
	}
}
```

测试结果

```java
Worker, the num is: 20000
```

关于`synchronized`加载不同地方的作用,请查阅参考资料里面的`深入理解 Java 并发之 synchronized 实现原理`文档.

---

## 4. ReentrantLock

### 4.1 基础知识

ReentrantLock 和 synchronized,前者的先进性体现在以下几点:

- 可响应中断的锁.当在等待锁的线程如果长期得不到锁,那么可以选择不继续等待而去处理其他事情,而 synchronized 的互斥锁则必须阻塞等待,不能被中断.

- 可实现公平锁.所谓公平锁指的是多个线程在等待锁的时候必须按照线程申请锁的时间排队等待,而非公平性锁则保证这点,每个线程都有获得锁的机会.synchronized 的锁和 ReentrantLock 使用的默认锁都是非公平性锁,但是 ReentrantLock 支持公平性的锁,在构造函数中传入一个 boolean 变量指定为 true 实现的就是公平性锁.不过一般而言,使用非公平性锁的性能优于使用公平性锁

- 每个 synchronized 只能支持绑定一个条件变量,这里的条件变量是指线程执行等待或者通知的条件,而 ReentrantLock 支持绑定多个条件变量,通过调用 lock.newCondition()可获取多个条件变量.不过使用多少个条件变量需要依据具体情况确定.

Q: 什么时候使用`ReentrantLock`?

A: 在一些内置锁无法满足一些高级功能的时候才考虑使用 `ReentrantLock`

### 4.2 示例代码

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockTest {
	private static ReentrantLock lock = new ReentrantLock();
	private static volatile long num = 0;

	public static void main(String[] args) {
		Commander sir = new Commander();
		new Thread(sir).start();
		new Thread(sir).start();
		new Thread(sir).start();
		new Thread(sir).start();
		new Thread(sir).start();
		new Thread(sir).start();
		new Thread(sir).start();

		new Thread(new Worker()).start();
	}

	static class Commander implements Runnable {
		@Override
		public void run() {
			increase();
		}
	}

	private static void increase() {
		// 一定要在finally里面释放锁,不要抛出异常,其他的线程永远都不可能获取到锁.
		lock.lock();
		try{
			num++;
		}finally{
			lock.unlock();
		}
	}

	static class Worker implements Runnable {
		@Override
		public void run() {
			System.out.println(this.getClass().getSimpleName() + ", the num is: " + num);
		}
	}
}
```

测试结果

```java
Worker, the num is: 7
```

---

## 5. 参考资料

a. [全面理解 Java 内存模型(JMM)及 volatile 关键字](https://blog.csdn.net/javazejian/article/details/72772461)

b. [深入理解 Java 并发之 synchronized 实现原理](https://blog.csdn.net/javazejian/article/details/72828483)

c. [Java 并发编程系列之十六：Lock 锁](https://blog.csdn.net/u011116672/article/details/51064186?utm_source=blogxgwz0)

d. [IBM:ReentrantLock 和 synchronized 对比](https://www.ibm.com/developerworks/cn/java/j-jtp10264/index.html)

e. [悲观锁与乐观锁](https://blog.csdn.net/qq_34337272/article/details/81072874)