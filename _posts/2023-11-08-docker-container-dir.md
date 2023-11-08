---
layout: post
title: Docker容器的相关目录
subtitle: No Subtitle ：>
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
**介绍docker容器的一些相关目录**。

联合文件挂载系统为overlay2。

## /var/lib/docker/containers

此目录为docker容器的元数据存储目录：

- 目录中的每个文件夹都代表一个容器
- 目录中的文件夹命名为容器id（完整的sha256值）
- 目录中文件夹的数量等于当前节点上容器的数量
- 容器被删除后，目录中对应的文件夹也会被删除

```shell
root@debian:/var/lib/docker/containers# ls
2feb0d4f6fb72373ed806d84ae9ec25bfe6c7efa2e31598024bf3d5f8d3b8b72  6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc
root@debian:/var/lib/docker/containers# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED      STATUS                  PORTS     NAMES
6e9d18b4213d   ubuntu    "/bin/bash"   2 days ago   Up 17 hours                       youthful_ganguly
2feb0d4f6fb7   busybox   "sh"          2 days ago   Exited (0) 2 days ago             hardcore_hertz
root@debian:/var/lib/docker/containers# docker rm 2feb0d4f6fb7
2feb0d4f6fb7
root@debian:/var/lib/docker/containers# ls
6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc
```

以上面的youthful_ganguly容器为例（这个名字是用户未指定容器名字时，docker随机为容器生成的字符串，镜像为ubuntu），其目录结构如下：

```shell
root@debian:/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# tree .
.
├── 6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc-json.log
├── checkpoints
├── config.v2.json
├── hostconfig.json
├── hostname
├── hosts
├── mounts
├── resolv.conf
└── resolv.conf.hash
```

### \<container-id>.json

此日志文件记录了用户在容器中的所有操作，选取其中的两行日志记录如下：

```shell
{"log":"\u001b[?2004l\r\u001b[?2004h\u001b]0;root@6e9d18b4213d: /home\u0007root@6e9d18b4213d:/home# l\u0008 \u0008\u0007\u0007ex\u0008 \u0008ch\u0008 \u0008\u0008 \u0008\u0008 \u0008t\u0008 \u0008\u0007cat test.txt \r\n","stream":"stdout","time":"2023-11-06T02:49:19.945733653Z"}
{"log":"\u001b[?2004l\r1\r\n","stream":"stdout","time":"2023-11-06T02:49:19.946891978Z"}
```

来自GPT-4的解释：这段日志来自Docker容器的日志文件，展示了在容器内部执行的命令的输出：

- 第一行包含了一些控制码（比如`\u001b[?2004l`），这些控制码用于控制终端的行为（比如打开或关闭特殊的输入/输出模式）。接着是一个提示符（`root@6e9d18b4213d:/home#`），表明root用户正在容器中的`/home`目录下操作。提示符后的字符似乎是用户尝试输入命令的一些错误操作，如`“l\u0008 \u0008”和“ex\u0008 \u0008”`，这表示用户输入了字符后又退格删除了。

- 第二行显示执行了`cat test.txt`命令，显示`test.txt`文件的内容，内容为“1”。

### checkpoints

checkpoints目录用于存储容器的检查点。检查点是容器当前状态的快照，可以用来在之后的某个时间点恢复容器的状态。这是一个高级功能，可以用于迁移、备份或者容灾恢复。

此容器下的checkpoints目录为空，意味着当前没有为该容器创建任何检查点。

### config.v2.json

config.v2.json用于记录容器的元数据，包括容器的进程号、启动镜像、状态和相关配置等。

**其中容器的pid在每次启动容器时可能变化**。

```json
 {
  "StreamConfig": {},
  "State": {
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "RemovalInProgress": false,
    "Dead": false,
    "Pid": 1451101,
    "ExitCode": 0,
    "Error": "",
    "StartedAt": "2023-11-06T01:47:55.990669624Z",
    "FinishedAt": "0001-01-01T00:00:00Z",
    "Health": null
  },
  "ID": "6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc",
  "Created": "2023-11-06T01:47:54.019241019Z",
  "Managed": false,
  "Path": "/bin/bash",
  "Args": [],
  "Config": {
    "Hostname": "6e9d18b4213d",
    "Domainname": "",
    "User": "",
    "AttachStdin": true,
    "AttachStdout": true,
    "AttachStderr": true,
    "Tty": true,
    "OpenStdin": true,
    "StdinOnce": true,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/bash"
    ],
    "Image": "ubuntu",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": null,
    "OnBuild": null,
    "Labels": {
      "org.opencontainers.image.ref.name": "ubuntu",
      "org.opencontainers.image.version": "22.04"
    }
  },
  "Image": "sha256:c6b84b685f35f1a5d63661f5d4aa662ad9b7ee4f4b8c394c022f25023c907b65",
  "NetworkSettings": {
    "Bridge": "",
    "SandboxID": "cbed746c95fda1eadcd2acac9fc3e3ee84a246424d535cd4ea1f3e6ed3bd680e",
    "HairpinMode": false,
    "LinkLocalIPv6Address": "",
    "LinkLocalIPv6PrefixLen": 0,
    "Networks": {
      "bridge": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": null,
        "NetworkID": "8f8740bb3d56298247c1315c6b436f8557f498ee35da425540a3536d96a82e27",
        "EndpointID": "f9af19afc02c201593db7ccbfd0fa94c417d827aba5465ead18b5f3a3f784e45",
        "Gateway": "172.17.0.1",
        "IPAddress": "172.17.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:11:00:02",
        "DriverOpts": null,
        "IPAMOperational": false
      }
    },
    "Service": null,
    "Ports": {},
    "SandboxKey": "/var/run/docker/netns/cbed746c95fd",
    "SecondaryIPAddresses": null,
    "SecondaryIPv6Addresses": null,
    "IsAnonymousEndpoint": true,
    "HasSwarmEndpoint": false
  },
  "LogPath": "/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc-json.log",
  "Name": "/youthful_ganguly",
  "Driver": "overlay2",
  "OS": "linux",
  "MountLabel": "",
  "ProcessLabel": "",
  "RestartCount": 0,
  "HasBeenStartedBefore": true,
  "HasBeenManuallyStopped": false,
  "MountPoints": {},
  "SecretReferences": null,
  "ConfigReferences": null,
  "AppArmorProfile": "docker-default",
  "HostnamePath": "/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/hostname",
  "HostsPath": "/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/hosts",
  "ShmPath": "",
  "ResolvConfPath": "/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/resolv.conf",
  "SeccompProfile": "",
  "NoNewPrivileges": false,
  "LocalLogCacheMeta": {
    "HaveNotifyEnabled": false
  }
}
```

### hostconfig.json

hostconfig.json存储了容器和宿主机相关的配置，包括一些资源限制和安全选项等内容。

```shell
 
{
  "Binds": null,
  "ContainerIDFile": "",
  "LogConfig": {
    "Type": "json-file",
    "Config": {}
  },
  "NetworkMode": "default",
  "PortBindings": {},
  "RestartPolicy": {
    "Name": "no",
    "MaximumRetryCount": 0
  },
  "AutoRemove": false,
  "VolumeDriver": "",
  "VolumesFrom": null,
  "CapAdd": null,
  "CapDrop": null,
  "CgroupnsMode": "private",
  "Dns": [],
  "DnsOptions": [],
  "DnsSearch": [],
  "ExtraHosts": null,
  "GroupAdd": null,
  "IpcMode": "private",
  "Cgroup": "",
  "Links": null,
  "OomScoreAdj": 0,
  "PidMode": "",
  "Privileged": false,
  "PublishAllPorts": false,
  "ReadonlyRootfs": false,
  "SecurityOpt": null,
  "UTSMode": "",
  "UsernsMode": "",
  "ShmSize": 67108864,
  "Runtime": "runc",
  "ConsoleSize": [
    0,
    0
  ],
  "Isolation": "",
  "CpuShares": 0,
  "Memory": 0,
  "NanoCpus": 0,
  "CgroupParent": "",
  "BlkioWeight": 0,
  "BlkioWeightDevice": [],
  "BlkioDeviceReadBps": null,
  "BlkioDeviceWriteBps": null,
  "BlkioDeviceReadIOps": null,
  "BlkioDeviceWriteIOps": null,
  "CpuPeriod": 0,
  "CpuQuota": 0,
  "CpuRealtimePeriod": 0,
  "CpuRealtimeRuntime": 0,
  "CpusetCpus": "",
  "CpusetMems": "",
  "Devices": [],
  "DeviceCgroupRules": null,
  "DeviceRequests": null,
  "KernelMemory": 0,
  "KernelMemoryTCP": 0,
  "MemoryReservation": 0,
  "MemorySwap": 0,
  "MemorySwappiness": null,
  "OomKillDisable": null,
  "PidsLimit": null,
  "Ulimits": null,
  "CpuCount": 0,
  "CpuPercent": 0,
  "IOMaximumIOps": 0,
  "IOMaximumBandwidth": 0,
  "MaskedPaths": [
    "/proc/asound",
    "/proc/acpi",
    "/proc/kcore",
    "/proc/keys",
    "/proc/latency_stats",
    "/proc/timer_list",
    "/proc/timer_stats",
    "/proc/sched_debug",
    "/proc/scsi",
    "/sys/firmware"
  ],
  "ReadonlyPaths": [
    "/proc/bus",
    "/proc/fs",
    "/proc/irq",
    "/proc/sys",
    "/proc/sysrq-trigger"
  ]
}
```

### hostname

存储容器中显示的hostname，与config.v2.json中的Config.Hostname一致。

```shell
6e9d18b4213d
```

### hosts

与Unix-like系统中的/etc/hosts类似，此文件用于控制容器内部的主机名解析。

```shell
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	6e9d18b4213d

# 容器中的hosts
root@6e9d18b4213d:/home# cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      6e9d18b4213d
```

### mounts

mounts目录通常用于存储与容器相关的挂载信息，当在容器中挂载卷（volume）或者绑定挂载（bind mount）时，相关信息将会在这个目录下记录。

当前容器的mounts目录是空的，也就是没有卷或绑定挂载被配置到容器中。

### resolv.conf

与Unix-like系统中的/etc/resolv.conf类似，resolv.conf用于设置容器的DNS解析器信息，告诉容器应该使用哪些DNS服务器来解析域名。

当启动一个Docker容器时，Docker会基于宿主机的DNS配置自动生成这个文件，或者可以通过Docker运行时的参数来指定使用特定的DNS服务器。此文件保证了容器内部能够解析外部服务的主机名，这对于容器访问互联网或者内部网络中的其他服务是必需的。如果这个文件没有正确配置，容器内的应用程序可能无法解析域名，从而导致网络连接问题。

```shell
# Generated by NetworkManager
nameserver 159.226.8.7
nameserver 159.226.8.6
nameserver 8.8.8.8
# NOTE: the libc resolver may not support more than 3 nameservers.
# The nameservers listed below may not be recognized.
nameserver 114.114.114.114

# 容器中的resolv.conf
root@6e9d18b4213d:/home# cat /etc/resolv.conf 
# Generated by NetworkManager
nameserver 159.226.8.7
nameserver 159.226.8.6
nameserver 8.8.8.8
# NOTE: the libc resolver may not support more than 3 nameservers.
# The nameservers listed below may not be recognized.
nameserver 114.114.114.114
```

resolv.conf.hash存储resolv.conf的sha256值。

```shell
sha256:f9d194d75b0240c75b6e6119dafb7db4f7de5871e4bb844fa13ce67aec926228
```

## bundle

docker容器的运行目录（不太确定该不该这样称呼）一般为`/run/containerd/io.containerd.runtime.v2.task/moby`：

- 目录中每个文件夹表示容器的bundle目录
- 目录中的文件夹命名为容器id
- 目录中的文件夹数量与当前节点上运行的容器数量一致，也就是说当容器停止或退出时，目录消失，启动时重新生成

```shell
root@debian:/run/containerd/io.containerd.runtime.v2.task/moby# ls
6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc
root@debian:/run/containerd/io.containerd.runtime.v2.task/moby# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED      STATUS        PORTS     NAMES
6e9d18b4213d   ubuntu    "/bin/bash"   2 days ago   Up 18 hours             youthful_ganguly
```

同样以上述容器为例，bundle目录的结构如下：

```shell
root@debian:/run/containerd/io.containerd.runtime.v2.task/moby/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# tree .
.
├── address
├── config.json
├── init.pid
├── log
├── log.json
├── options.json
├── rootfs
├── runtime
├── shim-binary-path
└── work -> /var/lib/containerd/io.containerd.runtime.v2.task/moby/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc
```

### address

address包含一个UNIX套接字地址，用于与容器的运行时通信。

```shell
unix:///run/containerd/s/956765f05bdd88e07bf23c8fe04b66ba59ef3e4958c514b5534b03568e4ebfb2
```

### config.json

符合OCI runtime spec的容器配置，很长，这里贴出没有json格式化的内容：

```shell
{"ociVersion":"1.0.2-dev","process":{"terminal":true,"user":{"uid":0,"gid":0},"args":["/bin/bash"],"env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","HOSTNAME=6e9d18b4213d","TERM=xterm"],"cwd":"/","capabilities":{"bounding":["CAP_CHOWN","CAP_DAC_OVERRIDE","CAP_FSETID","CAP_FOWNER","CAP_MKNOD","CAP_NET_RAW","CAP_SETGID","CAP_SETUID","CAP_SETFCAP","CAP_SETPCAP","CAP_NET_BIND_SERVICE","CAP_SYS_CHROOT","CAP_KILL","CAP_AUDIT_WRITE"],"effective":["CAP_CHOWN","CAP_DAC_OVERRIDE","CAP_FSETID","CAP_FOWNER","CAP_MKNOD","CAP_NET_RAW","CAP_SETGID","CAP_SETUID","CAP_SETFCAP","CAP_SETPCAP","CAP_NET_BIND_SERVICE","CAP_SYS_CHROOT","CAP_KILL","CAP_AUDIT_WRITE"],"permitted":["CAP_CHOWN","CAP_DAC_OVERRIDE","CAP_FSETID","CAP_FOWNER","CAP_MKNOD","CAP_NET_RAW","CAP_SETGID","CAP_SETUID","CAP_SETFCAP","CAP_SETPCAP","CAP_NET_BIND_SERVICE","CAP_SYS_CHROOT","CAP_KILL","CAP_AUDIT_WRITE"]},"apparmorProfile":"docker-default","oomScoreAdj":0},"root":{"path":"/var/lib/docker/overlay2/983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045/merged"},"hostname":"6e9d18b4213d","mounts":[{"destination":"/proc","type":"proc","source":"proc","options":["nosuid","noexec","nodev"]},{"destination":"/dev","type":"tmpfs","source":"tmpfs","options":["nosuid","strictatime","mode=755","size=65536k"]},{"destination":"/dev/pts","type":"devpts","source":"devpts","options":["nosuid","noexec","newinstance","ptmxmode=0666","mode=0620","gid=5"]},{"destination":"/sys","type":"sysfs","source":"sysfs","options":["nosuid","noexec","nodev","ro"]},{"destination":"/sys/fs/cgroup","type":"cgroup","source":"cgroup","options":["ro","nosuid","noexec","nodev"]},{"destination":"/dev/mqueue","type":"mqueue","source":"mqueue","options":["nosuid","noexec","nodev"]},{"destination":"/dev/shm","type":"tmpfs","source":"shm","options":["nosuid","noexec","nodev","mode=1777","size=67108864"]},{"destination":"/etc/resolv.conf","type":"bind","source":"/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/resolv.conf","options":["rbind","rprivate"]},{"destination":"/etc/hostname","type":"bind","source":"/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/hostname","options":["rbind","rprivate"]},{"destination":"/etc/hosts","type":"bind","source":"/var/lib/docker/containers/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc/hosts","options":["rbind","rprivate"]}],"hooks":{"prestart":[{"path":"/proc/2000/exe","args":["libnetwork-setkey","-exec-root=/var/run/docker","6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc","2d4ae3d11935"]}]},"linux":{"sysctl":{"net.ipv4.ip_unprivileged_port_start":"0","net.ipv4.ping_group_range":"0 2147483647"},"resources":{"devices":[{"allow":false,"access":"rwm"},{"allow":true,"type":"c","major":1,"minor":5,"access":"rwm"},{"allow":true,"type":"c","major":1,"minor":3,"access":"rwm"},{"allow":true,"type":"c","major":1,"minor":9,"access":"rwm"},{"allow":true,"type":"c","major":1,"minor":8,"access":"rwm"},{"allow":true,"type":"c","major":5,"minor":0,"access":"rwm"},{"allow":true,"type":"c","major":5,"minor":1,"access":"rwm"},{"allow":false,"type":"c","major":10,"minor":229,"access":"rwm"}],"memory":{},"cpu":{"shares":0},"blockIO":{"weight":0}},"cgroupsPath":"system.slice:docker:6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc","namespaces":[{"type":"mount"},{"type":"network"},{"type":"uts"},{"type":"pid"},{"type":"ipc"},{"type":"cgroup"}],"seccomp":{"defaultAction":"SCMP_ACT_ERRNO","architectures":["SCMP_ARCH_X86_64","SCMP_ARCH_X86","SCMP_ARCH_X32"],"syscalls":[{"names":["accept","accept4","access","adjtimex","alarm","bind","brk","capget","capset","chdir","chmod","chown","chown32","clock_adjtime","clock_adjtime64","clock_getres","clock_getres_time64","clock_gettime","clock_gettime64","clock_nanosleep","clock_nanosleep_time64","close","close_range","connect","copy_file_range","creat","dup","dup2","dup3","epoll_create","epoll_create1","epoll_ctl","epoll_ctl_old","epoll_pwait","epoll_pwait2","epoll_wait","epoll_wait_old","eventfd","eventfd2","execve","execveat","exit","exit_group","faccessat","faccessat2","fadvise64","fadvise64_64","fallocate","fanotify_mark","fchdir","fchmod","fchmodat","fchown","fchown32","fchownat","fcntl","fcntl64","fdatasync","fgetxattr","flistxattr","flock","fork","fremovexattr","fsetxattr","fstat","fstat64","fstatat64","fstatfs","fstatfs64","fsync","ftruncate","ftruncate64","futex","futex_time64","futimesat","getcpu","getcwd","getdents","getdents64","getegid","getegid32","geteuid","geteuid32","getgid","getgid32","getgroups","getgroups32","getitimer","getpeername","getpgid","getpgrp","getpid","getppid","getpriority","getrandom","getresgid","getresgid32","getresuid","getresuid32","getrlimit","get_robust_list","getrusage","getsid","getsockname","getsockopt","get_thread_area","gettid","gettimeofday","getuid","getuid32","getxattr","inotify_add_watch","inotify_init","inotify_init1","inotify_rm_watch","io_cancel","ioctl","io_destroy","io_getevents","io_pgetevents","io_pgetevents_time64","ioprio_get","ioprio_set","io_setup","io_submit","io_uring_enter","io_uring_register","io_uring_setup","ipc","kill","lchown","lchown32","lgetxattr","link","linkat","listen","listxattr","llistxattr","_llseek","lremovexattr","lseek","lsetxattr","lstat","lstat64","madvise","membarrier","memfd_create","mincore","mkdir","mkdirat","mknod","mknodat","mlock","mlock2","mlockall","mmap","mmap2","mprotect","mq_getsetattr","mq_notify","mq_open","mq_timedreceive","mq_timedreceive_time64","mq_timedsend","mq_timedsend_time64","mq_unlink","mremap","msgctl","msgget","msgrcv","msgsnd","msync","munlock","munlockall","munmap","nanosleep","newfstatat","_newselect","open","openat","openat2","pause","pidfd_open","pidfd_send_signal","pipe","pipe2","poll","ppoll","ppoll_time64","prctl","pread64","preadv","preadv2","prlimit64","pselect6","pselect6_time64","pwrite64","pwritev","pwritev2","read","readahead","readlink","readlinkat","readv","recv","recvfrom","recvmmsg","recvmmsg_time64","recvmsg","remap_file_pages","removexattr","rename","renameat","renameat2","restart_syscall","rmdir","rseq","rt_sigaction","rt_sigpending","rt_sigprocmask","rt_sigqueueinfo","rt_sigreturn","rt_sigsuspend","rt_sigtimedwait","rt_sigtimedwait_time64","rt_tgsigqueueinfo","sched_getaffinity","sched_getattr","sched_getparam","sched_get_priority_max","sched_get_priority_min","sched_getscheduler","sched_rr_get_interval","sched_rr_get_interval_time64","sched_setaffinity","sched_setattr","sched_setparam","sched_setscheduler","sched_yield","seccomp","select","semctl","semget","semop","semtimedop","semtimedop_time64","send","sendfile","sendfile64","sendmmsg","sendmsg","sendto","setfsgid","setfsgid32","setfsuid","setfsuid32","setgid","setgid32","setgroups","setgroups32","setitimer","setpgid","setpriority","setregid","setregid32","setresgid","setresgid32","setresuid","setresuid32","setreuid","setreuid32","setrlimit","set_robust_list","setsid","setsockopt","set_thread_area","set_tid_address","setuid","setuid32","setxattr","shmat","shmctl","shmdt","shmget","shutdown","sigaltstack","signalfd","signalfd4","sigprocmask","sigreturn","socket","socketcall","socketpair","splice","stat","stat64","statfs","statfs64","statx","symlink","symlinkat","sync","sync_file_range","syncfs","sysinfo","tee","tgkill","time","timer_create","timer_delete","timer_getoverrun","timer_gettime","timer_gettime64","timer_settime","timer_settime64","timerfd_create","timerfd_gettime","timerfd_gettime64","timerfd_settime","timerfd_settime64","times","tkill","truncate","truncate64","ugetrlimit","umask","uname","unlink","unlinkat","utime","utimensat","utimensat_time64","utimes","vfork","vmsplice","wait4","waitid","waitpid","write","writev"],"action":"SCMP_ACT_ALLOW"},{"names":["ptrace"],"action":"SCMP_ACT_ALLOW"},{"names":["personality"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":0,"op":"SCMP_CMP_EQ"}]},{"names":["personality"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":8,"op":"SCMP_CMP_EQ"}]},{"names":["personality"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":131072,"op":"SCMP_CMP_EQ"}]},{"names":["personality"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":131080,"op":"SCMP_CMP_EQ"}]},{"names":["personality"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":4294967295,"op":"SCMP_CMP_EQ"}]},{"names":["arch_prctl"],"action":"SCMP_ACT_ALLOW"},{"names":["modify_ldt"],"action":"SCMP_ACT_ALLOW"},{"names":["clone"],"action":"SCMP_ACT_ALLOW","args":[{"index":0,"value":2114060288,"op":"SCMP_CMP_MASKED_EQ"}]},{"names":["clone3"],"action":"SCMP_ACT_ERRNO","errnoRet":38},{"names":["chroot"],"action":"SCMP_ACT_ALLOW"}]},"maskedPaths":["/proc/asound","/proc/acpi","/proc/kcore","/proc/keys","/proc/latency_stats","/proc/timer_list","/proc/timer_stats","/proc/sched_debug","/proc/scsi","/sys/firmware"],"readonlyPaths":["/proc/bus","/proc/fs","/proc/irq","/proc/sys","/proc/sysrq-trigger"]}}
```

值得一提的是，docker默认为容器添加了一个prestart钩子：

```json
"hooks": {
    "prestart": [
      {
        "path": "/proc/2000/exe",
        "args": [
          "libnetwork-setkey",
          "-exec-root=/var/run/docker",
          "6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc",
          "2d4ae3d11935"
        ]
      }
    ]
  },
```

/proc/2000表示进程号为2000的进程，查看为dockerd（docker daemon），而/proc/2000/exe是一个符号链接，指向/usr/bin/dockerd。这个钩子的作用是是利用Docker守护进程来初始化或配置容器的网络设置，确保容器启动之前网络已正确配置。

```shell
root@debian:/proc/2000# ls -l exe 
lrwxrwxrwx 1 root root 0 Nov  8 14:58 exe -> /usr/bin/dockerd
root@debian:/proc/2000# ps -p 2000
    PID TTY          TIME CMD
   2000 ?        00:30:44 dockerd
```

### init.pid

记录当前容器进程的pid，同样的每次启动pid都可能变化。

容器就是一个特殊的进程，当我们用`kill -9 <container-pid>`命令时也会将容器直接退出。

### log

log是一个命名管道（FIFO），用于收集与转发容器的日志输出。

log.json是一个空文件，可能是为了满足规范的必须品（存疑）？前面已知容器中的日志输出会被存储在/var/lib/docker/containers/\<container-id>.json中。

### options.json

options.json包含了容器启动时的一些配置，主要是runtime的cgroup的相关信息。

```shell
{
  "binary_name": "runc",
  "root": "/var/run/docker/runtime-runc",
  "systemd_cgroup": true
}
```

### rootfs

rootfs空文件，OCI runtime spec中创建一个容器的必须品，容器实际的rootfs在同目录下的config.json中指明。

### runtime

标明容器的运行时，为runc。

```shell
runc
```

### shim-binary-path

docker（实际为Containerd）使用shim进程来管理容器的生命周期，这个文件包含了shim二进制文件的路径。

```shell
/usr/bin/containerd-shim-runc-v2
```

### work

work是一个指向`/var/lib/containerd/io.containerd.runtime.v2.task/moby/\<container-id>`的符号链接，此目录为空。

## rootfs

容器实际运行的rootfs在bundle文件中config.json中的root字段：

```shell
"root": {
    "path": "/var/lib/docker/overlay2/983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045/merged"
  }
```

一般为`/var/lib/docker/overlay2/\<mount-id>/merged`，其中`\<mount-id>`的值记录在`/var/lib/docker/image/overlay2/layerdb/mounts/\<container-id>`中的`mount-id`文件。

```shell
root@debian:/var/lib/docker/overlay2/983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045# ls
diff  link  lower  merged  work
root@debian:/var/lib/docker/overlay2/983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045# cd merged/
root@debian:/var/lib/docker/overlay2/983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045/merged# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

容器中的所有对文件的改动都会记录存储在这个目录，同样的直接在这个目录添加或者修改文件，也会同步在容器中。

`/var/lib/docker/overlay2/\<mount-id>/merged`目录只会在容器运行时存在（挂载）。
