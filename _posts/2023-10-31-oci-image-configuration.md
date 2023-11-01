---
layout: post
title: OCI Image Specification——Image Configuration
subtitle: OCI Image Configuration
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
## OCI Image Configuration

### Terminology

#### Layer

- 镜像的文件系统由各镜像层组成
- 每一层以基于tar的层形式表示此层对于父层的changeset，记录相对于其父层要添加、更改或删除的文件
- 每一层没有配置元数据（image的config.json），如环境变量或默认参数，这些是整个镜像的属性，而不是任何特定层的属性
- 通过使用基于层的文件系统或联合文件系统（如 AUFS和overlay2），或通过计算文件系统快照的差异，文件系统changeset可用于显示一系列镜像层，就好像它们是一个完整文件系统

#### Image JSON

也就是每个镜像的config.json文件。

- 每个镜像都有一个关联的JSON结构，其中描述了镜像的一些基本信息，如创建日期、作者，以及执行/运行时配置，如入口点、默认参数、网络和卷
- JSON结构还引用了镜像使用的每个层的加密哈希值，并提供了这些层的历史信息
- 该JSON被认为是不可变的，因为更改它将改变计算出的ImageID
- 更改它意味着创建一个新的镜像，而不是更改现有镜像

#### Layer DiffID

**Layer DiffID是镜像层未压缩tar包的摘要**，并以描述符摘要格式序列化，如 sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9。应重复打包和解包图层，以避免更改图Layer DiffID，例如使用tar-split保存tar头文件。

**注意：不要将Layer DiffID与layer digest（manifest中的）混淆，后者是压缩或未压缩内容的摘要**。

> 也就是说layer digest是对每一层实际内容做sha256计算所得，而layer diffid是对其tar包做sha256计算

#### Layer ChainID

Layer DiffID标识单个changeset，而Layer ChainID则标识这些changeset的后续应用。这就确保了我们既能处理镜像层本身，也能处理一系列changeset的应用结果。在将镜像层应用到根文件系统时，可与rootfs.diff_ids结合使用，以唯一、安全地识别结果。

##### Definition

layer chainid的计算方式如下：

```shell
ChainID(L₀) =  DiffID(L₀)
ChainID(L₀|...|Lₙ₋₁|Lₙ) = Digest(ChainID(L₀|...|Lₙ₋₁) + " " + DiffID(Lₙ))
```

|：将右操作数应用于左操作数的结果。例如，给定base layer（基础层）A 和changeset（变更集）B，我们将B应用于A的结果称为A|B。digest为做sha256计算。

##### Explanation

假如有3层A、B和C，顺序从底到顶，A为最底层，C为最高层，三个层的chainID（准确来说，高层的chainid是其前面多个层相作用的结果，也就是说ChainID(A|B|C)和ChainID(C)是不一样的）计算方式如下：

```shell
ChainID(A) = DiffID(A)
ChainID(A|B) = Digest(ChainID(A) + " " + DiffID(B))
ChainID(A|B|C) = Digest(ChainID(A|B) + " " + DiffID(C))
// 等价于
ChainID(A|B|C) = Digest(Digest(DiffID(A) + " " + DiffID(B)) + " " + DiffID(C))
```

可以看到单个层（或者说最底层）的chainid等于其diffid，而多个层的chainid是前面多层的结果。

#### ImageID

每个镜像的ID等于对其config文件做SHA256计算得出的哈希值	。它以256位的十六进制编码表示，例如，sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9。

镜像配置文件中包含了镜像中每个层的散列值（rootf.diff_ids），ImageID的这种计算方式使镜像具有内容可寻址性。

### Properties

- **created** *string*, OPTIONAL：创建镜像的日期和时间
- **author** *string*, OPTIONAL：创建并负责维护镜像的个人或实体的姓名和/或电子邮件地址
- **architecture** *string*, REQUIRED：运行镜像机器的CPU架构，[`GOARCH`](https://golang.org/doc/install/source#environment)
- **os** *string*, REQUIRED：运行镜像的操作系统名称，[`GOOS`](https://golang.org/doc/install/source#environment)
- **os.version** *string*, OPTIONAL：version与宿主机的version不匹配时，镜像可能无法运行
- **os.features** *array of strings*, OPTIONAL
- **variant** *string*, OPTIONAL：指定CPU架构的变体
- **config** *object*, OPTIONAL：使用此镜像运行容器时的执行参数。此字段可以为空，在这种情况下，应在创建容器时指定任何执行参数
  - **User** *string*, OPTIONAL：貌似是指定运行镜像的uid
  - **ExposedPorts** *object*, OPTIONAL：运行此镜像的容器中暴露的一组端口
  - **Env** *array of strings*, OPTIONAL
  - **Entrypoint** *array of strings*, OPTIONAL：作为容器启动时要执行的命令的参数列表。这些值是默认值，也可以由创建容器时指定的入口点代替
  - **Cmd** *array of strings*, OPTIONAL：容器入口点的默认参数。这些值是默认值，可以用创建容器时指定的任何值代替。如果没有指定entrypoint值，那么Cmd数组的第一个条目应被解释为要运行的可执行文件，这两者也可结合起来一起用。[entrypoint和cmd的区别](https://zhuanlan.zhihu.com/p/30555962)
  - **Volumes** *object*, OPTIONAL：挂载卷的目录，也就是持久化数据的位置
  - **WorkingDir** *string*, OPTIONAL：设置容器中entrypoint进程的当前工作目录。此值为默认值，可由创建容器时指定的工作目录代替
  - **Labels** *object*, OPTIONAL：容器的相关的元数据
  - **StopSignal** *string*, OPTIONAL：该字段包含将发送给容器以退出的系统调用信号，例如 SIGKILL 或 SIGRTMIN+3
  - **Memory** *integer*, OPTIONAL
  - **MemorySwap** *integer*, OPTIONAL
  - **CpuShares** *integer*, OPTIONAL
  - **Healthcheck** *object*, OPTIONAL
- **rootfs** *object*, REQUIRED：镜像所使用的各镜像层的内容地址（sha256），这也使得镜像配置文件的哈希值依赖于各镜像层的哈希值
  - **type** *string*, REQUIRED：固定为layers
  - **diff_ids** *array of strings*, REQUIRED：镜像层内容哈希值（DiffID）数组，顺序为从第一个到最后一个
- **history** *array of objects*, OPTIONAL：描述每个镜像层的历史，顺序为从第一个到最后一个
  - **created** *string*, OPTIONAL
  - **author** *string*, OPTIONAL
  - **created_by** *string*, OPTIONAL：创建此镜像层的命令
  - **comment** *string*, OPTIONAL
  - **empty_layer** *boolean*, OPTIONAL：该字段用于标记历史项是否创建了文件系统差异。如果该历史项与 rootfs 部分的实际层不对应（例如，Dockerfile的ENV命令不会导致文件系统发生变化），则该字段将设为 true

```json
{
    "created": "2015-10-31T22:22:56.015925234Z",
    "author": "Alyssa P. Hacker <alyspdev@example.com>",
    "architecture": "amd64",
    "os": "linux",
    "config": {
        "User": "alice",
        "ExposedPorts": {
            "8080/tcp": {}
        },
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "FOO=oci_is_a",
            "BAR=well_written_spec"
        ],
        "Entrypoint": [
            "/bin/my-app-binary"
        ],
        "Cmd": [
            "--foreground",
            "--config",
            "/etc/my-app.d/default.cfg"
        ],
        "Volumes": {
            "/var/job-result-data": {},
            "/var/log/my-app-logs": {}
        },
        "WorkingDir": "/home/alice",
        "Labels": {
            "com.example.project.git.url": "https://example.com/project.git",
            "com.example.project.git.commit": "45a939b2999782a3f005621a8d0f29aa387e1d6b"
        }
    },
    "rootfs": {
      "diff_ids": [
        "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
        "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      ],
      "type": "layers"
    },
    "history": [
      {
        "created": "2015-10-31T22:22:54.690851953Z",
        "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
      },
      {
        "created": "2015-10-31T22:22:55.613815829Z",
        "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
        "empty_layer": true
      },
      {
        "created": "2015-10-31T22:22:56.329850019Z",
        "created_by": "/bin/sh -c apk add curl"
      }
    ]
}
```

 
