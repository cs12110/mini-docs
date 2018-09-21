# Ftp 服务器搭建文档

本文档仅适用于 **centos7** 操作系统.

温馨提示: **关闭 ftp 服务器上的防火墙.**

---

## 1. 安装

`rpm`安装,安装包为: `vsftpd-3.0.2-22.el7.x86_64.rpm`.

```shell
[root@dev-2 ftp]# rpm -ivh vsftpd-3.0.2-22.el7.x86_64.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:vsftpd-3.0.2-22.el7              ################################# [100%]

[root@dev-2 ftp]# systemctl start vsftpd
[root@dev-2 ftp]# systemctl status vsftpd
● vsftpd.service - Vsftpd ftp daemon
   Loaded: loaded (/usr/lib/systemd/system/vsftpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2018-08-24 04:21:32 EDT; 3s ago
  Process: 38165 ExecStart=/usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf (code=exited, status=0/SUCCESS)
 Main PID: 38166 (vsftpd)
   CGroup: /system.slice/vsftpd.service
           └─38166 /usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf
```

---

## 2. 配置

配置 ftp 用户和密码,安装完成后,配置文件放置在`/ect/vsftpd/`目录下

```shell
[root@dev-2 vsftpd]# pwd
/etc/vsftpd
[root@dev-2 vsftpd]# ls
ftpusers  user_list  vsftpd.conf  vsftpd_conf_migrate.sh
```

### 2.1 增加配置

```shell
# 注释
#anonymous_enable=YES


# update by 3306
userlist_enable=YES
userlist_deny=NO
```

### 2.2 新增 ftp 用户

linux 新增`ftpuser`用户,并设置密码为`rojao123`

```shell
[root@dev-2 vsftpd]# useradd  ftpuser
[root@dev-2 vsftpd]# passwd ftpuser
Changing password for user ftpuser.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[root@dev-2 vsftpd]#
```

### 2.3 配置 user_list

配置 ftp 配置文件`user_list`,添加`ftpuser`

```shell
# vsftpd userlist
# If userlist_deny=NO, only allow users in this file
# If userlist_deny=YES (default), never allow users in this file, and
# do not even prompt for a password.
# Note that the default vsftpd pam config also checks /etc/vsftpd/ftpusers
# for users that are denied.
root
bin
daemon
adm
lp
sync
shutdown
halt
mail
news
uucp
operator
games
nobody
ftpuser
```

### 2.4 重启 ftp 服务

```shell
[root@dev-2 vsftpd]# systemctl  restart  vsftpd
```

默认存放存放路径为 ftpuser 的 home 目录: `/home/ftpuser/`

浏览器访问路劲为: `ftp://ip`
