# Rpc

Q: Rpc,如果要自己实现一个简单的rpc该怎么做呢?

A: 码在手,跟我走.

---

## 1. 代码

### 1.1 项目说明

| 模块名称          | 模块说明         |
| ----------------- | ---------------- |
| basic-rpc-api     | 公共接口和类模块 |
| basic-rpc-service | 服务提供模块     |
| basic-rpc-client  | 服务调用模块     |


### 1.2 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>basic-rpc</groupId>
    <artifactId>basic-rpc</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>basic-rpc-api</module>
        <module>basic-rpc-service</module>
        <module>basic-rpc-client</module>
    </modules>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.30</version>
        </dependency>
    </dependencies>
</project>
```

----

## 2. basic-rpc-api

公共api接口和实体类.

### 2.1 api接口

```java
package com.pkgs.api;

import com.pkgs.entity.BookEntity;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:17
 * <p>
 * since: 1.0.0
 */
public interface MyApi {

    /**
     * query by id
     *
     * @param id id
     * @return BookEntity
     */
    public BookEntity queryId(Integer id);
}
```

### 2.2 实体类

```java
package com.pkgs.entity;

import lombok.Data;

import java.io.Serializable;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:17
 * <p>
 * since: 1.0.0
 */
@Data
public class BookEntity implements Serializable {


    private Integer id;

    private String name;

    private String author;
}
```

---

## 3. basic-rpc-service

api实现模块.

### 3.1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>basic-rpc</artifactId>
        <groupId>basic-rpc</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>basic-rpc-service</artifactId>


    <dependencies>
        <dependency>
            <groupId>basic-rpc</groupId>
            <artifactId>basic-rpc-api</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```

### 3.2 impl

```java
package com.pkgs.api.impl;

import com.alibaba.fastjson.JSON;
import com.pkgs.api.MyApi;
import com.pkgs.entity.BookEntity;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:20
 * <p>
 * since: 1.0.0
 */
public class MyApiImpl implements MyApi {


    @Override
    public BookEntity queryId(Integer id) {

        BookEntity entity = new BookEntity();

        if (null != id && id == 1) {
            entity.setId(1);
            entity.setAuthor("马尔克斯");
            entity.setName("霍乱时期的爱情");
        } else {
            entity.setId(-1);
            entity.setAuthor("other");
            entity.setName("other");
        }

        System.out.println(id + " -> " + JSON.toJSONString(entity));

        return entity;
    }
}
```

### 3.3 启动类

```java
package com.pkgs;

import com.pkgs.api.MyApi;
import com.pkgs.api.impl.MyApiImpl;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Method;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:23
 * <p>
 * since: 1.0.0
 */
public class RpcServer {

    public static void main(String[] args) {
        try {
            int listeningPort = 7766;
            ServerSocket serverSocket = new ServerSocket(listeningPort);

            System.out.println("Listening port: " + listeningPort);


            while (true) {
                Socket socket = serverSocket.accept();

                ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());

                // 获取请求调用的数据
                String apiClassName = objectInputStream.readUTF();
                String methodName = objectInputStream.readUTF();
                Class[] parameterTypes = (Class[]) objectInputStream.readObject();
                Object[] argsOfMethod = (Object[]) objectInputStream.readObject();

                Class<?> target = null;
                if (apiClassName.equals(MyApi.class.getCanonicalName())) {
                    target = MyApiImpl.class;
                }

                if (null != target) {
                    Method method = target.getMethod(methodName, parameterTypes);
                    Object result = method.invoke(target.newInstance(), argsOfMethod);

                    ObjectOutputStream outputStream = new ObjectOutputStream(socket.getOutputStream());
                    outputStream.writeObject(result);

                    outputStream.flush();
                }

                objectInputStream.close();
                socket.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## 4. basic-rpc-client

远程调用服务模块.

### 4.1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>basic-rpc</artifactId>
        <groupId>basic-rpc</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>basic-rpc-service</artifactId>


    <dependencies>
        <dependency>
            <groupId>basic-rpc</groupId>
            <artifactId>basic-rpc-api</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```
### 4.2 handler

```java
package com.pkgs.handler;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.net.Socket;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:41
 * <p>
 * since: 1.0.0
 */
public class RpcHandler implements InvocationHandler {

    private Class<?> target;

    public RpcHandler(Class<?> target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {


        Socket socket = new Socket("127.0.0.1", 7766);

        String apiClassName = target.getName();
        String methodName = method.getName();
        Class<?>[] types = method.getParameterTypes();

        ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());

        objectOutputStream.writeUTF(apiClassName);
        objectOutputStream.writeUTF(methodName);
        objectOutputStream.writeObject(types);
        objectOutputStream.writeObject(args);
        objectOutputStream.flush();


        ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
        Object result = objectInputStream.readObject();

        objectInputStream.close();
        objectOutputStream.close();

        return result;
    }
}
```

### 4.3 启动类

```java
package com.pkgs;

import com.alibaba.fastjson.JSON;
import com.pkgs.api.MyApi;
import com.pkgs.entity.BookEntity;
import com.pkgs.handler.RpcHandler;

import java.lang.reflect.Proxy;

/**
 * <p/>
 *
 * @author cs12110 created at: 2019/2/19 17:37
 * <p>
 * since: 1.0.0
 */
public class RemoteClient {

    public static void main(String[] args) {
        MyApi rpc = (MyApi) rpc(MyApi.class);

        BookEntity entity = rpc.queryId(1);
        System.out.println(JSON.toJSONString(entity));

        entity = rpc.queryId(10);
        System.out.println(JSON.toJSONString(entity));

    }


    private static Object rpc(Class<?> clazz) {
        ClassLoader loader = clazz.getClassLoader();

        return Proxy.newProxyInstance(loader, new Class[]{clazz}, new RpcHandler(clazz));
    }
}
```

---

## 5. 测试

温馨提示: **首先运行basic-rpc-service的主类,然后运行basic-rpc-client的主类**.

服务器端输出

```java
Listening port: 7766
1 -> {"author":"马尔克斯","id":1,"name":"霍乱时期的爱情"}
10 -> {"author":"other","id":-1,"name":"other"}
```

调用端输出

```java
{"author":"马尔克斯","id":1,"name":"霍乱时期的爱情"}
{"author":"other","id":-1,"name":"other"}
```

---

## 6. 参考资料

a. [纯手写实现一个高可用的RPC](https://mp.weixin.qq.com/s?__biz=MzAxMjEwMzQ5MA==&mid=2448886008&idx=1&sn=6136ce25d0ec5f276dd71d9a7b225bf3&chksm=8fb552d5b8c2dbc34539968c586131d2e1272ae3e1d0850eec48fc5a828ca2433b29c5a89148&scene=21#wechat_redirect)




