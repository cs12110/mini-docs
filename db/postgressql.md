# PostgreSQL

PostgreSQL:`免费的对象-关系数据库服务器(ORDBMS),Slogan 是 "世界上最先进的开源关系型数据库".`

By the way,please call me: `post-gress-ql`.

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

##### 删除表

```sh
pg_db=# drop table if exists t_person_info;
DROP TABLE
```

##### 注解

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

A: 所以要用特殊的方式来看了.流下来没有技术的眼泪.

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

---

## 3. 增删改查

#### 3.1 增加数据

#### 3.2 删除数据

#### 3.3 更新数据

#### 3.4 查询数据

---

## 4. J&P

#### 4.1 deps

#### 4.2 code

---

## 5. 参考资料

a. [postgresql 官网](https://www.postgresql.org/docs/manuals/)

b. [菜鸟教程 postgresql 教程](https://www.runoob.com/postgresql/postgresql-tutorial.html)
