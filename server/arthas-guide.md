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
  [2]: 56202 com.arthasprojectApplication
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

| 命令      | 说明                                              | 使用案例                             | 备注                          |
| --------- | ------------------------------------------------- | ------------------------------------ | ----------------------------- |
| dashboard | 进程资源看板                                      | dashboard                            |                               |
| monitor   | 方法执行监控                                      | `monitor ${className} ${methodName}` |                               |
| watch     | 方法执行数据观测                                  | `watch ${className} ${methodName}`   |                               |
| trace     | 方法内部调用路径,并输出方法路径上的每个节点上耗时 | `trace ${className} ${methodName}`   |                               |
| stack     | 输出当前方法被调用的调用路径                      | `stack ${className} ${methodName}`   |                               |
| sysenv    | 系统环境                                          | sysenv                               |                               |
| jvm       | jvm 运行环境                                      | jvm                                  |                               |
| thread    | 线程监控                                          | thread -b                            | thread -b 查找被 block 的线程 |

---

## 2. 使用案例

### 2.1 接口耗时

Q: 在线上环境出现慢接口,但是之前没有相关切面或日志来监控执行耗时,请问有啥方法呀?

A: 这里刚好是 `trace` 的经典使用场景. 还有一个要注意的点就是:监测的方法/类必须在 jvm 里面存在.

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

Q: 那么怎么确定类或者方法是否在 jvm 里面存在呀?

A: 可以使用`sm(Search-Method)`方法来看一下

```shell
# sm可以看到现在有哪些类
[arthas@47954]$ sm com.arthasproject.
com.arthasproject.config.     com.arthasproject.controller. com.arthasproject.model.      com.arthasproject.mapper.
com.arthasproject.enums.      com.arthasproject.common.     com.arthasproject.service.    com.arthasproject.utils.
com.arthasproject.job.        com.arthasproject.api.        com.arthasproject.entity.

#
[arthas@47954]$ sm com.arthasproject.controller.LoanController
detailByBarcode     lambda$list$0       importLoanInfo      balance             del
draft               approval            list                $deserializeLambda$ reject
detail              submit

# 比如listener需要在某些方法执行完之后new出来,在没执行那些方法前就没办法被监听.
[arthas@47954]$ sm com.arthasproject.controller.LoanController balance
com.arthasproject.controller.LoanController$$EnhancerBySpringCGLIB$$1 balance(Ljava/lang/String;)Lcom/dslyy/fd/model/Result;
com.arthasproject.controller.LoanController balance(Ljava/lang/String;)Lcom/dslyy/fd/model/Result;
Affect(row-cnt:2) cost in 28 ms.
```

Q: 如果监听那些执行很多次的方法,那岂不是一直在输出?

A: 使用 `-n {times}`来指定监听次数,如: `trace com.arthasproject.service.impl.UserInfoServiceImpl listDepts -n 2`.

Q: 那怎么追踪 `lambda 表达式`里面的执行呀? 如下所示的追踪完全没看见 lambda 的踪影.

```java
@GetMapping("/lambda-exp")
public SingleResponse<List<String>> lambdaExp() {

    List<String> list = new ArrayList<>();
    list.add("1");
    list.add("2");
    list.add("3");

    List<String> values = list.stream().map((e) -> {
        try {
            Thread.sleep(1000);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return e;
    }).collect(Collectors.toList());

    return SingleResponse.of(values);
}
```

```shell
[arthas@50861]$ trace com.arthasproject.controller.ContactBookController lambdaExp
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 271 ms, listenerId: 1
`---ts=2022-04-28 21:47:09;thread_name=http-nio-0.0.0.0-8019-exec-1;id=7b;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@29436cb2
    `---[3098.66037ms] com.arthasproject.controller.ContactBookController$$EnhancerBySpringCGLIB$$1:lambdaExp()
        `---[3097.878743ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[3026.676764ms] com.arthasproject.controller.ContactBookController:lambdaExp()
                `---[5.281318ms] com.arthasproject.dto.resp.SingleResponse:of() #82
```

A: 怎么那么狗呀.先`trace`整一个类.

```java
[arthas@50861]$ trace com.arthasproject.controller.ContactBookController *
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 59) cost in 289 ms, listenerId: 2
`---ts=2022-04-28 21:48:04;thread_name=http-nio-0.0.0.0-8019-exec-2;id=7c;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@29436cb2
    `---[3020.168684ms] com.arthasproject.controller.ContactBookController$$EnhancerBySpringCGLIB$$1:lambdaExp()
        `---[3020.060573ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[3017.951143ms] com.arthasproject.controller.ContactBookController:lambdaExp()
                +---[min=1003.874864ms,max=1005.177331ms,total=3013.368565ms,count=3] com.arthasproject.controller.ContactBookController:lambda$lambdaExp$0()
                `---[0.037015ms] com.arthasproject.dto.resp.SingleResponse:of() #82
```

从上面可以看出来 lambda 表达式是:`lambda$lambdaExp$0()`,那我们切换一下监听: `trace ${class} *${methodName}*`.

```shell
[arthas@50861]$ trace com.arthasproject.controller.ContactBookController *lambdaExp*
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 4) cost in 161 ms, listenerId: 3
`---ts=2022-04-28 21:49:31;thread_name=http-nio-0.0.0.0-8019-exec-4;id=7e;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@29436cb2
    `---[3013.243714ms] com.arthasproject.controller.ContactBookController$$EnhancerBySpringCGLIB$$1:lambdaExp()
        `---[3013.132309ms] org.springframework.cglib.proxy.MethodInterceptor:intercept()
            `---[3010.29154ms] com.arthasproject.controller.ContactBookController:lambdaExp()
                +---[min=1000.431794ms,max=1004.208051ms,total=3005.817471ms,count=3] com.arthasproject.controller.ContactBookController:lambda$lambdaExp$0()
                `---[0.028095ms] com.arthasproject.dto.resp.SingleResponse:of() #82
```

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

Q: 如果想知道执行方法的传递参数和返回参数是啥,但是因为线上没日志,该咋办?

A: 这里可以使用 arthas 的 `watch` 来做监听. 各种黑魔法就对了.

监听返回参数

```shell
[arthas@56415]$ watch com.arthasproject.controller.UserController comment '{returnObj}' -x 3
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 384 ms, listenerId: 1
method=com.arthasproject.controller.UserController.comment location=AtExit
ts=2022-04-22 21:49:18; [cost=253.667698ms] result=@ArrayList[
    @SingleResponse[
        code=@Integer[0],
        tip=@String[成功],
        data=@Boolean[true],
    ],
]
method=com.arthasproject.controller.UserController$$EnhancerBySpringCGLIB$$1.comment location=AtExit
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
[arthas@56415]$ watch com.arthasproject.controller.UserController comment '{params}' -x 4
Press Q or Ctrl+C to abort.
Affect(class count: 2 , method count: 2) cost in 159 ms, listenerId: 5
method=com.arthasproject.controller.UserController.comment location=AtExit
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
method=com.arthasproject.controller.UserController$$EnhancerBySpringCGLIB$$1.comment location=AtExit
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
[arthas@56415]$ watch com.arthasproject.controller.UserController comment '{params,returnObj}' -x 4
```

当然也可以监听异常啦

```shell
[arthas@56415]$ watch com.arthasproject.controller.UserController comment '{params,returnObj,throwExp}' -x 4 -e
```

Q: 为啥监控进过 skyworking 代理的东西会出现如下问题?

```java
[arthas@8]$ watch com.arthasproject.controller.LoginController login '{params,returnObj}' -x 4
Error during processing the command: class redefinition failed: attempted to change the schema (add/remove fields)
[arthas@8]$ watch com.arthasproject.controller.LoginController login '{params}' -x 4
Error during processing the command: class redefinition failed: attempted to change the schema (add/remove fields)
[arthas@8]$ [dslyun@caiwu-uat ~]$
```

A: 好像经过 skywalking 这种代理之后需要特殊配置.[github link](https://github.com/apache/skywalking/blob/master/docs/en/FAQ/Compatible-with-other-javaagent-bytecode-processing.md)

```java
-Dskywalking.agent.is_cache_enhanced_class=true -Dskywalking.agent.class_cache_mode=MEMORY
```

经过测试,按照上述配置可以解决问题,参考配置如下:

```shell
-javaagent:/Users/mr3306/Box/soft/idea/skywalking/skywalking-agent/skywalking-agent.jar
-Dskywalking.agent.service_name=${appName}
-Dskywalking.collector.backend_service=${skywalkingIp}:${skywalkingPort}
-Dskywalking.agent.is_cache_enhanced_class=true
-Dskywalking.agent.class_cache_mode=MEMORY
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

A: 也可以指定反编译某一个方法的.可以使用命令:`jad ${className} ${methodName}`.

Q: 如果方法都有几千行代码呢? `:(`

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

### 2.8 redefine

Q: 上面提到使用 watch 来追踪数据,但如果里面的逻辑很复杂,需要日志来追踪咋办?

A: 这时候就可以使用 redefine 了.

Q: 那这个有什么限制呀?

A: 重启之后热加载的代码会失效,不能改变方法的接收参数,不能改变方法的返回参数.

最开始简单的接口

```java
package com.queen.ctrl;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/west-world")
public class WestWorldController {

    @RequestMapping("/portal")
    public Map<String, Object> portal() {
        Map<String, Object> map = new HashMap<>();
        map.put("portal", "cs12110");

        return map;
    }
}
```

修改后代码

```java
package com.queen.ctrl;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/west-world")
public class WestWorldController {

    @RequestMapping("/portal")
    public Map<String, Object> portal() {
        Map<String, Object> map = new HashMap<>();
        map.put("portal", "cs12110");

        // 添加日志打印
        // 注意: 这个不能变成 private static Logger logger = LoggerFactory.getLogger(WestWorldController.class)作为类属性,因为改变了这个类的field从而导致加载失败.
        Logger logger = LoggerFactory.getLogger(WestWorldController.class);
        logger.info("Function[portal] map:{}", map);

        return map;
    }
}
```

进行热加载

```shell
[arthas@36632]$ redefine /project/live/queen-project/target/classes/com/queen/ctrl/WestWorldController.class
/ctrl/WestWorldController.classproject/target/classes/com/queen
redefine success, size: 1, classes:
com.queen.ctrl.WestWorldController
```

加载完重新请求接口

```text
2022-04-25 11:15:34.625  INFO 36632 --- [nio-6336-exec-6] com.queen.ctrl.WestWorldController       : Function[portal] map:{portal=cs12110}
```

### 2.9 类的详细信息

```shell
[arthas@36632]$ sc -d com.queen.ctrl.WestWorldController
sc -d com.queen.ctrl.WestWorldController
 class-info        com.queen.ctrl.WestWorldController
 code-source       file:/project/live/queen-project/queen-project.jar!/BOOT
                   -INF/classes!/
 name              com.queen.ctrl.WestWorldController
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       WestWorldController
 modifier          public
 annotation        org.springframework.web.bind.annotation.RestController,org.
                   springframework.web.bind.annotation.RequestMapping
 interfaces
 super-class       +-java.lang.Object
 class-loader      +-org.springframework.boot.loader.LaunchedURLClassLoader@7d
                     af6ecc
                     +-sun.misc.Launcher$AppClassLoader@5c647e05
                       +-sun.misc.Launcher$ExtClassLoader@5c29bfd
 classLoaderHash   7daf6ecc

Affect(row-cnt:1) cost in 22 ms.
```

---

## 3. 参考文档

a. [arthas 官方文档](https://arthas.aliyun.com/doc/quick-start.html)

b. [arthas 进阶命令](https://arthas.aliyun.com/doc/advanced-use.html)

c. [arthas 使用案例](https://github.com/alibaba/arthas/issues?q=label%3Auser-case)
