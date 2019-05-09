# Summary

## Part I - 准备工作
* [前言](README.md)
* [k8s源码分析准备工作](prepare/README.md)
    * [源码准备](prepare/get-code.md)
    * [测试环境搭建-单节点](prepare/debug-environment.md)
    * [测试环境搭建-三节点](prepare/debug-environment-3node.md)
    * [源码调试](prepare/debug.md)

## Part II - 核心组件
* [概述](core/README.md)
* [scheduler](core/scheduler/README.md)
    * [调度器设计](core/scheduler/design.md)
    * [调度程序启动前逻辑](core/scheduler/before-scheduler-run.md)
    * [调度器框架](core/scheduler/scheduler-framework.md)
    * [一般调度过程](core/scheduler/generic-scheduler.md)
    * [预选过程](core/scheduler/predicate.md)
    * [优选过程](core/scheduler/priority.md)
    * [抢占调度](core/scheduler/preempt.md)
    * [调度器初始化](core/scheduler/init.md)
    * [专题-亲和性调度](core/scheduler/affinity.md)
    * [scheduler 总结](core/scheduler/summarize.md)
* [controller-manager](core/controller-manager/README.md)
    * [控制器概述](core/controller-manager/controller.md)
    * [自定义控制器](core/controller-manager/custom-controller.md)
* [apiserver](core/apiserver/README.md)
* [kube-proxy](core/kube-proxy/README.md)
    * [Proxy 服务框架](core/kube-proxy/arch.md)
    * [IPtables-Mode Proxier](core/kube-proxy/iptables.md)
    * [Ipvs-Mode Proxier](core/kube-proxy/ipvs.md)
* [kubelet](core/kubelet/README.md)

## Part III - 周边项目
* [概述](around/README.md)
* [client-go](around/client-go/README.md)
    * [Informer (一)](around/client-go/informer.md)
    * [Informer(二)](around/client-go/informer2.md)

