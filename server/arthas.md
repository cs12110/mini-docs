# Arthas

好像挺好玩的样子. 男默女泪.jpg

---

### 1. 安装与运行

#### 1.1 安装

安装就是去下载一个 jar 即可,满分. :"}

```shell
# root @ team3 in ~ [10:19:45]
$ curl -O https://arthas.aliyun.com/arthas-boot.jar

# root @ team3 in /opt/soft/arthas [10:20:36]
$ ls -al -h  |grep arthas
-rw-r--r--  1 root root 139K Dec  9 22:57 arthas-boot.jar
```

#### 1.2 运行

启动 arthas

```shell
# root @ team3 in /opt/soft/arthas [10:21:08]
$ java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.5.4
[INFO] Process 31527 already using port 3658
[INFO] Process 31527 already using port 8563
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 31527 police-screen-project-0.0.1-SNAPSHOT.jar
  [2]: 534 org.apache.catalina.startup.Bootstrap

# 输入所需要监测的下标,如1,回车即可.


[arthas@31527]$ dashboard
ID  NAME                     GROUP        PRIORIT STATE   %CPU     DELTA_T TIME    INTERRUP DAEMON
35  System Clock             main         5       TIMED_W 0.0      0.000   2659:39 false    true
-1  VM Periodic Task Thread  -            -1      -       0.0      0.000   71:27.4 false    true
-1  VM Thread                -            -1      -       0.0      0.000   9:45.83 false    true
11  Catalina-utility-1       main         1       TIMED_W 0.0      0.000   5:13.07 false    false
12  Catalina-utility-2       main         1       WAITING 0.0      0.000   5:6.431 false    false
14  mysql-cj-abandoned-conne main         5       TIMED_W 0.0      0.000   4:2.408 false    true
26  http-nio-3222-ClientPoll main         5       RUNNABL 0.0      0.000   3:47.37 false    true
15  http-nio-3222-BlockPolle main         5       RUNNABL 0.0      0.000   3:9.095 false    true
-1  C2 CompilerThread0       -            -1      -       0.0      0.000   1:43.89 false    true
30  HikariPool-1 housekeeper main         5       TIMED_W 0.0      0.000   1:27.65 false    true
-1  C1 CompilerThread1       -            -1      -       0.0      0.000   0:40.59 false    true
13  container-0              main         5       TIMED_W 0.0      0.000   0:20.84 false    false
Memory               used    total  max    usage  GC
heap                 174M    535M   910M   19.17% gc.ps_scavenge.count     3881
ps_eden_space        64M     147M   318M   20.30% gc.ps_scavenge.time(ms)  32693
ps_survivor_space    8M      11M    11M    70.67% gc.ps_marksweep.count    3
ps_old_gen           101M    376M   683M   14.91% gc.ps_marksweep.time(ms) 445
nonheap              124M    129M   -1     95.91%
code_cache           40M     41M    240M   16.72%
metaspace            75M     79M    -1     95.63%
compressed_class_spa 8M      9M     1024M  0.84%
ce
direct               80K     80K    -
Runtime
os.name                                           Linux
os.version                                        3.10.0-862.el7.x86_64
java.version                                      1.8.0_144
java.home                                         /opt/soft/jdk/jdk8/jre
systemload.average                                0.02
processors                                        2
timestamp/uptime                                  Fri Dec 10 10:24:12 CST 2021/11110689s
ID  NAME                     GROUP        PRIORIT STATE   %CPU     DELTA_T TIME    INTERRUP DAEMON
35  System Clock             main         5       TIMED_W 1.51     0.075   2659:39 false    true
-1  C2 CompilerThread0       -            -1      -       0.82     0.041   1:43.93 false    true
-1  C1 CompilerThread1       -            -1      -       0.14     0.007   0:40.60 false    true
553 Timer-for-arthas-dashboa system       5       RUNNABL 0.06     0.003   0:0.003 false    true
-1  VM Periodic Task Thread  -            -1      -       0.04     0.002   71:27.4 false    true
553 arthas-NettyHttpTelnetBo system       5       RUNNABL 0.01     0.000   0:0.017 false    true
-1  VM Thread                -            -1      -       0.01     0.000   9:45.83 false    true
11  Catalina-utility-1       main         1       TIMED_W 0.0      0.000   5:13.07 false    false
14  mysql-cj-abandoned-conne main         5       TIMED_W 0.0      0.000   4:2.409 false    true
26  http-nio-3222-ClientPoll main         5       RUNNABL 0.0      0.000   3:47.37 false    true
15  http-nio-3222-BlockPolle main         5       RUNNABL 0.0      0.000   3:9.095 false    true
12  Catalina-utility-2       main         1       WAITING 0.0      0.000   5:6.431 false    false
Memory               used    total  max    usage  GC
heap                 174M    535M   910M   19.21% gc.ps_scavenge.count     3881
ps_eden_space        64M     147M   318M   20.42% gc.ps_scavenge.time(ms)  32693
ps_survivor_space    8M      11M    11M    70.67% gc.ps_marksweep.count    3
ps_old_gen           101M    376M   683M   14.91% gc.ps_marksweep.time(ms) 445
nonheap              124M    129M   -1     95.93%
code_cache           40M     41M    240M   16.73%
metaspace            75M     79M    -1     95.63%
compressed_class_spa 8M      9M     1024M  0.84%
ce
direct               80K     80K    -
Runtime
os.name                                           Linux
os.version                                        3.10.0-862.el7.x86_64
java.version                                      1.8.0_144
java.home                                         /opt/soft/jdk/jdk8/jre
systemload.average                                0.01
processors                                        2
timestamp/uptime                                  Fri Dec 10 10:24:17 CST 2021/11110694s
```

Q: <u style='color:pink'>进入 arthas 之后,使用 jad 反编译 class 的时候,中文出现乱码该咋办?</u>

A: 可以在启动 arthas 的指定字符集编码,如下所示:

```shell
# root @ team3 in /opt/soft/arthas [10:21:08]
$ java -jar -Dfile.encoding=UTF-8 arthas-boot.jar
```

---

### 2. 使用指南

#### 2.1 常用命令

[命令详细解析 link](https://arthas.aliyun.com/doc/commands.html)

| 命名      | 释义                                              | 备注 | 示例                                   |
| --------- | ------------------------------------------------- | ---- | -------------------------------------- |
| dashboard | 系统的实时数据面板,按 ctrl+c 退出                 |      | dashboard                              |
| thread    | 查看当前线程信息,查看线程的堆栈                   |      | thread {threadId}                      |
| jvm       | 查看当前 JVM 信息                                 |      | jvm                                    |
| jad       | 反编译指定已加载类的源码                          |      | jad {pkgName.className}                |
| trace     | 方法内部调用路径,并输出方法路径上的每个节点上耗时 |      | trace {pkgName.className} {methodName} |
| sc        | 查看 JVM 已加载的类信息                           |      | sc \*{className}\*                     |
| sc        | 查看 JVM 已加载的类信息                           |      | sc \*{className}\*                     |

Q: <u>如果 trace 出现如下异常,该怎么处理?</u>

正常的 trace 如下:

```shell
[arthas@48449]$ trace controller.DemoController *
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 66) cost in 548 ms, listenerId: 1
`---ts=2021-12-10 13:29:54;thread_name=XNIO-1 task-1;id=112;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[53.336614ms] controller.DemoController$$EnhancerBySpringCGLIB$$1:mockFeishuLoginInfo()
        `---[52.060714ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[47.804842ms] controller.DemoController:mockFeishuLoginInfo()
                +---[0.050369ms] usercenter.resp.UserLoginRespDTO:getUserId() #106
                +---[0.582965ms] com.alibaba.fastjson.JSON:toJSONString() #107
                +---[46.685069ms] common.redis.RedisService:set() #107
                `---[0.019332ms] backend.data.domain.Results:success() #109
```

但是会出现异常,在某些类里面:

```shell
[arthas@1]$ trace controller.UserController login
Affect(class count: 1 , method count: 1) cost in 38 ms, listenerId: 3
Enhance error! exception: java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the schema (add/remove fields)
error happens when enhancing class: class redefinition failed: attempted to change the schema (add/remove fields), check arthas log: /home/sipaiuser/logs/arthas/arthas.log
```

A: 查看日志可以看到异常,应该是 springboot 代理导致的. orz

```java
2021-12-10 10:48:33 [arthas-command-execute] ERROR c.t.arthas.core.advisor.Enhancer -Enhancer error, matchingClasses: [class controller.UserController, class controller.UserController$$EnhancerBySpringCGLIB$$2e1d108]
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the schema (add/remove fields)
```

#### 2.2 应用场景

---

### 参考资料

a. [Arthas 官网 link](https://arthas.aliyun.com/doc/quick-start.html)
