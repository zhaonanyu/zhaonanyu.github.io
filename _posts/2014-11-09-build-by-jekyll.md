---
layout: post
title: "动手搭建自己的jekyll静态博客"
description: "every geek wants a blog all by himself lol"
tags: [GitHub Pages, jekyll, static site generator]
comments: true
---

>*动态网站太重了.轻量级的静态网站生成工具一时蔚然成风,至少在开(si)发(ji)者(lao)圈子里如此.--[kernel panic](http://ipn.li/kernelpanic/)*

------

>##重要
>
>我假设看这篇文章的人具有
>
> * **基本的Linux Shell的使用**
>
> * **拥有GitHub账户并了解Git的使用**
>
*你至少需要拥有一个[GitHub](http://github.com)账户,如果你不了解Linux Shell使用你当然也可以往下看,你所需要的命令我都没有省略*
------

**首先要了解jekyll是什么:**

>*jekyll是一个简单的免费的Blog生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如disqus。最关键的是jekyll可以免费部署在github上，而且可以绑定自己的域名。--互动百科*

------

##1. 安装jekyll
[jekyll hp](http://jekyllrb.com/)中有你想了解的全部,当然你或许会觉得信息量太大了,你只想要先把自己的**静态网站**先跑起来,那么这篇博客相信会帮到你.

**首先**,你需要安装jekyll所需要的环境,jekyll使用的是**ruby**(性能你懂得)所以首先的是安装ruby环境:

~~~shel
$sudo apt-get install ruby1.9.1-dev
~~~

当然你也需要安装**NodeJS**:

~~~shell
$sudo apt-get install NodeJS
~~~

接下来就可以**安装jekyll**了

~~~shell
$sudo gem install jekyll
$jekyll new my-awsome-site
$cd my-awsome-site
~/my-awsome-site$ jekyll serve
~~~

完成上面的命令以后你就可以用你的浏览器访问http://localhost:4000来测试你的测试页面了.*see? simple*

------

##2. jekyll使用入门

首先简单的说明刚才生成的**my-awsome-site**的文件夹结构

简单来说作为简单的入门使用,你只需要了解**_post**文件夹和**_config.yml**文件.

###2.1 _post文件夹
_post文件夹包含了你所有的post,你可以简单的使用[markdown](http://zh.wikipedia.org/wiki/Markdown)标记语言来编写你自己的帖子.**不过值得注意的是,对于一个jekyll模板来说,模板可能会自定义markdown标记所以先去了解你所使用的模板**,当然关于模板的事情我们后面在说.

###2.2 _config.yml文件
看名字就知道_config.yml是你站点的一个config,这个config文件也和模板密切相关,**想修改这个文件请了解你所使用的模板**.

###2.3 使用模板
用过wordpress的人一定会知道有模板这种东西,当然对于静态网站生成器来说使用起来也很简单.

你可以去[jekyll theme](http://jekyllthemes.org/)这个网站找自己喜欢的模板,然后在**GitHub**上fork过来,然后clone覆盖掉你的my-awsome-site就可以使用这个模板了.

**下一个模板,覆盖你的文件夹然后使用:**

~~~shell
$jekyll serve
~~~

来测试你的新模板

###2.4 编写你自己的post
相信你了解一些markdown(如果你经常使用GitHub的话).如果你不了解去找一下markdown的语法,相当简单,相信你十分钟就能掌握, learn it on [Wikipedia](http://zh.wikipedia.org/wiki/Markdown).

但是你需要了解对于jekyll的markdown来说有一个markdown的头:

~~~markdown
---
layout: post
title: "Sample Link Post"
description: "Example and code for using link posts."
tags: [sample post, link post]
comments: true
link: http://mademistakes.com  
---
~~~

这个头包含你的post的标题,tag等信息.刚才也提到了这个头和你所使用的jekyll的模板是密切相关的,**修改的时候先去了解你所使用的模板**.

------

##3. 发布你自己的静态网站
相信你现在可以在http://localhost:4000访问到你自己的jekyll网站了,那么如何发布呢?

GitHub提供了一个[GitHub Pages](https://pages.github.com/)的服务,你可以把你的html静态网站或者jekyll挂到你的GitHub的仓库中就可以访问到你的静态网站了.

###3.1 建立GitHub Pages仓库
要使用GitHub Pages服务,你需要在你的GitHub上新建一个名为:

***username*.github.io**

的仓库,*username*是你的用户名,创建了这个仓库并并push了你的网站之后你就可以使用http://username.github.io这样一个网址来访问你的网站了.

###3.2 push你的jekyll网站
首先你需要把你刚才建立的*username*.github.io这个仓库clone下来:

~~~shell
$git clone http://github.com/username/username.github.io
~~~

clone下来以后你会看到一个username.github.io的文件夹,把你刚才的my-awsome-site文件夹中你测试好的jekyll网站拷贝到这个文件夹中,然后使用命令:

~~~shell
$git add .
$git commit -m "init commit"
$git push
$git push origin master
~~~

完成了上面的操作后你的静态网站已经成功地推送到你的username.github.io这个仓库中了,去github页面找到你的username.github.io这个仓库:

**Setting -> GitHub Pages**

应该可以看到你的网站已经成功部署了,**首次提交的时候,你可能需要10分钟才能使用username.github.io这个网址访问你的网站**,当然之后对于你网站的修改你都几乎不需要等待.

------

### 10 mins later...
***相信你已经可以使用username.github.io访问到你的网站了***

#Enjoy XD

