# JVM&GC

之前为了面试死记硬背的东西,现在尝试着去理解,去利用这些东西.

---

## 1. jvm 参数

该章节讲述 jvm 相关配置参数和基本应用.

### 1.1 项目配置参数

```sh
# mr3306 @ mr3306 in ~/Box/projects/secret/codes/secret-admin on git:feature-1006432 x [23:17:40]
$  mvn clean package -s /Users/mr3306/Box/soft/maven/conf/secret-settings.xml


# mr3306 @ mr3306 in ~/Box/projects/secret/codes/secret-admin/secret-admin-controller/target on git:feature-1006432 x [13:47:01] C:137
$ java -jar -Xmx1024m -Xms512m -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/mr3306/Box/soft/eclipse-mat/oom secret-admin-controller.jar
```

Q: 怎么判断配置是否生效?

A: 在 jvisualvm 里面的`jvm arguments`里面可以看到配置的参数

```
-Xmx1024m
-Xms512m
-XX:+PrintGCDetails
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/Users/mr3306/Box/soft/eclipse-mat/oo
```

### 1.2 配置参数

参数配置前缀含义:

| 前缀 | 含义                              | 备注                                  |
| ---- | --------------------------------- | ------------------------------------- |
| -    | 标准 VM 选项,VM 规范的选项        |                                       |
| -X   | 非标准 VM 选项,不保证所有 VM 支持 | 例如: -Xmx                            |
| -XX  | 高级选项,高级特性,属于不稳定选项. | 例如: -XX:+HeapDumpOnOutOfMemoryError |

#### 1.2.1 常用配置参数

常用配置参数 [参数 link](http://www.51gjie.com/java/551.html):

| **参数名称**                     | **含义**                                                       | **默认值**            | 说明                                                                                                                                                                                                                                                                                                                                           |
| -------------------------------- | -------------------------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -Xms                             | 初始堆大小                                                     | 物理内存的 1/64(<1GB) | 默认(MinHeapFreeRatio 参数可以调整)空余堆内存小于 40%时,JVM 就会增大堆直到-Xmx 的最大限制.                                                                                                                                                                                                                                                     |
| -Xmx                             | 最大堆大小                                                     | 物理内存的 1/4(<1GB)  | 默认(MaxHeapFreeRatio 参数可以调整)空余堆内存大于 70%时,JVM 会减少堆直到 -Xms 的最小限制                                                                                                                                                                                                                                                       |
| -Xmn                             | 年轻代大小(1.4or lator)                                        |                       | **注意**：此处的大小是(eden+ 2 survivor space).与 jmap -heap 中显示的 New gen 是不同的. 整个堆大小=年轻代大小 + 年老代大小 + 持久代大小. 增大年轻代后,将会减小年老代大小.此值对系统性能影响较大,Sun 官方推荐配置为整个堆的 3/8                                                                                                                 |
| -XX:NewSize                      | 设置年轻代大小(for 1.3/1.4)                                    |                       |                                                                                                                                                                                                                                                                                                                                                |
| -XX:MaxNewSize                   | 年轻代最大值(for 1.3/1.4)                                      |                       |                                                                                                                                                                                                                                                                                                                                                |
| -XX:PermSize                     | 设置持久代(perm gen)初始值                                     | 物理内存的 1/64       |                                                                                                                                                                                                                                                                                                                                                |
| -XX:MaxPermSize                  | 设置持久代最大值                                               | 物理内存的 1/4        |                                                                                                                                                                                                                                                                                                                                                |
| -Xss                             | 每个线程的堆栈大小                                             |                       | JDK5.0 以后每个线程堆栈大小为 1M,以前每个线程堆栈大小为 256K.更具应用的线程所需内存大小进行 调整.在相同物理内存下,减小这个值能生成更多的线程.但是操作系统对一个进程内的线程数还是有限制的,不能无限生成,经验值在 3000~5000 左右 一般小的应用, 如果栈不是很深, 应该是 128k 够用的 大的应用建议使用 256k.这个选项对性能影响比较大,需要严格的测试. |
| -XX:NewRatio                     | 年轻代(包括 Eden 和两个 Survivor 区)与年老代的比值(除去持久代) |                       | -XX:NewRatio=4 表示年轻代与年老代所占比值为 1:4,年轻代占整个堆栈的 1/5 Xms=Xmx 并且设置了 Xmn 的情况下,该参数不需要进行设置.                                                                                                                                                                                                                   |
| -XX:SurvivorRatio                | Eden 区与 Survivor 区的大小比值                                |                       | 设置为 8,则两个 Survivor 区与一个 Eden 区的比值为 2:8,一个 Survivor 区占整个年轻代的 1/10                                                                                                                                                                                                                                                      |
| -XX:MaxTenuringThreshold         | 垃圾最大年龄                                                   |                       | 如果设置为 0 的话,则年轻代对象不经过 Survivor 区,直接进入年老代. 对于年老代比较多的应用,可以提高效率.如果将此值设置为一个较大值,则年轻代对象会在 Survivor 区进行多次复制,这样可以增加对象再年轻代的存活 时间,增加在年轻代即被回收的概率 该参数只有在串行 GC 时才有效.                                                                          |
| -XX:+HeapDumpOnOutOfMemoryError  | 设置 OutOfMemoryError 时打印堆的信息和 dump                    |                       | 在 oom 的时候将当前堆信息 dump,推荐设置与 -XX:HeapDumpPath=/your-path/dump 搭配使用                                                                                                                                                                                                                                                            |
| -XX:HeapDumpPath=/your-path/dump | 设置 dump heap 文件的位置                                      |                       | 在 oom 的时候将当前堆信息 dump,推荐设置                                                                                                                                                                                                                                                                                                        |

#### 1.2.2 辅助参数

| 参数                   | 说明                                                 | 备注                                                                                                                                                                                                                                                |
| ---------------------- | ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -XX:+PrintGC           | 打印 gc                                              | 输出形式:[GC 118250K->113543K(130112K), 0.0094143 secs] [Full GC 121376K->10414K(130112K), 0.0650971 secs]                                                                                                                                          |
| -XX:+PrintGCDetails    | 打印 GC 详情                                         | 输出形式:[GC [DefNew: 8614K->781K(9088K), 0.0123035 secs] 118250K->113543K(130112K), 0.0124633 secs] [GC [DefNew: 8614K->8614K(9088K), 0.0000665 secs][tenured: 112761k->10414k(121024k), 0.0433488 secs] 121376K->10414K(130112K), 0.0436268 secs] |
| -XX:+PrintGCTimeStamps | 打印 gc 时间戳                                       |                                                                                                                                                                                                                                                     |
| -Xloggc:filename       | 把相关日志信息记录到文件以便分析. 与上面几个配合使用 |                                                                                                                                                                                                                                                     |

#### 1.2.3 垃圾收集器参数

**串行收集器**

| 设置参数         | 备注           |
| ---------------- | -------------- |
| -XX:+UseSerialGC | 设置串行收集器 |

**并行收集器(吞吐量优先)**

| 设置参数                   | 备注                                                                                                                                                                  |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -XX:+UseParallelGC         | 设置为并行收集器.此配置仅对年轻代有效.即年轻代使用并行收集,而年老代仍使用串行收集.                                                                                    |
| -XX:ParallelGCThreads      | 配置并行收集器的线程数,即：同时有多少个线程一起进行垃圾回收.此值建议配置与 CPU 数目相等(-XX:ParallelGCThreads=20 )                                                    |
| -XX:+UseParallelOldGC      | 配置年老代垃圾收集方式为并行收集.JDK6.0 开始支持对年老代并行收集.                                                                                                     |
| -XX:MaxGCPauseMillis       | 设置每次年轻代垃圾回收的最长时间(单位毫秒).如果无法满足此时间,JVM 会自动调整年轻代大小,以满足此时间.(-XX:MaxGCPauseMillis=100)                                        |
| -XX:+UseAdaptiveSizePolicy | 设置此选项后,并行收集器会自动调整年轻代 Eden 区大小和 Survivor 区大小的比例,以达成目标系统规定的最低响应时间或者收集频率等指标.此参数建议在使用并行收集器时,一直打开. |

**并发收集器(响应时间优先)**

| 设置参数                           | 备注                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -XX:+UseConcMarkSweepGC            | 即 CMS 收集,设置年老代为并发收集.CMS 收集是 JDK1.4 后期版本开始引入的新 GC 算法.它的主要适合场景是对响应时间的重要性需求大于对吞吐量的需求,能够承受垃圾回收线程和应用线程共享 CPU 资源,并且应用中存在比较多的长生命周期对象.CMS 收集的目标是尽量减少应用的暂停时间,减少 Full GC 发生的几率,利用和应用程序线程并发的垃圾回收线程来标记清除年老代内存. |
| -XX:+UseParNewGC                   | 设置年轻代为并发收集.可与 CMS 收集同时使用.JDK5.0 以上,JVM 会根据系统配置自行设置,所以无需再设置此参数.                                                                                                                                                                                                                                              |
| -XX:CMSFullGCsBeforeCompaction     | 由于并发收集器不对内存空间进行压缩和整理,所以运行一段时间并行收集以后会产生内存碎片,内存使用效率降低.此参数设置运行 0 次 Full GC 后对内存空间进行压缩和整理,即每次 Full GC 后立刻开始压缩和整理内存.(-XX:CMSFullGCsBeforeCompaction=0)                                                                                                               |
| -XX:+UseCMSCompactAtFullCollection | 打开内存空间的压缩和整理,在 Full GC 后执行.可能会影响性能,但可以消除内存碎片.                                                                                                                                                                                                                                                                        |
| -XX:+CMSIncrementalMode            | 设置为增量收集模式.一般适用于单 CPU 情况.                                                                                                                                                                                                                                                                                                            |
| -XX:CMSInitiatingOccupancyFraction | 表示年老代内存空间使用到 70%时就开始执行 CMS 收集,以确保年老代有足够的空间接纳来自年轻代的对象,避免 Full GC 的发生.(-XX:CMSInitiatingOccupancyFraction=70)                                                                                                                                                                                           |

### 1.3 启动设置参数范例

启动命令范例:

```sh
java -jar -Xargs -XX:args yourJar.jar
```

启动范例:

```sh
# mr3306 @ mr3306 in ~/Box/projects/secret/codes/secret-admin/secret-admin-controller/target on git:feature-1006432 x [13:47:01] C:137
$ java -jar -Xmx1024m -Xms512m -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/Users/mr3306/Box/soft/eclipse-mat/oom secret-admin-controller.jar
```

---

## 2. gc 优化

该章节讲述项目 gc 的情况.

### 2.1 项目 gc 日志

GC 日志格式如下:

```java
GC (Allocation Failure) [PSYoungGen: 524800K①->12561K②(611840K③)] 524800K④->12649K⑤(2010112K⑥), 0.0345720 secs⑦] [Times: user=0.05⑧ sys=0.01⑨, real=0.03 secs⑩

① YoungGC 前新生代的内存占用量
② YoungGC 后新生代的内存占用量
③ 新生代总内存大小
④ YoungGC 前 JVM 堆内存占用量
⑤ YoungGC 后 JVM 堆内存使用量
⑥ JVM 堆内存总大小
⑦ YoungGC 耗时
⑧ YoungGC 用户耗时
⑨ YoungGC 系统耗时
⑩ YoungGC 实际耗时
```

`Allocation Failure`:表明本次引起 GC 的原因是因为在年轻代中没有足够的空间能够存储新的数据了.

`ParNew`: -XX:+UseParNewGC(新生代使用并行收集器,老年代使用串行回收收集器)或者-XX:+UseConcMarkSweepGC(新生代使用并行收集器,老年代使用 CMS).

`PSYoungGen`: 使用-XX:+UseParallelOldGC(新生代,老年代都使用并行回收收集器)或者-XX:+UseParallelGC(新生代使用并行回收收集器,老年代使用串行收集器)

现实的一条 GC 日志:

```java
[GC (Allocation Failure) [PSYoungGen: 171744K->1312K(172544K)] 378226K->207862K(789504K), 0.0082469 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
```

### 2.2 gc 监测

在服务器上可以使用`jstat`来查看 gc 情况

```sh
[root@secret-admin-test-deployment-f64dfb5cb-fs28g data]# jstat -gc 1  60s
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
2048.0 2048.0  0.0   1472.1 170496.0 77056.2   616960.0   209981.9  189784.0 174229.4 23168.0 20738.9    549    8.013   5      1.746    9.759
2048.0 2048.0 1088.1  0.0   170496.0 12832.5   616960.0   210013.9  189784.0 174229.9 23168.0 20738.9    550    8.028   5      1.746    9.775
2048.0 2048.0 1088.1  0.0   170496.0 118613.7  616960.0   210013.9  189784.0 174229.9 23168.0 20738.9    550    8.028   5      1.746    9.775
2048.0 2048.0  0.0   1104.1 170496.0 54626.7   616960.0   210053.9  189784.0 174229.9 23168.0 20738.9    551    8.039   5      1.746    9.785
2048.0 2048.0  0.0   1104.1 170496.0 159744.6  616960.0   210053.9  189784.0 174229.9 23168.0 20738.9    551    8.039   5      1.746    9.785
2048.0 2048.0 1136.1  0.0   170496.0 97537.2   616960.0   210101.9  189784.0 174229.9 23168.0 20738.9    552    8.056   5      1.746    9.803
2048.0 2048.0  0.0   1264.1 170496.0 34454.9   616960.0   210141.9  189784.0 174229.9 23168.0 20738.9    553    8.064   5      1.746    9.811
```

Q: 那么在什么情况下,需要根据 **gc** 进行项目的内存调整呢?

A: 如果 GC 执行时间满足下列所有条件,就没有必要进行 GC 优化[link](https://www.cnblogs.com/ityouknow/p/7653129.html)

```jjava
Minor GC执行非常迅速(50ms以内)
Minor GC没有频繁执行(大约10s执行一次)
Full GC执行非常迅速(1s以内)
Full GC没有频繁执行(大约10min执行一次)
```

在上面的给出例子里面:

- young gc 的耗时: 一次 young gc 的耗时为 0.015s(<50ms)
- full gc 的耗时: 一次 full gc 的耗时为 0.3492s(<1s)

这个项目的运行的内存情况还是可以接受的,不需要进行 gc 优化.

### 2.2 查询项目的 heap

```sh
# 命令格式: jmap -heap pid
[root@secret-admin-test-deployment-f64dfb5cb-fs28g data]# jmap -heap 1
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.144-b01

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 178782208 (170.5MB)
   MaxNewSize               = 715653120 (682.5MB)
   OldSize                  = 358088704 (341.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 174587904 (166.5MB)
   used     = 155953504 (148.72885131835938MB)
   free     = 18634400 (17.771148681640625MB)
   89.32663742844407% used
From Space:
   capacity = 2097152 (2.0MB)
   used     = 1228912 (1.1719818115234375MB)
   free     = 868240 (0.8280181884765625MB)
   58.599090576171875% used
To Space:
   capacity = 2097152 (2.0MB)
   used     = 0 (0.0MB)
   free     = 2097152 (2.0MB)
   0.0% used
PS Old Generation
   capacity = 631767040 (602.5MB)
   used     = 214841208 (204.88854217529297MB)
   free     = 416925832 (397.61145782470703MB)
   34.0063970415424% used
```

### 2.3 查看 thread

```sh
# 命令格式: jstack -l pid
[root@health-evaluate-service-test-deployment-d7bcd9448-jp4cr data]# jstack -l 1

# 查找qtp线程
[root@health-evaluate-service-test-deployment-d7bcd9448-jp4cr data]# jstack -l 1 |grep qtp
"qtp113676940-189" #189 prio=5 os_prio=0 tid=0x00007feba5390800 nid=0xc2 waiting on condition [0x00007feb55df3000]
"qtp113676940-186" #186 prio=5 os_prio=0 tid=0x00007feb2c00c800 nid=0xbe runnable [0x00007feb55ff5000]
"qtp113676940-177" #177 prio=5 os_prio=0 tid=0x00007feb20477000 nid=0xb6 waiting on condition [0x00007feaf6eaf000]
"qtp113676940-60" #60 prio=5 os_prio=0 tid=0x00007fec2df1f000 nid=0x47 runnable [0x00007feb55bf1000]
"qtp113676940-59" #59 prio=5 os_prio=0 tid=0x00007fec2df1c800 nid=0x46 runnable [0x00007feb55cf2000]
"qtp113676940-57-acceptor-0@29f48544-ServerConnector@4b07cad0{HTTP/1.1, (http/1.1)}{0.0.0.0:2750}" #57 prio=3 os_prio=0 tid=0x00007fec2df19000 nid=0x44 runnable [0x00007feb55ef4000]
"qtp113676940-54" #54 prio=5 os_prio=0 tid=0x00007fec2df13800 nid=0x41 waiting on condition [0x00007feb561f7000]
"qtp113676940-53" #53 prio=5 os_prio=0 tid=0x00007fec2df11800 nid=0x40 runnable [0x00007feb562f8000]
```

---

## 3. 回收算法

该章节讲述回收相关的算法.

### 3.1 回收判断

判断对象是否存活[link](https://www.cnblogs.com/ityouknow/p/5614961.html):

**引用计数**: 每个对象有一个引用计数属性,新增一个引用时计数+1,释放时计数减 -1,计数为 0 时可以回收.<u>此方法简单,无法解决对象相互循环引用的问题.</u>

**可达性分析(Reachability Analysis)**: 从 GC Roots 向下搜索,搜索所走过的路径称为引用链,当一个对象到 GC Roots 没有任何引用链相连时,此对象为不可达对象.

在 Java 语言中,GC Roots 包括:

- 虚拟机栈中引用的对象
- 方法区中类静态属性实体引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI 引用的对象

### 3.2 回收算法

---

## 4. dump

该章节主要是 dump 和 mat 的使用,[mat 使用教程 link](https://www.cnblogs.com/zh94/p/14051852.html)

### 4.1 dump

命令格式: `jmap -dump:file=${dumpFileName}.hprof,format=b ${pid}`

```sh
# 命令格式: jmap -dump:file=${dumpFileName}.hprof,format=b  ${pid}
[root@health-evaluate-service-prod-deployment-89df4bbdb-89rsl data]# jmap -dump:file=health-eva-prd.hprof,format=b 1
Dumping heap to /data/health-eva-prd.hprof ...
```

---

## 5. 参考文档

a. [jvm 相关参数配置 link](http://www.51gjie.com/java/551.html)

b. [java 运行时指定内存大小 link](https://blog.51cto.com/kankan/2488473)

c. [Java 堆内存设置 link](https://www.cnblogs.com/wx170119/p/10144512.html)

d. [主流 web 容器性能对比 link](https://www.cnblogs.com/maybo/p/7784687.html)

e. [垃圾回收算法 link](https://www.cnblogs.com/ityouknow/p/5614961.html)

f. [gc 优化 link](https://www.cnblogs.com/ityouknow/p/7653129.html)

g. [MAT 分析 dump 文件 link](https://www.cnblogs.com/zh94/p/14051852.html)
