## ollama+dify+deepseek 搭建个人知识库

主要流程,涉及:

- ollama 安装与使用
- docker 安装与使用
- dify 安装与使用

```mermaid
sequenceDiagram
autonumber
    ollama->>ollama: 拉取deepseek模型
    ollama->>ollama: 拉取bge-m3-embeding模型
    ollama->>ollama: 启动deepseek模型,验证是否正常启动
    ollama->>docker: docker下载安装dify
    docker->>dify: docker安装dify
    dify->>dify: dify登录配置用户
    dify->>dify: 配置deepseek模型
    dify->>dify: 配置embeding模型
    dify->>dify: 配置知识库
    dify->>dify: 配置chatbot
    dify->>dify: 发布api提供给外部使用
    dify-->>docker: 成功
    docker-->>ollama: 成功
```

### 1. ollama+deepseek

#### 1.1 ollama 下载模型

ollama 模型获取地址: [ollama deepseek 模型 link](https://ollama.com/search?q=deepseek)

```shell
# mr3306 @ mr3306 in ~/Box/soft/ollama [22:57:03]
$ ollama list
NAME                ID              SIZE      MODIFIED
bge-m3:latest       790764642607    1.2 GB    3 months ago
deepseek-r1:1.5b    a42b25d8c10a    1.1 GB    3 months ago
```

#### 1.2 ollama 运行模型

ollama 运行 deepseek

```shell
# mr3306 @ mr3306 in ~/Box/soft/ollama [17:52:12]
$ ollama list
NAME                ID              SIZE      MODIFIED
bge-m3:latest       790764642607    1.2 GB    3 months ago
deepseek-r1:1.5b    a42b25d8c10a    1.1 GB    3 months ago

# mr3306 @ mr3306 in ~/Box/soft/ollama [17:52:17]
$ ollama run a42b25d8c10a
pulling manifest
Error: pull model manifest: file does not exist

# mr3306 @ mr3306 in ~/Box/soft/ollama [17:52:34] C:1
$ ollama run deepseek-r1:1.5b
>>> who are you
<think>

</think>

Greetings! I'm DeepSeek-R1, an artificial intelligence assistant created by DeepSeek. I'm at
your service and would be delighted to assist you with any inquiries or tasks you may have.
```

验证接口是否能正常使用

```shell
# 验证接口是否能正常使用,注意model名称要为ollama运行的model名称
# mr3306 @ mr3306 in ~/Box/soft/dify-main/docker [17:57:06]
$ curl http://localhost:11434/api/generate -d '{
  "model": "deepseek-r1:1.5b",
  "prompt": "你好，你是谁？"
}'
```

#### 1.3 ollama 常用命令

| 命令/参数            | 描述                                   | 示例                               |
| -------------------- | -------------------------------------- | ---------------------------------- |
| ollama pull <模型名> | 下载指定模型                           | ollama pull llama2                 |
| ollama list          | 列出已下载的模型                       | ollama list                        |
| ollama run <模型名>  | 运行指定模型并进入交互对话模式         | ollama run mistral                 |
| ollama show <模型名> | 显示模型详细信息（许可证、参数等）     | ollama show llama2:13b             |
| ollama rm <模型名>   | 删除本地模型                           | ollama rm codellama:7b             |
| ollama serve         | 启动 Ollama API 服务（默认端口 11434） | ollama serve                       |
| ollama help          | 显示帮助信息                           | ollama help                        |
| ollama create <路径> | 从 Modelfile 创建自定义模型            | ollama create ./my-model           |
| ollama push <模型名> | 推送自定义模型到私有库（需配置）       | ollama push my-org/my-model:latest |
| --verbose            | 显示详细日志（可搭配其他命令使用）     | ollama run llama2 --verbose        |
| --nowrap             | 禁用输出自动换行                       | ollama run mistral --nowrap        |
| --insecure           | 允许非安全连接（如自签名证书）         | ollama serve --insecure            |

### 2. dify 安装与使用

Q: dify 咋读的呀?

A: `Define + Modify`,读:difai

#### 2.1 dify 安装

使用 docker 安装 dify,参考文档: [DeepSeek + Dify ：零成本搭建企业级本地私有化知识库保姆级教程 link](https://juejin.cn/post/7468566313339453480)

```shell
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:02:01]
$ pwd
/Users/mr3306/Box/soft/dify/docker

# 复制配置文件
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:02:02]
$ cp .env.example .env

# docker安装,下载需要等一段时间
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:02:11]
$ docker compose up -d

# 突然就多了好多镜像 orz
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:10:07]
$ docker images
REPOSITORY                      TAG           IMAGE ID       CREATED         SIZE
langgenius/dify-web             1.4.3         2f0bafdb5d6b   3 hours ago     535MB
langgenius/dify-api             1.4.3         5c84264c8d54   3 hours ago     1.98GB
langgenius/dify-plugin-daemon   0.1.2-local   95a00635bf89   5 days ago      1.38GB
postgres                        15-alpine     8c5b83868f93   9 days ago      274MB
redis                           latest        c09c2832ba40   2 weeks ago     128MB
langgenius/dify-sandbox         0.2.12        ba1b250b9505   5 weeks ago     578MB
redis                           6-alpine      70b635414ee2   7 weeks ago     30.2MB
nginx                           latest        1e5f3c5b981a   2 months ago    192MB
ubuntu/squid                    latest        feec1ad0ab0a   2 months ago    213MB
mysql                           5.7           5107333e08a8   18 months ago   501MB
semitechnologies/weaviate       1.19.0        8ec9f084ab23   2 years ago     52.5MB

# 查看docker相关启动容器
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:10:07]
$ docker ps
```

开启/关闭 dify 容器命令如下:

```shell
# 启动dify容器
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:02:11]
$ docker compose up -d

# 关闭dify容器
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:11:19]
$ docker compose down
```

dify 首次配置和安装,浏览器访问地址 [setting link](http://127.0.0.1/install):

```
http://127.0.0.1/install
```

配置邮箱:`cs12110@163.com`, 用户名: `cs12110`, 密码: `dify@112358`

![账号设置](images/account-setting.png)

登录成功之后,进入到 dify 首页

![dify首页](images/dify-home.png)

Q: 上面我们已经安装好了 dify 了,但是要怎么才能连接到我们本地的 ollama 启动 deepseek 呢?

A: 需要修改 dify 的相关配置,然后重启容器服务. 在`.env` 配置文件末尾追加相关配置, 需要特别注意 BASE_URL 的配置:`OLLAMA_API_BASE_URL=host.docker.internal:11434`

```shell
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:26:29]
$ vim .env

# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:27:04]
$ tail .env
# --- 配置私人ollama模型 ---
CUSTOM_MODEL_ENABLED=true
OLLAMA_API_BASE_URL=host.docker.internal:11434

# 重新启动dify相关服务
# 访问地址: http://127.0.0.1/apps
# mr3306 @ mr3306 in ~/Box/soft/dify/docker [18:27:09]
$ docker compose up -d
```

#### 2.2 配置模型

配置 ollama 的 deepseek 和 embeding 模型

![account-model](images/account-model.png)

首次需要先安装 ollama 模型提供者组件

![account-model](images/ollama-model.png)

配置 ollama 里面的 deepseek 模型,<span style="color:red">**注意**配置地址应为: http://host.docker.internal:11434</span>(需要`.env`配置文件里面一致)

![account-model](images/deepseek-model.png)

配置 embeding 模型

![account-model](images/embeding-setting.png)

![account-model](images/embeding-model.png)

#### 2.3 配置知识库

![account-model](images/kwdb-step1.png)

要选择 ollama 里面的嵌入模型:`bge-m3`

![account-model](images/kwdb-step2.png)

等待文件处理成功

![account-model](images/kwdb-step3.png)

Q: 知识库上传文件有大小限制要咋整?

A: 可以在`.env`配置文件里面进行调整. [Deepseek+Dify+Chatbox 应用及 Dify 配置 link](https://www.yorickbao.cn/index.php/archives/1922/)

```shell
# Upload file size limit, default 15M.（上传文件大小设置）
UPLOAD_FILE_SIZE_LIMIT=100

# The maximum number of files that can be uploaded at a time, default 5.（上传文件数量大小）
UPLOAD_FILE_BATCH_LIMIT=100

# The format of the image/video/audio/document sent when the multi-modal model is input,
# the default is base64, optional url.
# The delay of the call in url mode will be lower than that in base64 mode.
# It is generally recommended to use the more compatible base64 mode.
# If configured as url, you need to configure FILES_URL as an externally accessible address so that the multi-modal model can access the image/video/audio/document.
MULTIMODAL_SEND_FORMAT=base64
# Upload image file size limit, default 10M.
UPLOAD_IMAGE_FILE_SIZE_LIMIT=100
# Upload video file size limit, default 100M.
UPLOAD_VIDEO_FILE_SIZE_LIMIT=100
# Upload audio file size limit, default 50M.
UPLOAD_AUDIO_FILE_SIZE_LIMIT=100
```

#### 2.4 配置 chatbot

![account-model](images/studio-setting1.png)

![account-model](images/studio-setting2.png)

![account-model](images/studio-setting3.png)

这里可以看到回答使用了知识库相关文档.

#### 2.5 dify 外部使用

Q: 现在 dify 配置好了,也使用了相关知识库,那要怎么提供给外部使用?

A: 好像和主流的 ai 请求/返回数据格式不一样,应该需要转换.心累.

但起码是可以使用 api 进行调用,测试如下:

```shell
 curl -X POST 'http://127.0.0.1/v1/chat-messages' \
--header 'Authorization: Bearer app-nIWVOF8jO2BQYRGgSyQ4yZ7u' \
--header 'Content-Type: application/json' \
--data-raw '{
    "inputs": {},
    "query": "全链路日志",
    "response_mode": "streaming",
    "conversation_id": "",
    "user": "abc-123",
    "files": [
      {
        "type": "image",
        "transfer_method": "remote_url",
        "url": "https://cloud.dify.ai/logo/logo-site.png"
      }
    ]
}'
```
