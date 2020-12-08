# Json 序列化

在现实环境里面,前后端交互或者存储复杂的数据格式,json 无疑是一个通用的选择.

Q: 如果要将对象转换成 Json,要用什么东西呀?

A: 有很多呀,例如 gson,fastjson,jackson.

那么我们在这里看看常用的 fastjson 和 jackson 的使用.

---

### 1. fastjson

fastjson 操作起来超简单的.

#### 1.1 依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.73</version>
</dependency>
```

#### 1.2 案例

```java
import com.alibaba.fastjson.JSON;

import java.util.List;

/**
 * fastjson工具类
 *
 * @author cs12110
 * @version V1.0
 * @since 2020-12-08 10:31
 */
public class FastJsonUtil {

    /**
     * 转换成json字符串
     *
     * @param value value
     * @return String
     */
    public static String toJsonStr(Object value) {
        return toJsonStr(value, false);
    }

    /**
     * 转换成字符串
     *
     * @param value    对象
     * @param beautify json字符串是否格式化
     * @return String
     */
    public static String toJsonStr(Object value, boolean beautify) {
        return JSON.toJSONString(value, beautify);
    }

    /**
     * json字符串转换成对象
     *
     * @param json  json字符串
     * @param clazz 对象class
     * @param <T>   类型
     * @return T
     */
    public static <T> T toObject(String json, Class<T> clazz) {
        return JSON.parseObject(json, clazz);
    }

    /**
     * 转换成List
     *
     * @param json  json
     * @param clazz clazz
     * @param <T>   T
     * @return List
     */
    public static <T> List<T> toList(String json, Class<T> clazz) {
        return JSON.parseArray(json, clazz);
    }
}
```

#### 1.3 坑

Issue: [fastjson 的 Long 类型数据精度丢失](https://my.oschina.net/simpleton/blog/4257114)

---

### 2. jackson

因为 fastjson 存在精度丢失的坑,而且 springboot 默认集成了 jackson,所以 jackson 也是不错的一个选择.

#### 2.1 依赖

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.11.0</version>
</dependency>
```

#### 2.2 案例

```java
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JavaType;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.databind.ser.std.ToStringSerializer;

import java.util.Collections;
import java.util.List;

/**
 * jackson 工具类
 *
 * @author cs12110
 * @version V1.0
 * @since 2020-12-08 10:44
 */
public class JacksonUtil {

    private static ObjectMapper objectMapper = new ObjectMapper();

    static {
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        objectMapper.configure(JsonParser.Feature.ALLOW_COMMENTS, true);

        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(Long.class, new ToStringSerializer());
        simpleModule.addSerializer(Long.TYPE, new ToStringSerializer());

        objectMapper.registerModule(simpleModule);
    }

    /**
     * 转换成json字符串
     *
     * @param value 对象/List
     * @return String
     */
    public static String toJsonStr(Object value) {
        try {
            return objectMapper.writeValueAsString(value);
        } catch (Exception e) {
            throw new RuntimeException("");
        }
    }

    /**
     * 转换成对象
     *
     * @param json  json字符串
     * @param clazz 对象类型
     * @return T
     */
    public static <T> T toObject(String json, Class<T> clazz) {
        try {
            return objectMapper.readValue(json, clazz);
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 转换成列表
     *
     * @param json  json字符串
     * @param clazz 对象class
     * @return List
     */
    public static <T> List<T> toList(String json, Class<T> clazz) {
        try {
            JavaType javaType = objectMapper.getTypeFactory().constructParametricType(List.class, clazz);
            return objectMapper.readValue(json, javaType);
        } catch (Exception e) {
            e.printStackTrace();
            return Collections.emptyList();
        }
    }
}
```

---

### 3. 参考资料

a. [fastjson github link](https://github.com/alibaba/fastjson)

b. [jackson github link](https://github.com/FasterXML/jackson)
