# 对象复制

如果受够了 getter/setter 设置对象与对象之间的值的话,对象之间的复制,就不可避免了.

---

### 1. 基础知识

#### 1.1 浅复制/深复制

| 名词   | 解释                                                                                                                                                                                        | 备注 |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| 浅拷贝 | 创建一个新对象,然后将当前对象的非静态字段复制到该新对象,如果字段是值类型的,那么对该字段执行复制；如果该字段是引用类型的话,则复制引用但不复制引用的对象.因此,原始对象及其副本引用同一个对象. | -    |
| 深拷贝 | 创建一个新对象,然后将当前对象的非静态字段复制到该新对象,无论该字段是值类型的还是引用类型,都复制独立的一份.当你修改其中一个对象的任何内容时,都不会影响另一个对象的内容.                      | -    |

#### 1.2 其他

因为使用 Spring 里面的 BeanUtils 的功能复制对象,会发现如果对象里面的字段名称一致,而属性类型不一致的情况下,都不能转换该属性值. 泪目.jpg

---

### 2. 方案

使用`orika`来实现对象与对象之间的复制.

#### 2.1 依赖

```xml
<properties>
    <orika.version>1.4.2</orika.version>
</properties>

<dependency>
    <groupId>ma.glasnost.orika</groupId>
    <artifactId>orika-core</artifactId>
    <version>${orika.version}</version>
</dependency>
```

#### 2.2 工具类

```java
import java.util.List;
import java.util.Objects;

import ma.glasnost.orika.MapperFacade;
import ma.glasnost.orika.impl.DefaultMapperFactory;
import ma.glasnost.orika.metadata.Type;
import ma.glasnost.orika.metadata.TypeFactory;

/**
 * @author cs12110
 * @version V1.0
 * @since 2020-11-24 08:58
 */
public class BeanCopyUtil {

    private static MapperFacade mapperFacade;

    /*
     * 初始化
     */
    static {
        DefaultMapperFactory.Builder builder = new DefaultMapperFactory.Builder();
        DefaultMapperFactory factory = builder.build();
        mapperFacade = factory.getMapperFacade();
    }

    /**
     * 转换对象
     *
     * @param src 来源
     * @param dst 目标
     * @param <S> 来源类型
     * @param <D> 目标类型
     * @return 目标对象
     */
    public static <S, D> D copy(S src, Class<D> dst) {
        if (Objects.isNull(src) || Objects.isNull(dst)) {
            throw new RuntimeException("Illegal args");
        }
        return mapperFacade.map(src, dst);
    }

    /**
     * 转换对象(听说是更快速)
     *
     * @param src     原对象
     * @param srcType 原对象Type
     * @param dstType 目标对象Type
     * @param <S>     原对象类型
     * @param <D>     目标对象类型
     * @return D
     */
    public static <S, D> D copy(S src, Type<S> srcType, Type<D> dstType) {
        if (Objects.isNull(src) || Objects.isNull(srcType) || Objects.isNull(dstType)) {
            throw new RuntimeException("Illegal args");
        }
        return mapperFacade.map(src, srcType, dstType);
    }

    /**
     * 转换list
     *
     * @param it   原迭代器
     * @param dist 目标类型
     * @param <S>  原列表类型
     * @param <D>  目标列表类型
     * @return List
     */
    public static <S, D> List<D> copyList(Iterable<S> it, Class<D> dist) {
        if (Objects.isNull(it) || Objects.isNull(dist)) {
            throw new RuntimeException("Illegal args");
        }
        return mapperFacade.mapAsList(it, dist);
    }

    /**
     * 转换list
     *
     * @param it      迭代器
     * @param srcType 原类型
     * @param dstType 目标类型
     * @param <S>     原对象类型
     * @param <D>     目标对象类型
     * @return List
     */
    public static <S, D> List<D> copyList(Iterable<S> it, Type<S> srcType, Type<D> dstType) {
        if (Objects.isNull(it) || Objects.isNull(srcType) || Objects.isNull(dstType)) {
            throw new RuntimeException("Illegal args");
        }
        return mapperFacade.mapAsList(it, srcType, dstType);
    }

    /**
     * 获取type
     *
     * @param clazz clazz
     * @param <D>   目标类型
     * @return Type
     */
    public static <D> Type<D> getType(Class<D> clazz) {
        if (Objects.isNull(clazz)) {
            throw new RuntimeException("Illegal args");
        }
        return TypeFactory.valueOf(clazz);
    }
}
```

---

### 3. 测试

#### 3.1 定义对象

```java
import com.alibaba.fastjson.JSON;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2020-11-24 09:19
 */
@Data
public class StudentA {

    private String id;
    private String name;
    private String age;
    private String birthday;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

```java
import com.alibaba.fastjson.JSON;

import java.util.Date;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2020-11-24 09:19
 */
@Data
public class StudentB {

    private Long id;
    private String name;
    private Integer age;
    private Date birthday;

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }

}
```

#### 3.2 测试

```java
import com.alibaba.fastjson.JSON;
import com.pkgs.model.StudentA;
import com.pkgs.model.StudentB;
import com.pkgs.util.BeanCopyUtil;

import java.util.ArrayList;
import java.util.List;

/**
 * @author cs12110
 * @version V1.0
 * @since 2020-11-24 08:58
 */
public class BeanCopyService {

    public static void main(String[] args) {
        // date和string类型转换出错
        StudentA studentA = new StudentA();
        studentA.setAge("33");
        studentA.setId("01");
        studentA.setName("测试");
        // 转换不正常
        // studentA.setBirthday("1984-03-06 12:00:00");

        StudentB studentB = BeanCopyUtil.copy(studentA, StudentB.class);
        System.out.println(studentB);

        StudentB stuB = BeanCopyUtil
            .copy(studentA, BeanCopyUtil.getType(StudentA.class), BeanCopyUtil.getType(StudentB.class));

        System.out.println(stuB);

        List<StudentA> list = new ArrayList<>();
        list.add(studentA);

        List<StudentB> stuList = BeanCopyUtil.copyList(list, StudentB.class);
        System.out.println(JSON.toJSONString(stuList));

    }
}
```

测试结果:

```java
{"age":33,"id":1,"name":"测试"}
{"age":33,"id":1,"name":"测试"}
[{"age":33,"id":1,"name":"测试"}]
```

---

### 4. 参考资料

a. [orika 官网](http://orika-mapper.github.io/orika-docs/index.html)
