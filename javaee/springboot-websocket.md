# Soringboot-websocket

项目里面遇到,订单更新时,立刻通知管理端的用户的一个需求.

Q: 那你们是怎么实现的呀? :"}

A: 用调度器呀,每分钟查询一次数据库.

Q: 这....

是的,为了生活,就是这么 low,毫无底线.

---

### 1. websocket

Q: 给我翻译,翻译,什么叫websocket,什么tmd叫websocket.

A: 请看 [菜鸟教程 link](https://www.runoob.com/html/html5-websocket.html)

```
WebSocket 是 HTML5 开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。

WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

在 WebSocket API 中，浏览器和服务器只需要做一个握手的动作，然后，浏览器和服务器之间就形成了一条快速通道。两者之间就直接可以数据互相传送。

现在，很多网站为了实现推送技术，所用的技术都是 Ajax 轮询。轮询是在特定的的时间间隔（如每1秒），由浏览器对服务器发出HTTP请求，然后由服务器返回最新的数据给客户端的浏览器。这种传统的模式带来很明显的缺点，即浏览器需要不断的向服务器发出请求，然而HTTP请求可能包含较长的头部，其中真正有效的数据可能只是很小的一部分，显然这样会浪费很多的带宽等资源。

HTML5 定义的 WebSocket 协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。
```

---

### 2. springboot&websocket

演示项目地址: [springboot-websocket github link](https://github.com/cs12110/springboot-websocket)

#### 2.1 重点代码

springboot配置websocket

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

import lombok.extern.slf4j.Slf4j;

/**
 * @author cs12110
 * @version V1.0
 * @since 2020-11-09 16:25
 */
@Slf4j
@Configuration
public class WebsocketConfig {

    @Bean("serverEndpointExporter")
    public ServerEndpointExporter createServerEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

具体服务

```java
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;

import javax.websocket.OnClose;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;

import org.springframework.stereotype.Component;

import lombok.extern.slf4j.Slf4j;

/**
 * 怎么给指定的用户发送指定的消息?
 * <p>
 * 发送给其他用户不在线时,怎么处理?
 *
 * @author cs12110
 * @version V1.0
 * @since 2020-11-09 16:32
 */
@Slf4j
@Component
@ServerEndpoint("/chat/{token}")
public class WebsocketEndpointService {

    private static final Map<String, Session> SOCKET_SESSION_STORAGE = new ConcurrentHashMap<>();

    /**
     * 处理open事件,用户和服务器连接时产生事件,完成之后才能进行通信
     *
     * @param session session
     * @param token   用户token,标志用户是谁
     */
    @OnOpen
    public void handleOpenEvent(Session session, @PathParam("token") String token) {
        log.info("Function[handleOpenEvent] open by:{},token:{}", session.getId(), token);
        SOCKET_SESSION_STORAGE.put(token, session);

        // 返回确认连接信息
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        sendMessage(session, sdf.format(new Date()) + " pong");
    }

    /**
     * 处理关闭事件,例如用户关闭页面时会触发,移除token对应的session
     *
     * @param session session
     * @param token   用户token
     */
    @OnClose
    public void handleCloseEvent(Session session, @PathParam("token") String token) {
        SOCKET_SESSION_STORAGE.remove(token);
        log.info("Function[handleCloseEvent] close by:{}", session.getId());
    }

    /**
     * 消息事件,用户发送消息触发事件
     *
     * @param message 消息体,可以传入json
     * @param session session
     */
    @OnMessage
    public void handleWithMessage(String message, Session session) {
        try {
            log.info("Function[handleWithMessage] sessionId:{}, message:{}", session.getId(), message);
            sendMessage(session, message);
        } catch (Exception e) {
            log.error("Function[handleWithMessage]", e);
        }
    }

    /**
     * 点对点发送消息
     *
     * @param receiverToken 接收用户token
     * @param message       消息体
     */
    public void point(String receiverToken, String message) {
        Session session = SOCKET_SESSION_STORAGE.get(receiverToken);
        sendMessage(session, message);
    }

    /**
     * 广播消息
     *
     * @param message 消息体
     */
    public void broadcast(String message) {
        for (Map.Entry<String, Session> entry : SOCKET_SESSION_STORAGE.entrySet()) {
            sendMessage(entry.getValue(), message);
        }
    }

    /**
     * 发送消息
     *
     * @param session session
     * @param message 消息体
     */
    private void sendMessage(Session session, String message) {
        if (Objects.isNull(session) || !session.isOpen()) {
            log.error("Function[sendMessage]send failure,session is null or session is close");
            return;
        }
        try {
            session.getBasicRemote().sendText(message);
        } catch (Exception e) {
            log.info("Function[sendMessage]", e);
        }
    }

}
```



#### 2.2 测试

首先启动项目,然后使用浏览器打开`resources/web/websocket.html`页面进行操作.

```java

```


---

### 3. 扩展

主要存在问题: 

1. 怎么进行用户和session之间的关联
2. 在用户掉线,重新登入的情况下,怎么实现重发
3. websocket的性能
4. 怎么实现群发和用户分组(好像一个好的数据结构更重要)

如果上面的问题的解答的话,应该能挺好玩的.


---

### 4. 参考资料

a. [菜鸟教程资料 link](https://www.runoob.com/html/html5-websocket.html)

b. [Spring Boot 整合 Websocket 博客link](https://tycoding.cn/2019/06/12/boot/spring-boot-websocket/)