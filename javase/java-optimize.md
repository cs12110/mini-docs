# Java tips

提高 Java 性能的小 tips.

Java 性能优化, 是一门坑爹的艺术呀. 咧嘴笑.jpg

_文档不定时更新_

---

## 1. 字符串首字母修改为小写

**优化方法: 避免生成多余的对象(不仅字符串,系统优化也如是)**

普通的做法使用`str.substring()`,大概如下:

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

`Jodd`的工具类`StringUtil#changeFirstCharacterCase`里面,ta 们是这么做的:

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

测试结果如下,`Jodd`代码字符串截取快了 3 倍左右 (Jodd actually do a good job!).

```java
Test times: 1000000
normal spend time: 135
change spend time: 41
```

---

## 2. StringBuilder 的使用

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

## 3. 字符串截取

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

## 4. List&Map 初始化

Q: 常见的 List 和 Map,有什么好写的?

A: 那你用对了吗?

### 4.1 List 初始化

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

### 4.2 Map 初始化

前提: <u>HashMap 默认初始化为 16 个节点,每个节点是链表或者 tree,当节点的链表的长度>=8 时,演化成 tree.当节点长度>16\*0.75(12)的时候,开始扩充节点数(resize)为 2 倍(newCap = oldCap << 1).</u>

平常使用里面,一般都是

```java
Map<String, Object> map = new HashMap<>(4);
map.put("k1", 1);
map.put("k2", 1);
map.put("k3", 1);
map.put("k4", 1);
```

但这里有一点小小的问题,在 map 初始化的时候声明长度的时候,调用`HashMap#tableSize`获取应该初始化的节点长度(必须为 2 的幂).

比如是 4 的话,则返回 4,如果是 6 的话,返回 8,如果是 80 的话,返回 124,如此类推.

```java
/**
* Returns a power of two size for the given target capacity.
*/
static final int tableSizeFor(int cap) {
	int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	System.out.println("n:" + n);
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

Q: 那么上面那个`new HashMap<>(4)`问题出现在哪里?

A: 当 put 进第 4 个元素的时候(4>=4\*0.75)就要 resize 了,那么和减少无用的空间浪费是违背的.

Q: 那么该怎么改进?

A: 声明的时候,同时声明 loadFactor. 比如:`new HashMap(4,1)`.或者直接不声明 capacity 大小,全都交给 java 默认来做. 笑哭脸.jpg

---

## 5. 反射的简单优化

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

## 6. Class.forName 和 ClassLoader 的区别

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
	// 省略getter/setter
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

## 9.数据导入优化

需求: 用户界面导入文件,按行读取文件内容,并把内容插入数据库.

- 普通模式: 使用`IO`读取内容,对每一行数据进行处理插入数据.
- 优化版本 1: 使用`jdbc`批处理功能,减少数据在网络上传输的时间.
- 优化版本 2: 采用队列方法,一个线程负责读取文件内容放置队里,开启其他线程处理队列里面的内容+`jdbc`批处理.

测试文本:`all.txt`,为 100 行的数据文本.

### 9.1 普通处理

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

### 9.2 优化版本代码

自定义队列代码

```java
package com.pkgs;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * TODO: 本地线程
 *
 * @author cs12110 create at: 2019/1/20 20:53
 * Since: 1.0.0
 */
public class MemQueue {
    /**
     * 队列
     */
    private static final BlockingQueue<Object> BLOCK_QUEUE = new LinkedBlockingQueue<>();

    /**
     * 结束标志
     */
    private static volatile boolean isFinish = false;

    /**
     * 新增消息
     *
     * @param value 值
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

    public static boolean isIsFinish() {
        return isFinish;
    }

    public static void setIsFinish(boolean isFinish) {
        MemQueue.isFinish = isFinish;
    }
}
```

```java
package com.pkgs;

import java.io.File;
import java.io.RandomAccessFile;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * TODO: 文件导入优化
 *
 * @author cs12110 create at: 2019/1/20 20:56
 * Since: 1.0.0
 */
public class Optimize {

    private final static int THREAD_NUM = 5;

    private final static ExecutorService service = Executors.newFixedThreadPool(THREAD_NUM);

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        service.submit(new MyReader());

        /*
         *使用countdownLatch监测线程池里面的子线程是否全部执行完成
         */
        CountDownLatch latch = new CountDownLatch(THREAD_NUM);
        for (int index = 0; index < THREAD_NUM; index++) {
            service.submit(new MyConsumer(latch, "t" + index));
        }

        try {
            latch.await();
        } catch (Exception e) {
            //do nothing
        }

        long end = System.currentTimeMillis();


        System.out.println("Optimize spend: " + (end - start));
    }

    /**
     * 读取文件内容到queue
     */
    static class MyReader implements Runnable {
        @Override
        public void run() {
            System.out.println("Provider is running");
            try {
                File file = new File("d://all.txt");
                RandomAccessFile access = new RandomAccessFile(file, "r");
                String line;
                while (null != (line = access.readLine())) {
                    MemQueue.put(line);
                }
                access.close();
                System.out.println("Provider is done");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                MemQueue.setIsFinish(true);
            }
        }
    }

    /**
     * 消费者,消费queue消息
     */
    static class MyConsumer implements Runnable {
        private CountDownLatch latch;
        private String threadName;

        public MyConsumer(CountDownLatch latch, String threadName) {
            super();
            this.latch = latch;
            this.threadName = threadName;
        }

        @Override
        public void run() {
            long start = System.currentTimeMillis();
            int num = 0;
            while (true) {
                Object msg = MemQueue.pop();
                if (msg == null && MemQueue.isIsFinish()) {
                    break;
                }
                if (null != msg) {
                    processLine(String.valueOf(msg));
                    num++;
                }
            }
            long end = System.currentTimeMillis();
            System.out.println(threadName + "is done, get msg: " + num + ",spend times: " + (end - start));
            latch.countDown();
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
Provider is running
Provider is done
t1is done, get msg: 21,spend times: 119
t4is done, get msg: 17,spend times: 129
t0is done, get msg: 18,spend times: 125
t2is done, get msg: 22,spend times: 124
t3is done, get msg: 22,spend times: 123
Optimize spend: 166
```

总结:该方法能提高数据的处理速度(差不多 5 倍),但也消耗更大的资源和提高了程序的复杂性. ~~要怎么使用,请参考具体生产环境~~

---

## 10. 接口设计

至今,都还记得,这种疼.狗日的接口设置,太膨胀了,这怪谁?当然怪自己了.

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

## 11. 时间处理

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

代码增加了一点,但在线程安全的情况下,Calendar 还比 SimpleDateFormat 快一点.

---

## 12. 快速获取子文件路径

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
 * @author cs12110 2018年9月29日
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

## 13. 加载外部 jar

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
 * @author cs12110 2018年10月16日
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
 * @author cs12110 2018年10月16日
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

---

## 14. clone

Q: 如果有一个对象,只想获取 ta 的副本,修改副本的时候,不改变原来的值,该怎么做呀?

A: U can try clone for this, here some reference docs [link](https://blog.csdn.net/qq_33314107/article/details/80271963)

### 14.1 测试代码

```java
package cn.rojao.irs.ms;

import com.alibaba.fastjson.JSON;
import lombok.Data;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/27 9:50
 * <p>
 * since: 1.0.0
 */
public class MyClone {

    @Data
    static class Student {
        private String name;
        private int age;

        Student(String name, int age) {
            this.name = name;
            this.age = age;
        }

        @Override
        public String toString() {
            return JSON.toJSONString(this);
        }

    }

    public static void main(String[] args) {
        Student stu1 = new Student("haiyan", 16);
        System.out.println("stu1: " + stu1.toString());

        Student stu2 = stu1;
        stu2.name = "3306";

        System.out.println("stu2 == stu1 ? " + (stu1 == stu2));
        System.out.println("stu2: " + stu2.toString());
        System.out.println("stu1: " + stu1.toString());
    }

}
```

测试结果

```json
stu1: {"age":16,"name":"haiyan"}
stu2 == stu1 ? true
stu2: {"age":16,"name":"3306"}
stu1: {"age":16,"name":"3306"}
```

As u can see, 改变`stu2`的属性值同时改变了`stu1`的值,因为 ta 们同时指向同一个引用地址.

### 14.2 clone

```java
package cn.rojao.irs.ms;

import com.alibaba.fastjson.JSON;
import lombok.Data;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/27 9:50
 * <p>
 * since: 1.0.0
 */
public class MyClone {

    /**
     * 实现cloneable接口,并复写clone方法
     */
    @Data
    static class Student implements Cloneable {
        private String name;
        private int age;

        Student(String name, int age) {
            this.name = name;
            this.age = age;
        }

        @Override
        public String toString() {
            return JSON.toJSONString(this);
        }

        /**
         * clone 方法
         *
         * @return Object
         * @throws CloneNotSupportedException clone exception
         */
        public Object clone() throws CloneNotSupportedException {
            return super.clone();
        }

    }

    public static void main(String[] args) {
        try {
            Student stu1 = new Student("haiyan", 16);
            System.out.println("stu1: " + stu1.toString());

            Student stu2 = (Student) stu1.clone();
            stu2.name = "3306";

            System.out.println("stu2 == stu1 ? " + (stu1 == stu2));
            System.out.println("stu2: " + stu2.toString());
            System.out.println("stu1: " + stu1.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

测试结果

```json
stu1: {"age":16,"name":"haiyan"}
stu2 == stu1 ? false
stu2: {"age":16,"name":"3306"}
stu1: {"age":16,"name":"haiyan"}
```

### 14.2 深度复制

浅复制不能解决: `stu.Address` 里面 Address 的复制,所以这种嵌套类型的,要使用深度复制来解决.

工具类

```java
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;

/**
 * 深度克隆,克隆里面的每一个属性的实体类都必须实现{@link java.io.Serializable}接口,请知悉.
 * <p/>
 *
 * @author cs12110 created at: 2019/3/28 8:36
 * <p>
 * since: 1.0.0
 */
public class CloneMachineUtil {


    /**
     * 深度clone
     *
     * @param origin origin
     * @param <T>    T
     * @return T
     * @throws IllegalArgumentException if we can't clone for you
     */
    @SuppressWarnings("unchecked")
    public static <T> T deep(T origin) {
        try {
            PipedInputStream in = new PipedInputStream();
            PipedOutputStream out = new PipedOutputStream();
            in.connect(out);

            // objOut要比objIn先创建,泪目
            try (
                    ObjectOutputStream objOut = new ObjectOutputStream(out);
                    ObjectInputStream objIn = new ObjectInputStream(in)
            ) {
                objOut.writeObject(origin);
                Object cloneValue = objIn.readObject();

                return (T) cloneValue;
            } catch (Exception e) {
                e.printStackTrace();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        throw new IllegalArgumentException("We can't clone [" + origin + "] for you");
    }
}
```

测试类

```java
import com.alibaba.fastjson.JSON;
import lombok.Getter;
import lombok.Setter;

import java.io.Serializable;

/**
 * 各个类需要实现接口: `Serializable`
 * <p/>
 *
 * @author cs12110 created at: 2019/3/28 8:56
 * <p>
 * since: 1.0.0
 */
public class DeepCloneTest {

    @Getter
    @Setter
    private static class Address implements Serializable {
        private String block;
        private String house;
        private String num;
    }


    @Getter
    @Setter
    private static class Student implements Serializable {
        private String name;
        private Integer id;
        private Address address;
    }

    public static void main(String[] args) {

        Address address = new Address();
        address.setBlock("gz");
        address.setHouse("th");
        address.setNum("cb");

        Student student = new Student();
        student.setId(1);
        student.setName("haiyan");
        student.setAddress(address);

        Student clone = CloneMachineUtil.deep(student);

        System.out.println("origin: " + student);
        System.out.println("origin address: " + student.getAddress());
        System.out.println("origin json: " + JSON.toJSON(student));

        System.out.println("---------------------");

        System.out.println("clone: " + clone);
        System.out.println("clone address: " + clone.getAddress());
        System.out.println("clone json: " + JSON.toJSONString(clone));
    }
}
```

执行结果

```java
origin: test.pkgs.DeepCloneTest$Student@2ef1e4fa
origin address: test.pkgs.DeepCloneTest$Address@306a30c7
origin json: {"address":{"num":"cb","block":"gz","house":"th"},"name":"haiyan","id":1}
---------------------
clone: test.pkgs.DeepCloneTest$Student@574caa3f
clone address: test.pkgs.DeepCloneTest$Address@64cee07
clone json: {"address":{"block":"gz","house":"th","num":"cb"},"id":1,"name":"haiyan"}
```

你看,json 数据一模一样,但是引用地址改变了,深度克隆完成.

---

## 15. throw 与 throws

Q: throw 和 throws 有什么区别呀?

A: throw 用于代码手动抛出异常,throws 用在方法声明上.

简单来说: throws 是用在方法上面的,表面该方法可能会产生什么样的异常. throw 呢,就用来在代码里面抛出异常.

### 15.1 throw

Exception 里面又会各种 exception,最主要的分支就是 RuntimeException 了.

```java
/**
 * RuntimeException
 *
 * @author cs12110 create at 2019/4/30 15:22
 * @version 1.0.0
 */
public class RunExp {

    static class MyRunExp extends RuntimeException {
        public MyRunExp(String message) {
            super(message);
        }
    }

    public static void main(String[] args) {
        try {
            test(-1);
        } catch (MyRunExp e) {
            e.printStackTrace();
        }
        test(0);
    }

    private static void test(int index) {
        if (index < 0) {
            throw new MyRunExp("index must >= 0");
        }
        System.out.println("Ok,you will get: " + index);
    }

    //throw也可以结合trhows一起使用.
    //private static void test1(int index) throws MyExp {
    //    if (index < 0) {
    //        // 如果MyRunExp extends Exception的话,在方法里面使用throw必须要try...catch
    //        throw new MyExp("index must >= 0");
    //    }
    //    System.out.println("Ok,you will get: " + index);
    //}
}
```

测试结果

```java
com.woniubaoxian.test.RunExp$MyExp: index must >= 0
	at com.woniubaoxian.test.RunExp.test(RunExp.java:29)
	at com.woniubaoxian.test.RunExp.main(RunExp.java:19)
Ok,you will get: 0
```

### 15.2 throws

在方法 throws 的时候,<u>如果抛出的 exception 是继承 RuntiomeException 的,调用方法不用 try...catch 也不会报错.而抛出的是继承 Exception 的话,调用方法就必须 try...catch 了</u>.

```java
/**
 * Exception
 *
 * @author cs12110 create at 2019/4/30 15:22
 * @version 1.0.0
 */
public class RunExp {

    static class MyExp extends Exception {
        public MyExp(String message) {
            super(message);
        }
    }

    public static void main(String[] args) {
        try {
            test(-1);
        } catch (MyExp e) {
            e.printStackTrace();
        }
        try {
            test(0);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void test(int index) throws MyExp {
        if (index < 0) {
            // 如果MyRunExp extends Exception的话,在方法里面使用throw必须要try...catch
            throw new MyExp("index must >= 0");
        }
        System.out.println("Ok,you will get: " + index);
    }
}
```

测试结果

```java
com.woniubaoxian.test.RunExp$MyExp: index must >= 0
	at com.woniubaoxian.test.RunExp.test(RunExp.java:33)
	at com.woniubaoxian.test.RunExp.main(RunExp.java:19)
Ok,you will get: 0
```

---

## 16. Assert

在 Java 里面还有一个叫 Assert(断言)的家伙,用起来,泪流满面.

```java
/**
 * 如果条件==true往后执行,否则抛出异常
 */
assert yourCondition: "the error message"
```

```java

/**
 * Assert
 *
 * @author cs12110 create at 2019/5/23 19:57
 * @version 1.0.0
 */
public class AssertTest {

    public static void main(String[] args) {
        testAssert(null);
    }

    private static void testAssert(Object value) {
        /*
         *如果value == null 则抛出异常
         */
        assert value != null : "fuck this , your value is empty, don't fool me around";
        System.out.println("Ok, you can go now, I got your back, Miss");
    }
}
```

测试结果

```java
Exception in thread "main" java.lang.AssertionError: fuck this , your value is empty, don't fool me around
	at com.woniubaoxian.test.AssertTest.testAssert(AssertTest.java:16)
	at com.woniubaoxian.test.AssertTest.main(AssertTest.java:12)
```

---

## 17. 对象值复制

在更新数据的操作里面,需要先把之前的数据查询出来,不为空的最新值覆盖字段.

如果有很多的字段,要每一个 setter/getter 这就有点.....

那么,我们可以使用反射来做.

测试实体类

```java
@Data
public class Student {
    private Integer id;
    private String name;
    private Date birth;

    @Override
    public String toString() {
        return JSON.toJSONString(this, true);
    }

}
```

测试方法

```java
import java.lang.reflect.Field;
import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * <p>
 *
 * @author cs12110 create at 2019-07-10 22:54
 * <p>
 * @since 1.0.0
 */
public class ReflectApp {

    public static void main(String[] args) {
        Student stu1 = new Student();
        stu1.setId(1);
        stu1.setName("haiyan");
        stu1.setBirth(parseToDate("2017-06-11 10:20:30"));


        Student stu2 = new Student();
        stu2.setName("3306");
        copyValue(stu1, stu2);


        System.out.println(stu2.toString());
    }


    /**
     * 复制值
     *
     * @param from from
     * @param to   to
     * @param <T>  Anything you want
     */
    private static <T> void copyValue(T from, T to) {
        Class<?> clazz = from.getClass();
        Field[] fields = clazz.getDeclaredFields();

        try {
            for (Field f : fields) {
                f.setAccessible(true);
                Object newValue = f.get(from);
                if (null == newValue) {
                    f.set(to, f.get(from));
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 将日期字符串转换成日期对象
     *
     * @param date String
     * @return Date
     */
    private static Date parseToDate(String date) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        try {
            return sdf.parse(date);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

测试结果

```json
{
  "birth": 1497147630000,
  "id": 1,
  "name": "3306"
}
```

---

## 18. 枚举工具类

之前因为 element-ui 的影响,逐渐也开始把枚举定义成只有`value`,`label`两个属性了.

举个例子:

```java
@AllArgsConstructor
@Getter
public enum SysEnvEnum {

    /**
     * dev: 开发环境
     */
    DEV("dev", "开发环境"),

    /**
     * test: 测试环境
     */
    TEST("test", "测试环境"),

    /**
     * prod: 生产环境
     */
    PROD("prod", "生产环境");

    private final String value;
    private final String label;

    public static SysEnvEnum getByValue(String value) {
        for (SysEnvEnum e : SysEnvEnum.values()) {
            if (e.getValue().equals(value)) {
                return e;
            }
        }
        return null;
    }
}
```

上面这个:`SysEnvEnum#getByValue(String value)`几乎每一个枚举都差不多需要这个根据 value 来获取对应枚举的方法.

Q: 那有没有一种方法可以提高这一块的重用性呀?

A: 在同事的指导下,使用接口,枚举实现接口的方法实现.

```java
public interface ValueLabelFacade<K, V> {
    /**
     * 值字符串
     *
     * @return String
     */
    default String valueStr() {
        return String.valueOf(value());
    }

    /**
     * 值
     *
     * @return K
     */
    K value();

    /**
     * 描述
     *
     * @return V
     */
    V label();

}
```

修改枚举

```java
@AllArgsConstructor
@Getter
public enum SysEnvEnum implements ValueLabelFacade<String, String> {

    /**
     * dev: 开发环境
     */
    DEV("dev", "开发环境"),

    /**
     * test: 测试环境
     */
    TEST("test", "测试环境"),

    /**
     * prod: 生产环境
     */
    PROD("prod", "生产环境");

    private final String value;
    private final String label;

    public static SysEnvEnum getByValue(String value) {
        for (SysEnvEnum e : SysEnvEnum.values()) {
            if (e.getValue().equals(value)) {
                return e;
            }
        }
        return null;
    }

    @Override
    public String value() {
        return value;
    }

    @Override
    public String label() {
        return this.label;
    }
}
```

工具类

```java
public class EnumFacadeUtils {

    /**
     * 将枚举转成map,如果<code>interfaces</code> == null,返回: <code>Collections.emptyMap()</code>
     *
     * @return Map
     */
    public static <P, V> Map<P, V> toMap(ValueLabelFacade<P, V>[] interfaces) {
        if (Objects.isNull(interfaces) || interfaces.length == 0) {
            return Collections.emptyMap();
        }
        HashMap<P, V> map = new HashMap<>(interfaces.length);
        for (ValueLabelFacade<P, V> baseEnum : interfaces) {
            map.put(baseEnum.value(), baseEnum.label());
        }
        return map;
    }

    /**
     * 根据<code>value</code>获取枚举,需要向下转型
     * <p>
     * 实现接口枚举:
     * <pre>
     *     public enum IntStrEnum implements ValueLabelEnum<Integer, String>{
     *         ....
     *     }
     * </pre>
     * <p>
     * 使用范例:
     * <pre>
     *     IntStrEnum integerStringValueLabelEnum = (IntStrEnum) get(values, 1);
     * </pre>
     *
     * @return ValueLabelEnum
     */
    public static <K, V> ValueLabelFacade<K, V> get(ValueLabelFacade<K, V>[] interfaces, K value) {
        if (Objects.isNull(interfaces) || interfaces.length == 0) {
            return null;
        }
        for (ValueLabelFacade<K, V> baseEnum : interfaces) {
            if (Objects.equals(baseEnum.value(), value)) {
                return baseEnum;
            }
        }
        return null;
    }

    /**
     * 根据value获取对应的枚举label
     *
     * @return V
     */
    public static <K, V> V getLabel(ValueLabelFacade<K, V>[] interfaces, K value) {
        if (Objects.isNull(interfaces) || interfaces.length == 0) {
            return null;
        }
        for (ValueLabelFacade<K, V> baseEnum : interfaces) {
            if (Objects.equals(baseEnum.value(), value)) {
                return baseEnum.label();
            }
        }
        return null;
    }
}
```

测试

```java
public class SysEnvEnumTest {
    public static void main(String[] args) {
        Map<String, String> map = EnumFacadeUtils.toMap(SysEnvEnum.values());
        System.out.println(JSON.toJSONString(map, true));

        SysEnvEnum test = (SysEnvEnum) EnumFacadeUtils.get(SysEnvEnum.values(), "test");
        System.out.println(test);

        SysEnvEnum none = (SysEnvEnum) EnumFacadeUtils.get(SysEnvEnum.values(), "none");
        System.out.println(none);

        String label = EnumFacadeUtils.getLabel(SysEnvEnum.values(), "test");
        System.out.println(label);
    }
}
```

测试结果

```java
{
	"dev":"开发环境",
	"test":"测试环境",
	"prod":"生产环境"
}
TEST
null
测试环境
```

---

19. Builder 模式推演

Q: 这个还有什么好说的呀?

A: let me show you something.

```java
public class Funny<T> {

    private Consumer<T> consumer;
    private T value;

    public static <T> Funny<T> getInstance() {
        return new Funny<>();
    }

    public Funny<T> setValue(T val) {
        this.value = val;
        return this;
    }

    public Funny<T> consumer(Consumer<T> consumer) {
        this.consumer = consumer;
        return this;
    }

    public void exec() {
        consumer.accept(value);
    }

    public static void main(String[] args) {
        // 这里会出现类型异常
        Funny.getInstance().consumer(Funny::display).setValue("1231231231").exec();
    }

    public static void display(String val) {
        System.out.println(val);
    }
}
```

Q: 那么我们该怎么处理这种情况呀?

A: 在咨询大佬的情况下,奇怪的知识又增长了.

```java
public class Funny<T> {

    private Consumer<T> consumer;
    private T value;

    /**
     * 指定类型
     *
     * @param clazz 类型class
     * @return Funny
     */
    public static <T> Funny<T> getInstance(Class<T> clazz) {
        return new Funny<>();
    }

    public Funny<T> setValue(T val) {
        this.value = val;
        return this;
    }

    public Funny<T> consumer(Consumer<T> consumer) {
        this.consumer = consumer;
        return this;
    }

    public void exec() {
        consumer.accept(value);
    }

    public static void main(String[] args) {
        Funny.getInstance(String.class).consumer(Funny::display).setValue("1231231231").exec();
    }

    public static void display(String val) {
        System.out.println(val);
    }
}
```
