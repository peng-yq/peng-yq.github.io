---
layout: post
title: Container Hooks Toolkit
subtitle: Introduction of Container Hooks Toolkit
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

本文是对个人项目 [Containers Hooks Toolkit 的详细介绍](https://github.com/peng-yq/container-hooks-toolkit)。

## Overview

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/202407071324944.png"/>

`container-hooks-toolkit`用于向容器的配置文件(`config.json`)中插入自定义的`oci hooks`，以实现在容器的不同生命周期进行细粒度的操作，组件包括：

- `container-hooks-runtime`
- `container-hooks-ctk`
- `container-hooks`

### container-hooks-runtime

`container-hooks-runtime`是对主机上安装的`runc`的轻量级包装器，通过将指定的`oci hooks`注入容器的运行时规范，然后调用主机本地的`runc`，并传递修改后的带有钩子设置的容器运行时规范。`runc`在启动容器时，会自动运行注入的`oci hooks`。

`container-hooks-runtime`的配置文件包含了`container-hooks-runtime`的配置选项，路径为`/etc/container-hooks/config.toml`，支持对其修改从而定义`container-hooks-runtime`的日志文件路径、日志级别以及底层运行时：

```toml
[container-hooks-runtime]
debug = "/etc/container-hooks/container-hooks-runtime.log"
log-level = "info"
runtimes = ["runc", "docker-runc"]
```

`container-hooks-runtime`日志记录了容器生命周期的相关记录，默认路径为`/etc/container-hooks/container-hooks-runtime.log`。

### container-hooks-ctk

`container-hooks-ctk` 是一个命令行工具，主要用于配置各容器运行时支持`container hooks runtime`，以向符合`OCI`规范的容器中插入`OCI Hooks`。

> `container-hooks-runtime`的完成介绍和用法见[README of container-hooks-ctk](https://github.com/peng-yq/container-hooks-toolkit/blob/main/cmd/container-hooks-ctk/README.md)。

**`container-hooks-ctk`所有操作均需要`root`权限**。

```shell
container-hooks-ctk [global options] command [command options] [arguments...]
```

`container-hooks-ctk`支持的主要命令（每个命令又包含若干子命令）包括：

1. `runtime`：`runtime`命令用于配置各容器运行时支持/移除`container-hooks-runtime`。
2. `config`：`config`命令用于生成`container hooks toolkit`的配置文件。
3. `install`：`install`命令用于将`container hooks toolkit`复制到`/usr/bin`目录。

`container-hooks-ctk`生成的默认配置文件路径为`/etc/container-hooks/config.toml`，默认内容如下：

```toml
[container-hooks]
# 插入的oci hooks文件的路径
path = "/etc/container-hooks/hooks.json"

[container-hooks-ctk]
# container-hooks-ctk工具的路径
path = "/usr/bin/container-hooks-ctk"

[container-hooks-runtime]
# container hooks runtime日志文件路径
debug = "/etc/container-hooks/container-hooks-runtime.log"
# 日志文件记录级别
log-level = "info"
# 默认底层容器运行时
runtimes = ["runc", "docker-runc"]
```

### container-hooks

`container-hooks`是一个空程序，**仅用于判断当前容器是否已经添加自定义`hooks`**，避免重复添加。

## Usecase

`container-toolkit`支持`docker`，`containerd`和`cri-o`，简单用例介绍如下。

1.下载

```shell
git clone https://github.com/peng-yq/container-hooks-toolkit.git
```

2.编译

```shell
make all
```

> 下面的所有操作均需要`root`权限

3.复制到`/usr/bin`

```shell
cd bin
./container-hooks-ctk install --toolkit-root=$(pwd)
```

4.生成配置文件

```shell
container-hooks-ctk config
```

5.以`docker`为例，进行配置

```shell
container-hooks-ctk runtime configure --runtime=docker --default
systemctl restart docker
```

配置完成后的`docker`配置文件：

```json
{
    "default-runtime": "container-hooks-runtime",
    "registry-mirrors": [
        "https://yxzrazem.mirror.aliyuncs.com"
    ],
    "runtimes": {
        "container-hooks-runtime": {
            "path": "/usr/bin/container-hooks-runtime",
            "runtimeArgs": []
        }
    }
}
```

6.编写自定义`oci hooks`，格式如下，必须添加第一个`prestart hook`中的`container-hooks`用于避免重复添加定义`hooks`，需要写入至`/etc/container-hooks/hooks.json`文件中（此路径可在配置文件中修改，请注意json格式问题）

```shell
{
  "prestart": [
    {
      "path": "/usr/bin/container-hooks"
    },
    {
      "path": "/etc/container-hooks/test.sh"
    }
  ]
}
```

这里用一个简单的脚本文件做为实例，脚本内容如下，指向脚本会往`/etc/container-hooks/test.txt`中追加`Hello World`：

```shell
#!/bin/bash

echo "Hello World" >> /etc/container-hooks/test.txt
```

7.运行容器，执行自定义`hook`

```shell
sudo docker run hello-world
```

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/202407071152852.png"/>

## Container Runtime

`Runtime`在中文博客中一般被翻译为运行时，比如会经常看到`docker`、`containerd`、`cri-o`和`runc`等都被称为`runtime`。但实际上他们的定义是不一样的，`OCI`规范中的原文介绍如下：

`OCI`对容器的定义也很直接了当，这里也介绍一下。

### container

**An environment for executing processes with configurable isolation and resource limitations.** For example, namespaces, resource limits, and mounts are all part of the container environment.

我们知道容器是一种轻量级的虚拟化技术，与宿主机共用操作系统内核。容器其实就是可配置隔离和资源限制的进程执行环境，通过`namespace`技术（将系统资源：如网络接口、进程ID列表、挂载点虚拟化）来确保容器内的应用程序进程与系统上的其他进程相隔离，容器中的进程会感觉它们是在一个独立的系统中运行。并通过`cgroup`技术控制和限制容器可以使用的物理资源（如 CPU 时间、内存用量等）的能力。

### runtime

An implementation of this specification. **It reads the configuration files from a bundle, uses that information to create a container, launches a process inside the container, and performs other lifecycle actions.**

`OCI`对`runtime`的定义就是根据容器配置文件创建容器进程，控制容器生命周期的工具，即低级运行时，例如`runc`和`kata-container`。

### runtime caller

**An external program to execute a runtime, directly or indirectly.**

**Examples of direct callers include containerd, CRI-O, and Podman. Examples of indirect callers include Docker/Moby and Kubernetes.**

Runtime callers often execute a runtime via runc-compatible command line interface, however, its interaction interface is currently out of the scope of the Open Container Initiative Runtime Specification.

中文博客对运行时的描述其实是模糊的，极其容易让人混淆。`OCI`对此做了一个很好的定义，运行时 (`runtime`)只是最底层的低级的直接和容器交互的程序；而高级的面向用户的，一般直接或间接调用运行时的程序则称为`runtime caller`。`runtime caller`也可根据`runtime`的关系，再细分为直接和间接的，直接的例如`containerd`、`cri-o`等；间接的比如`k8s`和`docker`。一般来说用户直接接触的都是`indirect runtime caller`。

## Container Shim

在[Overview](##Overview)中提到`container-hooks-runtime`是`shim`，那`shim`是什么呢？

每一个`Containerd`或`Docker`容器（实际上`Docker`也是调用`Containerd`来创建容器，具体关系见下图）都有一个相应的 "`shim`" 守护进程，这个守护进程会提供一个`API`，`Containerd`使用该`API`来管理容器基本的生命周期（启动/停止），在容器中执行新的进程、调整`TTY`的大小以及与特定平台相关的其他操作。`shim`还有一个作用是向`Containerd`报告容器的退出状态，在容器退出状态被`Containerd`收集之前，`shim`会一直存在。这一点和僵尸进程很像，僵尸进程在被父进程回收之前会一直存在，只不过僵尸进程不会占用资源，而`shim`会占用资源。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/202407071338613.png"/>

`Shim`将`Containerd`进程从容器的生命周期中分离出来，具体的做法是`runc`在创建和运行容器之后退出，并将`shim`作为容器的父进程，即使`Containerd`进程挂掉或者重启，也不会对容器造成任何影响。这样做的好处很明显，你可以高枕无忧地升级或者重启`Containerd`，不会对运行中的容器产生任何影响。

## Container Bundle

从`runtime`的定义“It reads the configuration files from a bundle, uses that information to create a container, launches a process inside the container”中可以知道`runtime`读取`bundle`中的配置文件来创建容器。

`Bundle`只涉及如何将容器及其配置数据存储在本地文件系统中，以便兼容`OCI`规范的任何运行时加载。`Bundle`包含加载和运行容器所需的全部信息：

1. `config.json`：命名**必须**为`config.json`，且**必须**在`bundle`目录的根目录
2. 容器的根文件系统：在`config.json`中的`root.path`字段指定

`Bundle`目录中的内容才是容器必须，其目录本身非必须。

在实际调用过程中，`containerd`等`runtime caller`会将准备好的`bundle`目录等参数传递给`runc`等`runtime`，由`runc`来根据配置创建或控制容器的生命周期。

### Config.json

容器的配置文件包含了构建标准容器的元数据，包括用户指定进程、运行环境和环境变量等。内容比较多，详细介绍可见[博客](https://peng-yq.github.io/2023/10/24/runtime-spec-configuration/)。其中和本工具直接相关的为`POSIX-platform Hooks`部分的内容。

### RootFS

`RootFS`是容器在启动时**内部进程可见的文件系统**，即容器的根目录。当我们运行`docker exec`命令进入容器的时候看到的文件系统就是`rootfs`。`RootFS`通常包含一个操作系统运行所需的文件系统，例如可能包含典型的类`Unix`操作系统中的目录系统，如`/dev`、`/proc`、`/bin`、`/etc`、`/lib`、`/usr`、`/tmp`及运行 容器所需的配置文件、工具等。

就像`Linux`启动会先用只读模式挂载`rootfs`，运行完完整性检查之后，再切换成读写模式一样。容器挂载`rootfs`时，也会先挂载为只读模式，但是与`Linux`做法不同的是，在挂载完只读的`rootfs`之后，会利用联合挂载技术（`Union Mount`）在已有的`rootfs`上再挂一个读写层。容器在运行过程中文件系统发生的变化只会写到读写层，并通过`whiteout`技术隐藏只读层中的旧版本文件。

> 关于`overlay`、联合挂载等技术的详细描述见[手撕docker文件结构 —— overlayFS，image，container文件结构详解](https://zhuanlan.zhihu.com/p/374924046)

### How runc create, start and delete container?

以`docker run ubuntu:latest`为例，`containerd`传递给`runc`会经历四个阶段：

1. `create`
2. `start`
3. `delete`：因为基础的`ubuntu`容器没有设置`entrypoint`，因此容器启动后会马上退出
4. `delete –force`

`runc create`时的调用参数如下，准确来说应该是`global options`。

1. `–-root`指定了用于存储容器状态的根目录，这个根目录是`tmpfs`（类Unix系统上暂存档存储空间的常见名称，通常以挂载文件系统方式实现，并将资料存储在易失性存储器即内存而非永久存储设备中），这里是`/var/run/docker/runtime-runc/moby`。[关于moby的介绍](https://cn.linux-console.net/?p=7835#gsc.tab=0)
2. `--log`指定了`runc`日志文件。
3. `-–log-format`指定了`runc`日志文件格式为`json`。
4. `–-systemd-cgroup`开启`systemd cgroup`支持，这也是容器资源隔离机制的关键。
5. `--bundle`指定的目录和`runc`日志的目录一致。
6. `-–pid-file`指定了容器进程的`pid`路径。

```shell
--root /var/run/docker/runtime-runc/moby --log
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012fce6917e2f7e2c73958b8530d369fef05/log.json --log-format json --systemd-cgroup 
create --bundle 
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012
fce6917e2f7e2c73958b8530d369fef05 --pid-file 
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012
fce6917e2f7e2c73958b8530d369fef05/init.pid 
6dbd69e6f9e0c7efe3026ae83624012fce6917e2f7e2c73958b8530d369fef05
```

日志输出的`runc start`时的调用参数如下，相比于`create`参数少了一些，基本参数保持一致，`6dbd69e…`为容器id。

```shell
--root /var/run/docker/runtime-runc/moby --log 
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012
fce6917e2f7e2c73958b8530d369fef05/log.json --log-format json --systemd-cgroup 
start 6dbd69e6f9e0c7efe3026ae83624012fce6917e2f7e2c73958b8530d369fef05
```

日志输出的`runc delete`时的调用参数如下，和`start`的参数一致。

```shell
--root /var/run/docker/runtime-runc/moby --log 
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012
fce6917e2f7e2c73958b8530d369fef05/log.json --log-format json --systemd-cgroup 
delete 6dbd69e6f9e0c7efe3026ae83624012fce6917e2f7e2c73958b8530d369fef05
```

在`runc delete`后，还有个`runc delete –-force`的操作，参数如下。

```shell
--root /var/run/docker/runtime-runc/moby --log 
/run/containerd/io.containerd.runtime.v2.task/moby/6dbd69e6f9e0c7efe3026ae83624012
fce6917e2f7e2c73958b8530d369fef05/log.json --log-format json delete --force 
6dbd69e6f9e0c7efe3026ae83624012fce6917e2f7e2c73958b8530d369fef05
```

## Oci Hooks

什么是钩子呢？钩子主要有两个关键点，**一个是导致钩子调用的事件（某个事件发生前或者发生后），一个是钩子的具体代码。因此，钩子实际上就是在某个时间点或事件点触发的一系列函数或代码**。

容器中的钩子和容器的生命周期息息相关，**钩子能使容器感知其生命周期内的事件，并且当相应的生命周期钩子被调用时运行指定的代码**。[OCI Runtime Spec](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md#lifecycle)对容器生命周期的描述如下（单纯从低级运行时创建容器开始，不包括镜像）。

`Lifecycle`定义了容器从创建到退出之间的时间轴：

1. 容器开始创建：**通常为`OCI`规范运行时（`runc`）调用`create`命令 + `bundle` + `container id`**
2. 容器运行时环境创建中： 根据容器的`config.json`中的配置进行创建，此时用户指定程序还未运行，这一步后所有对`config.json`的更改均不会影响容器
3. `prestart hooks`
4. `createRuntime hooks`
5. `createContainer hooks`
6. 容器启动：**通常为`OCI`规范运行时（`runc`）调用`create`命令 + `bundle` + `container id`**
7. `startContainer hooks`
8. 容器执行用户指定程序
9. `poststart hooks`：任何`poststart`钩子执行失败只会`log a warning`，不影响其他生命周期（操作继续执行）就好像钩子成功执行一样
10. 容器进程退出：`error`、正常退出和运行时调用`kill`命令均会导致
11. 容器删除：**通常为`OCI`规范运行时（`runc`）调用`delete`命令 +  `container id`**
12. 容器摧毁：**区别于容器删除，`3`、`4`、`5`、`7`的钩子执行失败除了生成一个`error`外，会直接跳到这一步**。撤销第二步创建阶段执行的操作。
13. `poststop hooks`：执行失败后的操作和`poststart`一致

可以看到`oci`定义的容器生命周期中，如果在容器的`config.json`中定义了钩子，`runc`必须执行钩子，并且时间节点在前的钩子执行成功后才能执行下一个钩子；若有一个钩子执行失败，则会报错并摧毁容器（在容器创建后执行的钩子失败，并不会删除容器，而是启动失败）。

`OCI`规定单个钩子的字段如下：

- `path (string, REQUIRED) `：绝对路径
- `args (array of strings, OPTIONAL)`
- `env (array of strings, OPTIONAL)`
- `timeout (int, OPTIONAL)` ：终止钩子的秒数

#### Prestart

`Prestart`钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 `config.json `中的配置），但在执行 `pivot_root` 或任何同等操作之前调用。

**即将废弃，被后面三个钩子所取代，注意后面三个钩子在较老版本的runc中可能不支持**。

#### CreateRuntime Hooks

`createRuntime`钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据` config.json `中的配置），但在执行` pivot_root` 或任何同等操作之前调用。

**在容器命名空间被创建后调用**。

> `createRuntime` 钩子的定义目前未作明确规定，钩子作者只能期望运行时创建挂载命名空间并执行挂载操作。运行时可能尚未执行其他操作，如 `cgroups `和 `SELinux/AppArmor` 标签

#### CreateContainer Hooks

`createContainer`钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 `config.json `中的配置），但在执行` pivot_root` 或任何同等操作之前调用。

**在执行 pivot_root 操作之前，但在创建和设置挂载命名空间之后调用**。

#### StartContainer Hooks

`StartContainer`钩子作为启动操作的一部分，**必须在执行用户指定的进程之前调用`startContainer`挂钩**。此钩子可用于在容器中执行某些操作，例如在容器进程生成之前在`linux`上运行`ldconfig`二进制文件。

#### Poststart

`Poststart`钩子**必须在用户指定的进程执行后、启动操作返回前调用**。例如，此钩子可以通知用户容器进程已生成。

#### Poststop

`Poststart`钩子**必须在容器删除后、删除操作返回前调用**。清理或调试函数就是此类钩子的例子。

#### summary

**`namespace`是指`path`以及钩子必须在指定的`namespace`中解析或调用，比如`runtime`命名空间表示访问的钩子`path`是宿主机上的路径；而`container`为容器中的路径**。

| Name                    | Namespace | When                                                         |
| :---------------------- | :-------- | :----------------------------------------------------------- |
| `prestart` (Deprecated) | runtime   | After the start operation is called but before the user-specified program command is executed. |
| `createRuntime`         | runtime   | During the create operation, after the runtime environment has been created and before the pivot root or any equivalent operation. |
| `createContainer`       | container | During the create operation, after the runtime environment has been created and before the pivot root or any equivalent operation. |
| `startContainer`        | container | After the start operation is called but before the user-specified program command is executed. |
| `poststart`             | runtime   | After the user-specified process is executed but before the start operation returns. |
| `poststop`              | runtime   | After the container is deleted but before the delete operation returns. |

## How Container Hooks Toolkit Works

有了前面的铺垫，最后对`container hooks toolkit`是如何工作的进行介绍：

1. 当我们执行`docker run`命令创建并启动一个新的容器时，这个命令会被发送到 `Docker`守护进程 (`dockerd`)。
2. 接收到来自`Docker`客户端（`docker cli`）的命令后，`dockerd`解析这些命令。`dockerd`根据参数检查本地是否存在指定的镜像，如果不存在，它会从配置的`Docker`镜像仓库（默认是`Docker Hub`）下载镜像。`dockerd`创建一个容器配置，并将其传递给容器引擎 (`containerd`)。
3. `containerd`接收来自 `dockerd` 的请求，并负责进一步处理这些请求，如创建、启动、停止容器等。`containerd` 会生成一个容器规范（如` OCI `规范，也就是前面提到的`config.json`），并将其传递给较低级别的容器运行时，这里是我们配置的`container-hooks-runtime`。
4. `container-hooks-runtime`对传递来的参数进行解析，如果是`create/start`，就解析出`/etc/container-hooks/hooks.json`中钩子并插入到容器的`bundle`路径下的`config.json`中，并传递给`runc`。
5. 如果是非`create/start`命令，`container-hooks-runtime`就直接将参数传递给`runc`。
6. `runc`根据传递而来的`OCI`规范设置必要的命名空间、控制组、根文件系统等，如果设置了容器钩子，则在特定的生命周期节点运行，然后启动容器的入口点进程（`PID 1`）。

但在执行`docker run`命令时，对`runc`先传递`create`，再传递`start`，这样不就添加了两次自定义钩子吗？是的，但我们提供了`container-hooks`这个空程序，用于避免重复添加自定义钩子，但`container-hooks-runtime`解析到容器的配置文件`config.json`中的`prestart`钩子中存在`container-hooks`，则不会再次添加自定义钩子。

## Customized

提供一些更加定制化的思路：

案例1：在容器启动前自动对容器进行签名验证和完整性校验

此时直接编写`hooks`至`/etc/container-hooks/hooks.json`就行不通了，因为我们无法提前预知每个容器的启动镜像信息。可以对项目进行二次开发，不采用读取文件中的钩子的形式，而是直接在代码中进行插入并根据容器的配置进行参数调整。

需要修改的代码部分：

1. `/internel/runtime`
2. `/internel/modifier`



