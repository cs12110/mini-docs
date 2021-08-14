# Sharding-Jdbc

不知道这是第几次看这个玩意了,所以重新整理一个 helloworld 级别的项目,加深自己的理解(大佬请绕路).

这哪是 sharing-jdbc,简直就是 sharding-life.

Warning: 因为使用 <u>shardingsphere v5.0</u>没成功过,所以该文档使用的是 v4.0 版本依赖,请知悉.

---

### 1. 基础知识

[官网基础知识 link](https://shardingsphere.apache.org/document/legacy/4.x/document/cn/features/sharding/concept/sql/)

#### 1.1 逻辑表&绑定表

**逻辑表**: 水平拆分的数据库(表)相同逻辑和数据结构表的总称.例: 订单数据根据主键尾数拆分为 10 张表,分别是 t_order_0~t_order_9,他们的逻辑表名为 t_order.

**真实表**: 在分片的数据库中真实存在的物理表. 即上例中的 t_order_0 ~ t_order_9.

**数据节点**: 数据分片的最小单元. 由数据源名称和数据表组成，例：ds_0.t_order_0.

**绑定表**: 指分片规则一致的主表和子表.例如: t_order 表和 t_order_item 表,均按照 order_id 分片,则此两张表互为绑定表关系.绑定表之间的多表关联查询不会出现笛卡尔积关联,关联查询效率将大大提升.

#### 1.2 广播表

[广播表知识博客 link](https://blog.csdn.net/danielzhou888/article/details/104853954)

```properties
# 广播表, 其主节点是ds0
spring.shardingsphere.sharding.broadcast-tables=t_config
spring.shardingsphere.sharding.tables.t_config.actual-data-nodes=ds$->{0}.t_config
```

#### 1.3 表达式

[官网配置参考范例 link](https://shardingsphere.apache.org/document/legacy/4.x/document/cn/features/sharding/other-features/inline-expression/)

Q: 看官网里面的配置例子,各种: `ds$->{0..1}.t_order$->{user_id%2}`,这些表达式,都整的是啥呀?

A: 这些表达式是为了更简单的配置映射的数据库表配置,含义如下:

`${expression}`或`$->{expression}`标识行表达式:

- ${begin..end}: 表示范围区间

- ${[unit1, unit2, unit_x]}: 表示枚举值

例如:

`ds$->{0..1}`代表: ds 只有 ds0 和 ds1 两种

`t_order$->{user_id%2}`代表: 数据表根据 user_id%2 来取

---

### 2. 项目代码

采用 Springboot 的项目来测试使用,请知悉. [sharding-life-project github link](https://github.com/cs12110/sharding-life-project)

创建两个数据库: sharding_db0 和 sharding_db1

```sql
create database sharding_db0 charset 'utf8mb4';
create database sharding_db1 charset 'utf8mb4';
```

在数据库 sharding_db0 和 sharding_db1 分别创建 t_order 数据表

```sql
CREATE TABLE `t_order` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL,
  `order_id` bigint(20) DEFAULT NULL,
  `fee` decimal(10,2) DEFAULT NULL,
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1426198852486041603 DEFAULT CHARSET=utf8mb4;
```

#### 2.1 依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>sharding-life-project</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.2.RELEASE</version>
    </parent>

    <properties>
        <shardingsphere.version>4.0.0-RC1</shardingsphere.version>
        <mybatis.plus.spring.boot.verison>3.4.2</mybatis.plus.spring.boot.verison>
        <mysql.driver.version>8.0.20</mysql.driver.version>
        <druid.version>1.0.29</druid.version>
        <lombok.version>1.18.10</lombok.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis.plus.spring.boot.verison}</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>${druid.version}</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.driver.version}</version>
        </dependency>


        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
            <version>${shardingsphere.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <optional>true</optional>
        </dependency>

    </dependencies>

</project>
```

#### 2.2 配置

其实,使用 shardingsphere 最重要就是配置,在 5.x 的版本,配置一直没成功,心累.只要配置好相关配置,增删改查的场景就很简单的实现了,shardingsphere 还是一个很不错的东西呀.

```properties
server.port=7070
server.servlet.context-path=/sharding/

# 处理数据源问题
spring.main.allow-bean-definition-overriding=true

# 配置真实数据源,设置名称为ds0,ds1
spring.shardingsphere.datasource.names=ds0,ds1

# 配置第 1 个数据源
spring.shardingsphere.datasource.ds0.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.ds0.url=jdbc:mysql://118.89.113.147:3306/sharding_db0?characterEncoding=utf8&useUnicode=true&serverTimezone=CTT
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=Team@3306

# 配置第 2 个数据源
spring.shardingsphere.datasource.ds1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.ds1.url=jdbc:mysql://118.89.113.147:3306/sharding_db1?characterEncoding=utf8&useUnicode=true&serverTimezone=CTT
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=Team@3306

# 配置 t_order 表规则,因为这里面没进行分表,仅仅是分库而已
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order

# 指定数据库分片策略,约定user_id值是偶数添加到sharding_db0中，奇数添加到sharding_db1中
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.database-strategy.inline.algorithm-expression=ds$->{user_id % 2 }

# 指定数据表分片策略 约定user_id值是偶数添加到t_order表，如果user_id是奇数还是添加到t_order表 笑哭脸.jpg
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=user_id
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order

# 打开sql输出日志
spring.shardingsphere.props.sql.show=true
```

#### 2.2 测试

启动类代码如下:

```java
package com.sharding.life;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * @author cs12110
 * @version V1.0
 * @since 2021-08-10 10:42
 */
@SpringBootApplication
@MapperScan(basePackages = { "com.sharding.life.mapper" })
public class ShardingLifeApp {

    public static void main(String[] args) {
        SpringApplication.run(ShardingLifeApp.class, args);
    }

}
```

##### controller

```java
@RestController
@RequestMapping("/v1/sharding")
public class ShardingController {

    @Resource
    private ShardingService shardingService;

    @GetMapping("/page")
    public Resp page(QueryOrderReq queryOrderReq) {
        IPage<OrderDO> page = shardingService.getPage(queryOrderReq);
        return Resp.success(page);
    }

    @GetMapping("/list")
    public Resp list(Integer userId) {
        List<OrderDO> list = shardingService.getList(userId);
        return Resp.success(list);
    }

    @PostMapping("/save-order")
    public Resp saveOrder(@RequestBody OrderDO order) {
        shardingService.saveInfo(order);
        return Resp.success(order);
    }

    @DeleteMapping("/delete-order")
    public Resp deleteOrder(Long userId, Long orderId) {
        boolean result = shardingService.deleteOrder(userId, orderId);
        return Resp.success(result);
    }

    @PostMapping("/update-order")
    public Resp updateOrder(@RequestBody OrderDO order) {
        shardingService.updateInfo(order);
        return Resp.success(order);
    }
}
```

##### service

```java
package com.sharding.life.service;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.conditions.update.LambdaUpdateWrapper;
import com.baomidou.mybatisplus.core.metadata.IPage;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.sharding.life.dto.request.order.QueryOrderReq;
import com.sharding.life.entity.OrderDO;
import com.sharding.life.mapper.OrderMapper;

import java.util.List;
import java.util.Objects;

import org.springframework.stereotype.Service;

/**
 * @author cs12110
 * @version V1.0
 * @since 2021-08-13 17:59
 */
@Service
public class ShardingService extends ServiceImpl<OrderMapper, OrderDO> {

    public void saveInfo(OrderDO order) {
        baseMapper.insert(order);
    }

    public List<OrderDO> getList(Integer userId) {
        LambdaQueryWrapper<OrderDO> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        if (Objects.nonNull(userId)) {
            lambdaQueryWrapper.eq(OrderDO::getUserId, userId);
        }
        return baseMapper.selectList(lambdaQueryWrapper);
    }

    public IPage<OrderDO> getPage(QueryOrderReq queryOrderReq) {
        LambdaQueryWrapper<OrderDO> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        if (Objects.nonNull(queryOrderReq.getUserId())) {
            lambdaQueryWrapper.eq(OrderDO::getUserId, queryOrderReq.getUserId());
        }

        if (Objects.nonNull(queryOrderReq.getOrderId())) {
            lambdaQueryWrapper.eq(OrderDO::getOrderId, queryOrderReq.getOrderId());
        }

        Page<OrderDO> page = new Page<>(queryOrderReq.getPageNumber(), queryOrderReq.getPageSize());

        return baseMapper.selectPage(page, lambdaQueryWrapper);
    }

    public boolean deleteOrder(Long userId, Long orderId) {
        LambdaUpdateWrapper<OrderDO> lambdaUpdateWrapper = new LambdaUpdateWrapper<>();
        if (Objects.nonNull(orderId)) {
            lambdaUpdateWrapper.eq(OrderDO::getOrderId, orderId);
        }
        if (Objects.nonNull(userId)) {
            lambdaUpdateWrapper.eq(OrderDO::getUserId, userId);
        }

        int delete = baseMapper.delete(lambdaUpdateWrapper);

        return delete > 0;
    }

    public void updateInfo(OrderDO order) {
        LambdaUpdateWrapper<OrderDO> lambdaUpdateWrapper = new LambdaUpdateWrapper<>();
        if (Objects.nonNull(order.getUserId())) {
            lambdaUpdateWrapper.eq(OrderDO::getUserId, order.getUserId());
        }
        if (Objects.nonNull(order.getOrderId())) {
            lambdaUpdateWrapper.eq(OrderDO::getOrderId, order.getOrderId());
        }

        lambdaUpdateWrapper.set(OrderDO::getFee, order.getFee());

        baseMapper.update(null, lambdaUpdateWrapper);
    }
}
```

##### mapper

```java
package com.sharding.life.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.sharding.life.entity.OrderDO;

/**
 * @author cs12110
 * @version V1.0
 * @since 2021-08-13 18:04
 */
public interface OrderMapper extends BaseMapper<OrderDO> {
}
```

##### entity

```java
package com.sharding.life.entity;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;

import java.math.BigDecimal;
import java.util.Date;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2021-08-13 18:06
 */
@Data
@TableName("t_order")
public class OrderDO {
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;
    private Long userId;
    private Long orderId;
    private BigDecimal fee;
    private Date createTime;
}
```

---

### 3. 测试使用

启动项目之后,就可以进行增删改查的测试了. :"}

#### 3.1 新增

```sh
curl -L -X POST '127.0.0.1:7070/sharding/v1/sharding/save-order' \
-H 'Content-Type: application/json' \
--data-raw '{
 "orderId":"123",
 "userId":"1",
 "fee":100.00
}'
```

输出日志:

```java
2021-08-14 14:36:53.684  INFO 55711 --- [nio-7070-exec-8] ShardingSphere-SQL                       : Actual SQL: ds1 ::: INSERT INTO t_order   (id, user_id, order_id, fee) VALUES (?, ?, ?, ?) ::: [1426432622183084034, 1, 123, 100.00]
```

在上面日志可以看出,`user_id%2==1` 选择了 `ds1` 的数据配置,所以在 sharding_db1.t_order 新增了一条订单数据.

Q: 如果我新增的数据 userId = 2 呢,那么会在哪里新增?

A: 会根据策略选择 ds0 数据配置,在 sharding_db0.t_order 新增数据.

#### 3.2 更新

```sh
curl -L -X POST '127.0.0.1:7070/sharding/v1/sharding/update-order' \
-H 'Content-Type: application/json' \
--data-raw '{
    "orderId": "123",
    "fee": 20.00,
    "userId": "1"
}'
```

输出日志:

```java
2021-08-14 14:41:55.899  INFO 55711 --- [io-7070-exec-10] ShardingSphere-SQL                       : Actual SQL: ds1 ::: UPDATE t_order  SET fee=?

 WHERE (user_id = ? AND order_id = ?) ::: [20.00, 1, 123]
```

在上面日志可以看出,`user_id%2==1` 选择了 `ds1` 的数据配置,所以在 sharding_db1.t_order 更新了 userId = 1 & orderId=123 的订单 fee 字段.

Q: 如果在 sharding_db0.t_order 和 sharding_db1.t_order 都存在一条 order=123 的数据,我只传递 orderId=123 进去更新,会是怎样子的效果呀?

A: 请求参数如下:

```sh
curl -L -X POST '127.0.0.1:7070/sharding/v1/sharding/update-order' \
-H 'Content-Type: application/json' \
--data-raw '{
    "orderId": "123",
    "fee": 20.00,
    "userId": null
}'
```

后台输出日志如下:

```java
2021-08-14 14:45:02.488  INFO 55711 --- [nio-7070-exec-2] ShardingSphere-SQL                       : Actual SQL: ds0 ::: UPDATE t_order  SET fee=?

 WHERE (order_id = ?) ::: [20.00, 123]
2021-08-14 14:45:02.488  INFO 55711 --- [nio-7070-exec-2] ShardingSphere-SQL                       : Actual SQL: ds1 ::: UPDATE t_order  SET fee=?

 WHERE (order_id = ?) ::: [20.00, 123]
```

结论: **<u>如果根据策略匹配不到数据配置,会去所有配置节点执行该 sql(这个同样使用与更新/删除操作).</u>**

#### 3.3 删除

```sh
curl -L -X DELETE '127.0.0.1:7070/sharding/v1/sharding/delete-order?userId=2&orderId=1237' \
--data-raw ''
```

输出日志:

```java
2021-08-14 14:49:16.057  INFO 55711 --- [nio-7070-exec-7] ShardingSphere-SQL                       : Actual SQL: ds0 ::: DELETE FROM t_order

 WHERE (order_id = ? AND user_id = ?) ::: [1237, 2]
```

在上面日志可以看出,`user_id%2==0` 选择了 `ds0` 的数据配置,所以在 sharding_db1.t_order 删除 userId = 2 & orderId=1237 的订单.

#### 3.4 列表

```sh
curl -L -X GET '127.0.0.1:7070/sharding/v1/sharding/list?userId=1' \
--data-raw ''
```

输出日志:

```java
2021-08-14 14:51:37.272  INFO 55711 --- [nio-7070-exec-9] ShardingSphere-SQL                       : Actual SQL: ds1 ::: SELECT  id,user_id,order_id,fee,create_time  FROM t_order

 WHERE (user_id = ?) ::: [1]
```

```sh
curl -L -X GET '127.0.0.1:7070/sharding/v1/sharding/list?userId=2' \
--data-raw ''
```

输出日志:

```java
2021-08-14 14:51:59.889  INFO 55711 --- [io-7070-exec-10] ShardingSphere-SQL                       : Actual SQL: ds0 ::: SELECT  id,user_id,order_id,fee,create_time  FROM t_order

 WHERE (user_id = ?) ::: [2]
```

#### 3.5 分页

就差一个分页了,开心.

```sh
# 获取全部用户数据
curl -L -X GET '127.0.0.1:7070/sharding/v1/sharding/page?userId=&pageSize=10&pageNumber=1'
```

输出日志:

```java
2021-08-14 14:56:21.913  INFO 55711 --- [nio-7070-exec-5] ShardingSphere-SQL                       : Actual SQL: ds0 ::: SELECT  id,user_id,order_id,fee,create_time  FROM t_order LIMIT ? ::: [10]
2021-08-14 14:56:21.913  INFO 55711 --- [nio-7070-exec-5] ShardingSphere-SQL                       : Actual SQL: ds1 ::: SELECT  id,user_id,order_id,fee,create_time  FROM t_order LIMIT ? ::: [10]
```

```sh
# 获取userId=1的数据
curl -L -X GET '127.0.0.1:7070/sharding/v1/sharding/page?userId=1&pageSize=10&pageNumber=1'
```

输出日志:

```java
2021-08-14 14:55:12.987  INFO 55711 --- [nio-7070-exec-3] ShardingSphere-SQL                       : Actual SQL: ds1 ::: SELECT  id,user_id,order_id,fee,create_time  FROM t_order

 WHERE (user_id = ?) LIMIT ? ::: [1, 10]
```

#### 3.6 总结

- 要注意更新/删除操作在没有匹配到明确数据配置的操作.
- shardingspehere 的 helloworld 终于完成了. orz

扩展:

Q: 如果 order 还有 order_item 之类的,该怎么设置?

A: 这个的确是个问题,我也还没实践,不好意思.

Q: 如果广播表你还没告诉我要怎么弄呀.

A: 欲知后事如何,请听下回...,啊,没下回了,要自己去看了.

---

### 4. 参考文档

a. [ShardingSphere v4.x 官方文档](https://shardingsphere.apache.org/document/legacy/4.x/document/cn/overview/)

b. [SpringBoot+Sharding-JDBC 操作分库分表 blog](https://blog.csdn.net/s1078229131/article/details/106785894)
