# Log4Me

在系统里面日志是至关重要,那么我们来看看主流日志的 Hello world.

---

## 1. log4j2

log4j2 看起来是性能最优的[link](https://logging.apache.org/log4j/2.x/performance.html).

log4j2 官网配置[link](https://logging.apache.org/log4j/2.x/manual/configuration.html)

log4j2 优秀博客[link](https://blog.csdn.net/u011389474/article/details/70054256)

### 1.1 pom.xml

```xml
<dependencies>
    <!-- log4j2
        log4j2其实就是log4j的version2,好尴尬.
    -->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.11.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.11.1</version>
    </dependency>
    <!-- 异步日志 -->
    <dependency>
        <groupId>com.lmax</groupId>
        <artifactId>disruptor</artifactId>
        <version>3.3.4</version>
    </dependency>
</dependencies>
```

### 1.2 log4j2-test.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{36}:%L - %msg%n"/>
        </Console>


        <!-- 日志文件 -->
        <File name="MyFile" fileName="logs/app.log">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{36}:%L - %msg%n"/>
        </File>

        <Async name="Async">
            <AppenderRef ref="MyFile"/>
        </Async>
    </Appenders>

    <Loggers>

        <!-- 设置某个package或者class的日志等级 -->
        <Logger name="com.foo.Bar" level="TRACE"/>

        <AsyncRoot level="INFO">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="Async"/>
        </AsyncRoot>
    </Loggers>
</Configuration>
```

### 1.3 测试

```java
import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.core.LoggerContext;
import org.apache.logging.log4j.core.config.Configuration;
import org.apache.logging.log4j.core.config.LoggerConfig;

/**
 * log4j 测试
 * <p/>
 * <p>
 * since: 1.0.0
 */
public class Log4j2App {

    private static Logger logger = LogManager.getLogger(Log4j2App.class);

    public static void main(String[] args) {
        for (int index = 0; index < 5; index++) {
            logger.info("{}", index);
            logger.debug("{}", index);

            try {
                Thread.sleep(1000);
            } catch (Exception e) {
                e.printStackTrace();
            }

            if (index == 3) {
                if (changeLevel(Log4j2App.class, Level.DEBUG)) {
                    logger.debug("change level to:{}", Level.DEBUG);
                }
            }
        }
    }

    /**
     * 修改logger的level
     *
     * @param clazz class对象
     * @param level level
     * @return boolean
     */
    public static boolean changeLevel(Class<?> clazz, Level level) {
        try {
            // 获取logger config
            LoggerContext ctx = (org.apache.logging.log4j.core.LoggerContext) LogManager.getContext(false);
            Configuration configuration = ctx.getConfiguration();
            LoggerConfig loggerConfig = configuration.getLoggerConfig(clazz.getName());

            // 设置level
            loggerConfig.setLevel(level);

            // 更新环境
            ctx.updateLoggers();
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }
}
```

测试结果

```java
2019-01-03 18:25:09.716 INFO  Log4j2App: - 0
2019-01-03 18:25:10.717 INFO  Log4j2App: - 1
2019-01-03 18:25:11.717 INFO  Log4j2App: - 2
2019-01-03 18:25:12.717 INFO  Log4j2App: - 3
2019-01-03 18:25:13.720 DEBUG Log4j2App: - change level to:DEBUG
2019-01-03 18:25:13.720 INFO  Log4j2App: - 4
2019-01-03 18:25:13.720 DEBUG Log4j2App: - 4
```

---

## 2. logback

logback 也是一个不错的日志框架.

### 2.1 pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.7</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>1.1.7</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-access</artifactId>
        <version>1.1.7</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.1.7</version>
    </dependency>
</dependencies>
```

### 2.2 logback-test.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 控制台标志化输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{32}:%L - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="ROLLING"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/sys.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>log/spark-cms%d{yyyyMMdd}-%i.log</fileNamePattern>
            <!-- each file capacity 10mb -->
            <maxFileSize>10MB</maxFileSize>
            <!-- keep 7 days -->
            <maxHistory>7</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{32}:%L - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="ROLLING"/>
    </root>

</configuration>
```

### 2.3 测试

```java
import ch.qos.logback.classic.Level;
import org.slf4j.ILoggerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/1/3 18:52
 * <p>
 * since: 1.0.0
 */
public class LogbackApp {
    private static Logger logger = LoggerFactory.getLogger(LogbackApp.class);

    public static void main(String[] args) {
        for (int index = 0; index < 5; index++) {
            logger.debug("{}", index);
            logger.info("{}", index);

            if (index == 2) {
                boolean result = changeLevel(LogbackApp.class, Level.DEBUG);
                if (result) {
                    logger.info("change level to:{}", Level.DEBUG);
                }
            }
        }
    }

    public static boolean changeLevel(Class<?> clazz, Level level) {
        try {
            ILoggerFactory factory = LoggerFactory.getILoggerFactory();
            ch.qos.logback.classic.Logger target = (ch.qos.logback.classic.Logger) factory.getLogger(clazz.getName());
            target.setLevel(level);
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }
}
```

测试结果

```java
2019-01-03 19:07:58 INFO  LogbackApp:19 - 0
2019-01-03 19:07:58 INFO  LogbackApp:19 - 1
2019-01-03 19:07:58 INFO  LogbackApp:19 - 2
2019-01-03 19:07:58 INFO  LogbackApp:24 - change level to:DEBUG
2019-01-03 19:07:58 DEBUG LogbackApp:18 - 3
2019-01-03 19:07:58 INFO  LogbackApp:19 - 3
2019-01-03 19:07:58 DEBUG LogbackApp:18 - 4
2019-01-03 19:07:58 INFO  LogbackApp:19 - 4
```

### 2.4 日志参数

| 参数 | 作用                                                                                                                           |
| ---- | ------------------------------------------------------------------------------------------------------------------------------ |
| %p   | 输出日志信息优先级,即 DEBUG,INFO,WARN,ERROR,FATAL                                                                              |
| %d   | 输出日志时间点的日期或时间,默认格式为 ISO8601,也可以在其后指定格式,比如：%d{yyy MMM dd HH:mm:ss,SSS}                           |
| %r   | 输出自应用启动到输出该 log 信息耗费的毫秒数                                                                                    |
| %c   | 输出日志信息所属的类目,通常就是所在类的全名                                                                                    |
| %t   | 输出产生该日志事件的线程名                                                                                                     |
| %l   | 输出日志事件的发生位置,相当于%C.%M(%F:%L)的组合,包括类目名,发生的线程,以及在代码中的行数.举例:Testlog4.main (TestLog4.java:10) |
| %x   | 输出和当前线程相关联的 NDC(嵌套诊断环境),尤其用到像 Java servlets 这样的多客户多线程的应用中.                                  |
| %%   | 输出一个”%”字符                                                                                                                |
| %F   | 输出日志消息产生时所在的文件名称                                                                                               |
| %L   | 输出代码中的行号                                                                                                               |
| %m   | 输出代码中指定的消息,产生的日志具体信息                                                                                        |
| %n   | 输出一个回车换行符,Windows 平台为”\r\n”,Unix 平台为”\n”输出日志信息换行                                                        |

---

## 3. MDC

在现实里面,要追踪一条日志是很麻烦的事,超级麻烦的.

例如: `controller(my#saveOrUpdate) -> service(my#saveOrUpdate,my#isExistsValue) -> dao(my#saveOrUpdate).`

所以需要像 uuid 一样的东西`traceId`,在 controller 的执行前设置到 ThreadLocal 里面去,在这个 controller 调用过程里面,每一个调用到的方法的日志都添加上这个 uuid,那样子就能清晰的知道这调用流程和涉及的 exception 了(只要你在 controller 那里打印当前调用的标志,如 userId,根据 userId 获得 traceId,根据 traceId 获得整一个调用链路).

那么,MDC 可以助你一臂之力.

#### 3.1 设置 Filter

```java
package com.pkgs.component.filter;

import org.slf4j.MDC;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;

import javax.servlet.*;
import java.io.IOException;
import java.util.UUID;


/**
 * <p>
 *
 * @author cs12110 create at 2019-12-22 16:33
 * <p>
 * @since 1.0.0
 */
@Order(0)
@Configuration
public class TraceFilter implements Filter {

    private static final String TRACE_ID_KEY = "traceId";

    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain
    ) throws IOException, ServletException {
        // 设置mdc
        MDC.put(TRACE_ID_KEY, buildTraceId());
        chain.doFilter(request, response);
        // 移除mdc
        MDC.remove(TRACE_ID_KEY);
    }

    @Override
    public void destroy() {

    }


    /**
     * 获取uuid
     *
     * @return String
     */
    private String buildTraceId() {
        UUID uuid = UUID.randomUUID();
        return uuid.toString().replace("-", "").toLowerCase();
    }
}
```

### 3.2 设置日志输出格式

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 控制台标志化输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{16}:%L  [%X{traceId}] - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 将日志写入日志文件 -->
    <!-- <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <file>log/spark-all-cms.log</file>
        <append>true</append>日志追加
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{32}::%method - %msg%n</pattern>
        </encoder>
    </appender> -->

    <appender name="ROLLING"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/sys.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/rookie%d{yyyyMMdd}-%i.log</fileNamePattern>
            <!-- each file capacity 10mb -->
            <maxFileSize>128MB</maxFileSize>
            <!-- keep 7 days -->
            <maxHistory>7</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{32}:%L - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 如需打印sql,请开启 -->

    <logger name="org.apache.http" level="ERROR"/>
    <logger name="org.springboot.sample" level="ERROR"/>
    <logger name="org.springframework.web" level="ERROR"/>
    <logger name="org.springframework.context" level="ERROR"/>
    <logger name="springfox.documentation" level="ERROR"/>
    <logger name="org.springframework.jmx" level="ERROR"/>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="ROLLING"/>
    </root>

</configuration>
```

### 3.3 测试

```java
2019-12-22 17:05:25 INFO  c.p.c.a.ReqLogInterceptor:49  [f884503e1cc141d2a5a360b49eeef87b] - req:{"method":"POST","url":"http://127.0.0.1:4321/magix/sendMessage","mapping":"MagixController#sendMessage"}
2019-12-22 17:05:25 INFO  c.p.c.m.MagixController:33  [f884503e1cc141d2a5a360b49eeef87b] - Function[sendMessage] message:helloworld
2019-12-22 17:05:25 INFO  c.p.s.m.MagixService:23  [f884503e1cc141d2a5a360b49eeef87b] - Function[sendMessage] send:helloworld
2019-12-22 17:05:26 INFO  c.p.c.a.ReqLogInterceptor:49  [e86f902c29d948eba62cffb9a8b0e587] - req:{"method":"POST","url":"http://127.0.0.1:4321/magix/sendMessage","mapping":"MagixController#sendMessage"}
2019-12-22 17:05:26 INFO  c.p.c.m.MagixController:33  [e86f902c29d948eba62cffb9a8b0e587] - Function[sendMessage] message:helloworld
2019-12-22 17:05:26 INFO  c.p.s.m.MagixService:23  [e86f902c29d948eba62cffb9a8b0e587] - Function[sendMessage] send:helloworld
```

---

## 4. 参考资料

a. [logback 日志输出](https://www.cnblogs.com/wenbronk/p/6529161.html)
