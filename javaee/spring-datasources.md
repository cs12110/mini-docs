# spring 多数据源

在 2022-03-31 的时候,找到更方便的实现方式,请知悉(旧版本请参考第二章节内容).

FBI Warning: <span style='color:pink'>无论哪种方式,都会出现事务回滚的问题.</span>

---

## 1. dynamic-datasource

[官网地址 link](https://www.mybatis-plus.com/guide/dynamic-datasource.html)

### 1.1 依赖与配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>dynamic-dbs-project</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.7.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
            <version>8.0.22</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.0</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
            <version>3.5.0</version>
        </dependency>

        <!--数据库相关-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.2.3</version>
        </dependency>
    </dependencies>

</project>
```

```yaml
server:
  port: 8070
  servlet:
    context-path: /api/dynamic-dbs/

spring:
  application:
    name: dynamic-dbs
  datasource:
    dynamic:
      # 指定基础数据库,如果使用@DBS指定数据库,默认使用该数据库
      primary: stu
      datasource:
        # 配置学生数据源
        stu:
          url: jdbc:mysql://127.0.0.1:3306/student_db?characterEncoding=UTF-8&useUnicode=true&useSSL=false&serverTimezone=GMT
          username: root
          password: dsl@2022
          driver-class-name: com.mysql.cj.jdbc.Driver
        # 配置教师数据源
        tea:
          url: jdbc:mysql://127.0.0.1:3306/teacher_db?characterEncoding=UTF-8&useUnicode=true&useSSL=false&serverTimezone=GMT
          username: root
          password: dsl@2022
          driver-class-name: com.mysql.cj.jdbc.Driver

mybatis-plus:
  # 放在resource目录 classpath:/mapper/*Mapper.xml
  mapper-locations: classpath:/mapper/*Mapper.xml
  global-config:
    db-config:
      id-type: assign_id
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
    map-underscore-to-camel-case: true
    cache-enabled: false
```

```sql
create database student_db charset utf8mb4;
create database teacher_db charset utf8mb4;


use student_db;

create table student_info(
`id` bigint(11) primary key auto_increment,
`name` varchar(128) default null comment '学生名称',
`birthday` datetime default null comment '出生日期',
`gender` tinyint(2) default 0 comment '性别,0:未知,1:男,2:女'
)engine=innodb auto_increment=1 comment '学生信息表';


use teacher_db;

create table teacher_info(
`id` bigint(11) primary key auto_increment,
`name` varchar(128) default null comment '教师名称',
`birthday` datetime default null comment '出生日期',
`gender` tinyint(2) default 0 comment '性别,0:未知,1:男,2:女'
)engine=innodb auto_increment=1 comment '教师信息表';

use student_db;
insert into student_info(`name`,`birthday`,`gender`) values('酸菜鱼','1984-03-06 12:00:00','1');

use teacher_db;
insert into teacher_info(`name`,`birthday`,`gender`) values('haiyan','1994-03-06 12:00:00','1');
```

### 1.2 相关实现类

```java
package com.pkgs;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.pkgs.mapper")
public class App {

    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

```java
package com.pkgs.service;

import com.baomidou.dynamic.datasource.annotation.DS;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.pkgs.mapper.StudentInfoMapper;
import com.pkgs.mapper.TeacherInfoMapper;
import com.pkgs.model.entity.StudentInfo;
import com.pkgs.model.entity.TeacherInfo;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.List;

/**
 * <code>@DS</code> 注解可以注解在类上面,如果在类上面使用了<code>@DS("stu")</code>,
 * <p>
 * 然后在方法里面使用<code>@DS("tea")</code>,会根据就近原则使用tea数据源来处理.
 *
 * @author cs12110
 * @date 2022-03-31
 */
@Service
public class CombineService {

    @Resource
    private StudentInfoMapper studentInfoMapper;
    @Resource
    private TeacherInfoMapper teacherInfoMapper;

    public StudentInfo saveStudent(StudentInfo target) {
        studentInfoMapper.insert(target);

        return target;
    }

    public boolean deleteStudent(Long id) {
        studentInfoMapper.deleteById(id);
        return true;
    }


    public StudentInfo updateStudent(StudentInfo target) {
        studentInfoMapper.updateById(target);
        return target;
    }


    @DS("stu")
    public List<StudentInfo> getStudents() {
        LambdaQueryWrapper<StudentInfo> queryWrapper = new LambdaQueryWrapper<>();

        return studentInfoMapper.selectList(queryWrapper);
    }

    @DS("tea")
    public TeacherInfo saveTeacher(TeacherInfo target) {
        teacherInfoMapper.insert(target);
        return target;
    }

    @DS("tea")
    public boolean deleteTeacher(Long id) {
        teacherInfoMapper.deleteById(id);
        return true;
    }


    @DS("tea")
    public TeacherInfo updateTeacher(TeacherInfo target) {
        teacherInfoMapper.updateById(target);
        return target;
    }


    @DS("tea")
    public List<TeacherInfo> getTeachers() {
        LambdaQueryWrapper<TeacherInfo> queryWrapper = new LambdaQueryWrapper<>();

        return teacherInfoMapper.selectList(queryWrapper);
    }


    /**
     * 默认使用配置里面的primary数据源
     *
     * @return List
     */
    public List<StudentInfo> getPrimaryList() {
        LambdaQueryWrapper<StudentInfo> queryWrapper = new LambdaQueryWrapper<>();
        return studentInfoMapper.selectList(queryWrapper);
    }

    /**
     * 因为默认使用primary的数据源,这里查询会出错.
     *
     * @return List
     */
    public List<TeacherInfo> getWrongList() {
        LambdaQueryWrapper<TeacherInfo> queryWrapper = new LambdaQueryWrapper<>();
        return teacherInfoMapper.selectList(queryWrapper);
    }
}
```

```java
package com.pkgs.controller;

import com.pkgs.model.entity.StudentInfo;
import com.pkgs.model.entity.TeacherInfo;
import com.pkgs.model.resp.BaseResp;
import com.pkgs.service.CombineService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.List;

@RestController
@RequestMapping("dbs")
public class DbsController {

    @Resource
    private CombineService combineService;


    @PostMapping("/save-student")
    public BaseResp<StudentInfo> saveStudent(@RequestBody StudentInfo studentInfo) {
        StudentInfo value = combineService.saveStudent(studentInfo);
        return BaseResp.success(value);
    }

    @PostMapping("/delete-student/{studentId}")
    public BaseResp<Boolean> deleteStudent(@PathVariable("studentId") Long studentId) {
        boolean result = combineService.deleteStudent(studentId);
        return BaseResp.success(result);
    }


    @GetMapping("/students")
    public BaseResp<List<StudentInfo>> getStudents() {
        List<StudentInfo> values = combineService.getStudents();
        return BaseResp.success(values);
    }


    @PostMapping("/save-teacher")
    public BaseResp<TeacherInfo> saveTeacher(@RequestBody TeacherInfo teacherInfo) {
        TeacherInfo value = combineService.saveTeacher(teacherInfo);
        return BaseResp.success(value);
    }

    @PostMapping("/update-teacher")
    public BaseResp<TeacherInfo> updateTeacher(@RequestBody TeacherInfo teacherInfo) {
        TeacherInfo value = combineService.updateTeacher(teacherInfo);
        return BaseResp.success(value);
    }


    @PostMapping("/delete-teacher/{teacherId}")
    public BaseResp<Boolean> deleteTeacher(@PathVariable("teacherId") Long teacherId) {
        boolean result = combineService.deleteTeacher(teacherId);
        return BaseResp.success(result);
    }


    @GetMapping("/teachers")
    public BaseResp<List<TeacherInfo>> getTeachers() {
        List<TeacherInfo> values = combineService.getTeachers();
        return BaseResp.success(values);
    }


    @GetMapping("/combine-list")
    public BaseResp<List<Object>> getCombineList() {
        List<Object> values = new ArrayList<>();
        values.addAll(combineService.getStudents());
        values.addAll(combineService.getTeachers());
        return BaseResp.success(values);
    }

    @GetMapping("/primary-list")
    public BaseResp<List<StudentInfo>> getPrimaryList() {
        List<StudentInfo> values = combineService.getPrimaryList();
        return BaseResp.success(values);
    }

    @GetMapping("/wrong-list")
    public BaseResp<List<TeacherInfo>> getWrongList() {
        List<TeacherInfo> values = combineService.getWrongList();
        return BaseResp.success(values);
    }
}
```

```java
package com.pkgs.mapper;


import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.pkgs.model.entity.StudentInfo;

public interface StudentInfoMapper extends BaseMapper<StudentInfo> {
}
```

```java
package com.pkgs.mapper;


import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.pkgs.model.entity.TeacherInfo;

public interface TeacherInfoMapper  extends BaseMapper<TeacherInfo> {
}
```

```java
package com.pkgs.model.entity;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.Data;

import java.util.Date;

@Data
@TableName("student_info")
public class StudentInfo {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    private String name;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date birthday;

    private Integer gender;
}
```

```java
package com.pkgs.model.entity;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.Data;

import java.util.Date;

@Data
@TableName("teacher_info")
public class TeacherInfo {

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    private String name;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date birthday;

    private Integer gender;
}
```

### 1.3 测试使用

```shell
curl --location --request GET 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/students'
```

```shell
curl --location --request GET 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/teachers'
```

```shell
curl --location --request GET 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/combine-list'
```

```shell
curl --location --request GET 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/primary-list'
```

```shell
curl --location --request GET 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/wrong-list'
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/save-student' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name":"学生名称",
    "birthday":"2022-03-31 12:00:00",
    "gender":1
}'
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/update-student' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name":"学生名称",
    "birthday":"2022-03-31 12:00:00",
    "gender":1
}'
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/delete-student/1509453752142999553' \
--data-raw ''
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/save-teacher' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name":"老师名称",
    "birthday":"2022-03-31 12:00:00",
    "gender":1
}'
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/update-teacher' \
--header 'Content-Type: application/json' \
--data-raw ' {
        "id": 1509456083085479937,
        "name": "老师名称3444",
        "birthday": "2022-03-31 12:00:00",
        "gender": 1
    }'
```

```shell
curl --location --request POST 'http://127.0.0.1:8070/api/dynamic-dbs/dbs/delete-teacher/1509453877875650561' \
--data-raw ''
```

---

## 2. AOP

在现实环境里,因为服务拆分的问题,有可能出现多个数据源,并不像之前一个数据源就能搞定.

所以在 spring 里面学会使用多数据源配置很重要,本文档使用`SpringBoot+Mybatis-plus`,请知悉.

在 spring 配置多数据源主要有两种方式

- 扫描指定的 package
- 使用 aop 切面

这里只实现`指定package`,请知悉. [github 地址 link](https://github.com/cs12110/spring-dbs)

在 spring 配置多数据源主要有两种方式

- 扫描指定的 package
- 使用 aop 切面

这里只实现`指定package`,请知悉. [github 地址 link](https://github.com/cs12110/spring-dbs)

### 2.1 数据库准备

创建两个数据库:`order_db`和`product_db` _(请忽略惨不忍睹的数据库设计)_.

#### 2.1.1 DDL

```sql
CREATE DATABASE order_db charset utf8mb4;
CREATE DATABASE product_db charset utf8mb4;

-- 订单表
DROP TABLE
IF
	EXISTS order_db.t_order_info;

CREATE TABLE order_db.t_order_info (
	id BIGINT ( 20 ) PRIMARY KEY auto_increment,
	order_no VARCHAR(64) NOT NULL,
	product_code VARCHAR ( 64 ) NOT NULL,
	price NUMERIC ( 10, 2 ) DEFAULT 0,
	create_time datetime DEFAULT CURRENT_TIMESTAMP ,
	UNIQUE uidx_order_no(`order_no`)
) ENGINE = INNODB auto_increment = 1 COMMENT '订单表';

-- 产品表

DROP TABLE
IF
	EXISTS product_db.t_product_info;

CREATE TABLE product_db.t_product_info (
	id BIGINT ( 20 ) auto_increment PRIMARY KEY,
	product_code VARCHAR ( 64 ) NOT NULL,
	product_name VARCHAR ( 128 ) NOT NULL,
	price NUMERIC ( 10, 2 ) DEFAULT 0,
	stock INT ( 6 ) DEFAULT 0,
	create_time datetime DEFAULT CURRENT_TIMESTAMP,
    UNIQUE uidx_product_code ( `product_code` )
) ENGINE = INNODB auto_increment = 1 COMMENT '产品表';
```

#### 2.1.2 DML

```sql
INSERT INTO product_db.t_product_info ( product_code, product_name, price, stock ) VALUES( 'P202003281100', 'pc', '5400.00', '50' );

INSERT INTO order_db.t_order_info ( order_no, product_code, price ) VALUES( 'O202003281100', 'P202003281100', '4800' );
```

### 2.2 多数据源连接参数

```properties
# order db
datasource.order.url=jdbc:mysql://47.98.104.252:3306/order_db?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&useCursorFetch=true&useSSL=false
datasource.order.username=root
datasource.order.password=Root@3306
datasource.order.driver=com.mysql.jdbc.Driver

# product db
datasource.product.url=jdbc:mysql://47.98.104.252:3306/product_db?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&useCursorFetch=true&useSSL=false
datasource.product.username=root
datasource.product.password=Root@3306
datasource.product.driver=com.mysql.jdbc.Driver
```

### 2.3 加载参数

```java
package com.spring.dbs.conf.properties;

import lombok.Data;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:14
 * <p>
 * @since 1.0.0
 */
@Data
public class BasicDataSourceProperties {

    protected String url;
    protected String username;
    protected String password;
    protected String driver;
}
```

加载`order`数据源配置

```java
package com.spring.dbs.conf.properties;

import com.alibaba.fastjson.JSON;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:12
 * <p>
 * @since 1.0.0
 */
@Component
@ConfigurationProperties(prefix = "datasource.order")
public class OrderDataSourceProperties extends BasicDataSourceProperties {

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

加载`product`数据源配置

```java
package com.spring.dbs.conf.properties;

import com.alibaba.fastjson.JSON;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:16
 * <p>
 * @since 1.0.0
 */
@Component
@ConfigurationProperties(prefix = "datasource.product")
public class ProductDataSourceProperties extends BasicDataSourceProperties {

    @Override
    public String toString() {
        return JSON.toJSONString(this);
    }
}
```

### 2.4 配置数据源

#### 2.4.1 配置`order`数据源

```java
package com.spring.dbs.conf;

import com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean;
import com.spring.dbs.conf.properties.OrderDataSourceProperties;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.annotation.Resource;
import javax.sql.DataSource;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:11
 * <p>
 * @since 1.0.0
 */
@Slf4j
@Configuration
@MapperScan(
        basePackages = {
                "com.spring.dbs.mapper.order"
        },
        sqlSessionFactoryRef = "orderSqlSessionFactory"
)
public class OrderDataSourceConfiguration {

    @Resource
    private OrderDataSourceProperties orderDatasourceProperties;


    /**
     * 配置数据源
     *
     * @return DataSource
     */
    @Bean(name = "orderDataSource")
    public DataSource createDatasource() {
        DataSourceBuilder<?> dataSourceBuilder = DataSourceBuilder.create();
        dataSourceBuilder.url(orderDatasourceProperties.getUrl());
        dataSourceBuilder.username(orderDatasourceProperties.getUsername());
        dataSourceBuilder.password(orderDatasourceProperties.getPassword());
        dataSourceBuilder.driverClassName(orderDatasourceProperties.getDriver());

        return dataSourceBuilder.build();
    }


    /**
     * 使用指定的datasource构建SqlSessionFactory
     *
     * @param dataSource datasource ,使用Qualifier指定数据源
     * @return SqlSessionFactory
     */
    @Bean(name = "orderSqlSessionFactory")
    public SqlSessionFactory createSqlSessionFactory(@Qualifier("orderDataSource") DataSource dataSource) throws Exception {
        //SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();

        // 如果要使用mybatis-plus的baseMapper的功能,就要设置为MyabtisSqlSessionFactoryBean
        MybatisSqlSessionFactoryBean sqlSessionFactoryBean = new MybatisSqlSessionFactoryBean();
        // 设置数据源和映射的mapper地址
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();

        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath*:mapper/order/**.xml"));

        return sqlSessionFactoryBean.getObject();
    }

    /**
     * 使用SqlSessionFactory构建SqlSessionTemplate
     *
     * @param sqlSessionFactory sql session factory
     * @return SqlSessionTemplate
     */
    @Bean(name = "orderSqlSessionTemplate")
    public SqlSessionTemplate createSqlSessionTemplate(@Qualifier("orderSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        log.info("Function[createSqlSessionTemplate] create template now");
        return new SqlSessionTemplate(sqlSessionFactory);
    }


    /**
     * 构建事务管理器
     *
     * @param dataSource dataSource
     * @return DataSourceTransactionManager
     */
    @Bean(name = "orderTxManager")
    public DataSourceTransactionManager createTransactionManager(@Qualifier("orderDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

}
```

#### 2.4.2 配置`product`数据源

```java
package com.spring.dbs.conf;

import com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean;
import com.spring.dbs.conf.properties.ProductDataSourceProperties;
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionTemplate;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.annotation.Resource;
import javax.sql.DataSource;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:11
 * <p>
 * @since 1.0.0
 */
@Slf4j
@Configuration
@MapperScan(
        basePackages = {
                "com.spring.dbs.mapper.product"
        },
        sqlSessionFactoryRef = "productSqlSessionFactory"
)
public class ProductDataSourceConfiguration {

    @Resource
    private ProductDataSourceProperties productDataSourceProperties;

    /**
     * 配置数据源
     *
     * @return DataSource
     */
    @Bean(name = "productDataSource")
    public DataSource createDatasource() {
        DataSourceBuilder<?> dataSourceBuilder = DataSourceBuilder.create();
        dataSourceBuilder.url(productDataSourceProperties.getUrl());
        dataSourceBuilder.username(productDataSourceProperties.getUsername());
        dataSourceBuilder.password(productDataSourceProperties.getPassword());
        dataSourceBuilder.driverClassName(productDataSourceProperties.getDriver());

        return dataSourceBuilder.build();
    }


    /**
     * 使用指定的datasource构建SqlSessionFactory
     *
     * @param dataSource datasource ,使用Qualifier指定数据源
     * @return SqlSessionFactory
     */
    @Bean(name = "productSqlSessionFactory")
    public SqlSessionFactory createSqlSessionFactory(@Qualifier("productDataSource") DataSource dataSource) throws Exception {
        // SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();

        // 如果要使用mybatis-plus的baseMapper的功能,就要设置为MyabtisSqlSessionFactoryBean
        MybatisSqlSessionFactoryBean sqlSessionFactoryBean = new MybatisSqlSessionFactoryBean();
        // 设置数据源和映射的mapper地址
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath*:mapper/product/**.xml"));

        return sqlSessionFactoryBean.getObject();
    }

    /**
     * 使用SqlSessionFactory构建SqlSessionTemplate
     *
     * @param sqlSessionFactory sql session factory
     * @return SqlSessionTemplate
     */
    @Bean(name = "productSqlSessionTemplate")
    public SqlSessionTemplate createSqlSessionTemplate(@Qualifier("productSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        log.info("Function[createSqlSessionTemplate] create template now");
        return new SqlSessionTemplate(sqlSessionFactory);
    }


    /**
     * 构建事务管理器
     *
     * @param dataSource dataSource
     * @return DataSourceTransactionManager
     */
    @Bean(name = "productTxManager")
    public DataSourceTransactionManager createTransactionManager(@Qualifier("productDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

### 2.5 测试

#### 2.5.1 service 代码

```java
package com.spring.dbs.service;

import com.spring.dbs.entity.order.OrderInfo;
import com.spring.dbs.entity.product.ProductInfo;
import com.spring.dbs.mapper.order.OrderMapper;
import com.spring.dbs.mapper.product.ProductMapper;
import com.spring.dbs.model.response.InfoResponse;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.annotation.Resource;
import java.math.BigDecimal;
import java.util.Date;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-28 12:42
 * <p>
 * @since 1.0.0
 */
@Service
public class MyService {

    @Resource
    private OrderMapper orderMapper;
    @Resource
    private ProductMapper productMapper;


    public InfoResponse findInfo(String orderNo) {
        InfoResponse response = new InfoResponse();

        OrderInfo orderInfo = orderMapper.queryByOrderNo(orderNo);
        response.setOrderInfo(orderInfo);

        if (null != orderInfo) {
            ProductInfo productInfo = productMapper.queryInfo(orderInfo.getProductCode());
            response.setProductInfo(productInfo);
        }

        return response;
    }


    @Transactional(
            value = "orderTxManager",
            rollbackFor = RuntimeException.class
    )
    public InfoResponse saveOrderInfo(boolean rollback) {
        InfoResponse response = new InfoResponse();
        String unicode = String.valueOf(System.currentTimeMillis());

        OrderInfo orderInfo = createOrderInfo(unicode);
        orderMapper.insert(orderInfo);
        response.setOrderInfo(orderInfo);

        if (rollback) {
            throw new RuntimeException("Just throw an exception");
        }
        return response;
    }

    @Transactional(
            value = "productTxManager",
            rollbackFor = RuntimeException.class
    )
    public InfoResponse saveProductInfo(boolean rollback) {
        InfoResponse response = new InfoResponse();
        String unicode = String.valueOf(System.currentTimeMillis());

        ProductInfo productInfo = createProductInfo(unicode);
        productMapper.insert(productInfo);
        response.setProductInfo(productInfo);
        if (rollback) {
            throw new RuntimeException("Just throw an exception");
        }
        return response;
    }


    /**
     * 多数据源事务有问题,卧槽
     *
     * @param rollback 是否回滚
     * @return InfoResponse
     */
    @Transactional(rollbackFor = RuntimeException.class)
    public InfoResponse saveBothInfo(boolean rollback) {
        InfoResponse response = new InfoResponse();
        String unicode = String.valueOf(System.currentTimeMillis());

        OrderInfo orderInfo = createOrderInfo(unicode);
        orderMapper.insert(orderInfo);
        response.setOrderInfo(orderInfo);

        ProductInfo productInfo = createProductInfo(unicode);
        productMapper.insert(productInfo);
        response.setProductInfo(productInfo);
        if (rollback) {
            throw new RuntimeException("Just fuck throw an exception");
        }
        return response;

    }

    private OrderInfo createOrderInfo(String code) {
        OrderInfo orderInfo = new OrderInfo();
        orderInfo.setOrderNo("O" + code);
        orderInfo.setPrice(BigDecimal.valueOf(10.05));
        orderInfo.setProductCode("P" + code);
        orderInfo.setCreateTime(new Date());

        return orderInfo;
    }

    private ProductInfo createProductInfo(String code) {
        ProductInfo productInfo = new ProductInfo();
        productInfo.setCreateTime(new Date());
        productInfo.setPrice(BigDecimal.valueOf(100.04));
        productInfo.setProductCode("P" + code);
        productInfo.setProductName("orange");
        productInfo.setStock(100);

        return productInfo;
    }
}
```

#### 2.5.2 测试

测试获取数据

请求地址:`http://127.0.0.1:8080/my/findInfo?orderNo=O202003281100`

```json
{
  "orderInfo": {
    "id": 1,
    "orderNo": "O202003281100",
    "productCode": "P202003281100",
    "price": 4800.0,
    "createTime": "2020-03-28T07:26:45.000+0000"
  },
  "productInfo": {
    "id": 1,
    "productCode": "P202003281100",
    "productName": "pc",
    "price": 5400.0,
    "stock": 50,
    "createTime": "2020-03-28T07:26:44.000+0000"
  }
}
```

测试新增`order`

请求地址: `http://127.0.0.1:8080/my/saveInfo?operation=1&rollback=false`

```json
{
  "orderInfo": {
    "id": 2,
    "orderNo": "O1585380431042",
    "productCode": "P1585380431042",
    "price": 10.05,
    "createTime": "2020-03-28T07:27:11.042+0000"
  },
  "productInfo": null
}
```

测试新增`product`

请求地址: `http://127.0.0.1:8080/my/saveInfo?operation=2&rollback=false`

```json
{
  "orderInfo": null,
  "productInfo": {
    "id": 2,
    "productCode": "P1585380459849",
    "productName": "orange",
    "price": 100.04,
    "stock": 100,
    "createTime": "2020-03-28T07:27:39.849+0000"
  }
}
```

#### 2.5.3 结论

结论: **多数据初步构建成功,但是在新增数据涉及到数据源的时候,事务会出现问题.**

---

## 3. 参考资料

a. [Springboot 多数据源](https://blog.csdn.net/tuesdayma/article/details/81081666)
