# Mockito

Java 的测试里面,除了 Junit 的测试,Mockito,你值得拥有.

---

## 1. 简单介绍

在 Junit 的测试里面,比如要测试`service1#get(String id)`,那么你要全都实现.

如果`service1#get(String id)`依赖调用的东西还没写好,那该怎么办? 现在就该使用 mockito 了.

---

## 2. pom.xml

主要依赖`junit`和`mockito`,使用版本如下所示.

```xml
<!-- junit -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.10</version>
    <scope>test</scope>
</dependency>

<!-- Mockito -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>1.10.19</version>
    <scope>test</scope>
</dependency>
```

---

## 3. HelloWorld

定义实体类

```java
package com.pkgs.entity;

/**
 * 定义实体类
 *
 *
 * @author cs12110 at 2018年12月8日下午8:50:01
 *
 */
public class Person {

	private String id;
	private String name;

	public Person() {
	}

	public Person(String id, String name) {
		this.id = id;
		this.name = name;
	}

	//省略setter/getter
}
```

定义 Mapper

```java
package com.pkgs.mapper;

import com.pkgs.entity.Person;

/**
 * Mapper接口
 *
 *
 * @author cs12110 at 2018年12月8日下午8:46:39
 *
 */
public interface PersonMapper {
	Person get(String id);
	boolean update(Person person);
}
```

定义 Service

```java
package com.pkgs.service;

import com.pkgs.entity.Person;
import com.pkgs.mapper.PersonMapper;

/**
 * Person业务实现类
 *
 *
 * @author cs12110 at 2018年12月8日下午8:51:51
 *
 */
public class PersonService {

	/**
	 * mapper尚没有任何实现类
	 */
	private PersonMapper mapper;

	public PersonService(PersonMapper personDao) {
		super();
		this.mapper = personDao;
	}

	public boolean update(String id, String name) {
		Person person = mapper.get(id);
		if (null == person) {
			return false;
		}
		Person info = new Person();
		info.setId(id);
		info.setName(name);
		return mapper.update(info);
	}
}
```

定义测试类

```java
package com.test;

import static org.junit.Assert.assertTrue;

import org.junit.Before;
import org.junit.Test;
import org.mockito.Mockito;

import com.pkgs.entity.Person;
import com.pkgs.mapper.PersonMapper;
import com.pkgs.service.PersonService;

/**
 * 测试
 *
 *
 * @author cs12110 at 2018年12月8日下午8:46:15
 *
 */
public class PersonServiceTest {

	private PersonMapper mapper;
	private PersonService personService;

	@Before
	public void setUp() throws Exception {
		/**
		 * 构建mock,是不是像实现一个mapper接口?
		 */
		mapper = Mockito.mock(PersonMapper.class);
		Mockito.when(mapper.get("1")).thenReturn(new Person("1", "Person"));
		Mockito.when(mapper.update(Mockito.isA(Person.class))).thenReturn(true);

		personService = new PersonService(mapper);
	}

	@Test
	public void testUpdate() {
		boolean result = personService.update("1", "Apache");
    	/**
		 * 当为true,才往下执行,否则抛出异常
		 */
		assertTrue("must be true", result);

		Mockito.verify(mapper, Mockito.times(1)).get(Mockito.eq("1"));
		Mockito.verify(mapper, Mockito.times(1)).update(Mockito.isA(Person.class));
	}

	@Test
	public void testUpdateNotFound() {
		boolean result = personService.update("1", "Apache");
		assertTrue("must be true", result);

		Mockito.verify(mapper, Mockito.times(1)).get(Mockito.eq("1"));
        /**
         * 因为前面update了一次,这里的条件为never,所以会出现异常.
         */
		Mockito.verify(mapper, Mockito.never()).update(Mockito.isA(Person.class));
	}
}
```

---

## 4. 使用注解

使用注解可可以让代码更简洁,那么我们看看 mockito 怎么使用注解的.

```java
package com.test;

import java.util.List;

import org.junit.Before;
import org.junit.Test;
import org.mockito.Mockito;
import org.mockito.MockitoAnnotations;

/**
 * 测试注解
 *
 *
 * @author cs12110 at 2018年12月8日下午11:04:57
 *
 */
public class MockAnnoTest {

	@org.mockito.Mock
	private List<String> mock;

	@Before
	public void init() {
		// 必须使用如下来初始化
		MockitoAnnotations.initMocks(this);
	}

	@Test
	public void test() {

		Mockito.when(mock.get(1)).thenReturn("mock value");

		System.out.println(mock.get(1));

		Mockito.verify(mock).get(1);
	}
}
```

---

## 5. 校验行为

作用: **校验发生的行为是不是符合条件,如果符合则测试通过,不符合则抛出异常.**

```java
package com.test;

import java.awt.List;

import org.junit.Test;
import org.mockito.Mockito;

/**
 * 校验行为
 *
 *
 * @author cs12110 at 2018年12月8日下午9:06:27
 *
 */
public class VerifyBehavierTest {

	@Test
	public void test() {
		List mockList = Mockito.mock(List.class);
		mockList.add("1");
		mockList.add("1");
		mockList.add("2");
		mockList.add("3");

		mockList.remove("3");

		/**
		 * 判断是不是add了两次1,如果调用了两次则测试通过
         *
         * 使用Mockito.times(timeNum)可以精确校验调用过的次数.
		 */
		Mockito.verify(mockList, Mockito.times(2)).add("1");

		/**
		 * 判断是不是add了两次2,因为上面只add了一次,所以会抛出异常
		 */
		Mockito.verify(mockList, Mockito.times(2)).add("2");

		/**
		 * 判断是不是最少remove了一次3
		 */
		Mockito.verify(mockList, Mockito.atLeastOnce()).remove("3");
	}
}
```

---

## 6. 设置桩

设置桩: **规定某些操作,触发怎样的操作,或者返回怎样子的参数.**

温馨提示:

- 对于有返回值的方法,会默认返回 null,空集合,默认值.比如为 int/Integer 返回 0

- stubbing 可以被覆盖,但是覆盖已有的 stubbing 有可能不是很好

- stubbing 不管调用多少次,方法都会永远返回 stubbing 的值

- 当对同一个方法进行多次 stubbing,最后一次 stubbing 最重要

```java
/**
 * service
 *
 *
 * @author cs12110 at 2018年12月8日下午9:22:23
 *
 */
class MyStubbingService {
	/**
	 * 查询
	 *
	 * @param id id
	 * @return String 默认为null
	 */
	public String search(String id) {
	    return "return: " + id;
	}

}
```

```java
package com.test;

import org.junit.Test;
import org.mockito.Mockito;

/**
 * 设置桩
 *
 *
 * @author cs12110 at 2018年12月8日下午9:16:13
 *
 */
public class StubbingTest {

	@Test
	public void test() {
		MyStubbingService stubbing = Mockito.mock(MyStubbingService.class);

		Mockito.when(stubbing.search("1")).thenReturn("search by 1");
		Mockito.when(stubbing.search(null)).thenThrow(new IllegalArgumentException("the id must be somehint"));

		/**
		 * 返回: search by 1
		 */
		System.out.println(stubbing.search("1"));

		/**
		 * 返回: null,因为上面没规定返回什么
		 */
		System.out.println(stubbing.search("2"));

		/**
		 * 判断前面是否调用过stubbing.search("1")
		 */
		Mockito.verify(stubbing).search("1");

		/**
		 * 抛出IllegalArgumentException
		 */
		stubbing.search(null);
	}
}
```

---

## 7. 参数校验

作用: **校验参数是否符合规范/规则**

```java
package com.test;

import java.util.List;

import org.junit.Test;
import org.mockito.Mockito;

/**
 * 校验参数
 *
 *
 * @author cs12110 at 2018年12月8日下午9:52:51
 *
 */
public class VerifyArgTest {

	@Test
	@SuppressWarnings("unchecked")
	public void testArg() throws Exception {
		List<String> list = (List<String>) Mockito.mock(List.class);

		/**
		 * Get by int
		 */
		Mockito.when(list.get(Mockito.anyInt())).thenReturn("get by int");

		System.out.println(list.get(1));

		/**
		 * 运行过list.get(1),测试通过
		 */
		Mockito.verify(list).get(Mockito.anyInt());

		/**
		 * 主要要使用参数匹配器,而不是直接写3306了
		 */
		Mockito.verify(list).get(Mockito.eq(3306));
	}
}
```

---

## 8. 处理 void 方法

作用: **处理 void 方法**

```java
import java.util.List;

import org.junit.Test;
import org.mockito.Mockito;

/**
 * 测试调用void返回参数方法
 *
 *
 * @author cs12110 at 2018年12月8日下午9:58:34
 *
 */
public class VerifyVoidest {
	@Test
	public void testOrder() throws Exception {
		List<?> mock = Mockito.mock(List.class);
		/**
		 * mock.clear()的返回参数为null
		 */
		Mockito.doThrow(new RuntimeException("invoke void method")).when(mock).clear();
	}
}
```

---

## 9. 校验顺序

作用: **校验方法的调用顺序**

```java
package com.test;

import java.util.List;

import org.junit.Test;
import org.mockito.InOrder;
import org.mockito.Mockito;

/**
 * 测试调用顺序
 *
 *
 * @author cs12110 at 2018年12月8日下午9:58:34
 *
 */
public class VerifyOrderest {

	@Test
	@SuppressWarnings("unchecked")
	public void testOrder() throws Exception {
		List<Object> orderMock = (List<Object>) Mockito.mock(List.class);

		orderMock.add("add first");
		orderMock.add("add second");

		InOrder onMyMark = Mockito.inOrder(orderMock);

		/**
		 * 校验调用顺序,如果执行不按照上面规定的顺序则抛出异常
		 */
		onMyMark.verify(orderMock).add("add first");
		onMyMark.verify(orderMock).add("add second");

		// 测试多个mock
		List<Object> mock1 = (List<Object>) Mockito.mock(List.class);
		List<Object> mock2 = (List<Object>) Mockito.mock(List.class);

		mock1.add(1);
		mock2.add(2);

		/**
		 * 校验mock1必须比mock2先执行,且mock1.add的为1,mock2.add的为2
		 */
		InOrder multipleOrder = Mockito.inOrder(mock1, mock2);
		multipleOrder.verify(mock1).add(1);
		multipleOrder.verify(mock2).add(2);
	}
}
```

---

## 10. 根据调用顺序设置 stubbing

作用: **根据不同的调用顺序,设置返回值,注意超过之后返回最后一个 stubbing 值.**

```java
package com.test;

import org.junit.Test;
import org.mockito.Mockito;

/**
 * 校验调用不同顺序,返回不同的结果
 *
 *
 * @author cs12110 at 2018年12月8日下午10:25:32
 *
 */
public class VerifySpecialStubbing {
	@Test
	public void test() throws Exception {

		MyIf myIf = Mockito.mock(MyIf.class);
		Mockito.when(myIf.forMock("1")).thenReturn("invoke 1").thenReturn("invoke 2");

		System.out.println(myIf.forMock("1"));
		System.out.println(myIf.forMock("1"));
		System.out.println(myIf.forMock("1"));

	}
}

interface MyIf {
	String forMock(String str);
}
```

---

## 11. 参数捕获

作用: **捕获参数和参数值来做校验**

```java
package com.test;

import java.util.List;

import org.junit.Assert;
import org.junit.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;

/**
 * 校验参数
 *
 *
 * @author cs12110 at 2018年12月8日下午10:49:26
 *
 */
public class VerifyCaptureArgTest {

	@Test
	@SuppressWarnings("unchecked")
	public void test() {
		List<String> mock = Mockito.mock(List.class);

		ArgumentCaptor<String> argCaptor = ArgumentCaptor.forClass(String.class);

		mock.add("3306");

		Mockito.verify(mock).add(argCaptor.capture());

		System.out.println(argCaptor.getValue());

		Assert.assertEquals("3306", argCaptor.getValue());
	}
}
```

---

## 12. SpringBoot 与 Mockito

请确保`springboot`,`springboot-test`,`junit`,`mockito`依赖.

请注意: **测试 service 上面并没有声明任何的 Spring 注解**

### 12.1 Mapper 接口

```java
public interface MockitoMapper {
	public String say(String something);
}
```

### 12.2 Service 接口

```java
import org.springframework.beans.factory.annotation.Autowired;

public class MockitoService {

	@Autowired
	private MockitoMapper mockitoMapper;

	public void say(String s) {
		System.out.println(this + ":" + mockitoMapper.say(s));
	}
}
```

### 12.3 测试类

抽象父类

```java
package com.test.spring;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.MockitoAnnotations;
import org.slf4j.ILoggerFactory;
import org.slf4j.LoggerFactory;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import com.pkgs.WebApp;

/**
 * Spring test基类
 *
 * <pre>
 * `@SpringBootTest(classes = WebApp.class)` 为`@SpringBootApplication`的主方法类
 * </pre>
 *
 * @author cs12110 2018年12月13日
 * @see
 * @since 1.0
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = WebApp.class)
public abstract class AbstractSpringTest {
	static {
		ILoggerFactory factory = LoggerFactory.getILoggerFactory();
		// logback的logger,而不是slf4j的logger
		Logger logger = (Logger) factory.getLogger("org.springframework");
		logger.setLevel(Level.ERROR);
	}

	@Before
	public void before() {
		MockitoAnnotations.initMocks(this);
	}

	@Test
	public abstract void test();
}
```

测试类

```java
package com.test.mockito;

import org.junit.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Mockito;

import com.test.spring.AbstractSpringTest;

import com.pkgs.service.mockito.MockitoMapper;
import com.pkgs.service.mockito.MockitoService;

/**
 * 注意@Mock和@InjectMocks注解
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月18日上午10:23:59
 * @see
 * @since 1.0
 */
public class MockitoTest extends AbstractSpringTest {

	@Mock
	private MockitoMapper mockitoMapper;

	@InjectMocks
	private MockitoService mockitoService;

	@Test
	public void test() {
		Mockito.doReturn("world").when(mockitoMapper).say("hello");

		MockitoService spy = Mockito.spy(mockitoService);
		spy.say("hello");
	}
}
```

测试结果

```java
cn.rojao.service.mockito.MockitoService$MockitoMock$1187350920@26e74d50:world
```

---

## 13. 参考资料

a. [Mockito 官方教程](https://static.javadoc.io/org.mockito/mockito-core/2.23.4/org/mockito/Mockito.html)

b. [Mockito 使用指南](https://blog.csdn.net/shensky711/article/details/52771493)

c. [SpringBoot 与 Mockito 使用教程](https://www.jb51.net/article/134904.htm)
