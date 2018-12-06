# Spring 常用技巧

本文档记录 Spring 经常使用到的一些技巧.

---

## 1. SpringBoot 与 maven

在创建了一个父级项目`mvn-parent`之后,然后新建两个子模块`mvn-springboot`与`mvn-common`.但是只想在`mvn-sprigboot`模块引入`springboot`的依赖而`mvn-common`不引入,那么按照之前的做法在`mvn-parent`引入`springboot`的依赖是行不通.

在`mvn-springboot`引入`springboot`的 xml 配置如下:

```xml
<!-- Spring boot -->
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>1.5.2.RELEASE</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<!-- 其他依赖 -->
<dependencies>
	...
<dependencies>
```

## 2. 切面计算执行耗时

在性能测试里面,我们需要找到某一个方法执行耗时多久的情况,这是 AOP 绝佳的一个使用场景.

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

/**
 * 使用AOP统计Service里面每个执行方法耗时
 *
 * <p>
 *
 * <pre>
 * <code>Before</code>:方法前执行
 * </pre>
 *
 * <pre>
 * <code>AfterReturning</code>:运行方法后执行
 * </pre>
 *
 * <pre>
 * <code>AfterThrowing<code>:Throw后执行
 * </pre>
 *
 * <pre>
 * <code>After</code>:无论方法以何种方式结束,都会执行(类似于finally)
 * </pre>
 *
 * <pre>
 * <code>Around</code>:环绕执行
 * </pre>
 *
 * @author huanghuapeng 2017年9月18日
 * @see
 * @since 1.0
 */
@Aspect
@Component
public class ServiceExecTimeCalculateInteceptor {

    private static Logger logger = LoggerFactory
            .getLogger(ServiceExecTimeCalculateInteceptor.class);

    /**
     * 切面表达式
     *
     * <pre>
     * execution(public * * (. .)) 任意公共方法被执行时,执行切入点函数
     * </pre>
     *
     * <pre>
     * execution( * set* (. .)) 任何以一个"set" 开始的方法被执行时,执行切入点函数
     * </pre>
     *
     * <pre>
     * execution( * com.demo.service.AccountService.* (. .)) 当接口AccountService 中的任意方法被执行时,执行切入点函数
     * </pre>
     *
     * execution( * com.demo.service.*.* (. .)) 当service 包中的任意方法被执行时,执行切入点函数
     */
    private static final String serviceMethodsExp = "execution(* cn.rojao.irs.ds.service.impl.*.*(..))";

    /**
     * 计算用时
     *
     * <p>
     * 一定要返回执行后的结果,不然原方法没有返回值?
     *
     * @param joinPoint
     *            {@linkplain ProceedingJoinPoint}
     * @return Object
     */
    @Around(serviceMethodsExp)
    public Object timeAround(ProceedingJoinPoint joinPoint) {
        Object proceed = null;

        long start = System.currentTimeMillis();
        try {
            proceed = joinPoint.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }

        long end = System.currentTimeMillis();

        StringBuilder str = new StringBuilder();
        str.append(getExecMethodName(joinPoint));
        str.append(" spend: ");
        str.append((end - start));
        str.append(" ms");

        logger.info(str.toString());

        return proceed;
    }

    /**
     * 获取执行方法名称
     *
     * @param joinPoint
     *            切点
     * @return String
     */
    private String getExecMethodName(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String methodName = signature.getDeclaringTypeName() + "." + signature.getName();

        return methodName;
    }
}
```

---

## 3. Aop 与多注解

在一个被拦截的方法上有多个注解,这个的执行是怎样子的?

### 3.1 自定义注解

```java
package com.fei.springboot.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Anno1 {

}
```

```java
package com.fei.springboot.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Anno2 {

}
```

### 3.2 切面

```java
package com.fei.springboot.aop;

import java.lang.reflect.Method;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class Anno1Aspect {

	private Logger logger = LoggerFactory.getLogger(Anno1Aspect.class);

	@Pointcut("@annotation(com.fei.springboot.annotation.Anno1)")
	public void logPointCut() {

	}

	@Around("logPointCut()")
	public Object cut(ProceedingJoinPoint joinPoint) throws Throwable {

		MethodSignature signature = (MethodSignature) joinPoint.getSignature();
		Method method = signature.getMethod();

		logger.info("proceed:{}#{}", joinPoint.getTarget().getClass().getName(), method.getName());
		logger.info("执行判断方法");
		Object value = joinPoint.proceed();

		logger.info("return:{}", value);

		return value;
	}

}
```

```java
package com.fei.springboot.aop;

import java.lang.reflect.Method;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class Anno2Aspect {

	private Logger logger = LoggerFactory.getLogger(Anno2Aspect.class);

	@Pointcut("@annotation(com.fei.springboot.annotation.Anno2)")
	public void logPointCut() {

	}

	@Around("logPointCut()")
	public Object cut(ProceedingJoinPoint joinPoint) throws Throwable {
		MethodSignature signature = (MethodSignature) joinPoint.getSignature();
		Method method = signature.getMethod();

		logger.info("proceed:{}#{}", joinPoint.getTarget().getClass().getName(), method.getName());
		logger.info("执行判断方法");
		Object value = joinPoint.proceed();

		logger.info("return:{}", value);
		return "I'm aspect2," + value;
	}

}
```

### 3.3 业务类

```java
package com.fei.springboot.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.fei.springboot.annotation.Anno1;
import com.fei.springboot.annotation.Anno2;

@Service
public class AnnoService {

	private Logger logger = LoggerFactory.getLogger(AnnoService.class);

	@Anno1
	@Anno2
	public String say(String msg) {
		logger.info("Say:{}", msg);

		return "haiyan";
	}
}
```

### 3.4 测试类

```java
package test.pkgs;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.fei.springboot.RwApp;
import com.fei.springboot.service.AnnoService;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = RwApp.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@EnableAutoConfiguration
public class AnnoTest {
	@Autowired
	private AnnoService service;

	@Test
	public void test() {
		service.say("hello");
	}

}
```

### 3.5 测试结果

```java
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno1Aspect:30 - proceed:com.fei.springboot.service.AnnoService#say
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno1Aspect:31 - 执行判断方法
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno2Aspect:30 - proceed:com.fei.springboot.service.AnnoService#say
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno2Aspect:31 - 执行判断方法
2018-10-13 13:41:25 INFO  c.f.s.service.AnnoService:18 - Say:haiyan
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno2Aspect:34 - return:hello haiyan
2018-10-13 13:41:25 INFO  c.fei.springboot.aop.Anno1Aspect:34 - return:I'm aspect2,hello haiyan
```

执行流程: Anno1Aspect -> Anno2Aspect -> Service -> Anno2Aspect -> Anno1Aspect.

**Service 里面的方法只会被调用一次.相当于链式的调用**

---

## 4. Spring 事务问题

在 Spring 里面事务也是切面进行管理的.但是 aop 有一个比较坑爹的地方.

基于上面的代码,我们来模拟一下这个坑爹的场景.

### 4.1 业务类

```java
package com.fei.springboot.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.fei.springboot.annotation.Anno1;
import com.fei.springboot.annotation.Anno2;

@Service
public class FreakService {

	private Logger logger = LoggerFactory.getLogger(AnnoService.class);

	@Anno1
	public String anno1(String msg) {
		logger.info("anno1:{}", msg);

		return "anno1 " + msg;
	}

	@Anno2
	public String anno2(String msg) {
		logger.info("anno2:{}", msg);
		return "anno2 " + msg;
	}

	/**
	 * 注意这个方法
	 *
	 * @param msg
	 * @return
	 */
	@Anno2
	public String freak(String msg) {
		return anno1(msg);
	}

}
```

### 4.2 测试类

```java
package test.pkgs;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import com.fei.springboot.RwApp;
import com.fei.springboot.service.FreakService;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = RwApp.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@EnableAutoConfiguration
public class FreakTest {
	@Autowired
	private FreakService service;

	@Test
	public void testAnno() {
		service.anno1("haiyan");
		service.anno2("haiyan");
	}

	@Test
	public void testFreak() {
		service.freak("3306");
	}
}
```

### 4.3 测试结果

**测试 testAnno**

```java
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno1Aspect:30 - proceed:com.fei.springboot.service.FreakService#anno1
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno1Aspect:31 - 执行判断方法
2018-10-13 14:01:52 INFO  c.f.s.service.AnnoService:17 - anno1:haiyan
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno1Aspect:34 - return:anno1 haiyan
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno2Aspect:30 - proceed:com.fei.springboot.service.FreakService#anno2
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno2Aspect:31 - 执行判断方法
2018-10-13 14:01:52 INFO  c.f.s.service.AnnoService:24 - anno2:haiyan
2018-10-13 14:01:52 INFO  c.fei.springboot.aop.Anno2Aspect:34 - return:anno2 haiyan
```

**测试 testFreak**

```java
2018-10-13 13:55:23 INFO  c.fei.springboot.aop.Anno2Aspect:30 - proceed:com.fei.springboot.service.FreakService#freak
2018-10-13 13:55:23 INFO  c.fei.springboot.aop.Anno2Aspect:31 - 执行判断方法
2018-10-13 13:55:23 INFO  c.f.s.service.AnnoService:17 - anno1:3306
2018-10-13 13:55:23 INFO  c.fei.springboot.aop.Anno2Aspect:34 - return:anno1 3306
```

注意:**freak 调用 anno1 的时候,切面 Anno1Aspect 没有生效.**

产生原因:因为 freak 里面采用的是`this.anno1(msg)`方法,而不是被 spring aop 代理的代理类.

详细原因请参考该 blog:[spring aop 类内部调用不拦截原因及解决方案](https://blog.csdn.net/dream_broken/article/details/72911148)

### 4.4 解决方案

通过`ApplicationContext`重新获取代理的类来执行.

```java
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class SpringContextUtil implements ApplicationContextAware{
	private static ApplicationContext applicationContext = null;
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		if(SpringContextUtil.applicationContext == null){
			SpringContextUtil.applicationContext = applicationContext;
		}

	}
	public static ApplicationContext getApplicationContext() {
		return applicationContext;
	}
	public static Object getBean(String name){
		 ApplicationContext ctx = getApplicationContext();
		return ctx.getBean(name);
	}
	public static <T> T getBean(Class<T> clazz){
		return getApplicationContext().getBean(clazz);
	}
}
```

```java
package com.fei.springboot.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import com.fei.springboot.annotation.Anno1;
import com.fei.springboot.annotation.Anno2;
import com.fei.springboot.util.SpringContextUtil;

@Service
public class FreakService {

	private Logger logger = LoggerFactory.getLogger(AnnoService.class);

	@Anno1
	public String anno1(String msg) {
		logger.info("anno1:{}", msg);

		return "anno1 " + msg;
	}

	@Anno2
	public String anno2(String msg) {
		logger.info("anno2:{}", msg);
		return "anno2 " + msg;
	}

	/**
	 * 注意这个方法
	 *
	 * @param msg
	 * @return
	 */
	@Anno2
	public String freak(String msg) {
		return SpringContextUtil.getBean(this.getClass()).anno1(msg);
	}

}
```

重新测试,测试结果

```java
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno2Aspect:30 - proceed:com.fei.springboot.service.FreakService#freak
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno2Aspect:31 - 执行判断方法
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno1Aspect:30 - proceed:com.fei.springboot.service.FreakService#anno1
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno1Aspect:31 - 执行判断方法
2018-10-13 14:10:27 INFO  c.f.s.service.AnnoService:18 - anno1:3306
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno1Aspect:34 - return:anno1 3306
2018-10-13 14:10:27 INFO  c.fei.springboot.aop.Anno2Aspect:34 - return:anno1 3306
```

Spring AOP 丧尽天良.

---

## 5. Spring 定时器

Sometimes,我们要写相关的定时任务,要求定时 cron 表达式能够在配置文件文件配置.

配置文件时间表达式内容:

```yml
interval:
  invoke: 0 0/1 * * * *
```

定时任务代码

```java
import java.util.Date;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.Trigger;
import org.springframework.scheduling.TriggerContext;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.scheduling.support.CronTrigger;

/**
 * 定时任务执行器
 *
 *
 * <p>
 *
 * @author hhp 2018年10月17日
 * @see
 * @since 1.0
 */
@Configuration
@EnableScheduling
public class MySchedule implements SchedulingConfigurer {

	private static Logger logger = LoggerFactory.getLogger(MySchedule.class);

	/**
	 * application配置文件配置表达式
	 */
	@Value("${interval.invoke}")
	private String timeSegment = "";

	@Override
	public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
		taskRegistrar.setScheduler(taskExecutor());
		taskRegistrar.addTriggerTask(new Runnable() {
			@Override
			public void run() {
				// 执行方法调度具体业务代码
				logger.info("Start run");
			}
		}, new Trigger() {
			@Override
			public Date nextExecutionTime(TriggerContext triggerContext) {
				CronTrigger trigger = new CronTrigger(timeSegment);
				Date nextExecDate = trigger.nextExecutionTime(triggerContext);
				return nextExecDate;
			}
		});
	}

	@Bean(destroyMethod = "shutdown")
	public Executor taskExecutor() {
		return Executors.newScheduledThreadPool(10);
	}
}
```

---

## 6. Spring 配置默认值

在 Spring 里面可以使用`@Value`注解在变量上获取配置文件上面的值.

类似场景: 如果用户不在配置文件写这个配置,则使用系统默认参数.

```java
@Value("${async.port}")
private int asyncPort;
```

如果用户没配置就直接报错了,启动都启动不了.

那么使用默认值来避免这个异常.

```java
@Value("${async.port:12345}")
private int asyncPort;
```

---

## 7. 获取 ApplicationContext

Spring 与 ApplicationContext

在有些场景下面,我们需要获取到上下文`ApplicationContext`.

那么我们该怎么做呢?

### 7.1 实现`ApplicationContextAware`接口.

实现`ApplicationContextAware`接口,这里面有一个坑.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

/**
 *
 * ApplicationContext工具类
 *
 * <p>
 *
 * @author cs12110 2018年11月16日
 * @see
 * @since 1.0
 */
@Component("SpringAppCtx")
public class SpringAppCtx implements ApplicationContextAware {

	private static Logger logger = LoggerFactory.getLogger(SpringAppCtx.class);

	private static ApplicationContext ctx = null;

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {

		logger.info("Set context value: {}", applicationContext);
		SpringAppCtx.ctx = applicationContext;
	}

	public static ApplicationContext getApplicationContext() {
		return ctx;
	}

}
```

上面这种实现方法,有一个要命的地方是,就是`@PostConstruct`方法里面使用到的话,会得到的是`null`.

解决方法: 在调用类里面,添加注解`@DependsOn("SpringAppCtx")`.代码如下

```java

import javax.annotation.PostConstruct;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.DependsOn;
import org.springframework.scheduling.annotation.EnableScheduling;

import cn.rojao.task.SpringAppCtx;

@SpringBootApplication
@EnableScheduling
@DependsOn("SpringAppCtx")
public class App {
	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}

	/**
	 * 系统初始化方法
	 */
	@PostConstruct
	public void sysInit() {
		ApplicationContext ctx = SpringAppCtx.getApplicationContext();
		System.out.println();
		System.out.println(ctx);
		System.out.println();
	}
}
```

### 7.2 @Autowired 注入

最简单,最粗暴,我最喜欢了.

直接使用`@Autowired`注入.

```java
import javax.annotation.PostConstruct;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.DependsOn;
import org.springframework.scheduling.annotation.EnableScheduling;

import cn.rojao.task.SysInit;

@SpringBootApplication
@EnableScheduling
public class App {

	@Autowired
	private ApplicationContext ctx;

	public static void main(String[] args) {
		SpringApplication.run(App.class, args);
	}

	/**
	 * 系统初始化方法
	 */
	@PostConstruct
	public void sysInit() {
		System.out.println(ctx);
	}
}
```

---

## 8. SpringBoot 拦截器

### 8.1 基础概念

来自[过滤器(Filter)和拦截器(Interceptor)](https://www.cnblogs.com/protected/p/6649587.html)

Filter 和 Interceptor 的区别

- Filter 是基于函数回调(doFilter()方法)的,而 Interceptor 则是基于 Java 反射的(AOP 思想)

- Filter 依赖于 Servlet 容器,而 Interceptor 不依赖于 Servlet 容器.Filter 对几乎所有的请求起作用,而 Interceptor 只能对 action 请求起作用.

- Interceptor 可以访问 Action 的上下文,值栈里的对象,而 Filter 不能

- 在 action 的生命周期里,Interceptor 可以被多次调用,而 Filter 只能在容器初始化时调用一次.

Filter 和 Interceptor 的执行顺序

**过滤前-拦截前-action 执行-拦截后-过滤后**

### 8.2 代码实现

需要实现 Spring 里面的`org.springframework.web.servlet.HandlerInterceptor`接口,并注册.

**拦截器**

```java

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.servlet.HandlerInterceptor;

/**
 * 拦截器
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月6日
 * @see
 * @since 1.0
 */
public class JustWatchInterceptor implements HandlerInterceptor {

	private static Logger logger = LoggerFactory.getLogger(JustWatchInterceptor.class);

	/**
	 * 请求前拦截器
	 */
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		logger.info("interceptor:{}", request.getRequestURI());
		// 返回true才往下执行
		return true;
	}
}
```

**配置**

```java
import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

import cn.rojao.utils.interceptor.JustWatchInterceptor;
/**
 * MVC配置
 */
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new JustWatchInterceptor());
	}

	/**
	 * 添加自定义方法参数处理器,没有可省略
	 */
	@Override
	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
	}
}
```

**测试结果**

```java
2018-12-06 08:24:31 INFO  c.r.u.i.JustWatchInterceptor:30 - interceptor:/login.html
2018-12-06 08:24:31 INFO  c.r.u.i.JustWatchInterceptor:30 - interceptor:/sys/sysbasedata/getAllBaseData
2018-12-06 08:24:31 INFO  c.r.u.i.JustWatchInterceptor:30 - interceptor:/captcha.jpg
2018-12-06 08:24:31 INFO  c.r.u.i.JustWatchInterceptor:30 - interceptor:/public/images/icon.png
```
