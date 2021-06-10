# 线程池

在开发里面,偶尔要用到线程,但并不提倡使用`new Thread().start()`这种方式来创建.

在 Jvm 里面创建过多的线程会对资源是一种很大的消耗,所以推荐使用线程池.

使用常识:<span style="color:pink">**线程池在没有特殊的情况下并不手动关闭线程池.不是在每一次调用线程池的时候打开,使用完就关闭,除非自己手动关闭 :{.**</span>

---

## 1. 测试线程

下列例子全部使用该线程来测试

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class MyRun implements Runnable {

	private String threadName;

	public MyRun(String threadName) {
		super();
		this.threadName = threadName;
	}

	@Override
	public void run() {
		System.out.println(getTime() + " - " + threadName + " is running");
		try {
			Thread.sleep(2000);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	private static String getTime() {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		return sdf.format(new Date());
	}
}
```

---

## 2. ThreadPoolExecutor

这个就是我们的奥黛丽.赫本了(主角).

注意等待队列是有边界还是没有边界的,同时也要注意等待队列的大小.

比如: `new LinkedBlockingQueue<>(1)`和`new LinkedBlockingQueue<>()`的区别.

```java
public class MyThreadPoolExecuotr {
	public static void main(String[] args) {
		ThreadPoolExecutor executor = new ThreadPoolExecutor(2, 5, 10, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1));

		executor.submit(new MyRun("t1"));
		executor.submit(new MyRun("t2"));
		executor.submit(new MyRun("t3"));
		executor.submit(new MyRun("t4"));
	}
}
```

线程池的构造方法如下

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

各个参数的作用

- corePoolsize: 线程池核心线程数
- maxinumPoolSize: 线程池最大线程数
- keepAliveTime 和 unit: 这个两个组合使用,表示空闲的线程(只回收>coresize 的线程)在多少时间后被回收,最多缩减为 coresize 数量
- workQueue: 等待队列的大小

工作流程:<span style="color:pink">**陆续创建核心线程数大小的线程 -> 消费不过了,放到等待队列里面 -> 队列满了 -> 扩充线程池线程数,最大为最大线程数 -> 队列满了(有边界的队列才会满 orz),池已经扩充到最大,还消费不过来-> 默认采取拒绝策略.**<span>

测试结果

```java
2018-10-24 09:14:19 - t1 is running
2018-10-24 09:14:19 - t4 is running
2018-10-24 09:14:19 - t2 is running
2018-10-24 09:14:21 - t3 is running
```

---

## 3. SingleThreadPool

这个就是单线程线程池了,线程池里面只会存在一条线程.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SingleThreadPool {

	public static void main(String[] args) {
		ExecutorService executor = Executors.newSingleThreadExecutor();

		executor.submit(new MyRun("t1"));
		executor.submit(new MyRun("t2"));
		executor.submit(new MyRun("t3"));
		executor.submit(new MyRun("t4"));
	}

}
```

测试结果:**可以看看线程依次被消费.**

```java
2018-10-24 08:50:41 - t1 is running
2018-10-24 08:50:43 - t2 is running
2018-10-24 08:50:45 - t3 is running
2018-10-24 08:50:47 - t4 is running
```

`Executors.newSingleThreadExecutor()`里面是什么?

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

就是`coresize=1`和`maxsize=1`的`ThreadPoolExectuor`.

---

## 4. CachedThreadPool

缓存线程池,如果消费不过来会不断的扩充池的容量来消费线程.空闲线程会在 1 分钟后被回收.

```java
public class CacheThreadPool {
	public static void main(String[] args) {
		ExecutorService executor = Executors.newCachedThreadPool();
		executor.submit(new MyRun("t1"));
		executor.submit(new MyRun("t2"));
		executor.submit(new MyRun("t3"));
		executor.submit(new MyRun("t4"));
	}
}
```

测试结果: 线程被并行执行了.

```java
2018-10-24 08:56:10 - t4 is running
2018-10-24 08:56:10 - t1 is running
2018-10-24 08:56:10 - t3 is running
2018-10-24 08:56:10 - t2 is running
```

内部实现,还是我们的老战友:`ThreadPoolExecutor`.只不过`maxsize=Integer.MAX_VALUE`了,就是不断的扩充线程来消费线程.

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```

---

## 5. FixedThreadPool

当然也有固定线程数的线程池啦.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedThreadPool {
	public static void main(String[] args) {
		ExecutorService executor = Executors.newFixedThreadPool(2);

		executor.submit(new MyRun("t1"));
		executor.submit(new MyRun("t2"));
		executor.submit(new MyRun("t3"));
		executor.submit(new MyRun("t4"));
	}
}
```

测试结果

```java
2018-10-24 10:44:46 - t1 is running
2018-10-24 10:44:46 - t2 is running
2018-10-24 10:44:48 - t3 is running
2018-10-24 10:44:48 - t4 is running
```

内部实现

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```

---

## 6. 自定义 ThreadFactory

在线程池里面可以自定义 ThreadFactory,定义线程 Factory 的名称可以让线程有更易识别的名称标识,你值得拥有.

自定义 ThreadFactory 如下

```java

import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/1/18 11:31
 * <p>
 * since: 1.0.0
 */
public class CustomThreadFactory implements ThreadFactory {

    /**
     * 前缀名称
     */
    private String prefixName;

    /**
     * 线程下标
     */
    private final AtomicInteger atomicInteger = new AtomicInteger(1);

    /**
     * 构建线程工厂
     *
     * @param prefixName 线程前缀名称
     */
    public CustomThreadFactory(String prefixName) {
        this.prefixName = prefixName;
    }

    /**
     * 构建线程
     *
     * @param r {@link Runnable}
     * @return Thread
     */
    @Override
    public Thread newThread(Runnable r) {
        // 构建线程名称
        String threadName = prefixName + "-t" + atomicInteger.getAndIncrement();
        return new Thread(r, threadName);
    }
}
```

构建线程池如下

```java
ExecutorService threadExecutor = new ThreadPoolExecutor(
		2,
		10,
		0,
		TimeUnit.SECONDS,
		new LinkedBlockingDeque<>(),
		new CustomThreadFactory("my-factory"));
```

提交任务之后,可以通过 JvisiualVM 查看线程名称. :"}

---

## 7. 线程池实现原理

Q: 那么线程池是怎么实现的呢?

A: 别问我!!!

`ThreadPoolExecutor`的源码继承关系如下

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
 //...
}
```

就拿`AbstractExecutorService#submit()`来看,我们看一下这些是怎么做到的.

```java
/**
 * @throws RejectedExecutionException {@inheritDoc}
 * @throws NullPointerException       {@inheritDoc}
 */
public Future<?> submit(Runnable task) {
	if (task == null) throw new NullPointerException();
	RunnableFuture<Void> ftask = newTaskFor(task, null);
	execute(ftask);
	return ftask;
}
```

经过`newTaskFor()`之后,还是到了`ThreadPoolExecutor#execute()`

```java
public void execute(Runnable command) {
if (command == null)
	throw new NullPointerException();
int c = ctl.get();

// 当前开启线程数<coresize
if (workerCountOf(c) < corePoolSize) {
	if (addWorker(command, true))
		return;
	c = ctl.get();
}
// 放置等待队列,workQueue为BlockingQueue
if (isRunning(c) && workQueue.offer(command)) {
	int recheck = ctl.get();
	if (! isRunning(recheck) && remove(command))
		reject(command);
	else if (workerCountOf(recheck) == 0)
		addWorker(null, false);
}
// 等待队列满了,扩大线程池,如果扩大失败则拒绝任务
else if (!addWorker(command, false))
	reject(command);
}
```

摘取`addWork`的重要代码

```java
Worker w = null;
try {
	w = new Worker(firstTask);
	final Thread t = w.thread;
	if (t != null) {
		final ReentrantLock mainLock = this.mainLock;
		mainLock.lock();
		try {
			int rs = runStateOf(ctl.get());

			if (rs < SHUTDOWN ||
				(rs == SHUTDOWN && firstTask == null)) {
				if (t.isAlive()) // precheck that t is startable
					throw new IllegalThreadStateException();
				// workers为全局的hashset
				workers.add(w);
				int s = workers.size();
				if (s > largestPoolSize)
					largestPoolSize = s;
				workerAdded = true;
			}
		} finally {
			mainLock.unlock();
		}
		// 这里面的 start调用的是Worker运行之后里面的run方法
		if (workerAdded) {
			t.start();
			workerStarted = true;
		}
	}
} finally {
	if (! workerStarted)
		addWorkerFailed(w);
}
```

我们来看看`Worker#run()`方法是做什么的.

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
	public void run() {
		runWorker(this);
	}

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
			// 拿到需要被执行的线程,保证自己先运行完然后才拿数据
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
						// 直接调用该线程的run方法,简单粗暴.
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
}
```

`getTask()`方法如下

```java
private Runnable getTask() {
	boolean timedOut = false; // Did the last poll() time out?

	for (;;) {
		int c = ctl.get();
		int rs = runStateOf(c);

		// Check if queue empty only if necessary.
		if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
			decrementWorkerCount();
			return null;
		}

		int wc = workerCountOf(c);

		// Are workers subject to culling?
		boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

		if ((wc > maximumPoolSize || (timed && timedOut))
			&& (wc > 1 || workQueue.isEmpty())) {
			if (compareAndDecrementWorkerCount(c))
				return null;
			continue;
		}
        //获取等待队列里面的线程
		try {
			Runnable r = timed ?
				workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
				workQueue.take();
			if (r != null)
				return r;
			timedOut = true;
		} catch (InterruptedException retry) {
			timedOut = false;
		}
	}
}
```

---

## 8. 扩展知识

前提: 一个线程池可以容纳`最大的线程数=队列的容量 + maxSize`.

```java
public static void main(String[] args) {
	ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 10, 10, TimeUnit.SECONDS,
		new LinkedBlockingQueue<>(), new ThreadFactory() {
		AtomicInteger atomicInteger = new AtomicInteger(1);

		@Override
		public Thread newThread(Runnable r) {
			return new Thread(r, "dynamic-pool-" + atomicInteger.getAndIncrement());
		}
	});

	threadPoolExecutor.execute(() -> {
		System.out.println("");
	});

	System.out.println("线程池CoreSize: " + threadPoolExecutor.getCorePoolSize());
	System.out.println("线程池MaxSize: " + threadPoolExecutor.getMaximumPoolSize());
	System.out.println("正在运行线程数: " + threadPoolExecutor.getActiveCount());
	System.out.println("已完成线程数: " + threadPoolExecutor.getCompletedTaskCount());
	System.out.println("等待线程数: " + threadPoolExecutor.getQueue().size());
}
```

打印日志

```java
线程池CoreSize: 1
线程池MaxSize: 10
正在运行线程数: 0
已完成线程数: 1
等待线程数: 0
```

Q: 那么我们可以动态修改线程池的 coreSize 和 maxSize 吗?

A: 答案是肯定的,但是如果维护是一个难点,现在还没想到什么好一点的策略.

```java
/* 
 * 把coreSize=1和maxSize=10的线程池调整为 coreSize=2和maxSize=20
 */
threadPoolExecutor.setCorePoolSize(2);
threadPoolExecutor.setMaximumPoolSize(20);
```

Q: 如果现在问一句: 怎么动态削减线程数,会不会过分?

A: 这其实是一个好问题. 如果削减数量<当前运行的线程数会不会出现异常,该怎么削减,怎么扩张,这些都是一个很好的问题.如果能回答的话,就可以解决动态维护线程池了.但我现在还做不到 ^_<

---

## 9. 总结

感觉受到了欺骗,这就是`ThreadPoolExecutor`的事,所以掌握`ThreadPoolExecutor`至关重要.
