---
layout: post
title: OCI Image Spec——Content Descriptors
subtitle: OCI Content Descriptors
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## OCI Content Descriptors

OCI镜像由多个不同的组件组成，这些组件排列在一个 Merkle Directed Acyclic Graph (DAG，有向无环图) 中，可以参考这篇文章[A Peek into Docker Images](https://medium.com/tenable-techblog/a-peek-into-docker-images-b4d6b2362eb)。Merkle DAG中各组件之间的引用通过内容描述符来表达，内容描述符描述了目标内容的布局。内容描述符包括内容类型、内容标识符（摘要）和原始内容的字节大小。描述符还可选择包含所描述的artifact的类型。

### Properties

描述符包括如下属性：

- **`mediaType`** *string*
- **`digest`** *string*：目标内容的摘要值
- **`size`** *int64*：目标原始内容的大小（字节为单位）
- **`urls`** *array of strings*：OPTIONAL，目标对象的下载地址
- **`annotations`** *string-string map*：OPTIONAL，描述符的一些元数据
- **`data`** *string*：OPTIONAL，引用内容的嵌入式表示（base64编码），解码后的数据必须与引用内容一致，可以根据size和digest验证
- **`artifactType`** *string*：OPTIONAL，描述符指向artifact的媒体类型

### Digests

摘要是内容的标识符，通过哈希运算实现基于内容寻址。

摘要值的格式如下：

- digest（摘要）属性是一个字符串由algorithm（算法）和encoded（编码）两部分组成
- algorithm指定了进行摘要计算的哈希函数算法和编码算法，多个算法之间用algorithm-separator分隔开
- algorithm-component描述了算法的格式，小写或者数字
- encoded表示对哈希计算得到的摘要进行编码后的结果

```json
digest                ::= algorithm ":" encoded
algorithm             ::= algorithm-component (algorithm-separator algorithm-component)*
algorithm-component   ::= [a-z0-9]+
algorithm-separator   ::= [+._-]
encoded               ::= [a-zA-Z0-9=_-]+
```

样例

```json
sha256:6c3c624b58dbbcd3c0dd82b4c53f04194d1247c6eebdaab7c610cf7d66709b3b
sha512:401b09eab3c013d4ca54922bb802bec8fd5318192b0a75f201d8b372742...
multihash+base58:QmRZxt2b1FVZPNqd8hsiykDL3TdBDeTSPX9Kv46HmX4Gx8
sha256+b64u:LCa0a2j_xo_5m0U8HTBBNBNCLXBkg7-g-YpeiGJm564
```

#### Verification

在对描述符引用对象进行验证时，需要注意以下事项：

- 首先对内容大小进行验证，即与描述符size属性进行比较
- 避免在摘要计算前做大量操作
- 采用规划范的方式确保底层内容的稳定性

进行摘要计算的伪代码如下：对目标对象的字节内容进行哈希计算的结果进行编码（如果指定了编码算法），并与算法进行拼接完成格式处理，最后与摘要值进行对比

```json
let ID(C) = Descriptor.digest
let C = <bytes>
let D = '<alg>:' + Encode(H(C))
let verified = ID(C) == D
D == ID(C) == '<alg>:' + Encode(H(C))
```

**标准规范对摘要值算法的要求为sha256**，规范中定义的算法包括（如果使用其他未定义的算法需满足上述digest格式）：

| algorithm identifier | algorithm                                                    | *encoded* portion |
| -------------------- | ------------------------------------------------------------ | ----------------- |
| `sha256`             | [SHA-256](https://github.com/opencontainers/image-spec/blob/main/descriptor.md#sha-256) | /[a-f0-9]{64}/    |
| `sha512`             | [SHA-512](https://github.com/opencontainers/image-spec/blob/main/descriptor.md#sha-512) | /[a-f0-9]{128}/   |

### Examples

```json
{
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "size": 7682,
  "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
  "urls": [
    "https://example.com/example-manifest"
  ]
}
{
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "size": 123,
  "digest": "sha256:87923725d74f4bfb94c9e86d64170f7521aad8221a5de834851470ca142da630",
  "artifactType": "application/vnd.example.sbom.v1"
}
```

## 
