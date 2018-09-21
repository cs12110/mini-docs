# Flume整合Hadoop

测试大日志文件,flume 传输到 hdfs,flume 在服务器上存放位置为:`/opt/log-flume/flume`.

---

## 1. 配置文件

需要把 hadoop 集群的配置文件拷贝到 flume 的`conf/`文件下

包括:`core-site.xml`,`hdfs-site.xml`,`hive-site.xml`文件

文件结构如下:

```powershell
hadoop231:/opt/log-flume/flume/conf # ls
core-site.xml  flume-env.sh  flume-hdfs.conf  hdfs-site.xml  hive-site.xml  log4j.properties
```

`flume-hdfs.conf`配置文件如下

请一定要写对 hdfs 的路径: **`hdfsAgent.sinks.k1.hdfs.path=hdfs://ns1/logs/`**

```properties
#agent组件名称
hdfsAgent.sources=src1
hdfsAgent.sinks=k1
hdfsAgent.channels=c1

# 指定Flume source监听文件目录路径
# 当前用户需有对spoolDir文件的读写权限
hdfsAgent.sources.src1.type=spooldir
hdfsAgent.sources.src1.spoolDir=/opt/log-flume/flume/sp-dir
hdfsAgent.sources.src1.ignorePattern = ^(.)*\\.tmp$
hdfsAgent.sources.src1.fileSuffix=.COMPLETED
#hdfsAgent.sources.src1.batchSize=128

# 指定Flume sink
# hdfsAgent.sinks.k1.hdfs.path=hdfs://pro:9000/flume/
hdfsAgent.sinks.k1.type=hdfs
hdfsAgent.sinks.k1.hdfs.path=hdfs://ns1/logs/

hdfsAgent.sinks.k1.hdfs.writeFormat = Text
hdfsAgent.sinks.k1.hdfs.fileType = DataStream
hdfsAgent.sinks.k1.hdfs.storageFolderDateFormat=yyyy-MM-dd
#hdfsAgent.sinks.k1.hdfs.fileSuffix=.txt
hdfsAgent.sinks.k1.hdfs.rollInterval = 0
hdfsAgent.sinks.k1.hdfs.rollSize = 60554432
hdfsAgent.sinks.k1.hdfs.rollCount = 0
hdfsAgent.sinks.k1.hdfs.batchSize = 100
hdfsAgent.sinks.k1.hdfs.txnEventMax = 1000
hdfsAgent.sinks.k1.hdfs.callTimeout = 60000
hdfsAgent.sinks.k1.hdfs.appendTimeout = 60000
hdfsAgent.sinks.k1.hdfs.idleTimeout=60
# hdfsAgent.sinks.k1.hdfs.minBlockReplicas = 1

hdfsAgent.channels.c1.type=org.apache.flume.channel.DualChannel
hdfsAgent.channels.c1.capacity=20000
hdfsAgent.channels.c1.transactionCapacity=10000

# 绑定source到sink
hdfsAgent.sources.src1.channels=c1
hdfsAgent.sinks.k1.channel=c1

# 生产环境中需要开启这个选项
# hdfsAgent.sources.src1.deletePolicy=immediate
```

---

## 2. 相关脚本

### 2.1 启动脚本

使用`-Dflume.monitoring.type=http` 和 `-Dflume.monitoring.port=5240`开启监测端口

页面访问地址为:`http://ip:5240/metrics`

```sh
#!/bin/bash
echo -e 'Start'
# 删除所有遗留日志
rm -rf logs/*

nohup bin/flume-ng agent --conf conf --conf-file conf/flume-hdfs.conf --name hdfsAgent -Dflume.monitoring.type=http -Dflume.monitoring.port=5240 >/dev/null 2>&1 &

echo -e 'Done'
```

### 2.2 关闭脚本

```sh
#!/bin/bash

for each in `ps -ef|grep flume |grep -v grep |awk '{print $2}'`
do
	echo 'kill -9 ' $each
	kill -9 $each
done
```

---

### 3. 优化

由于 hdfs 写入比较慢,所以使用`sinkgroups`多个`sink`往`hdfs`写入,每个 sink 都是一个线程,如果机器允许,可以通过消耗硬件资源来换取时间.

配置文件如下:

```properties
#agent组件名称
hdfsAgent.sources=srch1
hdfsAgent.sinks=hdfsSink1 hdfsSink2
hdfsAgent.sinkgroups=sinkGroup

hdfsAgent.sinkgroups.sinkGroup.sinks=hdfsSink1 hdfsSink2
#hdfsAgent.sinkgroups.sinkGroup.sinks=hdfsSink1

hdfsAgent.sinkgroups.sinkGroup.processor.type=load_balance
hdfsAgent.sinkgroups.sinkGroup.processor.selector=random
hdfsAgent.sinkgroups.sinkGroup.processor.backoff=true

hdfsAgent.channels=ch1

# 指定Flume source监听文件目录路径
# 当前用户需有对spoolDir文件的读写权限
hdfsAgent.sources.srch1.type=spooldir
hdfsAgent.sources.srch1.spoolDir=/opt/log-flume/flume/sp-dir
hdfsAgent.sources.srch1.ignorePattern = ^(.)*\\.tmp$
hdfsAgent.sources.srch1.fileSuffix=.COMPLETED

# Flume sink1
hdfsAgent.sinks.hdfsSink1.type=hdfs
hdfsAgent.sinks.hdfsSink1.hdfs.path=hdfs://ns1/logs/
hdfsAgent.sinks.hdfsSink1.hdfs.writeFormat = Text
hdfsAgent.sinks.hdfsSink1.hdfs.fileType = DataStream
hdfsAgent.sinks.hdfsSink1.hdfs.storageFolderDateFormat=yyyy-MM-dd
hdfsAgent.sinks.hdfsSink1.hdfs.rollInterval = 0
hdfsAgent.sinks.hdfsSink1.hdfs.rollSize = 60554432
hdfsAgent.sinks.hdfsSink1.hdfs.rollCount = 0
hdfsAgent.sinks.hdfsSink1.hdfs.batchSize = 100
hdfsAgent.sinks.hdfsSink1.hdfs.txnEventMax = 1000
hdfsAgent.sinks.hdfsSink1.hdfs.callTimeout = 60000
hdfsAgent.sinks.hdfsSink1.hdfs.appendTimeout = 60000
hdfsAgent.sinks.hdfsSink1.hdfs.idleTimeout=60


# Flume sink2
hdfsAgent.sinks.hdfsSink2.type=hdfs
hdfsAgent.sinks.hdfsSink2.hdfs.path=hdfs://ns1/logs/
hdfsAgent.sinks.hdfsSink2.hdfs.writeFormat = Text
hdfsAgent.sinks.hdfsSink2.hdfs.fileType = DataStream
hdfsAgent.sinks.hdfsSink2.hdfs.storageFolderDateFormat=yyyy-MM-dd
hdfsAgent.sinks.hdfsSink2.hdfs.rollInterval = 0
hdfsAgent.sinks.hdfsSink2.hdfs.rollSize = 60554432
hdfsAgent.sinks.hdfsSink2.hdfs.rollCount = 0
hdfsAgent.sinks.hdfsSink2.hdfs.batchSize = 100
hdfsAgent.sinks.hdfsSink2.hdfs.txnEventMax = 1000
hdfsAgent.sinks.hdfsSink2.hdfs.callTimeout = 60000
hdfsAgent.sinks.hdfsSink2.hdfs.appendTimeout = 60000
hdfsAgent.sinks.hdfsSink2.hdfs.idleTimeout=60



hdfsAgent.channels.ch1.type=org.apache.flume.channel.DualChannel
hdfsAgent.channels.ch1.capacity=20000
hdfsAgent.channels.ch1.transactionCapacity=10000

# 绑定source到sink
hdfsAgent.sources.srch1.channels=ch1
hdfsAgent.sinks.hdfsSink1.channel=ch1
hdfsAgent.sinks.hdfsSink2.channel=ch1
```

---

## 4.参考资料

a. [Flume-1.6.0 官网](http://flume.apache.org/releases/content/1.6.0/FlumeUserGuide.html)

b. [相关 blog](http://www.aboutyun.com/thread-9170-1-1.html)
