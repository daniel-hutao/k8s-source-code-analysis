# 调度器设计

<!-- toc -->

## 概述

我们先整体了解一下Scheduler的设计原理，然后再看这些过程是如何用代码实现的。关于调度器的设计在官网有介绍，我下面结合官网给的说明，简化掉不影响理解的复杂部分，和大家介绍一下Scheduler的工作过程。

英文还可以的小伙伴们可以看一下官网的介绍先：[scheduler.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md)

官网有一段描述如下：

The Kubernetes scheduler runs as a process alongside the other master components such as the API server. Its interface to the API server is to watch for Pods with an empty PodSpec.NodeName, and for each Pod, it posts a binding indicating where the Pod should be scheduled.

简单翻译一下，也就是说Scheduler是一个跑在其他组件边上的独立程序，对接Apiserver寻找PodSpec.NodeName为空的Pod，然后用post的方式发送一个api调用，指定这些pod应该跑在哪个node上。

通俗地说，就是scheduler是相对独立的一个组件，主动访问api server，寻找等待调度的pod，然后通过一系列调度算法寻找哪个node适合跑这个pod，然后将这个pod和node的绑定关系发给api server，从而完成了调度的过程。

## 源码层级

从高level看，scheduler的源码可以分为3层：

- `cmd/kube-scheduler/scheduler.go`: main() 函数入口位置，在scheduler过程开始被调用前的一系列初始化工作。
- `pkg/scheduler/scheduler.go`: 调度框架的整体逻辑，在具体的调度算法之上的框架性的代码。
- `pkg/scheduler/core/generic_scheduler.go`: 具体的计算哪些node适合跑哪些pod的算法。

## 调度算法

调度过程整体如下图所示（官文里这个图没对齐，逼疯强迫症了！！！当然由于中文显示的问题，下图有中文的行也没法完全对齐，这个地方让我很抓狂。。。）：

```shell
对于一个给定的pod
+---------------------------------------------+
|             可用于调度的nodes如下：           |
|  +--------+     +--------+     +--------+   |
|  | node 1 |     | node 2 |     | node 3 |   |
|  +--------+     +--------+     +--------+   |
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
初步过滤: node 3 资源不足
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
|                 剩下的nodes:                 |
|     +--------+               +--------+     |
|     | node 1 |               | node 2 |     |
|     +--------+               +--------+     |
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
优先级算法计算结果:    node 1: 分数=2
                     node 2: 分数=5
+----------------------+----------------------+
                       |
                       v
            选择分值最高的节点 = node 2
```
Scheduler为每个pod寻找一个适合其运行的node，大体分成三步：

1. 通过一系列的“predicates”过滤掉不能运行pod的node，比如一个pod需要500M的内存，有些节点剩余内存只有100M了，就会被剔除；
2. 通过一系列的“priority functions”给剩下的node排一个等级，分出三六九等，寻找能够运行pod的若干node中最合适的一个node；
3. 得分最高的一个node，也就是被“priority functions”选中的node胜出了，获得了跑对应pod的资格。

## Predicates 和 priorities 策略

Predicates是一些用于过滤不合适node的策略 . Priorities是一些用于区分node排名（分数）的策略（作用在通过predicates过滤的node上）. K8s默认内建了一些predicates 和 priorities 策略，官方文档介绍地址： [scheduler_algorithm.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler_algorithm.md). Predicates 和 priorities 的代码分别在：

- pkg/scheduler/algorithm/predicates/predicates.go
- pkg/scheduler/algorithm/priorities.

## Scheduler 的拓展性

我们可以选择哪些预置策略生效，也可以添加自己的策略。几个月前我司有个奇葩调度需求，当时我就是通过增加一个priorities策略，然后重新编译了一个Scheduler来实现的需求。

## 调度策略的修改

默认调度策略是通过`defaultPredicates()` 和 `defaultPriorities()函数`定义的，源码在 `pkg/scheduler/algorithmprovider/defaults/defaults.go`，我们可以通过命令行flag `--policy-config-file`来覆盖默认行为。所以我们可以通过配置文件的方式或者修改`pkg/scheduler/algorithm/predicates/predicates.go` /`pkg/scheduler/algorithm/priorities`，然后注册到`defaultPredicates()`/`defaultPriorities()`来实现。配置文件类似下面这个样子：

```json
{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
	{"name" : "PodFitsHostPorts"},
	{"name" : "PodFitsResources"},
	{"name" : "NoDiskConflict"},
	{"name" : "NoVolumeZoneConflict"},
	{"name" : "MatchNodeSelector"},
	{"name" : "HostName"}
	],
"priorities" : [
	{"name" : "LeastRequestedPriority", "weight" : 1},
	{"name" : "BalancedResourceAllocation", "weight" : 1},
	{"name" : "ServiceSpreadingPriority", "weight" : 1},
	{"name" : "EqualPriority", "weight" : 1}
	],
"hardPodAffinitySymmetricWeight" : 10,
"alwaysCheckAllPredicates" : false
}
```

ok，看到这里大伙应该在流程上对Scheduler的原理有个感性的认识了，下一节我们就开始看一下Scheduler源码是怎么写的。