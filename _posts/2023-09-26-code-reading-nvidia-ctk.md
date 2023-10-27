---
layout: post
title: NVIDIA-Container-Toolkit—源码阅读—ctk
subtitle: NVIDIA-Container-Toolkit—CodeReading—ctk
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## nvidia-ctk

### main.go

定义了一个options结构体，用于表示CLI启动的模式，包括Debug和Quiet模式：

```go
type options struct {
	Debug bool
	Quiet bool
}
```

main函数创建了一个叫nvidia-ctk的cli程序，flag包括--debug和--quiet，这两个flag和options中的两个变量相关联，用于设置logger的日志级别，分别是DebugLevel和ErrorLevel，前者用于调试目的的日志级别，会输出详细的调试信息，后者用于输出错误信息。

子命令包括就是如下的各种命令。

### config

#### create-default

只有一个create-default.go，属于defaultsubcommand包。

定义了一个command结构体：包含一个logger日志记录器，可以使用Debugf、Errorf、Info等方法进行日志记录。

```go
type command struct {
	logger logger.Interface
}
```

NewCommand使得创建的命令可使用logger调用相关的方法进行日志记录。

```go
func NewCommand(logger logger.Interface) *cli.Command {
	c := command{
		logger: logger,
	}
	return c.build()	
}
```

build函数创建了一个default命令，用于生成默认的nvidia-ctk配置文件，包括一个--output的flag，用于指定输出文件，若没有指定，默认为标准输出。

default在执行前会调用validateFlags函数，其实是调用了opts.Validate()检查参数，检查opts（options）的InPlace和Output（对应了--output）。

runc函数是default命令的执行函数，主要包括

- 创建或加载配置文件（toml文件），涉及到internal中的相关代码
- 创建输出目录和输出文件
- 写入保存配置文件

```go
func (m command) run(c *cli.Context, opts *flags.Options) error {
	cfgToml, err := config.New()
	if err != nil {
		return fmt.Errorf("unable to load or create config: %v", err)
	}
	if err := opts.EnsureOutputFolder(); err != nil {
		return fmt.Errorf("failed to create output directory: %v", err)
	}
	output, err := opts.CreateOutput()
	if err != nil {
		return fmt.Errorf("failed to open output file: %v", err)
	}
	defer output.Close()
	_, err = cfgToml.Save(output)
	if err != nil {
		return fmt.Errorf("failed to write output: %v", err)
	}
	return nil
}
```

#### flags

##### options.go

属于flags包。

定义了一个Options结构体：
```go
type Options struct {
	Config  string
	Output  string
	InPlace bool
}
```

Validate函数用于检查参数，--in-place和--output只能设置一个：

```go
func (o Options) Validate() error {
	if o.InPlace && o.Output != "" {
		return fmt.Errorf("cannot specify both --in-place and --output")
	}
	return nil
}
```

GetOutput函数用于返回配置中的有效输出位置：设置了--in-place则返回Config，否则返回Output。

```go
func (o Options) GetOutput() string {
	if o.InPlace {
		return o.Config
	}
	return o.Output
}
```

EnsureOutputFolder用于创建输出文件夹，未指定则不做任何事。

```go
func (o Options) EnsureOutputFolder() error {
	output := o.GetOutput()
	if output == "" {
		return nil
	}
	if dir := filepath.Dir(output); dir != "" {
		return os.MkdirAll(dir, 0755)
	}
	return nil
}
```

CreateOutput函数创建输出文件，标准输出或者具体的输出文件。

```go
func (o Options) CreateOutput() (io.WriteCloser, error) {
	output := o.GetOutput()
	if output == "" {
		return nullCloser{os.Stdout}, nil
	}
	return os.Create(output)
}
type nullCloser struct {
	io.Writer
}
func (d nullCloser) Close() error {
	return nil
}
```

#### config.go

属于config包。

定义了一些结构体：options包括Config、InPlace和Output

```go
type command struct {
	logger logger.Interface
}
type options struct {
	flags.Options
	sets cli.StringSlice
}
```

其他定义：

```go
type configToml config.Toml
var errInvalidConfigOption = errors.New("invalid config option")
var errInvalidFormat = errors.New("invalid format")
```

build函数同样定义了一个config命令，包括如下四个参数：

```shell
--config-file
--set
--in-place
--output
```

config包含default作为子命令。

run函数是config命令的真正执行函数：

- 创建一个新的配置文件config.toml
- 遍历opts.sets中的元素，即--set选项，将其值转换为键值对，并设置到config.toml文件中
- 确保输出文件夹的存在
- 创建输出文件
- 将配置文件写入保存到输出文件

setFlagToKeyValue函数用于将--set标志转换为键值对：例如--set "colors=red,green,blue" 则转换为colors [red green blue]

### info

属于info包。

仅一个info.go文件，定义了一个info命令，没有任何flag和子命令，但似乎**也没有该命令的运行函数**。

### runtime

#### runtime.go

属于runtime包。

定义了一个runtime命令，子命令包括configure。

#### configure

属于configure包，仅一个configure.go。

定义的一些常量和结构体：

```go
const (
	defaultRuntime = "docker"
	// defaultNVIDIARuntimeName is the default name to use in configs for the NVIDIA Container Runtime
	defaultNVIDIARuntimeName = "nvidia"
	// defaultNVIDIARuntimeExecutable is the default NVIDIA Container Runtime executable file name
	defaultNVIDIARuntimeExecutable          = "nvidia-container-runtime"
	defaultNVIDIARuntimeExpecutablePath     = "/usr/bin/nvidia-container-runtime"
	defaultNVIDIARuntimeHookExpecutablePath = "/usr/bin/nvidia-container-runtime-hook"
	defaultContainerdConfigFilePath = "/etc/containerd/config.toml"
	defaultCrioConfigFilePath       = "/etc/crio/crio.conf"
	defaultDockerConfigFilePath     = "/etc/docker/daemon.json"
)
type config struct {
	dryRun         bool
	runtime        string
	configFilePath string
	mode           string      // 看下面代码包括oci-hook和config-file两种模式
	hookFilePath   string
	nvidiaRuntime struct {
		name         string
		path         string
		hookPath     string
		setAsDefault bool
	}
}
```

定义了一个configure命令，flag如下：

```shell
--dry-run  		更新配置但不写入磁盘
--runtime  		运行时，包括docker、cri-o、containerd,默认为
docker
--config   		目标运行时的配置文件目录
--config-mode 	运行时的配置模式（支持多种配置机制的运行时）
--oci-hook-path --config-mode=oci-hook时的钩子执行文件路径
--nvidia-runtime-name 定义nvidia runtime的名字，默认是nvidia
--nvidia-runtime-path 定义nvidia runtime的路径，别名runtime-path,默认
是/usr/bin/nvidia-container-runtime
--nvidia-runtime-hook-path 定义nvidia runtime hook的路径，默
认是/usr/bin/nvidia-container-runtime-hook
--nvidia-set-as-default  是否将nvidia runtime设置为默认运行时，别名set-as-default
```

validateFlags函数是configure命令运行前的操作：主要是检查运行时是否是绝对路径、配置的模式以及目标运行时。

configureWrapper函数是configure的实际运行函数，用于加载nvidia runtime到目标运行时中。

```go
func (m command) configureWrapper(c *cli.Context, config *config) error {
	switch config.mode {
	case "oci-hook":
		return m.configureOCIHook(c, config)
	case "config-file":
		return m.configureConfigFile(c, config)
	}
	return fmt.Errorf("unsupported config-mode: %v", config.mode)
}
```

configureConfigFile更新目标运行时的配置文件来添加nvidia runtime的支持，原理是根据config.runtime去调用pkg/config/engine的docker、containerd和crio对其配置文件进行配置。

```go
func (m command) configureConfigFile(c *cli.Context, config *config) error {
	configFilePath := config.resolveConfigFilePath()
	var cfg engine.Interface
	var err error
	switch config.runtime {
	case "containerd":
		cfg, err = containerd.New(
			containerd.WithLogger(m.logger),
			containerd.WithPath(configFilePath),
		)
	case "crio":
		cfg, err = crio.New(
			crio.WithLogger(m.logger),
			crio.WithPath(configFilePath),
		)
	case "docker":
		cfg, err = docker.New(
			docker.WithLogger(m.logger),
			docker.WithPath(configFilePath),
		)
	default:
		err = fmt.Errorf("unrecognized runtime '%v'", config.runtime)
	}
	if err != nil || cfg == nil {
		return fmt.Errorf("unable to load config for runtime %v: %v", config.runtime, err)
	}
	err = cfg.AddRuntime(
		config.nvidiaRuntime.name,
		config.nvidiaRuntime.path,
		config.nvidiaRuntime.setAsDefault,
	)
	if err != nil {
		return fmt.Errorf("unable to update config: %v", err)
	}
	outputPath := config.getOuputConfigPath()
	n, err := cfg.Save(outputPath)
	if err != nil {
		return fmt.Errorf("unable to flush config: %v", err)
	}
	if outputPath != "" {
		if n == 0 {
			m.logger.Infof("Removed empty config from %v", outputPath)
		} else {
			m.logger.Infof("Wrote updated config to %v", outputPath)
		}
		m.logger.Infof("It is recommended that %v daemon be restarted.", config.runtime)
	}
	return nil
}
```

configureOCIHook用于将config.nvidiaRuntime.hookPath中的内容写入到config.hookFilePath中：

```go
func (m *command) configureOCIHook(c *cli.Context, config *config) error {
	err := ocihook.CreateHook(config.hookFilePath, config.nvidiaRuntime.hookPath)
	if err != nil {
		return fmt.Errorf("error creating OCI hook: %v", err)
	}
	return nil
}
```

resolveConfigFilePath函数和getOuputConfigPath函数则是返回配置文件目录：

```go
func (c *config) resolveConfigFilePath() string {
	if c.configFilePath != "" {
		return c.configFilePath
	}
	switch c.runtime {
	case "containerd":
		return defaultContainerdConfigFilePath
	case "crio":
		return defaultCrioConfigFilePath
	case "docker":
		return defaultDockerConfigFilePath
	}
	return ""
}
func (c *config) getOuputConfigPath() string {
	if c.dryRun {
		return ""
	}
	return c.resolveConfigFilePath()
}
```

### hook

#### hook.go

属于hook包。

创建了一个hook命令，包含子命令

#### chmod

属于chmod包，就一个chmod.go。

config结构体：

```go
type config struct {
	paths         cli.StringSlice
	mode          string
	containerSpec string
}
```

定义了一个chmod命令，包括--path、--mode和--container-spec三个flag。

在chmod命令执行前，会执行validateFlags函数来就检查命令参数，主要是，mode和path是否为空。

run和internel中的部分联系紧密，后续补充。

getPaths函数用于判断root目录下是否有paths中的路径：

```go
func (m command) getPaths(root string, paths []string) []string {
	var pathsInRoot []string
	for _, f := range paths {
		path := filepath.Join(root, f)
		if _, err := os.Stat(path); err != nil {
			m.logger.Debugf("Skipping path %q: %v", path, err)
			continue
		}
		pathsInRoot = append(pathsInRoot, path)
	}
	return pathsInRoot
}
```

#### create-symlinks

属于symlinks包，只有一个create-symlinks.go。

config结构体：

```go
type config struct {
	hostRoot      string
	filenames     cli.StringSlice
	links         cli.StringSlice
	containerSpec string
}
```

创建了一个create-symlinks命令，包含的flag值：

```shell
--host-root		 用于指定主机文件系统上的根目录以解析符号链接
--csv-filename 	 用于指定要处理的（CSV）文件名，用户可以多次使用该标志来指定多个文件名
--link			 链接名，可多个
--container-spec 用于指定 OCI 容器规范的路径
```

changeRoot函数首先判断path是否为绝对路径（相对路径直接返回），如果是绝对路径，再解析出相对于当前路径current的相对路径，最后在与new进行拼接组成新的绝对路径并返回。

**这里是相对路径的话为啥直接就返回path了呢**

```go
func changeRoot(current string, new string, path string) (string, error) {
	if !filepath.IsAbs(path) {
		return path, nil
	}
	relative := path
	if current != "" {
		r, err := filepath.Rel(current, path)
		if err != nil {
			return "", err
		}
		relative = r
	}
	return filepath.Join(new, relative), nil
}
```

Locate函数用于获取指定文件名的符号链接目标。它首先检查文件是否为符号链接，如果是，则返回符号链接的目标路径，否则返回一个空切片。

```go
func (m command) Locate(filename string) ([]string, error) {
	info, err := os.Lstat(filename)
	if err != nil {
		return nil, fmt.Errorf("failed to get file info: %v", info)
	}
	if info.Mode()&os.ModeSymlink == 0 {
		m.logger.Debugf("%v is not a symlink", filename)
		return nil, nil
	}
	target, err := os.Readlink(filename)
	if err != nil {
		return nil, fmt.Errorf("error checking symlink: %v", err)
	}
	m.logger.Debugf("Resolved link: '%v' => '%v'", filename, target)
	return []string{target}, nil
}
```

createLink函数用于创建一个linkpath到target的链接，linkpath是link在containerRoot中的路径：

```go
func (m command) createLink(created map[string]bool, hostRoot string, containerRoot string, target string, link string) error {
	linkPath, err := changeRoot(hostRoot, containerRoot, link)
	if err != nil {
		m.logger.Warningf("Failed to resolve path for link %v relative to %v: %v", link, containerRoot, err)
	}
	if created[linkPath] {
		m.logger.Debugf("Link %v already created", linkPath)
		return nil
	}
	targetPath, err := changeRoot(hostRoot, "/", target)
	if err != nil {
		m.logger.Warningf("Failed to resolve path for target %v relative to %v: %v", target, "/", err)
	}
	m.logger.Infof("Symlinking %v to %v", linkPath, targetPath)
	err = os.MkdirAll(filepath.Dir(linkPath), 0755)
	if err != nil {
		return fmt.Errorf("failed to create directory: %v", err)
	}
	err = os.Symlink(target, linkPath)
	if err != nil {
		return fmt.Errorf("failed to create symlink: %v", err)
	}
	return nil
}
```

run函数看的有点迷惑，总体来说是加载容器状态，解析 CSV 文件，定位符号链接，创建符号链接，并处理自定义链接。

#### update-ldcache

属于ldcache包，仅有一个upate-ldcache.go。

似乎和gpu的一些东西有关，创建了一个update-ldcache命令。







