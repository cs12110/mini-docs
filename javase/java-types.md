# Java 类型

该文档主要用于获取泛型对应的具体类是啥.

主要有三种场景:

- 类只实现了接口(1 个/多个接口)
- 类只继承了抽象类(1 个/多个抽象类)
- 类实现了接口和继承了抽象类

---

### 1. 接口

```java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-08-28 08:38
 */
public class OnlyInterfaceTest {

    public interface ExampleItemInterface<K, V> {

    }

    public interface OtherInterface<P> {

    }

    public static class BookItemArg {

    }


    public static class CommonItemImpl implements ExampleItemInterface<Integer, String>, OtherInterface<String> {
    }

    public static class ExtItemImpl implements ExampleItemInterface<Integer, BookItemArg> {
    }

    public static void main(String[] args) {
        // 根据接口获取接口泛型类型数据
        displayInterfaceTypes(CommonItemImpl.class);
        displayInterfaceTypes(ExtItemImpl.class);
    }

    /**
     * 打印接口泛型参数数据,必须实现于接口
     * <pre>
     * Q: 获取接口,如果类实现了多个接口呢???
     * A: 按照CommonItemImpl举例:
     * 第一个实现类: com.dslyy.fd.model.dto.productbill.OnlyInterfaceTest$ExampleItemInterface<java.lang.Integer, java.lang.String>
     * 第二个实现类: com.dslyy.fd.model.dto.productbill.OnlyInterfaceTest$OtherInterface<java.lang.String>
     *
     * </pre>
     *
     * @param clazz class对象
     */
    public static void displayInterfaceTypes(Class<?> clazz) {
        // 可以根据下标获取对应的接口
        Type firstInterface = clazz.getGenericInterfaces()[0];
        // 如果接口没定义参数,直接强转为ParameterizedType会出现异常
        if (firstInterface instanceof ParameterizedType) {
            ParameterizedType impl = (ParameterizedType) firstInterface;
            Type[] argumentArr = impl.getActualTypeArguments();

            System.out.println();
            System.out.println();
            System.out.println("--- " + clazz.getName() + " ---");
            System.out.println();
            int index = 0;
            for (Type each : argumentArr) {
                System.out.println(index + ": " + each.getTypeName());
                index++;
            }
        }
    }
}
```

结果如下:

```
--- com.dslyy.fd.model.dto.productbill.OnlyInterfaceTest$CommonItemImpl ---

0: java.lang.Integer
1: java.lang.String


--- com.dslyy.fd.model.dto.productbill.OnlyInterfaceTest$ExtItemImpl ---

0: java.lang.Integer
1: com.dslyy.fd.model.dto.productbill.OnlyInterfaceTest$BookItemArg
```

---

### 2. 抽象类

```java

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-08-28 08:38
 */
public class OnlyAbstractTest {

    public static abstract class AbstractExampleItem<K, V> {

    }


    public static class BookItemArg {

    }


    public static class CommonItem extends AbstractExampleItem<Integer, String> {
    }

    public static class ExtItem extends AbstractExampleItem<Integer, BookItemArg> {
    }

    public static void main(String[] args) {
        // 根据接口获取接口泛型类型数据
        displayAbstractTypes(CommonItem.class);
        displayAbstractTypes(ExtItem.class);
    }

    /**
     * 打印抽象类泛型参数数据,必须继承于抽象类
     * <pre>
     *
     * </pre>
     *
     * @param clazz class对象
     */
    public static void displayAbstractTypes(Class<?> clazz) {
        // 获取父级类
        Type superClass = clazz.getGenericSuperclass();
        // 如果接口没定义参数,直接强转为ParameterizedType会出现异常
        if (superClass instanceof ParameterizedType) {
            ParameterizedType ext = (ParameterizedType) superClass;
            Type[] argumentArr = ext.getActualTypeArguments();

            System.out.println();
            System.out.println();
            System.out.println("--- " + clazz.getName() + " ---");
            System.out.println();
            int index = 0;
            for (Type each : argumentArr) {
                System.out.println(index + ": " + each.getTypeName());
                index++;
            }
        }
    }
}
```

打印结果

```
--- com.dslyy.fd.model.dto.productbill.OnlyAbstractTest$CommonItem ---

0: java.lang.Integer
1: java.lang.String


--- com.dslyy.fd.model.dto.productbill.OnlyAbstractTest$ExtItem ---

0: java.lang.Integer
1: com.dslyy.fd.model.dto.productbill.OnlyAbstractTest$BookItemArg
```

---

### 3. 使用案例

主要使用于转换数据,比如通过 json 字符串转换成对象

```java
@Test
public void test() {
    TypeReference<Map<String, Integer>> mapTypeReference = new TypeReference<Map<String, Integer>>() {
    };
    String jsonStr = "{\"id\":123}";

    Map<String, Integer> integerMap = JSON.parseObject(jsonStr, mapTypeReference);
    System.out.println(integerMap);
}
```

```java

```
