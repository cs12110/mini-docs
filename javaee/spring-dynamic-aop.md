# Spring 动态 Aop

Q: 为啥要这么想不开折腾这个呀?

A: 在很久,很久,很久之前,就想弄这个了,但是一直没成功. orz

如果实现过程遇到相关报错,请参考`Q&A`相关问题和解决方案.

---

### 1. ShowMe

动态 AOP 核心代码如下:

```java
package com.chatgpt.common;

import com.alibaba.fastjson.JSON;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.springframework.aop.Pointcut;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor;
import org.springframework.aop.framework.Advised;
import org.springframework.aop.framework.ProxyFactory;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.support.GenericBeanDefinition;

import java.lang.reflect.Method;
import java.util.function.Supplier;

/**
 * @author cs12110
 * @version V1.0
 * @see <a href='https://gitee.com/lybgeek/springboot-learning'>spring项目</a>
 * @since 2023-07-31 10:22
 */
@Slf4j
public class DynamicAopUtil {

    private static final String DYNAMIC_AOP_PREFIX = "DynamicAdvisor";

    @Data
    public static class AopItem {
        /**
         * beanId
         */
        private String beanId;
        /**
         * bean className
         */
        private String className;
        /**
         * bean需要监测的表达式
         */
        private String pointCutExpression;
    }


    /**
     * 操作类型
     */
    private enum OperationType {
        /**
         * 新增
         */
        ADD_AOP,
        /**
         * 移除
         */
        REMOVE_AOP
    }

    /**
     * 添加Aop数据
     *
     * @param beanFactory factory
     * @param aopItem     aop配置
     * @throws RuntimeException 配置不成功或者已存在
     */
    public static void addAop(DefaultListableBeanFactory beanFactory, AopItem aopItem) {
        // 判断是否存在该bean
        if (beanFactory.containsBean(buildBeanId(aopItem))) {
            throw new RuntimeException("已存在Aop[" + buildBeanId(aopItem) + "]");
        }

        try {
            AspectJExpressionPointcutAdvisor advisor = getAspectJExpressionPointcutAdvisor(beanFactory, aopItem);
            handleAdvise(beanFactory, advisor, OperationType.ADD_AOP);
        } catch (Exception exp) {
            log.error("Function[addAop] aopItem:" + JSON.toJSONString(aopItem), exp);
            throw new RuntimeException("动态添加Aop失败", exp);
        }
    }


    /**
     * 删除Aop数据
     *
     * @param beanFactory factory
     * @param aopItem     aop配置
     */
    public static void removeAop(DefaultListableBeanFactory beanFactory, AopItem aopItem) {
        final String beanId = buildBeanId(aopItem);

        // 判断是否存在该bean
        if (!beanFactory.containsBean(beanId)) {
            log.warn("Function[removeAop] without bean:{}", beanId);
            return;
        }

        AspectJExpressionPointcutAdvisor advisor = beanFactory.getBean(beanId, AspectJExpressionPointcutAdvisor.class);
        handleAdvise(beanFactory, advisor, OperationType.REMOVE_AOP);
        // 先摧毁,然后删除bean,应对多次注册的情况
        beanFactory.destroyBean(beanFactory.getBean(beanId));
        beanFactory.removeBeanDefinition(beanId);
    }

    /**
     * 构建切面
     *
     * @param beanFactory factory
     * @param aopItem     切面参数
     * @return {@link AspectJExpressionPointcutAdvisor}
     * @throws Exception 构建异常
     */
    private static AspectJExpressionPointcutAdvisor getAspectJExpressionPointcutAdvisor(
            DefaultListableBeanFactory beanFactory,
            AopItem aopItem
    ) throws Exception {
        Class<?> clazz = Class.forName(aopItem.getClassName());
        MethodInterceptor methodInterceptor = (MethodInterceptor) clazz.getConstructor().newInstance();


        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
        GenericBeanDefinition beanDefinition = (GenericBeanDefinition) builder.getBeanDefinition();
        beanDefinition.setBeanClass(AspectJExpressionPointcutAdvisor.class);
        AspectJExpressionPointcutAdvisor advisor = new AspectJExpressionPointcutAdvisor();
        advisor.setExpression(aopItem.getPointCutExpression());
        advisor.setAdvice(methodInterceptor);
        beanDefinition.setInstanceSupplier((Supplier<AspectJExpressionPointcutAdvisor>) () -> advisor);
        beanFactory.registerBeanDefinition(buildBeanId(aopItem), beanDefinition);

        return advisor;
    }

    /**
     * 构建beanId
     *
     * @param aopItem 切面参数
     * @return String
     */
    private static String buildBeanId(AopItem aopItem) {
        return DYNAMIC_AOP_PREFIX + "_" + aopItem.getBeanId();
    }

    private static void handleAdvise(
            DefaultListableBeanFactory beanFactory,
            AspectJExpressionPointcutAdvisor advisor,
            OperationType operationType
    ) {
        AspectJExpressionPointcut pointcut = (AspectJExpressionPointcut) advisor.getPointcut();
        for (String beanDefinitionName : beanFactory.getBeanDefinitionNames()) {
            Object bean = beanFactory.getBean(beanDefinitionName);
            if (!(bean instanceof Advised)) {
                if (OperationType.ADD_AOP.equals(operationType)) {
                    buildCandidateAdvised(beanFactory, advisor, bean, beanDefinitionName);
                    log.info("Function[handleAdvise] add advise:{} for bean:{}", advisor.getAdvice().getClass().getName(), bean.getClass().getName());
                }
                continue;
            }
            Advised advisedBean = (Advised) bean;

            boolean isFindMatchAdvised = findMatchAdvised(advisedBean.getClass(), pointcut);
            if (isFindMatchAdvised) {
                if (OperationType.ADD_AOP.equals(operationType)) {
                    advisedBean.addAdvice(advisor.getAdvice());
                    log.info("Function[handleAdvise] add advise:{} for bean:{}", advisor.getAdvice().getClass().getName(), bean.getClass().getName());
                }
                if (OperationType.REMOVE_AOP.equals(operationType)) {
                    advisedBean.removeAdvice(advisor.getAdvice());
                    log.info("Function[handleAdvise] delete advise:{} for bean:{}", advisor.getAdvice().getClass().getName(), bean.getClass().getName());
                }
            }
        }
    }

    private static void buildCandidateAdvised(
            DefaultListableBeanFactory beanFactory,
            AspectJExpressionPointcutAdvisor advisor,
            Object bean,
            String beanDefinitionName
    ) {
        AspectJExpressionPointcut pointcut = (AspectJExpressionPointcut) advisor.getPointcut();
        boolean isFindMatchCandidateAdvised = findMatchCandidateAdvised(bean.getClass(), pointcut);
        if (isFindMatchCandidateAdvised) {
            ProxyFactory proxyFactory = new ProxyFactory();
            proxyFactory.setTarget(bean);
            proxyFactory.setProxyTargetClass(true);
            proxyFactory.addAdvisor(advisor);
            beanFactory.destroySingleton(beanDefinitionName);
            beanFactory.registerSingleton(beanDefinitionName, proxyFactory.getProxy());
        }
    }


    private static Boolean findMatchAdvised(Class<?> targetClass, Pointcut pointcut) {
        for (Method method : targetClass.getMethods()) {
            if (pointcut.getMethodMatcher().matches(method, targetClass)) {
                return true;
            }
        }
        return false;
    }

    private static Boolean findMatchCandidateAdvised(Class<?> targetClass, Pointcut pointcut) {
        return findMatchAdvised(targetClass, pointcut);
    }
}
```

---

### 2. 使用测试

#### 2.1 相关类

```java
package com.chatgpt.controller;

import com.chatgpt.common.DynamicAopUtil;
import com.chatgpt.model.resp.commom.ResultResp;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-07-29 10:12
 */
@RestController
@RequestMapping("/aop")
public class AopController {

    @Resource
    private DefaultListableBeanFactory defaultListableBeanFactory;

    @PostMapping("/addAop")
    public ResultResp<Void> addAop(@RequestBody DynamicAopUtil.AopItem aopItem) {
        DynamicAopUtil.addAop(defaultListableBeanFactory, aopItem);

        return ResultResp.success(null);
    }

    @PostMapping("/removeAop")
    public ResultResp<Void> removeAop(@RequestBody DynamicAopUtil.AopItem aopItem) {
        DynamicAopUtil.removeAop(defaultListableBeanFactory, aopItem);
        return ResultResp.success(null);
    }
}
```

```java
package com.chatgpt.service;

import com.chatgpt.common.DynamicAopUtil;
import org.springframework.stereotype.Service;

/**
 * @author cs12110
 * @version V1.0
 * @since 2024-04-27 17:19
 */
@Service
public class HelloWorldService {

    public void stream(DynamicAopUtil.AopItem askReq) {
    }
}
```

拦截器代码:

```java
package com.chatgpt.common;

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-07-31 10:22
 */
@Slf4j
public class MethodArgInterceptor implements MethodInterceptor {


    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        try {
            return methodInvocation.proceed();
        } finally {
            log.info("Function[watch] className:{}, methodName:{}, args:{}",
                    methodInvocation.getThis().getClass().getName(),
                    methodInvocation.getMethod().getName(),
                    JSON.toJSONString(methodInvocation.getArguments())
            );
        }
    }
}
```

```java
package com.chatgpt.controller;

import com.chatgpt.common.DynamicAopUtil;
import com.chatgpt.model.resp.commom.ResultResp;
import com.chatgpt.service.HelloWorldService;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

/**
 * @author cs12110
 * @version V1.0
 * @since 2023-07-29 10:12
 */
@RestController
@RequestMapping("/dynamic")
public class DynamicController {

    @Resource
    private HelloWorldService helloWorldService;

    @PostMapping("/stream")
    public ResultResp<Void> stream(@RequestBody DynamicAopUtil.AopItem aopItem) {
        helloWorldService.stream(aopItem);
        return ResultResp.success(null);
    }


}
```

#### 2.2 请求使用

```sh
curl --location '127.0.0.1:8080/chat/dynamic/stream' \
--header 'Content-Type: application/json' \
--data '{
    "beanId":"methodArgInterceptor",
    "className":"com.chatgpt.common.MethodArgInterceptor",
    "pointCutExpression":"execution(* com.chatgpt.service.*(..))"
}'
```

新增切面:

```sh
curl --location '127.0.0.1:8080/chat/aop/addAop' \
--header 'Content-Type: application/json' \
--data '{
    "beanId":"methodArgInterceptor",
    "className":"com.chatgpt.common.MethodArgInterceptor",
    "pointCutExpression":"execution (* com.chatgpt.service.HelloWorldService.*(..))"
}'
```

日志打印:

```text
2024-04-27 21:41:52.390  INFO 50708 --- [nio-8080-exec-3] com.chatgpt.common.MethodArgInterceptor  : Function[watch] className:com.chatgpt.service.HelloWorldService, methodName:stream, args:[{"beanId":"methodArgInterceptor","className":"com.chatgpt.common.MethodArgInterceptor","pointCutExpression":"execution(* com.chatgpt.service.*(..))"}]
```

删除切面:

```sh
curl --location '127.0.0.1:8080/chat/aop/removeAop' \
--header 'Content-Type: application/json' \
--data '{
    "beanId":"methodArgInterceptor",
    "className":"com.chatgpt.common.MethodArgInterceptor",
    "pointCutExpression":"execution (* com.chatgpt.service.HelloWorldService.*(..))"
}'
```

移除切面之后,重新请求,没相关日志打印.

```text
2024-04-27 21:49:03.919  INFO 50746 --- [nio-8080-exec-4] com.chatgpt.common.DynamicAopUtil        : Function[handleAdvise] delete advise:com.chatgpt.common.MethodArgInterceptor for bean:com.chatgpt.service.HelloWorldService$$EnhancerBySpringCGLIB$$2acb6d61
```

---

### 3. Q&A

Q: 为啥我的会出现这个问题?

```java
2024-04-27 18:20:57.637 ERROR 50204 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[.[dispatcherServlet]      : Servlet.service() for servlet [dispatcherServlet] in context with path [/chat] threw exception [Handler dispatch failed; nested exception is java.lang.NoClassDefFoundError: org/aspectj/weaver/tools/PointcutPrimitive] with root cause

java.lang.ClassNotFoundException: org.aspectj.weaver.tools.PointcutPrimitive
```

A: 需要添加 spring aop 切面相关依赖,具体版本由项目定.

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

Q: 注销 bean 之后,重新注册上去咋整???

```java
org.springframework.beans.factory.support.BeanDefinitionOverrideException: Invalid bean definition with name 'DynamicAdvisor-methodArgInterceptor' defined in null: Cannot register bean definition [Generic bean: class [org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor]; scope=; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null] for bean 'DynamicAdvisor-methodArgInterceptor': There is already [Generic bean: class [org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor]; scope=; abstract=false; lazyInit=null; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null] bound.
```

A: 在注销的时候,应该清理掉相关的注册信息

```java
// 先摧毁,然后删除bean,应对多次注册的情况
beanFactory.destroyBean(beanFactory.getBean(beanId));
beanFactory.removeBeanDefinition(beanId);
```

---

### 4. 参考资料

a. [lybgeek spring learning 项目](https://gitee.com/lybgeek/springboot-learning)
