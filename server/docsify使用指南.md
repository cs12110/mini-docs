# Docsify 使用指南

docsify 一个类似 gitbook 一样的文档展示工具,你值得拥有.

---

## 1. 安装 docsify

该章节描述 docsify 的安装.

### 1.1 安装 nodejs

因为 docsify 依赖 nodejs,我们先安装 nodejs.

下载 nodejs 压缩包,解压并且设置环境变量即可.

```bash
[root@team-2 nodejs]# wget https://nodejs.org/dist/v10.13.0/node-v10.13.0-linux-x64.tar.xz
[root@team-2 nodejs]# ls
node-v10.13.0-linux-x64.tar.xz
[root@team-2 nodejs]# tar -xvf node-v10.13.0-linux-x64.tar.xz
```

设置环境变量

```bash
# nodejs env by 3306 at 2018-11-10
export NODEJS_HOME='/opt/soft/nodejs/node-v10.13.0-linux-x64/'
export PATH=$PATH:$NODEJS_HOME/bin
```

使用`source /etc/profile`重新加载环境变量即可.

```sh
[root@team-2 nodejs]# node -v
v10.13.0
[root@team-2 nodejs]# npm -v
6.4.1
```

### 1.2 安装 docsify

这个在网上都是使用`npm i docsify-cli -g`,但是出现了异常.

解决方法是进入到 npm 所在位置进行安装,如`lib/node_modules/`.

```sh
[root@team-2 node-v10.13.0-linux-x64]# ls -al  bin/npm
lrwxrwxrwx 1 500 500 38 Oct 30 15:48 bin/npm -> ../lib/node_modules/npm/bin/npm-cli.js
[root@team-2 node-v10.13.0-linux-x64]# cd lib/node_modules/
[root@team-2 node_modules]# npm i docsify -g
> docsify@4.8.5 postinstall /usr/lib/node_modules/docsify
> opencollective postinstall


                          Thanks for installing docsify
                 Please consider donating to our open collective
                        to help us maintain this package.

                           Number of contributors: 112
                               Number of backers: 6
                               Annual budget: $169
                              Current balance: $154

             Donate: https://opencollective.com/docsify/donate

+ docsify@4.8.5
added 57 packages from 30 contributors in 12.674s
```

等待安装完毕即可.

---

## 2. docsify 使用

因为使用的是 gitbook 的项目,`SUMMARY.md`遵循 gitbook 的章节模式,请知悉.

### 2.1 初始化文档

使用命令: `docsify init yourDocsFolder`

```sh
[root@team-2 mini-docs]# ls
db  iview  javaee  javase  mq  README.md  server  SUMMARY.md  tb
[root@team-2 mini-docs]#  docsify init .

Initialization succeeded! Please run docsify serve .

[root@team-2 mini-docs]#  ls
db  index.html	iview  javaee  javase  mq  README.md  server  SUMMARY.md  tb
```

### 2.2 设置 index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>cs12110</title>
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
  <meta name="description" content="Description">
  <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
  <link rel="stylesheet" href="//unpkg.com/docsify/lib/themes/vue.css">
</head>
<body>
  <div id="app"></div>
  <script>
    window.$docsify = {
      name: 'cs12110',
      repo: 'cs12110/mini-docs',
      loadSidebar: true,
      loadSidebar: 'SUMMARY.md'
    }
  </script>

  <script src="//unpkg.com/docsify/lib/docsify.min.js"></script>
  <!-- 复制代码插件 -->
  <script src="//unpkg.com/docsify-copy-code"></script>
  <!-- 代码高亮插件 -->
  <script src="//unpkg.com/prismjs/components/prism-java.min.js"></script>
  <script src="//unpkg.com/prismjs/components/prism-bash.min.js"></script>

</body>
</html>
```

### 2.2 启动预览

为了方便启动,我把启动写成脚本,内容如下.

只要给脚本加上权限,然后运行该脚本即可.

```bash
#!/bin/bash

# clean the nohup content
echo > nohup.out

# the port of docsify web
web_port=5566

nohup docsify serve --port=$web_port .  &

echo 'you can visit: http://47.98.104.252:5566/'
```

启动脚本之后,可以通过浏览器访问: `http://服务器ip:5566/`即可看到文档了.

### 2.3 定时更新

那么怎么做到定时更新呢?

我们加入 linux 的定时器,使用定时器,定时拉取 github 上面的最新文档即可.

首先把 docsify 的 commit 一下,我们才能 pull

```sh
[root@team-2 mini-docs]# git add .
[root@team-2 mini-docs]# git commit -m 'docsify'
[master 8a0c2d0] docsify
 2 files changed, 23 insertions(+)
 create mode 100644 .nojekyll
 create mode 100644 index.html
[root@team-2 mini-docs]# git status
# On branch master
# Your branch is ahead of 'origin/master' by 1 commit.
#   (use "git push" to publish your local commits)
#
nothing to commit, working directory clean
```

加入定时器

```sh
[root@team-2 mini-docs]# crontab -e

*/5 * * * *  sh /opt/soft/docsify/git-pull.sh
```

脚本内容为

```bash
#!/bin/bash

cd /opt/soft/docsify/mini-docs

git pull
```

By the way,定时器的日志可以通过:`cat /var/log/cron`日志查看.

---

## 3. 参考资料

a. [docsify 官网](https://docsify.js.org/#/)

b. [cs12110 的 github](https://github.com/cs12110)
