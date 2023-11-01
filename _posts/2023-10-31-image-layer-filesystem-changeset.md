---
layout: post
title: OCI Image Specification——Image Layer Filesystem Changeset
subtitle: OCI Image Layer Filesystem Changeset
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## Image Layer Filesystem Changeset

此部分内容描述了如何将文件系统以及对文件系统的更改（例如删除文件）转换成镜像的layer，以及各个layer是如何堆叠成一个完整的文件系统的。

### Change Types

在changeset（感觉怎么翻译都不太对，直接用原文）中的change类型如下：

- 添加
- 修改
- 删除

其中对添加和修改的处理方式类似，而删除则通过“whiteout”文件。

#### File Types

支持的文件类型包括：

- 普通文件
- 目录
- 套接字
- 软/符号链接
- 块设备
- 字符设备
- 命名管道（FIFOs）

> Linux中的设备类型可分为：字符设备（鼠标、键盘、串口）、块设备（硬盘、U盘）和网络设备

### File Attributes

支持的文件属性包括：

- Modification Time (mtime)
- User ID (uid)
- User Name (uname) secondary to uid
- Group ID (gid)
- Group Name (gname) secondary to gid
- Mode (mode)
- Extended Attributes (xattrs)
- Symlink reference (linkname + symbolic link type)
- Hardlink reference (linkname)

### Creating

#### Initial Root Filesystem

初始根文件系统是镜像的基础层或父层。

```shell
rootfs-c9d-v1/
```

#### Populate Initial Filesystem

创建文件和目录：

```shell
rootfs-c9d-v1/
  etc/
    my-app-config
  bin/
    my-app-binary
    my-app-tools
```

根文件系统会会以普通 tar 压缩文件的形式创建，相对路径为 rootfs-c9d-v1，包括如下内容：

```shell
./
./etc/
./etc/my-app-config
./bin/
./bin/my-app-binary
./bin/my-app-tools
```

#### Populate a Comparison Filesystem

创建一个新目录，并使用先前根文件系统的副本或快照对其进行初始化。可以保留文件属性以创建该副本的命令示例有：

```shell
cp -a rootfs-c9d-v1/ rootfs-c9d-v1.s1/
rsync -aHAX rootfs-c9d-v1/ rootfs-c9d-v1.s1/
mkdir rootfs-c9d-v1.s1 && tar --acls --xattrs -C rootfs-c9d-v1/ -c .| tar -C rootfs-c9d-v1.s1/-acls--xattrs-x
```

对快照的任何更改都不会影响复制的源目录。例如，rootfs-c9d-v1.s1 就是 rootfs-c9d-v1 的一个完全相同的快照。 

**写时复制或联合文件系统可有效制作目录快照**。

快照的初始布局：

```shell
rootfs-c9d-v1.s1/
  etc/
    my-app-config
  bin/
    my-app-binary
    my-app-tools
```

对快照进行一些更改（添加、修改和删除）：

```shell
rootfs-c9d-v1.s1/
  etc/
    my-app.d/
      default.cfg
  bin/
    my-app-binary
    my-app-tools
```

#### Determining Changes

比较两个目录时，相对根目录是顶层目录。比较目录时，会查找已添加、修改或删除的文件。在本例中，rootfs-c9d-v1/ 和 rootfs-c9d-v1.s1/ 被递归比较，每个目录都是相对根路径。

发现以下changeset：

```shell
Added:      /etc/my-app.d/
Added:      /etc/my-app.d/default.cfg
Modified:   /bin/my-app-tools
Deleted:    /etc/my-app-config
```

#### Representing Changes

创建一个只包含该changeset的tar压缩包：

- 添加和修改的文件和目录的全部内容
- 已删除的文件或目录，用"whiteout"标记

生成的rootfs-c9d-v1.s1 tar压缩包有以下条目：

```shell
./etc/my-app.d/
./etc/my-app.d/default.cfg
./bin/my-app-tools
./etc/.wh.my-app-config
```

删除的资源例如./etc/myapp-config，名称需以.wh为前缀。

### Applying Changesets

- 媒体类型为application/vnd.oci.image.layer.v1.tar的镜像层changeset会被应用，而不是被解压
- 应用镜像层changeset需要特别考虑“whiteout”文件
- 如果镜像层changeset中没有任何“whiteout”文件，则会像解压普通tar压缩包一样进行解压
  现有文件上的变更集
  本节说明了在目标路径已经存在的情况下应用层变更集的条目。

#### Changeset over existing files

如果要应用的layer changeset entry和现有目标路径都是目录，那么现有路径的属性必须替换为changeset entry的属性。在所有其他情况下，实现必须执行语义等同的以下操作：

- 删除文件路径（例如Linux系统上的unlink）
- 根据changeset entry内容和属性重新创建文件路径

### Whiteouts

- "whiteout"文件是带有特殊命名的空文件，用于表示需要删除的文件
- "whiteout"文件名由".wh."前缀加需要被删除的文件路径组成
- 以".wh."为前缀的文件是特殊的屏蔽标记，因此在一个文件系统不可能创建以".wh."开头的文件或目录
- whiteout被应用时（忽略文件），".wh."文件本身也需要被隐藏
- whiteout文件只能对下层/父层的资源产生效果，而不能对更高一层的资源产生效果
- 和".wh."处于同一层的文件只能由更高层中的".wh."文件来执行whiteout

这一部分主要还是将对changeset的应用，关于镜像各layer是如何变成一个完整的文件系统阅读[手撕docker文件结构 —— overlayFS，image，container文件结构详解](https://zhuanlan.zhihu.com/p/374924046)。

## 
