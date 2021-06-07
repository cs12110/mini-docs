# Redis 数据持久化

Redis 数据库里面的数据有两种形式:

- cache-only :不持久化数据,数据在服务终止之后抹除,此模式也不存在"数据恢复",安全性低/效率高/易扩展的方式

- persistence:数据持久化存储

Redis 数据持久化两种方式:

- Redis Database: 简称 RDB, 快照
- Append-only file: (简称 Aof,操作日志)

---

## 1. RDB

RDB 在某个时间点,将数据写入一个临时文件,持久化结束后,用这个临时文件替换旧的持久化文件,做到数据恢复的功能

优点: 使用单独的子进程进行持久化,主进程不会进行 IO 操作,保证 Redis 的高效.

缺点: RDB 是间隔一段时间进行持久化,如持久化过程 redis 出现异常,会发生数据丢失.

Redis 默认开启 RDB 功能.

```shell
#dbfilename：持久化数据存储在本地的文件
dbfilename dump.rdb

#dir：持久化数据存储在本地的路径，如果是在/redis/redis-3.0.6/src下启动的redis-cli，则数据会存储在当前src目录下
dir ./

#snapshot触发的时机，save <seconds> <changes>
#对于此值的设置，需要谨慎，评估系统的变更操作密集程度
#可以通过save ""来关闭snapshot功能
#设置sedis进行数据库镜像的频率。
#  900秒（15分钟）内至少1个key值改变
#  300秒（5分钟）内至少10个key值改变
#  60秒（1分钟）内至少10000个key值改变
save 900 1
save 300 10
save 60 10000

#当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等
stop-writes-on-bgsave-error yes

#是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间
rdbcompression yes
```

---

## 2. AOF

Append-only file:将"操作+数据"以格式化指令的方式追加到操作日志文件末尾.在 append 操作返回后(已经写入或者即将写入),才进行实际的数据变更,日志文件保存了历史所有的操作过程.当 server 恢复数据时,replay 该日志文件就可以做到数据恢复.

优点: 保持数据的完整性,如果设置追加 file 的时间为 1s,如果 redis 发生故障,也只会丢失 1s 的数据.在日志没被 rewrite(日志文件过大会被重写)之前,可以删除某些误操作的的命令.

缺点: AOF 文件比 RDB 文件大,而且恢复速度慢.

AOF 默认是关闭的,开启方法,修改`redis.conf`:`appendonly yes`

```shell
#此选项为aof功能的开关,默认为"no",可以通过"yes"来开启aof功能
#只有在"yes"下，aof重写/文件同步等特性才会生效
appendonly yes

#指定aof文件名称
appendfilename appendonly.aof

#指定aof操作中文件同步策略，有三个合法值：always everysec no
#默认为everysec
appendfsync everysec

#在aof-rewrite期间,appendfsync是否暂缓文件同步
# "no"表示"不暂缓"
# "yes"表示"暂缓"
# 默认为 no
no-appendfsync-on-rewrite no

#aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite,默认"64mb"
# 建议:"512mb"
auto-aof-rewrite-min-size 64mb

#相对于"上一次"rewrite,本次rewrite触发时aof文件应该增长的百分比
#每一次rewrite之后，redis都会记录下此时"新aof"文件的大小(例如A),那么当aof文件增长到A*(1 + p)之后
#触发下一次rewrite,每一次aof记录的添加,都会检测当前aof文件的尺寸
auto-aof-rewrite-percentage 100
```

关于 appendfsync

- **always**:每一条 aof 记录都立即同步到文件，这是最安全的方式，也以为更多的磁盘操作和阻塞延迟，是 IO 开支较大

- **everysec**:每秒同步一次，性能和安全都比较中庸的方式，也是 redis 推荐的方式。如果遇到物理服务器故障，有可能导致最近一秒内 aof 记录丢失(可能为部分丢失)

- **no**:redis 并不直接调用文件同步，而是交给操作系统来处理，操作系统可以根据 buffer 填充情况/通道空闲时间等择机触发同步；这是一种普通的文件操作方式。性能较好，在物理服务器故障时，数据丢失量会因 OS 配置有关
