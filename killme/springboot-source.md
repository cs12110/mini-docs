# Springboot 源码分析

这个文档不推荐各位大佬查看,因为会写得很乱. :"}

springboot 版本: `2.1.6.RELEASE`,一入 debug 深似海.

参考文档: [springboot 源码分析 link](https://blog.csdn.net/woshilijiuyi/article/details/82219585)

---

## 1. springboot 启动

确定项目运行环境 -> 加载 spring.factories 里面各种监听器配置 -> 启动监听器 ->

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
  // 根据type=ApplicationContextInitializer.class.getName()来匹配获取spring.factories里面的配置
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  // 根据type=ApplicationListener.class.getName()来匹配获取spring.factories里面的配置
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  // 根据main方法判断主运行class
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

```properties
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
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
  // 1.加载和启动监听器,加载spring.factories里面配置的listener
  // org.springframework.boot.SpringApplicationRunListener= org.springframework.boot.context.event.EventPublishingRunListener
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // 启动监听器: EventPublishingRunListener.starting()
  // this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
  // 广播: ApplicationStartingEvent 事件
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

准备上下文: `org.springframework.boot.SpringApplication#prepareContext`里面的`org.springframework.boot.SpringApplication#load`

```java
/**
  * Load beans into the application context.
  * @param context the context to load beans into
  * @param sources the sources to load
  */
protected void load(ApplicationContext context, Object[] sources) {
  if (logger.isDebugEnabled()) {
    logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
  }
  // 构建BeanDefinitionLoader有很重要的东西
  BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
  if (this.beanNameGenerator != null) {
    loader.setBeanNameGenerator(this.beanNameGenerator);
  }
  if (this.resourceLoader != null) {
    loader.setResourceLoader(this.resourceLoader);
  }
  if (this.environment != null) {
    loader.setEnvironment(this.environment);
  }
  loader.load();
}
```

```java
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
  Assert.notNull(registry, "Registry must not be null");
  Assert.notEmpty(sources, "Sources must not be empty");
  this.sources = sources;
  // 注解形式的Bean定义读取器 比如:@Configuration,@Bean,@Component,@Controller,@Service
  this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
  // XML形式的Bean定义读取器
  this.xmlReader = new XmlBeanDefinitionReader(registry);
  if (isGroovyPresent()) {
    this.groovyReader = new GroovyBeanDefinitionReader(registry);
  }
  // 类路径扫描器
  this.scanner = new ClassPathBeanDefinitionScanner(registry);
  // 扫描器添加排除过滤器
  this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```

准备环境

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
  // Create and configure the environment
  ConfigurableEnvironment environment = getOrCreateEnvironment();
  configureEnvironment(environment, applicationArguments.getSourceArgs());
  // 发布事件,这次事件被监听器: ConfigFileApplicationListener 监听
  listeners.environmentPrepared(environment);
  bindToSpringApplication(environment);
  if (!this.isCustomEnvironment) {
    environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
        deduceEnvironmentClass());
  }
  ConfigurationPropertySources.attach(environment);
  return environment;
}
```

springboot 加载配置监听器:`org.springframework.boot.context.config.ConfigFileApplicationListener`

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
  // 获取spring.factories配置文件里面的: EnvironmentPostProcessor
  List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
  // 最后调用本身的: postProcessEnvironment 方法
  postProcessors.add(this);
  AnnotationAwareOrderComparator.sort(postProcessors);
  for (EnvironmentPostProcessor postProcessor : postProcessors) {
    postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
  }
}
```

```properties
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor
```

spring 加载核心: `org.springframework.context.support.AbstractApplicationContext#refresh`,[参考 link](https://blog.csdn.net/qq_34203492/article/details/83865450)

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然 refresh() 还没结束，你又来个启动或销毁容器的操作，那不就乱套了嘛
   synchronized (this.startupShutdownMonitor) {

      // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();

      // 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      // 这块待会会展开说
      prepareBeanFactory(beanFactory);

      try {
         // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】

         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 回调方法
         invokeBeanFactoryPostProcessors(beanFactory);



         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。这里仅仅是注册，之后会看到回调这两方法的时机
         registerBeanPostProcessors(beanFactory);

         // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
         initMessageSource();

         // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
         initApplicationEventMulticaster();

         // 从方法名就可以知道，典型的模板方法(钩子方法)，不展开说
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         onRefresh();

         // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
         registerListeners();

         // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);

         // 最后，广播事件，ApplicationContext 初始化完成，不展开
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // 把异常往外抛
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

从上面的:`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`

```java
/**
  * Tell the subclass to refresh the internal bean factory.
  * @return the fresh BeanFactory instance
  * @see #refreshBeanFactory()
  * @see #getBeanFactory()
  */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  refreshBeanFactory();
  return getBeanFactory();
}
```

初始化 bean:`org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory`

```java
/**
  * This implementation performs an actual refresh of this context's underlying
  * bean factory, shutting down the previous bean factory (if any) and
  * initializing a fresh bean factory for the next phase of the context's lifecycle.
  */
@Override
protected final void refreshBeanFactory() throws BeansException {
  // 摧毁已经存在的beanFactory
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  try {
    // 初始化一个DefaultListableBeanFactory
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    // 这里面最重要,把bean加载到ioc
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}
```

注解解释注入 ioc:`org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization` [参考 link](https://www.cnblogs.com/hello-shf/p/11060546.html)

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  // Initialize conversion service for this context.
  if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
      beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
    beanFactory.setConversionService(
        beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
  }

  // Register a default embedded value resolver if no bean post-processor
  // (such as a PropertyPlaceholderConfigurer bean) registered any before:
  // at this point, primarily for resolution in annotation attribute values.
  if (!beanFactory.hasEmbeddedValueResolver()) {
    beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
  }

  // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
  String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
  for (String weaverAwareName : weaverAwareNames) {
    getBean(weaverAwareName);
  }

  // Stop using the temporary ClassLoader for type matching.
  beanFactory.setTempClassLoader(null);

  // Allow for caching all bean definition metadata, not expecting further changes.
  beanFactory.freezeConfiguration();

  // Instantiate all remaining (non-lazy-init) singletons.
  // 这里是注入ioc最重要的地方
  beanFactory.preInstantiateSingletons();
}
```

```java
@Override
public void preInstantiateSingletons() throws BeansException {
  if (logger.isTraceEnabled()) {
    logger.trace("Pre-instantiating singletons in " + this);
  }

  // Iterate over a copy to allow for init methods which in turn register new bean definitions.
  // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
  // Q: 这里面的this.beanDefinitionNames是什么时候赋值的???
  // A: org.springframework.boot.SpringApplication#prepareContext的org.springframework.boot.SpringApplication#load加载
  List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

  // Trigger initialization of all non-lazy singleton beans...
  for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      if (isFactoryBean(beanName)) {
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
        if (bean instanceof FactoryBean) {
          final FactoryBean<?> factory = (FactoryBean<?>) bean;
          boolean isEagerInit;
          if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                    ((SmartFactoryBean<?>) factory)::isEagerInit,
                getAccessControlContext());
          }
          else {
            isEagerInit = (factory instanceof SmartFactoryBean &&
                ((SmartFactoryBean<?>) factory).isEagerInit());
          }
          if (isEagerInit) {
            getBean(beanName);
          }
        }
      }
      else {
        // 执行这里
        getBean(beanName);
      }
    }
  }

  // Trigger post-initialization callback for all applicable beans...
  for (String beanName : beanNames) {
    Object singletonInstance = getSingleton(beanName);
    if (singletonInstance instanceof SmartInitializingSingleton) {
      final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
      if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          smartSingleton.afterSingletonsInstantiated();
          return null;
        }, getAccessControlContext());
      }
      else {
        smartSingleton.afterSingletonsInstantiated();
      }
    }
  }
}
```

`org.springframework.context.annotation.AnnotationBeanNameGenerator#buildDefaultBeanName(org.springframework.beans.factory.config.BeanDefinition)`

```java
protected String buildDefaultBeanName(BeanDefinition definition) {
  String beanClassName = definition.getBeanClassName();
  Assert.state(beanClassName != null, "No bean class name set");
  // 获取calssName,如: a.bc.Xyz获取 Xyz
  String shortClassName = ClassUtils.getShortName(beanClassName);
  // 将首字母转换成小写
  return Introspector.decapitalize(shortClassName);
}
```

```java
/**
  * Get the class name without the qualified package name.
  * @param className the className to get the short name for
  * @return the class name of the class without the package name
  * @throws IllegalArgumentException if the className is empty
  */
public static String getShortName(String className) {
  Assert.hasLength(className, "Class name must not be empty");
  int lastDotIndex = className.lastIndexOf(PACKAGE_SEPARATOR);
  int nameEndIndex = className.indexOf(CGLIB_CLASS_SEPARATOR);
  if (nameEndIndex == -1) {
    nameEndIndex = className.length();
  }
  String shortName = className.substring(lastDotIndex + 1, nameEndIndex);
  shortName = shortName.replace(INNER_CLASS_SEPARATOR, PACKAGE_SEPARATOR);
  return shortName;
}
```

```java
/**
  * Utility method to take a string and convert it to normal Java variable
  * name capitalization.  This normally means converting the first
  * character from upper case to lower case, but in the (unusual) special
  * case when there is more than one character and both the first and
  * second characters are upper case, we leave it alone.
  * <p>
  * Thus "FooBah" becomes "fooBah" and "X" becomes "x", but "URL" stays
  * as "URL".
  *
  * @param  name The string to be decapitalized.
  * @return  The decapitalized version of the string.
  */
public static String decapitalize(String name) {
    if (name == null || name.length() == 0) {
        return name;
    }

    // 如果XYz则直接返回XYz
    if (name.length() > 1 && Character.isUpperCase(name.charAt(1)) &&
                    Character.isUpperCase(name.charAt(0))){
        return name;
    }
    // 将首字母小写
    char chars[] = name.toCharArray();
    chars[0] = Character.toLowerCase(chars[0]);
    return new String(chars);
}
```
