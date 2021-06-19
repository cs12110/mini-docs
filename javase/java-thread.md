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

thread-2 持有锁`0x00000000d65ac450`,在等待 ~~`0x00000000d65ac450`~~ `0x00000000d65ac460`.

---

## 2. 生产者/消费者模式

面试中经常遇到的`生产者/消费者`模式,经典的`wait/notify`的使用场景.

tips: 无论是`wait`还是`notify`都是在`synchronized`代码块里面操作的.

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

A: 在没有被通知,中断或超时的情况下,线程还可以唤醒一个所谓的<u>虚假唤醒</u>(spurious wakeup,唤醒时,条件仍然不满足).虽然这种情况在实践中很少发生,但是应用程序必须通过以下方式防止其发生,即对应该导致该线程被提醒的条件进行测试,如果不满足该条件,则继续等待. [link](https://www.cnblogs.com/nulisaonian/p/6076674.html)

---

## 3. CountdownLatch

适用于: 在主线程需要等待所有子线程执行完,再往下执行的场景.

最简单的就是使用 `join` 来实现,但是 `join` 并不能完全实现多线程的作用.

关于 join 的内部实现,请参考源码. [Java 中 Thread 类的 join 方法到底是如何实现等待的?](https://www.zhihu.com/question/44621343/answer/97640972)

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

## 4. volatile

在线程里面所有操作的数据来自该线程线程的工作内存,操作完成后再刷回主存里面去.

Q: 在执行方法的时候,有哪些数据存放到工作空间里面呀?

A: 根据虚拟机规范.对于一个实例对象中的成员方法而言.如果方法中包含本地变量是基本数据类型(boolean,byte,short,char,int,long,float,double).将直接存储在工作内存的帧栈结构中.但倘若本地变量是引用类型.那么该变量的引用会存储在功能内存的帧栈中.而对象实例将存储在主内存(共享数据区域.堆)中.但对于实例对象的成员变量.不管它是基本数据类型或者包装类型(Integer、Double 等)还是引用类型.都会被存储到堆区.至于 static 变量以及类本身相关信息将会存储在主内存中.需要注意的是.在主内存中的实例对象可以被多线程共享.倘若两个线程同时调用了同一个对象的同一个方法.那么两条线程会将要操作的数据拷贝一份到自己的工作内存中.执行完成操作后才刷新到主内存.[link](https://blog.csdn.net/javazejian/article/details/72772461)

### 4.1 测试代码

```java
package com.test;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/20 15:09
 * <p>
 * since: 1.0.0
 */
public class MyVolatileTest {

    private boolean isStop = false;

    public static void main(String[] args) {
        MyVolatileTest test = new MyVolatileTest();

        new Thread(test.new Run2()).start();
        /*
         * 确保run2比run1先运行. ta们各自把isStop拷贝到自己的工作空间.
				 * 不然run2拿到的可能是run1修改后刷回主存的isStop=true了.
         */
        try {
            Thread.sleep(100);
        } catch (Exception e) {
            e.printStackTrace();
        }
        new Thread(test.new Run1()).start();
    }

    public class Run1 implements Runnable {
        @Override
        public void run() {
            isStop = true;
            System.out.println("Run1 set isStop=true");
        }
    }

    public class Run2 implements Runnable {
        @Override
        public void run() {
            System.out.println("Run2 start");
            while (!isStop) {
                // try {
                //     Thread.sleep(10);
                // } catch (Exception e) {
                //     e.printStackTrace();
                // }
            }
            System.out.println("Run2 done");
        }
    }
}
```

测试结果

```java
Run2 start
Run1 set isStop=true
```

### 4.2 volatile

```java
package com.test;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/20 15:09
 * <p>
 * since: 1.0.0
 */
public class MyVolatileTest {

    private volatile boolean isStop = false;

    public static void main(String[] args) {
        MyVolatileTest test = new MyVolatileTest();

        /*
         * 确保run2比run1先运行. ta们各自把isStop拷贝到自己的工作空间.
         *
         *
         *
         */
        new Thread(test.new Run2()).start();
        try {
            Thread.sleep(100);
        } catch (Exception e) {
            e.printStackTrace();
        }
        new Thread(test.new Run1()).start();
    }

    public class Run1 implements Runnable {
        @Override
        public void run() {
            isStop = true;
            System.out.println("Run1 set isStop=true");
        }
    }

    public class Run2 implements Runnable {
        @Override
        public void run() {
            System.out.println("Run2 start");
            while (!isStop) {
                // try {
                //     Thread.sleep(10);
                // } catch (Exception e) {
                //     e.printStackTrace();
                // }
            }
            System.out.println("Run2 done");
        }
    }
}
```

测试结果

```java
Run2 start
Run1 set isStop=true
Run2 done
```

---

## 5. Callable 与 Future

JVM 在 ta 的日志轻轻的写下了: 在多线程里面,时值天下三分,Thread,Runnable,Callable 各成一方豪强.

Q: 为什么要特地说一下 Callable 呀?

A: ~~我喜欢呀.~~ 在某些特殊的场景里面,**需要线程执行完之后的结果(如 mapreduce)**,Thread 与 Runnable 都不能实现,所以选择使用 Callable.

### 5.1 代码

```java
import com.pkgs.util.ThreadUtil;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.*;
import java.util.function.Supplier;

/**
 * callable
 *
 * <p/>
 *
 * @author cs12110 created at: 2019/2/22 16:48
 * <p>
 * since: 1.0.0
 */
public class MyFuture {

    /**
     * 日期提供类
     */
    private static Supplier<String> dateSupplier = () -> {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(new Date());
    };


    public static void main(String[] args) {
        int threadNum = 2;
        ExecutorService executorService = new ThreadPoolExecutor(
                threadNum,
                threadNum,
                0,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(threadNum),
                ThreadUtil.buildThreadFactory("callable-pool")
        );

        // 提交callable任务
        List<Future<Integer>> futureList = new ArrayList<>(threadNum);
        for (int i = 0; i < threadNum; i++) {
            Future<Integer> future = executorService.submit(new CallMe());
            futureList.add(future);
        }

        // 从future获取callable任务的返回值
        for (Future<Integer> f : futureList) {
            try {
                Integer value = f.get();
                System.out.println(dateSupplier.get() + " " + value);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * callable 线程
     */
    public static class CallMe implements Callable<Integer> {
        @Override
        public Integer call() {
            System.out.println(dateSupplier.get() + " " + Thread.currentThread().getName() + " start");
            try {
                Thread.sleep(5000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println(dateSupplier.get() + " " + Thread.currentThread().getName() + " done");
            return 1;
        }
    }
}
```

### 5.2 测试结果

```java
2019-02-22 17:18:10 pool-1 start
2019-02-22 17:18:10 pool-2 start
2019-02-22 17:18:15 pool-1 done
2019-02-22 17:18:15 pool-2 done
2019-02-22 17:18:15 1
2019-02-22 17:18:15 1
```

---

## 6. 定时器

在现实里面需要用到定时器之类的东西,但是如果一个简单的定时任务也用 quartz 这种定时框架,就有点大材小用了.

so,我们看一下最基础,最简单的实现.

### 6.1 timer

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;
import java.util.function.Supplier;

/**
 * @author cs12110 create at 2019/5/14 13:37
 * @version 1.0.0
 */
public class SimpleTimer {

    public static void main(String[] args) {
        Timer timer = new Timer();
        /*
         * 设置定时任务
         *
         * delay: 启动线程后多少毫秒开始执行
         *
         * period: 周期,多少毫秒执行一次
         */
        timer.schedule(new MyTimer(), 2000, 3000);
    }


    /**
     * 定时任务
     */
    static class MyTimer extends TimerTask {
        private Supplier<String> dateSupplier = () -> {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            return sdf.format(new Date());
        };

        @Override
        public void run() {
            // do whatever you want to do
            System.out.println(dateSupplier.get() + " - invokes");
        }
    }
}
```

测试结果

```java
2019-05-14 13:49:37 - invokes
2019-05-14 13:49:40 - invokes
2019-05-14 13:49:43 - invokes
2019-05-14 13:49:46 - invokes
2019-05-14 13:49:49 - invokes
2019-05-14 13:49:52 - invokes
2019-05-14 13:49:55 - invokes
2019-05-14 13:49:58 - invokes
```

### 6.2 ScheduledThreadPoolExecutor

Q: 怎么可以这么 low? 如果只写 runnable 可不可以定时执行呀?

A: 你这样子自问自答,让我很尴尬耶.

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

/**
 * @author cs12110 create at 2019/5/14 13:37
 * @version 1.0.0
 */
public class SimpleTimer {

    public static void main(String[] args) {
        /*
         * 定时调度线程池,coreSize=3
         */
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(3);

        /*
         * 设置定时调度
         *
         * delay: 延时,period: 周期, time unit: 时间单位
         */
        executor.scheduleAtFixedRate(new MyRun("t-i2p4"), 2L, 4L, TimeUnit.SECONDS);
        executor.scheduleAtFixedRate(new MyRun("t-i4p8"), 4L, 8L, TimeUnit.SECONDS);
    }


    static class MyRun implements Runnable {

        private String threadName;

        MyRun(String threadName) {
            this.threadName = threadName;
        }

        private Supplier<String> dateSupplier = () -> {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            return sdf.format(new Date());
        };

        @Override
        public void run() {
            // do whatever you want to do
            System.out.println(dateSupplier.get() + " - " + threadName + " invoke");
        }
    }
}
```

测试结果

```java
2019-05-14 13:58:19 - t-i2p4 invoke
2019-05-14 13:58:21 - t-i4p8 invoke
2019-05-14 13:58:23 - t-i2p4 invoke
2019-05-14 13:58:27 - t-i2p4 invoke
2019-05-14 13:58:29 - t-i4p8 invoke
2019-05-14 13:58:31 - t-i2p4 invoke
```

---

## 7. 信号灯

在多线程里面,如果想在线程池再限制一下同时运行的线程数,那么就可以使用信号灯:`Semaphore`

```java
package com.base;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.*;
import java.util.function.Supplier;

/**
 * <p>
 *
 * @author cs12110 create at 2019-07-24 21:42
 * <p>
 * @since 1.0.0
 */
public class SemaphoreApp {

    /**
     * 一次运行多少条线程
     */
    private static final int HOW_MUCH_THREAD_ONE_TIME = 2;


    public static void main(String[] args) {
        // 声明信号灯对象
        final Semaphore semaphore = new Semaphore(HOW_MUCH_THREAD_ONE_TIME);

        // 设置线程池的coreSize = 4
        ExecutorService executorService = Executors.newFixedThreadPool(4);


        // 提交线程
        for (int index = 0; index < 10; index++) {
            executorService.submit(new MyRun(semaphore, "t" + index));
        }


        // 关闭线程池
        executorService.shutdown();
    }


    static class MyRun implements Runnable {

        private String threadName;
        private Semaphore semaphore;

        private Supplier<String> dateSupplier = () -> {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss,SSS");

            return sdf.format(new Date());
        };


        public MyRun(Semaphore s, String threadName) {
            this.semaphore = s;
            this.threadName = threadName;
        }

        @Override
        public void run() {
            try {
                // 获取到信号才执行
                semaphore.acquire();

                log("start");
                Thread.sleep(3000);
                log("end");

                // 执行完成释放信号
                semaphore.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        private void log(String message) {
            System.out.println(dateSupplier.get() + " " + threadName + " " + message);
        }
    }
}
```

执行结果

```java
2019-07-24 22:05:20,779 t1 start
2019-07-24 22:05:20,779 t0 start
2019-07-24 22:05:23,784 t0 end
2019-07-24 22:05:23,784 t1 end
2019-07-24 22:05:23,784 t2 start
2019-07-24 22:05:23,784 t4 start
2019-07-24 22:05:26,789 t2 end
2019-07-24 22:05:26,789 t4 end
2019-07-24 22:05:26,789 t3 start
2019-07-24 22:05:26,789 t7 start
2019-07-24 22:05:29,793 t7 end
2019-07-24 22:05:29,793 t3 end
2019-07-24 22:05:29,794 t5 start
2019-07-24 22:05:29,794 t6 start
2019-07-24 22:05:32,800 t5 end
2019-07-24 22:05:32,800 t6 end
2019-07-24 22:05:32,801 t8 start
2019-07-24 22:05:32,801 t9 start
2019-07-24 22:05:35,801 t9 end
2019-07-24 22:05:35,801 t8 end
```

如果把信号灯的代码去掉,执行结果如下

```java
2019-07-24 22:07:07,256 t0 start
2019-07-24 22:07:07,262 t1 start
2019-07-24 22:07:07,263 t3 start
2019-07-24 22:07:07,263 t2 start
2019-07-24 22:07:10,260 t0 end
2019-07-24 22:07:10,261 t4 start
2019-07-24 22:07:10,264 t1 end
2019-07-24 22:07:10,264 t3 end
2019-07-24 22:07:10,264 t5 start
2019-07-24 22:07:10,265 t2 end
2019-07-24 22:07:10,265 t7 start
2019-07-24 22:07:10,265 t6 start
2019-07-24 22:07:13,267 t4 end
2019-07-24 22:07:13,267 t8 start
2019-07-24 22:07:13,269 t7 end
2019-07-24 22:07:13,269 t6 end
2019-07-24 22:07:13,269 t5 end
2019-07-24 22:07:13,270 t9 start
2019-07-24 22:07:16,271 t8 end
2019-07-24 22:07:16,274 t9 end
```

所以,信号灯,还是有点用的,我想.

---

## 8. interrupt

Q: 这个是什么东西呀,怎么之前没听你提过?

A: 我没提过是因为我不会,但这个东西是非常重要的一个东西呀. :{

Q: 中断?是不是调用的时候,线程就会停止运行?

A: 当年,我也是这么认为的,然后落到现在这个地步. [线程 interrupt link](https://www.cnblogs.com/hapjin/p/5450779.html)

```java
public class ThreadInterruptTest {

    public static class MyThread extends Thread {
        @Override
        public void run() {
            do {
                if (this.isInterrupted()) {
                    display("loop, interrupt:" + true);
                    break;
                }
            } while (true);

            display("after loop1");
            display("after loop2");
        }

    }

    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start();

        display("before invoke interrupt" + thread.isInterrupted());
        thread.interrupt();
        display("after invoke interrupt:" + thread.isInterrupted());

        sleepWith(1000 * 100);
    }

    private static void display(String log) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss,SSS");
        System.out.println(sdf.format(new Date()) + "\t" + log);
    }

    private static void sleepWith(long mills) {
        try {
            Thread.sleep(mills);
        } catch (Exception ignore) {
            // do nothing
        }
    }
}
```

测试结果:

```java
2021-06-19 11:26:23,694	before invoke interruptfalse
2021-06-19 11:26:23,695	after invoke interrupt:true
2021-06-19 11:26:23,695	loop, interrupt:true
2021-06-19 11:26:23,695	after loop1
2021-06-19 11:26:23,695	after loop2
```

上面可以看出来: 虽然线程 interrupt 了,但是并不代表线程退出运行.所以 interrupt 并不因为这线程立刻停止.<u><span style="color: pink">所谓“中断一个线程”,其实并不是让一个线程停止运行,仅仅是将线程的中断标志设为 true, 或者在某些特定情况下抛出一个 InterruptedException,它并不会直接将一个线程停掉,在被中断的线程的角度看来,仅仅是自己的中断标志位被设为 true 了,或者自己所执行的代码中抛出了一个 InterruptedException 异常,仅此而已.</span></u>[线程中断 interrupt link](https://segmentfault.com/a/1190000016083002)

Q: 那么 interrupt 有什么作用呀?

```java
public static class MyThread extends Thread {
    @Override
    public void run() {
        try {
            do {
                if (this.isInterrupted()) {
                    display("loop index:" + ", interrupt:" + true);

                    // 必须捕抓该异常,那么该怎么通知上层???
                    throw new InterruptedException("make it worse");
                }
            } while (true);
        } catch (Exception e) {
            display(e.toString());

            // 只能这样子通知上层了,上层监听线程的interrupt状态
            Thread.currentThread().interrupt();
        }
    }
}
```

测试结果

```java
2021-06-19 11:33:15,554	before invoke interruptfalse
2021-06-19 11:33:15,555	after invoke interrupt:true
2021-06-19 11:33:15,555	loop index:, interrupt:true
2021-06-19 11:33:15,556	java.lang.InterruptedException: make it worse
```

---

## 9. 参考资料

a. [全面理解 Java 内存模型(JMM)及 volatile 关键字](https://blog.csdn.net/javazejian/article/details/72772461)

b. [深入理解 Java 并发之 synchronized 实现原理](https://blog.csdn.net/javazejian/article/details/72828483)

c. [线程的 interrupt link](https://www.cnblogs.com/hapjin/p/5450779.html)
