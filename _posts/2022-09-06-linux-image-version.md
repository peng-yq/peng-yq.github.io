---
layout: post
title: "Linux镜像版本"
subtitle: "Linux Image Version"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - Linux
---

之前在下载Linux发行版镜像时，一直没有注意镜像的版本，各个发行版均下载的amd64版本。当时也想过为啥命名是amd64，并且可以在intel的机器上完美运行，64当然很好理解即64位架构。

但在下载Ubuntu12.04时，我发现除了amd64外还有i386版本，在搜索资料后，终于了解了两种版本的区别。

- amd64位版本和i386版本的选择和电脑的CPU厂家没有关系，只和电脑位数有关系，前者是64位，而后者是32位
- amd64命名来源是基于amd发明了64位架构
- i386命名来源是基于intel 80386处理器
