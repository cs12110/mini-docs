# Easy-IOC

本文档主要用于实现一个最简单的 IOC,请知悉.

**大神请绕道,大神请绕道,大神请绕道**

FYI(for your information): `Pathfinder`,探索者,火星救援里面 `Watney` 找到的那个探测器.

---

## 1. PathfinderUtil

该工具类用于查找`packageName`下面的所有`class`.

```java
import java.io.File;
import java.net.URL;
import java.net.URLDecoder;
import java.util.List;

/**
 * 探索者
 *
 *
 * <p>
 *
 * @author hhp 2018年10月16日
 * @see
 * @since 1.0
 */
public class PathfinderUtil {

	private static final ClassLoader APP_CLASSLOADER = PathfinderUtil.class.getClassLoader();

	/**
	 * 获取App classloader
	 *
	 * @return {@link ClassLoader}
	 */
	public static ClassLoader getAppClassLoader() {
		return APP_CLASSLOADER;
	}

	/**
	 * 获取package下面的所有class,包括子package
	 *
	 * @param store
	 *            存储列表
	 * @param packageName
	 *            package名称,如<code>com.api.service</code>,且必须为package名称
	 */
	public static void findout(List<Class<?>> store, final String packageName) {
		String filePath = packageName.replaceAll("\\.", "/");
		URL resource = APP_CLASSLOADER.getResource(filePath);
		// 资源不存在,或者直接是文件的时候
		if (null == resource) {
			return;
		}

		try {
			// 应对中文编码
			String path = URLDecoder.decode(resource.getFile(), "UTF-8");
			File file = new File(path);
			for (File f : file.listFiles()) {
				String fileName = f.getName();
				if (f.isDirectory()) {
					String subPkgName = packageName + "." + fileName;
					findout(store, subPkgName);
				} else {
					if (fileName.endsWith(".class")) {
						int bound = fileName.length() - 6;
						String removeSubffixName = fileName.substring(0, bound);
						String className = packageName + "." + removeSubffixName;

						store.add(loadClass(className));
					}
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 加载class
	 *
	 * @param clazzName
	 *            class名称,例如: <code>com.api.ClassUtil</code>
	 * @return Class<?>
	 */
	public static Class<?> loadClass(String clazzName) {
		Class<?> targetClass = null;
		try {
			// 使用classLoader加载
			targetClass = APP_CLASSLOADER.loadClass(clazzName);
		} catch (Exception e) {
			e.printStackTrace();
		}
		return targetClass;
	}
}
```

---

## 2. IocStore

Ioc 集中地,主要用于实例化被扫描包里面的所有 class 和初始化 class 里面的所有要求注入的数据.

```java
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import pkg.ioc.util.PathfinderUtil;

/**
 * Ioc集合
 *
 *
 * <p>
 *
 * @author hhp 2018年10月16日
 * @see
 * @since 1.0
 */
public class IocStore {

	/**
	 * 扫描包名称
	 */
	private static String packageName = null;
	/**
	 * 是否设置成功
	 */
	private static boolean isSetting = false;

	/**
	 * 所有类列表
	 */
	private static List<Class<?>> classList = new ArrayList<Class<?>>();

	/**
	 * IOC
	 */
	private static Map<String, Object> iocMap = new HashMap<String, Object>();

	/**
	 * 初始化参数
	 *
	 * @param pkgName
	 *            扫描包名称
	 */
	public static void before(String pkgName) {
		if (null == pkgName) {
			return;
		}
		packageName = pkgName.trim();
		try {
			doScanner();
			doInstance();
			doInject();
			isSetting = true;
		} catch (Exception e) {
			e.printStackTrace();
			isSetting = false;
		}
	}

	/**
	 * 初始化扫描包里面的所有类
	 */
	private static void doScanner() {
		PathfinderUtil.findout(classList, packageName);
	}

	/**
	 * 初始化扫描包里面的所有类
	 *
	 * @throws IllegalAccessException
	 * @throws InstantiationException
	 */
	private static void doInstance() throws InstantiationException, IllegalAccessException {
		for (Class<?> e : classList) {
			if (!e.isInterface()) {
				Object instance = e.newInstance();
				Class<?>[] interfaces = e.getInterfaces();
				for (Class<?> intfc : interfaces) {
					iocMap.put(intfc.getName(), instance);
				}
				iocMap.put(e.getName(), instance);
			}
		}
	}

	/**
	 * 注入
	 *
	 * @throws IllegalAccessException
	 * @throws IllegalArgumentException
	 */
	private static void doInject() throws IllegalArgumentException, IllegalAccessException {
		for (Class<?> e : classList) {
			Object target = iocMap.get(e.getName());
			for (Field f : e.getDeclaredFields()) {
				f.setAccessible(true);
				Object value = iocMap.get(f.getType().getName());
				if (value != null) {
					f.set(target, value);
				}
			}
		}
	}

	/**
	 * 获取类
	 *
	 * @param clazz
	 *            类的class
	 * @return T
	 */
	@SuppressWarnings("unchecked")
	public static <T> T geatBean(Class<T> clazz) {
		if (!isSetting) {
			throw new RuntimeException("Using IocStore#before() setting scanner package first,please");
		}
		return (T) iocMap.get(clazz.getName());
	}
}
```

---

## 3. 测试

模拟最简单的 mvc 模型来测试.

### 3.1 Service

```java
public interface Service {
	public String say(String something);
}
```

```java
public class ServiceImpl implements Service {
	@Override
	public String say(String something) {
		return this.getClass() + " say: " + something;
	}
}
```

### 3.2 Controller

```java
public class MyCtrl {

	private Service service;

	public void say(String s) {
		System.out.println(service.say(s));
	}
}
```

### 3.3 IocTest

```java
public class IocTest {

	public static void main(String[] args) {
		IocStore.before("pkg.ioc");

		MyCtrl ctrl = IocStore.geatBean(MyCtrl.class);
		ctrl.say("12345");
	}
}
```

测试结果

```java
class pkg.ioc.test.service.impl.ServiceImpl say: 12345
```

---

## 4. 总结

- 没有各种自定义注解来区别 service,controller 什么的.
- 注入就是要注入,你 new 都没用. 笑哭脸.jpg
- 写代码呢,最紧要就是开心.

---

## 5. 参考资料

a. [子敬的项目](https://gitee.com/kwilove/java_learning)
