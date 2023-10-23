---
layout: post
title: Open Container Initiative Runtime Specification
subtitle: OCI Runtime-Spec
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

本篇是对[Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)的介绍，主要是Linux平台。

OCI Runtime-Spec主要包括对容器配置文件、运行环境和生命周期的规范：

- 容器的配置文件 (config.json)，详细说明了创建容器的平台和其他详细信息
- 执行环境确保容器中的应用程序在不同运行时的环境一致
- 容器的生命周期定义了容器生命的各个阶段

## OCI and CNCF

Open Container Initiative (OCI) 是一个开放的行业标准化组织，旨在定义容器格式和运行时规范。它的目标是为容器生态系统提供一个通用的标准，使不同的容器运行时和工具能够互操作。OCI 的主要成果是 OCI image-spec和 OCI runtime-spec，它们定义了容器镜像的格式和容器运行时的规范。

Cloud Native Computing Foundation (CNCF) 是一个非营利性的开源基金会，致力于推动云原生的发展和采用。云原生是一种构建和运行应用程序的方法，它倡导将应用程序设计为由多个微小、独立且可扩展的服务组成的体系结构，并利用容器、编排、自动化和微服务等技术来实现高度可伸缩性和弹性。CNCF 的使命是通过托管和培育开源项目、推广最佳实践和教育培训等方式，推动云原生技术的创新和普及。

OCI 和 CNCF 的关系在于，CNCF 托管和支持了一些与容器和云原生技术相关的开源项目，其中包括符合 OCI 标准的容器运行时和容器镜像工具，包括 Kubernetes（容器编排系统）、Prometheus（监控系统）、Envoy（代理服务器）等。

## Standard Containers

标准容器定义规范包括：

- 配置文件格式
- 一系列标准操作
- 执行环境

何为标准容器：

1. Standard operations：能执行“创建”、“启动”、“停止”、“复制”、“快照”、“上传”和“下载”等标准操作
2. Content-agnostic：标准容器是与其内容无关的，比如不同内容的容器启动方式都是一样的
3. Infrastructure-agnostic：标准容器和其运行的基础设施是无关的，比如可以在笔记本和服务器上运行，只要支持OCI规范即可
4. Designed for automation：标准容器支持自动化操作，这也是云原生的优势
5. Industrial-grade delivery：标准容器是符合工业级交付标准的

## Bundle

Bundle只涉及如何将容器及其配置数据存储在本地文件系统中，以便兼容OCI规范的任何运行时加载。Bundle包含加载和运行容器所需的全部信息：

1. config.json：命名**必须**为config.json，且**必须**在bundle目录的根目录
2. 容器的根文件系统：在config.json中的root.path字段指定

Bundle目录中的内容才是容器必须，其目录本身非必须。

## Runtime and Lifecycle

### state

容器的state（状态）除了容器的status（运行状态），还包括其他持久化数据（下列属性除了标注可选均为必须，格式除了标注均为string），除了以下元素外，state可能包含其他元素：

- ociversion
- id：容器的id，**本机上对于特定容器，其id必须唯一；不同主机上的容器id不要求必须唯一**
- status：容器的运行状态，除了以下状态，**不同的容器运行时可能定义其他状态**
	- creating：容器正在创建，对应lifecycle step 2
	- created：容器已完成创建，**容器进程既没有退出，也没有执行用户指定的程序（config.json的process元素）**，对应lifecycle step 2后
	- running：容器进程**正在执行用户指定的程序，但没有退出**，对应lifecycle step 8后
	- stopped：容器进程已退出，对应lifecycle step 10
- pid (int)：status处于created或running的容器进程id，后面一段话没看懂，直接摘抄：`For hooks executed in the runtime namespace, it is the pid as seen by the runtime. For hooks executed in the container namespace, it is the pid as seen by the container.`
- bundle：绝对路径
- annotations (map, optional)：与容器相关联的注释列表

example（id一般为sha-256值）：

```json
{
    "ociVersion": "0.2.0",
    "id": "oci-container1",
    "status": "running",
    "pid": 4422,
    "bundle": "/containers/redis",
    "annotations": {
        "myKey": "myValue"
    }
}
```

### lifecycle

Lifecycle定义了容器从创建到退出之间的时间轴（下列均为精简的，可能没那么准确，见[官方描述](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md#lifecycle)）：

1. 容器开始创建：**通常为OCI规范运行时（runc）调用create命令+bundle+container id**
2. 容器运行时环境创建中： 根据容器的config.json中的配置进行创建，此时用户指定程序还未运行，这一步后所有对config.json的更改均不会影响容器
3. prestart hooks
4. createRuntime hooks
5. createContainer hooks
6. 容器启动：**通常为OCI规范运行时（runc）调用start命令+container id**
7. startContainer hooks
8. 容器执行用户指定程序
9. poststart hooks：任何poststart钩子执行失败只会log a warning，不影响其他生命周期（操作继续执行）就好像钩子成功执行一样
10. 容器进程退出：error、正常退出和运行时调用kill命令均会导致
11. 容器删除：**通常为OCI规范运行时（runc）调用delete命令+container id**
12. 容器摧毁：**区别于容器删除，3、4、5、7的钩子执行失败除了生成一个error外，会直接跳到这一步**。撤销第二步创建阶段执行的操作。
13. poststop hooks：执行失败后的操作和poststart一致

### operations

运行时必须实现以下操作（参数均为一般参数）：

- state \<container-id>：返回容器的[state信息](###state)
- create \<container-id> \<path-to-bundle>：config.json中除了process的所有元素必须被应用，process.args在start操作前必须未被应用，其他字段可在create操作时应用。config.json中的元素若不能被应用则会报错，并且容器不会被创建。这一步之后所有对config.json的更改均不会影响容器
- start \<container-id>：用户指定程序在这一步必须执行
- kill \<container-id> \<signal>
- delete \<container-id>：删除容器必须删除在create步骤中创建的资源，与容器关联但不是由该容器创建的资源不得删除。容器删除后，其ID可能会被后续容器使用

## Configuration

Configuration描述了容器config.json中各元素，篇幅太多，另起一文。

## Features Structure

Features Structure通常是运行时提供的其实现的功能特性的json文件。

## Glossary

[Glossary](https://github.com/opencontainers/runtime-spec/blob/main/glossary.md)描述了OCI对容器相关定义，不翻译直接摘抄一些。

### container

**An environment for executing processes with configurable isolation and resource limitations.** For example, namespaces, resource limits, and mounts are all part of the container environment.

> 总结的很好，容器其实就是可配置隔离和资源限制的进程执行环境；毕竟我们运行容器实际是为了执行容器中的特定程序

### runtime

An implementation of this specification. **It reads the configuration files from a bundle, uses that information to create a container, launches a process inside the container, and performs other lifecycle actions.**

> OCI对runtime的定义就是最终执行容器生命周期的工具，即低级运行时，例如runc

### runtime caller

**An external program to execute a runtime, directly or indirectly.**

**Examples of direct callers include containerd, CRI-O, and Podman. Examples of indirect callers include Docker/Moby and Kubernetes.**

Runtime callers often execute a runtime via runc-compatible command line interface, however, its interaction interface is currently out of the scope of the Open Container Initiative Runtime Specification.

> 中文博客对运行时的描述其实是模糊的，经常看到runc是运行时、containerd是运行时、docker和k8s也是运行时，极其容易让人混淆。OCI对此做了一个很好的定义，运行时只是最底层的低级的直接和容器交互的程序；而高级的面向用户的，一般直接或间接调用运行时的程序则称为runtime caller
