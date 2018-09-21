# Redis集群安装文档

Redis缓存数据库安装文档

这个是基于同一台服务器,不同端口上的安装

Redis集群节点要求>=6个节点,所以我们按照6个节点安装

| ip地址      | 端口号 | 存放位置                 |
| ----------- | ------ | ------------------------ |
| 10.33.1.200 | 7000   | /opt/dev/redis/redis7000 |
| 10.33.1.200 | 7001   | /opt/dev/redis/redis7001 |
| 10.33.1.200 | 7002   | /opt/dev/redis/redis7002 |
| 10.33.1.200 | 7003   | /opt/dev/redis/redis7003 |
| 10.33.1.200 | 7004   | /opt/dev/redis/redis7004 |
| 10.33.1.200 | 7005   | /opt/dev/redis/redis7005 |


## 1. 下载Redis安装文件

下载redis软件

```sh
[root@hadoop200 redis]# wget http://download.redis.io/releases/redis-4.0.8.tar.gz
```

解压redis
```sh
[root@hadoop200 redis]# tar -xvf redis-4.0.8.tar.gz
```

---

## 2. 安装编译依赖

安装redis集群所需依赖

### 2.1 安装gcc(如果已经存在,请忽略该步骤)


```sh
[root@hadoop200 redis-4.0.8]# yum install -y  gcc
```

同时你可以先download那些rpm下来,自己手动安装,模拟现实生产环境,因为生产环境很多都不能连接外网的
```sh
[root@hadoop200 redis-4.0.8]# yum install -y --downloadonly --downloaddir=. gcc
```
然后安装,如果已经存在了,安装出现冲突,请使用更新命令,而不是强制安装

安装命令
```sh
[root@hadoop200 gcc]# rpm -ivh yourRPM.rpm
```

更新命令
```sh
[root@hadoop200 gcc]# rpm -Uvh yourRPM.rpm
```

### 2.2 安装ruby环境

以下**很重要,很重要,很重要**

集群需要依赖ruby构建,请先安装ruby依赖环境

```sh
[root@hadoop200 redis]#yum install -y  ruby ruby-devel rubygems.noarch
[root@hadoop200 redis]#gem install redis -v 3.3.3
```

---

## 3. 编译Redis

编译时,直接使用`make`命令,会出现如下异常
```sh
[root@hadoop200 redis-4.0.8]# make
cd src && make all
make[1]: Entering directory `/opt/dev/redis/redis-4.0.8/src'
    CC Makefile.dep
make[1]: Leaving directory `/opt/dev/redis/redis-4.0.8/src'
make[1]: Entering directory `/opt/dev/redis/redis-4.0.8/src'
    CC adlist.o
In file included from adlist.c:34:0:
zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory
 #include <jemalloc/jemalloc.h>
                               ^
compilation terminated.
make[1]: *** [adlist.o] Error 1
make[1]: Leaving directory `/opt/dev/redis/redis-4.0.8/src'
make: *** [all] Error 2
```

请使用如下命令编译:`make MALLOC=libc`
```sh
[root@hadoop200 redis-4.0.8]# make MALLOC=libc
[root@hadoop200 redis-4.0.8]# make install
```

安装完之后,会在src目录生成可执行文件
```sh
[root@hadoop200 redis]# ls redis-4.0.8/src/ |grep redis
redisassert.h
redis-benchmark
redis-benchmark.c
redis-benchmark.o
redis-check-aof
redis-check-aof.c
redis-check-aof.o
redis-check-rdb
redis-check-rdb.c
redis-check-rdb.o
redis-cli
redis-cli.c
redis-cli.o
redismodule.h
redis-sentinel
redis-server
redis-trib.rb
```

---

## 4. 启动单个Redis

启动刚安装好的redis,看是否安装成功

首先启动`redis-server`服务

```sh
[root@hadoop200 redis-4.0.8]# src/redis-server &
[1] 11360
[root@hadoop200 redis-4.0.8]# 11360:C 03 Feb 09:38:02.408 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
11360:C 03 Feb 09:38:02.408 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=11360, just started
11360:C 03 Feb 09:38:02.408 # Warning: no config file specified, using the default config. In order to specify a config file use src/redis-server /path/to/redis.conf
11360:M 03 Feb 09:38:02.409 * Increased maximum number of open files to 10032 (it was originally set to 1024).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 4.0.8 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 11360
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

11360:M 03 Feb 09:38:02.410 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11360:M 03 Feb 09:38:02.410 # Server initialized
11360:M 03 Feb 09:38:02.410 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
11360:M 03 Feb 09:38:02.410 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
11360:M 03 Feb 09:38:02.410 * Ready to accept connections
```

使用`redis-cli`连接
```sh
[root@hadoop200 redis-4.0.8]# src/redis-cli -h 127.0.0.1 -p 6379
127.0.0.1:6379> set haiyan 'You are my ...'
OK
127.0.0.1:6379> get haiyan
"You are my ..."
127.0.0.1:6379>
```

上面可以看出,单个节点的redis是安装成功的


现在**关闭节点**,开始构建集群
```
[root@hadoop200 redis]# netstat -lnp|grep 6379
tcp        0      0 0.0.0.0:6379            0.0.0.0:*               LISTEN      11360/src/redis-ser
tcp6       0      0 :::6379                 :::*                    LISTEN      11360/src/redis-ser
[root@hadoop200 redis]# kill -9 11360
```

---

## 5. 集群部署

构建redis集群

### 5.1 修改集群配置
将reids-4.0.8文件夹重命名为`reids7000`
```sh
[root@hadoop200 redis]# mv redis-4.0.8/ redis7000
```


**请详细检查,以下每个节点的配置**

redis7000修改`redis7000/redis.conf`配置文件

修改如下配置

```sh
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7000

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7000.pid
```

拷贝redis7000
```sh
[root@hadoop200 redis]# cp -r redis7000/ redis7001/
[root@hadoop200 redis]# cp -r redis7000/ redis7002/
[root@hadoop200 redis]# cp -r redis7000/ redis7003/
[root@hadoop200 redis]# cp -r redis7000/ redis7004/
[root@hadoop200 redis]# cp -r redis7000/ redis7005/
[root@hadoop200 redis]# ls
redis7000  redis7001  redis7002  redis7003  redis7004  redis7005
```


修改redis7001配置文件
```sh
[root@hadoop200 redis]# vim redis7001/redis.conf
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7001

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7001.pid
```


修改redis7002配置文件
```sh
[root@hadoop200 redis]# vim redis7002/redis.conf
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7002

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7002.pid
```

修改redis7003配置文件
```sh
[root@hadoop200 redis]# vim redis7003/redis.conf
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7003

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7003.pid
```


修改redis7004配置文件
```sh
[root@hadoop200 redis]# vim redis7004/redis.conf
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7004

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7004.pid
```

修改redis7005配置文件
```sh
[root@hadoop200 redis]# vim redis7005/redis.conf
# 一定要是该服务器的ip
bind 10.33.1.200

# reids端口号
port 7005

# 开启集群模式
cluster-enabled yes

# pid文件名称
pidfile /var/run/redis_7005.pid
```


### 5.2 Redis集群节点启动脚本

启动所有节点脚本`redis-startup.sh`
```sh
#!/bin/bash

redis7000='/opt/dev/redis/redis7000/'
redis7001='/opt/dev/redis/redis7001/'
redis7002='/opt/dev/redis/redis7002/'
redis7003='/opt/dev/redis/redis7003/'
redis7004='/opt/dev/redis/redis7004/'
redis7005='/opt/dev/redis/redis7005/'

cd $redis7000
redis-server $redis7000/redis.conf &


cd $redis7001
redis-server $redis7001/redis.conf &


cd $redis7002
redis-server $redis7002/redis.conf &


cd $redis7003
redis-server $redis7003/redis.conf &


cd $redis7004
redis-server $redis7004/redis.conf &

cd $redis7005
redis-server $redis7005/redis.conf &
```

赋予脚本可执行权限:`chmod +x redis-startup.sh`

执行脚本,启动后
```sh
[root@hadoop200 redis]# ps -ef|grep redis-server
root     11802     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7000 [cluster]
root     11803     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7001 [cluster]
root     11804     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7002 [cluster]
root     11805     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7003 [cluster]
root     11806     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7004 [cluster]
root     11807     1  0 10:12 pts/0    00:00:00 redis-server 10.33.1.200:7005 [cluster]
root     11829  9388  0 10:13 pts/0    00:00:00 grep --color=auto redis-server
```


### 5.3 构建集群
```sh
[root@hadoop200 src]# cd /opt/dev/redis/redis7001/src/
[root@hadoop200 src]# ./redis-trib.rb  create --replicas 1 10.33.1.200:7000 10.33.1.200:7001 10.33.1.200:7002 10.33.1.200:7003 10.33.1.200:7004 10.33.1.200:7005
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
10.33.1.200:7000
10.33.1.200:7001
10.33.1.200:7002
Adding replica 10.33.1.200:7004 to 10.33.1.200:7000
Adding replica 10.33.1.200:7005 to 10.33.1.200:7001
Adding replica 10.33.1.200:7003 to 10.33.1.200:7002
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 11ed8457d0e67a244b8cdb84bfd03ec01716eec3 10.33.1.200:7000
   slots:0-5460 (5461 slots) master
M: 5a57bdb952c5ac0cb27ce707077b5133671bbc8b 10.33.1.200:7001
   slots:5461-10922 (5462 slots) master
M: 3978aa0383752cde0d606ba8d7a08bf198efa83e 10.33.1.200:7002
   slots:10923-16383 (5461 slots) master
S: f2474de393b6e765c102b5665158ee1f7a44a194 10.33.1.200:7003
   replicates 3978aa0383752cde0d606ba8d7a08bf198efa83e
S: 2a7924c5125c05f1c4f7122fb8fa24a991fc845a 10.33.1.200:7004
   replicates 11ed8457d0e67a244b8cdb84bfd03ec01716eec3
S: 7b566d98efc44ce17b13dfe75df1d795298937b3 10.33.1.200:7005
   replicates 5a57bdb952c5ac0cb27ce707077b5133671bbc8b
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
11802:M 03 Feb 10:42:47.387 # configEpoch set to 1 via CLUSTER SET-CONFIG-EPOCH
11803:M 03 Feb 10:42:47.388 # configEpoch set to 2 via CLUSTER SET-CONFIG-EPOCH
11804:M 03 Feb 10:42:47.389 # configEpoch set to 3 via CLUSTER SET-CONFIG-EPOCH
11805:M 03 Feb 10:42:47.389 # configEpoch set to 4 via CLUSTER SET-CONFIG-EPOCH
11806:M 03 Feb 10:42:47.391 # configEpoch set to 5 via CLUSTER SET-CONFIG-EPOCH
11807:M 03 Feb 10:42:47.392 # configEpoch set to 6 via CLUSTER SET-CONFIG-EPOCH
>>> Sending CLUSTER MEET messages to join the cluster
11802:M 03 Feb 10:42:47.433 # IP address for this node updated to 10.33.1.200
11807:M 03 Feb 10:42:47.477 # IP address for this node updated to 10.33.1.200
11806:M 03 Feb 10:42:47.577 # IP address for this node updated to 10.33.1.200
11805:M 03 Feb 10:42:47.579 # IP address for this node updated to 10.33.1.200
11803:M 03 Feb 10:42:47.579 # IP address for this node updated to 10.33.1.200
11804:M 03 Feb 10:42:47.580 # IP address for this node updated to 10.33.1.200
Waiting for the cluster to join....11802:M 03 Feb 10:42:52.318 # Cluster state changed: ok

11805:S 03 Feb 10:42:52.408 * Before turning into a slave, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
11805:S 03 Feb 10:42:52.408 # Cluster state changed: ok
11806:S 03 Feb 10:42:52.409 * Before turning into a slave, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
11806:S 03 Feb 10:42:52.409 # Cluster state changed: ok
11807:S 03 Feb 10:42:52.409 * Before turning into a slave, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
11807:S 03 Feb 10:42:52.409 # Cluster state changed: ok
11803:M 03 Feb 10:42:52.420 # Cluster state changed: ok
>>> Performing Cluster Check (using node 10.33.1.200:7000)
M: 11ed8457d0e67a244b8cdb84bfd03ec01716eec3 10.33.1.200:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: f2474de393b6e765c102b5665158ee1f7a44a194 10.33.1.200:7003
   slots: (0 slots) slave
   replicates 3978aa0383752cde0d606ba8d7a08bf198efa83e
S: 7b566d98efc44ce17b13dfe75df1d795298937b3 10.33.1.200:7005
   slots: (0 slots) slave
   replicates 5a57bdb952c5ac0cb27ce707077b5133671bbc8b
S: 2a7924c5125c05f1c4f7122fb8fa24a991fc845a 10.33.1.200:7004
   slots: (0 slots) slave
   replicates 11ed8457d0e67a244b8cdb84bfd03ec01716eec3
11804:M 03 Feb 10:42:52.433 # Cluster state changed: ok
M: 3978aa0383752cde0d606ba8d7a08bf198efa83e 10.33.1.200:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 5a57bdb952c5ac0cb27ce707077b5133671bbc8b 10.33.1.200:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
[root@hadoop200 src]# 11807:S 03 Feb 10:42:52.693 * Connecting to MASTER 10.33.1.200:7001
11807:S 03 Feb 10:42:52.693 * MASTER <-> SLAVE sync started
11807:S 03 Feb 10:42:52.693 * Non blocking connect for SYNC fired the event.
11807:S 03 Feb 10:42:52.693 * Master replied to PING, replication can continue...
11807:S 03 Feb 10:42:52.693 * Trying a partial resynchronization (request 2c5c7adaf87fd10e31526813b630b2bda79c207b:1).
11803:M 03 Feb 10:42:52.693 * Slave 10.33.1.200:7005 asks for synchronization
11803:M 03 Feb 10:42:52.693 * Partial resynchronization not accepted: Replication ID mismatch (Slave asked for '2c5c7adaf87fd10e31526813b630b2bda79c207b', my replication IDs are '48c48731f91a465ee9b6c3e4d77dc92ff25c5ec8' and '0000000000000000000000000000000000000000')
11803:M 03 Feb 10:42:52.693 * Starting BGSAVE for SYNC with target: disk
11803:M 03 Feb 10:42:52.694 * Background saving started by pid 12130
11807:S 03 Feb 10:42:52.694 * Full resync from master: 5df6ce3a2c9fe5e66f6c894f34ad54f253bc49f7:0
11807:S 03 Feb 10:42:52.694 * Discarding previously cached master state.
12130:C 03 Feb 10:42:52.697 * DB saved on disk
12130:C 03 Feb 10:42:52.697 * RDB: 0 MB of memory used by copy-on-write
11803:M 03 Feb 10:42:52.723 * Background saving terminated with success
11803:M 03 Feb 10:42:52.723 * Synchronization with slave 10.33.1.200:7005 succeeded
11807:S 03 Feb 10:42:52.723 * MASTER <-> SLAVE sync: receiving 175 bytes from master
11807:S 03 Feb 10:42:52.723 * MASTER <-> SLAVE sync: Flushing old data
11805:S 03 Feb 10:42:52.724 * Connecting to MASTER 10.33.1.200:7002
11805:S 03 Feb 10:42:52.724 * MASTER <-> SLAVE sync started
11805:S 03 Feb 10:42:52.724 * Non blocking connect for SYNC fired the event.
11805:S 03 Feb 10:42:52.724 * Master replied to PING, replication can continue...
11805:S 03 Feb 10:42:52.724 * Trying a partial resynchronization (request 43ffad19206069f59a7a7470a4786e2bd4d429a9:1).
11804:M 03 Feb 10:42:52.725 * Slave 10.33.1.200:7003 asks for synchronization
11804:M 03 Feb 10:42:52.725 * Partial resynchronization not accepted: Replication ID mismatch (Slave asked for '43ffad19206069f59a7a7470a4786e2bd4d429a9', my replication IDs are 'd5ed5542ecd6344f3e2c831bbf41170b8692192d' and '0000000000000000000000000000000000000000')
11804:M 03 Feb 10:42:52.725 * Starting BGSAVE for SYNC with target: disk
11804:M 03 Feb 10:42:52.725 * Background saving started by pid 12131
11805:S 03 Feb 10:42:52.725 * Full resync from master: dc8a653b48634b04955e649135ae9e7da0b04531:0
11805:S 03 Feb 10:42:52.725 * Discarding previously cached master state.
11807:S 03 Feb 10:42:52.725 * MASTER <-> SLAVE sync: Loading DB in memory
11807:S 03 Feb 10:42:52.725 * MASTER <-> SLAVE sync: Finished with success
12131:C 03 Feb 10:42:52.727 * DB saved on disk
12131:C 03 Feb 10:42:52.727 * RDB: 0 MB of memory used by copy-on-write
11804:M 03 Feb 10:42:52.735 * Background saving terminated with success
11804:M 03 Feb 10:42:52.735 * Synchronization with slave 10.33.1.200:7003 succeeded
11805:S 03 Feb 10:42:52.736 * MASTER <-> SLAVE sync: receiving 175 bytes from master
11805:S 03 Feb 10:42:52.736 * MASTER <-> SLAVE sync: Flushing old data
11805:S 03 Feb 10:42:52.738 * MASTER <-> SLAVE sync: Loading DB in memory
11805:S 03 Feb 10:42:52.738 * MASTER <-> SLAVE sync: Finished with success
11806:S 03 Feb 10:42:52.837 * Connecting to MASTER 10.33.1.200:7000
11806:S 03 Feb 10:42:52.837 * MASTER <-> SLAVE sync started
11806:S 03 Feb 10:42:52.837 * Non blocking connect for SYNC fired the event.
11806:S 03 Feb 10:42:52.837 * Master replied to PING, replication can continue...
11806:S 03 Feb 10:42:52.837 * Trying a partial resynchronization (request 0b5542f50edd6d57f8e56842a27257d5a968b231:1).
11802:M 03 Feb 10:42:52.837 * Slave 10.33.1.200:7004 asks for synchronization
11802:M 03 Feb 10:42:52.837 * Partial resynchronization not accepted: Replication ID mismatch (Slave asked for '0b5542f50edd6d57f8e56842a27257d5a968b231', my replication IDs are '08b4bf28fee89b4cf3c1e69f0c332268457d6932' and '0000000000000000000000000000000000000000')
11802:M 03 Feb 10:42:52.837 * Starting BGSAVE for SYNC with target: disk
11802:M 03 Feb 10:42:52.837 * Background saving started by pid 12132
11806:S 03 Feb 10:42:52.837 * Full resync from master: 2f7ef2fe5001734d3aaf4409ee24bc08683394ac:0
11806:S 03 Feb 10:42:52.837 * Discarding previously cached master state.
12132:C 03 Feb 10:42:52.839 * DB saved on disk
12132:C 03 Feb 10:42:52.839 * RDB: 0 MB of memory used by copy-on-write
11802:M 03 Feb 10:42:52.925 * Background saving terminated with success
11802:M 03 Feb 10:42:52.926 * Synchronization with slave 10.33.1.200:7004 succeeded
11806:S 03 Feb 10:42:52.926 * MASTER <-> SLAVE sync: receiving 175 bytes from master
11806:S 03 Feb 10:42:52.926 * MASTER <-> SLAVE sync: Flushing old data
11806:S 03 Feb 10:42:52.928 * MASTER <-> SLAVE sync: Loading DB in memory
11806:S 03 Feb 10:42:52.928 * MASTER <-> SLAVE sync: Finished with success
```


查看集群master-slave状态
```sh
[root@hadoop200 src]# ./redis-trib.rb check 10.33.1.200:7000  
>>> Performing Cluster Check (using node 10.33.1.200:7000)
M: 11ed8457d0e67a244b8cdb84bfd03ec01716eec3 10.33.1.200:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: f2474de393b6e765c102b5665158ee1f7a44a194 10.33.1.200:7003
   slots: (0 slots) slave
   replicates 3978aa0383752cde0d606ba8d7a08bf198efa83e
S: 7b566d98efc44ce17b13dfe75df1d795298937b3 10.33.1.200:7005
   slots: (0 slots) slave
   replicates 5a57bdb952c5ac0cb27ce707077b5133671bbc8b
S: 2a7924c5125c05f1c4f7122fb8fa24a991fc845a 10.33.1.200:7004
   slots: (0 slots) slave
   replicates 11ed8457d0e67a244b8cdb84bfd03ec01716eec3
M: 3978aa0383752cde0d606ba8d7a08bf198efa83e 10.33.1.200:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 5a57bdb952c5ac0cb27ce707077b5133671bbc8b 10.33.1.200:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```


构建成功之后,使用集群命令登录
```sh
[root@hadoop200 src]# redis-cli -c -h 10.33.1.200 -p 7000
10.33.1.200:7000> set haiyan 'You are my ...'
-> Redirected to slot [14488] located at 10.33.1.200:7002
OK
10.33.1.200:7002> get haiyan
"You are my ..."
```

---

## 6. 集群开启与关闭
**如果6个节点全部down之后,不用重新构建集群,只要重新开启所有节点就可以了**


### 6.1 关闭节点

关闭redis节点,可以用`kill -9 pid`来杀掉进程

`kill-them-all.sh`脚本如下,杀掉全部redis-server服务,在生成环境不建议使用该脚本

```sh
#!/bin/bash

for pid in `ps -ef |grep redis-server |awk '{print $2}'`
do
	kill -9 $pid
	echo 'kill -9 ' $pid
done
```

### 6.2 开启节点
使用`redis-startup.sh`脚本启动

注意: 这个脚本在关闭终端之后,redis-server也会被杀掉,如果要后台运行,请使用`nohup yourCmd &`来运行

```
#!/bin/bash

redis7000='/opt/dev/redis/redis7000/'
redis7001='/opt/dev/redis/redis7001/'
redis7002='/opt/dev/redis/redis7002/'
redis7003='/opt/dev/redis/redis7003/'
redis7004='/opt/dev/redis/redis7004/'
redis7005='/opt/dev/redis/redis7005/'

cd $redis7000
redis-server $redis7000/redis.conf &


cd $redis7001
redis-server $redis7001/redis.conf &


cd $redis7002
redis-server $redis7002/redis.conf &


cd $redis7003
redis-server $redis7003/redis.conf &


cd $redis7004
redis-server $redis7004/redis.conf &

cd $redis7005
redis-server $redis7005/redis.conf &
```

---


## 7. 查看集群信息

使用cluster nodes查看集群的主从节点

```powershell
hadoop233:~ # redis-cli -h 10.10.2.233 -p 7000
10.10.2.233:7000> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:42
cluster_my_epoch:42
cluster_stats_messages_sent:1480174
cluster_stats_messages_received:1471928
10.10.2.233:7000> cluster nodes
b30ec23c4185ff978e702699b8e33dc313015e46 10.10.2.234:7001 slave 108fd33b3a0f5020547496e5f3c1db4ef83acf4e 0 1529372146939 14 connected
e6582b8a34f6b510e8c417202e9fec3e71abcfbc 10.10.2.233:7000 myself,master - 0 0 42 connected 10923-16383
0113e0a31e05956893729a14614b419c69c9c1e9 10.10.2.235:7001 slave 2de1ed4241b45d5a4c221411f718ec87b096cd0d 0 1529372149943 13 connected
5215446a61efbb524ad4429444541ce9aee66afb 10.10.2.233:7001 slave e6582b8a34f6b510e8c417202e9fec3e71abcfbc 0 1529372147939 42 connected
108fd33b3a0f5020547496e5f3c1db4ef83acf4e 10.10.2.235:7000 master - 0 1529372144937 14 connected 0-5460
2de1ed4241b45d5a4c221411f718ec87b096cd0d 10.10.2.234:7000 master - 0 1529372148942 13 connected 5461-10922
10.10.2.233:7000>

```

---

## 8. 参考资料

a. [Reids官网](https://redis.io/topics/cluster-tutorial)

b. [Gem redis](https://rubygems.org/gems/redis/versions/3.3.3)

c. [CSDN博客](http://blog.csdn.net/yulei_qq/article/details/51957463)
