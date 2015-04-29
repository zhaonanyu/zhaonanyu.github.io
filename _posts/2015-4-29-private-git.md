---
layout: post
title: "使用git进行协同开发"
description: "using git for private dev"
tags: [Linux, sys_config, git]
comments: true
---

github是个好东西，但是如果你正在做的项目不想公布源码又想使用git 的话就只能自己搭建git了。

我们不准备采取何川师兄在其博客《git workflow》中所描述的工作流：http://hechuanx.cn/home/?p=54

而使用集中式工作流

##部署流程

**安装git**
`$ sudo apt-get install git`
然后创建一个git用户，用于git服务：
`$ sudo adduser git`

**创建所需要的repo**
`$ sudo git init --bare project.git`
将仓库的所有者修改为git用户
`$ sudo chown -R git:git project.git`

**禁用shell登录**
处于安全方面的考虑，刚才创建的git用户不能使用shell登录，我们要对它的默认shell进行修改，修改默认shell需要修改/etc/passwd文件
`$sudo vim /etc/passwd`
找到git用户的那一行
`git:x:1001:1001:,,,:/home/git:/bin/bash`
将/bin/bash修改为/usr/bin/git-shell：
`git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell`

**使用**
下面就可以使用了
克隆仓库：
`$ git clone ssh://git@xxx.xxx.xxx/home/git/xxxx.git`
剩下的相信大家都很熟悉了

---

**其他**
为什么使用`git init --bare`:http://blog.csdn.net/ljchlx/article/details/21805231
集中式工作流：http://blog.jobbole.com/76847/