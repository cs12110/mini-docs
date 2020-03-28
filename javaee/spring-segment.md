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

---

## 2. 切面计算执行耗时

在性能测试里面,我们需要找到某一个方法执行耗时多久的情况,这是 AOP 绝佳的一个使用场景.

举个栗子: `execution (* com.sample.service..*.*(..))`,整个表达式可以分为五个部分:

- execution(): 表达式主体
- 第一个*号：表示返回类型, *号表示所有的类型.
- 包名:表示需要拦截的包名, 后面的两个句点表示当前包和当前包的所有子包,com.sample.service 包,子孙包下所有类的方法.
- 第二个* 号：表示类名,*号表示所有的类.
- `*(..)`：最后这个星号表示方法名,\*号表示所有的方法,后面括弧里面表示方法的参数,两个句点表示任何参数.

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
 * @author cs12110 2017年9月18日
 * @since 1.0
 */
@Aspect
@Component
public class ServiceExecTimeCalculateInteceptor {
    private static Logger logger = LoggerFactory.getLogger(ServiceExecTimeCalculateInteceptor.class);

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

在 Spring 里面事务也是切面进行管理的.但是 aop 有一个比较坑爹的地方:`spring事务是基于接口或基于类的代理被创建(注意一定要是代理,不能手动new 一个对象,并且此类(有无接口都行)一定要被代理,spring中的bean只要纳入了IOC管理都是被代理的).在同一个类,一个方法调用另一个方法有事务的方法,事务是不会起作用的.`

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

### 5.1 自定义定时器

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

### 5.2 并发定时器

在使用`@Scheduled(cron="${schedule.video-cron:0/30 * * * * ?}")`的时候,发现一个问题: 默认情况下,只有一个定时线程在运行定时任务,也就是说同一时间只能运行一个方法,不能做到并行. [参考牛神博客 link](https://blog.csdn.net/cowbin2012/article/details/85219887)

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/13 9:45
 * <p>
 * since: 1.0.0
 */
@Component
@Slf4j
public class JustTestTask {

    @Scheduled(cron = "0/5 * * * * *")
    public void method1() {
        log.info("method1");
        try {
            Thread.sleep(2000 * 10);
        } catch (Exception e) {
            // do nothing
        }
    }

    @Scheduled(cron = "0/5 * * * * *")
    public void method2() {
        log.info("method2");
        try {
            Thread.sleep(2000 * 20);
        } catch (Exception e) {
            // do nothing
        }
    }
}
```

测试结果

```java
2019-02-13 10:11:55 INFO  c.r.t.JustTestTask:20 - method1
2019-02-13 10:12:15 INFO  c.r.t.JustTestTask:30 - method2
2019-02-13 10:12:55 INFO  c.r.t.JustTestTask:20 - method1
2019-02-13 10:13:15 INFO  c.r.t.JustTestTask:30 - method2
2019-02-13 10:13:55 INFO  c.r.t.JustTestTask:20 - method1
2019-02-13 10:14:15 INFO  c.r.t.JustTestTask:30 - method2
2019-02-13 10:14:55 INFO  c.r.t.JustTestTask:20 - method1
```

Q: 那么定时任务该怎么进行并发呢?

A: 请参考如下代码.

```java
import cn.rojao.util.ThreadUtil;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.ThreadFactory;

/**
 * 为了应对定时器的并发处理配置
 * <p>
 * <p/>
 *
 * @author cs12110 created at: 2019/2/13 8:57
 * <p>
 * since: 1.0.0
 */
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {

    /**
     * 定时任务数量
     */
    private int taskNum = 2;

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(setTaskExecutors());
    }

    @Bean
    public ExecutorService setTaskExecutors() {
        ThreadFactory factory = ThreadUtil.buildThreadFactory("schedule-pool");
        return new ScheduledThreadPoolExecutor(taskNum, factory);
    }
}
```

测试结果

```java
2019-02-13 10:17:00 INFO  c.r.t.JustTestTask:30 - method2
2019-02-13 10:17:00 INFO  c.r.t.JustTestTask:20 - method1
2019-02-13 10:17:25 INFO  c.r.t.JustTestTask:20 - method1
2019-02-13 10:17:45 INFO  c.r.t.JustTestTask:30 - method2
2019-02-13 10:17:50 INFO  c.r.t.JustTestTask:20 - method1
```

---

## 6. Spring 配置默认值

### 6.1 @Value 的使用

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

### 6.2 配置文件转换成 bean

Q: 如果可以转化成一个对象,而不是全都使用@Value 来注入属性值,该怎么办?

A: 可以使用`@Component`+`@ConfigurationProperties`来实现.如果对象使用`@Configuration` 注解,在转换成 `json` 字符串的时候会出现异常,所以采用`@Component` 注解.

```yml
# my entity
my:
  name: cs12110
  age: 36
  gender: male
```

```java
package com.pkgs.entity.rookie;

import com.alibaba.fastjson.JSON;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/21 16:41
 * <p>
 * since: 1.0.0
 */
@Data
@Configuration
@ConfigurationProperties(prefix = "my")
public class MyEntity {
    private String name;
    private Integer age;
    private String gender;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

测试代码

```java
@Autowired
private MyEntity entity;

@PostConstruct
public void init() {
	log.info(entity.toString());
}
```

测试结果

```java
2019-03-21 16:50:18 INFO  com.pkgs.App:43 - {"age":36,"gender":"male","name":"cs12110"}
```

Q: wait.要是我的配置并不在 application.yml 里面怎么办?

A: 可以使用`@ConfigurationProperties`+`@PropertySource`指定配置文件.

比如`bank.properties`配置位于`resources/config/`文件夹里面.

```java
@Data
@PropertySource("classpath:config/bank.properties")
@ConfigurationProperties(prefix = "bank")
@Component
public class MyBankProperties {

    private String name;
    private String address;
    private Integer level;


    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

配置文件内容如下

```properties
bank.name=my bank
bank.address=my pocket
bnak.level=5
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

---

## 9. Springboot 过滤器

是时候看看 spring 里面里面怎么弄过滤器了. :"}

记住: `order`的值越低,优先级越高.

### 9.1 注解模式

实现方式: 实现`Filter`接口,并添加`@Component`注解即可.

```java
package com.pkgs.conf.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import java.io.IOException;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/19 8:22
 * <p>
 * since: 1.0.0
 */
@Slf4j
@Component
@Order(10)
public class FirstFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("First filter");

        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

```java
package com.pkgs.conf.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import java.io.IOException;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/19 8:22
 * <p>
 * since: 1.0.0
 */
@Slf4j
@Component
@Order(30)
public class SecondFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("Second filter");

        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

测试结果

```java
2019-03-19 08:38:05 INFO  c.p.c.f.FirstFilter:28 - First filter
2019-03-19 08:38:05 INFO  c.p.c.f.SecondFilter:28 - Second filter
```

### 9.2 bean 方式

Q: 那么问题来了,有没有更复杂一点的实现方法呀?

A: 满足你!!!

去除过滤器上面的注解,使用 bean 的方式实现,如下

```java
package com.pkgs.conf.filter;

import lombok.extern.slf4j.Slf4j;

import javax.servlet.*;
import java.io.IOException;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/19 8:22
 * <p>
 * since: 1.0.0
 */
@Slf4j
public class FirstFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("First filter");

        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

```java
package com.pkgs.conf.filter;

import lombok.extern.slf4j.Slf4j;

import javax.servlet.*;
import java.io.IOException;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/19 8:22
 * <p>
 * since: 1.0.0
 */
@Slf4j
public class SecondFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("Second filter");

        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }
}
```

```java
package com.pkgs.conf.web;

import com.pkgs.conf.filter.FirstFilter;
import com.pkgs.conf.filter.SecondFilter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.servlet.Filter;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/3/19 8:23
 * <p>
 * since: 1.0.0
 */
@Configuration
public class FilterConf {

    @Bean
    public FilterRegistrationBean firstFilter() {
        FilterRegistrationBean<Filter> registrationBean = new FilterRegistrationBean<>();

        // order: 值越小,优先级越高
        registrationBean.setFilter(new FirstFilter());
        registrationBean.setOrder(10);

        return registrationBean;
    }


    @Bean
    public FilterRegistrationBean secondFilter() {
        FilterRegistrationBean<Filter> registrationBean = new FilterRegistrationBean<>();

        // order: 值越小,优先级越高
        registrationBean.setFilter(new SecondFilter());
        registrationBean.setOrder(20);

        return registrationBean;
    }
}
```

测试结果

```java
2019-03-19 08:44:33 INFO  c.p.c.f.FirstFilter:24 - First filter
2019-03-19 08:44:33 INFO  c.p.c.f.SecondFilter:24 - Second filter
```

---

## 10. Springboot 防止重复提交

借助自定义注解和拦截器防止重复提交.

### 10.1 自定义注解

```java
import java.lang.annotation.*;

/**
 * 防止重复提交
 * <p/>
 *
 * @author cs12110 created at: 2018/12/28 13:23
 * <p>
 * since: 1.0.0
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AntiResubmit {

    /**
     * 描述
     *
     * @return String
     */
    String desc() default "";
}
```

### 10.2 切面

```java
import com.alibaba.fastjson.JSON;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import java.util.HashMap;
import java.util.Map;

/**
 * 防止重复提交切面
 * <p/>
 *
 * @author cs12110 created at: 2018/12/28 13:25
 * <p>
 * since: 1.0.0
 */
@Component
@Aspect
public class AntiResubmitAspect {

    private static Logger logger = LoggerFactory.getLogger(AntiResubmitAspect.class);

    /**
	 * @annotation为AntiResubmit注解完整位置.
	 */
    @Pointcut("@annotation(cn.pkgs.utils.anno.AntiResubmit)")
    public void execute() {

    }

    @Around("execute()")
    public Object around(ProceedingJoinPoint point) {
        try {
            ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder
                    .getRequestAttributes();
            HttpServletRequest request = attributes.getRequest();
            HttpSession session = request.getSession(false);

            //key,可以存放在redis,做分布式判断
            String key = session.getId() + "#" + request.getServletPath();
            Object token = session.getAttribute(key);

            if (token == null) {
                session.setAttribute(key, 1);
                //执行方法
                Object value = point.proceed();
                session.removeAttribute(key);
                return value;
            } else {
                logger.error("repeat:{}", key);
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }

        //重复提交
        Map<String, Object> map = new HashMap<>(2);
        map.put("status", 403);
        map.put("msg", "Repeat submit");
        return JSON.toJSONString(map);
    }
}
```

---

## 11. Spring Event

在 spring 里面也有事件通知时间.可以做到发布/订阅的功能.

Warning: **`事件都是同步的,如果发布事件处的业务存在事务,监听器处理也会在相同的事务中.如果对于事件的处理不想受到影响,可以在app运行类上开启异步处理@EnableAsync,listener的onApplicationEvent方法上加@Aync支持异步.`**

### 11.1 自定义事件

```java
import org.springframework.context.ApplicationEvent;

/**
 * 自定义事件
 *
 * @author cs12110 create at 2019/5/9 20:31
 * @version 1.0.0
 */
public class SysEvent extends ApplicationEvent {
    /**
     * 事件
     *
     * @param source 可以传入事件特定的值
     */
    public SysEvent(Object source) {
        super(source);
    }
}
```

### 11.2 事件处理器

```java
import com.alibaba.fastjson.JSON;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

/**
 * 监听SysEvent事件
 *
 * @author cs12110 create at 2019/5/9 20:32
 * @version 1.0.0
 */
@Component
public class SysEventListener implements ApplicationListener<SysEvent> {

    private static Logger logger = LoggerFactory.getLogger(SysEventListener.class);

    @Override
    public void onApplicationEvent(SysEvent sysEvent) {
        try {
            logger.info("{}", JSON.toJSONString(sysEvent.getSource()));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### 11.3 事件发生器

```java
 @Autowired
private ApplicationContext applicationContext;

@PostConstruct
public void init() {
	log.info(entity.toString());
	applicationContext.publishEvent(new SysEvent("1"));
	applicationContext.publishEvent(new SysEvent("2"));
	applicationContext.publishEvent(new SysEvent("3"));

	try {
		Thread.sleep(5000);
	} catch (Exception e) {
		e.printStackTrace();
	}
	applicationContext.publishEvent(new SysEvent("4"));
}
```

测试结果

```java
2019-05-09 20:38:11 INFO  c.p.l.SysEventListener:21 - "1"
2019-05-09 20:38:11 INFO  c.p.l.SysEventListener:21 - "2"
2019-05-09 20:38:11 INFO  c.p.l.SysEventListener:21 - "3"
2019-05-09 20:38:16 INFO  c.p.l.SysEventListener:21 - "4"
```

---

## 12. 文件上传/下载

![postman上传文件](https://blog.csdn.net/maowendi/article/details/80537304)

设置文件上传 size:

```yml
spring:
  thymeleaf:
    mode: HTML5
    encoding: UTF-8
    prefix: classpath:/template
  servlet:
    multipart:
      # Single file max size  即单个文件大小
      max-file-size: 50Mb
      # All files max size    即总上传的数据大小
      max-request-size: 50Mb
```

### 12.1 文件上传

**controller**

```java
/**
    * 上传文件
    *
    * @param multipartFile file
    * @return String
    */
@RequestMapping("/uploadFile")
@ResponseBody
public String uploadFile(@RequestParam("file") MultipartFile multipartFile) {
    Map<String, Object> map = magixService.uploadFile(multipartFile);

    return JSON.toJSONString(map);
}
```

**service**

```java
private static final String FILE_STORAGE_PATH = "/opt/";

public Map<String, Object> uploadFile(MultipartFile multipartFile) {
Map<String, Object> map = new HashMap<>(2);
map.put("status", 0);
if (multipartFile == null) {
    log.warn("Function[uploadFile] file is null");
    map.put("message", "file is null");
} else {
    try {

        String filename = multipartFile.getOriginalFilename();
        byte[] values = multipartFile.getBytes();

        String filePath = FILE_STORAGE_PATH + buildFileName(filename);
        FileOutputStream out = new FileOutputStream(filePath);
        out.write(values);
        out.flush();
        out.close();

        map.put("url",filePath);
        map.put("status", 1);
    } catch (Exception e) {
        map.put("status", 0);
        map.put("message", e.getMessage());
        log.error("Function[uploadFile]", e);
    }
}
log.info("Function[uploadFile] result:{}", JSON.toJSONString(map));

return map;
}

private String buildFileName(String fileName) {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmmss");
    String format = sdf.format(new Date());
    if (null == fileName) {
        return format;
    }

    int last = fileName.lastIndexOf(".");
    if (-1 == last) {
        return fileName + format;
    }
    String suffix = fileName.substring(last);
    String name = fileName.substring(0, last);

    return name + format + suffix;
}
```

### 12.2 文件下载

**controller**

```java
/**
* 下载文件
*
* @param request  request
* @param response response
*/
@RequestMapping("/downloadFile")
public void downloadFile(HttpServletRequest request, HttpServletResponse response) {
    String fileName = request.getParameter("fileName");

    log.info("Function[downloadFile] download file:{}", fileName);

    magixService.downloadFile(response, fileName);
}
```

**service**

```java
public void downloadFile(HttpServletResponse response, String fileName) {
    try {
        File file = new File(fileName);
        if (!file.exists()) {
            log.warn("Function[download] File:{} is null", fileName);
            return;
        }

        // 设置头部文件和文件路名中文
        response.setHeader("content-type", "application/octet-stream");
        response.setContentType("application/octet-stream");
        response.setHeader("Content-Disposition", "attachment;filename=" + java.net.URLEncoder.encode(fileName, "UTF-8"));


        ServletOutputStream outputStream = response.getOutputStream();
        byte[] arr = new byte[1024];
        int len = 0;

        FileInputStream inputStream = new FileInputStream(file);
        while (-1 != (len = inputStream.read(arr))) {
            outputStream.write(arr, 0, len);
        }

        outputStream.flush();
        outputStream.close();
    } catch (Exception e) {
        log.error("Function[downloadFile],fileName:" + fileName, e);
    }
}
```

---

## 13. 全局异常

全局异常也挺重要的,心累.

### 13.1 定义业务异常

```java
import com.alibaba.fastjson.JSON;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.EqualsAndHashCode;
import lombok.Getter;

/**
 * @author cs12110 create at 2020/3/17 13:59
 * @version 1.0.0
 */
@Data
@EqualsAndHashCode(callSuper = false)
public class BizException extends RuntimeException {

    @AllArgsConstructor
    @Getter
    enum StatusEnum {

        /**
         * 操作成功:1
         */
        SUCCESS(1, "成功"),
        /**
         * 操作失败:0
         */
        FAILURE(0, "失败");

        private final int value;
        private final String desc;
    }

    private Integer status;
    private String message;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

### 13.2 全局异常处理

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * @author cs12110 create at 2020/3/17 14:01
 * @version 1.0.0
 */
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 全局异常
     *
     * @param e e
     */
    @ResponseBody
    @ExceptionHandler(value = Exception.class)
    public BizException handleWithGlobalException(Exception e) {
        log.error("handleWithGlobalException", e);

        BizException exception = new BizException();
        exception.setMessage(e.getMessage());
        exception.setStatus(BizException.StatusEnum.FAILURE.getValue());

        return exception;
    }

}
```

---

## 14. 跨域问题

```java
import org.springframework.stereotype.Component;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * TODO: 跨域访问拦截器
 *
 * @author cs12110 create at: 2019/3/17 14:11
 * Since: 1.0.0
 */
@Component
public class CrossOriginFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        // 设置允许跨域访问
        response.setHeader("Access-Control-Allow-Origin", request.getHeader("Origin"));
        response.setHeader("Access-Control-Allow-Methods", "POST, GET,PUT, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers", "x-requested-with");

        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```

---

## 15. Bean 复制

哇咔咔,经常遇到要`把对象属性复制给另一个对象`的场景,如果一个一个的 `getter/setter` 进去,简直折磨人.

那么 Spring 的 BeanUtils,值得拥有.

### 15.1 定义类

```java
@Data
static class Candy {
    private String name;
    private String color;
    private String flavour;
    private BigDecimal price;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}

@Data
static class MyBox {
    private String id;
    private String name;
    private List<Candy> candyList;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}

@Data
static class YourBox {
    private String id;
    private String name;
    private List<Candy> candyList;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

### 15.2 BeanUtil

```java
public static void main(String[] args) {
    MyBox box = createMyBox();
    YourBox yourBox = new YourBox();

    // 这里面必须为对象,但是有些场景yourBox可能只是类怎么玩?
    BeanUtils.copyProperties(box, yourBox);

    System.out.println(yourBox);
}


/**
    * 构建MyBox对象
    *
    * @return MyBox
    */
private static MyBox createMyBox() {
    Candy strawberry = new Candy();
    strawberry.setColor("red");
    strawberry.setFlavour("sour");
    strawberry.setName("memory");
    strawberry.setPrice(BigDecimal.valueOf(30.00));

    List<Candy> candies = new ArrayList<>(2);
    candies.add(strawberry);


    MyBox box = new MyBox();
    box.setId("B" + System.currentTimeMillis());
    box.setName("Candy-" + System.currentTimeMillis());
    box.setCandyList(candies);


    return box;
}
```

测试结果

```json
{
  "candyList": [
    { "color": "red", "flavour": "sour", "name": "memory", "price": 30.0 }
  ],
  "id": "B1585186223771",
  "name": "Candy-1585186223771"
}
```

小小抽取

```java
/**
* 复制对象
*
* @param from 来源对象
* @param to   目标对象class
* @param <V>  目标对象类型
* @return V
*/
public static <V> V copy(Object from, Class<V> to) {
    V v = null;
    try {
        // 反射一下
        v = to.newInstance();
        BeanUtils.copyProperties(from, v);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return v;
}
```
