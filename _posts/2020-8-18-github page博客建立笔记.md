---
layout:     post
title:      "Github Page + Jekyll 博客搭建记录"
subtitle:   ""
date:       2020-8-18
author:     "Ricky Wu"
header-img: "img/post-bg.jpg"
tags:
    - blog
---

> 采用 github pages + jekyll 搭建个人博客，博客借鉴了[黄玄](https://huangxuan.me/)的个人博客主题，感谢作者，搭建动力来源是[这篇文章](https://momomoxiaoxi.com/2016/10/22/Websecurity/)，感谢沈师傅。

### Github page 仓库创建

首先到[这里](https://github.com/new)，创建一个github仓库。 因为要使用github page，仓库名称有固定的格式：`username.github.io`，其中**username**必须是Github账户的用户名（比如我的zchengwu）。

> **注意：** username如果不是Github账户名，这个仓库就会成为`username.github.io`的子站点，比如访问地址会是：`username.github.io/aaa.github.io`。`username.github.io`是Github默认分配给你的域名，同名仓库即代表着默认网站内容。而`username.github.io/仓库名称`，是用来访问你的其它仓库的地址。

进入仓库设置界面，可以选择Github内置主题，编辑`README.md`，它的内容会显示在网站首页，`_config.yml`是jekyll的全局配置文件。每次修改仓库内容，都会触发重新部署，有错误会通过邮件发通知。

> Github内置支持的几个主题，官方的仓库在这里：[https://pages.github.com/themes](https://link.zhihu.com/?target=https%3A//pages.github.com/themes)，每个README.md里都有介绍如何设置。

更换主题有两种方法：

1. fork一个主题模板仓库，修改仓库名为我们自己的，然后对内部进行修改。
2. 简单地在仓库设置里面Choose theme，然后参照官方仓库的主题目录，需要什么拿什么，进行配置。

---

###  环境

首先需要了解一下，Github Pages的[依赖环境](https://pages.github.com/versions/)。

``` 
=> jekyll 3.9.0
```

### 在Ubuntu 18.04上安装Jekyll

官方Ubuntu上安装[说明](https://jekyllrb.com/docs/installation/ubuntu/)，这里给出一个安装具体版本的指令

```
# uninstall jekyll
$ gem uninstall jekyll

# install an older version of jekyll
$ gem install jekyll -v 1.5.1
```

### 在模板基础上的改动

fork稳定版的模板`huxblog-boilerplate`，需要改动的地方绝大多数都在README.zh.md里说明了。有几个小细节：

#### README.zh.md

这个参考指南建议上传的时候从文件夹中移除，我在尝试过程中在文件尾部报了一个大括号未匹配的错误，没有找到语法上的错误。

#### font-awesome加载失败问题

```html
.../xxx.github.io/_includes/head.html:
 26: <!-- link href="http://cdn.staticfile.org/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet" type="text/css" -->
 27: <link href="//cdnjs.cloudflare.com/ajax/libs/font-awesome/4.6.3/css/font-awesome.min.css" rel="stylesheet" type="text/css">
```

#### disqus 留言

创建一个帐号以后，找到`Add Disqus to Site`，为网站创建一个 shortname，在`_config.yml`里添加：

```yml
...
disqus_username: my_short_name
...
```

#### 部署 git 4 条指令

```
$ git add .
$ git commit -m "xxx"
$ git pull origin master
$ git push origin master
```

### 绑定域名

见参考

---

参考：

[Github Pages + jekyll 全面介绍极简搭建个人网站和博客](https://zhuanlan.zhihu.com/p/51240503)

[Github Pages + Jekyll 创建个人博客](https://www.jianshu.com/p/37a9719ef3d0)

[jekyll文件目录介绍](https://jekyllrb.com/docs/structure/)

[模板在这里](http://huangxuan.me/huxblog-boilerplate/)

[github怎么绑定自己的域名？](https://www.zhihu.com/question/31377141)

[how-to-enable-https-support-on-custom-domains](https://github.community/t/how-to-enable-https-support-on-custom-domains/10351)