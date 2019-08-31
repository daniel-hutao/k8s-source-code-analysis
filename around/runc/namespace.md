# RunC 源码通读指南之 NameSpace

<!-- toc -->

## 概述

随着 docker 的诞生和容器技术应用与高速发展，长期一直在后台默默奉献一些 linux 特性如 namespace、cgroup 等技术走向前台。Namespace 是 linux 内核所提供的特性，用于隔离内核资源的方式，可以说没有隔离就不会存在容器。

Linux 官方描述" namespace 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。"详细介绍[namespace说明参考](http://man7.org/linux/man-pages/man7/namespaces.7.html) 。 Linux 内核里面实现了7种不同类型的 namespace:

```shell
名称        宏定义             隔离内容
Cgroup    CLONE_NEWCGROUP   Cgroup root directory 
IPC       CLONE_NEWIPC      System V IPC, POSIX message queues 
Network   CLONE_NEWNET      Network devices, stacks, ports, etc. 
Mount     CLONE_NEWNS       Mount points
PID       CLONE_NEWPID      Process IDs 
User      CLONE_NEWUSER     User and group IDs 
UTS       CLONE_NEWUTS      Hostname and NIS domain name 
```



本文将聚焦在 runC 源码关于容器初始化过程中 namespace 如何应用与实现资源隔离。

从容器的 run 执行流程来看： **容器对象创建阶段** startContainer() => createContainer() => loadFactory()  =>  libcontainer.New() 完成 container 对象的创建后, startContainer() 中已创建的 runner 对象 run() 方法执行，进入**容器对象运行阶段**:  startContainer() => runner.run() => newProcess() => runner.container.Run(process) => linuxContainer.strat() => linuxContainer.newParentProcess(process) => =>linuxContainer.commandTemplate() => linuxContaine.newInitProcess() =>parent.start() => initProcess.start() 。

Parent.start() 执行其实则是 runC init 命令的执行:

1. ParentProcces 创建runC init子进程，中间会被 /runc/libcontainer/nsenter 劫持( c 代码部分 preamble )，使 runc init 子进程位于容器配置指定的各个 **namespace** 内（实现 namespace配置 ）
2. ParentProcess 用init管道将容器配置信息传输给runC init进程，runC init再据此配置信息进行容器的初始化操作。初始化完成之后，再向另一个管道exec.fifo进行写操作，进入阻塞状态等待runC start

因此本文我们将从两个方面展开分析，第一则是 runC init 流程执行关于 namespace 设置的时机，第二则是 c 代码部分 nsenter 的实现（ namespace 关键应用代码）。

## RunC init 执行流程与 namespace 

创建容器的 init 进程时相关 namespace 配置项

!FILENAME libcontainer/container_linux.go:512

```go
func (c *linuxContainer) newInitProcess(p *Process, cmd *exec.Cmd, messageSockPair, logFilePair filePair) (*initProcess, error) {
	cmd.Env = append(cmd.Env, "_LIBCONTAINER_INITTYPE="+string(initStandard))
	nsMaps := make(map[configs.NamespaceType]string)  
	for _, ns := range c.config.Namespaces {    // 容器 namesapces 配置
		if ns.Path != "" {
			nsMaps[ns.Type] = ns.Path
		}
	}
	_, sharePidns := nsMaps[configs.NEWPID]
  // 创建 init 进程同步namespace配置项数据（后面有详述bootstrapData）
	data, err := c.bootstrapData(c.config.Namespaces.CloneFlags(), nsMaps)
	if err != nil {
		return nil, err
	}
	init := &initProcess{
		cmd:             cmd,
		messageSockPair: messageSockPair,
		logFilePair:     logFilePair,
		manager:         c.cgroupManager,
		intelRdtManager: c.intelRdtManager,
		config:          c.newInitConfig(p),
		container:       c,
		process:         p,
		bootstrapData:   data,              // 指定 init process bootstrapData值
		sharePidns:      sharePidns,
	}
	c.initProcess = init
	return init, nil
}
```

InitProcess.start() 容器的初始化配置，此处 cmd.start() 调用实则是 runC init命令执行:

- **先执行 nsenter C代码部分**，实现对container的process进行Namespace相关设置如uid/gid、pid、uts、ns、cgroup等。
- **返执行 init 命令 Go 代码部分**，LinuxFactory.StartInitialization()对网络/路由、rootfs、selinux、console、主机名、apparmor、Sysctl、seccomp、capability等容器配置

!FILENAME libcontainer/process_linux.go:282

```go
func (p *initProcess) start() error {
  //  当前执行空间进程称为bootstrap进程
  //  启动了 cmd，即启动了 runc init 命令,创建 runc init 子进程 
  //  同时也激活了C代码nsenter模块的执行（为了 namespace 的设置 clone 了三个进程parent、child、init）
  //  C 代码执行后返回 go 代码部分,最后的 init 子进程为了好区分此处命名为" nsInit "（即配置了Namespace的init）
  //  runc init go代码为容器初始化其它部分(网络、rootfs、路由、主机名、console、安全等)
  
	err := p.cmd.Start()   // +runc init 命令执行，Namespace应用代码执行空间时机
  //...
  	if p.bootstrapData != nil {
     // 将 bootstrapData 写入到 parent pipe 中，此时 runc init 可以从 child pipe 里读取到这个数据
		if _, err := io.Copy(p.messageSockPair.parent, p.bootstrapData); err != nil {
			return newSystemErrorWithCause(err, "copying bootstrap data to pipe")
		}
	}
  //...
}
```



此时来到 runC init 命令执行代码部分，前面有说到**先执行 nsenter C 代码逻辑**（后面详述），再返回到 Go init 代码部分，而Go init 代码部分不是本文 namespace 介绍的重点，考虑到执行流程理解的连续性，我先简述一下此块，有助于将整个过程串联起来理解。

RunC init 命令执行 Go 调用 C 代码称之 preamble ,即在 import nsenter 模块时机将会在 Go 的 runtime 启动之前，先执行此先导代码块，nsenter 的初始化 init(void) 方法内对 nsexec() 调用 。

!FILENAME init.go:10

```go
	_ "github.com/opencontainers/runc/libcontainer/nsenter"
```

!FILENAME libcontainer/nsenter/nsenter.go:3

```C
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

*注：此处 C 代码 nsexec() 分析部分将后面将详细解析*



**再执行 go 代码 init 命令执行逻辑部分**,创建 factory 对象，执行 factory.StartInitialization() => linuxStandardInit.Init() 完成容器的相关初始化配置(网络/路由、rootfs、selinux、console、主机名、apparmor、Sysctl、seccomp、capability 等)

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

Linux 版本的 factory 实现，查看 StartInitialization() 实现代码

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

网络/路由、rootfs、selinux、console、主机名、apparmor、sysctl、seccomp、capability 等容器的相关初始化配置。管道 exec.fifo 进行写操作，进入阻塞状态等待 runC start

!FILENAME libcontainer/standard_init_linux.go:46

```go
func (l *linuxStandardInit) Init() error {
  //...
  // 留意此两个关于网络nework/route配置，将专文详细介绍network
  // 配置network,
  //  配置路由
  // selinux配置
  // + 准备rootfs
  // 配置console
  // 完成rootfs设置
  // 主机名设置
  // 应用apparmor配置
  // Sysctl系统参数调节
  // path只读属性配置
  // 告诉runC进程，我们已经完成了初始化工作
  // 进程标签设置
  // seccomp配置
  // 设置正确的capability，用户以及工作目录
  // 确定用户指定的容器进程在容器文件系统中的路径
  // 关闭管道，告诉runC进程，我们已经完成了初始化工作
  // 在exec用户进程之前等待exec.fifo管道在另一端被打开
  // 我们通过/proc/self/fd/$fd打开它
  // ......
  // 向exec.fifo管道写数据，阻塞，直到用户调用`runc start`，读取管道中的数据
  // 此时当前进程已处于阻塞状态，等待信号执行后面代码
  //
	if _, err := unix.Write(fd, []byte("0")); err != nil {
		return newSystemErrorWithCause(err, "write 0 exec fifo")
	}
  // 关闭fifofd管道 fix CVE-2016-9962
  // 初始化Seccomp配置
  // 调用系统exec()命令，执行entrypoint
	if err := syscall.Exec(name, l.config.Args[0:], os.Environ()); err != nil {
		return newSystemErrorWithCause(err, "exec user process")
	}
	return nil
}
```

此时整个 run 的容器执行流程在执行用户程序 entrypoint 后已接近尾声。从整个执行过程来看 namespace 的配置逻辑主要在 nsenter C 代码内，下面先简要查看 runc 内对 namespace 相关的定义与实现方法，后面将详细介绍 nsenter 的逻辑代码实现。

## RunC Namespace 定义与实现

先来看一下容器内的执行进程 config 配置的 namespaces 定义

!FILENAME libcontainer/configs/config.go:81

```go
// Config defines configuration options for executing a process inside a contained environment.
type Config struct {
  //...
	Namespaces Namespaces `json:"namespaces"`     // NameSpaces 在 config 定义
  //...
}
```

!FILENAME libcontainer/configs/namespaces.go:5

```go
type Namespaces []Namespace     // Namespace 类型slice
```

!FILENAME libcontainer/configs/namespaces_linux.go:80

```go
type Namespace struct {
	Type NamespaceType `json:"type"`
	Path string        `json:"path"`
}
```

GetPath() 获取 namespace 路径"/proc/\$pid/ns/\$nsType"

!FILENAME libcontainer/configs/namespaces_linux.go:85

```go
// 获取指定pid的指定类型 namespace 路径"/proc/$pid/ns/$nsType"
func (n *Namespace) GetPath(pid int) string {
	return fmt.Sprintf("/proc/%d/ns/%s", pid, NsName(n.Type))
}

// Namespace类型字串转化为系统文件名
func NsName(ns NamespaceType) string {
	switch ns {
	case NEWNET:
		return "net"
	case NEWNS:
		return "mnt"
	case NEWPID:
		return "pid"
	case NEWIPC:
		return "ipc"
	case NEWUSER:
		return "user"
	case NEWUTS:
		return "uts"
	case NEWCGROUP:
		return "cgroup"
	}
	return ""
}
```

Namespaces 类提供的操作方法列表

!FILENAME libcontainer/configs/namespaces_linux.go:89

```go
// 删除,从Namespaces slice中删除指定类型的Namespace项
func (n *Namespaces) Remove(t NamespaceType) bool {
//...
}
// 增加
func (n *Namespaces) Add(t NamespaceType, path string) {
//...
}
// 是否存在
func (n *Namespaces) Contains(t NamespaceType) bool {
//...
}
// 获取指定Namespace类型的Path
func (n *Namespaces) PathOf(t NamespaceType) string {
//...
}
```

ParentProcess 用 init 管道将容器配置信息传输给 runc init 进程，那么我们就来看一下 init 管道所传输的 bootstrapData 数据内容的定义，bootstrapData()最后返回序列化后的数据读取器io reader

!FILENAME  libcontainer/container_linux.go:1945

```go
func (c *linuxContainer) bootstrapData(cloneFlags uintptr, nsMaps map[configs.NamespaceType]string) (io.Reader, error) {
  // 创建 netlink 消息
	r := nl.NewNetlinkRequest(int(InitMsg), 0)

	// 写入 cloneFlags 
	r.AddData(&Int32msg{
		Type:  CloneFlagsAttr,
		Value: uint32(cloneFlags),
	})

	// 写入自定义 namespace paths
	if len(nsMaps) > 0 {
		nsPaths, err := c.orderNamespacePaths(nsMaps)
		if err != nil {
			return nil, err
		}
		r.AddData(&Bytemsg{
			Type:  NsPathsAttr,
			Value: []byte(strings.Join(nsPaths, ",")),
		})
	}

  // 为新 user 写入 ns paths
	_, joinExistingUser := nsMaps[configs.NEWUSER]
	if !joinExistingUser {
		// write uid mappings
		if len(c.config.UidMappings) > 0 {
			if c.config.RootlessEUID && c.newuidmapPath != "" {
				r.AddData(&Bytemsg{
					Type:  UidmapPathAttr,
					Value: []byte(c.newuidmapPath),
				})
			}
			b, err := encodeIDMapping(c.config.UidMappings)
			if err != nil {
				return nil, err
			}
			r.AddData(&Bytemsg{
				Type:  UidmapAttr,
				Value: b,
			})
		}

		// 写 gid mappings
		if len(c.config.GidMappings) > 0 {
			b, err := encodeIDMapping(c.config.GidMappings)
			if err != nil {
				return nil, err
			}
			r.AddData(&Bytemsg{
				Type:  GidmapAttr,
				Value: b,
			})
			if c.config.RootlessEUID && c.newgidmapPath != "" {
				r.AddData(&Bytemsg{
					Type:  GidmapPathAttr,
					Value: []byte(c.newgidmapPath),
				})
			}
			if requiresRootOrMappingTool(c.config) {
				r.AddData(&Boolmsg{
					Type:  SetgroupAttr,
					Value: true,
				})
			}
		}
	}

	if c.config.OomScoreAdj != nil {
		// 如存在配置 OomScorAdj ，写 oom_score_adj 
		r.AddData(&Bytemsg{
			Type:  OomScoreAdjAttr,
			Value: []byte(fmt.Sprintf("%d", *c.config.OomScoreAdj)),
		})
	}

	// 写 rootless
	r.AddData(&Boolmsg{
		Type:  RootlessEUIDAttr,
		Value: c.config.RootlessEUID,
	})

	return bytes.NewReader(r.Serialize()), nil
}
```

## Nsenter C代码解析

刚读这段代码时有些理解上混乱，多层父子进行之间交错传递，经过反复仔细重读和推敲代码后才逐渐清晰作者的 代码逻辑思想。

在初期理解代码逻辑时本人存在的几个疑惑点：

1. 为什么需要 fork 三层级关系的进程来实现 namespaces 的配置？

2. 是否每次 fork 的子进程将继承其父的 namespaces 配置 ？
3. 是否有什么值传回给bootstrap进程？

我相信看完代码分析后能得到答案。

Runc init 会有三个进程:

- 第一个进程称为“ parent ”，读取 bootstrapData 并解析为 Config，对 User map 设置，并通过消息协调后面两个进程的运行管理，在收到 grandchild 回复任务完成消息后退出。
- 第二个进程称为“ child ”,由 Parent 创建，完成 namespace 的设置 ，fork 出 grandChild 进程并发送给Parent 后发送任务完成消息后退出。
- 第三个进程称为“ grandChild ”或" init "，进行最后的环境准备工作(sid、uid、gid、cgroup namespace)，执行完成后return 至 init Go runtime 代码处继续执行最后进入 go 代码。

先来看下 Init pipe 配置 datas 读取并解析后的 config 定义

!FILENAME libcontainer/nsenter/nsexec.c:70

```C
struct nlconfig_t {
	char *data;

	/* Process settings. */
	uint32_t cloneflags;
	char *oom_score_adj;
	size_t oom_score_adj_len;

	/* User namespace settings. */
	char *uidmap;
	size_t uidmap_len;
	char *gidmap;
	size_t gidmap_len;
	char *namespaces;
	size_t namespaces_len;
	uint8_t is_setgroup;

	/* Rootless container settings. */
	uint8_t is_rootless_euid;	/* boolean */
	char *uidmappath;
	size_t uidmappath_len;
	char *gidmappath;
	size_t gidmappath_len;
};
```

Nsexec() 为 nsenter 主干执行逻辑代码,所有 namespaces 配置都在此 func 内执行完成 

!FILENAME libcontainer/nsenter/nsexec.c:575

```c
void nsexec(void)
{
	int pipenum;
	jmp_buf env;
	int sync_child_pipe[2], sync_grandchild_pipe[2];  //用于后面child和grandchild进程通信
	struct nlconfig_t config = { 0 };

  // 配置发送给父进程的 logs 管道
	setup_logpipe();

  // 从环境变量 _LIBCONTAINER_INITPIPE 中取得 child pipe 的 fd 编号
  // linuxContainer.commandTemplate() 指定了容器相关的环境变量" _LIBCONTAINER_* "
	pipenum = initpipe();
	if (pipenum == -1)
    // 由于正常启动的 runc 是没有这个环境变量的，所以这里会直接返回，然后就开始正常的执行 go 程序了
		return;

   // 确保当前的二进制文件是已经复制过的，用来规避 CVE-2019-5736 漏洞
   // ensure_cloned_binary 中使用了两种方法：
   // - 使用 memfd，将二进制文件写入 memfd，然后重启 runc
   // - 复制二进制文件到临时文件，然后重启 runc
	if (ensure_cloned_binary() < 0)
		bail("could not ensure we are a cloned binary");

	write_log(DEBUG, "nsexec started");

  // 从 child pipe 中读取 namespace config 并解析为 config 结构
  // "child pipe" 为 linuxContainer.newParentProcess() 创建 init pipe（sockPair）
	nl_parse(pipenum, &config);

  // 设置 oom score，这个只能在特权模式下设置，所以在这里就要修改完成
	update_oom_score_adj(config.oom_score_adj, config.oom_score_adj_len);

  // 设置进程不可 dump
	if (config.namespaces) {
		if (prctl(PR_SET_DUMPABLE, 0, 0, 0, 0) < 0)
			bail("failed to set process as non-dumpable");
	}

  // 创建和子进程通信的 pipe,sync_child_pipe 前面有定义
	if (socketpair(AF_LOCAL, SOCK_STREAM, 0, sync_child_pipe) < 0)
		bail("failed to setup sync pipe between parent and child");

  // 创建和孙进程通信的 pipe,sync_grandchild_pipe 前面有定义
	if (socketpair(AF_LOCAL, SOCK_STREAM, 0, sync_grandchild_pipe) < 0)
		bail("failed to setup sync pipe between parent and grandchild");

  // setjmp 将当前执行位置的环境保存下来，用于多进程环境下的程序跳转
  // 此处因后面对自身进行 fork 进程，通过不同进程的 env 值进行跳转逻辑执行 
	switch (setjmp(env)) {
      // +后面详述
      //...
  }
```

**Parent** 父进程创建子进程( Child 自身也创建子进程称为 Grandchild ).接收 child 配置 uid_map 和 gid_map 请求消息 ,为容器与宿主完成 uid/gid range 映射后发送确认给 child ；在接收到 child 发送的 grand pid 后，通过容器外传进来的 child pipe 把子和孙进程 PID，写回去，然后让容器外的 runc（bootstrap进程）接管 PID；然后等待child 完成任务消息。其后发送 grandchild 准备运行消息后等待 grandchild 回复完成任务消息后退出进程。

!FILENAME libcontainer/nsenter/nsexec.c:700

```c
		/*
		 * Stage 0: We're in the parent. Our job is just to create a new child
		 *          (stage 1: JUMP_CHILD) process and write its uid_map and
		 *          gid_map. That process will go on to create a new process, then
		 *          it will send us its PID which we will send to the bootstrap
		 *          process.
		 */
	// 第一次执行的时候 setjmp 返回 0，对应 JUMP_PARENT
	case JUMP_PARENT:{
			int len;
			pid_t child, first_child = -1;
			bool ready = false;

			/* For debugging. */
			prctl(PR_SET_NAME, (unsigned long)"runc:[0:PARENT]", 0, 0, 0);
 
      // clone_parent 创建了和当前进程完全一致的一个进程（子进程）
      // 在 clone_parent 中，通过 longjmp() 跳转到 env 保存的位置
      // 并且 setjmp 返回值为 JUMP_CHILD
      // 这样这个子进程就会根据 switch 执行到 JUMP_CHILD 分支
      // 而当前 runc init 和 子 runc init 之间通过上面创建的
      // sync_child_pipe 进行同步通信
			child = clone_parent(&env, JUMP_CHILD);
			if (child < 0)
				bail("unable to fork: child_func");

     // 通过 sync_child_pipe 循环读取来自子进程的消息，“消息”定义如下：
     // enum sync_t {
	   //      SYNC_USERMAP_PLS = 0x40,	/* Request parent to map our users. */
	   //      SYNC_USERMAP_ACK = 0x41,	/* Mapping finished by the parent. */
	   //      SYNC_RECVPID_PLS = 0x42,	/* Tell parent we're sending the PID. */
	   //      SYNC_RECVPID_ACK = 0x43,	/* PID was correctly received by parent. */
	   //      SYNC_GRANDCHILD = 0x44,	/* The grandchild is ready to run. */
	   //      SYNC_CHILD_READY = 0x45,	/* The child or grandchild is ready to return. */
     //   };
    
      // 与 child 子进程互通消息并处理
      // 通过 sync_child_pipe 循环读取来自子进程的消息
			while (!ready) {
				enum sync_t s;

				syncfd = sync_child_pipe[1];
				close(sync_child_pipe[0]);
        
        // 等待(读取) Child 的消息
				if (read(syncfd, &s, sizeof(s)) != sizeof(s))
					bail("failed to sync with child: next state");

				switch (s) {
        // 这里设置 user map，因为子进程修改自身的 user namespace 之后，就没有权限再设置 user map 了
				case SYNC_USERMAP_PLS:   // 收到子进程请求设置 usermap 消息
            
					if (config.is_rootless_euid && !config.is_setgroup)
						update_setgroups(child, SETGROUPS_DENY);

					/* Set up mappings. */
					update_uidmap(config.uidmappath, child, config.uidmap, config.uidmap_len);
					update_gidmap(config.gidmappath, child, config.gidmap, config.gidmap_len);
            
          // 向子进程发送 SYNC_USERMAP_ACK，表示处理完成
					s = SYNC_USERMAP_ACK;
					if (write(syncfd, &s, sizeof(s)) != sizeof(s)) {
						kill(child, SIGKILL);
						bail("failed to sync with child: write(SYNC_USERMAP_ACK)");
					}
					break;
				case SYNC_RECVPID_PLS:{   // 收到子进程传递的 grandchild 的 PID 接收请求消息
						first_child = child;
            // 接收孙进程的pid
						if (read(syncfd, &child, sizeof(child)) != sizeof(child)) {
							kill(first_child, SIGKILL);
							bail("failed to sync with child: read(childpid)");
						}

						s = SYNC_RECVPID_ACK;   // 回复接收确认消息给 child 
						if (write(syncfd, &s, sizeof(s)) != sizeof(s)) {
							kill(first_child, SIGKILL);
							kill(child, SIGKILL);
							bail("failed to sync with child: write(SYNC_RECVPID_ACK)");
						}

				    // 通过容器外传进来的 child pipe 把子和孙进程 PID，写回去，然后让容器外的 runc 接管 PID
            // 这个是因为 clone_parent 的时候参数传了 CLONE_PARENT，导致子孙的父进程都是容器外的那
            // 个 runc， 所以当前进程无法接管这些 PID
						len = dprintf(pipenum, "{\"pid\": %d, \"pid_first\": %d}\n", child, first_child);
						if (len < 0) {
							kill(child, SIGKILL);
							bail("unable to generate JSON for child pid");
						}
					}
					break;
				case SYNC_CHILD_READY:      // 收到子进程任务完成消息
          // 子进程已经处理完了所有事情，父进程可退出循环
					ready = true;
					break;
				default:
					bail("unexpected sync value: %u", s);
				}
			}

      // 与 Grandchild 孙进程互通消息并处理
      // 通过 sync_grandchild_pipe 循环读取来自孙进程的消息
			ready = false;
			while (!ready) {
				enum sync_t s;

				syncfd = sync_grandchild_pipe[1];
				close(sync_grandchild_pipe[0]);

				s = SYNC_GRANDCHILD;     //  发送 "SYNC_GRANDCHILD" 准备运行消息
				if (write(syncfd, &s, sizeof(s)) != sizeof(s)) {
					kill(child, SIGKILL);
					bail("failed to sync with child: write(SYNC_GRANDCHILD)");
				}

				if (read(syncfd, &s, sizeof(s)) != sizeof(s))
					bail("failed to sync with child: next state");

				switch (s) {
				case SYNC_CHILD_READY:   //  接收孙进程任务完成消息
					ready = true;
					break;
				default:
					bail("unexpected sync value: %u", s);
				}
			}
       // 退出。很明显，当前 runc init 退出的时候，子 runc init 一定也退出了，
       // 但是孙 runc init 还没有退出
       // 这也是为什么容器外的 runc 等待子进程退出，却又向 pipe 里写数据的原因，
       // 因为孙 runc init 还在等着容器配置
       // 进程正常退出（不给 go 代码执行的机会）
			exit(0);
		}
```

**Child** 子进程加入了 init pipe 传递的 namespaces 配置，unshare 设置了 user namespace，并通知 parent 对 usermap(uid/gid map) 进行配置后，将当前容器的 uid 设置为 0 (root) ；最后创建将 fork 的 grantchild 进程pid发送给 parent 。

!FILENAME libcontainer/nsenter/nsexec.c:969

```c
		/*
		 * Stage 1: We're in the first child process. Our job is to join any
		 *          provided namespaces in the netlink payload and unshare all
		 *          of the requested namespaces. If we've been asked to
		 *          CLONE_NEWUSER, we will ask our parent (stage 0) to set up
		 *          our user mappings for us. Then, we create a new child
		 *          (stage 2: JUMP_INIT) for PID namespace. We then send the
		 *          child's PID to our parent (stage 0).
		 */  
	case JUMP_CHILD:{
			pid_t child;
			enum sync_t s;

			syncfd = sync_child_pipe[0];
			close(sync_child_pipe[1]);

			/* For debugging. */
			prctl(PR_SET_NAME, (unsigned long)"runc:[1:CHILD]", 0, 0, 0);

      // 通过 setns 加入现有的 namespaces 
			if (config.namespaces)
				join_namespaces(config.namespaces);

      // 如果 clone flag 里有 CLONE_NEWUSER，说明需要创建新的 user namespace，
      // 使用 unshare() 创建 user namespace 
			if (config.cloneflags & CLONE_NEWUSER) {
				if (unshare(CLONE_NEWUSER) < 0)
					bail("failed to unshare user namespace");
				config.cloneflags &= ~CLONE_NEWUSER;

				/* Switching is only necessary if we joined namespaces. */
				if (config.namespaces) {
					if (prctl(PR_SET_DUMPABLE, 1, 0, 0, 0) < 0)
						bail("failed to set process as dumpable");
				}
        
        // 等待父 runc init 配置 user map 
        // 发送 SYNC_USERMAP_PLS 消息给 parent ,并接收其 SYNC_USERMAP_ACK 确认消息
				s = SYNC_USERMAP_PLS;
				if (write(syncfd, &s, sizeof(s)) != sizeof(s))
					bail("failed to sync with parent: write(SYNC_USERMAP_PLS)");
				if (read(syncfd, &s, sizeof(s)) != sizeof(s))
					bail("failed to sync with parent: read(SYNC_USERMAP_ACK)");
				if (s != SYNC_USERMAP_ACK)
					bail("failed to sync with parent: SYNC_USERMAP_ACK: got %u", s);
				/* Switching is only necessary if we joined namespaces. */
				if (config.namespaces) {
					if (prctl(PR_SET_DUMPABLE, 0, 0, 0, 0) < 0)
						bail("failed to set process as dumpable");
				}

        // 设置当前进程的 uid 为 0，即容器内的 root 用户
				if (setresuid(0, 0, 0) < 0)
					bail("failed to become root in user namespace");
			}
	   	// 使用 unshare() 其他需要新建的 namespace
			if (unshare(config.cloneflags & ~CLONE_NEWCGROUP) < 0)
				bail("failed to unshare namespaces");


     // 创建孙进程，当前进程已经完成了 namespace 的设置，孙进程会继承这些设置
			child = clone_parent(&env, JUMP_INIT);
			if (child < 0)
				bail("unable to fork: init_func");

     // 将孙进程 PID 传给 parent 消息" SYNC_RECVPID_PLS + Grandchild_pid "
			s = SYNC_RECVPID_PLS;
			if (write(syncfd, &s, sizeof(s)) != sizeof(s)) {
				kill(child, SIGKILL);
				bail("failed to sync with parent: write(SYNC_RECVPID_PLS)");
			}
			if (write(syncfd, &child, sizeof(child)) != sizeof(child)) {
				kill(child, SIGKILL);
				bail("failed to sync with parent: write(childpid)");
			}
    
      // 等待父 runc init 接收PID 确认消息" SYNC_RECVPID_ACK "
			if (read(syncfd, &s, sizeof(s)) != sizeof(s)) {
				kill(child, SIGKILL);
				bail("failed to sync with parent: read(SYNC_RECVPID_ACK)");
			}
			if (s != SYNC_RECVPID_ACK) {
				kill(child, SIGKILL);
				bail("failed to sync with parent: SYNC_RECVPID_ACK: got %u", s);
			}

      // 发送 SYNC_CHILD_READY 给 parent , Child 任务已完成 
			s = SYNC_CHILD_READY;
			if (write(syncfd, &s, sizeof(s)) != sizeof(s)) {
				kill(child, SIGKILL);   
				bail("failed to sync with parent: write(SYNC_CHILD_READY)");
			}
			// 子 runc init 的工作到此结束，进程正常退出（不给 go 代码执行的机会）
			exit(0);
		}
```

**Grandchild** (final child) 孙进程是真正启动容器 entrypoint 的 init 进程，并且在启动之前，进行最后的环境准备工作(sid、uid、gid、cgroup namespace)，执行完成后return 至 init Go runtime 代码处继续执行。

!FILENAME libcontainer/nsenter/nsexec.c:969

```C
		/*
		 * Stage 2: We're the final child process, and the only process that will
		 *          actually return to the Go runtime. Our job is to just do the
		 *          final cleanup steps and then return to the Go runtime to allow
		 *          init_linux.go to run.
		 */
	case JUMP_INIT:{
       // 
			enum sync_t s;

			syncfd = sync_grandchild_pipe[0];   
			close(sync_grandchild_pipe[1]);
			close(sync_child_pipe[0]);
			close(sync_child_pipe[1]);

			/* For debugging. */
			prctl(PR_SET_NAME, (unsigned long)"runc:[2:INIT]", 0, 0, 0);

      // 等待（读取pipe） parent(祖父) 进程的 SYNC_GRANDCHILD 准备运行消息
			if (read(syncfd, &s, sizeof(s)) != sizeof(s))
				bail("failed to sync with parent: read(SYNC_GRANDCHILD)");
			if (s != SYNC_GRANDCHILD)
				bail("failed to sync with parent: SYNC_GRANDCHILD: got %u", s);
     
      // 设置sid 
			if (setsid() < 0)
				bail("setsid failed");
    
      // 设置uid root
			if (setuid(0) < 0)
				bail("setuid failed");
    
      // 设置gid root
			if (setgid(0) < 0)
				bail("setgid failed");

			if (!config.is_rootless_euid && config.is_setgroup) {
				if (setgroups(0, NULL) < 0)
					bail("setgroups failed");
			}

      // 等待来自容器外 runc 的 child pipe 的关于 cgroup namespace 的消息 0x80（CREATECGROUPNS）
			if (config.cloneflags & CLONE_NEWCGROUP) {
				uint8_t value;
        
        // 从 pipenum 读取，请注意此处还从 bootstrap 进程通迅 pipe 获取配置
				if (read(pipenum, &value, sizeof(value)) != sizeof(value))
					bail("read synchronisation value failed");
				if (value == CREATECGROUPNS) {
          // 使用 unshare() 创建 cgroup namespace
					if (unshare(CLONE_NEWCGROUP) < 0)
						bail("failed to unshare cgroup namespace");
				} else
					bail("received unknown synchronisation value");
			}

      // 发送孙进程准备完成的消息给 parent, 此消息发送后 parent 进程接收后已完成其全部任务退出
			s = SYNC_CHILD_READY;
			if (write(syncfd, &s, sizeof(s)) != sizeof(s))
				bail("failed to sync with patent: write(SYNC_CHILD_READY)");

      // 关闭资源
			/* Close sync pipes. */
			close(sync_grandchild_pipe[0]);
			/* Free netlink data. */
			nl_free(&config);

      // 父/祖父 runc init 都退出了
      // return，然后开始执行 go 代码
			return;
		}
	default:
		bail("unexpected jump value");
	}

	/* Should never be reached. */
	bail("should never be reached");
}
```

此时代码已 return 回到了 runC init 命令的 go 代码继续执行，执行的进程空间仍是已完成 namespace 配置后的最后的进程(即 grandchild 进程在容器流程中称为 init 进程)，后面的init go执行流程本文前面已有简单介绍，更详细的执行流程分析可参照《RunC 源码通读指南之 Run》。

**相关文档**： // TODO 补充链接

- 《RunC 源码通读指南之 Run》
- 《RunC 源码通读指南之 Create & Start》
- 《RunC 源码通读指南之 Cgroup》
- 《RunC 源码通读指南之 Networks》

**~本文 END~**

