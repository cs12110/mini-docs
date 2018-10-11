# Redis 数据导入优化

在 Java 使用 jedis 操作 redis 的单个节点时候,可以使用 jedis 的 pipelined 来操作数据,类似于 jdbc 的批量处理来加快速度.

但是由于 JedisCluster 连接 redis 集群的时候,因为没有提供 pipelined,所以在某些导入大量数据的时候,耗时很尴尬.

那么,我们来看看 redis 集群该怎么优化数据导入. :"}

---

## 1. 基础知识

那么可以使用黑魔法, 流程如下:

`从集群里面获取redis的节点信息` -> `根据key计算槽,由槽确定redis节点` -> `获取该节点的jedis连接,通过jedis获取pipelined` -> `使用pipelined操作数据`.

---

## 2. 代码

嗯,是时候了.

### 2.1 pom.xml

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
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
```

### 2.2 RedisInfo 信息对象

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.annotation.JSONField;
import lombok.Getter;
import lombok.Setter;
import redis.clients.jedis.Jedis;

/**
 * Redis信息对象
 * <p/>
 * <p>
 * 使用lombok生成setter/getter,请知悉
 *
 * @author cs12110 created at: 2019/1/24 9:29
 * <p>
 * since: 1.0.0
 */
@Getter
@Setter
public class RedisInfo {

    /**
     * nodeId
     */
    private String nodeId;

    /**
     * IP地址和端口
     */
    private String hostAndPort;

    /**
     * 属于master节点
     */
    private boolean master;

    /**
     * 归属于哪个master节点
     */
    private String masterNodeId;

    /**
     * 是否连接
     */
    private boolean connected;

    /**
     * 槽,格式为: minSlot-maxSlot
     */
    private String slots;

    /**
     * 最小槽值
     */
    private int minSlot;

    /**
     * 最大槽值
     */
    private int maxSlot;

    /**
     * Jedis连接
     */
    @JSONField(serialize = false)
    private Jedis jedis;

    @Override
    public String toString() {
        return JSON.toJSONString(this, true);
    }
}
```

### 2.3 Redis 工具类

```java
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.JedisPool;
import redis.clients.util.JedisClusterCRC16;

import java.util.*;

/**
 * 划分redis存储槽工具类
 * <p>
 * <p/>
 *
 * @author cs12110 created at: 2019/1/24 8:16
 * <p>
 * since: 1.0.0
 */
public class RedisUtil {

    private static JedisCluster clusterConn;
    private static List<RedisInfo> redisInfoList;


    /**
     * 初始化
     *
     * @param cluster 如: "10.10.2.233:7000,10.10.2.233:7001,10.10.2.233:7002,10.10.2.233:7003,10.10.2.233:7004,10.10.2.233:7005"
     */
    public static void init(String cluster) {
        clusterConn = getClusterConn(cluster);
        redisInfoList = buildRedisInfoList(clusterConn);
    }

    /**
     * 获取集群里面的节点信息
     *
     * @return List
     */
    public static List<RedisInfo> getRedisInfoList() {
        return redisInfoList;
    }

    /**
     * 获取redis数据库集群连接
     *
     * @return JedisCluster
     */
    public static JedisCluster getJedisCluster() {
        return clusterConn;
    }

    /**
     * 根据key获取nodeId
     *
     * @param key 存储key
     * @return String
     */
    public static String getRedisNoteIdByKey(String key) {
        int slot = JedisClusterCRC16.getSlot(key);
        String nodeId = null;
        for (RedisInfo info : redisInfoList) {
            if (slot >= info.getMinSlot() && slot <= info.getMaxSlot()) {
                nodeId = info.getNodeId();
                break;
            }
        }
        return nodeId;
    }

    /**
     * 根据nodeId获取redis节点连接
     *
     * @param nodeId nodeId
     * @return Jedis
     */
    public static Jedis getJedisByNodeId(String nodeId) {
        Jedis jedis = null;
        for (RedisInfo info : redisInfoList) {
            if (nodeId.equals(info.getNodeId())) {
                jedis = info.getJedis();
                break;
            }
        }
        return jedis;
    }


    /**
     * 获取{@link RedisInfo} 集合
     * <p>
     * redis集群信息如下所示:
     * <pre>
     * ed043778d0fcec31875047a13da2e20f3de01dde 10.10.2.233:7000 master - 0 1548290362208 1 connected 0-5460
     * ae4f7cc61ff3574ec718bf5552430cd8e82a90d5 10.10.2.233:7004 slave 5b4adecb0b3e00ded33d15f84c00e0f65308c0b6 0 1548290361208 5 connected
     * 5b4adecb0b3e00ded33d15f84c00e0f65308c0b6 10.10.2.233:7001 master - 0 1548290362708 2 connected 5461-10922
     * 30cccbb2ae2e1ffe5c7cd97ef96050dc3b5b7eb3 10.10.2.233:7003 slave ed043778d0fcec31875047a13da2e20f3de01dde 0 1548290360208 4 connected
     * 0c82230779b010201909dcf7cf09a4742a05dfd4 10.10.2.233:7002 myself,master - 0 0 3 connected 10923-16383
     * c6374cce096fd5478fe2f7726100febfd242ae0d 10.10.2.233:7005 slave 0c82230779b010201909dcf7cf09a4742a05dfd4 0 1548290358207 6 connected
     * </pre>
     *
     * @param jedisCluster redis集群连接
     * @return List
     */
    private static List<RedisInfo> buildRedisInfoList(JedisCluster jedisCluster) {
        List<RedisInfo> redisInfoList = new ArrayList<>();
        //获取redis数据库集群节点
        Map<String, JedisPool> clusterNodes = jedisCluster.getClusterNodes();
        for (Map.Entry<String, JedisPool> entry : clusterNodes.entrySet()) {
            //获取jedis
            JedisPool jedisPool = entry.getValue();
            Jedis jedis = jedisPool.getResource();
            String clusterNodesCommand = jedis.clusterNodes();
            String[] nodeInfoArr = clusterNodesCommand.split("\n");
            for (String nodeInfo : nodeInfoArr) {
                if (nodeInfo.contains("myself")) {
                    RedisInfo redisInfo = buildRedisInfo(jedis, nodeInfo);
                    redisInfoList.add(redisInfo);
                }
            }
        }
        return redisInfoList;
    }


    /**
     * 创建redis信息对象
     *
     * @param jedis    jedis连接
     * @param nodeInfo 节点信息字符串
     * @return RedisInfo
     */
    private static RedisInfo buildRedisInfo(Jedis jedis, String nodeInfo) {

        int withSlotInfoLen = 9;

        String[] infoArr = nodeInfo.split(" ");
        RedisInfo redisInfo = new RedisInfo();
        redisInfo.setNodeId(infoArr[0]);
        redisInfo.setHostAndPort(infoArr[1]);
        redisInfo.setMaster(infoArr[2].contains("master"));
        redisInfo.setMasterNodeId(infoArr[3]);
        redisInfo.setConnected("connected".equalsIgnoreCase(infoArr[7]));
        if (infoArr.length == withSlotInfoLen) {
            String[] arr = infoArr[8].split("-");
            int minSlot = Integer.parseInt(arr[0]);
            int maxSlot = Integer.parseInt(arr[1]);

            redisInfo.setSlots(infoArr[8]);
            redisInfo.setMinSlot(minSlot);
            redisInfo.setMaxSlot(maxSlot);
        }
        redisInfo.setJedis(jedis);
        return redisInfo;
    }


    /**
     * 获取redis数据库集群连接
     *
     * @param redisClusterNodeStr redis集群字符串,格式: ip1:port1,ip2:port2,..,ipn:portn
     * @return JedisCluster
     */
    private static JedisCluster getClusterConn(String redisClusterNodeStr) {
        Set<HostAndPort> nodes = new HashSet<>();
        String[] nodeArr = redisClusterNodeStr.split(",");
        for (String each : nodeArr) {
            String str = each.trim();
            String[] infoArr = str.split(":");
            HostAndPort hp = new HostAndPort(infoArr[0], Integer.parseInt(infoArr[1]));
            nodes.add(hp);
        }
        return new JedisCluster(nodes);
    }
}
```

---

## 3. 测试

### 3.1 普通操作

```java
/**
 * <p/>
 *
 * @author cs12110 created at: 2019/1/24 9:42
 * <p>
 * since: 1.0.0
 */
public class RedisNormalTest {

    private static int addTimes = 10000;

    public static void main(String[] args) {
        String cluster = "10.10.2.233:7000,10.10.2.233:7001,10.10.2.233:7002,10.10.2.233:7003,10.10.2.233:7004,10.10.2.233:7005";
        RedisUtil.init(cluster);

        long start = System.currentTimeMillis();
        byNormal();

        long end = System.currentTimeMillis();

        System.out.println("spend: " + (end - start) + " ms");

        clear();
    }

    private static void byNormal() {
        JedisCluster cluster = RedisUtil.getJedisCluster();
        for (int index = 0; index < addTimes; index++) {
            String key = String.valueOf(index);
            String value = String.valueOf(index);
            cluster.set(key, value);
        }
    }


    private static void clear() {
        JedisCluster cluster = RedisUtil.getJedisCluster();

        for (int index = 0; index < addTimes; index++) {
            String key = String.valueOf(index);
            cluster.del(key);
        }
    }
}
```

测试结果

```java
spend: 15759 ms
```

### 3.2 pipelined 操作

```java
/**
 * <p/>
 *
 * @author cs12110 created at: 2019/1/24 9:42
 * <p>
 * since: 1.0.0
 */
public class PipelinedTest {

    private static int addTimes = 10000;

    public static void main(String[] args) {
        String cluster = "10.10.2.233:7000,10.10.2.233:7001,10.10.2.233:7002,10.10.2.233:7003,10.10.2.233:7004,10.10.2.233:7005";
        RedisUtil.init(cluster);

        long start = System.currentTimeMillis();
        byPipeline();

        long end = System.currentTimeMillis();

        System.out.println("spend: " + (end - start) + " ms");

        clear();
    }


    private static void byPipeline() {
        Map<String, List<String>> map = new HashMap<>();
        for (int index = 0; index < addTimes; index++) {
            String key = String.valueOf(index);
            String value = String.valueOf(index);
            String nodeId = RedisUtil.getRedisNoteIdByKey(key);
            List<String> list = map.get(nodeId);

            if (null == list) {
                list = new ArrayList<>();
            }
            list.add(value);
            map.put(nodeId, list);
        }

        for (Map.Entry<String, List<String>> entry : map.entrySet()) {
            String key = entry.getKey();
            List<String> list = entry.getValue();

            Jedis jedis = RedisUtil.getJedisByNodeId(key);
            Pipeline pipelined = jedis.pipelined();

            for (String e : list) {
                pipelined.set(e, e);
            }
            pipelined.sync();
        }
    }

    private static void clear() {
        JedisCluster cluster = RedisUtil.getJedisCluster();

        for (int index = 0; index < addTimes; index++) {
            String key = String.valueOf(index);
            cluster.del(key);
        }
    }
}
```

测试结果

```java
spend: 88 ms
```

### 3.3 总结

窝草,太牛掰,pipelined.

---

## 4. LettuceConnectionFactory

在最新的 LettuceConnectionFactory 里面可以根据如下操作获取 pipeline.

```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.connection.RedisClusterConnection;
import org.springframework.data.redis.connection.RedisClusterNode;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.stereotype.Component;
import redis.clients.jedis.Jedis;
import redis.clients.util.JedisClusterCRC16;

import java.util.ArrayList;
import java.util.List;

/**
 * Redis的速度与激情,你值得拥有.
 * <p>
 * <p>
 * <p/>
 * <pre>
 *
 * 通过JedisClusterCRC16计算key的槽 -> 根据槽获取master节点的id -> 根据id获取master节点的jedis连接 ->
 *
 * 通过jedis获取pipelined -> 通过pipelined操作数据.
 * </pre>
 *
 * @author cs12110 created at: 2019/1/26 14:09
 * <p>
 * since: 1.0.0
 */
@Component
public class FastRedis {

    @Autowired
    private LettuceConnectionFactory lettuceConnectionFactory;

    private List<RedisClusterNode> masterNodeList;

    private volatile boolean isAlreadyInit = false;

    /**
     * 初始化
     */
    private void init() {
        if (!isAlreadyInit) {
            synchronized (FastRedis.class) {
                masterNodeList = getMasterNodeInfo();
                isAlreadyInit = true;
            }
        }
    }

    /**
     * 获取master id
     *
     * @param key key值
     * @return String
     */
    public String getMasterIdByKey(String key) {
        init();
        // 根据key计算槽值
        int slot = JedisClusterCRC16.getSlot(key);

        RedisClusterNode target = null;
        for (RedisClusterNode node : masterNodeList) {
            if (node.getSlotRange().contains(slot)) {
                target = node;
                break;
            }
        }
        return null == target ? null : target.getId();
    }

    /**
     * 获取jedis连接
     *
     * @param masterId masterId
     * @return Jedis
     */
    public Jedis getMasterConn(String masterId) {
        init();
        for (RedisClusterNode node : masterNodeList) {
            if (node.getId().equals(masterId)) {
                return new Jedis(node.getHost(), node.getPort());
            }
        }
        return null;
    }


    /**
     * 获取master节点信息列表
     *
     * @return List
     */
    private List<RedisClusterNode> getMasterNodeInfo() {
        RedisClusterConnection clusterConnection = lettuceConnectionFactory.getClusterConnection();
        Iterable<RedisClusterNode> redisClusterNodes = clusterConnection.clusterGetNodes();
        List<RedisClusterNode> list = new ArrayList<>();
        for (RedisClusterNode node : redisClusterNodes) {
            if (node.isMaster()) {
                list.add(node);
            }
        }
        return list;
    }

}
```

---

## 5. 参考资料

a. [cowboy 的开源项目](https://gitee.com/cowboy2016/springboot2-open)
