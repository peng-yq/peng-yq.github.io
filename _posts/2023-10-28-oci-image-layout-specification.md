---
layout: post
title: OCI Image Spec——Image Layout Specification
subtitle: OCI Image Layout Specification
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
##  OCI Image Layout Specification

OCI镜像布局是OCI内容可寻址数据和位置可寻址引用 (refs，**image manifest和image index**，原理是[基于内容可寻址存储](https://en.wikipedia.org/wiki/Content-addressable_storage#Content-addressed_vs._location-addressed)，保存根据内容计算得出的“密钥”) 的目录结构，该布局可用于各种不同的传输机制：归档格式（如 tar、zip）、共享文件系统环境（如 nfs）或网络文件获取（如 http、ftp、rsync）。

**给定OCI Image Layout Specification和refs后，工具（原文是tools，这里准确的说我觉得是runtime caller，也就是docker、k8s以及containerd）可通过以下方法创建OCI runtime bundle：**

- **根据ref信息查找image manifest，可能通过image index查找**
- **按照指定顺序应用镜像的layers**
- **将image config转换为容器的config.json**

### Content

Image layout包括如下内容：

- blobs目录
  - blobs中的文件根据其内容命名（sha256计算）
  - 包含内容可寻址数据
  - 此目录必须存在，可以为空
- oci-layout文件
  - 此文件必须存在
  - 此文件必须是一个json对象
  - 此文件必须包含imageLayoutVersion field
  - 可能包含其他fields
- index.json文件
  - 此文件必须存在
  - 此文件必须是一个image index json对象

image layout example：

```shell
./index.json
./oci-layout
./blobs/sha256/3588d02542238316759cbf24502f4344ffcc8a60c803870022f335d1390c13b4
./blobs/sha256/4b0bc1c4050b03c95ef2a8e36e25feac42fd31283e8c30b3ee5df6b043155d3c
./blobs/sha256/7968321274dc6b6171697c33df7815310468e694ac5be0ec03ff053bb135e768
```

### Blobs

- Blobs目录格式一般为blobs/alg/encoded，encoded为具体的内容。其中alg/encoded必须与对象其描述符中的摘要 <alg>:<encoded>相匹配。例如，blobs/sha256/da39a3ee5e6b4b0d3255bfef95601890afd80709 的内容必须与摘要 sha256:da39a3ee5e6b4b0d3255bfef95601890afd80709 匹配
- <alg> 和 <encoded> 条目名称的字符集必须与描述符中描述的相应语法元素相匹配
- blobs 目录可能包含未被任何参考引用的 blobs
- blobs 目录可能缺少被引用的 blobs，在这种情况下，缺少的 blobs 应由外部 blob 存储器补齐

Blobs示例：

image manifest blobs示例：

```json
$ cat ./blobs/sha256/afff3924849e458c5ef237db5f89539274d5e609db5db935ed3959c90f1f2d51 | jq
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    },
...
```

image config示例：

```json
$ cat ./blobs/sha256/5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270 | jq
{
  "architecture": "amd64",
  "author": "Alyssa P. Hacker <alyspdev@example.com>",
  "config": {
    "Hostname": "8dfe43d80430",
    "Domainname": "",
    "User": "",
    "AttachStdin": false,
    "AttachStdout": false,
    "AttachStderr": false,
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": null,
    "Image": "sha256:6986ae504bbf843512d680cc959484452034965db15f75ee8bdd1b107f61500b",
...
```

image layer示例：

```shell
$ cat ./blobs/sha256/9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0
[gzipped tar stream]
```

### oci-layout file

包含imageLayoutVersion，和OCI image spec version一致。

```json
{
    "imageLayoutVersion": "1.0.0"
}
```

### index.json file

此文件是image layout的引用和各文件描述符的入口点。

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.index.v1+json",
      "size": 7143,
      "digest": "sha256:0228f90e926ba6b96e4f39cf294b2586d38fbb5a1e385c05cd1ee40ea54fe7fd",
      "annotations": {
        "org.opencontainers.image.ref.name": "stable-release"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      },
      "annotations": {
        "org.opencontainers.image.ref.name": "v1.0"
      }
    },
    {
      "mediaType": "application/xml",
      "size": 7143,
      "digest": "sha256:b3d63d132d21c3ff4c35a061adf23cf43da8ae054247e32faa95494d904a007e",
      "annotations": {
        "org.freedesktop.specifications.metainfo.version": "1.0",
        "org.freedesktop.specifications.metainfo.type": "AppStream"
      }
    }
  ],
  "annotations": {
    "com.example.index.revision": "r124356"
  }
}
```

## 
