# Zookeeper 之配置中心

在分布式应用广泛的今天,怎么更方便的修改各自系统之间的配置成了一门坑爹的艺术.

嗯,于是,那年,有一个 zookeeper 看不下去了,大喊一声,都 tmd 朝我来吧.

下面是最简单的分布式中心,还没能整合到 spring 里面去,也没能解决 zk 的客户端监听问题. orz

---

## 1. zookeeper 集群安装

为了应对单点异常,我们这里安装 zookeeper 的集群.

提示: **服务器要记得打开相关防火墙端口**

### 1.1 安装

下载安装文件

```sh
[root@dev-115 zookeeper]# wget http://mirrors.hust.edu.cn/apache/zookeeper/stable/zookeeper-3.4.12.tar.gz
[root@dev-115 zookeeper]# tar -xvf zookeeper-3.4.12.tar.gz
[root@dev-115 zookeeper]# mv zookeeper-3.4.12 zk-1
```

新建 datadir 文件夹

```sh
[root@dev-115 zookeeper]# cd zk-1
[root@dev-115 zk-1]# mv conf/zoo_sample.cfg conf/zoo.cfg
[root@dev-115 zk-1]# cd datadir/
[root@dev-115 datadir]# pwd
/opt/soft/zookeeper/zk-1/datadir
[root@dev-115 datadir]# touch myid
[root@dev-115 datadir]# echo 1 > myid
```

修改配置文件

```properties
# The number of milliseconds of each tick
tickTime=2000

# The number of ticks that the initial
# synchronization phase can take
initLimit=10

# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5

# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/opt/soft/zookeeper/zk-1/datadir/

# the port at which the clients will connect
clientPort=2181

# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60

#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

# server.{num} 是各自myid文件里面的数字
# 后面端口号,要各不相同
server.1=10.33.1.115:2888:3888
server.2=10.33.1.115:2889:3889
server.3=10.33.1.115:2890:3890
```

复制`zk-1`为`zk-2`,`zk-3`,并修改`datadir文件夹里面myid数值`,配置文件端口号`clientPort`和`dataDir`路径.

- **zk-2**: myid 为 2,clientPort 为 2182,datadir 为/opt/soft/zookeeper/zk-2/datadir/

- **zk-3**: myid 为 3,clientPort 为 2183,datadir 为/opt/soft/zookeeper/zk-3/datadir/

### 1.2 启动集群

启动 **zk-1**

```sh
[root@dev-115 zk-1]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-1/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
[root@dev-115 zk-1]#
```

启动 **zk-2**

```sh
[root@dev-115 zk-2]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-2/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
[root@dev-115 zk-2]#
```

启动 **zk-3**

```sh
[root@dev-115 zk-3]# bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-3/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
[root@dev-115 zk-3]#
```

```sh
[root@dev-115 zk-1]# jps -lm |grep zoo
2594 org.apache.zookeeper.server.quorum.QuorumPeerMain /opt/soft/zookeeper/zk-3/bin/../conf/zoo.cfg
2552 org.apache.zookeeper.server.quorum.QuorumPeerMain /opt/soft/zookeeper/zk-2/bin/../conf/zoo.cfg
2524 org.apache.zookeeper.server.quorum.QuorumPeerMain /opt/soft/zookeeper/zk-1/bin/../conf/zoo.cfg
[root@dev-115 zk-1]#
```

查看集群状态

```sh
[root@dev-115 zk-3]# ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-3/bin/../conf/zoo.cfg
Mode: follower

[root@dev-115 zk-2]# ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-2/bin/../conf/zoo.cfg
Mode: leader

[root@dev-115 zk-1]# ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/soft/zookeeper/zk-1/bin/../conf/zoo.cfg
Mode: follower
```

可以看出 zk-2 为 leader 节点.

---

## 2. 项目配置

连接 zookeeper,并配置项目数据库相关连接配置.

### 2.1 配置项目参数

```sh
[root@dev-115 zk-1]# bin/zkCli.sh
[zk: localhost:2181(CONNECTED) 9] create /dcc ''
Created /dcc
[zk: localhost:2181(CONNECTED) 10] create /dcc/jdbc.user "root"
Created /dcc/jdbc.user
[zk: localhost:2181(CONNECTED) 11] create /dcc/jdbc.url  "jdbc@1212312"
Created /dcc/jdbc.url
[zk: localhost:2181(CONNECTED) 12] create /dcc/jdbc.password  "yourDbPassword"
Created /dcc/jdbc.password
[zk: localhost:2181(CONNECTED) 13] quit
```

设置值命令

```sh
[zk: localhost:2181(CONNECTED) 25] set /dcc/jdbc.url 'jdbc:mysql://10.10.2.233:3306/ups_web'
```

### 2.1 Zookeeper 常用命令

```shell
# 启动服务
[root@dev-1 bin]# ./zkServer.sh start

# 结束服务
[root@dev-1 bin]# ./zkServer.sh stop

# 重启服务
[root@dev-1 bin]# ./zkServer.sh restart

# 服务状态
[root@dev-1 bin]# ./zkServer.sh status
```

---

## 3. 代码实现

### 3.1 配置文件

**pom.xml**

```xml
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.11</version>
</dependency>

<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.13</version>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>1.7.25</version>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

**log4j.propeties**

```propertis
# Define some default values that can be overridden by system properties
zookeeper.root.logger=INFO, CONSOLE
zookeeper.console.threshold=INFO
zookeeper.log.dir=logs
zookeeper.log.file=zookeeper.log
zookeeper.log.threshold=INFO


#
# ZooKeeper Logging Configuration
#
log4j.rootLogger=${zookeeper.root.logger}

#
# Set package log level
#
log4j.logger.org.I0Itec.zkclient=ERROR
log4j.logger.org.apache.zookeeper=ERROR


#
# Log INFO level and above messages to the console
#
log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.Threshold=${zookeeper.console.threshold}
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} - %-5p [%c:%L] - %m%n
```

### 3.2 代码

```java
import org.I0Itec.zkclient.ZkClient;

/**
 * Zookeeper工具类
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @see
 * @since 1.0
 */
public class ZkUtil {

	/**
	 * zookeeper存放配置文件位置
	 */
	public static final String PROJECT_CONF = "/dcc/jdbc";

	/**
	 * zookeeper集群
	 */
	public static final String[] CLUSTER_HOSTS = {"10.33.1.115:2181", "10.33.1.115:2182", "10.33.1.115:2183"};

	/**
	 * 获取zookeeper连接
	 *
	 * @return {@link ZkClient}
	 */
	public static ZkClient getZkClient() {
		return getZkClient(CLUSTER_HOSTS);
	}

	/**
	 * 获取zookeeper连接
	 * <p>
	 * 格式必须为:`10.33.1.115:2181`
	 *
	 * @param cluster
	 *            集群数组
	 * @return {@link ZkClient}
	 */
	public static ZkClient getZkClient(String[] cluster) {
		StringBuilder connect = new StringBuilder();
		for (int index = 0, len = cluster.length; index < len; index++) {
			connect.append(cluster[index]);
			if (index < len - 1) {
				connect.append(",");
			}
		}
		ZkClient client = new ZkClient(connect.toString());
		return client;
	}
}
```

```java
import java.io.Serializable;

/**
 * Mysql连接配置实体类
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @see
 * @since 1.0
 */
public class MySqlConf implements Serializable {
	/**
	 *
	 */
	private static final long serialVersionUID = 1L;

	private String host;

	private String user;

	private String password;

	public String getHost() {
		return host;
	}

	public void setHost(String host) {
		this.host = host;
	}

	public String getUser() {
		return user;
	}

	public void setUser(String user) {
		this.user = user;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	@Override
	public String toString() {
		return "MySQLConf [host=" + host + ", user=" + user + ", password=" + password + "]";
	}
}
```

```java
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.zoo.pkgs.util.ZkUtil;

/**
 * zookeeper同步业务实现类
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @see
 * @since 1.0
 */
public class SyncService implements Runnable {

	private static final Logger logger = LoggerFactory.getLogger(SyncService.class);

	@Override
	public void run() {
		while (true) {
			sync();
		}
	}

	/**
	 * 同步zookeeper配置
	 */
	private void sync() {
		logger.info("Sync setting from zookeeper");

		ZkClient client = ZkUtil.getZkClient();
		client.subscribeDataChanges(ZkUtil.PROJECT_CONF, new IZkDataListener() {
			@Override
			public void handleDataDeleted(String dataPath) throws Exception {
				logger.info("zookeeper remove:{}", dataPath);
			}
			@Override
			public void handleDataChange(String dataPath, Object data) throws Exception {
				logger.info("zookeeper value update:{},{}", dataPath, data);
			}
		});

		// 这里就是tmd坑爹的地方了,黑脸
		try {
			Thread.sleep(Integer.MAX_VALUE);
		} catch (Exception e) {
		}

		client.close();
		logger.info("It is done");
	}
}
```

```java
import java.text.SimpleDateFormat;
import java.util.Date;

import org.I0Itec.zkclient.ZkClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.zoo.pkgs.conf.MySqlConf;
import com.zoo.pkgs.service.SyncService;
import com.zoo.pkgs.util.ZkUtil;

/**
 * 运行入口
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @see
 * @since 1.0
 */
public class App {

	public static void main(String[] args) {
		new Thread(new UpdateWorker()).start();
		new Thread(new SyncService()).start();
	}

	static class UpdateWorker implements Runnable {
		private static final Logger logger = LoggerFactory.getLogger(UpdateWorker.class);

		@Override
		public void run() {
			final ZkClient zookeeper = ZkUtil.getZkClient();
			SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			if (!zookeeper.exists(ZkUtil.PROJECT_CONF)) {
				zookeeper.createPersistent(ZkUtil.PROJECT_CONF, true);
			}
			long time = 0;
			while (true) {

				MySqlConf sqlConf = new MySqlConf();
				sqlConf.setHost(sdf.format(new Date()));
				sqlConf.setPassword("1212312");
				sqlConf.setUser("root");

				zookeeper.writeData(ZkUtil.PROJECT_CONF, sqlConf);

				logger.info("write: {}", (++time));
				try {
					Thread.sleep(3000);
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```

### 3.3 测试结果

```java
2018-12-06 13:27:59,386 - INFO  [com.zoo.pkgs.service.SyncService:35] - Sync setting from zookeeper
2018-12-06 13:27:59,605 - INFO  [com.zoo.pkgs.App$UpdateWorker:51] - write: 1
2018-12-06 13:27:59,621 - INFO  [com.zoo.pkgs.service.SyncService:45] - zookeeper value update:/dcc/jdbc,MySQLConf [host=2018-12-06 13:27:59, user=root, password=1212312]
2018-12-06 13:28:02,617 - INFO  [com.zoo.pkgs.App$UpdateWorker:51] - write: 2
2018-12-06 13:28:02,623 - INFO  [com.zoo.pkgs.service.SyncService:45] - zookeeper value update:/dcc/jdbc,MySQLConf [host=2018-12-06 13:28:02, user=root, password=1212312]
```

---

## 4. 参考资料

a. [小不点啊的 zookeeper 系列教材](https://www.cnblogs.com/leeSmall/category/1290349.html)
