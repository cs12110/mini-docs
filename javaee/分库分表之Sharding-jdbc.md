# 分库分表之 Sharding-jdbc

当一个表的数据表变得很大的时候,就要考虑分库分表了.

分库分表主要有`sharding-jdbc`和`mycat`,这里不做对比.

本文档只使用于简单的 sharding-jdbc 入门,请知悉.

---

## 1. 数据库设计

因为涉及到分库分表,我们建立连个数据库`frag_0`和`frag_1`.

每个数据库里面分别有 2 张表: `order_t`和`user_t`.

表设计:

- `user_t.uuid`为 uuid 字段
- `order_t.order_id`为时间戳

### 1.1 order 表结构

```sql
DROP TABLE IF EXISTS `order_t`;
CREATE TABLE `order_t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(32) DEFAULT NULL,
  `order_id` varchar(32) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `money` float DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8;
```

### 1.2 user 表结构

```sql
DROP TABLE IF EXISTS `user_t`;
CREATE TABLE `user_t` (
  `uuid` varchar(32) NOT NULL,
  `name` varchar(255) DEFAULT NULL COMMENT '姓名',
  `age` int(3) DEFAULT NULL COMMENT '年龄',
  `gender` varchar(5) DEFAULT NULL COMMENT '性别'
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户表';
```

---

## 2. 示例代码

### 2.1 pom.xml

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <sharding-sphere.version>3.0.0</sharding-sphere.version>
    <commons-dbcp.version>1.4</commons-dbcp.version>
    <slf4j.version>1.7.7</slf4j.version>
    <logback.version>1.2.0</logback.version>
    <fastjson.version>1.2.30</fastjson.version>
    <mysql.version>5.1.38</mysql.version>
    <junit.version>4.12</junit.version>
    <slf4j.version>1.7.5</slf4j.version>
    <logback.version>1.0.13</logback.version>
</properties>


<dependencies>
    <!-- sharding jdbc -->
    <dependency>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-core</artifactId>
        <version>${sharding-sphere.version}</version>
    </dependency>

    <dependency>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-jdbc-orchestration</artifactId>
        <version>${sharding-sphere.version}</version>
    </dependency>

    <dependency>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-core</artifactId>
        <version>${sharding-sphere.version}</version>
    </dependency>

    <dependency>
        <groupId>commons-dbcp</groupId>
        <artifactId>commons-dbcp</artifactId>
        <version>${commons-dbcp.version}</version>
    </dependency>

    <!-- mysql -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>${mysql.version}</version>
    </dependency>

    <!-- fastjson -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>${fastjson.version}</version>
    </dependency>

    <!-- junit -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <scope>test</scope>
        <version>${junit.version}</version>
    </dependency>

    <!-- logback -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>${logback.version}</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>${logback.version}</version>
    </dependency>

</dependencies>
```

### 2.2 日志配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<!-- 控制台标志化输出 -->
	<appender name="STDOUT"
		class="ch.qos.logback.core.ConsoleAppender">
		<encoder>
			<pattern>%d{yyyy-MM-dd HH:mm:ss} %logger{32}:%L - %msg%n
			</pattern>
		</encoder>
	</appender>

	<appender name="ROLLING"
		class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>logs/sys.log</file>
		<encoder>
			<pattern>%d{yyyy-MM-dd HH:mm:ss} %logger{32}:%L - %msg%n
			</pattern>
		</encoder>
	</appender>

	<root level="INFO">
		<appender-ref ref="STDOUT" />
		<appender-ref ref="ROLLING" />
	</root>

</configuration>
```

### 2.3 sharding 工具类

分片策略,请参考: [sharding-jdbc—分片策略 link](https://www.cnblogs.com/mr-yang-localhost/p/8313360.html)

#### 2.3.1 数据库拆分策略实现类

```java
package com.pkgs.guider.strategy;

import java.util.Collection;

import com.pkgs.guider.util.KeyUtil;

import io.shardingsphere.api.algorithm.sharding.PreciseShardingValue;
import io.shardingsphere.api.algorithm.sharding.standard.PreciseShardingAlgorithm;

/**
 * 分库策略
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月19日下午1:35:57
 * @see
 * @since 1.0
 */
public class ShardDbStrategy implements PreciseShardingAlgorithm<String> {

	@Override
	public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<String> shardingValue) {

		/*
		 * availableTargetNames: 配置的数据库,如ds0,ds1
		 *
		 * shardingValue:
		 * {"logicTableName":"t_user","value":"a8efce7649cc4e5488b921ee706b5789"
		 * ,"columnName":"uuid"}
		 */
		String uuid = shardingValue.getValue();
		int sum = KeyUtil.sumAsciiOfStr(uuid);
		int index = sum % availableTargetNames.size();

		int i = 0;
		String db = null;
		for (String each : availableTargetNames) {
			if (i++ == index) {
				db = each;
				break;
			}
		}
		return db;
	}

}
```

#### 2.3.2 数据表拆分策略实现类

只用于用户.

```java
package com.pkgs.guider.strategy;

import java.util.Collection;

import io.shardingsphere.api.algorithm.sharding.PreciseShardingValue;
import io.shardingsphere.api.algorithm.sharding.standard.PreciseShardingAlgorithm;

/**
 * 分表策略
 *
 *
 *
 * <p>
 *
 * 根据uuid的和与表数量求模,确定表
 *
 * @author cs12110 2018年12月19日下午1:35:57
 * @see
 * @since 1.0
 */
public class ShardUserTableStrategy implements PreciseShardingAlgorithm<String> {

	@Override
	public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<String> shardingValue) {

		// int tableNum = 2;
		//
		// String uuid = shardingValue.getValue();
		// int sum = KeyUtil.sumAsciiOfStr(uuid);
		// int index = sum % tableNum;

		return "user_t";
	}
}
```

#### 2.3.3 Key 工具类

```java
package com.pkgs.guider.util;

import java.util.UUID;

/**
 * Id生成器
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月19日下午1:37:32
 * @see
 * @since 1.0
 */
public class KeyUtil {

	/**
	 * uuid
	 *
	 * @return uuid
	 */
	public static String uuid() {
		UUID random = UUID.randomUUID();
		String raw = random.toString();
		StringBuilder after = new StringBuilder(raw.length());
		for (char ch : raw.toCharArray()) {
			if (ch != '-') {
				after.append(ch);
			}
		}
		return after.toString();
	}

	/**
	 * 计算uuid的ascii和
	 *
	 * @param uuid
	 *            uuid
	 * @return long
	 */
	public static int sumAsciiOfStr(String uuid) {
		int num = 0;
		for (char ch : uuid.toCharArray()) {
			num += (int) (ch);
		}
		return num < 0 ? Math.abs(num) : num;
	}

}
```

### 2.2.4 DataSource 工具类

```java
package com.pkgs.guider.shard;

import java.util.Collection;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ConcurrentHashMap;

import javax.sql.DataSource;

import org.apache.commons.dbcp.BasicDataSource;

import com.pkgs.guider.strategy.ShardDbStrategy;
import com.pkgs.guider.strategy.ShardUserTableStrategy;

import io.shardingsphere.api.config.ShardingRuleConfiguration;
import io.shardingsphere.api.config.TableRuleConfiguration;
import io.shardingsphere.api.config.strategy.StandardShardingStrategyConfiguration;
import io.shardingsphere.shardingjdbc.api.ShardingDataSourceFactory;

/**
 * Sharding jdbc 工具类
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月19日上午10:28:47
 * @see
 * @since 1.0
 */
public class ShardJdbcUtil {

	private static final String MYSQL_USER = "root";
	private static final String MYSQL_PASSWORD = "Rojao@123";

	/**
	 * 创建数据源
	 *
	 * @return {@link DataSource}
	 * @throws Exception
	 */
	public static DataSource buildShardingDataSource() throws Exception {
		// 配置真实数据源
		Map<String, DataSource> dataSourceMap = new HashMap<>();
		dataSourceMap.put("ds0", buildDataSource1());
		dataSourceMap.put("ds1", buildDataSource2());

		// 配置Order表规则
		TableRuleConfiguration orderTableRuleConfig = buildOrderTableRuleConfig();
		TableRuleConfiguration userTableRuleConfig = buildUserTableRuleConfig();

		// 配置分片规则
		ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
		Collection<TableRuleConfiguration> tableRuleConfigs = shardingRuleConfig.getTableRuleConfigs();
		tableRuleConfigs.add(orderTableRuleConfig);
		tableRuleConfigs.add(userTableRuleConfig);

		// 设置显示打印执行sql语句
		Properties properties = new Properties();
		properties.put("sql.show", true);

		// 获取数据源对象
		DataSource dataSource = ShardingDataSourceFactory.createDataSource(dataSourceMap, shardingRuleConfig,
				new ConcurrentHashMap<>(), properties);

		return dataSource;
	}

	/**
	 * Order表分库分表规则
	 *
	 * @return {@link TableRuleConfiguration}
	 */
	private static TableRuleConfiguration buildUserTableRuleConfig() {
		// 配置User表规则
		TableRuleConfiguration userTableRuleConfig = new TableRuleConfiguration();
		userTableRuleConfig.setLogicTable("t_user");
		userTableRuleConfig.setActualDataNodes("ds${0..1}.user_t");

		userTableRuleConfig.setDatabaseShardingStrategyConfig(
				new StandardShardingStrategyConfiguration("uuid", new ShardDbStrategy()));

		// 配置分表策略
		userTableRuleConfig.setTableShardingStrategyConfig(
				new StandardShardingStrategyConfiguration("uuid", new ShardUserTableStrategy()));

		return userTableRuleConfig;
	}

	/**
	 * Order表分库分表规则
	 *
	 * @return {@link TableRuleConfiguration}
	 */
	private static TableRuleConfiguration buildOrderTableRuleConfig() {
		// 配置Order表规则
		TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration();
		orderTableRuleConfig.setLogicTable("t_order");
		// 已经定下来,不需要分表策略也可以
		orderTableRuleConfig.setActualDataNodes("ds${0..1}.order_t");

		// 配置分库策略
		orderTableRuleConfig.setDatabaseShardingStrategyConfig(
				new StandardShardingStrategyConfiguration("user_id", new ShardDbStrategy()));

		// 配置分表策略
		// orderTableRuleConfig.setTableShardingStrategyConfig(
		// new InlineShardingStrategyConfiguration("order_id", "order_${order_id
		// % 2}"));

		return orderTableRuleConfig;
	}

	/**
	 * 创建数据源
	 *
	 * @return {@link DataSource}
	 */
	private static DataSource buildDataSource1() {
		// 配置第一个数据源
		BasicDataSource dataSource = new BasicDataSource();
		dataSource.setDriverClassName("com.mysql.jdbc.Driver");
		dataSource.setUrl("jdbc:mysql://10.10.2.233:3306/frag_0?useSSL=false");
		dataSource.setUsername(MYSQL_USER);
		dataSource.setPassword(MYSQL_PASSWORD);
		return dataSource;
	}

	/**
	 * 创建数据源
	 *
	 * @return {@link DataSource}
	 */
	private static DataSource buildDataSource2() {
		// 配置第二个数据源
		BasicDataSource dataSource = new BasicDataSource();
		dataSource.setDriverClassName("com.mysql.jdbc.Driver");
		dataSource.setUrl("jdbc:mysql://10.10.2.233:3306/frag_1?useSSL=false");
		dataSource.setUsername(MYSQL_USER);
		dataSource.setPassword(MYSQL_PASSWORD);

		return dataSource;
	}
}
```

---

## 3. 测试使用

请留意数据库的数据变化和相关日志打印.

### 3.1 新增

#### 3.1.1 新增用户

```java
@Test
public void insertUser() {
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "insert into t_user(uuid,name,age,gender) values(?,?,?,?)";
        PreparedStatement pstm = connection.prepareStatement(sql);
        for (int index = 0; index < 5; index++) {
            pstm.setString(1, KeyUtil.uuid());
            pstm.setString(2, "user" + index);
            pstm.setInt(3, 20 + index);
            pstm.setString(4, "f");
            pstm.executeUpdate();
        }
        pstm.close();
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

#### 3.1.1 新增订单

```java
@Test
public void insertOrder() {
    // 数据库已经存在的用户Id
    List<String> userIdList = Arrays.asList("76c6e4a30b644fe7aefbaf9847b98e3c", "dc7c5ebdf52e46aa827f4784903086d1",
            "369bf6506bd949359ab06a312f84b406", "999c9de0ed2245c8b3abe434d57bb20e",
            "d4a0f658fa664538870bbfb195f43eb6");
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "insert into t_order(user_id,order_id,name,money) values(?,?,?,?)";
        PreparedStatement pstm = connection.prepareStatement(sql);
        for (int index = 0; index < 20; index++) {
            int random = (int) (Math.random() * userIdList.size());
            pstm.setString(1, userIdList.get(random));
            pstm.setLong(2, System.currentTimeMillis());
            pstm.setString(3, "name" + index);
            pstm.setFloat(4, index * 10);
            pstm.executeUpdate();
        }
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3.2 删除

```java
@Test
public void delete() {
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "delete from t_order where user_id=? and order_id=?";
        PreparedStatement pstm = connection.prepareStatement(sql);
        pstm.setString(1, "369bf6506bd949359ab06a312f84b406");
        pstm.setString(2, "1545201925403");
        pstm.executeUpdate();

        pstm.close();
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3.3 更新

```java
@Test
public void update() {
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "update t_order set money=? where user_id=? and order_id=?";
        PreparedStatement pstm = connection.prepareStatement(sql);
        pstm.setFloat(1, 3306.33f);
        pstm.setString(2, "d4a0f658fa664538870bbfb195f43eb6");
        pstm.setString(3, "1545201925828");
        pstm.executeUpdate();

        pstm.close();
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3.4 查询

```java
@Test
public void select() {
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "select * from t_order";
        PreparedStatement pstm = connection.prepareStatement(sql);

        ResultSet resultSet = pstm.executeQuery();

        ResultSetMetaData metaData = resultSet.getMetaData();
        int count = metaData.getColumnCount();

        while (resultSet.next()) {
            Map<String, Object> map = new HashMap<String, Object>();
            for (int index = 1; index <= count; index++) {
                String key = metaData.getColumnName(index);
                Object value = resultSet.getObject(index);
                map.put(key, value);
            }
            System.out.println(JSON.toJSONString(map, true));
        }

        resultSet.close();
        pstm.close();
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3.5 关联查询

```java
@Test
public void joins() {
    try {
        DataSource dataSource = ShardJdbcUtil.buildShardingDataSource();
        Connection connection = dataSource.getConnection();
        String sql = "select * from t_order left join t_user on t_order.user_id = t_user.uuid";
        PreparedStatement pstm = connection.prepareStatement(sql);
        ResultSet resultSet = pstm.executeQuery();

        ResultSetMetaData metaData = resultSet.getMetaData();
        int count = metaData.getColumnCount();

        while (resultSet.next()) {
            Map<String, Object> map = new HashMap<String, Object>();
            for (int index = 1; index <= count; index++) {
                String key = metaData.getColumnName(index);
                Object value = resultSet.getObject(index);
                map.put(key, value);
            }
            System.out.println(JSON.toJSONString(map, true));
        }
        resultSet.close();
        pstm.close();
        connection.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

### 3.6 总结

- 使用 uuid 求模导致数据不均衡,比如存放 10 条数据,可能存在 8 条在 db1.user_t 里面.

- 分库分表适用于数据量比较大的数据表.

- 规划一个好的分库分表策略至关重要.

---

## 4. 参考资料

a. [sharding-jdbc 官网](http://shardingsphere.io/document/current/cn/manual/)

b. [sharding-jdbc 实现分表](https://www.cnblogs.com/boothsun/p/7825853.html)

c. [Springboot 与 sharding-jdbc 整合](http://shardingsphere.io/document/current/cn/manual/sharding-jdbc/configuration/config-spring-boot/)

d. [美团分布式 ID 生成解决方案](https://tech.meituan.com/MT_Leaf.html)

e. [Snowflake 生成算法](https://segmentfault.com/a/1190000011282426)
