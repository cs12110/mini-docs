# Git

我还是很喜欢 Git 呀!

---

## 1. 安装 Git

没有 yum 搞不定的事,如果有那就 rpm.

```sh
[root@bi141 ~]# yum install -y git
[root@bi141 ~]# git --version
git version 1.8.3.1
```

---

## 2. 基础命令

### 设置全局用户和邮箱

在 git 里面,邮箱和用户是很重要的.

查看 git 里面配置参数

```sh
$ git config --list
http.sslcainfo=D:/Pro/Git/mingw64/ssl/certs/ca-bundle.crt
http.sslbackend=openssl
diff.astextplain.textconv=astextplain
filter.lfs.clean=git-lfs clean -- %f
filter.lfs.smudge=git-lfs smudge -- %f
filter.lfs.process=git-lfs filter-process
filter.lfs.required=true
user.email=cs12110@163.com
user.name=cs12110
core.repositoryformatversion=0
core.filemode=false
core.bare=false
core.logallrefupdates=true
core.symlinks=false
core.ignorecase=true
remote.origin.url=https://github.com/cs12110/mini-docs.git
remote.origin.fetch=+refs/heads/*:refs/remotes/origin/*
branch.master.remote=origin
branch.master.merge=refs/heads/master
```

配置用户邮箱和姓名

```sh
$ git config --global user.name cs12110

$ git config --global user.email cs12110@163.com
```

### 设置某个 git 仓库的用户和邮箱

Q: But,如果你有多个 git 仓库需要设置不同的邮箱和用户名呢?

A: 就你事多,就你事多,就你事多....

进入到仓库所在位置,使用 local 设置,命令如下:

```sh
$ git config --local user.name mr3306

$ git config --local user.email mr3306@163.com
```

### 初始化

构建空的本地仓库

```sh
# 构建本地仓库
[root@bi141 git]# git init
Initialized empty Git repository in /git/.git/

# 新建readme.md
[root@bi141 git]# touch readme.md

# 将文件添加到本地缓存并提交到本地仓库
[root@bi141 git]# git add .
[root@bi141 git]# git commit -m 'init'
[master (root-commit) e6def31] init
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 readme.md
```

### 克隆

从 github 拷贝远程仓库到本地

```sh
[root@bi141 git]# git clone https://github.com/cs12110/4test.git
Cloning into '4test'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 5 (delta 0), reused 5 (delta 0), pack-reused 0
Unpacking objects: 100% (5/5), done.
[root@bi141 git]# ls
4test
```

上面命令是克隆主分支的代码,如果最新代码在其他分支怎么玩?

```sh
# 显示所有的分支
[root@bi141 git]# git branch -r

# clone你想要的分支
[root@bi141 git]# git clone -b branchName gitUrl
```

### push&pull

pull: 更新远程代码带本地

```sh
[root@bi141 4test]# git pull origin master
From https://github.com/cs12110/4test
 * branch            master     -> FETCH_HEAD
Already up-to-date.
```

Q: 如果想要获取所有分支的最新代码呢?

```sh
[root@bi141 4test]# git pull --all
```

push: 推送本地代码到远程服务器

```sh
[root@bi141 4test]# git push origin master
Username for 'https://github.com': cs12110
Password for 'https://cs12110@github.com':
Everything up-to-date
```

### 分支与合并

```sh
# 直接使用checkout -b 分支名称 创建分支
[root@bi141 git]# git checkout -b dev
Switched to a new branch 'dev'
[root@bi141 git]# git branch
* dev
  master
```

分支进行修改和提交

```sh
[root@bi141 git]# touch dev-1.txt
[root@bi141 git]# touch dev-2.txt
[root@bi141 git]# git add .
[root@bi141 git]# git commit -m 'add 1-2 txt'

[root@bi141 git]# touch dev-3.txt
[root@bi141 git]# touch dev-4.txt
[root@bi141 git]# git add .
[root@bi141 git]# git commit -m 'add 3-4 txt'
```

合并到主分支

```sh
[root@bi141 git]# git checkout master
Switched to branch 'master'
[root@bi141 git]# git merge --squash dev
[root@bi141 git]# git add .
[root@bi141 git]# git commit -m 'merge dev'
```

### 删除分支

注意:**删除分支前,记得合并分支的代码到主分支.**

**删除本地分支**

```sh
[root@bi141 git]# git branch
  dev
* master
[root@bi141 git]# git branch -D dev
Deleted branch dev (was 683c8ac).
[root@bi141 git]# git branch
* master
```

**删除远程分支**

```sh
[root@bi141 git]git push origin --delete dev
```

### 版本回退

#### checkout

这个一般用于

```sh
# 当前版本历史和相关文件
[root@bi141 git]# git log --oneline
c091ae5 3.txt
e9f3128 2.txt
1079ee7 1.txt
e6def31 init
[root@bi141 git]# ls
1.txt  2.txt  3.txt  readme.md

# 版本回退
[root@bi141 git]# git checkout 1079ee7
[root@bi141 git]# git log --oneline
1079ee7 1.txt
e6def31 init
[root@bi141 git]# ls
1.txt  readme.md

# 查看所有commit log
[root@bi141 git]# git reflog --oneline
1079ee7 HEAD@{0}: checkout: moving from master to 1079ee7
c091ae5 HEAD@{1}: commit: 3.txt
e9f3128 HEAD@{2}: commit: 2.txt
1079ee7 HEAD@{3}: commit: 1.txt
e6def31 HEAD@{4}: commit (initial): init
```

### reset

```sh
PC@DESKTOP-DV8C0GM MINGW64 ~/Desktop/git (master)
$ git log
commit 5d342570b33c94c2cdeeb53448e9d44f49aeab1f
Author: cs12110 <cs12110@163.com>
Date:   Wed Apr 17 09:15:16 2019 +0800

    merge dev

commit a0065f180454424e94e30e0eecbd74e9fd48cfc3
Author: cs12110 <cs12110@163.com>
Date:   Wed Apr 17 09:13:57 2019 +0800

    add 2.txt

commit 3ebe6cce761b5d6ea7c6e1ec85636ccc079b6cbc
Author: cs12110 <cs12110@163.com>
Date:   Wed Apr 17 09:13:37 2019 +0800

    add 1.txt
```

```sh
PC@DESKTOP-DV8C0GM MINGW64 ~/Desktop/git (master)
$ git reset --hard a0065f180454424e94e30e0eecbd74e9fd48cfc3
HEAD is now at a0065f1 add 2.txt

PC@DESKTOP-DV8C0GM MINGW64 ~/Desktop/git (master)
$ git log
commit a0065f180454424e94e30e0eecbd74e9fd48cfc3
Author: cs12110 <cs12110@163.com>
Date:   Wed Apr 17 09:13:57 2019 +0800

    add 2.txt

commit 3ebe6cce761b5d6ea7c6e1ec85636ccc079b6cbc
Author: cs12110 <cs12110@163.com>
Date:   Wed Apr 17 09:13:37 2019 +0800

    add 1.txt
```

```sh
PC@DESKTOP-DV8C0GM MINGW64 ~/Desktop/git (master)
$ git reflog
a0065f1 HEAD@{0}: reset: moving to a0065f180454424e94e30e0eecbd74e9fd48cfc3
5d34257 HEAD@{1}: commit: merge dev
a0065f1 HEAD@{2}: checkout: moving from dev to master
f19f652 HEAD@{3}: commit: add 3.txt by dev
a0065f1 HEAD@{4}: checkout: moving from master to dev
a0065f1 HEAD@{5}: commit: add 2.txt
3ebe6cc HEAD@{6}: commit (initial): add 1.txt
```

### 修改上一次提交信息

```sh
[root@bi141 git]# git log -n 1
commit 3e6a8176c3e415b7953b016db6c273b09e848a74
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:17:49 2019 +0800

    ok
[root@bi141 git]# git commit --amend
[master 57b4bea] merge dev,for dev-5.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 dev-5.txt
[root@bi141 git]# git log -n 1
commit 57b4bea2f5b7883f2ec8695a993f618d7315f642
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:17:49 2019 +0800

    merge dev,for dev-5.txt
```

### 合并 commit

FBI Warning: **这个非常危险,生产环境不建议使用.**

适用场景: 当仓库存在太多无用的 commit 时,可以把多个 commit 合并成一个 commit.

合并前,提交历史为:

```sh
[root@bi141 git]# git log
commit 57b4bea2f5b7883f2ec8695a993f618d7315f642
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:17:49 2019 +0800

    merge dev,for dev-5.txt

commit 754b2cca1239df8053a5a66f4e975c449f6d5bd2
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:14:40 2019 +0800

    merge dev

commit e822fc497c39404ecd8631e5565b079a848738f4
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:06:57 2019 +0800

    readme.md
```

合并后

```sh
# 注意合并必须要有一个为pick,且必须合并数量<commitNum
[root@bi141 git]# git rebase -i HEAD~2
pick   754b2cc merge dev
squash 57b4bea merge dev,for dev-5.txt


[root@bi141 git]# git log
commit 6a3d525c2ede7673fb6f77f46eebba92ccf112ca
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:14:40 2019 +0800

    merge dev

    merge dev,for dev-5.txt

commit e822fc497c39404ecd8631e5565b079a848738f4
Author: Gogs <gogs@fake.local>
Date:   Wed Feb 27 14:06:57 2019 +0800

    readme.md
```

### 对比差异

有些时候需要看看本地和远程 master 的分支上面有什么区别.

那么可以使用下面这个命令

```sh
mr3306:spring-rookie mr3306$ git fetch
mr3306:spring-rookie mr3306$ git diff master origin/master
```

---

## 3. 参考资料

a. [廖雪峰 Git 教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
