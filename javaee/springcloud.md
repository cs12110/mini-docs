# Spring cloud

微服务,分布式,这些都是挡在解决温饱道路上的两座大山.

参考项目为: [cs12110 的 ms-project](https://github.com/cs12110/ms-project)

---

## 1. 基础知识

---

## 2. 项目使用

### 2.1 注册中心

注册中心的高可用,注册中心是 spring cloud 的指挥所,要是一不小心挂掉了,那就尴尬了.

所以配置高可用的注册中心,还是很必要,很必要,很必要的.

再使用`5100`端口,启动项目之后,再使用`5101`启动项目,配置注册中心的 url 为:`eureka.client.serviceUrl.defaultZone=http://127.0.0.1:5100/eureka,http://127.0.0.1:5101`

### 2.1 配置文件

```properties
# the name of application
spring.application.name=ms-reg


# Spring setting
server.port = 5100
# spring.profiles.active=dev
server.tomcat.accept-count=1000
server.tomcat.max-threads=1000
server.tomcat.max-connections=2000

# log setting
logging.config=classpath:logconfig/logback-test.xml

# eureka setting
eureka.client.fetchRegistry=false
eureka.client.registerWithEureka=false
eureka.server.evictionIntervalTimerInMs=2000

# multiple register center
eureka.client.serviceUrl.defaultZone=http://127.0.0.1:5100/eureka,http://127.0.0.1:5101


# kick the disable service out
eureka.server.enable-self-preservation=false
```

### 2.2 运行类

```java
package com.ms.pkgs;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.bind.annotation.RestController;

/**
 * 注册中心
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @since 1.0
 */
@Configuration
@EnableAutoConfiguration
@EnableEurekaServer
@RestController
public class RegApp {

    public static void main(String[] args) {
        SpringApplication.run(RegApp.class, args);
    }
}
```

### 2.3 运行结果

在上述运行成功之后,可以通过浏览器打开:`http://127.0.0.1:5100/`查看spring cloud的运行情况

[register-center](img/register-center.png)

---

## Other

a. [Spring cloud 配置参数参考文档](https://blog.csdn.net/xingbaozhen1210/article/details/80290588)

