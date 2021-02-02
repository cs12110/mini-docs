# Springboot 源码分析

这个文档不推荐各位大佬查看,因为会写得很乱. :"}

springboot 版本: `2.1.6.RELEASE`,一入 debug 深似海.

参考文档: [springboot 源码分析 link](https://blog.csdn.net/woshilijiuyi/article/details/82219585)

---

## 1. springboot 启动

### 1.1 构建环境

在一个简单的 springboot 项目启动大概类似如下:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;


@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```

那么我们可以从`run`开始,认清生活的艰难. 2nd & 11.

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  return
  // 构建springboot启动环境参数
  new SpringApplication(primarySources)
  // 运行
  .run(args);
}
```

构建 springboot 启动环境参数

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  // 判断运行环境是不是web环境,根据运行环境是否带有某些class判断
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  // 扫描各种jar里面的META-INF/spring.factories,获取spring factories
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  // 根据main方法判断主运行class
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 1.2 运行

```java
/**
  * Run the Spring application, creating and refreshing a new
  * {@link ApplicationContext}.
  * @param args the application arguments (usually passed from a Java main method)
  * @return a running {@link ApplicationContext}
  */
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  configureHeadlessProperty();
  // 1.加载和启动监听器,加载spring.factories里面配置的各种listener
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // 启动监听器
  listeners.starting();
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    // 2. 准备容器环境
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    configureIgnoreBeanInfo(environment);
    // 打印banner
    Banner printedBanner = printBanner(environment);
    // 3. 创建容器
    context = createApplicationContext();
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
        new Class[] { ConfigurableApplicationContext.class }, context);
    // 4. 准备容器
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    // 5. 刷新环境
    refreshContext(context);
    // 6. 刷新容器之后的扩展
    afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    listeners.started(context);
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

获取监听器:`org.springframework.boot.SpringApplication#getRunListeners`,往下追踪可以看到:`org.springframework.boot.SpringApplication#getSpringFactoriesInstances`

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
  ClassLoader classLoader = getClassLoader();
  // Use names and ensure unique to protect against duplicates
  // SpringFactoriesLoader.loadFactoryNames(type, classLoader): 加载MEAT-INF/spring.factories里面的配置
  Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
  // 创建工厂类
  List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
  AnnotationAwareOrderComparator.sort(instances);
  return instances;
}
```

springboot加载配置监听器:`org.springframework.boot.context.config.ConfigFileApplicationListener`
