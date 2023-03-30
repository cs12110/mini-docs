# Java 枚举

枚举常用使用案例.

---

### 1. 工具类

```java
import io.swagger.annotations.ApiModelProperty;
import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-03-27 22:02
 */
@Getter
@AllArgsConstructor
public enum GenderEnum {
    MALE(1, "男"),
    FEMALE(2, "女"),
    UNKNOWN(0, "不详");

    @ApiModelProperty(value = "值", required = true)
    private final Integer value;
    @ApiModelProperty(value = "标签", required = true)
    private final String label;
}

```
```java
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-03-27 21:59
 */
public class FixedEnumUtil {

    /**
     * getter名称
     */
    private static final String LABEL_GETTER = "getLabel";
    private static final String VALUE_GETTER = "getValue";
    private static final String NAME_GETTER = "name";

    /**
     * 判断desc是不是枚举里面的一种
     *
     * @param enumClass 枚举类
     * @param descVal   描述值
     * @return boolean
     * @throws RuntimeException 如果有任何异常
     */
    public static boolean contains(Class<?> enumClass, String descVal) {
        try {
            // 获取字段的枚举类型
            Object[] enumValArr = enumClass.getEnumConstants();
            // 获取枚举的描述
            Method labelGetterMth = enumClass.getDeclaredMethod(LABEL_GETTER);
            Object matchEnumVal = null;
            for (Object each : enumValArr) {
                Object enumDescVal = labelGetterMth.invoke(each);
                if (Objects.equals(enumDescVal, descVal)) {
                    matchEnumVal = each;
                    break;
                }
            }
            return Objects.nonNull(matchEnumVal);
        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }


    /**
     * 判断desc是不是枚举里面的一种
     *
     * @param enumClass 枚举类
     * @return String
     * @throws RuntimeException 如果有任何异常
     */
    public static String labels(Class<?> enumClass) {
        try {
            // 获取字段的枚举类型
            Object[] enumValArr = enumClass.getEnumConstants();
            // 获取枚举的描述
            Method labelGetterMth = enumClass.getDeclaredMethod(LABEL_GETTER);
            StringBuilder label = new StringBuilder();
            label.append("[");
            for (int index = 0, len = enumValArr.length; index < len; index++) {
                Object enumDescVal = labelGetterMth.invoke(enumValArr[index]);
                label.append(enumDescVal);
                if (index < len - 1) {
                    label.append(",");
                }
            }
            label.append("]");
            return label.toString();
        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }

    /**
     * 获取枚举的desc列表
     *
     * @param enumClass 枚举class
     * @return List
     */
    public static List<String> getLabelList(Class<?> enumClass) {
        try {
            // 获取字段的枚举类型
            Object[] enumValArr = enumClass.getEnumConstants();
            // 获取枚举的描述
            Method labelGetterMth = enumClass.getDeclaredMethod(LABEL_GETTER);

            List<String> descList = new ArrayList<>();
            for (int index = 0, len = enumValArr.length; index < len; index++) {
                Object enumDescVal = labelGetterMth.invoke(enumValArr[index]);
                descList.add(String.valueOf(enumDescVal));
            }
            return descList;
        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }


    /**
     * 获取枚举的描述desc
     *
     * @param targetVal excel值
     * @param field     field字段
     * @return String
     * @throws RuntimeException 如果有任何异常
     */
    public static String getLabelFromEnum(Object targetVal, Field field) {
        try {
            // 判断是否枚举
            if (!(targetVal instanceof Enum<?>)) {
                return null;
            }
            // 获取字段的枚举类型
            Class<?> enumClazzVal = field.getType();
            Object[] enumValArr = enumClazzVal.getEnumConstants();

            Object matchVal = null;
            for (Object each : enumValArr) {
                if (Objects.equals(each, targetVal)) {
                    matchVal = each;
                    break;
                }
            }

            // 获取枚举的描述
            Method labelGetterMth = enumClazzVal.getDeclaredMethod(LABEL_GETTER);
            Object descVal = labelGetterMth.invoke(matchVal);

            return String.valueOf(descVal);

        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }

    /**
     * 从desc匹配枚举
     *
     * @param label     描述值
     * @param enumClazz 枚举类
     * @return enum
     * @throws RuntimeException 如果有任何异常
     */
    public static <T> T getEnumFromLabel(String label, Class<T> enumClazz) {
        try {
            // 获取字段的枚举类型
            //Class<?> enumClazzVal = field.getType();
            Object[] enumValArr = enumClazz.getEnumConstants();

            // 获取枚举的描述
            Method labelGetterMth = enumClazz.getDeclaredMethod(LABEL_GETTER);
            Object matchEnumVal = null;
            for (Object each : enumValArr) {
                Object enumDescVal = labelGetterMth.invoke(each);
                if (Objects.equals(enumDescVal, label)) {
                    matchEnumVal = each;
                    break;
                }
            }

            return (T) matchEnumVal;
        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }

    @Data
    public static class EnumItem {
        @ApiModelProperty("value")
        private String value;
        @ApiModelProperty("label")
        private String label;
    }

    /**
     * 转换成map
     *
     * @param enumClazz 枚举类
     * @return Map
     */
    public static <T> Map<String, EnumItem> parse2Map(Class<T> enumClazz) {
        try {
            T[] enumArr = enumClazz.getEnumConstants();
            Method labelGetterMth = enumClazz.getDeclaredMethod(LABEL_GETTER);
            Method valueGetterMth = enumClazz.getDeclaredMethod(VALUE_GETTER);
            Method nameGetterMth = enumClazz.getMethod(NAME_GETTER);

            Map<String, EnumItem> map = new HashMap<>();
            for (T each : enumArr) {
                Object name = nameGetterMth.invoke(each);
                Object label = labelGetterMth.invoke(each);
                Object value = valueGetterMth.invoke(each);

                EnumItem enumItem = new EnumItem();
                enumItem.setValue(String.valueOf(value));
                enumItem.setLabel(String.valueOf(label));

                map.put(String.valueOf(name), enumItem);
            }

            return map;
        } catch (Exception exp) {
            throw new RuntimeException(exp);
        }
    }
}
```

测试使用:

```java
import com.alibaba.fastjson.JSON;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-03-27 22:35
 */
public class FixedEnumTest {

    public static void main(String[] args) {
        System.out.println(FixedEnumUtil.contains(GenderEnum.class, "男"));
        System.out.println(FixedEnumUtil.contains(GenderEnum.class, "洒洒水啦"));

        System.out.println();
        System.out.println(FixedEnumUtil.labels(GenderEnum.class));

        System.out.println();
        System.out.println(FixedEnumUtil.getLabelList(GenderEnum.class));

        System.out.println();
        System.out.println(FixedEnumUtil.getEnumFromLabel("女", GenderEnum.class));

        System.out.println();
        System.out.println(JSON.toJSONString(FixedEnumUtil.parse2Map(GenderEnum.class), Boolean.TRUE));
    }
}
```

```
true
false

[男,女,不详]

[男, 女, 不详]

FEMALE

{
	"UNKNOWN":{
		"label":"不详",
		"value":"0"
	},
	"MALE":{
		"label":"男",
		"value":"1"
	},
	"FEMALE":{
		"label":"女",
		"value":"2"
	}
}
```

---

### 2. EasyExcel 转换枚举

```java
package com.pkgs.fd.model.dto.fieldcovert;

import com.alibaba.excel.converters.Converter;
import com.alibaba.excel.metadata.GlobalConfiguration;
import com.alibaba.excel.metadata.data.ReadCellData;
import com.alibaba.excel.metadata.data.WriteCellData;
import com.alibaba.excel.metadata.property.ExcelContentProperty;
import com.pkgs.fd.utils.FixedEnumUtil;
import lombok.extern.slf4j.Slf4j;

import java.lang.reflect.Field;
import java.util.Objects;

/**
 * 枚举转换类
 * <p>
 * 枚举类型包含label,value字段,必须有getLabel方法
 *
 * @author cs12110
 * @since v1.0.0
 */
public class FixedEnumConverter implements Converter<Object> {
    /**
     * 根据desc返回enum
     */
    @Override
    public Object convertToJavaData(
            ReadCellData<?> cellData,
            ExcelContentProperty contentProperty,
            GlobalConfiguration globalConfiguration
    ) throws Exception {
        // 枚举的desc值
        String descVal = (String) cellData.getData();
        Field field = contentProperty.getField();
        return FixedEnumUtil.getEnumFromLabel(descVal, field.getType());
    }

    /**
     * 根据enum返回desc
     */
    @Override
    public WriteCellData<String> convertToExcelData(
            Object realEnumVal,
            ExcelContentProperty contentProperty,
            GlobalConfiguration globalConfiguration
    ) throws Exception {
        // 根据realEnumVal获取实际的枚举,根据实际的枚举,获取具体的desc的值
        String descVal = FixedEnumUtil.getLabelFromEnum(realEnumVal, contentProperty.getField());
        return new WriteCellData<>(Objects.isNull(descVal) ? "" : descVal);
    }
}
```

---
