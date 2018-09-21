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

## 5.参考资料

a. [classloader 原理](http://blog.csdn.net/xyang81/article/details/7292380)
