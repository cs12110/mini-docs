# PostgreSQL

PostgreSQL:`免费的对象-关系数据库服务器(ORDBMS),Slogan 是 "世界上最先进的开源关系型数据库".`

By the way,please call me: `post-gress-ql`.

此篇文档建立在熟悉 SQL 语法的前提上,请知悉.

---

## 1. 安装 PostgreSQL

操作系统为: `centos7`,请知悉.

### 1.1 下载安装资源

```sh
# 下载安装资源
# root @ team-2 in /opt/soft/postgresql [17:02:06]
$ yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 安装postgresql
# root @ team-2 in /opt/soft/postgresql [17:02:37]
$ yum install postgresql12

# 安装PostgreSQL-server,足足等下载都1个世纪了
# root @ team-2 in /opt/soft/postgresql [17:09:35]
$ yum install postgresql12-server

# 初始化数据库
# root @ team-2 in /opt/soft/postgresql [17:17:59]
$ /usr/pgsql-12/bin/postgresql-12-setup initdb
Initializing database ... OK

# 启动postgresql任务
# root @ team-2 in /opt/soft/postgresql [17:19:10]
$ systemctl enable postgresql-12
$ systemctl start postgresql-12

# 查看服务状态
# root @ team-2 in /opt/soft/postgresql [17:20:27]
$ systemctl status postgresql-12
● postgresql-12.service - PostgreSQL 12 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-12.service; enabled; vendor preset: disabled)
```

### 1.2 服务命令

```sh
# 这里面有一个有毒的地方,就是root没有权限进入psql,我擦
# 所以得使用postgres用户(安装时默认创建),然后再启动postgresql服务
# root @ team-2 in /opt/soft/postgresql [17:22:10]
$ su - postgres
-bash-4.2$ psql
psql (12.2)
Type "help" for help.

# 创建root为superuser
postgres=# create user root superuser;
CREATE ROLE

# \q退出界面
postgres=# \q

# 开启postgresql服务
# root @ team-2 in /opt/soft/postgresql [17:22:10]
$ systemctl start postgresql-12

# 重启服务
# root @ team-2 in /opt/soft/postgresql [17:22:10]
$ systemctl restart postgresql-12

# 关闭postgresql服务
# root @ team-2 in /opt/soft/postgresql [17:21:03]
$ systemctl stop postgresql-12
```

### 1.3 开启远程访问

```sh
# 找到配置文件位置
# root @ team-2 in /usr/pgsql-12/lib [14:42:09]
$ find / -name postgresql.conf
/var/lib/pgsql/12/data/postgresql.conf

# 修改postgresql.conf
# root @ team-2 in /var/lib/pgsql/12/data [14:43:51]
$ vim postgresql.conf

listen_addresses = '*'
port = 5432

# 修改pg_hba.conf
# root @ team-2 in /var/lib/pgsql/12/data [14:46:49]
$ vim pg_hba.conf
local   replication     all                                     peer
host    all             all             127.0.0.1/32            ident
host    all             all             0.0.0.0/0               md5
```

```sh
# 修改密码
postgres=# alter role postgres with password 'postgres@5432';
ALTER ROLE
postgres=# \q
```

重启`postgresql`服务之后,就可以使用远程登录了

```sh
-bash-4.2$ ./psql -h 47.98.104.252 -p 5432 -U postgres  pg_db;
Password for user postgres:
```

---

## 2. postgrsql 语法

postgresql 常用命令.

### 2.1 数据库操作

postgresql 连接远程数据库: `psql -h localhost -p 5432 -U postgress dbName`

- -h: 数据库地址
- -p: 端口号,默认为`5432`
- -U: 登录用户

##### 显示数据库

```sh
# 创建数据库
postgres=# create database pg_db;
CREATE DATABASE


# 显示数据库
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 pg_db     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
```

##### 进入数据库

```sh
# \c dbName
postgres=# \c pg_db;
You are now connected to database "pg_db" as user "postgres".
pg_db=#
```

##### 删除数据库

```sh
postgres=# drop database if exists pg_db;
DROP DATABASE
```

### 2.2 表操作

关于数据表的操作.

##### 显示表

```sh
# \d: 显示表,想mysql的show tables
# \d tableName:显示表的信息
pg_db=# \d
Did not find any relations.
```

##### 创建表

```sh
# 创建表
pg_db=# create table if not exists t_person_info(
id smallint primary key not null,
name varchar(128) ,
age int not null,
address text
);
CREATE TABLE

# 显示数据库的表
pg_db=# \d
             List of relations
 Schema |     Name      | Type  |  Owner
--------+---------------+-------+----------
 public | t_person_info | table | postgres
(1 row)

# 显示表的详情
pg_db=# \d t_person_info
                   Table "public.t_person_info"
 Column  |          Type          | Collation | Nullable | Default
---------+------------------------+-----------+----------+---------
 id      | smallint               |           | not null |
 name    | character varying(128) |           |          |
 age     | integer                |           | not null |
 address | text                   |           |          |
Indexes:
    "t_person_info_pkey" PRIMARY KEY, btree (id)
```

Q: 如果要设置 id 自增该怎么办?

A: 可以使用 `SERIAL`.

```sh
# id声明为serial
pg_db=#
create table t_bank_card_info(
id serial primary key,
bank_name varchar(128) not null,
card_no varchar(32) not null);

CREATE TABLE

# 显示表信息,默认创建了t_bank_card_info_id_seq序列
pg_db=# \d t_bank_card_info
                                     Table "public.t_bank_card_info"
  Column   |          Type          | Collation | Nullable |                   Default
-----------+------------------------+-----------+----------+----------------------------------------------
 id        | integer                |           | not null | nextval('t_bank_card_info_id_seq'::regclass)
 bank_name | character varying(128) |           | not null |
 card_no   | character varying(32)  |           | not null |
Indexes:
    "t_bank_card_info_pkey" PRIMARY KEY, btree (id)


# 插入数据
pg_db=# insert into t_bank_card_info(bank_name,card_no) values('工商银行','112358');
INSERT 0 1
pg_db=# select * from t_bank_card_info;
 id | bank_name | card_no
----+-----------+---------
  1 | 工商银行  | 112358

```

##### 删除表

```sh
pg_db=# drop table if exists t_person_info;
DROP TABLE
```

##### 注释

在 postgresql 里面的注释和 mysql 的差别比较大.

```sh
# 给表t_person_info添加注释
pg_db=# comment on table t_person_info is 'person info table';

# 给表t_person_info的name字段添加注解
pg_db=# comment on column t_person_info.name is 'name of person';
```

Q: 那不是很简单吗?

A: 但你能想到`\d t_person_info`看不到注释吗?

Q: What???

A: 所以要用特殊的方式来看了.流下来没有技术的眼泪.jpg

```sql
-- 查看表的注释
SELECT
	relname AS tabname,
	cast( obj_description ( relfilenode, 'pg_class' ) AS VARCHAR ) AS COMMENT
FROM
	pg_class c
WHERE
	relname = 't_person_info';
```

```sql
--  查看表字段的注解
SELECT
	attr_t.attname AS column_name,
	desc_t.description AS column_desc
FROM
	pg_class class_t,
	pg_attribute attr_t,
	pg_type type_t,
	pg_description desc_t
WHERE
	attr_t.attnum > 0
	AND attr_t.attrelid = class_t.oid
	AND attr_t.atttypid = type_t.oid
	AND desc_t.objoid = attr_t.attrelid
	AND desc_t.objsubid = attr_t.attnum
	AND class_t.relname = 't_person_info'
ORDER BY
	class_t.relname DESC;
```

##### 表结构操作

```sh
# 修改表名称: t_asset_info -> t_asset
pg_db=# alter table t_asset_info rename to t_asset

# 修改表字段名称
pg_db=# alter table t_asset_info rename personid to person_id;
```

##### 索引操作

表里面需要 **`足够多的数据`** 才会走索引,泪目.

```sh
# 数据表里面有10001条数据
pg_db=# select count(1) as num from t_asset_info;
  num
-------
 10001

# 表结构,owner_id没添加索引
g_db=# \d t_asset_info;
                                         Table "public.t_asset_info"
   Column    |            Type             | Collation | Nullable |                 Default
-------------+-----------------------------+-----------+----------+------------------------------------------
 id          | integer                     |           | not null | nextval('t_asset_info_id_seq'::regclass)
 owner_id    | character varying(32)       |           | not null |
 asset_name  | character varying(128)      |           | not null |
 price       | money                       |           | not null |
 create_time | timestamp without time zone |           |          |
```

在`owner_id`没有索引的情况下,执行计划:`Seq Scan`全表扫描.

```sh
pg_db=# explain analyze select * from t_asset_info where owner_id ='OWNER-1000';
                                               QUERY PLAN
---------------------------------------------------------------------------------------------------------
 Seq Scan on t_asset_info  (cost=0.00..218.01 rows=1 width=40) (actual time=0.222..2.132 rows=1 loops=1)
   Filter: ((owner_id)::text = 'OWNER-1000'::text)
   Rows Removed by Filter: 10000
 Planning Time: 2.886 ms
 Execution Time: 2.192 ms
```

在`owner_id`有索引的情况下,执行计划:`Index Cond`.

```sh
# 添加索引
pg_db=# create index idx_t_asset_info_owener_id on t_asset_info(owner_id);
CREATE INDEX
pg_db=# \d t_asset_info;
                                         Table "public.t_asset_info"
   Column    |            Type             | Collation | Nullable |                 Default
-------------+-----------------------------+-----------+----------+------------------------------------------
 id          | integer                     |           | not null | nextval('t_asset_info_id_seq'::regclass)
 owner_id    | character varying(32)       |           | not null |
 asset_name  | character varying(128)      |           | not null |
 price       | money                       |           | not null |
 create_time | timestamp without time zone |           |          |
Indexes:
    "idx_t_asset_info_owener_id" btree (owner_id)

pg_db=# explain analyze select * from t_asset_info where owner_id ='OWNER-1000';
                                                                QUERY PLAN

-----------------------------------------------------------------------------------------------------------------------------------------
-
 Index Scan using idx_t_asset_info_owener_id on t_asset_info  (cost=0.29..8.30 rows=1 width=40) (actual time=0.139..0.140 rows=1 loops=1)
   Index Cond: ((owner_id)::text = 'OWNER-1000'::text)
 Planning Time: 8.327 ms
 Execution Time: 0.194 ms
(4 rows)
```

Q: 那我不要想要这些索引了,该怎么删除呀?

A: 当初叫人家小索引....

```sh
# 如果要删除索引,这有点奇怪,如果多张表有同一个名称的索引,怎么办???
pg_db=# drop index idx_person_id ;
DROP INDEX
```

---

## 3. 增删改查

Here we go.

使用表结构如下

```sh
pg_db=# \d t_person_info;
                   Table "public.t_person_info"
 Column |          Type          | Collation | Nullable | Default
--------+------------------------+-----------+----------+---------
 id     | integer                |           | not null |
 name   | character varying(128) |           | not null |
 age    | smallint               |           | not null |
```

```sh
pg_db=# \d t_asset_info;
                     Table "public.t_asset_info"
   Column   |          Type          | Collation | Nullable | Default
------------+------------------------+-----------+----------+---------
 id         | integer                |           | not null |
 personid   | integer                |           | not null |
 asset_name | character varying(256) |           | not null |
 price      | money                  |           | not null |
```

#### 3.1 增加数据

看起来这个和 mysql 的语法没什么区别.

```sh
# 新增数据
pg_db=# insert into t_person_info(id,name,age) values(1,'3306',33);
INSERT 0 1

# 查询数据
pg_db=# select id as pid,name ,age from t_person_info where id =1;
 pid | name | age
-----+------+-----
   1 | 3306 |  33
(1 row)

# 批量插入数据
pg_db=# insert into t_person_info(id,name,age) values(2,'haiyan',30),(3,'Aya',20);
INSERT 0 2
pg_db=# select id as pid,name ,age from t_person_info;
 pid |  name  | age
-----+--------+-----
   1 | 3306   |  33
   2 | haiyan |  30
   3 | Aya    |  **20**
```

#### 3.2 删除数据

```sql
# 删除名称不为'haiyan'的数据
pg_db=# delete from t_person_info where name not like 'haiyan';
DELETE 2

pg_db=# select * from t_person_info;
 id |  name  | age
----+--------+-----
  2 | haiyan |  30
```

#### 3.3 更新数据

```sql
g_db=# update t_person_info set age = 18 where id =2 ;
UPDATE 1
pg_db=# select * from t_person_info;
 id |  name  | age
----+--------+-----
  2 | haiyan |  18
```

#### 3.4 查询数据

关联多表查询的`join`和 SQL 里面的`join`相似,请知悉.

```sh
# 资产表信息
pg_db=# insert into t_asset_info(id,personid,asset_name,price) values(1,2,'car',1900.90);
INSERT 0 1
pg_db=# select * from t_asset_info;
 id | personid | asset_name |   price
----+----------+------------+-----------
  1 |        2 | car        | $1,900.90

# 查询
pg_db=# select t_person_info.id ,t_person_info.name , t_person_info.age ,t_asset_info.asset_name , t_asset_info.price  from t_person_info left join t_asset_info on t_asset_info.personid = t_person_info.id;
 id |  name  | age | asset_name |   price
----+--------+-----+------------+-----------
  2 | haiyan |  18 | car        | $1,900.90
```

---

## 4. J&P

Java say hi to postgresql.

在 java 处理 postgresql 的 money 类型的数据时候,使用 double 获取会出现问题,需要进行转换,心累

```sql
select create_time,price::numeric from t_asset_info where owner_id = ?
```

在程序里面然后可以使用`BigDecimal`处理

```java
Date createTime = (Date) resultSet.getObject("create_time");
BigDecimal price = (BigDecimal) resultSet.getObject("price");
```

#### 4.1 deps

```xml
<dependency>
   <groupId>org.postgresql</groupId>
   <artifactId>postgresql</artifactId>
   <version>42.1.1</version>
</dependency>
```

#### 4.2 code

##### 配置

```java
package com.pg;

/**
 * @author cs12110 create at 2020/3/16 10:12
 * @version 1.0.0
 */
public class PostgresqlConf {

    /**
     * URL地址
     */
    public static final String URL = "jdbc:postgresql://47.98.104.252:5432/pg_db";

    /**
     * 数据库用户
     */
    public static final String USER = "postgres";

    /**
     * 登录密码
     */
    public static final String PASSWORD = "postgres@5432";
}
```

##### 工具类

```java
package com.pg;

import java.sql.Connection;
import java.sql.DriverManager;

/**
 * @author cs12110 create at 2020/3/16 10:10
 * @version 1.0.0
 */
public class PostgresqlFactory {

    /**
     * 数据库连接驱动名称
     */
    private static final String POSTGRESQL_DRIVER_NAME = "org.postgresql.Driver";

    /**
     * 获取数据库连接
     *
     * @param url      url
     * @param user     user
     * @param password password
     * @return Connection
     */
    public static Connection getConnection(String url, String user, String password) {
        Connection conn = null;
        try {
            Class.forName(POSTGRESQL_DRIVER_NAME);
            conn = DriverManager.getConnection(url, user, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return conn;
    }

}
```

##### 测试使用

```java
package com.pg;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

/**
 * @author cs12110 create at 2020/3/16 10:15
 * @version 1.0.0
 */
public class PostgresqlMocker {

    public static void main(String[] args) {
        try (
                Connection connection = PostgresqlFactory.getConnection(
                        PostgresqlConf.URL,
                        PostgresqlConf.USER,
                        PostgresqlConf.PASSWORD
                )) {

            select(connection);
            update(connection);
            insert(connection);
            delete(connection);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    /**
     * 选择数据
     *
     * @param connection connection
     */
    private static void select(Connection connection) {
        String sql = "select * from t_person_info where id = ?";
        try {
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setObject(1, 2);

            ResultSet resultSet = preparedStatement.executeQuery();

            while (resultSet.next()) {
                System.out.println("Person name: " + resultSet.getString("name"));
            }

            preparedStatement.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 更新数据
     *
     * @param connection connection
     */
    private static void update(Connection connection) {
        String sql = "update t_person_info set name = ? where id = ?";
        try {
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setObject(1, "Miss haiyan");
            preparedStatement.setObject(2, 2);

            int update = preparedStatement.executeUpdate();

            System.out.println("Update result: " + update);

            preparedStatement.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 新增数据
     *
     * @param connection connection
     */
    private static void insert(Connection connection) {
        String sql = "insert into t_person_info (id,name,age) values(?,?,?)";
        try {
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setObject(1, 10);
            preparedStatement.setObject(2, "Mr 3306");
            preparedStatement.setObject(3, 36);

            int update = preparedStatement.executeUpdate();

            System.out.println("Insert result: " + update);

            preparedStatement.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 删除数据
     *
     * @param connection connection
     */
    private static void delete(Connection connection) {
        String sql = "delete from t_person_info where id = ? ";
        try {
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setObject(1, 10);

            int update = preparedStatement.executeUpdate();

            System.out.println("Delete result: " + update);

            preparedStatement.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 4.3 碎碎念

哈,在这里我们要怎么获取出来自动增长的 key 呢?就像 mybatis 插入数据后,会组装 id 到对象里面,很惊艳,对不对?

```java
/**
* 新增数据
*
* @param connection connection
*/
private static void insert(Connection connection) {
    String sql = "insert into t_asset_info (owner_id,asset_name ,price,create_time) values(?,?,?,now())";
    try {
        // 使用参数: Statement.RETURN_GENERATED_KEYS,设置返回主键
        PreparedStatement preparedStatement = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
        for (int index = 0; index < 10; index++) {
            preparedStatement.setObject(1, "OWNER-" + index);
            preparedStatement.setObject(2, "ASSET-" + index);
            preparedStatement.setObject(3, BigDecimal.valueOf(index));
            preparedStatement.addBatch();
        }

        // 执行批量查询操作,并获取插入数据主键的result set
        int[] values = preparedStatement.executeBatch();
        ResultSet generatedKeys = preparedStatement.getGeneratedKeys();

        // 获取自动增长的组建
        List<Object> genIdList = new ArrayList<>();
        while (generatedKeys.next()) {
            genIdList.add(generatedKeys.getObject(1));
        }

        System.out.println(JSON.toJSONString(genIdList));
        System.out.println("Insert result: " + JSON.toJSONString(values));
        preparedStatement.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

---

## 5. 参考资料

a. [postgresql 官网](https://www.postgresql.org/docs/manuals/)

b. [菜鸟教程 postgresql 教程](https://www.runoob.com/postgresql/postgresql-tutorial.html)

c. [postgresql 执行计划](https://blog.csdn.net/JAVA528416037/article/details/91998019)

d. [postgresql 主键自增](https://blog.csdn.net/u011042248/article/details/49422305)
