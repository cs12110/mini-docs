# Mybatis 源码阅读

为了解决温饱,我干了这杯砒霜. Cheer up ,boys.

---

## 1. Configuration

在 mybatis 里面所有的配置和执行都会通过这个类.

```java
// 负责处理SQL查询参数
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
}

// 负责处理返回结果
public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler, ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
}

// 负责执行SQL
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

这三个里面都有一个`interceptorChain.pluginAll()`,这个是 Mybatis 的拦截器.

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<Interceptor>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }
}
```

这里面调用到接口`Interceptor`

```java
/**
 * @author Clinton Begin
 */
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  Object plugin(Object target);

  void setProperties(Properties properties);

}
```

下面看一个拦截器的例子.

FYI: 拦截器在创建 Configuration 对象的时候,就实例化了.

```java
package org.pkgs.interceptors;

import java.sql.Statement;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 运行程序里面,先执行 `plugin`方法,然后执行`interceptor`方法
 *
 *
 *
 * <p>
 *
 * @author huanghuapeng 2018年4月18日上午11:17:09
 *
 */
@Intercepts({ @Signature(type = ResultSetHandler.class, method = "handleResultSets", args = { Statement.class }) })
public class MyPlugin implements Interceptor {

  private static Logger logger = LoggerFactory.getLogger(MyPlugin.class);

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    logger.info("This is interceptor" + (valueOfInvocation(invocation)));
    return invocation.proceed();
  }

  private Map<String, Object> valueOfInvocation(Invocation in) {
    Map<String, Object> map = new HashMap<String, Object>();
    map.put("target", in.getTarget());
    map.put("class", in.getClass());
    map.put("args", in.getArgs());
    map.put("method", in.getMethod());
    return map;
  }

  @Override
  public Object plugin(Object target) {
    logger.info("This is plugin");
    return Plugin.wrap(target, this);
  }

  @Override
  public void setProperties(Properties properties) {
  }

}
```

```java
package org.apache.ibatis.plugin;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

import org.apache.ibatis.reflection.ExceptionUtil;

/**
 * @author Clinton Begin
 */
public class Plugin implements InvocationHandler {

  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }

}
```

---

## 2. 选择数据流程

### 2.1 选择数据流程

从 MapperProxy 根据执行方法名称获取 MapperStatement -> 使用 Executor.query(ms,...)执行 -> CacheingExecutor 代理的 SimpleExecutor -> 如果一级缓存里面有数据,则从缓存里面获取 -> 没有则从执行 SimpleExecutor 里面的 doQuery 方法 -> doQuery()调用 Configuration.newStatementHandler 组装 StatementHandler(该方法判断判断是否存在复合条件的拦截器,如果有的话则对该方法进行动态代理,在执行的时候因为进行业务增强) -> StatementHandler 执行查询 -> 返回数据.

### 2.2 MapperProxy 的构建

mybatis 里面的 Session 是通过 SqlSessionFactory 构建.

```java
InputStream stream = Resources.getResourceAsStream(configXmlPath);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(stream);
```

内部最终调用的 build 方法如下:

```java
 public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```

`XMLConfigBuilder#parse`方法里面调用了下面这个方法来构建整一个环境

```java
private void parseConfiguration(XNode root) {
  try {
    //issue #117 read properties first
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfs(settings);
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginElement(root.evalNode("plugins"));
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlerElement(root.evalNode("typeHandlers"));
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

在`mapperElement(root.evalNode("mappers"));`调用如下方法

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
  }

  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
```

`bindMapperForNamespace()`负责将 xml 内容解释,获取到 class,并将该 class 存为 Configuration.mapperRegistry.

```java
public <T> void addMapper(Class<T> type) {
  mapperRegistry.addMapper(type);
}
```

mapperReistry 里面

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      // It's important that the type is added before the parser is run
      // otherwise the binding may automatically be attempted by the
      // mapper parser. If the type is already known, it won't try.
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    } finally {
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

上面就是构建的流程了.

那么看看获取的流程:`Configuration#getMapper`

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

快来人呀,快来人呀,你看 ta 们终于 newInstance 了.

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

---

## 3. Mapper 的构建

Q: 像下面这个代码是怎么和 mybatis 产生关联的呀?

```java
SqlSession session = sqlSessionFactory.openSession();
NoteMapper mapper = session.getMapper(NoteMapper.class);
NoteBean note = mapper.selectNote(1);
```

A: 从`session.getMapper(NoteMapper.class);`开始,一步,一步,一步,一步,一步往下走.

```java
// org.apache.ibatis.session.Configuration#mapperRegistry
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}

@SuppressWarnings("unchecked")
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    // 使用Proxy来生成代理类
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

```java
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    // 使用Proxy.newProxyInstance生成代理类
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    // 构建MapperProxy
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

}
```

```java
package org.apache.ibatis.binding;

import java.io.Serializable;
import java.lang.invoke.MethodHandles;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Map;

import org.apache.ibatis.lang.UsesJava7;
import org.apache.ibatis.reflection.ExceptionUtil;
import org.apache.ibatis.session.SqlSession;

/**
 * @author Clinton Begin
 * @author Eduardo Macarron
 */
 // 实现java里面的InvocationHandler接口
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 放入缓存
    // mapperMethod.execute()获取mybatis的id,获取对应M
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

  @UsesJava7
  private Object invokeDefaultMethod(Object proxy, Method method, Object[] args)
      throws Throwable {
    final Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class
        .getDeclaredConstructor(Class.class, int.class);
    if (!constructor.isAccessible()) {
      constructor.setAccessible(true);
    }
    final Class<?> declaringClass = method.getDeclaringClass();
    return constructor
        .newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bindTo(proxy).invokeWithArguments(args);
  }

  /**
   * Backport of java.lang.reflect.Method#isDefault()
   */
  private boolean isDefaultMethod(Method method) {
    return ((method.getModifiers()
        & (Modifier.ABSTRACT | Modifier.PUBLIC | Modifier.STATIC)) == Modifier.PUBLIC)
        && method.getDeclaringClass().isInterface();
  }
}
```

```java
// mybatis构建接口里面方法对应的key,然后在执行方法的时候根据这个key获取绑定的方法
private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
        Class<?> declaringClass, Configuration configuration) {
  String statementId = mapperInterface.getName() + "." + methodName;
  if (configuration.hasStatement(statementId)) {
    return configuration.getMappedStatement(statementId);
  } else if (mapperInterface.equals(declaringClass)) {
    return null;
  }
  for (Class<?> superInterface : mapperInterface.getInterfaces()) {
    if (declaringClass.isAssignableFrom(superInterface)) {
      MappedStatement ms = resolveMappedStatement(superInterface, methodName,
          declaringClass, configuration);
      if (ms != null) {
        return ms;
      }
    }
  }
  return null;
}
```

---

## 4. LambdaQueryWrapper

一直很好奇`LambdaQueryWrapper.eq()`这种实现,觉得挺神奇的.

主要代码上面的注释: `支持序列化的Function`.

```java
package com.baomidou.mybatisplus.core.toolkit.support;

import java.io.Serializable;
import java.util.function.Function;

/**
 * 支持序列化的 Function
 *
 * @author miemie
 * @since 2018-05-12
 */
@FunctionalInterface
public interface SFunction<T, R> extends Function<T, R>, Serializable {
}
```

重要类:`com.baomidou.mybatisplus.core.conditions.AbstractLambdaWrapper`

```java
protected String columnToString(SFunction<T, ?> column, boolean onlyColumn) {
    return getColumn(LambdaUtils.resolve(column), onlyColumn);
}

/**
  * 获取 SerializedLambda 对应的列信息，从 lambda 表达式中推测实体类
  * <p>
  * 如果获取不到列信息，那么本次条件组装将会失败
  *
  * @param lambda     lambda 表达式
  * @param onlyColumn 如果是，结果: "name", 如果否： "name" as "name"
  * @return 列
  * @throws com.baomidou.mybatisplus.core.exceptions.MybatisPlusException 获取不到列信息时抛出异常
  * @see SerializedLambda#getImplClass()
  * @see SerializedLambda#getImplMethodName()
  */
private String getColumn(SerializedLambda lambda, boolean onlyColumn) throws MybatisPlusException {
    String fieldName = PropertyNamer.methodToProperty(lambda.getImplMethodName());
    Class aClass = lambda.getInstantiatedMethodType();
    if (!initColumnMap) {
        columnMap = LambdaUtils.getColumnMap(aClass);
    }
    Assert.notNull(columnMap, "can not find lambda cache for this entity [%s]", aClass.getName());
    ColumnCache columnCache = columnMap.get(LambdaUtils.formatKey(fieldName));
    Assert.notNull(columnCache, "can not find lambda cache for this property [%s] of entity [%s]",
        fieldName, aClass.getName());
    return onlyColumn ? columnCache.getColumn() : columnCache.getColumnSelect();
}
```

工具类: `com.baomidou.mybatisplus.core.toolkit.LambdaUtils#resolve`

```java
/**
  * 解析 lambda 表达式, 该方法只是调用了 {@link SerializedLambda#resolve(SFunction)} 中的方法，在此基础上加了缓存。
  * 该缓存可能会在任意不定的时间被清除
  *
  * @param func 需要解析的 lambda 对象
  * @param <T>  类型，被调用的 Function 对象的目标类型
  * @return 返回解析后的结果
  * @see SerializedLambda#resolve(SFunction)
  */
public static <T> SerializedLambda resolve(SFunction<T, ?> func) {
    Class<?> clazz = func.getClass();
    return Optional.ofNullable(FUNC_CACHE.get(clazz))
        .map(WeakReference::get)
        .orElseGet(() -> {
            SerializedLambda lambda = SerializedLambda.resolve(func);
            FUNC_CACHE.put(clazz, new WeakReference<>(lambda));
            return lambda;
        });
}
```

转换类: `com.baomidou.mybatisplus.core.toolkit.support.SerializedLambda#resolve`

```java
/**
  * 通过反序列化转换 lambda 表达式，该方法只能序列化 lambda 表达式，不能序列化接口实现或者正常非 lambda 写法的对象
  *
  * @param lambda lambda对象
  * @return 返回解析后的 SerializedLambda
  */
public static SerializedLambda resolve(SFunction<?, ?> lambda) {
    if (!lambda.getClass().isSynthetic()) {
        throw ExceptionUtils.mpe("该方法仅能传入 lambda 表达式产生的合成类");
    }
    try (ObjectInputStream objIn = new ObjectInputStream(new ByteArrayInputStream(SerializationUtils.serialize(lambda))) {
        @Override
        protected Class<?> resolveClass(ObjectStreamClass objectStreamClass) throws IOException, ClassNotFoundException {
            Class<?> clazz = super.resolveClass(objectStreamClass);
            return clazz == java.lang.invoke.SerializedLambda.class ? SerializedLambda.class : clazz;
        }
    }) {
        return (SerializedLambda) objIn.readObject();
    } catch (ClassNotFoundException | IOException e) {
        throw ExceptionUtils.mpe("This is impossible to happen", e);
    }
}
```

序列化类: `com.baomidou.mybatisplus.core.toolkit.SerializationUtils#serialize`,获取序列化数据.

```java
/**
  * Serialize the given object to a byte array.
  * @param object the object to serialize
  * @return an array of bytes representing the object in a portable fashion
  */
public static byte[] serialize(Object object) {
    if (object == null) {
        return null;
    }
    ByteArrayOutputStream baos = new ByteArrayOutputStream(1024);
    try {
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(object);
        oos.flush();
    } catch (IOException ex) {
        throw new IllegalArgumentException("Failed to serialize object of type: " + object.getClass(), ex);
    }
    return baos.toByteArray();
}
```

```json
{
  "capturedArgCount": 0,
  "capturingClass": "cn/lambda/SerializationTest",
  "functionalInterfaceClass": "cn/lambda/SerializationTest$MyGetter",
  "functionalInterfaceMethodName": "apply",
  "functionalInterfaceMethodSignature": "(Ljava/lang/Object;)Ljava/lang/Object;",
  "implClass": "cn/lambda/SerializationTest$Item",
  "implMethodKind": 5,
  "implMethodName": "getValue",
  "implMethodSignature": "()Ljava/lang/String;",
  "instantiatedMethodType": "(Lcn/lambda/SerializationTest$Item;)Ljava/lang/String;"
}
```

```java
ClassLoader classLoader = SerializationTest.class.getClassLoader();
Class<?> targetClass = classLoader.loadClass(implClass.replace("/", "."));
```

```java
public class SerializationUtils {

    /**
     * 必须有: <code>@FunctionalInterface</code>注解和实现: <code>Serializable</code>
     */
    @FunctionalInterface
    interface MyGetter<T, R> extends Function<T, R>, Serializable {
    }

    /**
     * 获取field名称
     *
     * @param myGetter getter
     * @return String
     */
    public static <T, R> String getFieldName(MyGetter<T, R> myGetter) {
        try {
            // 黑魔法,writeReplace要参考序列化内容
            Method method = myGetter.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(Boolean.TRUE);

            SerializedLambda serializedLambda = (SerializedLambda) method.invoke(myGetter);

            // 这里可以获取到重要的信息
            // System.out.println(JSON.toJSONString(serializedLambda,true));

            return transferGetter2FieldName(serializedLambda.getImplMethodName());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 转换getter方法为属性
     *
     * @param getterName getter名称
     * @return String
     */
    private static String transferGetter2FieldName(String getterName) {
        if (null == getterName || "".equals(getterName.trim())) {
            return null;
        }

        String excludePrefix;
        if (getterName.startsWith("is")) {
            // 布尔类型
            excludePrefix = getterName.substring(2);
        } else {
            // 去除get
            excludePrefix = getterName.substring(3);
        }
        return excludePrefix.substring(0, 1).toLowerCase() + excludePrefix.substring(1);
    }
}
```

Q: writeReplace 究竟是个啥?

A: 请参考如下描述, [博客 link](https://blog.csdn.net/leisurelen/article/details/105980615),[博客 link](https://blog.csdn.net/u012503481/article/details/100896507)

```java
函数式接口如果继承了Serializable，使用Lambda表达式来传递函数式接口时，编译器会为Lambda表达式生成一个writeReplace方法，这个生成的writeReplace方法会返回java.lang.invoke.SerializedLambda；可以从反射Lambda表达式的Class证明writeReplace的存在（具体操作与截图在后面）；所以在序列化Lambda表达式时，实际上写入对象流中的是一个SerializedLambda对象，且这个对象包含了Lambda表达式的一些描述信息；
```

```java
@Data
public static class Item {
    private String valueObject;
    private String label;
    private boolean ok;
    private Boolean failure;
    private String myCamelName;
}

public static void main(String[] args) {
    System.out.println(getFieldName(Item::getValueObject));
    System.out.println(getFieldName(Item::getLabel));
    System.out.println(getFieldName(Item::isOk));
    System.out.println(getFieldName(Item::getFailure));
    System.out.println(getFieldName(Item::getMyCamelName));
}
```