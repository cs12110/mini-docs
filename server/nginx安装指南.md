# Nginx 安装指南

在负载均和和反向代理里面,Nginx 是一个不错的选择.

---

## 1. 安装 nginx

### 1.1 安装依赖

```sh
[root@dev-115 soft]# yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel
```

安装 pcre

```sh
[root@dev-115 pcre]# wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz
[root@dev-115 pcre]# tar -xvf pcre-8.35.tar.gz
[root@dev-115 pcre-8.35]# mkdir /opt/soft/nginx/pcre/pcre-8.35/install
[root@dev-115 pcre-8.35]# ./configure --prefix=/opt/soft/nginx/pcre/pcre-8.35/install
[root@dev-115 pcre-8.35]# make && make install
```

### 1.2 安装 nginx

```sh
[root@dev-115 nginx]# wget http://nginx.org/download/nginx-1.6.2.tar.gz
[root@dev-115 nginx-1.6.2]# tar -xvf nginx-1.6.2.tar.gz
[root@dev-115 nginx]# cd nginx-1.6.2
[root@dev-115 nginx-1.6.2]# ./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/opt/soft/nginx/pcre/pcre-8.35/
[root@dev-115 nginx-1.6.2]# make && make install
```

检查 nginx 的版本

```sh
[root@dev-115 nginx]# cd /usr/local/webserver/nginx
[root@dev-115 nginx]# sbin/nginx -v
nginx version: nginx/1.6.2
[root@dev-115 nginx]#
```

启动 nginx

```sh
[root@dev-115 nginx]# sbin/nginx
[root@dev-115 nginx]# curl 127.0.0.1:80
<!DOCTYPE html>
<html>
....
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

可以看出 nginx 已经安装成功了.

---

## 2. 配置 nginx

### 2.1 默认配置

nginx 默认配置文件内容如下

```nginx
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include            mime.types;
    default_type       application/octet-stream;
    sendfile           on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        location = /50x.html {
            root   html;
        }
   }
}
```

语法规则:`location [=|~|~*|^~] /uri/ {...}`

| 符号     | 描述                                                   |
| -------- | ------------------------------------------------------ |
| =        | 表示精确匹配                                           |
| ^~       | 表示 uri 以某个常规字符串开头,理解为匹配 url 路径即可. |
| ~        | 表示区分大小写的正则匹配                               |
| ~\*      | 表示不区分大小写的正则匹配                             |
| !~和!~\* | 分别为区分大小写不匹配及不区分大小写不匹配 的正则      |
| /        | 通用匹配,任何请求都会匹配到.                           |


### 2.2 upstream 模块

在 http 节点下,添加 upstream 节点,格式如下

```nginx
upstream myNginx {
    server 10.0.6.108:7080;
    server 10.0.0.85:8980;
}
```

```nginx
# proxy_pass格式为:http:// + upstreamName,即"http://myNginx".
location / {
    proxy_pass http://myNginx;
}
```

**upstream 负载均衡**:upstream 按照轮询(默认)方式进行负载,每个请求按时间顺序逐一分配到不同的后端服务器,如果后端服务器 down 掉,能自动剔除,适用于图片服务器集群和纯静态页面服务器集群.虽然这种方式简便,成本低廉. 但缺点是:可靠性低和负载分配不均衡.

upstream 其他策略如下:

**weight(权重)**

指定轮询几率,weight 和访问比率成正比,用于后端服务器性能不均的情况.如下所示,10.0.0.88 的访问比率要比 10.0.0.77 的访问比率高一倍.

```nginx
upstream linuxidc{
    server 10.0.0.77 weight=5;
    server 10.0.0.88 weight=10;
}
```

**ip_hash**

每个请求按访问 ip 的 hash 结果分配,这样每个访客固定访问一个后端服务器,可以解决 session 的问题.

```nginx
upstream favresin{
    ip_hash;
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;`
}
```

**fair(第三方)**

按后端服务器的响应时间来分配请求,响应时间短的优先分配.与 weight 分配策略类似.

```nginx
upstream favresin{
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
    fair;
}
```

**url_hash(第三方)**

按 url 的 hash 结果来分配请求,使每个 url 定向到同一个后端服务器,后端服务器为缓存时比较有效.
注意:在 upstream 中加入 hash 语句,server 语句中不能写入 weight 等其他的参数,hash_method 是使用的 hash 算法.

```nginx
upstream resinserver{
    server 10.0.0.10:7777;
    server 10.0.0.11:8888;
    hash \$request_uri;
    hash_method crc32;
}
```

upstream 还可以为每个设备设置状态值,这些状态值的含义分别如下:

- down: 表示单前的 server 暂时不参与负载
- weight: 默认为 1.weight 越大,负载的权重就越大
- max_fails: 允许请求失败的次数默认为 1.当超过最大次数时,返回 proxy_next_upstream 模块定义的错误
- fail_timeout: max_fails 次失败后,暂停的时间
- backup: 其它所有的非 backup 机器 down 或者忙的时候,请求 backup 机器,所以这台机器压力会最轻

实例

```nginx
#定义负载均衡设备的 Ip 及设备状态

upstream bakend{
    ip_hash;
    server 10.0.0.11:9090 down;
    server 10.0.0.11:8080 weight=2;
    server 10.0.0.11:6060;
    server 10.0.0.11:7070 backup;
}
```

---

## 3. 常用命令

```sh
# 检查配置文件是否配置成功
[root@dev-115 nginx]# sbin/nginx -t

# 重新载入配置文
[root@dev-115 sbin]# nginx -s reload

# 重启 Nginx
[root@dev-115 sbin]# nginx -s reopen

# 停止 Nginx
[root@dev-115 sbin]# nginx -s stop
```

---

## 4. 参考资料

a. [Nginx location 匹配规则](https://www.cnblogs.com/lidabo/p/4169396.html)

b. [Nginx 负载均衡配置](http://blog.51cto.com/13178102/2063271)
