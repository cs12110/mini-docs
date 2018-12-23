# ClassLoader

程序在启动的时候,并不会一次性把 class 全部加载进去,根据程序需要,通过 java 类加载器(classloader)来动态加载某一个 class 文件到内存中,只有 class 文件加载到内存中去,才能被其他 class 引用.所以 classloader 就是用来处理动态加载 class 文件到内存.

---

## 1. Java 默认的三个`classloader`

- `Bootstrap ClassLoader`:启动类加载器,是 Java 类加载的层次最顶层的类加载器,负责加载 JDK 的核心类库,如:`rt.jar`,`resource.java`等.

- `Extension ClassLoader`:扩展类加载器,负责加载扩展类库,默认加载 JAVA_HOME/jre/lib/ext/目录下的所有 jar

- `App ClassLoader`: 系统类加载器,负责加载应用程序 classpath 下面的所有 jar 和 class.

---

## 2. Java 代码

```java
package com.app;

/**
 * 测试class loader
 *
 * @author Mr3306
 *
 */
public class MyClazzLoader {
	public static void main(String[] args) {
		/**
		 * 获取class loader
		 */
		MyClazzLoader my = new MyClazzLoader();
		ClassLoader classLoader = my.getClass().getClassLoader();

		while (null != classLoader) {
			Class<? extends ClassLoader> clazz = classLoader.getClass();
			System.out.println(clazz.getName());
			/**
			 * 获取父级
			 */
			classLoader = classLoader.getParent();
		}
	}
}
```

运行结果

```java
sun.misc.Launcher$AppClassLoader
sun.misc.Launcher$ExtClassLoader
```

---

## 3. Classloder 加载类

同时可以使用`ClassLoader`来加载 Java 对象

```java
package com.app;

/**
 * 测试class loader
 *
 * @author Mr3306
 *
 */
public class MyClazzLoader {
	public static void main(String[] args) {
		/**
		 * 获取当前类加载器,使用的是AppClassLoader
		 */
		ClassLoader cxtLoader = Thread.currentThread().getContextClassLoader();
		System.out.println(cxtLoader);

		/**
		 * 使用class loader加载class
		 */
		try {
			Class<?> tmp = cxtLoader.loadClass(MyClazzLoader.class.getName());
			MyClazzLoader instance = (MyClazzLoader) tmp.newInstance();
			System.out.println(instance);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

测试结果

```java
sun.misc.Launcher$AppClassLoader@18b4aac2
com.app.MyClazzLoader@7fbe847c
```

---

## 4. 小 tips

在使用 classloader 的时候,偶尔会使用`Object.class.getClassloader();`会获取,但是因为加载`Object`的是`BootstrapClassloder`所以得到的为`null`;

```java
public class MyClazz {

	public static void main(String[] args) {
		ClassLoader classLoader = Object.class.getClassLoader();
		System.out.println(classLoader);

		ClassLoader appClassloader = MyClazz.class.getClassLoader();
		System.out.println(appClassloader);
	}
}
```

测试结果

```java
null
sun.misc.Launcher$AppClassLoader@18b4aac2
```

所以要用到 classloader 的时候,请使用自定义的类来获取`classloder`,那样子就不会为`null`了.

---

## 5. 简单例子

下面演示一个简单的动态加载类和该类的依赖的例子. :"}

### 5.1 自定义 classloader

```java
import java.net.URL;
import java.net.URLClassLoader;

/**
 * 动态加载类
 *
 *
 * @author cs12110 at 2018年12月23日下午10:05:19
 *
 */
public class DynamicLoader {

	private static URL[] urls = {};
	private static MyClassLoader classLoder = new MyClassLoader(urls);

	/**
	 * classloader
	 *
	 */
	static class MyClassLoader extends URLClassLoader {
		public MyClassLoader(URL[] urls) {
			super(urls);
		}

		@Override
		public void addURL(URL url) {
			super.addURL(url);
		}
	}

	/**
	 * 添加加载的url
	 *
	 * @param fileUrl url
	 */
	public static void addUrl(URL fileUrl) {
		classLoder.addURL(fileUrl);
	}

	/**
	 * 加载类,如果不存在返回null
	 *
	 * @param className 类名称
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public static <T> T load(String className) {
		T instance = null;
		try {
			Class<?> clazz = classLoder.loadClass(className);
			instance = (T) clazz.newInstance();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return instance;
	}
}
```

### 5.3 测试类

```java
package com.pkgs.invoker;

import java.io.File;

import com.pkgs.api.DynamicApi;
import com.pkgs.entity.PageEntity;
import com.pkgs.util.DynamicLoader;

/**
 * 测试类
 *
 *
 * @author cs12110 at 2018年12月23日下午10:14:38
 *
 */
public class DynamicInvoker {

	public static void main(String[] args) {
		// dynamic-service-0.0.1-SNAPSHOT.jar依赖的mysql连接driver,动态加载进来
		String mysqlDriverJarPath = "D:\\plugins\\deps\\mysql-connector-java-5.1.40.jar";
		// 实现公共接口的类jar
		String dynamicServiceJarPath = "D:\\plugins\\service\\dynamic-service-0.0.1-SNAPSHOT.jar";
		try {

			File mysqlDriverJarFile = new File(mysqlDriverJarPath);
			File file = new File(dynamicServiceJarPath);

			// 加入资源
			DynamicLoader.addUrl(mysqlDriverJarFile.toURI().toURL());
			DynamicLoader.addUrl(file.toURI().toURL());

			// 动态加载实现类,DynamicApi和PageEntity都是公共模块的类
			DynamicApi api = DynamicLoader.load("com.pkgs.service.ZhihuImpl");
			PageEntity page = api.getPage("www.zhihu.com");

			System.out.println(page);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
```

测试结果

```java
PageEntity [url=www.zhihu.com, html=ok ,you can connection to db now, time=1545574512950]
```

---

## 6.参考资料

a. [classloader 原理](http://blog.csdn.net/xyang81/article/details/7292380)
