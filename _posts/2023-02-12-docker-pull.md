---
layout: post
title: "Docker Pull速度慢的解决办法"
subtitle: "The solution to the slow speed of Docker Pull"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - docker
---
Docker Pull速度慢首先考虑网络的问题！！！如果一直不动或者极慢甚至TLS连接都建立失败，那大概率就是网络的问题了，考虑换个网络试试。

如果网络没有问题，就更换镜像源，可以多试试几个镜像源。

[阿里云的教程](https://help.aliyun.com/document_detail/60750.html)