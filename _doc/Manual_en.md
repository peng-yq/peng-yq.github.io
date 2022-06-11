## Getting Started

> *This step is only used to view the blog locally, **it can be omitted***.

It is recommended to start under Linux (WSL is also possible) or Mac OS, the manual is demonstrated by Ubuntu 20.04.

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

Download [template](https://github.com/peng-yq/peng-yq.github.io/releases/tag/1.0) and unzip it, the following operations are all performed in the template folder.

```shell
bundle init
bundle config set --local path 'vendor/bundle'
bundle add jekyll
bundle exec jekyll new --force --skip-bundle .
bundle install
```

Start the service (default is `127.0.0.1:4000`).

```shell
bundle exec jekyll serve  # 或者npm start
```

## Turn the template into your blog

> The structure of the blog looks complicated, but it is easy to understand if you spend a little time with the front-end foundation.

Only the parts that need to be changed are explained below in order to build your blog in the shortest time.

### Configs

> Modify the *_config.yml* file

**基础配置**

```yml
title: PYQ Blog                 
SEOTitle: PYQ的博客 | PYQ Blog   
header-img: img/home-bg.jpg     
email: eilo.pengyq@gmail.com    
description: "关于计算机科学、程序与设计、想法与思考 | PYQ，Computer Science & Digital Technology Lover，Student | 这里是 @PYQ 的个人博客，与你一起发现更大的世界。"  
keyword: "PYQ, pengyq, Eilo, @PYQ, PYQ的博客, PYQ Blog, 博客, 个人网站, 互联网, Web, JavaScript, 前端, 计算机科学，网络空间安全，Linux，科技，数码"        
url: "https://peng-yq.github.io"             
baseurl: ""                

# Publish posts with future dates or collect documents.
future: true

# SNS settings
RSS: false                       # RSS subscription, off by default
# fill in your SNS account name
github_username:    peng-yq      
weibo_username:     username
zhihu_username:     username
facebook_username:  username
```

### Sidebar

```yml
# Sidebar settings
sidebar: true                         
sidebar-about-description: "Hi, it's PYQ here! <br> 想成为真正的工程师。"  
sidebar-avatar: /img/avatar-pyq.jpg    
```

**Tags**

```yml
# Featured Tags
featured-tags: true                     
featured-condition-size: 1              # How many tags of posts will appear on the homepage
```

**Comment**

The comment system uses Disqus (**If you don't want to open it, just delete the code or comment it out**), click [Disqus official website](https://disqus.com/) Register an account and select `I want to install disqus on my site`, then follow the prompts to complete the settings

```yml
# Disqus settings
disqus_username: pyq          #Fill in the short_name you set (not user_name)
```

> comments can also use Gitalk

**Website Analysis**

For website analysis, you can use [Baidu Analytics](http://tongji.baidu.com/web/welcome/login) and [Google Analytics](http://www.google.cn/analytics/) to convert your `track_id `Replace (**If you don't want to open, please delete the code or comment it out**)

```yml
# Analytics settings
# Baidu Analytics
ba_track_id: 83e259f69b37d02a4633a2b7d960139c

#Google Analytics
ga_track_id: 'UA-90855596-1'            
ga_domain: auto
```

**Friends**

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

> Modify *index.html*

```html
---
layout: page
description: "「想成为真正的工程师」"
---
```

### About

> Modify*about.html*

```html
---
layout: page
title: "About"
description: "《Hello, I'm PYQ》"    
header-img: "img/about-bg.jpg"
multilingual: true
---
```

> Modify the Chinese/English markdown files under *_inclouds/about*

### Archive

> Modify*archive.html*

```html
---
title: Archive
layout: default
description: "「我干了什么 究竟拿时间换了什么」"
header-img: "img/tag-bg.jpg"
---
```

### 404

> Modify*404.html*

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

### Picture

> Just replace the corresponding image under the *img* file

The *img/in-post* directory is used to place the pictures in the post, and the post cover is placed in the *img* directory

- If you don't want to change the file name in html, you can directly replace the image without changing the file name (note that the suffix, the file type, needs to be the same)
- The icon adopts .ico file, jpg/png and other files [convert ico website](https://convertio.co/zh/ico-converter/)
- [Photo material website](https://unsplash.com/), [Icon material website](https://www.flaticon.com/)
- Note that **the picture should not be too big**, it will affect the loading speed of the blog! ! ! [Compressed website](https://www.picdiet.com/)

![image-20220611164053891](https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202206111640385.png)

## Writing

> The post can be placed in the *_posts* directory, pay attention to the naming: for example **2021-12-19-linux-file-system.md**

The writing uses Markdown, just add front-matter at the beginning, [post example](https://github.com/peng-yq/peng-yq.github.io/blob/main/_posts/2021-12 -19-linux-file-system.md)

**front-matter**

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

corresponding effect

![image-20220611170138544](https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202206111701396.png)

**Pay attention to the image path in the post, otherwise an error will be displayed, and the format can be seen in sample**

## keynote

Hux's template also supports keynote, because I haven't used it yet, so the specific tutorial can be found at https://github.com/Huxpro/huxpro.github.io/blob/master/_doc/Manual.md#keynote-layout

## Github-Page

After modifying the template, you can open Github-Page to put the blog on the network

- Create a repository named **your_username.github.io**
- Put all files in the modified template in the repository

🎉Search your_username.github.io in your browser to access your blog🥳

