# Java tools

在生产环境里面通常是一片漆黑的 linux,并没有 jvisiual vm 这种图形监测工具.

所以,jdk 里面自带的各种小工具可以排上用场了.该文档整理参考[码云-cowboy](https://gitee.com/cowboy2016).

---

## 1. jps

显示服务器上面的正在运行的 java 进程.


### 1.1 命令参数

| 参数名称 | 功能                               |
| :------: | ---------------------------------- |
|    l     | 输出主类全名或 jar 路径            |
|    q     | 只输出 LVMID                       |
|    m     | 输出 JVM 启动时传递给 main()的参数 |
|    v     | 输出 JVM 启动时显示指定的 JVM 参数 |


### 1.2 测试例子

```sh
[root@team-2 ~]# jps -lm |grep -v 'Jps'
19844 app/4fun-zhihu-0.0.1-SNAPSHOT.jar
26565 org.apache.catalina.startup.Bootstrap start
24103 app/4fun-service-0.0.1-SNAPSHOT.jar
```



---

## 2. jstat 

jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令,它可以显示出虚拟机进程中的类装载,内存,垃圾收集,JIT编译等运行数据.

### 2.1 命令参数

| 参数名称         | 功能                                                       |
| ---------------- | ---------------------------------------------------------- |
| class            | class loader的行为统计                                     |
| compiler         | HotSpt JIT编译器行为统计                                   |
| gc               | 垃圾回收堆的行为统计                                       |
| gccapacity       | 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计     |
| gcutil           | 垃圾回收统计概述                                           |
| gccause          | 垃圾收集统计概述(同-gcutil),附加最近两次垃圾回收事件的原因 |
| gcnew            | 新生代行为统计                                             |
| gcnewcapacity    | 新生代与其相应的内存空间的统计                             |
| gcold            | 年老代和永生代行为统计                                     |
| gcoldcapacity    | 年老代行为统计                                             |
| gcpermcapacity   | 永生代行为统计                                             |
| printcompilation | HotSpot编译方法统计                                        |


### 2.2 测试例子

使用看java进程的gc情况

```sh
[root@team-2 ~]# jstat -gc 19844 5s
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
3968.0 3968.0 2564.9  0.0   32320.0  17476.3   80356.0    39242.3   28160.0 27576.1 3072.0 2889.1  84316  339.274  116    10.107  349.381
3968.0 3968.0 2564.9  0.0   32320.0  17476.3   80356.0    39242.3   28160.0 27576.1 3072.0 2889.1  84316  339.274  116    10.107  349.381
3968.0 3968.0 2564.9  0.0   32320.0  17476.3   80356.0    39242.3   28160.0 27576.1 3072.0 2889.1  84316  339.274  116    10.107  349.381
```
- s0c: 幸存区1容量  s0u: 幸存区1使用大小
- ec: 伊甸园区容量  eu: 伊甸园区使用大小
- oc: 老年代容量    ou: 老年代使用大小
- mc: 元数据容量    mu: 元数据使用大小
- YGC: 年轻代gc次数  YGCT: ygc所花总时长
- FGC: 全局gc次数    FGCT: fullgc所花总时长


---

## 3. jmap

jmap(JVM Memory Map)命令用于生成heap dump文件,如果不使用这个命令,还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候.自动生成dump文件.jmap不仅能生成dump文件,还可以查询finalize执行队列,Java堆和永久代的详细信息,如当前使用率,当前使用的是哪种收集器等.


### 3.1 命令参数

| 参数名称      | 功能                                                      |
| ------------- | --------------------------------------------------------- |
| dump          | 生成堆转储快照                                            |
| finalizerinfo | 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象 |
| heap          | 显示Java堆详细信息                                        |
| histo         | 显示堆中对象的统计信息                                    |
| permstat      | 显示永久代信息                                            |
| F             | 当-dump没有响应时，强制生成dump快照                       |


下面看看两个常用的命令

### 3.2 测试例子:dump

生成内存快照,可以用来分析当前系统的内存使用情况.

```sh
[root@team-2 ~]# jmap -dump:live,format=b,file=/root/dump.hprof 19844 
Dumping heap to /root/dump.hprof ...
Heap dump file created
[root@team-2 ~]# ll
total 261728
-rw------- 1 root root  28775499 Mar 28 16:25 dump.hprof
```

### 3.3 测试例子:heap

打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况,可以用此来判断内存目前的使用情况以及垃圾回收情况

```sh
[root@team-2 ~]# jmap -heap 19844 
Attaching to process ID 19844, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.144-b01

using thread-local object allocation.
Mark Sweep Compact GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 482344960 (460.0MB)
   NewSize                  = 10485760 (10.0MB)
   MaxNewSize               = 160759808 (153.3125MB)
   OldSize                  = 20971520 (20.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 37158912 (35.4375MB)
   used     = 18647560 (17.78369903564453MB)
   free     = 18511352 (17.65380096435547MB)
   50.18327770199515% used
Eden Space:
   capacity = 33095680 (31.5625MB)
   used     = 15830216 (15.096870422363281MB)
   free     = 17265464 (16.46562957763672MB)
   47.831668664913366% used
From Space:
   capacity = 4063232 (3.875MB)
   used     = 2817344 (2.68682861328125MB)
   free     = 1245888 (1.18817138671875MB)
   69.33751260080645% used
To Space:
   capacity = 4063232 (3.875MB)
   used     = 0 (0.0MB)
   free     = 4063232 (3.875MB)
   0.0% used
tenured generation:
   capacity = 82284544 (78.47265625MB)
   used     = 40185480 (38.32386016845703MB)
   free     = 42099064 (40.14879608154297MB)
   48.83721540706357% used

14148 interned Strings occupying 1610448 bytes.
```

---

## 4. jstack

jstack用于生成java虚拟机当前时刻的线程快照.线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合,生成线程快照的主要目的是定位线程出现长时间停顿的原因,如线程间死锁,死循环,请求外部资源导致的长时间等待等.线程出现停顿的时候通过jstack来查看各个线程的调用堆栈,就可以知道没有响应的线程到底在后台做什么事情,或者等待什么资源. 如果java程序崩溃生成core文件,jstack工具可以用来获得core文件的java stack和native stack的信息,从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题.另外,jstack工具还可以附属到正在运行的java程序中,看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态,jstack是非常有用的.

### 4.1 命令参数

| 参数名称 | 功能                                       |
| :------: | ------------------------------------------ |
|    F     | 当正常输出请求不被响应时,强制输出线程堆栈  |
|    l     | 除堆栈外,显示关于锁的附加信息              |
|    m     | 如果调用到本地方法的话,可以显示C/C++的堆栈 |



### 4.2 测试例子

```shell
[root@team-2 ~]# jstack 19844
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.144-b01 mixed mode):

"Attach Listener" #32 daemon prio=9 os_prio=0 tid=0x00007ffb04003000 nid=0x46f8 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"spider-thread:5" #20 prio=5 os_prio=0 tid=0x00007ffafc2e5000 nid=0x4dac waiting on condition [0x00007ffb08787000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000ecf3f860> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingDeque.takeFirst(LinkedBlockingDeque.java:492)
	at java.util.concurrent.LinkedBlockingDeque.take(LinkedBlockingDeque.java:680)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
...
```



---

## 5. jinfo

jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数. 之前的jps -v口令只能查看到显示指定的参数,如果想要查看未被显示指定的参数的值就要使用jinfo口令.

### 5.1 命令参数

| 参数名称 | 功能                                       |
| :------: | ------------------------------------------ |
|   flag   | 输出指定args参数的值                       |
|  flags   | 不需要args参数，输出所有JVM参数的值        |
| sysprops | 输出系统属性，等同于System.getProperties() |

### 5.2 测试例子

```sh
[root@team-2 ~]# jinfo -sysprops  19844
Attaching to process ID 19844, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.144-b01
Java System Properties:

com.sun.management.jmxremote.authenticate = false
java.runtime.name = Java(TM) SE Runtime Environment
java.vm.version = 25.144-b01
sun.boot.library.path = /opt/soft/jdk/jdk1.8/jre/lib/amd64
java.vendor.url = http://java.oracle.com/
java.vm.vendor = Oracle Corporation
path.separator = :
java.rmi.server.randomIDs = true
file.encoding.pkg = sun.io
java.vm.name = Java HotSpot(TM) 64-Bit Server VM
sun.os.patch.level = unknown
...
```