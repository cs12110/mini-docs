# Java pattern

理解设计模式,最难的是在合适的场景里面使用. orz

---

## 1. 单例模式

### 1.1 饿汉模式

这种也是比较常用的一种模式,缺点就是在 jvm 加载的时候这个类就被加载了,连拒绝的机会都不给你.

```java
/**
 * 单例模式
 *
 * <p>
 *
 * @author cs12110 create at 2019-04-02 21:50
 * <p>
 * @since 1.0.0
 */
public class Singleton {

    private static Singleton singleton = new Singleton();

    private Singleton() {

    }

    public static Singleton getInstance() {
        return singleton;
    }

}
```

### 1.2 装逼模式

Q: 那么如果想要在使用到的时候才加载使用呢?

A: 那就有各种奇葩炫技创建方式了.double check 都出来了,问你怕不怕.

```java
/**
 * 单例模式
 *
 * <p>
 *
 * @author cs12110 create at 2019-04-02 21:50
 * <p>
 * @since 1.0.0
 */
public class Singleton {

    private static Singleton singleton;

    private Singleton() {

    }

    public static Singleton getInstance() {
        // double check
        if (null == singleton) {
            synchronized (Singleton.class) {
                if (null == singleton) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

---

## 2. builder 模式

觉得这个模式还是挺实用的.

使用场景: 比如在方法里面组建一个实体类,那个实体类可能有 10 多个参数,你从方法的参数传过去,这就...

so, here we go.

```java
@Data
class Student {
    private String id;
    private String name;
    private int age;


    /**
     * 设置为private
     *
     * @param builder builder
     */
    private Student(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.age = builder.age;
    }

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }

    /**
     * builder
     */
    public static class Builder {
        private String id;
        private String name;
        private int age;

        public Builder setId(String id) {
            this.id = id;
            return this;
        }

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setAge(int age) {
            this.age = age;
            return this;
        }

        public Student build() {
            return new Student(this);
        }
    }
}
```

测试

```java
public class Test {
    public static void main(String[] args) {
        Student student = new Student.Builder()
                .setId("cs12110")
                .setName("mr3306")
                .setAge(33)
                .build();

        System.out.println(student.toString());
    }
}
```
---

## 3. 工厂模式

偶尔有用到这个模式,也不知道是不是用对,这真的很烦呀.
