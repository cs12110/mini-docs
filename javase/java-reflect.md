# Java Reflect

黑魔法警告.

---

## 1. 初始化

Q: 这有啥好说的?

A: 如果我们要初始化一个如下的类:

```java
package com.reflect.a;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-02-15 21:38
 */
@Data
public class MyItem {

    private String attr;

    MyItem(String attr) {
        this.attr = attr;
    }

}
```

| 修饰符    | class | package | subclass | world |
| --------- | ----- | ------- | -------- | ----- |
| public    | Y     | Y       | Y        | Y     |
| protected | Y     | Y       | Y        | N     |
| default   | Y     | Y       | N        | N     |
| private   | Y     | N       | N        | N     |

由上面的控制访问可以看出,默认级别的只能在 class 和 package 才能使用.

Q: 那我要是想在 a 的 package 外边初始化 MyItem 对象,要怎么操作呀?

A: 可以使用反射来操作.

```java
package com.reflect.b;

import com.alibaba.fastjson.JSON;
import com.reflect.a.MyItem;

import java.lang.reflect.Constructor;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-02-15 21:39
 */
public class MyItemTest {
    public static void main(String[] args) {
        try {
            // 获取构造器
            Constructor<MyItem> constructor = MyItem.class.getDeclaredConstructor(String.class);

            // 设置构造器可以访问
            constructor.setAccessible(Boolean.TRUE);

            // 构造生成对象
            MyItem myItem = constructor.newInstance("3306");

            System.out.println(JSON.toJSONString(myItem));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

测试结果:

```json
{ "attr": "3306" }
```

---

## 2. 修改静态常量

Q: 如果我想修改工具类里面的静态常量,我可以做到呢?如下所示的工具类:

```java
package com.reflect.a;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-02-15 21:38
 */
@Data
public class MyItem {

    private String attr;

    public MyItem(String attr) {
        this.attr = attr;
    }

    public static MyItem create(String attr) {
        return new MyItem(attr);
    }
}
```

```java
package com.reflect.b;

import com.reflect.a.MyItem;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-02-15 21:51
 */
public class MyUtil {

    private final static MyItem myItem = MyItem.create("myUtil");

    public static MyItem getMyItem() {
        return myItem;
    }
}
```

A: 是时候展示黑魔法了.

```java
package com.reflect.b;

import com.alibaba.fastjson.JSON;
import com.reflect.a.MyItem;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-02-15 21:39
 */
public class MyItemTest {
    public static void main(String[] args) {
        try {
            System.out.println(JSON.toJSONString(MyUtil.getMyItem()));


            // 获取字段
            Field myItemField = MyUtil.class.getDeclaredField("myItem");
            myItemField.setAccessible(Boolean.TRUE);

            // 设置static final字段可以读取
            Field modifiersField = Field.class.getDeclaredField("modifiers");
            modifiersField.setAccessible(Boolean.TRUE);
            modifiersField.set(myItemField, myItemField.getModifiers() & ~Modifier.FINAL);

            // 反射设置新的值
            myItemField.set(null, MyItem.create("cs12110"));


            System.out.println(JSON.toJSONString(MyUtil.getMyItem()));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

测试结果:

```json
{"attr":"myUtil"}
{"attr":"cs12110"}
```

---