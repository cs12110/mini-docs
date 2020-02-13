# Java

本章节主要对 Java 框架知识点进行梳理,如 Spring,mybatis.

---

#### 布隆过滤器

[布隆过滤器原理与使用 link](https://www.jianshu.com/p/2104d11ee0a2)

redis 里面没有自带的布隆过滤器,所以需要安装插件.

```sh
# 安装路径
[root@team-2 module]# pwd
/opt/soft/redis-4.0.8/module

# git下载模块
[root@team-2 module]# git clone https://github.com/RedisBloom/RedisBloom.git
[root@team-2 module]# cd RedisBloom/
[root@team-2 RedisBloom]# make &make install

[root@team-2 RedisBloom]# ls
changelog  contrib  Dockerfile  docs  LICENSE  Makefile  mkdocs.yml  ramp.yml  README.md  redisbloom.so  rmutil  src  tests

# 启动布隆过滤器
[root@team-2 redis-4.0.8]# redis-server ./redis.conf --loadmodule ./module/RedisBloom/redisbloom.so
```

布隆过滤器使用

```sh
# 创建布隆过滤器
47.98.104.252:6336> bf.reserve myboom 0.01 10000
OK

# 新增值
47.98.104.252:6336>

# 判断值是否存在
47.98.104.252:6336> bf.add myboom haiyan
(integer) 1
47.98.104.252:6336> bf.exists myboom haiyan
(integer) 1
47.98.104.252:6336> bf.exists myboom haiyan1
(integer) 0
```
