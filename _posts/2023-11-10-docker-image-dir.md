---
layout: post
title: Docker镜像的相关目录
subtitle: No Subtitle ：>
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
##  docker pull的基本流程

> 这里暂不对源码进行分析，只介绍基本流程

cv from [docker 在本地如何管理 image（镜像）?](https://icloudnative.io/posts/how-manage-image/)

- docker发送**image的名称+tag（或者digest）**给registry服务器，服务器根据收到的image的名称+tag（或者digest），找到**相应image的manifest**，然后将manifest返回给docker
- docker得到manifest后，读取里面**image配置文件（config）的 digest(sha256)，这个sha256码就是 image的ID**
- **根据ID在本地找有没有存在同样ID的image**，有的话就不用继续下载了
- 如果没有，那么会给registry服务器发请求（里面包含**配置文件的digest(sha256)和media type**），拿到 **image的配置文件（Image Config）**
- 根据**配置文件中的diff_ids**（每个diffid对应一个layer tar包的sha256），在本地找对应的layer是否存在
- 如果layer不存在，则根据manifest里面**layer的digest(sha256)和media type**去服务器拿相应的layer（区别于config中的diffid，manifest中的digest通常是对manifest中记录的layer的mediatype做sha256计算所得，通常来说这个mediatype一般为tar的压缩格式即tar.gzip）。
- 拿到后进行解压，并**检查解压后tar包的sha256是否与配置文件（Image Config）中的diff_id一致**，不一致说明文件被篡改，下载失败
- 根据docker所用的后台文件系统类型，**解压tar包并放到指定的目录**
- 所有的layer都下载完成后，整个image下载完成

manifest文件仅用于镜像文件的下载，镜像下载完成后被删除。

## /var/lib/docker/image/overlay2

docker中镜像元数据存储的路径一般是`/var/lib/docker/image/<union-fs-name>`，`<union-fs-name>`指主机上docker使用的联合文件系统类型，主要为overlay2，此外还有aufs等。

此目录下主要包括如下内容：

```shell
root@debian:/var/lib/docker/image/overlay2# ls
distribution  imagedb  layerdb  repositories.json
```

### distribution

distribution目录中存储manifest中layer的digest_id与config中layer的diff_id的映射关系。

```shell
root@debian:/var/lib/docker/image/overlay2/distribution# ls
diffid-by-digest  v2metadata-by-diffid
root@debian:/var/lib/docker/image/overlay2/distribution# ls -l diffid-by-digest/
total 16
drwxr-xr-x 2 root root 16384 Nov  9 10:11 sha256
root@debian:/var/lib/docker/image/overlay2/distribution# ls -l v2metadata-by-diffid/
total 20
drwxr-xr-x 2 root root 20480 Nov  9 10:11 sha256
```

#### diffid-by-digest

diffid-by-digest存储digest对应的diff_id。

```shell
root@debian:/var/lib/docker/image/overlay2/distribution/diffid-by-digest/sha256# cat ff63b82a42d3d44071011eb2a1262a77c15520b940511c05ebd7a7c3019fdff6 
sha256:60af18e613ad753f3b25da7242ae2f8ee3481f6768bc22c23be352f9906157df
```

#### v2metadata-by-diffid

v2metadata-by-diffid存储diffid对应的digest_id以及layer的其他元数据。

```shell
root@debian:/var/lib/docker/image/overlay2/distribution/v2metadata-by-diffid/sha256# cat 60af18e613ad753f3b25da7242ae2f8ee3481f6768bc22c23be352f9906157df | jq
[
  {
    "Digest": "sha256:ff63b82a42d3d44071011eb2a1262a77c15520b940511c05ebd7a7c3019fdff6",
    "SourceRepository": "docker.io/library/golang",
    "HMAC": ""
  }
]
```

### imagedb

imagedb目录主要存储镜像的配置信息：

```shell
root@debian:/var/lib/docker/image/overlay2/imagedb# ls -l
total 8
drwx------ 3 root root 4096 Jun 22  2022 content
drwx------ 3 root root 4096 Jun 22  2022 metadata
```

#### content

content目录存储镜像的config文件，文件数量和主机上的镜像数量一致，文件以对应的镜像id（也就是config文件的sha256值）命名：

```shell
root@debian:/var/lib/docker/image/overlay2/imagedb/content/sha256# ls
c6b84b685f35f1a5d63661f5d4aa662ad9b7ee4f4b8c394c022f25023c907b65
root@debian:/var/lib/docker/image/overlay2/imagedb/content/sha256# docker images
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
ubuntu                              latest    c6b84b685f35   2 months ago    77.8MB
```

文件的内容为镜像的config内容。

#### metadata

没找到太多资料，说的是里面存放的是本地image的一些信息，从服务器上pull下来的image不会存数据到这个目录。

### layerdb

layerdb目录存储容器和镜像层的相关数据：

```shell
root@debian:/var/lib/docker/image/overlay2/layerdb# ls -l
total 40
drwxr-xr-x   5 root root  4096 Nov  9 10:24 mounts
drwxr-xr-x 174 root root 28672 Nov  9 10:18 sha256
drwxr-xr-x   2 root root  4096 Nov  9 10:18 tmp
```

#### mounts

mounts目录存储容器的相关元数据，文件以容器的id命名，数量与当前主机上的容器数量一致：

```shell
root@debian:/var/lib/docker/image/overlay2/layerdb/mounts# ls
6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc  
root@debian:/var/lib/docker/image/overlay2/layerdb/mounts# docker ps -a
CONTAINER ID   IMAGE             COMMAND                  CREATED        STATUS                      PORTS     NAMES
6e9d18b4213d   ubuntu            "/bin/bash"              4 days ago     Exited (137) 43 hours ago             youthful_ganguly
```

单个文件主要包含以下内容，用于定位真正的镜像层数据（位于/var/lib/docker/overlay2/）

```shell
root@debian:/var/lib/docker/image/overlay2/layerdb/mounts/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# ls
init-id  mount-id  parent

root@debian:/var/lib/docker/image/overlay2/layerdb/mounts/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# cat init-id 
983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045-init

root@debian:/var/lib/docker/image/overlay2/layerdb/mounts/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# cat mount-id 
983919beba7649a2c10150782c4025b83643c9c058bbb35330083b9d731a0045

root@debian:/var/lib/docker/image/overlay2/layerdb/mounts/6e9d18b4213dbd02abfb392ee14c3d00c103a4b21701495cf70eb9c91dbb49cc# cat parent 
sha256:dc0585a4b8b71f7f4eb8f2e028067f88aec780d9ab40c948a8d431c1aeadeeb5
```

- mount-id：存储在/var/lib/docker/overlay2/的目录名称，其中的merged目录为容器运行的目录（镜像层、init层和容器层的联合挂载）
- init-id：init-id是在mount-id后加了一个-init，同时init-id就是存储在/var/lib/docker/overlay2/的目录名称，存储/etc/hosts、/etc/resolv.conf等配置文件，为容器的init层
- parent：容器所基于的**镜像的最上层的chain_id**

#### sha256

sha256目录存储镜像层的元文件信息，以layer的chain-id命名。

> chain-id是由diff-id转换而来的，具体可见oci-image-spec中的解释。
>
> 简单来说最底层的layer的chain-id等于其diff-id；
>
> 高层的layer的chain-id等于上一层的chain-id与其diff-id的sha256值，例如第二层b的chain-id为sha256(chain-id(a) + diff-id(b)) = sha256(diff-id(a) + diff-id(b)) = sha256sum "sha256:xxxxx sha256:xxx"

单个文件内容如下：

```shell
root@debian:/var/lib/docker/image/overlay2/layerdb/sha256/fc0c41d1a2e2d77359e7e8ce875933293ef769d32feb2cd3dc79b61f3edbbe6c# ls -l
total 144
-rw-r--r-- 1 root root     64 Jul 21 17:31 cache-id
-rw-r--r-- 1 root root     71 Jul 21 17:31 diff
-rw-r--r-- 1 root root     71 Jul 21 17:31 parent
-rw-r--r-- 1 root root      7 Jul 21 17:31 size
-rw-r--r-- 1 root root 127848 Jul 21 17:31 tar-split.json.gz

root@debian:/var/lib/docker/image/overlay2/layerdb/sha256/fc0c41d1a2e2d77359e7e8ce875933293ef769d32feb2cd3dc79b61f3edbbe6c# cat cache-id 
24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5

root@debian:/var/lib/docker/image/overlay2/layerdb/sha256/fc0c41d1a2e2d77359e7e8ce875933293ef769d32feb2cd3dc79b61f3edbbe6c# cat diff 
sha256:c00acf6f4f58f29b845ee034bf403750e0735ebcfcd2500427f5fa131033366b

root@debian:/var/lib/docker/image/overlay2/layerdb/sha256/fc0c41d1a2e2d77359e7e8ce875933293ef769d32feb2cd3dc79b61f3edbbe6c# cat parent 
sha256:21e7914e06fb3d26ac5cd44a4c2782a2dc524c9efaa1eed3b144b7b34e7948df

root@debian:/var/lib/docker/image/overlay2/layerdb/sha256/fc0c41d1a2e2d77359e7e8ce875933293ef769d32feb2cd3dc79b61f3edbbe6c# cat size
7518792
```

- cache-id：docker下载layer的时候在本地生成的一个随机uuid，指向真正存放layer文件的地方（/var/lib/docker/overlay2/\<cache-id>）
- diff：当前layer的diff_id。
- parent：当前layer的父layer的diff_id，最底层的layer不含此文件
- size：当前layer的大小，单位是字节
- tar-split.json.gz：layer压缩包的split文件，通过这个文件可以还原layer的tar包，在docker save导出image的时候会用到

## /var/lib/docker/overlay2/\<cache-id>

此目录用于存储镜像层的实际数据：

```shell
root@debian:/var/lib/docker/overlay2/24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5# ls
committed  diff  link  lower  work
root@debian:/var/lib/docker/overlay2/24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5# ls diff/
etc  harbor  home  run  usr  var
root@debian:/var/lib/docker/overlay2/24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5# cat link
GXXH6DHT3KL47AJ55XB3GQWO5B
root@debian:/var/lib/docker/overlay2/24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5# ls -l /var/lib/docker/overlay2/l/GXXH6DHT3KL47AJ55XB3GQWO5B
lrwxrwxrwx 1 root root 72 Jul 21 17:31 /var/lib/docker/overlay2/l/GXXH6DHT3KL47AJ55XB3GQWO5B -> ../24268712204d9aef7a10308a577cf6c149506f7414ff367a4ff869cf27c5fcf5/diff
```

- diff：存储该层的实际内容
- link：层的标识符（更短的符号链接，位于/var/lib/docker/overlay2/l，指向此目录的diff）
- lower：父层镜像的符号链接，**当启动容器时会将link以及lower的符号链接指向的目录联合挂载到merged目录，构建出整个容器的层次结构**
- merged：容器的工作目录，只有在容器启动时存在
- work：用于cow



