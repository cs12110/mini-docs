# Spark ELT

在 Spark 里面时常需要把数据库的数据搬到 hive.

So,我们在这里做一个复制 mysql 数据数据到 hive 的转换任务.


相关依赖和打包设置:[link](spark-deps.md)

---

## 1. 主要配置类

### 1.1 配置类

```java
import cn.rojao.engine.anno.AliasAnno;
import com.alibaba.fastjson.JSON;
import lombok.Data;

import java.io.Serializable;

/**
 * 基础配置
 * <p/>
 *
 * @author cs12110 created at: 2018/12/25 17:53
 * <p>
 * since: 1.0.0
 */
@Data
@AliasAnno("base")
public class BaseConfig implements Serializable {

    /**
     *
     */
    private static final long serialVersionUID = 1L;

    /**
     * 应用名称
     */
    private String appName;

    /**
     * Spark master url
     */
    private String sparkMaster;

    /**
     * Hive warehouse
     */
    private String hiveWarehouse;

    /**
     * Hive database
     */
    private String hiveDatabase;

    /**
     * redis集群
     */
    private String redisCluster;

    /**
     * redis密码
     */
    private String redisAuth;

    /**
     * hdfs目录
     */
    private String hdfs;

    /**
     * 复写,默认为true
     */
    private String dataOverwrite = "true";


    @Override
    public String toString() {
        return JSON.toJSONString(this, true);
    }
}
```

```java
package cn.rojao.engine.conf.etl;

import cn.rojao.engine.conf.BaseConfig;
import com.alibaba.fastjson.JSON;
import lombok.Data;

/**
 * Etl转换配置
 * <p/>
 *
 * @author cs12110 created at: 2018/12/26 8:04
 * <p>
 * since: 1.0.0
 */
@Data
public class EtlConfig extends BaseConfig {

    private static final long serialVersionUID = 1L;

    private String dbDriver;
    private String dbUrl;
    private String dbUser;
    private String dbPassword;
    private String dbTables;


    @Override
    public String toString() {
        return JSON.toJSONString(this, true);
    }
}
```

### 1.2 读取配置文件工具类

```java
package cn.rojao.engine.util;

import cn.rojao.engine.conf.BaseConfig;

import java.lang.reflect.Field;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

/**
 * 配置工厂类
 * <p/>
 *
 * @author cs12110 created at: 2018/12/26 10:10
 * <p>
 * since: 1.0.0
 */
public class ConfFactory {

    /**
     * appName function
     */
    private static Function<String, String> appNameFunction = (e) -> {
        if (null == e || "".equals(e.trim())) {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmmss");
            return "app-" + sdf.format(new Date());
        }
        return e;
    };

    /**
     * 构建配置类
     *
     * @param configClass 配置类class
     * @param appName     app名称
     * @param path        配置文件路径
     * @return T which T extends {@link BaseConfig}
     */
    public static <T extends BaseConfig> T build(Class<T> configClass, String appName, String path) {
        T instance = null;
        try {
            instance = configClass.newInstance();
            instance.setAppName(appNameFunction.apply(appName));

            //获取配置文件工具类
            PropertiesUtil propertiesUtil = new PropertiesUtil(path);

            //装载配置文件信息到配置类
            wrapper(propertiesUtil.getCacheMap(), instance);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return instance;
    }

    /**
     * 装载属性值到实体类
     *
     * @param map    map
     * @param entity 实体类
     * @param <T>    T
     */
    private static <T> void wrapper(Map<Object, Object> map, T entity) {
        if (null == entity || null == map) {
            return;
        }
        Class<?> clazz = entity.getClass();
        try {
            for (Field f : getFieldOfClass(clazz)) {
                String key = fixFieldName(f.getName());
                Object value = map.get(key);
                if (value != null) {
                    f.set(entity, value);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取clazz的所有的字段
     *
     * @param clazz class
     * @return List
     */
    private static List<Field> getFieldOfClass(Class<?> clazz) {
        List<Field> fields = new ArrayList<>();
        while (clazz != Object.class) {
            Field[] fieldArr = clazz.getDeclaredFields();
            for (Field f : fieldArr) {
                f.setAccessible(true);
                fields.add(f);
            }
            clazz = clazz.getSuperclass();
        }
        return fields;
    }

    /**
     * 将字段属性值转换成配置文件的key,转换规则为: aAbB(驼峰) -> a.ab.b
     *
     * @param fieldName 字段名称
     * @return String
     */
    private static String fixFieldName(String fieldName) {
        StringBuilder builder = new StringBuilder();
        for (char ch : fieldName.toCharArray()) {
            if (ch >= 'A' && ch <= 'Z') {
                ch += 32;
                builder.append(".");
                builder.append(ch);
            } else {
                builder.append(ch);
            }
        }
        return builder.toString();
    }
}
```

---

## 2. ETL 转换类

```java
package cn.rojao.engine.etl;

import cn.rojao.engine.conf.etl.EtlConfig;
import cn.rojao.engine.util.ConfFactory;
import cn.rojao.engine.util.LogUtil;
import cn.rojao.engine.util.SparkUtil;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SaveMode;
import org.apache.spark.sql.SparkSession;

import java.util.Properties;

/**
 * 将mysql数据的数据导入hive
 * <p/>
 *
 * @author cs12110 created at: 2018/12/25 17:08
 * <p>
 * since: 1.0.0
 */
public class Mysql2Hive {

    /**
     * 多个表之间分隔符
     */
    private static final String TABLE_SPLIT = ",";

    public static void main(String[] args) {
        if (null == args) {
            throw new RuntimeException("You must be set config on the first parameter for " + Mysql2Hive.class);
        }

        LogUtil.display("using: ", args[0]);

        //获取配置文件内容
        EtlConfig config = buildConfig(args[0]);
        LogUtil.display(config.toString());

        workout(config);
    }

    /**
     * 获取配置信息对象
     *
     * @param path 配置文件路径
     * @return EtlConfig
     */
    private static EtlConfig buildConfig(String path) {
        return ConfFactory.build(EtlConfig.class, Mysql2Hive.class.getName(), path);
    }

    /**
     * 转换数据
     *
     * @param config Etl配置
     */
    private static void workout(EtlConfig config) {
        //切换数据库
        SparkSession session = SparkUtil.openSession(config);
        try {
            session.sql("use " + config.getHiveDatabase());

            //配置数据库连接信息
            Properties mysqlConfig = new Properties();
            mysqlConfig.setProperty("driver", config.getDbDriver());
            mysqlConfig.setProperty("user", config.getDbUser());
            mysqlConfig.setProperty("password", config.getDbPassword());

            for (String table : config.getDbTables().split(TABLE_SPLIT)) {
                LogUtil.display("Start translate {} into hive", table);

                //存放到hive
                String tableName = table.trim();
                Dataset<Row> tableValue = session.read().jdbc(config.getDbUrl(), tableName, mysqlConfig);

                SaveMode sm = SaveMode.Overwrite;
                if (!"true".equals(config.getDataOverwrite())) {
                    sm = SaveMode.Append;
                }
                tableValue.write().mode(sm).saveAsTable(tableName);
            }
        } catch (Exception e) {
            LogUtil.display("Got an exception:{}", e.getMessage());
        } finally {
            SparkUtil.closeSession(session);
        }
    }
}
```

---

## 3. 运行测试

### 3.1 配置文件内容

```properties
# Spark
spark.master=spark://10.10.1.141:7077,10.10.1.142:7077

# Hive
hive.warehouse=hdfs://10.10.1.142:18001/user/hive/warehouse
hive.database=ups_db

# Mysql
db.driver=com.mysql.jdbc.Driver
db.url=jdbc:mysql://10.10.2.233/ups_web
db.user=root
db.password=xyz@123
db.tables=internet_video,tag_internetvideo_info,tag_internetvideo_map
```

### 3.2 运行脚本

```java

#!/bin/sh

use_memory='8G'
spark_location='spark://10.10.1.141:7077,10.10.1.142:7077'

task_app_prefix='/opt/bi/spark/ups-task/'

executor_conf=$task_app_prefix'/setting/etl-mysql.properties'
executor_class='cn.rojao.engine.etl.Mysql2Hive'
executor_app=$task_app_prefix'/app/hvs-engine-0.0.1-SNAPSHOT.jar'

echo -e  '\n\nStartup spark task\n\n'

spark-submit --master ${spark_location} \--executor-memory ${use_memory} \
--class ${executor_class} ${executor_app}  ${executor_conf} 
```