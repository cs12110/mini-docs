# 数据库

这个章节主要梳理数据库知识.

---

## 1. fun fact

在 mysql 数据库上面,可以通过如下命令显示数据库表的结构

```sql
mysql> show create table top_answer_t \G;
*************************** 1. row ***************************
       Table: top_answer_t
Create Table: CREATE TABLE `top_answer_t` (
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

Q: 在 mysql 执行过程中,可能因为某些异常导致表被锁,那么怎么查看以及解锁呢?

A: follow me.

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

Q: 在什么情况下,会锁行/锁表呀?

A: I don't know yet.

---
