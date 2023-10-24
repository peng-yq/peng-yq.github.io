---
layout: post
title: "容器钩子"
subtitle: "Container Hook"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
  - docker
---

## container hooks

### hooks

什么是钩子呢？

[Wikipedia中的定义如下](https://en.wikipedia.org/wiki/Hooking)。

>In computer programming, the term hooking covers a range of techniques used to alter or augment the behavior of an operating system, of applications, or of other software components by intercepting function calls or messages or events passed between software components. Code that handles such intercepted function calls, events or messages is called a "hook".

[一张很形象的图](https://www.zhihu.com/question/20610442/answer/128226574)。

<img src="https://picx.zhimg.com/80/v2-d995ff87806caf4e6619a2bbc3e523da_720w.webp?source=1940ef5c">

钩子主要有两个关键点，一个是导致钩子调用的事件（某个事件发生前或者发生后），一个是钩子的具体代码。因此，钩子实际上就是在某个时间点或事件点触发的一系列函数或代码。

### lifecycle

容器中的钩子和容器的生命周期息息相关，钩子能使容器感知其生命周期内的事件，并且当相应的生命周期钩子被调用时运行指定的代码。[oci runtime-spec](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md#lifecycle)对容器生命周期的描述如下（单纯从低级运行时创建容器开始，不包括镜像）。

> The lifecycle describes the timeline of events that happen from when a container is created to when it ceases to exist.
>
> - OCI compliant runtime's create command is invoked with a reference to the location of the bundle and a unique identifier.
> - **The container's runtime environment MUST be created according to the configuration in config.json**. If the runtime is unable to create the environment specified in the config.json, it MUST generate an error. While the resources requested in the config.json MUST be created, the user-specified program (from process) MUST NOT be run at this time. Any updates to config.json after this step MUST NOT affect the container.
> - The **prestart hooks** MUST be invoked by the runtime. If any prestart hook fails, the runtime MUST generate an error, stop the container, and continue the lifecycle at step 12.
> - The **createRuntime hooks** MUST be invoked by the runtime. If any createRuntime hook fails, the runtime MUST generate an error, stop the container, and continue the lifecycle at step 12.
> - The **createContainer hooks** MUST be invoked by the runtime. If any createContainer hook fails, the runtime MUST generate an error, stop the container, and continue the lifecycle at step 12.
> - Runtime's start command is invoked with the unique identifier of the container.
> - The **startContainer hooks** MUST be invoked by the runtime. If any startContainer hook fails, the runtime MUST generate an error, stop the container, and continue the lifecycle at step 12.
> - The runtime MUST run the user-specified program, as specified by process.
> - The **poststart hooks** MUST be invoked by the runtime. If any poststart hook fails, the runtime MUST log a warning, but the remaining hooks and lifecycle continue as if the hook had succeeded.
> - The container process exits. This MAY happen due to erroring out, exiting, crashing or the runtime's kill operation being invoked.
> - Runtime's delete command is invoked with the unique identifier of the container.
> - The container MUST be destroyed by undoing the steps performed during create phase (step 2).
> - The poststop hooks MUST be invoked by the runtime. If any poststop hook fails, the runtime MUST log a warning, but the remaining hooks and lifecycle continue as if the hook had succeeded.

Lifecycle定义了容器从创建到退出之间的时间轴：

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

可以看到oci定义的容器生命周期中，如果在容器的config.json中定义了钩子，runc必须执行钩子，并且时间节点在前的钩子执行成功后才能执行下一个钩子；若有一个钩子执行失败，则会报错并摧毁容器（在容器创建后执行的钩子失败，并不会删除容器，而是启动失败）。

除了上述oci通过runc创建并启动容器的流程来对容器生命周期的描述外，docker和k8s也有各自对容器生命周期的描述（出于容器的不同状态），均符合oci规范。

Docker

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202309081015834.png">

[Pod 的生命周期-K8s](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-lifetime)

### oci hooks

结合生命周期来看。

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

#### Prestart

Prestart钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**废弃，被后面三个钩子所取代**

#### CreateRuntime Hooks

createRuntime钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**在容器命名空间被创建后调用**。

> createRuntime 钩子的定义目前未作明确规定，钩子作者只能期望运行时创建挂载命名空间并执行挂载操作。运行时可能尚未执行其他操作，如 cgroups 和 SELinux/AppArmor 标签

#### CreateContainer Hooks

createContainer钩子必须作为创建操作的一部分，在运行时环境创建完成后（根据 config.json 中的配置），但在执行 pivot_root 或任何同等操作之前调用。

**在执行 pivot_root 操作之前，但在创建和设置挂载命名空间之后调用**。

#### StartContainer Hooks

StartContainer钩子作为启动操作的一部分，**必须在执行用户指定的进程之前调用startContainer挂钩**。此钩子可用于在容器中执行某些操作，例如在容器进程生成之前在linux上运行ldconfig二进制文件。

#### Poststart

Poststart钩子**必须在用户指定的进程执行后、启动操作返回前调用**。例如，此钩子可以通知用户容器进程已生成。

#### Poststop

Poststart钩子**必须在容器删除后、删除操作返回前调用**。清理或调试函数就是此类钩子的例子。

#### summary

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

### hooks supported in docker and k8s

docker并不支持通过参数直接添加使用hooks，或者说现在不支持（调研中发现有一两篇博客写到可以通过docker run --hooks-path添加钩子，但通过`docker --help | grep hook`命令找不到任何和hook相关的参数）。

k8s只支持[postStart和preStop hooks](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)，通过在pod的yaml文件中定义即可。但实际上我认为prestart（或者说其细分后的三个hooks）更加实用一些，比如在创建或启动容器前进行一些网络检查和配置。有意思的是在[k8s的issue中有一个关于“PreStart lifecycle hook Required”的讨论](https://github.com/kubernetes/kubernetes/issues/96560)，讨论提到了修改Dockerfile并在entrypoint.sh中执行，以及使用[k8s提供的init容器](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/init-containers/)对prestart的代替。

补充：[PreStart and PostStop event hooks #140](https://github.com/kubernetes/kubernetes/issues/140)中有老哥回答k8s不会再支持prestart hook了，取而代之的是init容器。

## oci-add-hooks (awslabs)

### introduction

oci-add-hooks是awslabs编写的一个开源的runtime，其本质是一个runc的wrapper，即oci-add-hooks去做一些runc调用前的操作，执行完毕后再调用runc去启动容器。

**Copy from [oci-add-hooks](https://github.com/awslabs/oci-add-hooks)**

> oci-add-hooks will parse the file specified by hook-config-path as specified in the runtime-spec section on POSIX-platform hooks. It will merge these hooks into the config.json file located at the path passed to bundle, writing the changes back to disk. If hooks are already present in the spec, it will pre-pend these hooks to the existing ones. It will then strip out the options and args that are specific to oci-add-hooks and passthrough to the binary pointed at by runtime-path.

### package required

#### json-lossless

[json-lossless](https://pkg.go.dev/github.com/joeshaw/json-lossless@v0.0.0-20181204200226-e0cd1ca6349b#section-readme) 是一个Go库，它可以从JSON中填充结构体，并允许序列化回JSON，而不会丢失结构体中未明确定义的字段。

#### encoding/json

一些教程：

- [go-json-的编码和解码](https://medium.com/@xfstart07/go-json-%E7%9A%84%E7%BC%96%E7%A0%81%E5%92%8C%E8%A7%A3%E7%A0%81-e689522a1f1f)
- [你需要知道的那些go语言json技巧](https://www.liwenzhou.com/posts/Go/json-tricks/)

#### io/ioutil

> "io/ioutil" is deprecated: As of Go 1.16, the same functionality is now provided by package io or package os, and those implementations should be preferred in new code.

oci-add-hooks这个项目比较老了（init commit是2018年；latest commit是2021年），源码中使用的是io/ioutil中的WriteFile和ReadFile方法，在GO 1.16后已被弃用，但为了兼容性考虑，我们依旧可以正常调用这个包，只不过当我们调用ioutil里面的方法，最后都是跳转到了io以及os包（code中的WriteFile和ReadFile方法是跳转到os包）。

[从Golang项目中删除已弃用的ioutil包](https://tehub.com/a/b28Y1oz6AC)

除开上面三个包，还有一些其他的包，比如fmt、syscall等

### hook.go

对config和hooks结构体进行定义：

- 在结构体中嵌入**lossless.JSON \`json:"-"\`**，`"-"` 来指示编码和解码过程中忽略字段
- config结构体中嵌套了hooks结构体，注意是*hooks类型，之所以嵌套一层而不直接使用hooks，是因为[oci runtime-spec](https://github.com/opencontainers/runtime-spec/blob/main/config.md#posix-platform-hooks)中容器config文件中钩子字段格式要求
- hooks结构体中定义了[oci runtime-spec定义的钩子](https://github.com/opencontainers/runtime-spec/blob/main/config.md#posix-platform-hooks)

```go
type config struct {
	lossless.JSON `json:"-"`

	Hooks *hooks `json:"hooks"`
}

type hooks struct {
	lossless.JSON `json:"-"`

	Prestart        []json.RawMessage `json:"prestart"`
	CreateRuntime   []json.RawMessage `json:"createRuntime"`
	CreateContainer []json.RawMessage `json:"createContainer"`
	StartContainer  []json.RawMessage `json:"startContainer"`
	Poststart       []json.RawMessage `json:"poststart"`
	Poststop        []json.RawMessage `json:"poststop"`
}
```

在config结构体和hooks结构体上定义MarshalJSON和UnmarshalJSON方法（对应着JSON的序列化和反序列化），以实现json.Marshaler和json.Unmarshaler接口，实现二次封装：

```go
func (c *config) UnmarshalJSON(data []byte) error {
	return c.JSON.UnmarshalJSON(c, data)
}

func (c *config) MarshalJSON() ([]byte, error) {
	return c.JSON.MarshalJSON(c)
}

func (h *hooks) UnmarshalJSON(data []byte) error {
	return h.JSON.UnmarshalJSON(h, data)
}

func (h *hooks) MarshalJSON() ([]byte, error) {
	return h.JSON.MarshalJSON(h)
}
```

writeFile方法：

1. 首先将定义的config结构体转换为JSON格式的byte数组
2. 通过os.stat获取给定路径path的文件描述，包括Name()，Size()， IsDir()，ModTime()，Mode()，这里只用到了权限Mode()
3. 如果给定路径path文件不存在，我们将文件的权限设置为0666（8进制的666，对应unix系统UGO权限rw_rw_rw_，即所有用户均可读写但不可执行的文件）；如果给定路径path文件存在，则沿用文件本身的权限
4. 最后将JSON写入文件

```go
func (c *config) writeFile(path string) error {
	bytes, err := json.Marshal(c)
	if err != nil {
		return err
	}
	// get current permissions and write with same permissions
	var mode os.FileMode
	info, err := os.Stat(path)
	if err != nil {
		// If the file isn't here we still want to write it
		// default to 0666
		mode = 0666
	} else {
		mode = info.Mode()
	}
	return ioutil.WriteFile(path, bytes, mode.Perm())
}
```

merge方法：

1. 主要是将in中的hooks合并到c中对应字段，其中mergeHook是将c中的钩子追加到in对应钩子的末尾，涉及到钩子执行顺序的问题，越前的钩子越先执行

```go
func (c *config) merge(in *config) {
	// if nil nothing to add
	if in == nil {
		return
	}
	c.Hooks.Prestart = mergeHook(c.Hooks.Prestart, in.Hooks.Prestart)
	c.Hooks.CreateRuntime = mergeHook(c.Hooks.CreateRuntime, in.Hooks.CreateRuntime)
	c.Hooks.CreateContainer = mergeHook(c.Hooks.CreateContainer, in.Hooks.CreateContainer)
	c.Hooks.StartContainer = mergeHook(c.Hooks.StartContainer, in.Hooks.StartContainer)
	c.Hooks.Poststart = mergeHook(c.Hooks.Poststart, in.Hooks.Poststart)
	c.Hooks.Poststop = mergeHook(c.Hooks.Poststop, in.Hooks.Poststop)
}
```

readHooks方法：

1. 将给定路径path的文件转换为*config类型的结构体并返回

```go
func readHooks(path string) (*config, error) {
	bytes, err := ioutil.ReadFile(path)
	if err != nil {
		return nil, err
	}
	h := config{
		Hooks: &hooks{},
	}
	if err = json.Unmarshal(bytes, &h); err != nil {
		return nil, err
	}
	return &h, nil
}
```

mergeHook方法：

1. 将a中的字段追加到b对应字段的末尾，并返回合并后的JSON

```go
func mergeHook(a, b []json.RawMessage) []json.RawMessage {
	if a == nil && b == nil {
		return []json.RawMessage{}
	}
	if a == nil {
		return b
	}
	if b == nil {
		return a
	}
	return append(b, a...)
}
```

### main.go

一些变量和常量定义：

1. signalBufferSize用于捕获发送给该进程的os.Signal的缓冲区大小。32表示可以同时捕获并处理的信号数量上限为 32

```go
const (
	// Size of the buffer for catching os.Signal sent to this process
	signalBufferSize = 32
	exitCodeFailure  = 1
)

var (
	errUnableToFindRuntime = errors.New("unable to find runtime")

	commit string
)
```

main函数：

1. 作者先解释了为什么采用手动的的标志解析（flag parsing），而不使用默认的flag包。默认的flag包没有只选择性解析某些标志并忽略未知标志的能力
2. 只接受以下两个命令：
   - self --version
   - self --hook-config-path /path/to/hookcfg --runtime-path /path/to/runc ... runtime flags
3. 分别赋值给hookConfigPath, runcPath, passthroughArgs，并调用run函数，run函数执行的结果决定了程序是否正常退出

```go
func main() {
	// We are doing manual flag parsing b/c the default flag package
	// doesn't have the ability to parse only some flags and ignore unknown
	// ones. Just requiring positional arguments for simplicity.
	// We are expecting command line like one of the following:
	// self --version
	// self --hook-config-path /path/to/hookcfg --runtime-path /path/to/runc, ... runtime flags
	// If we don't match one of these these, we can exit
	if len(os.Args) == 2 && os.Args[1] == "--version" {
		fmt.Println("commit:", commit)
		os.Exit(0)
	} else if len(os.Args) < 6 || (os.Args[1] != "--hook-config-path" && os.Args[3] != "--runtime-path") {
		os.Exit(exitCodeFailure)
	}
	// If are args are present, grab the values
	hookConfigPath := os.Args[2]
	runcPath := os.Args[4]
	passthroughArgs := os.Args[5:]
	os.Exit(run(hookConfigPath, runcPath, passthroughArgs))
}
```

run函数

```go
func run(hookConfigPath, runcPath string, runcArgs []string) int {
	// If required args aren't present, bail
	if hookConfigPath == "" || runcPath == "" {
		return exitCodeFailure
	}

	// If a hookConfigPath passed, process the bundle and pass modified
	// spec to runc
	return processBundle(hookConfigPath, runcPath, runcArgs)
}
```

processBundle函数：

1. 首先需要解析出runcArgs中的bundle的路径即"--bundle"，并形成config.json的路径，赋值给bundlePath变量
2. 调用addHooks函数将hookPath中的钩子合并到bundle的config.json中，并将*config返回值赋给merged
3. 调用writeFile方法，写入bundle的config.json文件
4. 最后就是调用runc了，并传入对应的参数

```go
func processBundle(hookPath, runcPath string, runcArgs []string) int {
	// find the bundle json location
	for i, val := range runcArgs {
		if val == "--bundle" && i != len(runcArgs)-1 {
			// get the bundle Path
			bundlePath := runcArgs[i+1]
			bundlePath = filepath.Join(bundlePath, "config.json")
			// Add the hooks from hookPath to our bundle/config.json
			merged, err := addHooks(bundlePath, hookPath)
			if err != nil {
				return exitCodeFailure
			}
			err = merged.writeFile(bundlePath)
			if err != nil {
				return exitCodeFailure
			}
			break
		}
	}
	// launch runc
	path, err := verifyRuntimePath(runcPath)
	if err != nil {
		return exitCodeFailure
	}
	return launchRunc(path, runcArgs)
}
```

addHooks函数：

1. hookPath中的钩子合并到bundlePath中（bundle的config.json）

```go
func addHooks(bundlePath, hookPath string) (*config, error) {
	specHooks, err := readHooks(bundlePath)
	if err != nil {
		return nil, err
	}
	addHooks, err := readHooks(hookPath)
	if err != nil {
		return nil, err
	}
	specHooks.merge(addHooks)
	return specHooks, nil
}
```

verifyRuntimePath函数：

1. 检查用户定义的runc路径是否正确，当然只是通过检查路径是否为空？是否是目录？是否是常规文件（IsRegular就测试了IsDir，有一定的重合）
2. 总体来说这个路径检查函数是最基本的检查，要想runc正常执行，还是需要用户在定义的时候就正确定义

```go
func verifyRuntimePath(userDefinedRuncPath string) (string, error) {
	info, err := os.Stat(userDefinedRuncPath)
	if err == nil && !info.Mode().IsDir() && info.Mode().IsRegular() {
		return userDefinedRuncPath, nil
	}
	return "", errUnableToFindRuntime
}
```

launchRunc函数：

1. 使用prepareCommand函数创建一个命令对象cmd，包含了runc可执行文件的路径和参数
2. 创建一个缓冲区大小为signalBufferSize的信号通道proc，用于捕获发送给该进程的信号
3. 调用signal.Notify(proc)，将信号通道注册为接收所有信号的接收器，以便在启动命令之前处理信号
4. 调用cmd.Start()启动命令
5. 在启动命令之后，启动一个goroutine，通过cmd.Process.Signal(sig)将信号发送给runc进程
6. 调用processRuncError(err) 处理runc命令的错误，并返回相应的退出码

使用goroutine接收信号并将其转发给命令进程，是为了确保程序能够及时地响应和处理与命令相关的信号，以保证程序的行为符合预期。通过使用goroutine，我们可以在主线程继续等待命令进程结束的同时，异步接收信号并将其转发给命令进程。这样可以确保不会错过任何与命令相关的信号，同时不会阻塞主线程的执行。

```go
func launchRunc(runcPath string, runcArgs []string) int {
	cmd := prepareCommand(runcPath, runcArgs)
	proc := make(chan os.Signal, signalBufferSize)
	// Handle signals before we start command to make sure we don't
	// miss any related to cmd.
	signal.Notify(proc)
	err := cmd.Start()
	if err != nil {
		return exitCodeFailure
	}
	// Forward signals after we start command
	go func() {
		for sig := range proc {
			cmd.Process.Signal(sig)
		}
	}()

	err = cmd.Wait()

	return processRuncError(err)
}
```

prepareCommand函数：

1. 对cmd做一些赋值

```go
func prepareCommand(runcPath string, args []string) *exec.Cmd {
	cmd := exec.Command(runcPath, args...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	return cmd
}
```

processRuncError函数用于处理runc调用的一些错误。

### conclusion

最后来理一下oci-add-hooks的原理，我们知道根据oci runtime-spec的定义，如果想要使用钩子，则需要在bundle的config.json中定义相应的hooks字段。而runc默认情况下为bundle生成的config.json是不含hooks字段的，因此我们需要在bundle的config.json中对hooks字段进行定义。通过在docker的daemon.json中定义runtime为oci-add-hooks，启动容器时通过oci-add-hooks将用户的自定义钩子加入到bundle的config.json中，最后再调用runc作为真正的运行时，从而执行钩子。

## nvidia-container-toolkit



<img src="https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/_images/runtime-architecture.png">

### nvidia container runtime

nvida container runtime和oci-add-hooks类似，同样是runc的wrapper实现。

> This component used to be a complete fork of runC with NVIDIA-specific code injected into it. Since 2019, it is a thin wrapper around the native runC installed on the host system. nvidia-container-runtime takes a runC spec as input, injects the NVIDIA Container Runtime Hook as a prestart hook into it, and then calls out to the native runC, passing it the modified runC spec with that hook set. For versions of the NVIDIA Container Runtime from v1.12.0, this runtime also performs additional modifications to the OCI runtime spec to inject specific devices and mounts not handled by the NVIDIA Container CLI.
>
> It’s important to note that this component is not necessarily specific to docker (but it is specific to runC).

### nvidia container runtime hook

> This component includes an executable that implements the interface required by a runC prestart hook. This script is invoked by runC after a container has been created, but before it has been started, and is given access to the config.json associated with the container. It then takes information contained in the config.json and uses it to invoke the nvidia-container-cli CLI with an appropriate set of flags. One of the most important flags being which specific GPU devices should be injected into the container.