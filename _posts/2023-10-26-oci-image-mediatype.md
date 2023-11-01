---
layout: post
title: OCI Image Spec——Media Types
subtitle: OCI Image Media Types
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## OCI Image Media Types

OCI镜像规范定义的镜像文件相关的媒体类型如下（原文很清楚，这里就不翻译了），各媒体类型所表示的资源定义后文再详细展开：

- `application/vnd.oci.descriptor.v1+json`: Content Descriptor
- `application/vnd.oci.layout.header.v1+json`: OCI Layout
- `application/vnd.oci.image.index.v1+json`: Image Index
- `application/vnd.oci.image.manifest.v1+json`: Image manifest
- `application/vnd.oci.image.config.v1+json`: Image config
- `application/vnd.oci.image.layer.v1.tar`: "Layer", as a tar archive
- `application/vnd.oci.image.layer.v1.tar+gzip`: "Layer", as a tar archive compressed with gzip
- `application/vnd.oci.image.layer.v1.tar+zstd`: "Layer", as a tar archive compressed with zstd
- `application/vnd.oci.empty.v1+json`: Empty for unused descriptors

各媒体类型的关联如下图所示：描述符可用于所有引用，描述符说明了引用对象的媒体类型、大小和摘要值等；image index可以引用到不同平台的image manifest；一个image manifest包含一个image config和多个image layer。

<img src="https://github.com/opencontainers/image-spec/raw/main/img/media-types.png"> 
