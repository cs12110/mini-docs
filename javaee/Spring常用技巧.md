# Spring 常用技巧

本文档记录 Spring 经常使用到的一些技巧.

---

## 1.切面计算执行耗时

在性能测试里面,我们需要找到某一个方法执行耗时多久的情况,这是 AOP 绝佳的一个使用场景.

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

/**
 * 使用AOP统计Service里面每个执行方法耗时
 *
 * <p>
 *
 * <pre>
 * <code>Before</code>:方法前执行
 * </pre>
 *
 * <pre>
 * <code>AfterReturning</code>:运行方法后执行
 * </pre>
 *
 * <pre>
 * <code>AfterThrowing<code>:Throw后执行
 * </pre>
 *
 * <pre>
 * <code>After</code>:无论方法以何种方式结束,都会执行(类似于finally)
 * </pre>
 *
 * <pre>
 * <code>Around</code>:环绕执行
 * </pre>
 *
 * @author huanghuapeng 2017年9月18日
 * @see
 * @since 1.0
 */
@Aspect
@Component
public class ServiceExecTimeCalculateInteceptor {

    private static Logger logger = LoggerFactory
            .getLogger(ServiceExecTimeCalculateInteceptor.class);

    /**
     * 切面表达式
     *
     * <pre>
     * execution(public * * (. .)) 任意公共方法被执行时,执行切入点函数
     * </pre>
     *
     * <pre>
     * execution( * set* (. .)) 任何以一个"set" 开始的方法被执行时,执行切入点函数
     * </pre>
     *
     * <pre>
     * execution( * com.demo.service.AccountService.* (. .)) 当接口AccountService 中的任意方法被执行时,执行切入点函数
     * </pre>
     *
     * execution( * com.demo.service.*.* (. .)) 当service 包中的任意方法被执行时,执行切入点函数
     */
    private static final String serviceMethodsExp = "execution(* cn.rojao.irs.ds.service.impl.*.*(..))";

    /**
     * 计算用时
     *
     * <p>
     * 一定要返回执行后的结果,不然原方法没有返回值?
     *
     * @param joinPoint
     *            {@linkplain ProceedingJoinPoint}
     * @return Object
     */
    @Around(serviceMethodsExp)
    public Object timeAround(ProceedingJoinPoint joinPoint) {
        Object proceed = null;

        long start = System.currentTimeMillis();
        try {
            proceed = joinPoint.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }

        long end = System.currentTimeMillis();

        StringBuilder str = new StringBuilder();
        str.append(getExecMethodName(joinPoint));
        str.append(" spend: ");
        str.append((end - start));
        str.append(" ms");

        logger.info(str.toString());

        return proceed;
    }

    /**
     * 获取执行方法名称
     *
     * @param joinPoint
     *            切点
     * @return String
     */
    private String getExecMethodName(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String methodName = signature.getDeclaringTypeName() + "." + signature.getName();

        return methodName;
    }
}
```
