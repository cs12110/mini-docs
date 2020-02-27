# Nexus

Maven 搭建私服`Nexus`.

---

## 1. 安装 Nexus

默认用户:`admin`,默认密码:`admin123`.

#### 安装 nexus

```sh
# 解压路径
[root@team-2 mvn-nexus]# pwd
/opt/soft/mvn-nexus
[root@team-2 mvn-nexus]# ls
nexus-2.14  sonatype-work

# 修改默认端口号,可以在conf/nexus.properties,设置端口号为:9400
[root@team-2 mvn-nexus]# cd nexus-2.14/
[root@team-2 nexus-2.14]# ls
bin  conf  lib  LICENSE.txt  logs  nexus  NOTICE.txt  tmp
```

需要设置运行的用户

```sh
[root@team-2 nexus-2.14]# bin/nexus start
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
If you insist running as root, then set the environment variable RUN_AS_USER=root before running this script.

# 设置nexus脚本里面的RUN_AS_USER的值
[root@team-2 nexus-2.14]# vim bin/nexus
#  port needs to be allocated prior to the user being changed.
RUN_AS_USER=root
```

#### 启动 nexus

```sh
# 开启nexus
[root@team-2 nexus-2.14]# bin/nexus start
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
Starting Nexus Repository Manager...
Started Nexus Repository Manager.

# 关闭nexus
[root@team-2 nexus-2.14]# bin/nexus stop
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
Stopping Nexus Repository Manager...
```

请开启开启相关防火墙端口

```sh
[root@team-2 ~]# firewall-cmd --add-port=9400/tcp --zone=public --permanent
success
[root@team-2 ~]# firewall-cmd --reload
success
```

启动后,浏览器访问地址: `http://47.98.104.252:9400/nexus`.

![](imgs/mvn-nexus-home.jpg)

---

## 2. 使用私服

---

## 3. 参考资料

a. [Maven 入门:搭建 Nexus](https://www.cnblogs.com/huangwentian/p/9182819.html)
