# 数据库

这个章节主要梳理数据库知识.

---

## 1. fun fact

在 mysql 数据库上面,可以通过如下命令显示数据库表的结构

```sql
mysql> show create table top_answer_t \G;
*************************** 1. row ***************************
       Table:top_answer_t
Create Table:CREATE TABLE `top_answer_t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `question` varchar(256) DEFAULT NULL COMMENT '题目',
  `author` varchar(256) DEFAULT NULL COMMENT '作者',
  `link` varchar(256) DEFAULT NULL COMMENT '连接',
  `upvote_num` int(8) DEFAULT NULL COMMENT '点赞数',
  `comment_num` int(6) DEFAULT NULL COMMENT '留言数',
  `summary` longtext COMMENT '总结',
  `create_at` varchar(32) DEFAULT NULL COMMENT '创建时间',
  `update_at` varchar(32) DEFAULT NULL COMMENT '更新时间',
  `question_id` varchar(64) DEFAULT NULL,
  `answer_id` varchar(64) DEFAULT NULL,
  `steal_at` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `linkIndex` (`link`)
) ENGINE=InnoDB AUTO_INCREMENT=9842 DEFAULT CHARSET=utf8 COMMENT='精华回答表'
```

---

## 2. 锁表与解锁

Q:在 mysql 执行过程中,可能因为某些异常导致表被锁,那么怎么查看以及解锁呢?

A:follow me.

- id 为进程
- command 为 waitting 的就是锁住的表
- info 为执行某条语句的信息

```sql
mysql> show processlist;
+-------+------+----------------------+---------+---------+------+----------+------------------+
| Id    | User | Host                 | db      | Command | Time | State    | Info             |
+-------+------+----------------------+---------+---------+------+----------+------------------+
| 28069 | root | 47.98.104.252:33920  | 4fun_db | Sleep   |    3 |          | NULL             |
| 28119 | root | 47.98.104.252:42908  | 4fun_db | Sleep   |    3 |          | NULL             |
| 29927 | root | 47.98.104.252:36044  | 4fun_db | Sleep   |    3 |          | NULL             |
| 30845 | root | 47.98.104.252:47180  | 4fun_db | Sleep   |    3 |          | NULL             |
| 31246 | root | 47.98.104.252:59242  | 4fun_db | Sleep   |    3 |          | NULL             |
+-------+------+----------------------+---------+---------+------+----------+------------------+
5 rows in set (0.00 sec)

mysql> kill idOfProcess(例如28069)
```

Command 几个重要参数如下,[详细 link](https://blog.csdn.net/sinat_25873421/article/details/80335125)

| 参数值                  | 含义                     |
| ----------------------- | ------------------------ |
| Locked 　               | 被其他查询锁住了         |
| Sleeping 　 　          | 正在等待客户端发送新请求 |
| Sorting for group 　 　 | 正在为 GROUP BY 做排序   |
| Sorting for order 　 　 | 正在为 ORDER BY 做排序   |

---

## 3. 锁行/锁表

### 3.1 条件无索引

第一种情况:<u>条件无索引(`nickname` 无索引)</u>

a. 模拟执行,某些原因没提交事务

```sql
BEGIN;
UPDATE mkt_user SET nickname='heeson2' WHERE nickname = 'heeson2';
```

b. 另外线程再执行：

```sql
UPDATE mkt_user SET nickname='test' WHERE nickname ='test02';
```

结果: 锁表,b 无法正常执行,需要 a 执行 `commit`,b 才能恢复正常

结论:`update where` 条件无索引,导致锁表

### 3.2 条件有索引

第二种情况:<u>条件有索引(`boundPhone` 带索引)</u>

a. 模拟执行,某些原因没提交事务

```sql
BEGIN;
UPDATE mkt_user SET nickname='test 名字 1' WHERE boundPhone = '11552761891';
```

b. 另外线程再执行：

```sql
UPDATE mkt_user SET nickname='test 名字 2' WHERE boundPhone ='11094663082';
```

结果：b 正常执行,并不依赖 a 提交事务

结论: 条件有索引,仅仅锁行

### 3.3 条件有索引,in 语句复杂查询

第三种情况:<u>条件有索引(`boundPhone` 带索引),in 语句是复杂查询</u>

a. 模拟执行,某些原因没提交事务

```sql
BEGIN;
UPDATE mkt_user SET nickname='test 名字 1' WHERE boundPhone in (select phone where usr_user where id = 'xxxxx');
```

b. 另外线程再执行

```sql
UPDATE mkt_user SET nickname='test 名字 1' WHERE boundPhone in (select phone where usr_user where id = 'yyyyy');
```

结果：锁表,b 执行受阻,需要 a 执行 commit,b 才能恢复正常

结论:条件有索引,但 in 语句是不确定的值,导致锁表.还有条件含有`<>(非)`,`>`,`<`等不确定值的条件也会导致锁表.

### 3.4 结论

- update where 后面的字段有索引或者是主键的时候,只会锁住索引或者主键对应的行.

- update where 后面的字段为普通字段不带索引的时候,会锁住整张表.

- update where 后面的字段尽量不要用 `in`,`<>`,`>`,`<`不确定的值的条件,会锁住整张表.

---

## 4. 索引操作

### 4.1 添加索引

格式: `alter table tableName add index indexName(`columnName`);`

举个栗子:

```sql
mysql> alter table top_answer_t add index questionIndex(`question`);
```

### 4.2 删除索引

格式: `alter table tableName drop index indexName;`

举个栗子:

```sql
mysql> alter table top_answer_t drop index questionIndex;
```

---

## 5. 字段设计

在 mysql 里面如果一个字段被设置为`boolean`类型,会被替换成 `tinyint(1)`.false 为 0,true 为 1.

但是坑爹的地方是,tinyint 不止可以存 0 和 1,还可以存[0,9].

Q: 在 hibernate 里面,tinyint(1)取出来会被替换成 true 和 false,你想拿数字的话,该怎么办?

A: 把 tinyint 的长度改变,不设置为 1. 卧槽.

---

## 6. 新增字段

Q: 新增字段有什么好写的?

A: 主要是`after`之类的使用啦.

```mysql> show create table t_my\G;
*************************** 1. row ***************************
       Table: t_my
Create Table: CREATE TABLE `t_my` (
  `id` int(11) NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `gender` tinyint(2) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

一般增加字段只能增加在最后面,但是如果可以增加在某个字段后面,岂不美哉?

```sql
mysql> alter table t_my  add column age int(4) default 0  after name;
Query OK, 0 rows affected (0.12 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show create table t_my\G;
*************************** 1. row ***************************
       Table: t_my
Create Table: CREATE TABLE `t_my` (
  `id` int(11) NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(4) DEFAULT '0',
  `gender` tinyint(2) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

---

## 7. 最左原则

参考文档 [link](https://blog.csdn.net/LJFPHP/article/details/90056936)

在表只有三个字段建立索引

```sql
CREATE TABLE `t_my` (
  `id` int(11) NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(4) DEFAULT '0',
  `gender` tinyint(2) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name_age_gender` (`name`,`age`,`gender`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

那么如下 sql 也是可以使用索引的

```sql
mysql> explain select * from t_my where age =1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_my
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_name_age_gender
      key_len: 106
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```

执行计划可以看出,即使使用了`age`来查询也是能使用到索引的,卧槽.

That world is so crazy!!!

Q: 那么如果表里面有其他的字段呢?

```sql
CREATE TABLE `t_my` (
  `id` int(11) NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(4) DEFAULT '0',
  `gender` tinyint(2) DEFAULT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name_age_gender` (`name`,`age`,`gender`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

然后执行上面那条 sql 会是怎样?

```sql
mysql> explain select * from t_my where age =1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_my
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```
