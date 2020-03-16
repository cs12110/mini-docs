# Mysql 之读写分离

使用 mysql 主从复制+动态数据实现,主从是实现分离的第一步,然后使用程序来控制调度.

这是何等的窝草啊.

---

## 1. Mysql 主从安装

注意:**本实验关闭了服务器防火墙**

```sh
[root@dev-115 opt]# systemctl stop firewalld 
```


服务器资源

| 服务器 | ip地址      | 备注                  |
| ------ | ----------- | --------------------- |
| master | 10.33.1.115 | root用户密码:rojao123 |
| slave  | 10.33.1.116 | root用户密码:rojao123 |


数据用户

| ip地址      | 数据库用户 | 登录密码   |
| ----------- | ---------- | ---------- |
| 10.33.1.115 | root       | Msyql@3306 |
| 10.33.1.115 | cs12110    | Msyql@3306 |
| 10.33.1.116 | root       | Msyql@3306 |


使用mysql版本: `5.7.18`


### 1.1 安装master节点

首先卸载坑爹的`mariadb`

```sh
[root@dev-115 mysql]# rpm -qa|grep mariadb
mariadb-libs-5.5.52-1.el7.x86_64
[root@dev-115 mysql]# rpm -e mariadb-libs-5.5.52-1.el7.x86_64 --nodeps
[root@dev-115 mysql]# rpm -qa|grep mariadb
[root@dev-115 mysql]#
```

安装mysql数据库

```sh
[root@dev-115 mysql]# rpm -Uvh mysql-community-* 
warning: mysql-community-client-5.7.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-common-5.7.18-1.e################################# [ 25%]
   2:mysql-community-libs-5.7.18-1.el7################################# [ 50%]
   3:mysql-community-client-5.7.18-1.e################################# [ 75%]
   4:mysql-community-server-5.7.18-1.e################################# [100%]
[root@dev-115 mysql]# 
```


修改mysql配置文件,开启二进制日志

```sql
[mysqld]
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M

#
# 开启二进制日志`log-bin`=`存放二进制日志文件`
# server-id设置为100

log-bin=/var/lib/mysql/backup
server-id=100

# 数据存放位置
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

开启mysql服务,获取第一次生成的`root`临时密码

```sh
[root@dev-115 opt]# systemctl start mysqld 
[root@dev-115 opt]# cat /var/log/mysqld.log  |grep password
2018-10-12T05:58:21.575086Z 1 [Note] A temporary password is generated for root@localhost: l/tHEghg,4&m
```

登录mysql并修改root用户密码

```sql
mysql> set password = 'Mysql@3306';
Query OK, 0 rows affected (0.00 sec)

mysql>  grant all privileges on *.* to 'root'@'%' identified by 'Mysql@3306';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```

在配置的二进制的文件夹下生成了日志文件

```sh
[root@dev-115 opt]# ls /var/lib/mysql |grep backup
backup.000001
backup.000002
backup.index
```




### 1.2 安装slave节点

首先卸载坑爹的`mariadb`

```sh
[root@dev-116 mysql]# rpm -qa|grep mariadb
mariadb-libs-5.5.52-1.el7.x86_64
[root@dev-116 mysql]# rpm -e mariadb-libs-5.5.52-1.el7.x86_64 --nodeps
[root@dev-116 mysql]# rpm -qa|grep mariadb
[root@dev-116 mysql]#
```

安装mysql数据库

```sh
[root@dev-116 mysql]# rpm -Uvh mysql-community-* 
warning: mysql-community-client-5.7.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql-community-common-5.7.18-1.e################################# [ 25%]
   2:mysql-community-libs-5.7.18-1.el7################################# [ 50%]
   3:mysql-community-client-5.7.18-1.e################################# [ 75%]
   4:mysql-community-server-5.7.18-1.e################################# [100%]
[root@dev-115 mysql]# 
```


修改mysql配置文件,开启二进制日志

```sql
[mysqld]
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M

#
# 注意server-id要和master的不一致
log-bin=/var/lib/mysql/backup
server-id=1000

datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

开启mysql服务,获取`root`临时密码

```sh
[root@dev-116 mysql]# systemctl start mysqld 
[root@dev-116 mysql]# cat /var/log/mysqld.log |grep password
2018-10-12T06:12:42.649625Z 1 [Note] A temporary password is generated for root@localhost: qI-m3nFSq>2R
```

登录mysql并修改root用户密码

```sql
mysql> set password = 'Mysql@3306';
Query OK, 0 rows affected (0.00 sec)

mysql>  grant all privileges on *.* to 'root'@'%' identified by 'Mysql@3306';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```

在配置的二进制的文件夹下生成了日志文件

```sh
[root@dev-116 opt]# ls /var/lib/mysql |grep backup
backup.000001
backup.000002
backup.index
```

### 1.3 主从复制

成功安装好`master`和`slave`的节点之后,现在开始构建主从复制.

因为主从复制不建议使用数据库的`root`用户,所以创建一个为`cs12110`的数据库用户.

在`master(10.33.1.115)`上创建用户,并配置权限

```sql
mysql> grant replication slave on *.* to 'cs12110'@'%' identified by 'Mysql@3306';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> 
```


在`master`节点`10.33.1.115`获取二进制日志信息

```sh
mysql> show master status;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| backup.000002 |     1116 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> 
```


在`slave`节点`10.33.1.116`设置`slave`

如果设置出错,请使用`stop slave`停掉slave,再重新设置`change master to`

```sql
mysql> change master to master_host='10.33.1.115',master_user='cs12110',master_password='Mysql@3306',master_log_file='backup.000002',master_log_pos=1116;
Query OK, 0 rows affected, 2 warnings (0.02 sec)

mysql> start slave;
Query OK, 0 rows affected (0.01 sec)
```


在`slave`节点上查看`slave`信息,如下所示才算成功

> Slave_IO_State: Waiting for master to send event
> 
> Slave_IO_Running: Yes
> 
> Slave_SQL_Running: Yes

```sql
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.33.1.115
                  Master_User: cs12110
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: backup.000002
          Read_Master_Log_Pos: 1116
               Relay_Log_File: dev-116-relay-bin.000002
                Relay_Log_Pos: 317
        Relay_Master_Log_File: backup.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
        .....
```



### 1.4 数据同步测试

登录master节点,创建数据库和表,然后在slave节点上查询看数据是否同步.

```sql
mysql> create database sync_db charset=utf8;
Query OK, 1 row affected (0.01 sec)

mysql> use sync_db;
Database changed
mysql> create table test_t(
    -> id int primary key comment 'key',
    -> name varchar(256) comment 'name');
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test_t(id,name) values(1,'haiyan');
Query OK, 1 row affected (0.02 sec)
```


在从节点上查询数据

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sync_db            |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use sync_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-------------------+
| Tables_in_sync_db |
+-------------------+
| test_t            |
+-------------------+
1 row in set (0.00 sec)

mysql> select * from test_t;
+----+--------+
| id | name   |
+----+--------+
|  1 | haiyan |
+----+--------+
1 row in set (0.00 sec)

mysql> 
```

如上所示主从完成

---

## 2. SpringBoot 动态数据源

采用`26994263`的github项目作为基础.

### 2.1 github项目

[Spring 动态数据源博客](https://blog.csdn.net/liunian02050328/article/details/75090297)

[读写分离项目Github](https://github.com/269941633/spring-boot-mybatis-mysql-write-read)


### 2.2 备注

按照自己的集群修改配置文件

```yml
logging:
  config: classpath:logback.xml
  path: logs
server:
  port: 80
  session-timeout: 60


mybatis:
     mapperLocations: classpath:/com/fei/springboot/dao/*.xml
     typeAliasesPackage: com.fei.springboot.dao    
     mapperScanPackage: com.fei.springboot.dao
     configLocation: classpath:/mybatis-config.xml

mysql:
    datasource:
        readSize: 1  #读库个数
        type: com.alibaba.druid.pool.DruidDataSource
        mapperLocations: classpath:/com/fei/springboot/dao/*.xml
        configLocation: classpath:/mybatis-config.xml
        write:
           url: jdbc:mysql://10.33.1.115:3306/sync_db?useUnicode=true&characterEncoding=utf-8&useSSL=false
           username: root
           password: Mysql@3306
           driver-class-name: com.mysql.jdbc.Driver
           minIdle: 5
           maxActive: 100
           initialSize: 10
           maxWait: 60000
           timeBetweenEvictionRunsMillis: 60000
           minEvictableIdleTimeMillis: 300000
           validationQuery: select 'x'
           testWhileIdle: true
           testOnBorrow: false
           testOnReturn: false
           poolPreparedStatements: true
           maxPoolPreparedStatementPerConnectionSize: 50
           removeAbandoned: true
           filters: stat
        read01:
           url: jdbc:mysql://10.33.1.116:3306/sync_db?useUnicode=true&characterEncoding=utf-8&useSSL=false
           username: root
           password: Mysql@3306
           driver-class-name: com.mysql.jdbc.Driver
           minIdle: 5
           maxActive: 100
           initialSize: 10
           maxWait: 60000
           timeBetweenEvictionRunsMillis: 60000
           minEvictableIdleTimeMillis: 300000
           validationQuery: select 'x'
           testWhileIdle: true
           testOnBorrow: false
           testOnReturn: false
           poolPreparedStatements: true
           maxPoolPreparedStatementPerConnectionSize: 50
           removeAbandoned: true
           filters: stat
```


修改核心java文件

```java
package com.fei.springboot.config.dbconfig;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.atomic.AtomicInteger;

import javax.sql.DataSource;

import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.DefaultResourceLoader;
import org.springframework.core.io.Resource;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.PlatformTransactionManager;

import com.fei.springboot.util.SpringContextUtil;
import com.github.pagehelper.PageHelper;

@Configuration
@AutoConfigureAfter(DataSourceConfiguration.class)
@MapperScan(basePackages = "com.fei.springboot.dao")
public class MybatisConfiguration {

	private static Logger log = LoggerFactory.getLogger(MybatisConfiguration.class);

	@Value("${mysql.datasource.readSize}")
	private String readDataSourceSize;

	// XxxMapper.xml文件所在路径
	@Value("${mysql.datasource.mapperLocations}")
	private String mapperLocations;

	// 加载全局的配置文件
	@Value("${mysql.datasource.configLocation}")
	private String configLocation;

	@Autowired
	@Qualifier("writeDataSource")
	private DataSource writeDataSource;
	@Autowired
	@Qualifier("readDataSource01")
	private DataSource readDataSource01;


	@Bean(name = "sqlSessionFactory")
	public SqlSessionFactory sqlSessionFactorys() throws Exception {
		log.info("--- sqlSessionFactory init ---");
		try {
			SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
			sessionFactoryBean.setDataSource(roundRobinDataSouceProxy());

			// 读取配置
			sessionFactoryBean.setTypeAliasesPackage("com.fei.springboot.domain");

			// 设置mapper.xml文件所在位置
			Resource[] resources = new PathMatchingResourcePatternResolver().getResources(mapperLocations);
			sessionFactoryBean.setMapperLocations(resources);
			// 设置mybatis-config.xml配置文件位置
			sessionFactoryBean.setConfigLocation(new DefaultResourceLoader().getResource(configLocation));

			// 添加分页插件、打印sql插件
			Interceptor[] plugins = new Interceptor[]{pageHelper(), new SqlPrintInterceptor()};
			sessionFactoryBean.setPlugins(plugins);

			return sessionFactoryBean.getObject();
		} catch (IOException e) {
			log.error("mybatis resolver mapper*xml is error", e);
			return null;
		} catch (Exception e) {
			log.error("mybatis sqlSessionFactoryBean create error", e);
			return null;
		}
	}

	/**
	 * 分页插件
	 * 
	 * @return
	 */
	@Bean
	public PageHelper pageHelper() {
		PageHelper pageHelper = new PageHelper();
		Properties p = new Properties();
		p.setProperty("offsetAsPageNum", "true");
		p.setProperty("rowBoundsWithCount", "true");
		p.setProperty("reasonable", "true");
		p.setProperty("returnPageInfo", "check");
		p.setProperty("params", "count=countSql");
		pageHelper.setProperties(p);
		return pageHelper;
	}
	/**
	 * 把所有数据库都放在路由中
	 * 
	 * @return
	 */
	@Bean(name = "roundRobinDataSouceProxy")
	public AbstractRoutingDataSource roundRobinDataSouceProxy() {

		Map<Object, Object> targetDataSources = new HashMap<Object, Object>();
		// 把所有数据库都放在targetDataSources中,注意key值要和determineCurrentLookupKey()中代码写的一至，
		// 否则切换数据源时找不到正确的数据源
		targetDataSources.put(DataSourceType.write.getType(), writeDataSource);
		targetDataSources.put(DataSourceType.read.getType() + "1", readDataSource01);

		final int readSize = Integer.parseInt(readDataSourceSize);

		// 路由类，寻找对应的数据源
		AbstractRoutingDataSource proxy = new AbstractRoutingDataSource() {
			private AtomicInteger count = new AtomicInteger(0);
			/**
			 * 这是AbstractRoutingDataSource类中的一个抽象方法，
			 * 而它的返回值是你所要用的数据源dataSource的key值，有了这个key值，
			 * targetDataSources就从中取出对应的DataSource，如果找不到，就用配置默认的数据源。
			 */
			@Override
			protected Object determineCurrentLookupKey() {
				String typeKey = DataSourceContextHolder.getReadOrWrite();
				if (typeKey == null) {
					throw new NullPointerException("数据库路由时，决定使用哪个数据库源类型不能为空...");
				}

				if (typeKey.equals(DataSourceType.write.getType())) {
					logger.info("--- 使用数据库write");
					return DataSourceType.write.getType();
				}

				// 读库， 简单负载均衡
				int number = count.getAndAdd(1);
				int lookupKey = number % readSize;
				logger.info("--- 使用数据库read-" + (lookupKey + 1));
				return DataSourceType.read.getType() + (lookupKey + 1);
			}
		};

		proxy.setDefaultTargetDataSource(writeDataSource);// 默认库
		proxy.setTargetDataSources(targetDataSources);
		return proxy;
	}

	@Bean
	public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}

	// 事务管理
	@Bean
	public PlatformTransactionManager annotationDrivenTransactionManager() {
		Object dt = SpringContextUtil.getBean("roundRobinDataSouceProxy");
		return new DataSourceTransactionManager((DataSource) dt);
	}

}
```

---

## 3. mbp之mysql

在mbp上面使用dmg安装mysql,版本为:`8.0.15`


mysql服务所在路径如下:

```sh
mr3306:bin root# ls /usr/local//mysql/support-files/
magic			mysql.server
mysql-log-rotate	mysqld_multi.server
```

```sh
# 启动MySQL服务
/usr/local/mysql/support-files/mysql.server start

# 停止MySQL服务
/usr/local/mysql/support-files/mysql.server stop

# 重启MySQL服务
/usr/local/mysql/support-files/mysql.server restart
```

还有赋予远程登录权限的命令也修改了,变成了三个步骤.

```sql
# 创建账户
create user 'root'@'%' identified by  'root@3306'

# 赋予权限
mysql> grant all privileges on *.* to 'root'@'%' with grant option;

# 刷新权限
mysql> flush privileges;
```