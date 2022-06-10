---
layout: post
title: "Linux文件系统"
subtitle: "The Linux File System"
author: "PYQ"
header-img: "img/post-bg-linux.jpg"
header-mask: 0.3
catalog: true
tags:
  - Linux
---

## Ⅰ. FHS

GNU/Linux自诞生以来拥有众多的发行版，著名的Linux发行版情报网站[Distrowatch.com](https://distrowatch.com/)上已有数百个不同的发行版，其中包括Archlinux、Debian、Manjaro、Ubuntu、Cent OS等著名发行版。

![Linux发展](/img/in-post/the-linux-file-system-1.jpeg)

*常见的Linux发行版 [图片来源: CodeSheep](https://b23.tv/ZL6UM9V)*

这些不同的Linux发行版拥有不同的包管理器以及不同的UI界面，但它们都有相同的目录结构。提到GNU/Linux系统的目录结构，不得不说一下[FHS（Filesystem Hierarchy Standard，文件系统层次结构标准）](https://www.pathname.com/fhs/)，正是这一标准的存在，详细的定义了类Unix操作系统中各种应用软件，管理工具，开发工具，脚本以及帮助文档的位置，让遵循该标准的各种GNU/Linux发行版目录结构几乎都是一致的。

![FHS目录结构](/img/in-post/the-linux-file-system-2.png)

> Ubuntu 20.04目录结构

![Ubuntu文件结构](/img/in-post/the-linux-file-system-3.png)

> Cent OS 8目录结构

![Cent OS文件结构](/img/in-post/the-linux-file-system-4.png)

## Ⅱ. Linux目录结构

在FHS中，根据文件的共享性和文件是否要求被修改的特点，其将文件划分为**可共享文件/不可共享文件**以及**可变数据文件/静态数据文件**两大类。FHS将不同属性的文件划归到不同的目录，以便系统管理。

![FHS文件划分](/img/in-post/the-linux-file-system-5.png)

下面将对Linux主要文件目录进行描述：

1. /：整个文件系统层次的根目录，可以理解为树的根节点。

2. ~：打开终端会话/切换用户的起始位置。

3. /bin：存放着整个系统必须的二进制文件或可执行文件，可以通过命令进行运行，包括vim、ls、gzip等。类似的目录有/usr/bin、/usr/local/bin（/usr/local存放手动编译的二进制文件）等。

4. /sbin：存放着只能由root用户运行的系统二进制文件，比如mount。类似的文件目录有/usr/sbin、/usr/local/sbin等。

5. /boot：引导程序文件所在目录，包括Linux内核等。

6. /dev：设备文件目录，比如光驱、硬盘等，访问该目录下某个文件相当于访问某个硬件设备。

7. /etc：系统的配置文件目录，一些软件启动时默认读取配置文件的目录。

8. /home：存放用户文件目录，该目录下不同的账号对应不同的目录。

9. /lib：主要存放动态链接库，/bin与/sbin目录下的二进制文件所依赖的库均存放于此。

10. /lib64：64位系统有/lib64文件夹。

11. /media：可移除媒体（如CD-ROM）的挂载点。

12. /mnt：临时挂载的文件系统，比如CD-ROM、U盘等。

13. /proc：虚拟文件系统，是系统内存的映射，可直接访问该目录来获取系统信息。

14. /opt：可选应用软件包，对系统使用几乎没有任何影响。

15. /root：root用户的主目录。

16. /srv：存放本机或本服务器提供的服务。

17. /sys：存放着与Linux系统相关的文件。

18. /tmp：存放临时文件，系统重启时该目录中文件不会被保留。

19. /usr：默认软件与文件存放于该目录下。

20. /var：存放操作系统运行过程中会发生变化的文件，比如系统日志与缓存文件。

    

