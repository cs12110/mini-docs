# Spring issues

spring,springbooot,spring cloud 各种 issues.

---

## 1. Spring 注解

注解的使用.

### 1.1 @Autowired 与 @Resource

**@Autowired**: 注解是`按照类型(byType)`装配依赖对象,默认情况下它要求依赖对象必须存在,如果允许 null 值,可以设置它的 required 属性为 false.如果我们想使用按照名称(byName)来装配,可以结合@Qualifier 注解一起使用.

**@Resource**: 默认`按照 byName` 自动注入,由 J2EE 提供,需要导入包 javax.annotation.Resource.@Resource 有两个重要的属性: `name` 和 `type`,而 Spring 将@Resource 注解的 name 属性解析为 bean 的名字,而 type 属性则解析为 bean 的类型.所以,如果使用 name 属性,则使用 byName 的自动注入策略,而使用 type 属性时则使用 byType 自动注入策略.如果既不制定 name 也不制定 type 属性,这时将通过反射机制使用 byName 自动注入策略.

@Resource 装配顺序:

- 如果同时指定了 name 和 type,则从上下文中找到唯一匹配的 bean 进行装配,找不到则抛出异常.

- 如果指定了 name,则从上下文中查找名称(id)匹配的 bean 进行装配,找不到则抛出异常.

- 如果指定了 type,则从上下文中找到类似匹配的唯一 bean 进行装配,找不到或是找到多个,都会抛出异常.

- 如果既没有指定 name,又没有指定 type,则自动按照 byName 方式进行装配,如果没有匹配,则按照类型进行匹配.

`@Resource` 的作用相当于`@Autowired`,只不过`@Autowired` 按照 `byType` 自动注入.

### 1.2 @ConditionalOnBean 与@ConditionalOnClass

| 注解                       | 作用                                           | 备注 |
| -------------------------- | ---------------------------------------------- | ---- |
| @ConditionalOnBean         | 当给定的在 bean 存在时,则实例化当前 Bean       | -    |
| @ConditionalOnMissingBean  | 当给定的在 bean 不存在时,则实例化当前 Bean     | -    |
| @ConditionalOnClass        | 当给定的类名在类路径上存在,则实例化当前 Bean   | -    |
| @ConditionalOnMissingClass | 当给定的类名在类路径上不存在,则实例化当前 Bean | -    |



### 1.3 @ConditionalOnProperty

在 feign 的`HttpClientFeignLoadBalancedConfiguration`里面有下面的注解

```java
@ConditionalOnProperty(
    value = {"feign.httpclient.enabled"},
    matchIfMissing = true
)
```

Q: 那么这些都是些啥玩意呢?

A: 来,来,来.

```yml
student:
  name: haiyan
  age: 18
  #stu-no: cs12110
```

```java
/**
 * `@ConditionalOnProperty(prefix = "student", name = "stu-no", matchIfMissing = true)`
 * 即使配置文件里面的student配置缺少stu-no字段时,也可以加载
 * <p>
 * `@ConditionalOnProperty(prefix = "student", name = "stu-no", matchIfMissing = false)`
 * 当使配置文件里面的student配置缺少stu-no字段时,不可以加载并抛出异常
 *
 * @author cs12110 create at 2020/4/9 18:22
 * @version 1.0.0
 */
@Data
@Component
@ConditionalOnProperty(prefix = "student", name = "stu-no", matchIfMissing = true)
@ConfigurationProperties(prefix = "student")
public class StudentConf {
    private String stuNo;
    private String name;
    private Integer age;
}
```

---

## 2. SpringCloud

### 2.1 ribbon 与 feign

### 2.2 zuul 与 ribbon
