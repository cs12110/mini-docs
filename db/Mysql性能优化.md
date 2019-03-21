# MySql 优化指南

优化第一定律: 交给更厉害的人.

---

## 1. 基础概念

合理安排资源.调整系统参数使 MySQL 运行更快.更节省资源.

优化是多方面的,包括查询优化.更新优化.服务器优化等很多方面.没有特定方式特定的方法,总是要具体场景,具体分析,但是我们要掌握基本的优化手段.

原则:**减少系统瓶颈,减少资源占用,增加系统的反应速度.**

---

## 2. 性能参数

### 2.1 显示状态

可以通过 `SHOW STATUS` 语句查看 MySQL 数据库的**所有性能参数**.

tips: 同时你可以使用 `SHOW STATUE LIKE 'yourInterestName'`来显示其中的一个参数值

```sql
mysql> show status like 'Com_select';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_select    | 2     |
+---------------+-------+
1 row in set (0.00 sec)
```

### 2.2 慢查询

Q: 什么是慢查询?

A: mysql 日志里面记录了执行某条 sql 语句超过某个时间后的记录,方便我们做一个后期的优化,可以通过 Slow_queries 显示慢查询.

```sql
mysql> SHOW STATUS LIKE 'Slow_queries';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 0     |
+---------------+-------+
1 row in set (0.00 sec)
```

### 2.3 获取操作次数

获取: Com\_(select/insert/udpate/delete) 操作的次数

```sql
mysql> show status like 'Com_select';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_select    | 2     |
+---------------+-------+
1 row in set (0.00 sec)
```

### 2.4 Uptime

获取 mysql 运算时间

```sql
mysql> show status like 'Uptime';
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| Uptime        | 6133889 |
+---------------+---------+
1 row in set (0.00 sec)
```

### 2.5 显示表索引

Q: 但是我想知道我的表有哪些索引,我该怎么办呀?

A: 可以使用`show index from tableName`.

```sql
mysql> show index from top_answer_t \G;
*************************** 1. row ***************************
        Table: top_answer_t
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 8951
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE
      Comment: 
Index_comment: 
1 row in set (0.00 sec)
```

---

## 3. 查询优化

### 3.1 查看执行计划

`explain`对我说: 你听我解释一下呀,你倒是听我解释一下呀.

在 mysql 里面可以通过`explain`来查看 sql 的执行计划

```sql
mysql> explain select * from top_answer_t\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 9379
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
### 3.2 结果说明

| 参数名称      | 参数含义                                                                                                       | 备注                                                                 |
| ------------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| id            | SELECT 识别符.这是 SELECT 查询序列号.                                                                          | -                                                                    |
| select_type   | 表示查询中每个 select 子句的类型(简单 OR 复杂).                                                                | 参考select_type说明                                                  |
| table         | 表示查询的表.                                                                                                  | -                                                                    |
| type          | 表示表的连接类型.                                                                                              | 参考type说明                                                         |
| possible_keys | 指出 MySQL 能使用哪个索引在该表中找到行.<br>如果该列为 NULL,说明没有使用索引,可以对该列创建索引来提高性能.     | -                                                                    |
| key           | 显示 MySQL 实际决定使用的键(索引).如果没有选择索引,键是 NULL.                                                  | 强制使用索引:`USE INDEX(列名)`<br> 忽略使用索引:`IGNORE INDEX(列名)` |
| key_len       | 显示 MySQL 决定使用的键长度.如果键是 NULL,则长度为 NULL.<br>注意：key_len 是确定了 MySQL 将实际使用的索引长度. | -                                                                    |
| ref           | 显示使用哪个列或常数与 key 一起从表中选择行.                                                                   | -                                                                    |
| rows          | 显示 MySQL 认为它执行查询时必须检查的行数.                                                                     | -                                                                    |
| Extra         | 该列包含 MySQL 解决查询的详细信息                                                                              | 参考Extra说明                                                        |


#### 3.2.1 select_type说明

| select_type值   | 说明                                                       |
| --------------- | ---------------------------------------------------------- |
| SIMPLE          | 查询中不包含子查询或者 UNION                               |
| PRIMARY         | 查询中若包含任何复杂的子部分,最外层查询则被标记为:PRIMARY. |
| UNION           | 表示连接查询(UNION)的第 2 个或后面的查询语句.              |
| DEPENDENT UNION | UNION 中的第二个或后面的 SELECT 语句,取决于外面的查询.     |
| UNION RESULT    | 连接查询的结果.                                            |
| SUBQUERY        | 子查询中的第 1 个 SELECT 语句.                             |
| DERIVED         | SELECT(FROM 子句的子查询).                                 |

#### 3.2.2  type

##### 3.2.2.1 理想类型

**最佳类型[下面五种情况都是很理想的索引使用情况]:**

| 参数名称      | 说明                                                                                                                                                                                                                 |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `system`      | 表仅有一行,这是 const 类型的特列,平时不会出现,这个也可以忽略不计.                                                                                                                                                    |
| `const`       | 数据表最多只有一个匹配行,常用于 PRIMARY KEY 或者 UNIQUE 索引的查询,可理解为 const 是最优化的.                                                                                                                        |
| `eq_ref`      | mysql 手册是这样说的:"对于每个来自于前面的表的行组合,从该表中读取一行.这可能是最好的联接类型,除了 const 类型.它用在一个索引的所有部分被联接使用并且索引是 UNIQUE 或 PRIMARY KEY".eq_ref 可以用于使用=比较带索引的列. |
| `ref`         | 查询条件索引既不是 UNIQUE 也不是 PRIMARY KEY 的情况.ref 可用于`=`或`<`或`>`操作符的带索引的列                                                                                                                        |
| `ref_or_null` | 该联接类型如同 ref,但是添加了 MySQL 可以专门搜索包含 NULL 值的行.在解决子查询中经常使用该联接类型的优化.                                                                                                             |

###### const
```sql
mysql> explain select * from top_answer_t where id =1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

###### eq_ref

```sql
mysql> EXPLAIN SELECT * FROM top_answer_t LEFT JOIN map_topic_answer_t ON map_topic_answer_t.answer_id = top_answer_t.id where map_topic_answer_t.topic_id=35\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: map_topic_answer_t
   partitions: NULL
         type: ref
possible_keys: topicIdIndex,answerIdIndex
          key: topicIdIndex
      key_len: 5
          ref: const
         rows: 183
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: 4fun_db.map_topic_answer_t.answer_id
         rows: 1
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)
```

###### ref

```sql
mysql> EXPLAIN SELECT * FROM top_answer_t LEFT JOIN map_topic_answer_t ON map_topic_answer_t.answer_id = top_answer_t.id\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 9379
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: map_topic_answer_t
   partitions: NULL
         type: ref
possible_keys: answerIdIndex
          key: answerIdIndex
      key_len: 5
          ref: 4fun_db.top_answer_t.id
         rows: 4
     filtered: 100.00
        Extra: NULL
```


##### 3.2.2.2 泪崩查询

| 参数名称          | 说明                                                                                                                                                                      |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `index_merge`     | 该联接类型表示使用了索引合并优化方法.在这种情况下,key 列包含了使用的索引的清单,key_len 包含了使用的索引的最长的关键元素.                                                  |
| `unique_subquery` | 该类型替换了下面形式的 IN 子查询的 ref: `value IN (SELECT primary_key FROM single_table WHERE some_expr)`,unique_subquery 是一个索引查找函数,可以完全替换子查询,效率更高. |
| `index_subquery`  | 该联接类型类似于 unique_subquery.可以替换 IN 子查询,但只适合下列形式的子查询中的非唯一索引: `value IN (SELECT key_column FROM single_table WHERE some_expr)`              |
| `range`           | 只检索给定范围的行,使用一个索引来选择行.                                                                                                                                  |
| `index`           | 该联接类型与 ALL 相同,除了只有索引树被扫描.这通常比 ALL 快,因为索引文件通常比数据文件小.                                                                                  |
| `all`             | 对于每个来自于先前的表的行组合,进行完整的表扫描.(性能最差) B tree b+tree                                                                                                  |

###### range

```sql
mysql> explain select * from top_answer_t where id <5\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 4
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.02 sec)
```


#### 3.2.3 Extra

| 名称                                                              | 说明                                                                                                                                                               |
| ----------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Distinct                                                          | MySQL 发现第 1 个匹配行后,停止为当前的行组合搜索更多的行.                                                                                                          |
| Not exists                                                        | MySQL 能够对查询进行 LEFT JOIN 优化,发现 1 个匹配 LEFT JOIN 标准的行后,不再为前面的的行组合在该表内检查更多的行.                                                   |
| range checked for each record (index map: #)                      | MySQL 没有发现好的可以使用的索引,但发现如果来自前面的表的列值已知,可能部分索引可以使用.                                                                            |
| Using filesort                                                    | MySQL 需要额外的一次传递,以找出如何按排序顺序检索行.                                                                                                               |
| Using index                                                       | 从只使用索引树中的信息而不需要进一步搜索读取实际的行来检索表中的列信息.                                                                                            |
| Using temporary                                                   | 为了解决查询,MySQL 需要创建一个临时表来容纳结果.                                                                                                                   |
| Using where                                                       | WHERE 子句用于限制哪一个行匹配下一个表或发送到客户.                                                                                                                |
| Using sort_union(...)<br>Using union(...)<br>Using intersect(...) | 这些函数说明如何为 index_merge 联接类型合并索引扫描.                                                                                                               |
| Using index for group-by                                          | 类似于访问表的 Using index 方式,Using index for group-by 表示 MySQL 发现了一个索引,可以用来查 询 GROUP BY 或 DISTINCT 查询的所有列,而不要额外搜索硬盘访问实际的表. |

### 3.3 索引查询注意事项

索引可以提供查询的速度,但并不是使用了带有索引的字段查询都会生效,有些情况下是不生效的,需要注意！

数据库索引,是数据库管理系统中一个排序的数据结构,以协助快速查询.更新数据库表中数据.索引的实现通常使用 B 树(B-tree(多路搜索树,并不是二叉的)是一种常见的数据结构)及其变种 B+树.

在数据之外,数据库系统还维护着满足特定查找算法的数据结构,这些数据结构以某种方式引用(指向)数据,这样就可以在这些数据结构上实现高级查找算法.这种数据结构,就是索引.

为表设置索引要付出代价的:<span style="color:red">一是增加了数据库的存储空间,二是在插入和修改数据时要花费较多的时间(因为索引也要随之变动)</span>.

Q: 为什么使用索引就能加快查询速度呢?

A: 索引就是通过事先排好序,从而在查找时可以应用二分查找等高效率的算法.

一般的顺序查找,复杂度为`O(n)`,而二分查找复杂度为`O(log2n)`.当 n 很大时,二者的效率相差及其悬殊.

举个例子:表中有一百万条数据,需要在其中寻找一条特定 id 的数据.如果顺序查找,平均需要查找 50 万条数据.而用二分法,至多不超过 20 次就能找到.二者的效率差了 2.5 万倍！

#### 3.3.1 使用 LIKE 关键字的查询

在使用 LIKE 关键字进行查询的查询语句中,如果匹配字符串的第一个字符为“%”,索引不起作用.只有“%”不在第一个位置,索引才会生效.

```sql
mysql> explain select * from top_answer_t where link like '%123%' \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 9379
     filtered: 11.11
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```


```sql
mysql> explain select * from top_answer_t where link like '123%' \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_answer_t
   partitions: NULL
         type: range
possible_keys: linkIndex
          key: linkIndex
      key_len: 771
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
```

#### 3.3.2 使用联合索引的查询


最左原则: MySQL 可以为多个字段创建索引,一个索引可以包括 16 个字段.对于联合索引,只有查询条件中使用了这些字段中第一个字段时,索引才会生效.


#### 3.3.3 使用 OR 关键字的查询

查询语句的查询条件中只有 OR 关键字,且 OR 前后的两个条件中的列都是索引时,索引才会生效,否则,索引不生效.


#### 3.3.4 子查询优化

MySQL 从 4.1 版本开始支持子查询,使用子查询进行 SELECT 语句嵌套查询,可以一次完成很多逻辑上需要多个步骤才能完成的 SQL 操作.子查询虽然很灵活,但是执行效率并不高.

那么问题又来了啊? 什么叫子查询?为什么它效率不高?

(把内层查询结果当作外层查询的比较条件的)

例子:

```sql
select goods_id,goods_name from goods where goods_id = (select max(goods_id) from goods);
```

执行子查询时,MYSQL 需要创建临时表,查询完毕后再删除这些临时表,所以,子查询的速度会受到一定的影响.这多了一个创建临时表和销毁表的过程啊.

优化方式:**可以使用连接查询(JOIN)代替子查询,连接查询时不需要建立临时表,其速度比子查询快.**

---

## 4. 数据库结构优化

一个好的数据库设计方案对于数据库的性能往往会起到事半功倍的效果.这句话是什么意思呢,就是说我们的数据库优化不仅仅要局限于查询优化,要从这块跳出来做好最开始的设计优化,如果你这个主要设计是不合理的这些个查询优化效果也只是杯水车薪.

需要考虑数据冗余.查询和更新的速度.字段的数据类型是否合理等多方面的内容.

### 4.1 分表

对于字段较多的表,如果有些字段的使用频率很低,可以将这些字段分离出来形成新表.

因为当一个表的数据量很大时,会由于使用频率低的字段的存在而变慢.

项目实战的时候会将一个完全信息的表里面的数据拆分出来 形成多个新表 每个新表负责那一块的数据查询 然后这个拆分是定时的

### 4.2 增加中间表

对于需要经常联合查询的表,可以建立中间表以提高查询效率.这个大家能不能理解?

通过建立中间表,将需要通过联合查询的数据插入到中间表中,然后将原来的联合查询改为对中间表的查询.

举个例子啊:比如我们需要在五张表里面做查询数据,left join 每次查询要连接五张表是吧 我这里做一个中间表 把查询结果都放在这个中间表里面,只要在这个表里面查询就行了啊

通常都是在统计当中有使用啊,每次统计报表的时候都是离线统计啊,后台有有一个线程对你这统计结果查询号放入一个中间表,然后你对这个中间表查询就行了.

### 4.3 增加冗余字段

设计数据表时应尽量遵循范式理论的规约,尽可能的减少冗余字段,让数据库设计看起来精致.优雅.但是,合理的加入冗余字段可以提高查询速度.

表的规范化程度越高,表和表之间的关系越多,需要连接查询的情况也就越多,性能也就越差.

```sql
CREATE TABLE `tb_cart` (
	id INT (11) PRIMARY KEY auto_increment COMMENT 'id',
	`user_id` INT (11) COMMENT '用户id',
	`item_id` INT (11) COMMENT '商品id',
	# 冗余字段
	`item_name` VARCHAR (128) COMMENT '商品名称',
	`num` INT COMMENT '购买数量'
) ENGINE = INNODB auto_increment = 1 COMMENT '购物车';
```

**注意**:冗余字段的值在一个表中修改了,就要想办法在其他表中更新,否则就会导致数据不一致的问题.

---

## 5. 插入数据的优化

Q: 为什么索引会影响插入速度呢?

A: ​索引越多,当你写入数据的时候就会越慢,因为我们在插入数据的时候不只是把数据写入文件,而且还要把这个数据写到索引中,索引索引越多插入越慢

数据库引擎如下:

- MyISAM适合:做很多count 的计算;插入不频繁,查询非常频繁;没有事务.

- InnoDB适合:可靠性要求比较高,或者要求事务;表更新和查询都相当的频繁,并且表锁定的机会比较大的情况.

| 属性              | MyISAM   | Heap   | BDB             | InnoDB |
| ----------------- | -------- | ------ | --------------- | ------ |
| 事务              | 不支持   | 不支持 | 支持            | 支持   |
| 锁粒度            | 表锁     | 表锁   | 页锁(page, 8KB) | 行锁   |
| 存储              | 拆分文件 | 内存中 | 每个表一个文件  | 表空间 |
| 隔离等级          | 无       | 无     | 读已提交        | 所有   |
| 可移植格式        | 是       | N/A    | 否              | 是     |
| 引用完整性        | 否       | 否     | 否              | 是     |
| 数据主键          | 否       | 否     | 是              | 是     |
| MySQL缓存数据记录 | 无       | 有     | 有              | 有     |
| 可用性            | 全版本   | 全版本 | MySQL－Max      | 全版本 |

### 5.1 MyISAM

#### 5.1.1 禁用索引

对于非空表,插入记录时,MySQL 会根据表的索引对插入的记录建立索引.如果插入大量数据,建立索引会降低插入数据速度.

为了解决这个问题,可以在批量插入数据之前禁用索引,数据插入完成后再开启索引.

禁用索引的语句:

```sql
ALTER TABLE table_name DISABLE KEYS
```

开启索引语句:

```sql
ALTER TABLE table_name ENABLE KEYS
```

对于空表批量插入数据,则不需要进行操作,因为 MyISAM 引擎的表是在导入数据后才建立索引.

#### 5.1.2 禁用唯一性检查

唯一性校验会降低插入记录的速度,可以在插入记录之前禁用唯一性检查,插入数据完成后再开启.

禁用唯一性检查的语句:`SET UNIQUE_CHECKS = 0;`

开启唯一性检查的语句:`SET UNIQUE_CHECKS = 1;`

#### 5.1.3 批量插入数据

插入数据时,可以使用一条 INSERT 语句插入一条数据,也可以插入多条数据.

```sql
INSERT INTO sentence_t (author, sentence)
VALUES
	('haiyan', '只有意志...');

INSERT INTO sentence_t (author, sentence)
VALUES
	('3306', '窝草');
```

```sql
INSERT INTO sentence_t (author, sentence)
VALUES
	('3306', '窝草'),
	("haiyan", "只有意志...");
```


Q: 为什么第二种方式的插入速度比第一种方式快?

A: 我们第一个语句发送到 sql 后要解析两次, 第二个SQL只需要解析一次.

#### 5.1.4 使用 LOAD DATA INFILE

当需要批量导入数据时,使用`LOAD DATA INFILE`语句比`INSERT`语句插入速度快很多.

### 5.2. InnoDB

#### 5.2.1 禁用唯一性检查

禁用唯一性检查的语句:`SET UNIQUE_CHECKS = 0;`

开启唯一性检查的语句:`SET UNIQUE_CHECKS = 1;`

#### 5.2.2 禁用外键检查

插入数据之前执行禁止对外键的检查,数据插入完成后再恢复,可以提供插入速度.

禁用:`SET foreign_key_checks = 0;`

开启:`SET foreign_key_checks = 1;`

#### 5.2.3 禁止自动提交

插入数据之前执行禁止事务的自动提交,数据插入完成后再恢复,可以提高插入速度.

禁用:`SET autocommit = 0;`

开启:`SET autocommit = 1;`

---

## 6. 服务器优化

### 6.1 优化服务器硬件

服务器的硬件性能直接决定着 MySQL 数据库的性能,硬件的性能瓶颈,直接决定 MySQL 数据库的运行速度和效率.

需要从以下几个方面考虑:

- 配置较大的内存.足够大的内存,是提高 MySQL 数据库性能的方法之一.内存的 IO 比硬盘快的多,可以增加系统的缓冲区容量,使数据在内存停留的时间更长,以减少磁盘的 IO.

- 配置高速磁盘,比如 SSD.

- 合理分配磁盘 IO,把磁盘 IO 分散到多个设备上,以减少资源的竞争,提高并行操作能力.

- 配置多核处理器,MySQL 是多线程的数据库,多处理器可以提高同时执行多个线程的能力.

### 6.2  优化 MySQL 的参数

通过优化 MySQL 的参数可以提高资源利用率,配置参数都在 my.conf 或者 my.ini 文件的[mysqld]组中,常用的参数如下:

| 属性名称                       | 含义                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| key_buffer_size                | 表示索引缓冲区的大小.索弓I缓冲区所有的线程共享.增加索列缓冲区可以得到更好处理的索引(对所有读和多写).当然,这个值也不是越大越好,它的大小取决于内存的大小.如果这个值太大,导致操作系统频繁换页,也会降低系统性能.                                                                                                                                                                                                                                                                                                                                 |
| table_cache                    | 表示同时打开的表的个数.这个值越大,能够同吋打开的表的个数越多,这个值不是越大越好,因为同时打开的表太多会彭响操作系统的性能.                                                                                                                                                                                                                                                                                                                                                                                                                    |
| query_cache_size               | 表示查询缓冲区的大小.该参数需要和query_cache_type配合使用.当query_cache_type值是0时,所有的查询都不使用查询缓冲区.但是query_cache_type=0 不会导致MySQL释放query_cache_size所配置的缓冲区内存.当query_cache_type=l时,所有的查询都将用查询缓冲区,除非在查询语句中指定SQL_NO_CACHE,如SELECT, SQL_NO_CACHE * FROM tb_name.当 query_cache_type=2 时.只有在查询语句中使用SQL_CACHE关键字,查询才会使用查询缓冲区.使用查询缓冲区可以提高查询的速度,这种方式只适用于修改操作少且经常执行相同的查询操作的情况.                                          |
| sort_buffer_size               | 表示排序缓存区的大小.这个值越大,进行排序的速度越快.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| read_buffer_size               | 表示每个线程连续扫描时为扫描的每个表分配的缓冲区的大小(字节).当线程从表中连续读取记录时需要用到这个缓冲区.SET SESSION read_buffer_size=n可以临时设置该参数的值.                                                                                                                                                                                                                                                                                                                                                                              |
| read_rnd_buffer_size           | 表示为毎个线程保留的缓冲区的大小,与read_buffer_size相似.但主要用于存储按特定顺序读取出来的记录.也可以用SET SESSION read_rnd_buffer_size=n 來临时设置该歩数的值.如果频繁进行多次连续扫描,可以增加该值.                                                                                                                                                                                                                                                                                                                                        |
| innodb_buffer_pool_size        | 表示InnoDR类型的表和索引的最大缓存.这个值越大,查询的速度就会越快.但是这个值太大会影响操作系统的性能.                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| max_connections                | 表示数据库的最大连接数.这个连接数不是越大越好,因为这重连接会浪费内存的资源.过多的连接可能会导致MySQL服务器僵死.                                                                                                                                                                                                                                                                                                                                                                                                                              |
| innodb_flush_log_at_trx_commit | 表示何时将缓冲区的数据写入日志丈件,并且将日志文  件写入磁盘中.该券数对于innoDR 51擎非常重要.该泰数有3个值,分别为0,1和2. 值为0时表示每隔1秒将数据写入日志文件并将日志文件写入磁盘c值为1时表示每次提交事务时将数据写入日志文件并将日志文件写入磁盘;值为2时表示每次提交事务时将数据写入日志文件,每隔1秒将日志丈伴写入磁盘.该参数的默认值为1.默认值1安全性最高,但是每次事务提交或事务外的指令都需要把日志写入(flush)硬盘,是比较费时的:0值更快一点,但安全方面比较差;2值日志仍然会每秒写入到硬盘,所以即使出现故障, 一般也不会丢失超过 1-2秒的更新. |
| back_log                       | 表示在mysql暂时停止回答新请求之前的短时间内,多少个请求可以被存在堆栈中.换句话说,该值表示对到来的Tcpflp连接的侦听队列的大小.只有期望在一个短时间内有很多连接,才需要增加该参数的值.操作系统在这个队列大小上也有限制.设定back_log高于操作系统的限制将是无效的.                                                                                                                                                                                                                                                                                  |
| interactive_timeout            | 表示服务器在关闭连按前等待行动的秒数                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| sort_buffer_size               | 表示每个需要进行排序的线程分配的缓冲区的大小.增加这个参数的值可以捉高ORDERBY或GROUP BY操作的速度.默认数值是2 097 144 (2MB).                                                                                                                                                                                                                                                                                                                                                                                                                  |
| thread_cache_size              | 表示可以复用的线程的数量.如果有很多新的线程,为了提高性能可增大该参数的值.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| wait_timeout                   | 表示服务器在关闭一个连按时等待行动的秒数.默认数值是28800.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |