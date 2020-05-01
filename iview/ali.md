# Ali 面试题

阿里面试涉及的问题和答案,答案全都是网上博客来的,请知悉.

被虐,但也所获匪浅.

---

## 优秀博客

- [JVM 内存划分](https://blog.csdn.net/antony9118/article/details/51375662)
- [Java 内存模型](https://blog.csdn.net/javazejian/article/details/72772461)
- [Spring 启动流程](https://blog.csdn.net/moshenglv/article/details/53517343)
- [Mybatis 源码阅读](https://blog.csdn.net/u011676300/article/details/82904713)
- [Socket 通信原理](https://www.cnblogs.com/itfly8/p/5844803.html)
- [SQL 优化](https://blog.csdn.net/jie_liang/article/details/77340905)
- [Java 设计模式](http://www.runoob.com/design-pattern/design-pattern-tutorial.html)
- [Spring 源码阅读](http://www.cnblogs.com/fangjian0423/p/springMVC-directory-summary.html)
- [MyBatis 源码阅读](https://blog.csdn.net/nmgrd/article/details/54608702)

Q: 数据库建立索引的规则是什么?

---

## JVM 加载机制

当写好一个 Java 程序之后,不是管是 CS 还是 BS 应用,都是由若干个`.class`文件组织而成的一个完整的 Java 应用程序.当程序在运行时,即会调用该程序的一个入口函数来调用系统的相关功能,而这些功能都被封装在不同的 class 文件当中,所以经常要从这个 class 文件中要调用另外一个 class 文件中的方法,如果另外一个文件不存在的,则会引发系统异常.而程序在启动的时候,并不会一次性加载程序所要用的所有 class 文件,而是根据程序的需要,通过 Java 的类加载机制(ClassLoader)来动态加载某个 class 文件到内存当中的,从而只有 class 文件被载入到了内存之后,才能被其它 class 所引用.

**所以 ClassLoader 就是用来动态加载 class 文件到内存当中用的.**

### JVM 的 ClassLoader

**BootStrap ClassLoader**:称为启动类加载器,是 Java 类加载层次中最顶层的类加载器,负责加载 JDK 中的核心类库,如:rt.jar,resources.jar,charsets.jar 等.

可通过如下程序获得该类加载器从哪些地方加载了相关的 jar 或 class 文件:

```java
URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
for(int i = 0; i < urls.length; i++) {
	System.out.println(urls[i].toExternalForm());
}
```

以下内容是上述程序从本机 JDK 环境所获得的结果:

```java
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/resources.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/rt.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/sunrsasign.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/jsse.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/jce.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/lib/charsets.jar
file:/C:/Program%20Files/Java/jdk1.6.0_22/jre/classes/
```

其实上述结果也是通过查找 sun.boot.class.path 这个系统属性所得知的.

```java
System.out.println(System.getProperty("sun.boot.class.path"));
```

打印结果:

```java
C:\Program Files\Java\jdk1.6.0_22\jre\lib\resources.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\rt.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\sunrsasign.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\jce.jar;C:\Program Files\Java\jdk1.6.0_22\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.6.0_22\jre\classes
```

**Extension ClassLoader**:称为扩展类加载器,负责加载 Java 的扩展类库,默认加载 JAVA_HOME/jre/lib/ext/目下的所有 jar.

**App ClassLoader**:称为系统类加载器,负责加载应用程序 classpath 目录下的所有 jar 和 class 文件.

注意: 除了 Java 默认提供的三个 ClassLoader 之外,用户还可以根据需要定义自已的 ClassLoader,而这些自定义的 ClassLoader 都必须继承自 java.lang.ClassLoader 类,也包括 Java 提供的另外二个 ClassLoader(Extension ClassLoader 和 App ClassLoader)在内,但是 Bootstrap ClassLoader 不继承自 ClassLoader,因为它不是一个普通的 Java 类,底层由 C++编写,已嵌入到了 JVM 内核当中,当 JVM 启动后,Bootstrap ClassLoader 也随着启动,负责加载完核心类库后,并构造 Extension ClassLoader 和 App ClassLoader 类加载器.

### 双亲委托

**原理介绍**

ClassLoader 使用的是 **双亲委托模型** 来搜索类的,每个 ClassLoader 实例都有一个父类加载器的引用(不是继承的关系,是一个包含的关系),虚拟机内置的类加载器(Bootstrap ClassLoader)本身没有父类加载器,但可以用作其它 ClassLoader 实例的的父类加载器.当一个 ClassLoader 实例需要加载某个类时,它会试图亲自搜索某个类之前,先把这个任务委托给它的父类加载器,这个过程是由上至下依次检查的,首先由最顶层的类加载器 Bootstrap ClassLoader 试图加载,如果没加载到,则把任务转交给 Extension ClassLoader 试图加载,如果也没加载到,则转交给 App ClassLoader 进行加载,如果它也没有加载得到的话,则返回给委托的发起者,由它到指定的文件系统或网络等 URL 中加载该类.如果它们都没有加载到这个类时,则抛出 ClassNotFoundException 异常.否则将这个找到的类生成一个类的定义,并将它加载到内存当中,最后返回这个类在内存中的 Class 实例对象.

**为什么要使用双亲委托这种模型呢?**

因为这样可以 **避免重复加载**,当父亲已经加载了该类的时候,就没有必要子 ClassLoader 再加载一次.考虑到安全因素,我们试想一下,如果不使用这种委托模式,那我们就可以随时使用自定义的 String 来动态替代 java 核心 api 中定义的类型,这样会存在非常大的安全隐患,而双亲委托的方式,就可以避免这种情况,因为 String 已经在启动时就被引导类加载器(Bootstrcp ClassLoader)加载,所以用户自定义的 ClassLoader 永远也无法加载一个自己写的 String,除非你改变 JDK 中 ClassLoader 搜索类的默认算法.

**但是 JVM 在搜索类的时候,又是如何判定两个 class 是相同的呢?**

​JVM 在判定两个 class 是否相同时,不仅要判断两个类名是否相同,而且要判断是否由同一个类加载器实例加载的.只有两者同时满足的情况下,JVM 才认为这两个 class 是相同的.就算两个 class 是同一份 class 字节码,如果被两个不同的 ClassLoader 实例所加载,JVM 也会认为它们是两个不同 class.比如网络上的一个 Java 类 org.classloader.simple.NetClassLoaderSimple,javac 编译之后生成字节码文件 NetClassLoaderSimple.class,ClassLoaderA 和 ClassLoaderB 这两个类加载器并读取了 NetClassLoaderSimple.class 文件,并分别定义出了 java.lang.Class 实例来表示这个类,对于 JVM 来说,它们是两个不同的实例对象,但它们确实是同一份字节码文件,如果试图将这个 Class 实例生成具体的对象进行转换时,就会抛运行时异常 java.lang.ClassCaseException,提示这是两个不同的类型.现在通过实例来验证上述所描述的是否正确.

---

## HashMap 相关

### HashMap 的增长

在 Java 8 之前,HashMap 和其他基于 map 的类都是通过链地址法解决冲突,它们使用单向链表来存储相同索引值的元素.在最坏的情况下,这种方式会将 HashMap 的 get 方法的性能从 O(1)降低到 O(n).为了解决在频繁冲突时 hashmap 性能降低的问题,Java 8 中使用平衡树来替代链表存储冲突的元素.这意味着我们可以将最坏情况下的性能从 O(n)提高到 O(logn).

在 Java 8 中使用常量 TREEIFY_THRESHOLD 来控制是否切换到平衡树来存储.目前,这个常量值是 8,这意味着当有超过 8 个元素的索引一样时,HashMap 会使用树来存储它们.

这一改变是为了继续优化常用类.大家可能还记得在 Java 7 中为了优化常用类对 ArrayList 和 HashMap 采用了延迟加载的机制,在有元素加入之前不会分配内存,这会减少空的链表和 HashMap 占用的内存.

这一动态的特性使得 HashMap 一开始使用链表,并在冲突的元素数量超过指定值时用平衡二叉树替换链表.不过这一特性在所有基于 hash table 的类中并没有,例如 Hashtable 和 WeakHashMap.
目前,只有 ConcurrentHashMap,LinkedHashMap 和 HashMap 会在频繁冲突的情况下使用平衡树.

什么时候会产生冲突

HashMap 中调用 hashCode()方法来计算 hashCode.
由于在 Java 中两个不同的对象可能有一样的 hashCode,所以不同的键可能有一样 hashCode,从而导致冲突的产生.

总结

- HashMap 在处理冲突时使用链表存储相同索引的元素.
- 从 Java 8 开始,HashMap,ConcurrentHashMap 和 LinkedHashMap 在处理频繁冲突时将使用平衡树来代替链表,当同一 hash 桶中的元素数量超过特定的值便会由链表切换到平衡树,这会将 get()方法的性能从 O(n)提高到 O(logn).
- 当从链表切换到平衡树时,HashMap 迭代的顺序将会改变.不过这并不会造成什么问题,因为 HashMap 并没有对迭代的顺序提供任何保证.
- 从 Java 1 中就存在的 Hashtable 类为了保证迭代顺序不变,即便在频繁冲突的情况下也不会使用平衡树.这一决定是为了不破坏某些较老的需要依赖于 Hashtable 迭代顺序的 Java 应用.
- 除了 Hashtable 之外,WeakHashMap 和 IdentityHashMap 也不会在频繁冲突的情况下使用平衡树.
- 使用 HashMap 之所以会产生冲突是因为使用了键对象的 hashCode()方法,而 equals()和 hashCode()方法不保证不同对象的 hashCode 是不同的.需要记住的是,相同对象的 hashCode 一定是相同的,但相同的 hashCode 不一定是相同的对象.
- 在 HashTable 和 HashMap 中,冲突的产生是由于不同对象的 hashCode()方法返回了一样的值.

以上就是 Java 中 HashMap 如何处理冲突.这种方法被称为链地址法,因为使用链表存储同一桶内的元素.通常情况 HashMap,HashSet,LinkedHashSet,LinkedHashMap,ConcurrentHashMap,HashTable,IdentityHashMap 和 WeakHashMap 均采用这种方法处理冲突.

从 JDK 8 开始,HashMap,LinkedHashMap 和 ConcurrentHashMap 为了提升性能,在频繁冲突的时候使用平衡树来替代链表.因为 HashSet 内部使用了 HashMap,LinkedHashSet 内部使用了 LinkedHashMap,所以他们的性能也会得到提升.

HashMap 的快速高效,使其使用非常广泛.其原理是,调用 hashCode()和 equals()方法,并对 hashcode 进行一定的哈希运算得到相应 value 的位置信息,将其分到不同的桶里.桶的数量一般会比所承载的实际键值对多.当通过 key 进行查找的时候,往往能够在常数时间内找到该 value.

但是,当某种针对 key 的 hashcode 的哈希运算得到的位置信息重复了之后,就发生了哈希碰撞.这会对 HashMap 的性能产生灾难性的影响.

在 Java 8 之前, 如果发生碰撞往往是将该 value 直接链接到该位置的其他所有 value 的末尾,即相互碰撞的所有 value 形成一个链表.

因此,在最坏情况下,HashMap 的查找时间复杂度将退化到 O(n).

但是在 Java 8 中,该碰撞后的处理进行了改进.当一个位置所在的冲突过多时,存储的 value 将形成一个排序二叉树,排序依据为 key 的 hashcode.

则,在最坏情况下,HashMap 的查找时间复杂度将从 O(1)退化到 O(logn).

虽然是一个小小的改进,但意义重大:

- a. O(n)到 O(logn)的时间开销.

- b. 如果恶意程序知道我们用的是 Hash 算法,则在纯链表情况下,它能够发送大量请求导致哈希碰撞,然后不停访问这些 key 导致 HashMap 忙于进行线性查找,最终陷入瘫痪,即形成了拒绝服务攻击(DoS).

### HashMap 和 HashSet 的区别

| hashMap 实现的是 map 接口                                                        | hashSet 实现的是 set 接口                  |
| -------------------------------------------------------------------------------- | ------------------------------------------ |
| hashMap 是键对值存储                                                             | hashset 存储的仅仅是值                     |
| hashMap 使用 put()存入数据                                                       | hashset 使用 add()存入数据                 |
| hashMap 效率比较快,因为他是使用唯一的键来获取对象                                | hashSet 相对于 hashMap 来说效率较慢        |
| hashMap 使用的是键对象来计算 hashcode 值                                         | hashSet 使用的是成员对象来计算 hashcode 值 |
| hashMap 的键具有唯一性,并且允许 null 值和 null 键,且不保证内部数据的顺序恒久不变 | hashSet 具有去除重复项的功能               |

### HashMap 和 HashTable 的区别

HashTable:

方法是同步的,方法不允许 value=null,key=null

HashMap:

方法是非同步的,方法允许 key=null,value=null

HashMap 把 Hashtable 的 contains 方法去掉了,改成 containsvalue 和 containsKey.因为 contains 方法容易让人引起误解.

### LinkedHashMap

在 HashMap 里边,put 进去的元素,取出来是无序的,如果要保证有序,那么可以使用 LinkedHashMap.

```java
public class LinkedMap {
	public static void main(String[] args) {
		Map<String, String> map = new LinkedHashMap<>();

		map.put("1", "1");
		map.put("2", "2");
		map.put("3", "3");
		/*
		 * key,value允许为null
		 */
		map.put(null, null);

		map.forEach((k, v) -> {
			System.out.println(k + ":" + v);
		});
	}
}
```

```java
1:1
2:2
3:3
null:null
```

### HashTable 和 ConcurrentHashMap 的区别

**为什么我们需要 ConcurrentHashMap 和 CopyOnWriteArrayList**

同步的集合类(Hashtable 和 Vector),同步的封装类(使用 Collections.synchronizedMap()方法和 Collections.synchronizedList()方法返回的对象)可以创建出线程安全的 Map 和 List.但是有些因素使得它们不适合高并发的系统.它们仅有单个锁,对整个集合加锁,以及为了防止 ConcurrentModificationException 异常经常要在迭代的时候要将集合锁定一段时间,这些特性对可扩展性来说都是障碍.

ConcurrentHashMap 和 CopyOnWriteArrayList 保留了线程安全的同时,也提供了更高的并发性.ConcurrentHashMap 和 CopyOnWriteArrayList 并不是处处都需要用,大部分时候你只需要用到 HashMap 和 ArrayList,它们用于应对一些普通的情况.

**ConcurrentHashMap 和 Hashtable 的区别**

Hashtable 和 ConcurrentHashMap 有什么分别呢?它们都可以用于多线程的环境,但是当 Hashtable 的大小增加到一定的时候,性能会急剧下降,因为迭代时需要被锁定很长的时间.因为 ConcurrentHashMap 引入了分割(segmentation),不论它变得多么大,仅仅需要锁定 map 的某个部分,而其它的线程不需要等到迭代完成才能访问 map.简而言之,**在迭代的过程中,ConcurrentHashMap 仅仅锁定 map 的某个部分,而 Hashtable 则会锁定整个 map**.

a. HashTable 的线程安全使用的是一个单独的全部 Map 范围的锁,ConcurrentHashMap 抛弃了 HashTable 的单锁机制,使用了锁分离技术,使得多个修改操作能够并发进行,只有进行 SIZE()操作时 ConcurrentHashMap 会锁住整张表.

b. HashTable 的 put 和 get 方法都是同步方法,  而 ConcurrentHashMap 的 get 方法多数情况都不用锁,put 方法需要锁.

但是 ConcurrentHashMap 不能替代 HashTable,因为两者的迭代器的一致性不同的,hash table 的迭代器是强一致性的,而 concurrenthashmap 是弱一致的. ConcurrentHashMap 的 get,clear,iterator 都是弱一致性的.

---

## 线程池的使用

执行流程: **创建 coresize 的初始线程 ->coresize 全部已执行,新进任务放进等待队列 -> 队列已满 -> 开启线程数至 maxsize -> 还执行不过来,进行策略拒绝.**

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
BlockingQueue<Runnable> workQueue);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
 BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}

```

- corePoolSize:核心池的大小,这个参数跟后面讲述的线程池的实现原理有非常大的关系.在创建了线程池后,默认情况下,线程池中并没有任何线程,而是等待有任务到来才创建线程去执行任务,除非调用了 prestartAllCoreThreads()或者 prestartCoreThread()方法,从这 2 个方法的名字就可以看出,是预创建线程的意思,即在没有任务到来之前就创建 corePoolSize 个线程或者一个线程.默认情况下,在创建了线程池后,线程池中的线程数为 0,当有任务来之后,就会创建一个线程去执行任务,当线程池中的线程数目达到 corePoolSize 后,就会把到达的任务放到缓存队列当中;

- maximumPoolSize:线程池最大线程数,这个参数也是一个非常重要的参数,它表示在线程池中最多能创建多少个线程;

- keepAliveTime:表示线程没有任务执行时最多保持多久时间会终止.默认情况下,只有当线程池中的线程数大于 corePoolSize 时,keepAliveTime 才会起作用,直到线程池中的线程数不大于 corePoolSize,即当线程池中的线程数大于 corePoolSize 时,如果一个线程空闲的时间达到 keepAliveTime,则会终止,直到线程池中的线程数不超过 corePoolSize.但是如果调用了 allowCoreThreadTimeOut(boolean)方法,在线程池中的线程数不大于 corePoolSize 时,keepAliveTime 参数也会起作用,直到线程池中的线程数为 0;

- unit:参数 keepAliveTime 的时间单位,有 7 种取值,在 TimeUnit 类中有 7 种静态属性:

```java
TimeUnit.DAYS; //天
TimeUnit.HOURS; //小时
TimeUnit.MINUTES; //分钟
TimeUnit.SECONDS; //秒
TimeUnit.MILLISECONDS; //毫秒
TimeUnit.MICROSECONDS; //微妙
TimeUnit.NANOSECONDS; //纳秒
```

- workQueue:一个阻塞队列,用来存储等待执行的任务,这个参数的选择也很重要,会对线程池的运行过程产生重大影响,一般来说,这里的阻塞队列有以下几种选择:

```java
ArrayBlockingQueue;
LinkedBlockingQueue;
SynchronousQueue;
```

ArrayBlockingQueue 和 PriorityBlockingQueue 使用较少,一般使用 LinkedBlockingQueue 和 Synchronous.线程池的排队策略与 BlockingQueue 有关.

- threadFactory:线程工厂,主要用来创建线程;
- handler:表示当拒绝处理任务时的策略,有以下四种取值:

```java
//丢弃任务并抛出RejectedExecutionException异常.
ThreadPoolExecutor.AbortPolicy;
//也是丢弃任务,但是不抛出异常.
ThreadPoolExecutor.DiscardPolicy;
//丢弃队列最前面的任务,然后重新尝试执行任务(重复此过程)
ThreadPoolExecutor.DiscardOldestPolicy;
//由调用线程处理该任务
ThreadPoolExecutor.CallerRunsPolicy;
```

---

## Java 里面的异常体系

### Java 异常体系

Java 异常体系如下:

![1535693252798](imgs/exception.png)

### 基础概念

**Error 与 Exception**

Error 是程序无法处理的错误,比如 OutOfMemoryError,ThreadDeath 等.这些异常发生时,Java 虚拟机(JVM)一般会选择线程终止.Exception 是程序本身可以处理的异常,这种异常分两大类运行时异常和非运行时异常.程序中应当尽可能去处理这些异常.

**运行时异常和非运行时异常**

运行时异常都是 RuntimeException 类及其子类异常,如 NullPointerException,IndexOutOfBoundsException 等,这些异常是不检查异常,程序中可以选择捕获处理,也可以不处理.这些异常一般是由程序逻辑错误引起的,程序应该从逻辑角度尽可能避免这类异常的发生.非运行时异常是 RuntimeException 以外的异常,类型上都属于 Exception 类及其子类.从程序语法角度讲是必须进行处理的异常,如果不处理,程序就不能编译通过. 如 IOException,SQLException 等以及用户自定义的 Exception 异常,一般情况下不自定义检查异常.

**throw,throws 关键字**

throw 关键字是用于方法体内部,用来抛出一个 Throwable 类型的异常.如果抛出了检查异常,则还应该在方法头部声明方法可能抛出的异常类型.该方法的调用者也必须检查处理抛出的异常.如果所有方法都层层上抛获取的异常,最终 JVM 会进行处理,处理也很简单,就是打印异常消息和堆栈信息.如果抛出的是 Error 或 RuntimeException,则该方法的调用者可选择处理该异常.有关异常的转译会在下面说明.throws 关键字用于方法体外部的方法声明部分,用来声明方法可能会抛出某些异常.仅当抛出了检查异常,该方法的调用者才必须处理或者重新抛出该异常.当方法的调用者无力处理该异常的时候,应该继续抛出,而不是囫囵吞枣一般在 catch 块中打印一下堆栈信息做个勉强处理.

---

## Wait 和 Sleep 的区别

sleep 是线程类(Thread)的方法,导致此线程暂停执行指定时间,给执行机会给其他线程,但是监控状态依然保持,到时后会自动恢复.调用 sleep 不会释放对象锁.
wait 是 Object 类的方法,对此对象调用 wait 方法导致本线程放弃对象锁,进入等待此对象的等待锁定池,只有针对此对象发出 notify 方法(或 notifyAll)后本线程才进入对象锁定池准备获得对象锁进入运行状态.

- 这两个方法来自不同的类分别是 Thread 和 Object

- 最主要是 sleep 方法没有释放锁,而 wait 方法释放了锁,使得其他线程可以使用同步控制块或者方法.

- wait,notify 和 notifyAll 只能在同步控制方法或者同步控制块里面使用,而 sleep 可以在任何地方使用(使用范围)

```java
synchronized(x){
 　　x.notify()
 　　//或者wait()
}
```

- sleep 必须捕获异常,而 wait,notify 和 notifyAll 不需要捕获异常

sleep 方法属于 Thread 类中方法,表示让一个线程进入睡眠状态,等待一定的时间之后,自动醒来进入到可运行状态,不会马上进入运行状态,因为线程调度机制恢复线程的运行也需要时间,一个线程对象调用了 sleep 方法之后,并不会释放他所持有的所有对象锁,所以也就不会影响其他进程对象的运行.但在 sleep 的过程中过程中有可能被其他对象调用它的 interrupt(),产生 InterruptedException 异常,如果你的程序不捕获这个异常,线程就会异常终止,进入 TERMINATED 状态,如果你的程序捕获了这个异常,那么程序就会继续执行 catch 语句块(可能还有 finally 语句块)以及以后的代码.

注意 sleep()方法是一个静态方法,也就是说他只对当前对象有效,通过 t.sleep()让 t 对象进入 sleep,这样的做法是错误的,它只会是使当前线程被 sleep 而不是 t 线程

wait 属于 Object 的成员方法,一旦一个对象调用了 wait 方法,必须要采用 notify()和 notifyAll()方法唤醒该进程;如果线程拥有某个或某些对象的同步锁,那么在调用了 wait()后,这个线程就会释放它持有的所有同步资源,而不限于这个被调用了 wait()方法的对象.wait()方法也同样会在 wait 的过程中有可能被其他对象调用 interrupt()方法而产生.

---

## Mybatis 缓存

### 延迟加载

resultMap 中的 association 和 collection 标签具有延迟加载的功能.

延迟加载的意思是说,在关联查询时,利用延迟加载,先加载主信息.使用关联信息时再去加载关联信息.

**设置延迟加载**

需要在 SqlMapConfig.xml 文件中,在`<settings>`标签中设置下延迟加载.

lazyLoadingEnabled,aggressiveLazyLoading

| 设置项                | 描述                                                                               | 允许值          | 默认值  |
| --------------------- | ---------------------------------------------------------------------------------- | --------------- | ------- |
| lazyLoadingEnabled    | 全局性设置懒加载.如果设为‘false’,则所有相关联的都会被初始化加载.                   | `true`,`false`  | `false` |
| aggressiveLazyLoading | 当设置为‘true’的时候,懒加载的对象可能被任何懒属性全部加载.否则,每个属性都按需加载. | `true`, `false` | `true`  |

### 查询缓存

**Mybatis 的一级缓存**是指 SqlSession.

一级缓存的作用域是一个 SqlSession.Mybatis 默认开启一级缓存.

在同一个 SqlSession 中,执行相同的查询 SQL,第一次会去查询数据库,并写到缓存中;第二次直接从缓存中取.当执行 SQL 时两次查询中间发生了增删改操作,则 SqlSession 的缓存清空.

**Mybatis 的二级缓存**是指 mapper 映射文件.

二级缓存的作用域是同一个 namespace 下的 mapper 映射文件内容,多个 SqlSession 共享.Mybatis 需要手动设置启动二级缓存.

在同一个 namespace 下的 mapper 文件中,执行相同的查询 SQL,第一次会去查询数据库,并写到缓存中;第二次直接从缓存中取.当执行 SQL 时两次查询中间发生了增删改操作,则二级缓存清空.

**一级缓存原理**

![](imgs/wKiom1WII2CwQRhlAADBHk2wFdY170.jpg)

一级缓存区域是根据 SqlSession 为单位划分的.

每次查询会先去缓存中找,如果找不到,再去数据库查询,然后把结果写到缓存中.Mybatis 的内部缓存使用一个 HashMap,key 为 hashcode+statementId+sql 语句.Value 为查询出来的结果集映射成的 java 对象.

SqlSession 执行 insert,update,delete 等操作 commit 后会清空该 SQLSession 缓存.

**二级缓存原理**

![](imgs/wKioL1WIJXvA4ngUAADEvZunxso732.jpg)

二级缓存是 mapper 级别的.Mybatis 默认是没有开启二级缓存.

第一次调用 mapper 下的 SQL 去查询用户信息.查询到的信息会存到该 mapper 对应的二级缓存区域内.

第二次调用相同 namespace 下的 mapper 映射文件中相同的 SQL 去查询用户信息.会去对应的二级缓存内取结果.

如果调用相同 namespace 下的 mapper 映射文件中的增删改 SQL,并执行了 commit 操作.此时会清空该 namespace 下的二级缓存.

### 缓存设置

在核心配置文件 SqlMapConfig.xml 中加入以下内容(开启二级缓存总开关):

cacheEnabled 设置为  true

![](imgs/wKiom1WIJFChnBasAADcJX3IbNs777.jpg)

在映射文件中,加入以下内容,开启二级缓存:

![](imgs/wKiom1WIJHGj-78eAACCk6Tv9vs396.jpg)

**实现序列化**

由于二级缓存的数据不一定都是存储到内存中,它的存储介质多种多样,所以需要给缓存的对象执行序列化.

如果该类存在父类,那么父类也要实现序列化.

![](imgs/wKioL1WIJmXQfEQ4AAC1EcHDT6w451.jpg)

**禁用二级缓存**

该 statement 中设置 userCache=false 可以禁用当前 select 语句的二级缓存,即每次查询都是去数据库中查询,默认情况下是 true,即该 statement 使用二级缓存.

![](imgs/wKiom1WIJPvRdgaUAAC-FQgNUyI548.jpg)

**刷新二级缓存**

![](imgs/wKioL1WIJvXykyTeAACdJiTWDLM099.jpg)

---

## Mysql

### 组合索引

联合索引又叫复合索引.对于复合索引:Mysql 从左到右的使用索引中的字段,一个查询可以只使用索引中的一部份,但只能是最左侧部分.例如索引是 key index(a,b,c). 可以支持[a],[a,b],[a,c] ,[a,b,c] 4 种组合进行查找,但不支持 b,c 进行查找.当最左侧字段是常量引用时,索引就十分有效.

假设有表结构如下

```sql
DROP TABLE IF EXISTS `abc`;
CREATE TABLE `abc`(
  `a` varchar(255) DEFAULT NULL,
  `b` varchar(255) DEFAULT NULL,
  `c` varchar(255) DEFAULT NULL,
  `d` varchar(255) DEFAULT NULL,
  KEY `abcIndex`(`a`,`b`,`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

对字段`a`,`b`,`c`建立联合索引

下面是可以使用到索引的情况

```sql
select * from abc where a ='1' and d='1';
select * from abc where a ='1' and b='1' and c ='1';
select * from abc where a ='1' and c='1';
select * from abc where a ='1' and b='1';
```

下面是不能使用到索引的情况

```sql
select * from abc where b='1' and c ='1';
```

### 乐观锁与悲观锁

悲观锁与乐观锁是两种常见的资源并发锁设计思路,也是并发编程中一个非常基础的概念.本文将对这两种常见的锁机制在数据库数据上的实现进行比较系统的介绍.

**悲观锁(Pessimistic Lock)**

悲观锁的特点是先获取锁,再进行业务操作,即"悲观"的认为获取锁是非常有可能失败的,因此要先确保获取锁成功再进行业务操作.通常所说的"一锁二查三更新"即指的是使用悲观锁.通常来讲在数据库上的悲观锁需要数据库本身提供支持,即通过常用的 select ... for update 操作来实现悲观锁.当数据库执行 select for update 时会获取被 select 中的数据行的行锁,因此其他并发执行的 select for update 如果试图选中同一行则会发生排斥(需要等待行锁被释放),因此达到锁的效果.select for update 获取的行锁会在当前事务结束时自动释放,因此必须在事务中使用.

这里需要注意的一点是不同的数据库对 select for update 的实现和支持都是有所区别的,例如 oracle 支持 select for update no wait,表示如果拿不到锁立刻报错,而不是等待,mysql 就没有 no wait 这个选项.另外 mysql 还有个问题是 select for update 语句执行中所有扫描过的行都会被锁上,这一点很容易造成问题.因此如果在 mysql 中用悲观锁务必要确定走了索引,而不是全表扫描.

**乐观锁(Optimistic Lock)**

乐观锁的特点先进行业务操作,不到万不得已不去拿锁.即"乐观"的认为拿锁多半是会成功的,因此在进行完业务操作需要实际更新数据的最后一步再去拿一下锁就好.

乐观锁在数据库上的实现完全是逻辑的,不需要数据库提供特殊的支持.一般的做法是在需要锁的数据上增加一个版本号,或者时间戳,然后按照如下方式实现：

```java
SELECT data AS old_data, version AS old_version FROM ...;

// 根据获取的数据进行业务操作,得到new_data和new_version
UPDATE SET data = new_data, version = new_version WHERE version = old_version

if (updated row > 0) {
    // 乐观锁获取成功,操作完成
} else {
    // 乐观锁获取失败,回滚并重试
}

```

乐观锁是否在事务中其实都是无所谓的,其底层机制是这样：在数据库内部 update 同一行的时候是不允许并发的,即数据库每次执行一条 update 语句时会获取被 update 行的写锁,直到这一行被成功更新后才释放.因此在业务操作进行前获取需要锁的数据的当前版本号,然后实际更新数据时再次对比版本号确认与之前获取的相同,并更新版本号,即可确认这之间没有发生并发的修改.如果更新失败即可认为老版本的数据已经被并发修改掉而不存在了,此时认为获取锁失败,需要回滚整个业务操作并可根据需要重试整个过程.

**总结**

乐观锁在不发生取锁失败的情况下开销比悲观锁小,但是一旦发生失败回滚开销则比较大,因此适合用在取锁失败概率比较小的场景,可以提升系统并发性能

乐观锁还适用于一些比较特殊的场景,例如在业务操作过程中无法和数据库保持连接等悲观锁无法适用的地方

---

## 动态代理和 Cglib 的区别

### 基础知识

Spirng 的 AOP 的动态代理实现机制有两种,分别是:

JDK 动态代理

> 通过实现 InvocationHandlet 接口创建自己的调用处理器
>
> 通过为 Proxy 类指定 ClassLoader 对象和一组 interface 来创建动态代理
>
> 通过反射机制获取动态代理类的构造函数,其唯一参数类型就是调用处理器接口类型
>
> 通过构造函数创建动态代理类实例,构造时调用处理器对象作为参数参入
>
> JDK 动态代理是面向接口的代理模式,如果被代理目标没有接口那么 Spring 也无能为力
>
> Spring 通过 java 的反射机制生产被代理接口的新的匿名实现类,重写了其中 AOP 的增强方法

CGLib 动态代理

> **CGLib 是一个强大,高性能的 Code 生产类库,可以实现运行期动态扩展 java 类,Spring 在运行期间通过 CGlib 继承要被动态代理的类,重写父类的方法,实现 AOP 面向切面编程呢.**

### 两者对比

**比较:**

JDK 动态代理是 **面向接口**,在创建代理实现类时比 CGLib 要快,创建代理速度快.

CGLib 动态代理是通过字节码底层继承要代理类来实现(如果被代理类被 final 关键字所修饰,那么抱歉会失败),在创建代理这一块没有 JDK 动态代理快,但是运行速度比 JDK 动态代理要快.

**使用注意:**

JDK 动态代理特点

- 代理对象必须实现一个或多个接口
- 以接口形式接收代理实例,而不是代理类

CGLIB 动态代理特点

- 代理对象不能被 final 修饰
- 以类或接口形式接收代理实例

---

## 消息队列对比

主流消息队列对比

![](imgs/messagequeue.png)

---

## Spring 相关

### SpringBoot 和 Spring MVC 的区别

原文: [link](https://blog.csdn.net/u014590757/article/details/79602309)

spring boot 只是一个配置工具,整合工具,辅助工具.

springmvc 是框架,项目中实际运行的代码

Spring 框架就像一个家族，有众多衍生产品例如 boot,security,jpa 等等.但他们的基础都是 Spring 的 ioc 和 aop. ioc 提供了依赖注入的容器， aop 解决了面向横切面的编程，然后在此两者的基础上实现了其他延伸产品的高级功能.

Spring MVC 是基于 Servlet 的一个 MVC 框架主要解决 WEB 开发的问题，因为 Spring 的配置非常复杂，各种 XML, JavaConfig,hin 处理起来比较繁琐.于是为了简化开发者的使用，从而创造性地推出了 Spring boot，约定优于配置，简化了 spring 的配置流程.

说得更简便一些：Spring 最初利用"工厂模式"(DI)和"代理模式"(AOP)解耦应用组件.大家觉得挺好用，于是按照这种模式搞了一个 MVC 框架(一些用 Spring 解耦的组件)，用开发 web 应用( SpringMVC ).然后发现每次开发都写很多样板代码，为了简化工作流程，于是开发出了一些"懒人整合包"(starter)，这套就是 Spring Boot.

Spring MVC 的功能

Spring MVC 提供了一种轻度耦合的方式来开发 web 应用.

Spring MVC 是 Spring 的一个模块，式一个 web 框架.通过 Dispatcher Servlet, ModelAndView 和 View Resolver，开发 web 应用变得很容易.解决的问题领域是网站应用程序或者服务开发——URL 路由,Session,模板引擎,静态 Web 资源等等.

Spring Boot 的功能

Spring Boot 实现了自动配置，降低了项目搭建的复杂度.

众所周知 Spring 框架需要进行大量的配置，Spring Boot 引入自动配置的概念，让项目设置变得很容易.Spring Boot 本身并不提供 Spring 框架的核心特性以及扩展功能，只是用于快速,敏捷地开发新一代基于 Spring 框架的应用程序.也就是说，它并不是用来替代 Spring 的解决方案，而是和 Spring 框架紧密结合用于提升 Spring 开发者体验的工具.同时它集成了大量常用的第三方库配置(例如 Jackson, JDBC, Mongo, Redis, Mail 等等)，Spring Boot 应用中这些第三方库几乎可以零配置的开箱即用(out-of-the-box)，大部分的 Spring Boot 应用都只需要非常少量的配置代码，开发者能够更加专注于业务逻辑.

Spring Boot 只是承载者，辅助你简化项目搭建过程的.如果承载的是 WEB 项目，使用 Spring MVC 作为 MVC 框架，那么工作流程和你上面描述的是完全一样的，因为这部分工作是 Spring MVC 做的而不是 Spring Boot.

对使用者来说，换用 Spring Boot 以后，项目初始化方法变了，配置文件变了，另外就是不需要单独安装 Tomcat 这类容器服务器了，maven 打出 jar 包直接跑起来就是个网站，但你最核心的业务逻辑实现与业务流程实现没有任何变化.

所以，用最简练的语言概括就是：

Spring 是一个"引擎"；

Spring MVC 是基于 Spring 的一个 MVC 框架；

Spring Boot 是基于 Spring4 的条件注册的一套快速开发整合包.

### Spring 各个组件功能

**核心容器(Spring core)**

核心容器提供 Spring 框架的基本功能.Spring 以 bean 的方式组织和管理 Java 应用中的各个组件及其关系.Spring 使用 BeanFactory 来产生和管理 Bean,它是工厂模式的实现.BeanFactory 使用控制反转(IoC)模式将应用的配置和依赖性规范与实际的应用程序代码分开.BeanFactory 使用依赖注入的方式提供给组件依赖.

**Spring 上下文(Spring context)**

Spring 上下文是一个配置文件,向 Spring 框架提供上下文信息.Spring 上下文包括企业服务,如 JNDI,EJB,电子邮件,国际化,校验和调度功能.

**Spring 面向切面编程(Spring AOP)**

通过配置管理特性,Spring AOP 模块直接将面向方面的编程功能集成到了 Spring 框架中.所以,可以很容易地使 Spring 框架管理的任何对象支持 AOP.Spring AOP 模块为基于 Spring 的应用程序中的对象提供了事务管理服务.通过使用 Spring AOP,不用依赖 EJB 组件,就可以将声明性事务管理集成到应用程序中.

**Spring DAO 模块**

DAO 模式主要目的是将持久层相关问题与一般的的业务规则和工作流隔离开来.Spring 中的 DAO 提供一致的方式访问数据库,不管采用何种持久化技术,Spring 都提供一直的编程模型.Spring 还对不同的持久层技术提供一致的 DAO 方式的异常层次结构.

**Spring ORM 模块**

Spring 与所有的主要的 ORM 映射框架都集成的很好,包括 Hibernate,JDO 实现,TopLink 和 IBatis SQL Map 等.Spring 为所有的这些框架提供了模板之类的辅助类,达成了一致的编程风格.

**Spring Web 模块**

Web 上下文模块建立在应用程序上下文模块之上,为基于 Web 的应用程序提供了上下文.Web 层使用 Web 层框架,可选的,可以是 Spring 自己的 MVC 框架,或者提供的 Web 框架,如 Struts,Webwork,tapestry 和 jsf.

**Spring MVC 框架(Spring WebMVC)**

MVC 框架是一个全功能的构建 Web 应用程序的 MVC 实现.通过策略接口,MVC 框架变成为高度可配置的.Spring 的 MVC 框架提供清晰的角色划分：控制器,验证器,命令对象,表单对象和模型对象,分发器,处理器映射和视图解析器.Spring 支持多种视图技术.

### Spring Cloud 各个组件

### Spring 请求响应过程

请求如下图所示

![](imgs/spring.png)

### 获取 ApplicationContext

在项目中,经常遇到这样的问题:有些类需要使用 new 来创建对象,但是类中需要使用 spring 容器中定义的 bean,此时无法通过 spring 的自动注入来注入我们需要使用的 bean.所以需要手动的从 spring 容器中获取 bean.要获取 bean 必须先获取到 ApplicationContext 对象,有以下方式可以获取该对象.

#### 方式一

手动创建 ApplicationContext 对象,并保存起来.

```java
public class ApplicationContextUtil {
    private static ApplicationContext context;
    static {
        context = new ClassPathXmlApplicationContext("applicationContext.xml");
    }
    public static ApplicationContext getApplicationContext() {
        return context;
    }
}
```

#### 方式二

在 web 环境中通过 spring 提供的工具类获取,需要 ServletContext 对象作为参数.然后才通过 ApplicationContext 对象获取 bean.下面两个工具方式的区别是,前者在获取失败时返回 null,后者抛出异常.另外,由于 spring 是容器的对象放在 ServletContext 中的,所以可以直接在 ServletContext 取出 WebApplicationContext 对象.

```java
ApplicationContext context1 = WebApplicationContextUtils.getWebApplicationContext(ServletContext sc);
ApplicationContext context2 = WebApplicationContextUtils.getRequiredWebApplicationContext(ServletContext sc);
```

```java
WebApplicationContext webApplicationContext =(WebApplicationContext) servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
```

#### 方式三

工具类继承抽象类 ApplicationObjectSupport,并在工具类上使用@Component 交由 spring 管理.这样 spring 容器在启动的时候,会通过父类

ApplicationObjectSupport 中的 setApplicationContext()方法将 ApplicationContext 对象设置进去.可以通过 getApplicationContext()得到 ApplicationContext 对象.

#### 方式四

工具类继承抽象类 WebApplicationObjectSupport,查看源码可知 WebApplicationObjectSupport 是继承了 ApplicationObjectSupport,所以获取 ApplicationContext 对象的方式和上面一样,也是使用 getApplicationContext()方法.

#### 方式五

工具类实现 ApplicationContextAware 接口,并重写 setApplicationContext(ApplicationContext applicationContext)方法,在工具类中使用@Component 注解让 spring 进行管理.spring 容器在启动的时候,会调用 setApplicationContext()方法将 ApplicationContext 对象设置进去.

```java
@Component
public class ApplicationContextUtil implements ApplicationContextAware {
    private static ApplicationContext context;

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        context = applicationContext;
    }

	public static ApplicationContext getApplicationContext() {
        return context;
    }
}
```

### @Autowired 与 @Resource

年初刚加入到现在的项目时,在使用注解时我用的@Resource.后来,同事:你怎么使用@Resource 注解?我:使用它有错吗?同事:没错,但是现在都使用@Autowired.我:我研究一下.

在大学,学习 J2EE 实训时一直使用的是@Resource 注解,后来我就养成习惯了.现在对这两个注解做一下解释:

- @Resource 默认按照名称方式进行 bean 匹配,@Autowired 默认按照类型方式进行 bean 匹配
- @Resource(import javax.annotation.Resource;)是 J2EE 的注解,@Autowired( import org.springframework.beans.factory.annotation.Autowired;)是 Spring 的注解

Spring 属于第三方的,J2EE 是 Java 自己的东西.使用@Resource 可以减少代码和 Spring 之间的耦合.

**@Resource 注入**

现在有一个接口 Human 和两个实现类 ManImpl,WomanImpl,在 service 层的一个 bean 中要引用了接口 Human,这种情况处理如下:

接口 Human

```java
package testwebapp.com.wangzuojia.service;

public interface Human {
	public void speak();
	public void walk();
}
```

实现类 ManImpl

```java
package testwebapp.com.wangzuojia.service.impl;

import org.springframework.stereotype.Service;
import testwebapp.com.wangzuojia.service.Human;


@Service
public class ManImpl implements Human {

	public void speak() {
		System.out.println(" man speaking!");
	}

	public void walk() {
		System.out.println(" man walking!");
	}

}
```

实现类 WomanImpl

```java
package testwebapp.com.wangzuojia.service.impl;

import org.springframework.stereotype.Service;
import testwebapp.com.wangzuojia.service.Human;

@Service
public class WomanImpl implements Human {

	public void speak() {
		System.out.println(" woman speaking!");
	}

	public void walk() {
		System.out.println(" woman walking!");
	}

}
```

主调类 SequenceServiceImpl

```java
package testwebapp.com.wangzuojia.service.impl;

import java.util.Map;
import javax.annotation.Resource;
import org.springframework.stereotype.Service;
import testwebapp.com.wangzuojia.dao.SequenceMapper;
import testwebapp.com.wangzuojia.service.Human;
import testwebapp.com.wangzuojia.service.SequenceService;

@Service
public class SequenceServiceImpl implements SequenceService {

	@Resource
	private SequenceMapper sequenceMapper;

	public void generateId(Map<String, String> map) {
		sequenceMapper.generateId(map);
	}

	//起服务此处会报错
	@Resource
	private Human human;

}
```

这时候启动 tomcat 会包如下错误:

```java
严重: Exception sendingcontext initialized event to listener instance of classorg.springframework.web.context.ContextLoaderListenerorg.springframework.beans.factory.BeanCreationException: Error creating beanwith name 'sequenceServiceImpl': Injection of resource dependencies failed;nested exception isorg.springframework.beans.factory.NoUniqueBeanDefinitionException: Noqualifying bean of type [testwebapp.com.wangzuojia.service.Human] is defined:expected single matching bean**but found 2: manImpl,
```

报错的地方给我们提示了:*but found 2: manImpl,womanImpl*  意思是 Human 有两个实现类.解决方案如下:

```java
package testwebapp.com.wangzuojia.service.impl;


import java.util.Map;
import javax.annotation.Resource;
import org.springframework.stereotype.Service;
import testwebapp.com.wangzuojia.dao.SequenceMapper;
import testwebapp.com.wangzuojia.service.Human;
import testwebapp.com.wangzuojia.service.SequenceService;

@Service
public class SequenceServiceImpl implements SequenceService {

	@Resource
	private SequenceMapper sequenceMapper;

	public void generateId(Map<String, String> map) {
		sequenceMapper.generateId(map);
	}

//注意是manImpl不是ManImpl,因为使用@Service,容器为我们创建bean时默认类名首字母小写
	@Resource(name = "manImpl")
	private Human human;
}
```

这样启动服务就不会报错了.如果是使用的@Autowired 注解,要配上@Qualifier("manImpl"),代码如下:

```java
package testwebapp.com.wangzuojia.service.impl;

import java.util.Map;
import javax.annotation.Resource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import testwebapp.com.wangzuojia.dao.SequenceMapper;
import testwebapp.com.wangzuojia.service.Human;
import testwebapp.com.wangzuojia.service.SequenceService;

@Service
public class SequenceServiceImpl implements SequenceService {

	@Resource
	private SequenceMapper sequenceMapper;

	public void generateId(Map<String, String> map) {
		sequenceMapper.generateId(map);
	}
	@Autowired
	@Qualifier("manImpl")
	private Human human;
}
```

### BeanFactory 与 FactoryBean

区别:BeanFactory 是个 Factory,也就是 IOC 容器或对象工厂,FactoryBean 是个 Bean.在 Spring 中,所有的 Bean 都是由 BeanFactory(也就是 IOC 容器)来进行管理的.但对 FactoryBean 而言,这个 Bean 不是简单的 Bean,而是一个能生产或者修饰对象生成的工厂 Bean,它的实现与设计模式中的工厂模式和修饰器模式类似

**BeanFactory**

BeanFactory 定义了 IOC 容器的最基本形式,并提供了 IOC 容器应遵守的的最基本的接口,也就是 Spring IOC 所遵守的最底层和最基本的编程规范.在 Spring 代码中,BeanFactory 只是个接口,并不是 IOC 容器的具体实现,但是 Spring 容器给出了很多种实现,如  DefaultListableBeanFactory,XmlBeanFactory,ApplicationContext 等,都是附加了某种功能的实现.

```java
package org.springframework.beans.factory;
import org.springframework.beans.BeansException;
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class<?> targetType) throws NoSuchBeanDefinitionException;
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    String[] getAliases(String name);
}
```

**FactoryBean**

一般情况下,Spring 通过反射机制利用<bean>的 class 属性指定实现类实例化 Bean,在某些情况下,实例化 Bean 过程比较复杂,如果按照传统的方式,则需要在<bean>中提供大量的配置信息.配置方式的灵活性是受限的,这时采用编码的方式可能会得到一个简单的方案.Spring 为此提供了一个 org.springframework.bean.factory.FactoryBean 的工厂类接口,用户可以通过实现该接口定制实例化 Bean 的逻辑.
FactoryBean 接口对于 Spring 框架来说占用重要的地位,Spring 自身就提供了 70 多个 FactoryBean 的实现.它们隐藏了实例化一些复杂 Bean 的细节,给上层应用带来了便利.从 Spring3.0 开始,FactoryBean 开始支持泛型,即接口声明改为 FactoryBean<T>的形式

```java
package org.springframework.beans.factory;
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    boolean isSingleton();
}
```

该接口中还定义了以下 3 个方法:

- T getObject():返回由 FactoryBean 创建的 Bean 实例,如果 isSingleton()返回 true,则该实例会放到 Spring 容器中单实例缓存池中;

- boolean isSingleton():返回由 FactoryBean 创建的 Bean 实例的作用域是 singleton 还是 prototype;

- Class<T> getObjectType():返回 FactoryBean 创建的 Bean 类型.

当配置文件中`<bean>`的 class 属性配置的实现类是 FactoryBean 时,通过 getBean()方法返回的不是 FactoryBean 本身,而是 FactoryBean#getObject()方法所返回的对象,相当于 FactoryBean#getObject()代理了 getBean()方法.

例:如果使用传统方式配置下面 Car 的`<bean>`时,Car 的每个属性分别对应一个`<property>`元素标签.

```java
package  com.baobaotao.factorybean;
public   class  Car  {
	private  int maxSpeed ;
	private  String brand ;
	private  double price ;
	public   int  getMaxSpeed()   {
		return   this.maxSpeed ;
	}
	public   void  setMaxSpeed( int  maxSpeed )   {
		this.maxSpeed  = maxSpeed;
	}
	public  String getBrand()   {
		return   this.brand ;
	}
	public   void  setBrand( String brand )   {
		this.brand  = brand;
	}
	public   double  getPrice()   {
		return   this.price ;
	}
	public   void  setPrice( double  price )   {
		this.price  = price;
	}
}
```

如果用 FactoryBean 的方式实现就灵活点,下例通过逗号分割符的方式一次性的为 Car 的所有属性指定配置值:

```java
package  com.baobaotao.factorybean;
import  org.springframework.beans.factory.FactoryBean;
public   class  CarFactoryBean  implements  FactoryBean<Car>  {
    private  String carInfo ;
    public  Car getObject()   throws  Exception  {
        Car car =  new  Car() ;
        String []  infos =  carInfo.split( "," ) ;
        car.setBrand( infos [ 0 ]) ;
        car.setMaxSpeed( Integer. valueOf( infos [ 1 ])) ;
        car.setPrice( Double. valueOf( infos [ 2 ])) ;
        return  car;
    }
    public  Class<Car> getObjectType()   {
        return  Car.class ;
    }
    public   boolean  isSingleton()   {
        return   false ;
    }
    public  String getCarInfo()   {
        return   this.carInfo ;
    }

    // 接受逗号分割符设置属性信息
    public   void  setCarInfo( String carInfo )   {
        this.carInfo  = carInfo;
    }
}
```

有了这个 CarFactoryBean 后,就可以在配置文件中使用下面这种自定义的配置方式配置 CarBean 了:

```xml
<bean id="car" class="com.baobaotao.factorybean.CarFactoryBean" P:carInfo="法拉利,400,2000000"/>
```

当调用 getBean("car")时,Spring 通过反射机制发现 CarFactoryBean 实现了 FactoryBean 的接口,这时 Spring 容器就调用接口方法 CarFactoryBean#getObject()方法返回.如果希望获取 CarFactoryBean 的实例,则需要在使用 getBean(beanName)方法时在 beanName 前显示的加上"&"前缀:如 getBean("&car");

**区别**

BeanFactory 是个 Factory,也就是 IOC 容器或对象工厂,FactoryBean 是个 Bean.在 Spring 中,所有的 Bean 都是由 BeanFactory(也就是 IOC 容器)来进行管理的.但对 FactoryBean 而言,这个 Bean 不是简单的 Bean,而是一个能生产或者修饰对象生成的工厂 Bean,它的实现与设计模式中的工厂模式和修饰器模式类似.

### @Component 与@Service 区别

在 spring 集成的框架中,注解在类上的`@Component`,`@Repository`,`@Service`等注解能否被互换?或者说这些注解有什么区别?

引用 spring 的官方文档中的一段描述:

在 Spring2.0 之前的版本中,`@Repository`注解可以标记在任何的类上,用来表明该类是用来执行与数据库相关的操作(即 dao 对象),并支持自动处理数据库操作产生的异常

在 Spring2.5 版本中,引入了更多的 Spring 类注解:`@Component`,`@Service`,`@Controller`.`Component`是一个通用的 Spring 容器管理的单例 bean 组件.而`@Repository`, `@Service`, `@Controller`就是针对不同的使用场景所采取的特定功能化的注解组件.

因此,当你的一个类被`@Component`所注解,那么就意味着同样可以用`@Repository`, `@Service`, `@Controller`来替代它,同时这些注解会具备有更多的功能,而且功能各异.

最后,如果你不知道要在项目的业务层采用`@Service`还是`@Component`注解.那么,`@Service`是一个更好的选择.

就如上文所说的,`@Repository`早已被支持了在你的持久层作为一个标记可以去自动处理数据库操作产生的异常(译者注:因为原生的 java 操作数据库所产生的异常只定义了几种,但是产生数据库异常的原因却有很多种,这样对于数据库操作的报错排查造成了一定的影响;而 Spring 拓展了原生的持久层异常,针对不同的产生原因有了更多的异常进行描述.所以,在注解了`@Repository`的类上如果数据库操作中抛出了异常,就能对其进行处理,转而抛出的是翻译后的 spring 专属数据库异常,方便我们对异常进行排查处理).

| 注解        |                     含义                      |
| ----------- | :-------------------------------------------: |
| @Component  | 最普通的组件,可以被注入到 spring 容器进行管理 |
| @Repository |                 作用于持久层                  |
| @Service    |               作用于业务逻辑层                |
| @Controller |        作用于表现层(spring-mvc 的注解)        |

回答 2

这几个注解几乎可以说是一样的:因为被这些注解修饰的类就会被 Spring 扫描到并注入到 Spring 的 bean 容器中.

这里,有两个注解是不能被其他注解所互换的:

- `@Controller` 注解的 bean 会被 spring-mvc 框架所使用.
- `@Repository` 会被作为持久层操作(数据库)的 bean 来使用

如果想使用自定义的组件注解,那么只要在你定义的新注解中加上`@Component`即可:

```java
@Component

@Scope("prototype")

public @interface ScheduleJob {...}
```

这样,所有被`@ScheduleJob`注解的类就都可以注入到 spring 容器来进行管理.我们所需要做的,就是写一些新的代码来处理这个自定义注解(译者注:可以用反射的方法),进而执行我们想要执行的工作.

回答 3

`@Component`就是跟`<bean>`一样,可以托管到 Spring 容器进行管理.

@Service, @Controller , @Repository = {@Component + 一些特定的功能}.这个就意味着这些注解在部分功能上是一样的.

当然,下面三个注解被用于为我们的应用进行分层:

- `@Controller`注解类进行前端请求的处理,转发,重定向.包括调用 Service 层的方法
- `@Service`注解类处理业务逻辑
- `@Repository`注解类作为 DAO 对象(数据访问对象,Data Access Objects),这些类可以直接对数据库进行操作

有这些分层操作的话,代码之间就实现了松耦合,代码之间的调用也清晰明朗,便于项目的管理;假想一下,如果只用`@Controller`注解,那么所有的请求转发,业务处理,数据库操作代码都糅合在一个地方,那这样的代码该有多难拓展和维护.

总结

- `@Component`, `@Service`, `@Controller`, `@Repository`是 spring 注解,注解后可以被 spring 框架所扫描并注入到 spring 容器来进行管理
- `@Component`是通用注解,其他三个注解是这个注解的拓展,并且具有了特定的功能
- `@Repository`注解在持久层中,具有将数据库操作抛出的原生异常翻译转化为 spring 的持久层异常的功能.
- `@Controller`层是 spring-mvc 的注解,具有将请求进行转发,重定向的功能.
- `@Service`层是业务逻辑层注解,这个注解只是标注该类处于业务逻辑层.
- 用这些注解对应用进行分层之后,就能将请求处理,义务逻辑处理,数据库操作处理分离出来,为代码解耦,也方便了以后项目的维护和开发.

---

## Spring 事务和传播机制

### 事务的嵌套概念

所谓事务的嵌套就是两个事务方法之间相互调用.spring 事务开启 ,或者是基于接口的或者是基于类的代理被创建(**注意一定要是代理,不能手动 new 一个对象,并且此类(有无接口都行)一定要被代理——spring 中的 bean 只要纳入了 IOC 管理都是被代理的**).所以在同一个类中一个方法调用另一个方法有事务的方法,事务是不会起作用的.

Spring 默认情况下会对运行期例外(RunTimeException),即 uncheck 异常,进行事务回滚.

如果遇到 checked 异常就不回滚.

如何改变默认规则:

1. 让 checked 例外也回滚:在整个方法前加上 `@Transactional(rollbackFor=Exception.class)`

2. 让 unchecked 例外不回滚: `@Transactional(notRollbackFor=RunTimeException.class)`

3. 不需要事务管理的(只查询的)方法:`@Transactional(propagation=Propagation.NOT_SUPPORTED)`

上面三种方式也可在 xml 配置

### spring 事务传播属性

在 spring 的  TransactionDefinition 接口中一共定义了六种事务传播属性:

```java
//支持当前事务,如果当前没有事务,就新建一个事务.这是最常见的选择. 
PROPAGATION_REQUIRED;

//支持当前事务,如果当前没有事务,就以非事务方式执行. 
PROPAGATION_SUPPORTS;

//支持当前事务,如果当前没有事务,就抛出异常. 
PROPAGATION_MANDATORY;

//新建事务,如果当前存在事务,把当前事务挂起. 
PROPAGATION_REQUIRES_NEW;

//以非事务方式执行操作,如果当前存在事务,就把当前事务挂起. 
PROPAGATION_NOT_SUPPORTED;

//以非事务方式执行,如果当前存在事务,则抛出异常. 
PROPAGATION_NEVER;

//如果当前存在事务,则在嵌套事务内执行.如果当前没有事务,则进行与PROPAGATION_REQUIRED类似的操作. 
PROPAGATION_NESTED;
```

前六个策略类似于 EJB CMT,第七个(PROPAGATION_NESTED)是 Spring 所提供的一个特殊变量. 
它要求事务管理器或者使用 JDBC 3.0 Savepoint API 提供嵌套事务行为(如 Spring 的 DataSourceTransactionManager)

举例浅析 Spring 嵌套事务

**ServiceA#methodA(我们称之为外部事务),ServiceB#methodB(我们称之为外部事务)**

```java
ServiceA {
    void methodA() {
        ServiceB.methodB();
    }
}

ServiceB {
    void methodB() {
}

```

**PROPAGATION_REQUIRED**

假如当前正要执行的事务不在另外一个事务里,那么就起一个新的事务  
比如说,ServiceB.methodB 的事务级别定义为 PROPAGATION_REQUIRED, 那么由于执行 ServiceA.methodA 的时候

- 如果 ServiceA.methodA 已经起了事务,这时调用 ServiceB.methodB,ServiceB.methodB 看到自己已经运行在 ServiceA.methodA 的事务内部,就不再起新的事务.这时只有外部事务并且他们是共用的,所以这时 ServiceA.methodA 或者 ServiceB.methodB 无论哪个发生异常 methodA 和 methodB 作为一个整体都将一起回滚.
- 如果 ServiceA.methodA 没有事务,ServiceB.methodB 就会为自己分配一个事务.这样,在 ServiceA.methodA 中是没有事务控制的.只是在 ServiceB.methodB 内的任何地方出现异常,ServiceB.methodB 将会被回滚,不会引起 ServiceA.methodA 的回滚

**PROPAGATION_SUPPORTS**

如果当前在事务中,即以事务的形式运行,如果当前不在一个事务中,那么就以非事务的形式运行

**PROPAGATION_MANDATORY**

必须在一个事务中运行.也就是说,他只能被一个父事务调用.否则,他就要抛出异常

**PROPAGATION_REQUIRES_NEW**

启动一个新的, 不依赖于环境的 "内部" 事务. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行.

比如我们设计 ServiceA.methodA 的事务级别为 PROPAGATION_REQUIRED,ServiceB.methodB 的事务级别为 PROPAGATION_REQUIRES_NEW,那么当执行到 ServiceB.methodB 的时候,ServiceA.methodA 所在的事务就会挂起,ServiceB.methodB 会起一个新的事务,等待 ServiceB.methodB 的事务完成以后,他才继续执行.他与 PROPAGATION_REQUIRED 的事务区别在于事务的回滚程度了.因为 ServiceB.methodB 是新起一个事务,那么就是存在两个不同的事务.

- 如果 ServiceB.methodB 已经提交,那么 ServiceA.methodA 失败回滚,ServiceB.methodB 是不会回滚的.
- 如果 ServiceB.methodB 失败回滚,如果他抛出的异常被 ServiceA.methodA 的 try..catch 捕获并处理,ServiceA.methodA 事务仍然可能提交;如果他抛出的异常未被 ServiceA.methodA 捕获处理,ServiceA.methodA 事务将回滚.

使用场景:

不管业务逻辑的 service 是否有异常,Log Service 都应该能够记录成功,所以 Log Service 的传播属性可以配为此属性.最下面将会贴出配置代码.

**PROPAGATION_NOT_SUPPORTED**

当前不支持事务.比如 ServiceA.methodA 的事务级别是 PROPAGATION_REQUIRED ,而 ServiceB.methodB 的事务级别是 PROPAGATION_NOT_SUPPORTED ,那么当执行到 ServiceB.methodB 时,ServiceA.methodA 的事务挂起,而他以非事务的状态运行完,再继续 ServiceA.methodA 的事务.

**PROPAGATION_NEVER**

不能在事务中运行.假设 ServiceA.methodA 的事务级别是 PROPAGATION_REQUIRED, 而 ServiceB.methodB 的事务级别是 PROPAGATION_NEVER ,那么 ServiceB.methodB 就要抛出异常了.

**PROPAGATION_NESTED**

开始一个 "嵌套的" 事务,   它是已经存在事务的一个真正的子事务. 潜套事务开始执行时,   它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交.

比如我们设计 ServiceA.methodA 的事务级别为 PROPAGATION_REQUIRED,ServiceB.methodB 的事务级别为 PROPAGATION_NESTED,那么当执行到 ServiceB.methodB 的时候,ServiceA.methodA 所在的事务就会挂起,ServiceB.methodB 会起一个**新的子事务并设置 savepoint**,等待 ServiceB.methodB 的事务完成以后,他才继续执行..因为 ServiceB.methodB 是外部事务的子事务,那么 1,如果 ServiceB.methodB 已经提交,那么 ServiceA.methodA 失败回滚,ServiceB.methodB 也将回滚.2,如果 ServiceB.methodB 失败回滚,如果他抛出的异常被 ServiceA.methodA 的 try..catch 捕获并处理,ServiceA.methodA 事务仍然可能提交;如果他抛出的异常未被 ServiceA.methodA 捕获处理,ServiceA.methodA 事务将回滚.

理解 Nested 的关键是 savepoint.他与 PROPAGATION_REQUIRES_NEW 的区别是:

PROPAGATION_REQUIRES_NEW 完全是一个新的事务,它与外部事务相互独立; 而 PROPAGATION_NESTED 则是外部事务的子事务, 如果外部事务 commit, 嵌套事务也会被 commit, 这个规则同样适用于 roll back.

**在 spring 中使用 PROPAGATION_NESTED 的前提:**

a. 我们要设置 transactionManager 的 nestedTransactionAllowed 属性为 true, 注意, 此属性默认为 false!!!

b. java.sql.Savepoint 必须存在, 即 jdk 版本要 1.4+

c. Connection.getMetaData().supportsSavepoints() 必须为 true, 即 jdbc drive 必须支持 JDBC 3.0

确保以上条件都满足后, 你就可以尝试使用 PROPAGATION_NESTED 了.

### Log Service 配置事务传播

不管业务逻辑的 service 是否有异常,Log Service 都应该能够记录成功,通常有异常的调用更是用户关心的.Log Service 如果沿用业务逻辑 Service 的事务的话在抛出异常时将没有办法记录日志(事实上是回滚了).所以希望 Log Service 能够有独立的事务.日志和普通的服务应该具有不同的策略.Spring 配置文件 transaction.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-2.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.0.xsd">
	<!-- configure transaction -->

	<tx:advice id="defaultTxAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" read-only="true" />
			<tx:method name="query*" read-only="true" />
			<tx:method name="find*" read-only="true" />
			<tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		</tx:attributes>
	</tx:advice>

	<tx:advice id="logTxAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" read-only="true" />
			<tx:method name="query*" read-only="true" />
			<tx:method name="find*" read-only="true" />
			<tx:method name="*" propagation="REQUIRES_NEW"
				rollback-for="java.lang.Exception" />
		</tx:attributes>
	</tx:advice>

	<aop:config>
		<aop:pointcut id="defaultOperation"
			expression="@within(com.homent.util.DefaultTransaction)" />
		<aop:pointcut id="logServiceOperation"
			expression="execution(* com.homent.service.LogService.*(..))" />

		<aop:advisor advice-ref="defaultTxAdvice" pointcut-ref="defaultOperation" />
		<aop:advisor advice-ref="logTxAdvice" pointcut-ref="logServiceOperation" />
	</aop:config>
</beans>
```

​
如上面的 Spring 配置文件所示,日志服务的事务策略配置为 propagation="REQUIRES_NEW",告诉 Spring 不管上下文是否有事务,Log Service 被调用时都要求一个完全新的只属于 Log Service 自己的事务.通过该事务策略,Log Service 可以独立的记录日志信息,不再受到业务逻辑事务的干扰.

---

## Spring IOC 和 AOP 的应用

IOC:IOC,另外一种说法叫 DI(Dependency Injection),即依赖注入.它并不是一种技术实现,而是一种设计思想.在任何一个有实际开发意义的程序项目中,我们会使用很多类来描述它们特有的功能,并且通过类与类之间的相互协作来完成特定的业务逻辑.这个时候,每个类都需要负责管理与自己有交互的类的引用和依赖,代码将会变的异常难以维护和极度的高耦合.而 IOC 的出现正是用来解决这个问题,我们通过 IOC 将这些相互依赖对象的创建.协调工作交给 Spring 容器去处理,每个对象只需要关注其自身的业务逻辑关系就可以了.在这样的角度上来看,获得依赖的对象的方式,进行了反转,变成了由 spring 容器控制对象如何获取外部资源(包括其他对象和文件资料等等).

举例:某一天,你生病了,但是你不清楚自己到底得了什么病,你只知道自己头疼,咳嗽,全身无力.这个时候你决定去药店买药,药店有很多种药,仅仅是治疗头疼就有好几十种,还有西药中药等区别.然后你自己看了看说明书,选择了一盒你自己觉得最能治疗自己病症的药,付钱吃药,期待可以早点好起来.
但是这个过程,对于一个病人来说,太辛苦了.头疼,咳嗽,全身无力,还要一个个的看药品说明书,一个个的比较哪个药比较好,简直是太累了.这个时候,你决定直接去医院看医生.
医生给你做了检查,知道你的病症是什么,有什么原因引起的;同时医生非常了解有哪些药能治疗你的病痛,并且能根据你的自身情况进行筛选.只需要短短的十几分钟,你就能拿到对症下药的药品,即省时又省力.

在上面这个例子中,IOC 起到的就是医生的作用,它收集你的需求要求,并且对症下药,直接把药开给你.你就是对象,药品就是你所需要的外部资源.通过医生,你不用再去找药品,而是通过医生把药品开给你.这就是整个 IOC 的精髓所在.

AOP:面向切面编程,往往被定义为促使软件系统实现关注点的分离的技术.系统是由许多不同的组件所组成的,每一个组件各负责一块特定功能.除了实现自身核心功能之外,这些组件还经常承担着额外的职责.例如日志.事务管理和安全这样的核心服务经常融入到自身具有核心业务逻辑的组件中去.这些系统服务经常被称为横切关注点,因为它们会跨越系统的多个组件.

AOP 的概念不好像 IOC 一样实例化举例,现在我们以一个系统中的具体实现来讲讲 AOP 具体是个什么技术.

我们以系统中常用到的事务管控举例子.在系统操作数据库的过程中,不可避免地要考虑到事务相关的内容.如果在每一个方法中都新建一个事务管理器,那么无疑是对代码严重的耦合和侵入.为了简化我们的开发过程(实际上 spring 所做的一切实现都是为了简化开发过程),需要把事务相关的代码抽成出来做为一个独立的模块.通过 AOP,确认每一个操作数据库方法为一个连接点,这些连接点组成了一个切面.当程序运行到其中某个一个切点时,我们将事务管理模块顺势织入对象中,通过通知功能,完成整个事务管控的实现.这样一来,所有的操作数据库的方法中不需要再单独关心事务管理的内容,只需要关注自身的业务代码的实现即可.所有的事务管控相关的内容都通过 AOP 的方式进行了实现.简化了代码的内容,将目标对象复杂的内容进行解耦,分离业务逻辑与横切关注点.

下面介绍一下 AOP 相关的术语:

通知: 通知定义了切面是什么以及何时使用的概念.

Spring 切面可以应用 5 种类型的通知:

> 前置通知(Before):在目标方法被调用之前调用通知功能.
>
> 后置通知(After):在目标方法完成之后调用通知,此时不会关心方法的输出是什么.
>
> 返回通知(After-returning):在目标方法成功执行之后调用通知.
>
> 异常通知(After-throwing):在目标方法抛出异常后调用通知.
>
> 环绕通知(Around):通知包裹了被通知的方法,在被通知的方法调用之前和调用之后执行自定义的行为.
>
> 连接点:是在应用执行过程中能够插入切面的一个点.
>
> 切点:切点定义了切面在何处要织入的一个或者多个连接点.
>
> 切面:是通知和切点的结合.通知和切点共同定义了切面的全部内容.
>
> 引入:引入允许我们向现有类添加新方法或属性.
>
> 织入:是把切面应用到目标对象,并创建新的代理对象的过程.切面在指定的连接点被织入到目标对象中.在目标对象的生命周期中有多个点可以进行织入.
>
> 编译期:在目标类编译时,切面被织入.这种方式需要特殊的编译器.AspectJ 的织入编译器就是以这种方式织入切面的.
>
> 类加载期:切面在目标加载到 JVM 时被织入.这种方式需要特殊的类加载器(class loader)它可以在目标类被引入应用之前增强该目标类的字节码.
>
> 运行期: 切面在应用运行到某个时刻时被织入.一般情况下,在织入切面时,AOP 容器会为目标对象动态地创建一个代理对象.SpringAOP 就是以这种方式织入切面的.

---

## RPC 与 Rest

接口调用通常包含两个部分,序列化和通信协议.常见的序列化协议包括 json,xml,hession,protobuf,thrift,text,bytes 等;通信比较流行的是 http,soap,websockect,RPC 通常基于 TCP 实现,常用框架例如 dubbo,netty,mina,thrift

**两种接口**

> Rest:严格意义上说接口很规范,操作对象即为资源,对资源的四种操作(post,get,put,delete),并且参数都放在 URL 上,但是不严格的说 Http+json,Http+xml,常见的 http api 都可以称为 Rest 接口.
>
> Rpc:我们常说的远程方法调用,就是像调用本地方法一样调用远程方法,通信协议大多采用二进制方式

**http vs 高性能二进制协议**

http 相对更规范,更标准,更通用,无论哪种语言都支持 http 协议.如果你是对外开放 API,例如开放平台,外部的编程语言多种多样,你无法拒绝对每种语言的支持,相应的,如果采用 http,无疑在你实现 SDK 之前,支持了所有语言,所以,现在开源中间件,基本最先支持的几个协议都包含 RESTful.

RPC 协议性能要高的多,例如 Protobuf,Thrift,Kyro 等,(如果算上序列化)吞吐量大概能达到 http 的二倍.响应时间也更为出色.千万不要小看这点性能损耗,公认的,微服务做的比较好的,例如,netflix,阿里,曾经都传出过为了提升性能而合并服务.如果是交付型的项目,性能更为重要,因为你卖给客户往往靠的就是性能上微弱的优势.

RESTful 你可以看看,无论是 Google,Amazon,netflix(据说很可能转向 grpc),还是阿里,实际上内部都是采用性能更高的 RPC 方式.而对外开放的才是 RESTful.

Rest 调用及测试都很方便,Rpc 就显得有点麻烦,但是 Rpc 的效率是毋庸置疑的,所以建议在多系统之间采用 Rpc,对外提供服务,Rest 是很适合的
duboo 在生产者和消费者两个微服务之间的通信采用的就是 Rpc,无疑在服务之间的调用 Rpc 更变现的优秀

Rpc 在微服务中的利用

1. RPC 框架是架构微服务化的首要基础组件,它能大大降低架构微服务化的成本,提高调用方与服务提供方的研发效率,屏蔽跨进程调用函数(服务)的各类复杂细节

2) RPC 框架的职责是: 让调用方感觉就像调用本地函数一样调用远端函数,让服务提供方感觉就像实现一个本地函数一样来实现服务

![](imgs/20170107161455853.jpg)

**RPC 的好处**

RPC 的主要功能目标是让构建分布式计算(应用)更容易,在提供强大的远程调用能力时不损失本地调用的语义简洁性. 为实现该目标,RPC 框架需提供一种透明调用机制让使用者不必显式的区分本地调用和远程调用.

服务化的一个好处就是,不限定服务的提供方使用什么技术选型,能够实现大公司跨团队的技术解耦.

如果没有统一的服务框架,RPC 框架,各个团队的服务提供方就需要各自实现一套序列化,反序列化,网络框架,连接池,收发线程,超时处理,状态机等"业务之外"的重复技术劳动,造成整体的低效.所以,统一 RPC 框架把上述"业务之外"的技术劳动统一处理,是服务化首要解决的问题

**几种协议**

Socket 使用时可以指定协议 Tcp,Udp 等;

RIM 使用 Jrmp 协议,Jrmp 又是基于 TCP/IP;

RPC 底层使用 Socket 接口,定义了一套远程调用方法;

HTTP 是建立在 TCP 上,不是使用 Socket 接口,需要连接方主动发数据给服务器,服务器无法主动发数据个客户端;

Web Service 提供的服务是基于 web 容器的,底层使用 http 协议,类似一个远程的服务提供者,比如天气预报服务,对各地客户端提供天气预报,是一种请求应答的机制,是跨系统跨平台的.就是通过一个 servlet,提供服务出去.

hessian 是一套用于建立 web service 的简单的二进制协议,用于替代基于 XML 的 web service,是建立在 rpc 上的,hessian 有一套自己的序列化格式将数据序列化成流,然后通过 http 协议发送给服务器.
