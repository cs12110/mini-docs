# Spring

各种 Spring 里面用到的玩意.

---

## 1. PropertiesEditor

### 1.1 基础知识

在 Spring 里面定义一个 `RequestMapping` 的接口: `public Object info(Integer id) { ... }`,前台传递的参数从 `HttpServletRequest` 里面获取出来的都是字符串类型的.

Q: 怎么做到类型转换的呢?

A: Spring 里面使用了 Java 里面的一个类`PropertiesEditor`来做简单的数据兑换.实现类在:`org.springframework.beans.propertyeditors`包里面.

### 1.2 源码

我们看一下 Spring 里面的经典数字数据转换

`PropertyEditorSupport` 实现`java.beans.PropertyEditor`接口,并使用模板模式设计.

```java
public class PropertyEditorSupport implements PropertyEditor{
    // ...
}
```

数字转换实现类

```java
package org.springframework.beans.propertyeditors;

import java.beans.PropertyEditorSupport;
import java.text.NumberFormat;

import org.springframework.lang.Nullable;
import org.springframework.util.NumberUtils;
import org.springframework.util.StringUtils;

public class CustomNumberEditor extends PropertyEditorSupport {

	private final Class<? extends Number> numberClass;

	@Nullable
	private final NumberFormat numberFormat;

	private final boolean allowEmpty;


	public CustomNumberEditor(Class<? extends Number> numberClass, boolean allowEmpty) throws IllegalArgumentException {
		this(numberClass, null, allowEmpty);
	}

	public CustomNumberEditor(Class<? extends Number> numberClass,
			@Nullable NumberFormat numberFormat, boolean allowEmpty) throws IllegalArgumentException {

		if (!Number.class.isAssignableFrom(numberClass)) {
			throw new IllegalArgumentException("Property class must be a subclass of Number");
		}
		this.numberClass = numberClass;
		this.numberFormat = numberFormat;
		this.allowEmpty = allowEmpty;
	}

	@Override
	public void setAsText(String text) throws IllegalArgumentException {
		if (this.allowEmpty && !StringUtils.hasText(text)) {
			// Treat empty String as null value.
			setValue(null);
		}
		else if (this.numberFormat != null) {
			// Use given NumberFormat for parsing text.
			setValue(NumberUtils.parseNumber(text, this.numberClass, this.numberFormat));
		}
		else {
			// Use default valueOf methods for parsing text.
			setValue(NumberUtils.parseNumber(text, this.numberClass));
		}
	}

	@Override
	public void setValue(@Nullable Object value) {
		if (value instanceof Number) {
			super.setValue(NumberUtils.convertNumberToTargetClass((Number) value, this.numberClass));
		}
		else {
			super.setValue(value);
		}
	}

	@Override
	public String getAsText() {
		Object value = getValue();
		if (value == null) {
			return "";
		}
		if (this.numberFormat != null) {
			// Use NumberFormat for rendering value.
			return this.numberFormat.format(value);
		}
		else {
			// Use toString method for rendering value.
			return value.toString();
		}
	}
}
```

### 1.2 测试代码

```java
public class EditorTest {

    @Test
    public void test() {
        PropertyEditorSupport propertiesEditor = new CustomNumberEditor(Integer.class, false);
        propertiesEditor.setAsText("123");
        Object value = propertiesEditor.getValue();
        System.out.println(value.getClass() + ":" + value);
    }
}
```

测试结果

```java
class java.lang.Integer:123
```
