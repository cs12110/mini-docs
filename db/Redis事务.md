# Redis 事务

在被问到对 redis 事务了不了解的时候,你会后悔没看这篇文章的.

**FBI WARNING**: 下面仅适用于单机版本的redis事务,请知悉.

同时,**请注意不同版本Redis对事务的异常的策略!!!**

---

## 1. 基础知识


### 1.1 事务概念

事务是恢复和并发控制的基本单位,事务应该具有4个属性:`原子性`,`一致性`,`隔离性`,`持久性`.

这四个属性通常称为ACID特性.

- 原子性(atomicity): 一个事务是一个不可分割的工作单位,事务中包括的诸操作要么都做,要么都不做.

- 一致性(consistency): 事务必须是使数据库从一个一致性状态变到另一个一致性状态.一致性与原子性是密切相关的.

- 隔离性(isolation): 一个事务的执行不能被其他事务干扰.即一个事务内部的操作及使用的数据对并发的其他事务是隔离的,并发执行的各个事务之间不能互相干扰.

- 持久性(durability): 持久性也称永久性(permanence),指一个事务一旦提交,它对数据库中数据的改变就应该是永久性的.接下来的其他操作或故障不应该对其有任何影响.

### 1.2 Redis事务执行流程

Redis 事务可以一次执行多个命令, 并且带有以下两个重要的保证:

- 批量操作在发送 EXEC 命令前被放入队列缓存
- 收到 EXEC 命令后进入事务执行(版本确定是抛弃整一个事务,还是继续执行事务中其他成功的命令)
- 在事务执行过程,其他客户端提交的命令请求不会插入到事务执行命令序列中

一个事务从开始到执行会经历以下三个阶段

- 开始事务
- 命令入队
- 执行事务

Redis 事务的执行具有原子性的: 事务中的命令要么全部被执行,要么全部都不执行.

**Redis 版本以及对事务的处理**

在 Redis 2.6.5 以前,Redis 只执行事务中那些入队成功的命令,而忽略那些入队失败的命令.

从 Redis 2.6.5 开始,服务器会对命令入队失败的情况进行记录,并在客户端调用 EXEC 命令时,拒绝执行并自动放弃这个事务.

---

## 2. 命令操作

请注意测试的redis版本.

版本在`2.6.5`之后,事务里面有一个命令异常,就会丢弃整一个事务.

### 2.1 基础命令

| 命令    | 描述                                                                                     |
| :-----: | ---------------------------------------------------------------------------------------- |
| MULTI   | 标记一个事务块的开始                                                                     |
| EXEC    | 执行所有事务块内的命令                                                                   |
| DISCARD | 取消事务,放弃执行事务块内的所有命令                                                      |
| UNWATCH | 取消 WATCH 命令对所有 key 的监视                                                         |
| WATCH   | 监视一个(或多个) key,如果在事务执行之前这个(或这些)key 被其他命令所改动,那么事务将被打断 |

### 2.2 命令使用

本次测试,使用的 redis 版本如下:

```sh
[root@hadoop201 src]# ./redis-server --version
Redis server v=2.8.3 sha=00000000:0 malloc=jemalloc-3.2.0 bits=64 build=e9558e7fd4e66350
```

初始化数据

```sh
[root@hadoop201 src]# ./redis-cli  
127.0.0.1:6379> set like haiyan 
OK
127.0.0.1:6379> set hate others
OK
127.0.0.1:6379> 
```

**测试成功事务**

```sh
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set like haiyan1
QUEUED
127.0.0.1:6379> set hate myself
QUEUED
127.0.0.1:6379> EXEC
1) OK
2) OK
127.0.0.1:6379> get like 
"haiyan1"
127.0.0.1:6379> get hate
"myself"
```
事务里面没有错误命令全部执行成功.



**失败事务**

```sh
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set like haiyan2
QUEUED
127.0.0.1:6379> hset hate others
(error) ERR wrong number of arguments for 'hset' command
127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get like
"haiyan1"
```
可以看出来,`set like haiyan2`也没有成功.


**丢弃事务**

```sh
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set like haiyan2
QUEUED
127.0.0.1:6379> DISCARD
OK
127.0.0.1:6379> get like 
```


### 2.3 旧版本测试 


测试版本为

```sh
[root@dev-116 ~]# redis-server --version
Redis server v=2.6.0 sha=00000000:0 malloc=jemalloc-3.0.0 bits=64
[root@dev-116 ~]# 
```

```sh
redis 127.0.0.1:6379> set like haiyan
OK
redis 127.0.0.1:6379> set hate others
OK
redis 127.0.0.1:6379> get like 
"haiyan"
redis 127.0.0.1:6379> get hate
"others"
redis 127.0.0.1:6379> MULTI
OK
redis 127.0.0.1:6379> set like haiyan1 
QUEUED
redis 127.0.0.1:6379> hset hate myself
(error) ERR wrong number of arguments for 'hset' command
redis 127.0.0.1:6379> set like haiyan2
QUEUED
redis 127.0.0.1:6379> EXEC
1) OK
2) OK
redis 127.0.0.1:6379> hget like
(error) ERR wrong number of arguments for 'hget' command
redis 127.0.0.1:6379> get like
"haiyan2"
redis 127.0.0.1:6379> 
```

这里面可以看出来即使`hset hate myself`错误之后,还是执行了`set like haiyan2`.

在`2.6.5`版本前就会出现: 即使事务中有某个/某些命令在执行时产生了错误, 事务中的其他命令仍然会继续执行.

---

## 参考文档

a. [redis 官网](https://redis.io/topics/transactions)

b. [redis 事务和 watch](https://www.jianshu.com/p/361cb9cd13d5)
