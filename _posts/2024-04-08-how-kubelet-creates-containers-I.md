---
layout: post
title: K8s源码阅读—Kubelet创建容器流程（一）
subtitle: How Kubelet Creates Containers I
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

当我们在`k8s master`节点发起创建`pod/deployment`时，需要经过很多步骤，这里只记录步骤`6-9`：

1. **提交请求**：使用`kubectl`命令或`Kubernetes `API提交创建`Pod`或`Deployment`的请求时，该请求首先会发送到`K8s-apiserver`。
2. **API服务器验证与授权**：`k8s-apiserver`会验证请求的合法性（例如，检查请求的格式是否正确）并进行授权检查，确保用户或服务有权执行该操作。
3. **存储到etcd**：一旦请求通过验证和授权，`k8s-apiserver`会将`pod/deployment`的定义存储到`etcd`中。
4. **调度**：创建`pod/deployment`的请求存储到`etcd`后，`k8s-scheduler`会被通知有新的`pod/deployment`需要调度。调度器会根据`pod/deployment`的需求（如资源需求、亲和性规则等）和集群的当前状态（如各节点的资源利用率）来选择一个最合适的节点来运行该`pod/deployment`。
5. **调度决策更新到etcd**：一旦`k8s-scheduler`做出决策，它会将`pod/deployment`应该运行在哪个节点的信息更新到`etcd`中。
6. **Kubelet响应**：`Kubelet`会定期从`k8s-apiserver`查询它负责的`pod/deployment`信息。当`Kubelet`发现有新的`pod/deployment`需要在其节点上运行时，它会开始准备创建`pod/deployment`的容器。
7. **容器运行时拉取镜像**：如果`pod/deployment`的容器使用的镜像不在节点上，`Kubelet`会指示容器运行时（如`Docker`、`containerd`等）拉取镜像。
8. **创建容器**：镜像拉取完成后，`Kubelet`会创建容器。这包括设置网络、挂载存储卷等必要的准备工作。
9. **启动容器**：容器创建完成后，`Kubelet`会启动容器。如果`Pod`中有多个容器，`Kubelet`会根据`Pod`的定义顺序依次启动它们。
10. **健康检查**：容器启动后，`Kubele`t会执行定义在`Pod`中的健康检查（如果有的话）。只有当所有的检查都通过时，`Pod`才被视为健康的。
11. **注册服务和负载均衡**：如果`Pod`是某个服务的一部分，`K8s`会更新内部的服务和负载均衡器，将新的`Pod`纳入到服务中。
12. **监控和日志**：一旦`Pod`运行起来，`Kubelet`会持续监控其状态，并将日志和指标上报给集群的监控系统。

<img src="https://arthurchiao.art/assets/img/what-happens-when-k8s-creates-pods/kube-scheduler.svg">

阅读的源码为[K8s-release-1.29分支](https://github.com/kubernetes/kubernetes/tree/release-1.29)，`Kubelet`代码十分庞大并且精细，这里只对创建容器的相关流程进行简单记录（忽略了很多细节，只记录函数调用流程，具体细节需参考源码），太复杂了所以记录的比较乱。

参考的一些博客：

1. [源码解析：K8s 创建 pod 时，背后发生了什么（版本比较老了）](https://arthurchiao.art/blog/what-happens-when-k8s-creates-pods-1-zh/)
2. [kubernetes源码-kubelet 原理和源码分析（版本相对较新）](https://isekiro.com/kubernetes%E6%BA%90%E7%A0%81-kubelet-%E5%8E%9F%E7%90%86%E5%92%8C%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%80/)

### 关于kubelet

`kubelet`工具中的描述：

```shell
The kubelet is the primary "node agent" that runs on each node. It can register the node with the apiserver using one of: the hostname; a flag to override the hostname; or specific logic for a cloud provider.

The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object that describes a pod. The kubelet takes a set of PodSpecs that are provided through various mechanisms (primarily through the apiserver) and ensures that the containers described in those PodSpecs are running and healthy. The kubelet doesn't manage containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are two ways that a container manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. Files under this path will be monitored periodically for updates. The monitoring period is 20s by default and is configurable via a flag.

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint is checked every 20 seconds (also configurable with a flag).
```

`kubelet`是每个节点上的关键节点代理，负责注册节点并确保`PodSpec`中描述的容器运行且健康。它支持通过主机名、标志覆盖或云提供商逻辑进行节点注册。`PodSpecs`（`YAML`或`JSON`）可以通过`apiserver`或两种其他方式提供：一种是监控特定路径下文件的变化，另一种是定期检查`HTTP`端点。默认的监控周期均为`20`秒，但可以通过命令行标志进行调整。

### 启动Kubelet

```go
// cmd/kubelet/kubelet.go
func main() {
    // 构建一个cobra类型的kubelet命令
	command := app.NewKubeletCommand()
    // 启动kubelet
	code := cli.Run(command)
	os.Exit(code)
}
```

`NewKubeletCommand`函数比较长，主要涉及到`kubelet`命令行接口的创建和配置，通过`cobra`库来实现。它展示了如何定义命令、解析和验证命令行参数、加载和验证配置、以及最终执行命令的逻辑。

```go
// cmd/kubelet/app/server.go
// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand() *cobra.Command {
// 省略
	cmd := &cobra.Command{
		Use: componentKubelet,
// 省略		
			// run the kubelet
			return Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate)
		},
	}
// 省略
	return cmd
}
```

可以看到`Run`函数是`Kubelet`的入口函数，`Run`函数对`run`函数进行了一些封装，并记录一些信息以调试。

```go
// cmd/kubelet/app/server.go
// Run runs the specified KubeletServer with the given Dependencies. This should never exit.
// The kubeDeps argument may be nil - if so, it is initialized from the settings on KubeletServer.
// Otherwise, the caller is assumed to have set up the Dependencies object and a default one will
// not be generated.
func Run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) error {
	// To help debugging, immediately log version
	klog.InfoS("Kubelet version", "kubeletVersion", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	if err := initForOS(s.KubeletFlags.WindowsService, s.KubeletFlags.WindowsPriorityClass); err != nil {
		return fmt.Errorf("failed OS init: %w", err)
	}
	if err := run(ctx, s, kubeDeps, featureGate); err != nil {
		return fmt.Errorf("failed to run Kubelet: %w", err)
	}
	return nil
}
```

`run`函数超级长（下面省略了细节），通过一系列的初始化步骤和配置来启动kubelet，包括特性门控（Feature Gates）的设置、配置验证、依赖项的准备、客户端的创建、认证配置、资源管理、事件记录器的设置等。

```go
// cmd/kubelet/app/server.go
func run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) (err error) {
	// Set global feature gates based on the value on the initial KubeletServer（特性门控，根据配置启用或禁用某些kubelet特性）
	// validate the initial KubeletServer (we set feature gates first, because this validation depends on feature gates，验证配置)
	// Warn if MemoryQoS enabled with cgroups v1
	// Obtain Kubelet Lock File（锁文件处理）
    // Register current configuration with /configz endpoint
	// About to get clients and such, detect standaloneMode
	// if in standalone mode, indicate as much by setting all clients to nil
    // make a separate client for events
	// make a separate client for heartbeat with throttling disabled and a timeout attached
	// The timeout is the minimum of the lease duration and status update frequency
	// Get cgroup driver setting from CRI
	// Setup event recorder if required.
	// 初始化容器管理器
		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				// 省略
			},
			s.FailSwapOn,
			kubeDeps.Recorder,
			kubeDeps.KubeClient,
		)
	// 启动kubelet	
        if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
			return err
		}
		if err != nil {
			return err
		}
	}
	return nil
}
```

`RunKubelet`函数是kubelet启动和运行的核心，它负责初始化`kubelet`的配置、依赖项和运行环境，并调用`startedKubelet`启动`Kubelet`，

```go	
// cmd/kubelet/app/server.go
// RunKubelet is responsible for setting up and running a kubelet.  It is used in three different applications:
//
//	1 Integration tests
//	2 Kubelet binary
//	3 Standalone 'kubernetes' binary
//
// Eventually, #2 will be replaced with instances of #3
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
	// Query the cloud provider for our node name, default to hostname if kubeDeps.Cloud == nil
	// Setup event recorder if required.
		k, err := createAndInitKubelet(kubeServer,
		kubeDeps,
		hostname,
		hostnameOverridden,
		nodeName,
		nodeIPs)
	if err != nil {
		return fmt.Errorf("failed to create kubelet: %w", err)
	}
    // NewMainKubelet should have set up a pod source config if one didn't exist
	// when the builder was run. This is just a precaution.
	if kubeDeps.PodConfig == nil {
		return fmt.Errorf("failed to create kubelet, pod source config was nil")
	}
	podCfg := kubeDeps.PodConfig
	// process pods and exit.
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %w", err)
		}
		klog.InfoS("Started kubelet as runonce")
	} else {
		startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableServer)
		klog.InfoS("Started kubelet")
	}
	return nil
}
```

`startKubelet`用于启动`kubelet`、`kubelet apiserver`和`pod`资源管理器，都是通过`goroutine`异步执行。

```go
// cmd/kubelet/app/server.go
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go k.Run(podCfg.Updates())

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(kubeCfg, kubeDeps.TLSOptions, kubeDeps.Auth, kubeDeps.TracerProvider)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(netutils.ParseIPSloppy(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
	go k.ListenAndServePodResources()
}
```

在`K8s`中，管理`Pod`配置的核心组件是`PodConfig`结构体。这个结构体的主要职责是整合来自不同来源的`Pod`配置信息，形成一个统一的配置结构。这些配置信息来源可能包括API请求、配置文件、环境变量等。一旦有新的配置信息或现有配置信息发生变化，`PodConfig`就会负责将这些变化以增量的方式通知给所有注册的监听器，确保系统的每个部分都能及时准确地响应这些变化。

`PodConfig`的设计采用了观察者模式，其中`PodConfig`充当被观察者，而各个监听器则是观察者。当`PodConfig`中的数据发生变化时，它会通过`updates`通道向所有监听器发送更新通知。这种设计模式使得系统能够灵活地响应配置变化，同时保持了各组件之间的解耦，提高了系统的可维护性和扩展性。

`PodConfig`内部使用`podStorage`结构体来管理当前的Pod状态，并确保按顺序向`updates`通道传递更新通知。`podStorage`通过一系列的锁（如`podLock`和`updateLock`）来处理并发操作，确保`Pod`状态的一致性和更新通知的有序性。此外，`podStorage`还维护了一个`sourcesSeen`集合，用于记录已经发送过至少一次SET操作的配置源，这有助于`PodConfig`识别和管理来自不同源的配置信息。

```go
// pkg/kubelet/config/config.go
// PodConfig is a configuration mux that merges many sources of pod configuration into a single
// consistent structure, and then delivers incremental change notifications to listeners
// in order.
type PodConfig struct {
	pods *podStorage
	mux  *config.Mux

	// the channel of denormalized changes passed to listeners
	updates chan kubetypes.PodUpdate

	// contains the list of all configured sources
	sourcesLock sync.Mutex
	sources     sets.String
}

// podStorage manages the current pod state at any point in time and ensures updates
// to the channel are delivered in order.  Note that this object is an in-memory source of
// "truth" and on creation contains zero entries.  Once all previously read sources are
// available, then this object should be considered authoritative.
// 通过一些锁来处理对pod的并发操作
type podStorage struct {
	podLock sync.RWMutex
	// map of source name to pod uid to pod reference
	pods map[string]map[types.UID]*v1.Pod
	mode PodConfigNotificationMode

	// ensures that updates are delivered in strict order
	// on the updates channel
	updateLock sync.Mutex
	updates    chan<- kubetypes.PodUpdate

	// contains the set of all sources that have sent at least one SET
	sourcesSeenLock sync.RWMutex
	sourcesSeen     sets.String

	// the EventRecorder to use
	recorder record.EventRecorder

	startupSLIObserver podStartupSLIObserver
}

// pkg/kubelet/types/pod_update.go
// PodUpdate defines an operation sent on the channel. You can add or remove single services by
// sending an array of size one and Op == ADD|REMOVE (with REMOVE, only the ID is required).
// For setting the state of the system to a given state for this source configuration, set
// Pods as desired and Op to SET, which will reset the system state to that specified in this
// operation for this source channel. To remove all pods, set Pods to empty object and Op to SET.
//
// Additionally, Pods should never be nil - it should always point to an empty slice. While
// functionally similar, this helps our unit tests properly check that the correct PodUpdates
// are generated.
type PodUpdate struct {
	Pods   []*v1.Pod
	Op     PodOperation
	Source string
}

// PodOperation defines what changes will be made on a pod configuration.
type PodOperation int

// These constants identify the PodOperations that can be made on a pod configuration.
const (
	// SET is the current pod configuration.
	SET PodOperation = iota
	// ADD signifies pods that are new to this source.
	ADD
	// DELETE signifies pods that are gracefully deleted from this source.
	DELETE
	// REMOVE signifies pods that have been removed from this source.
	REMOVE
	// UPDATE signifies pods have been updated in this source.
	UPDATE
	// RECONCILE signifies pods that have unexpected status in this source,
	// kubelet should reconcile status with this source.
	RECONCILE
)

// k8s.io/api/core/v1/types.go
// Pod is a collection of containers that can run on a host. This resource is created
// by clients and scheduled onto hosts.
type Pod struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
    // Spec和Status包含了Pod的元数据
	// Specification of the desired behavior of the pod.
	Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	// Most recently observed status of the pod.
	// This data may not be up to date.
	// Populated by the system.
	// Read-only.
	Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

回到`kubelet`的启动过程中，查看`kubelet`最终的启动接口`k.Run(podCfg.Updates())`，这个函数签名可以看出通过启动`Kubelet`实例，让它开始监听来自`updates`通道的信息，使得`Kubelet`能够异步接收来自其他组件（如`k8s-apiserver`）的`Pod`更新通知。并根据这些信息管理节点上的`Pod`生命周期（如启动、停止、更新容器等）。

```go
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	ctx := context.Background()
	if kl.logServer == nil {
		file := http.FileServer(http.Dir(nodeLogDir))
		if utilfeature.DefaultFeatureGate.Enabled(features.NodeLogQuery) && kl.kubeletConfiguration.EnableSystemLogQuery {
			kl.logServer = http.StripPrefix("/logs/", http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
				if nlq, errs := newNodeLogQuery(req.URL.Query()); len(errs) > 0 {
					http.Error(w, errs.ToAggregate().Error(), http.StatusBadRequest)
					return
				} else if nlq != nil {
					if req.URL.Path != "/" && req.URL.Path != "" {
						http.Error(w, "path not allowed in query mode", http.StatusNotAcceptable)
						return
					}
					if errs := nlq.validate(); len(errs) > 0 {
						http.Error(w, errs.ToAggregate().Error(), http.StatusNotAcceptable)
						return
					}
					// Validation ensures that the request does not query services and files at the same time
					if len(nlq.Services) > 0 {
						journal.ServeHTTP(w, req)
						return
					}
					// Validation ensures that the request does not explicitly query multiple files at the same time
					if len(nlq.Files) == 1 {
						// Account for the \ being used on Windows clients
						req.URL.Path = filepath.ToSlash(nlq.Files[0])
					}
				}
				// Fall back in case the caller is directly trying to query a file
				// Example: kubectl get --raw /api/v1/nodes/$name/proxy/logs/foo.log
				file.ServeHTTP(w, req)
			}))
		} else {
			kl.logServer = http.StripPrefix("/logs/", file)
		}
	}
	if kl.kubeClient == nil {
		klog.InfoS("No API server defined - no node status update will be sent")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.ErrorS(err, "Failed to initialize internal modules")
		os.Exit(1)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start two go-routines to update the status.
		//
		// The first will report to the apiserver every nodeStatusUpdateFrequency and is aimed to provide regular status intervals,
		// while the second is used to provide a more timely status update during initialization and runs an one-shot update to the apiserver
		// once the node becomes ready, then exits afterwards.
		//
		// Introduce some small jittering to ensure that over time the requests won't start
		// accumulating at approximately the same time from the set of nodes due to priority and
		// fairness effect.
		go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(context.Background())
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start component sync loops.同步pod状态，更新pod状态到缓存
	kl.statusManager.Start()

	// Start syncing RuntimeClasses if enabled.为pod选择合适容器运行时
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.管理和上报pod生命周期
	kl.pleg.Start()

	// Start eventedPLEG only if EventedPLEG feature gate is enabled.
	if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
		kl.eventedPleg.Start()
	}
	// 听来自file/apiserver/http 事件源发送过来的事件（PodUpdate），并对事件做出对应的同步处理。
	kl.syncLoop(ctx, updates, kl)
}
```

上述代码中的` kl.initializeModules()`初始化了一些模块：

- `metrics.Register` ，注册`Prometheus`监控指标。

- `setupDataDirs` ，设置文件存储目录（`/var/lib/kubelet`的相关目录）。

  - `the root directory `
  - `the pods directory `
  - `the plugins directory `
  - `the pod-resources directory `
- ` ContainerLogsDir` ，创建容器日志目录（`/var/log/containers`）。
- `imageManager.Start()` ，启动镜像管理器。
- `serverCertificateManager.Start() `，启动证书管理器。
- `oomWatcher.Start()` ，启动`oom`监听器。
- `resourceAnalyzer.Start()` ，启动资源分析管理器。
