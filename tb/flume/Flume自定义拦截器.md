# Flume 自定义拦截器

需求:`flume`只把符合条件的数据存放到 kafka 或其他后续步骤,那么`flume`应该完成数据清洗的步骤.

---

## 1. 相关代码

### 1.1 主要流程

**实现自定义 flume 的 interceptor** -> **修改 flume 配置文件,指定使用自定义拦截器** -> **重启 flume**

### 1.2 Interceptor 代码

```java
package org.apache.flume.interceptor;

import java.util.ArrayList;
import java.util.List;

import org.apache.flume.Context;
import org.apache.flume.Event;
/**
 * 自定义拦截器
 *
 * <p>
 * </p>
 *
 * 需要实现<code>Interceptor</code>接口<br/>
 * 在intercept里符合条件的判断条件的就返回event,否则返回null
 *
 *
 *
 * @author huanghuapeng 2017年11月10日
 * @see
 * @since 1.0
 */
public class MyInterceptor implements Interceptor {

	@Override
	public void close() {
	}

	@Override
	public void initialize() {
	}

	/**
	 * 实现自己的判断方法
	 */
	@Override
	public Event intercept(Event event) {
		try {
			String body = new String(event.getBody(), "UTF-8");
			if (body.startsWith("allow")) {
				return event;
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	/**
	 * 注意不为null的才添加进list,并返回
	 */
	@Override
	public List<Event> intercept(List<Event> events) {
		List<Event> allowList = new ArrayList<Event>();
		for (Event event : events) {
			if (null != intercept(event)) {
				allowList.add(event);
			}
		}
		return allowList;
	}

	/**
	 *
	 * build方法必须返回自定义的interceptor
	 */
	public static class Builder implements Interceptor.Builder {
		// 使用Builder初始化Interceptor
		@Override
		public Interceptor build() {
			return new MyInterceptor();
		}

		@Override
		public void configure(Context context) {
		}
	}

}
```

编译之后生成`MyInterceptor$Builder.class`和`MyInterceptor.class`,把这两个文件放入`flume-ng-core-1.6.0`里面的`org.apache.flume.interceptor`下,上传服务器`flume/lib/`中去

---

## 2. 配置文件

**修改配置文件 flume.conf**

```properties
#agent组件名称

agent1.sources=r1
agent1.sinks=k1
agent1.channels=c1


# 指定Flume source监听文件目录路径
# 当前用户需有对spoolDir文件的读写权限

agent1.sources.r1.type=spooldir
agent1.sources.r1.spoolDir=/home/dev/flume1.6/irs-flume/test/

# 指定Flume sink

agent1.sinks.k1.type=logger


# 指定Flume channel

agent1.channels.c1.type=memory
agent1.channels.c1.capacity=1000
agent1.channels.c1.transactionCapacity=100

# 绑定source到sink

agent1.sources.r1.channels =c1
agent1.sinks.k1.channel=c1

# 指定拦截器
agent1.sources.r1.interceptors=i1
agent1.sources.r1.interceptors.i1.type=org.apache.flume.interceptor.MyInterceptor$Builder
```

**指定使用拦截器尤为重要**

```properties
# 指定拦截器
agent1.sources.r1.interceptors=i1
agent1.sources.r1.interceptors.i1.type=org.apache.flume.interceptor.MyInterceptor$Builder
```

**重新启动 flume.**

---

## 3. 测试

在 test 文件夹里面添加 hello 文件,文件内容如下:

```shell
allow 1
allow 2
3
4
allow 5
```

预期结果: 输出 `allow 1`,`allow 2`,`allow 5`

测试结果

```java
2017-11-10 15:48:08,589 (pool-3-thread-1) [INFO - org.apache.flume.client.avro.ReliableSpoolingFileEventReader.rollCurrentFile(ReliableSpoolingFileEventReader.java:348)] Preparing to move file /home/dev/flume1.6/irs-flume/test/hello to /home/dev/flume1.6/irs-flume/test/hello.COMPLETED
2017-11-10 15:48:12,658 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 61 6C 6C 6F 77 20 31                            allow 1 }
2017-11-10 15:48:12,659 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 61 6C 6C 6F 77 20 32                            allow 2 }
2017-11-10 15:48:12,659 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 61 6C 6C 6F 77 20 35                            allow 5 }
```

结论: **符合预期,使用自定义的`interceptor`可以实现数据清洗.**
