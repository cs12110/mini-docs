# Gateway

该文档很粗糙,仅仅作为临时笔记,请知悉.

---

## 1. 基础知识

关于网关拦截器的命名规则:

```java
17.2.1. Naming Custom Filters And References In Configuration

Custom filters class names should end in `GatewayFilterFactory.`

For example, to reference a filter named `Something` in configuration files,
the filter must be in a class named `SomethingGatewayFilterFactory`.

It is possible to create a gateway filter named without the GatewayFilterFactory suffix, such as class AnotherThing. This filter could be referenced as AnotherThing in configuration files. This is not a supported naming convention and this syntax may be removed in future releases. Please update the filter name to be compliant.
```

---

## 2. 范例

```java
package cn.example.bankend.config;

import java.util.stream.Collectors;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.HttpMessageConverter;

import cn.example.bankend.filter.ForbidGatewayFilterFactory;
import cn.example.bankend.filter.InnerUserTokenGatewayFilterFactory;
import cn.example.bankend.filter.MasterDataGatewayFilterFactory;
import cn.example.bankend.filter.ExampleRequestRateLimiterGatewayFilterFactory;
import cn.example.bankend.filter.MemberPermitGatewayFilterFactory;
import cn.example.bankend.filter.SecurityGatewayFilterFactory;
import cn.example.bankend.filter.TokenValidateGatewayFilterFactory;
import cn.example.bankend.filter.WebSocketGatewayFilterFactory;

/**
 * @author ssh
 * @version V1.0
 * @since 2019-11-13 15:31
 */
@Configuration
public class GatewayFilterConfig {

    @Bean
    public TokenValidateGatewayFilterFactory tokenValidateGatewayFilterFactory() {
        return new TokenValidateGatewayFilterFactory();
    }

    @Bean
    public MasterDataGatewayFilterFactory masterDataGatewayFilterFactory() {
        return new MasterDataGatewayFilterFactory();

    }

    @Bean
    public ExampleRequestRateLimiterGatewayFilterFactory ExampleRequestRateLimiterGatewayFilterFactory(
        RedisRateLimiter redisRateLimiter) {
        return new ExampleRequestRateLimiterGatewayFilterFactory(redisRateLimiter);
    }

    @Bean
    public ForbidGatewayFilterFactory forbidGatewayFilterFactory() {
        return new ForbidGatewayFilterFactory();
    }

    @Bean
    public WebSocketGatewayFilterFactory webSocketGatewayFilterFactory() {
        return new WebSocketGatewayFilterFactory();
    }

    @Bean
    public SecurityGatewayFilterFactory securityGatewayFilterFactory() {
        return new SecurityGatewayFilterFactory();
    }

    @Bean
    public InnerUserTokenGatewayFilterFactory innerUserTokenGatewayFilterFactory() {
        return new InnerUserTokenGatewayFilterFactory();
    }

    @Bean
    public MemberPermitGatewayFilterFactory memberPermitGatewayFilterFactory() {
        return new MemberPermitGatewayFilterFactory();
    }

    @Bean
    @ConditionalOnMissingBean
    public HttpMessageConverters httpMessageConverters(ObjectProvider<HttpMessageConverter<?>> converters){
        return new HttpMessageConverters(converters.orderedStream().collect(Collectors.toList()));
    }
}
```

```java
package cn.example.bankend.filter;

import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.util.function.Consumer;

import javax.annotation.Resource;

import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.context.annotation.Lazy;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.http.server.reactive.ServerHttpRequest;

import cn.example.backend.data.domain.DefaultResult;
import cn.example.bankend.config.FilterConfig;
import cn.example.bankend.constant.GlobalConstant;
import cn.example.bankend.remote.UserCenterRemoteService;
import cn.example.bankend.remote.dto.UserTokenMsgRespDTO;
import cn.example.bankend.support.GatewayFilterSupport;
import lombok.extern.slf4j.Slf4j;

/**
 * @author none
 * @version V1.0
 * @since 2019-11-13 15:16
 */
@Slf4j
public class TokenValidateGatewayFilterFactory
    extends AbstractGatewayFilterFactory<FilterConfig> implements Ordered {

    public TokenValidateGatewayFilterFactory() {
        super(FilterConfig.class);
    }

    @Autowired
    @Lazy
    UserCenterRemoteService userCenterRemoteService;

    @Override
    public GatewayFilter apply(FilterConfig config) {
        return (exchange, chain) -> {
            if (GatewayFilterSupport.unAuth(config, exchange.getRequest())) {
                return chain.filter(exchange);
            }
            String token = exchange.getRequest().getHeaders().getFirst(GlobalConstant.TOKEN);
            if (StringUtils.isBlank(token)) {
                return GatewayFilterSupport.unAuthError(exchange.getRequest(), exchange.getResponse(), "token is null");
            }
            DefaultResult<UserTokenMsgRespDTO> result = userCenterRemoteService.validToken(token);
            if (result == null || result.getData() == null) {
                return GatewayFilterSupport.unAuthError(exchange.getRequest(), exchange.getResponse(),
                    result == null ? "invoke usercenter result null" : result.getMessage());
            }
            UserTokenMsgRespDTO tokenMsgRespDTO = result.getData();
            ServerHttpRequest.Builder builder = exchange.getRequest().mutate().headers(new Consumer<HttpHeaders>() {
                @Override
                public void accept(HttpHeaders httpHeaders) {
                    httpHeaders.set("userId", String.valueOf(tokenMsgRespDTO.getUserId()));
                    httpHeaders.set("phone", tokenMsgRespDTO.getPhone());
                    if (tokenMsgRespDTO.getIsTest() != null){
                        httpHeaders.set("isTestUser", String.valueOf(tokenMsgRespDTO.getIsTest()));
                    }
                    try {
                        if (StringUtils.isNotBlank(tokenMsgRespDTO.getUserName())) {
                            httpHeaders.set("userName", URLEncoder.encode(tokenMsgRespDTO.getUserName(), "UTF-8"));
                         }
                    } catch (Exception e) {
                        log.error("用户姓名编码异常",e);
                    }
                }
            });
            return chain.filter(exchange.mutate().request(builder.build()).build());
        };
    }
    @Override
    public int getOrder() {
        return -100;
    }

}
```

```java
package cn.example.bankend.support;

import com.nepxion.discovery.common.util.JsonUtil;

import java.net.InetAddress;
import java.nio.CharBuffer;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.EnvironmentAware;
import org.springframework.core.env.Environment;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;

import cn.example.backend.data.domain.DefaultResult;
import cn.example.bankend.config.FilterConfig;
import cn.example.bankend.constant.GlobalConstant;
import lombok.extern.slf4j.Slf4j;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

/**
 * @version V1.0
 * @since 2019-12-11 17:41
 */
@Slf4j
@Component
public class GatewayFilterSupport implements EnvironmentAware {

    private static Environment env;

    @Override
    public void setEnvironment(Environment environment) {
        env = environment;
    }

    public static Mono<Void> unAuthError(ServerHttpRequest request, ServerHttpResponse response, String reason) {
        return error(request, response, HttpStatus.UNAUTHORIZED,
            "token validate fail,reason=" + reason + ",token=" + request.getHeaders().get(GlobalConstant.TOKEN));
    }

    public static Mono<Void> tooManyError(ServerHttpRequest request, ServerHttpResponse response) {
        return error(request, response, HttpStatus.TOO_MANY_REQUESTS, "too many request reason=");
    }

    public static Mono<Void> badRequest(ServerHttpRequest request, ServerHttpResponse response) {
        return error(request, response, HttpStatus.BAD_REQUEST, "bad request");
    }

    public static Mono<Void> forbidError(ServerHttpRequest request, ServerHttpResponse response) {
        return forbidError(request, response, "forbid request");
    }

    public static Mono<Void> forbidError(ServerHttpRequest request, ServerHttpResponse response,String reason) {
        return error(request, response, HttpStatus.FORBIDDEN, reason);
    }

    public static boolean isWhiteHeader(ServerHttpRequest request,FilterConfig filterConfig) {
        FilterConfig.HeaderConfig whiteHeaderConfig = filterConfig.getWhiteHeader();
        if (null == whiteHeaderConfig){
            return false;
        }
        String whiteHeaderValue = request.getHeaders().getFirst(whiteHeaderConfig.getHeaderName());
        return StringUtils.isNotBlank(whiteHeaderValue) && whiteHeaderValue.equals(whiteHeaderConfig.getHeaderValue());
    }

    public static boolean isWhiteUrl(String whitePrePathConfig, String path) {
        if (StringUtils.isBlank(whitePrePathConfig) || StringUtils.isBlank(path)) {
            return false;
        }
        List<String> whiteList = Arrays.asList(whitePrePathConfig.split(GlobalConstant.SEPARATOR_COMMA));
        whiteList = whiteList.stream().filter(white -> path.indexOf(white) > -1).collect(Collectors.toList());
        return whiteList.size() > 0;
    }

    public static boolean isBlackUrl(String blackPrePathConfig, String path) {
        if (StringUtils.isBlank(blackPrePathConfig) || StringUtils.isBlank(path)) {
            return false;
        }
        List<String> blackList = Arrays.asList(blackPrePathConfig.split(GlobalConstant.SEPARATOR_COMMA));
        blackList = blackList.stream().filter(black -> path.indexOf(black) > -1).collect(Collectors.toList());
        return blackList.size() > 0;
    }

    public static boolean unAuth(FilterConfig filterConfig,ServerHttpRequest request){
        String path = request.getURI().getPath();
        String environment = env.getProperty("spring.profiles.active");
        //api接口文档地址不做校验
        if (path.contains("v2/api-docs")){
            return environment.equals("prod") ? false:true;
        }
        if (isWhiteHeader(request,filterConfig)){
            return true;
        }
        if(isWhiteUrl(filterConfig.getWhitePathPre(),path)){
            return true;
        }
        if(isBlackUrl(filterConfig.getBlackPathPre(),path)){
            return false;
        }
        if (StringUtils.isNotBlank(filterConfig.getWhitePathPre())){
            return false;
        }
        if (StringUtils.isNotBlank(filterConfig.getBlackPathPre())){
            return true;
        }
        return true;
    }

    public static boolean isLimitPath(String configPath, String path) {
        if (StringUtils.isBlank(configPath) || StringUtils.isBlank(path)) {
            return false;
        }
        List<String> limitList = Arrays.asList(configPath.split(GlobalConstant.SEPARATOR_COMMA));
        limitList = limitList.stream().filter(limit -> path.equals(limit)).collect(Collectors.toList());
        return limitList.size() > 0;
    }

    public static boolean isForbidPath(String forbidPathPre, String path) {
        if (path.contains("v2/api-docs")){
            return false;
        }
        return isBlackUrl(forbidPathPre,path);
    }

    /**
     * 获取用户真实IP地址，不使用request.getRemoteAddr();的原因是有可能用户使用了代理软件方式避免真实IP地址,
     * <p>
     * 可是，如果通过了多级反向代理的话，X-Forwarded-For的值并不止一个，而是一串IP值，究竟哪个才是真正的用户端的真实IP呢？
     * 答案是取X-Forwarded-For中第一个非unknown的有效IP字符串。
     * <p>
     * 如：X-Forwarded-For：192.168.1.110, 192.168.1.120, 192.168.1.130,
     * 192.168.1.100
     * <p>
     * 用户真实IP为： 192.168.1.110
     *
     * @param request
     * @return
     */
    public static String getIpAddress(ServerHttpRequest request) {
        String ip = request.getHeaders().getFirst("X-Forwarded-For");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("HTTP_CLIENT_IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("HTTP_X_FORWARDED_FOR");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("X-Real-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeaders().getFirst("REMOTE-HOST");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddress().getHostName();
            if (ip.equals("127.0.0.1") || ip.equals("0:0:0:0:0:0:0:1")) {
                try {
                    //       根据网卡取本机配置的IP
                    InetAddress inet = InetAddress.getLocalHost();
                    ip = inet.getHostAddress();
                } catch (java.net.UnknownHostException e) {
                    log.error("java.net.UnknownHostException",e);
                }

            }
        }
        if (ip != null && ip.length() > 15) {
            if (ip.indexOf(",") > 0) {
                ip = ip.substring(0, ip.indexOf(","));
            }
        }
        return ip;
    }

    private static Mono<Void> error(ServerHttpRequest request, ServerHttpResponse response, HttpStatus status,
        String errorMsg) {
        recordLog(request, errorMsg);
        response.setStatusCode(status);
        response.getHeaders().add("Content-Type", "application/json;charset=UTF-8");
        DefaultResult defaultResult = new DefaultResult();
        defaultResult.setCode(String.valueOf(status.value()));
        defaultResult.setMessage(status.getReasonPhrase());

        DataBuffer bodyDataBuffer = response.bufferFactory().wrap(JsonUtil.toJson(defaultResult).getBytes());
        return response.writeWith(Mono.just(bodyDataBuffer));
    }

    public static void recordLog(ServerHttpRequest request, String errorMsg) {
        StringBuilder sb = new StringBuilder();
        sb.append(errorMsg);
        sb.append(",remotIp=").append(getIpAddress(request));
        sb.append(",path=").append(request.getPath().value());
        String method = request.getMethodValue();
        if ("POST".equals(method)) {
            log.error(sb.toString());
        } else if ("GET".equals(method)) {
            log.error(sb.append(",params:").append(request.getQueryParams().isEmpty() ? "" : request.getQueryParams().toString()).toString());
        } else{
            log.error(sb.append(",method=").append(method).toString());
        }
    }
}
```

---

## 3. 配置

```yaml
spring:
  redis:
    host: redis.example-test.com
    port: 6379
    password: ZM4PKwKrtDTFuW29
    pool:
      max-total: 8
      min-idle: 0
      max-active: 8
      max-wait: 10000
  cloud:
    nacos:
      discovery:
        server-addr: config.example-test.com:8848
        username: commonnacos
        password: commonnacos
    gateway:
      routes:
        - id: user-center
          uri: lb://user-center
          predicates:
            - Path=/api/usercenter/**
          filters:
            - name: TokenValidate
              args:
                black-path-pre: /api/usercenter/userinfo
            - name: ExampleRequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100000
                redis-rate-limiter.burstCapacity: 100000
                key-resolver: "#{@pathKeyResolver}"
                limit-path: /api/usercenter/login/send_code,/api/usercenter/login

        - id: user-project
          uri: lb://user-project
          predicates:
            - Path=/api/userproject/**
        
        - id: prescription-project
          uri: lb://prescription-project
          predicates:
            - Path=/api/prescription-project/**
          filters:
            - name: InnerUserToken
              args:
                white-path-pre: /api/prescription-project/v1/storage,/api/prescription-project/v1/prescription/update/doctor,/api/prescription-project/v1/person/health/update/doctor,/api/prescription-project/v1/person/health/update/disease,api/prescription-project/v1/person/health/findById,/api/prescription-project/v1/person/health/findByParams,/api/prescription-project/v1/prescription/count,/api/prescription-project/v1/prescription/personCount,/api/prescription-project/v1/person/count,/api/prescription-project/v1/person/health,/api/prescription-project/v1/signature/callback
                white-header.headerName: platformId
                white-header.headerValue: 2
            - name: TokenValidate
              args:
                white-path-pre: /api/prescription-project/v1/storage,/api/prescription-project/v1/prescription/update/doctor,/api/prescription-project/v1/person/health/update/doctor,/api/prescription-project/v1/person/health/update/disease,api/prescription-project/v1/person/health/findById,/api/prescription-project/v1/person/health/findByParams,/api/prescription-project/v1/prescription/count,/api/prescription-project/v1/prescription/personCount,/api/prescription-project/v1/person/count,/api/prescription-project/v1/person/health,/api/prescription-project/v1/signature/callback
                white-header.headerName: platformId
                white-header.headerValue: 1      
        - id: health-treatment-client
          uri: lb://health-treatment-client
          predicates:
            - Path=/api/health-treatment-client/**
          filters:
            - name: TokenValidate
              args:
                white-path-pre: /api/health-treatment-client/v1/provinces,/api/health-treatment-client/v1/bank/bankInfoQuery,/api/health-treatment-client/v1/health/healthTeamsDoctorQuery,/api/health-treatment-client/v1/health/healthTeamsNurseQuery,/api/health-treatment-client/v1/open/,/api/health-treatment-client/v1/health/healthTeamsPharmacistQuery,/api/health-treatment-client/v1/health/healthTeamsConsultantQuery,/api/health-treatment-client/swagger,/api/health-treatment-client/v2,/api/health-treatment-client/webjars

        - id: hospital-doctor-admin
          uri: lb://hospital-doctor-admin
          predicates:
            - Path=/api/hospital-doctor-admin/**
          filters:
            - name: InnerUserToken
              args:
                black-path-pre: /api/hospital-doctor-admin

        - id: pi-services-frontend
          uri: lb://pi-services-frontend
          predicates:
            - Path=/api/pi/services/**
          filters:
            - name: InnerUserToken
              args:
                white-path-pre: /api/pi/services/user/login

        - id: pi-services-frontend-client
          uri: lb://pi-services-frontend-client
          predicates:
            - Path=/api/pi/client/**
          filters:
            - name: TokenValidate
              args:
                black-path-pre: /api/pi/client/order,/api/pi/client/ware/detail,/api/pi/client/mall/prescription,/api/pi/client/mall/order,/api/pi/client/mall/trades/emptyCart,/api/pi/client/mall/trades/orderProductVerify,/api/pi/client/mall/trades/editShoppingCart,/api/pi/client/mall/card/,/api/pi/client/lvgu

        - id: pi-pharmacy-backstage
          uri: lb://pi-pharmacy-backstage
          predicates:
            - Path=/api/pi/pharmacy/backstage/**

        - id: file-center
          uri: lb://file-center
          predicates:
            - Path=/api/file-center/**

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

---

## 参考资料

a. [Springcloud gateway 官网 link](https://cloud.spring.io/spring-cloud-gateway/reference/html/#developer-guide)
