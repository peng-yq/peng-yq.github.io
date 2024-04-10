---
layout: post
title: K8s源码阅读—Kubelet创建容器流程（三）
subtitle: How Kubelet Creates Containers Ⅲ
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

在[K8s源码阅读—Kubelet创建容器流程（二）](https://peng-yq.github.io/2024/04/08/how-kubelet-creates-containers-Ⅱ/)中我们将`kubelet`创建容器的流程推进到了容器运行时创建容器的阶段。

### SyncPod (container runtime)

`SyncPod`负责将当前`Pod`的运行状态同步到期望状态。这个过程包括了一系列的步骤，涉及到计算沙箱（`sandbox`）和容器的变化、杀死不必要的沙箱和容器、创建新的沙箱和容器等。下面是函数的主要步骤和逻辑：

1. **计算沙箱和容器的变化**：首先，通过`computePodActions`函数计算出需要对`Pod`进行哪些操作，比如是否需要创建新的沙箱、哪些容器需要被杀死、哪些容器需要被创建等。
2. **如果必要，杀死`Pod`沙箱**：如果沙箱需要重新创建（例如，因为`Pod`的网络配置发生了变化），则先杀死现有的沙箱。
3. **杀死不应该运行的容器**：根据步骤1的计算结果，杀死那些不应该继续运行的容器。
4. **如果必要，创建沙箱**：如果需要新的沙箱（因为是新`Pod`或者沙箱被杀死了），则创建它。
5. **创建临时（`ephemeral`）容器**：临时容器是一种特殊的容器，它们主要用于调试。
6. **创建初始化（`init`）容器**：初始化容器在`Pod`中的普通容器启动之前运行，用于执行一些初始化任务（`k8s`特有的）。
7. **如果启用了`InPlacePodVerticalScaling`，调整运行中的容器大小**：这涉及到根据`Pod`的资源请求动态调整容器的资源限制。
8. **创建普通容器**：最后，根据`Pod`的定义创建需要启动的普通容器。

`K8s`中的`sandbox`：

> 在`K8s`中，`sandbox`通常指的是容器的运行环境，它为容器提供了一个隔离的环境。在`K8s`的容器运行时接口（`Container Runtime Interface, CRI`）中，`sandbox`是轻量级的容器执行环境，它为一个或多个容器提供网络和其他的操作系统级别的隔离。每个沙箱都运行在它自己的网络命名空间中，这意味着它有自己独立的IP地址和网络配置，从而隔离了不同沙箱中的容器网络。在`K8s`中，`Pod`是最小的部署单元，它可以包含一个或多个容器。`Pod`内的所有容器共享相同的网络命名空间（即，它们都位于同一个`sandbox`中）。这种设计使得`Pod`内的容器能够像在同一台机器上一样相互通信，同时也能从`Pod`外部的网络流量中被隔离开。

```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go
func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// Step 1: Compute sandbox and container changes.
	podContainerChanges := m.computePodActions(ctx, pod, podStatus)
	if podContainerChanges.CreateSandbox {	
        	...
			klog.V(4).InfoS("SyncPod received new pod, will create a sandbox for it", "pod", klog.KObj(pod))
	}
	// Step 2: Kill the pod if the sandbox has changed.
	if podContainerChanges.KillPod {
		if podContainerChanges.CreateSandbox {
			klog.V(4).InfoS("Stopping PodSandbox for pod, will start new one", "pod", klog.KObj(pod))
		} else {
			klog.V(4).InfoS("Stopping PodSandbox for pod, because all other containers are dead", "pod", klog.KObj(pod))
		}
		...	
	} else {
		// Step 3: kill any running containers in this pod which are not to keep.
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {
			klog.V(3).InfoS("Killing unwanted container for pod", "containerName", containerInfo.name, "containerID", containerID, "pod", klog.KObj(pod))
			...
				return
			}
		}
	}
	m.pruneInitContainersBeforeStart(ctx, pod, podStatus)
	var podIPs []string
	if podStatus != nil {
		podIPs = podStatus.IPs
	}
	// Step 4: Create a sandbox for the pod if necessary.
	...
	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		...		
        return nil
	}
	// Step 5: start ephemeral containers
	// These are started "prior" to init containers to allow running ephemeral containers even when there
	// are errors starting an init container. In practice init containers will start first since ephemeral
	// containers cannot be specified on pod creation.
	for _, idx := range podContainerChanges.EphemeralContainersToStart {
		start(ctx, "ephemeral container", metrics.EphemeralContainer, ephemeralContainerStartSpec(&pod.Spec.EphemeralContainers[idx]))
	}
	if !utilfeature.DefaultFeatureGate.Enabled(features.SidecarContainers) {
		// Step 6: start the init container.
		if container := podContainerChanges.NextInitContainerToStart; container != nil {
			// Start the next init container.
			if err := start(ctx, "init container", metrics.InitContainer, containerStartSpec(container)); err != nil {
				return
			}
			// Successfully started the container; clear the entry in the failure
			klog.V(4).InfoS("Completed init container for pod", "containerName", container.Name, "pod", klog.KObj(pod))
		}
	} else {
		// Step 6: start init containers.
		for _, idx := range podContainerChanges.InitContainersToStart {
			container := &pod.Spec.InitContainers[idx]
			// Start the next init container.
		..
		}
	}
	// Step 7: For containers in podContainerChanges.ContainersToUpdate[CPU,Memory] list, invoke UpdateContainerResources
	if isInPlacePodVerticalScalingAllowed(pod) {
		if len(podContainerChanges.ContainersToUpdate) > 0 || podContainerChanges.UpdatePodResources {
			m.doPodResizeAction(pod, podStatus, podContainerChanges, result)
		}
	}
	// Step 8: start containers in podContainerChanges.ContainersToStart.
	for _, idx := range podContainerChanges.ContainersToStart {
		start(ctx, "container", metrics.Container, containerStartSpec(&pod.Spec.Containers[idx]))
	}
	return
}
```

#### computePodActions

`computePodActions`是`syncpod`的第一步，获取需要对该`pod`进行的操作，返回一个`podActions`结构体（包含了需要对`pod`进行的操作）。

```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go
// podActions keeps information what to do for a pod.
type podActions struct {
	// Stop all running (regular, init and ephemeral) containers and the sandbox for the pod.
	KillPod bool
	// Whether need to create a new sandbox. If needed to kill pod and create
	// a new pod sandbox, all init containers need to be purged (i.e., removed).
	CreateSandbox bool
	// The id of existing sandbox. It is used for starting containers in ContainersToStart.
	SandboxID string
	// The attempt number of creating sandboxes for the pod.
	Attempt uint32
	...
}
```

`computePodActions`的工作流程：

1. **检查是否需要创建新的`Pod`沙箱**：如果`Pod`的规格发生了变化，可能需要创建一个新的沙箱。这是因为`Pod`的一些关键属性，如网络配置，是在沙箱级别设置的。如果确定需要创建新的沙箱，那么所有现有的容器都需要被杀死并重建。
2. **处理初始化容器**：如果当前是创建新的沙箱的情况，函数会检查`Pod`的初始化容器。初始化容器在`Pod`的正常容器启动之前运行，用于执行一些启动前的准备工作。如果所有初始化容器都已成功完成，那么函数会继续处理正常容器。如果启用了`SidecarContainers`特性，初始化容器的处理逻辑会有所不同。
3. **处理临时（`Ephemeral`）容器**：即使`Pod`的初始化还没有完成，也可以启动临时容器。临时容器主要用于调试目的，它们不会被重启。
4. **处理正常容器**：这部分逻辑检查每个正常容器的状态，决定是否需要重启容器。如果容器不在运行状态，或者其配置与`Pod`规格不符，或者未通过存活检查（`liveness probe`），则会被计划重启。
5. **处理`Pod`垂直扩缩容**：如果允许`Pod`进行垂直扩缩容（即在不重启`Pod`的情况下改变容器的资源限制），这部分逻辑会检查是否需要更新容器的资源配置。
6. **处理结束**：最后，函数根据上述检查和决策构建`podActions`结构体并返回。这个结构体包含了所有需要执行的操作，如是否需要杀死`Pod`、创建新的沙箱、启动或重启哪些容器等。

```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go
// computePodActions checks whether the pod spec has changed and returns the changes if true.
func (m *kubeGenericRuntimeManager) computePodActions(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus) podActions {
    createPodSandbox, attempt, sandboxID := runtimeutil.PodSandboxChanged(pod, podStatus)
	changes := podActions{
		KillPod:           createPodSandbox,
		CreateSandbox:     createPodSandbox,
		SandboxID:         sandboxID,
		Attempt:           attempt,
		ContainersToStart: []int{},
		ContainersToKill:  make(map[kubecontainer.ContainerID]containerToKillInfo),
	}
	....
    return changes
}
```

#### CreateSandbox

前面提到`sandbox`提供了一个隔离环境，使得同一个`pod`中的容器可以相互通信，创建`sandbox`的步骤如下：

1. **创建`Pod`沙箱**：这是容器运行时环境的一部分，为Pod中的容器提供隔离环境。代码首先检查`podContainerChanges.CreateSandbox`标志，如果为`true`，则进行沙箱创建。创建沙箱之前，会对Pod的安全上下文中的系统调用参数（`sysctls`）进行格式转换，以满足运行时（如`runc`）的要求。在创建沙箱的过程中，如果启用了动态资源分配特性，还会准备相应的资源。如果沙箱创建成功，会获取沙箱的状态，并可能更新`Pod`的`IP`地址。
2. **处理`Pod IP`地址**：在创建沙箱后，会根据沙箱状态更新`Pod`的`IP`地址。这是因为新创建的沙箱可能会有新的网络配置和`IP`地址。
3. **生成`Pod`沙箱配置**：接下来，根据`Pod`的定义和尝试次数生成`Pod`沙箱的配置。这个配置将用于启动Pod中的容器。
4. **启动容器**：最后，定义了一个`start`函数，用于启动Pod中的各种类型的容器（如正常容器、初始化容器、临时容器等）。在启动容器之前，会检查是否处于回退状态（例如，由于重试失败），并更新相应的监控指标。启动容器时，会传入`Pod`的`IP`地址和沙箱配置。如果容器启动失败，会记录错误并更新监控指标。

首先看一下`createPodSandbox`的代码，主要流程为生成沙箱配置、创建日志目录、启动沙箱并返回沙箱`ID`。

```go
// pkg/kubelet/kubeletruntime/kubeletruntime_sandbox.go
// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(ctx context.Context, pod *v1.Pod, attempt uint32) (string, string, error) {
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	if err != nil {
		message := fmt.Sprintf("Failed to generate sandbox config for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to generate sandbox config for pod", "pod", klog.KObj(pod))
		return "", message, err
	}
	// Create pod logs directory
	err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
	if err != nil {
		message := fmt.Sprintf("Failed to create log directory for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to create log directory for pod", "pod", klog.KObj(pod))
		return "", message, err
	}
	runtimeHandler := ""
	if m.runtimeClassManager != nil {
		runtimeHandler, err = m.runtimeClassManager.LookupRuntimeHandler(pod.Spec.RuntimeClassName)
		if err != nil {
			message := fmt.Sprintf("Failed to create sandbox for pod %q: %v", format.Pod(pod), err)
			return "", message, err
		}
		if runtimeHandler != "" {
			klog.V(2).InfoS("Running pod with runtime handler", "pod", klog.KObj(pod), "runtimeHandler", runtimeHandler)
		}
	}
	podSandBoxID, err := m.runtimeService.RunPodSandbox(ctx, podSandboxConfig, runtimeHandler)
	if err != nil {
		message := fmt.Sprintf("Failed to create sandbox for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to create sandbox for pod", "pod", klog.KObj(pod))
		return "", message, err
	}
	return podSandBoxID, "", nil
}
```

#### StartContainer

`SyncPod`函数中编写了一个名为`start`的匿名函数，用于启动容器（`"container", "init container" or "ephemeral container"`）。`start`封装了启动容器时的共通步骤，包括记录启动结果、处理启动失败的情况、更新相关指标和日志记录等。

1. **初始化启动结果**: 创建一个新的同步结果`startContainerResult`，用于记录容器的启动状态。
2. **判断退避策略**: 通过`doBackOff`函数检查是否需要对容器启动进行退避（例如，由于之前的启动失败）。如果需要退避，则记录失败结果并返回错误。
3. **更新指标**: 增加`StartedContainersTotal`指标的计数，表示尝试启动了一个容器。如果是`Windows`宿主进程容器，还会增加`StartedHostProcessContainersTotal`指标的计数。
4. **日志记录**: 记录即将创建容器的日志信息。
5. **启动容器**: 调用`startContainer`函数尝试启动容器。如果启动失败，会更新`StartedContainersErrorsTotal`指标，记录失败原因，并根据错误类型进行不同级别的日志记录。

```go
	// Helper containing boilerplate common to starting all types of containers.
	// typeName is a description used to describe this type of container in log messages,
	// currently: "container", "init container" or "ephemeral container"
	// metricLabel is the label used to describe this type of container in monitoring metrics.
	// currently: "container", "init_container" or "ephemeral_container"
	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, spec.container.Name)
		result.AddSyncResult(startContainerResult)
		isInBackOff, msg, err := m.doBackOff(pod, spec.container, podStatus, backOff)
		if isInBackOff {
			startContainerResult.Fail(err, msg)
			klog.V(4).InfoS("Backing Off restarting container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
			return err
		}
		metrics.StartedContainersTotal.WithLabelValues(metricLabel).Inc()
		if sc.HasWindowsHostProcessRequest(pod, spec.container) {
			metrics.StartedHostProcessContainersTotal.WithLabelValues(metricLabel).Inc()
		}
		klog.V(4).InfoS("Creating container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
		// NOTE (aramase) podIPs are populated for single stack and dual stack clusters. Send only podIPs.
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			...
			return err
		}
		return nil
	}
```

`start`匿名函数的关键是调用了`startContainer`函数来启动容器：

- **拉取镜像**

  - 首先，判断是否启用了`RuntimeClassInImageCriAPI`特性门控。如果启用且指定了运行时类，则尝试获取对应的`runtimeHandler`。如果无法获取，则记录错误并返回。

  - 使用`m.imagePuller.EnsureImageExists`函数确保所需的镜像存在。这个函数会尝试拉取镜像，如果镜像拉取失败，则记录事件并返回错误。

- **创建容器**

  - 计算重启次数`restartCount`。如果容器之前已经存在，则其重启次数应该是之前的重启次数加1。如果是新容器，或者在节点重启后容器运行时的状态被清除，重启次数为0。

  - 通过`spec.getTargetID`获取目标ID，用于`ephemeral container`。

  - 调用`m.generateContainerConfig`生成容器配置。这个过程中可能会进行一些清理操作，所以定义了`defer cleanupAction()`以确保在函数返回前执行清理。

  - 在创建容器前，调用`m.internalLifecycle.PreCreateContainer`执行预创建钩子。如果钩子执行失败，记录事件并返回错误。

  - 使用`m.runtimeService.CreateContainer`创建容器。如果创建失败，记录事件并返回错误。

- **启动容器**

  - 在容器创建后，调用`m.internalLifecycle.PreStartContainer`执行启动前钩子。如果钩子执行失败，记录事件并返回错误。

  - 使用`m.runtimeService.StartContainer`启动容器。如果启动失败，记录事件并返回错误。

  - 记录容器启动事件。

  - 为了支持集群日志记录，创建从容器日志到传统容器日志位置的符号链接。这是为了兼容旧版本。

- **执行启动后的生命周期钩子**

  - 如果容器配置了启动后的生命周期钩子，则执行这个钩子。

  - 如果钩子执行失败，记录错误日志，并尝试杀死容器以清理资源。

```go
func (m *kubeGenericRuntimeManager) startContainer(ctx context.Context, podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, spec *startSpec, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
	container := spec.container // 包含了镜像、名称、命令、环境变量等容器元数据
	// Step 1: pull the image.
	podRuntimeHandler := ""
	var err error
    // 获取对应的runtimehandler
	if utilfeature.DefaultFeatureGate.Enabled(features.RuntimeClassInImageCriAPI) {
		...
		}
	}
	// 确保容器启动的镜像存在
	imageRef, msg, err := m.imagePuller.EnsureImageExists(ctx, pod, container, pullSecrets, podSandboxConfig, podRuntimeHandler)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return msg, err
	}
	// Step 2: create the container.
	// For a new container, the RestartCount should be 0，更新一下容器的restartcount
	restartCount := 0
	containerStatus := podStatus.FindContainerStatusByName(container.Name)
	if containerStatus != nil {
		restartCount = containerStatus.RestartCount + 1
	} else {
		...
		logDir := BuildContainerLogsDirectory(pod.Namespace, pod.Name, pod.UID, container.Name)
		restartCount, err = calcRestartCountByLogDir(logDir)
		if err != nil {
			klog.InfoS("Cannot calculate restartCount from the log directory", "logDir", logDir, "err", err)
			restartCount = 0
		}
	}

	target, err := spec.getTargetID(podStatus)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return s.Message(), ErrCreateContainerConfig
	}
	// 生成容器的配置信息
	containerConfig, cleanupAction, err := m.generateContainerConfig(ctx, container, pod, restartCount, podIP, imageRef, podIPs, target)
	...
	// 设置容器的一些cpu和内存等信息
	err = m.internalLifecycle.PreCreateContainer(pod, container, containerConfig)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Internal PreCreateContainer hook failed: %v", s.Message())
		return s.Message(), ErrPreCreateHook
	}
	// 调用远程服务（容器运行时，CRI）来创建容器，实际是通过runtimeService.startContainer()	
	containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)
	...
	err = m.internalLifecycle.PreStartContainer(pod, container, containerID)
	...
	m.recordContainerEvent(pod, container, containerID, v1.EventTypeNormal, events.CreatedContainer, fmt.Sprintf("Created container %s", container.Name))

	// Step 3: start the container.
	err = m.runtimeService.StartContainer(ctx, containerID)
	...

	// Step 4: execute the post start hook.
	if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
    ...

	return "", nil
}
```



