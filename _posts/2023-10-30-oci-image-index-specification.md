---
layout: post
title: OCI Image Specification——Image Index Specification
subtitle: OCI Image Index Specification
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
## OCI Image Index Specification

Image Index是更高层次的manifest，它指向一个或多个平台的image manifest：

- **`schemaVersion`** *int*：2
- **`mediaType`** *string*：application/vnd.oci.image.index.v1+json
- **`artifactType`** *string*：OPTIONAL 
- **`manifests`** *array of objects*：一系列manifest
  - **`mediaType`** *string*：application/vnd.oci.image.manifest.v1+json
  - **`platform`** *object*：镜像需要的运行时环境
    - **`architecture`** *string*：CPU架构
    - **`os`** *string*
    - **`os.version`** *string*
    - **`os.features`** *array of strings*
    - **`variant`** *string*
    - **`features`** *array of strings*
- **`subject`** *descriptor*：指向其他manifest
- **`annotations`** *string-string map*

下面的image index示例包含了两个平台的image manifest

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

下面的image index示例包含了两种不同类型的manifest

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.index.v1+json",
      "size": 7682,
      "digest": "sha256:601570aaff1b68a61eb9c85b8beca1644e698003e0cdb5bce960f193d265a8b7"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

## 
