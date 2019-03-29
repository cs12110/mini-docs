# maven 设置

maven 在国内有点坑,在下载某依赖的时候.

所以,我们做一点,小小的,小小的设置.

---

## 1. 修改依赖存放路径

修改`<localRepository>`节点指定依赖存放位置

```xml
<localRepository>D:/Pro/maven/apache-maven-3.3.9/log-repo</localRepository>
```

---

## 2. 更改中心库

修改`<mirrors>`节点,添加如下内容,指定 maven 中心库.

```xml
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
```

---

## 3. 指定 jdk 版本

修改`<profiles>`节点,指定 jdk 版本

```xml
<profile>
    <id>jdk-1.8</id>
    <activation>
    <activeByDefault>true</activeByDefault>
    <jdk>1.8</jdk>
    </activation>
    <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
    </properties>
</profile>
```
