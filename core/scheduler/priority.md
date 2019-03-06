# 优选过程

<!-- toc -->

## 进入priority逻辑

!FILENAME pkg/scheduler/core/generic_scheduler.go:186

```
priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
```

今天的分析从这行代码开始。

`PrioritizeNodes`要做的事情是给已经通过predicate的nodes赋上一个分值，从而抉出一个最优node用于运行当前pod. 前面已经有分析predicate过程的经验了，所以priority过程看起来应该会轻松很多。我们先看函数签名：

!FILENAME pkg/scheduler/core/generic_scheduler.go:624

```GO
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	meta interface{},
	priorityConfigs []algorithm.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
) (schedulerapi.HostPriorityList, error) 
```

源码中的注释先理解一下：

> - PrioritizeNodes通过并发调用一个个priority函数来给node排优先级。每一个priority函数会给一个1-10之间的分值，0最低10最高。
> - 每一个priority函数可以有自己的权重，单个函数返回的分值*权重后得到一个加权分值，最终所有的加权分值加在一起就是这个node的最终分值。

行，大概知道优选过程要干嘛了，然后我们关注一下`PrioritizeNodes()`函数的形参定义和返回值：

- pod *v1.Pod* // pod就不用说了；
- *nodeNameToInfo map[string]*schedulercache.NodeInfo // 这个也不需要讲，字面意思代表一切；
- meta interface{} // 和predicate里的meta不太一样，略复杂，暂不深究；
- priorityConfigs []algorithm.PriorityConfig // 包含优选算法各种信息；
- nodes []*v1.Node // node集合，不需要解释了；
- extenders []algorithm.SchedulerExtender // extender逻辑放到后面单独讲。

返回值只需要看一下`schedulerapi.HostPriorityList`类型的含义了，这个类型之前也提过，再贴一次：

!FILENAME pkg/scheduler/api/types.go:305

```go
type HostPriority struct {
	Host string
	Score int
}
type HostPriorityList []HostPriority
```

到这里基本知道了函数的入参与返回值，也就能猜到这个函数做了什么了。但是细想可能有点抓狂，`PrioritizeNodes`有点长，里面到底什么逻辑？什么逻辑？逻辑？辑？

带着纠结继续往下看吧～