# RocketMq

`mq 第一定律`: <u>要有生产者,于是便有了生产者</u>.

`mq 第二定律`: <u>要有消费者,于是便有了消费者</u>.

---

## 1. 基础配置

主要包括 mq 的依赖,配置和工具类等.

### 1.1 pom.xml

本文档使用 ons 依赖,请知悉

```xml
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

## 参考文档

a. [官方文档 link](https://help.aliyun.com/document_detail/114448.html?spm=5176.ons.0.2.b11f176fCYbeCm)

b. 配置参数

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
