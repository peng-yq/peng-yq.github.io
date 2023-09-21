---
layout: post
title: NVIDIA-Container-Toolkit—源码阅读—pkg
subtitle: NVIDIA-Container-Toolkit—CodeReading—pkg
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---
## config

### enginee

#### docker

##### option.go

属于docker包。

定义的一些类型结构体：命名为builder应该是设计模式中的builder模式。

```go
type builder struct {
	logger logger.Interface
	path   string
}
// Option defines a function that can be used to configure the config builder
type Option func(*builder)
```

WithLogger和WithPath函数用于为config builder设置logger和path：比较常用的[函数式选项(Functional Options)编程模式](https://zhuanlan.zhihu.com/p/436468481)

```go
func WithLogger(logger logger.Interface) Option {
	return func(b *builder) {
		b.logger = logger
	}
}
func WithPath(path string) Option {
	return func(b *builder) {
		b.path = path
	}
}
```

build函数用于加载配置文件：

```go
func (b *builder) build() (*Config, error) {
	if b.path == "" {
		empty := make(Config)
		return &empty, nil
	}
	return b.loadConfig(b.path)
}
```

loadConfig用于从磁盘上加载配置文件并返回给Config对象。

```go
func (b *builder) loadConfig(config string) (*Config, error) {
	info, err := os.Stat(config)
	if os.IsExist(err) && info.IsDir() {
		return nil, fmt.Errorf("config file is a directory")
	}
	cfg := make(Config)
	if os.IsNotExist(err) {
		b.logger.Infof("Config file does not exist; using empty config")
		return &cfg, nil
	}
	b.logger.Infof("Loading config from %v", config)
	readBytes, err := ioutil.ReadFile(config)
	if err != nil {
		return nil, fmt.Errorf("unable to read config: %v", err)
	}
	reader := bytes.NewReader(readBytes)
	if err := json.NewDecoder(reader).Decode(&cfg); err != nil {
		return nil, err
	}
	return &cfg, nil
}
```

##### docker.go

docker中修改runtime：

```json
{
  "runtimes": {
    "runtime-name": {
      "path": "",
      "runtimeArgs": ["",
	""]
    }
  }
}
```

属于docker包。

定义的一些常量和结构体：

```go
const (
	defaultDockerRuntime = "runc"
)
// Config defines a docker config file.
// TODO: This should not be public, but we need to access it from the tests in tools/container/docker
type Config map[string]interface{}
```

New函数构造了一个docker config，常用的函数选项式编程。

> 但这里的返回类型我没太看明白

```go	
func New(opts ...Option) (engine.Interface, error) {
	b := &builder{}
	for _, opt := range opts {
		opt(b)
	}
	if b.logger == nil {
		b.logger = logger.New()
	}
	return b.build()
}
```

AddRuntime函数添加新的运行时到docker的config中：**但是daemon.json中是runtimeArgs**。

```go
func (c *Config) AddRuntime(name string, path string, setAsDefault bool) error {
	if c == nil {
		return fmt.Errorf("config is nil")
	}
	config := *c
	// Read the existing runtimes
	runtimes := make(map[string]interface{})
	if _, exists := config["runtimes"]; exists {
		runtimes = config["runtimes"].(map[string]interface{})
	}
	// Add / update the runtime definitions
	runtimes[name] = map[string]interface{}{
		"path": path,
		"args": []string{},
	}
	config["runtimes"] = runtimes
	if setAsDefault {
		config["default-runtime"] = name
	}
	*c = config
	return nil
}
```

DefaultRuntime函数返回docker配置文件中的默认运行时：

```go
func (c Config) DefaultRuntime() string {
	r, ok := c["default-runtime"].(string)
	if !ok {
		return ""
	}
	return r
}
```

RemoveRuntime删除docker配置文件中的运行时：如果删除的运行时是当前的默认运行时，则设置配置中的默认运行时为runc，然后删除掉runtimes字段中的对应运行时，若此时只有这一个运行时，则runtimes字段也删除。

```go
func (c *Config) RemoveRuntime(name string) error {
	if c == nil {
		return nil
	}
	config := *c
	if _, exists := config["default-runtime"]; exists {
		defaultRuntime := config["default-runtime"].(string)
		if defaultRuntime == name {
			config["default-runtime"] = defaultDockerRuntime
		}
	}
	if _, exists := config["runtimes"]; exists {
		runtimes := config["runtimes"].(map[string]interface{})
		delete(runtimes, name)
		if len(runtimes) == 0 {
			delete(config, "runtimes")
		}
	}
	*c = config
	return nil
}
```

Save函数将config写入指定路径：先转换为json再写入。

```go
func (c Config) Save(path string) (int64, error) {
	output, err := json.MarshalIndent(c, "", "    ")
	if err != nil {
		return 0, fmt.Errorf("unable to convert to JSON: %v", err)
	}

	n, err := engine.Config(path).Write(output)
	return int64(n), err
}
```

#### containerd

##### option.go

属于containerd包。

containerd更换运行时，containerd的配置文件是toml文件：

```toml
[plugins."io.containerd.grpc.v1.cri"]
[plugins."io.containerd.grpc.v1.cri".containerd]
  default_runtime_name = "nvidia"

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
      privileged_without_host_devices = false
      runtime_engine = ""
      runtime_root = ""
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
        BinaryName = "/usr/bin/nvidia-container-runtime"
```

定义的一些常量和结构体：

```go
const (
	defaultRuntimeType = "io.containerd.runc.v2"
)
type builder struct {
	logger               logger.Interface
	path                 string
	runtimeType          string
	useLegacyConfig      bool
	containerAnnotations []string
}
type Option func(*builder)
```

with系列函数不再赘述。

build函数用于构造config：

```go
func (b *builder) build() (engine.Interface, error) {
	if b.path == "" {
		return nil, fmt.Errorf("config path is empty")
	}
	if b.runtimeType == "" {
		b.runtimeType = defaultRuntimeType
	}
	config, err := b.loadConfig(b.path)
	if err != nil {
		return nil, fmt.Errorf("failed to load config: %v", err)
	}
	config.RuntimeType = b.runtimeType
	config.UseDefaultRuntimeName = !b.useLegacyConfig
	config.ContainerAnnotations = b.containerAnnotations
	version, err := config.parseVersion(b.useLegacyConfig)
	if err != nil {
		return nil, fmt.Errorf("failed to parse config version: %v", err)
	}
	switch version {
	case 1:
		return (*ConfigV1)(config), nil
	case 2:
		return config, nil
	}
	return nil, fmt.Errorf("unsupported config version: %v", version)
}
```

loadConfig从磁盘上加载containerd config：

```go
func (b *builder) loadConfig(config string) (*Config, error) {
	info, err := os.Stat(config)
	if os.IsExist(err) && info.IsDir() {
		return nil, fmt.Errorf("config file is a directory")
	}
	if os.IsNotExist(err) {
		b.logger.Infof("Config file does not exist; using empty config")
		config = "/dev/null"
	} else {
		b.logger.Infof("Loading config from %v", config)
	}
	tomlConfig, err := toml.LoadFile(config)
	if err != nil {
		return nil, err
	}
	cfg := Config{
		Tree: tomlConfig,
	}
	return &cfg, nil
}
```

parseVersion函数用于解析config的版本：

```go
func (c *Config) parseVersion(useLegacyConfig bool) (int, error) {
	defaultVersion := 2
	if useLegacyConfig {
		defaultVersion = 1
	}
	switch v := c.Get("version").(type) {
	case nil:
		switch len(c.Keys()) {
		case 0: // No config exists, or the config file is empty, use version inferred from containerd
			return defaultVersion, nil
		default: // A config file exists, has content, and no version is set
			return 1, nil
		}
	case int64:
		return int(v), nil
	default:
		return -1, fmt.Errorf("unsupported type for version field: %v", v)
	}
}
```

##### containerd.go

属于containerd包。

包括一个containerd的配置文件的结构体以及一个构造函数：

```go
type Config struct {
	*toml.Tree
	RuntimeType           string
	UseDefaultRuntimeName bool
	ContainerAnnotations  []string
}
func New(opts ...Option) (engine.Interface, error) {
	b := &builder{}
	for _, opt := range opts {
		opt(b)
	}
	if b.logger == nil {
		b.logger = logger.New()
	}
	return b.build()
}
```

##### config_v2.go

属于containerd包。

AddRuntime函数将运行时添加到containerd的配置文件中，也就是在开头的那些配置内容。

getRuntimeAnnotations函数获得配置文件中容器注解。

DefaultRuntime函数用于返回配置文件中的默认运行时，RemoveRuntime函数用于移除配置文件中的运行时，**这里两个注释也写错了**

Save函数将配置写入路径。

##### config_v1.go

和v2有细微区别。

#### crio

##### option.go

属于crio包。

builder、option以及with系列函数和docker部分一样，不再赘述。

build和loadConfig函数也类似，但是换成了toml。

##### crio.go

属于crio包。

#### config.go

属于engine包。

Write函数用于将内容写入配置文件中：如果路径为空，将内容写入标准输出。如果内容为空，尝试删除文件。在写入文件之前，会检查路径所在的目录是否存在，如果不存在，则尝试创建目录。函数返回写入的字节数和可能出现的错误。

```go
// Config represents a runtime config
type Config string
// Write writes the specified contents to a config file.
func (c Config) Write(output []byte) (int, error) {
	path := string(c)
	if path == "" {
		n, err := os.Stdout.Write(output)
		if err == nil {
			os.Stdout.WriteString("\n")
		}
		return n, err
	}
	if len(output) == 0 {
		err := os.Remove(path)
		if err != nil {
			return 0, fmt.Errorf("unable to remove empty file: %v", err)
		}
		return 0, nil
	}
	if dir := filepath.Dir(path); dir != "" {
		err := os.MkdirAll(dir, 0755)
		if err != nil {
			return 0, fmt.Errorf("unable to create directory %v: %v", dir, err)
		}
	}
	f, err := os.Create(path)
	if err != nil {
		return 0, fmt.Errorf("unable to open %v for writing: %v", path, err)
	}
	defer f.Close()
	return f.Write(output)
}
```

#### api.go

属于engine包，定义了一些接口。

```go
// Interface defines the API for a runtime config updater.
type Interface interface {
	DefaultRuntime() string
	AddRuntime(string, string, bool) error
	RemoveRuntime(string) error
	Save(string) (int64, error)
}
```

### ocihook

#### oci-hook.go

属于ocihook包。

CreateHook函数根据传入的路径创建一个钩子文件。它接受两个参数：hookFilePath 表示钩子文件的路径，nvidiaContainerRuntimeHookExecutablePath 表示 NVIDIA 容器运行时钩子可执行文件的路径。

generateOciHook是创建一个podman类型的钩子，先检查给定的路径是否在PATH里面，不再就添加，然后生成一个podman类型的hook，并返回：

```go
func generateOciHook(executablePath string) podmanHook {
	pathParts := []string{"/usr/local/sbin", "/usr/local/bin", "/usr/sbin", "/usr/bin", "/sbin", "/bin"}
	dir := filepath.Dir(executablePath)
	var found bool
	for _, pathPart := range pathParts {
		if pathPart == dir {
			found = true
			break
		}
	}
	if !found {
		pathParts = append(pathParts, dir)
	}
	envPath := "PATH=" + strings.Join(pathParts, ":")
	always := true
	hook := podmanHook{
		Version: "1.0.0",
		Stages:  []string{"prestart"},
		Hook: specHook{
			Path: executablePath,
			Args: []string{filepath.Base(executablePath), "prestart"},
			Env:  []string{envPath},
		},
		When: When{
			Always:   &always,
			Commands: []string{".*"},
		},
	}
	return hook
}
```

#### hooks.go

属于ocihook包，定义了一些不同规范的hook。

