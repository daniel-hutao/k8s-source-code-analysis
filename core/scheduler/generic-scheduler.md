# 一般调度过程

<!-- toc -->

## 进入Scheduler的第三层逻辑

今天分析的代码，就已经算kube-scheduler的第三层逻辑了，我们要找到预选和优选的入口，讲完太长，干脆后面单独分2节讲预选和优选过程。所以本小节会比较简短哦～

今天我们从`pkg/scheduler/core/generic_scheduler.go:139`开始，也就是从这个**generic scheduler的Schedule()方法**下手！

我们依旧关心主干先，这个方法主要涉及的是预选过程+优选过程，看下主要代码：

!FILENAME pkg/scheduler/core/generic_scheduler.go:139

```go
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	nodes, err := nodeLister.List()
	trace.Step("Computing predicates")
	filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
	trace.Step("Prioritizing")
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```

如上，我手一抖就删的只剩下这几行了，大伙应该从这不到十行的代码里找到3个步骤：

1. "Computing predicates"：调用findNodesThatFit()方法；
2. "Prioritizing"：调用PrioritizeNodes()方法；
3. "Selecting host"：调用selectHost()方法。

接着当然是先浏览一下这3步分别完成了哪些工作咯～

### Computing predicates

这个过程的入口是：

```go
filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
```

从变量命名上其实就可以猜到一大半，filteredNodes肯定就是过滤出来的nodes，也就是经受住了预选算法考验的node集合，我们从`findNodesThatFit`方法的函数签名中可以得到准确一些的信息：

!FILENAME pkg/scheduler/core/generic_scheduler.go:389

```go
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error)
```

入参是1个pod + 一堆node，返回值是一堆node（这个堆堆当然<=入参的nodes），很明显，predicates就是干这个事情了！

### Prioritizing

Prioritizing的入口看着复杂一点：

```go
priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
```

注意到这里的返回值叫做priorityList，什么什么List也就是不止一个了，优选过程不是选出1个最佳节点吗？我们继续看：

!FILENAME pkg/scheduler/core/generic_scheduler.go:624

```go
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	meta interface{},
	priorityConfigs []algorithm.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
) (schedulerapi.HostPriorityList, error)
```

首选关注返回值是什么意思：

!FILENAME pkg/scheduler/api/types.go:305

```go
type HostPriority struct {
	// Name of the host
	Host string
	// Score associated with the host
	Score int
}
// HostPriorityList declares a []HostPriority type.
type HostPriorityList []HostPriority
```

看到这里就清晰了，原来有个`HostPriority`类型记录一个Host的名字和分值，`HostPriorityList`类型也就是`HostPriority`类型的集合，意味着记录了多个Host的名字和分值，于是我们可以判断`PrioritizeNodes()`方法的作用是计算前面的predicates过程筛选出来的nodes各自的Score.所以肯定还有一个根据Score决定哪个node胜出的逻辑咯～，继续往下看吧～

### Selecting host

这个过程比较明显了，我们直接看代码：

!FILENAME pkg/scheduler/core/generic_scheduler.go:227

```go
func (g *genericScheduler) selectHost(priorityList schedulerapi.HostPriorityList) (string, error)
```

这个selectHost()方法大家应该都已经猜到了，就是从上一步的优选过程的结果集中选出一个Score最高的Host，并且返回这个Host的name.

genericScheduler的Schedule()方法主要就是这3个过程，下一讲我们开始分析predicates过程。
