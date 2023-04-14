---
layout: post
title: "【OS Security Lab】基于PAM的用户权能分配"
subtitle: "[OS Security Lab] PAM-based user capability distribution"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - os security
---

## 1.指出每个权能对应的系统调用，简要解释功能

Capabilities 机制是在 Linux 内核 2.2 之后引入的，原理就是将之前与超级用户 root（UID=0）关联的特权细分为不同的功能组，每个功能组都可以独立启用和禁用。其本质上就是将内核调用分门别类，具有相似功能的内核调用被分到同一组中。这样一来，权限检查的过程就变成了：在执行特权操作时，如果线程的有效身份不是 root，就去检查其是否具有该特权操作所对应的 capabilities，并以此为依据，决定是否可以执行特权操作。Capabilities 可以在进程执行时赋予，也可以直接从父进程继承。

在Capilities中，只有进程和可执行文件才具有能力，每个进程拥有三组能力集

- cap_permitted表示进程所拥有的最大能力集
- cap_effective表示进程当前可用的能力集，可看做是cap_permitted的一个子集
- cap_inheitable则表示进程可以传递给其子进程的能力集

可执行文件也拥有三组能力集

- cap_allowed表示程序运行时可从原进程的cap_inheritable中集成的能力集
- cap_forced表示运行文件时必须拥有才能完成其服务的能力集
- cap_effective则表示文件开始运行时可以使用的能力。

Capabilities各项权能、系统调用号已经相关解释可在`capability.h`文件中找到。

```shell
sudo find / -name capability.h # 查看系统中capability.h
man capabilities # 查看capability manual
```

| 权能                   | 系统调用号      | 解释                                                        |
| ---------------------- | --------------- | ----------------------------------------------------------- |
| CAP_CHOWN              | 0(chown)        | 对文件的UIDs和GIDs做任意修改                                |
| CAP_DAC_OVERRIDE       | 1               | 忽略对文件的DAC访问限制                                     |
| CAP_DAC_READ_SEARCH    | 2               | 忽略DAC中对文件和目录的读、搜索权限                         |
| CAP_FOWNER             | 3               | 忽略进程UID与文件UID的匹配检查                              |
| CAP_FSETID             | 4               | 文件修改时不清除setuid和setgid位，不匹配时设置setgid位      |
| CAP_KILL               | 5(kill)         | 绕过发送信号时的权限检查                                    |
| CAP_SETGID             | 6(setgid)       | 设置或管理进程GID                                           |
| CAP_SETUID             | 7(setuid)       | 管理或设置进程UID                                           |
| CAP_SETPCAP            | 8(capset)       | 允许授予或删除其他进程的任何权能                            |
| CAP_LINUX_IMMUTABLE    | 9(chattr)       | 允许设置文件的不可修改位(IMMUTABLE)和只添加(APPND-ONLY)属性 |
| CAP_NET_BIND_SERVICE   | 10              | 允许绑定到小于1024的端口                                    |
| CAP_NET_BROADCAST      | 11              | 允许socket发送监听组播                                      |
| CAP_NET_ADMIN          | 12              | 允许执行网络管理任务                                        |
| CAP_NET_RAW            | 13(socket)      | 允许使用原始套接字                                          |
| CAP_IPC_LOCK           | 14(mlock)       | 允许锁定共享内存片段                                        |
| CAP_IPC_OWNER          | 15              | 忽略IPC所有权检查                                           |
| CAP_SYS_MOUDLE         | 16(init_module) | 插入和删除内核模块                                          |
| CAP_SYS_RAWIO          | 17              | 允许对ioperm/iopl的访问                                     |
| CAP_SYS_CHROOT         | 18(chroot)      | 允许使用chroot()系统调用                                    |
| CAP_SYS_PTRACE         | 19(ptrace)      | 允许跟踪任何进程                                            |
| CAP_SYS_PACCT          | 20(acct)        | 允许配置进程记账                                            |
| CAP_SYS_ADMIN          | 21              | 允许执行系统管理任务                                        |
| CAP_SYS_BOOT           | 22(reboot)      | 允许重新启动系统                                            |
| CAP_SYS_NICE           | 23(nice)        | 允许提升优先级，设置其他进程优先级                          |
| CAP_SYS_RESOURCE       | 24(setrlimit)   | 设置资源限制                                                |
| CAP_SYS_TIME           | 25(stime)       | 允许改变系统时钟                                            |
| CAP_SYS_TTY_CONFIG     | 26(vhangup)     | 允许配置TTY设备                                             |
| CAP_MKNOD              | 27(mknod)       | 允许使用mknod()系统调用，创建特殊文件                       |
| CAP_LEASE              | 28(fcntl)       | 为任意文件建立租约                                          |
| CAP_AUDIT_WRITE        | 29              | 允许向内核审计日志写记录                                    |
| CAP_AUDIT_CONTROL      | 30              | 启用或禁用内核审计，修改审计过滤器规则                      |
| CAP_SETFCAP            | 31              | 设置文件权能                                                |
| CAP_MAC_OVERRIDE       | 32              | 允许MAC配置或状态改变，为smack LSM实现                      |
| CAP_MAC_ADMIN          | 33              | 覆盖强制访问控制                                            |
| CAP_SYSLOG             | 34(syslog)      | 执行特权syslog(2)操作                                       |
| CAP_WAKE_ALARM         | 35              | 触发将唤醒系统的东西                                        |
| CAP_BLOCK_SUSPEND      | 36(epoll)       | 可以阻塞系统挂起的特性                                      |
| CAP_AUDIT_READ         | 37              | 允许通过多播网络链接套接字读取审计日志                      |
| CAP_PERFMON            | 38              | 进行性能监测                                                |
| CAP_BPF                | 39              | 采用有特权的BPF操作                                         |
| CAP_CHECKPOINT_RESTORE | 40              | 检查点和恢复功能                                            |

## 2.基于PAM用户权限设置系统

### 2.1 PAM

PAM是”Pluggable Authentication Modules”的缩写，它是一种系统级身份验证框架，用于在UNIX和Linux系统上实现身份验证服务。PAM通过使用动态共享库来允许不同的身份验证方法和技术，如传统的口令、双因素身份验证和基于证书的身份验证等。这个模块化设计的架构可以使管理员灵活地控制和配置系统中的身份验证流程。

使用PAM，管理员可以选择和组合多种身份验证模块来建立一个强大而又灵活的身份验证系统。它的架构可以让管理员自由地选择适合他们需要的认证方法。例如，可以使用Kerberos认证来对网络用户进行身份验证，而使用普通的口令认证来对本地用户进行身份验证。

PAM框架可以分为四个基本阶段：

1. 认证（authentication）：在这个阶段，PAM根据用户名和密码等凭证来验证用户的身份。
2. 授权（authorization）：在这个阶段，PAM确定用户是否有权限访问系统或资源。
3. 账号管理（account management）：在这个阶段，PAM验证用户的账户信息，如是否过期或禁用。
4. 会话管理（session management）：在这个阶段，PAM管理用户会话的开始和结束，并配置环境变量和其他会话特定的属性。

### 2.2 Setuid、Setgid和Sticky bit

在执行时, 一般可执行文件只拥有调用该文件的用户具有的权限，而setuid，setgid 可以来改变这种设置。

- setuid：设置使文件在执行阶段具有文件所有者的权限，典型的文件是 /usr/bin/passwd，如果一般用户执行该文件，则在执行过程中， 该文件可以获得root权限，从而可以更改用户的密码。
- setgid：该权限只对目录有效，目录被设置该位后，任何用户在此目录下创建的文件都具有和该目录所属的组相同的组。
- sticky bit： 该位可以理解为防删除位， 一个文件是否可以被某用户删除，主要取决于该文件所属的组是否对该用户具有写权限。 如果没有写权限，则这个目录下的所有文件都不能被删除， 同时也不能添加新的文件。如果希望用户能够添加文件，但同时不能删除文件，则可以对文件设置sticky bit位。设置该位后，就算用户对目录具有写权限，也不能删除该文件。

### 2.3 实验过程中踩得坑以及ping命令的变化

实验过程中参考了之前的一些实验博客：

- [基于PAM的用户权能分配—Tremb1e's Blog](https://www.tremb1e.com/archives/user-capability-assignment-based-on-pam)
- [基于PAM的用户权能分配—破落之实](https://blog.csdn.net/u013648063/article/details/106944141)
- [基于PAM的用户权能分配—在路上](https://zhuanlan.zhihu.com/p/146366045)

上述实验步骤和方法都比较固定和类似：

1. 通过`setcap -r` 命令删除掉ping、passwd等命令的capability权能
2. 通过`chmod u-s `取消setuid位，从而使得普通用户无法执行上述命令
3. 再通过`setcap`命令为用户添加相应capability权能使得可以不依赖setuid位即可执行上述命令
4. 最后通过相关脚本，以及`session  optional  pam_exec.so debug log=/tmp/pam_exec.log seteuid`命令实现用户登录时的权限设置

**实验过程中遇到的坑**

虽然上述步骤很详细了，但在实际过程中依然遇到了问题。最主要的就是移除setuid位后，并且没有赋予权能的情况下，普通用户也能使用ping命令，一开始以为是系统版本的原因，测试了20.04和18.04两个版本均有此问题（再低的版本例如12.04应该不会出现这个问题，当然也没有去验证，由于版本太久做出来意义也不大，因此实验依旧采用最近的版本）。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162203483.png">

为了解决这个问题，首先我们需要理解ping命令的原理，参考了[你真的理解過 PING 這個指令嗎?](https://www.hwchiu.com/ping-implementation.html)。总结如下：

- ping命令的原理是通过ICMP报文，而ICMP报文需要使用RAW socket，这个是需要root权限的，因此早期的Linux发行版在安装ping命令时为了使普通用户也能使用，通过设置setuid位使得普通用户在执行ping命令时拥有了root权限

- 但由于setuid位所带来的权限太大，容易导致安全问题，后面的发行版则取消了设置setuid位改为通过capability权能使得普通用户可以使用ping命令，这样可以提供更加细致的权限管理，系统安全性也更高

但是在最近几个发行版中（Ubuntu18/20.04等），即使没有设置setuid位和capability权能也能使用ping命令。我们可以使用`strace`命令跟踪ping程序：

- 早期版本的ping命令通过`socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = 3`取得 一个基于RAW socket的ICMP socket
- 而最新版本的ping命令则是通过`socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP) = 3`取得一个基于DGRAM的 ICMP Socket
<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162206359.png">

但并非所有的group都可以使用DGRAM，组ID(GID)必须符合 net.ipv4.ping_group_range，可以通过`sysctl net.ipv4.ping_group_range`查看。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162207287.png">

可以看到GID范围默认是0~2147483647，而我们创建的用户GID为1000处于这个区间，因此可以顺利执行ping命令。

为了避免使用DGRAM，通过`sysctl net.ipv4.ping_group_range='0 0'`使得只允许root用户使用（如果使用容器进行实验，需要在创建容器时进行修改）。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162214182.png">

**实验环境**

- Ubuntu 20.04
- Docker version 23.0.1
- image：Ubuntu 18.04

初试环境配置

```shell
docker run --name="OS-LAB" -it --sysctl net.ipv4.ping_group_range='0 0' ubuntu:18.04 /bin/bash
# 创建容器
apt update
# root (安装ping命令)
apt install iputils-ping -y
# 安装VIM
apt install vim -y
# 修改默认shell为bash（选择no）
dpkg-reconfigure dash
# 新建一个用户让其可以使用ping命令
adduser user-ping
# 新建一个用户让其不能使用ping命令
adduser user-none
```

一开始由于设置了setuid位，普通用户也能使用ping命令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162226343.png">

此时我们取消掉setuid位，可以看到普通用户已经不能使用ping命令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162228738.png">

此时我们添加权能，普通用户又可以使用ping命令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162242362.png">

为了实现user-ping用户在登录时可自动拥有cap_net_raw权能，而其他用户登录时仍然不能使用ping命令，我们通过shell脚本以及pam完成。

> 脚本如下

```shell
#!/bin/bash

[ "$PAM_TYPE" == "open_session" ] || exit 0


if [ "$PAM_USER" == "user-ping" ]; then
        setcap cap_net_raw+eip $(which ping)
        echo "USER-PING CAN USE PING COMMAND :)"
else
        setcap -r $(which ping)
        echo "OOPS, YOU CAN'T USE PING COMMAND :("
fi
```

```shell
# 使脚本可执行
chmod u+x /root/login.sh
# root用户
vim /etc/pam.d/common-session
# 添加下列指令，注意脚本存储位置
session optional pam-exec.so debug log=/tmp/pam-exec.log seteuid /root/login.sh
```

user-ping用户可使用ping命令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162324100.png">

user-none用户不可使用ping命令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202303162326116.png">

## 参考资料

[1] [man capabilities: 介绍权能](https://man7.org/linux/man-pages/man7/capabilities.7.html)

[2] [Linux Capabilities 入门教程：概念篇](https://icloudnative.io/posts/linux-capabilities-why-they-exist-and-how-they-work/)

[3] [PAM](https://blogs.gnome.org/raywang/2007/02/01/pam%E5%85%A5%E9%97%A8%E4%BB%8B%E7%BB%8D-%E4%B9%9F%E5%BE%88%E4%B8%8D%E9%94%99/)

[4] [PAM中文手册](https://www.docs4dev.com/docs/zh/linux-pam/1.1.2/reference/sag-pam_exec.html)

[5] [基于PAM的用户权能分配—Tremb1e's Blog](https://www.tremb1e.com/archives/user-capability-assignment-based-on-pam)

[6] [基于PAM的用户权能分配—破落之实](https://blog.csdn.net/u013648063/article/details/106944141)

[7] [基于PAM的用户权能分配—在路上](https://zhuanlan.zhihu.com/p/146366045)

[8] [你真的理解過 PING 這個指令嗎?](https://www.hwchiu.com/ping-implementation.html)