---
layout: post
title: NVIDIA-Container-Toolkit—源码阅读—runtime
subtitle: NVIDIA-Container-Toolkit—CodeReading—runtime
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - cloud native
---

## modifier

### stable.go

定义了一个stableRuntimeModifier结构体用于加入hook：

```go
type stableRuntimeModifier struct {
	logger                         logger.Interface
	nvidiaContainerRuntimeHookPath string
}
```

NewStableRuntimeModifier构造函数：

```go
func NewStableRuntimeModifier(logger logger.Interface, nvidiaContainerRuntimeHookPath string) oci.SpecModifier {
	m := stableRuntimeModifier{
		logger:                         logger,
		nvidiaContainerRuntimeHookPath: nvidiaContainerRuntimeHookPath,
	}
	return &m
}
```

Modify函数将hook应用/添加到spec中：

```go
func (m stableRuntimeModifier) Modify(spec *specs.Spec) error {
	if spec.Hooks != nil {
		for _, hook := range spec.Hooks.Prestart {
			if isNVIDIAContainerRuntimeHook(&hook) {
				m.logger.Infof("Existing nvidia prestart hook (%v) found in OCI spec", hook.Path)
				return nil
			}
		}
	}
	path := m.nvidiaContainerRuntimeHookPath
	m.logger.Infof("Using prestart hook path: %v", path)
	args := []string{filepath.Base(path)}
	if spec.Hooks == nil {
		spec.Hooks = &specs.Hooks{}
	}
	spec.Hooks.Prestart = append(spec.Hooks.Prestart, specs.Hook{
		Path: path,
		Args: append(args, "prestart"),
	})
	return nil
}
```

### hook_remover.go

定义了一个nvidiaContainerRuntimeHookRemover结构体，用于移除hook：

```go
type nvidiaContainerRuntimeHookRemover struct {
	logger logger.Interface
}
var _ oci.SpecModifier = (*nvidiaContainerRuntimeHookRemover)(nil)
```

isNVIDIAContainerRuntimeHook函数用于判断spec中的hook是否是定义的hook，通过path是否匹配来判断：

```go
func isNVIDIAContainerRuntimeHook(hook *specs.Hook) bool {
	bins := map[string]struct{}{
		config.NVIDIAContainerRuntimeHookExecutable: {},
		config.NVIDIAContainerToolkitExecutable:     {},
	}
	_, exists := bins[filepath.Base(hook.Path)]
	return exists
}
```

Modify函数用于移除spec中的hook中的定义hook，构造新的hook进行替换：

```go
func (m nvidiaContainerRuntimeHookRemover) Modify(spec *specs.Spec) error {
	if spec == nil {
		return nil
	}
	if spec.Hooks == nil {
		return nil
	}
	if len(spec.Hooks.Prestart) == 0 {
		return nil
	}
	var newPrestart []specs.Hook
	for _, hook := range spec.Hooks.Prestart {
		if isNVIDIAContainerRuntimeHook(&hook) {
			m.logger.Debugf("Removing hook %v", hook)
			continue
		}
		newPrestart = append(newPrestart, hook)
	}
	if len(newPrestart) != len(spec.Hooks.Prestart) {
		m.logger.Debugf("Updating 'prestart' hooks to %v", newPrestart)
		spec.Hooks.Prestart = newPrestart
	}
	return nil
}
```

## oci

### args.go

定义了一个常量：也就是容器的常用配置文件名

```go
const (
	specFileName = "config.json"
)
```

IsBundleFlag函数检查参数是否是“--bundle”或者“-b”：

```go
func IsBundleFlag(arg string) bool {
	if !strings.HasPrefix(arg, "-") {
		return false
	}
	trimmed := strings.TrimLeft(arg, "-")
	return trimmed == "b" || trimmed == "bundle"
}
```

HasCreateSubcommand函数：按照代码逻辑只有previousWasBundle为false，并且当前命令为“create”才为true，“-b create”这种情况则会返回false

>  PS：runc在创建容器时的命令为create --bundle

```go
func HasCreateSubcommand(args []string) bool {
	var previousWasBundle bool
	for _, a := range args {
		if !previousWasBundle && IsBundleFlag(a) {
			previousWasBundle = true
			continue
		}
		if !previousWasBundle && a == "create" {
			return true
		}
		previousWasBundle = false
	}
	return false
}
```

GetSpecFilePath函数用于解析出给定bundle的配置文件路径：

```go
func GetSpecFilePath(bundleDir string) string {
	specFilePath := filepath.Join(bundleDir, specFileName)
	return specFilePath
}
```

GetBundleDirFromArgs函数用于解析出参数中的bundle目录：

```go
func GetBundleDirFromArgs(args []string) (string, error) {
	var bundleDir string
	for i := 0; i < len(args); i++ {
		param := args[i]
		parts := strings.SplitN(param, "=", 2)
		if !IsBundleFlag(parts[0]) {
			continue
		}
		if len(parts) == 2 {
			bundleDir = parts[1]
			continue
		}
		if i+1 < len(args) {
			bundleDir = args[i+1]
			i++
			continue
		}
		return "", fmt.Errorf("bundle option requires an argument")
	}
	return bundleDir, nil
}
```

GetBundleDir再做一次封装，隐藏底层函数实现细节：

```go
func GetBundleDir(args []string) (string, error) {
	bundleDir, err := GetBundleDirFromArgs(args)
	if err != nil {
		return "", fmt.Errorf("error getting bundle dir from args: %v", err)
	}
	return bundleDir, nil
}
```

### runtime_low_level.go

findRuntime函数在PATH中查找给定的candidates中的runtime，只返回找到的第一个runtime

```go
func findRuntime(logger logger.Interface, candidates []string) (string, error) {
	if len(candidates) == 0 {
		return "", fmt.Errorf("at least one runtime candidate must be specified")
	}
	locator := lookup.NewExecutableLocator(logger, "/")
	for _, candidate := range candidates {
		logger.Debugf("Looking for runtime binary '%v'", candidate)
		targets, err := locator.Locate(candidate)
		if err == nil && len(targets) > 0 {
			logger.Debugf("Found runtime binary '%v'", targets)
			return targets[0], nil
		}
		logger.Debugf("Runtime binary '%v' not found: %v (targets=%v)", candidate, err, targets)
	}
	return "", fmt.Errorf("no runtime binary found from candidate list: %v", candidates)
}
```

NewLowLevelRuntime函数：从给定的低级运行时中选择构建一个运行时返回

```go
func NewLowLevelRuntime(logger logger.Interface, candidates []string) (Runtime, error) {
	runtimePath, err := findRuntime(logger, candidates)
	if err != nil {
		return nil, fmt.Errorf("error locating runtime: %v", err)
	}
	logger.Infof("Using low-level runtime %v", runtimePath)
	return NewRuntimeForPath(logger, runtimePath)
}
```

### runtime_path.go

定义了一些结构体：

```go
type pathRuntime struct {
	logger      logger.Interface
	path        string
	execRuntime Runtime
}
var _ Runtime = (*pathRuntime)(nil)
```

 NewRuntimeForPath函数用于检查给定path是否为可执行文件，并构建一个runtime类型返回：

```go
func NewRuntimeForPath(logger logger.Interface, path string) (Runtime, error) {
	info, err := os.Stat(path)
	if err != nil {
		return nil, fmt.Errorf("invalid path '%v': %v", path, err)
	}
	if info.IsDir() || info.Mode()&0111 == 0 {
		return nil, fmt.Errorf("specified path '%v' is not an executable file", path)
	}
	shim := pathRuntime{
		logger:      logger,
		path:        path,
		execRuntime: syscallExec{},
	}
	return &shim, nil
}
```

Exec用于执行runtime参数：低级运行时（runc...）+ 参数

```go
func (s pathRuntime) Exec(args []string) error {
	runtimeArgs := []string{s.path}
	if len(args) > 1 {
		runtimeArgs = append(runtimeArgs, args[1:]...)
	}
	return s.execRuntime.Exec(runtimeArgs)
}
```

### runtime_modifier.go

定义的结构体：

```go
type modifyingRuntimeWrapper struct {
	logger   logger.Interface
	runtime  Runtime
	ociSpec  Spec
	modifier SpecModifier
}
var _ Runtime = (*modifyingRuntimeWrapper)(nil)
```

NewModifyingRuntimeWrapper函数：根据给定内容构建一个Runtime，如果modifier为空，则直接使用指定的低级运行时

```go
func NewModifyingRuntimeWrapper(logger logger.Interface, runtime Runtime, spec Spec, modifier SpecModifier) Runtime {
	if modifier == nil {
		logger.Infof("Using low-level runtime with no modification")
		return runtime
	}
	rt := modifyingRuntimeWrapper{
		logger:   logger,
		runtime:  runtime,
		ociSpec:  spec,
		modifier: modifier,
	}
	return &rt
}
```

Exec函数用于让modifyingRuntimeWrapper.runtime执行参数，只有在参数含有create时才对spec应用modifier，否则直接将参数传给低级运行时：

```go
func (r *modifyingRuntimeWrapper) Exec(args []string) error {
	if HasCreateSubcommand(args) {
		err := r.modify()
		if err != nil {
			return fmt.Errorf("could not apply required modification to OCI specification: %v", err)
		}
		r.logger.Infof("Applied required modification to OCI specification")
	} else {
		r.logger.Infof("No modification of OCI specification required")
	}
	r.logger.Infof("Forwarding command to runtime")
	return r.runtime.Exec(args)
}
```

modify函数用于将spec应用特定的modifier：

```go
func (r *modifyingRuntimeWrapper) modify() error {
	_, err := r.ociSpec.Load()
	if err != nil {
		return fmt.Errorf("error loading OCI specification for modification: %v", err)
	}
	err = r.ociSpec.Modify(r.modifier)
	if err != nil {
		return fmt.Errorf("error modifying OCI spec: %v", err)
	}
	err = r.ociSpec.Flush()
	if err != nil {
		return fmt.Errorf("error writing modified OCI specification: %v", err)
	}
	return nil
}
```

### runtime_syscall_exec.go

定义的结构体：

```go
type syscallExec struct{}
var _ Runtime = (*syscallExec)(nil)
```

Exec函数用于执行命令，没太多好讲的：

> update：最后一个return写的很严谨，因为在调用exec后，调用的那个进程会替换syscall.Exec，不会有任何返回，即使err为nil也是错误的，因为在成功执行调用后，不会执行后续代码。

```go
func (r syscallExec) Exec(args []string) error {
	err := syscall.Exec(args[0], args, os.Environ())
	if err != nil {
		return fmt.Errorf("could not exec '%v': %v", args[0], err)
	}
	return fmt.Errorf("unexpected return from exec '%v'", args[0])
}
```

### runtime.go

就定义了一个Runtime接口，Exec用于执行命令：

```go
type Runtime interface {
	Exec([]string) error
}
```

> spec相关的代码写的都挺难看懂的（

### runtime_mock.go

用于测试的

### spec_memory.go

定义的结构体：这个是opencontainers官方提供的runtime-spec，也就是容器的config模板

```go
type memorySpec struct {
	*specs.Spec
}
```

NewMemorySpec创建一个memorySpec结构体：

```go
func NewMemorySpec(spec *specs.Spec) Spec {
	s := memorySpec{
		Spec: spec,
	}
	return &s
}
```

Load、Flush和Modify三个对spec操作的函数：

```go
func (s *memorySpec) Load() (*specs.Spec, error) {
	return s.Spec, nil
}
func (s *memorySpec) Flush() error {
	return nil
}
func (s *memorySpec) Modify(m SpecModifier) error {
	if s.Spec == nil {
		return fmt.Errorf("cannot modify nil spec")
	}
	return m.Modify(s.Spec)
}
```

LookupEnv函数：用于在容器的spec config中查找环境变量是否含有key，有则返回值和true，主要和runtime-spec中的process的env变量有关

```go
func (s memorySpec) LookupEnv(key string) (string, bool) {
	if s.Spec == nil || s.Spec.Process == nil {
		return "", false
	}
	for _, env := range s.Spec.Process.Env {
		if !strings.HasPrefix(env, key) {
			continue
		}
		parts := strings.SplitN(env, "=", 2)
		if parts[0] == key {
			if len(parts) < 2 {
				return "", true
			}
			return parts[1], true
		}
	}
	return "", false
}
```

### spec.go

定义的一些结构体：主要是对runtime-spec（config）各种操作的接口函数：

```go
type SpecModifier interface {
	Modify(*specs.Spec) error
}
type Spec interface {
	Load() (*specs.Spec, error)
	Flush() error
	Modify(SpecModifier) error
	LookupEnv(string) (string, bool)
}
```

NewSpec用于根据给定的参数创建一个spec文件：

```go
func NewSpec(logger logger.Interface, args []string) (Spec, error) {
	bundleDir, err := GetBundleDir(args)
	if err != nil {
		return nil, fmt.Errorf("error getting bundle directory: %v", err)
	}
	logger.Debugf("Using bundle directory: %v", bundleDir)
	ociSpecPath := GetSpecFilePath(bundleDir)
	logger.Infof("Using OCI specification file path: %v", ociSpecPath)
	ociSpec := NewFileSpec(ociSpecPath)
	return ociSpec, nil
}
```

### spec_mock.go

通过go:generate moq -stub -out spec_mock.go . Spec对spec.go自动生成的文件，用于测试。

### spec_file.go

定义的一些结构体：

```go
type fileSpec struct {
	memorySpec
	path string
}
var _ Spec = (*fileSpec)(nil)
```

NewFileSpec函数用于构造一个filespec结构体：

```go
func NewFileSpec(filepath string) Spec {
	oci := fileSpec{
		path: filepath,
	}
	return &oci
}
```

LoadFrom函数读取内容并解析成spec类型：

```go
func LoadFrom(reader io.Reader) (*specs.Spec, error) {
	decoder := json.NewDecoder(reader)
	var spec specs.Spec
	err := decoder.Decode(&spec)
	if err != nil {
		return nil, fmt.Errorf("error reading OCI specification: %v", err)
	}
	return &spec, nil
}
```

Load函数用于加载spec：

```go
func (s *fileSpec) Load() (*specs.Spec, error) {
	specFile, err := os.Open(s.path)
	if err != nil {
		return nil, fmt.Errorf("error opening OCI specification file: %v", err)
	}
	defer specFile.Close()
	spec, err := LoadFrom(specFile)
	if err != nil {
		return nil, fmt.Errorf("error loading OCI specification from file: %v", err)
	}
	s.Spec = spec
	return s.Spec, nil
}
```

Modify函数用于修改spec内容：

```go
func (s *fileSpec) Modify(m SpecModifier) error {
	return s.memorySpec.Modify(m)
}
```

flushTo函数用于将oci spec写入指定路径的文件：

```go
func flushTo(spec *specs.Spec, writer io.Writer) error {
	if spec == nil {
		return nil
	}
	encoder := json.NewEncoder(writer)
	err := encoder.Encode(spec)
	if err != nil {
		return fmt.Errorf("error writing OCI specification: %v", err)
	}
	return nil
}
```

Flush函数对上述函数进行了一次封装，并添加了创建文件等操作：

> 这里方法的接收者为s fileSpec，而非s *fileSpec，可能是fileSpec后续还可能用

```go
func (s fileSpec) Flush() error {
	if s.Spec == nil {
		return fmt.Errorf("no OCI specification loaded")
	}
	specFile, err := os.Create(s.path)
	if err != nil {
		return fmt.Errorf("error opening OCI specification file: %v", err)
	}
	defer specFile.Close()
	return flushTo(s.Spec, specFile)
}
```

### state.go

定义了一个state结构体存储spec中的state内容：

```go
type State specs.State
```

specs.State包括：

```go
// State holds information about the runtime state of the container.
type State struct {
	// Version is the version of the specification that is supported.
	Version string `json:"ociVersion"`
	// ID is the container ID
	ID string `json:"id"`
	// Status is the runtime status of the container.
	Status ContainerState `json:"status"`
	// Pid is the process ID for the container process.
	Pid int `json:"pid,omitempty"`
	// Bundle is the path to the container's bundle directory.
	Bundle string `json:"bundle"`
	// Annotations are key values associated with the container.
	Annotations map[string]string `json:"annotations,omitempty"`
}
```

ReadContainerState函数用于解析state结构体：

```go
func ReadContainerState(reader io.Reader) (*State, error) {
	var s State
	d := json.NewDecoder(reader)
	if err := d.Decode(&s); err != nil {
		return nil, fmt.Errorf("failed to decode container state: %v", err)
	}
	return &s, nil
}
```

LoadContainerState函数用于从指定的文件中加载state结构体：

```go
func LoadContainerState(filename string) (*State, error) {
	if filename == "" || filename == "-" {
		return ReadContainerState(os.Stdin)
	}
	inputFile, err := os.Open(filename)
	if err != nil {
		return nil, fmt.Errorf("failed to open file: %v", err)
	}
	defer inputFile.Close()
	return ReadContainerState(inputFile)
}
```

LoadSpec函数用于将容器的bundle.json解析为spec类型，bundle目录从state中解析：

```go
func (s *State) LoadSpec() (*specs.Spec, error) {
	specFilePath := GetSpecFilePath(s.Bundle)
	specFile, err := os.Open(specFilePath)
	if err != nil {
		return nil, fmt.Errorf("failed to open OCI spec file: %v", err)
	}
	defer specFile.Close()
	spec, err := LoadFrom(specFile)
	if err != nil {
		return nil, fmt.Errorf("failed to load OCI spec: %v", err)
	}
	return spec, nil
}
```

GetContainerRoot函数用于返回容器的root目录，state中有root字段（包括path：绝对路径或者相对于bundle的相对路径，以及readonly（bool））：

```go
func (s *State) GetContainerRoot() (string, error) {
	spec, err := s.LoadSpec()
	if err != nil {
		return "", err
	}
	var containerRoot string
	if spec.Root != nil {
		containerRoot = spec.Root.Path
	}
	if filepath.IsAbs(containerRoot) {
		return containerRoot, nil
	}
	return filepath.Join(s.Bundle, containerRoot), nil
}
```

## nvidia-container-runtime

### main.go

main函数很简单，创建一个新的runtime，并执行传入的参数。

```go
func main() {
	r := runtime.New()
	err := r.Run(os.Args)
	if err != nil {
		os.Exit(1)
	}
}
```

## runtime

### api.go

定义的一些结构体：

```go
type rt struct {
	logger       *Logger
	modeOverride string
}
type Interface interface {
	Run([]string) error
}
type Option func(*rt)
```

New和WithModeOverride都是两构造函数，没太多好说的。

```go
func New(opts ...Option) Interface {
	r := rt{}
	for _, opt := range opts {
		opt(&r)
	}
	if r.logger == nil {
		r.logger = NewLogger()
	}
	return &r
}
func WithModeOverride(mode string) Option {
	return func(r *rt) {
		r.modeOverride = mode
	}
}
```

### logger.go

定义的一些结构体：

1. Logger结构体包括：
   - logger库的接口，即Info、Debugf等日志输出函数
   - previousLogger，类型同上
   - 日志文件logFiles
2. loggerConfig结构体包括：
   - file，应该是日志文件名
   - format，应该是日志格式
   - debug，应该是是否开启debug模式
   - version

```go
type Logger struct {
	logger.Interface
	previousLogger logger.Interface
	logFiles       []*os.File
}
type loggerConfig struct {
	file    string
	format  string
	debug   bool
	version bool
}
```

NewLogger函数用于构建一个空的logger，只对Interface进行了设置：

```go
func NewLogger() *Logger {
	return &Logger{
		Interface: logrus.New(),
	}
}
```

parseArgs函数用于检查args参数，并返回一个loggerConfig结构体（对file、format、debug和version的值均进行了设置）：

```go
func parseArgs(args []string) loggerConfig {
	c := loggerConfig{}
	expected := map[string]*string{
		"log-format": &c.format,
		"log":        &c.file,
	}
	found := make(map[string]bool)
	for i := 0; i < len(args); i++ {
		if len(found) == 4 {
			break
		}
		param := args[i]
		parts := strings.SplitN(param, "=", 2)
		trimmed := strings.TrimLeft(parts[0], "-")
		// If this is not a flag we continue
		if parts[0] == trimmed {
			continue
		}
		// Check the version flag
		if trimmed == "version" {
			c.version = true
			found["version"] = true
			// For the version flag we don't process any other flags
			continue
		}
		// Check the debug flag
		if trimmed == "debug" {
			c.debug = true
			found["debug"] = true
			continue
		}
		destination, exists := expected[trimmed]
		if !exists {
			continue
		}
		var value string
		if len(parts) == 2 {
			value = parts[2]
		} else if i+1 < len(args) {
			value = args[i+1]
			i++
		} else {
			continue
		}
		*destination = value
		found[trimmed] = true
	}
	return c
}
```

getLevel函数则用于返回当前的日志记录级别：

1. c中的debug为true时，则日志记录级别为DebugLevel
2. 否则解析给定的logLevel

```go
func (c loggerConfig) getLevel(logLevel string) (logrus.Level, error) {
	if c.debug {
		return logrus.DebugLevel, nil
	}
	if logLevel, err := logrus.ParseLevel(logLevel); err == nil {
		return logLevel, nil
	}
	return logrus.InfoLevel, fmt.Errorf("invalid log-level '%v'", logLevel)
}
```

createLogFile函数用于创建日志文件，权限是0644（读写 读 读）。

```go
func createLogFile(filename string) (*os.File, error) {
	if filename == "" || filename == os.DevNull {
		return nil, nil
	}
	if dir := filepath.Dir(filepath.Clean(filename)); dir != "." {
		err := os.MkdirAll(dir, 0755)
		if err != nil {
			return nil, err
		}
	}
	return os.OpenFile(filename, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
}
```

Reset函数用于关闭日志文件，并将日志记录器恢复为UpdateLogger之前的状态：

1. 也就是l.Interface = previous；l.previousLogger = nil；l.logFiles = nil

```go
func (l *Logger) Reset() error {
	defer func() {
		previous := l.previousLogger
		if previous == nil {
			previous = logrus.New()
		}
		l.Interface = previous
		l.previousLogger = nil
		l.logFiles = nil
	}()
	var errs []error
	for _, f := range l.logFiles {
		err := f.Close()
		if err != nil {
			errs = append(errs, err)
		}
	}
	var err error
	for _, e := range errs {
		if err == nil {
			err = e
			continue
		}
		return fmt.Errorf("%v; %w", e, err)
	}
	return err
}
```

Update用于构建一个日志记录器：

1. parseArgs解析参数构建loggerConfig结构体
2. getLevel解析出日志文件记录级别
3. createLogFile创建日志文件
4. 对日志记录器logger进行设置，包括日志记录级别、日志格式和日志输出
5. 更新调用Update的logger

### runtime_factory.go

newModeModifier根据mode返回不同的modifier，这里只关注legacy模式，也就是hook模式：

```go
func newModeModifier(logger logger.Interface, mode string, cfg *config.Config, ociSpec oci.Spec, image image.CUDA) (oci.SpecModifier, error) {
	switch mode {
	case "legacy":
		return modifier.NewStableRuntimeModifier(logger, cfg.NVIDIAContainerRuntimeHookConfig.Path), nil
	case "csv":
		return modifier.NewCSVModifier(logger, cfg, image)
	case "cdi":
		return modifier.NewCDIModifier(logger, cfg, ociSpec)
	}
	return nil, fmt.Errorf("invalid runtime mode: %v", cfg.NVIDIAContainerRuntimeConfig.Mode)
}
```

newSpecModifier返回所有的modifier，这里省略。

newNVIDIAContainerRuntime用于构造runtime：

1. 解析低级运行时，如果命令参数没有create，则直接返回低级运行时
2. 解析spec文件
3. 解析modifier
4. 构造runtime

> nvidia这个项目只有runc命令有“create”时会应用修改器，否则直接调用低级运行时，避免了多次修改spec文件；相比下awslabs写的那个oci-add-hooks显得比较玩具了，每次都会修改spec

```go
func newNVIDIAContainerRuntime(logger logger.Interface, cfg *config.Config, argv []string) (oci.Runtime, error) {
	lowLevelRuntime, err := oci.NewLowLevelRuntime(logger, cfg.NVIDIAContainerRuntimeConfig.Runtimes)
	if err != nil {
		return nil, fmt.Errorf("error constructing low-level runtime: %v", err)
	}
	if !oci.HasCreateSubcommand(argv) {
		logger.Debugf("Skipping modifier for non-create subcommand")
		return lowLevelRuntime, nil
	}
	ociSpec, err := oci.NewSpec(logger, argv)
	if err != nil {
		return nil, fmt.Errorf("error constructing OCI specification: %v", err)
	}
	specModifier, err := newSpecModifier(logger, cfg, ociSpec)
	if err != nil {
		return nil, fmt.Errorf("failed to construct OCI spec modifier: %v", err)
	}
	r := oci.NewModifyingRuntimeWrapper(
		logger,
		lowLevelRuntime,
		ociSpec,
		specModifier,
	)
	return r, nil
}
```

### runtime.go

hasVersionFlag函数：用于判断参数中是否有-version

```go
func hasVersionFlag(args []string) bool {
	for i := 0; i < len(args); i++ {
		param := args[i]
		parts := strings.SplitN(param, "=", 2)
		trimmed := strings.TrimLeft(parts[0], "-")
		if parts[0] == trimmed {
			continue
		}
		if trimmed == "version" {
			return true
		}
	}
	return false
}
```

Run函数，runtime的运行函数：

1. 根据参数决定是否打印出运行时的参数信息
2. 加载配置文件：/etc/nvidia-container-runtime/config.toml
3. 根据配置文件内容，更新日志记录器
4. 更新cfg
5. 构建runtime
6. 执行参数，回到了runtime_modifier.go中的Exec，只有在create的时候才会应用修改器（这样一看oci-add-hooks项目每次都对config.json进行了修改

```go
func (r rt) Run(argv []string) (rerr error) {
	defer func() {
		if rerr != nil {
			r.logger.Errorf("%v", rerr)
		}
	}()
	printVersion := hasVersionFlag(argv)
	if printVersion {
		fmt.Printf("%v version %v\n", "NVIDIA Container Runtime", info.GetVersionString(fmt.Sprintf("spec: %v", specs.Version)))
	}
	cfg, err := config.GetConfig()
	if err != nil {
		return fmt.Errorf("error loading config: %v", err)
	}
	r.logger.Update(
		cfg.NVIDIAContainerRuntimeConfig.DebugFilePath,
		cfg.NVIDIAContainerRuntimeConfig.LogLevel,
		argv,
	)
	defer func() {
		if rerr != nil {
			r.logger.Errorf("%v", rerr)
		}
		r.logger.Reset()
	}()
	if r.modeOverride != "" {
		cfg.NVIDIAContainerRuntimeConfig.Mode = r.modeOverride
	}
	cfg.NVIDIACTKConfig.Path = config.ResolveNVIDIACTKPath(r.logger, cfg.NVIDIACTKConfig.Path)
	cfg.NVIDIAContainerRuntimeHookConfig.Path = config.ResolveNVIDIAContainerRuntimeHookPath(r.logger, cfg.NVIDIAContainerRuntimeHookConfig.Path)
	configJSON, err := json.MarshalIndent(cfg, "", "  ")
	if err == nil {
		r.logger.Infof("Running with config:\n%v", string(configJSON))
	} else {
		r.logger.Infof("Running with config:\n%+v", cfg)
	}
	r.logger.Debugf("Command line arguments: %v", argv)
	runtime, err := newNVIDIAContainerRuntime(r.logger, cfg, argv)
	if err != nil {
		return fmt.Errorf("failed to create NVIDIA Container Runtime: %v", err)
	}
	if printVersion {
		fmt.Print("\n")
	}
	return runtime.Exec(argv)
}
```

