---
layout: post
title: OCI Image Spec——Conversion to OCI Runtime Configuration
subtitle: Conversion to OCI Runtime Configuration
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
##  Conversion to OCI Runtime Configuration

当将OCI image转换为OCI runtime bundle时，有两个部分是直接相关联的：

- 将各镜像层挂载成一个完整的rootfs，这一部分在Image Layer Filesystem changeset中进行了说明，通过联合挂载技术
- 将镜像的配置文件转换为容器的config.json

这部分主要说明了镜像的配置文件转换为容器的config.json的细节。

### Verbatim Fields

某些镜像配置字段在容器运行时配置中有相同的对应字段。其中有些是纯粹基于注释的字段，配置转换器必须将以下字段逐字提取到生成的运行时配置中的相应字段：

| Image Field         | Runtime Field  | Notes |
| ------------------- | -------------- | ----- |
| `Config.WorkingDir` | `process.cwd`  |       |
| `Config.Env`        | `process.env`  | 1     |
| `Config.Entrypoint` | `process.args` | 2     |
| `Config.Cmd`        | `process.args` | 2     |

1. 转换器可以向process.env添加其他条目，但不应添加与Config.Env中的变量名相同的条目
2. 如果同时指定了Config.Entrypoint和Config.Cmd，转换器必须将Config.Cmd的值追加到Config.Entrypoint的值上，并将process.args设置为合并后的值

#### Annotation Fields

| Image Field         | Runtime Field | Notes |
| ------------------- | ------------- | ----- |
| `os`                | `annotations` | 1,2   |
| `architecture`      | `annotations` | 1,3   |
| `variant`           | `annotations` | 1,4   |
| `os.version`        | `annotations` | 1,5   |
| `os.features`       | `annotations` | 1,6   |
| `author`            | `annotations` | 1,7   |
| `created`           | `annotations` | 1,8   |
| `Config.Labels`     | `annotations` |       |
| `Config.StopSignal` | `annotations` | 1,9   |

1. 如果用户使用Config.Labels明确指定了此注解，则此字段中指定的值优先级较低，转换器必须使用 Config.Labels 中的值
2. 此字段的值必须设置为注释中 org.opencontainers.image.os 的值
3. 此字段的值必须在注释中设置为 org.opencontainers.image.architecture 的值
4. 此字段的值必须设置为注释中 org.opencontainers.image.variant 的值
5. 此字段的值必须设置为注释中 org.opencontainers.image.os.version 的值
6. 此字段的值必须设置为注释中 org.opencontainers.image.os.features 的值
7. 此字段的值必须在注释中设置为 org.opencontainers.image.author.author 的值
8. 此字段的值必须设置为注释中 org.opencontainers.image.created 的值
9. 此字段的值必须设置为注释中 org.opencontainers.image.stopSignal 的值

### Parsed Fields

| Image Field   | Runtime Field    |
| ------------- | ---------------- |
| `Config.User` | `process.user.*` |

如果Config.User中的用户或用户组的值是数字（uid 或 gid），那么这些值必须分别逐字复制到process.user.uid和 process.user.gid。如果Config.User中的user或group值不是数字，那么转换器应该使用适合容器上下文的方法来解析用户信息。对于类 Unix 系统，这可能需要通过NSS或解析提取容器根文件系统中的/etc/passwd来确定process.user.uid 和process.user.gid的值。

此外，转换器应将process.user.additionalGids的值设置为与Config.User所描述的容器上下文中的用户相对应的值。对于类Unix系统，这可能需要通过NSS或解析/etc/group进行解析，并确定process.user.uid 中指定的用户的组成员资格。如果Config.User中的用户值是数字或Config.User指定了组，转换器不应修改process.user.additionalGids。

如果Config.User未定义，则转换后的process.user值由执行定义。如果Config.User与容器上下文中的用户不对应，转换器必须返回错误。

### Optional Fields

| Image Field           | Runtime Field |
| --------------------- | ------------- |
| `Config.ExposedPorts` | `annotations` |
| `Config.Volumes`      | `mounts`      |
