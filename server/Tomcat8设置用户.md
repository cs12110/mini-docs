# Tomcat8 设置用户

该文档仅适用于 tomcat8 用于设置 tomcat 用户,请知悉.

## 1. 配置 manager.xml

### 1.1 配置 manager.xml

因为 tomcat 默认的配置,不允许远程连接管理 tomcat,需要在目录`conf/Catalina/localhost`下配置一个`manager.xml`,内容如下

```xml
<Context privileged="true" antiResourceLocking="false"
         docBase="${catalina.home}/webapps/manager">
             <Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="^.*$" />
</Context>
```

### 1.2 配置访问用户和密码

在`conf/tomcat-users.xml`节点`<tomcat-users>...</tomcat-users>`添加如下内容.

```xml
<role rolename="manager-gui"/>
<user username="tomcat" password="tomcat" roles="manager-gui"/>
```

---

## 2. 重启 tomcat

```shell
bin/shutdown.sh

bin/startup.sh
```

---

## 3. 测试

![tomcat status](imgs/tomcat.png)
