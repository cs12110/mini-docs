# wechat

呦,呦,WeChat 哦.

不得不说,官网的文档很迷,让我一个 cv engineer 都懵逼了.

---

## 1. 接入公众号平台

接入是至关重要的第一步.

- map:获取是微信回传的参数
- value:获取携带的参数,比如什么事件

```java
/**
 * <p>
 *
 * @author cs12110 create at 2019-12-07 20:42
 * <p>
 * @since 1.0.0
 */
@Slf4j
@Controller
@RequestMapping("/wechat/station/")
public class WechatStationController {


    /**
     * wechat echo
     *
     * @param request  http request
     * @param response http response
     * @return Object
     */
    @RequestMapping("/echo")
    @ResponseBody
    public Object echo(HttpServletRequest request, HttpServletResponse response) {

        Enumeration<String> parameterNames = request.getParameterNames();
        Map<String, Object> map = new HashMap<>(5);
        while (parameterNames.hasMoreElements()) {
            String key = parameterNames.nextElement();
            map.put(key, request.getParameter(key));
        }

        String value = HttpUtil.try2Read(request);
        log.info("echo receive map:{},value:{}", JSON.toJSONString(map),value);

        return map.get("echostr");
    }
}
```

---

## 参考资料

a. [公众号文档](https://developers.weixin.qq.com/doc/offiaccount/Getting_Started/Overview.html)

b. [公众号测试账号](https://mp.weixin.qq.com/debug/cgi-bin/sandboxinfo?action=showinfo&t=sandbox/index)
