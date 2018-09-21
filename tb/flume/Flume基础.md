# Flume 说明文档

收集系统日志,过滤分发到其他地方.

核心组件: `source` -> `channel` -> `sink`

---

## 1. 配置说明

### 1.1 描述组件

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1
```

- `sourcer`: 接收文件,还是接收`http`还是`thrift`等.

- `sink`: 指定输出是`hdfs`还是`database`.

- `channel`: 在 source 与 sink 之间进行缓冲,可以设置为内存,文件等.

### 1.2 source,sink,channel 配置

```properties
# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
```

### 1.3 channel 连接 sink 与 source

```properties
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

---

## 2. 安装教程

### 2.1 下载资源

```shell
[dev@alice apache-flume-1.8.0-bin]$  wget http://mirrors.shuosc.org/apache/flume/1.8.0/apache-flume-1.8.0-bin.tar.gz
```

### 2.2 添加 flume 配置文件

在 conf/文件夹下添加 flume.conf 文件

```properties
[dev@alice apache-flume-1.8.0-bin]$ vim conf/flume.conf
# 指定Agent组件名称
agent1.sources=r1
agent1.sinks=k1
agent1.channels=c1

# 指定Flume source监听文件目录路径
# 当前用户需有对spoolDir文件的读写权限
agent1.sources.r1.type=spooldir
agent1.sources.r1.spoolDir=/home/dev/tmp

# 指定Flume sink
agent1.sinks.k1.type=logger


# 指定Flume channel
agent1.channels.c1.type=memory
agent1.channels.c1.capacity=1000
agent1.channels.c1.transactionCapacity=100

# 绑定source到sink
agent1.sources.r1.channels =c1
agent1.sinks.k1.channel=c1
```

### 2.3 启动 flume

```shell
[dev@alice apache-flume-1.8.0-bin]$ bin/flume-ng agent --conf conf/ --conf-file conf/flume.conf  --name agent1 -Dflume.root.logger=INFO,console &
```

参数解释

> --conf: 指定配置文件夹,包含 flume-env.sh 和 log4j 的配置文件
>
> --conf-file: 配置文件位置
>
> --name: agent 名称

如果不想打印在控制台上,可以使用如下启用命令,日志默认输出在`logs/flume.log`文件里面

```shell
[dev@alice apache-flume-1.8.0-bin]$ bin/flume-ng agent --conf conf/ --conf-file conf/flume.conf  --name agent1 &
```

### 2.4 测试

```shell
vim hello

    hello world

cp hello /home/dev/tmp/


# 日志输出
2017-11-09 22:17:06,290
(SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{} body: 68 65 6C 6C 6F 20 77 6F 72 6C 64                hello world }
```

### 2.5 监测 flume

```shell
nohup bin/flume-ng agent --conf conf/ --conf-file conf/flume-kafka.conf  --name agent_kafka -Dflume.monitoring.type=http -Dflume.monitoring.port=5240 >flume-kafka.out 2>&1 &
```

> - -Dflume.monitoring.type=http: 监测 flume 使用协议
> - -Dflume.monitoring.port=5240: 监测 flume 使用端口

使用:

```html
访问格式:   http://<hostname>:<port>/metrics
浏览器上访问: http://10.10.2.70:5240/metrics
```

---

## 3. 参考资料

a. [Flume 监测资料](https://flume.apache.org/FlumeUserGuide.html#json-reporting)
