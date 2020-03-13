# PostgreSQL

如果纠结于读音的话,ta 读作: `post-gress-ql`.

PostgreSQL:`免费的对象-关系数据库服务器(ORDBMS),Slogan 是 "世界上最先进的开源关系型数据库".`

---

## 1. 安装 PostgreSQL

操作系统为: `centos7`,请知悉.

### 1.1 基础知识

`函数`: 通过函数,可以在数据库服务器端执行指令程序.

`索引`: 用户可以自定义索引方法,或使用内置的 B 树,哈希表与 GiST 索引.

`触发器`: 触发器是由 SQL 语句查询所触发的事件.如：一个 INSERT 语句可能触发一个检查数据完整性的触发器.触发器通常由 INSERT 或 UPDATE 语句触发. 多版本并发控制：PostgreSQL 使用多版本并发控制(MVCC,Multiversion concurrency control)系统进行并发控制,该系统向每个用户提供了一个数据库的"快照",用户在事务内所作的每个修改,对于其他的用户都不可见,直到该事务成功提交.

`规则`: 允许一个查询能被重写,通常用来实现对视图(VIEW)的操作,如插入(INSERT),更新(UPDATE),删除(DELETE).

`数据类型`：包括文本,任意精度的数值数组,JSON 数据,枚举类型,XML 数据等.

`全文检索`：通过 Tsearch2 或 OpenFTS,8.3 版本中内嵌 Tsearch2.

`NoSQL`：JSON,JSONB,XML,HStore 原生支持,至 NoSQL 数据库的外部数据包装器.

### 1.2 下载安装资源

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

### 1.3 服务命令

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

### 1.4 postgrsql 语法

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
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

```

---

## 2.

---

## 参考资料

a. [postgresql 官网](https://www.postgresql.org/docs/manuals/)

b. [菜鸟教程 postgresql 教程](https://www.runoob.com/postgresql/postgresql-tutorial.html)
