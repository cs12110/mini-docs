# Spark Deps

用于记录 spark 开发所需的 jar 和打包配置,请知悉.

---

## 1. pom.xml

```xml
<?xml version="1.0"?>
<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
         xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>hvs-project</groupId>
        <artifactId>hvs-project</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <artifactId>hvs-engine</artifactId>



    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <encoding>UTF-8</encoding>
        <scala.version>2.11.8</scala.version>
        <spark.version>2.1.0</spark.version>
        <hadoop.version>2.7.3</hadoop.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>



    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>


        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-core_2.10 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-mllib_2.10 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka_2.11 -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-flume_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>${hadoop.version}</version>
        </dependency>
        <dependency>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
            <version>2.6</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.3.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.3.0</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.hbase/hbase-common -->
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-common</artifactId>
            <version>1.3.0</version>
        </dependency>



        <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-streaming-kafka_2.11 -->
        <!-- <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka_2.11</artifactId>
            <version>1.6.3</version>
        </dependency> -->
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka-0-10_2.11</artifactId>
            <version>2.1.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>0.10.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>2.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.10</version>
        </dependency>

        <dependency>
            <groupId>org.spark-project.hive</groupId>
            <artifactId>hive-beeline</artifactId>
            <version>1.2.1.spark2</version>
        </dependency>
        <dependency>
            <groupId>org.spark-project.hive</groupId>
            <artifactId>hive-metastore</artifactId>
            <version>1.2.1.spark2</version>
        </dependency>
        <dependency>
            <groupId>org.spark-project.hive</groupId>
            <artifactId>hive-cli</artifactId>
            <version>1.2.1.spark2</version>
        </dependency>

        <!-- <dependency>
          <groupId>org.apache.kafka</groupId>
          <artifactId>spark-examples_2.11-2.1.0</artifactId>
          <version>${spark.version}</version>
          <scope>system</scope>
          <systemPath>${project.basedir}/lib/spark-examples_2.11-2.1.0.jar</systemPath>
        </dependency> -->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>

        <dependency>
            <groupId>de.ruedigermoeller</groupId>
            <artifactId>fst</artifactId>
            <version>2.50</version>
        </dependency>


        <dependency>
            <groupId>org.codehaus.janino</groupId>
            <artifactId>commons-compiler</artifactId>
            <version>2.6.1</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.41</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.32</version>
        </dependency>
    </dependencies>
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
                            <mainClass>cn.rojao.irs.ire.App</mainClass>

                        </manifest>
                    </archive>
                    <!-- 过滤掉不希望包含在jar中的配置文件 -->
                    <excludes>
                        <exclude>conf/*</exclude>
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
</project>
```

---

## 2. assembly.xml

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
            <directory>${project.build.directory}/classes/conf</directory>
            <outputDirectory>/conf</outputDirectory>
            <includes>
                <include>*.xml</include>
                <include>*.properties</include>
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
