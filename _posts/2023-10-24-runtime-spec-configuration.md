---
layout: post
title: OCI Runtime Specification——Configuration
subtitle: OCI Runtime-Spec/config.json
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

[OCI Runtime Spec Configuration](https://github.com/opencontainers/runtime-spec/blob/main/config.md) 

容器的配置文件包含了构建标准容器的元数据，包括用户指定进程、运行环境和环境变量等。

本篇介绍以Linux平台为主。

## Specification version

只含ociVersion一个元素用于标识oci运行时规范版本号。

```json
// string, REQUIRED
"ociVersion": "0.1.0"
```

## Root

root对象指定了容器的根文件系统：

- path (string, REQUIRED) ：指定了容器的根文件系统，POSIX平台为绝对路径或者相对容器bundle的相对路径
- readonly (bool, OPTIONAL)：默认是false，标识根文件系统是否是只读

```json
"root": {
    "path": "rootfs",
    "readonly": true
}
```

## Mounts

mounts对象指定了根文件系统以外的额外挂载：

- destination (string, REQUIRED)：指定了容器中目录的挂载点，必须为绝对路径
- source (string, OPTIONAL)：设备名称，也可是绑定挂载的文件或目录名称或虚拟名称。绑定挂载的路径值可以是绝对路径，也可以是相对于bundle的路径，在选项中包含bind或rbind则为绑定挂载
- options (array of strings, OPTIONAL) ：所使用文件系统的挂载选项

> 绑定挂载和非绑定挂载：
>
> 绑定挂载是将一个目录或文件系统的内容绑定到另一个目录上，使得两者之间的更改是同步的。相反，非绑定挂载是指将文件系统的内容挂载到目标位置，但不会与源位置之间建立同步关系。非绑定挂载的一个常见示例是将一个独立的文件系统挂载到容器中的某个目录，而不与主机的相应目录建立同步关系。这样容器内对该目录的更改不会影响主机上的对应目录，反之亦然。

options元素内容较多，见[linux-mount-options](https://github.com/opencontainers/runtime-spec/blob/main/config.md#linux-mount-options)。

### POSIX-platform Mounts

posix平台mounts对象还可能包含以下元素：

- type (string, OPTIONAL)：文件系统类型，*/proc/filesystems* (e.g., "minix", "ext2", "ext3", "jfs", "xfs", "reiserfs", "msdos", "proc", "nfs", "iso9660")；绑定挂载为none
- uidMappings (array of type LinuxIDMapping, OPTIONAL)
- gidMappings (array of type LinuxIDMapping, OPTIONAL)

```json
"mounts": [
    {
        "destination": "/tmp",
        "type": "tmpfs",
        "source": "tmpfs",
        "options": ["nosuid","strictatime","mode=755","size=65536k"]
    },
    {
        "destination": "/data",
        "type": "none",
        "source": "/volumes/testing",
        "options": ["rbind","rw"]
    }
]
```

## Process

process对象指定了容器进程，start命令时这个字段必须：

- terminal (bool, OPTIONAL)：指定进程是否附加终端，默认为 false。如果在Linux下设置为 true，就会为进程分配一对伪终端，并在进程的标准数据流中复制伪终端pty
- consoleSize (object, OPTIONAL)：指定终端console的大小，terminal须为true
	- height (uint, REQUIRED)
	- width (uint, REQUIRED)
- cwd (string, REQUIRED) ：为可执行文件设置的工作目录，必须是绝对路径
- env (array of strings, OPTIONAL) ：环境变量
- args (array of strings, OPTIONAL) ：POSIX平台args数组中至少有一个对象

### POSIX process

对于支持POSIX rlimits的系统（如Linux和Solaris），进程对象支持以下特定于进程的属性：

- rlimits (array of objects, OPTIONAL)：指定进程的资源限制
  - type (string, REQUIRED)：限制的资源类型，例如RLIMIT_MSGQUEUE
  - soft (uint64, REQUIRED) ：对应资源的限制值
  - hard (uint64, REQUIRED) ：软限制可以设置的上限，只有特权进程才能提高硬限制

### Linux Process

基于Linux的系统中的process对象还支持以下元素：

- apparmorProfile (string, OPTIONAL)：指定进程的 AppArmor 配置文件名称
- capabilities (object, OPTIONAL) ：指定了进程所拥有的capabilities权能
  - effective (array of strings, OPTIONAL)：进程当前实际所拥有的权能集合
  - bounding (array of strings, OPTIONAL)：进程所能拥有的最大权能集合
  - inheritable (array of strings, OPTIONAL)：进程可以传递给其子进程的权能集合
  - permitted (array of strings, OPTIONAL)：进程被授予的权能，理论上拥有的权能集合，effective是permitted的子集
  - ambient (array of strings, OPTIONAL)：执行新程序时继承的权能，注意区别于inheritable，不能再继续传递
- noNewPrivileges (bool, OPTIONAL) ：为true时可防止进程获得额外能力
- oomScoreAdj (int, OPTIONAL) 
- scheduler (object, OPTIONAL)：描述进程的调度器属性
  - policy (string, REQUIRED) ：调度策略，比如先进先出啥的
  - nice (int32, OPTIONAL)：进程的优先级，值越小优先级越高
  - priority (int32, OPTIONAL) ：进程的静态优先级
  - flags (array of strings, OPTIONAL) ：调度的flag值
  - runtime (uint64, OPTIONAL)：给定时间段内允许进程运行的时间量（纳秒），由截止时间调度程序使用
  - deadline (uint64, OPTIONAL) ：进程完成执行的绝对截止时间，由截止时间调度程序使用
  - period (uint64, OPTIONAL) ：截止时间调度程序用于确定进程运行时间的周期长度（纳秒）
- selinuxLabel (string, OPTIONAL) ：指定了进程的selinux标签
- ioPriority (object, OPTIONAL)：指定容器进程的i/o优先级，适用容器中的所有进程
  - class (string, REQUIRED)：i/o调度类别
  - priority (int, REQUIRED)：指定了类中的优先级

>AppArmor（Application Armor）是一种Linux内核安全模块，用于实施操作系统级别的应用程序安全策略。它是一种强制访问控制（MAC）系统，旨在限制应用程序的权限，确保它们只能执行其预定义的操作，而不能越权访问系统资源。
>
>AppArmor通过定义应用程序的安全策略文件（profile）来工作。这些策略文件描述了应用程序可以访问的文件、目录、网络端口和其他系统资源。当应用程序运行时，AppArmor会监视其行为，并根据策略文件中定义的规则来限制其访问权限。如果应用程序试图执行未授权的操作，AppArmor会阻止其访问，并记录相关的安全事件。
>
>AppArmor提供了一种额外的安全层，可以帮助保护系统免受应用程序的潜在威胁。它可以用于限制网络服务、容器、虚拟机和其他应用程序的权限。AppArmor的配置可以针对特定的应用程序进行细粒度的调整，从而提供了更高的安全性和隔离性。

### User

user对象允许对进程作为哪个用户运行进行特定控制：

- uid (int, REQUIRED) ：指定了容器命名空间的用户id
- gid (int, REQUIRED) ：指定了容器命名空间的组id
- umask (int, OPTIONAL)：指定用户的umask值，[在Linux中设置UMASK值](https://www.cnblogs.com/wish123/p/7073114.html)
- additionalGids (array of ints, OPTIONAL) 

Linux中process对象的样例：

```json
"process": {
    "terminal": true,
    "consoleSize": {
        "height": 25,
        "width": 80
    },
    "user": {
        "uid": 1,
        "gid": 1,
        "umask": 63,
        "additionalGids": [5, 6]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "sh"
    ],
    "apparmorProfile": "acme_secure_profile",
    "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
    "ioPriority": {
        "class": "IOPRIO_CLASS_IDLE",
        "priority": 4
    },
    "noNewPrivileges": true,
    "capabilities": {
        "bounding": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
       "permitted": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
       "inheritable": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
        "effective": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL"
        ],
        "ambient": [
            "CAP_NET_BIND_SERVICE"
        ]
    },
    "rlimits": [
        {
            "type": "RLIMIT_NOFILE",
            "hard": 1024,
            "soft": 1024
        }
    ]
}
```

## Hostname

hostname (string, OPTIONAL)：指定了容器中进程所看到的主机名

```json
"hostname": "mrsdalloway"
```

## Domainname

domainname (string, OPTIONAL)：指定了容器中进程所看到的域名

```json
"domainname": "foobarbaz.test"
```

## Platform-specific configuration

这个部分是特定平台的相关配置，篇幅较长，另起一文。

## POSIX-platform Hooks

这部分有些钩子的时间点有点模糊，可以参考[容器的生命周期](https://github.com/opencontainers/runtime-spec/blob/6331715093bfcf25137411bfacb403235ed7d018/runtime.md#lifecycle)。

hooks (object, OPTIONAL) ：配置与容器生命周期相关的特定操作，按顺序进行调用：

- prestart (array of objects, OPTIONAL, DEPRECATED) :所有类型的钩子均有相同的键
  - path (string, REQUIRED) ：绝对路径
  - args (array of strings, OPTIONAL)
  - env (array of strings, OPTIONAL) 
  - timeout (int, OPTIONAL) ：终止钩子的秒数
- createRuntime (array of objects, OPTIONAL)
- createContainer (array of objects, OPTIONAL)
- startContainer (array of objects, OPTIONAL)
- poststart (array of objects, OPTIONAL)
- poststop (array of objects, OPTIONAL) 

容器的状态必须通过 stdin 传递给钩子，以便它们可以根据容器的当前状态执行相应的工作。

### Prestart

Prestart钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**废弃，被后面三个钩子所取代**

### CreateRuntime Hooks

createRuntime钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**在容器命名空间被创建后调用**。

> createRuntime 钩子的定义目前未作明确规定，钩子作者只能期望运行时创建挂载命名空间并执行挂载操作。运行时可能尚未执行其他操作，如 cgroups 和 SELinux/AppArmor 标签

### CreateContainer Hooks

createContainer钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**在执行 pivot_root 操作之前，但在创建和设置挂载命名空间之后调用**。

### StartContainer Hooks

StartContainer钩子作为启动操作的一部分，**必须在执行用户指定的进程之前调用startContainer挂钩**。此钩子可用于在容器中执行某些操作，例如在容器进程生成之前在linux上运行ldconfig二进制文件。

### Poststart

Poststart钩子**必须在用户指定的进程执行后、启动操作返回前调用**。例如，此钩子可以通知用户容器进程已生成。

### Poststop

Poststart钩子**必须在容器删除后、删除操作返回前调用**。清理或调试函数就是此类钩子的例子。

### summary

**namespace是指path以及钩子必须在指定的namespace中解析或调用**。

| Name                    | Namespace | When                                                         |
| ----------------------- | --------- | ------------------------------------------------------------ |
| `prestart` (Deprecated) | runtime   | After the start operation is called but before the user-specified program command is executed. |
| `createRuntime`         | runtime   | During the create operation, after the runtime environment has been created and before the pivot root or any equivalent operation. |
| `createContainer`       | container | During the create operation, after the runtime environment has been created and before the pivot root or any equivalent operation. |
| `startContainer`        | container | After the start operation is called but before the user-specified program command is executed. |
| `poststart`             | runtime   | After the user-specified process is executed but before the start operation returns. |
| `poststop`              | runtime   | After the container is deleted but before the delete operation returns. |

```json
"hooks": {
    "prestart": [
        {
            "path": "/usr/bin/fix-mounts",
            "args": ["fix-mounts", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        },
        {
            "path": "/usr/bin/setup-network"
        }
    ],
    "createRuntime": [
        {
            "path": "/usr/bin/fix-mounts",
            "args": ["fix-mounts", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        },
        {
            "path": "/usr/bin/setup-network"
        }
    ],
    "createContainer": [
        {
            "path": "/usr/bin/mount-hook",
            "args": ["-mount", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        }
    ],
    "startContainer": [
        {
            "path": "/usr/bin/refresh-ldcache"
        }
    ],
    "poststart": [
        {
            "path": "/usr/bin/notify-start",
            "timeout": 5
        }
    ],
    "poststop": [
        {
            "path": "/usr/sbin/cleanup.sh",
            "args": ["cleanup.sh", "-f"]
        }
    ]
}
```

## Annotations

annotations (object, OPTIONAL)：包含容器的任意元数据

```json
"annotations": {
    "com.example.gpu-cores": "2"
}
```

## Configuration Schema Example

```json
{
    "ociVersion": "1.0.1",
    "process": {
        "terminal": true,
        "user": {
            "uid": 1,
            "gid": 1,
            "additionalGids": [
                5,
                6
            ]
        },
        "args": [
            "sh"
        ],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        "capabilities": {
            "bounding": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "permitted": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "inheritable": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "effective": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL"
            ],
            "ambient": [
                "CAP_NET_BIND_SERVICE"
            ]
        },
        "rlimits": [
            {
                "type": "RLIMIT_CORE",
                "hard": 1024,
                "soft": 1024
            },
            {
                "type": "RLIMIT_NOFILE",
                "hard": 1024,
                "soft": 1024
            }
        ],
        "apparmorProfile": "acme_secure_profile",
        "oomScoreAdj": 100,
        "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
        "ioPriority": {
            "class": "IOPRIO_CLASS_IDLE",
            "priority": 4
        },
        "noNewPrivileges": true
    },
    "root": {
        "path": "rootfs",
        "readonly": true
    },
    "hostname": "slartibartfast",
    "mounts": [
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc"
        },
        {
            "destination": "/dev",
            "type": "tmpfs",
            "source": "tmpfs",
            "options": [
                "nosuid",
                "strictatime",
                "mode=755",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/pts",
            "type": "devpts",
            "source": "devpts",
            "options": [
                "nosuid",
                "noexec",
                "newinstance",
                "ptmxmode=0666",
                "mode=0620",
                "gid=5"
            ]
        },
        {
            "destination": "/dev/shm",
            "type": "tmpfs",
            "source": "shm",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "mode=1777",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/mqueue",
            "type": "mqueue",
            "source": "mqueue",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys",
            "type": "sysfs",
            "source": "sysfs",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys/fs/cgroup",
            "type": "cgroup",
            "source": "cgroup",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "relatime",
                "ro"
            ]
        }
    ],
    "hooks": {
        "prestart": [
            {
                "path": "/usr/bin/fix-mounts",
                "args": [
                    "fix-mounts",
                    "arg1",
                    "arg2"
                ],
                "env": [
                    "key1=value1"
                ]
            },
            {
                "path": "/usr/bin/setup-network"
            }
        ],
        "poststart": [
            {
                "path": "/usr/bin/notify-start",
                "timeout": 5
            }
        ],
        "poststop": [
            {
                "path": "/usr/sbin/cleanup.sh",
                "args": [
                    "cleanup.sh",
                    "-f"
                ]
            }
        ]
    },
    "linux": {
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
        ],
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
        ],
        "sysctl": {
            "net.ipv4.ip_forward": "1",
            "net.core.somaxconn": "256"
        },
        "cgroupsPath": "/myRuntime/myContainer",
        "resources": {
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
            },
            "pids": {
                "limit": 32771
            },
            "hugepageLimits": [
                {
                    "pageSize": "2MB",
                    "limit": 9223372036854772000
                },
                {
                    "pageSize": "64KB",
                    "limit": 1000000
                }
            ],
            "memory": {
                "limit": 536870912,
                "reservation": 536870912,
                "swap": 536870912,
                "kernel": -1,
                "kernelTCP": -1,
                "swappiness": 0,
                "disableOOMKiller": false
            },
            "cpu": {
                "shares": 1024,
                "quota": 1000000,
                "period": 500000,
                "realtimeRuntime": 950000,
                "realtimePeriod": 1000000,
                "cpus": "2-3",
                "idle": 1,
                "mems": "0-7"
            },
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
            ],
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
        },
        "rootfsPropagation": "slave",
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
        },
        "timeOffsets": {
            "monotonic": {
                "secs": 172800,
                "nanosecs": 0
            },
            "boottime": {
                "secs": 604800,
                "nanosecs": 0
            }
        },
        "namespaces": [
            {
                "type": "pid"
            },
            {
                "type": "network"
            },
            {
                "type": "ipc"
            },
            {
                "type": "uts"
            },
            {
                "type": "mount"
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
        ],
        "maskedPaths": [
            "/proc/kcore",
            "/proc/latency_stats",
            "/proc/timer_stats",
            "/proc/sched_debug"
        ],
        "readonlyPaths": [
            "/proc/asound",
            "/proc/bus",
            "/proc/fs",
            "/proc/irq",
            "/proc/sys",
            "/proc/sysrq-trigger"
        ],
        "mountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c715,c811"
    },
    "annotations": {
        "com.example.key1": "value1",
        "com.example.key2": "value2"
    }
}
```



