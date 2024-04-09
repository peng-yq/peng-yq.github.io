---
layout: post
title: K8s源码阅读—Kubelet创建容器流程（二）
subtitle: How Kubelet Creates Containers Ⅱ
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

在[K8s源码阅读—Kubelet创建容器流程（一）](https://peng-yq.github.io/2024/04/08/how-kubelet-creates-containers-I/)中对`kubelet`的启动流程进行了分析，在最终的入口函数中，`kubelet`通过`kl.syncLoop(ctx, updates, kl)`监听来自`file/apiserver/http `事件源发送过来的事件，并对事件做出对应的同步处理。

> 通过上述`3`种事件源创建的`pod`亦有区别，可分为静态`pod`（`static pod`）和镜像`pod`（`mirror pod`）：
>
> 静态`Pod`不通过`master`节点上的`apiserver`操作及管理，直接由特定节点上的`kubelet`进程来管理的。无法与我们常用的控制器`Deployment`或者`DaemonSet`等进行关联，由`kubelet`进程自己来监控，当`pod`崩溃时重启该`pod` ，`kubelet`也无法对他们进行健康检查。静态`pod`始终绑定在某个`kubelet`，并且始终运行在同一个节点上。`kubelet`会自动为每一个静态`pod`在`apiserver`上创建一个镜像`pod`，因此我们可以在`apiserver`中查询到该`pod`，但是不能通过`apiserver`进行控制（例如不能删除）。
>
> 创建静态`pod`有两种方式：配置文件和`HTTP`两种方式。

### List/watch

`kubelet`监听各个事件源主要是通过`list/watch`机制：在`K8s`中，`list/watch`机制是一种高效的数据同步方式，尤其在与`apiserver`交互时。这种机制广泛应用于`K8s`的各个组件之间，例如kubelet等，以便它们能够实时地跟踪集群状态的变化。

- `List`：`List`操作允许一个客户端（例如`kubelet`）从`k8s-apiserver`获取资源的当前状态。`list`操作会返回一个资源的列表，例如集群中所有的`Pod`、`Service`、`Deployment`等。`List`操作通常在客户端首次启动时执行，以初始化其本地状态。

- `Watch`：一旦客户端通过`List`操作获取了资源的当前快照，它就可以开始执行`Watch`操作。`Watch`操作允许客户端监听之后对指定资源的任何更改（包括增加、修改和删除）。当这些更改发生时，`api-server`会将更改的通知发送给所有正在监听该资源的客户端。

`list/watch`机制的工作步骤如下：

1. 初始化： 客户端首先执行`List`操作获取资源的当前状态，并基于这个列表初始化自己的本地状态。
2. 监听更改： 客户端随后执行`Watch`操作，从它执行`List`操作的那一刻开始监听资源的更改。
3. 处理事件： 当资源发生更改时，`apiserver`会将更改事件发送给所有监听该资源的客户端。客户端接收到这些事件后，会根据事件操作更新其本地状态。
4. 重新同步： 如果由于网络问题或其他原因导致`watch`连接断开，客户端可能会重新执行`list`操作来获取最新的资源状态，并重新建立`watch`监听。

### SyncLoop

**使用`list/watch`机制的优势在于其高效性。客户端不需要定期轮询`apiserver`来检查资源是否有变化，这大大减少了网络流量和`apiserver`的负载。相反，客户端只会在实际发生更改时接收更新，使得数据同步更加即时和高效。**

`syncLoop`负责监听和处理来自不同来源的`Pod`更新，通过一个无限循环持续运行以保证`Kubelet`能够响应`Pod`的变化和状态同步。

```go
// pkg/kubelet/kubelet.go
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
    // 获取了PLEG（Pod Lifecycle Event Generator）的事件通道，负责监视Pod生命周期事件，如创建、启动、停止等，并将这些事件通知给Kubelet。
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
		for {
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.ErrorS(err, "Skipping pod synchronization")
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base
		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

`kl.syncLoopIteration`函数被调用来处理实际的同步逻辑，包括处理来自不同来源的`Pod`更新（`file/apiserver/http `），执行`Pod`的创建、更新或删除操作。这个函数会根据需要调用`handler`处理每个`Pod`更新。

```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(ctx context.Context, configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	// 使用select语句监听上述通道，根据接收到的事件类型，调用不同的处理逻辑；select语句中的case是伪随机选择的，如果有多个通道同时准备好，选择哪个通道处理是随机的。
    select {
    // 处理来自配置源的更新事件
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
			return false
		}
		// 通过switch case根据不同的事件操作调用对应的handler函数进行处理
		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.ErrorS(nil, "Kubelet does not support snapshot update")
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		kl.sourcesReady.AddSource(u.Source)
	// 处理PLEG生成的Pod生命周期事件。
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
    // 触发周期性的Pod同步。
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjSlice(podsToSync))
		handler.HandlePodSyncs(podsToSync)
    // 监听来自健康检查管理器（如存活探针、就绪探针和启动探针）的更新
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
    // 触发清理任务
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
		} else {
			start := time.Now()
			klog.V(4).InfoS("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(ctx); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
			duration := time.Since(start)
			if duration > housekeepingWarningDuration {
				klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than expected", "expected", housekeepingWarningDuration, "actual", duration.Round(time.Millisecond))
			}
			klog.V(4).InfoS("SyncLoop (housekeeping) end", "duration", duration.Round(time.Millisecond))
		}
	}
	return true
}
```

### Handler

下面重点来看下对配置源`configCh`事件的处理逻辑，对于`ADD`操作，`kubelet`会调用`handler.HandlePodAdditions(u.Pods)`来添加`Pod`。

```go
// pkg/kubelet/kubelet.go
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
    // 按照pod的创建时间对这个列表进行排序，确保处理顺序从旧到新。
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		kl.podResizeMutex.Lock()
		defer kl.podResizeMutex.Unlock()
	}
	for _, pod := range pods {
        // 获取当前节点上所有pod
		existingPods := kl.podManager.GetPods()
		kl.podManager.AddPod(pod) // 添加pod至podmanager
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		// mirrorpod的处理
        if wasMirror {
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}
		// 检查pod是否被请求中止
		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			......
		}
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodCreate,
			StartTime:  start,
		})
	}
}
```

对于成功通过准入控制的`pod`，会通过 `podWorkers` 更新 `pod` 的状态，启动或更新容器的执行流程。

#### UpdatePod

`UpdatePod`的代码居多，巨长，巨复杂。`p.podWorkerLoop`给每个`pod`都创建一个`goroutine` ，用于监听`pod`的更新动作并做出同步。每次更新实际都是将`pod`的相关信息发送到`podUpdates`通道，通道的另一端就是`p.podWorkerLoop`在监听。

```go
// pkg/kubelet/pod_workers.go
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
	.....
    // 如果它不存在，启动Pod工作器goroutine
    podUpdates, exists := p.podUpdates[uid] // uid是pod的id
    if !exists {
        // 缓冲通道以避免阻塞此方法
        podUpdates = make(chan struct{}, 1)
        p.podUpdates[uid] = podUpdates
        // 确保静态Pod按照它们被UpdatePod接收的顺序启动
        if kubetypes.IsStaticPod(pod) {
            p.waitingToStartStaticPodsByFullname[status.fullname] =
                append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
        }
        // 允许测试Pod更新通道中的延迟
        var outCh <-chan struct{}
        if p.workerChannelFn != nil {
            outCh = p.workerChannelFn(uid, podUpdates)
        } else {
            outCh = podUpdates
        }
        // 生成一个Pod工作器
        go func() {
            defer runtime.HandleCrash()
            defer klog.V(3).InfoS("Pod worker has stopped", "podUID", uid)
            p.podWorkerLoop(uid, outCh)
        }()
    }
	...
    // 通知Pod工作器有一个待处理的更新
    status.pendingUpdate = &options
    status.working = true
    klog.V(4).InfoS("Notifying pod of pending update", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
    select {
    case podUpdates <- struct{}{}:
    default:
    }
    if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
        klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
        status.cancelFn()
        return
    }
}
```

#### podWorkerLoop

`podWorkerLoop`从`podUpdates`队列中根据`uid`获取`pod`的状态信息，并根据`update.WorkType`的不同调用不同的接口来同步`pod `，除开已终止的和终止中的，统一使用`SyncPod`同步`Pod`。

```go
func (p *podWorkers) podWorkerLoop(podUID types.UID, podUpdates <-chan struct{}) {
	var lastSyncTime time.Time
	for range podUpdates {
		// 从队列中获取待处理的Pod更新事件
		ctx, update, canStart, canEverStart, ok := p.startPodSync(podUID)
		if !ok {
			// 如果没有等待的更新，意味着初始化通道但没有填充pendingUpdate
			continue
		}
		if !canEverStart {
			// 如果Pod在允许启动之前已被终止，退出循环
			return
		}
		if !canStart {
			// 如果Pod还未准备好启动，继续等待更多更新
			continue
		}
		podUID, podRef := podUIDAndRefForUpdate(update.Options)
		var isTerminal bool
		err := func() error {
			var status *kubecontainer.PodStatus
			var err error
			switch {
			case update.Options.RunningPod != nil:
				// 当我们接收到一个正在运行的Pod时，我们不需要状态，因为我们保证正在终止，并且跳过更新Pod
			default:
				// 等待从PLEG通过缓存看到下一个刷新（最多2秒）
				status, err = p.podCache.GetNewerThan(update.Options.Pod.UID, lastSyncTime)
				if err != nil {
					p.recorder.Eventf(update.Options.Pod, v1.EventTypeWarning, events.FailedSync, "error determining status: %v", err)
					return err
				}
			}
			// 根据工作类型采取适当的行动
			switch {
			case update.WorkType == TerminatedPod:
				// 同步已终止的Pod
				err = p.podSyncer.SyncTerminatedPod(ctx, update.Options.Pod, status)
			case update.WorkType == TerminatingPod:
				// 同步终止中的Pod
				var gracePeriod *int64
				if opt := update.Options.KillPodOptions; opt != nil {
					gracePeriod = opt.PodTerminationGracePeriodSecondsOverride
				}
				podStatusFn := p.acknowledgeTerminating(podUID)
				if update.Options.RunningPod != nil {
					err = p.podSyncer.SyncTerminatingRuntimePod(ctx, update.Options.RunningPod)
				} else {
					err = p.podSyncer.SyncTerminatingPod(ctx, update.Options.Pod, status, gracePeriod, podStatusFn)
				}
			default:
				// 除开已终止的和终止中的，统一使用SyncPod同步Pod
				isTerminal, err = p.podSyncer.SyncPod(ctx, update.Options.UpdateType, update.Options.Pod, update.Options.MirrorPod, status)
			}
			lastSyncTime = p.clock.Now()
			return err
		}()
		...
	}
}
```

#### SyncPod

`SyncPod`函数是单个Pod同步（设置）的事务脚本，旨在将`Pod`的运行时状态与其期望的配置状态（即`Pod`正在运行）对齐。该方法是可重入的，并期望使`Pod`逐步达到其规范的期望状态。`Pod`的反向操作（拆除）通过`SyncTerminatingPod`和`SyncTerminatedPod`处理。如果`SyncPod`正常退出，则表示`Pod`的运行时状态与期望的配置状态同步。如果`SyncPod`因为短暂错误退出，下一次调用`SyncPod`时，预期将向达到期望状态迈进。当`Pod`因容器退出（对于`RestartNever`或`RestartOnFailure`）到达终止生命周期阶段时，`SyncPod`将以`isTerminal`退出，下一个调用的方法将是`SyncTerminatingPod`。如果`Pod`因任何其他原因终止，`SyncPod`将接收到一个上下文取消信号，并应尽快退出。

> 如果一个函数是可重入的，那么它就可以在执行的任何时刻被中断，转而去执行另一个任务，稍后再恢复执行，而不会丢失状态、数据不一致或产生其他错误。可重入函数的执行不会被其他的函数调用所干扰，即使多个任务并发执行同一个可重入函数，也不会影响各自的执行结果。

参数包括：

- `updateType`：标识这是首次创建还是更新，仅用于度量，因为此方法必须是可重入的。
- `pod`：正在设置的`Pod`。
- `mirrorPod`：此Pod在kubelet中已知的镜像Pod（如果有）。
- `podStatus`：此Pod最近观察到的状态，可用于决定此次`SyncPod`循环中应采取的行动。

工作流程如下：

1. 如果`Pod`正在创建，记录`Pod`工作器启动延迟。
2. 调用`generateAPIPodStatus`为`Pod`准备一个`v1.PodStatus`。
3. 如果`Pod`首次被视为运行，记录`Pod`启动延迟。
4. 在状态管理器中更新`Pod`的状态。
5. 如果由于软性准入而不应运行，则停止`Pod`的容器。
6. 确保为可运行的`Pod`启动任何后台跟踪。
7. 如果`Pod`是静态`Pod`且尚未有镜像`Pod`，则创建一个镜像`Pod`。
8. 如果`Pod`的数据目录不存在，则创建它们。
9. 等待卷附加/挂载。
10. 获取`Pod`的拉取密钥。
11. 调用容器运行时的`SyncPod`回调。
12. 更新`Pod`的入口和出口流量限制。

如果工作流程的任何步骤出现错误，将返回该错误，并在下一次`SyncPod`调用时重复。

```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	ctx, otelSpan := kl.tracer.Start(ctx, "syncPod", trace.WithAttributes(
		semconv.K8SPodUIDKey.String(string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		semconv.K8SPodNameKey.String(pod.Name),
		attribute.String("k8s.pod.update_type", updateType.String()),
		semconv.K8SNamespaceNameKey.String(pod.Namespace),
	))
	klog.V(4).InfoS("SyncPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer func() {
		klog.V(4).InfoS("SyncPod exit", "pod", klog.KObj(pod), "podUID", pod.UID, "isTerminal", isTerminal)
		otelSpan.End()
	}()
	// Latency measurements for the main workflow are relative to the
	// first time the pod was seen by kubelet.
	var firstSeenTime time.Time
	if firstSeenTimeStr, ok := pod.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]; ok {
		firstSeenTime = kubetypes.ConvertToTimestamp(firstSeenTimeStr).Get()
	}
	// Record pod worker start latency if being created
	// TODO: make pod workers record their own latencies
	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// This is the first time we are syncing the pod. Record the latency
			// since kubelet first saw the pod if firstSeenTime is set.
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
		} else {
			klog.V(3).InfoS("First seen time not recorded for pod",
				"podUID", pod.UID,
				"pod", klog.KObj(pod))
		}
	}
	// note：记录延迟用于性能优化
	// Generate final API pod status with pod and status manager status
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. (See #24576)
	// TODO(random-liu): After writing pod spec into container labels, check whether pod is using host network, and
	// set pod IP to hostIP directly in runtime.GetPodStatus
	// note：生成api pod状态
	podStatus.IPs = make([]string, 0, len(apiPodStatus.PodIPs))
	for _, ipInfo := range apiPodStatus.PodIPs {
		podStatus.IPs = append(podStatus.IPs, ipInfo.IP)
	}
	if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
		podStatus.IPs = []string{apiPodStatus.PodIP}
	} // note：处理pod ip地址
	// If the pod is terminal, we don't need to continue to setup the pod
	if apiPodStatus.Phase == v1.PodSucceeded || apiPodStatus.Phase == v1.PodFailed {
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		isTerminal = true
		return isTerminal, nil
	} // note：pod成功或失败退出表示生命周期结束，设置isTerminal为true，不会对其进行进一步调度

	// If the pod should not be running, we request the pod's containers be stopped. This is not the same
	// as termination (we want to stop the pod, but potentially restart it later if soft admission allows
	// it later). Set the status and phase appropriately
	// note：首先判断这个pod是否应该运行
	runnable := kl.canRunPod(pod)
	if !runnable.Admit {
		// Pod is not runnable; and update the Pod and Container statuses to why.
		// note：pod不应该运行，则更新pod状态（等待被调度或不能运行）
		if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
			apiPodStatus.Phase = v1.PodPending
		}
		// note：设置原因和一些关键信息
		apiPodStatus.Reason = runnable.Reason
		apiPodStatus.Message = runnable.Message
		// Waiting containers are not creating.
		// note：对于Pod中的所有初始化容器和普通容器，如果它们的状态是等待（Waiting），则将等待原因设置为"Blocked"。
		// 这表示这些容器因为某些原因被阻塞了，无法创建或启动。
		const waitingReason = "Blocked"
		for _, cs := range apiPodStatus.InitContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
		for _, cs := range apiPodStatus.ContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
	}

	// Record the time it takes for the pod to become running
	// since kubelet first saw the pod if firstSeenTime is set.
	// note：记录pod启动的耗时，用于监控pod启动性能和调度效率
	existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
		!firstSeenTime.IsZero() {
		metrics.PodStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
	}

	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// Pods that are not runnable must be stopped - return a typed error to the pod worker
	// note：调用runtime来关闭pod中的所有容器
	if !runnable.Admit {
		klog.V(2).InfoS("Pod is not runnable and must have running containers stopped", "pod", klog.KObj(pod), "podUID", pod.UID, "message", runnable.Message)
		var syncErr error
		p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err := kl.killPod(ctx, pod, p, nil); err != nil {
			if !wait.Interrupted(err) {
				kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
				syncErr = fmt.Errorf("error killing pod: %w", err)
				utilruntime.HandleError(syncErr)
			}
		} else {
			// There was no error killing the pod, but the pod cannot be run.
			// Return an error to signal that the sync loop should back off.
			syncErr = fmt.Errorf("pod cannot be run: %v", runnable.Message)
		}
		return false, syncErr
	}

	// If the network plugin is not ready, only start the pod if it uses the host network
	// note：下面是网络插件状态检查和资源注册
	...
    // Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	// note：创建pod容器管理器，用于管理pod的cgroups
	pcm := kl.containerManager.NewPodContainerManager()
	// If pod has already been terminated then we need not create
	// or update the pod's cgroup
	// note：如果pod请求终止，不创建和更新cgroups
	// TODO: once context cancellation is added this check can be removed
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// When the kubelet is restarted with the cgroups-per-qos
		// flag enabled, all the pod's running containers
		// should be killed intermittently and brought back up
		// under the qos cgroup hierarchy.
		// Check if this is the pod's first sync
		// note：查看该pod是否启动了
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		// Don't kill containers in pod if pod's cgroups already
		// exists or the pod is running for the first time
		// note：cgroup存在并且是首次启动则不杀死容器
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
			if err := kl.killPod(ctx, pod, p, nil); err == nil {
				if wait.Interrupted(err) {
					return false, err
				}
				podKilled = true
			} else {
				klog.ErrorS(err, "KillPod failed", "pod", klog.KObj(pod), "podStatus", podStatus)
			}
		}
		// note： QoS Cgroups（Quality of Service Control Groups）是Kubernetes中用于实现资源管理和隔离的一种机制，
		// 它基于Linux的Cgroups（控制组）功能。Cgroups允许Linux内核限制、记录和隔离进程组使用的物理资源（如CPU、内存等）。
		// 在Kubernetes中，QoS Cgroups用于根据Pod的服务质量（Quality of Service，QoS）类别来管理和隔离Pod的资源使用，
		// 确保关键任务的资源分配优先级高于其他较不重要的任务。
		// 有三种级别
		// Create and Update pod's Cgroups
		// Don't create cgroups for run once pod if it was killed above
		// The current policy is not to restart the run once pods when
		// the kubelet is restarted with the new flag as run once pods are
		// expected to run only once and if the kubelet is restarted then
		// they are not expected to run again.
		// We don't create and apply updates to cgroup if its a run once pod and was killed above
		// note：如果Pod被杀死并且Pod的重启策略为Never（只运行一次的Pod），则不创建或更新Cgroups。
		// 如果Pod的Cgroups不存在，先尝试更新QoS（服务质量）Cgroups，然后确保Pod的Cgroups存在。
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					klog.V(2).InfoS("Failed to update QoS cgroups while syncing pod", "pod", klog.KObj(pod), "err", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
					return false, fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	// note：为了让直接由kubelet启动的pod在集群中可见，为其创建静态pod
	if kubetypes.IsStaticPod(pod) {
		deleted := false
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kubepod.IsMirrorPodOf(mirrorPod, pod) {
				// The mirror pod is semantically different from the static pod. Remove
				// it. The mirror pod will get recreated later.
				// note：删除被标记为删除或语义与静态pod不同的镜像pod
				klog.InfoS("Trying to delete pod", "pod", klog.KObj(pod), "podUID", mirrorPod.ObjectMeta.UID)
				podFullName := kubecontainer.GetPodFullName(pod)
				var err error
				deleted, err = kl.mirrorPodClient.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
				if deleted {
					klog.InfoS("Deleted mirror pod because it is outdated", "pod", klog.KObj(mirrorPod))
				} else if err != nil {
					klog.ErrorS(err, "Failed deleting mirror pod", "pod", klog.KObj(mirrorPod))
				}
			}
		}
		// note： 创建静态pod
		if mirrorPod == nil || deleted {
			node, err := kl.GetNode()
			if err != nil {
				klog.V(4).ErrorS(err, "No need to create a mirror pod, since failed to get node info from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else if node.DeletionTimestamp != nil {
				klog.V(4).InfoS("No need to create a mirror pod, since node has been removed from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else {
				klog.V(4).InfoS("Creating a mirror pod for static pod", "pod", klog.KObj(pod))
				if err := kl.mirrorPodClient.CreateMirrorPod(pod); err != nil {
					klog.ErrorS(err, "Failed creating a mirror pod for", "pod", klog.KObj(pod))
				}
			}
		}
	}

	// Make data directories for the pod
	// note：为pod创建相关的目录：pod的目录，卷和插件的目录
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.ErrorS(err, "Unable to make pod data directories for pod", "pod", klog.KObj(pod))
		return false, err
	}

	// Wait for volumes to attach/mount
	if err := kl.volumeManager.WaitForAttachAndMount(ctx, pod); err != nil {
		if !wait.Interrupted(err) {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.ErrorS(err, "Unable to attach or mount volumes for pod; skipping pod", "pod", klog.KObj(pod))
		}
		return false, err
	}

	// Fetch the pull secrets for the pod
	pullSecrets := kl.getPullSecretsForPod(pod)

	// Ensure the pod is being probed
	kl.probeManager.AddPod(pod)

	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		// Handle pod resize here instead of doing it in HandlePodUpdates because
		// this conveniently retries any Deferred resize requests
		// TODO(vinaykul,InPlacePodVerticalScaling): Investigate doing this in HandlePodUpdates + periodic SyncLoop scan
		//     See: https://github.com/kubernetes/kubernetes/pull/102884#discussion_r663160060
		if kl.podWorkers.CouldHaveRunningContainers(pod.UID) && !kubetypes.IsStaticPod(pod) {
			pod = kl.handlePodResourcesResize(pod)
		}
	}
	sctx := context.WithoutCancel(ctx)
	// note：创建容器
	result := kl.containerRuntime.SyncPod(sctx, pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		// Do not return error if the only failures were pods in backoff
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				// Do not record an event here, as we keep all event logging for sync pod failures
				// local to container runtime, so we get better errors.
				return false, err
			}
		}

		return false, nil
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && isPodResizeInProgress(pod, &apiPodStatus) {
		// While resize is in progress, periodically call PLEG to update pod cache
		runningPod := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err, _ := kl.pleg.UpdateCache(&runningPod, pod.UID); err != nil {
			klog.ErrorS(err, "Failed to update pod cache", "pod", klog.KObj(pod))
			return false, err
		}
	}

	return false, nil
}
```

对于本篇而言，`syncPod`函数中最重要的步骤为`result := kl.containerRuntime.SyncPod(sctx, pod, podStatus, pullSecrets, kl.backOff)`，终于到了容器运行时这一部分了。
