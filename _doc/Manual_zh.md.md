## 环境搭建

> *该步骤仅用于在本地查看博客效果，**可省略***

推荐在Linux（WSL也可以）或Mac OS下进行操作，手册通过Ubuntu 20.04进行演示

### Ruby

```shell
sudo apt update
sudo apt install ruby-full
ruby --version
```

### Bundler

```shell
source 'https://rubygems.org'
sudo gem install bundler
```

### Jekyll

下载[模板](https://github.com/peng-yq/peng-yq.github.io/releases/tag/1.0)并解压，下面的操作均在模板文件夹中进行操作

```shell
bundle init
bundle config set --local path 'vendor/bundle'
bundle add jekyll
bundle exec jekyll new --force --skip-bundle .
bundle install
```

开启服务（默认为`127.0.0.1:4000`）

```shell
bundle exec jekyll serve  # 或者npm start
```

## 将模板变为你的博客

> 博客的结构看起来复杂，在有前端基础的情况下花一点时间是很容易看明白的

下面只对需要改动的部分进行说明，以实现以最短的时间构建您的博客

### Configs

> 修改*_config.yml*文件

**基础配置**

```yml
title: PYQ Blog   # 博客的名字
SEOTitle: PYQ的博客 | PYQ Blog   # 标签页中的名字
header-img: img/home-bg.jpg     # 主页背景图片
email: eilo.pengyq@gmail.com    # 邮箱
description: "关于计算机科学、程序与设计、想法与思考 | PYQ，Computer Science & Digital Technology Lover，Student | 这里是 @PYQ 的个人博客，与你一起发现更大的世界。"  # 个人描述
keyword: "PYQ, pengyq, Eilo, @PYQ, PYQ的博客, PYQ Blog, 博客, 个人网站, 互联网, Web, JavaScript, 前端, 计算机科学，网络空间安全，Linux，科技，数码"        # 博客的关键词
url: "https://peng-yq.github.io"              # 域名
baseurl: ""                 # 子域名，比如https://peng-yq.github.io/blog，此处省略不填即可

# 发布具有未来日期的帖子或收集文档。
future: true

# SNS 设置
RSS: false                       # RSS订阅，默认关闭
# 填写你的SNS账号名即可
github_username:    peng-yq      
weibo_username:     username
zhihu_username:     username
facebook_username:  username
```

**侧边栏**

```yml
# Sidebar settings
sidebar: true                           # 是否开启侧边栏，默认开启
sidebar-about-description: "Hi, it's PYQ here! <br> 想成为真正的工程师。"  # 修改你的侧边栏描述，<br>标签为换行
sidebar-avatar: /img/avatar-pyq.jpg     # 放置你的照片或头像
```

**Tags**

```yml
# Featured Tags
featured-tags: true                     # 是否开启主页侧边栏中的tags标签，默认开启
featured-condition-size: 1              # 超过多少数量posts的tag会出现在首页
```

**评论系统**

评论系统采用Disqus（**若不想开启，直接删除代码或者注释掉即可**），美观好用（就是国内似乎被🧱了），进入[Disqus官网](https://disqus.com/)注册账号，并选择`I want to install disqus on my site`，后续根据提示完成设置即可

```yml
# Disqus settings
disqus_username: pyq          #填写你设置的short_name（非user_name）
```

> 评论系统还可以采用Gitalk

**网站分析**

网站分析可通过 [Baidu Analytics](http://tongji.baidu.com/web/welcome/login) 和 [Google Analytics](http://www.google.cn/analytics/)，将自己的`track_id`进行替换（**若不想开启，请删除代码或者注释掉即可**）

```yml
# Analytics settings
# Baidu Analytics
ba_track_id: 83e259f69b37d02a4633a2b7d960139c

#Google Analytics
ga_track_id: 'UA-90855596-1'            
ga_domain: auto
```

**友链**

```yml
friends: [
    {
        title: "PYQ Daily Blog",
        href: "https://pengyq.top"
    },
    {
        title: "MaQi的主页",
        href: "https://maqi.ink"
    }
]
```

### Home

> 修改*index.html*

```html
---
layout: page
description: "「想成为真正的工程师」"
---
```

### About

> 修改*about.html*

```html
---
layout: page
title: "About"
description: "《Hello, I'm PYQ》"    
header-img: "img/about-bg.jpg"
multilingual: true
---
```

> 修改*_inclouds/about*下的中/英markdown文件

### Archive

> 修改*archive.html*

```html
---
title: Archive
layout: default
description: "「我干了什么 究竟拿时间换了什么」"
header-img: "img/tag-bg.jpg"
---
```

### 404

> 修改*404.html*

```html
---
layout: default
title: 404
hide-in-nav: true
description: "你来到了没有知识的荒原 :("
header-img: "img/404-bg.jpg"
permalink: /404.html
---
```

### 图片

> 将*img*文件下的对应图片进行替换即可

*img/in-post*目录用于放置post中的图片，post封面则放在*img*目录下

- 若不想需改html中的文件名，可直接替换图片不修改文件名（注意后缀即文件类型需一样）
- 图标采用.ico文件，jpg/png等文件[转换ico网站](https://convertio.co/zh/ico-converter/)
- [图片素材网站](https://unsplash.com/)，[图标素材网站](https://www.flaticon.com/)
- 注意**图片不要太大**，会影响网站加载速度！！！[压缩网站](https://www.picdiet.com/)

![image-20220611164053891](https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202206111640385.png)

## 写作

> post放在*_posts*目录即可，注意命名：例如**2021-12-19-linux-file-system.md**

写作采用Markdown，只需要在开头添加front-matter即可，[post样例](https://github.com/peng-yq/peng-yq.github.io/blob/main/_posts/2021-12-19-linux-file-system.md)

**front-matter样例**

```
---
layout: post
title: "Linux文件系统"
subtitle: "The Linux File System"
author: "PYQ"
header-img: "img/post-bg-linux.jpg"
mathjax: true
header-mask: 0.3
catalog: true
tags:
  - Linux
 ---
```

对应的效果

![image-20220611170138544](https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202206111701396.png)

**注意post中的图片路径，否则会显示错误，格式可见样例**

## keynote

Hux大佬的模板还支持了keynote，因为我暂时还没用到，因此具体教程可见https://github.com/Huxpro/huxpro.github.io/blob/master/_doc/Manual.md#keynote-layout

## 开启Github-Page

在修改完模板后，就可以开启Github-Page将博客放至公网了

- 建立一个repo名为**your_username.github.io**的仓库
- 将修改好的模板中的所有文件放在仓库中

🎉在浏览器中输入your_username.github.io就可访问你的博客啦🥳

