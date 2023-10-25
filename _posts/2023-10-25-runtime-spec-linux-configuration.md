---
layout: post
title: Open Container Initiative Runtime Linux Specification Configuration
subtitle: OCI Runtime-Spec/config.json (linux spec)
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

本篇主要是container configuration（config.json）中[linux容器规范部分](https://github.com/opencontainers/runtime-spec/blob/6331715093bfcf25137411bfacb403235ed7d018/config-linux.md)。 

Linux容器规范使用了各种内核特性，如namespaces、cgroups、capabilities、LSM 和文件系统隔离。

## Default Filesystems

Linux ABI包括系统调用和几个特殊的文件路径，需要为Linux环境下运行的应用程序正确设置这些文件路径。在每个容器的文件系统中，应该提供以下文件系统：

| Path     | Type                                                         | Use                                                          |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| /proc    | [proc](https://www.kernel.org/doc/Documentation/filesystems/proc.txt) | 提供了有关运行中进程和系统状态的信息                         |
| /sys     | [sysfs](https://www.kernel.org/doc/Documentation/filesystems/sysfs.txt) | 提供了对内核和硬件设备的访问接口                             |
| /dev/pts | [devpts](https://www.kernel.org/doc/Documentation/filesystems/devpts.txt) | 用于处理终端设备的访问                                       |
| /dev/shm | [tmpfs](https://www.kernel.org/doc/Documentation/filesystems/tmpfs.txt) | 用于在内存中创建共享内存段，使容器内的进程可以使用共享内存进行进程间通信（IPC）操作，如共享数据、信号量和互斥锁等 |

> ABI是 "Application Binary Interface"（应用程序二进制接口）的缩写。ABI 定义了一个应用程序与操作系统或其他软件组件之间的接口规范，以便它们能够相互通信和交互。它规定了二进制程序的格式、函数调用约定、参数传递方式、系统调用接口、寄存器的使用方法等方面的规则和约定。
>
> API 是 "Application Programming Interface"（应用程序编程接口）的缩写。API 定义了软件组件之间的接口规范和交互方式，以便应用程序可以与其他软件组件（如操作系统、库、服务等）进行通信和交互。API 提供了一组预定义的函数、协议、数据结构和工具，用于构建应用程序和组件之间的通信通道。开发人员可以使用 API 来访问和利用其他软件组件的功能，而无需了解其内部实现细节。

## Namespaces

命名空间将全局系统资源抽象化，使命名空间内的进程看起来拥有自己的全局资源独立实例。作为命名空间成员的其他进程可以看到对全局资源的更改，但命名空间外的其他进程则看不到。每个容器都有单独的命名空间：

- type (string, REQUIRED)：命名空间类型，应支持以下命名空间
  - pid：容器内的进程只能看到同一容器内或同一pid命名空间内的其他进程
  - network：容器有自己的网络栈
  - mount：容器有独立的挂载表
  - ipc：容器内的进程只能通过系统级IPC与同一容器内的其他进程通信
  - uts：容器将能够拥有自己的主机名和域名
  - user：容器将能够把主机上的用户和组ID重新映射到容器内的本地用户和组
  - cgroup：容器有单独的cgroup层次
  - time：容器有自己的时间（clock）
- path (string, OPTIONAL)：与具体命名空间类型相关联的命名空间文件。该值必须是运行时挂载命名空间中的绝对路径，运行时必须将容器进程置于与该路径相关联的命名空间中。

```json
"namespaces": [
    {
        "type": "pid",
        "path": "/proc/1234/ns/pid"
    },
    {
        "type": "network",
        "path": "/var/run/netns/neta"
    },
    {
        "type": "mount"
    },
    {
        "type": "ipc"
    },
    {
        "type": "uts"
    },
    {
        "type": "user"
    },
    {
        "type": "cgroup"
    },
    {
        "type": "time"
    }
]
```

## User namespace mappings

uidMappings (array of objects, OPTIONAL) ：描述了从主机到容器的用户命名空间的uid映射

gidMappings (array of objects, OPTIONAL)：描述了从主机到容器的用户命名空间的gid映射

上述两个对象都有如下元素：

- containerID (uint32, REQUIRED) ：容器中的起始uid/gid
- hostID (uint32, REQUIRED) ：映射到容器containerID的主机上的起始uid/gid
- size (uint32, REQUIRED)：映射的id数量

运行时不应为实现映射而修改引用文件系统的所有权，并且内核可能会限制映射条目的数量。

```json
"uidMappings": [
    {
        "containerID": 0,
        "hostID": 1000,
        "size": 32000
    }
],
"gidMappings": [
    {
        "containerID": 0,
        "hostID": 1000,
        "size": 32000
    }
]
```

## Offset for Time Namespace

timeOffsets (object, OPTIONAL) ：标识时间命名空间中的偏移：

- secs (int64, OPTIONAL) ：容器中时钟的偏移量（以秒为单位）
- nanosecs (uint32, OPTIONAL) ：容器中时钟的偏移量（以纳秒为单位）

## Devices

devices (array of objects, OPTIONAL)：列出了容器中必须可用的设备，运行时需要提供这些设备的支持（例如使用mknod、从运行时命名空间绑定挂载或使用符号链接等）：

- type (string, REQUIRED) ：设备类型
- path (string, REQUIRED) ：容器中设备的路径
- major, minor (int64, REQUIRED unless type is p) ：设备的major和minor值
- fileMode (uint32, OPTIONAL) ：设备的文件模式
- uid (uint32, OPTIONAL)：容器命名空间中设备拥有者的id
- gid (uint32, OPTIONAL)：容器命名空间中设备拥有者的组id

>"major" 数字用于标识设备驱动程序，它指示了设备所属的驱动程序或设备类型。"minor" 数字用于标识设备在其所属驱动程序中的具体实例或编号。通过组合 "major" 和 "minor" 数字，可以唯一地标识一个设备。

```json
"devices": [
    {
        "path": "/dev/fuse",
        "type": "c",
        "major": 10,
        "minor": 229,
        "fileMode": 438,
        "uid": 0,
        "gid": 0
    },
    {
        "path": "/dev/sda",
        "type": "b",
        "major": 8,
        "minor": 0,
        "fileMode": 432,
        "uid": 0,
        "gid": 0
    }
]
```

## Default Devices

运行时必须支持如下设备：

- /dev/null
- /dev/zero
- /dev/full
- /dev/random
- /dev/urandom
- /dev/tty
- /dev/console
- /dev/ptmx

## Control groups  (cgroups)

cgroups用于限制容器的 CPU、内存、IO、pids、网络和RDMA资源。

### Cgroups Path

cgroupsPath (string, OPTIONAL)：cgroups路径，可以用来控制容器的cgroups层次结构，也可用于在现有容器中运行新进程。

### Cgroup ownership

```json
"cgroupsPath": "/myRuntime/myContainer",
"resources": {
    "memory": {
    "limit": 100000,
    "reservation": 200000
    },
    "devices": [
        {
            "allow": false,
            "access": "rwm"
        }
    ]
}
```

### Allowed Device list

devices (array of objects, OPTIONAL)：

- allow (boolean, REQUIRED) ：设备是否被允许
- type (string, OPTIONAL) ：设备类型，a (all, default)、b (block)或c (char)
- major, minor (int64, OPTIONAL)
- access (string, OPTIONAL)：设备的cgroup权限，r (read), w (write), and m (mknod)

```json
"devices": [
    {
        "allow": false,
        "access": "rwm"
    },
    {
        "allow": true,
        "type": "c",
        "major": 10,
        "minor": 229,
        "access": "rw"
    },
    {
        "allow": true,
        "type": "b",
        "major": 8,
        "minor": 0,
        "access": "r"
    }
]
```

### Memory

memory (object, OPTIONAL)：用于设置容器的内存限制，单位为字节，-1表示不做限制：

- limit (int64, OPTIONAL)：内存使用限制
- swap (int64, OPTIONAL)：内存和swap限制
- kernel (int64, OPTIONAL, NOT RECOMMENDED) ：为内核内存做强制限制
- kernelTCP (int64, OPTIONAL, NOT RECOMMENDED)：为内核tcp buffer内存做强制限制

其他的一些元素：

- swappiness (uint64, OPTIONAL) 
- disableOOMKiller (bool, OPTIONAL) 
- useHierarchy (bool, OPTIONAL)
- checkBeforeUpdate (bool, OPTIONAL)

```json
"memory": {
    "limit": 536870912,
    "reservation": 536870912,
    "swap": 536870912,
    "kernel": -1,
    "kernelTCP": -1,
    "swappiness": 0,
    "disableOOMKiller": false
}
```

### CPU

cpu (object, OPTIONAL)：

- shares (uint64, OPTIONAL)：任务中可用的共享CPU时间
- quota (int64, OPTIONAL)
- burst (uint64, OPTIONAL)
- period (uint64, OPTIONAL)
- realtimeRuntime (int64, OPTIONAL) 
- realtimePeriod (uint64, OPTIONAL) 
- cpus (string, OPTIONAL)
- mems (string, OPTIONAL)
- idle (int64, OPTIONAL)

```json
"cpu": {
    "shares": 1024,
    "quota": 1000000,
    "burst": 1000000,
    "period": 500000,
    "realtimeRuntime": 950000,
    "realtimePeriod": 1000000,
    "cpus": "2-3",
    "mems": "0-7",
    "idle": 0
}
```

### Block IO

```json
"blockIO": {
    "weight": 10,
    "leafWeight": 10,
    "weightDevice": [
        {
            "major": 8,
            "minor": 0,
            "weight": 500,
            "leafWeight": 300
        },
        {
            "major": 8,
            "minor": 16,
            "weight": 500
        }
    ],
    "throttleReadBpsDevice": [
        {
            "major": 8,
            "minor": 0,
            "rate": 600
        }
    ],
    "throttleWriteIOPSDevice": [
        {
            "major": 8,
            "minor": 16,
            "rate": 300
        }
    ]
}
```

### Huge page limits

```json
"hugepageLimits": [
    {
        "pageSize": "2MB",
        "limit": 209715200
    },
    {
        "pageSize": "64KB",
        "limit": 1000000
    }
]
```

### Network

```json
"network": {
    "classID": 1048577,
    "priorities": [
        {
            "name": "eth0",
            "priority": 500
        },
        {
            "name": "eth1",
            "priority": 1000
        }
    ]
}
```

### PIDs

limit (int64, REQUIRED)：指定任务的最大数量

```json
"pids": {
    "limit": 32771
}
```

### RDMA

```json
"rdma": {
    "mlx5_1": {
        "hcaHandles": 3,
        "hcaObjects": 10000
    },
    "mlx4_0": {
        "hcaObjects": 1000
    },
    "rxe3": {
        "hcaObjects": 10000
    }
}
```

## Unified

```json
"unified": {
    "io.max": "259:0 rbps=2097152 wiops=120\n253:0 rbps=2097152 wiops=120",
    "hugetlb.1GB.max": "1073741824"
}
```

## IntelRdt

intel相关的。

```json
"linux": {
    "intelRdt": {
        "closID": "guaranteed_group",
        "l3CacheSchema": "L3:0=7f0;1=1f",
        "memBwSchema": "MB:0=20;1=70"
    }
}
```

## Sysctl

```json
"sysctl": {
    "net.ipv4.ip_forward": "1",
    "net.core.somaxconn": "256"
}
```

## Seccomp

Linux内核的沙箱应用机制相关。

```json
"seccomp": {
    "defaultAction": "SCMP_ACT_ALLOW",
    "architectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "getcwd",
                "chmod"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
```

### The Container Process State

容器运行时通过UNIX套接字发送的容器进程的状态：

- ociVersion (string, REQUIRED)
- fds (array, OPTIONAL)：文件描述符名字
- pid (int, REQUIRED)：容器进程id
- metadata (string, OPTIONAL)
- state (state, REQUIRED)

```json
{
    "ociVersion": "1.0.2",
    "fds": [
        "seccompFd"
    ],
    "pid": 4422,
    "metadata": "MKNOD=/dev/null,/dev/net/tun;BPF_MAP_TYPES=hash,array",
    "state": {
        "ociVersion": "1.0.2",
        "id": "oci-container1",
        "status": "creating",
        "pid": 4422,
        "bundle": "/containers/redis",
        "annotations": {
            "myKey": "myValue"
        }
    }
}
```

## Rootfs Mount Propagation

```json
"rootfsPropagation": "slave",
```

## Masked Paths

使路径（容器命名空间）不可读。

```json
"maskedPaths": [
    "/proc/kcore"
]
```

## Readonly Paths

使路径（容器命名空间）只读。

```json
"readonlyPaths": [
    "/proc/sys"
]
```

## Mount Label

selinux相关的内容。

```json
"mountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c715,c811"
```

## Personality
