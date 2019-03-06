# Summary

## Part I - 准备工作
* [前言](README.md)
* [k8s源码分析准备工作](prepare/README.md)
    * [源码准备](prepare/get-code.md)
    * [测试环境搭建](prepare/debug-environment.md)

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
* [apiserver](core/apiserver/README.md)
* [proxy](core/proxy/README.md)
* [kubelet](core/kubelet/README.md)
* [controller-manager](core/controller-manager/README.md)

## Part III - 周边项目
* [概述](around/README.md)
* [client-go](around/client-go/README.md)

