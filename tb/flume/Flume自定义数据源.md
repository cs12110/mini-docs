# Flume 自定义数据源

现实生产环境中,`flume`自带的数据源可能不能满足生产需求,那么就需要实现自定义的`source`.

---

## 1. 自定义 source 代码

### 1.1 MysqlSource 代码

**相关依赖 jar 必须放入 flume 的 lib 文件夹下**

```java
package org.apache.flume.source;

import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.regex.Pattern;

import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.PollableSource;
import org.apache.flume.conf.Configurable;
import org.apache.flume.event.EventBuilder;
import org.apache.flume.util.JdbcUtil;
import org.apache.flume.util.JsonUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 自定义mysql source
 *
 * @author Mr3306
 *
 */
public class MysqlSource extends AbstractSource implements Configurable, PollableSource {

	private static Logger LOG = LoggerFactory.getLogger(MysqlSource.class);

	/**
	 * 数据库Url
	 */
	private String url = "";

	/**
	 * 数据库用户
	 */
	private String user = "";

	/**
	 * 登录密码
	 */
	private String password = "";

	/**
	 * 执行sql
	 */
	private String sql = "";

	/**
	 * 间隔时间:秒
	 */
	private String interval;

	@Override
	public Status process() throws EventDeliveryException {
		try {

			LOG.info("Url: " + url);
			LOG.info("User: " + user);
			LOG.info("Password: " + password);
			LOG.info("Sql: " + sql);

			Connection conn = JdbcUtil.getConnection(url, user, password);
			Statement stm = conn.createStatement();
			ResultSet result = stm.executeQuery(sql);

			ResultSetMetaData metaData = result.getMetaData();
			int columnCount = metaData.getColumnCount();

			List<Event> events = new ArrayList<Event>();
			while (result.next()) {
				Map<String, Object> map = JdbcUtil.translateResultSetToMap(result, metaData, columnCount);
				/**
				 * 增加头部信息,测试在interceptor获取出来
				 */
				Map<String, String> headMap = new HashMap<String, String>(2);
				headMap.put("type", "mysql");
				headMap.put("timestamp", String.valueOf(System.currentTimeMillis()));

				String json = JsonUtil.toStr(map);
				Event event = EventBuilder.withBody(json.getBytes());

				event.setHeaders(headMap);
				events.add(event);
			}
			result.close();
			stm.close();
			conn.close();

			LOG.info("Send msg size: " + events.size() + " and interval is: " + interval);
			this.getChannelProcessor().processEventBatch(events);

			// 歇一会儿
			startInterval(interval);

		} catch (Exception e) {
			e.printStackTrace();
		}

		return Status.READY;
	}

	@Override
	public void configure(Context ctx) {
		interval = ctx.getString("interval", "1000");
		url = ctx.getString("url", JdbcUtil.URL);
		user = ctx.getString("user", JdbcUtil.USER);
		password = ctx.getString("password", JdbcUtil.PASSWORD);
		sql = ctx.getString("sql", JdbcUtil.DEF_SQL);
	}

	private static final Pattern NUM_PATTERN = Pattern.compile("^\\d+$");

	/**
	 * 暂停时长
	 *
	 * @param seconds
	 *            秒数
	 */
	private void startInterval(String seconds) {
		int times = 10000;
		boolean isNum = NUM_PATTERN.matcher(seconds).matches();
		if (isNum) {
			times = Integer.parseInt(seconds) * 1000;
		}

		LOG.info("Actully the interval time is: " + times + ", isNum:" + isNum);
		try {
			Thread.sleep(times);
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

}
```

代码中继承了`AbstractSource`和实现了接口`Configurable`,`PollableSource`.

其中复写方法`process()`实现业务逻辑,复写`configure()`来获取自定义配置参数值.

### 1.2 拦截器 MyInterceptor 代码

在拦截器里面对数据进行处理

```java
package org.apache.flume.interceptor;

import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.apache.flume.Context;
import org.apache.flume.Event;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 实现自定义拦截器
 *
 * @author Mr3306
 *
 */
public class MyInterceptor implements Interceptor {

	private static final Logger LOG = LoggerFactory.getLogger(MyInterceptor.class);

	/**
	 * 使用字符串编码
	 */
	private static final String CHARSET = "UTF-8";

	@Override
	public void close() {

	}

	@Override
	public void initialize() {

	}

	@Override
	public Event intercept(Event event) {
		if (null == event) {
			return null;
		}

		LOG.info("Passing away my interceptor!!!");

		/*
		 * 获取event携带的信息
		 */
		Map<String, String> headMap = event.getHeaders();
		String str = new String(event.getBody(), Charset.forName(CHARSET));
		if (str != null && !"".equals(str)) {
			LOG.info("HeadMap.type: " + headMap.get("type") + ",timestamp: " + headMap.get("timestamp"));
			event.setBody(("Interceptor by 3306: " + str).getBytes());
		}
		return event;
	}

	@Override
	public List<Event> intercept(List<Event> events) {
		if (null == events || events.isEmpty()) {
			return null;
		}
		List<Event> allowList = new ArrayList<Event>(events.size());
		events.forEach((each) -> {
			Event tmp = intercept(each);
			if (null != tmp) {
				allowList.add(tmp);
			}
		});
		return allowList;
	}

	/**
	 * Builder which builds new instance of the StaticInterceptor.
	 */
	public static class Builder implements Interceptor.Builder {
		@Override
		public void configure(Context context) {
		}

		@Override
		public Interceptor build() {
			LOG.info("Build MyInterceptor instance");
			return new MyInterceptor();
		}
	}
}
```

### 1.3 LoggerSink

`LoggerSink`如要完整显示所有传递过来的携带信息,需要进行改写.

```java
package org.apache.flume.sink;

import org.apache.flume.Channel;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.Transaction;
import org.apache.flume.conf.Configurable;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 修改LoggerSink
 *
 * @author Mr3306
 *
 */
public class LoggerSink extends AbstractSink implements Configurable {

	private static final Logger logger = LoggerFactory.getLogger(LoggerSink.class);

	@Override
	public void configure(Context context) {

	}

	@Override
	public Status process() throws EventDeliveryException {
		Status result = Status.READY;
		Channel channel = getChannel();
		Transaction transaction = channel.getTransaction();
		Event event = null;

		try {
			transaction.begin();
			event = channel.take();

			if (event != null) {
				if (logger.isInfoEnabled()) {

					String str = new String(event.getBody());

					logger.info("Event: " + str);
				}
			} else {
				result = Status.BACKOFF;
			}
			transaction.commit();
		} catch (Exception ex) {
			transaction.rollback();
			throw new EventDeliveryException("Failed to log event: " + event, ex);
		} finally {
			transaction.close();
		}

		return result;
	}
}
```

### 1.4 JdbcUtil 代码

主要实现 Jdbc 的连接

```java
package org.apache.flume.util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.util.HashMap;
import java.util.Map;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Jdbc辅助类
 *
 * @author Mr3306
 *
 */
public class JdbcUtil {

	private static final Logger LOG = LoggerFactory.getLogger(JdbcUtil.class);

	/**
	 * MySQL连接驱动
	 */
	private static final String MYSQL_DRIVER = "com.mysql.jdbc.Driver";

	/**
	 * url
	 */
	public static final String URL = "jdbc:mysql://192.168.1.103:3306/big_fish?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8";

	/**
	 * 登录用户
	 */
	public static final String USER = "root";

	/**
	 * 密码
	 */
	public static final String PASSWORD = "root";

	/**
	 * 缺省sql
	 */
	public static final String DEF_SQL = "select 1 =1 as num";

	/**
	 * 获取默认连接
	 *
	 * @return {@link Connection}
	 */
	public static Connection getDefConnection() {
		return getConnection(URL, USER, PASSWORD);
	}

	/**
	 * 获取数据库连接
	 *
	 * @param url
	 *            数据库连接url
	 * @param user
	 *            用户
	 * @param password
	 *            登录密码
	 * @return {@link Connection}
	 */
	public static Connection getConnection(String url, String user, String password) {
		Connection conn = null;
		try {
			Class.forName(MYSQL_DRIVER);
			conn = DriverManager.getConnection(url, user, password);
		} catch (Exception e) {
			e.printStackTrace();
			LOG.error("Get connection occur a error: " + e);
		}

		return conn;
	}

	/**
	 * 将ResultSet里面带有的字段和值转换成Map
	 *
	 * @param result
	 *            {@link ResultSet}
	 * @param metaData
	 *            ({@link ResultSetMetaData}
	 * @param columnCount
	 *            the count of column
	 * @return Map<String,Object>
	 */
	public static Map<String, Object> translateResultSetToMap(ResultSet result, ResultSetMetaData metaData,
			int columnCount) {
		Map<String, Object> map = new HashMap<String, Object>(1);
		try {
			for (int index = 1; index <= columnCount; index++) {
				String key = metaData.getColumnName(index);
				Object value = result.getObject(key);
				map.put(key, value);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return map;
	}

}
```

获取 sql 里面查询出来的所有列的名称和值代码如下:

```java
Statement stm = connection.createStatement();
ResultSet result = stm.executeQuery(sql);

/**
 * 获取源数据
 */
ResultSetMetaData metaData = result.getMetaData();
/**
 * 列数
 */
int columnCount = metaData.getColumnCount();

while (result.next()) {
	Map<String, Object> map = new HashMap<String, Object>();
	for (int index = 1; index <= columnCount; index++) {
		String key = metaData.getColumnName(index);
		Object value = result.getObject(key);
		map.put(key, value);
	}
	System.out.println(map);
}
```

### 1.5 JsonUtil 代码

```java
package org.apache.flume.util;

import com.alibaba.fastjson.JSON;

/**
 * JSON辅助类
 *
 * @author Mr3306
 *
 */
public class JsonUtil {

	/**
	 * 转换成json字符串
	 *
	 * @param t
	 *            对象
	 * @return String
	 */
	public static <T> String toStr(T t) {
		if (null == t) {
			return "";
		}
		return JSON.toJSONString(t);
	}

}
```

---

## 2. 运行配置

### 2.1 配置文件

传入数据库的`url`,`user`,`password`,以及需要执行的`sql`语句,提高代码的复用性

```properties
agent.sources = src1
agent.channels = ch1
agent.sinks = sink1

agent.sources.src1.type = org.apache.flume.source.MysqlSource
agent.sources.src1.interval=20
agent.sources.src1.url=jdbc:mysql://192.168.1.100:3306/big_fish?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8
agent.sources.src1.user=root
agent.sources.src1.password=root
agent.sources.src1.sql=select `from`,`to` from chat_t where 1=1

agent.sources.src1.channels = ch1
agent.sources.src1.interceptors = i1
agent.sources.src1.interceptors.i1.type=org.apache.flume.interceptor.MyInterceptor$Builder


agent.sinks.sink1.type = logger
agent.sinks.sink1.channel = ch1

agent.channels.ch1.type = memory
agent.channels.ch1.capacity = 1000
```

### 2.3 启动脚本

```sh
#!/bin/bash

echo -e 'Delete all the logs'

rm -rf logs/*

rm -rf spool.out


echo -e 'Startup the flume of spool dir'

bin/flume-ng agent --conf conf/ --conf-file conf/flume-mysql.conf --name agent  &
```

---

## 3.测试结果

```json
27 Nov 2017 00:58:30,535 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.process:63)  - Url: jdbc:mysql://192.168.1.100:3306/big_fish?useUnicode=true&characterEncoding=utf8&characterSetResults=utf8
27 Nov 2017 00:58:30,535 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.process:64)  - User: root
27 Nov 2017 00:58:30,535 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.process:65)  - Password: root
27 Nov 2017 00:58:30,535 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.process:66)  - Sql: select `from`,`to` from chat_t where 1=1
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.process:92)  - Send msg size: 3 and interval is: 20
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:44)  - Passing away my interceptor!!!
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:52)  - HeadMap.type: mysql,timestamp: 1511715510553
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:44)  - Passing away my interceptor!!!
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:52)  - HeadMap.type: mysql,timestamp: 1511715510553
27 Nov 2017 00:58:30,554 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:44)  - Passing away my interceptor!!!
27 Nov 2017 00:58:30,555 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.interceptor.MyInterceptor.intercept:52)  - HeadMap.type: mysql,timestamp: 1511715510553
27 Nov 2017 00:58:30,555 INFO  [PollableSourceRunner-MysqlSource-src1] (org.apache.flume.source.MysqlSource.startInterval:129)  - Actully the interval time is: 20000, isNum:true
27 Nov 2017 00:58:34,533 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:61)  - Event: Interceptor by 3306: {"from":"from","to":"to"}
27 Nov 2017 00:58:34,534 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:61)  - Event: Interceptor by 3306: {"from":"from","to":"to"}
27 Nov 2017 00:58:34,534 INFO  [SinkRunner-PollingRunner-DefaultSinkProcessor] (org.apache.flume.sink.LoggerSink.process:61)  - Event: Interceptor by 3306: {"from":"from","to":"to"}
```
