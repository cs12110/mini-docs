# CGLib 与 Proxy

CGLib 与 Java 自带的 Proxy.

---

## 1. CGLib 的使用

温馨提示: **CGlib 不能代理 final 和 static 的方法**

### 2.1 pom.xml

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.10</version>
</dependency>
```

### 2.2 工具类

```java
import net.sf.cglib.proxy.*;

import java.lang.reflect.Method;

public class CgLibHandler implements MethodInterceptor {

    private static Enhancer enhancer = new Enhancer();

    /**
     * 创建代理
     *
     * @param target target class
     * @param <T>    T
     * @return T
     */
    public <T> T getProxy(Class<T> target) {
        enhancer.setSuperclass(target);
        enhancer.setCallback(this);
        Object proxy = enhancer.create();
        return (T) proxy;
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        doBefore(method.getName());
        Object result = methodProxy.invokeSuper(o, objects);
        doAfter(method.getName());
        return result;
    }

    /**
     * do before
     *
     * @param methodName 方法名称
     */
    private void doBefore(String methodName) {
        System.out.println("do before: " + methodName);
    }

    /**
     * do after
     *
     * @param methodName 方法名称
     */
    private void doAfter(String methodName) {
        System.out.println("do after: " + methodName);
    }
}
```

### 2.3 测试

```java
public class MyService {

    public void say(String something) {
        System.out.println("say:" + something);
    }

}
```

```java
public class CgTest {

    public static void main(String[] args) {
        CgLibHandler libUtil = new CgLibHandler();
        MyService proxy = libUtil.getProxy(MyService.class);
        proxy.say("cs12110 by cglib");
    }
}
```

测试结果

```java
do before: say
say:cs12110 by cglib
do after: say
```

---

## 2. Proxy

Java 动态代理.不需要依赖第三方 jar,直接就可以实现.

**但代理的类一定要继承接口.**

### 2.1 代码

```java

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * Java动态代理
 * <p/>
 *
 * @author cs12110 created at: 2019/1/8 15:20
 * <p>
 * since: 1.0.0
 */
public class ProxyHandler implements InvocationHandler {

    /**
     * 组装代理
     *
     * @param clazz 接口实现类class对象
     * @param <T>   接口
     * @return T
     */
    @SuppressWarnings("{unchecked}")
    public static <T> T wrapper(Class<? extends T> clazz) {
        T proxy = null;
        try {
            T instance = clazz.newInstance();
            proxy = (T) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), new ProxyHandler(instance));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return proxy;
    }

    /**
     * 需被代理执行对象
     */
    private Object target;

    private ProxyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        doBefore();
        Object invoke = method.invoke(target, args);
        doAfter();
        return invoke;
    }

    private void doBefore() {
        System.out.println("do before");
    }

    private void doAfter() {
        System.out.println("do after");
    }
}
```

### 2.2 测试代码

```java
public interface MyService {

    public void say(String str);
}
```

```java
public class MyServiceImpl implements MyService {
    @Override
    public void say(String str) {
        System.out.println("say: " + str);
    }
}
```

```java
public class ProxyHandlerTest {

    @Test
    public void test() {
        // 这里一定要是接口,不然会出现异常 :"}
        MyService service = ProxyHandler.wrapper(MyServiceImpl.class);

        service.say("123");
    }
}
```

测试结果

```java
do before
say: 123
do after
```

---

## 3. 结论

如果要实现更灵活的代理(不使用接口的情况),CGLib 无疑是最佳的选择.
