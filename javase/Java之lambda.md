# Java 之 lambda 表达式

在 java8 之后可以使用一些新特性来编程了,那么我们也学习一下怎么使用.

---

## 1. 常用function

在`java.util.function`有一些常用到的函数编程的类.


| 接口             | 功能                                             | 备注                       |
| ---------------- | ------------------------------------------------ | -------------------------- |
| `Predicate<T>`   | 判断T对象是否符合规则,符合返回true,否则返回false | 默认方法为:Predicate#test  |
| `Consumer<T>`    | 消费接口,不返回任何参数                          | 默认方法为:Consumer#accept |
| `Function<I, J>` | 接受I对象,返回J对象                              | 默认方法为:Function#apply  |
| `Supplier<T>`    | 空参数,返回T对象,相当生成器                      | 默认方法:Supplier#get      |

---

## 2. 公共实体类

公共使用测试实体类

```java
public class MyObj {
	private String myId;

	public String getMyId() {
		return myId;
	}

	public void setMyId(String myId) {
		this.myId = myId;
	}
}
```

```java
public class YourObj {
	private String id;

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}
}
```

测试抽象类

```java
import org.junit.Before;
import org.junit.Test;

public abstract class AbstractTest {

	@Before
	public void before() {
	}

	@Test
	public abstract void test();
}
```
---

## 3. Supplier

```java
import java.util.function.Supplier;

import com.test.handler.AbstractTest;

public class FunctionTest extends AbstractTest {
	@Override
	public void test() {

		/**
		 * 内部类
		 */
		Supplier<MyObj> byInner = new Supplier<MyObj>() {
			@Override
			public MyObj get() {
				MyObj my = new MyObj();
				my.setMyId("byInner:" + System.currentTimeMillis());
				return my;
			}
		};

		/**
		 * lambda
		 */
		Supplier<MyObj> byFun = () -> {
			MyObj my = new MyObj();
			my.setMyId("byFun:" + System.currentTimeMillis());
			return my;
		};

		System.out.println(byInner.get());
		System.out.println(byFun.get());
	}
}
```

测试结果

```java
com.test.simple.MyObj@481a996b
com.test.simple.MyObj@3d51f06e
```
---

## 4. Predicate

```java
import java.util.function.Predicate;
import com.test.handler.AbstractTest;

public class FunctionTest extends AbstractTest {
	@Override
	public void test() {
		MyObj my = new MyObj();
		my.setMyId("1");

		/**
		 * 实现内部类实现
		 */
		Predicate<MyObj> byInner = new Predicate<MyObj>() {
			@Override
			public boolean test(MyObj t) {
				return t.getMyId() != null;
			}
		};

		/**
		 * 使用lambda
		 */
		Predicate<MyObj> byFun = e -> e.getMyId() != null;

		System.out.println("byInner:" + byInner.test(my));
		System.out.println("byFun:" + byFun.test(my));
	}
}
```

测试结果

```java
byInner:true
byFun:true
```

---

## 5. Consumer

```java
import java.util.function.Consumer;
import com.test.handler.AbstractTest;

public class FunctionTest extends AbstractTest {
	@Override
	public void test() {
		MyObj my = new MyObj();
		my.setMyId("2");

		Consumer<MyObj> byInner = new Consumer<MyObj>() {
			@Override
			public void accept(MyObj t) {
				System.out.println("byInner->" + t + ":" + t.getMyId());
			}
		};

		Consumer<MyObj> byFun = e -> {
			System.out.println("byFun->" + e + ":" + e.getMyId());
		};

		byInner.accept(my);
		byFun.accept(my);
	}
}
```

测试结果

```java
byInner->com.test.simple.MyObj@3d51f06e:2
byFun->com.test.simple.MyObj@3d51f06e:2
```


---


## 6. Function

```java

import java.util.function.Function;

import com.test.handler.AbstractTest;

public class FunctionTest extends AbstractTest {
	@Override
	public void test() {

		/**
		 * 内部类
		 */
		Function<MyObj, YourObj> byInner = new Function<MyObj, YourObj>() {
			@Override
			public YourObj apply(MyObj t) {
				YourObj your = new YourObj();
				your.setId(t.getMyId());
				return your;
			}
		};

		/**
		 * lambda
		 */
		Function<MyObj, YourObj> byFun = e -> {
			YourObj your = new YourObj();
			your.setId(e.getMyId());
			return your;
		};

		MyObj my = new MyObj();
		my.setMyId("cs12110");

		YourObj your = byInner.apply(my);
		System.out.println(your + ":" + your.getId());

		YourObj fun = byFun.apply(my);
		System.out.println(fun + ":" + fun.getId());
	}
}
```

测试结果

```java
com.test.simple.YourObj@3d51f06e:cs12110
com.test.simple.YourObj@7ed7259e:cs12110
```