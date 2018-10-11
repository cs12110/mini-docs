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

---