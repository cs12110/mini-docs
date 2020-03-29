# Seata

分布式事务,为了解决温饱.

本文档只说明重要的代码部分,如需详情请查看[github](https://github.com/cs12110/spring-tx-plus)的项目.

---

## 1. 安装 seata

下载安装包

```sh
# github下载真的锻炼人的耐性和考验人的性命
# root @ team-3 in /opt/soft/seata/ [23:41:33]
$ wget https://github.com/seata/seata/releases/download/v1.1.0/seata-server-1.1.0.tar.gz

# root @ team-3 in /opt/soft/seata/ [23:41:33]
$ tar -xvf seata-server-1.1.0.tar.gz

# root @ team-3 in /opt/soft/seata/ [23:41:33]
$ cd seata/

# root @ team-3 in /opt/soft/seata/seata [23:45:21]
$ pwd
/opt/soft/seata/seata

# root @ team-3 in /opt/soft/seata/seata [23:45:26]
$ ls
LICENSE  bin  conf  lib
```

开启相关接口

```sh
# root @ team-3 in /opt/soft/seata/seata [23:37:19]
$ firewall-cmd --add-port=8091/tcp --zone=public --permanent
success

# root @ team-3 in /opt/soft/seata/seata [23:41:08]
$ firewall-cmd --reload
success
```

开启 seata 服务

```sh
# 使用nohup启动服务,请注意,对内存有要求.
# root @ team3 in /opt/soft/seata/seata [0:01:12]
$ nohup bin/seata-server.sh -p 8091 -h 118.89.113.147 -m file &
```

---

## 2. 项目使用

项目基于:`springboot`+`mybatis plus`+`seata`,请知悉.

项目地址: [github link](https://github.com/cs12110/spring-tx-plus).

### 2.1 项目结构说明

- spring-tx-business: 项目调用入口,调用 order 和 storage 服务
- spring-tx-common: 公共模块
- spring-tx-order: 订单模块
- spring-tx-storage: 库存模块

启动顺序: `spring-tx-order`->`spring-tx-storage`->`spring-tx-business`.

### 2.2 依赖和配置

seata 的依赖如下:

```xml
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

相关配置,涉及到全局事务的都要配置

```properties
seata.tx-service-group=spring_tx_group
seata.service.grouplist=118.89.113.147:8091
```

依赖数据脚本

```sql
#
# Order
#
DROP DATABASE IF EXISTS seata_order;
CREATE DATABASE seata_order;

CREATE TABLE seata_order.orders (
	id INT ( 11 ) NOT NULL AUTO_INCREMENT,
	user_id INT ( 11 ) DEFAULT NULL,
	product_id INT ( 11 ) DEFAULT NULL,
	pay_amount DECIMAL ( 10, 0 ) DEFAULT NULL,
	STATUS VARCHAR ( 100 ) DEFAULT NULL,
	add_time DATETIME DEFAULT CURRENT_TIMESTAMP,
	last_update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY ( id )
) ENGINE = INNODB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8;

CREATE TABLE seata_order.undo_log (
	id BIGINT ( 20 ) NOT NULL AUTO_INCREMENT,
	branch_id BIGINT ( 20 ) NOT NULL,
	xid VARCHAR ( 100 ) NOT NULL,
	context VARCHAR ( 128 ) NOT NULL,
	rollback_info LONGBLOB NOT NULL,
	log_status INT ( 11 ) NOT NULL,
	log_created DATETIME NOT NULL,
	log_modified DATETIME NOT NULL,
	PRIMARY KEY ( id ),
	UNIQUE KEY ux_undo_log ( xid, branch_id )
) ENGINE = INNODB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8;

#
# Storage
#
DROP DATABASE IF	EXISTS seata_storage;
CREATE DATABASE seata_storage;

CREATE TABLE seata_storage.product (
	id INT ( 11 ) NOT NULL AUTO_INCREMENT,
	price DOUBLE DEFAULT NULL,
	stock INT ( 11 ) DEFAULT NULL,
	last_update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	PRIMARY KEY ( id )
) ENGINE = INNODB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8;

INSERT INTO seata_storage.product ( id, price, stock ) VALUES	( 1, 5, 10 );

CREATE TABLE seata_storage.undo_log (
	id BIGINT ( 20 ) NOT NULL AUTO_INCREMENT,
	branch_id BIGINT ( 20 ) NOT NULL,
	xid VARCHAR ( 100 ) NOT NULL,
	context VARCHAR ( 128 ) NOT NULL,
	rollback_info LONGBLOB NOT NULL,
	log_status INT ( 11 ) NOT NULL,
	log_created DATETIME NOT NULL,
	log_modified DATETIME NOT NULL,
	PRIMARY KEY ( id ),
	UNIQUE KEY ux_undo_log ( xid, branch_id )
) ENGINE = INNODB AUTO_INCREMENT = 1 DEFAULT CHARSET = utf8;
```

### 2.3 公共部分代码

在`common`模块里面,有一个很重要的拦截器和过滤器

#### 过滤器

```java
package com.spring.seata.tx.common.filter;

import com.alibaba.fastjson.JSON;
import io.seata.core.context.RootContext;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.Map;

/**
 * 处理全局事务xid过滤器
 *
 * <p>
 *
 * @author cs12110 create at 2020-03-29 01:21
 * <p>
 * @since 1.0.0
 */
@Component
@Slf4j
public class SeataFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        displayHeader(httpServletRequest);


        // 获取请求头部的xid
        boolean isBind = false;
        String xid = httpServletRequest.getHeader(RootContext.KEY_XID.toLowerCase());

        if (StringUtils.isNotEmpty(xid)) {
            RootContext.bind(xid);
            isBind = true;
        }

        try {
            // 不进行异常捕抓
            chain.doFilter(request, response);
        } finally {
            if (isBind) {
                RootContext.unbind();
            }
        }
    }


    /**
     * 打印请求头部
     *
     * @param httpServletRequest {@link HttpServletRequest}
     */
    private void displayHeader(HttpServletRequest httpServletRequest) {
        Map<String, Object> headerMap = new HashMap<>(16);

        Enumeration<String> headerNames = httpServletRequest.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String element = headerNames.nextElement();
            headerMap.put(element, httpServletRequest.getHeader(element));
        }

        log.info("Function[displayHeader] request header:{}", JSON.toJSONString(headerMap, true));
    }

    @Override
    public void destroy() {

    }
}
```

#### 拦截器

```java
package com.spring.seata.tx.common.interceptor;

import io.seata.core.context.RootContext;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.http.client.support.HttpRequestWrapper;

import java.io.IOException;

/**
 * 设置请求的头部xid
 *
 * <p>
 *
 * @author cs12110 create at 2020-03-29 12:02
 * <p>
 * @since 1.0.0
 */
@Slf4j
public class SeataRestTemplateInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        HttpRequestWrapper wrapper = new HttpRequestWrapper(request);

        String xid = RootContext.getXID();

        if (StringUtils.isNotEmpty(xid)) {
            log.info("Function[interceptor] wrapper xid:{}", xid);
            wrapper.getHeaders().add(RootContext.KEY_XID, xid);
        }

        return execution.execute(wrapper, body);
    }

}
```

配置拦截到到`RestTemplate`

```java
package com.spring.seata.tx.common.conf;

import com.spring.seata.tx.common.interceptor.SeataRestTemplateInterceptor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.web.client.RestTemplate;

import javax.annotation.PostConstruct;
import javax.annotation.Resource;
import java.util.Collection;
import java.util.List;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-29 11:55
 * <p>
 * @since 1.0.0
 */
@Slf4j
@Configuration
public class SeataRestTemplateAutoConfiguration {

    @Autowired(required = false)
    private Collection<RestTemplate> restTemplates;


    @Resource
    private SeataRestTemplateInterceptor seataRestTemplateInterceptor;


    @Bean(name = "seataRestTemplateInterceptor")
    public SeataRestTemplateInterceptor createSeataRestTemplateInterceptor() {
        return new SeataRestTemplateInterceptor();
    }


    /**
     * 初始化,添加restTemplate的拦截器
     */
    @PostConstruct
    public void init() {
        if (null != restTemplates && !restTemplates.isEmpty()) {
            for (RestTemplate rt : restTemplates) {
                // 添加seata拦截器
                List<ClientHttpRequestInterceptor> interceptors = rt.getInterceptors();
                interceptors.add(seataRestTemplateInterceptor);

                rt.setInterceptors(interceptors);
            }
        }
    }
}
```

### 2.4 全局事务入口

全局事务入口,在`spring-tx-business`模块

```java
package com.spring.seata.tx.business.service;

import com.spring.seata.tx.business.component.ServiceUrlComponent;
import com.spring.seata.tx.common.exception.BizException;
import com.spring.seata.tx.common.model.response.BizResponse;
import io.seata.spring.annotation.GlobalTransactional;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import javax.annotation.Resource;

/**
 * <p>
 *
 * @author cs12110 create at 2020-03-29 17:46
 * <p>
 * @since 1.0.0
 */
@Slf4j
@Service
public class BusinessService {

    @Resource
    private ServiceUrlComponent serviceUrlComponent;

    @Resource
    private RestTemplate restTemplate;

    /**
     * 使用分布式事务,commitOrder && commitStorage == true时,全局事务才会提交,如果其中一个是false,全局事务回滚.
     *
     * @param commitOrder   提交订单
     * @param commitStorage 提交库存
     * @return BizResponse
     */
    @GlobalTransactional
    public BizResponse doBusiness(boolean commitOrder, boolean commitStorage) {
        log.info("doBusiness,commitOrder:{},commitStorage:{}", commitOrder, commitStorage);
        try {
            callOrderService(commitOrder);
            callStorageService(commitStorage);
        } catch (Exception e) {
            log.error("doBusiness,commitOrder:" + commitOrder + ",commitStorage:" + commitStorage, e);
            throw new BizException("error:" + e.getMessage());
        }

        return BizResponse.createSuccessResponse(commitStorage);
    }


    private void callOrderService(boolean commit) {
        String url = serviceUrlComponent.getOrderUrl() + "/api/order/handleWithOrder?commit=" + commit;

        ResponseEntity<BizResponse> result = restTemplate.getForEntity(url, BizResponse.class);

        log.info("Function[callOrderService] result:{}", result.getBody());
    }

    private void callStorageService(boolean commit) {
        String url = serviceUrlComponent.getStorageUrl() + "/api/storage/handleWithStorage?commit=" + commit;

        ResponseEntity<BizResponse> result = restTemplate.getForEntity(url, BizResponse.class);

        log.info("Function[callStorageService] result:{}", result.getBody());
    }

}
```

分布式事务提交,调用示例

```sh
# mr3306 @ mr3306 in /opt/docs/mini-docs/javaee on git:master x [22:07:05]
$ curl 'http://127.0.0.1:8000/api/business/deal?commitOrder=true&commitStorage=true'
```

分布式事务回滚,调用示例

```sh
# mr3306 @ mr3306 in /opt/docs/mini-docs/javaee on git:master x [22:07:05]
$ curl 'http://127.0.0.1:8000/api/business/deal?commitOrder=true&commitStorage=false'
```

---

## 3. 总结

- 多个数据源,每个数据库都有一个`undo_log`的表
- 在执行完事务之后,不论是回滚还是提交,undo_log 的记录会被删除,如果要看记录,建议断点查看.
- 分布式事务的耗时很不乐观. orz

---

## 4. 参考资料

a. [seata 官网](https://seata.io/zh-cn/docs/overview/what-is-seata.html)

b. [seata samples](https://github.com/seata/seata-samples)

c. [springboot 与 seata 整合 blog](https://www.cnblogs.com/huanchupkblog/p/12185851.html)
