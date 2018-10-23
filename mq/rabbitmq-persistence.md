# 消息持久化

如果消息没有持久化的话,在 rabbitmq 服务器 down 掉的时候,消息就被被抹除.

使用持久化之后,可以保证数据在服务器 down 机的时候不会被抹除,重启之后会被消费.

---

## 1. 生产者代码

```java
package com.app.durable;

import com.app.util.RabbitUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.MessageProperties;

/**
 *
 * 测试:
 *
 * 当rabbitmq服务器down的时候,没有持久化的数据也会被抹除
 *
 * 所以使用持久化,可以保证数据被消费,也不是被抹除
 *
 * <p>
 *
 * @author huanghuapeng 2017年7月6日
 * @see
 * @since 1.0
 */
public class MsgProducer {

      private static String durableQueueName = "durableQueue";

      private static String normalQueueName = "normalQueue";

      public static void main(String[] args) {
            sendNormal();
            sendDurableMsg();

      }

      /**
       * 持久化mq
       */
      public static void sendDurableMsg() {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();
                  Channel ch = conn.createChannel();

                  /*
                   * 第二个参数设置持久化,设置durable为true,同样在消费端queue的声明也必须为true
                   *
                   * text格式持久化?
                   */
                  ch.queueDeclare(durableQueueName, true, false, false, null);
                  ch.basicPublish("", durableQueueName, MessageProperties.PERSISTENT_TEXT_PLAIN,
                              "This is durable message".getBytes());

                  ch.close();
                  conn.close();

            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
      /**
       * 发送简单的mq
       */
      public static void sendNormal() {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();
                  Channel ch = conn.createChannel();

                  ch.queueDeclare(normalQueueName, false, false, false, null);
                  ch.basicPublish("", normalQueueName, null, "This is durable message".getBytes());

                  ch.close();
                  conn.close();

            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
}
```

---

## 2. 消费者代码

```java
package com.app.durable;

import java.io.IOException;

import com.app.util.RabbitUtil;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;
import com.rabbitmq.client.AMQP.BasicProperties;

public class MsgConsumer {
      private static String queueName = "durableQueue";

      public static void main(String[] args) {
            try {
                  Connection conn = RabbitUtil.getDefRabbitConnection();
                  final Channel ch = conn.createChannel();
                  ch.queueDeclare(queueName, true, false, false, null);
                  Consumer consumer = new DefaultConsumer(ch) {
                        @Override
                        public void handleDelivery(String consumerTag, Envelope envelope, BasicProperties properties,
                                    byte[] body) throws IOException {
                              String msg = new String(body, "UTF-8");
                              System.out.println("Rev: " + msg);
                              ch.basicAck(envelope.getDeliveryTag(), false);
                        }
                  };
                  ch.basicConsume(queueName, false, consumer);
            } catch (Exception e) {
                  e.printStackTrace();
            }
      }
}
```
