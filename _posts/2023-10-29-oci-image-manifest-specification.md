---
layout: post
title: OCI Image Specification——Image Manifest Specification
subtitle: OCI Image Manifest Specification
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
## OCI Image Manifest Specification

Image manifest的作用如下：

- 内容可寻址的镜像，**能将镜像的配置（注意是配置而非此manifest）可以通过哈希计算为镜像和其组件生成唯一ID**
- 支持多体系结构镜像，OCI image spec将此定义在image index中
- 转换为OCI runtime spec

### Image Manifest

**Image manifest和Image index的区别**：

- image index包含多个多个架构和操作系统的镜像信息
- image manifest只包含特定架构和操作系统镜像的配置和layer

image manifest：

- **`schemaVersion`** *int*：必须为2，以确保与旧版本的Docker向后兼容
- **`mediaType`** *string*：`application/vnd.oci.image.manifest.v1+json`
- **`artifactType`** *string*：OPTIONAL，如果是artifact的manifest
- **`config`** *descriptor*：指向容器的配置
  - **`mediaType`** *string*：application/vnd.oci.image.config.v1+json
  - **`digest`**
  - **`size`**
- **`layers`** *array of objects*：包含的每一个对象都是descriptor描述符类型，至少包含一个layer
  - **`mediaType`** *string*
  - **`digest`**：对mediaType类型的layer做sha256计算得到的值
  - **`size`**

> 当config.mediaType的值为application/vnd.oci.image.config.v1+json时，layers必须满足以下要求：
>
> - index 0必须为基础层
> - 随后的层必须按照堆栈顺序（即从 layers[0] 到 layers[len(layer)-1]）依次排列
> - 最终的文件系统布局必须与将layers应用到空目录的结果一致

- **`subject`** *descriptor*：指定另一个manifest的描述符
- **`annotations`** *string-string map*

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7",
    "size": 7023
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
      "size": 32654
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b",
      "size": 16724
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736",
      "size": 73109
    }
  ],
  "subject": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
    "size": 7682
  },
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

### Guidance for an Empty Descriptor

空内容的artifact的manifest如下，size为2是最小json { }的大小，digest是对{ }做哈希计算得到的结果：

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.example+type",
  "config": {
    "mediaType": "application/vnd.oci.empty.v1+json",
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
    "size": 2
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.empty.v1+json",
      "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
      "size": 2
    }
  ],
  "annotations": {
    "oci.opencontainers.image.created": "2023-01-02T03:04:05Z",
    "com.example.data": "payload"
  }
}
```

关于artifact：

>在容器生态系统中，"artifact"（工件）一词通常用于指代容器镜像或容器相关的构建产物。它是构建、打包和分发应用程序的结果，可以被部署到容器运行时环境中。容器镜像是最常见的容器工件。除了容器镜像，还有其他类型的容器工件，如容器注册表（container registry）中的存储库（repository）、容器运行时（container runtime）的二进制文件、构建脚本、配置文件等。这些工件都是构建、管理和交付容器化应用程序所必需的组成部分。使用容器工件，可以实现应用程序的版本控制、可重复构建和分发、快速部署和扩展等优势。它们为容器化应用程序的开发、测试和部署提供了便利性和一致性。

 
