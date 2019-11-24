# springcloud feign

1. 为什么要用 Feign,能解决什么事
2. 自己撸一把 主要的几个注解的配置
3. 主要的配置
4. 主要的几个注解 EnableFeignClients FeignClient 工作原理 内部是如何玩转的 HTTP 的,能换 HTTP.
5. Ribbon 作用,实现,工作原理,负载和 Eureka 的关系 Hystrix 作用,实现,工作原理,常用隔离术
6. Feign 是如何集成 Ribbon,Hystrix,Feign Ribbon 模式,断路器模式,后备模式,舱壁模式

---

## 1. 基础知识

Feign 是一个声明式的 Web Service 客户端.它的出现使开发 Web Service 客户端变得很简单.使用 Feign 只需要创建一个接口加上对应的注解,比如：FeignClient 注解.Feign 有可插拔的注解,包括 Feign 注解和 JAX-RS 注解.Feign 也支持编码器和解码器,Spring Cloud Open Feign 对 Feign 进行增强支持 Spring MVC 注解,可以像 Spring Web 一样使用 HttpMessageConverters 等.

### 1.1 基础知识

### 1.2 helloworld

---

## 2. 常用配置与注解

### 2.1 常用配置

feign 常用配置如下:

### 2.2 常用注解

`FeignClient`注解被`@Target（ElementType.TYPE）`修饰,表示`FeignClient`常用注解如下:

|    注解名称     | 作用                                                                                                                  | 备注 |
| :-------------: | --------------------------------------------------------------------------------------------------------------------- | ---- |
|      name       | 指定 FeignClient 的名称,如果项目使用了 Ribbon, name 属性会作为微服务的名称,用于服务发现                               | -    |
|       url       | url 一般用于调试,可以手动指定@FeignClient 调用的地址.                                                                 | -    |
|    decode404    | 当发生 404 错误时,如果该字段为 true,会调用 decoder 进行解码,否则抛出 FeignException.                                  | -    |
|  configuration  | Feign 配置类,可以自定义 Feign 的 Encoder,Decoder,LogLevel,Contract.                                                   | -    |
|    fallback     | 定义容错的处理类,当调用远程接口失败或超时时,会调用对应接口的容错逻辑,fallback 指定的类必须实现@FeignClient 标记的接口 | -    |
| fallbackFactory | 工厂类,用于生成 fallback 类示例,通过这个属性我们可以实现每个接口通用的容错逻辑,减少重复的代码.                        | -    |
|      path       | 定义当前 FeignClient 的统一前缀.                                                                                      | -    |

### 2.3 服务降级模式

参考资料:[feign 断路模式 link](https://blog.csdn.net/liu1390910/article/details/98938841)

|    模式    | 作用                                                                                                                                                          | 备注 |
| :--------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| 断路器模式 | 当服务调用时,断路器将监视这个调用.如果用时太长,断路器会介入并中断调用,如果某一个远程资源的调用失败次数足够多,断路器将采取快速失败,阻止将来调用远程失败的资源. | -    |
|  后备模式  | 当服务调用失败时,尝试通过其他方式执行操作,而不是生成一个异常.                                                                                                 | -    |
|  舱壁模式  | 可以把远程资源的调用分到线程池中,并降低一个缓慢的远程资源调用拖垮整个应用程序的风险.                                                                          | -    |

---

## 3. Feign 工作原理

### 3.1 工作原理

### 3.2 替换 http

feign 默认使用 java 原生的`URLConnection`来请求,每一次都是打开一个线程来进行 http 请求.如果请求数量大的话,会累崩服务器的.所以在生产环境里面,需要更高效的 http 请求工具类代替原生的请求.

Q: So, just tell me ,what the fuck do you want ???

A: Calm down,be cool. httpclient is work for me.

在 pom.xml 添加 httpclient 依赖

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>${feign-httpclient}</version>
</dependency>
```

yml 配置文件加入相关配置

```yml
feign:
  httpclient:
    enabled: true
```

---

## 4. 其他组件

### 4.1 ribbon

### 4.2 eureka

### 4.3 hystrix

---

---

## 6. 参考资料

a. [feign 工作原理](https://blog.csdn.net/u010066934/article/details/80967709)
