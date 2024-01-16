# mysql 分区

觉得分区有点像单库的分表.

主要适用场景: 归档数据, 冷热隔离数据.

---

### 1. 基础知识

#### 1.1 使用案例

```sql
CREATE TABLE t_partition_info (
    id INT auto_increment,
    content VARCHAR(50) default null comment '名称',
    create_time datetime default current_timestamp,
    primary key (`id`)
) PARTITION BY RANGE (YEAR(create_time)) (
    PARTITION p0 VALUES LESS THAN (2000),
    PARTITION p1 VALUES LESS THAN (2010),
    PARTITION p2 VALUES LESS THAN (2020),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```

Q: 为啥会出现下面的提示?

```sql
SQL Error [1503] [HY000]: A PRIMARY KEY must include all columns in the table's partitioning function
```

A: 构建分区的字段需要包含主键字段,so, ddl 如下:

```sql
CREATE TABLE t_partition_info (
    id INT auto_increment,
    content VARCHAR(50) default null comment '名称',
    create_time datetime default current_timestamp,
    primary key (`id`,`create_time`)
) PARTITION BY RANGE (YEAR(create_time)) (
    PARTITION p0 VALUES LESS THAN (2000),
    PARTITION p1 VALUES LESS THAN (2010),
    PARTITION p2 VALUES LESS THAN (2020),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```

```sql
INSERT into t_partition_info(id, content, create_time) values(1,'2005年内容','2005-01-01 12:00:00');
INSERT into t_partition_info(id, content, create_time) values(2,'2015年内容','2015-01-01 12:00:00');
INSERT into t_partition_info(id, content, create_time) values(3,'2025年内容','2025-01-01 12:00:00');
```

```sql
-- 遍历所有分区查询数据
select * from t_partition_info where 1=1 and create_time >='2015-01-01 00:00:00' ;

-- 使用指定分区查询数据, 因为2015在p2分区,所以查询不到数据
select * from t_partition_info PARTITION(p0) where 1=1 and create_time >='2015-01-01 00:00:00' ;
```

Q: 怎么知道查询是否使用了分区?

A: 可以使用 explain 来查看

```sql
-- 使用全部分区
explain select * from t_partition_info  where 1=1 and create_time >='2000-01-01 00:00:00' ;

id|select_type|table           |partitions|type|possible_keys|key|key_len|ref|rows|filtered|Extra      |
--+-----------+----------------+----------+----+-------------+---+-------+---+----+--------+-----------+
 1|SIMPLE     |t_partition_info|p1,p2,p3  |ALL |             |   |       |   |   3|   33.33|Using where|
```

```sql
-- 这里只使用到p3分区
explain select * from t_partition_info  where 1=1 and create_time >='2020-01-01 00:00:00' ;

id|select_type|table           |partitions|type|possible_keys|key|key_len|ref|rows|filtered|Extra      |
--+-----------+----------------+----------+----+-------------+---+-------+---+----+--------+-----------+
 1|SIMPLE     |t_partition_info|p3        |ALL |             |   |       |   |   1|   100.0|Using where|
```

#### 1.2 多表关联

Q: 多表关联怎么处理?

A: 看看下面这个栗子. :"}

```sql
CREATE TABLE t_partition_info (
    id INT auto_increment,
    content VARCHAR(50) default null comment '名称',
    create_time datetime default current_timestamp,
    primary key (`id`,`create_time`)
) engine = innodb PARTITION BY RANGE (YEAR(create_time)) (
    PARTITION p0 VALUES LESS THAN (2000),
    PARTITION p1 VALUES LESS THAN (2010),
    PARTITION p2 VALUES LESS THAN (2020),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);

CREATE TABLE t_ext_info (
    id INT auto_increment,
    info_id INT default null comment 't_partition_info.',
    content VARCHAR(50) default null comment '名称',
    create_time datetime default current_timestamp,
    primary key (`id`)
);


INSERT into t_partition_info(id, content, create_time) values(1,'2005年内容','2005-01-01 12:00:00');
INSERT into t_partition_info(id, content, create_time) values(2,'2015年内容','2015-01-01 12:00:00');
INSERT into t_partition_info(id, content, create_time) values(3,'2025年内容','2025-01-01 12:00:00');

INSERT into t_ext_info(id,info_id, content, create_time) values(1,1,'测试数据1',now());
INSERT into t_ext_info(id,info_id, content, create_time) values(2,1,'测试数据2',now());
```

```sql
-- 指定partition,不指定时查询全部partition
select
	*
from
	t_partition_info partition(p1)
left join t_ext_info on
	t_partition_info.id = t_ext_info.info_id

id|content|create_time        |id|info_id|content|create_time        |
--+-------+-------------------+--+-------+-------+-------------------+
 1|2005年内容|2005-01-01 12:00:00| 1|      1|测试数据1  |2024-01-15 21:33:52|
 1|2005年内容|2005-01-01 12:00:00| 2|      1|测试数据2  |2024-01-15 21:33:52|

-- 查看执行计划
explain select
	*
from
	t_partition_info partition(p1)
left join t_ext_info on
	t_partition_info.id = t_ext_info.info_id

id|select_type|table           |partitions|type|possible_keys|key|key_len|ref|rows|filtered|Extra                                             |
--+-----------+----------------+----------+----+-------------+---+-------+---+----+--------+--------------------------------------------------+
 1|SIMPLE     |t_partition_info|p1        |ALL |             |   |       |   |   1|   100.0|                                                  |
 1|SIMPLE     |t_ext_info      |          |ALL |             |   |       |   |   2|   100.0|Using where; Using join buffer (Block Nested Loop)|
```

#### 1.3 分区字段数据更新

Q: 分区数据字段更新该咋整?

```sql
-- id=1的数据,现在在分区p1里面
select * from t_partition_info;

id|content|create_time        |
--+-------+-------------------+
 1|2005年内容|2005-01-01 12:00:00|
 2|2015年内容|2015-01-01 12:00:00|
 3|2025年内容|2025-01-01 12:00:00|


-- 查询p1分区,id=1的数据
select * from t_partition_info partition(p1)  where id =1;

id|content|create_time        |
--+-------+-------------------+
 1|2005年内容|2005-01-01 12:00:00|


-- 更新id=1的create_time= '2024-01-01 12:00:00'
update t_partition_info set create_time ='2024-01-15 12:00:00' where id ='1';

-- 指定p1分区查询不到数据
select * from t_partition_info partition(p1)  where id =1;

-- 指定p1分区查询不到数据
select * from t_partition_info partition(p1)  where id =1;

-- 指定p3分区能查询到数据
select * from t_partition_info partition(p3)  where id =1;
```

#### 1.4 设计案例

Q: 如果最近一个月的数据放在一个分区,其他日期都放在其他分期要怎么处理?

A: 可以在分区表达式里面进行相关动态计算划分.

```sql
CREATE TABLE your_table (
    id INT,
    name VARCHAR(50),
    create_time TIMESTAMP,
    value INT
) PARTITION BY RANGE (TO_DAYS(create_time)) (
    PARTITION p_past VALUES LESS THAN (TO_DAYS(CURDATE() - INTERVAL 1 MONTH)),
    PARTITION p_recent VALUES LESS THAN (MAXVALUE)
);
```

我以为可以按照上面这种建分区的,然而,然而:

```
SQL Error [1064] [42000]: Constant, random or timezone-dependent expressions in (sub)partitioning function are not allowed near '),
    PARTITION p_recent VALUES LESS THAN (MAXVALUE)
)' at line 7
```

说明: TO_DAYS 是 MySQL 中的一个日期函数,它的功能是将日期转换为对应的天数.具体而言,TO_DAYS 返回一个日期的天数表示,从公元 0000-00-00 开始算起,直到指定的日期.

```sql
select
	DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s') AS current_datetime,
	TO_DAYS(CURDATE() - INTERVAL 1 MONTH) as last_of_day,
	TO_DAYS(CURDATE()) current_of_day
from
	dual

current_datetime   |last_of_day|current_of_day|
-------------------+-----------+--------------+
2024-01-16 08:57:57|     739235|        739266|
```

---

### 2. 分区策略

Q: 那么分区有哪些策略?

A:

#### 2.1 范围分区 (Range Partitioning)

范围分区根据某一列的范围进行分区.例如,按照日期列进行范围分区:

```sql
CREATE TABLE sales (
    id INT,
    sale_date DATE,
    amount DECIMAL(10, 2)
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p0 VALUES LESS THAN (2000),
    PARTITION p1 VALUES LESS THAN (2005),
    PARTITION p2 VALUES LESS THAN (2010),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```

#### 2.2 列表分区 (List Partitioning)

列表分区根据列中的离散值进行分区.例如,按照区域进行列表分区:

```sql
CREATE TABLE employees (
    emp_id INT,
    emp_name VARCHAR(50),
    emp_region VARCHAR(20)
) PARTITION BY LIST (emp_region) (
    PARTITION p_east VALUES IN ('East'),
    PARTITION p_west VALUES IN ('West'),
    PARTITION p_central VALUES IN ('Central'),
    PARTITION p_other VALUES IN (DEFAULT)
);
```

#### 2.3 散列分区 (Hash Partitioning)

散列分区通过对某列的哈希值进行分区,将数据均匀分布到不同分区.例如,按照用户 ID 进行散列分区:

```sql
CREATE TABLE orders (
    order_id INT,
    user_id INT,
    order_date DATE,
    total_amount DECIMAL(10, 2)
) PARTITION BY HASH (user_id) PARTITIONS 4;
```

#### 2.4 列分区 (Key Partitioning)

列分区类似于散列分区,但是它是基于列的哈希值进行的.例如,按照用户 ID 进行列分区:

```sql
CREATE TABLE transactions (
    trans_id INT,
    user_id INT,
    trans_date DATE,
    trans_amount DECIMAL(10, 2)
) PARTITION BY KEY (user_id);
```

#### 2.5 查询示例

对于范围分区,你可以使用以下 SQL 来查询特定分区的数据:

```sql
-- 查询范围分区 p1 中的数据
SELECT *
FROM sales PARTITION (p1)
WHERE sale_date BETWEEN '2001-01-01' AND '2004-12-31';
```

---

### 3. 参考资料

a. [mysql 分区 blog](https://www.cnblogs.com/mzhaox/p/11201715.html)
