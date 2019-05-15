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

---

## 2. Spring 初始化过程

[优秀博客 link](https://fangjian0423.github.io/)

---

## 3. DispatchServlet

在 spring mvc 的请求响应过程里面,`DispatchServlet`可是第一主角.

### 3.1 继承情况

首先 DispatchServlet 最根本还是一个 Servlet, 只不过 Spring 在 Serlvet 进行了封装而已,例如 RequestMapping.

```java
public class DispatcherServlet extends FrameworkServlet {
	...
}
```

```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
	...
}
```

```java
/**
 * HttpServlet来自: package javax.servlet.http;
 */
public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
	...
}
```

### 3.2 重要方法

```java
/**
* Process the actual dispatching to the handler.
* <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
* The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
* to find the first that supports the handler class.
* <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
* themselves to decide which methods are acceptable.
* @param request current HTTP request
* @param response current HTTP response
* @throws Exception in case of any kind of processing failure
*/
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
HttpServletRequest processedRequest = request;
HandlerExecutionChain mappedHandler = null;
boolean multipartRequestParsed = false;

WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

try {
	ModelAndView mv = null;
	Exception dispatchException = null;

	try {
		processedRequest = checkMultipart(request);
		multipartRequestParsed = (processedRequest != request);

		// Determine handler for the current request.
		mappedHandler = getHandler(processedRequest);
		if (mappedHandler == null) {
			noHandlerFound(processedRequest, response);
			return;
		}

		// Determine handler adapter for the current request.
		HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

		// Process last-modified header, if supported by the handler.
		String method = request.getMethod();
		boolean isGet = "GET".equals(method);
		if (isGet || "HEAD".equals(method)) {
			long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
			if (logger.isDebugEnabled()) {
				logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
			}
			if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
				return;
			}
		}

		if (!mappedHandler.applyPreHandle(processedRequest, response)) {
			return;
		}

		// Actually invoke the handler.
		mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

		if (asyncManager.isConcurrentHandlingStarted()) {
			return;
		}

		applyDefaultViewName(processedRequest, mv);
		mappedHandler.applyPostHandle(processedRequest, response, mv);
	}
	catch (Exception ex) {
		dispatchException = ex;
	}
	catch (Throwable err) {
		// As of 4.3, we're processing Errors thrown from handler methods as well,
		// making them available for @ExceptionHandler methods and other scenarios.
		dispatchException = new NestedServletException("Handler dispatch failed", err);
	}
	processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
catch (Exception ex) {
	triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
}
catch (Throwable err) {
	triggerAfterCompletion(processedRequest, response, mappedHandler,
			new NestedServletException("Handler processing failed", err));
}
finally {
	if (asyncManager.isConcurrentHandlingStarted()) {
		// Instead of postHandle and afterCompletion
		if (mappedHandler != null) {
			mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
		}
	}
	else {
		// Clean up any resources used by a multipart request.
		if (multipartRequestParsed) {
			cleanupMultipart(processedRequest);
		}
	}
}
}
```