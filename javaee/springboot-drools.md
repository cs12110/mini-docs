# drools

Q: 这个是啥?

A: 还能是啥?是祥子的黄包车,是孔乙己的书,是马孔多的生活. [官网 link](https://www.drools.org/)

---

### 1. 使用案例

简单场景: 当单据金额>=1000,推送 OA 审批,当单据金额<1000 时,自动审批通过

#### 1.1 依赖和配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>springboot-drools</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- Inherit defaults from Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- web env -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- aspect -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>

        <!--- rookie env -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.30</version>
        </dependency>

        <dependency>
            <groupId>org.drools</groupId>
            <artifactId>drools-core</artifactId>
            <version>7.24.0.Final</version>
        </dependency>
        <dependency>
            <groupId>org.kie</groupId>
            <artifactId>kie-spring</artifactId>
            <version>7.24.0.Final</version>
        </dependency>
    </dependencies>
</project>
```

```yaml
server:
  port: 8080
  servlet:
    context-path: /api/

spring:
  application:
    name: spring-boot-rule-engine
```

#### 1.2 动态规则 service

```java
package org.ruleengine.service;

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.ruleengine.model.entity.SettInfo;
import org.springframework.stereotype.Component;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 16:05
 */
@Slf4j
@Component
public class SettInfoService {


    public void autoAudit(SettInfo settInfo) {
        log.info("Function[autoAudit] sett:{}", JSON.toJSONString(settInfo));
    }

    public void pushToOa(SettInfo settInfo) {
        log.info("Function[pushToOa] sett:{}", JSON.toJSONString(settInfo));
    }
}
```

```java
package org.ruleengine.service;

import org.ruleengine.model.entity.UserInfo;
import org.springframework.stereotype.Service;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 22:08
 */
@Service
public class UserInfoService {

    public UserInfo getById(Long id) {
        UserInfo userInfo = new UserInfo();
        userInfo.setId(id);
        userInfo.setName("test");
        userInfo.setLevel(1);

        return userInfo;
    }
}
```

```java
package org.ruleengine.service;

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.kie.api.io.Resource;
import org.kie.api.io.ResourceType;
import org.kie.api.runtime.KieContainer;
import org.kie.api.runtime.StatelessKieSession;
import org.kie.internal.builder.KnowledgeBuilder;
import org.kie.internal.builder.KnowledgeBuilderFactory;
import org.kie.internal.io.ResourceFactory;
import org.kie.internal.utils.KieHelper;
import org.ruleengine.model.entity.SettInfo;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 14:39
 */
@Slf4j
@Component
public class RuleEngineService {

    @Autowired
    private ApplicationContext applicationContext;

    @Autowired
    private SettInfoService settInfoService;


    public void execute(SettInfo settInfo) {
        try {
            log.info("Function[execute] drools script: \n{}", getDroolsScript());

            Resource scriptResource = ResourceFactory.newByteArrayResource(
                    getDroolsScript().getBytes(StandardCharsets.UTF_8)
            );

            KnowledgeBuilder knowledgeBuilder = KnowledgeBuilderFactory.newKnowledgeBuilder();
            knowledgeBuilder.add(scriptResource, ResourceType.DRL);

            if (knowledgeBuilder.hasErrors()) {
                throw new RuntimeException(knowledgeBuilder.getErrors().toString());
            }

            // 构建KieContainer
            KieHelper kieHelper = new KieHelper();
            kieHelper.addContent(getDroolsScript(), ResourceType.DRL);
            KieContainer kieContainer = kieHelper.getKieContainer();

            // 执行规则
            StatelessKieSession statelessKieSession = kieContainer.newStatelessKieSession();
            // 注入bean
            statelessKieSession.setGlobal("settInfoService", settInfoService);
            statelessKieSession.setGlobal("applicationContext", applicationContext);

            // 执行规则
            statelessKieSession.execute(settInfo);

            // 关闭kieContainer
            kieContainer.dispose();
        } catch (Exception e) {
            log.info("Function[execute] error: " + JSON.toJSONString(settInfo), e);
            throw new RuntimeException(e);
        }
    }

    /**
     * 获取drools脚本, 后期可以设置为数据库获取相关的脚本数据
     * <p>
     * 当金额>=1000时,推送到OA
     * 当金额<1000时,自动审批
     * <pre>
     * package sett
     *
     * global org.ruleengine.service.SettInfoService settInfoService
     * global org.springframework.context.ApplicationContext applicationContext
     *
     * import com.alibaba.fastjson.JSON
     *
     * import org.ruleengine.service.UserInfoService
     *
     * import org.ruleengine.model.entity.SettInfo
     * import org.ruleengine.model.entity.UserInfo
     *
     * rule "sett-amount-ge-1000"
     *     when
     *         settInfo : SettInfo(amount >= 1000)
     *         userInfo : UserInfo(id != 0) from ((UserInfoService)applicationContext.getBean(UserInfoService.class)).getById(1L)
     *     then
     *         System.out.println("amount > 1000: "+JSON.toJSONString(settInfo));
     *         System.out.println("amount > 1000: "+JSON.toJSONString(userInfo));
     *         settInfo.setTips("规则1");
     *         settInfoService.pushToOa(settInfo);
     * end
     * rule "sett-amount-lt-1000"
     *     when
     *         settInfo : SettInfo(amount < 1000)
     *     then
     *         System.out.println("amount < 1000: "+JSON.toJSONString(settInfo));
     *         settInfo.setTips("规则2");
     *         settInfoService.autoAudit(settInfo);
     * end
     *
     * </pre>
     *
     * @return String
     */
    private String getDroolsScript() {
        StringBuilder script = new StringBuilder();
        // 设置包名
        script.append("package sett").append(System.lineSeparator());

        // 设置全局变量
        script.append("global org.ruleengine.service.SettInfoService settInfoService").append(System.lineSeparator());
        script.append("global org.springframework.context.ApplicationContext applicationContext").append(System.lineSeparator());

        // 设置导入类
        script.append("import com.alibaba.fastjson.JSON").append(System.lineSeparator());
        script.append("import org.ruleengine.service.UserInfoService").append(System.lineSeparator());
        script.append("import org.ruleengine.model.entity.SettInfo").append(System.lineSeparator());
        script.append("import org.ruleengine.model.entity.UserInfo").append(System.lineSeparator());

        // 设置规则1
        script.append("rule \"sett-amount-ge-1000\"").append(System.lineSeparator());
        script.append("    //salience: 设置规则优先级, 数字越大优先级越高").append(System.lineSeparator());
        script.append("    salience 1").append(System.lineSeparator());
        script.append("    //no-loop: 防止死循环，当规则使用update之类的函数修改了Fact对象时，使当前规则再次被激活从而导致死循环").append(System.lineSeparator());
        script.append("    no-loop true").append(System.lineSeparator());
        script.append("    when").append(System.lineSeparator());
        script.append("        settInfo : SettInfo(amount >= 1000)").append(System.lineSeparator());
        script.append("        userInfo : UserInfo(id != 0) from ((UserInfoService)applicationContext.getBean(UserInfoService.class)).getById(1L)").append(System.lineSeparator());
        script.append("    then").append(System.lineSeparator());
        script.append("        System.out.println(\"amount > 1000: \"+JSON.toJSONString(settInfo));").append(System.lineSeparator());
        script.append("        System.out.println(\"amount > 1000: \"+JSON.toJSONString(userInfo));").append(System.lineSeparator());
        script.append("        settInfo.setTips(\"规则1\");").append(System.lineSeparator());
        script.append("        settInfoService.pushToOa(settInfo);").append(System.lineSeparator());
        script.append("end").append(System.lineSeparator());

        // 设置规则2
        script.append("rule \"sett-amount-lt-1000\"").append(System.lineSeparator());
        script.append("    salience 1").append(System.lineSeparator());
        script.append("    no-loop true").append(System.lineSeparator());
        script.append("    when").append(System.lineSeparator());
        script.append("        settInfo : SettInfo(amount < 1000)").append(System.lineSeparator());
        script.append("    then").append(System.lineSeparator());
        script.append("        System.out.println(\"amount < 1000: \"+JSON.toJSONString(settInfo));").append(System.lineSeparator());
        script.append("        settInfo.setTips(\"规则2\");").append(System.lineSeparator());
        script.append("        settInfoService.autoAudit(settInfo);").append(System.lineSeparator());
        script.append("end").append(System.lineSeparator());

        return script.toString();
    }
}
```

脚本内容:

```shell
package sett

global org.ruleengine.service.SettInfoService settInfoService
global org.springframework.context.ApplicationContext applicationContext

import com.alibaba.fastjson.JSON
import org.ruleengine.service.UserInfoService
import org.ruleengine.model.entity.SettInfo
import org.ruleengine.model.entity.UserInfo

rule "sett-amount-ge-1000"
    //salience: 设置规则优先级, 数字越大优先级越高
    salience 1
    //no-loop: 防止死循环，当规则使用update之类的函数修改了Fact对象时，使当前规则再次被激活从而导致死循环
    no-loop true
    when
        settInfo : SettInfo(amount >= 1000)
        userInfo : UserInfo(id != 0) from ((UserInfoService)applicationContext.getBean(UserInfoService.class)).getById(1L)
    then
        System.out.println("amount > 1000: "+JSON.toJSONString(settInfo));
        System.out.println("amount > 1000: "+JSON.toJSONString(userInfo));
        settInfo.setTips("规则1");
        settInfoService.pushToOa(settInfo);
end

rule "sett-amount-lt-1000"
    salience 1
    no-loop true
    when
        settInfo : SettInfo(amount < 1000)
    then
        System.out.println("amount < 1000: "+JSON.toJSONString(settInfo));
        settInfo.setTips("规则2");
        settInfoService.autoAudit(settInfo);
end

```

---

### 2. 测试使用

#### 2.1 参数定义

```java
package org.ruleengine.model.entity;

import lombok.Data;

import java.math.BigDecimal;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 14:40
 */
@Data
public class SettInfo {

    /**
     * 单据单号
     */
    private String settNo;

    /**
     * 单据类型
     */
    private Integer type;

    /**
     * 单据金额
     */
    private BigDecimal amount;
}
```

```java
package org.ruleengine.model.entity;

import lombok.Data;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 22:08
 */
@Data
public class UserInfo {

    private Long id;

    private String name;

    private Integer level;
}
```

```java
package org.ruleengine.controler;

import org.ruleengine.model.entity.SettInfo;
import org.ruleengine.model.resp.RespResult;
import org.ruleengine.service.RuleEngineService;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-11-25 14:13
 */
@RestController
@RequestMapping("/rule-engine")
public class RuleEngineController {
    @Resource
    private RuleEngineService ruleEngineService;

    @ResponseBody
    @PostMapping("/lift")
    public RespResult<?> lift(@RequestBody SettInfo settInfo) {

        ruleEngineService.execute(settInfo);

        return RespResult.success(null);
    }
}
```

#### 2.2 测试

```sh
curl --location 'http://127.0.0.1:8080/api/rule-engine/lift' \
--header 'Content-Type: application/json' \
--data '{
    "settNo":"JS0001",
    "type":"1",
    "amount":1800
}'
```

```
2023-11-25 23:15:31.626  INFO 14904 --- [nio-8080-exec-9] o.d.c.k.builder.impl.KieRepositoryImpl   : KieModule was added: MemoryKieModule[releaseId=org.default:artifact:1.0.0]
amount > 1000: {"amount":1800,"settNo":"JS0001","type":1}
amount > 1000: {"id":1,"level":1,"name":"test"}
2023-11-25 23:15:31.645  INFO 14904 --- [nio-8080-exec-9] org.ruleengine.service.SettInfoService   : Function[pushToOa] sett:{"amount":1800,"settNo":"JS0001","tips":"规则1","type":1}
```

```sh
curl --location 'http://127.0.0.1:8080/api/rule-engine/lift' \
--header 'Content-Type: application/json' \
--data '{
    "settNo":"JS0001",
    "type":"1",
    "amount":800
}'
```

```
2023-11-25 23:15:59.557  INFO 14904 --- [io-8080-exec-10] o.d.c.k.builder.impl.KieRepositoryImpl   : KieModule was added: MemoryKieModule[releaseId=org.default:artifact:1.0.0]
amount < 1000: {"amount":800,"settNo":"JS0001","type":1}
2023-11-25 23:15:59.568  INFO 14904 --- [io-8080-exec-10] org.ruleengine.service.SettInfoService   : Function[autoAudit] sett:{"amount":800,"settNo":"JS0001","tips":"规则2","type":1}
```

---

### 3. 参考资料

a. [drools 官网 link](https://www.drools.org/)
b. [Drools 规则引擎应用 看这一篇就够了 link](https://www.cnblogs.com/ityml/p/15993391.html)
c. [Drools 使用总结和踩坑 link](https://juejin.cn/post/6969879653209079845)
c. [打工人学习 Drools 高级语法 link](https://cloud.tencent.com/developer/article/1751900)
c. [Spring Boot 整合 Drools 规则引擎 link](https://zhuanlan.zhihu.com/p/326419830)
