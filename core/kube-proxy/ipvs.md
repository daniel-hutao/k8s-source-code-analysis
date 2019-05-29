# Ipvs-mode proxier

<!-- toc -->

## 概述

关于ipvs-mode proxier基础知识可参看官方文档([英文版](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs)、[中文版](https://www.codercto.com/a/22682.html))，其官方文档主要介绍以下几方面内容： 

1. ipvs技术简介和对比iptables-mode所带来的好处；
2. ipvs-mode proxier按用户配置不同所生成的用户层iptables规则示例(**masquerade-all/cluster-cidr/Load Balancer/NodePort/externalIPs**)； 
3. 如何kube-proxy运行ipvs模式、运行必要条件、运行debug和排错操作。

本文分析将聚焦在代码层的实现解析 (如运行时必要条件检测的代码实现是怎么样的？ipvs为实现service层是如何实现的？iptables规则代码是怎样的？proxier完整的实现逻辑与方式是怎样？等等) 。

ipvs proxy模式主要依赖几个底层技术如 ipvs/ipset /iptables/netlink(用户空间与内核空间异步通信机制),有必要预先对其基础用途或技术细节进行扩展知识的熟悉，将有助于对整个ipvs-mode proxier的实现更深层次的理解。 

Ipvs-mode proxier使用ipvs NAT模式实现，ipvs集群操作(如虚拟服务器、RealServer)是通过netlink内核通迅创建标准的协议格式通迅消息体进行交互实现。  Ipvs-mode proxier也同样使用了iptables固定模板规则结合ipset集来进行动态管理变化更新。

Ipvs-mode proxier整个代码机制逻辑与iptables-mode一致([参看iptable-mode代码逻辑示意图](https://github.com/farmer-hutao/k8s-source-code-analysis/blob/master/core/kube-proxy/image/iptables-proxier.png))。同样是通过同步apiserver事件及更新信息，生成相应的路由规则。但ipvs-mode服务规则不同于iptables-mode,不仅使用了ipset扩展的方式简化iptables规则条目和优化性能，而且还使用ipvs技术实现更丰富的集群负载策略管理。其规则生成操作须对ipset集、iptables规则、ipvs集群进行同步更新操作，关键逻辑代码在syncProxyRules()内。



## Ipvs-mode proxer 对象创建与初始化

**ProxyServer**实例化时初始化了proxier模式，如果代理模式指定为Ipvs,则创建proxier对象，且指定其service与endpoints的事件处理器。

!FILENAME cmd/kube-proxy/app/server_others.go:59

```go
func newProxyServer(...) (*ProxyServer, error) {
           //...
  else if proxyMode == proxyModeIPVS {          //当proxy模式指定为IPVS模式(命令参数或配置文件)
		klog.V(0).Info("Using ipvs Proxier.")
		proxierIPVS, err := ipvs.NewProxier(        //创建ipvs-mode proxier对象
		       //...
		)
    // porxyServer的proxier对象与事件处理器的指定
		proxier = proxierIPVS
		serviceEventHandler = proxierIPVS
		endpointsEventHandler = proxierIPVS
           //...
}
```

ipvs-mode  proxier对象实例化NewProxier()，对ipvs环境进行**初始化**。

相关内核参数调整说明：

- *net/ipv4/conf/all/route_localnet* 是否允许外部访问localhost；
- *net/bridge/bridge-nf-call-iptables*  为二层的网桥在转发包时也会被iptables的FORWARD规则所过滤，这样就会出现L3层的iptables rules去过滤L2的帧的问题；
- *net/ipv4/vs/conntrack*  开启[NFCT](http://ja.ssi.bg/nfct/HOWTO.txt)(Netfilter connection tracking连接与状态跟踪)；
- *net/ipv4/vs/conn_reuse_mode*  网络连接复用模式的选择；
- *net/ipv4/vs/expire_nodest_conn* 值为0，当LVS转发数据包，发现目的RS无效（删除）时，会丢弃该数据包，但不删除相应连接。值为1时，则马上释放相应连接；
- *net/ipv4/vs/expire_quiescent_template*  值为0，当RS的weight值=0（如，健康检测失败，应用程序将RS weight置0）时，会话保持的新建连接 还会继续调度到该RS上；值为1，则马上将会话保持的连接模板置为无效，重新调度新的RS。如果有会话保持的业务，建议该值配置为1；
- *net/ipv4/ip_forward* 是否打开ipv4的IP转发模式;
- *net/ipv4/conf/all/arp_ignore*  定义对目标地址为本地IP的ARP询问不同的应答模式(0~8),模式1表示：只回答目标IP地址是来访网络接口本地地址的ARP查询请求；
- *net/ipv4/conf/all/arp_announce* 对网络接口上，本地IP地址的发出的，ARP回应，作出相应级别的限制；值为2表示：对查询目标使用最适当的本地地址；

!FILENAME pkg/proxy/ipvs/proxier.go:280

```go
func NewProxier(...) (*Proxier, error) {
	// sysctl配置项 "net/ipv4/conf/all/route_localnet" 值为1
	if val, _ := sysctl.GetSysctl(sysctlRouteLocalnet); val != 1 {
		if err := sysctl.SetSysctl(sysctlRouteLocalnet, 1); err != nil {
			return nil, fmt.Errorf("can't set sysctl %s: %v", sysctlRouteLocalnet, err)
		}
	}
  //...
  // sysctl配置项 "net/bridge/bridge-nf-call-iptables"  值为1
  sysctl.GetSysctl(sysctlBridgeCallIPTables)
  // sysctl配置项 "net/ipv4/vs/conntrack" 值为1
  sysctl.SetSysctl(sysctlVSConnTrack, 1)
  // sysctl配置项 "net/ipv4/vs/conn_reuse_mode" 值为0
  sysctl.SetSysctl(sysctlConnReuse, 0)
  // sysctl配置项 "net/ipv4/vs/expire_nodest_conn" 值为1
  sysctl.SetSysctl(sysctlExpireNoDestConn, 1)
  // sysctl配置项 "net/ipv4/vs/expire_quiescent_template" 值为1
  sysctl.SetSysctl(sysctlExpireQuiescentTemplate, 1)
  // sysctl配置项 "net/ipv4/ip_forward" 值为1 
  sysctl.SetSysctl(sysctlForward, 1)
  // sysctl配置项 "net/ipv4/conf/all/arp_ignore" 值为1
  sysctl.SetSysctl(sysctlArpIgnore, 1)
  // sysctl配置项 "net/ipv4/conf/all/arp_announce" 值为2
  sysctl.SetSysctl(sysctlArpAnnounce, 2)
  //...
  // 生成masquerade标志用于SNAT规则
	masqueradeValue := 1 << uint(masqueradeBit)
	masqueradeMark := fmt.Sprintf("%#08x/%#08x", masqueradeValue, masqueradeValue)

  // node ip检测
	if nodeIP == nil {
		klog.Warningf("invalid nodeIP, initializing kube-proxy with 127.0.0.1 as nodeIP")
		nodeIP = net.ParseIP("127.0.0.1")
	}

	isIPv6 := utilnet.IsIPv6(nodeIP)
	klog.V(2).Infof("nodeIP: %v, isIPv6: %v", nodeIP, isIPv6)
  
  // 检测是否有为proxier配置clusterCIDR参数
  // clusterCIDR指定集群中pod使用的网段，以此来区分内部与外部流量
	if len(clusterCIDR) == 0 {
		klog.Warningf("clusterCIDR not specified, unable to distinguish between internal and external traffic")
	} else if utilnet.IsIPv6CIDR(clusterCIDR) != isIPv6 {
		return nil, fmt.Errorf("clusterCIDR %s has incorrect IP version: expect isIPv6=%t", clusterCIDR, isIPv6)
	}

  // 检测是否指定了proxy调度器scheduler算法，如果未指定，则为默认"RR"平均负载算法
	if len(scheduler) == 0 {
		klog.Warningf("IPVS scheduler not specified, use %s by default", DefaultScheduler)
		scheduler = DefaultScheduler
	}

  // healthcheck服务器对象创建
	healthChecker := healthcheck.NewServer(hostname, recorder, nil, nil) 

  // 创建Proxier对象
	proxier := &Proxier{
		//更新SVC、EP信息存放map和changeTracker
    portsMap:              make(map[utilproxy.LocalPort]utilproxy.Closeable),
		serviceMap:            make(proxy.ServiceMap),
		serviceChanges:        proxy.NewServiceChangeTracker(newServiceInfo, &isIPv6, recorder),
		endpointsMap:          make(proxy.EndpointsMap),
		endpointsChanges:      proxy.NewEndpointChangeTracker(hostname, nil, &isIPv6, recorder),
		//同步周期
    syncPeriod:            syncPeriod,
		minSyncPeriod:         minSyncPeriod,
    
		excludeCIDRs:          excludeCIDRs,
		iptables:              ipt,                      //iptables执行处理器
		masqueradeAll:         masqueradeAll,            //伪装所有访问Service的ClusterIP流量
		masqueradeMark:        masqueradeMark,           //伪装标志号 
		exec:                  exec,                    // osExec命令执行器
		clusterCIDR:           clusterCIDR,
		hostname:              hostname,
		nodeIP:                nodeIP,
		portMapper:            &listenPortOpener{},     
		recorder:              recorder,
		healthChecker:         healthChecker,
		healthzServer:         healthzServer,
		ipvs:                  ipvs,                   //ipvs接口
    ipvsScheduler:         scheduler,              //集群调度算法(默认RR)
		ipGetter:              &realIPGetter{nl: NewNetLinkHandle()}, //node ip获取器
    //iptables规则数据存放buffer
		iptablesData:          bytes.NewBuffer(nil),
		filterChainsData:      bytes.NewBuffer(nil),
		natChains:             bytes.NewBuffer(nil),
		natRules:              bytes.NewBuffer(nil),
		filterChains:          bytes.NewBuffer(nil),
		filterRules:           bytes.NewBuffer(nil),
  
		netlinkHandle:         NewNetLinkHandle(),         //netlink执行处理器
		ipset:                 ipset,                      //ipset执行处理器
		nodePortAddresses:     nodePortAddresses,    
		networkInterfacer:     utilproxy.RealNetwork{},             
		gracefuldeleteManager: NewGracefulTerminationManager(ipvs),   // RS清理管理器
	}
  // 遍历ipsetInfo定义，初始化kubernetes ipset默认集。（后面在ipset默认集创建时有详细介绍）
	proxier.ipsetList = make(map[string]*IPSet)
	for _, is := range ipsetInfo {
		proxier.ipsetList[is.name] = NewIPSet(ipset, is.name, is.setType, isIPv6, is.comment)
	}
	burstSyncs := 2
	klog.V(3).Infof("minSyncPeriod: %v, syncPeriod: %v, burstSyncs: %d", minSyncPeriod, syncPeriod, burstSyncs)
	proxier.syncRunner = async.NewBoundedFrequencyRunner("sync-runner", proxier.syncProxyRules, minSyncPeriod, syncPeriod, burstSyncs)   //同步runner
  proxier.gracefuldeleteManager.Run()   //后台线程定时(/分钟)清理RS(realServer记录) 
	return proxier, nil
}
```



## Proxier 服务与端点更新机制

ipvs模式和iptables模式的service和endpoints更新变化信息同步机制是一致的(更详细说明可参考iptables-mode proxier文章)，但为了本文的完整性和相对独立性，这里我们也简单的过一下部分代码。

在构建ipvs-mode proxier对象时指定同步运行器async.NewBoundedFrequencyRunner，同步proxy的规则处理则是syncProxyRules()。同样ipvs-proxier类对象有两个属性对象：**serviceChanges**(ServiceChangeTracker)和**endpointsChanges**(EndpointChangeTracker)是就是用来跟踪并记录service和endpoints的变化信息更新至相应的两个属性Items map(serviceChange和endpointsChange)。

!FILENAME pkg/proxy/ipvs/proxier.go:429

```go
proxier.syncRunner = async.NewBoundedFrequencyRunner("sync-runner", proxier.syncProxyRules, minSyncPeriod, syncPeriod, burstSyncs)
```

在框架层第二层proxy server的运行时最后的调用就是"s.Proxier.SyncLoop()"

!FILENAME pkg/proxy/ipvs/proxier.go:631

```go
func (proxier *Proxier) SyncLoop() {
	// Update healthz timestamp at beginning in case Sync() never succeeds.
  // ...
	proxier.syncRunner.Loop(wait.NeverStop)     //执行NewBoundedFrequencyRunner对象Loop
}
```

!FILENAME pkg/util/async/bounded_frequency_runner.go:169

```go
func (bfr *BoundedFrequencyRunner) Loop(stop <-chan struct{}) {
	bfr.timer.Reset(bfr.maxInterval)
	for {
		select {
		case <-stop:
			bfr.stop()
			return
		case <-bfr.timer.C():           //定时器方式执行
			bfr.tryRun()
		case <-bfr.run:                 //按需方式执行（发送运行指令信号）
			bfr.tryRun()
		}
	}
}
```

BoundedFrequencyRunner.*tryRun()* 按指定频率执行回调函数func  "bfr.fn()"

!FILENAME pkg/util/async/bounded_frequency_runner.go:211

```go
func (bfr *BoundedFrequencyRunner) tryRun() {
    bfr.mu.Lock()
    defer bfr.mu.Unlock()

  //限制条件允许运行func
    if bfr.limiter.TryAccept() {
         bfr.fn()                                 // 重点执行部分，调用func，上下文来看此处就是
                                                  // 对syncProxyRules()的调用
        bfr.lastRun = bfr.timer.Now()             // 记录运行时间
        bfr.timer.Stop()                          
        bfr.timer.Reset(bfr.maxInterval)          // 重设下次运行时间
        klog.V(3).Infof("%s: ran, next possible in %v, periodic in %v", bfr.name, bfr.minInterval, bfr.maxInterval)
        return
    }

  //限制条件不允许运行，计算下次运行时间
  elapsed := bfr.timer.Since(bfr.lastRun)    // elapsed:上次运行时间到现在已过多久
  nextPossible := bfr.minInterval - elapsed  // nextPossible:下次运行至少差多久（最小周期）
  nextScheduled := bfr.maxInterval - elapsed // nextScheduled:下次运行最迟差多久(最大周期)
    klog.V(4).Infof("%s: %v since last run, possible in %v, scheduled in %v", bfr.name, elapsed, nextPossible, nextScheduled)

    if nextPossible < nextScheduled {
        bfr.timer.Stop()
        bfr.timer.Reset(nextPossible)
        klog.V(3).Infof("%s: throttled, scheduling run in %v", bfr.name, nextPossible)
    }
}
```



## SyncProxyRules 同步 Proxy 规则

syncProxyRules()为proxier的**核心逻辑**，类似于iptables proxier实现了对apiserver同步的service、endpoints信息的同步与监听，同时在其生成初始和变化时同步ipvs规则（iptables、ipvs虚拟主机、ipset集规则），最终实现kubernetes的"service"机制。

syncProxyRules()代码部分过长，下面将分开对重点部分一一进行分析。

ipvs-mode proxier的同步ipvs规则主要完成以下几个主要步骤操作：

- 同步与新更service和endpoints；
- 初始化链和ipset集；
- 每个服务构建ipvs规则(iptables/ipvs/ipset),服务类型不同生成的规则也相应不同；
- 清理过旧规则及信息 。



### 更新 service 与 endpoint变化信息 

ipvs-mode proxier的service和endpoint变化更新的机制与iptables-mode的完全一致，详细可以参考iptables-mode的"syncProxyRule 同步配置与规则"内的相关内容，这里就不再详细赘述。

Proxier类对象有两个属性：**serviceChanges**和**endpointsChanges**是就是用来跟踪Service和Endpoint的更新信息，以及两个Tracker及方法：**ServiceChangeTracker**服务信息变更Tracker，**EndpointChangeTracker** 端点信息变更Tracker，实时监听apiserver的变更事件。

UpdateServiceMap() svc 服务的更新实现，将serviceChanges的服务项与proxier serviceMap进行更新(合并、删除废弃项)返回，UpdateEndpointsMap() 端点更新的实现，将endpointsChanges的端点项与proxier endpointMap进行更新(合并、删除废弃项)并返回已更新信息。

!FILENAME pkg/proxy/ipvs/proxier.go:730

```go
serviceUpdateResult := proxy.UpdateServiceMap(proxier.serviceMap, proxier.serviceChanges)
endpointUpdateResult := proxy.UpdateEndpointsMap(proxier.endpointsMap, proxier.endpointsChanges)
```

### 创建 kube 顶层链和连接信息

!FILENAME pkg/proxy/ipvs/proxier.go:748

```go
	proxier.natChains.Reset()     //nat链
	proxier.natRules.Reset()      //nat规则
	proxier.filterChains.Reset()  //filter链
	proxier.filterRules.Reset()   //filter规则
	//写表头
	writeLine(proxier.filterChains, "*filter")
	writeLine(proxier.natChains, "*nat")

	proxier.createAndLinkeKubeChain()  //创建kubernetes的表连接链数据
```

!FILENAME pkg/proxy/ipvs/proxier.go:1418

```go
func (proxier *Proxier) createAndLinkeKubeChain() {
  
  //通过iptables-save获取现有的filter和NAT表存在的链数据
	existingFilterChains := proxier.getExistingChains(proxier.filterChainsData, utiliptables.TableFilter)
	existingNATChains := proxier.getExistingChains(proxier.iptablesData, utiliptables.TableNAT)

	// 顶层链数据的构建
  // NAT表链： KUBE-SERVICES / KUBE-POSTROUTING / KUBE-FIREWALL 
  //          KUBE-NODE-PORT / KUBE-LOAD-BALANCER / KUBE-MARK-MASQ
  // Filter表链： KUBE-FORWARD
	for _, ch := range iptablesChains {
    //不存在则创建链，创建顶层链
		if _, err := proxier.iptables.EnsureChain(ch.table, ch.chain); err != nil {
			klog.Errorf("Failed to ensure that %s chain %s exists: %v", ch.table, ch.chain, err)
			return
		}
    //nat表写链
		if ch.table == utiliptables.TableNAT {
			if chain, ok := existingNATChains[ch.chain]; ok {
				writeBytesLine(proxier.natChains, chain)   //现存在的链
			} else {
        // "KUBE-POSTROUTING"
				writeLine(proxier.natChains, utiliptables.MakeChainLine(kubePostroutingChain))
			}
		} else {  // filter表写链
			if chain, ok := existingFilterChains[KubeForwardChain]; ok {
				writeBytesLine(proxier.filterChains, chain)  //现存在的链
			} else {
        // "KUBE-FORWARD"
				writeLine(proxier.filterChains, utiliptables.MakeChainLine(KubeForwardChain))
			}
		}
	}
  // 默认链下创建kubernete服务专用跳转规则
  // iptables -I OUTPUT -t nat --comment "kubernetes service portals" -j KUBE-SERVICES
  // iptables -I PREROUTING -t nat --comment "kubernetes service portals" -j KUBE-SERVICES
  // iptables -I POSTROUTING -t nat --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
  // iptables -I FORWARD -t filter --comment "kubernetes forwarding rules" -j KUBE-FORWARD
	for _, jc := range iptablesJumpChain {
		args := []string{"-m", "comment", "--comment", jc.comment, "-j", string(jc.to)}
		if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jc.table, jc.from, args...); err != nil {
			klog.Errorf("Failed to ensure that %s chain %s jumps to %s: %v", jc.table, jc.from, jc.to, err)
		}
	}
  // 写kubernetes专用的POSTROUTING nat规则
  // -A KUBE-POSTROUTING -m comment --comment "..." -m mark --mark $masqueradeMark -j MASQUERADE
	writeLine(proxier.natRules, []string{
		"-A", string(kubePostroutingChain),
		"-m", "comment", "--comment", `"kubernetes service traffic requiring SNAT"`,
		"-m", "mark", "--mark", proxier.masqueradeMark,
		"-j", "MASQUERADE",
	}...)

  // 写kubernetes专用的masquerade伪装地址标记规则
  // -A KUBE-MARK-MASQ -j MARK --set-xmark $masqueradeMark
	writeLine(proxier.natRules, []string{
		"-A", string(KubeMarkMasqChain),
		"-j", "MARK", "--set-xmark", proxier.masqueradeMark,
	}...)
}
```

### Dummy 接口和 ipset 默认集创建

!FILENAME pkg/proxy/ipvs/proxier.go:760

```go
 //为服务地址的绑定，确保已创建虚拟接口kube-ipvs0
	_, err := proxier.netlinkHandle.EnsureDummyDevice(DefaultDummyDevice)
	if err != nil {
		klog.Errorf("Failed to create dummy interface: %s, error: %v", DefaultDummyDevice, err)
		return
	}

 // 确保kubernetes专用的ipset集已创建
	for _, set := range proxier.ipsetList {
		if err := ensureIPSet(set); err != nil {
			return
		}
		set.resetEntries()
	}
```

proxier.ipsetList的定义信息,在proxier对象创建时初始化了ipsetList列表

!FILENAME pkg/proxy/ipvs/proxier.go:113

```go
var ipsetInfo = []struct {
	name    string             //ipset set名称
  setType utilipset.Type     //set类型{HashIPPortIP|HashIPPort|HashIPPortNet|BitmapPort}
	comment string             //comment描述信息
}{
	{kubeLoopBackIPSet, utilipset.HashIPPortIP, kubeLoopBackIPSetComment},
   //...
}
```

| ipset集名称                    | 类型             | 描述                                                         |
| ------------------------------ | ---------------- | ------------------------------------------------------------ |
| KUBE-LOOP-BACK                 | hash:ip,port,ip  | Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose |
| KUBE-CLUSTER-IP                | hash:ip,port     | Kubernetes service cluster ip + port for masquerade purpose  |
| KUBE-EXTERNAL-IP               | hash:ip,port     | Kubernetes service external ip + port for masquerade and filter purpose |
| KUBE-LOAD-BALANCER             | hash:ip,port     | Kubernetes service lb portal                                 |
| KUBE-LOAD-BALANCER-FW          | hash:ip,port     | Kubernetes service load balancer ip + port for load balancer with sourceRange |
| KUBE-LOAD-BALANCER-LOCAL       | hash:ip,port     | Kubernetes service load balancer ip + port with externalTrafficPolicy=local |
| KUBE-LOAD-BALANCER-SOURCE-IP   | hash:ip,port,ip  | Kubernetes service load balancer ip + port + source IP for packet filter purpose |
| KUBE-LOAD-BALANCER-SOURCE-CIDR | hash:ip,port,net | Kubernetes service load balancer ip + port + source cidr for packet filter purpose |
| KUBE-NODE-PORT-TCP             | BitmapPort       | Kubernetes nodeport TCP port for masquerade purpose          |
| KUBE-NODE-PORT-LOCAL-TCP       | BitmapPort       | BitmapPort,Kubernetes nodeport TCP port with externalTrafficPolicy=local |
| KUBE-NODE-PORT-UDP             | BitmapPort       | Kubernetes nodeport UDP port for masquerade purpose          |
| KUBE-NODE-PORT-LOCAL-UDP       | BitmapPort       | Kubernetes nodeport UDP port with externalTrafficPolicy=local |
| KUBE-NODE-PORT-SCTP            | BitmapPort       | Kubernetes nodeport SCTP port for masquerade purpose         |
| KUBE-NODE-PORT-LOCAL-SCTP      | BitmapPort       | Kubernetes nodeport SCTP port with externalTrafficPolicy=local |

### 每个服务生成 ipvs 规则 

代码逻辑包含在一个for循环内，对serviceMap内的每个服务进行遍历处理，对不同的服务类型(clusterip/nodePort/externalIPs/load-balancer)进行不同的处理(ipset集/ipvs虚拟主机/ipvs后端服务器)。

ipvs模式，通过svc创建的集群都绑定在默认dummy(kube-ipvs0)虚拟网卡，创建ipvs集群IP执行以下几项操作：

- 节点中存在虚拟接口为 kube-ipvs0,且服务 IP 地址绑定到虚拟接口
- 分别为每个kube服务 IP 地址创建 IPVS 虚拟服务器
- 为每个 IPVS 虚拟服务器创建RealServers (kube服务 endpoints)

!FILENAME pkg/proxy/ipvs/proxier.go:784

```go 
for svcName, svc := range proxier.serviceMap {
   //...... 后面详细分析
}
```

基于此服务的有效endpoint列表，更新**KUBE-LOOP-BACK**的ipset集，以备后面生成相应iptables规则(SNAT伪装地址)。

!FILENAME pkg/proxy/ipvs/proxier.go:796

```go
		for _, e := range proxier.endpointsMap[svcName] {
			ep, ok := e.(*proxy.BaseEndpointInfo)
			if !ok {
				klog.Errorf("Failed to cast BaseEndpointInfo %q", e.String())
				continue
			}
			if !ep.IsLocal {                //非本地
				continue
			}
			epIP := ep.IP()                 //端点IP
			epPort, err := ep.Port()        //端点Port
			
			if epIP == "" || err != nil {   //有效IP和端口正常
				continue
			}
      // 构造ipset集的entry记录项
			entry := &utilipset.Entry{
				IP:       epIP,
				Port:     epPort,
				Protocol: protocol,
				IP2:      epIP,
				SetType:  utilipset.HashIPPortIP,
			}
      // 类型校验KUBE-LOOP-BACK集合entry记录项
			if valid := proxier.ipsetList[kubeLoopBackIPSet].validateEntry(entry); !valid {
				klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoopBackIPSet].Name))
				continue
			}
      // 插入此entry记录至active记录队列
			proxier.ipsetList[kubeLoopBackIPSet].activeEntries.Insert(entry.String())
		}
```

**clusterIP**服务类型流量的承接(clusterIP为默认方式，仅资源集群内可访问),ipset集**KUBE-CLUSTER-IP**更新,以备后面生成相应iptables规则。

!FILENAME pkg/proxy/ipvs/proxier.go:827

```go
		//构建ipset entry
		entry := &utilipset.Entry{                      
			IP:       svcInfo.ClusterIP.String(),
			Port:     svcInfo.Port,
			Protocol: protocol,
			SetType:  utilipset.HashIPPort,
		}
		// 类型校验ipset entry
		if valid := proxier.ipsetList[kubeClusterIPSet].validateEntry(entry); !valid {
			klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeClusterIPSet].Name))
			continue
		}
    // 名为KUBE-CLUSTER-IP的ipset集插入entry,以备后面统一生成IPtables规则
		proxier.ipsetList[kubeClusterIPSet].activeEntries.Insert(entry.String())
		// 构建ipvs虚拟服务器VS服务对象
		serv := &utilipvs.VirtualServer{
			Address:   svcInfo.ClusterIP,
			Port:      uint16(svcInfo.Port),
			Protocol:  string(svcInfo.Protocol),
			Scheduler: proxier.ipvsScheduler,
		}
    // 设置IPVS服务的会话保持标志和超时时间
		if svcInfo.SessionAffinityType == v1.ServiceAffinityClientIP {
			serv.Flags |= utilipvs.FlagPersistent
			serv.Timeout = uint32(svcInfo.StickyMaxAgeSeconds)
		}

    // 将clusterIP绑定至dummy虚拟接口上，syncService()处理中需置bindAddr地址为True。
    // ipvs为服务创建VS(虚拟主机)
		if err := proxier.syncService(svcNameString, serv, true); err == nil {
			activeIPVSServices[serv.String()] = true
			activeBindAddrs[serv.Address.String()] = true
      // 为虚拟主机/服务(vip)同步endpoints信息。
      // IPVS为VS更新RS(realServer后端服务器)
			if err := proxier.syncEndpoint(svcName, false, serv); err != nil {
				klog.Errorf("Failed to sync endpoint for service: %v, err: %v", serv, err)
			}
		} else {
			klog.Errorf("Failed to sync service: %v, err: %v", serv, err)
		}
```

syncService()  更新和同步ipvs服务信息及服务IP与dummy接口的绑定

!FILENAME pkg/proxy/ipvs/proxier.go:1498

```go
func (proxier *Proxier) syncService(svcName string, vs *utilipvs.VirtualServer, bindAddr bool) error {
  //获取IPVS虚拟主机服务信息
	appliedVirtualServer, _ := proxier.ipvs.GetVirtualServer(vs) 
  //无此虚拟主机服务或此服务信息变更
	if appliedVirtualServer == nil || !appliedVirtualServer.Equal(vs) {
		if appliedVirtualServer == nil {
      // 服务未找到，则创建新的服务
			klog.V(3).Infof("Adding new service %q %s:%d/%s", svcName, vs.Address, vs.Port, vs.Protocol)
			if err := proxier.ipvs.AddVirtualServer(vs); err != nil {
				klog.Errorf("Failed to add IPVS service %q: %v", svcName, err)
				return err
			}
		} else {
      // 服务信息改变，则更新存在服务信息，在更新期间服务VIP不会关闭
			klog.V(3).Infof("IPVS service %s was changed", svcName)
			if err := proxier.ipvs.UpdateVirtualServer(vs); err != nil {
				klog.Errorf("Failed to update IPVS service, err:%v", err)
				return err
			}
		}

  // 将服务IP绑定到dummy接口上
	if bindAddr {
		klog.V(4).Infof("Bind addr %s", vs.Address.String())
		_, err := proxier.netlinkHandle.EnsureAddressBind(vs.Address.String(), DefaultDummyDevice) //netlinkHandle处理的实现在文章最后的netlink工具介绍部分详细说明 
		if err != nil {
			klog.Errorf("Failed to bind service address to dummy device %q: %v", svcName, err)
			return err
		}
	}
	return nil
}
```

syncEndpoint() 为虚拟主机/服务(clusterip)同步endpoints信息，实现ipvs为VS更新RS(realServer后端服务器)。

!FILENAME pkg/proxy/ipvs/proxier.go:1532

```go
func (proxier *Proxier) syncEndpoint(svcPortName proxy.ServicePortName, onlyNodeLocalEndpoints bool, vs *utilipvs.VirtualServer) error {
	appliedVirtualServer, err := proxier.ipvs.GetVirtualServer(vs)
	if err != nil || appliedVirtualServer == nil {
		klog.Errorf("Failed to get IPVS service, error: %v", err)
		return err
	}

  // curEndpoints表示当前系统IPVS目标列表
	curEndpoints := sets.NewString()
  // newEndpoints表示从apiServer监听到的Endpoints
	newEndpoints := sets.NewString()

  // 依据虚拟服务器获取RS(realservers)列表
	curDests, err := proxier.ipvs.GetRealServers(appliedVirtualServer)
	if err != nil {
		klog.Errorf("Failed to list IPVS destinations, error: %v", err)
		return err
	}
	for _, des := range curDests {
		curEndpoints.Insert(des.String())    // 写入curEndpoints
	}

  //迭代endpointsMaps信息，将非本地的enpoints写入newEndpoints
	for _, epInfo := range proxier.endpointsMap[svcPortName] {
		if onlyNodeLocalEndpoints && !epInfo.GetIsLocal() {
			continue
		}
		newEndpoints.Insert(epInfo.String())   
	}

	// 创建新的endpoints
	for _, ep := range newEndpoints.List() {
		ip, port, err := net.SplitHostPort(ep)
		if err != nil {
			klog.Errorf("Failed to parse endpoint: %v, error: %v", ep, err)
			continue
		}
		portNum, err := strconv.Atoi(port)
		if err != nil {
			klog.Errorf("Failed to parse endpoint port %s, error: %v", port, err)
			continue
		}

		newDest := &utilipvs.RealServer{
			Address: net.ParseIP(ip),
			Port:    uint16(portNum),
			Weight:  1,
		}
    //判断当前系统ipvs列表是否存在
		if curEndpoints.Has(ep) {
      //检测是否在gracefulDelete列表，如果是则此处立即删除
			uniqueRS := GetUniqueRSName(vs, newDest)
			if !proxier.gracefuldeleteManager.InTerminationList(uniqueRS) {
				continue
			}
			klog.V(5).Infof("new ep %q is in graceful delete list", uniqueRS)
			err := proxier.gracefuldeleteManager.MoveRSOutofGracefulDeleteList(uniqueRS)
			if err != nil {
				klog.Errorf("Failed to delete endpoint: %v in gracefulDeleteQueue, error: %v", ep, err)
				continue
			}
		}
    // 不存在则新增RealServer项(对应目标endpoint)
		err = proxier.ipvs.AddRealServer(appliedVirtualServer, newDest)
		if err != nil {
			klog.Errorf("Failed to add destination: %v, error: %v", newDest, err)
			continue
		}
	}
  // 删除过旧的endpoints
	for _, ep := range curEndpoints.Difference(newEndpoints).UnsortedList() {
    // 如果curEndpoint在gracefulDelete内，跳过
		uniqueRS := vs.String() + "/" + ep
		if proxier.gracefuldeleteManager.InTerminationList(uniqueRS) {
			continue
		}
		ip, port, err := net.SplitHostPort(ep)
		if err != nil {
			klog.Errorf("Failed to parse endpoint: %v, error: %v", ep, err)
			continue
		}
		portNum, err := strconv.Atoi(port)
		if err != nil {
			klog.Errorf("Failed to parse endpoint port %s, error: %v", port, err)
			continue
		}

		delDest := &utilipvs.RealServer{
			Address: net.ParseIP(ip),
			Port:    uint16(portNum),
		}

		klog.V(5).Infof("Using graceful delete to delete: %v", uniqueRS)
    // 删除RS
		err = proxier.gracefuldeleteManager.GracefulDeleteRS(appliedVirtualServer, delDest)
		if err != nil {
			klog.Errorf("Failed to delete destination: %v, error: %v", uniqueRS, err)
			continue
		}
	}
	return nil
}
```

**externalIPs**服务类型流量的承接，服务是否启用ExternalIPs,在指定的Node上开启监听端口(代码逻辑判断是否为本地ip),而非像nodeport所有节点监听。ipset集**KUBE-EXTERNAL-IP**更新，以备后面生成相应iptables规则。

!FILENAME pkg/proxy/ipvs/proxier.go:866

```go
		for _, externalIP := range svcInfo.ExternalIPs {
			if local, err := utilproxy.IsLocalIP(externalIP); err != nil {
				klog.Errorf("can't determine if IP is local, assuming not: %v", err)
        // 如果指定的externealIP为本地地址且协议不为SCTP
			} else if local && (svcInfo.GetProtocol() != v1.ProtocolSCTP) {
				lp := utilproxy.LocalPort{
					Description: "externalIP for " + svcNameString,
					IP:          externalIP,
					Port:        svcInfo.Port,
					Protocol:    protocol,
				}
				if proxier.portsMap[lp] != nil {   //端口已存在
					klog.V(4).Infof("Port %s was open before and is still needed", lp.String())
					replacementPortsMap[lp] = proxier.portsMap[lp]
				} else {
					socket, err := proxier.portMapper.OpenLocalPort(&lp)  //打开本地端口socket
					if err != nil {
						msg := fmt.Sprintf("can't open %s, skipping this externalIP: %v", lp.String(), err)

						proxier.recorder.Eventf(       //通知事件
							&v1.ObjectReference{
								Kind:      "Node",
								Name:      proxier.hostname,
								UID:       types.UID(proxier.hostname),
								Namespace: "",
							}, v1.EventTypeWarning, err.Error(), msg)
						klog.Error(msg)
						continue
					}
					replacementPortsMap[lp] = socket    //存放端口信息
				}
			} 

			// 创建ipset entry
			entry := &utilipset.Entry{
				IP:       externalIP,
				Port:     svcInfo.Port,
				Protocol: protocol,
				SetType:  utilipset.HashIPPort,
			}
			// We have to SNAT packets to external IPs.
			if valid := proxier.ipsetList[kubeExternalIPSet].validateEntry(entry); !valid {
				klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeExternalIPSet].Name))
				continue
			}
      // 名为KUBE-EXTERNAL-IP的ipset集插入entry,以备后面统一生成IPtables规则
			proxier.ipsetList[kubeExternalIPSet].activeEntries.Insert(entry.String())

			// 为服务定义ipvs虚拟主机信息
			serv := &utilipvs.VirtualServer{
				Address:   net.ParseIP(externalIP),
				Port:      uint16(svcInfo.Port),
				Protocol:  string(svcInfo.Protocol),
				Scheduler: proxier.ipvsScheduler,
			}
			if svcInfo.SessionAffinityType == v1.ServiceAffinityClientIP {
				serv.Flags |= utilipvs.FlagPersistent
				serv.Timeout = uint32(svcInfo.StickyMaxAgeSeconds)
			}
      // 将clusterIP绑定至dummy虚拟接口上，syncService()处理中需置bindAddr地址为True。
      // ipvs为服务创建VS(虚拟主机)
      // 为虚拟主机/服务同步endpoints信息。
      // IPVS为VS更新RS(realServer后端服务器)
      //...(同clusterip)
		}
```

**load-balancer**服务类型流量的承接,服务的LoadBalancerSourceRanges和externalTrafficPolicy=local被指定时将对KUBE-LOAD-BALANCER-LOCAL、KUBE-LOAD-BALANCER-FW、KUBE-LOAD-BALANCER-SOURCE-CIDR、KUBE-LOAD-BALANCER-SOURCE-IP ipset集更新,以备后面生成相应iptables规则。

!FILENAME pkg/proxy/ipvs/proxier.go:937

```go
		for _, ingress := range svcInfo.LoadBalancerStatus.Ingress {
			if ingress.IP != "" {
				// 构建ipset entry 
				entry = &utilipset.Entry{
					IP:       ingress.IP,
					Port:     svcInfo.Port,
					Protocol: protocol,
					SetType:  utilipset.HashIPPort,
				}
        // 增加SLB(service load balancer)ingressIP:Port与kube服务IP集对应
        // KUBE-LOAD-BALANCER ipset集更新 
				if valid := proxier.ipsetList[kubeLoadBalancerSet].validateEntry(entry); !valid {
					klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoadBalancerSet].Name))
					continue
				}
				proxier.ipsetList[kubeLoadBalancerSet].activeEntries.Insert(entry.String())
        // 服务指定externalTrafficPolicy=local时,KUBE-LOAD-BALANCER-LOCAL ipset集更新
				if svcInfo.OnlyNodeLocalEndpoints {
					if valid := proxier.ipsetList[kubeLoadBalancerLocalSet].validateEntry(entry); !valid {
						klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoadBalancerLocalSet].Name))
						continue
					}
					proxier.ipsetList[kubeLoadBalancerLocalSet].activeEntries.Insert(entry.String())
				}
        // 服务的LoadBalancerSourceRanges被指定时，基于源IP保护的防火墙策略开启，KUBE-LOAD-BALANCER-FW ipset集更新
				if len(svcInfo.LoadBalancerSourceRanges) != 0 {
					if valid := proxier.ipsetList[kubeLoadbalancerFWSet].validateEntry(entry); !valid {
						klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoadbalancerFWSet].Name))
						continue
					}
					proxier.ipsetList[kubeLoadbalancerFWSet].activeEntries.Insert(entry.String())
					allowFromNode := false
					for _, src := range svcInfo.LoadBalancerSourceRanges {
						// 构建ipset entry
						entry = &utilipset.Entry{
							IP:       ingress.IP,
							Port:     svcInfo.Port,
							Protocol: protocol,
							Net:      src,
							SetType:  utilipset.HashIPPortNet,
						}
            // 枚举所有源CIDR白名单列表，KUBE-LOAD-BALANCER-SOURCE-CIDR ipset集更新
						if valid := proxier.ipsetList[kubeLoadBalancerSourceCIDRSet].validateEntry(entry); !valid {
							klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoadBalancerSourceCIDRSet].Name))
							continue
						}
						proxier.ipsetList[kubeLoadBalancerSourceCIDRSet].activeEntries.Insert(entry.String())

						// ignore error because it has been validated
						_, cidr, _ := net.ParseCIDR(src)
						if cidr.Contains(proxier.nodeIP) {
							allowFromNode = true
						}
					}
          
          // 允许来自Node流量（LB对应后端hosts之间交互）
					if allowFromNode {
						entry = &utilipset.Entry{
							IP:       ingress.IP,
							Port:     svcInfo.Port,
							Protocol: protocol,
							IP2:      ingress.IP,
							SetType:  utilipset.HashIPPortIP,
						}
            
            // 枚举所有白名单源IP列表，KUBE-LOAD-BALANCER-SOURCE-IP ipset集更新
						if valid := proxier.ipsetList[kubeLoadBalancerSourceIPSet].validateEntry(entry); !valid {
							klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, proxier.ipsetList[kubeLoadBalancerSourceIPSet].Name))
							continue
						}
						proxier.ipsetList[kubeLoadBalancerSourceIPSet].activeEntries.Insert(entry.String())
					}
				}

				// 构建ipvs 虚拟主机对象
				serv := &utilipvs.VirtualServer{
					Address:   net.ParseIP(ingress.IP),   // SLB ip
					Port:      uint16(svcInfo.Port),      // SLB 端口
					Protocol:  string(svcInfo.Protocol),  // 协议
					Scheduler: proxier.ipvsScheduler,     // RR
				}
				if svcInfo.SessionAffinityType == v1.ServiceAffinityClientIP {
					serv.Flags |= utilipvs.FlagPersistent
					serv.Timeout = uint32(svcInfo.StickyMaxAgeSeconds)
				}
         // ipvs为服务创建VS(虚拟主机)，LB ingressIP绑定dummy接口
         // ipvs为VS更新RS(realServer后端服务器)
			   //...(同clusterip)
			}
		}
```

**NodePort**服务类型流量的承接,服务将在每个节点上都将开启指定的nodeport端口,并更新相应的ipset集。

!FILENAME pkg/proxy/ipvs/proxier.go:1040

```go
if svcInfo.NodePort != 0 {
			addresses, err := utilproxy.GetNodeAddresses(proxier.nodePortAddresses, proxier.networkInterfacer)  // 获取node addresses
			if err != nil {
				klog.Errorf("Failed to get node ip address matching nodeport cidr: %v", err)
				continue
			}

			var lps []utilproxy.LocalPort
			for address := range addresses {   
				lp := utilproxy.LocalPort{
					Description: "nodePort for " + svcNameString,
					IP:          address,
					Port:        svcInfo.NodePort,
					Protocol:    protocol,
				}
				if utilproxy.IsZeroCIDR(address) {
					// Empty IP address means all
					lp.IP = ""
					lps = append(lps, lp)       
					break
				}
        lps = append(lps, lp)  //整理与格式化后的lps列表
			}

      // 为node节点的IPs打开端口并保存持有socket句柄
			for _, lp := range lps {
				if proxier.portsMap[lp] != nil {
					klog.V(4).Infof("Port %s was open before and is still needed", lp.String())
					replacementPortsMap[lp] = proxier.portsMap[lp]
          
				} else if svcInfo.GetProtocol() != v1.ProtocolSCTP {
          // 打开和监听端口(非SCTP协议)
					socket, err := proxier.portMapper.OpenLocalPort(&lp)
					if err != nil {
						klog.Errorf("can't open %s, skipping this nodePort: %v", lp.String(), err)
						continue
					}
					if lp.Protocol == "udp" {
            // UDP协议，清理udp conntrack记录
						isIPv6 := utilnet.IsIPv6(svcInfo.ClusterIP)
						conntrack.ClearEntriesForPort(proxier.exec, lp.Port, isIPv6, v1.ProtocolUDP)
					}
					replacementPortsMap[lp] = socket
				} //socket保存
			}

			// Nodeports无论是否为本地都需要SNAT
			// 构建ipset entry
			entry = &utilipset.Entry{
				// No need to provide ip info
				Port:     svcInfo.NodePort,      
				Protocol: protocol,
				SetType:  utilipset.BitmapPort,
			}
			var nodePortSet *IPSet
      //基于协议类型选择ipset集
			switch protocol {      
			case "tcp":               // KUBE-NODE-PORT-TCP 
				nodePortSet = proxier.ipsetList[kubeNodePortSetTCP]
			case "udp":               // KUBE-NODE-PORT-UDP
				nodePortSet = proxier.ipsetList[kubeNodePortSetUDP]
			case "sctp":              // KUBE-NODE-PORT-SCTP
				nodePortSet = proxier.ipsetList[kubeNodePortSetSCTP]
			default:
				klog.Errorf("Unsupported protocol type: %s", protocol)
			}
			if nodePortSet != nil {
				if valid := nodePortSet.validateEntry(entry); !valid {
					klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, nodePortSet.Name))
					continue
				}
        // 更新ipset集
				nodePortSet.activeEntries.Insert(entry.String())
			}

      // 服务externaltrafficpolicy=local指定时，基于协议类型更新ipset集entry
			if svcInfo.OnlyNodeLocalEndpoints {
				var nodePortLocalSet *IPSet
				switch protocol {
				case "tcp":           //KUBE-NODE-PORT-LOCAL-TCP
					nodePortLocalSet = proxier.ipsetList[kubeNodePortLocalSetTCP]
				case "udp":           //KUBE-NODE-PORT-LOCAL-UDP
					nodePortLocalSet = proxier.ipsetList[kubeNodePortLocalSetUDP]
				case "sctp":          //KUBE-NODE-PORT-LOCAL-SCTP
					nodePortLocalSet = proxier.ipsetList[kubeNodePortLocalSetSCTP]
				default:
					klog.Errorf("Unsupported protocol type: %s", protocol)
				}
				if nodePortLocalSet != nil {
					if valid := nodePortLocalSet.validateEntry(entry); !valid {
						klog.Errorf("%s", fmt.Sprintf(EntryInvalidErr, entry, nodePortLocalSet.Name))
						continue
					}
          //  更新ipset集
					nodePortLocalSet.activeEntries.Insert(entry.String())
				}
			}

      // 为Node每个ip address创建ipvs路由项(VS/RS)
			var nodeIPs []net.IP
			for address := range addresses {
				if !utilproxy.IsZeroCIDR(address) {
					nodeIPs = append(nodeIPs, net.ParseIP(address))
					continue
				}
				// zero cidr
				nodeIPs, err = proxier.ipGetter.NodeIPs()  
				if err != nil {
					klog.Errorf("Failed to list all node IPs from host, err: %v", err)
				}
			}
			for _, nodeIP := range nodeIPs {
				// 构建ipvs VS对象
				serv := &utilipvs.VirtualServer{
					Address:   nodeIP,                     //node ip地址
					Port:      uint16(svcInfo.NodePort),   //node端口
					Protocol:  string(svcInfo.Protocol),   //协议
					Scheduler: proxier.ipvsScheduler,      //RR
				}
				if svcInfo.SessionAffinityType == v1.ServiceAffinityClientIP {
					serv.Flags |= utilipvs.FlagPersistent
					serv.Timeout = uint32(svcInfo.StickyMaxAgeSeconds)
				}
        // 这里不需要将Node IP绑定到dummy接口，参数值为false
        // ipvs为服务创建VS(虚拟主机)
        // ipvs为VS更新RS(realServer后端服务器)
        //...(同clusterip)
		}
```

### SyncIPSetEntries 同步 ipset 记录

!FILENAME pkg/proxy/ipvs/proxier.go:1176

```go
for _, set := range proxier.ipsetList {
	set.syncIPSetEntries()
}
```

!FILENAME pkg/proxy/ipvs/ipset.go:125

```go
func (set *IPSet) syncIPSetEntries() {
	appliedEntries, err := set.handle.ListEntries(set.Name)
	if err != nil {
		klog.Errorf("Failed to list ip set entries, error: %v", err)
		return
	}

  // currentIPSetEntries代表从apiServer上一直监听着的endpoints列表
	currentIPSetEntries := sets.NewString()
	for _, appliedEntry := range appliedEntries {
		currentIPSetEntries.Insert(appliedEntry)
	}
  
  // 求差集
  // s1 = {a1, a2, a3}
  // s2 = {a1, a2, a4, a5}
  // s1.Difference(s2) = {a3}
  // s2.Difference(s1) = {a4,a5}
	if !set.activeEntries.Equal(currentIPSetEntries) {
		// 清理过旧记录（取currentIPSetEntries在activeEntries中没有的entries）
		for _, entry := range currentIPSetEntries.Difference(set.activeEntries).List() {
			if err := set.handle.DelEntry(entry, set.Name); err != nil {
				if !utilipset.IsNotFoundError(err) {
					klog.Errorf("Failed to delete ip set entry: %s from ip set: %s, error: %v", entry, set.Name, err)
				}
			} else {
				klog.V(3).Infof("Successfully delete legacy ip set entry: %s from ip set: %s", entry, set.Name)
			}
		}
		// 新增记录（取activeEntries在currentIPSetEntries中没有的entries）
		for _, entry := range set.activeEntries.Difference(currentIPSetEntries).List() {
			if err := set.handle.AddEntry(entry, &set.IPSet, true); err != nil {
				klog.Errorf("Failed to add entry: %v to ip set: %s, error: %v", entry, set.Name, err)
			} else {
				klog.V(3).Infof("Successfully add entry: %v to ip set: %s", entry, set.Name)
			}
		}
	}
}
```

### 创建 iptables 规则数据

!FILENAME pkg/proxy/ipvs/proxier.go:1182

```go
proxier.writeIptablesRules() 
```

基于ipset定义创建iptables NAT表的kubernetes初始固定链规则数据。

!FILENAME pkg/proxy/ipvs/proxier.go:1269

```go
	for _, set := range ipsetWithIptablesChain {
		if _, find := proxier.ipsetList[set.name]; find && !proxier.ipsetList[set.name].isEmpty() {
			args = append(args[:0], "-A", set.from)
			if set.protocolMatch != "" {
				args = append(args, "-p", set.protocolMatch)
			}
			args = append(args,
				"-m", "comment", "--comment", proxier.ipsetList[set.name].getComment(),
				"-m", "set", "--match-set", set.name,
				set.matchType,
			)
      // -A $setFrom -p $prot -m comment --comment $commentStr 
      //    -m set --match-set $setName $setType -j $setTo
			writeLine(proxier.natRules, append(args, "-j", set.to)...)
		}
	}
```

> 依据ipsetWithIptablesChain定义生成以下创建固定链规则数据
>
> **KUBE-POSTROUTING匹配KUBE-LOOP-BACK ipset表则伪装地址**
>
> *-A **KUBE-POSTROUTING** -m comment --comment "Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose" -m set --match-set **KUBE-LOOP-BACK** dst,dst,src -j  MASQUERADE*
>
> **LoadBalancer服务类型相关规则**
>
> *-A **KUBE-SERVICES** -m comment --comment "Kubernetes service lb portal"  -m set --match-set **KUBE-LOAD-BALANCER** dst,dst -j KUBE-LOAD-BALANCER*
>
> *-A **KUBE-LOAD-BALANCER** -m comment --comment "Kubernetes service load balancer ip + port for load balancer with sourceRange" -m set --match-set **KUBE-LOAD-BALANCER-FW** dst,dst -j KUBE-FIREWALL* 
> *-A **KUBE-FIREWALL** -m comment --comment "Kubernetes service load balancer ip + port + source cidr for packet filter" -m set --match-set **KUBE-LOAD-BALANCER-SOURCE-CIDR**  dst,dst,src -j RETURN*
> *-A **KUBE-FIREWALL** -m comment --comment "Kubernetes service load balancer ip + port + source IP for packet filter purpose" -m set --match-set **KUBE-LOAD-BALANCER-SOURCE-IP** dst,dst,src -j RETURN* 
> *-A **KUBE-LOAD-BALANCER** -m comment --comment "Kubernetes service load balancer ip + port with externalTrafficPolicy=local" -m set --match-set **KUBE-LOAD-BALANCER-LOCAL** dst,dst -j RETURN*
>
> **Nodeport服务类型相关规则**
>
> *-A **KUBE-NODE-PORT** -p tcp -m comment --comment "Kubernetes service load balancer ip + port with externalTrafficPolicy=local" -m set --match-set **KUBE-NODE-PORT-LOCAL-TCP** dst -j RETURN*
> *-A **KUBE-NODE-PORT** -p tcp -m comment --comment "Kubernetes nodeport TCP port for masquerade purpose" -m set --match-set **KUBE-NODE-PORT-TCP** dst -j KUBE-MARK-MASQ*
> *-A **KUBE-NODE-PORT** -p udp -m comment --comment "Kubernetes nodeport UDP port with externalTrafficPolicy=local" -m set --match-set **KUBE-NODE-PORT-LOCAL-UDP** dst -j RETURN*
> *-A **KUBE-NODE-PORT** -p udp -m comment --comment "Kubernetes nodeport UDP port for masquerade purpose" -m set --match-set **KUBE-NODE-PORT-UDP** dst -j KUBE-MARK-MASQ*
> *-A **KUBE-SERVICES** -p sctp -m comment --comment "Kubernetes nodeport SCTP port for masquerade purpose" -m set --match-set **KUBE-NODE-PORT-SCTP** dst -j KUBE-NODE-PORT*
> *-A **KUBE-NODE-PORT** -p sctp -m comment --comment "Kubernetes nodeport SCTP port with externalTrafficPolicy=local" -m set --match-set **KUBE-NODE-PORT-LOCAL-SCTP** dst -j RETURN*

kube-proxy启动参数"--masquerade-all=true"， 针对类型为clusterip服务生成相应的NAT表KUBE-SERVICES链规则数据，masquerade-all实现访问service ip流量伪装。

!FILENAME pkg/proxy/ipvs/proxier.go:1284

```go
//ipset名称为"KUBE-CLUSTER-IP"不为空，即clusterip类型服务	
if !proxier.ipsetList[kubeClusterIPSet].isEmpty() {
		args = append(args[:0],
			"-A", string(kubeServicesChain),
			"-m", "comment", "--comment", proxier.ipsetList[kubeClusterIPSet].getComment(),
			"-m", "set", "--match-set", kubeClusterIPSet,
		)
     //当proxy配置为masqueradeAll=true
		if proxier.masqueradeAll {
      //nat表：-A KUBE-SERVICES -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP dst,dst -j KUBE-MARK-MASQ
			writeLine(proxier.natRules, append(args, "dst,dst", "-j", string(KubeMarkMasqChain))...)
		} else if len(proxier.clusterCIDR) > 0 {
      //当指定了clusterCIDR，针对非集群到服务VIP的流量masquerades规则 （dst,dst 目标ip:目标端口）
      // nat表：-A KUBE-SERVICES -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP dst,dst ! -s $clusterCIDR -j KUBE-MARK-MASQ
			writeLine(proxier.natRules, append(args, "dst,dst", "! -s", proxier.clusterCIDR, "-j", string(KubeMarkMasqChain))...)
		} else {
      // 所有来自服务VIP出流量masquerades规则 （src,dst 源ip:目标端口）
      // 如：VIP:<random port> to VIP:<service port>
      // nat表：-A KUBE-SERVICES -m comment --comment "Kubernetes service cluster ip + port for masquerade purpose" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ
			writeLine(proxier.natRules, append(args, "src,dst", "-j", string(KubeMarkMasqChain))...)
		}
	}
```

为服务externalIPs专用ipset集(存在配置externalIPs的服务)生成相应的iptables NAT表规则数据。

!FILENAME pkg/proxy/ipvs/proxier.go:1311

```go
	if !proxier.ipsetList[kubeExternalIPSet].isEmpty() {
    // 为external IPs添加masquerade规则
		args = append(args[:0],
			"-A", string(kubeServicesChain),
			"-m", "comment", "--comment", proxier.ipsetList[kubeExternalIPSet].getComment(),
			"-m", "set", "--match-set", kubeExternalIPSet,
			"dst,dst",
		)
    // -A KUBE-SERVICES -m comment --comment "Kubernetes service external ip + port for masquerade and filter purpose" -m set --match-set KUBE-EXTERNAL-IP dst,dst -j KUBE-MARK-MASQ
		writeLine(proxier.natRules, append(args, "-j", string(KubeMarkMasqChain))...)

    // 允许external ips流量,而非来自本地网桥流量(如来自一个容器流量或本地处理的forward至服务流量)
		externalTrafficOnlyArgs := append(args,
			"-m", "physdev", "!", "--physdev-is-in",
			"-m", "addrtype", "!", "--src-type", "LOCAL")
    // -m set match-set KUBE-EXTERNAL-IP dst,dst -m PHYSDEV  ! --physdev-is-in -m addrtype ! --src-type LOCAL -j ACCEPT
		writeLine(proxier.natRules, append(externalTrafficOnlyArgs, "-j", "ACCEPT")...)
		dstLocalOnlyArgs := append(args, "-m", "addrtype", "--dst-type", "LOCAL")
	
    // 识别与允许本地流量
    // -m set match-set KUBE-EXTERNAL-IP dst,dst -m addrtype --dst-type LOCAL -j ACCEPT
		writeLine(proxier.natRules, append(dstLocalOnlyArgs, "-j", "ACCEPT")...)
	}
```

acceptIPVSTraffic 在NAT表的KUBE-SERVICE链最后添加对所有目地址为ipvs虚拟服务的流量ACCEPT规则（此规则应放置于KUBE-SERVICE的最底部）。默认服务类型clusterip则生成规则*-A KUBE-SERVICE -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT*，如果有服务类型为LoadBalancer则生成规则*-A KUBE-SERVICE -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT*。

!FILENAME pkg/proxy/ipvs/proxier.go:1397

```go
proxier.acceptIPVSTraffic()
//  -A KUBE-SERVICE -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT
//  -A KUBE-SERVICE -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT
func (proxier *Proxier) acceptIPVSTraffic() {
	sets := []string{kubeClusterIPSet, kubeLoadBalancerSet}
	for _, set := range sets {
		var matchType string
		if !proxier.ipsetList[set].isEmpty() {
			switch proxier.ipsetList[set].SetType {
			case utilipset.BitmapPort:
				matchType = "dst"
			default:
				matchType = "dst,dst"   //目标ip，目标端口
			}
			writeLine(proxier.natRules, []string{
				"-A", string(kubeServicesChain),
				"-m", "set", "--match-set", set, matchType,
				"-j", "ACCEPT",
			}...)
		}
	}
}
```

增加masqueradeMark，允许NodePort流量转发(即使默认FORWARD规则策略不允许)。

!FILENAME pkg/proxy/ipvs/proxier.go:1361

```go
 // -A KUBE-FORWARD -m comment --comment "kubernetes forwarding rules" -m mark --mark 0x4000 -j ACCEPT
	writeLine(proxier.filterRules,
		"-A", string(KubeForwardChain),
		"-m", "comment", "--comment", `"kubernetes forwarding rules"`,
		"-m", "mark", "--mark", proxier.masqueradeMark,
		"-j", "ACCEPT",
	)
```

clusterCIDR被指定时生成两条filter表KUBE-FORWARD链规则数据，接受源或目标来自一个pod流量。(注：kube-proxy组件配置**--cluster-dir**参数指定集群中pod使用的网段)

!FILENAME pkg/proxy/ipvs/proxier.go:1369

```go
	if len(proxier.clusterCIDR) != 0 {
    // 两条规则确保kubernetes forward规则定义的初始包被接受（clusterCIDR所指定的源或目标流量）
    
    // -A KUBE-FORWARD -s $clusterCIDR -m -comment --comment "kubernetes forwarding conntrack pod source rule" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
		writeLine(proxier.filterRules,
			"-A", string(KubeForwardChain),
			"-s", proxier.clusterCIDR,
			"-m", "comment", "--comment", `"kubernetes forwarding conntrack pod source rule"`,
			"-m", "conntrack",
			"--ctstate", "RELATED,ESTABLISHED",
			"-j", "ACCEPT",
		)
    // -A KUBE-FORWARD  -m -comment --comment "kubernetes forwarding conntrack pod source rule" -d $clusterCIDR -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
		writeLine(proxier.filterRules,
			"-A", string(KubeForwardChain),
			"-m", "comment", "--comment", `"kubernetes forwarding conntrack pod destination rule"`,
			"-d", proxier.clusterCIDR,
			"-m", "conntrack",
			"--ctstate", "RELATED,ESTABLISHED",
			"-j", "ACCEPT",
		)
	}
```

### 刷新 iptables 规则

!FILENAME pkg/proxy/ipvs/proxier.go:1186

```go
	// 合并iptables规则
	proxier.iptablesData.Reset()
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())

	klog.V(5).Infof("Restoring iptables rules: %s", proxier.iptablesData.Bytes())
 // 基于iptables格式化规则数据，使用iptables-restore刷新iptables规则
	err = proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	if err != nil {
		klog.Errorf("Failed to execute iptables-restore: %v\nRules:\n%s", err, proxier.iptablesData.Bytes())
		// Revert new local ports.
		utilproxy.RevertPorts(replacementPortsMap, proxier.portsMap)
		return
	}
```

>  ipvs-mode Proxier整个逻辑实现已分析完，其关键逻辑即syncProxyRules(){…}内代码，其中还有一些细节技术未展开叙述,如几个关键的依赖底层技术ipset的实现runner、ipvs路由(VS/RS)操作基于netlink机制通迅机制的实现等，因篇幅过长，后续再看具体情况补充。

**~本文 END~ **

