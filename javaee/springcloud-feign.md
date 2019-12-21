# springcloud feign

spring cloud 组件

- Eureka: 服务启动时(Eureka Client)都会将服务注册到 Eureka Server,并且 Eureka Client 还可以反过来从 Eureka Server 拉取注册表,从而知道其他服务在哪里.

- Ribbon: 服务间发起请求的时候,基于 Ribbon 做负载均衡,从一个服务的多台机器中选择一台服务器.

- Feign: 基于 Feign 的动态代理机制,根据注解和选择的机器,拼接请求 URL 地址,发起请求.

- Hystrix: 发起请求是通过 Hystrix 的线程池来走的,不同的服务走不同的线程池,实现了不同服务调用的隔离,避免了服务雪崩的问题.

- Zuul: 如果前端、移动端要调用后端系统,统一从 Zuul 网关进入,由 Zuul 网关转发请求给对应的服务.

<!-- 1. 为什么要用 Feign,能解决什么事
2. 自己撸一把 主要的几个注解的配置
3. 主要的配置
4. 主要的几个注解 EnableFeignClients FeignClient 工作原理 内部是如何玩转的 HTTP 的,能换 HTTP.
5. Ribbon 作用,实现,工作原理,负载和 Eureka 的关系 Hystrix 作用,实现,工作原理,常用隔离术
6. Feign 是如何集成 Ribbon,Hystrix,Feign Ribbon 模式,断路器模式,后备模式,舱壁模式 -->

---

## 1. 基础知识

Feign 是一个声明式的 Web Service 客户端.它的出现使开发 Web Service 客户端变得很简单.使用 Feign 只需要创建一个接口加上对应的注解,比如:FeignClient 注解.Feign 有可插拔的注解,包括 Feign 注解和 JAX-RS 注解.Feign 也支持编码器和解码器,Spring Cloud Open Feign 对 Feign 进行增强支持 Spring MVC 注解,可以像 Spring Web 一样使用 HttpMessageConverters 等.

### 1.1 helloworld

[Spring cloud Hello world 项目地址 link](https://github.com/cs12110/ms-project)

```java

package com.ms.pkgs;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.feign.EnableFeignClients;

/**
 * Feign客户端
 *
 *
 * <p>
 *
 * feign继承了ribbon和hystrix,可以实现负载均衡+熔断
 *
 * @author cs12110 2018年12月6日
 * @since 1.0
 */
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class FeignApp {

	public static void main(String[] args) {
		SpringApplication.run(FeignApp.class, args);
	}
}
```

```java
package com.ms.pkgs.service;

import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;

import com.ms.pkgs.service.fallback.FeginServiceFallbackImpl;


/**
 * 调用接口
 *
 *
 * <p>
 *
 * <pre>
 *
 * `@FeignClient` 里面的name为服务项目的名称
 *
 * `@RequestMapping` 为 服务的请求地址
 * </pre>
 *
 * @author cs12110 2018年12月6日
 * @since 1.0
 */
@FeignClient(name = "ms-service", fallback = FeginServiceFallbackImpl.class)
public interface FeignService {

    @RequestMapping("/hello/say")
    public String say(String something);
}
```

```java
package com.ms.pkgs.service.fallback;

import org.springframework.stereotype.Component;

import com.ms.pkgs.service.FeignService;

/**
 * fallback
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月17日下午1:35:03
 * @since 1.0
 */
@Component
public class FeginServiceFallbackImpl implements FeignService {

    @Override
    public String say(String something) {
        return "fallback say: " + something;
    }

}
```

---

## 2. 常用配置与注解

```properties
# ------------ Execution相关的属性的配置 ------------
# 隔离策略，默认是Thread, 可选Thread｜Semaphore
hystrix.command.default.execution.isolation.strategy
# 命令执行超时时间，默认1000ms
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds
# 执行是否启用超时，默认启用true
hystrix.command.default.execution.timeout.enabled
# 发生超时是是否中断，默认true
hystrix.command.default.execution.isolation.thread.interruptOnTimeout
# 最大并发请求数，默认10，该参数当使用ExecutionIsolationStrategy.SEMAPHORE策略时才有效。如果达到最大并发请求数，请求会被拒绝。理论上选择semaphore size的原则和选择thread size一致，但选用semaphore时每次执行的单元要比较小且执行速度快（ms级别），否则的话应该用thread。
# semaphore应该占整个容器（tomcat）的线程池的一小部分。
# Fallback相关的属性,这些参数可以应用于Hystrix的THREAD和SEMAPHORE策略
hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests
# 如果并发数达到该设置值，请求会被拒绝和抛出异常并且fallback不会被调用。默认10
hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests
# 当执行失败或者请求被拒绝，是否会尝试调用hystrixCommand.getFallback() 。默认true



# ------------ Circuit Breaker相关的属性 ------------
hystrix.command.default.fallback.enabled
# 用来跟踪circuit的健康性，如果未达标则让request短路。默认true
hystrix.command.default.circuitBreaker.enabled
# 一个rolling window内最小的请求数。如果设为20，那么当一个rolling window的时间内（比如说1个rolling window是10秒）收到19个请求，即使19个请求都失败，也不会触发circuit break。默认20
hystrix.command.default.circuitBreaker.requestVolumeThreshold
# 触发短路的时间值，当该值设为5000时，则当触发circuit break后的5000毫秒内都会拒绝request，也就是5000毫秒后才会关闭circuit。默认5000
hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds
# 错误比率阀值，如果错误率>=该值，circuit会被打开，并短路所有请求触发fallback。默认50
hystrix.command.default.circuitBreaker.errorThresholdPercentage
# 强制打开熔断器，如果打开这个开关，那么拒绝所有request，默认false
hystrix.command.default.circuitBreaker.forceOpen
# 强制关闭熔断器 如果这个开关打开，circuit将一直关闭且忽略circuitBreaker.errorThresholdPercentage
hystrix.command.default.circuitBreaker.forceClosed



# ------------ Metrics相关参数 ------------
# 设置统计的时间窗口值的，毫秒值，circuit break 的打开会根据1个rolling window的统计来计算。若rolling window被设为10000毫秒，则rolling window会被分成n个buckets，每个bucket包含success，failure，timeout，rejection的次数的统计信息。默认10000
hystrix.command.default.metrics.rollingStats.timeInMilliseconds
# 设置一个rolling window被划分的数量，若numBuckets＝10，rolling window＝10000，那么一个bucket的时间即1秒。必须符合rolling window % numberBuckets == 0。默认10
hystrix.command.default.metrics.rollingStats.numBuckets
# 执行时是否enable指标的计算和跟踪，默认true
hystrix.command.default.metrics.rollingPercentile.enabled
# 设置rolling percentile window的时间，默认60000
hystrix.command.default.metrics.rollingPercentile.timeInMilliseconds
#  设置rolling percentile window的numberBuckets。逻辑同上。默认6
hystrix.command.default.metrics.rollingPercentile.numBuckets
#  如果bucket size＝100，window＝10s，若这10s里有500次执行，只有最后100次执行会被统计到bucket里去。增加该值会增加内存开销以及排序的开销。默认100
hystrix.command.default.metrics.rollingPercentile.bucketSize
# 记录health 快照（用来统计成功和错误绿）的间隔，默认500ms
hystrix.command.default.metrics.healthSnapshot.intervalInMilliseconds


# ------------ Request Context 相关参数 ------------
# 默认true，需要重载getCacheKey()，返回null时不缓存
hystrix.command.default.requestCache.enabled
# 记录日志到HystrixRequestLog，默认true
hystrix.command.default.requestLog.enabled



# ------------ Collapser Properties 相关参数 ------------
# 单次批处理的最大请求数，达到该数量触发批处理，默认Integer.MAX_VALUE
hystrix.collapser.default.maxRequestsInBatch
# 触发批处理的延迟，也可以为创建批处理的时间＋该值，默认10
hystrix.collapser.default.timerDelayInMilliseconds
# 是否对HystrixCollapser.execute() and HystrixCollapser.queue()的cache，默认true
hystrix.collapser.default.requestCache.enabled


# ------------ ThreadPool 相关参数 ------------
# 线程数默认值10适用于大部分情况（有时可以设置得更小），如果需要设置得更大，那有个基本得公式可以follow：requests per second at peak when healthy × 99th percentile latency in seconds + some breathing room
# 每秒最大支撑的请求数 (99%平均响应时间 + 缓存值)
# 比如：每秒能处理1000个请求，99%的请求响应时间是60ms，那么公式是：1000 （0.060+0.012）
# 基本得原则时保持线程池尽可能小，他主要是为了释放压力，防止资源被阻塞。
# 当一切都是正常的时候，线程池一般仅会有1到2个线程激活来提供服务
# 并发执行的最大线程数，默认10
hystrix.threadpool.default.coreSize
# BlockingQueue的最大队列数，当设为－1，会使用SynchronousQueue，值为正时使用LinkedBlcokingQueue。该设置只会在初始化时有效，之后不能修改threadpool的queue size，除非reinitialising thread executor。默认－1。
hystrix.threadpool.default.maxQueueSize
#  即使maxQueueSize没有达到，达到queueSizeRejectionThreshold该值后，请求也会被拒绝。因为maxQueueSize不能被动态修改，这个参数将允许我们动态设置该值。if maxQueueSize == -1，该字段将不起作用
hystrix.threadpool.default.queueSizeRejectionThreshold
# 如果corePoolSize和maxPoolSize设成一样（默认实现）该设置无效。如果通过plugin（https://github.com/Netflix/Hystrix/wiki/Plugins）使用自定义实现，该设置才有用，默认1.
hystrix.threadpool.default.keepAliveTimeMinutes
# 线程池统计指标的时间，默认10000
hystrix.threadpool.default.metrics.rollingStats.timeInMilliseconds
# 将rolling window划分为n个buckets，默认10
hystrix.threadpool.default.metrics.rollingStats.numBuckets
```

### 2.1 常用注解

`FeignClient`注解被`@Target(ElementType.TYPE)`修饰,表示`FeignClient`常用注解如下:

|    注解名称     | 作用                                                                                                                  | 备注 |
| :-------------: | --------------------------------------------------------------------------------------------------------------------- | ---- |
|      name       | 指定 FeignClient 的名称,如果项目使用了 Ribbon, name 属性会作为微服务的名称,用于服务发现                               | -    |
|       url       | url 一般用于调试,可以手动指定@FeignClient 调用的地址.                                                                 | -    |
|    decode404    | 当发生 404 错误时,如果该字段为 true,会调用 decoder 进行解码,否则抛出 FeignException.                                  | -    |
|  configuration  | Feign 配置类,可以自定义 Feign 的 Encoder,Decoder,LogLevel,Contract.                                                   | -    |
|    fallback     | 定义容错的处理类,当调用远程接口失败或超时时,会调用对应接口的容错逻辑,fallback 指定的类必须实现@FeignClient 标记的接口 | -    |
| fallbackFactory | 工厂类,用于生成 fallback 类示例,通过这个属性我们可以实现每个接口通用的容错逻辑,减少重复的代码.                        | -    |
|      path       | 定义当前 FeignClient 的统一前缀.                                                                                      | -    |

### 2.2 服务降级模式

参考资料:[Hystrix 容错机制 link](https://blog.csdn.net/ZZY1078689276/article/details/88655457)

|        模式        | 作用                                                                                                                                                          | 备注 |
| :----------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| 服务熔断(Breaker)  | 当服务调用时,断路器将监视这个调用.如果用时太长,断路器会介入并中断调用,如果某一个远程资源的调用失败次数足够多,断路器将采取快速失败,阻止将来调用远程失败的资源. | -    |
| 服务回退(Fallback) | 当服务调用失败时,尝试通过其他方式执行操作,而不是生成一个异常.                                                                                                 | -    |
| 服务隔离(舱壁模式) | 可以把远程资源的调用分到线程池中,并降低一个缓慢的远程资源调用拖垮整个应用程序的风险.                                                                          | -    |

#### 2.2.1 服务熔断

服务熔断的概念来源于电路系统,在电路系统中存在一种熔断器(Circuit Breaker),当流经该熔断器的电路过大时就会自动切断电路.在微服务架构中,也存在类似现实世界中的服务熔断器,当某个异常条件被触发,直接熔断整个服务,而不是一直等待该服务超时.

这里需要注意的是,服务熔断器会把所有的调用结果都记录下来,如果发现异常的调用次数在单位时间内达到一定的阈值,那么服务熔断机制才会被触发,快速失败就会生效;反之则按照正常的流程执行远程调用.也就是说服务熔断器本身是有状态的,其状态主要分为`Closed`,`Open`,`Half-Open`三种情况.

- Closed:熔断器关闭状态,不对服务调用进行限制,但会对调用失败次数进行积累,达到一定阈值或比例时会启动熔断机制.

- Open:熔断器打开状态,此时对服务的调用将直接返回错误,不执行真正的网络调用.同时,熔断器设置了一个时钟选项,当时钟达到一定时间(这个时间一般设置成平均故障处理时间,也就是 MTTR)时会进入半熔断状态.

- Half-Open:半熔断状态,允许一定量的服务请求,如果调用都成功或达到一定比例则认为调用链路已恢复,关闭熔断器;否则认为调用链路仍然存在问题,又回到熔断器打开状态.

#### 2.2.2 服务回退

服务回退(Fallback)是处理因为服务依赖而导致的服务调用失败的一种有效容错机制(服务调用失败的情况包括异常、拒绝、超时、短路).当远程服务调用发生失败时,服务回退不是直接抛出该异常,而是产生另外的处理机制来应对该异常,相当于执行了另一条路径上的代码或返回一个默认的处理结果,而这条路径上的代码或这个默认处理结果一定满足业务逻辑的实现需求,它的作用只是告知服务的消费者当前调用中存在问题.

#### 2.2.3 服务隔离

在微服务的架构设计中存在一种舱壁隔离模式(Bulkhead Isolation Pattern),顾名思义就是像舱壁一样对资源或失败单元进行隔离,如果一个船舱破了进水,只损失一个船舱,其它船舱可以不受影响.舱壁隔离模式在微服务架构中的应用就是各种服务隔离思想.

所谓的隔离,本质上是对系统或资源进行分割,从而实现当系统发生故障时能够限定传播范围和影响范围,即发生故障后只有出问题的服务不可用,而保证其他服务仍然可用.

而服务的隔离又分为两种情况,分别是线程隔离与进程隔离

**线程隔离**:线程隔离主要通过线程池(Thread Pool)进行隔离,在实际使用时我们会把业务进行分类并交给不同的线程池进行处理.当某个线程池处理一种业务请求发生问题时,不会将故障扩散到其它线程池,也就是说不会影响到其它线程池中所运行的业务,从而保证其它服务可用.
线程隔离机制通过为每个依赖服务分配独立的线程池以实现资源隔离.当其中的一个服务所分配的线程被耗尽,也就是该服务的线程全部处于同步等待状态,也不会影响其他服务的调用,因为其他服务运行在独立的线程池中.

**进程隔离**: 进程隔离相较于线程隔离更加的容易理解,进程隔离就是将一个系统拆分为多个子系统以实现子系统间的物理隔离.由于各个子系统运行在独立的容器和 JVM 中,通过进程隔离使得某一个子系统出现问题不会影响到其它子系统.

---

## 3. Feign 工作原理

### 3.1 工作原理

[工作原理 link](https://blog.csdn.net/weixin_40663800/article/details/88117920)

`通过@EnableFeignCleints注解开启FeignCleint` -> `根据Feign的规则实现接口,接口添加@FeignCleint注解` -> `程序启动进行包扫描,扫描所有的@ FeignCleint的注解的类注入到ioc容器中` -> `接口的方法被调用,通过jdk的代理生成具体的RequesTemplate` -> `RequesTemplate在生成Request` -> `Request交给Client去处理,其中Client可以是HttpUrlConnection,HttpClient也可以是Okhttp` -> `Client被封装到LoadBalanceClient类,结合Ribbon做负载均衡`.

### 3.2 替换 http

feign 默认使用 java 原生的`URLConnection`来请求,每一次都是打开一个线程来进行 http 请求.如果请求数量大的话,会累崩服务器的.所以在生产环境里面,需要更高效的 http 请求工具类代替原生的请求.

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

spring cloud 的框架流程如下

---

## 5. 参考资料

a. [feign 官方文档](https://cloud.spring.io/spring-cloud-static/spring-cloud-openfeign/2.0.4.RELEASE/single/spring-cloud-openfeign.html)

b. [feign 工作原理博客](https://blog.csdn.net/u010066934/article/details/80967709)

c. [深入理解 Feign 之源码解析](https://zhuanlan.zhihu.com/p/28593019)

d. [SpringCloud 基础知识](https://blog.csdn.net/weixin_41556963/article/details/88321923)
