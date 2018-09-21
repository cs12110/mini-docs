# Java IOC

在 Spring 里面 IOC 是一个很重要的功能,如果自己要实现一个最简单的 IOC,那么该怎么实现?窝草,好麻烦 :(

涉及知识点:

> a. 自定义注解
>
> b. 反射
>
> c. 获取 package 下面的所有 class

---

## 1. 代码结构

代码结构(tips: win 平台生成树形结构命令-> `tree /f` )

```shell
│  IocStorage.java
│  TestMyIoc.java
│
├─anno
│      FuckOff.java
│      Resource.java
│      Service.java
│
├─dao
│      ServiceDao.java
│
├─service
│  │  IocService.java
│  │
│  └─impl
│          IocServiceImpl.java
│
└─util
        ClassUtil.java
```

---

## 2. 代码实现

### 2.1 自定义注解类

在测试中,建立三个自定义注解`Resource`,`Service`,`FuckOff`

`Resource.java`自定义注解类如下

```java
package com.ioc.anno;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

/**
 * 用户注解dao
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:50:26
 *
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Resource {
	Class<?> source();
}
```

`Service.java`自定义注解类如下

```java
package com.ioc.anno;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

/**
 * 用户注解service
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:50:13
 *
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Service {
	Class<?> source();
}
```

`FuckOff.java`自定义注解类如下:

```java
package com.ioc.anno;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

/**
 * 用于区别不需要被IOC干预的类
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:50:37
 *
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface FuckOff {

}
```

### 2.2 dao 类

```java
package com.ioc.dao;

/**
 * 业务逻辑处理类
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:54:10
 *
 */
public class ServiceDao {
	public void say(String somethig) {
		System.out.println("ServiceDao Say:" + somethig);
	}
}
```

### 2.3 Service 接口与实现类

`IocService.java`接口

```java
package com.ioc.service;

/**
 * 业务接口
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:56:22
 *
 */
public interface IocService {
	/**
	 * say
	 *
	 * @param str
	 *            whatever you want
	 */
	void say(String str);
}
```

实现类`IocServiceImpl.java`**涉及自定义注解**

```java
package com.ioc.service.impl;

import com.ioc.anno.Resource;
import com.ioc.anno.Service;
import com.ioc.dao.ServiceDao;
import com.ioc.service.IocService;

/**
 * 业务接口实现类
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午4:57:30
 *
 */
@Service(source = IocServiceImpl.class)
public class IocServiceImpl implements IocService {

	@Resource(source = ServiceDao.class)
	private ServiceDao dao;

	public void say(String str) {
		dao.say(str);
	}
}
```

### 2.4 获取 package 里面所有 class 的辅助类

```java
package com.ioc.util;

import java.io.File;
import java.net.URL;
import java.util.Enumeration;
import java.util.HashSet;
import java.util.Set;

import jodd.util.URLDecoder;

/**
 * 获取Class工具类
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午3:59:01
 *
 */
public class ClassUtil {

	/**
	 * 获取class
	 */
	private final static Class<?> worker = new ClassUtil().getClass();

	/**
	 * 获取package下面的所有的class,并加载
	 *
	 * @param pkgName
	 *            指定package
	 * @return Set
	 */
	public static Set<Class<?>> getClassSet(String pkgName) {
		Set<Class<?>> classSet = new HashSet<Class<?>>();
		try {
			String path = pkgName.replace(".", "/");
			Enumeration<URL> resources = worker.getClassLoader().getResources(path);
			while (resources.hasMoreElements()) {
				URL each = resources.nextElement();
				if ("file".equals(each.getProtocol())) {
					String filePath = URLDecoder.decode(each.getFile(), "UTF-8");
					findClazzs(filePath, pkgName, true, classSet);
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return classSet;
	}

	/**
	 * 获取指定package里面的class文件
	 *
	 * @param pkgPath
	 *            package路径
	 * @param pkgName
	 *            package名称
	 * @param recursive
	 *            是否递归获取
	 * @param clazzSet
	 *            class集合
	 */
	private static void findClazzs(String pkgPath, String pkgName, boolean recursive, Set<Class<?>> clazzSet) {
		File file = new File(pkgPath);
		if (!file.exists()) {
			return;
		}

		/**
		 * 获取符合的文件,包括文件夹
		 */
		File[] listFiles = file.listFiles((each) -> {
			return (recursive && each.isDirectory()) || each.getName().endsWith(".class");
		});

		for (File each : listFiles) {
			if (each.isDirectory()) {
				findClazzs(each.getAbsolutePath(), pkgName + "." + each.getName(), recursive, clazzSet);
			} else {
				String name = each.getName();
				int subIndex = each.getName().indexOf(".");
				String clazzName = pkgName + "." + name.substring(0, subIndex);
				try {
					// 加载class
					Class<?> clazz = worker.getClassLoader().loadClass(clazzName);
					clazzSet.add(clazz);
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		}
	}
}
```

### 2.5 IOC 容器

**这个就是最简单的 IOC 容器了**

```java
package com.ioc;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

import com.ioc.anno.FuckOff;
import com.ioc.anno.Resource;
import com.ioc.util.ClassUtil;

/**
 * 注解为{@link FuckOff}不扫描,不然会因为没有无参数的构成方法而初始化异常
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午5:00:44
 *
 */
@FuckOff
public class IocStorage {
	/**
	 * Bean容器
	 */
	private static Map<String, Object> map = new HashMap<String, Object>();

	/**
	 * 扫描package
	 */
	private String pkgName;

	public IocStorage(String pkgName) {
		this.pkgName = pkgName;
		initIocStorage();
	}

	/**
	 * 初始化
	 */
	public void initIocStorage() {
		Set<Class<?>> classSet = ClassUtil.getClassSet(pkgName);
		for (Class<?> each : classSet) {
			if (each.isInterface() || each.isAnnotation() || each.isAnnotationPresent(FuckOff.class)) {
				continue;
			}
			try {
				Object master = each.newInstance();
				Field[] fields = each.getDeclaredFields();
				for (Field f : fields) {
					boolean isResourceAnno = f.isAnnotationPresent(Resource.class);
					if (isResourceAnno) {
						f.setAccessible(true);
						Resource resourceAnno = f.getAnnotation(Resource.class);
						Class<?> injectObjClass = resourceAnno.source();
						String key = injectObjClass.getName();
						Object injectObj = map.get(key);
						if (null == injectObj) {
							injectObj = injectObjClass.newInstance();
							map.put(key, injectObj);
						}
						f.set(master, injectObj);
					}
				}
				map.put(each.getName(), master);
			} catch (Exception e) {
				System.out.println("初始化异常");
			}
		}

	}

	/**
	 * 获取Bean
	 *
	 * @param beanId
	 *            beanId
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public <T> T getBean(String beanId) {
		return (T) map.get(beanId);
	}
}
```

---

## 3. 测试类

```java
package com.ioc;

import com.ioc.service.impl.IocServiceImpl;

/**
 * 测试类
 *
 * <p>
 *
 * @author huanghuapeng 2018年1月2日下午5:04:23
 *
 */
public class TestMyIoc {
	public static void main(String[] args) {
		IocStorage storage = new IocStorage("com.ioc");
		IocServiceImpl impl = storage.getBean("com.ioc.service.impl.IocServiceImpl");

		impl.say("we working at IOC,good luck");
	}

}
```

测试结果

```java
ServiceDao Say:we working at IOC,good luck
```

---

## 3. 参考资料

a. [获取指定 package 下的 class](http://blog.csdn.net/u011666411/article/details/50456644)

b. [自定义 IOC](http://blog.csdn.net/u011229848/article/details/52845821)
