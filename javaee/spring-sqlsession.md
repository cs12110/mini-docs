# Spring 与 SqlSession

在 Mybatis(没集成 Spring 的前提下)里面的一级缓存是默认打开的.

Q: 那么在 Spring 里面里面会是怎样子的呢?

A: 默认是每一次的增删改查都从新获取 SqlSession,`如果当前没有事务的前提下,执行完则关闭 SqlSession,所以一级缓存是没有任何作用`.

---

## 1. 源码分析

整合 Spring 之后,使用 SqlSessionTemplate 来操作增删改查.

项目配置 SqlSessionTemplate 的配置如下:

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    return new SqlSessionTemplate(sqlSessionFactory,
            this.properties.getExecutorType());
}
```

`SqlSessionTemplate`位于`org.mybatis.spring`的包下面.

```java
package org.mybatis.spring;

public class SqlSessionTemplate implements SqlSession, DisposableBean {
    ....
}
```

调用的构造方法如下:

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
    PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
}
```

这里面使用动态代理构造了一个`sqlSessionProxy`的对象,而且该对象只代理了`SqlSession`接口,`InvocationHandler`为该类最下面的`SqlSessionInterceptor`.

```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
}
```

比如执行 select 方法

```java
@Override
public void select(String statement, Object parameter, ResultHandler handler) {
    this.sqlSessionProxy.select(statement, parameter, handler);
}
```

都会指向动态代理`SqlSessionInterceptor#invoke`方法.

而`SqlSessionInterceptor#invoke`方法每一次都重新获取`SqlSession`,然后在`finally`里面`closeSqlSession`掉了.

`closeSqlSession`方法请参考:`org.mybatis.spring.SqlSessionUtils`里面的代码.

```java
public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    // 获取当前事务,如果当前没有事务,则关闭session
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    if ((holder != null) && (holder.getSqlSession() == session)) {
        if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Releasing transactional SqlSession [" + session + "]");
        }
        holder.released();
    } else {
        if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Closing non transactional SqlSession [" + session + "]");
        }
        session.close();
    }
}
```

---

## 2. 总结

spring 要命.
