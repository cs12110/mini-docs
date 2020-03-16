# mysql 数据备份与恢复

在现实生产环境中,数据的备份和恢复至关重要.

---

## 1. 使用`log-bin`备份

对数据备份一般使用 mysql 的二进制文件或者 mysqldump 来对数据进行备份.

现在我们使用 mysql 的二进制日志文件进行数据备份.

首先,**mysql 默认是关闭二进制日志文件的.**

我们第一步就是设置打开二进制日志设置.

mysql 默认安装的配置文件在`/etc/my.cnf`里面,配置如下:

`log_bin`为目录的路径,而不是具体文件.

```properties
[mysqld]
# 添加server-id和log_bin的文件存放位置
server-id=12345
log_bin=/var/lib/mysql/mysql-bin
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

重启 mysql 的服务

```sh
[root@VM_168_161_centos mysql]# systemctl restart mysqld
[root@VM_168_161_centos mysql]# systemctl status mysqld
```

登录 mysql 数据,看 log 日志文件是否开启

```sql
mysql> show variables like '%log_bin%';
+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /var/lib/mysql/mysql-bin       |
| log_bin_index                   | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
6 rows in set (0.00 sec)
```

可以看出,设置是生效的.

现在对数据库进行操作

```sql
mysql> use you_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> drop table my_tab;
Query OK, 0 rows affected (0.05 sec)

mysql>
```

在日志存放文件夹里面,会对应生产相关日志文件

```sh
[root@VM_168_161_centos mysql]# ls /var/lib/mysql |grep mysql-bin
mysql-bin.000001
mysql-bin.000002
mysql-bin.000003
mysql-bin.000004
mysql-bin.index
[root@VM_168_161_centos mysql]#
```

查看某一个日志文件里面的内容,需要用`mysqlbinlog`命令

```sh
[root@VM_168_161_centos mysql]# mysqlbinlog mysql-bin.000002
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#180321 22:41:27 server id 12345  end_log_pos 123 CRC32 0xbb6ff5cc 	Start: binlog v 4, server v 5.7.18-log created 180321 22:41:27
BINLOG '
F2+yWg85MAAAdwAAAHsAAAAAAAQANS43LjE4LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
Acz1b7s=
'/*!*/;
# at 123
#180321 22:41:27 server id 12345  end_log_pos 154 CRC32 0xf9ca55ba 	Previous-GTIDs
# [empty]
# at 154
#180321 22:42:22 server id 12345  end_log_pos 219 CRC32 0x980020f2 	Anonymous_GTID	last_committed=0	sequence_number=1
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#180321 22:42:22 server id 12345  end_log_pos 349 CRC32 0x823a317a 	Query	thread_id=4	exec_time=0	error_code=0
use `you_db`/*!*/;
SET TIMESTAMP=1521643342/*!*/;
SET @@session.pseudo_thread_id=4/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create table my_tab(
id int(11),
name varchar(256)
)
/*!*/;
# at 349
#180321 22:42:26 server id 12345  end_log_pos 396 CRC32 0xe568d62e 	Rotate to mysql-bin.000003  pos: 4
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
[root@VM_168_161_centos mysql]#
```

在上面的日志文件里面可以看见创建 my_tab 的日志记录.

那么我们现在来恢复这个刚才被删掉的表.输入密码即可

```sql
[root@VM_168_161_centos mysql]# mysqlbinlog mysql-bin.000002 --start-position=219 --stop-position=349 | mysql -u root -p  --database=you_db
Enter password:
[root@VM_168_161_centos mysql]#
```

登录数据库验证恢复数据

```sql
mysql> use you_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------+
| Tables_in_you_db |
+------------------+
| my_tab           |
+------------------+
1 row in set (0.00 sec)
```

---

## 2. 其他查看命令

```sql
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       201 |
| mysql-bin.000002 |       396 |
| mysql-bin.000003 |       177 |
| mysql-bin.000004 |       537 |
+------------------+-----------+
4 rows in set (0.00 sec)

mysql> show binlog events in 'mysql-bin.000002'\G;
*************************** 1. row ***************************
   Log_name: mysql-bin.000002
        Pos: 4
 Event_type: Format_desc
  Server_id: 12345
End_log_pos: 123
       Info: Server ver: 5.7.18-log, Binlog ver: 4
*************************** 2. row ***************************
   Log_name: mysql-bin.000002
        Pos: 123
 Event_type: Previous_gtids
  Server_id: 12345
End_log_pos: 154
       Info:
*************************** 3. row ***************************
   Log_name: mysql-bin.000002
        Pos: 154
 Event_type: Anonymous_Gtid
  Server_id: 12345
End_log_pos: 219
       Info: SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
*************************** 4. row ***************************
   Log_name: mysql-bin.000002
        Pos: 219
 Event_type: Query
  Server_id: 12345
End_log_pos: 349
       Info: use `you_db`; create table my_tab(
id int(11),
name varchar(256)
)
*************************** 5. row ***************************
   Log_name: mysql-bin.000002
        Pos: 349
 Event_type: Rotate
  Server_id: 12345
End_log_pos: 396
       Info: mysql-bin.000003;pos=4
5 rows in set (0.00 sec)

ERROR:
No query specified
```

---

## 3. 锁表与解锁

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

mysql> kill idOfProcess
```

---

## 4. 参考资料

a. [优秀博客-cnblogs](https://www.cnblogs.com/kevingrace/p/5907254.html)
