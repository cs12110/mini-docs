# 服务器

本文档主要用于记录服务器相关命令和脚本,请知悉.

以下命令基于`centos7`操作系统.

---

## 1. 设置静态 IP

在生产环境里面,服务器设置静态 ip 地址是必不可少的.

```shell
[root@dev-115 ~]# vi /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE="Ethernet"
# 设置为static
BOOTPROTO="static"
DEFROUTE="yes"
PEERDNS="yes"
PEERROUTES="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_PEERDNS="yes"
IPV6_PEERROUTES="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="873ced93-6651-4be2-927c-b74fc1ebf786"
DEVICE="ens33"
# on boot设置为yes
ONBOOT="yes"

# 配置ip地址,子网掩码,网关和dns服务器
IPADDR=10.33.1.115
NETMASK=255.255.255.255
GATEWAY=10.33.1.1
DNS1=8.8.8.8

# 重启网络服务
[root@dev-115 ~]# systemctl restart network
```

---

## 2. CentOS7 与防火墙

在 CentOS7 里面防火墙的命令变了,很多,很多,很多.

### 2.1 常用命令

**查看状态**

```shell
[root@team-2 ~]# systemctl status firewalld
```

**开启防火墙**

```shell
[root@team-2 ~]# systemctl start firewalld
```

**关闭防火墙**

```shell
[root@team-2 ~]# systemctl stop firewalld
```

**禁用防火墙**

```shell
[root@team-2 ~]# systemctl disable firewalld
```

### 2.2 开启端口

**开启端口**

```shell
[root@team-2 ~]# firewall-cmd --add-port=7003/tcp --zone=public --permanent
[root@team-2 ~]# firewall-cmd --reload
```

**查看已开启端口**

```shell
[root@team-2 ~]# firewall-cmd --list-port
```

**移除端口**

```shell
[root@team-2 ~]# firewall-cmd --remove-port=4993/tcp  --zone=public --permanent
[root@team-2 ~]# firewall-cmd --reload
```

### 2.3 SuSE 关闭防火墙

```shell
hadoop249:/home/bi # rcSuSEfirewall2 stop
```

常用命令

```shell
hadoop249:/home/bi # rcSuSEfirewall2 start
hadoop249:/home/bi # rcSuSEfirewall2 stop
hadoop249:/home/bi # rcSuSEfirewall2 restart
```

---

## 3. 服务器资源

我们该如何查看服务器中的磁盘和内存呢?

### 3.1 磁盘

**查看磁盘容量**

```shell
[root@team-2 ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        40G  7.8G   30G  21% /
devtmpfs        911M     0  911M   0% /dev
tmpfs           920M     0  920M   0% /dev/shm
tmpfs           920M  428K  920M   1% /run
tmpfs           920M     0  920M   0% /sys/fs/cgroup
tmpfs           184M     0  184M   0% /run/user/0
```

**查看当前文件夹占用大小**

```shell
[root@team-2 opt]# du -sh *
189M	pkgs
8.0K	server-doc.md
531M	soft
[root@team-2 opt]#
```

### 3.2 内存

**查看内存**

- `-g`:按照 GB 来显示.
- `-m`:按照 MB 来显示.

```shell
[root@team-2 ~]# free -g
                total        used        free      shared  buff/cache   available
Mem:              1           0           0           0           0           0
Swap:             0           0           0
[root@team-2 ~]# free -m
               total        used        free      shared  buff/cache   available
Mem:           1839         845          86           0         906         810
Swap:           511           0         511
```

### 3.3 top 命令

使用 top 命令可以查看进程的 cpu 和内存占用比

```shell
[root@team-2 ~]# top -u root

Tasks:  73 total,   1 running,  72 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.7 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1883492 total,    88396 free,   866548 used,   928548 buff/cache
KiB Swap:   524284 total,   524284 free,        0 used.   829512 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 2342 root      20   0   37212   9824    404 S  0.3  0.5   8:04.28 Linux2.6
18766 root      20   0   85488    828    404 S  0.3  0.0 123:17.20 slm
31198 root      20   0   75248    764    392 S  0.3  0.0  10:48.20 slm
```

- VIRT: virtual memory usage 虚拟内存
- RES: resident memory usage 常驻内存
- SHR: shared memory 共享内存
- PR: 优先级,会有两种格式,一种是数字(默认 20),一种是 RT 字符串
- NI: nice 值,负值表示高优先级,正值表示低优先级
- S: 进程状态, D=不可终端的睡眠状态,R=运行,S=睡眠,T=跟踪,Z=僵尸进程
- TIME+: 进程使用 cpu 时间,单位 1/100 秒

---

## 4. 查找进程

在服务器中我们该怎么查找到特定的进程?

### 4.1 按端口号

一般`netstat`命令用来查找端口是否被使用,而使用情况要使用`ps`命令来查看.

```shell
[root@team-2 ~]# netstat -lnp|grep 8080
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      11735/java
[root@team-2 ~]# ps -ef|grep 11735
root     11735     1  0 Aug21 ?        00:34:09 /opt/soft/jdk/jdk1.8/bin/java -Djava.util.logging.config.file=/opt/soft/tomcat/tomcat8/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Dignore.endorsed.dirs= -classpath /opt/soft/tomcat/tomcat8/bin/bootstrap.jar:/opt/soft/tomcat/tomcat8/bin/tomcat-juli.jar -Dcatalina.base=/opt/soft/tomcat/tomcat8 -Dcatalina.home=/opt/soft/tomcat/tomcat8 -Djava.io.tmpdir=/opt/soft/tomcat/tomcat8/temp org.apache.catalina.startup.Bootstrap start
```

Q: 哈,那样子怎么 win 上面根据端口查找进程呀?

A: follow me.

```shell
C:\Users\Administrator>netstat -ano|findstr 8800
  TCP    0.0.0.0:8800           0.0.0.0:0              LISTENING       592
  TCP    [::]:8800              [::]:0                 LISTENING       592

C:\Users\Administrator>taskkill /F /PID 592
成功: 已终止 PID 为 592 的进程。
```

### 4.2 按进程名称

- `grep -v 'str'`: 不匹配 str

```shell
[root@team-2 ~]# ps -ef |grep english |grep -v 'grep'
root     30337     1  0 Sep05 ?        00:11:23 java -jar app/english-web-0.0.1-SNAPSHOT.jar
```

### 4.3 jps 命令

该命令为 java 专有,必须安装**jdk 环境**,且显示出来都是 jvm 的进程.

```shell
[root@team-2 ~]# jps -lm
30337 app/english-web-0.0.1-SNAPSHOT.jar
11735 org.apache.catalina.startup.Bootstrap start
20665 sun.tools.jps.Jps -lm
11609 org.fengfei.lanproxy.server.ProxyServerContainer
```

### 4.4 kill

用来杀死进程,用来强制杀死进程!!!

- 命令格式: `kill -9 pid`

```shell
[root@team-2 ~]# kill -9 11735
```

---

## 5. 服务器文件传输

### 5.1 文件压缩

服务器中经常遇到要将多个文件压缩成一个压缩包的情况,仅仅是 tar 只会把文件打包而不会压缩,所以将文件压缩为 gz 的压缩包.

`pkgs.tar.gz`: 自定义压缩包名称.

`pkgs`: 将要被压缩的文件夹.

```shell
[root@team-2 opt]# tar -zcvf pkgs.tar.gz pkgs/
```

### 5.2 文件解压

一般压缩包有两种格式:`tar`和`zip`,嗯,是的,对`rar`很排斥.

**解压 tar**

```shell
[root@team-2 opt]# tar -xvf pkgs.tar.gz
```

**解压 zip**

```shell
[root@team-2 opt]# unzip yourZip.zip
```

### 5.3 服务器之间文件传输

在服务器之间,怎么传输文件呢?不会是用 U 盘 copy 来 copy 去吧?

放心,服务器有`scp`命令, `scp -r pkgs 目标服务器用户@目标服务器ip:存放文件夹`.

- `scp file`: 传输单个文件.
- `scp -r pkgs/`:传输整一个文件夹.

```shell
[root@team-2 ~]# scp -r pkgs/ root@10.10.1.224:/tmp/
```

哈,scp 不仅可以从本地上传文件到服务器,还可以用服务器下载文件到本地.

上传本地文件到服务器

```shell
[root@team-2 ~]# scp -r pkgs/ root@10.10.1.224:/tmp/
```

下载服务器文件到本地

```shell
[root@team-2 ~]# scp -r root@10.10.1.224:/tmp/ pkgs/
```

你看上传和下载刚好文件夹位置反过来,这么方便的 scp,让我觉得那些前辈太厉害.

---

## 6. 服务器用户

### 6.1 新增用户

`useradd -m -d`: 创建并指定 home 目录.

`passwd user`: 为新创建用户创建密码.

```shell
hadoop231:~ # useradd haiyan -m -d /opt/haiyan
hadoop231:~ # passwd haiyan
Changing password for haiyan.
New Password:
Bad password: too simple
Reenter New Password:
Password changed.
```

### 6.2 改变目录拥有者

有些时候,将某些目录赋给某个用户,让用户拥有那个目录的所有权限.

```shell
hadoop231:/opt/soft # mkdir root-dir
hadoop231:/opt/soft # ll
drwxr-xr-x 2 root root 4096 Sep 15 23:15 root-dir
hadoop231:/opt/soft # id haiyan
uid=1001(haiyan) gid=100(users) groups=16(dialout),33(video),100(users)
hadoop231:/opt/soft # chown haiyan:users -R root-dir/
hadoop231:/opt/soft # ll
drwxr-xr-x 2 haiyan users 4096 Sep 15 23:15 root-dir
```

在 linux 里面,一个文件有`r(read)`,`w(write)`,`x(execute)`分别对应着:4,2,1 数值.

那么上面的`drwxr-xr-x`是什么意思呢?

最前面的`d`代表`文件夹`,后面每三个字符为一组,分别代表着:当前用户的拥有的权限,当组用户拥有的权限,其他用户拥有的权限.

---

## 7. maven 打包

### 7.1 普通

下面的命令打包会运行 test 文件夹里面的代码,如果 test 代码要运行很久,请使用`6.2`的打包方式.

```shell
[root@team-2 ~]# mvn clean package
```

### 7.2 跳过测试打包

打包推荐使用如下命令

```shell
[root@team-2 ~]# mvn clean package -Dmaven.test.skip=true
```

### 7.3 模块相互依赖打包

在项目里面,可能会出现`mvn-module1`依赖`mvn-module2`的情况.

如果在`mvn-module1`目录下打包会出现异常,请在`mvn-parent`里面打包.

如下`bi-plugins`有三个子模块`bi-plugins-func`,`bi-plugins-invoke`,`bi-plugins-impl`.

```shell
Administrator@QBKF7V9BTMOUJFI MINGW64 /d/Pro/project/bi/v1.0/trunk/etl/bi-plugins
$ mvn clean package -Dmaven.test.skip=true
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] bi-plugins ......................................... SUCCESS [  0.087 s]
[INFO] bi-plugins-func .................................... SUCCESS [  0.892 s]
[INFO] bi-plugins-invoke .................................. SUCCESS [  1.921 s]
[INFO] bi-plugins-impl .................................... SUCCESS [  0.837 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.889 s
[INFO] Finished at: 2018-10-18T11:07:27+08:00
[INFO] Final Memory: 33M/334M
[INFO] ------------------------------------------------------------------------
```

---

## 8. 时间设置

### 8.1 date

**设置日期**

```shell
hadoop235:/etc # date -s  20181008
```

**设置时间**

```shell
hadoop235:/etc # date -s  10:19:40
```

### 8.2 ntp

集群里面有些节点的时间可能会出现差异,所以使用阿里云的时间服务器同步互联网时间,用来规范集群里面的每一个节点的时间.

Suse:

```shell
hadoop235:/etc # date -s  10:20:10
Mon Oct  8 10:20:10 CST 2018
hadoop235:/etc # date
Mon Oct  8 10:20:17 CST 2018
hadoop235:/etc # sntp -P no -r ntp1.aliyun.com
hadoop235:/etc # date
Mon Oct  8 10:18:34 CST 2018
```

CentOS,依赖`ntp`

```shell
[root@bi143 hadoop-2.7.3]# ntpdate -u ntp1.aliyun.com
```

加入定时任务

```shell
hadoop235:/etc # crontab -e

# 每五分钟同步一次
*/5 * * * * /usr/sbin/sntp -P no -r ntp1.aliyun.com
```

### 8.3 crontab

命令格式: `* * * * * command`

> 分　 时　 日　 月　 周　 命令
>
> 第 1 列表示分钟 1 ～ 59 每分钟用*或者 */1 表示
>
> 第 2 列表示小时 1 ～ 23（0 表示 0 点）
>
> 第 3 列表示日期 1 ～ 31
>
> 第 4 列表示月份 1 ～ 12
>
> 第 5 列标识号星期 0 ～ 6（0 表示星期天）
>
> 第 6 列要运行的命令

相关例子

每 5 分钟执行一次: `*/5 * * * * sh /home/a.sh`

每天凌晨 2 点执行一次: `0 2 * * * sh /home/a.sh`

---

## 9. 软件安装

在 linux 上最方便的两种安装软件的方式,编译的那些都是逆天的. :{

### 9.1 yum

**查找软件**

```shell
[root@hadoop235 ~]# yum list git
[root@hadoop235 ~]# yum list |grep git
```

**安装软件**

```shell
[root@hadoop235 ~]# yum install -y git.x86_64
```

**卸载软件**

```shell
[root@hadoop235 ~]# yum remove -y git.x86_64
```

**查询本地是否安装**

```shell
[root@hadoop235 ~]# yum list  installed |grep git.x86_64
```

**只下载不安装**

```shell
[root@hadoop235 ~]# yum install -y --downloadonly --downloaddir=git-rpm/ git.x86_64
```

### 9.2 rpm

切记: `能升级更新软件,就绝壁不要强制安装了` 笑哭脸.jpg

**安装软件**

```shell
[root@hadoop235 git-rpm]# rpm -ivh perl-Git-1.8.3.1-14.el7_5.noarch.rpm  git-1.8.3.1-14.el7_5.x86_64.rpm
```

**更新软件**

```shell
[root@hadoop235 git-rpm]# rpm -Uvh git-1.8.3.1-14.el7_5.x86_64.rpm
```

**查询软件**

```shell
[root@hadoop235 git-rpm]# rpm -qa|grep git
git-1.8.3.1-14.el7_5.x86_64
```

**卸载软件**

```shell
[root@hadoop235 git-rpm]# rpm -e --nodeps  git-1.8.3.1-14.el7_5.x86_64
```

---

## 10. 修改 hostname

有时候需要修改服务器的主机名称,一般是修改`/etc/hosts`下面的文件,such as

```shell
10.10.1.249     hadoop249
```

但是如果主机之前设置了其他名称,则不会立刻生效,即使使用`service network restart`重启网络服务.

那么我们现在需要使用 hostname 命令来设置一下

```shell
hadoop249:/home/bi/.ssh # hostname hadoop249
hadoop249:/home/bi/.ssh # hostname
hadoop249
```

---

## 11. 找出消耗资源的线程

首先找到 pid

```shell
[bi@bi141 ~]$ jps -lm
11538 org.apache.hadoop.yarn.server.resourcemanager.ResourceManager
12836 org.apache.hadoop.hbase.master.HMaster start
7908 org.apache.hadoop.hdfs.server.namenode.NameNode
8260 org.apache.hadoop.hdfs.qjournal.server.JournalNode
11623 org.apache.hadoop.util.RunJar /opt/bi/kylin/apache-kylin-2.2.0-bin/bin/../tomcat/bin/bootstrap.jar org.apache.catalina.startup.Bootstrap start
1674 org.apache.zookeeper.server.quorum.QuorumPeerMain /opt/bi/zookeeper/zookeeper-3.4.13/bin/../conf/zoo.cfg
18746 org.apache.hadoop.mapreduce.v2.hs.JobHistoryServer
13517 sun.tools.jps.Jps -lm
```

根据 pid 找到进程里面所有的子线程

命令: `top -H -p [pid]`

```shell
[bi@bi143 ~]$ top -H -p 27793
top - 12:02:25 up 6 days,  1:05,  1 user,  load average: 0.01, 0.02, 0.05
Threads:  27 total,   0 running,  27 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.2 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 12304688 total,  4511672 free,  1419176 used,  6373840 buff/cache
KiB Swap:  3905532 total,  3861268 free,    44264 used. 10510164 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
27793 bi        20   0 5997096 136252  19280 S  0.0  1.1   0:00.00 java
27821 bi        20   0 5997096 136252  19280 S  0.0  1.1   0:01.45 java
27822 bi        20   0 5997096 136252  19280 S  0.0  1.1   0:03.68 java
```

假设线程:`27821`占用了 100%的 CPU 资源. 把`27821`转换成 16 进制:`6CAD`.

```java
public class HexStr {
	public static void main(String[] args) {
		int var = 27821;
		String hex = Integer.toHexString(var);
		System.out.println(hex.toUpperCase());
	}
}
```

获取 jvm 里面的堆栈信息

```shell
[bi@bi143 ~]$ jstack 27793 > stackInfo.txt
[bi@bi143 ~]$ ll stackInfo.txt
-rw-rw-r--. 1 bi bi 14236 Nov  6 12:05 stackInfo.txt
[bi@bi143 ~]$ cat stackInfo.txt |grep -i 6CAD
"main" #1 prio=5 os_prio=0 tid=0x00007fba64010000 nid=0x6cad in Object.wait() [0x00007fba6b00f000]
[bi@bi143 ~]$
```

可以看出`27821`是 `main` 线程.

---

## 12. dump

`heap dump` 记录内存信息的,`thread dump` 是记录 CPU 信息的.

### 12.1 head dump

```shell
[root@team-2 ~]# jps -lm
545 sun.tools.jps.Jps -lm
11735 org.apache.catalina.startup.Bootstrap start
11609 org.fengfei.lanproxy.server.ProxyServerContainer
[root@team-2 ~]# jmap -dump:format=b,file=11609.hprof 11609
Dumping heap to /root/11609.hprof ...
Heap dump file created
```

生成 hprof 文件之后,当然要分析啦.

这里面可以使用 jdk 自带的 jhat 来查看,当然建议使用专业的如 Eclipse MAT 来查看.

```shell
[root@team-2 ~]# jhat -port 5500 11609.hprof
Reading from 11609.hprof...
Dump file created Fri Nov 23 23:19:26 CST 2018
Snapshot read, resolving...
Resolving 139016 objects...
Chasing references, expect 27 dots...........................
Eliminating duplicate references...........................
Snapshot resolved.
Started HTTP server on port 5500
Server is ready.
```

### 12.2 thread dump

```shell
[root@team-2 ~]# jps -lm
11735 org.apache.catalina.startup.Bootstrap start
953 sun.tools.jps.Jps -lm
11609 org.fengfei.lanproxy.server.ProxyServerContainer
[root@team-2 ~]# jstack -l 11609 > 11609stack.info
```

---

## 13. 查找进程关联文件

使用`lsof`查找进程关联的文件.

如果服务器尚未安装 lsof,请先安装,命令如下:

```shell
[root@team-2 /]# yum install -y lsof
```

查看进程关联文件

```shell
[root@team-2 mini-docs]# lsof -p 1536
COMMAND  PID USER   FD      TYPE DEVICE SIZE/OFF    NODE NAME
node    1536 root  cwd       DIR  253,1     4096  527608 /opt/soft/docsify/mini-docs
```

---

## 14. 常用脚本

在项目里面需要一个简单的启用脚本的话,这是一个简单的范例. :"}

```bash
#/bin/bash

# app name
app_name='4fun-spider-0.0.1-SNAPSHOT.jar'
num_regex='^[0-9]+$'

# find the pid of app
pid=`ps -ef |grep $app_name |grep -v 'grep' | awk '{print $2}'`

# if exists
if [[ "$pid" =~ $num_regex ]];then
	echo -e "\nWe kill: " $pid " of [" $app_name “]”
	kill -9 $pid
	rm -rf nohup.out logs
fi

# start the app
nohup java -jar app/$app_name &

# sleep five seconds
sleep 5s

pid=`ps -ef |grep $app_name |grep -v 'grep' | awk '{print $2}'`

if [[ "$pid" =~ $num_regex ]];then
        echo -e "\n--- Start [" $app_name "] success ----\n"
	echo -e "\nwith pid: " $pid "\n"
else
	echo -e "\n--- Start [" $app_name "] failure ----\n"
fi
```

---

## 15. 远程监控 jvm

在开启 java 项目的时候,可以开启远程监控,方便查看 jvm 在服务器上面的资源情况.

- Djava.rmi.server.hostname 填写 jvm 所在服务器 ip 地址
- Dcom.sun.management.jmxremote.port 填写监控端口

例子 1:

```bash
nohup java  \
-Dcom.sun.management.jmxremote.rmi.port=9876 \
-Dcom.sun.management.jmxremote.port=9876 \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Djava.rmi.server.hostname="47.98.104.252" \
-jar app/$app_name &
```

例子 2:

```bash
nohup java  \
-Dcom.sun.management.jmxremote.rmi.port=9876 \
-Dcom.sun.management.jmxremote.port=9876 \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Djava.rmi.server.hostname="47.98.104.252" \
-Xms2048M \
-Xmx20M \
-XX:PermSize=1000M \
-XX:MaxPermSize=1048M \
-XX:MaxDirectMemorySize=64M \
-jar *.jar  &
```

Q: 为什么在阿里云上面开启这个检测之后,使用远程连接,连接不上?

A: 因为除了上面的那端口,还有其他通讯端口. ~~,需要在防火墙上开启出来.~~

```shell
[root@team-2 4fun-spider]# jps  -lm |grep 4fun-spider
21825 app/4fun-spider-0.0.1-SNAPSHOT.jar
[root@team-2 4fun-spider]# netstat -lnp|grep 21825
tcp        0      0 0.0.0.0:9876            0.0.0.0:*               LISTEN      21825/java
tcp        0      0 0.0.0.0:37277           0.0.0.0:*               LISTEN      21825/java
```

As you can see, 开启`9876`端口 ~~,还开启了`37277`~~(据说随机开启,冷漠脸).

netstat 命令解释:

- -n 表示表示输出中不显示主机,端口和用户名.
- -lb 表示只显示监听 listening 端口.
- -t 表示只显示 tcp 协议的端口.
- -p 表示显示进程的 PID 和进程名称.

---

## 16. wget&curl

### 16.1 wget

Q: 如果在服务器要下载网上的资源链接该怎么办呢?

A: wget,你值得拥有.命令格式: `wget 'resourceUrl' -O outputFileName`,resourceUrl 使用`'`来包住(`-O outputFileName`为可选命令,O 为大写字母).

比如使用服务器下载`vscode`:

```shell
[root@team-2 ~]# wget 'https://vscode.cdn.azure.cn/stable/05f146c7a8f7f78e80261aa3b2a2e642586f9eb3/VSCode-win32-x64-1.32.1.zip'
```

在网络不稳定的情况下,wget 会出现断开的可能,又要重新下载,这叫我情何以堪.

所以可以使用 wget 的断点续传功能: `wget -c fileUrl`

发现下载任务停止之后,使用`ctrl+z`(不是 ctrl+c)暂停任务,然后再使用`wget -c fileUrl`来续传下载文件.

```shell
[root@team-2 ~]# wget -c 'https://vscode.cdn.azure.cn/stable/05f146c7a8f7f78e80261aa3b2a2e642586f9eb3/VSCode-win32-x64-1.32.1.zip'
```

### 16.2 curl

Q: 在服务器上没有浏览器,该怎么判断服务器上的接口呢?

A: 都长这么大了,要自己学会 curl 了.

```shell
[root@team-2 ~]# curl '127.0.0.1:8081/rest/answers?topicId=35&pageIndex=0&pageSize=5'
```

Q: 要是 post 需要带参数怎么办呀?

A: 例子如下

```shell
[root@team-2 ~]# curl -XPOST 'http://10.33.1.111:9200/movie_lib/movies/_delete_by_query?pretty' -H 'Content-Type: application/json' -d '
{
  "query": {
    "match_all": {}
  }
}'
```

---

## 17. nginx

自从有了服务器之后,就有点飘了.什么域名啦,什么 nginx 啦,什么 Tomcat 啦,什么 mysql 啦,全都 tmd 给我加上.万事有个 but,卧槽.

Q: 在 Tomcat 里面启动的项目,怎么使用域名访问,而不是每次都是些 ip?

A: 假设服务器的 tomcat 项目的访问地址为:`http://47.98.104.252:8080/schedule/views/Schedule.jsp`,那么我们需要在 nginx 加上这个项目的代理即可.

```nginx
location /schedule/ {
  proxy_pass http://127.0.0.1:8080/schedule/;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto https;
}
```

---
