# spring 多数据源

在现实环境里,因为服务拆分的问题,有可能出现多个数据源,并不像之前一个数据源就能搞定.

所以在 spring 里面学会使用多数据源配置很重要,本文档使用`SpringBoot+Mybatis-plus`,请知悉.

在 spring 配置多数据源主要有两种方式

- 扫描指定的 package
- 使用 aop 切面

这里只实现`指定package`,请知悉. [github 地址 link](https://github.com/cs12110/spring-dbs)

---

## 1. 数据库准备

创建两个数据库:`order_db`和`product_db` _(请忽略惨不忍睹的数据库设计)_.

### 1.1 DDL

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

### 1.2 DML

```sql
INSERT INTO product_db.t_product_info ( product_code, product_name, price, stock ) VALUES( 'P202003281100', 'pc', '5400.00', '50' );

INSERT INTO order_db.t_order_info ( order_no, product_code, price ) VALUES( 'O202003281100', 'P202003281100', '4800' );
```

---

## 2. 多数据源连接参数

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

---

## 3. 加载参数

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

---

## 4. 配置数据源

### 4.1 配置`order`数据源

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

### 4.2 配置`product`数据源

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

## 5 测试

### 5.1 service 代码

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

### 5.2 测试

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

### 5.3 结论

结论: **多数据初步构建成功,但是在新增数据涉及到数据源的时候,事务会出现问题.**

---

## 6. 参考资料

a. [Springboot 多数据源](https://blog.csdn.net/tuesdayma/article/details/81081666)
