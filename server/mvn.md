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

---

## 3. 相关重要参数

### 3.1 dependencyManagement 与 dependencies

1. dependencies: 自动引入声明在 dependencies 里的所有依赖,并默认被所有的子项目继承.如果项目中不写依赖项,则会从父项目继承(属性全部继承)声明在父项目 dependencies 里的依赖项.

2. dependencyManagement: 只是声明依赖(可以理解为只在父项目,外层来声明项目中要引入哪些 jar 包),因此子项目需要显式的声明需要的依赖.如果不在子项目中声明依赖,是不会从父项目中继承的;只有在子项目中写了该依赖项,并且没有指定具体版本,才会从父项目中继承该项,并且 version 和 scope 都读取自父 pom;如果子项目中指定了版本号,那么会使用子项目中指定的 jar 版本.同时 dependencyManagement 让子项目引用依赖,而不用显示的列出版本号.Maven 会沿着父子层次向上走,直到找到一个拥有 dependencyManagement 元素的项目,然后它就会使用在这个 dependencyManagement 元素中指定的版本号,实现所有子项目使用的依赖项为同一版本.

3. dependencyManagement 中的 dependencies 并不影响项目的依赖项;而独立 dependencies 元素则影响项目的依赖项.只有当外层的 dependencies 元素中没有指明版本信息时,dependencyManagement 中的 dependencies 元素才起作用.一个是项目依赖,一个是 maven

### 3.2 profiles

```xml
<profiles>
		<profile>
			<id>dev</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<properties>
				<project.active>dev</project.active>
			</properties>
		</profile>
		<profile>
			<id>test</id>
			<properties>
				<project.active>test</project.active>
			</properties>
		</profile>
</profiles>
```

指定执行环境,如:`mvn clean install -P test`

```sh
mvn clean install -P profileId
```
