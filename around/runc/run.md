# RunC 源码通读指南之 Run

<!--toc-->

## 概述

RunC 是 Docker 贡献出来，按照 OCI 运行时标准制定的一种具体实现，是一个可执行应用程序包工具。可通过OCI 镜像格式标准文件包 bundles 来创建和运行容器以及对容器的生命周期管理[详情参考](https://github.com/opencontainers/runc)。 本文《附录一》附有使用 runc 工具命令来运行容器的示例可供参考。

从 docker 容器的整个架构和执行流程视角来看是由 containerd-shim 组件调用了 runc 来创建和运行容器，其创建容器时的配置文件/run/docker/libcontainerd/​\$containerID/config.json,进行读取转化为 spec 标准作为 runc 创建容器全局配置参数。 

本文将重点聚焦在 runc run 命令的执行代码的整个流程上（容器的创建至容器的运行）的解析，而有些细节实现比如 namespace 、cgroup 、网络等将在此套系列文档有详细介绍可供参阅。

从 runc run 代码执行结构可简单分为为四块执行组成部分：1. Run 命令执行入口  2. 容器对象创建  3. 容器执行 init 初始化 4. 容器用户程序与运行 ,本文下面将顺序地进行展开解析。



## CLI run 执行入口

Cli app 的 run 命令执行，读取命令参数和读取与转化 config.json 为 spec 标准配置。

!FILENAME run.go:65

```go
Action: func(context *cli.Context) error {
    // 命令参数校验
		if err := checkArgs(context, 1, exactArgs); err != nil {
			return err
		}
    // 获取"pid-file"传参配置，转化为绝对路径
		if err := revisePidFile(context); err != nil {
			return err
		}
    // 读取 config.json
		spec, err := setupSpec(context)
		if err != nil {
			return err
		}
   // +startContainer() 启动容器
		status, err := startContainer(context, spec, CT_ACT_RUN, nil)
		if err == nil {
			// exit with the container's exit status so any external supervisor is
			// notified of the exit with the correct exit status.
			os.Exit(status)
		}
		return err
	},
```

StartContainer() 启动容器顶层代码执行过程：

- 读取传入的 container id 参数
- 通过 spec 配置与 id 等传参创建容器对象
- 构建 runner 启动器并执行

!FILENAME utils_linux.go:430

```go
func startContainer(context *cli.Context, spec *specs.Spec, action CtAct, criuOpts *libcontainer.CriuOpts) (int, error) {
  // 通过 spec 创建容器结构，在 createContainer 中将 spec 转换为了 runc 的 container config*
	id := context.Args().First() //命令行输入的container id参数
	if id == "" {
		return -1, errEmptyID
	}

	notifySocket := newNotifySocket(context, os.Getenv("NOTIFY_SOCKET"), id)
	if notifySocket != nil {
		notifySocket.setupSpec(context, spec)
	}

  // +创建容器对象
	container, err := createContainer(context, id, spec)   
	if err != nil {
		return -1, err
	}

	if notifySocket != nil {
		err := notifySocket.setupSocket()
		if err != nil {
			return -1, err
		}
	}

	// Support on-demand socket activation by passing file descriptors into the container init process.
	listenFDs := []*os.File{}
	if os.Getenv("LISTEN_FDS") != "" {
		listenFDs = activation.Files(false)
	}

	logLevel := "info"
	if context.GlobalBool("debug") {
		logLevel = "debug"
	}

  // 构建 runner 启动器
	r := &runner{
		enableSubreaper: !context.Bool("no-subreaper"),    
		shouldDestroy:   true,                             
		container:       container,                         // 容器
		listenFDs:       listenFDs,                        
		notifySocket:    notifySocket,                      
		consoleSocket:   context.String("console-socket"),    
		detach:          context.Bool("detach"),           
		pidFile:         context.String("pid-file"),       
		preserveFDs:     context.Int("preserve-fds"),       
		action:          action,                            // CT_ACT_RUN 执行标志
		criuOpts:        criuOpts,                          // criu 热迁移选项
		init:            true,                              // 用于设置 process.Init 字段
    logLevel:        logLevel,                          // 日志级别 default info
	} 
  return r.run(spec.Process)  // run() 启动
}
```



## Container 容器对象创建

RunC 代码实例化容器对象的代码模式是通过工厂方法实现，实例化 LinuxFactory 类型工厂和  linuxContainer 类型容器对象。

!FILENAME utils_linux.go:230

```go
func createContainer(context *cli.Context, id string, spec *specs.Spec) (libcontainer.Container, error) {
	rootlessCg, err := shouldUseRootlessCgroupManager(context)
	if err != nil {
		return nil, err
	}
  // spec 转换 config
	config, err := specconv.CreateLibcontainerConfig(&specconv.CreateOpts{
		CgroupName:       id,
		UseSystemdCgroup: context.GlobalBool("systemd-cgroup"),
		NoPivotRoot:      context.Bool("no-pivot"),
		NoNewKeyring:     context.Bool("no-new-keyring"),
		Spec:             spec,
		RootlessEUID:     os.Geteuid() != 0,
		RootlessCgroups:  rootlessCg,
	})
	if err != nil {
		return nil, err
	}

	factory, err := loadFactory(context)   //+  创建工厂实例
	if err != nil {
		return nil, err
	}
	return factory.Create(id, config)      //+  工厂实例化容器对象 
}
```

loadFactory() 创建容器工厂 libcontainer.Factory ，配置 cgroup 管理器 、root path 、intel RDT 管理器 、user map、热迁移路径。

!FILENAME utils_linux.go:31

```go
// loadFactory returns the configured factory instance for execing containers.
func loadFactory(context *cli.Context) (libcontainer.Factory, error) {
	root := context.GlobalString("root")   
	abs, err := filepath.Abs(root)             // 根目录绝对路径
	if err != nil {
		return nil, err
	}

	cgroupManager := libcontainer.Cgroupfs     // cgroup Manger 默认为 Cgroupfs 
 	rootlessCg, err := shouldUseRootlessCgroupManager(context)
	if err != nil {
		return nil, err
	}
	if rootlessCg {
		cgroupManager = libcontainer.RootlessCgroupfs
	}
	if context.GlobalBool("systemd-cgroup") {  // systemd-cgroup 是否全局指定开启
		if systemd.UseSystemd() {
			cgroupManager = libcontainer.SystemdCgroups
		} else {
			return nil, fmt.Errorf("systemd cgroup flag passed, but systemd support for managing cgroups is not available")
		}
	}

	intelRdtManager := libcontainer.IntelRdtFs   // intel RDT 
	if !intelrdt.IsCatEnabled() && !intelrdt.IsMbaEnabled() {
		intelRdtManager = nil
	}

	newuidmap, err := exec.LookPath("newuidmap")  // newuidmap 容器内外 uid 映射
	if err != nil {
		newuidmap = ""
	}
	newgidmap, err := exec.LookPath("newgidmap")  // newgidmap 容器内外 uid 映射
	if err != nil {
		newgidmap = ""utils_linux.go
	}

  // 创建容器工厂
	return libcontainer.New(abs, cgroupManager, intelRdtManager,
		libcontainer.CriuPath(context.GlobalString("criu")),
		libcontainer.NewuidmapPath(newuidmap),
		libcontainer.NewgidmapPath(newgidmap))
}
```

创建LinuxFactory类型的factoy对象，用于容器对象的创建工厂

!FILENAME libcontainer/factory_linux.go:131

```go
func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
	if root != "" {
    //确保存储容器状态的根目录创建
		if err := os.MkdirAll(root, 0700); err != nil {     
			return nil, newGenericError(err, SystemError)
		}
	}
	l := &LinuxFactory{
    // 存储容器状态的根目录，默认"/run/runc/"
		Root:      root,
    // 指向当前的 exe 程序，即 runc 本身
		InitPath:  "/proc/self/exe",   
    // os.Args[0] 是当前 runc 的路径，本质上和 InitPath 是一样的，即 runc init
		InitArgs:  []string{os.Args[0], "init"},
    // 配置校验器对象
		Validator: validate.New(),
    // 热迁移路径设置
		CriuPath:  "criu",
	}
	Cgroupfs(l)  //为 LinuxFactory 配置 NewCgroupsManage实现 func
	for _, opt := range options {
		if opt == nil {
			continue
		}
		if err := opt(l); err != nil {
			return nil, err
		}
	}
	return l, nil
}
```

基于全局配置，容器工厂创建 linuxContainer 容器对象

!FILENAME libcontainer/factory_linux.go:188

```go
func (l *LinuxFactory) Create(id string, config *configs.Config) (Container, error) {
  // 确保containerRoot目录被创建
	if l.Root == "" {
		return nil, newGenericError(fmt.Errorf("invalid root"), ConfigInvalid)
	}
  // 校验参数
	if err := l.validateID(id); err != nil {
		return nil, err
	}
	if err := l.Validator.Validate(config); err != nil {
		return nil, newGenericError(err, ConfigInvalid)
	}
  // 容器根路径
	containerRoot, err := securejoin.SecureJoin(l.Root, id)
	if err != nil {
		return nil, err
	}
	if _, err := os.Stat(containerRoot); err == nil {
		return nil, newGenericError(fmt.Errorf("container with id exists: %v", id), IdInUse)
	} else if !os.IsNotExist(err) {
		return nil, newGenericError(err, SystemError)
	}
	if err := os.MkdirAll(containerRoot, 0711); err != nil {
		return nil, newGenericError(err, SystemError)
	}
	if err := os.Chown(containerRoot, unix.Geteuid(), unix.Getegid()); err != nil {
		return nil, newGenericError(err, SystemError)
	}
  
  // 创建 linux 容器结构
	c := &linuxContainer{
		id:            id,               // 容器 ID
		root:          containerRoot,    // 容器状态文件存放目录，默认是 /run/runc/$容器ID/
		config:        config,           // 容器配置
		initPath:      l.InitPath,       // /proc/self/exe，就是runc
		initArgs:      l.InitArgs,       // 即runc init
		criuPath:      l.CriuPath,       // 热迁移path "criu"
    // Uid / Gid 配置
		newuidmapPath: l.NewuidmapPath,
		newgidmapPath: l.NewgidmapPath,
    // cgroup配置
		cgroupManager: l.NewCgroupsManager(config.Cgroups, nil),
	}
  // 英特尔RDT（资源调配技术）配置
	if intelrdt.IsCatEnabled() || intelrdt.IsMbaEnabled() {
		c.intelRdtManager = l.NewIntelRdtManager(config, id, "")
	}
	c.state = &stoppedState{c: c}       // 开始置为"stopped"状态
	return c, nil
}
```



## Runner 执行和 init 容器初始化

根据 startContainer() 顶层的执行流程，在创建容器化对象后，构建 runner 对象并执行run()。此 run() 则是容器的运行阶段的入口，它包含两大块执行过程：Start() 容器环境初始化启动 和 exec () 容器用户程序执行，start() 初始化启动过程比较复杂一些容器的初始工作都在此过程中，包含cgroup 、namespace等等核心初始化工作(专文说明)，而 start() 过程则比较简单仅取消bootstrap进程的阻塞态，让其完成执行entrypoint。后面将进行详细的描述。

Runner.run() 创建初始化进程对象，调用 linuxContainer.run() 运行进程

utils_linux.go:271

```go
func (r *runner) run(config *specs.Process) (int, error) {
	var err error
	defer func() {
		if err != nil {
			r.destroy()
		}
	}()
	if err = r.checkTerminal(config); err != nil {
		return -1, err
	}
  // +基于config 创建 init process对象 (指定 "run init")
	process, err := newProcess(*config, r.init, r.logLevel)
	if err != nil {
		return -1, err
	}
	if len(r.listenFDs) > 0 {
		process.Env = append(process.Env, fmt.Sprintf("LISTEN_FDS=%d", len(r.listenFDs)), "LISTEN_PID=1")
		process.ExtraFiles = append(process.ExtraFiles, r.listenFDs...)
	}
	baseFd := 3 + len(process.ExtraFiles)
	for i := baseFd; i < baseFd+r.preserveFDs; i++ {
		_, err = os.Stat(fmt.Sprintf("/proc/self/fd/%d", i))
		if err != nil {
			return -1, errors.Wrapf(err, "please check that preserved-fd %d (of %d) is present", i-baseFd, r.preserveFDs)
		}
		process.ExtraFiles = append(process.ExtraFiles, os.NewFile(uintptr(i), "PreserveFD:"+strconv.Itoa(i)))
	}
	rootuid, err := r.container.Config().HostRootUID()    
	if err != nil {
		return -1, err
	}
	rootgid, err := r.container.Config().HostRootGID()
	if err != nil {
		return -1, err
	}
	var (
		detach = r.detach || (r.action == CT_ACT_CREATE)
	)
	
  // 处理io和tty相关配置
	handler := newSignalHandler(r.enableSubreaper, r.notifySocket)
	tty, err := setupIO(process, rootuid, rootgid, config.Terminal, detach, r.consoleSocket)
	if err != nil {
		return -1, err
	}
	defer tty.Close()

	switch r.action {
	case CT_ACT_CREATE:
		err = r.container.Start(process)
	case CT_ACT_RESTORE:
		err = r.container.Restore(process, r.criuOpts)
	case CT_ACT_RUN:                                    
    err = r.container.Run(process)                   // +调用linuxContainer.run()
	default:
		panic("Unknown action")
	}
	if err != nil {
		return -1, err
	}
	if err = tty.waitConsole(); err != nil {
		r.terminate(process)
		return -1, err
	}
	if err = tty.ClosePostStart(); err != nil {
		r.terminate(process)
		return -1, err
	}
	if r.pidFile != "" {
		if err = createPidFile(r.pidFile, process); err != nil {
			r.terminate(process)
			return -1, err
		}
	}
	status, err := handler.forward(process, tty, detach)
	if err != nil {
		r.terminate(process)
	}
	if detach {
		return 0, nil
	}
	r.destroy()
	return status, err
}
```

newProcess() 基于 spec 配置初始化并创建libconatiner.process 进程对象返回

!FILENAME utils_linux.go:106

```go
func newProcess(p specs.Process, init bool, logLevel string) (*libcontainer.Process, error) {
	lp := &libcontainer.Process{
		Args: p.Args,
		Env:  p.Env,
    User:            fmt.Sprintf("%d:%d", p.User.UID, p.User.GID),   // uid:gid
		Cwd:             p.Cwd,
		Label:           p.SelinuxLabel,            // selinux 标签
		NoNewPrivileges: &p.NoNewPrivileges, 
		AppArmorProfile: p.ApparmorProfile,         // Apparmor 配置 
		Init:            init,                      // runc init
		LogLevel:        logLevel,
	}

	if p.ConsoleSize != nil {                     // console 窗口设置
		lp.ConsoleWidth = uint16(p.ConsoleSize.Width)
		lp.ConsoleHeight = uint16(p.ConsoleSize.Height)
	}
 
	if p.Capabilities != nil {                    // capabilities 配置
		lp.Capabilities = &configs.Capabilities{}
		lp.Capabilities.Bounding = p.Capabilities.Bounding
		lp.Capabilities.Effective = p.Capabilities.Effective
		lp.Capabilities.Inheritable = p.Capabilities.Inheritable
		lp.Capabilities.Permitted = p.Capabilities.Permitted
		lp.Capabilities.Ambient = p.Capabilities.Ambient
	}
	for _, gid := range p.User.AdditionalGids {    // gid 配置
		lp.AdditionalGroups = append(lp.AdditionalGroups, strconv.FormatUint(uint64(gid), 10))
	}
	for _, rlimit := range p.Rlimits {             // limit 资源限制配置
		rl, err := createLibContainerRlimit(rlimit)
		if err != nil {
			return nil, err
		}
		lp.Rlimits = append(lp.Rlimits, rl)
	}
	return lp, nil
}
```

LinuxContainer.Run() 为上层 CT_ACT_RUN 执行流程调用：

- Start()  Init 进程执行启动
- exec()  用户进程EntryPoint 执行

libcontainer/container_linux.go:250

```go
func (c *linuxContainer) Run(process *Process) error {
	if err := c.Start(process); err != nil {   // +容器环境 init 启动
		return err
	}
	if process.Init {
		return c.exec()                          // +EntryPoint 执行
	}
	return nil
}
```

调用 linuxContainer.start() 

libcontainer/container_linux.go:233

```go
func (c *linuxContainer) Start(process *Process) error {
  //...
  if err := c.start(process); err != nil {        // +linuxContainer.start() 运行 process
		if process.Init {
			c.deleteExecFifo()
		}
		return err
	}
	return nil
}
```

linuxContainer.start() 为一个完整的上层容器实始化执行流程代码，首先通过上面传参的 process 进程对象创建 “父” 进程并启动（核心逻辑处），完成启动后保存容器状态到 state.json 文件（默认"/run/runc/$containerID/ state.json"），最后如果容器有定义运行后勾子将被调用执行。

libcontainer/container_linux.go:335

```go
func (c *linuxContainer) start(process *Process) error {
  // +创建的父进程
	parent, err := c.newParentProcess(process)
	if err != nil {
		return newSystemErrorWithCause(err, "creating new parent process")
	}
	parent.forwardChildLogs()
  // +启动父进程  
	if err := parent.start(); err != nil {
		// terminate the process to ensure that it properly is reaped.
		if err := ignoreTerminateErrors(parent.terminate()); err != nil {
			logrus.Warn(err)
		}
		return newSystemErrorWithCause(err, "starting container process")
	}
	// 容器启动状态 state 保存（写入 state.json 文件）
	c.created = time.Now().UTC()
	if process.Init {
		c.state = &createdState{
			c: c,
		}
		state, err := c.updateState(parent)
		if err != nil {
			return err
		}
		c.initProcessStartTime = state.InitProcessStartTime

		if c.config.Hooks != nil {
			s, err := c.currentOCIState()
			if err != nil {
				return err
			}
      // postStrat 容器运行后勾子执行
			for i, hook := range c.config.Hooks.Poststart {
				if err := hook.Run(s); err != nil {
					if err := ignoreTerminateErrors(parent.terminate()); err != nil {
						logrus.Warn(err)
					}
					return newSystemErrorWithCausef(err, "running poststart hook %d", i)
				}
			}
		}
	}
	return nil
}
```

newParentProcess() 创建父进程的过程:

- 创建父子进程通信的 pipe （ bootstrapData 配置数据用此传递）
- 创建 cmd 对象  （ 此处的cmd 对象就是执行 runc init ,后面有详述 ）
- 返回 newInitProcess() initProcess 对象

libcontainer/container_linux.go:441

```go
func (c *linuxContainer) newParentProcess(p *Process) (parentProcess, error) {
  // 创建用于父子进程通信的 pipe
	parentInitPipe, childInitPipe, err := utils.NewSockPair("init")
	if err != nil {
		return nil, newSystemErrorWithCause(err, "creating new init pipe")
	}
	messageSockPair := filePair{parentInitPipe, childInitPipe}

  //...
  // +创建父进程的 cmd
	cmd, err := c.commandTemplate(p, childInitPipe, childLogPipe)  
	if err != nil {
		return nil, newSystemErrorWithCause(err, "creating new command template")
	}

  //...
  // +返回标准 init 进程
	return c.newInitProcess(p, cmd, messageSockPair, logFilePair)   
}
```

创建父进程的 cmd 对象

libcontainer/container_linux.go:473

```go
func (c *linuxContainer) commandTemplate(p *Process, childInitPipe *os.File, childLogPipe *os.File) (*exec.Cmd, error) {
  // 这里可以看到 cmd 就是 runc init
	cmd := exec.Command(c.initPath, c.initArgs[1:]...)
	cmd.Args[0] = c.initArgs[0]
  
  // 将设置给容器 entrypoint 的 std 流给了 runc init 命令，这些流最终会通过 runc init 传递给 entrypoint
	cmd.Stdin = p.Stdin
	cmd.Stdout = p.Stdout
	cmd.Stderr = p.Stderr
	cmd.Dir = c.config.Rootfs
	if cmd.SysProcAttr == nil {
		cmd.SysProcAttr = &syscall.SysProcAttr{}
	}
	cmd.Env = append(cmd.Env, fmt.Sprintf("GOMAXPROCS=%s", os.Getenv("GOMAXPROCS")))
	cmd.ExtraFiles = append(cmd.ExtraFiles, p.ExtraFiles...)
	if p.ConsoleSocket != nil {
		cmd.ExtraFiles = append(cmd.ExtraFiles, p.ConsoleSocket)
		cmd.Env = append(cmd.Env,
			fmt.Sprintf("_LIBCONTAINER_CONSOLE=%d", stdioFdCount+len(cmd.ExtraFiles)-1),
		)
	}
  // 这个 childInitPipe 用于跟父进程通信（父进程就是当前这个 runc 进程）
	cmd.ExtraFiles = append(cmd.ExtraFiles, childInitPipe)
   // 通过环境变量 _LIBCONTAINER_INITPIPE 把 fd 号传递给 runc init，由于 std 流会占用前三个 fd 编号（0，1，2）
    // 所以 fd 要加上 3（stdioFdCount）
	cmd.Env = append(cmd.Env,
		fmt.Sprintf("_LIBCONTAINER_INITPIPE=%d", stdioFdCount+len(cmd.ExtraFiles)-1),
		fmt.Sprintf("_LIBCONTAINER_STATEDIR=%s", c.root),
	)

  //...
	return cmd, nil
}
```

返回 initProcess 对象

libcontainer/container_linux.go:512

```go
func (c *linuxContainer) newInitProcess(p *Process, cmd *exec.Cmd, messageSockPair, logFilePair filePair) (*initProcess, error) {
  // 这里通过环境变量 _LIBCONTAINER_INITTYPE 设置 init 类型为 standard（initStandard
	cmd.Env = append(cmd.Env, "_LIBCONTAINER_INITTYPE="+string(initStandard))
	nsMaps := make(map[configs.NamespaceType]string)
	for _, ns := range c.config.Namespaces {
		if ns.Path != "" {
			nsMaps[ns.Type] = ns.Path
		}
	}
	_, sharePidns := nsMaps[configs.NEWPID]
   // 构造 namespace 配置，然后序列化成字节数据
	data, err := c.bootstrapData(c.config.Namespaces.CloneFlags(), nsMaps)
	if err != nil {
		return nil, err
	}
	init := &initProcess{   
		cmd:             cmd,                       //  cmd 对象,也就是 run int
		messageSockPair: messageSockPair,           //  通信 sockpair
		logFilePair:     logFilePair,
		manager:         c.cgroupManager,     
		intelRdtManager: c.intelRdtManager,
		config:          c.newInitConfig(p),
		container:       c,                         // 容器对象
		process:         p,                         // 传参的进程对象
		bootstrapData:   data,                      // namespaces 配置序列化数据
		sharePidns:      sharePidns,
	}
	c.initProcess = init        
	return init, nil                              //返回 init 进程对象
}
```

InitProcess.start() 则是容器运行最核心的代码执行逻辑块：

当前执行进程我们称之为“bootstrap进程“，cmd.Start() 实则执行了 "runc init" 命令，同时也激活了nsenter 模块C 代码的优先执行配置namespace （详细可参阅《RunC 源码通读指南之 NameSpace》），完后返回执行 init Go 代码部分完成后续的初始化工作，最后向管道 exec.fifo 进行写操作，init 进程进入阻塞状态等待信号完成容器内的entrypoint执行。

!FILENAME libcontainer/process_linux.go:282

```go
func (p *initProcess) start() error {
    defer p.messageSockPair.parent.Close()
  //  当前执行空间进程称为bootstrap进程
  //  启动了 cmd，即启动了 runc init 命令,创建 runc init 子进程 
  //  同时也激活了C代码nsenter模块的执行（为了 namespace 的设置 clone 了三个进程parent、child、init）
  //  C 代码执行后返回 go 代码部分,最后的 init 子进程为了好区分此处命名为" nsInit "（即配置了Namespace的init）
  //  runc init go代码为容器初始化其它部分(网络、rootfs、路由、主机名、console、安全等)
    err := p.cmd.Start()  
   
    //...
    // 为进程 runc init 应用 Cgroup （p.cmd.Process.Pid()）
    if err := p.manager.Apply(p.pid()); err != nil {
        return newSystemErrorWithCause(err, "applying cgroup configuration for process")
    }
    
    //...
    // messageSockPair 管道写入 bootstrapData 
    if _, err := io.Copy(p.messageSockPair.parent, p.bootstrapData); err != nil {
        return newSystemErrorWithCause(err, "copying bootstrap data to pipe")
    }
  
    // 获取 nsInit pid
    childPid, err := p.getChildPid()
    if err != nil {
        return newSystemErrorWithCause(err, "getting the final child's pid from pipe")
    }

    //... 
    // 为 nsInit 进程应用 Cgroup 
    if err := p.manager.Apply(childPid); err != nil {
        return newSystemErrorWithCause(err, "applying cgroup configuration for process")
    }
    // 为 child 进程应用 intel RDT 
    if p.intelRdtManager != nil {
        if err := p.intelRdtManager.Apply(childPid); err != nil {
            return newSystemErrorWithCause(err, "applying Intel RDT configuration for process")
        }
    }
 
  // 设置 cgroup namesapce
	if p.config.Config.Namespaces.Contains(configs.NEWCGROUP) && p.config.Config.Namespaces.PathOf(configs.NEWCGROUP) == "" {
		if _, err := p.messageSockPair.parent.Write([]byte{createCgroupns}); err != nil {
			return newSystemErrorWithCause(err, "sending synchronization value to init process")
		}
	}

	// 等待子进程退出
	if err := p.waitForChildExit(childPid); err != nil {
		return newSystemErrorWithCause(err, "waiting for our first child to exit")
	}

  //...
  // 创建网络接口
	if err := p.createNetworkInterfaces(); err != nil {
		return newSystemErrorWithCause(err, "creating network interfaces")
	}
  // 发送 initConfig 进程配置到 messageSockPair.parent 管道
	if err := p.sendConfig(); err != nil {
		return newSystemErrorWithCause(err, "sending config to init process")
	}
	var (
		sentRun    bool
		sentResume bool
	)

  // 解析runc init子进程的所有同步消息，当io.EOF返回
	ierr := parseSync(p.messageSockPair.parent, func(sync *syncT) error {
		switch sync.Type {
		case procReady:  // 
      // 配置 limit 资源限制
			if err := setupRlimits(p.config.Rlimits, p.pid()); err != nil {
				return newSystemErrorWithCause(err, "setting rlimits for ready process")
			}
			// // prestart hook 启动前执行勾子
			if !p.config.Config.Namespaces.Contains(configs.NEWNS) {
				// Setup cgroup before prestart hook, so that the prestart hook could apply cgroup permissions.åå
				if err := p.manager.Set(p.config.Config); err != nil {
					return newSystemErrorWithCause(err, "setting cgroup config for ready process")
				}
				if p.intelRdtManager != nil {
					if err := p.intelRdtManager.Set(p.config.Config); err != nil {
						return newSystemErrorWithCause(err, "setting Intel RDT config for ready process")
					}
				}

				if p.config.Config.Hooks != nil {
					s, err := p.container.currentOCIState()
					if err != nil {
						return err
					}
					// initProcessStartTime hasn't been set yet.
					s.Pid = p.cmd.Process.Pid
					s.Status = "creating"
					for i, hook := range p.config.Config.Hooks.Prestart {
						if err := hook.Run(s); err != nil {
							return newSystemErrorWithCausef(err, "running prestart hook %d", i)
						}
					}
				}
			}
			// 与子进程 runC init 同步
			if err := writeSync(p.messageSockPair.parent, procRun); err != nil {
				return newSystemErrorWithCause(err, "writing syncT 'run'")
			}
			sentRun = true
		case procHooks:         // prochook 勾子执行
      //  配置 cgroup
 			if err := p.manager.Set(p.config.Config); err != nil {
				return newSystemErrorWithCause(err, "setting cgroup config for procHooks process")
			}
      // 配置 intel RDT 资源管理
			if p.intelRdtManager != nil {
				if err := p.intelRdtManager.Set(p.config.Config); err != nil {
					return newSystemErrorWithCause(err, "setting Intel RDT config for procHooks process")
				}
			}
			if p.config.Config.Hooks != nil {
		    //...
        // 执行勾子定义任务
				for i, hook := range p.config.Config.Hooks.Prestart {
					if err := hook.Run(s); err != nil {
						return newSystemErrorWithCausef(err, "running prestart hook %d", i)
					}
				}
			}
			// 与子进程 runc-init 同步
			if err := writeSync(p.messageSockPair.parent, procResume); err != nil {
				return newSystemErrorWithCause(err, "writing syncT 'resume'")
			}
			sentResume = true
		default:
			return newSystemError(fmt.Errorf("invalid JSON payload from child"))
		}
   
		return nil
	})
 
  //...
	return nil
}
```



## RunC Init 容器初始化

**Nsenter 模块C 代码执行逻辑**

RunC init 命令执行 Go 调用 C 代码称之 preamble ,即在 import nsenter 模块时机将会在 Go 的 runtime 启动之前，先执行此先导代码块，nsenter 的初始化 init(void) 方法内对 nsexec() 调用 

!FILENAME init.go:10

```go
 _ "github.com/opencontainers/runc/libcontainer/nsenter"
```

!FILENAME libcontainer/nsenter/nsenter.go:3

```go
package nsenter
/*
#cgo CFLAGS: -Wall
extern void nsexec();
void __attribute__((constructor)) init(void) {
    nsexec();
}
*/
import "C"
```

nsexec clone 三个进程:

- 第一个进程称为“ parent ”，读取 bootstrapData 并解析为 Config，对 User map 设置，并通过消息协调后面两个进程的运行管理，在收到 grandchild 回复任务完成消息后退出。
- 第二个进程称为“ child ”,由 Parent 创建，完成 namespace 的设置 ，fork 出 grandChild 进程并发送给Parent 后发送任务完成消息后退出。
- 第三个进程称为“ grandChild ”或" init "，进行最后的环境准备工作(sid、uid、gid、cgroup namespace)，执行完成后return 至 init Go runtime 代码处继续执行最后进入 go 代码。

!FILENAME libcontainer/nsenter/nsexec.c:575

```c
void nsexec(void)
{
  //...
    switch (setjmp(env)) {
      //...
        case JUMP_PARENT:{
          //..
        }
        case JUMP_CHILD:{
          //...
        }
         case JUMP_INIT:{
           //...
         }
      //...
  }
```

注：此块详细代码解析专文请参阅《RunC 源码通读指南之 NameSpace》

**RunC init (Go 代码部分)执行逻辑**

创建 factory 对象，执行 factory.StartInitialization() => linuxStandardInit.Init() 完成容器的相关初始化配置(网络/路由、rootfs、selinux、console、主机名、apparmor、Sysctl、seccomp、capability 等)

!FILENAME init.go:15

```go
func init() {
 //...
var initCommand = cli.Command{
    Name:  "init",
    Usage: `initialize the namespaces and launch the process (do not call it outside of runc)`,
    Action: func(context *cli.Context) error {
        factory, _ := libcontainer.New("")                          // +创建 factory 对象
        if err := factory.StartInitialization(); err != nil {       // +执行 init 初始化
            os.Exit(1)
        }
        panic("libcontainer: container init failed to exec")
    },
}
```

libcontainer.New() 创建 factory 对象返回

!FILENAME libcontainer/factory_linux.go:131

```go
func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
  //...
    l := &LinuxFactory{
  //...
    }
  //... 
    return l, nil
}
```

factory.StartInitialization() 初始化 

!FILENAME libcontainer/factory_linux.go:282

```go
func (l *LinuxFactory) StartInitialization() (err error) {
  //...
    i, err := newContainerInit(it, pipe, consoleSocket, fifofd) 
  //...
  // newContainerInit()返回的initer实现对象的Init()方法调用 "linuxStandardInit.Init()"
  return i.Init()                    
}
```

创建 container 容器对象

!FILENAME libcontainer/factory_linux.go:188

```go
func (l *LinuxFactory) Create(id string, config *configs.Config) (Container, error) {
  // 创建 linux 容器结构
    c := &linuxContainer{ 
  //...
    }
    return c, nil
}
```

linuxContainer.Init() 对网络/路由、rootfs、selinux、console、主机名、apparmor、sysctl、seccomp、capability 等容器的相关初始化配置。管道 exec.fifo 进行写操作，进入阻塞状态等待 runC start

!FILENAME libcontainer/standard_init_linux.go:46

```go
func (l *linuxStandardInit) Init() error {
  //...
  // 此两个关于网络 nework/route 配置，将由网络专文详细介绍
  // 配置network,
	if err := setupNetwork(l.config); err != nil {
		return err
	}
  //  配置路由
	if err := setupRoute(l.config.Config); err != nil {
		return err
	}
  // selinux 配置
	label.Init()
  // 准备 rootfs
	if err := prepareRootfs(l.pipe, l.config); err != nil {
		return err
	}
  // 配置 console
	if l.config.CreateConsole {
		if err := setupConsole(l.consoleSocket, l.config, true); err != nil {
			return err
		}
		if err := system.Setctty(); err != nil {
			return errors.Wrap(err, "setctty")
		}
	}
  // 完成 rootfs 设置
	if l.config.Config.Namespaces.Contains(configs.NEWNS) {
		if err := finalizeRootfs(l.config.Config); err != nil {
			return err
		}
	}
  // 主机名设置
	if hostname := l.config.Config.Hostname; hostname != "" {
		if err := unix.Sethostname([]byte(hostname)); err != nil {
			return errors.Wrap(err, "sethostname")
		}
	}
  // 应用 apparmor 配置
	if err := apparmor.ApplyProfile(l.config.AppArmorProfile); err != nil {
		return errors.Wrap(err, "apply apparmor profile")
	}
  // Sysctl 系统参数调节
	for key, value := range l.config.Config.Sysctl {
		if err := writeSystemProperty(key, value); err != nil {
			return errors.Wrapf(err, "write sysctl key %s", key)
		}
	}
  // path 只读属性设置
	for _, path := range l.config.Config.ReadonlyPaths {
		if err := readonlyPath(path); err != nil {
			return errors.Wrapf(err, "readonly path %s", path)
		}
	}
	for _, path := range l.config.Config.MaskPaths {
		if err := maskPath(path, l.config.Config.MountLabel); err != nil {
			return errors.Wrapf(err, "mask path %s", path)
		}
	}
  // 获取父进程退出信号
	pdeath, err := system.GetParentDeathSignal()
	if err != nil {
		return errors.Wrap(err, "get pdeath signal")
	}
  // 设置安全属性 nonewprivileges
	if l.config.NoNewPrivileges {
		if err := unix.Prctl(unix.PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0); err != nil {
			return errors.Wrap(err, "set nonewprivileges")
		}
	}
  // 告诉runC进程，我们已经完成了初始化工作
	if err := syncParentReady(l.pipe); err != nil {
		return errors.Wrap(err, "sync ready")
	}
  // 进程标签设置
	if err := label.SetProcessLabel(l.config.ProcessLabel); err != nil {
		return errors.Wrap(err, "set process label")
	}
	defer label.SetProcessLabel("")
  
  // seccomp配置
	if l.config.Config.Seccomp != nil && !l.config.NoNewPrivileges {
		if err := seccomp.InitSeccomp(l.config.Config.Seccomp); err != nil {
			return err
		}
	}
  // 设置正确的capability，用户以及工作目录
	if err := finalizeNamespace(l.config); err != nil {
		return err
	}

	if err := pdeath.Restore(); err != nil {
		return errors.Wrap(err, "restore pdeath signal")
	}

	if unix.Getppid() != l.parentPid {
		return unix.Kill(unix.Getpid(), unix.SIGKILL)
	}

  // 确定用户指定的容器进程在容器文件系统中的路径
	name, err := exec.LookPath(l.config.Args[0])
	if err != nil {
		return err
	}
  
  // 关闭管道，告诉runC进程，我们已经完成了初始化工作
	l.pipe.Close()

  
  // 在exec用户进程之前等待exec.fifo管道在另一端被打开
  // 我们通过/proc/self/fd/$fd打开它
	fd, err := unix.Open(fmt.Sprintf("/proc/self/fd/%d", l.fifoFd), unix.O_WRONLY|unix.O_CLOEXEC, 0)
	if err != nil {
		return newSystemErrorWithCause(err, "open exec fifo")
	}
  //
  // 此处操作应注意，作为容器运行的分界线，后面有说明 
  //
  // 向exec.fifo管道写数据，阻塞，直到用户调用`runc start`，读取管道中的数据
	if _, err := unix.Write(fd, []byte("0")); err != nil {
		return newSystemErrorWithCause(err, "write 0 exec fifo")
	}

  // 关闭fifofd管道
	unix.Close(l.fifoFd)
  
  // 初始化Seccomp配置
	if l.config.Config.Seccomp != nil && l.config.NoNewPrivileges {
		if err := seccomp.InitSeccomp(l.config.Config.Seccomp); err != nil {
			return newSystemErrorWithCause(err, "init seccomp")
		}
	}
  // 调用系统exec()命令，执行entrypoint
	if err := syscall.Exec(name, l.config.Args[0:], os.Environ()); err != nil {
		return newSystemErrorWithCause(err, "exec user process")
	}
	return nil
}
```



## 容器用户程序与运行

我们可以看到下面容器的运行是非常简单的实现，因为在容器的 init 阶段已将所有环境都准备好了，此时只需读取管道中的数据（等同时发送bootstrap进程继续执行信号），将进程处于阻塞状态的 init 进程继续后面代码执行用户定义的entrypoint程序。 

libcontainer/container_linux.go:266

```go
func (c *linuxContainer) exec() error {
	path := filepath.Join(c.root, execFifoFilename)

	fifoOpen := make(chan struct{})
	select {
	case <-awaitProcessExit(c.initProcess.pid(), fifoOpen):
		return errors.New("container process is already dead")
	case result := <-awaitFifoOpen(path):
		close(fifoOpen)
		if result.err != nil {
			return result.err
		}
		f := result.file
		defer f.Close()
		if err := readFromExecFifo(f); err != nil {      // 读操作来解除bootstrap阻塞
			return err
		}
		return os.Remove(path)
	}
}
```

最后重新来看看 init 激活后会执行的代码：

!FILENAME libcontainer/standard_init_linux.go:192

```go
func (l *linuxStandardInit) Init() error {
  //...
  // unix.Write()阻塞
  // 初始化Seccomp配置
  if l.config.Config.Seccomp != nil && l.config.NoNewPrivileges {
		if err := seccomp.InitSeccomp(l.config.Config.Seccomp); err != nil {
			return newSystemErrorWithCause(err, "init seccomp")
		}
	}
    
  // 调用系统exec()命令，执行entrypoint
	if err := syscall.Exec(name, l.config.Args[0:], os.Environ()); err != nil {
		return newSystemErrorWithCause(err, "exec user process")
	}
	return nil
}
```



## 附录：

### 附录一：RunC 创建容器及命令

RunC run 创建与运行容器实例：

```shell
# 1. 准备rootfs文件
$> mkdir /mycontainer; cd /mycontainer
$> docker export $(docker create busybox) | tar -C rootfs -xvf -

# 2. 创建一个config.json文件(标准的OCI格式的文件)
$> runc spec 

# 3. rootfs和config.json (OCI runtime bundles)都有了就可以创建容器
$> runc run $mycontainerid
```

篇幅原因不附实例的config.json文件，可参考官方 [ config.json ](https://github.com/opencontainers/runtime-spec/blob/master/config.md) 和 [OCI Runtime spec 运行时规范(中文)](https://www.jianshu.com/p/87b4876fbf65)

RunC 容器的整个生命周期管理操作：

```shell
# 创建
$> runc create $mycontainerid
# 启动
$> runc start $mycontainerid
# 查看
$> runc list
# 删除
$> runc delete $mycontainerid
```



**相关文档**：

《RunC 源码通读指南之 Namespace》

《RunC 源码通读指南之 Cgroup》

《RunC 源码通读指南之 Create & Start》

《RunC 源码通读指南之 Networks》



~~ **本文 END** ~~
