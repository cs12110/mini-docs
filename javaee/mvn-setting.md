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

---

## 4. maven 打包

### 4.1 maven

```xml
<build>
    <resources>
        <!-- 控制资源文件的拷贝 -->
        <resource>
            <directory>src/main/resources</directory>
            <targetPath>${project.build.directory}/classes</targetPath>
        </resource>
    </resources>
    <plugins>
        <!-- 设置源文件编码方式 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>cn.rojao.WebApp</mainClass>    <!-- 入口类名 -->
                </manifest>
                </archive>
                <excludes>
                    <exclude>**/logconfig/*.xml</exclude>
                    <exclude>**/static/**</exclude>
                    <exclude>**/logback-spring.xml</exclude>
                    <exclude>**/application-test.yml</exclude>
                </excludes>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                <id>copy</id>
                <phase>install</phase>
                <goals>
                    <goal>copy-dependencies</goal>
                </goals>
                <configuration>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>  <!-- 拷贝所以依赖存放位置 -->
                </configuration>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-resources</id>
                    <phase>install</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/static</outputDirectory>
                        <resources>
                            <resource>
                                <directory>/src/main/resources/static</directory>
                                <filtering>false</filtering>
                                    <excludes>
                                        <exclude>/src/main/resources/static/public/fonts/**</exclude>
                                    </excludes>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```sh
mvn clean install -Dmaven.test.skip=true
```

### 4.2 assembly

```xml
<build>
    <resources>
        <!-- 控制资源文件的拷贝 -->
        <resource>
            <directory>src/main/resources</directory>
            <targetPath>${project.build.directory}/classes</targetPath>
        </resource>
    </resources>
    <plugins>
        <!-- 设置源文件编码方式 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
        <!-- The configuration of maven-jar-plugin -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>2.4</version>
            <!-- The configuration of the plugin -->
            <configuration>
                <!-- Configuration of the archiver -->
                <archive>

                    <!-- 生成的jar中，不要包含pom.xml和pom.properties这两个文件 -->
                    <addMavenDescriptor>false</addMavenDescriptor>

                    <!-- Manifest specific configuration -->
                    <manifest>
                        <!-- 是否要把第三方jar放到manifest的classpath中 -->
                        <addClasspath>true</addClasspath>
                        <!-- 生成的manifest中classpath的前缀，因为要把第三方jar放到lib目录下，所以classpath的前缀是lib/ -->
                        <classpathPrefix>../lib/</classpathPrefix>
                        <!-- 应用的main class -->
                        <mainClass>cn.rojao.report.ReportApp</mainClass>
                    </manifest>
                </archive>
                <!-- 过滤掉不希望包含在jar中的配置文件 -->
                <excludes>
                    <exclude>**/application.yml</exclude>
                    <exclude>logconfig/*.xml</exclude>
                </excludes>
            </configuration>
        </plugin>

        <!-- The configuration of maven-assembly-plugin -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>2.4</version>
            <!-- The configuration of the plugin -->
            <configuration>
                <!-- Specifies the configuration file of the assembly plugin -->
                <descriptors>
                    <descriptor>src/main/resources/conf/assembly.xml</descriptor>
                </descriptors>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

`assembly.xml`

```xml
<assembly>
    <id>package</id>
    <includeBaseDirectory>false</includeBaseDirectory>
    <!-- 最终打包成一个用于发布的zip文件 -->
    <formats>
        <format>zip</format>
    </formats>
   
    <!-- Adds dependencies to zip package under lib directory -->
    <dependencySets>
        <dependencySet>
            <!--
               不使用项目的artifact，第三方jar不要解压，打包进zip文件的lib目录
           -->
            <useProjectArtifact>false</useProjectArtifact>
            <outputDirectory>lib</outputDirectory>
            <unpack>false</unpack>
        </dependencySet>
    </dependencySets>

    <fileSets>
        <!-- 把项目相关的说明文件，打包进zip文件的根目录 -->
        <!--<fileSet>-->
            <!--<directory>${project.basedir}</directory>-->
            <!--<outputDirectory>/</outputDirectory>-->
        <!--</fileSet>-->
		<fileSet>
            <directory>${project.basedir}/src/bin</directory>
            <outputDirectory>/bin</outputDirectory>
        </fileSet>
        <!-- 把项目的配置文件，打包进zip文件的config目录 -->
        <fileSet>
            <directory>${project.build.directory}/classes</directory>
            <outputDirectory>/config</outputDirectory>
            <includes>
                <include>*.yml</include>
                <include>*.xml</include>
                <include>*.properties</include>
            </includes>
        </fileSet>
        
        <fileSet>
            <directory>${project.build.directory}/classes/conf</directory>
            <outputDirectory>/config</outputDirectory>
            <includes>
                <include>*.xml</include>
                <include>*.properties</include>
            </includes>
        </fileSet>
        
        <fileSet>
            <directory>${project.build.directory}/classes/logconfig</directory>
            <outputDirectory>/logconfig</outputDirectory>
            <includes>
                <include>*.xml</include>
            </includes>
        </fileSet>
        
        
         <fileSet>
            <directory>${project.build.directory}/classes/dic</directory>
            <outputDirectory>/dic</outputDirectory>
            <includes>
                <include>*.dic</include>
            </includes>
        </fileSet>
        
        <fileSet>
            <directory>${project.build.directory}/classes/bin</directory>
            <outputDirectory>/bin</outputDirectory>
            <includes>
                <include>*</include>
            </includes>
        </fileSet>
        
        <!-- 把项目自己编译出来的jar文件，打包进zip文件的根目录 -->
        <fileSet>
            <directory>${project.build.directory}</directory>
            <outputDirectory>/app</outputDirectory>
            <includes>
                <include>*.jar</include>
            </includes>
        </fileSet>
    </fileSets>
</assembly>
```

```sh
mvn clean package -Dmaven.test.skip=true
```
