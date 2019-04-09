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

## 2. Other

要留意 mysql 的锁行,锁表以及解决方法.
