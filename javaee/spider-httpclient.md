# 爬虫

在生活中,可以用爬虫做一些自己感兴趣的东西,比如爬取知乎的精华回答啦.

所以爬虫,你值得拥有.

这里面不说 Python,只用 Java 来玩一下. :"}

---

## 1. pom.xml

java 里面一般使用 httpclient+jsoup 来做爬虫.

```xml
 <!-- Import http client -->
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.6</version>
</dependency>
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.10.2</version>
</dependency>
```

---

## 2. 代码

在现实使用里面,根据观察,因为爬虫的 get/post 都是差不多,那么我们可以使用模板模式,提高代码复用性.

### 2.1 模板

```java
package com.pkgs.handler;

import org.apache.http.HttpEntity;
import org.apache.http.HttpStatus;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.client.methods.HttpUriRequest;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.message.BasicNameValuePair;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * 工具类
 * <p>
 * FYI:
 * <pre>
 * P: parameter type
 * R: result type
 * </pre>
 *
 * @author cs12110 at 2018年12月10日下午9:40:14
 */
public abstract class AbstractHandler<P, R> {

    /**
     * 日志类
     */
    protected Logger logger = LoggerFactory.getLogger(getClass());

    /**
     * User Agent
     */
    private static final String USER_AGENT = "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.110 Safari/537.36";


    /**
     * 设置值
     *
     * @param value 值
     */
    public void setValue(P value) {
    }

    /**
     * By get
     *
     * @param url url
     */
    public R get(String url) {
        String resultStr = null;

        CloseableHttpClient client = HttpClients.createDefault();
        HttpGet get = new HttpGet(url);
        setUserAgent(get);

        // 使用 try...resource
        try (CloseableHttpResponse result = client.execute(get)) {
            int code = result.getStatusLine().getStatusCode();
            if (code == HttpStatus.SC_OK) {
                HttpEntity entity = result.getEntity();
                resultStr = EntityUtils.toString(entity, "utf-8");
            } else {
                logger.info("failure to get:{},{}", url, result.getStatusLine());
            }

            closeHttpClient(client);
        } catch (Exception e) {
            logger.error("{}", e);
        }

        return parse(resultStr, url);
    }


    /**
     * By post
     *
     * @param url    url
     * @param params 查询参数
     * @return R
     */
    public R post(String url, Map<String, String> params) {
        String resultStr = null;
        CloseableHttpClient client = HttpClients.createDefault();
        HttpPost post = new HttpPost(url);
        setUserAgent(post);

        // 设置请求参数
        if (null != params && params.size() > 0) {
            List<BasicNameValuePair> list = new ArrayList<>();
            for (Map.Entry<String, String> entry : params.entrySet()) {
                list.add(new BasicNameValuePair(entry.getKey(), entry.getValue()));
            }
            try {
                UrlEncodedFormEntity entity = new UrlEncodedFormEntity(list, "utf-8");
                post.setEntity(entity);
            } catch (Exception e) {
                logger.error("{}", e);
            }
        }

        // 请求
        try (CloseableHttpResponse result = client.execute(post)) {
            int code = result.getStatusLine().getStatusCode();
            if (code == HttpStatus.SC_OK) {
                HttpEntity entity = result.getEntity();
                resultStr = EntityUtils.toString(entity, "utf-8");
            } else {
                logger.info("failure to post:{},{}", url, result.getStatusLine());
            }
            closeHttpClient(client);
        } catch (Exception e) {
            logger.error("{}", e);
        }

        return parse(resultStr, url);
    }

    /**
     * 设置user-agent
     *
     * @param req {@link HttpUriRequest}
     */
    private void setUserAgent(HttpUriRequest req) {
        req.setHeader("User-Agent", USER_AGENT);
    }

    /**
     * 关闭http client
     *
     * @param client client
     */
    private static void closeHttpClient(CloseableHttpClient client) {
        if (null != client) {
            try {
                client.close();
            } catch (Exception e) {
                // do nothing
            }
        }
    }

    /**
     * 转换成实体类
     *
     * @param html   html
     * @param reqUrl 请求地址
     * @return R
     */
    public abstract R parse(String html, String reqUrl);
}
```

### 2.2 子类实现

这里只提供一个子类的实现方式,详情请点击 [link](https://github.com/cs12110/4fun/tree/master/4fun-spider/src/main/java/com/pkgs/handler)

```java
package com.pkgs.handler;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import com.pkgs.util.SysUtil;
import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.select.Elements;

import com.pkgs.entity.TopicEntity;

/**
 * 话题转换类
 *
 * @author cs12110 at 2018年12月10日下午9:54:47
 */
public class TopTopicHandler extends AbstractHandler<Object, List<TopicEntity>> {

    @Override
    public List<TopicEntity> parse(String html, String reqUrl) {
        if (null == html) {
            return Collections.emptyList();
        }
        List<TopicEntity> list = new ArrayList<>();
        try {
            Document doc = Jsoup.parse(html);
            Elements topics = doc.select(".zm-topic-cat-item");
            for (Element e : topics) {
                String dataId = e.attr("data-id");
                String name = e.text();
                TopicEntity topic = new TopicEntity();
                topic.setDataId(dataId);
                topic.setName(name);
                topic.setUpdateTime(SysUtil.getTime());
                //设置为已经爬取
                topic.setDone(1);
                list.add(topic);
            }
        } catch (Exception e) {
            logger.error("{}", e);
        }
        return list;
    }
}
```

---

## 3. 参考资料

a. [cs12110的4fun project](https://github.com/cs12110/4fun)