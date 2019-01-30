# Swagger Api

在开发的接口的时候,通常需要大量的测试和相关的接口说明文档.

那么 swagger2,你值得拥有.

---

## 1. SpringBoot 整合 Swagger2

SpringBoot 整合 Swagger2.

### 1.1 pom.xml

Springboot 的依赖,在这里就不贴了,就贴一下 swagger2 的依赖,请谅.

```xml
<!-- swagger -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.2.2</version>
</dependency>
```

### 1.2 配置类

在添加完依赖之后,要对 swagger 进行一些简单的配置.

```java
package com.pkgs.conf;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

/**
 * Swagger2 conf
 * <p>
 * 访问地址: `http://127.0.0.1:8081/swagger-ui.html`
 * <p/>
 *
 * @author cs12110 created at: 2019/1/30 14:25
 * <p>
 * since: 1.0.0
 */
@Configuration
@EnableSwagger2
public class Swagger2Conf {


    /**
     * swagger2配置文件,配置扫描包
     *
     * @return Docket
     */
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(buildApiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.pkgs.ctrl"))
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 创建API信息
     *
     * @return ApiInfo
     */
    private ApiInfo buildApiInfo() {
        return new ApiInfoBuilder()
                .title("Api home")
                .contact("cs12110")
                .version("1.0")
                .description("rest api docs")
                .build();
    }
}
```

### 1.3 接口类

```java
package com.pkgs.ctrl;

import com.alibaba.fastjson.JSON;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.pkgs.conf.anno.AntiResubmitAnno;
import com.pkgs.entity.AnswerEntity;
import com.pkgs.service.AnswerService;
import com.pkgs.service.TopicService;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiImplicitParams;
import io.swagger.annotations.ApiOperation;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * TODO: controller
 * <p>
 * Author: cs12110 create at: 2019/1/6 17:10
 * Since: 1.0.0
 */
@Controller
@RequestMapping("/rest")
@Api("RestCtrl")
public class RestCtrl {

    @Autowired
    private AnswerService answerService;

    @Autowired
    private TopicService topicService;

    /**
     * 获取回答
     *
     * @param topicId   话题Id
     * @param pageIndex 分页下标
     * @param pageSize  分页大小
     * @return String
     */
    @RequestMapping("/answers")
    @ResponseBody
    @AntiResubmitAnno
    @ApiOperation(value = "根据topicId,分页获取精华回答", notes = "获取精华回答")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "topicId", value = "话题id", required = true, dataType = "String", paramType = "query"),
            @ApiImplicitParam(name = "pageIndex", value = "page index", required = true, dataType = "Integer", paramType = "query"),
            @ApiImplicitParam(name = "pageSize", value = "page size", required = true, dataType = "Integer", paramType = "query")
    })
    public String answers(String topicId, Integer pageIndex, Integer pageSize) {
        if ("null".equals(topicId)) {
            topicId = null;
        }

        Page<AnswerEntity> page = new Page<>(pageIndex, pageSize);
        List<AnswerEntity> list = answerService.queryWithTopic(topicId, page);

        Map<String, Object> map = new HashMap<>(2);
        map.put("page", page);
        map.put("list", list);

        return JSON.toJSONString(map);

    }


    /**
     * 获取顶级的话题
     *
     * @return String
     */
    @RequestMapping("/topTopics")
    @ResponseBody
    @AntiResubmitAnno
    public String topTopic() {
        return JSON.toJSONString(topicService.queryTopTopics());
    }

    /**
     * 获取父级下面的所有有答案的类型
     *
     * @param parentId 父级话题Id
     * @return List
     */
    @RequestMapping("/topics")
    @ResponseBody
    @AntiResubmitAnno
    public String topic(String parentId) {
        return JSON.toJSONString(topicService.queryTopics(parentId));
    }

}
```

### 1.4 访问页面

```html
http://ip:port/swagger-ui.html#/
```

---

## 2. Swagger 注解说明

该章节摘录于: [link](https://www.dalaoyang.cn/article/21),请知悉.

### 2.1 @Api

作用:用在请求的类上,表示对类的说明.

参数说明:

```java
tags="说明该类的作用,可以在 UI 界面上看到的注解"

value="该参数没什么意义,在 UI 界面上也看到,所以不需要配置"
```

示例:

```java
@Api(tags="APP用户注册Controller")
```

### 2.2 @ApiOperation

作用: 用在请求的方法上,说明方法的用途、作用

参数说明:

```java
value="说明方法的用途、作用"
notes="方法的备注说明"
```

示例

```java
@ApiOperation(value="用户注册",notes="手机号、密码都是必输项,年龄随边填,但必须是数字")
```

### 2.3 @ApiImplicitParams

作用: 用在请求的方法上,表示一组参数说明

参数说明:

```java
@ApiImplicitParam:用在@ApiImplicitParams注解中,指定一个请求参数的各个方面
        name:参数名
        value:参数的汉字说明、解释
        required:参数是否必须传
        paramType:参数放在哪个地方
            · header:  请求参数的获取:@RequestHeader
            · query:   请求参数的获取:@RequestParam
            · path(用于restful接口): 请求参数的获取,@PathVariable
            · body(不常用)
            · form(不常用)
        dataType:参数类型,默认String,其它值dataType="Integer"
        defaultValue:参数的默认值
```

示例:

```java
@ApiImplicitParams({
    @ApiImplicitParam(name="mobile",value="手机号",required=true,paramType="form"),
    @ApiImplicitParam(name="password",value="密码",required=true,paramType="form"),
    @ApiImplicitParam(name="age",value="年龄",required=true,paramType="form",dataType="Integer")
})
```

### 2.4 @ApiResponses

作用:用在请求的方法上,表示一组响应

参数说明:

```java
@ApiResponse:用在@ApiResponses中,用于表达一个错误的响应信息
  code:数字,例如 400
  message:信息,例如"请求参数没填好"
  response:抛出异常的类
```

示例:

```java
@ApiOperation(value = "select1 请求",notes = "多个参数,多种的查询参数类型")
@ApiResponses({
    @ApiResponse(code=400,message="请求参数没填好"),
    @ApiResponse(code=404,message="请求路径没有或页面跳转路径不对")
})
```

### 2.5 @ApiModel

- @ApiModel:用于响应类上,表示一个返回响应数据的信息(这种一般用在 post 创建的时候,使用@RequestBody 这样的场景,请求参数无法使用@ApiImplicitParam 注解进行描述的时候)

- @ApiModelProperty:用在属性上,描述响应类的属性

```java
import io.swagger.annotations.ApiModel;
import io.swagger.annotations.ApiModelProperty;

import java.io.Serializable;

@ApiModel(description= "返回响应数据")
public class RestMessage implements Serializable{
    @ApiModelProperty(value = "是否成功")
    private boolean success=true;
    @ApiModelProperty(value = "返回对象")
    private Object data;
    @ApiModelProperty(value = "错误编号")
    private Integer errCode;
    @ApiModelProperty(value = "错误信息")
    private String message;
}
```

```java
@ApiOperation(value="保存用户", notes="保存用户")
@RequestMapping(value="/saveUser", method= RequestMethod.POST)
public String saveUser(@RequestBody @ApiParam(name="用户对象",value="传入json格式",required=true) User user){
    userDao.save(user);
    return "success!";
}
```

---

## 3. 参考资料

a. [Swagger使用博客](https://www.dalaoyang.cn/article/21)