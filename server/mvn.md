# maven

maven 呀,maven,你怎么那么不听话呢???

---

## 1. 仓库设置

在国内应该 great wall 的原因,下载某些依赖的时候,会很慢,很慢,很慢.

所以,是有必要更换更快的仓库,那么 ali 的会是一个不错的选择.

```xml

 <mirrors>
     <mirror>
      <id>maven</id>
      <name>central maven</name>
      <url>http://central.maven.org/maven2/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>nexus-aliyun</id>
      <mirrorOf>*</mirrorOf>
      <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
</mirrors>
```

---

## 2. 设备 jdk 版本

设置全局的 jdk 版本,可以节省在每一个项目里面设置的繁琐步骤.

```xml
<profiles>
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
</profiles>
```
