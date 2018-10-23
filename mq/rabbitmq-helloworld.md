# RabbitMq 与 Java

嗯,又一个 hello world.

---

## 1. pom.xml

```xml
<dependency>
     <groupId>com.rabbitmq</groupId>
     <artifactId>amqp-client</artifactId>
     <version>4.1.0</version>
</dependency>
```

---

## 2. 系统常量类

```java
public class Const {

      /**
       * rabbit host
       */
      //public final static String RABBIT_HOST = "192.168.43.60";
      public final static String RABBIT_HOST = "192.168.38.239";

      /**
       * rabbit port
       */
      public final static Integer RABBIT_PORT = 5672;

      /**
       * Simple queue name
       */
      public final static String SIMPLE_QUEUE = "simpleQueue";

      /**
       * Work queue name
       */
      public final static String WORKER_QUEUE = "work_queue";
}
```

---

## 3. 连接工具类

```java
/**
 *
 * Rabbit util
 *
 * <p>
 *
 * @author huanghuapeng 2017年6月20日
 * @see
 * @since 1.0
 */
public class RabbitUtil {
      /**
       * 获取Rabbit连接
       *
       * @return Connection
       */
      public static Connection getDefRabbitConnection() {
            Connection rabbitConn = null;
            try {
                  ConnectionFactory factory = new ConnectionFactory();
                  factory.setHost(Const.RABBIT_HOST);
                  factory.setPort(Const.RABBIT_PORT);

                  factory.setVirtualHost("/root");

                  factory.setUsername("root");
                  factory.setPassword("root");

                  rabbitConn = factory.newConnection();
            } catch (Exception e) {
                  e.printStackTrace();
            }
            return rabbitConn;
      }
}
```

---

## 4. 时间工具类

```java
public class TimesUtil {

      /**
       * 休眠
       *
       * @param millSeconds
       */
      public static void sleep(long millSeconds) {
            try {
                  Thread.sleep(millSeconds);
            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
}
```

---

## 5. 生产者

```java
import com.app.util.RabbitUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;

/**
 * 消息生产类
 *
 *
 * <p>
 *
 * @author huanghuapeng 2017年6月24日
 * @see
 * @since 1.0
 */
public class MsgProducer {

      /*
       * 消息队列名称
       */
      private static final String QUEUE_NAME = "tasks";

      /*
       * 标识消息名称
       */
      private String name;

      public MsgProducer(String name) {
            this.name = name;
      }

      /**
       * 发送消息
       */
      public void sendMsgToQueue() throws Exception {
            Connection connection = RabbitUtil.getDefRabbitConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            for (int i = 0; i < 10; i++) {
                  String message = "NO." + i;
                  channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));
                  System.out.println(name + " sending(" + QUEUE_NAME + "): " + message);
            }
            channel.close();
            connection.close();
      }
}
```

---

## 6. 消费者

```java
import java.io.IOException;

import com.app.util.RabbitUtil;
import com.app.util.TimesUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class MyConsumer {

    /**
     * 消息队列名称
     */
    private static final String QUEUE_NAME = "tasks";

    /**
     * 名称
     */
    private String name;

    /**
     * 休眠时间
     */
    private int sleepTime;

    public MyConsumer(String name, int sleepTime) {
        this.name = name;
        this.sleepTime = sleepTime;
    }

    /**
     * 处理消息
     *
     * @throws Exception
     */
    public void process() throws Exception {

        Connection connection = RabbitUtil.getDefRabbitConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        /**
         * 处理消息
         */
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {

                String message = new String(body, "UTF-8");
                System.out.println("\nRecevier_" + name + "(" + QUEUE_NAME + "):" + message);

                TimesUtil.sleep(sleepTime);
            }
        };

        /**
         * 向队列确认消息已被消费
         */
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }

}
```

---

## 7. 测试类

```java
public class WorkRabbitTest {

      /**
       * 在这里,先启动消息消费类监听,然后启动消息生产类
       */
      public static void main(String[] args) throws Exception {

            MyConsumer recv1 = new MyConsumer("A", 200);
            recv1.process();

            /*
             *   在没有设置channel.basicQos(1)设置不同于A的时间间隔,可以
             *
             * 看出,B虽然耗时比A久,但是ta们相互处理对等量的数据.假设queue
             *
             * 里面有10条message,那么A处理5条,B处理5条.
             *
             *
             *   如果设置了channel.basicQos(1),每次消费完一条消息再取,那么
             *
             *时间间隔短的会消费更多的消息.即A消费的消息比B多,不再是多等关系.
             *
             *  Qos的使用,还要设置
             *
             *  channel.basicAck(envelope.getDeliveryTag(), false);
             *
             *  channel.basicConsume(QUEUE_NAME, false, consumer);
             *
             *
             */
            MyConsumer recv2 = new MyConsumer("B", 500);
            recv2.process();

            MsgProducer sender = new MsgProducer("Consumer");
            sender.sendMsgToQueue();

      }
}
```

---

## 8. Qos

### 8.1 修改消费者代码

```java
import java.io.IOException;

import com.app.util.RabbitUtil;
import com.app.util.TimesUtil;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class MyQosConsumer {

    /**
     * 消息队列名称
     */
    private static final String QUEUE_NAME = "tasks";

    /**
     * 名称
     */
    private String name;

    /**
     * 休眠时间
     */
    private int sleepTime;

    public MyQosConsumer(String name, int sleepTime) {
        this.name = name;
        this.sleepTime = sleepTime;
    }

    /**
     * 处理消息
     *
     * @throws Exception
     */
    public void process() throws Exception {

        Connection connection = RabbitUtil.getDefRabbitConnection();
        final Channel channel = connection.createChannel();

        /**
         * 每次消费完一条信息,再取出来
         *
         * 能者多劳
         */
        channel.basicQos(1);

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        /**
         * 处理消息
         */
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {

                String message = new String(body, "UTF-8");
                System.out.println("Recevier_" + name + "(" + QUEUE_NAME + "):" + message);

                /**
                 * 向队列确认消息已被消费,手动确认
                 *
                 * 当队列收到 ack 确认后，会把下一条消息推送过来，并将该消息从队列中删除
                 */
                channel.basicAck(envelope.getDeliveryTag(), false);

                TimesUtil.sleep(sleepTime);
            }
        };

        /**
         * 这里面的autoAck设为false
         */
        channel.basicConsume(QUEUE_NAME, false, consumer);
    }
}
```

### 8.2 测试 Qos

```java
public class WorkRabbitTestQos {

      /**
       * 在这里,先启动消息消费类监听,然后启动消息生产类
       */
      public static void main(String[] args) throws Exception {

            MyQosConsumer qos1 = new MyQosConsumer("A", 200);
            qos1.process();

            MyQosConsumer qos2 = new MyQosConsumer("B", 2000);
            qos2.process();

            MsgProducer sender = new MsgProducer("Consumer");
            sender.sendMsgToQueue();

      }
}
```
