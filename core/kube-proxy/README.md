# kube-proxy

<u>**本章 owner：XiaoYang**</u>

在kubernetes中几个基础概念如pod、endpoints、 service。提供相同服务的一组pod可以抽象成一个service,通过service提供的统一入口对外提供服务。kube-proxy组件被安装在kubernetes集群的各个node节点上，实现了kubernetes service机制(实现集群内客户端pod访问service,或者外部通过NodePort、XLB或ingress等方式访问)。kube-proxy提供了三种服务负载模式:

- 基于用户态的userspace模式
- iptables模式
- ipvs模式

kube-proxy源码框架风格继承了kubernetes组件的一惯风格，而且kube-proxy源码更为简洁，基本上分为三层。第一层为标准的CLI应用App创建层，使用kubernetes通用的Cobra框架来构建App和初始化配置。第二层为主应用进程proxy服务器相关对象的创建与运行层。第三层为kubernetes的"service层"的实现机制层(proxyMode选择决定)。

kube-proxy源码分析以下几部分内容进行展开：

- kube-proxy服务框架分析，CLI应用创建与初始化以及proxyserver创建和Run主框架代码；
- kube-proxy框架整体逻辑分析，具体的proxy逻辑代码；
- kube-proxy三种模式源码分析，详细分析userspace/iptables/ipvs三种模式的实现(每种模式单文分析)；
- kube-proxy其它，如关键数据结构、类关系图等。

## 本章规划

1. [Proxy 服务框架](./arch.md)
2. [IPtables-Mode Proxier](./iptables.md)
3. [Ipvs-Mode Proxier](./ipvs.md)

