# mysql 事务隔离级别

源连接: [MySQL 的四种事务隔离级别 link](https://www.cnblogs.com/huanongying/p/7021555.html)

参考资料:[美团技术博客](https://tech.meituan.com/tags/mysql.html)

---

## 1. 基础知识

### 1.1 事务的基本要素

[mysql acid 事务特性 link](https://www.cnblogs.com/kismetv/p/10331633.html)

- 原子性(Atomicity): 一个事务是一个不可分割的工作单位,其中的操作要么都做,要么都不做.

- 隔离性(Isolation): 事务内部的操作与其他事务是隔离的,并发执行的各个事务之间不能互相干扰.

- 持久性(Durability): 持久性是指事务一旦提交,它对数据库的改变就应该是永久性的,接下来的其他操作或故障不应该对其有任何影响.

- 一致性(Consistency): 一致性是指事务执行结束后,数据库的完整性约束没有被破坏,事务执行的前后都是合法的数据状态.数据库的完整性约束包括但不限于:实体完整性(如行的主键存在且唯一),列完整性(如字段的类型,大小,长度要符合要求),外键约束,用户自定义完整性(如转账前后,两个账户余额的和应该不变).

### 1.2 事务的并发问题

- 脏读:事务 A 读取了事务 B 更新的数据,然后 B 回滚操作,那么 A 读取到的数据是脏数据

- 不可重复读:事务 A 多次读取同一数据,事务 B 在事务 A 多次读取的过程中,对数据作了更新并提交,导致事务 A 多次读取同一数据时,结果 不一致.

- 幻读:系统管理员 A 将数据库中所有学生的成绩从具体分数改为 ABCDE 等级,但是系统管理员 B 就在这个时候插入了一条具体分数的记录,当系统管理员 A 改结束后发现还有一条记录没有改过来,就好像发生了幻觉一样,这就叫幻读.

小结:**不可重复读的和幻读很容易混淆,不可重复读侧重于修改,幻读侧重于新增或删除.解决不可重复读的问题只需锁住满足条件的行,解决幻读需要锁表.**

### 1.3 mysql 事务隔离级别

四种隔离个别如下表所示

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | :--: | :--------: | :--: |
| 读未提交（read-uncommitted） |  是  |     是     |  是  |
| 不可重复读（read-committed） |  否  |     是     |  是  |
| 可重复读（repeatable-read）  |  否  |     否     |  是  |
| 串行化（serializable）       |  否  |     否     |  否  |

mysql 的默认事务隔离级别为:**`repeatable-read`**

```sql
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)
```

---

## 2. 实验测试

表结构和初始化数据

```sql
mysql> create table account(
    -> id int(11) primary key auto_increment,
    -> `name` varchar(128) ,
    -> balance int
    -> ) engine=innodb charset='utf8' auto_increment=1;
Query OK, 0 rows affected (0.07 sec)

mysql> insert into account(name,balance) values('lilei',450),('hanmei',16000),('lucy',2400);
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

### 2.1 读未提交

**打开客户端 a,设置隔离级别为为提交**

```sql
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction ;
Query OK, 0 rows affected (0.01 sec)

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     450 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

```

在客户端 a 提交事务之前开启另一个客户端 b,进行如下操作

```sql
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance=balance-50 where id = 1 ;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

# 这里面可以看出b客户端的数据已经修改400了
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

b 客户端的事务尚未提交,但是 a 客户端也可以查询到 b 修改的数据

```sql
# b 客户端修改数据前,a客户端数据
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     450 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

# b 客户端修改数据后(事务尚未提交),a客户端数据
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

但是如果 b 客户端因为某种原因进行回滚,那么 a 客户端读取到数据就是**脏数据**了.

```sql
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.01 sec)

# b客户端进行回滚后的数据如下
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     450 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

现在在 a 客户端对数据进行更新

```sql
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

# 更新之后应该为:400-50,但是实际上并不是!
mysql> update account set balance = balance -50 where id =1 ;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

如果你想解决这个问题,就只有把隔离级别设置为: `读已提交`

### 2.2 读已提交

打开客户端 a,并设置事务隔离级别为:`读已提交`.

```sql
mysql>  set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

打开客户端 b,并设置事务隔离级别为:`读已提交`.

```sql
mysql>  set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction ;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance=balance-50 where id =1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     350 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

这时 b 客户端的事务还没提交,在 a 客户端查询数据

```sql
# 可以看出来并有脏读出现
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.01 sec)
```

现在 b 客户端进行事务提交,a 客户端再进行查询

```sql
mysql> commit;
Query OK, 0 rows affected (0.00 sec)
```

```sql
mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     400 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.01 sec)

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     350 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

这就出现不可重复读的事务问题了.

### 2.3 可重复读

还是熟悉的 2 个客户端,设置事务为:`可重复读`(也是 mysql 的默认事务隔离级别)

a 客户端

```sql
mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account ;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     350 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

mysql>
```

b 客户端

```sql
mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> update account set balance = balance-50 where id =1 ;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     300 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

在 b 客户端事务还没提交,a 客户端查询数据

```sql
# b修改数据前查询
mysql> select * from account ;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     350 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)

# b修改数据后查询
mysql> select * from account ;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     350 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
3 rows in set (0.00 sec)
```

这解决了不可重复读的事务问题.

但是在 mysql 里面 repeatable read+mvcc 机制解决了幻读问题,所以幻读问题在 repeatable read 事务隔离机制里面复现不了.

### 2.4 串行化

**WARNING**:shit,这个测试和博客的结果不一致.

打开客户端 a,设置事务隔离级别为:`串行化`

```sql
mysql> set session transaction isolation level serializable;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from account;
+----+--------+---------+
| id | name   | balance |
+----+--------+---------+
|  1 | lilei  |     300 |
|  2 | hanmei |   16000 |
|  3 | lucy   |    2400 |
+----+--------+---------+
```

打开客户端 b,设置事务隔离级别为:`串行化`

```sql
mysql> set session transaction isolation level serializable;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

# 博客里面这个会出现异常,但这里并不会,窝草.
mysql> insert into account(name,balance) values('haiyan',3306);
Query OK, 1 row affected (0.01 sec)
```

---

## 3. 总结

- 事务隔离级别为读提交时,写数据只会锁住相应的行

- 事务隔离级别为可重复读时,如果检索条件有索引(包括主键索引)的时候,默认加锁方式是 next-key 锁;如果检索条件没有索引,更新数据时会锁住整张表.一个间隙被事务加了锁,其他事务是不能在这个间隙插入记录的,这样可以防止幻读.

- 事务隔离级别为串行化时,读写数据都会锁住整张表

- 隔离级别越高,越能保证数据的完整性和一致性,但是对并发性能的影响也越大.

---

## 4. 参考资料

a. [mysql mvcc 机制](https://blog.csdn.net/whoamiyang/article/details/51901888)

b. [mysql 事务隔离机制](https://www.cnblogs.com/huanongying/p/7021555.html)
