# Springboot 与 Rabbitmq 整合

该文档仅适用于 springboot 与 rabbitmq 整合,请知悉.

---

## 1. 配置文件

项目所需配置文件内容如下所示

### 1.1 pom.xml

```xml
 <!-- Spring boot -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.0.4.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>


<dependencies>
    <!-- RabbitMq依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>
```

### 1.2 application.properties 配置文件

相关参数意义: [参数配置参考文档 link](https://blog.csdn.net/lxhjh/article/details/69054342)

```properties
# rabbit
spring.rabbitmq.addresses=47.98.104.252:5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.virtual-host=/
spring.rabbitmq.connection-timeout=15000

spring.rabbitmq.listener.simple.concurrency=1
spring.rabbitmq.listener.simple.acknowledge-mode=manual
spring.rabbitmq.listener.direct.acknowledge-mode=manual
spring.rabbitmq.listener.simple.prefetch=1
```

### 1.3 logback-test.xml 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 控制台标志化输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{16}:%L - %msg%n</pattern>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

---

## 2. 代码

### 2.1 队列配置类

```java
package com.simple.conf;

/**
 * <p/>
 *
 * @author cs12110 created at: 2018/12/28 9:11
 * <p>
 * since: 1.0.0
 */
public class QueueConf {

    /**
     * 简单mq队列名称
     */
    public static final String SIMPLE_QUEUE_NAME = "springrabbit-simplemq";
}
```

### 2.2 App 类

```java
package com.simple;

import com.simple.conf.QueueConf;
import com.simple.provider.Provider;
import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.WebApplicationType;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

import javax.annotation.PostConstruct;

/**
 * 测试消息队列与Springboot整合
 * <p>
 * <p/>
 *
 * @author cs12110 created at: 2018/12/27 17:17
 * <p>
 * since: 1.0.0
 */
@SpringBootApplication
@EnableAsync
@EnableScheduling
public class SimpleApp {

    public static void main(String[] args) {
        System.setProperty("spring.devtools.restart.enabled", "false");
        // 去除web环境
        SpringApplicationBuilder builder = new SpringApplicationBuilder(SimpleApp.class).web(WebApplicationType.NONE);
        // 运行项目
        builder.run(args);
    }

    @Autowired
    private Provider p;

    @PostConstruct
    public void send() {
        //发送消息
        for (int index = 0; index < 10; index++) {
            p.sendMsg(index + ":" + System.currentTimeMillis());
        }
    }

    /**
     * 声明队列,不然会出错.
     *
     * @return Queue
     */
    @Bean
    public Queue buildQueue() {
        /*
         * 第一个参数: 队列名称
         *
         * 第二个参数: 消息是否持久化
         */
        return new Queue(QueueConf.SIMPLE_QUEUE_NAME, true);
    }
}
```

### 2.3 生产者

```java
package com.simple.provider;

import com.simple.conf.QueueConf;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * <p/>
 *
 * @author cs12110 created at: 2018/12/27 17:23
 * <p>
 * since: 1.0.0
 */
@Service
public class Provider {

    private static Logger logger = LoggerFactory.getLogger(Provider.class);

    @Autowired
    private RabbitTemplate template;

    public void sendMsg(String msg) {
        template.convertAndSend("", QueueConf.SIMPLE_QUEUE_NAME, msg);
        logger.info("send:{} is done", msg);
    }
}
```

### 2.4 消费者

```java
package com.simple.consumer;

import com.rabbitmq.client.Channel;
import com.simple.conf.QueueConf;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Headers;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

import java.util.Map;


/**
 * <p/>
 *
 * @author cs12110 created at: 2018/12/27 17:44
 * <p>
 * since: 1.0.0
 */
@Component
public class Consumer {

    private static Logger logger = LoggerFactory.getLogger(Consumer.class);

    @RabbitHandler
    @RabbitListener(queues = QueueConf.SIMPLE_QUEUE_NAME)
    public void consume1(@Payload String msg, @Headers Map<String, Object> headers, Channel channel) {
        try {
            Thread.sleep(1000);
            Long deliveryTag = (Long) headers.get(AmqpHeaders.DELIVERY_TAG);
            logger.info("{} get:[{}] attach with [{}]", "c1", msg, deliveryTag);
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


//    @RabbitHandler
//    @RabbitListener(queues = QueueConf.SIMPLE_QUEUE_NAME)
//    public void consume2(@Payload String msg, @Headers Map<String, Object> headers, Channel channel) {
//        try {
//            Thread.sleep(1000);
//            Long deliveryTag = (Long) headers.get(AmqpHeaders.DELIVERY_TAG);
//            logger.info("{} get:[{}] attach with [{}]", "c2", msg, deliveryTag);
//            channel.basicAck(deliveryTag, false);
//        } catch (Exception e) {
//            e.printStackTrace();
//        }
//    }
}
```

---

## 3. 测试

主要测试 rabbitmq 的两种常用的消费模式:`simple`和`worker`.

### 3.1 simple

直接运行 SimpleApp 即可,相关日志如下

```java
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:0:1545963294577 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:1:1545963295004 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:2:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:3:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:4:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:5:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:6:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:7:1545963295006 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:8:1545963295006 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:9:1545963295006 is done
```

```java
2018-12-28 10:14:56 INFO  c.s.c.Consumer:49 - c2 get:[0:1545963294577] attach with [1]
2018-12-28 10:14:56 INFO  c.s.c.Consumer:35 - c1 get:[1:1545963295004] attach with [1]
2018-12-28 10:14:57 INFO  c.s.c.Consumer:49 - c2 get:[2:1545963295005] attach with [2]
2018-12-28 10:14:57 INFO  c.s.c.Consumer:35 - c1 get:[3:1545963295005] attach with [2]
2018-12-28 10:14:58 INFO  c.s.c.Consumer:49 - c2 get:[4:1545963295005] attach with [3]
2018-12-28 10:14:58 INFO  c.s.c.Consumer:35 - c1 get:[5:1545963295005] attach with [3]
2018-12-28 10:14:59 INFO  c.s.c.Consumer:49 - c2 get:[6:1545963295005] attach with [4]
2018-12-28 10:14:59 INFO  c.s.c.Consumer:35 - c1 get:[7:1545963295006] attach with [4]
2018-12-28 10:15:00 INFO  c.s.c.Consumer:49 - c2 get:[8:1545963295006] attach with [5]
2018-12-28 10:15:00 INFO  c.s.c.Consumer:35 - c1 get:[9:1545963295006] attach with [5]
```

### 3.2 worker

那么该怎么测试 worker 呢?将 Consumer 类里面的 consum2 注释去除即可.

```java
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:0:1545963294577 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:1:1545963295004 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:2:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:3:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:4:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:5:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:6:1545963295005 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:7:1545963295006 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:8:1545963295006 is done
2018-12-28 10:14:55 INFO  c.s.p.Provider:27 - send:9:1545963295006 is done
```

```java
2018-12-28 10:14:56 INFO  c.s.c.Consumer:49 - c2 get:[0:1545963294577] attach with [1]
2018-12-28 10:14:56 INFO  c.s.c.Consumer:35 - c1 get:[1:1545963295004] attach with [1]
2018-12-28 10:14:57 INFO  c.s.c.Consumer:49 - c2 get:[2:1545963295005] attach with [2]
2018-12-28 10:14:57 INFO  c.s.c.Consumer:35 - c1 get:[3:1545963295005] attach with [2]
2018-12-28 10:14:58 INFO  c.s.c.Consumer:49 - c2 get:[4:1545963295005] attach with [3]
2018-12-28 10:14:58 INFO  c.s.c.Consumer:35 - c1 get:[5:1545963295005] attach with [3]
2018-12-28 10:14:59 INFO  c.s.c.Consumer:49 - c2 get:[6:1545963295005] attach with [4]
2018-12-28 10:14:59 INFO  c.s.c.Consumer:35 - c1 get:[7:1545963295006] attach with [4]
2018-12-28 10:15:00 INFO  c.s.c.Consumer:49 - c2 get:[8:1545963295006] attach with [5]
2018-12-28 10:15:00 INFO  c.s.c.Consumer:35 - c1 get:[9:1545963295006] attach with [5]
```

---

## 4. 参考资料

a. [csHB 的 springboot 与 rabbitmq 项目](https://github.com/csHB)

b. [rabbitmq 官网](https://www.rabbitmq.com/tutorials/tutorial-one-spring-amqp.html)
