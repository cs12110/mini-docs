# Java tips

提高 Java 性能的小 tips.

文档不能提供详细的解决方法,但吸收优秀代码,提供一种思路,Java 性能优化,是一门坑爹的艺术呀. 咧嘴笑.jpg

_文档不定时更新_

## 1. 字符串首字母修改为小写

**优化方法: 避免生成多余的对象(不仅字符串,系统优化也如是)**

在我们普通的做法里面为了方便,会使用字符串截取

代码类似:

```java
public static String changeFirstChar(boolean upper, String str) {
	if (null == str || "".equals(str.trim())) {
		return str;
	}
	String firstCharStr = str.substring(0, 1);
	if (upper) {
		firstCharStr = firstCharStr.toUpperCase();
	} else {
		firstCharStr = firstCharStr.toLowerCase();
	}
	return firstCharStr + str.substring(1);
}
```

在`Jodd`的工具类`StringUtil.java`里面的`changeFirstCharacterCase`里面,ta 们优化了这个字符串改变首字母的方法,代码如下:

```java
private static String changeFirstCharacterCase(boolean capitalize, String string) {
		int strLen = string.length();
		if (strLen == 0) {
			return string;
		}
		char ch = string.charAt(0);
		char modifiedCh;
		if (capitalize) {
			modifiedCh = Character.toUpperCase(ch);
		} else {
			modifiedCh = Character.toLowerCase(ch);
		}

		if (modifiedCh == ch) {
			return string;
		}

		char[] chars = string.toCharArray();
		chars[0] = modifiedCh;
		return new String(chars);
	}
```

测试结果如下,可以看出`Jodd`代码比自己写的字符串截取快了 3 倍左右(Jodd actually do a good job!).

```java
Test times: 1000000
normal spend time: 135
change spend time: 41
```

---

## 2. 使用设计模式

在项目中,有一个功能:校验一条日志,返回`LogCheckResult`对象

LogCheckResult 类代码如下

```java
public class LogCheckResult {
    /**
     * 默认成功检查类
     */
    public static LogCheckResult DEFAULT_SUCCESS_RESULT = new LogCheckResult(true, "", "");
    /**
     * 是否合法标志
     */
    private boolean isOk;
    /**
     * 日志字符串
     */
    private String log;
    /**
     * 错误信息
     */
    private String errMsg;

    public LogCheckResult(boolean isOk, String log, String errMsg) {
        super();
        this.isOk = isOk;
        this.log = log;
        this.errMsg = errMsg;
    }

    public boolean isOk() {
        return isOk;
    }

    public void setOk(boolean isOk) {
        this.isOk = isOk;
    }

    public String getLog() {
        return log;
    }

    public void setLog(String log) {
        this.log = log;
    }

    public String getErrMsg() {
        return errMsg;
    }

    public void setErrMsg(String errMsg) {
        this.errMsg = errMsg;
    }

    @Override
    public String toString() {
        return "{isOk:" + isOk + ", log:" + log + ", errMsg:" + errMsg + "}";
    }

}
```

如果日志成功,LogCheckResult 里面的 log 和 errMsg 属性为空

在项目中使用的时候,可以把 LogCheckResult 变成 Builder 模式,这样更简洁明了.

```java
public class LogCheckResult {
    /**
     * 默认成功检查类
     */
    public static LogCheckResult DEFAULT_SUCCESS_RESULT = new LogCheckResult.Builder(true).build();
    /**
     * 是否合法标志
     */
    private boolean isOk;
    /**
     * 日志字符串
     */
    private String log;
    /**
     * 错误信息
     */
    private String errMsg;

    private LogCheckResult(Builder builder) {
        this.isOk = builder.isOk;
        this.log = builder.log;
        this.errMsg = builder.errMsg;
    }

    public static class Builder {
        private boolean isOk;
        private String log;
        private String errMsg;

        public Builder(boolean isOk) {
            this.isOk = isOk;
        }
        public Builder setLog(String log) {
            this.log = log;
            return this;
        }
        public Builder setErrMsg(String errMsg) {
            this.errMsg = errMsg;
            return this;
        }
        public LogCheckResult build() {
            return new LogCheckResult(this);
        }
    }

    public boolean isOk() {
        return isOk;
    }
    public String getLog() {
        return log;
    }
    public String getErrMsg() {
        return errMsg;
    }
    @Override
    public String toString() {
        return "{isOk:" + isOk + ", log:" + log + ", errMsg:" + errMsg + "}";
    }
}
```

---

## 3. List 初始化

哈,List 里面的根本组成就是`Object[] values`.如果在初始化,不进行长度的控制,默认长度初始化为`10`,那么在 add 新的元素的时候,当数组长度达到(size+1>length)的时候,就会按照`length*1.5`倍来增长数组.也就是在默认初始化的情况下:

```java
import java.util.ArrayList;
import java.util.List;

public class LengthOfList {
	public static void main(String[] args) {
		List<Integer> list = new ArrayList<>();
		for (int index = 0; index < 11; index++) {
			list.add(index);
		}
		System.out.println(list.size());
	}
}
```

执行这个之后的数组长度为 15,有后面 4 个元素为`null`

如果使用`new ArrayList(int capacity)`来初始化,情况得以改善

```java
import java.util.ArrayList;
import java.util.List;

public class LengthOfList {
	public static void main(String[] args) {
		List<Integer> list = new ArrayList<>(11);
		for (int index = 0; index < 11; index++) {
			list.add(index);
		}
		System.out.println(list.size());
	}
}
```

执行之后,list 里面的 values 长度为 11,而不是 15.

可能 11 个长度太小没什么影响,那么请想象一下 100w 或者 100 个并发,每个并发使用 1w 长度的数组,这样子就省下不少空间了.

所以,在明确长度的情况下,请使用`new ArrayList(int capacity)`来构造 List.

---

## 4. 反射的简单优化

在 Java 里面,反射真的是一个有趣的家伙.当然这个优化一次执行可能就是快几百毫秒,几乎感觉不出来.如果在大循环里面,每次快几百毫秒,那么整一个就快了,好多,好多,好多了.

为加快 Java 的反射执行速度,而不整那么多幺娥子,那么我们做一个简单的优化.

把相关需要反射的东西,在一次执行后,缓存到内存中,下一次获取直接从缓存中获取.

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 反射工具类
 *
 * <p>
 *
 * @author cs1210 2018年3月1日下午4:11:35
 *
 */
public class ReflectUtil {

	/**
	 * 保存对象缓存Map
	 */
	private static final Map<String, Object> objectCache = new ConcurrentHashMap<String, Object>();

	/**
	 * 保存Class缓存Map
	 */
	private static final Map<String, Class<?>> classCache = new ConcurrentHashMap<String, Class<?>>();

	/**
	 * 实例化对象
	 *
	 * @param clazz
	 *            对象Class
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public static <T> T newInstance(Class<?> clazz) {
		Object target = objectCache.get(clazz.getName());
		try {
			if (null == target) {
				target = clazz.newInstance();
				objectCache.put(clazz.getName(), target);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return (T) target;
	}

	/**
	 * 实例化对象
	 * @param className 类名称(必须是完整的路径,如: x.y.z)
	 * @return
	 */
	public static <T> T get(String className) {
		try {
			Class<?> clazz = classCache.get(className);
			if (clazz == null) {
				clazz = Class.forName(className);
				classCache.put(className, clazz);
			}
			return newInstance(clazz);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}
}
```

测试代码

```java
/**
 * 测试代码
 *
 * <p>
 *
 * @author cs12110 2018年3月1日下午4:21:21
 *
 */
public class MyObj {

	private String id;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	public static void main(String[] args) {
		long start = System.currentTimeMillis();
		for (int index = 0; index < 1000000; index++) {
			MyObj object = ReflectUtil.get(MyObj.class.getName());
			object.getId();
		}
		long end = System.currentTimeMillis();
		System.out.println((end - start));
	}
}
```

在不使用缓存的情况下,执行耗时: 484ms

在使用缓存的情况下,执行耗时: 24 ms

可以看出,这个优化的效果还是很可观的,而且循环越大,缓存的效果越明显.

---

## 5. Class.forName 和 ClassLoader 的区别

在使用数据库连接的时候,一般会调用`Class.forName(driverName)`来注册数据库连接驱动.

`Class.forName`和`ClassLoader.loadClass`都是用来加载类的.它们之间有什么区别?

**主要区别: class.forName 加载的时候,会执行 class 里面的静态代码块代码.而 classloader 加载就不执行静态代码块里面的代码**

```java
/**
 * 静态代码块类
 *
 * <p>
 *
 * @author cs12110 2018年4月10日上午9:09:40
 *
 */
public class StaticBlockClazz {

	static {
		System.out.println("This is static method in StaticBlockClazz");
	}

	public StaticBlockClazz() {
		System.out.println("This is StaticBlockClazz()");
	}

}
```

测试类

```java
/**
 * 测试classLoader和class.forName的区别
 *
 * <p>
 *
 * @author cs12110 2018年4月10日上午9:11:17
 *
 */
public class JustLittleTest {

	public static void main(String[] args) {
		System.out.println("----- Start ------");
		classLoader(StaticBlockClazz.class.getName());
		classForName(StaticBlockClazz.class.getName());
		System.out.println("----- End ------");

	}

	private static void classLoader(String clazz) {
		try {
			ClassLoader loader = JustLittleTest.class.getClassLoader();
			loader.loadClass(clazz);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	private static void classForName(String clazz) {
		try {
			Class.forName(clazz);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

首先测试`ClassLoader`

```java
----- Start ------
----- End ------
```

测试`class.forName`

```
----- Start ------
This is static method in StaticBlockClazz
----- End ------
```

上面的结论是通过的,那么为什么导致这样子?

据说在调用`newInstance()`的方法,生成对象的时候才会执行静态代码块里面的东西.而`ClassLoader`仅仅是把`.class`文件加载到 jvm 而没有执行`newInstance()`的操作,而`Class.forName`是会默认调用`newInstance()`的操作,具体还需要去看**Jdk**代码才清楚

---

## 6. `Function<K,V>`的使用

在`Jodd`的项目代码里面看见了一个使用`Function<K,V>`的地方,觉得很好奇.所以测试了一下.

适用场景: map 里面根据 key 查找,如果该不存在的情况下,会创建新的 value 存入 map 里面

```java
public class FunctionTest {
	public static void main(String[] args) {
		Map<String, MyClazz> map = new HashMap<String, MyClazz>();
		map.put("1", new MyClazz("1"));
		map.put("2", new MyClazz("2"));

		/*
		 * Function的使用
		 */
		Function<String, MyClazz> fun = (n -> new MyClazz(n));
		/*
		 * 新建id为4的MyCalzz,并放入map
		 */
		MyClazz clazz = map.computeIfAbsent("4", fun);

		System.out.println(clazz);
		System.out.println(map.size());
	}

}

class MyClazz {
	private String id;
	public MyClazz(String id) {
		super();
		this.id = id;
	}
	public String getId() {
		return id;
	}
	public void setId(String id) {
		this.id = id;
	}
	@Override
	public String toString() {
		return "MyClazz [id=" + id + "]";
	}
}
```

---

## 7. 运行栈信息的使用

还是来自`Jodd`里面的日志模块,这个项目太牛掰了.

```java
/**
 * 获取运行信息
 *
 * <p>
 *
 * @author cs12110 2018年4月20日下午6:24:28
 *
 */
class StackInfo {

	/**
	 * 返回调用这个方法class字符串
	 *
	 * 格式为: className(被裁剪).方法名称:行号
	 *
	 * @return
	 */
	protected String getCallerClass() {
		Exception exception = new Exception();
		StackTraceElement[] stackTrace = exception.getStackTrace();
		for (StackTraceElement stackTraceElement : stackTrace) {
			String className = stackTraceElement.getClassName();
			if (className.equals(this.getClass().getName())) {
				continue;
			}
			return shortenClassName(className) + '.' + stackTraceElement.getMethodName() + ':'
					+ stackTraceElement.getLineNumber();
		}
		return "N/A";
	}

	/**
	 * 获取类名称
	 *
	 * 类名前面的包名,只保留第一个字母
	 *
	 * @param className
	 *            className
	 * @return String
	 */
	protected String shortenClassName(final String className) {
		int lastDotIndex = className.lastIndexOf('.');
		if (lastDotIndex == -1) {
			return className;
		}
		StringBuilder shortClassName = new StringBuilder(className.length());
		int start = 0;
		while (true) {
			shortClassName.append(className.charAt(start));
			int next = className.indexOf('.', start);
			if (next == lastDotIndex) {
				break;
			}
			start = next + 1;
			shortClassName.append('.');
		}
		shortClassName.append(className.substring(lastDotIndex));
		return shortClassName.toString();
	}
}
```

测试类:

```java
public class TestGithub {
	public static void main(String[] args) {
		StackInfo git = new StackInfo();
		System.out.println(git.getCallerClass());
	}
}
```

测试结果:

```java
t.TestGithub.main:7
```

---

## 8. 内省

在 JAVA 里面,可以使用反射获取到`class` 里面的信息.除了使用反射,我们这里提供另外一种方式获取`class`里面的各种属性.这个就是: `Introspector`[,ɪntrə(ʊ)'spektər]内省

```java
import java.beans.BeanInfo;
import java.beans.Introspector;
import java.beans.MethodDescriptor;
import java.beans.PropertyDescriptor;

/**
 * 内省
 *
 * <p>
 *
 * @author cs12110 2018年4月25日上午11:28:58
 *
 */
public class MyIntrospector {
	public static void main(String[] args) {
		getMethodDescriptor(MyBean.class);
		getPropertiesDescriptor(MyBean.class);
	}

	/**
	 * 获取方法描述
	 *
	 * @param clazz
	 *            方法描述
	 */
	private static void getMethodDescriptor(Class<?> clazz) {
		try {
			/*
			 * java内置内省器
			 */
			BeanInfo beanInfo = Introspector.getBeanInfo(clazz);
			MethodDescriptor[] methodDescArr = beanInfo.getMethodDescriptors();

			for (MethodDescriptor mtd : methodDescArr) {
				System.out.println(mtd);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

	/**
	 * 获取类里面声明的属性(会存在一个class的属性,混进了不得了的东西???)
	 *
	 * @param clazz
	 *            class对象
	 */
	private static void getPropertiesDescriptor(Class<?> clazz) {
		try {
			/*
			 * java内置内省器
			 */
			BeanInfo beanInfo = Introspector.getBeanInfo(clazz);
			PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();

			for (PropertyDescriptor pdsc : propertyDescriptors) {
				System.out.println("\n");
				System.out.println(pdsc);
			}
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

}

class Bean {
	private Integer id;
	public Integer getId() {
		return id;
	}
	public void setId(Integer id) {
		this.id = id;
	}
}

class MyBean extends Bean {
	private String beanName;
	private String value;
	public String getBeanName() {
		return beanName;
	}
	public void setBeanName(String beanName) {
		this.beanName = beanName;
	}
	public String getValue() {
		return value;
	}
	public void setValue(String value) {
		this.value = value;
	}
}
```

测试结果

```java
java.beans.MethodDescriptor[name=getClass; method=public final native java.lang.Class java.lang.Object.getClass()]
java.beans.MethodDescriptor[name=setBeanName; method=public void cn.rojao.def.MyBean.setBeanName(java.lang.String)]
java.beans.MethodDescriptor[name=setId; method=public void cn.rojao.def.Bean.setId(java.lang.Integer)]
java.beans.MethodDescriptor[name=wait; method=public final void java.lang.Object.wait() throws java.lang.InterruptedException]
java.beans.MethodDescriptor[name=notifyAll; method=public final native void java.lang.Object.notifyAll()]
java.beans.MethodDescriptor[name=getId; method=public java.lang.Integer cn.rojao.def.Bean.getId()]
java.beans.MethodDescriptor[name=getBeanName; method=public java.lang.String cn.rojao.def.MyBean.getBeanName()]
java.beans.MethodDescriptor[name=notify; method=public final native void java.lang.Object.notify()]
java.beans.MethodDescriptor[name=wait; method=public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException]
java.beans.MethodDescriptor[name=hashCode; method=public native int java.lang.Object.hashCode()]
java.beans.MethodDescriptor[name=getValue; method=public java.lang.String cn.rojao.def.MyBean.getValue()]
java.beans.MethodDescriptor[name=wait; method=public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException]
java.beans.MethodDescriptor[name=equals; method=public boolean java.lang.Object.equals(java.lang.Object)]
java.beans.MethodDescriptor[name=setValue; method=public void cn.rojao.def.MyBean.setValue(java.lang.String)]
java.beans.MethodDescriptor[name=toString; method=public java.lang.String java.lang.Object.toString()]

java.beans.PropertyDescriptor[name=beanName; propertyType=class java.lang.String; readMethod=public java.lang.String cn.rojao.def.MyBean.getBeanName(); writeMethod=public void cn.rojao.def.MyBean.setBeanName(java.lang.String)]

java.beans.PropertyDescriptor[name=class; propertyType=class java.lang.Class; readMethod=public final native java.lang.Class java.lang.Object.getClass()]

java.beans.PropertyDescriptor[name=id; propertyType=class java.lang.Integer; readMethod=public java.lang.Integer cn.rojao.def.Bean.getId(); writeMethod=public void cn.rojao.def.Bean.setId(java.lang.Integer)]

java.beans.PropertyDescriptor[name=value; propertyType=class java.lang.String; readMethod=public java.lang.String cn.rojao.def.MyBean.getValue(); writeMethod=public void cn.rojao.def.MyBean.setValue(java.lang.String)]
```

结论: 可以获取到类里面的各种东西.

---

## 9. 线程池的使用

在现实场景中,有些地方需要用到多线程,但不推荐使用 new Thread 这种方法来创建新的线程,线程多的时候,JVM 在维护线程和 CPU 切换上面都要耗费大量的资源.所以推荐使用线程池.

**线程池在使用中不用关闭,在应用开启的时候初始化,然后到应用结束没特殊情况都不应手动关闭.**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 线程池
 *
 * @author root
 *
 */
public class ThreadPoolUtil {

	/**
	 * 线程池
	 *
	 * 线程提交大于>coresize的时候,放到queue里面,队列满的话,开启新的线程来处理,最大线程数为maxsize,超过这个数量的时候,
	 * 采用拒绝策略
	 *
	 */
	private final static ExecutorService POOL = new ThreadPoolExecutor(2, 8, 10, TimeUnit.SECONDS,
			new ArrayBlockingQueue<>(1024));

	/**
	 * 提交线程多条线程
	 *
	 * @param taskList
	 *            线程组
	 * @return List
	 */
	public static List<Future<?>> batch(List<Runnable> taskList) {
		List<Future<?>> futureList = new ArrayList<>(taskList.size());
		for(Runnable each: taskList){
			futureList.add(submit(each));
		}
		return futureList;
	}

	/**
	 * 提交任务
	 *
	 * @param task
	 *            任务
	 * @return {@link Future}
	 */
	public static Future<?> submit(Runnable task) {
		return POOL.submit(task);
	}

}
```

测试类如下

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Future;

public class TestThreadPool {

	public static void main(String[] args) {
		int taskNum = 5;
		List<Runnable> list = new ArrayList<>(taskNum);
		for (int index = 0; index < taskNum; index++) {
			list.add(new MyRun(index));
		}

		List<Future<?>> futureList = ThreadPoolUtil.batch(list);
		for (Future<?> f : futureList) {
			try {
				f.get();
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		System.out.println("After all the task done, you can do anything you want.");
	}

}

class MyRun implements Runnable {

	private int threadId;

	public MyRun(int threadId) {
		this.threadId = threadId;
	}

	@Override
	public void run() {
		System.out.println(this.getClass() + "[" + threadId + "] is running");
	}
}
```

测试结果

```java
class issue.pool.MyRun[1] is running
class issue.pool.MyRun[0] is running
class issue.pool.MyRun[2] is running
class issue.pool.MyRun[3] is running
class issue.pool.MyRun[4] is running
After all the task done, you can do anything you want.
```

---

## 10.数据导入优化

在生产环境里面,有一个需求: 用户界面导入文件,按行读取文件内容,并把内容插入数据库.

在系统里面使用的方式是: 使用`IO`读取内容,每一行进行处理插入数据.

优化版本 1: 使用`jdbc`批处理功能,减少数据在网络上传输的时间.

优化版本 2: 采用队列方法,一个线程负责读取文件内容放置队里,开启其他线程处理队列里面的内容+线程使用`jdbc`批处理.

测试文本:`all.txt`,为 100 行的数据文本.

### 10.1 普通处理

```java
package com.pkgs;

import java.io.File;
import java.io.RandomAccessFile;

public class Simple {

	public static void main(String[] args) {
		Simple simple = new Simple();
		long start = System.currentTimeMillis();
		try {
			File file = new File("d://all.txt");
			RandomAccessFile access = new RandomAccessFile(file, "r");
			String line = null;
			while (null != (line = access.readLine())) {
				simple.processLine(line);
			}
			access.close();

		} catch (Exception e) {
			e.printStackTrace();
		}
		long end = System.currentTimeMillis();
		System.out.println(Simple.class + " spend times: " + (end - start));
	}

	private void processLine(String line) {
		try {
			// 每条数据处理,大概耗时5毫秒
			Thread.sleep(5);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

测试结果

```java
class com.pkgs.Simple spend times: 504
```

### 10.2 优化版本代码

自定义队列代码

```java

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * 队列中心
 *
 *
 * <p>
 *
 * @author cs12110 2018年11月15日
 * @see
 * @since 1.0
 */
public class MemQueue {

	/**
	 * 队列
	 */
	private static final BlockingQueue<Object> BLOCK_QUEUE = new LinkedBlockingQueue<>();

	/**
	 * 结束标志
	 */
	private static final String IS_DONE_VALUE = "$$%%^^**&&##++--//??@@!!";

	/**
	 * 新增消息
	 *
	 * @param value
	 *            值
	 */
	public static void put(Object value) {
		try {
			BLOCK_QUEUE.put(value);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	/**
	 * 消费消息
	 *
	 * @return object
	 */
	public static Object pop() {
		return BLOCK_QUEUE.poll();
	}

	/**
	 * 设置All done标志
	 */
	public static void allDone() {
		put(IS_DONE_VALUE);
	}

	/**
	 * 判断是否结束
	 *
	 * @param value
	 *            值
	 * @return boolean
	 */
	public static boolean itIsDone(Object value) {
		return IS_DONE_VALUE.equals(String.valueOf(value));
	}

}
```

```java

import java.io.File;
import java.io.RandomAccessFile;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import test.MemQueue;

public class Optimize {

	private final static int THREAD_NUM = 5;

	private final static ExecutorService service = Executors.newFixedThreadPool(THREAD_NUM);
	/**
	 * 消息全部放置完成
	 */
	private static volatile boolean isAllDone = false;

	public static void startup(String[] args) {
		long start = System.currentTimeMillis();
		service.submit(new MyReader());
		for (int index = 0, size = THREAD_NUM - 1; index < size; index++) {
			service.submit(new MyConsumer("t" + index));
		}

		while (!isAllDone) {
		}
		long end = System.currentTimeMillis();

		System.out.println("Optimize spend: " + (end - start));
	}

	static class MyReader implements Runnable {
		@Override
		public void run() {
			System.out.println("Start running: " + this);
			try {
				File file = new File("d://all.txt");
				RandomAccessFile access = new RandomAccessFile(file, "r");
				String line = null;
				while (null != (line = access.readLine())) {
					MemQueue.put(line);
				}
				access.close();
				System.out.println("It's done: " + this);
			} catch (Exception e) {
				e.printStackTrace();
			} finally {
				MemQueue.allDone();
			}
		}
	}

	/**
	 * 消费者
	 */
	static class MyConsumer implements Runnable {
		private String threadName;
		private int num = 0;

		public MyConsumer(String threadName) {
			super();
			this.threadName = threadName;
		}

		@Override
		public void run() {
			long start = System.currentTimeMillis();
			while (!isAllDone) {
				Object msg = MemQueue.pop();
				if (MemQueue.itIsDone(msg)) {
					isAllDone = true;
					break;
				}
				if (null != msg) {
					processLine(String.valueOf(msg));
					num++;
				}
			}
			long end = System.currentTimeMillis();
			System.out.println(threadName + " using: " + num + " , spend times: " + (end - start));
		}

		private void processLine(String line) {
			try {
				// 每条数据处理,大概耗时5毫秒
				Thread.sleep(5);
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

}
```

测试结果

```java
Start running: test.per.Optimize$MyReader@24d2263f
It's done: test.per.Optimize$MyReader@24d2263f
t0 using: 25 , spend times: 127
Optimize spend: 140
Optimize is done
t2 using: 24 , spend times: 128
t1 using: 26 , spend times: 131
t3 using: 25 , spend times: 125
```

总结:在上述的测试数据里面,该方法能提高数据的处理速度(差不多 5 倍),但也消耗更大的资源和提高了程序的复杂性.要怎么使用,请参考具体生成环境.

---

## 11. 接口设计

至今,都还记得,这种疼.

狗日的接口设置,太膨胀了,这怪谁?当然怪自己了.

在之前项目的接口里面,有如下接口

```java
BusVod selectById(Integer id);

BusVod selectByCode(String code);

BusVod findOneById(Integer id);
```

这三个接口可以合成一个接口的. :{

```java
BusVod selectOne(BusVod search);
```

下次要记得了,含泪点赞.

---

## 12. 时间处理

在 Java 里面经常使用`SimpleDateFormat`来做时间的格式化,但这个东西线程不安全.

下面这代码有异常的. :"}

```java
SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

Callable<Date> task = new Callable<Date>() {
    public Date call() throws Exception {
        return dateFormat.parse("2018-08-14 15:00:00");
    }
};

ExecutorService exec = Executors.newFixedThreadPool(5);
List<Future<Date>> results = new ArrayList<Future<Date>>();
for (int i = 0; i < 10; i++) {
    results.add(exec.submit(task));
}
//exec.shutdown();
// 输出结果
for (Future<Date> result : results) {
    try {
        System.out.println(result.get());
    } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    } catch (ExecutionException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
}
```

有两种解决方法: 第一个是每一条线程都 new 一个 SimpleDateFormat 来玩.

另一种是使用 Calendar.

```java
Calendar instance = Calendar.getInstance();
StringBuilder date = new StringBuilder();
date.append(instance.get(Calendar.YEAR)).append("-");

/*
 * 0为1月,所以+1
 */
int month = Calendar.MONTH + 1;
date.append(month < 10 ? "0" + month : month).append("-");

int day = instance.get(Calendar.DAY_OF_MONTH);
date.append(day < 10 ? "0" + day : day).append(" ");

date.append(instance.get(Calendar.HOUR_OF_DAY)).append(":");
date.append(instance.get(Calendar.MINUTE)).append(":");
date.append(instance.get(Calendar.SECOND));

return date.toString();
```

代码增加了一点,但在线程安全的情况下,Calendar 还是比 SimpleDateFormat 快一点.

---

## 13. 动态代理

在日常的性能测试里面,有很多需要计算某个方法的执行耗时.

那么我们可以使用动态代理了.

```java
package issue.pool;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

/**
 * Stopwatch
 *
 * @author root
 *
 */
public class StopwatchAop implements InvocationHandler {
	/**
	 * 本地缓存
	 */
	private static Map<String, Object> cache = new HashMap<String, Object>();

	/**
	 * 创建动态代理对象
	 *
	 * @param clazz 对象
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public static <T> T wrapper(Class<T> clazz) {
		if (null == clazz) {
			return null;
		}
		String key = clazz.getName();
		Object value = cache.get(key);
		if (value != null) {
			return (T) value;
		} else {
			T instance = null;
			try {
				instance = clazz.newInstance();
			} catch (Exception e) {
				e.printStackTrace();
			}
			Object proxy = Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(),
					new StopwatchAop(instance));

			cache.put(key, proxy);
			return (T) proxy;
		}
	}

	private Object target;

	private StopwatchAop(Object target) {
		super();
		this.target = target;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

		long start = System.currentTimeMillis();

		Object result = method.invoke(target, args);
		long end = System.currentTimeMillis();

		System.out.println(buildLog(method, start, end));

		return result;
	}

	/**
	 * 创建日志
	 *
	 * @param m     method
	 * @param start 开始时间
	 * @param end   结束时间
	 * @return String
	 */
	private String buildLog(Method m, long start, long end) {
		long spend = end - start;

		String methodName = m.getName();
		String className = m.getDeclaringClass().getName();

		StringBuilder log = new StringBuilder();
		log.append(className).append("#").append(methodName);
		log.append(" - ");
		log.append("spend times: ").append(spend);
		return log.toString();
	}

}
```

---

## 14. StringBuilder 的使用

在字符串的连接里面如果不涉及线程安全,那么 StringBuilder 你值得拥有.

但在使用中还可以有一点小小的优化,如下:

如果字符串确定长度的话,也请使用 `new StringBuilder(capacity)`

```java
public class StrBuilder {
	public static void main(String[] args) {
		before();
		after();
	}
	/**
	 * 循环的每一次都生成了一个StringBuilder对象
	 */
	private static void before() {
		System.out.println("--- Before ---");
		for (int index = 0; index < 5; index++) {
			StringBuilder builder = new StringBuilder();
			builder.append("this is").append(index);
			System.out.println(builder);
		}
	}

	/**
	 * 只有一个StringBuilder对象
	 */
	private static void after() {
		System.out.println("--- After ---");
		StringBuilder builder = new StringBuilder();
		for (int index = 0; index < 5; index++) {
			builder.setLength(0);
			builder.append("this is").append(index);
			System.out.println(builder);
		}
	}

}
```

---

## 15. 字符串截取

在现实生产环境里面,经常遇到要到某个字符串进行截取的情况.一般也是使用`split()`,但如果是简单的获取,还有更好的做法.

普通做法如下:

```java
@Test
public void testArr() throws Exception {
	String str = "a,b,c,d,e";
	long start = System.currentTimeMillis();
	String[] arr = null;
	for (int index = 0; index < 100000; index++) {
		arr = str.split(",");
		String a = arr[0];
		// doSomething with a
	}

	long end = System.currentTimeMillis();

	System.out.println("Arr spend: " + (end - start));
}
```

简单优化:

```java
@Test
public void testSub() throws Exception {
	String str = "a,b,c,d,e";
	long start = System.currentTimeMillis();

	int buond = 0;
	for (int index = 0; index < 100000; index++) {
		buond = str.indexOf(",");
		if (buond != -1) {
			String a = str.substring(0, buond);
			// doSomething with a
		}
	}

	long end = System.currentTimeMillis();

	System.out.println("Sub spend: " + (end - start));
}
```

测试结果

```java
Arr spend: 565
Sub spend: 53
```

结论: 字符串,还真的能折腾不少东西. :{

---

## 16. 快速获取所有子文件路径

自己需要获取某个文件夹下面的所有子文件的路径,但是使用单线程递归获取耗时,有点久.

So,下面是一个使用空间换取时间的例子.

```java
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

/**
 * 快速查找文件夹下的子文件夹
 *
 * <p>
 *
 * @author hhp 2018年9月29日
 * @see
 * @since 1.0
 */
public class QuicklyFileSeeker {

	/**
	 * 查找路径
	 */
	private String path;

	/**
	 *
	 * @param path
	 *            路径
	 */
	public QuicklyFileSeeker(String path) {
		this.path = path;
	}

	/**
	 * 返回该文件
	 *
	 * @return
	 */
	public List<String> seek() {
		List<String> store = new ArrayList<>();
		try {
			ForkJoinPool pool = new ForkJoinPool();
			pool.submit(new MyForkJoinSeeker(store, path));

			// 等待子线程执行完成
			while (!pool.isQuiescent()) {
			}
			pool.shutdown();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return store;
	}

	/**
	 * 查找子文件线程
	 *
	 * @since 1.0
	 */
	class MyForkJoinSeeker extends RecursiveAction {

		private static final long serialVersionUID = 1L;

		private String path;
		private List<String> store;

		/**
		 * 初始化
		 *
		 * @param store
		 *            存放列表
		 * @param path
		 *            查找路径
		 */
		public MyForkJoinSeeker(List<String> store, String path) {
			super();
			this.store = store;
			this.path = path;
		}

		@Override
		protected void compute() {
			File file = new File(path);
			if (file != null) {
				if (file.isFile()) {
					store.add(file.getAbsolutePath());
				} else {
					File[] listFiles = file.listFiles();
					if (null != listFiles) {
						for (File f : listFiles) {
							if (f.isDirectory()) {
								// 开启子线程递归
								invokeAll(new MyForkJoinSeeker(store, f.getAbsolutePath()));
							} else {
								store.add(f.getAbsolutePath());
							}
						}
					}
				}
			}
		}
	}

}
```

---

## 17. 加载外部 jar

生产场景:项目 a 提交接口给项目 b -> 项目 b 实现这个接口后打包成 jar -> 放置特定文件夹 -> 项目 a 自定加载使用项目 b 实现的接口.

解决方案:使用自定义的`classloader`来加载实体类,然后实例化.

**实现自定义 classloader**

```java
import java.net.URL;
import java.net.URLClassLoader;

/**
 * URL classloder
 *
 *
 * <p>
 *
 * @author hhp 2018年10月16日
 * @see
 * @since 1.0
 */
public class PluginClassLoader extends URLClassLoader {

	/**
	 * URL
	 *
	 * @param urls
	 */
	public PluginClassLoader(URL[] urls) {
		super(urls);
	}

	public PluginClassLoader(URL[] urls, ClassLoader parent) {
		super(urls, parent);
	}

	/**
	 * 新增jar包
	 *
	 * @param url
	 *            jar的url地址
	 */
	public void addJar(URL url) {
		this.addURL(url);
	}
}
```

实现`InstanceBuilder`

```java
import java.io.File;
import java.net.URL;

/**
 * 获取实体类
 *
 *
 * <p>
 *
 * @author hhp 2018年10月16日
 * @see
 * @since 1.0
 */
public class InstanceBuilder {

	private File file = null;

	private static PluginClassLoader pluginClassLoader = null;

	static {
		initClassLoader();
	}

	/**
	 * 构造方法
	 *
	 * @param jarPath
	 *            jar文件路径,为绝对路径,如:<code>/home/dev/biplugin-jars/outside.jar</code>
	 * @param className
	 *            加载类名称,如<code>com.plugin.BiSourceImpl</code>
	 */
	public InstanceBuilder(String jarPath) {
		if (pluginClassLoader == null) {
			initClassLoader();
		}
		file = new File(jarPath);
		if (!isLegalJarFile()) {
			throw new RuntimeException("Jar[" + jarPath + "] isn't a legal jar file.");
		}
	}

	/**
	 * 构造PluginClassLoader
	 */
	public static void initClassLoader() {
		URL[] urls = new URL[]{};
		pluginClassLoader = new PluginClassLoader(urls);
	}

	/**
	 * 判断Jar文件是否合法
	 *
	 * @return boolean
	 */
	private boolean isLegalJarFile() {
		if (!file.exists()) {
			return false;
		}
		if (!file.isFile()) {
			return false;
		}
		if (!file.getName().endsWith(".jar")) {
			return false;
		}
		return true;
	}

	/**
	 * 创建实体类
	 *
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public <T> T get(String className) {
		try {
			pluginClassLoader.addJar(file.toURI().toURL());
			Class<?> clazz = pluginClassLoader.loadClass(className);
			return (T) clazz.newInstance();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}
}
```

**测试类**

```java
import com.func.BiSource;
public class SuperInvoke {
	public static void main(String[] args) {
		String jarPath = "D:\\Mydoc\\biplugin\\biplugin-outside.jar";
		String className = "com.plugin.BiSourceImpl";

		InstanceBuilder builder = new InstanceBuilder(jarPath);
		BiSource target = builder.get(className);
		target.say("窝草");
	}
}
```

```java
窝草
class com.plugin.BiSourceImpl say: 窝草
```
