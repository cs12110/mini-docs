# Spring cloud

微服务,分布式,这些都是挡在解决温饱道路上的两座大山.

参考项目为: [cs12110 的 ms-project](https://github.com/cs12110/ms-project)

---

## 1. 基础知识

### 1.1 什么是微服务?

微服务是一种架构风格,一个大型复杂软件应用由一个或多个微服务组成.系统中的各个微服务可被独立部署,各个微服务之间是松耦合的.每个微服务仅关注于完成一件任务并很好地完成该任务.在所有情况下,每个任务代表着一个小的业务能力.

### 1.2 微服务与单体架构区别

- 单体架构所有的模块全都耦合在一块,代码量大,维护困难,微服务每个模块就相当于一个单独的项目,代码量明显减少,遇到问题也相对来说比较好解决.

- 单体架构所有的模块都共用一个数据库,存储方式比较单一,微服务每个模块都可以使用不同的存储方式(比如有的用 redis,有的用 mysql 等),数据库也是单个模块对应自己的数据库.

- 单体架构所有的模块开发所使用的技术一样,微服务每个模块都可以使用不同的开发技术,开发模式更灵活

---

## 2. 注册中心

### 2.1 注册中心

注册中心的高可用,注册中心是 spring cloud 的指挥所,要是一不小心挂掉了,那就尴尬了.

所以配置高可用的注册中心,还是很必要,很必要,很必要的.

设置本机的 host

```sh
mr3306:ms-project mr3306$ cat /etc/hosts |grep peer
# for spring cloud
127.0.0.1 peer1
127.0.0.1 peer2
```

### 2.1 配置文件

**application.properties**

```properties
# the name of application
spring.application.name=ms-reg
# 注意切换
spring.profiles.active=peer1

# Spring setting
server.tomcat.accept-count=1000
server.tomcat.max-threads=1000
server.tomcat.max-connections=2000

# log setting
logging.config=classpath:logconfig/logback-test.xml
```

**application-peer1.properties**

```properties
# Spring setting
server.port=5100
# multiple register center
eureka.instance.hostname=register-center-peer1
eureka.client.serviceUrl.defaultZone=http://peer2:5101/eureka
```

**application-peer2.properties**

```properties
# Spring setting
server.port=5101
# multiple register center
eureka.instance.hostname=register-center-peer2
eureka.client.serviceUrl.defaultZone=http://peer1:5100/eureka
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

分别切换激活环境为:`peer1` 和 `peer2` 运行项目.

### 2.3 运行结果

在上述运行成功之后,可以通过浏览器打开:`http://127.0.0.1:5100/`查看 spring cloud 的运行情况

![register-center](img/register-center.png)

---

## Other

a. [Spring cloud 配置参数参考文档](https://blog.csdn.net/xingbaozhen1210/article/details/80290588)

b. [史上最简单的 SpringCloud 教程](https://blog.csdn.net/forezp/article/details/70148833)

c. [Spring Cloud 系列文章](http://ityouknow.com/spring-cloud)
