# Proxy 服务框架

<!-- toc -->

## 程序入口与初始化

kube-proxy使用通用Cobra框架构建一个标准的CLI应用.同kubernets基础组件(如:scheduler)一样的方式来创建应用和运行应用，因此应用command构建层不将对其深入分析，我们将重心聚焦在后面的proxyserver创建与proxyserver run代码分析。

!FILENAME cmd/kube-proxy/proxy.go:35

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewProxyCommand()  //Cobra命令风格应用对象创建
  //......
  
  //对前面实例化Command对象的执行
	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```

**NewProxyCommand()**返回命令command对象,command.Execute()则调用定义的Run: func{},执行内部的opts.Run().

!FILENAME cmd/kube-proxy/app/server.go:370

```go
func NewProxyCommand() *cobra.Command {
  // 创建options对象(全局的应用配置对象)
	opts := NewOptions()

	cmd := &cobra.Command{
		Use: "kube-proxy",
		Long: `The Kubernetes network proxy runs on each node...略`,
		Run: func(cmd *cobra.Command, args []string) {   //Command应用RUN命令func定义
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cmd.Flags())
      
			if err := initForOS(opts.WindowsService); err != nil {
				klog.Fatalf("failed OS init: %v", err)
			}
      // 完成所有需求的options配置(外置配置文件、自定义主机名处理、特征功能开关设置等)
			if err := opts.Complete(); err != nil {
				klog.Fatalf("failed complete: %v", err)
			}
      // 校验kube-proxy配置的有效性
			if err := opts.Validate(args); err != nil {
				klog.Fatalf("failed validate: %v", err)
			}
      klog.Fatal(opts.Run())  //应用真正执行Run(),后面进行分析
		},
	}

	var err error
	opts.config, err = opts.ApplyDefaults(opts.config)   //应用默认配置，完成配置的初始化工作
	if err != nil {
		klog.Fatalf("unable to create flag defaults: %v", err)
	}

	opts.AddFlags(cmd.Flags())

	cmd.MarkFlagFilename("config", "yaml", "yml", "json")

  return cmd  //返回命令command对象
}
```

NewOptions() 全局配置对象创建

!FILENAME cmd/kube-proxy/app/server.go:184

```go
func NewOptions() *Options {
	return &Options{
		config:      new(kubeproxyconfig.KubeProxyConfiguration), //实例化配置config对象
		healthzPort: ports.ProxyHealthzPort,                      //Healthz端口
		metricsPort: ports.ProxyStatusPort,                       //metrics端口
		scheme:      scheme.Scheme,
		codecs:      scheme.Codecs,
		CleanupIPVS: true,
	}
}

//完整的配置结构体与命令执行时用户传递的命令选项相对应
type KubeProxyConfiguration struct {
	metav1.TypeMeta
	FeatureGates map[string]bool              //功能特征模块开关
	BindAddress string                        //默认所有接口0.0.0.0 
	HealthzBindAddress string                 //默认0.0.0.0:10256
	MetricsBindAddress string                 //默认127.0.0.1:10249
	EnableProfiling bool                      //"/debug/pprof"
	ClusterCIDR string                        //ClusterCIDR
	HostnameOverride string                   //自定义Hostname
	ClientConnection apimachineryconfig.ClientConnectionConfiguration  //apiserver client
  IPTables KubeProxyIPTablesConfiguration   //IPTABLES配置项(地址伪装、同步周期等)
  IPVS KubeProxyIPVSConfiguration           //IPVS配置项(同步周期、调度器等)
	OOMScoreAdj *int32                        //修改OOMScoreAdj分值
	Mode ProxyMode                            //proxy模式
	PortRange string                          //端口range
	ResourceContainer string                  //"Default: /kube-proxy"
	UDPIdleTimeout metav1.Duration            //UDP空闲超时
	Conntrack KubeProxyConntrackConfiguration //Conntrack对象
	ConfigSyncPeriod metav1.Duration          //同步周期
	NodePortAddresses []string                //Node地址
}

const (
	ProxyHealthzPort = 10256
  ProxyStatusPort = 10249
)

```

**opts.Run()** 创建proxy服务并启动

!FILENAME cmd/kube-proxy/app/server.go:250

```go
func (o *Options) Run() error {
	if len(o.WriteConfigTo) > 0 {
		return o.writeConfigFile()           //写配置文件
	}

	proxyServer, err := NewProxyServer(o)  //基于配置创建proxy服务
	if err != nil {
		return err
	}

	return proxyServer.Run()              //运行proxy服务，后续详细分析proxyserver执行代码逻辑
}
```

上面已完成对**kube-proxy第一层**(CLI创建、配置项初始化、启动)分析,下面进入proxy Server创建与启动运行代码分析( kube-proxy第二层 )。

## Proxy Server创建

**NewProxyServer()** *非windows版本代码* 

!FILENAME cmd/kube-proxy/app/server_others.go:55

```go
func NewProxyServer(o *Options) (*ProxyServer, error) {
	return newProxyServer(o.config, o.CleanupAndExit, o.CleanupIPVS, o.scheme, o.master)
}

func newProxyServer(
	config *proxyconfigapi.KubeProxyConfiguration,
	cleanupAndExit bool,
	cleanupIPVS bool,
	scheme *runtime.Scheme,
	master string) (*ProxyServer, error) {

	if config == nil {
		return nil, errors.New("config is required")
	}

	if c, err := configz.New(proxyconfigapi.GroupName); err == nil {
		c.Set(config)
	} else {
		return nil, fmt.Errorf("unable to register configz: %s", err)
	}

  //协议IPV4 or IPV6
	protocol := utiliptables.ProtocolIpv4
	if net.ParseIP(config.BindAddress).To4() == nil {
		klog.V(0).Infof("IPv6 bind address (%s), assume IPv6 operation", config.BindAddress)
		protocol = utiliptables.ProtocolIpv6
	}

  // 关键依赖工具实现的api接口iptables/ipvs/ipset/dbus
  // 从此处则可以看出kube-proxy底层技术的依赖（当然不同的proxy-mode，实现技术也不一样 ）
	var iptInterface utiliptables.Interface
	var ipvsInterface utilipvs.Interface
	var kernelHandler ipvs.KernelHandler
	var ipsetInterface utilipset.Interface
	var dbus utildbus.Interface


  // exec命令执行器对象创建
	execer := exec.New()
  // dbus对象创建（linux实现进程间通信机制）
	dbus = utildbus.New()
  // iptables操作对象创建
	iptInterface = utiliptables.New(execer, dbus, protocol)
  // IPVS
	kernelHandler = ipvs.NewLinuxKernelHandler()
  // ipset
	ipsetInterface = utilipset.New(execer)
  // IPVS环境检测
	canUseIPVS, _ := ipvs.CanUseIPVSProxier(kernelHandler, ipsetInterface)
	if canUseIPVS {
		ipvsInterface = utilipvs.New(execer)
	}

	// We omit creation of pretty much everything if we run in cleanup mode
	if cleanupAndExit {
		return &ProxyServer{
			execer:         execer,
			IptInterface:   iptInterface,
			IpvsInterface:  ipvsInterface,
			IpsetInterface: ipsetInterface,
			CleanupAndExit: cleanupAndExit,
		}, nil
	}

  //api client
	client, eventClient, err := createClients(config.ClientConnection, master)
	if err != nil {
		return nil, err
	}

	//主机名
	hostname, err := utilnode.GetHostname(config.HostnameOverride)
	if err != nil {
		return nil, err
	}
  // 事件广播器
	eventBroadcaster := record.NewBroadcaster()
  // Create event recorder
	recorder := eventBroadcaster.NewRecorder(scheme, v1.EventSource{Component: "kube-proxy", Host: hostname})

	nodeRef := &v1.ObjectReference{
		Kind:      "Node",
		Name:      hostname,
		UID:       types.UID(hostname),
		Namespace: "",
	}

	var healthzServer *healthcheck.HealthzServer
	var healthzUpdater healthcheck.HealthzUpdater
   
  //创建默认的healthzServer服务对象
	if len(config.HealthzBindAddress) > 0 {
		healthzServer = healthcheck.NewDefaultHealthzServer(config.HealthzBindAddress, 2*config.IPTables.SyncPeriod.Duration, recorder, nodeRef)
		healthzUpdater = healthzServer
	}

	var proxier proxy.ProxyProvider
	var serviceEventHandler proxyconfig.ServiceHandler
	var endpointsEventHandler proxyconfig.EndpointsHandler

  // proxyMode模式配置获取
	proxyMode := getProxyMode(string(config.Mode), iptInterface, kernelHandler, ipsetInterface, iptables.LinuxKernelCompatTester{})
  // 节点绑定IP
	nodeIP := net.ParseIP(config.BindAddress)
	if nodeIP.IsUnspecified() {
		nodeIP = utilnode.GetNodeIP(client, hostname)
	}
  
	if proxyMode == proxyModeIPTables {                   // proxyMode为"IPTables"
		klog.V(0).Info("Using iptables Proxier.")
		if config.IPTables.MasqueradeBit == nil {
			// MasqueradeBit must be specified or defaulted.
			return nil, fmt.Errorf("unable to read IPTables MasqueradeBit from config")
		}

		//创建iptables proxier对象
		proxierIPTables, err := iptables.NewProxier(
			iptInterface,
			utilsysctl.New(),
			execer,
			config.IPTables.SyncPeriod.Duration,
			config.IPTables.MinSyncPeriod.Duration,
			config.IPTables.MasqueradeAll,
			int(*config.IPTables.MasqueradeBit),
			config.ClusterCIDR,
			hostname,
			nodeIP,
			recorder,
			healthzUpdater,
			config.NodePortAddresses,
		)
		if err != nil {
			return nil, fmt.Errorf("unable to create proxier: %v", err)
		}
		metrics.RegisterMetrics()
    //iptables proxier对象和事件处理
		proxier = proxierIPTables               
		serviceEventHandler = proxierIPTables
		endpointsEventHandler = proxierIPTables
		// No turning back. Remove artifacts that might still exist from the userspace Proxier.
		klog.V(0).Info("Tearing down inactive rules.")
		//模式转换清理userspace/ipvs模式所创建iptables规则
		userspace.CleanupLeftovers(iptInterface)
		if canUseIPVS {
			ipvs.CleanupLeftovers(ipvsInterface, iptInterface, ipsetInterface, cleanupIPVS)
		}
	} else if proxyMode == proxyModeIPVS {                  // proxyMode为"IPVS"
		klog.V(0).Info("Using ipvs Proxier.")
    //ipvs proxier对象创建
		proxierIPVS, err := ipvs.NewProxier(
			iptInterface,
			ipvsInterface,
			ipsetInterface,
			utilsysctl.New(),
			execer,
			config.IPVS.SyncPeriod.Duration,
			config.IPVS.MinSyncPeriod.Duration,
			config.IPVS.ExcludeCIDRs,
			config.IPTables.MasqueradeAll,
			int(*config.IPTables.MasqueradeBit),
			config.ClusterCIDR,
			hostname,
			nodeIP,
			recorder,
			healthzServer,
			config.IPVS.Scheduler,
			config.NodePortAddresses,
		)
		if err != nil {
			return nil, fmt.Errorf("unable to create proxier: %v", err)
		}
		metrics.RegisterMetrics()
    //ipvs proxier对象和事件处理
		proxier = proxierIPVS
		serviceEventHandler = proxierIPVS
		endpointsEventHandler = proxierIPVS
		klog.V(0).Info("Tearing down inactive rules.")
    //模式转换清理userspace/iptables模式规则
		userspace.CleanupLeftovers(iptInterface)
		iptables.CleanupLeftovers(iptInterface)
	} else {                                              // proxyMode为"userspace"
		klog.V(0).Info("Using userspace Proxier.")
    //创建RR模式负载均衡
		loadBalancer := userspace.NewLoadBalancerRR()
    //设置EndpointsConfigHandler(endpoints事件处理)
		endpointsEventHandler = loadBalancer
    //创建userspace proxier对象
		proxierUserspace, err := userspace.NewProxier(
			loadBalancer,
			net.ParseIP(config.BindAddress),
			iptInterface,
			execer,
			*utilnet.ParsePortRangeOrDie(config.PortRange),
			config.IPTables.SyncPeriod.Duration,
			config.IPTables.MinSyncPeriod.Duration,
			config.UDPIdleTimeout.Duration,
			config.NodePortAddresses,
		)
		if err != nil {
			return nil, fmt.Errorf("unable to create proxier: %v", err)
		}
    //userspace proxier对象和service事件处理
		serviceEventHandler = proxierUserspace
		proxier = proxierUserspace

		klog.V(0).Info("Tearing down inactive rules.")
		//模式转换清理iptables/ipvs模式所创建iptables规则
		iptables.CleanupLeftovers(iptInterface)
		if canUseIPVS {
			ipvs.CleanupLeftovers(ipvsInterface, iptInterface, ipsetInterface, cleanupIPVS)
		}
	}
 
  //注册reloadfunc为proxier的sync()同步方法
	iptInterface.AddReloadFunc(proxier.Sync)
  
  // 构建ProxyServer对象
	return &ProxyServer{
		Client:                 client,                  //apiServer client
		EventClient:            eventClient,             //事件client
		IptInterface:           iptInterface,            //iptables接口
		IpvsInterface:          ipvsInterface,           //ipvs接口
		IpsetInterface:         ipsetInterface,          //ipset接口
		execer:                 execer,                  //exec命令执行器   
		Proxier:                proxier,                 //proxier创建对象
		Broadcaster:            eventBroadcaster,        //事件广播器
		Recorder:               recorder,                //事件记录器
		ConntrackConfiguration: config.Conntrack,        //Conntrack配置
		Conntracker:            &realConntracker{},      //Conntrack对象
		ProxyMode:              proxyMode,               //proxy模式
		NodeRef:                nodeRef,                 //node节点reference信息
		MetricsBindAddress:     config.MetricsBindAddress,        //metric服务地址配置
		EnableProfiling:        config.EnableProfiling,           //debug/pprof配置
		OOMScoreAdj:            config.OOMScoreAdj,               //OOMScoreAdj值配置
		ResourceContainer:      config.ResourceContainer,         //容器资源配置
		ConfigSyncPeriod:       config.ConfigSyncPeriod.Duration, //同步周期配置
		ServiceEventHandler:    serviceEventHandler,              //处理service事件proxier对象
		EndpointsEventHandler:  endpointsEventHandler,            //处理endpoints事件proxier对象
		HealthzServer:          healthzServer,                    //健康检测服务
	}, nil
}
```

## Proxy Server运行

!FILENAME cmd/kube-proxy/app/server.go:481

```go
func (s *ProxyServer) Run() error {
	klog.Infof("Version: %+v", version.Get())

  // 如果CleanupAndExit设置为true,则删除存在的所有iptables规则项，然后应用退出。
	if s.CleanupAndExit {
		encounteredError := userspace.CleanupLeftovers(s.IptInterface)
		encounteredError = iptables.CleanupLeftovers(s.IptInterface) || encounteredError
		encounteredError = ipvs.CleanupLeftovers(s.IpvsInterface, s.IptInterface, s.IpsetInterface, s.CleanupIPVS) || encounteredError
		if encounteredError {
			return errors.New("encountered an error while tearing down rules.")
		}
		return nil
	}

  // 根据启动参数配置"oom-score-adj"分值调整，取值区间[-1000, 1000]
	var oomAdjuster *oom.OOMAdjuster
	if s.OOMScoreAdj != nil {
		oomAdjuster = oom.NewOOMAdjuster()
		if err := oomAdjuster.ApplyOOMScoreAdj(0, int(*s.OOMScoreAdj)); err != nil {
			klog.V(2).Info(err)
		}
	}

  //"resource-container"设置是否运行在容器里
	if len(s.ResourceContainer) != 0 {
		if err := resourcecontainer.RunInResourceContainer(s.ResourceContainer); err != nil {
			klog.Warningf("Failed to start in resource-only container %q: %v", s.ResourceContainer, err)
		} else {
			klog.V(2).Infof("Running in resource-only container %q", s.ResourceContainer)
		}
	}

  //事件广播器
	if s.Broadcaster != nil && s.EventClient != nil {
		s.Broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: s.EventClient.Events("")})
	}

	// 根据配置启动healthz健康检测服务
	if s.HealthzServer != nil {
		s.HealthzServer.Run()
	}

  // 根据配置启动metrics服务。URI: "/proxyMode" 与 "/metrics"
	if len(s.MetricsBindAddress) > 0 {
		mux := mux.NewPathRecorderMux("kube-proxy")
		healthz.InstallHandler(mux)
		mux.HandleFunc("/proxyMode", func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "%s", s.ProxyMode)
		})
		mux.Handle("/metrics", prometheus.Handler())
		if s.EnableProfiling {
			routes.Profiling{}.Install(mux)
		}
		configz.InstallHandler(mux)
		go wait.Until(func() {
			err := http.ListenAndServe(s.MetricsBindAddress, mux)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("starting metrics server failed: %v", err))
			}
		}, 5*time.Second, wait.NeverStop)
	}

  // 如果需要(命令选项或配置项)调节conntrack配置值
	if s.Conntracker != nil {
		max, err := getConntrackMax(s.ConntrackConfiguration)
		if err != nil {
			return err
		}
		if max > 0 {
			err := s.Conntracker.SetMax(max)
			if err != nil {
				if err != readOnlySysFSError {
					return err
				}
        
				const message = "DOCKER RESTART NEEDED (docker issue #24000): /sys is read-only: " +
					"cannot modify conntrack limits, problems may arise later."
				s.Recorder.Eventf(s.NodeRef, api.EventTypeWarning, err.Error(), message)
			}
		}
     //设置conntracker的TCPEstablishedTimeout
		if s.ConntrackConfiguration.TCPEstablishedTimeout != nil && s.ConntrackConfiguration.TCPEstablishedTimeout.Duration > 0 {
			timeout := int(s.ConntrackConfiguration.TCPEstablishedTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPEstablishedTimeout(timeout); err != nil {
				return err
			}
		}
    //设置conntracker的TCPCloseWaitTimeout
		if s.ConntrackConfiguration.TCPCloseWaitTimeout != nil && s.ConntrackConfiguration.TCPCloseWaitTimeout.Duration > 0 {
			timeout := int(s.ConntrackConfiguration.TCPCloseWaitTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPCloseWaitTimeout(timeout); err != nil {
				return err
			}
		}
	}
 
   // informer机制获取与监听Services和Endpoints的配置与事件信息
   // 注册ServiceEventHandler服务事件的处理
   // 注册EndpointsEventHandler端点事件的处理
	informerFactory := informers.NewSharedInformerFactory(s.Client, s.ConfigSyncPeriod)
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.ServiceEventHandler)
	go serviceConfig.Run(wait.NeverStop)

	endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
	endpointsConfig.RegisterEventHandler(s.EndpointsEventHandler)
	go endpointsConfig.Run(wait.NeverStop)
  
	go informerFactory.Start(wait.NeverStop)

  // "新生儿降生的哭声"，作者命名比较有生活情趣^_^
  //  服务成功启动，将启动事件广播。
  //   s.Recorder.Eventf(s.NodeRef, api.EventTypeNormal, "Starting", "Starting kube-proxy.")
  //  	nodeRef := &v1.ObjectReference{
	//    	Kind:      "Node",
	//    	Name:      hostname,
	//    	UID:       types.UID(hostname),
	//   	Namespace: "",
	//   }
	s.birthCry()

  // Proxier(代理服务提供者)进行循环配置同步与处理proxy逻辑（本文聚焦应用主框架，后有专篇分析Proxier）
	// 此时将进入了proxier运行，默认使用的iptables模式的proxier对象
	s.Proxier.SyncLoop()
	return nil
}
```

s.Proxier.SyncLoop()的运行将进入**kube-proxy第三层**(service实现机制层),Proxier实例化对象是在proxy server对象创建时通过config配置文件或"-proxy-mode"指定（userspace / iptables / ipvs模式），而默认使用的iptables模式proxier对象。第三层的代码分析将针对三种模式设立专篇分析，此处为关键部分请记住此处，在后面proxier的分析文章重点关注。

在第二层的框架层我们还须关注kube-proxy与kubernetes集群同步信息的机制informer。kube-proxy组件在proxy server的run()运行创建service、endpoints的informer对其list同步数据和watch监听事件(add、delete、update)，调用注册的proxier handler进行处理(**第三层proxier的同步处理机制调用与触发**)。 

## 同步规则机制(Informer)

kube-proxy同样使用client-go标准的ApiServer同步方式，创建informer,注册事件处理器handler，持续监控watch事件并调用handler处理事件add/update/delete，后端处理则由proxier(userspace/iptables/ipvs)实现。因为**endpoints**的同步与**service同步**方式一致，则下面仅说明service代码实现。

!FILENAME pkg/proxy/config/config.go:174

```go
func NewServiceConfig(serviceInformer coreinformers.ServiceInformer, resyncPeriod time.Duration) *ServiceConfig {
	result := &ServiceConfig{
		lister:       serviceInformer.Lister(),                //监听器
		listerSynced: serviceInformer.Informer().HasSynced,    //监听器同步状态值
	}

  //在服务informer上添加了资源事件的处理器handleFunc。（Add/update/delete）
	serviceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    result.handleAddService,
			UpdateFunc: result.handleUpdateService,
			DeleteFunc: result.handleDeleteService,
		},
		resyncPeriod,     
	)

	return result
}
```

!FILENAME pkg/proxy/config/config.go:119

```go
func (c *ServiceConfig) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	klog.Info("Starting service config controller")
	defer klog.Info("Shutting down service config controller")

	if !controller.WaitForCacheSync("service config", stopCh, c.listerSynced) {
		return
	}

	for i := range c.eventHandlers {
		klog.V(3).Info("Calling handler.OnServiceSynced()")
		c.eventHandlers[i].OnServiceSynced()               //服务与proxier处理同步
	}

	<-stopCh
}
```

HandleAddService()新增事件处理

!FILENAME pkg/proxy/config/config.go:217

```go
func (c *ServiceConfig) handleAddService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
		return
	}
	for i := range c.eventHandlers {
		klog.V(4).Info("Calling handler.OnServiceAdd")
		c.eventHandlers[i].OnServiceAdd(service)            //服务新增事件与proxier处理同步
	}
}
```

HandleUpdateService()更新事件处理

!FILENAME pkg/proxy/config/config.go:229

```go
func (c *ServiceConfig) handleUpdateService(oldObj, newObj interface{}) {
	oldService, ok := oldObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", oldObj))
		return
	}
	service, ok := newObj.(*v1.Service)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", newObj))
		return
	}
	for i := range c.eventHandlers {
		klog.V(4).Info("Calling handler.OnServiceUpdate")
		c.eventHandlers[i].OnServiceUpdate(oldService, service) //服务更新事件与proxier处理同步 
	}
}
```

HandleDeleteService()删除事件处理

!FILENAME pkg/proxy/config/config.go:246

```go
func (c *ServiceConfig) handleDeleteService(obj interface{}) {
	service, ok := obj.(*v1.Service)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
			return
		}
		if service, ok = tombstone.Obj.(*v1.Service); !ok {
			utilruntime.HandleError(fmt.Errorf("unexpected object type: %v", obj))
			return
		}
	}
	for i := range c.eventHandlers {
		klog.V(4).Info("Calling handler.OnServiceDelete")
		c.eventHandlers[i].OnServiceDelete(service)         //服务删除事件与proxier处理同步
	}
}
```

第三层proxier分析，请参看iptables、ipvs、userspace-mode proxier分析文档。

**〜本文END〜**