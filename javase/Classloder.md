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

如果需要动态加载外部的 jar 里面的类,该怎么办?怎么办?怎么办?

### 5.1 自定义 classloader

```java
import java.net.URL;
import java.net.URLClassLoader;

/**
 * 定制个人的classloader,用于加载用户第三方对象
 *
 *
 * <p>
 *
 * @author hhp 2018年10月17日
 * @see
 * @since 1.0
 */
public class PluginClassLoader extends URLClassLoader {

	/**
	 * URL
	 *
	 * @param urls
	 */
	public PluginClassLoader(URL[] urls) {
		super(urls);
	}

	public PluginClassLoader(URL[] urls, ClassLoader parent) {
		super(urls, parent);
	}

	/**
	 * 新增jar包
	 *
	 * @param url
	 *            jar的url地址
	 */
	public void addJar(URL url) {
		this.addURL(url);
	}
}
```

### 5.2 工具类

```java
import java.io.File;
import java.net.URL;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 获取实体类,不是给Spring代理的第三方实现类
 *
 *
 * <p>
 *
 * @author hhp 2018年10月16日
 * @see
 * @since 1.0
 */
public class InstanceBuilder {

	private static final Logger logger = LoggerFactory.getLogger(InstanceBuilder.class);

	private static PluginClassLoader pluginClassLoader = null;
	static {
		initClassLoader();
	}

	/**
	 * 构造方法
	 *
	 * @param jarPath
	 *            jar文件路径,为绝对路径,如:<code>/home/dev/biplugin-jars/outside.jar</code>
	 * @param className
	 *            加载类名称,如<code>com.plugin.BiSourceImpl</code>
	 */
	public InstanceBuilder(String jarPath) {
		if (jarPath == null || "".equals(jarPath.trim())) {
			logger.error("Don't mess me around,please set the path of jar");
			throw new RuntimeException("Don't mess me around,please set the path of jar");
		}

		if (pluginClassLoader == null) {
			initClassLoader();
		}
		File file = new File(jarPath);
		if (!isLegalJarFile(file)) {
			throw new RuntimeException("Jar[" + jarPath + "] isn't a legal jar file.");
		}

		/*
		 * 加载用户jar
		 */
		try {
			pluginClassLoader.addJar(file.toURI().toURL());
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 构造PluginClassLoader
	 */
	public static void initClassLoader() {
		URL[] urls = new URL[]{};
		pluginClassLoader = new PluginClassLoader(urls);
	}

	/**
	 * 判断Jar文件是否合法
	 *
	 * @return boolean
	 */
	private static String suffix = ".jar";
	private boolean isLegalJarFile(File file) {
		if (!file.exists()) {
			logger.error("Jar file doesn't exists on:{}", file.getPath());
			return false;
		}
		if (!file.isFile()) {
			logger.error("Path {} is a dir not a file", file.getPath());
			return false;
		}

		if (!file.getName().endsWith(suffix)) {
			logger.error("File {} is not a jar", file.getPath());
			return false;
		}
		return true;
	}

	/**
	 * 创建实体类
	 *
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public <T> T get(String className) {
		try {
			Class<?> clazz = pluginClassLoader.loadClass(className);
			return (T) clazz.newInstance();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}
}
```

### 5.3 测试类

```java
import org.junit.Test;

import cn.rojao.bi.plugins.func.LogWorker;
import cn.rojao.bi.puglins.invoke.loader.InstanceBuilder;

public class TestJar {

	@Test
	public void testName() throws Exception {
		String userJarPath = "D:/Mydoc/biplugin/bi-plugins-impl-0.0.1-SNAPSHOT-jar-with-dependencies.jar";
		String logWorkerImpl = "cn.rojao.bi.plugins.impl.NetLogWorkerImpl";
		String logWorkerImplConfigPath = "D:/Mydoc/biplugin/impl.properties";

		InstanceBuilder builder = new InstanceBuilder(userJarPath);
		LogWorker impl = builder.get(logWorkerImpl);

		impl.process(logWorkerImplConfigPath);
	}
}
```

---

## 6.参考资料

a. [classloader 原理](http://blog.csdn.net/xyang81/article/details/7292380)
