# RocketMq

`mq 第一定律`: <u>要有生产者,于是便有了生产者</u>.

`mq 第二定律`: <u>要有消费者,于是便有了消费者</u>.

---

## 1. 基础配置

主要包括 mq 的依赖,配置和工具类等.

FYI: 消息订阅的时候,有两种方式:`集群模式`和`广播模式`,默认为:`集群模式`. u know what I mean.jpg

设置为`集群模式`:

```java
// 集群订阅方式设置（不设置的情况下，默认为集群订阅方式）
properties.put(PropertyKeyConst.MessageModel, PropertyValueConst.CLUSTERING);
```

设置为`广播模式`:

```java
// 广播订阅方式设置
properties.put(PropertyKeyConst.MessageModel, PropertyValueConst.BROADCASTING);
```

### 1.1 pom.xml

本文档使用 ons 依赖,请知悉

```xml
<dependencies>
    <dependency>
        <groupId>com.aliyun.openservices</groupId>
        <artifactId>ons-client</artifactId>
        <version>1.8.0.Final</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.32</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.10</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

### 1.2 rocketmq 连接配置

因为偷懒不想自己搭建,所以在阿里买了一个月 10 块的 mq. 笑哭脸.jpg

如下是 mq 的连接配置等信息

```java
package com.mq.conf;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 17:10
 * <p>
 * @since 1.0.0
 */
public class RocketMqConf {

    /**
     * name server: 为rocketmq控制台里面的`TCP 协议接入点`
     */
    public static final String NAME_SERVER = "http://MQ_INST_1182622285991866_BcLA4qTg.mq-internet-access.mq-internet.aliyuncs.com:80";
    /**
     * group id
     */
    public static final String GID = "GID_3306";

    /**
     * access id
     */
    public static final String ACCESS_ID = "LTAI4Fg5faJ8Faz93mrm****";

    /**
     * access key
     */
    public static final String ACCESS_KEY = "vTxcXoExPjNnKto6KhdfEIirV3****";

    /**
     * topic
     */
    public static final String TOPIC = "rocket-mq-topic";

    /**
     * tag
     */
    public static final String TAGS = "tag-a,tag-b";

}
```

### 1.3 工具类

```java
package com.mq.util;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.function.Supplier;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 17:39
 * <p>
 * @since 1.0.0
 */
public class SysUtil {

    private static Supplier<String> dateSupplier = () -> {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

        return sdf.format(new Date());
    };


    public static String getTime() {
        return dateSupplier.get();
    }
}

```

---

## 2. 生产者[`producer`]

### 2.1 基础知识

使用官方构建方法,可以看见需要哪些必要的参数.

```java
/**
* 创建Producer
*
* <p>
*     <code>properties</code>
*     应该至少包含以下几项必选配置内容:
*     <ol>
*         <li>{@link PropertyKeyConst#ProducerId}</li>
*         <li>{@link PropertyKeyConst#AccessKey}</li>
*         <li>{@link PropertyKeyConst#SecretKey}</li>
*         <li>{@link PropertyKeyConst#ONSAddr}</li>
*     </ol>
*     以下为可选内容
*     <ol>
*         <li>{@link PropertyKeyConst#OnsChannel}</li>
*         <li>{@link PropertyKeyConst#SendMsgTimeoutMillis}</li>
*         <li>{@link PropertyKeyConst#NAMESRV_ADDR} 该属性会覆盖{@link PropertyKeyConst#ONSAddr}</li>
*     </ol>
* </p>
*
* <p>
*     返回创建的{@link Producer}实例是线程安全, 可复用, 发送各个主题. 一般情况下, 一个进程中构建一个实例足够满足发送消息的需求.
* </p>
*
* <p>
*   示例代码:
*   <pre>
*        Properties props = ...;
*        // 设置必要的属性
*        Producer producer = ONSFactory.createProducer(props);
*        producer.start();
*
*        //producer之后可以当成单例使用
*
*        // 发送消息
*        Message msg = ...;
*        SendResult result = produer.send(msg);
*
*        // 应用程序关闭退出时
*        producer.shutdown();
*   </pre>
* </p>
*
* @param properties Producer的配置参数
* @return {@link Producer} 实例
*/
public static Producer createProducer(final Properties properties) {
return onsFactory.createProducer(properties);
}
```

关于有哪些配置,请参考:`PropertyKeyConst`,同时可以参考该文章最后. :"}

### 2.2 构建生产者

```java
package com.mq.producer;

import com.aliyun.openservices.ons.api.*;
import com.mq.conf.RocketMqConf;
import com.mq.util.SysUtil;

import java.util.Properties;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 17:20
 * <p>
 * @since 1.0.0
 */
public class RocketMqProducer {

    public static void main(String[] args) {
        Producer producer = buildMqProducer();
        try {
            //使用前必须开启生产者
            producer.start();
            int times = 10;

            for (int i = 0; i < times; i++) {
                String body = "message:" + i;
                // message key,作为消费者幂等校验,如订单id
                String bizId = "O" + System.currentTimeMillis();

                // 同步发送消息
                Message message = new Message(RocketMqConf.TOPIC, RocketMqConf.TAGS, bizId, body.getBytes());
                SendResult result = producer.send(message);


                display(result);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭生产者
            producer.shutdown();
        }


    }


    /**
     * 打印日志信息
     *
     * @param value 值
     */
    private static void display(Object value) {
        System.out.println(SysUtil.getTime() + " - " + value);
    }

    /**
     * 创建生产者
     *
     * @return {@link Producer}
     */
    private static Producer buildMqProducer() {

        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.NAMESRV_ADDR, RocketMqConf.NAME_SERVER);
        properties.setProperty(PropertyKeyConst.AccessKey, RocketMqConf.ACCESS_ID);
        properties.setProperty(PropertyKeyConst.SecretKey, RocketMqConf.ACCESS_KEY);
        properties.setProperty(PropertyKeyConst.GROUP_ID, RocketMqConf.GID);


        return ONSFactory.createProducer(properties);
    }


}
```

### 2.3 测试结果

```java
2020-02-15 17:51:51 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE596A0000]
2020-02-15 17:51:51 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5A0D0002]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5A410005]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5AC20008]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5AF6000B]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5B2A000E]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5B5D0011]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5B940014]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5BC60017]
2020-02-15 17:51:52 - SendResult[topic=rocket-mq-topic, messageId=C0A801677B6E18B4AAC24BEE5BF9001A]
```

### 2.4 源码

在 rocketmq 里面发送消息会有重试机制. 调用流程:`ProducerImpl#send`->`defaultMQProducer#send` -> `DefaultMQProducerImpl#sendDefaultImpl`

```java
// 获取重试次数,默认同步发送次数为:2+1
// defaultMQProducer.getRetryTimesWhenSendFailed()默认为2
int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
int times = 0;
String[] brokersSent = new String[timesTotal];
for (; times < timesTotal; times++) {
    // 发送成功则break掉,否则continue
}
```

Q: 那么应该在哪里设置这个参数呀?

A: 我也没找到设置的地方. orz

---

## 3. 消费者[`consumer`]

### 3.1 基础知识

```java
/**
* 创建Consumer
* <p>
*     <code>properties</code>应该至少包含以下几项配置内容:
*     <ol>
*         <li>{@link PropertyKeyConst#ConsumerId}</li>
*         <li>{@link PropertyKeyConst#AccessKey}</li>
*         <li>{@link PropertyKeyConst#SecretKey}</li>
*         <li>{@link PropertyKeyConst#ONSAddr}</li>
*     </ol>
*     以下为可选配置项:
*     <ul>
*         <li>{@link PropertyKeyConst#ConsumeThreadNums}</li>
*         <li>{@link PropertyKeyConst#ConsumeTimeout}</li>
*         <li>{@link PropertyKeyConst#OnsChannel}</li>
*     </ul>
* </p>
* @param properties Consumer的配置参数
* @return {@code Consumer} 实例
*/
public static Consumer createConsumer(final Properties properties) {
return onsFactory.createConsumer(properties);
}
```

### 3.2 构建消费者

```java
package com.mq.consumer;

import com.aliyun.openservices.ons.api.*;
import com.mq.conf.RocketMqConf;
import com.mq.util.SysUtil;

import java.util.Properties;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 19:12
 * <p>
 * @since 1.0.0
 */
public class RocketMqConsumer {

    /**
     * 构建消息消费监听器
     */
    public static class MyMessageListener implements MessageListener {

        @Override
        public Action consume(Message message, ConsumeContext context) {

            try {
                String bizId = message.getKey();
                String topic = message.getTopic();
                String tag = message.getTag();
                byte[] body = message.getBody();

                dealWithMessage(topic, tag, bizId, new String(body));
            } catch (Exception e) {
                e.printStackTrace();
                // 出现异常,重新消费
                return Action.ReconsumeLater;
            }

            // 发送ack,确认消息被成功消费
            return Action.CommitMessage;
        }

        /**
         * 根据业务逻辑处理消息
         *
         * @param topic   topic
         * @param tag     tag
         * @param bizId   业务id,用于幂等校验
         * @param message 消息体(建议存储json格式数据)
         */
        private void dealWithMessage(String topic, String tag, String bizId, String message) {
            System.out.println(SysUtil.getTime() + " - topic:" + topic + ",tag:" + tag + ",bizId:" + bizId + ",body:" + message);
        }
    }

    public static void main(String[] args) {

        try {
            Consumer consumer = buildConsumer();

            // 设置消费者监听topic和tag,以及监听处理器
            consumer.subscribe(RocketMqConf.TOPIC, RocketMqConf.TAGS, new MyMessageListener());

            // 开启消费端
            consumer.start();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    /**
     * 创建消息消费端
     *
     * @return {@link Consumer}
     */
    private static Consumer buildConsumer() {
        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.NAMESRV_ADDR, RocketMqConf.NAME_SERVER);
        properties.setProperty(PropertyKeyConst.GROUP_ID, RocketMqConf.GID);
        properties.setProperty(PropertyKeyConst.AccessKey, RocketMqConf.ACCESS_ID);
        properties.setProperty(PropertyKeyConst.SecretKey, RocketMqConf.ACCESS_KEY);

        return ONSFactory.createConsumer(properties);
    }
}
```

### 3.3 测试消费

```java
2020-02-15 19:28:09 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766089083,body:message:0
2020-02-15 19:28:09 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766089918,body:message:1
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766089968,body:message:2
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090018,body:message:3
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090121,body:message:4
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090174,body:message:5
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090232,body:message:6
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090284,body:message:7
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090334,body:message:8
2020-02-15 19:28:10 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:O1581766090383,body:message:9
```

---

## 4. 事务消息

哇咔咔,终于到这里了,我的事务消息.

### 4.1 基础知识

##### 流程图

流程图如下所示:

![](imgs/rocketmq-tx.png)

##### 事务状态

| 状态                                  | 备注                                                                                    |
| ------------------------------------- | --------------------------------------------------------------------------------------- |
| TransactionStatus.CommitTransaction   | 提交事务,允许订阅方消费该消息                                                           |
| TransactionStatus.RollbackTransaction | 回滚事务,消息将被丢弃不允许消费                                                         |
| TransactionStatus.Unknow              | 无法判断状态,期待消息队列 RocketMQ 版的 Broker 向发送方再次询问该消息对应的本地事务的状 |

##### 消息接口

发送事务消息:

```java
/**
* 该方法用来发送一条事务型消息. 一条事务型消息发送分为三个步骤:
* <ol>
*     <li>本服务实现类首先发送一条半消息到到消息服务器;</li>
*     <li>通过<code>executer</code>执行本地事务;</li>
*     <li>根据上一步骤执行结果, 决定发送提交或者回滚第一步发送的半消息;</li>
* </ol>
* @param message 要发送的事务型消息
* @param executer 本地事务执行器
* @param arg 应用自定义参数，该参数可以传入本地事务执行器
* @return 发送结果.
*/
SendResult send(final Message message,final LocalTransactionExecuter executer,final Object arg);
```

### 4.2 构建事务生产者

```java
package com.mq.producer;

import com.alibaba.fastjson.JSON;
import com.aliyun.openservices.ons.api.*;
import com.aliyun.openservices.ons.api.transaction.LocalTransactionChecker;
import com.aliyun.openservices.ons.api.transaction.LocalTransactionExecuter;
import com.aliyun.openservices.ons.api.transaction.TransactionProducer;
import com.aliyun.openservices.ons.api.transaction.TransactionStatus;
import com.mq.conf.RocketMqConf;
import com.mq.util.SysUtil;
import lombok.Data;

import java.util.Properties;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 17:20
 * <p>
 * @since 1.0.0
 */
public class TxRocketMqProducer {

    @Data
    public static class Person {
        private String id;
        private String name;
        private Integer age;

        @Override
        public String toString() {
            return JSON.toJSONString(this);
        }
    }


    /**
     * 模拟现实场景里面spring的service
     */
    public static class BusinessService {

        public boolean save(Person person) {
            // 年龄不合法
            if (person.getAge() == 0) {
                return false;
            }

            return true;
        }

    }


    /**
     * 本地事务回调检查监听器
     */
    public static class MyLocalTransactionChecker implements LocalTransactionChecker {

        @Override
        public TransactionStatus check(Message msg) {

            System.out.println("MyLocalTransactionChecker" + " " + SysUtil.getTime() + " check, message:" + new String(msg.getBody()));

            return TransactionStatus.CommitTransaction;
        }
    }

    /**
     * 本地事务执行器
     */
    @Data
    public static class MyLocalTransactionExecutor implements LocalTransactionExecuter {

        private BusinessService businessService;

        @Override
        public TransactionStatus execute(Message msg, Object arg) {
            TransactionStatus status = TransactionStatus.Unknow;
            try {
                Person p = (Person) arg;

                /*
                 * 当age=1时,返回Unknow状态,broker会回调本地的事务检查监听器
                 *
                 * 当age=0时,因为本地校验参数不合法,返回RollbackTransaction,broker删除半消息
                 *
                 * 当age=2时,本地事务执行完成,半消息投递给消费端
                 */
                if (p.getAge() != 1) {
                    boolean result = businessService.save(p);
                    if (result) {
                        status = TransactionStatus.CommitTransaction;
                    } else {
                        status = TransactionStatus.RollbackTransaction;
                    }
                }

                System.out.println("Deal with:" + arg + " status: " + status);
            } catch (Exception e) {
                e.printStackTrace();
            }

            // 本地事务成功,发送确认消息状态,半消息才会投递到消费者
            return status;
        }
    }

    public static void main(String[] args) {
        TransactionProducer producer = buildMqProducer();

        // 初始化businessService并且设置如executor
        BusinessService businessService = new BusinessService();
        MyLocalTransactionExecutor transactionExecutor = new MyLocalTransactionExecutor();
        transactionExecutor.setBusinessService(businessService);
        try {
            //使用前必须开启生产者
            producer.start();

            for (int index = 0; index < 3; index++) {
                try {
                    Person person = new Person();
                    person.setId("P" + System.currentTimeMillis());
                    person.setAge(index);
                    person.setName("alice" + index);

                    Message message = new Message(RocketMqConf.TOPIC, RocketMqConf.TAGS, person.getId(), JSON.toJSONString(person).getBytes());

                    /*
                     * message: 消息主体
                     *
                     * transactionExecutor: 事务执行器
                     *
                     * arg: 传递到执行器的参数
                     */
                    SendResult result = producer.send(message, transactionExecutor, person);
                    display(result);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 因为事务消息需要回调本地事务检查器,所以不能关闭生产者
            //producer.shutdown();
        }
    }


    /**
     * 打印日志信息
     *
     * @param value 值
     */
    private static void display(Object value) {
        String simpleName = TxRocketMqProducer.class.getSimpleName();
        System.out.println(simpleName + " " + SysUtil.getTime() + " - " + value);
    }

    /**
     * 创建生产者
     *
     * @return {@link Producer}
     */
    private static TransactionProducer buildMqProducer() {

        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.NAMESRV_ADDR, RocketMqConf.NAME_SERVER);
        properties.setProperty(PropertyKeyConst.AccessKey, RocketMqConf.ACCESS_ID);
        properties.setProperty(PropertyKeyConst.SecretKey, RocketMqConf.ACCESS_KEY);
        properties.setProperty(PropertyKeyConst.GROUP_ID, RocketMqConf.GID);

        return ONSFactory.createTransactionProducer(properties, new MyLocalTransactionChecker());
    }

}
```

#### 4.3 测试

开启消费者(因为消费端沿用上面的消费端,所以这里面不重复代码) -> 开启生产者

模拟情况:

- 当 age=0 时,直接发送 rollback,消费端不会接收到该消息.
- 当 age=1 发送 unknown 回调本地事务检查器之后,提交半消息,消费端才能接收消息.
- 当 age=2 直接投递,消费者消费到消息.

生产者打印信息如下:

```java
Deal with:{"age":0,"id":"P1581773152483","name":"alice0"} status: RollbackTransaction
java.lang.RuntimeException: local transaction branch failed ,so transaction rollback
	at com.aliyun.openservices.ons.api.impl.rocketmq.TransactionProducerImpl.send(TransactionProducerImpl.java:135)
	at com.mq.producer.TxRocketMqProducer.main(TxRocketMqProducer.java:135)
Deal with:{"age":1,"id":"P1581773153511","name":"alice1"} status: Unknow
TxRocketMqProducer 2020-02-15 21:25:53 - SendResult[topic=rocket-mq-topic, messageId=C0A80167807A18B4AAC24CB24CE70002]
Deal with:{"age":2,"id":"P1581773153639","name":"alice2"} status: CommitTransaction
TxRocketMqProducer 2020-02-15 21:25:53 - SendResult[topic=rocket-mq-topic, messageId=C0A80167807A18B4AAC24CB24D670005]
MyLocalTransactionChecker 2020-02-15 21:25:57 check, message:{"age":1,"id":"P1581773153511","name":"alice1"}
```

消费者打印日志如下:

```java
RocketMqConsumer 2020-02-15 21:25:53 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:P1581773153639,body:{"age":2,"id":"P1581773153639","name":"alice2"}
RocketMqConsumer 2020-02-15 21:25:57 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:P1581773153511,body:{"age":1,"id":"P1581773153511","name":"alice1"}
```

**结论**: <u>符合 rocketmq 所描述的事务最终一致性</u>.

---

## 5. 顺序消息

在现实环境里面,可能要求消息被按照生产消息顺序的先后被消费.

### 5.1 顺序消息生产者

最大的区别也就是生产者使用的`OrderProducer`,而不是普通的`Producer`.

```java
package com.mq.order.producer;

import com.aliyun.openservices.ons.api.*;
import com.aliyun.openservices.ons.api.order.OrderProducer;
import com.mq.conf.RocketMqConf;
import com.mq.util.SysUtil;

import java.util.Properties;

/**
 * @author huanghuapeng create at 2020/2/20 9:57
 * @version 1.0.0
 */
public class RocketMqOrderProducer {

    public static void main(String[] args) {
        OrderProducer producer = buildMqProducer();
        try {
            //使用前必须开启生产者
            producer.start();
            int times = 10;

            for (int i = 0; i < times; i++) {
                String body = "message:" + i;
                String shardingKey = "key:" + i;
                // message key,作为消费者幂等校验,如订单id
                String bizId = "ORDER-" + i;

                // 同步发送消息
                Message message = new Message(RocketMqConf.TOPIC, RocketMqConf.TAGS, bizId, body.getBytes());
                SendResult result = producer.send(message, shardingKey);

                display(result);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭生产者
            producer.shutdown();
        }
    }


    /**
     * 打印日志信息
     *
     * @param value 值
     */
    private static void display(Object value) {
        System.out.println(SysUtil.getTime() + " - " + value);
    }

    /**
     * 创建顺序消息生产者
     *
     * @return {@link Producer}
     */
    private static OrderProducer buildMqProducer() {

        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.NAMESRV_ADDR, RocketMqConf.NAME_SERVER);
        properties.setProperty(PropertyKeyConst.AccessKey, RocketMqConf.ACCESS_ID);
        properties.setProperty(PropertyKeyConst.SecretKey, RocketMqConf.ACCESS_KEY);
        properties.setProperty(PropertyKeyConst.GROUP_ID, RocketMqConf.GID);
        properties.setProperty("retryTimesWhenSendFailed", "" + 5);


        return ONSFactory.createOrderProducer(properties);
    }

}
```

### 5.2 顺序消息消费者

沿用之前普通消息消费者即可.

### 5.3 测试结果

生产者发送消息

```java
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFA8640000]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAA2D0002]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAA530004]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAA790008]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAAA1000B]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAAC8000E]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAAEF0011]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAB160014]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAB3D0017]
2020-02-20 10:01:39 - SendResult[topic=rocket-mq-topic, messageId=AC10070C147C18B4AAC263FFAB64001A]
```

测试结果

`消费者1`消费消息如下:

```java
RocketMqConsumer1 2020-02-20 10:10:25,738 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-0,body:message:0
RocketMqConsumer1 2020-02-20 10:10:25,791 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-1,body:message:1
RocketMqConsumer1 2020-02-20 10:10:25,828 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-2,body:message:2
```

`消费者2`消费消息如下:

```java
RocketMqConsumer2 2020-02-20 10:10:25,877 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-3,body:message:3
RocketMqConsumer2 2020-02-20 10:10:25,912 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-4,body:message:4
RocketMqConsumer2 2020-02-20 10:10:25,949 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-5,body:message:5
RocketMqConsumer2 2020-02-20 10:10:25,990 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-6,body:message:6
RocketMqConsumer2 2020-02-20 10:10:26,039 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-7,body:message:7
RocketMqConsumer2 2020-02-20 10:10:26,091 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-8,body:message:8
RocketMqConsumer2 2020-02-20 10:10:26,129 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:ORDER-9,body:message:9
```

---

## 6. 定时消息

### 6.1 定时消息生产者

```java
package com.mq.common.producer;

import com.alibaba.fastjson.JSON;
import com.aliyun.openservices.ons.api.*;
import com.mq.conf.RocketMqConf;
import com.mq.util.SysUtil;
import lombok.Data;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Properties;
import java.util.function.Supplier;

/**
 * <p>
 *
 * @author cs12110 create at 2020-02-15 17:20
 * <p>
 * @since 1.0.0
 */
public class RocketMqOnTimeProducer {

    private static Supplier<String> dateSupplier = () -> {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.format(new Date());
    };


    @Data
    private static class MsgInfo {
        private String id;
        private String timestamp;

    }

    public static void main(String[] args) {
        Producer producer = buildMqProducer();
        try {
            //使用前必须开启生产者
            producer.start();
            int times = 10;

            for (int i = 0; i < times; i++) {

                MsgInfo info = new MsgInfo();
                info.setId("OnTime message:" + i);
                info.setTimestamp(dateSupplier.get());

                // message key,作为消费者幂等校验,如订单id
                String bizId = "OT" + System.currentTimeMillis();

                // 同步发送消息
                Message message = new Message(RocketMqConf.TOPIC,
                        RocketMqConf.TAGS, bizId, JSON.toJSONString(info).getBytes());

                /*
                 * 设置5s后投递
                 *
                 * 这里还有种定时的操作,使用SimpleDateFormatter转换日期格式字符串,获取到毫秒数
                 */
                message.setStartDeliverTime(System.currentTimeMillis() + 5000);
                SendResult result = producer.send(message);


                display(message.getMsgID(), result);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 关闭生产者
            producer.shutdown();
        }


    }


    /**
     * 打印日志信息
     *
     * @param value 值
     */
    private static void display(String messageId, Object value) {
        System.out.println(SysUtil.getTime() + " - " + value + ", messageId:" + messageId);
    }

    /**
     * 创建生产者
     *
     * @return {@link Producer}
     */
    private static Producer buildMqProducer() {

        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.NAMESRV_ADDR, RocketMqConf.NAME_SERVER);
        properties.setProperty(PropertyKeyConst.AccessKey, RocketMqConf.ACCESS_ID);
        properties.setProperty(PropertyKeyConst.SecretKey, RocketMqConf.ACCESS_KEY);
        properties.setProperty(PropertyKeyConst.GROUP_ID, RocketMqConf.GID);
        properties.setProperty("retryTimesWhenSendFailed", "" + 5);

        return ONSFactory.createProducer(properties);
    }
}
```

### 6.2 定时消息消费者

使用普通消费者,这里就不累赘重复写代码了,请知悉. :"}

### 6.3 测试结果

生产者发送消息

```java
2020-02-20 22:38:06 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B437C60000]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B4388A0002]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B439120005]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B4397C0008]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B439E9000B]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B43A78000E]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B43ADF0011]
2020-02-20 22:38:07 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B43B680014]
2020-02-20 22:38:08 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B43BCF0017]
2020-02-20 22:38:08 - SendResult[topic=rocket-mq-topic, messageId=C0A801663B5218B4AAC266B43C89001A]
```

消费者消费消息

```java
RocketMqConsumer2 2020-02-20 22:38:11 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209486111,body:{"id":"OnTime message:0","timestamp":"2020-02-20 22:38:06"}
RocketMqConsumer2 2020-02-20 22:38:11 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209486986,body:{"id":"OnTime message:1","timestamp":"2020-02-20 22:38:06"}
RocketMqConsumer2 2020-02-20 22:38:12 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487823,body:{"id":"OnTime message:8","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:12 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209488009,body:{"id":"OnTime message:9","timestamp":"2020-02-20 22:38:08"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487480,body:{"id":"OnTime message:5","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487720,body:{"id":"OnTime message:7","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487336,body:{"id":"OnTime message:4","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487122,body:{"id":"OnTime message:2","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487583,body:{"id":"OnTime message:6","timestamp":"2020-02-20 22:38:07"}
RocketMqConsumer2 2020-02-20 22:38:13 - topic:rocket-mq-topic,tag:tag-a,tag-b,bizId:OT1582209487228,body:{"id":"OnTime message:3","timestamp":"2020-02-20 22:38:07"}
```

可以看出这个定时投递,在消费者消费过程耗时也不一定严格遵守(可能网络传输导致延时什么的),但是应对一般时效性不高的消息,这个也是足够用的.

---

## 7. 参考文档

a. [官方文档 link](https://help.aliyun.com/document_detail/114448.html?spm=5176.ons.0.2.b11f176fCYbeCm)

b. [rocketmq 消息博客](https://www.jianshu.com/p/60528f097223)

c. 配置参数

```java
package com.aliyun.openservices.ons.api;

import javax.annotation.Generated;

@Generated("ons-api")
public class PropertyKeyConst {

    /**
     * 消费模式，包括集群模式、广播模式
     */
    public static final String MessageModel = "MessageModel";

    /**
     * Deprecated，请使用 GROUP_ID 替换
     */
    @Deprecated
    public static final String ProducerId = "ProducerId";

    /**
     * Deprecated，请使用 GROUP_ID 替换
     */
    @Deprecated
    public static final String ConsumerId = "ConsumerId";

    /**
     * Group ID，客户端ID
     */
    public static final String GROUP_ID = "GROUP_ID";
    /**
     * AccessKey, 用于标识、校验用户身份
     */
    public static final String AccessKey = "AccessKey";

    /**
     * SecretKey, 用于标识、校验用户身份
     */
    public static final String SecretKey = "SecretKey";

    /**
     * 使用STS时，需要配置STS Token, 详情参考https://help.aliyun.com/document_detail/28788.html
     */
    public static final String SecurityToken = "SecurityToken";

    /**
     * 消息发送超时时间，如果服务端在配置的对应时间内未ACK，则发送客户端认为该消息发送失败。
     */
    public static final String SendMsgTimeoutMillis = "SendMsgTimeoutMillis";

    /**
     * 消息队列服务接入点
     */
    public static final String ONSAddr = "ONSAddr";

    /**
     * Name Server地址
     */
    public static final String NAMESRV_ADDR = "NAMESRV_ADDR";

    /**
     * 消费线程数量
     */
    public static final String ConsumeThreadNums = "ConsumeThreadNums";

    /**
     * 设置客户端接入来源，默认ALIYUN
     */
    public static final String OnsChannel = "OnsChannel";

    /**
     * 消息类型，可配置为NOTIFY、METAQ
     */
    public static final String MQType = "MQType";
    /**
     * 是否启动vip channel
     */
    public static final String isVipChannelEnabled = "isVipChannelEnabled";

    /**
     * 顺序消息消费失败进行重试前的等待时间 单位(毫秒)
     */
    public static final String SuspendTimeMillis = "suspendTimeMillis";

    /**
     * 消息消费失败时的最大重试次数
     */
    public static final String MaxReconsumeTimes = "maxReconsumeTimes";

    /**
     * 设置每条消息消费的最大超时时间,超过这个时间,这条消息将会被视为消费失败,等下次重新投递再次消费. 每个业务需要设置一个合理的值. 单位(分钟)
     */
    public static final String ConsumeTimeout = "consumeTimeout";
    /**
     * 设置事务消息的第一次回查延迟时间
     */
    public static final String CheckImmunityTimeInSeconds = "CheckImmunityTimeInSeconds";

    /**
     * 是否每次请求都带上最新的订阅关系
     */
    public static final String PostSubscriptionWhenPull = "PostSubscriptionWhenPull";

    /**
     * BatchConsumer每次批量消费的最大消息数量, 默认值为1, 允许自定义范围为[1, 32], 实际消费数量可能小于该值.
     */
    public static final String ConsumeMessageBatchMaxSize = "ConsumeMessageBatchMaxSize";

    /**
     * Consumer允许在客户端中缓存的最大消息数量，默认值为5000，设置过大可能会引起客户端OOM，取值范围为[100, 50000]
     * <p>
     * 考虑到批量拉取，实际最大缓存量会少量超过限定值
     * <p>
     * 该限制在客户端级别生效，限定额会平均分配到订阅的Topic上，比如限制为1000条，订阅2个Topic，每个Topic将限制缓存500条
     */
    public static final String MaxCachedMessageAmount = "maxCachedMessageAmount";

    /**
     * Consumer允许在客户端中缓存的最大消息容量，默认值为512 MiB，设置过大可能会引起客户端OOM，取值范围为[16, 2048]
     * <p>
     * 考虑到批量拉取，实际最大缓存量会少量超过限定值
     * <p>
     * 该限制在客户端级别生效，限定额会平均分配到订阅的Topic上，比如限制为1000MiB，订阅2个Topic，每个Topic将限制缓存500MiB
     */
    public static final String MaxCachedMessageSizeInMiB = "maxCachedMessageSizeInMiB";

    /**
     * 设置实例名，注意：如果在一个进程中将多个Producer或者是多个Consumer设置相同的InstanceName，底层会共享连接。
     */
    public static final String InstanceName = "InstanceName";

    /**
     * MQ消息轨迹开关
     */
    public static final String MsgTraceSwitch = "MsgTraceSwitch";
    /**
     * Mqtt消息序列ID
     */
    public static final String MqttMessageId = "mqttMessageId";

    /**
     * Mqtt消息
     */
    public static final String MqttMessage = "mqttMessage";

    /**
     * Mqtt消息保留关键字
     */
    public static final String MqttPublishRetain = "mqttRetain";

    /**
     * Mqtt消息保留关键字
     */
    public static final String MqttPublishDubFlag = "mqttPublishDubFlag";

    /**
     * Mqtt的二级Topic，是父Topic下的子类
     */
    public static final String MqttSecondTopic = "mqttSecondTopic";

    /**
     * Mqtt协议使用的每个客户端的唯一标识
     */
    public static final String MqttClientId = "clientId";

    /**
     * Mqtt消息传输的数据可靠性级别
     */
    public static final String MqttQOS = "qoslevel";

    /**
     * 设置实例ID，充当命名空间的作用
     */

    public static final String INSTANCE_ID = "INSTANCE_ID";

    /**
     * 是否开启mqtransaction，用于使用exactly-once投递语义
     */
    public static final String EXACTLYONCE_DELIVERY = "exactlyOnceDelivery";

    /**
     * exactlyonceConsumer record manager 刷新过期记录周期
     */
    public static final String EXACTLYONCE_RM_REFRESHINTERVAL = "exactlyOnceRmRefreshInterval";

    /**
     * 每次获取最大消息数量
     */
    public static final String MAX_BATCH_MESSAGE_COUNT = "maxBatchMessageCount";

}
```
