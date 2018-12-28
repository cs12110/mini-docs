# JSON Web Token

在分布式系统里面,这么做权限校验是一个很值得思考的问题.

在这里面我们选取网上所说的 token 来做.

---

## 1. 基础知识

该章节摘录于[CSDN 博客 link](https://blog.csdn.net/u011277123/article/details/78918390)

### 1.1 JWT 的构成

一条 jwt 的 token 如下所示,可以看出由`.`分成三个部分.

- header: 头部信息,构成第一部分
- payload: 携带数据,构成第二部分
- signature: 签证消息,构成第三部分

```java
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwYXlsb2FkIjoie1wibmFtZVwiOlwiaGFpeWFuXCIsXCJpZFwiOjEyMTEwfSIsImV4cCI6MTU0NTk5MzQ2OH0.qvs7rdJDTsoaekBsCSbNlD7Z6cmh1_kGmWluNLWeRL0
```

### 1.2 header

头部承载两部分信息: 声明类型和声明加密算法

这里声明类型为`JWT`,加密算法为:`HS256`.

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

将头部进行 base64 加密(对称加密),构成了第一部分:`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

### 1.3 payload

该部分承载 token 里面有效信息,有效信息包括: `标准注册声明`,`公共声明`,`私有声明`.

**标准注册声明**

| 标志  | 备注                                                        |
| :---: | ----------------------------------------------------------- |
|  iss  | jwt签发者                                                   |
|  sub  | jwt所面向的用户                                             |
|  aud  | 接收jwt的一方                                               |
|  exp  | jwt的过期时间,这个过期时间必须要大于签发时间                |
|  nbf  | 定义在什么时间之前,该jwt都是不可用的                        |
|  iat  | jwt的签发时间                                               |
|  jti  | jwt的唯一身份标识,主要用来作为一次性token,从而回避重放攻击. |

**公共声明**: 公共的声明可以添加任何的信息,一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息,因为该部分在客户端可解密.

**私有声明**: 私有声明是提供者和消费者所共同定义的声明,一般不建议存放敏感信息,因为 base64 是对称解密的,意味着该部分信息可以归类为明文信息.

携带数据如下所示

```json
{
  "name": "haiyan",
  "id": 12110
}
```

经过 base64 加密之后得到第二部分

```java
eyJwYXlsb2FkIjoie1wibmFtZVwiOlwiaGFpeWFuXCIsXCJpZFwiOjEyMTEwfSIsImV4cCI6MTU0NTk5MzQ2OH0
```

### 1.4 signature

签证信息由三部分:`header(base64)+payload(base64)+secret`组成.

```java
qvs7rdJDTsoaekBsCSbNlD7Z6cmh1_kGmWluNLWeRL0
```

---

## 2. pom.xml

代码测试所需依赖 jar 如下所示.

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.4.0</version>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.30</version>
</dependency>
```

---

## 3. Jwt 工具类

Jwt 工具类:提供生成 token,校验,解密,刷新等功能.

```java
import com.alibaba.fastjson.JSON;
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.interfaces.Claim;
import com.auth0.jwt.interfaces.DecodedJWT;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;

/**
 * Jwt工具类
 * <p/>
 *
 * @author cs12110 created at: 2018/12/28 15:47
 * <p>
 * since: 1.0.0
 */
public class JwtUtil {

    /**
     * 加密字符串:盐
     */
    private static String secret = "cs12110";
    /**
     * 加密算法
     */
    private static Algorithm algorithm = Algorithm.HMAC256(secret);


    /**
     * 创建token
     *
     * @param data   数据
     * @param expire 过期时间
     * @return String
     */
    public static String create(Object data, long expire) {
        // 通过数据库校验用户是否通过,通过则进行token的构建
        Map<String, Object> head = new HashMap<>(2);
        head.put("alg", "HS256");
        head.put("typ", "JWT");

        return JWT.create()
                // 头部
                .withHeader(head)
                // payload
                .withClaim("payload", JSON.toJSONString(data))
                // 过时时间
                .withExpiresAt(new Date(System.currentTimeMillis() + expire))
                // 加密
                .sign(algorithm);
    }


    /**
     * 校验token
     *
     * @param token token
     * @return String
     */
    public static String verify(String token) {
        Map<String, Object> map = new HashMap<>(2);
        map.put("status", 200);
        try {
            JWTVerifier verifier = JWT.require(algorithm).build();
            verifier.verify(token);
        } catch (Exception e) {
            //token过期
            e.printStackTrace();

            map.put("status", 404);
            map.put("msg", "token is illegal");
        }
        return map.toString();
    }

    /**
     * 解密
     *
     * @param token token
     * @return String
     */
    public static String decode(String token) {
        String payload = null;
        try {
            JWTVerifier build = JWT.require(algorithm).build();
            DecodedJWT verify = build.verify(token);
            Claim data = verify.getClaim("payload");

            //如果没有值,返回null
            payload = data.asString();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return payload;
    }


    /**
     * 刷新token
     *
     * @param token  token
     * @param expire 过时时间
     * @return String
     */
    public static String refresh(String token, long expire) {
        return create(decode(token), expire);
    }
}
```

---

## 4. 测试类

测试 Jwt 的使用.

### 4.1 校验

```java
@Test
public void verify() {
    Map<String, Object> map = new HashMap<>(2);
    map.put("id", 12110);
    map.put("name", "haiyan");
    //设置1秒过时
    String token = JwtUtil.create(map, 1000);
    //token尚没过时,校验成功
    System.out.println(JwtUtil.verify(token));
    try {
        Thread.sleep(2000);
    } catch (Exception e) {
        e.printStackTrace();
    }
    //token过时抛出异常,并返回失败信息
    System.out.println(JwtUtil.verify(token));
}
```

测试结果

```java
{status=200}
com.auth0.jwt.exceptions.TokenExpiredException: The Token has expired on Fri Dec 28 18:45:05 CST 2018.
	at com.auth0.jwt.JWTVerifier.assertDateIsFuture(JWTVerifier.java:441)
.....
{msg=token is illegal, status=404}
```

### 4.2 加密/解密

```java
@Test
public void encodeAndDecode() {
    Map<String, Object> map = new HashMap<>(2);
    map.put("id", 12110);
    map.put("name", "haiyan");


    //设置1秒过时
    String token = JwtUtil.create(map, 1000);

    //如果过时,解密会抛出异常
    //        try {
    //            Thread.sleep(2000);
    //        } catch (Exception e) {
    //            e.printStackTrace();
    //        }

    String payload = JwtUtil.decode(token);

    System.out.println("token: " + token);
    System.out.println();
    System.out.println("payload: " + payload);
}
```

测试结果

```java
token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwYXlsb2FkIjoie1wibmFtZVwiOlwiaGFpeWFuXCIsXCJpZFwiOjEyMTEwfSIsImV4cCI6MTU0NTk5MzQ2OH0.qvs7rdJDTsoaekBsCSbNlD7Z6cmh1_kGmWluNLWeRL0

payload: {"name":"haiyan","id":12110}
```

### 4.3 刷新 token

刷新 token 是一门学问.

```java
 @Test
public void refresh() {
    Map<String, Object> map = new HashMap<>(2);
    map.put("id", 12110);
    map.put("name", "haiyan");
    //设置1秒过时
    String token = JwtUtil.create(map, 1000);
    //token尚没过时,校验成功
    System.out.println(JwtUtil.verify(token));

    //刷新token,延时5秒
    token = JwtUtil.refresh(token, 5000);
    try {
        Thread.sleep(2000);
    } catch (Exception e) {
        e.printStackTrace();
    }

    //校验通过
    System.out.println(JwtUtil.verify(token));
}
```

测试结果

```java
{status=200}
{status=200}
```

---

## 5. 参考资料

a. [jwt 校验博客](https://blog.csdn.net/u011277123/article/details/78918390)
