---
layout: post
title: NVIDIA-Container-Toolkit—源码阅读—tools
subtitle: NVIDIA-Container-Toolkit—CodeReading—tools
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## tools/container

tree

```shell
.
├── containerd
│   ├── config_v1_test.go
│   ├── config_v2_test.go
│   └── containerd.go
├── container.go
├── crio
│   └── crio.go
├── docker
│   ├── docker.go
│   └── docker_test.go
├── nvidia-toolkit
│   ├── run.go
│   └── run_test.go
├── operator
│   ├── operator.go
│   └── operator_test.go
├── README.md
└── toolkit
    ├── executable.go
    ├── executable_test.go
    ├── replacements.go
    ├── runtime.go
    ├── runtime_test.go
    └── toolkit.go
```

## README.md

该文件夹包含一些工具，可用于将 docker、containerd 或 cri-o 配置为使用NVIDIA Container Toolkit。

用法如下：

```shell
docker setup \
    --runtime-name NAME \
        /run/nvidia/toolkit
```

## container.go

定义了一个container包。

引入的两个外部模块：

1. [urfave/cli 框架](https://jaycechant.info/2020/golang-in-action-day-8/)
2. [logrus](http://www.zengyuzhao.com/archives/277)

一些常量和结构体定义：

```go
const (
	restartModeNone    = "none"
	restartModeSignal  = "signal"
	restartModeSystemd = "systemd"
)
// Options defines the shared options for the CLIs to configure containers runtimes.
type Options struct {
	Config        string
	Socket        string
	RuntimeName   string
	RuntimeDir    string
	SetAsDefault  bool
	RestartMode   string
	HostRootMount string
}
```

ParseArgs函数：提取出命令行参数的RuntimeDir赋值给Options结构体指针o中的RuntimeDir，这里的命令行参数指除了定义了的子命令以及flag外的参数，只接受一个。

```go
// ParseArgs parses the command line arguments to the CLI
func ParseArgs(c *cli.Context, o *Options) error {
	if o.RuntimeDir != "" {
		logrus.Debug("Runtime directory already set; ignoring arguments")
		return nil
	}
	args := c.Args()
	logrus.Infof("Parsing arguments: %v", args.Slice())
	if c.NArg() != 1 {
		return fmt.Errorf("incorrect number of arguments")  
	}
	o.RuntimeDir = args.Get(0)
	logrus.Infof("Successfully parsed arguments")
	return nil
}
```

> 因为cfg是interface，所以container.go中所有cfg的函数都和调用的运行时（docker、containerd和cri-o）有关。

Configure和UpdateConfig：根据options中的相关内容（RuntimeName、SetAsDefault和RuntimeDir）创建Runtimes，并更新到cfg。

Unconfigure和RevertConfig与Configure和UpdateConfig类似，不过是将Runtimes从cfg移除。

flush则将对cfg的更改写入到o.config中。

```go
// Configure applies the options to the specified config
func (o Options) Configure(cfg engine.Interface) error {
	err := o.UpdateConfig(cfg)
	if err != nil {
		return fmt.Errorf("unable to update config: %v", err)
	}
	return o.flush(cfg)
}
// Unconfigure removes the options from the specified config
func (o Options) Unconfigure(cfg engine.Interface) error {
	err := o.RevertConfig(cfg)
	if err != nil {
		return fmt.Errorf("unable to update config: %v", err)
	}
	return o.flush(cfg)
}
// flush flushes the specified config to disk
func (o Options) flush(cfg engine.Interface) error {
	logrus.Infof("Flushing config to %v", o.Config)
	n, err := cfg.Save(o.Config)
	if err != nil {
		return fmt.Errorf("unable to flush config: %v", err)
	}
	if n == 0 {
		logrus.Infof("Config file is empty, removed")
	}
	return nil
}
// UpdateConfig updates the specified config to include the nvidia runtimes
func (o Options) UpdateConfig(cfg engine.Interface) error {
	runtimes := operator.GetRuntimes(
		operator.WithNvidiaRuntimeName(o.RuntimeName),
		operator.WithSetAsDefault(o.SetAsDefault),
		operator.WithRoot(o.RuntimeDir),
	)
	for name, runtime := range runtimes {
		err := cfg.AddRuntime(name, runtime.Path, runtime.SetAsDefault)
		if err != nil {
			return fmt.Errorf("failed to update runtime %q: %v", name, err)
		}
	}
	return nil
}
// RevertConfig reverts the specified config to remove the nvidia runtimes
func (o Options) RevertConfig(cfg engine.Interface) error {
	runtimes := operator.GetRuntimes(
		operator.WithNvidiaRuntimeName(o.RuntimeName),
		operator.WithSetAsDefault(o.SetAsDefault),
		operator.WithRoot(o.RuntimeDir),
	)
	for name := range runtimes {
		err := cfg.RemoveRuntime(name)
		if err != nil {
			return fmt.Errorf("failed to remove runtime %q: %v", name, err)
		}
	}
	return nil
}
```

Restart（根据o.RestartMode）和SystemdRestart（也就是systemctl restart，不过会判断o.HostRootMount是否进入虚拟环境执行命令）用于重启套接字和systemd服务。

```go
// Restart restarts the specified service
func (o Options) Restart(service string, withSignal func(string) error) error {
	switch o.RestartMode {
	case restartModeNone:
		logrus.Warningf("Skipping restart of %v due to --restart-mode=%v", service, o.RestartMode)
		return nil
	case restartModeSignal:
		return withSignal(o.Socket)
	case restartModeSystemd:
		return o.SystemdRestart(service)
	}

	return fmt.Errorf("invalid restart mode specified: %v", o.RestartMode)
}
// SystemdRestart restarts the specified service using systemd
func (o Options) SystemdRestart(service string) error {
	var args []string
	var msg string
	if o.HostRootMount != "" {
		msg = " on host"
		args = append(args, "chroot", o.HostRootMount)
	}
	args = append(args, "systemctl", "restart", service)

	logrus.Infof("Restarting %v%v using systemd: %v", service, msg, args)

	cmd := exec.Command(args[0], args[1:]...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	err := cmd.Run()
	if err != nil {
		return fmt.Errorf("error restarting %v using systemd: %v", service, err)
	}

	return nil
}
```

## operator

定义了一个operator包。

一些常量和结构体定义：
```go
const (
	defaultRuntimeName = "nvidia"
	defaultRoot = "/usr/bin"
)
// Runtime defines a runtime to be configured.
// The path and whether the runtime is the default runtime can be specfied
// 下面三个成员均根据config中来
type Runtime struct {
	name         string
	Path         string
	SetAsDefault bool
}
// Runtimes defines a set of runtimes to be configure for use in the GPU Operator
type Runtimes map[string]Runtime
type config struct {
	root              string
	nvidiaRuntimeName string
	setAsDefault      bool
}
// Option is a functional option for configuring set of runtimes.
type Option func(*config)
```

GetRuntimes函数用于配置Runtime，并添加到Runtimes，并返回当前所有可用的运行时：包括一个Runtime、"nvidia-cdi"和"nvidia-legacy"相关的运行时。

```go
// GetRuntimes returns the set of runtimes to be configured for use with the GPU Operator.
func GetRuntimes(opts ...Option) Runtimes {
	c := &config{}
	for _, opt := range opts {
		opt(c)
	}

	if c.root == "" {
		c.root = defaultRoot
	}
	if c.nvidiaRuntimeName == "" {
		c.nvidiaRuntimeName = defaultRuntimeName
	}

	runtimes := make(Runtimes)
	runtimes.add(c.nvidiaRuntime())

	modes := []string{"cdi", "legacy"}
	for _, mode := range modes {
		runtimes.add(c.modeRuntime(mode))
	}
	return runtimes
}
```

DefaultRuntimeName函数返回Runtimes中的默认Runtime的name，即Runtime中setAsDefault为true即`SetAsDefault: c.setAsDefault && name == c.nvidiaRuntimeName`，也就是config中开启了setAsDefault，config中的nvidiaRuntimeName，实际其实是来自options中的RuntimeName。

```go
// DefaultRuntimeName returns the name of the default runtime.
func (r Runtimes) DefaultRuntimeName() string {
	for _, runtime := range r {
		if runtime.SetAsDefault {
			return runtime.name
		}
	}
	return ""
}
```

add函数没太多好说的，就是添加Runtimes[Rutime.name] = Runtime到Runtimes中。

```go
// Add a runtime to the set of runtimes.
func (r *Runtimes) add(runtime Runtime) {
	(*r)[runtime.name] = runtime
}
```

nvidiaRuntime创建一个新的runtime，如果config中的nvidiaRuntimeName为"nvidia-cdi"或者"nvidia-legacy"，创建的新runtime的名字为默认的nvidia；否则沿用config中的名字。（这里是因为GetRuntimes中modRuntime会对"nvidia-cdi"和"nvidia-legacy"进行配置）

modeRuntime和newRuntime是用来设置nvidia相关的runtime。

```go
// nvidiaRuntime creates a runtime that corresponds to the nvidia runtime.
// If name is equal to one of the predefined runtimes, `nvidia` is used as the runtime name instead. 
func (c config) nvidiaRuntime() Runtime {
	predefinedRuntimes := map[string]struct{}{
		"nvidia-cdi":    {},
		"nvidia-legacy": {},
	}
	name := c.nvidiaRuntimeName
	if _, isPredefinedRuntime := predefinedRuntimes[name]; isPredefinedRuntime {
		name = defaultRuntimeName
	}
	return c.newRuntime(name, "nvidia-container-runtime")
}
// modeRuntime creates a runtime for the specified mode.
func (c config) modeRuntime(mode string) Runtime {
	return c.newRuntime("nvidia-"+mode, "nvidia-container-runtime."+mode)
}
// newRuntime creates a runtime based on the configuration
func (c config) newRuntime(name string, binary string) Runtime {
	return Runtime{
		name:         name,
		Path:         filepath.Join(c.root, binary),
		SetAsDefault: c.setAsDefault && name == c.nvidiaRuntimeName,
	}
}
```

With函数系列通过选项函数用于设置config相关参数，调用的时候需要With...(name)(c)，这也是GetRuntimes函数中使用opt(c)的原因。

```go
// WithRoot sets the root directory for the runtime binaries.
func WithRoot(root string) Option {
	return func(c *config) {
		c.root = root
	}
}
// WithNvidiaRuntimeName sets the name of the nvidia runtime.
func WithNvidiaRuntimeName(name string) Option {
	return func(c *config) {
		c.nvidiaRuntimeName = name
	}
}
// WithSetAsDefault sets the default runtime to the nvidia runtime.
func WithSetAsDefault(set bool) Option {
	return func(c *config) {
		c.setAsDefault = set
	}
}
```

## docker

是一个main执行程序。

定义的一些常量和结构体：

```go
const (
	defaultConfig       = "/etc/docker/daemon.json"
	defaultSocket       = "/var/run/docker.sock"
	defaultSetAsDefault = true
	// defaultRuntimeName specifies the NVIDIA runtime to be use as the default runtime if setting the default runtime is enabled
	defaultRuntimeName   = "nvidia"
	defaultRestartMode   = "signal"
	defaultHostRootMount = "/host"
	reloadBackoff     = 5 * time.Second
	maxReloadAttempts = 6
	socketMessageToGetPID = "GET /info HTTP/1.0\r\n\r\n"
)
// options stores the configuration from the command line or environment variables
type options struct {
	container.Options
}
```

main函数主要是通过[urfave/cli包](https://cli.urfave.org/)构建了一个cli程序，支持setup和cleanup两个子命令，并支持一系列flag（setup和cleanup都支持）。

```go
--config
--socket
--restart-mode
--host-root
--runtime-name (或者--nvidia-runtime-name，--runtime-class)
--nvidia-runtime-dir 或者--runtime-dir
--set-as-default
```

Setup函数创建了一个新的docker config，并根据option进行更改和写入文件，最后重启docker服务。

Cleanup函数和Setup函数类似。

```go
// Setup updates docker configuration to include the nvidia runtime and reloads it
func Setup(c *cli.Context, o *options) error {
	log.Infof("Starting 'setup' for %v", c.App.Name)
	cfg, err := docker.New(
		docker.WithPath(o.Config),
	)
	if err != nil {
		return fmt.Errorf("unable to load config: %v", err)
	}
	err = o.Configure(cfg)
	if err != nil {
		return fmt.Errorf("unable to configure docker: %v", err)
	}
	err = RestartDocker(o)
	if err != nil {
		return fmt.Errorf("unable to restart docker: %v", err)
	}
	log.Infof("Completed 'setup' for %v", c.App.Name)
	return nil
}
```

RestartDocker函数调用option.Restart函数对docker进行重启，根据option中的RestartMode选择通过SignalDocker函数用发送信号进行重启，还是直接通过systemd进行重启，默认是signal。

SignalDocker函数发送一个SIGHUP信号给docker daemon来重启docker，最多尝试6次。

## containerd

是一个main执行程序。

定义了一些常量和结构体，runc.v1和v2的区别见[容器中的 Shim 到底是个什么鬼？](https://cloud.tencent.com/developer/article/1948155)：

```go
const (
	defaultConfig        = "/etc/containerd/config.toml"
	defaultSocket        = "/run/containerd/containerd.sock"
	defaultRuntimeClass  = "nvidia"
	defaultRuntmeType    = "io.containerd.runc.v2"
	defaultSetAsDefault  = true
	defaultRestartMode   = "signal"
	defaultHostRootMount = "/host"
	reloadBackoff     = 5 * time.Second
	maxReloadAttempts = 6
	socketMessageToGetPID = ""
)
// options stores the configuration from the command line or environment variables
type options struct {
	container.Options
	// containerd-specific options
	useLegacyConfig bool
	runtimeType     string
	ContainerRuntimeModesCDIAnnotationPrefixes cli.StringSlice
}
```

main函数主要是通过[urfave/cli包](https://cli.urfave.org/)构建了一个cli程序，支持的子命令和flag和docker的一致，不再赘述。

> 有趣的是这里在运行命令时，如果出错用的是log.Fatal()，而docker中是log.Errorf，再exit(1)，当然两者效果是一样的。

Setup函数创建了一个新的containerd config，并根据option进行更改和写入文件，最后重启containerd服务。新建config也和docker操作不一样。

```go
// Setup updates a containerd configuration to include the nvidia-containerd-runtime and reloads it
func Setup(c *cli.Context, o *options) error {
	log.Infof("Starting 'setup' for %v", c.App.Name)
	cfg, err := containerd.New(
		containerd.WithPath(o.Config),
		containerd.WithRuntimeType(o.runtimeType),
		containerd.WithUseLegacyConfig(o.useLegacyConfig),		containerd.WithContainerAnnotations(o.containerAnnotationsFromCDIPrefixes()...),
	)
	if err != nil {
		return fmt.Errorf("unable to load config: %v", err)
	}
	err = o.Configure(cfg)
	if err != nil {
		return fmt.Errorf("unable to configure containerd: %v", err)
	}
	err = RestartContainerd(o)
	if err != nil {
		return fmt.Errorf("unable to restart containerd: %v", err)
	}
	log.Infof("Completed 'setup' for %v", c.App.Name)
	return nil
}
```

Cleanup函数、RestartContainerd和SignalContainerd函数不再赘述，和docker中描述的类似。

containerAnnotationsFromCDIPrefixes函数应该是用于容器调用nvidia GPU的相关函数，CDI是容器设备接口，用于Linux容器的设备。

> **但是我想说containerd的两个config的测试写的是真的牛x**

## crio

是一个main执行程序。

定义了一些常量和结构体，不同于docker和containerd，cri-o的默认重启模式是systemd，也只支持systemd：

```go
const (
	defaultConfigMode = "hook"
	// Hook-based settings
	defaultHooksDir     = "/usr/share/containers/oci/hooks.d"
	defaultHookFilename = "oci-nvidia-hook.json"
	// Config-based settings
	defaultConfig        = "/etc/crio/crio.conf"
	defaultSocket        = "/var/run/crio/crio.sock"
	defaultRuntimeClass  = "nvidia"
	defaultSetAsDefault  = true
	defaultRestartMode   = "systemd"
	defaultHostRootMount = "/host"
)
// options stores the configuration from the command linek or environment variables
type options struct {
	container.Options
	configMode string
	// hook-specific options
	hooksDir     string
	hookFilename string
}
```

main函数主要是通过[urfave/cli包](https://cli.urfave.org/)构建了一个cli程序，支持的子命令和flag和docker、containerd的一致，不再赘述。

Setup、setupHook和setupConfig三个函数用于配置cri-o的config文件，默认模式是hook。

```go
// Setup installs the prestart hook required to launch GPU-enabled containers
func Setup(c *cli.Context, o *options) error {
	log.Infof("Starting 'setup' for %v", c.App.Name)
	switch o.configMode {
	case "hook":
		return setupHook(o)
	case "config":
		return setupConfig(o)
	default:
		return fmt.Errorf("invalid config-mode '%v'", o.configMode)
	}
}
// setupHook installs the prestart hook required to launch GPU-enabled containers
func setupHook(o *options) error {
	log.Infof("Installing prestart hook")
	hookPath := filepath.Join(o.hooksDir, o.hookFilename)
	err := ocihook.CreateHook(hookPath, filepath.Join(o.RuntimeDir, config.NVIDIAContainerRuntimeHookExecutable))
	if err != nil {
		return fmt.Errorf("error creating hook: %v", err)
	}
	return nil
}
// setupConfig updates the cri-o config for the NVIDIA container runtime
func setupConfig(o *options) error {
	log.Infof("Updating config file")
	cfg, err := crio.New(
		crio.WithPath(o.Config),
	)
	if err != nil {
		return fmt.Errorf("unable to load config: %v", err)
	}
	err = o.Configure(cfg)
	if err != nil {
		return fmt.Errorf("unable to configure cri-o: %v", err)
	}
	err = RestartCrio(o)
	if err != nil {
		return fmt.Errorf("unable to restart crio: %v", err)
	}
	return nil
}
```

cri-o重启的相关函数，可以看到signal的函数直接就是不支持。

```go
// RestartCrio restarts crio depending on the value of restartModeFlag
func RestartCrio(o *options) error {
	return o.Restart("crio", func(string) error { return fmt.Errorf("supporting crio via signal is unsupported") })
}
```

## toolkit

### executable.go

包是main，定义了executable的相关函数。

定义的一些结构体：

```go
type executableTarget struct {
	dotfileName string
	wrapperName string
}
type executable struct {
	source   string
	target   executableTarget
	env      map[string]string
	preLines []string
	argLines []string
}
```

install函数下载NVIDIA container toolkit组件的可执行文件到destFolder，可执行文件被复制重命名为一个.real文件，还创建了一个wrapper（其实就是个可执行的脚本文件）来设置需要的环境变量。

installFileToFolderWithName函数很简单，就是把源可执行文件复制重命名为源可执行文件.real，并设置相同的访问执行权限。

**ps：这里注释的wapper写错了x**

```go
// install installs an executable component of the NVIDIA container toolkit. The source executable
// is copied to a `.real` file and a wapper is created to set up the environment as required.
func (e executable) install(destFolder string) (string, error) {
	log.Infof("Installing executable '%v' to %v", e.source, destFolder)
	dotfileName := e.dotfileName()
	installedDotfileName, err := installFileToFolderWithName(destFolder, dotfileName, e.source)
	if err != nil {
		return "", fmt.Errorf("error installing file '%v' as '%v': %v", e.source, dotfileName, err)
	}
	log.Infof("Installed '%v'", installedDotfileName)
	wrapperFilename, err := e.installWrapper(destFolder, installedDotfileName)
	if err != nil {
		return "", fmt.Errorf("error wrapping '%v': %v", installedDotfileName, err)
	}
	log.Infof("Installed wrapper '%v'", wrapperFilename)
	return wrapperFilename, nil
}
```

installWrapper函数，流程就是创建wrapper文件、写入wrapper文件和修改wrapper文件权限使其可执行。

```go
func (e executable) installWrapper(destFolder string, dotfileName string) (string, error) {
	wrapperPath := filepath.Join(destFolder, e.wrapperName())
	wrapper, err := os.Create(wrapperPath)
	if err != nil {
		return "", fmt.Errorf("error creating executable wrapper: %v", err)
	}
	defer wrapper.Close()
	err = e.writeWrapperTo(wrapper, destFolder, dotfileName)
	if err != nil {
		return "", fmt.Errorf("error writing wrapper contents: %v", err)
	}
	err = ensureExecutable(wrapperPath)
	if err != nil {
		return "", fmt.Errorf("error making wrapper executable: %v", err)
	}
	return wrapperPath, nil
}
```

writeWrapperTo函数：首先创建了替换规则，即将占位符@destDir@替换为destFolder。向wrapper中写入shell程序内容：遍历e.preLines，将每行应用替换规则后写入wrapper。将destFolder加入PATH环境变量，然后对e中的环境变量值进行了排序，并对环境变量应用替换规则以单个环境变量名=环境变量值\的形式写入wrapper（每个占一行）。然后写入可执行程序dotfileName及其调用参数（也需应用替换规则），并通过"$@"将调用wrapper后面的参数传递给dotfileName。

```go
func (e executable) writeWrapperTo(wrapper io.Writer, destFolder string, dotfileName string) error {
	r := newReplacements(destDirPattern, destFolder)
	// Add the shebang
	fmt.Fprintln(wrapper, "#! /bin/sh")
	// Add the preceding lines if any
	for _, line := range e.preLines {
		fmt.Fprintf(wrapper, "%s\n", r.apply(line))
	}
	// Update the path to include the destination folder
	var env map[string]string
	if e.env == nil {
		env = make(map[string]string)
	} else {
		env = e.env
	}
	path, specified := env["PATH"]
	if !specified {
		path = "$PATH"
	}
	env["PATH"] = strings.Join([]string{destFolder, path}, ":")
	var sortedEnvvars []string
	for e := range env {
		sortedEnvvars = append(sortedEnvvars, e)
	}
	sort.Strings(sortedEnvvars)
	for _, e := range sortedEnvvars {
		v := env[e]
		fmt.Fprintf(wrapper, "%s=%s \\\n", e, r.apply(v))
	}
	// Add the call to the target executable
	fmt.Fprintf(wrapper, "%s \\\n", dotfileName)
	// Insert additional lines in the `arg` list
	for _, line := range e.argLines {
		fmt.Fprintf(wrapper, "\t%s \\\n", r.apply(line))
	}
	// Add the script arguments "$@"
	fmt.Fprintln(wrapper, "\t\"$@\"")
	return nil
}
```

ensureExecutable函数比较简单，就是把wrapper文件的权限变为user、group和others均可执行。

### replacements.go

同属于main，定义了一些replacements的函数。

定义的一些常量和结构体：

```go
const (
	destDirPattern = "@destDir@"
)
type replacements map[string]string
```

newReplacements函数将给定的rules创建了一个map结构replacements，奇数的rule为索引（旧值），偶数的为值（新值）。

```go
func newReplacements(rules ...string) replacements {
	r := make(replacements)
	for i := 0; i < len(rules)-1; i += 2 {
		old := rules[i]
		new := rules[i+1]
		r[old] = new
	}
	return r
}
```

apply函数将input中的旧值替换为新值。

```go
func (r replacements) apply(input string) string {
	output := input
	for old, new := range r {
		output = strings.ReplaceAll(output, old, new)
	}
	return output
}
```

### runtime.go

同属于main，定义了runtime相关的函数。

定义了一些常量：

```go
const (
	nvidiaContainerRuntimeSource = "/usr/bin/nvidia-container-runtime"
)
```

installContainerRuntimes函数将所有runtime进行复制和执行wrapper设置环境等，关键还是install函数

**ps：driverRoot变量没用到**

```go
// installContainerRuntimes sets up the NVIDIA container runtimes, copying the executables
// and implementing the required wrapper
func installContainerRuntimes(toolkitDir string, driverRoot string) error {
	runtimes := operator.GetRuntimes()
	for _, runtime := range runtimes {
		r := newNvidiaContainerRuntimeInstaller(runtime.Path)
		_, err := r.install(toolkitDir)
		if err != nil {
			return fmt.Errorf("error installing NVIDIA container runtime: %v", err)
		}
	}
	return nil
}
```

newNvidiaContainerRuntimeInstaller函数构造了executableTarget结构体，包含了执行文件的重命名版本（.real）和执行文件，再调用newRuntimeInstaller生成一个新的*executable。

```go
func newNvidiaContainerRuntimeInstaller(source string) *executable {
	wrapperName := filepath.Base(source)
	dotfileName := wrapperName + ".real"
	target := executableTarget{
		dotfileName: dotfileName,
		wrapperName: wrapperName,
	}
	return newRuntimeInstaller(source, target, nil)
}	
```

newRuntimeInstaller函数：preLines是一个脚本，检查是否加载了 "nvidia" 驱动模块。如果没有加载，输出一条消息，并直接调用 runc 命令。否则，不执行任何操作。然后创建了一个XDG_CONFIG_HOME的环境变量，其值为@destDir@.config。

```go
func newRuntimeInstaller(source string, target executableTarget, env map[string]string) *executable {
	preLines := []string{
		"",
		"cat /proc/modules | grep -e \"^nvidia \" >/dev/null 2>&1",
		"if [ \"${?}\" != \"0\" ]; then",
		"	echo \"nvidia driver modules are not yet loaded, invoking runc directly\"",
		"	exec runc \"$@\"",
		"fi",
		"",
	}
	runtimeEnv := make(map[string]string)
	runtimeEnv["XDG_CONFIG_HOME"] = filepath.Join(destDirPattern, ".config")
	for k, v := range env {
		runtimeEnv[k] = v
	}
	r := executable{
		source:   source,
		target:   target,
		env:      runtimeEnv,
		preLines: preLines,
	}
	return &r
}
```

### toolkit.go

同属于main包，真正的main函数，构建了一个toolkit的cli程序，提供了install和delete两个子命令及若干flag。

Install函数用于安装NVIDIA container toolkit的所有组件，包括如下步骤：

- 移除所有已有的安装：os.RemoveAll(opts.toolkitRoot)
- 创建目录：createDirectories(opts.toolkitRoot, toolkitConfigDir)，toolkitConfigDir：opts.toolkitRoot/.config/nvidia-container-runtime
- 安装容器相关的库：installContainerLibraries(opts.toolkitRoot)
- 下载容器运行时：installContainerRuntimes(opts.toolkitRoot, opts.DriverRoot)
- 下载 NVIDIA container CLI：installContainerCLI(opts.toolkitRoot)
- 下载NVIDIA container runtime hook：installRuntimeHook(opts.toolkitRoot, toolkitConfigPath)，toolkitConfigPath：opts.toolkitRoot/.config/nvidia-container-runtime/config.toml
- 下载NVIDIA Container Toolkit CLI：installContainerToolkitCLI(opts.toolkitRoot) `nvidia-ctk`
- 下载NVIDIA container toolkit config：installToolkitConfig(cli, toolkitConfigPath, nvidiaContainerCliExecutable, nvidiaCTKPath, nvidiaContainerRuntimeHookPath, opts)
- 创建CDISpec：generateCDISpec(opts, nvidiaCTKPath)

installContainerToolkitCLI函数：nvidia-ctk的安装

```go
func installContainerToolkitCLI(toolkitDir string) (string, error) {
	e := executable{
		source: "/usr/bin/nvidia-ctk",
		target: executableTarget{
			dotfileName: "nvidia-ctk.real",
			wrapperName: "nvidia-ctk",
		},
	}
	return e.install(toolkitDir)
}
```

installContainerCLI函数：nvidia-container-cli的安装

```go
func installContainerCLI(toolkitRoot string) (string, error) {
	log.Infof("Installing NVIDIA container CLI from '%v'", nvidiaContainerCliSource)

	env := map[string]string{
		"LD_LIBRARY_PATH": toolkitRoot,
	}
	e := executable{
		source: nvidiaContainerCliSource,
		target: executableTarget{
			dotfileName: "nvidia-container-cli.real",
			wrapperName: "nvidia-container-cli",
		},
		env: env,
	}
	installedPath, err := e.install(toolkitRoot)
	if err != nil {
		return "", fmt.Errorf("error installing NVIDIA container CLI: %v", err)
	}
	return installedPath, nil
}
```

installRuntimeHook函数：NVIDIA runtime hook的安装，还创建了一个toolkitRoot/nvidia-container-toolkit到nvidia-container-runtime-hook（wrapper）的软链接。

```go
func installRuntimeHook(toolkitRoot string, configFilePath string) (string, error) {
	log.Infof("Installing NVIDIA container runtime hook from '%v'", nvidiaContainerRuntimeHookSource)

	argLines := []string{
		fmt.Sprintf("-config \"%s\"", configFilePath),
	}
	e := executable{
		source: nvidiaContainerRuntimeHookSource,
		target: executableTarget{
			dotfileName: "nvidia-container-runtime-hook.real",
			wrapperName: "nvidia-container-runtime-hook",
		},
		argLines: argLines,
	}
	installedPath, err := e.install(toolkitRoot)
	if err != nil {
		return "", fmt.Errorf("error installing NVIDIA container runtime hook: %v", err)
	}
	err = installSymlink(toolkitRoot, "nvidia-container-toolkit", installedPath)
	if err != nil {
		return "", fmt.Errorf("error installing symlink to NVIDIA container runtime hook: %v", err)
	}
	return installedPath, nil
}

// installSymlink creates a symlink in the toolkitDirectory that points to the specified target.
// Note: The target is assumed to be local to the toolkit directory
func installSymlink(toolkitRoot string, link string, target string) error {
	symlinkPath := filepath.Join(toolkitRoot, link)
	targetPath := filepath.Base(target)
	log.Infof("Creating symlink '%v' -> '%v'", symlinkPath, targetPath)
	err := os.Symlink(targetPath, symlinkPath)
	if err != nil {
		return fmt.Errorf("error creating symlink '%v' => '%v': %v", symlinkPath, targetPath, err)
	}
	return nil
}
```

createDirectories用于创建参数中的所有目录，权限是0755即RWXR_XR_X	

```go
func createDirectories(dir ...string) error {
	for _, d := range dir {
		log.Infof("Creating directory '%v'", d)
		err := os.MkdirAll(d, 0755)
		if err != nil {
			return fmt.Errorf("error creating directory: %v", err)
		}
	}
	return nil
}
```

## nvidia-toolkit

定义了一些常量和结构体：toolkit的pid文件为/run/nvidia/toolkit.pid，支持的运行时包括docker（default）、crio和containerd

```go
const (
	runDir         = "/run/nvidia"
	pidFile        = runDir + "/toolkit.pid"
	toolkitCommand = "toolkit"
	toolkitSubDir  = "toolkit"
	defaultToolkitArgs = ""
	defaultRuntime     = "docker"
	defaultRuntimeArgs = ""
)
var availableRuntimes = map[string]struct{}{"docker": {}, "crio": {}, "containerd": {}}
var waitingForSignal = make(chan bool, 1)
var signalReceived = make(chan bool, 1)
// options stores the command line arguments
type options struct {
	noDaemon    bool
	runtime     string
	runtimeArgs string
	root        string
}
// Version defines the CLI version. This is set at build time using LD FLAGS
var Version = "development"
```

main函数创建了一个nvidia-toolkit的cli程序，描述如下：

```shell
c.Name = "nvidia-toolkit"
c.Usage = "Install the nvidia-container-toolkit for use by a given runtime"
c.UsageText = "[DESTINATION] [-n | --no-daemon] [-r | --runtime] [-u | --runtime-args]"
c.Description = "DESTINATION points to the host path underneath which the nvidia-container-toolkit should be installed.\nIt will be installed at ${DESTINATION}/toolkit"
```

Run函数定义了该nvidia-toolkit的核心逻辑：包括验证命令的flag、初始化、下载toolkit、设置runtime

ParseArgs函数用于解析出命令行中的路径作为root，关于root的描述：the folder where the NVIDIA Container Toolkit is to be installed. It will be installed to `ROOT`/toolkit，代码写的挺有意思。

```go
func ParseArgs(args []string) ([]string, string, error) {
	log.Infof("Parsing arguments")
	if len(args) < 2 {
		return args, "", nil
	} // 只有命令
	var lastPositionalArg int
	for i, arg := range args {
		if strings.HasPrefix(arg, "-") {
			break
		}
		lastPositionalArg = i
	}
	if lastPositionalArg == 0 {
		return args, "", nil
	} // 0是命令本身加flag的情况
	if lastPositionalArg == 1 {
		return append([]string{args[0]}, args[2:]...), args[1], nil
	} // 1是命令 root flag的情况
	return nil, "", fmt.Errorf("unexpected positional argument(s) %v", args[2:lastPositionalArg+1])  // 有多个root报错
}
```

verifyFlags函数比较简单，做检查参数的一些工作，包括安装文件夹root是否指定、用户输入的runtime是否支持。

initialize函数执行了一些初始化操作，如创建PID文件，获取文件锁，向pid文件写入pid,并设置了信号处理程序，以便在接收到特定信号时执行相应的操作。

installToolkit函数调用toolkit install --toolkit-root path命令将toolkit文件安装在指定目录（o.root/toolkit），又回到toolkit.go中去了。

```go
func installToolkit(o *options) error {
	log.Infof("Installing toolkit")
	cmdline := []string{
		toolkitCommand,
		"install",
		"--toolkit-root",
		filepath.Join(o.root, toolkitSubDir),
	}
	cmd := exec.Command("sh", "-c", strings.Join(cmdline, " "))
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	err := cmd.Run()
	if err != nil {
		return fmt.Errorf("error running %v command: %v", cmdline, err)
	}
	return nil
}	
```

setupRuntime函数o.runtime setup o.runtimeArgs toolkitDir命令，也就是docker、containerd、crio三个运行时的相关配置。相对应的cleanupRuntime函数就是对应runtime的clean up命令。

```go
func setupRuntime(o *options) error {
	toolkitDir := filepath.Join(o.root, toolkitSubDir)
	log.Infof("Setting up runtime")
	cmdline := fmt.Sprintf("%v setup %v %v\n", o.runtime, o.runtimeArgs, toolkitDir)
	cmd := exec.Command("sh", "-c", cmdline)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	err := cmd.Run()
	if err != nil {
		return fmt.Errorf("error running %v command: %v", o.runtime, err)
	}
	return nil
}
```

shutdown函数比较简单，移除pid文件。

## conclusion

tools/container文件夹下创建了几个cli程序：

- docker
- containerd
- crio
- toolkit
- nvidia-toolkit

前面三个cli程序子命令和flag指相同，用于配置或取消对应的运行时使用nvidia-runtime。

toolkit用于安装NVIDIA container toolkit的所有组件：

- 容器相关的库
- nvidia、nvidia-cdi和nvidia-legacy的运行时
- nvidia-container-cli
- nvidia-container-runtime-hook
- nvidia-ctk
- 相关的配置文件和cdi spec

nvidia-toolkit主要是调用前面两个程序完成所有组件的安装和配置：

- toolkit install --toolkit-root path
- o.runtime setup o.runtimeArgs toolkitDir