# nacos

配置中心的使用. :"}

---

### 1. 安装教程

下载 nacos1.3 安装包[github 地址](https://github.com/alibaba/nacos/releases/tag/1.3.0)

```sh
# 解压文件夹,在conf/application.properties文件可以调整nacos的端口
# root @ team3 in /opt/soft/nacos/nacos [18:19:03]
$ ls
LICENSE  NOTICE  bin  conf  startup-nacos.sh  target
```

编写启动脚本`startup-nacos.sh`

```sh
#!/bin/sh

echo -e '\n--- Startup nacos by standalone mode ---\n'

bin/startup.sh -m standalone

echo -e '\n --- Using port:3308 ---\n'
```

启动成功之后,可以通过网页的形式访问 nacos:`http://118.89.113.147:3308/nacos/#/login`

默认账户: `nocos`,默认密码:`nacos`.

---

### 2. 整合 spring boot



---

### 3. 参考文档

a. [nacos 官网](https://nacos.io/zh-cn/docs/quick-start.html)
