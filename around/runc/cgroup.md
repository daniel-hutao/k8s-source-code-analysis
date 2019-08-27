# RunC 源码通读指南之 Cgroup

<!--toc-->

## 概述

Runc 作为 OCI 运行时标准的实现版本工具，其继承早期版本 Docker 的核心库 libcontainer 来 实现的 linux 系统层的资源隔离 、限制及控制等功能。Docker 容器技术通过利用 linux 的内核特性功能 cgroup 来限制与控制 container 的资源使用。本文将通过对 Runc 的 Cgroup 相关源码解析揭开如何对 cgroup 利用与实现对容器资源管控的面纱。linux 系统层的 cgroup 的基础知识本文将不做过多的介绍，[**请参考**](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)。

Runc 支持两种 cgroup driver: 一种是 cgroupfs ,一种是 systemd，在 runc 源码目录上也可以看到相应的两个目录 fs 和 systemd。而 kubelet 指定的 cgroup driver 为 cgroupfs,我们本文也仅关注于 cgroupfs 的分析,代码文件libcontainer/cgroups/fs/apply_raw.go 为实现 cgroupfs Manager管理操作。更多关于 cgroup 文件、目录详细说明可参考本文后的**附录一**。

linux 系统默认情况下将 mount cgroupfs 目录" /sys/fs/cgroup/ "和" /proc/*$pid*/cgroup "两个目录下操作实现对 进程的资源限制。对于 cgroupfs 文件系统操作方式也同样是 runC 实现的 cgroup 操作的关键接口所在,这也是runC对Cgroup操作根本原理所在。

本文先从 runc 执行过程中涉及 cgroup 的初始化、配置、应用的整个相关的过程分析,如要了解完整的 container run 的所有详细执行过程可参考本套 RunC 系列文档《 RunC 源码通读指南之 Run 》，本文仅指出执行流程中与 cgroup 相关初始化及应用的过程。其后分析了cgroup manager 和 subsystem 的实现详细分析，其实就是对 cgroup 的 CRUD 操作实现细节。最后在附录内附有 cgroup包的文件说明、公共 utils 方法功能解析说明以及各 subsystem 限制资源配置项的用途说明。

##  RunC 执行流程与 cgroup 的应用

Container 的创建过程由 factory 调用 create 方法实现，在创建 factory 对象时指定了NewCgroupsManage func，在 factory 创建 container 时调用 func 为容器配置了fs.Manager对象。调用过程 runc create 命令创建容器开始: startContainer() => createContainer() => loadFactory()  =>  libcontainer.New()

!FILENAME libcontainer/factory_linux.go:131

```go
func New(root string, options ...func(*LinuxFactory) error) (Factory, error) {
  //...
	l := &LinuxFactory{
		Root:      root,
		InitPath:  "/proc/self/exe",
		InitArgs:  []string{os.Args[0], "init"},
		Validator: validate.New(),
		CriuPath:  "criu",
	}
	Cgroupfs(l)                   //为LinuxFactory配置NewCgroupsManage实现func
	//...
	return l, nil
}
```

初始化配置LinuxFactory对象的NewCgroupsManage的func赋值，func将根据参数配置返回一个fs.Manager对象

!FILENAME libcontainer/factory_linux.go:65

```go
// Cgroupfs is an options func to configure a LinuxFactory to return containers
// that use the native cgroups filesystem implementation to create and manage
// cgroups.
func Cgroupfs(l *LinuxFactory) error {
	l.NewCgroupsManager = func(config *configs.Cgroup, paths map[string]string) cgroups.Manager {
		return &fs.Manager{
			Cgroups: config,
			Paths:   paths,
		}
	}
	return nil
}
```

创建 Container 容器对象，返回 linuxContainer 结构。LinuxFactory.NewCgroupsManager() 调用根据全局 config 赋值并返回  Cgroup Manager 对象 fs.Manger 

!FILENAME libcontainer/factory_linux.go:188

```go
func (l *LinuxFactory) Create(id string, config *configs.Config) (Container, error) {
  //...
	c := &linuxContainer{
		id:            id,
		root:          containerRoot,
		config:        config,
		initPath:      l.InitPath,
		initArgs:      l.InitArgs,
		criuPath:      l.CriuPath,
		newuidmapPath: l.NewuidmapPath,
		newgidmapPath: l.NewgidmapPath,
		cgroupManager: l.NewCgroupsManager(config.Cgroups, nil),  //为容器指定fs.Manager
	}
  //...
	return c, nil
}
```



从容器的执行流程来看，此时已完成 container 对象的创建，接下来startContainer()中已创建的 runner 对象 run() 方法执行，容器进入运行阶段。执行流程runc run命令：runner.run() => newProcess() => runner.container.Run(process) => linuxContainer.strat() => linuxContainer.newParentProcess(process) => =>linuxContainer.commandTemplate() => linuxContaine.newInitProcess() =>parent.start() => initProcess.start() 。

Parent.start() 执行其实则是 runc init 命令的执行,其基本的执行流程(详细请参考《 RunC 源码通读指南之 Run 》)：

1. parentproces 创建**runc init子进程**，中间会被 /runc/libcontainer/nsenter 劫持(c代码部分preamble)，使 runc init 子进程位于容器配置指定的各个 namespace 内
2. parentProcess 用**init管道**将容器配置信息传输给runc init进程，runc init再据此进行容器的初始化操作。初始化完成之后，再向另一个管道exec.fifo进行写操作，进入阻塞状态

InitProcess.start()执行过程中对cgroup 资源组的配置与应用工作

!FILENAME libcontainer/process_linux.go:282

```go
func (p *initProcess) start() error {
	defer p.messageSockPair.parent.Close()
  //  当前执行空间进程称为bootstrap进程
  //  启动了 cmd，即启动了 runc init 命令,创建 runc init 子进程 
  //  同时也激活了C代码nsenter模块的执行（为了 namespace 的设置 clone 了三个进程parent、child、init）
  //  C 代码执行后返回 go 代码部分,最后的 init 子进程为了好区分此处命名为" nsInit "（即配置了Namespace的init）
  //  runc init go代码为容器初始化其它部分(网络、rootfs、路由、主机名、console、安全等)
	err := p.cmd.Start()  // runc init
   
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
 
  //...
  // 解析runc init子进程的所有同步消息，当io.EOF返回
	ierr := parseSync(p.messageSockPair.parent, func(sync *syncT) error {
		switch sync.Type {
		case procReady:   
			// prestart hooks 启动前执行勾子
			if !p.config.Config.Namespaces.Contains(configs.NEWNS) {
        // 根据全局配置设置Cgroup 
				if err := p.manager.Set(p.config.Config); err != nil {
					return newSystemErrorWithCause(err, "setting cgroup config for ready process")
				}
           //...
		       // 运行执行前勾子
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
		case procHooks:   
      //  配置 cgroup
 			if err := p.manager.Set(p.config.Config); err != nil {
				return newSystemErrorWithCause(err, "setting cgroup config for procHooks process")
			}
      //...
			if p.config.Config.Hooks != nil {
		    //...
        // 执行勾子定义任务
			  // 与子进程 runc-init 同步
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

从整个执行过程来看，容器 init go 代码运行初始化配置后向exec.fifo管道写数据，阻塞，直到用户调用`runc start`，读取管道中的数据将最后执行用户定义的entrypoint程序。

上面已为Cgroup在容器创建过程中的配置与应用的管理过程，而接下来我们将看看底层是如何实现cgroup的。

## Cgroup manager 实现

Cgroup manger 为 Runc 实现对系统的 cgroup 操作的管理器抽象。manger对象实现对 cgroup 的配置项值设置、pid应用、销毁 、暂停/恢复、获取配置等操作。这里Apply() 和 Set() 注意一下两者的差别，一个是设置子系统的相关资源约束项的值，一个是将进程pid操作应用至相关的cgroup子系统。

我们先来查看几个关键接口、结构体定义：

- cgroup manager接口定义

!FILENAME libcontainer/cgroups/cgroups.go:11

```go
type Manager interface {
  // 为指定的 pid 应用 cgroup 配置
	Apply(pid int) error
	// 返回 cgroup 集内所有 pid
	GetPids() ([]int, error)
  // 返回 cgroup 集和 subcgroups 的所有 pid
	GetAllPids() ([]int, error)
  // 返回 cgroup 集统计信息
	GetStats() (*Stats, error)
	// 任务暂停与恢复操作
	Freeze(state configs.FreezerState) error
  // 销毁 cgroup 集
	Destroy() error
  // 获取保存 cgroup 状态文件路径
	GetPaths() map[string]string
	// 设置 Cgroup 配置值
	Set(container *configs.Config) error           // +configs.Config 容器进程配置结构
}
```

- Configs.Config 容器进程配置的结构体内 cgroups 相关定义

!FILENAME libcontainer/configs/config.go:81

```go
// 定义容器内执行进程的配置项，此处仅关注 Cgroup 相关
type Config struct {
   //...
   // 容器的 Cgroups 资源限制配置
	Cgroups *Cgroup `json:"cgroups"`              //+Cgroup 结构
   // ....
   // 当 RootlessCgroups 设置为 true,cgroups 错误将被忽略
	RootlessCgroups bool `json:"rootless_cgroups,omitempty"`
}
```

- Configs.cgroups 的结构定义

!FILENAME libcontainer/configs/cgroup_linux.go:11

```go
type Cgroup struct {
	// Deprecated, use Path instead
	Name string `json:"name,omitempty"`
  
	// name of parent of cgroup or slice
	// Deprecated, use Path instead
	Parent string `json:"parent,omitempty"`

  // Path指定由容器创建(和/或)连接的cgroup的路径。假定路径相对于主机系统cgroup挂载点。
	Path string `json:"path"`

	// ScopePrefix describes prefix for the scope name
	ScopePrefix string `json:"scope_prefix"`

	// Paths represent the absolute cgroups paths to join.
	// This takes precedence over Path.
	Paths map[string]string
	// 资源包含各种Cgroup的应用设置
	*Resources                                    // +参考下面定义
}       

// 每项详细说明参考本文附录一
type Resources struct {
	AllowAllDevices *bool `json:"allow_all_devices,omitempty"`
	AllowedDevices []*Device `json:"allowed_devices,omitempty"`
	DeniedDevices []*Device `json:"denied_devices,omitempty"`
	Devices []*Device `json:"devices"`
	Memory int64 `json:"memory"`
	MemoryReservation int64 `json:"memory_reservation"`
	MemorySwap int64 `json:"memory_swap"`
	KernelMemory int64 `json:"kernel_memory"`
	KernelMemoryTCP int64 `json:"kernel_memory_tcp"`
	CpuShares uint64 `json:"cpu_shares"`
	CpuQuota int64 `json:"cpu_quota"`
	CpuPeriod uint64 `json:"cpu_period"`
	CpuRtRuntime int64 `json:"cpu_rt_quota"`
	CpuRtPeriod uint64 `json:"cpu_rt_period"`
	CpusetCpus string `json:"cpuset_cpus"`
	CpusetMems string `json:"cpuset_mems"`
	PidsLimit int64 `json:"pids_limit"`
	BlkioWeight uint16 `json:"blkio_weight"`
	BlkioLeafWeight uint16 `json:"blkio_leaf_weight"`
	BlkioWeightDevice []*WeightDevice `json:"blkio_weight_device"`
	BlkioThrottleReadBpsDevice []*ThrottleDevice `json:"blkio_throttle_read_bps_device"`
	BlkioThrottleWriteBpsDevice []*ThrottleDevice `json:"blkio_throttle_write_bps_device"`
	BlkioThrottleReadIOPSDevice []*ThrottleDevice `json:"blkio_throttle_read_iops_device"`
	BlkioThrottleWriteIOPSDevice []*ThrottleDevice `json:"blkio_throttle_write_iops_device"`
	Freezer FreezerState `json:"freezer"`
	HugetlbLimit []*HugepageLimit `json:"hugetlb_limit"`
	OomKillDisable bool `json:"oom_kill_disable"`
	MemorySwappiness *uint64 `json:"memory_swappiness"`
	NetPrioIfpriomap []*IfPrioMap `json:"net_prio_ifpriomap"`
	NetClsClassid uint32 `json:"net_cls_classid_u"`
}
```



Runc 代码在 Manager 接口的实现类有两个版本driver,一个实现类 fs.Manager ,一个是  systemd.Manager ，本文主要分析 cgroupfs 驱动即 fs.Manager 的实现代码。

我们先看一下 fs.Manager 的定义 

!FILENAME libcontainer/cgroups/fs/apply_raw.go:65

```go
type Manager struct {
	mu       sync.Mutex
	Cgroups  *configs.Cgroup           // 全局配置的 cgroup 项定义（configs.Cgroup前面有结构体说明）
	Rootless bool                      
	Paths    map[string]string         // 存放子系统名与路径
}
```

cgroupData cgroup 配置数据定义

!FILENAME libcontainer/cgroups/fs/apply_raw.go:98

```go
type cgroupData struct {
	root      string                    // cgroup 根路径
	innerPath string                    // 指定由容器创建(和/或)连接的cgroup的路径
	config    *configs.Cgroup           // Cgroup 全局配置项
	pid       int                       // 进程id
}
```

manager.**Apply()** 将指定的 pid 应用资源的限制

!FILENAME libcontainer/cgroups/fs/apply_raw.go:132

```go
func (m *Manager) Apply(pid int) (err error) {
	if m.Cgroups == nil {                       // 全局 cgroup 配置是否存在检测
		return nil
	}
  //...
	var c = m.Cgroups
	d, err := getCgroupData(m.Cgroups, pid)     // +获取与构建 cgroupData 对象
  //...
	m.Paths = make(map[string]string)
  // 如果全局配置存在 cgroup paths 配置，
	if c.Paths != nil {                        
		for name, path := range c.Paths {
			_, err := d.path(name)                 // 查找子系统的 cgroup path 是否存在
			if err != nil {
				if cgroups.IsNotFound(err) {
					continue
				}
				return err
			}
			m.Paths[name] = path
		}
		return cgroups.EnterPid(m.Paths, pid)    // 将 pid 写入子系统的 cgroup.procs 文件
	}

  // 遍历所有 cgroup 子系统,将配置应用 cgroup 资源限制
	for _, sys := range subsystems {
		p, err := d.path(sys.Name())             // 查找子系统的 cgroup path
		if err != nil {
		  //...
			return err
		}
		m.Paths[sys.Name()] = p                 
    if err := sys.Apply(d); err != nil {     // 各子系统 apply() 方法调用
    //...
	}
	return nil
}
```

获取与构建 cgroupData 对象

!FILENAME libcontainer/cgroups/fs/apply_raw.go:291

```go
func getCgroupData(c *configs.Cgroup, pid int) (*cgroupData, error) {
	root, err := getCgroupRoot()          // +获取cgroup root根目录
	if err != nil {
		return nil, err
	}

  //...
	return &cgroupData{                  
		root:      root,
		innerPath: innerPath,                   
		config:    c,
		pid:       pid,
	}, nil
}
```

cgroupRoot 全局变量为空则通过查找" /proc/self/mountinfo "满足条件为" filesystem 列为 cgroup "的挂载点目录，则为 cgroup 的根目录

!FILENAME libcontainer/cgroups/fs/apply_raw.go:77

```go
func getCgroupRoot() (string, error) {
	cgroupRootLock.Lock()
	defer cgroupRootLock.Unlock()

	if cgroupRoot != "" {
		return cgroupRoot, nil
	}

	root, err := cgroups.FindCgroupMountpointDir()  // 查找"/proc/self/mountinfo"挂载点目录
	if err != nil {
		return "", err
	}

	if _, err := os.Stat(root); err != nil {        //判断是否存在
		return "", err  
	}

	cgroupRoot = root
	return cgroupRoot, nil
}
```

manager.**Set()**  根据容器的全局配置 Config 的 Cgroups 资源限制项，将 configs 写入至 cgroup 子系统文件

!FILENAME libcontainer/cgroups/fs/apply_raw.go:282

```go
func (m *Manager) Set(container *configs.Config) error {
  //...
	paths := m.GetPaths()
  // 遍历所有子系统，设置容器的全局配置 Config 的 Cgroups 资源限制项
	for _, sys := range subsystems {
		path := paths[sys.Name()]
		if err := sys.Set(path, container.Cgroups); err != nil {
		//...
	}
	return nil
}
```

manager.**Freeze()**   根据容器的全局 configs 配置应用 cgroup 暂停与恢复操作状态值

!FILENAME libcontainer/cgroups/fs/apply_raw.go:264

```go
func (m *Manager) Freeze(state configs.FreezerState) error {
	paths := m.GetPaths()
	dir := paths["freezer"]                     // 获取子系统的 path
	prevState := m.Cgroups.Resources.Freezer
	m.Cgroups.Resources.Freezer = state
	freezer, err := subsystems.Get("freezer")   
	if err != nil {
		return err
	}
	err = freezer.Set(dir, m.Cgroups)          // 设置 state 状态值
  //...
}
```

其它 manager 方法：manager.**GetPids()**    /manager.**GetAllPids()**  / manager.**GetPaths()**  / manager.**Destroy()** / manager.**GetStats()** 略

## Cgroup  subsystem 实现

Cgroupfs 子系统接口、关键类型、关键全局变量定义

!FILENAME libcontainer/cgroups/fs/apply_raw.go

```go
// 子系统接口定义
type subsystem interface {
  // 返回子系统的名称
	Name() string
	// 返回cgroup stats状态
	GetStats(path string, stats *cgroups.Stats) error
	// 移除cgroup
	Remove(*cgroupData) error
  // 创建和加入cgroup
	Apply(*cgroupData) error
	// 设置cgroup配置项值
	Set(path string, cgroup *configs.Cgroup) error
}

// 子系统集类型定义
type subsystemSet []subsystem

// subsystems全局变量定义了支持的子系统列表；
// "&CpusetGroup{}..."都为subsystem接口的具体实现
var (
	subsystems = subsystemSet{
		&CpusetGroup{},
		&DevicesGroup{},
		&MemoryGroup{},
		&CpuGroup{},
		&CpuacctGroup{},
		&PidsGroup{},
		&BlkioGroup{},
		&HugetlbGroup{},
		&NetClsGroup{},
		&NetPrioGroup{},
		&PerfEventGroup{},
		&FreezerGroup{},
		&NameGroup{GroupName: "name=systemd", Join: true},
	}
```

"cgroups/fs/" 目录下包含了各种支持的子系统实现代码，下面我们用 cpu subsystem 的实现代码详细分析作为代表，其它子系统的实现逻辑类似，本文将不作详细的一一分析。

"cgroups/fs/cpu.go" 文件内包含了 cpu subsystem 的实现代码：

CpuGroup.**Name()** 获取 cpu 子系统名称

!FILENAME libcontainer/cgroups/fs/cpu.go:18

```go
func (s *CpuGroup) Name() string {
	return "cpu"
}
```

CpuGroup.**Apply()** 基于 Cgroup configs 设置项，应用 CPU 资源限制

!FILENAME libcontainer/cgroups/fs/cpu.go:22

```go
func (s *CpuGroup) Apply(d *cgroupData) error {
	path, err := d.path("cpu")
	if err != nil && !cgroups.IsNotFound(err) {
		return err
	}
	return s.ApplyDir(path, d.config, d.pid) // +创建目录和pid写cgroup.procs文件应用cpu限制
}
```

创建目录和 pid 写入 cgroup.procs 文件应用 cpu 限制

!FILENAME libcontainer/cgroups/fs/cpu.go:32

```go
func (s *CpuGroup) ApplyDir(path string, cgroup *configs.Cgroup, pid int) error {
	if path == "" {
		return nil
	}
  // 创建目录
	if err := os.MkdirAll(path, 0755); err != nil {
		return err
	}
  // 设置 RT(realtime)调度值： cpu.rt_period_us ，cpu.rt_runtime_us
	if err := s.SetRtSched(path, cgroup); err != nil {
		return err
	}
  // pid加入cgroup procs文件应用cgroup组限制
	return cgroups.WriteCgroupProc(path, pid)
}
```

CpuGroup.**Set()** 基于 Cgroup configs 设置项，写入配置值至相应的子系统控制资源文件，实现 CPU 的限制调节

!FILENAME libcontainer/cgroups/fs/cpu.go:66

```go
func (s *CpuGroup) Set(path string, cgroup *configs.Cgroup) error {
	if cgroup.Resources.CpuShares != 0 {
    // 写入文件值，控制cgroup组之间的配额占比
		if err := writeFile(path, "cpu.shares", strconv.FormatUint(cgroup.Resources.CpuShares, 10)); err != nil {
			return err
		}
	}
	if cgroup.Resources.CpuPeriod != 0 {
    // 写入文件值，CFS调度 CPU 时间的周期
		if err := writeFile(path, "cpu.cfs_period_us", strconv.FormatUint(cgroup.Resources.CpuPeriod, 10)); err != nil {
			return err
		}
	}
	if cgroup.Resources.CpuQuota != 0 {
    // 写入文件值，CFS调度 期间内可使用的 cpu 时间
		if err := writeFile(path, "cpu.cfs_quota_us", strconv.FormatInt(cgroup.Resources.CpuQuota, 10)); err != nil {
			return err
		}
	}
  // 设置 RT(realtime)调度值： cpu.rt_period_us ，cpu.rt_runtime_us
	return s.SetRtSched(path, cgroup)
}
```

CpuGroup.**GetStats()** 获取子系统的 cpu.stat 状态文件信息

!FILENAME libcontainer/cgroups/fs/cpu.go:89

```go
func (s *CpuGroup) GetStats(path string, stats *cgroups.Stats) error {
	f, err := os.Open(filepath.Join(path, "cpu.stat"))   // cpu.stat文件
	if err != nil {
		if os.IsNotExist(err) {
			return nil
		}
		return err
	}
	defer f.Close()

	sc := bufio.NewScanner(f)
	for sc.Scan() {
		t, v, err := getCgroupParamKeyValue(sc.Text())  // K/V解析
		if err != nil {
			return err
		}
		switch t {
		case "nr_periods":                              //"nr_periods"  进入周期的次数
			stats.CpuStats.ThrottlingData.Periods = v

		case "nr_throttled":                            //"nr_throttled" 运行时间被调整的次数
			stats.CpuStats.ThrottlingData.ThrottledPeriods = v

		case "throttled_time":                          //"throttled_time" 用于调整的时间
			stats.CpuStats.ThrottlingData.ThrottledTime = v
		}
	}
	return nil
}
```

CpuGroup.**remove()** 删除 cpu 子系统

!FILENAME libcontainer/cgroups/fs/cpu.go:85

```go
func (s *CpuGroup) Remove(d *cgroupData) error {
	return removePath(d.path("cpu"))  //删除path
}
```

其它 cgroup 的子系统资源设置项说明可参考后面的附录。

## 附录 一  

#### Cgroup 包内文件说明

```shell
├── cgroups.go             ---------------------------->| 定义cgroup Manager接口操作  
├── cgroups_test.go
├── cgroups_unsupported.go
├── fs
│   ├── apply_raw.go      ---------------------------->| cgroupfs driver Manager实现 
│   ├── apply_raw_test.go              
│   ├── blkio.go          -------[blkio子系统]--------->|
│   ├── blkio_test.go                                  |
│   ├── cpu.go            -------[cpu子系统]----------->|
│   ├── cpu_test.go                                    |    
│   ├── cpuacct.go        -------[cpuacct子系统]------->|
│   ├── cpuset.go         -------[cpuset子系统]-------->|
│   ├── cpuset_test.go                                 |  
│   ├── devices.go        -------[devices子系统]------->|\
│   ├── devices_test.go                                | |\  各子系统实现，Cgroupfs目录文件
│   ├── freezer.go        -------[freezer子系统]------->| |/  的CRUD方法
│   ├── freezer_test.go                                |/
│   ├── fs_unsupported.go                              |
│   ├── hugetlb.go        -------[hugetlb子系统]------->|
│   ├── hugetlb_test.go                                |
│   ├── kmem.go                                        |
│   ├── kmem_disabled.go                               |
│   ├── memory.go         -------[memory子系统]-------->|
│   ├── memory_test.go                                 |
│   ├── name.go           -------[systemd子系统]------->|
│   ├── net_cls.go        -------[netcls子系统]-------->|
│   ├── net_cls_test.go                                |
│   ├── net_prio.go       -------[netprio子系统]------->|
│   ├── net_prio_test.go                               |
│   ├── perf_event.go     -------[perf event子系统]---->|
│   ├── pids.go           -------[pid子系统]----------->|
│   ├── pids_test.go
│   ├── stats_util_test.go
│   ├── util_test.go      
│   ├── utils.go         ---------------------------->| fs包内共享的工具方法
│   └── utils_test.go
├── stats.go             ---------------------------->| cgroup stats 定义与对象构建
├── systemd
│   ├── apply_nosystemd.go
│   └── apply_systemd.go ---------------------------->| cgroup systemd driver manager 实现 
├── utils.go             ---------------------------->| 包内共享的工具方法
└── utils_test.go
```

#### Cgroup 子系统资源配置项相关说明

```go
1. [pids] --- 限制cgroup及其所有子孙cgroup里面能创建的总的task数量
   pids.max 所允许创建的总的最大进程数量
   pids.current 现有的总的进程数量

2. [memory] --- 限制内存子系统限制内存使用量
   memory.memsw.limit_in_bytes 内存＋swap空间使用的总量限制
   memory.limit_in_bytes 内存使用量限制
   memory.kmem.limit_in_bytes 限制内核使用的内存大小
   memory.soft_limit_in_bytes 内存软限制
   memory.kmem.tcp.limit_in_bytes 设置tcp 缓存内存的hard limit
   memory.oom_control 设置/读取内存超限控制信息
   memory.swappiness 用来调整cgroup使用swap的状态，表示不使用交换分区

3. [cpu] --- 限制进程的cpu总体占用率
   cpu.rt_period_us  实时任务统计CPU使用时间的周期
   cpu.rt_runtime_us 实时任务周期内允许任务使用单个CPU核的时间，如果系统中有多个核，则可以使用核倍数的时间
   cpu.shares  控制各个cgroup组之间的配额占比
   //下面两个组合使用，限制该组中的所有进程在单位时间里可以使用的 cpu 时间
   //如:将 cpu.cfs_quota_us 设为 50000，相对于 cpu.cfs_period_us 的 100000 即 50%
   //cfs_quota_us 也是可以大于 cfs_period_us 的，这主要是对于多核情况
   cpu.cfs_period_us  时间周期，默认为 100000，即百毫秒
   cpu.cfs_quota_us   期间内可使用的 cpu 时间，默认 -1，即无限制

4. [cpuset] --- 多核心的cpu环境，为cgroup任务分配独立的内存节点和CPU节点
   cpuset.cpus  限制只能使用特定CPU节点
   cpuset.mems  限制只能使用特定内存节点

5. [devices] --- 以控制进程能够访问某些设备
    // echo "a 1:5 r" > devices.deny
    // a|b|C     all/block/character
    // r|w|m     read/write/create
   devices.deny  拒绝
   devices.allow 允许

6. [blkio] --- 设置限制每个块设备的输入输出控制。例如:磁盘，光盘以及usb等等
    // CFQ Completely Fair Queuing 完全公平队列
   blkio.weight   权重值(范围100-1000)
   blkio.leaf_weight
   blkio.weight_device   块设备级的值 (major:minor weight) （优先级高于blkio.weight）
   blkio.leaf_weight_device
    // 限制IOPS使用上限
   blkio.throttle.read_bps_device   读设备 bytes/s 
   blkio.throttle.write_bps_device  写设备 bytes/s 
   blkio.throttle.read_iops_device  读设备 io/s  
   blkio.throttle.write_iops_device 写设备 io/s  

7. [net_prio] --- 配置每个网络接口的流量优先级
   net_prio.ifpriomap 优先级图谱

8. [net_cls] --- 标记每个网络包,可供QOS/netfilter使用
   net_cls.classid  标签id

9. [freezer] ---  暂停和恢复cgroup任务
   freezer.state  当前的状态,两个状态是写有效(Frozen已冻结/Thawed解冻状态)

10.[hugetlb] --- 对于HugeTLB系统进行限制，这是一个大页文件系统
   hugetlb.XX.limit_in_bytes 限制大页字节数

```

## 附录 二

cgroups 包下 utils 定义的方法用途简析

!FILENAME libcontainer/cgroups/utils.go

```go
// 查找/proc/self/mountinfo下满足条件“cgroupPath，subsystem”的项，返回"Cgroup根目录与挂载点"
func FindCgroupMountpointAndRoot(cgroupPath, subsystem string) (string, string, error) {...}
func findCgroupMountpointAndRootFromReader(reader io.Reader, cgroupPath, subsystem string) (string, string, error) {...}
// 获取"/proc/self/cgroup"进行匹配是否存在subsystem
func isSubsystemAvailable(subsystem string) bool{...}  

// 查"/proc/self/mountinfo"满足条件"filesystem列为cgroup"的挂载点目录；
// 一般为"/sys/fs/cgroup/"
func FindCgroupMountpointDir() (string, error) {...}
// 获取所有Cgroup信息结构化Mount slice返回；[]Mount{Mountpoint/Root/Subsystems[]}
func GetCgroupMounts(all bool) ([]Mount, error) {...}
func getCgroupMountsHelper(ss map[string]bool,mi io.Reader, all bool) ([]Mount, error) {...}
  
// 打开/proc/<pid>/cgroup文件调用parseCgroupFromReader()
func ParseCgroupFile(path string) (map[string]string, error) {...}
// 解析"/proc/[pid]/cgroup"输出map[subsystem]cgroup-path
func parseCgroupFromReader(r io.Reader) (map[string]string, error) {...}

// 返回subsystem的cgroup-path
func getControllerPath(subsystem string, cgroups map[string]string) (string, error) {...}

// 获取当前进程的(root)Cgroup-path,通过解析"/proc/self/cgroup"文件
func GetOwnCgroup(subsystem string) (string, error) {...}
// 获取当前进程的(mnt)Cgroup-path
func GetOwnCgroupPath(subsystem string) (string, error) {...}

// 获取init进程的(root)Cgroup-path,通过解析"/proc/1/cgroup"文件
func GetInitCgroup(subsystem string) (string, error) {...}
// 获取init进程的(mnt)Cgroup-path
func GetInitCgroupPath(subsystem string) (string, error) {...}
// 获取Cgroup mnt path
func getCgroupPathHelper(subsystem, cgroup string) (string, error) {...}

// 获取指定被加入cgroup path的所有pid
func GetPids(path string) ([]int, error){...}
// 获取指定被加入cgroup path和subcgroups的所有pid
func GetAllPids(path string) ([]int, error) {...}
// 打开指定dir的cgroup subsystem的"cgroup.procs" 读取pids
func readProcsFile(dir string) ([]int, error) {...}
// 写入指定的 pid 至 Cgroup subsystem "cgroup.procs"文件
func EnterPid(cgroupPaths map[string]string, pid int) error
// 打开指定dir的cgroup subsystem的"cgroup.procs"写入指定的pid
func WriteCgroupProc(dir string, pid int) error {...}

// 获取大页大小列表
func GetHugePageSize() ([]string, error) {...}
// 读"/sys/kernel/mm/hugepages"目录下文件，解析文件名获取hg大小
func getHugePageSizeFromFilenames(fileNames []string) ([]string, error) {...}

```



**相关文档**： // TODO 补充链接

- 《RunC 源码通读指南之 Run》
- 《RunC 源码通读指南之 Create & Start》
- 《RunC 源码通读指南之 Namespace》
- 《RunC 源码通读指南之 Networks》

**~本文 END~**

