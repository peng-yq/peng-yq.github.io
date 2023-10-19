---
layout: post
title: NVIDIA-Container-Toolkit—源码阅读—hook
subtitle: NVIDIA-Container-Toolkit—CodeReading—hook
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## nvidia-container-runtime-hook

### hook_config.go

定义的一些结构体和变量：

```go
const (
	configPath = "/etc/nvidia-container-runtime/config.toml"
	driverPath = "/run/nvidia/driver"
)
var defaultPaths = [...]string{}
type HookConfig config.Config
```

getDefaultHookConfig函数用于返回默认的配置文件，并转换为hookconfig类型（但其实就是把默认的返回了：

```go
func getDefaultHookConfig() (HookConfig, error) {
	defaultCfg, err := config.GetDefault()
	if err != nil {
		return HookConfig{}, err
	}
	return *(*HookConfig)(defaultCfg), nil
}
```

loadConfig函数用于加载配置文件，configflag存在则使用该路径作为配置文件路径，并将required设置为 true。否则，它使用driverPath和configPath拼接出一个完整的配置文件路径，并和configPath添加到configPaths中。然后，遍历 configPaths 中的每个路径，尝试创建配置对象。如果成功创建配置对象，则返回该对象的指针。如果遇到文件不存在的错误并且配置文件不是必需的，则继续下一次循环。如果所有的配置文件路径都无法成功加载配置对象，则返回默认的配置对象：

```go
func loadConfig() (*config.Config, error) {
	var configPaths []string
	var required bool
	if len(*configflag) != 0 {
		configPaths = append(configPaths, *configflag)
		required = true
	} else {
		configPaths = append(configPaths, path.Join(driverPath, configPath), configPath)
	}
	for _, p := range configPaths {
		cfg, err := config.New(
			config.WithConfigFile(p),
		)
		if err == nil {
			return cfg.Config()
		} else if os.IsNotExist(err) && !required {
			continue
		}
		return nil, fmt.Errorf("couldn't open required configuration file: %v", err)
	}
	return config.GetDefault()
}
```

getHookConfig函数先调用loadConfig加载配置文件，然后转成HookConfig结构并返回，同时还对权能进行了一些配置：

```go
func getHookConfig() (*HookConfig, error) {
	cfg, err := loadConfig()
	if err != nil {
		return nil, fmt.Errorf("failed to load config: %v", err)
	}
	config := (*HookConfig)(cfg)
	allSupportedDriverCapabilities := image.SupportedDriverCapabilities
	if config.SupportedDriverCapabilities == "all" {
		config.SupportedDriverCapabilities = allSupportedDriverCapabilities.String()
	}
	configuredCapabilities := image.NewDriverCapabilities(config.SupportedDriverCapabilities)
	if !allSupportedDriverCapabilities.IsSuperset(configuredCapabilities) {
		configName := config.getConfigOption("SupportedDriverCapabilities")
		log.Panicf("Invalid value for config option '%v'; %v (supported: %v)\n", configName, config.SupportedDriverCapabilities, allSupportedDriverCapabilities.String())
	}
	return config, nil
}
```

getConfigOption函数用于获取toml文件中的fieldName的值，若不存在则直接返回fieldName：

```go
func (c HookConfig) getConfigOption(fieldName string) string {
	t := reflect.TypeOf(c)
	f, ok := t.FieldByName(fieldName)
	if !ok {
		return fieldName
	}
	v, ok := f.Tag.Lookup("toml")
	if !ok {
		return fieldName
	}
	return v
}
```

getSwarmResourceEnvvars函数用于读取config中SwarmResource的值：

```go
func (c *HookConfig) getSwarmResourceEnvvars() []string {
	if c.SwarmResource == "" {
		return nil
	}
	candidates := strings.Split(c.SwarmResource, ",")
	var envvars []string
	for _, c := range candidates {
		trimmed := strings.TrimSpace(c)
		if len(trimmed) > 0 {
			envvars = append(envvars, trimmed)
		}
	}
	return envvars
}
```

### container_config.go

> 包含了很多GPU相关的东西，简要看看

定义了一些NVIDIA GPU相关的常量，其中CAP_SYS_ADMIN权能允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等：

```go
const (
	envCUDAVersion          = "CUDA_VERSION"
	envNVRequirePrefix      = "NVIDIA_REQUIRE_"
	envNVRequireCUDA        = envNVRequirePrefix + "CUDA"
	envNVDisableRequire     = "NVIDIA_DISABLE_REQUIRE"
	envNVVisibleDevices     = "NVIDIA_VISIBLE_DEVICES"
	envNVMigConfigDevices   = "NVIDIA_MIG_CONFIG_DEVICES"
	envNVMigMonitorDevices  = "NVIDIA_MIG_MONITOR_DEVICES"
	envNVDriverCapabilities = "NVIDIA_DRIVER_CAPABILITIES"
)
const (
	capSysAdmin = "CAP_SYS_ADMIN"
)
const (
	deviceListAsVolumeMountsRoot = "/var/run/nvidia-container-devices"
)
```

定义了容器和nvidia的配置，注意下面这个Image是cuda image而非镜像，同时Nvidia字段判断容器是否是GPU容器：

```go
type nvidiaConfig struct {
	Devices            string
	MigConfigDevices   string
	MigMonitorDevices  string
	DriverCapabilities string
	Requirements []string
}
type containerConfig struct {
	Pid    int
	Rootfs string
	Image  image.CUDA
	Nvidia *nvidiaConfig
}
```

OCI-runtime spec中的一些定义的缩减，只保留了需要的部分：

```go
type Root struct {
	Path string `json:"path"`
}
type Process struct {
	Env          []string         `json:"env,omitempty"`
	Capabilities *json.RawMessage `json:"capabilities,omitempty" platform:"linux"`
}
type LinuxCapabilities struct {
	Bounding    []string `json:"bounding,omitempty" platform:"linux"`
	Effective   []string `json:"effective,omitempty" platform:"linux"`
	Inheritable []string `json:"inheritable,omitempty" platform:"linux"`
	Permitted   []string `json:"permitted,omitempty" platform:"linux"`
	Ambient     []string `json:"ambient,omitempty" platform:"linux"`
}
type Mount struct {
	Destination string   `json:"destination"`
	Type        string   `json:"type,omitempty" platform:"linux,solaris"`
	Source      string   `json:"source,omitempty"`
	Options     []string `json:"options,omitempty"`
}
type Spec struct {
	Version *string  `json:"ociVersion"`
	Process *Process `json:"process,omitempty"`
	Root    *Root    `json:"root,omitempty"`
	Mounts  []Mount  `json:"mounts,omitempty"`
}
```

定义了一个钩子状态信息的结构体：

```go
type HookState struct {
	Pid int `json:"pid,omitempty"`
	Bundle string `json:"bundle"`
	BundlePath string `json:"bundlePath"`
}
```

loadSpec函数用于加载指定路径的spec文件，并对spec中的一些字段进行记录：

```go
func loadSpec(path string) (spec *Spec) {
	f, err := os.Open(path)
	if err != nil {
		log.Panicln("could not open OCI spec:", err)
	}
	defer f.Close()
	if err = json.NewDecoder(f).Decode(&spec); err != nil {
		log.Panicln("could not decode OCI spec:", err)
	}
	if spec.Version == nil {
		log.Panicln("Version is empty in OCI spec")
	}
	if spec.Process == nil {
		log.Panicln("Process is empty in OCI spec")
	}
	if spec.Root == nil {
		log.Panicln("Root is empty in OCI spec")
	}
	return
}
```

isPrivileged函数用于判断容器是否是一个特权容器，也就是权能是否包含CAP_SYS_ADMIN，代码略（就是对spec中的特定字段解析加匹配

其他一些和GPU相关的函数省略

### capabilities.go

只有一个将string转换为相应的flag的函数：

```go
func capabilityToCLI(cap string) string {
	switch cap {
	case "compute":
		return "--compute"
	case "compat32":
		return "--compat32"
	case "graphics":
		return "--graphics"
	case "utility":
		return "--utility"
	case "video":
		return "--video"
	case "display":
		return "--display"
	case "ngx":
		return "--ngx"
	default:
		log.Panicln("unknown driver capability:", cap)
	}
	return ""
}
```

### main.go

> 读源码的时候感觉这部分代码风格和之前阅读的感觉很不一样，用gitlens查看了一下果然是不同的人写的，有一说一这个写的差一点意思（bushi

定义了一些flag：

```go 
var (
	debugflag   = flag.Bool("debug", false, "enable debug output")
	versionflag = flag.Bool("version", false, "enable version output")
	configflag  = flag.String("config", "", "configuration file")
)
```

exit函数用于处理发生panic时的情况，打印运行时错误信息，根据debugflag值选择打印调试堆栈信息：

```go
func exit() {
	if err := recover(); err != nil {
		if _, ok := err.(runtime.Error); ok {
			log.Println(err)
		}
		if *debugflag {
			log.Printf("%s", debug.Stack())
		}
		os.Exit(1)
	}
	os.Exit(0)
}
```

getCLIPath函数用于返回nvidia-container-cli的路径，如果config中指明了路径，则直接返回；未指明时则需要先设置PATH环境变量，然后在PATH中查找nvidia-container-cli：

```go
func getCLIPath(config config.ContainerCLIConfig) string {
	if config.Path != "" {
		return config.Path
	}
	if err := os.Setenv("PATH", lookup.GetPath(config.Root)); err != nil {
		log.Panicln("couldn't set PATH variable:", err)
	}
	path, err := exec.LookPath("nvidia-container-cli")
	if err != nil {
		log.Panicln("couldn't find binary nvidia-container-cli in", os.Getenv("PATH"), ":", err)
	}
	return path
}
```

getRootfsPath函数用于返回容器的rootfs路径：

```go
func getRootfsPath(config containerConfig) string {
	rootfs, err := filepath.Abs(config.Rootfs)
	if err != nil {
		log.Panicln(err)
	}
	return rootfs
}
```

doPrestart函数用于根据hook和nvidia-container-cli的config配置设置调用参数，并调用nvidia-container-cli，若hook config中的Nvidia字段为空时，表示该容器不是一个GPU容器，直接返回：

```go
func doPrestart() {
	var err error
	defer exit()
	log.SetFlags(0)
	hook, err := getHookConfig()
	if err != nil || hook == nil {
		log.Panicln("error getting hook config:", err)
	}
	cli := hook.NVIDIAContainerCLIConfig
	container := getContainerConfig(*hook)
	nvidia := container.Nvidia
	if nvidia == nil {
		return
	}
	if !hook.NVIDIAContainerRuntimeHookConfig.SkipModeDetection && info.ResolveAutoMode(&logInterceptor{}, hook.NVIDIAContainerRuntimeConfig.Mode, container.Image) != "legacy" {
		log.Panicln("invoking the NVIDIA Container Runtime Hook directly (e.g. specifying the docker --gpus flag) is not supported. Please use the NVIDIA Container Runtime (e.g. specify the --runtime=nvidia flag) instead.")
	}
	rootfs := getRootfsPath(container)
	args := []string{getCLIPath(cli)}
	if cli.Root != "" {
		args = append(args, fmt.Sprintf("--root=%s", cli.Root))
	}
	if cli.LoadKmods {
		args = append(args, "--load-kmods")
	}
	if cli.NoPivot {
		args = append(args, "--no-pivot")
	}
	if *debugflag {
		args = append(args, "--debug=/dev/stderr")
	} else if cli.Debug != "" {
		args = append(args, fmt.Sprintf("--debug=%s", cli.Debug))
	}
	if cli.Ldcache != "" {
		args = append(args, fmt.Sprintf("--ldcache=%s", cli.Ldcache))
	}
	if cli.User != "" {
		args = append(args, fmt.Sprintf("--user=%s", cli.User))
	}
	args = append(args, "configure")
	if cli.Ldconfig != "" {
		args = append(args, fmt.Sprintf("--ldconfig=%s", cli.Ldconfig))
	}
	if cli.NoCgroups {
		args = append(args, "--no-cgroups")
	}
	if len(nvidia.Devices) > 0 {
		args = append(args, fmt.Sprintf("--device=%s", nvidia.Devices))
	}
	if len(nvidia.MigConfigDevices) > 0 {
		args = append(args, fmt.Sprintf("--mig-config=%s", nvidia.MigConfigDevices))
	}
	if len(nvidia.MigMonitorDevices) > 0 {
		args = append(args, fmt.Sprintf("--mig-monitor=%s", nvidia.MigMonitorDevices))
	}
	for _, cap := range strings.Split(nvidia.DriverCapabilities, ",") {
		if len(cap) == 0 {
			break
		}
		args = append(args, capabilityToCLI(cap))
	}
	for _, req := range nvidia.Requirements {
		args = append(args, fmt.Sprintf("--require=%s", req))
	}
	args = append(args, fmt.Sprintf("--pid=%s", strconv.FormatUint(uint64(container.Pid), 10)))
	args = append(args, rootfs)
	env := append(os.Environ(), cli.Environment...)
	err = syscall.Exec(args[0], args, env)
	log.Panicln("exec failed:", err)
}
```

usage函数用于展示nvidia-container-runtime-hook的用法：

```go
func usage() {
	fmt.Fprintf(os.Stderr, "Usage of %s:\n", os.Args[0])
	flag.PrintDefaults()
	fmt.Fprintf(os.Stderr, "\nCommands:\n")
	fmt.Fprintf(os.Stderr, "  prestart\n        run the prestart hook\n")
	fmt.Fprintf(os.Stderr, "  poststart\n        no-op\n")
	fmt.Fprintf(os.Stderr, "  poststop\n        no-op\n")
}
```

main函数：

```go
func main() {
	flag.Usage = usage
	flag.Parse()
	if *versionflag {
		fmt.Printf("%v version %v\n", "NVIDIA Container Runtime Hook", info.GetVersionString())
		return
	}
	args := flag.Args()
	if len(args) == 0 {
		flag.Usage()
		os.Exit(2)
	}
	switch args[0] {
	case "prestart":
		doPrestart()
		os.Exit(0)
	case "poststart":
		fallthrough
	case "poststop":
		os.Exit(0)
	default:
		flag.Usage()
		os.Exit(2)
	}
}
```

## conclusion

nvidia-container-hook只是根据config去设置nvidia-contailer-cli的参数，再调用nvidia-container-cli。

