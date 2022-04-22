# Arthas

Arthas: 阿尔萨斯.

---

## 1. 基础知识

### 1.1 下载安装

Q: [github link](https://github.com/alibaba/arthas/releases),直接使用`wget`命令会出现授权异常,有什么解决方案呀?

A: 可以使用`wget --no-check-certificate --content-disposition '${url}'`来下载.

```shell
[root@VM-12-17-centos arthas]# clear
[root@VM-12-17-centos arthas]# wget --no-check-certificate --content-disposition 'https://objects.githubusercontent.com/github-production-release-asset-2e65be/146633589/90bb976b-4e08-4f4b-8069-77c59f4b0d79?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20220421%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220421T015343Z&X-Amz-Expires=300&X-Amz-Signature=4114a7c244ab61bb9b4a880cc2b4e8c12b60a061bf7faaec120c2f70b226d109&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=146633589&response-content-disposition=attachment%3B%20filename%3Darthas-bin.zip&response-content-type=application%2Foctet-stream'
--2022-04-21 09:54:07--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/146633589/90bb976b-4e08-4f4b-8069-77c59f4b0d79?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20220421%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220421T015343Z&X-Amz-Expires=300&X-Amz-Signature=4114a7c244ab61bb9b4a880cc2b4e8c12b60a061bf7faaec120c2f70b226d109&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=146633589&response-content-disposition=attachment%3B%20filename%3Darthas-bin.zip&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.108.133, 185.199.111.133, 185.199.110.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.108.133|:443... connected.

[root@VM-12-17-centos arthas]# unzip arthas-bin.zip
[root@VM-12-17-centos arthas]# ls
arthas-agent.jar  arthas-boot.jar    arthas-core.jar    arthas-spy.jar  as-service.bat  async-profiler    lib          math-game.jar
arthas-bin.zip    arthas-client.jar  arthas.properties  as.bat          as.sh           install-local.sh  logback.xml
```

启动 arthas:

```shell
[root@VM-12-17-centos arthas]# java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.6.0
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
# 这里面输出的是jvm里面的进程
* [1]: 28543 /opt/soft/nacos/nacos/target/nacos-server.jar
# 输入下标来选取监听的进程,如1
1
```

```shell
[root@VM-12-17-centos arthas]# java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.6.0
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 28543 /opt/soft/nacos/nacos/target/nacos-server.jar
1
[INFO] arthas home: /opt/soft/arthas
[INFO] Try to attach process 28543
[INFO] Attach process 28543 success.
[INFO] arthas-client connect 127.0.0.1 3658
  ,---.  ,------. ,--------.,--.  ,--.  ,---.   ,---.
 /  O  \ |  .--. ''--.  .--'|  '--'  | /  O  \ '   .-'
|  .-.  ||  '--'.'   |  |   |  .--.  ||  .-.  |`.  `-.
|  | |  ||  |\  \    |  |   |  |  |  ||  | |  |.-'    |
`--' `--'`--' '--'   `--'   `--'  `--'`--' `--'`-----'

wiki       https://arthas.aliyun.com/doc
tutorials  https://arthas.aliyun.com/doc/arthas-tutorials.html
version    3.6.0
main_class
pid        28543
time       2022-04-21 10:06:15

# 查看进程资源看板
[arthas@28543]$ dashboard
```

Q: 但是在 mac 上面启动会报错

```shell
# mr3306 @ mr3306 in ~/Box/soft/arthas [21:42:30]
$ java -jar arthas-boot.jar
[INFO] arthas-boot version: 3.5.4
[INFO] Found existing java process, please choose one and input the serial number of the process, eg : 1. Then hit ENTER.
* [1]: 55354 org.jetbrains.kotlin.daemon.KotlinCompileDaemon
  [2]: 56202 com.weiqicha.WxappApplication
  [3]: 55276
  [4]: 56204 org.jetbrains.jps.cmdline.Launcher
2
[INFO] arthas home: /Users/mr3306/.arthas/lib/3.6.0/arthas
[INFO] Try to attach process 56202
Exception in thread "main" java.lang.IllegalArgumentException: Can not find tools.jar under java home: /Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home, please try to start arthas-boot with full path java. Such as /opt/jdk/bin/java -jar arthas-boot.jar
	at com.taobao.arthas.boot.ProcessUtils.findJavaHome(ProcessUtils.java:222)
	at com.taobao.arthas.boot.ProcessUtils.startArthasCore(ProcessUtils.java:233)
	at com.taobao.arthas.boot.Bootstrap.main(Bootstrap.java:572)
```

A: 需要设置 java 环境变量

```shell
# mr3306 @ mr3306 in ~/Box/soft/arthas [21:36:39]
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (3):
    1.8.311.11 (x86_64) "Oracle Corporation" - "Java" /Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home
    1.8.0_311 (x86_64) "Oracle Corporation" - "Java SE 8" /Library/Java/JavaVirtualMachines/jdk1.8.0_311.jdk/Contents/Home
    1.8.0_201 (x86_64) "Oracle Corporation" - "Java SE 8" /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home
/Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home
```

```shell
# root @ mr3306 in /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home [21:38:42]
$ cat /etc/profile
# java env
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home
export PATH=$JAVA_HOME/bin:$PATH

# root @ mr3306 in /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home [21:40:04]
$ source /etc/profile
```

### 1.2 常用命令

[arthas 常用命令 link](https://arthas.aliyun.com/doc/advanced-use.html)

| 命令      | 说明                                              | 使用案例                     | 备注                          |
| --------- | ------------------------------------------------- | ---------------------------- | ----------------------------- |
| dashboard | 进程资源看板                                      | dashboard                    |                               |
| monitor   | 方法执行监控                                      | monitor className methodName |                               |
| watch     | 方法执行数据观测                                  | watch className methodName   |                               |
| trace     | 方法内部调用路径,并输出方法路径上的每个节点上耗时 | trace className methodName   |                               |
| stack     | 输出当前方法被调用的调用路径                      | stack className methodName   |                               |
| sysenv    | 系统环境                                          | sysenv                       |                               |
| jvm       | jvm 运行环境                                      | jvm                          |                               |
| thread    | 线程监控                                          | thread -b                    | thread -b 查找被 block 的线程 |

---

## 2. 使用案例

### 2.1 接口耗时

Q: 在线上环境出现慢接口,但是之前没有相关切面或日志来监控执行耗时,请问有啥方法呀?

A: 这里刚好是 `trace` 的经典使用场景.

```shell
# 注意命令格式 trace className methodName
# 监测类: com.arthasproject.service.impl.UserInfoServiceImpl 方法: listDepts
# 在监测之后,需要进行一次请求之后才会被监测到
[arthas@13564]$ trace  com.arthasproject.service.impl.UserInfoServiceImpl listDepts
trace  com.arthasproject.service.impl.UserInfoServiceImpl listDepts
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 254 ms, listenerId: 4

`---ts=2022-04-21 10:21:18;thread_name=XNIO-1 task-1;id=108;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[6255.563001ms] com.arthasproject.service.impl.UserInfoServiceImpl$$EnhancerBySpringCGLIB$$1:listDepts()
        `---[6255.5409ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[6255.4914ms] com.arthasproject.service.impl.UserInfoServiceImpl:listDepts()
                +---[6222.0164ms] com.arthasproject.service.feign.DataSyncFeignService:listDepts() #175
                +---[0.0061ms] com.arthasproject.model.vo.page.PageResult:getData() #177
                +---[0.005199ms] cn.hutool.core.collection.CollectionUtil:isNotEmpty() #178
                +---[0.024401ms] com.baomidou.mybatisplus.core.toolkit.Wrappers:lambdaQuery() #180
                +---[0.0289ms] com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper:in() #180
                `---[32.7122ms] com.arthasproject.service.DeptMappingService:list() #180
```

A: 从上面的监测可以看到是`com.arthasproject.service.feign.DataSyncFeignService:listDepts() #175`执行耗时是最高的.如果需要优化,就要重点关注这一个接口.

### 2.2 调用栈

```shell
[arthas@172856]$ stack com.arthasproject.service.impl.UserInfoServiceImpl listDepts
stack com.arthasproject.service.impl.UserInfoServiceImpl listDepts
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 203 ms, listenerId: 2

ts=2022-04-21 10:56:29;thread_name=XNIO-1 task-1;id=157;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    @com.arthasproject.service.impl.UserInfoServiceImpl.listDepts()
        at com.arthasproject.service.impl.UserInfoServiceImpl$$FastClassBySpringCGLIB$$1.invoke(<generated>:-1)
        at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
        at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:771)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
        at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:749)
        at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:95)
```

Q: 上面这种也有很多不必要的信息,可以过滤一下没?

A: as you wish.

```shell
[arthas@172856]$ stack com.arthasproject.service.impl.UserInfoServiceImpl listDepts | grep 'com.arthasproject' > stack.txt
```

现在在 stack.txt 输出的内容如下:

```text
    @com.arthasproject.service.impl.UserInfoServiceImpl.listDepts()
        at com.arthasproject.service.impl.UserInfoServiceImpl$$FastClassBySpringCGLIB$$1.invoke(<generated>:-1)
        at com.arthasproject.service.impl.UserInfoServiceImpl$$EnhancerBySpringCGLIB$$1.listDepts(<generated>:-1)
        at com.arthasproject.controller.CommonController.listDept$original$P7QGrkQo(CommonController.java:104)
        at com.arthasproject.controller.CommonController.listDept$original$P7QGrkQo$accessor$MNZjf4Pp(CommonController.java:-1)
        at com.arthasproject.controller.CommonController$auxiliary$JsAN8YS9.call(null:-1)
        at com.arthasproject.controller.CommonController.listDept(CommonController.java:-1)
        at com.arthasproject.controller.CommonController$$FastClassBySpringCGLIB$$1.invoke(<generated>:-1)
        at com.arthasproject.controller.CommonController$$EnhancerBySpringCGLIB$$1.listDept(<generated>:-1)
    @com.arthasproject.service.impl.UserInfoServiceImpl$$EnhancerBySpringCGLIB$$1.listDepts()
        at com.arthasproject.controller.CommonController.listDept$original$P7QGrkQo(CommonController.java:104)
        at com.arthasproject.controller.CommonController.listDept$original$P7QGrkQo$accessor$MNZjf4Pp(CommonController.java:-1)
        at com.arthasproject.controller.CommonController$auxiliary$JsAN8YS9.call(null:-1)
        at com.arthasproject.controller.CommonController.listDept(CommonController.java:-1)
        at com.arthasproject.controller.CommonController$$FastClassBySpringCGLIB$$1.invoke(<generated>:-1)
        at com.arthasproject.controller.CommonController$$EnhancerBySpringCGLIB$$1.listDept(<generated>:-1)
```

### 2.3 监测参数

可以使用`watch`来监测返回参数啥的,不过看起来没啥作用.可以看看是不是返回 null 啥的. orz

Q: 如果想知道执行方法的传递参数和返回参数是啥,但是因为线上没日志,该咋办?

A: 这里可以使用 arthas 的 `watch` 来做监听. 各种黑魔法就对了.

监听返回参数

```shell
[arthas@56415]$ watch com.weiqicha.wxapp.controller.UserController comment '{returnObj}' -x 3
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 384 ms, listenerId: 1
method=com.weiqicha.wxapp.controller.UserController.comment location=AtExit
ts=2022-04-22 21:49:18; [cost=253.667698ms] result=@ArrayList[
    @SingleResponse[
        code=@Integer[0],
        tip=@String[成功],
        data=@Boolean[true],
    ],
]
method=com.weiqicha.wxapp.controller.UserController$$EnhancerBySpringCGLIB$$1.comment location=AtExit
ts=2022-04-22 21:49:18; [cost=478.877372ms] result=@ArrayList[
    @SingleResponse[
        code=@Integer[0],
        tip=@String[成功],
        data=@Boolean[true],
    ],
]
```

监听请求参数

```shell
[arthas@56415]$ watch com.weiqicha.wxapp.controller.UserController comment '{params}' -x 4
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 159 ms, listenerId: 5
method=com.weiqicha.wxapp.controller.UserController.comment location=AtExit
ts=2022-04-22 21:53:46; [cost=291.735941ms] result=@ArrayList[
    @Object[][
        @UserCommentReq[
            form=@Form[
                comment=@String[1345],
            ],
            pictures=@ArrayList[
                @String[1],
                @String[连接1],
            ],
        ],
    ],
]
method=com.weiqicha.wxapp.controller.UserController$$EnhancerBySpringCGLIB$$1.comment location=AtExit
ts=2022-04-22 21:53:46; [cost=294.679299ms] result=@ArrayList[
    @Object[][
        @UserCommentReq[
            form=@Form[
                comment=@String[1345],
            ],
            pictures=@ArrayList[
                @String[1],
                @String[连接1],
            ],
        ],
    ],
]
```

监听请求/返回参数

```shell
[arthas@56415]$ watch com.weiqicha.wxapp.controller.UserController comment '{params,returnObj}' -x 4
```

### 2.4 monitor

用于监测方法执行相关统计.

```shell
[arthas@172856]$ monitor com.arthasproject.service.impl.UserInfoServiceImpl listDepts
sonitor com.arthasproject.service.impl.UserInfoServiceImpl listDept
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 215 ms, listenerId: 4

 timestamp   class             method             tota  succ  fail  avg-  fail
                                                  l     ess         rt(m  -rat
                                                                    s)    e
-------------------------------------------------------------------------------
 2022-04-21  com.arthasproject.ser  listDepts          2     2     0     5894  0.00
  11:07:38   vice.impl.UserIn                                       .89   %
             foServiceImpl
 2022-04-21  com.arthasproject.ser  listDepts          2     2     0     5896  0.00
  11:07:38   vice.impl.UserIn                                       .51   %
             foServiceImpl$$E
             nhancerBySpringC
             GLIB$$1
```

### 2.5 反编译

Q: 在服务器上发现慢接口,但是因为代码不在旁边的情况,如果需要看详细的实现逻辑要怎么处理呀?

A: 可以使用 arthas 的反编译来查看相关源码.

```shell
# 使用2.1里面的例子来反编译
[arthas@13564]$ jad com.arthasproject.service.impl.UserInfoServiceImpl
jad com.arthasproject.service.impl.UserInfoServiceImpl
```

Q: 上面是可以输出源码,如果源码很大,一大堆打印到界面上,体验真不好.

A: 那样子可以输出到文件里面,再使用 `vim` 来进行查看等操作.

```shell
# 将反编译内容输出到文件
[arthas@13564]$ jad com.arthasproject.service.impl.UserInfoServiceImpl > code.txt
# 查看文件所在目录
[arthas@13564]$ pwd
```

### 2.6 jvm 参数

可以通过`jvm`命令来查看当前进程的一些`jvm`运行参数.那样子就知道自己设置的一些参数是否生效了.

```shell
[arthas@28543]$ jvm
 RUNTIME
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 MACHINE-NAME                                 28543@VM-12-17-centos
 JVM-START-TIME                               2022-04-19 16:39:17
 MANAGEMENT-SPEC-VERSION                      1.2
 SPEC-NAME                                    Java Virtual Machine Specification
 SPEC-VENDOR                                  Oracle Corporation
 SPEC-VERSION                                 1.8
 VM-NAME                                      OpenJDK 64-Bit Server VM
 VM-VENDOR                                    Red Hat, Inc.
 VM-VERSION                                   25.312-b07
 INPUT-ARGUMENTS                              -Djava.ext.dirs=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64/jre/lib/ext:/usr/lib/jvm/java-1.8.0
                                              -openjdk-1.8.0.312.b07-1.el7_9.x86_64/lib/ext
                                              -Xms512m
                                              -Xmx512m
                                              -Xmn256m
```

### 2.7 系统环境

Q: 在某些时候我们想知道运行的系统环境是啥,要怎么知道呀?

A: 可以使用`sysenv`来查看.

```shell
[arthas@28543]$ sysenv
 KEY                             VALUE
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 JAVA                            /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64/bin/java
 PATH                            /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/opt/soft/maven/apache-maven-3.8.4/bin:/opt/soft/tomcat/apache-tomcat-10.0.
                                 14/bin:/opt/soft/nodejs/node-v14.18.3-linux-x64/bin:/root/bin
 HISTSIZE                        3000
 MODE                            standalone
 JAVA_HOME                       /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64
...
```

---

## 3. 参考文档

a. [arthas 官方文档](https://arthas.aliyun.com/doc/quick-start.html)

b. [arthas 进阶命令](https://arthas.aliyun.com/doc/advanced-use.html)

c. [arthas 使用案例](https://github.com/alibaba/arthas/issues?q=label%3Auser-case)
