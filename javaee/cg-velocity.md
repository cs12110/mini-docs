# Java 之代码生成器

在生产开发中,在 mvc 模式下面,经常是增删改查复制来,复制去.

如果可以使用模板来生成相关代码呢? 这不是很好吗?

那么 velocity, 你值得拥有. [IBM 教程 link](https://www.ibm.com/developerworks/cn/java/j-lo-velocity1/index.html)

---

## 1. Helloword

万事开头 hello world.

### 1.1 pom.xml

```xml
<!-- 代码生成器依赖 -->
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>2.0</version>
</dependency>

<!-- 添加fastjson -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.30</version>
</dependency>
```

### 1.2 Hellovelocity.vm

注意编写格式: `#`后面不带空格.

虽然可以使用`$variable`定义变量,但推荐使用:`${variable}`来定义变量,因为`${}`可以支持更多的数据操作.

```js
#set (${iAmVariable} = "good!")

Welcome $name to velocity.com

#foreach ($i in $list)
$i
#end

${iAmVariable}
```

### 1.3 测试代码

```java
package com.pkgs.gen;

import java.io.IOException;
import java.io.StringWriter;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

import org.apache.velocity.Template;
import org.apache.velocity.VelocityContext;
import org.apache.velocity.app.VelocityEngine;
import org.apache.velocity.runtime.RuntimeConstants;
import org.apache.velocity.runtime.resource.loader.ClasspathResourceLoader;

/**
 * 代码生成器
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月14日下午1:29:20
 * @see
 * @since 1.0
 */
public class HelloVelocity {

	public static void main(String[] args) throws IOException {
		VelocityEngine engine = new VelocityEngine();
		engine.setProperty(RuntimeConstants.RESOURCE_LOADER, "classpath");
		engine.setProperty("classpath.resource.loader.class", ClasspathResourceLoader.class.getName());

		engine.init();

		// Hellovelocity文件放置位置
		Template template = engine.getTemplate("genbody/Hellovelocity.vm");
		VelocityContext ctx = new VelocityContext();

		// 给模板的name属性字段赋值
		ctx.put("name", "my-first-velocity");
		ctx.put("date", (new Date()).toString());

		// 给模板的list字段赋值
		List<Object> list = new ArrayList<Object>();
		list.add("1");
		list.add("2");

		ctx.put("list", list);

		StringWriter writer = new StringWriter();
		template.merge(ctx, writer);

		System.out.println(writer.toString());

		writer.close();
	}
}
```

### 1.4 测试结果

```java
Welcome my-first-velocity to velocity.com

1
2

good!
```

---

## 2. 基础应用

这里我们看一下 velocity 里面的基础语法.

### 2.1 定义变量

设置变量格式为: `$variableName` 或者 `${variableName}`.

```java
#set($name ="velocity")
```

相当于在 java 设置了变量: `String name="velocity";`

```java
#set($hello="hello $name")
```

这个等式会给`$hello` 赋值为: `hello velocity`

### 2.2 变量赋值

在 velocity 的模板里面的变量是弱类型的,类比如 js 的 var,赋值类型就自由了.

```java
#set($stu = $student)
#set($stu = "hello")
#set($stu.name = $student.name)
#set($stu.name = $student.getName($arg))
#set($stu = 3306)
#set($stu = ["foo",$strValue])
```

### 2.3 循环

当然少不了基础结构啦.

```java
#foreach($each in $list)
    System.out.println($each);
#end
```

### 2.4 if&else

```java
#if(condition)
...
#elseif(codniton)
...
#else
...
#end
```

### 2.5 关系操作符

关系操作: `AND,OR,NOT`分别对应 `&&, ||, !`

```java
#if($foo && $bar)
#end
```

### 2.6 方法定义

方法定义

```java
#marco(methodName arg1 arg2)
//do something
#end
```

调用方法

```java
#methodName(arg1 arg2)
```

完整示例

```java
#macro(say ${something})
    "say: " ${something}
#end

#say("haiyan")
```

### 2.7 #parse & #include

这两个指令的作用都是引入外部文件.

- #parse: 将引用内容当做类似源码文件,会将内容在引入的地方进行解释.
- #include: 将引入文件当做资源文件,会将内容原封不动的以文本输出.

`test.vm`文件

```java
#set(${name}="cs12110")
```

`ByParse.vm`

```java
#parse("test.vm")
```

输出结果为: "cs12110"

`ByInclude.vm`

```java
#include("test.vm")
```

输出结果为: `#set(#{name}="cs12110")`

---

## 3. 使用案例

作为简单测试,我们这里使用模板生成实体类即可.

### 3.1 创建模板

定义实体类模板

```java
package $pkgName;

import java.io.Serializable;
import com.alibaba.fastjson.JSON;


/**
 * ${comment.todo}
 *
 * <p>
 *
 * author: ${comment.user} at ${comment.time}
 *
 * since: ${comment.version}
 */
public class ${entityName} implements Serializable{
	
	private static final long serialVersionUID = 1L;
	
	#foreach($f in $fields)
	/**
	 * ${f.comment}
	 */
	private ${f.type} ${f.name}; 
	#end
	
	#foreach ($f in $fields)
	/**
	 * get: ${f.name}
	 */
	public $f.type get${f.name}() {
		return ${f.name};
	}
	
	/**
	 * set: ${f.name}
	 */
	public void set${f.name}(${f.type} ${f.name}) {
		this.${f.name} = ${f.name};
	}
	#end
	
	@Override
	public String toString() {
		return JSON.toJSONString(this,true);
	}
}
```

### 3.2 定义类的字段属性

```java
/**
 * 类属性实体类
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月14日下午3:08:27
 * @see
 * @since 1.0
 */
public class FieldAttr {
	/**
	 * 字段类型
	 */
	private String type;
	/**
	 * 字段名称
	 */
	private String name;

	/**
	 * 字段说明
	 */
	private String comment;

	public FieldAttr(String type, String name, String comment) {
		this.type = type;
		this.name = name;
		this.comment = comment;
	}

	public String getType() {
		return type;
	}

	public String getName() {
		return name;
	}

	public String getComment() {
		return comment;
	}
}
```

### 3.3 IO 工具类

```java
import java.io.File;
import java.io.FileOutputStream;
import java.io.OutputStream;

/**
 * 文件工具类
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月14日下午3:19:49
 * @see
 * @since 1.0
 */
public class FileUtil {
	/**
	 * 保存为文件
	 *
	 * @param path
	 *            文件路径
	 * @param content
	 *            内容名称
	 */
	public static void write(String path, String content) {
		OutputStream out = null;
		try {
			out = new FileOutputStream(new File(path));
			out.write(content.getBytes());
			out.flush();
			out.close();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			if (null != out) {
				try {
					out.close();
				} catch (Exception ex) {
				}
			}
		}

	}
}
```

### 3.4 启动入口

编写启动 main 类,内容如下

```java
import java.io.StringWriter;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.velocity.Template;
import org.apache.velocity.VelocityContext;
import org.apache.velocity.app.VelocityEngine;
import org.apache.velocity.runtime.RuntimeConstants;
import org.apache.velocity.runtime.resource.loader.ClasspathResourceLoader;

/**
 * 测试类
 *
 *
 *
 * <p>
 *
 * @author cs12110 2018年12月14日下午3:11:41
 * @see
 * @since 1.0
 */
public class EntityUtil {

	/**
	 * engine
	 */
	private static VelocityEngine engine = new VelocityEngine();

	public static void main(String[] args) {
		engine.setProperty(RuntimeConstants.RESOURCE_LOADER, "classpath");
		engine.setProperty("classpath.resource.loader.class", ClasspathResourceLoader.class.getName());
		engine.init();

		String content = buildStudentEntity();

		// 写出文件
		FileUtil.write("d://StudentEntity.java", content);
	}

	/**
	 * 创建学生类
	 */
	private static String buildStudentEntity() {
		VelocityContext ctx = new VelocityContext();

		// 设置package名称和类名称以及相关注释内容值
		ctx.put("pkgName", "com.pkgs.entity");
		ctx.put("entityName", "StudentEntity");
		ctx.put("comment", buildInfoMap());

		// 设置字段信息
		List<FieldAttr> fields = new ArrayList<>();
		fields.add(new FieldAttr("Integer", "id", "id"));
		fields.add(new FieldAttr("String", "name", "名称"));
		fields.add(new FieldAttr("Float", "score", "成绩"));
		ctx.put("fields", fields);

		// 获取模板,模板为:resource/genbody/目录下
		Template template = engine.getTemplate("genbody/Entity.vm");
		StringWriter writer = new StringWriter();
		template.merge(ctx, writer);

		String content = writer.toString();
		try {
			writer.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
		return content;
	}

	/**
	 * 创建注释信息map
	 *
	 * @return Map
	 */
	private static Map<String, String> buildInfoMap() {
		Map<String, String> map = new HashMap<String, String>();
		map.put("user", "cs12110");
		map.put("todo", "build entity");
		map.put("time", "2018-12-14");
		map.put("version", "1.0.0");

		return map;
	}
}
```

### 3.5 测试结果

在 D 盘生成了 StudentEntity 文件,拷贝入相关目录没有出现任何异常,测试成功.

```java
package com.pkgs.entity;

import java.io.Serializable;
import com.alibaba.fastjson.JSON;


/**
 * build entity
 *
 * <p>
 *
 * author: cs12110 at 2018-12-14
 *
 * since: 1.0.0
 */
public class StudentEntity implements Serializable{
	
	private static final long serialVersionUID = 1L;
	
	/**
	 * id
	 */
	private Integer id; 
	/**
	 * 名称
	 */
	private String name; 
	/**
	 * 成绩
	 */
	private Float score; 
	
	/**
	 * get: id
	 */
	public Integer getid() {
		return id;
	}
	
	/**
	 * set: id
	 */
	public void setid(Integer id) {
		this.id = id;
	}
	/**
	 * get: name
	 */
	public String getname() {
		return name;
	}
	
	/**
	 * set: name
	 */
	public void setname(String name) {
		this.name = name;
	}
	/**
	 * get: score
	 */
	public Float getscore() {
		return score;
	}
	
	/**
	 * set: score
	 */
	public void setscore(Float score) {
		this.score = score;
	}
	
	@Override
	public String toString() {
		return JSON.toJSONString(this,true);
	}
}
```

### 3.6 总结

可以看出使用模板生成还是挺快捷的,但前提是你要编写相关的模板文件,并且多次测试才能使用在现实生产环境中去.

---

## 4. 参考资料

a. [IBM 教程](https://www.ibm.com/developerworks/cn/java/j-lo-velocity1/index.html)
