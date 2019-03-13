# 优选过程

<!-- toc -->

## 走近priority过程

!FILENAME pkg/scheduler/core/generic_scheduler.go:186

```
priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
```

今天的分析从这行代码开始。

`PrioritizeNodes`要做的事情是给已经通过predicate的nodes赋上一个分值，从而抉出一个最优node用于运行当前pod. 前面已经有分析predicate过程的经验了，所以priority过程看起来应该会轻松很多吧～（现实可能比较残酷，我第一次看完predicate后看priority是一脸蒙，和想象中的不太一样；大伙得耐下性子多思考，实在有障碍就先不求甚解，整体过完后再二刷代码，再不行三刷，总会大彻大悟的！）

我们先看看函数签名：

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

源码中的注释先理解一下：

> - PrioritizeNodes通过并发调用一个个priority函数来给node排优先级。每一个priority函数会给一个1-10之间的分值，0最低10最高。
> - 每一个priority函数可以有自己的权重，单个函数返回的分值*权重后得到一个加权分值，最终所有的加权分值加在一起就是这个node的最终分值。

行，大概知道优选过程要干嘛了，然后我们关注一下`PrioritizeNodes()`函数的形参定义和返回值：

- pod *v1.Pod* // pod就不用说了；
- *nodeNameToInfo map[string]*schedulercache.NodeInfo // 这个也不需要讲，字面意思代表一切；
- meta interface{} // 和predicate里的meta不太一样，先略过；
- priorityConfigs []algorithm.PriorityConfig // 包含优选算法各种信息，比较重要；
- nodes []*v1.Node // node集合，不需要解释了；
- extenders []algorithm.SchedulerExtender // extender逻辑放到后面单独讲。

返回值只需要看一下`schedulerapi.HostPriorityList`类型的含义了，这个类型之前也提过，后面频繁涉及到操作这个结构，所以这里再贴一次，大伙得烂熟于心才行！

!FILENAME pkg/scheduler/api/types.go:305

```go
type HostPriority struct {
	Host string
	Score int
}
type HostPriorityList []HostPriority
```

着重分析一下这2个type，虽然很简单，还是有必要啰嗦一下，必须记在心里。**HostPriority**这个struct的属性是*Host*和*Score*，一个是string一个是int，所以很明显**HostPriority**所能够保存的信息是一个node的名字和分值，再仔细一点说就是这个结构保存的是一个node在一个priority算法计算后所得到的结果；然后看**HostPriorityList**类型，这个类型是上一个类型的集合，集合表达的是一个node多个算法还是多个node一个算法呢？稍微思考一下可以知道**HostPriorityList**中存的是多个Host和Score的组合，也就是每个node的Score信息都有了。我们知道一个算法作用在所有node上就会得到每个node对应的一个Score，所以**HostPriorityList**这个结构是要保存一个算法作用于所有node之后，得到的所有node的Score信息的。（这里我们先理解成一个算法的结果，作为函数返回值这里肯定是要保留所有算法作用后的最终node的Score，所以函数后半部分肯定有整合分值的步骤。）

## PrioritizeNodes流程

前面说到`PrioritizeNodes()`函数也就是node优选的具体逻辑，这个函数略长，我们分段讲解。

### results

PrioritizeNodes()函数开头的逻辑很简单，我们先从第一行看到results定义的这一行。

!FILENAME pkg/scheduler/core/generic_scheduler.go:634

```go
if len(priorityConfigs) == 0 && len(extenders) == 0 {
    // 这个if很明显是处理特殊场景的，就是优选算法一个都没有配置的时候怎么做（extenders同样没有）；
    // 这个result是要当作返回值的，HostPriorityList类型前面唠叨了很多了，大家得心里有数；
   result := make(schedulerapi.HostPriorityList, 0, len(nodes))
   for i := range nodes {
       // 这一行代码是唯一的“逻辑了”，下面直到for结束都是简单代码；所以我们看一下EqualPriorityMap
       // 函数的作用就行了。这里我不贴代码，这个函数很短，作用就是设置每个node的Score相同（都为1）
       // hostPriority的类型也就是schedulerapi.HostPriority类型，再次强调这个类型是要烂熟于心的；
      hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
      if err != nil {
         return nil, err
      }
       // 最终的result也就是设置了每个node的Score为1的schedulerapi.HostPriorityList类型数据；
      result = append(result, hostPriority)
   }
   return result, nil
}
// 这里只是简单定义3个变量，一把锁，一个并发等待相关的wg，一个错误集合errs；
var (
   mu   = sync.Mutex{}
   wg   = sync.WaitGroup{}
   errs []error
)
// 这里定义了一个appendError小函数，逻辑很简单，并发场景下将错误信息收集到errs中；
appendError := func(err error) {
   mu.Lock()
   defer mu.Unlock()
   errs = append(errs, err)
}
// 最后一个变量results也不难理解，类型是[]schedulerapi.HostPriorityList，这里需要注意这个类型
// 的作用，它保存的是所有算法作用所有node之后得到的结果集，相当于一个二维数组，每个格子是1个算法
// 作用于1个节点的结果，一行也就是1个算法作用于所有节点的结果；一行展成1个二维就是所有算法作用于所有节点；
results := make([]schedulerapi.HostPriorityList, len(priorityConfigs), len(priorityConfigs))
```

到这里要求大家心中能够想象上面提到的results是什么样的，不好想象可以借助纸笔。下面的代码会往这个二维结构里面存储数据。

### 老式priority函数

我们既然讲到“老式”，后面肯定有对应的“新式”。虽然这种函数已经DEPRECATED了，不过对于我们学习掌握优选流程还是很有帮助的。贴这块代码之前我们先关注一下多次出现的`priorityConfigs`这个变量的类型：

函数形参中有写到：`priorityConfigs []algorithm.PriorityConfig`，所以我们直接看**PriorityConfig**是什么类型：

!FILENAME pkg/scheduler/algorithm/types.go:62

```go
// PriorityConfig is a config used for a priority function.
type PriorityConfig struct {
   Name   string
   Map    PriorityMapFunction
   Reduce PriorityReduceFunction
   // TODO: Remove it after migrating all functions to
   // Map-Reduce pattern.
   Function PriorityFunction
   Weight   int
}
```

**PriorityConfig**中有一个Name，一个Weight，很好猜到意思。剩下的Map、Reduce和Function目测代表的就是优选函数的新旧两种表达方式了。我们先看旧的Function属性的类型PriorityFunction是什么：

!FILENAME pkg/scheduler/algorithm/types.go:59

```go
type PriorityFunction func(pod *v1.Pod, nodeNameToInfo map[string]*schedulercache.NodeInfo, nodes []*v1.Node) (schedulerapi.HostPriorityList, error)
```

很明显这个类型代表了一个priority函数，入参是pod、nodeNameToInfo和nodes，返回值是HostPriorityList，也就是我们前面提到的1个priority函数作用于每个node后得到了Score信息，存结果的结构就是这个HostPriorityList；

然后讲回**PrioritizeNodes**过程：

!FILENAME pkg/scheduler/core/generic_scheduler.go:661

```go
// DEPRECATED: we can remove this when all priorityConfigs implement the
// Map-Reduce pattern.
for i := range priorityConfigs {
    // 如果第i个优选配置定义了老函数，则调用之；
	if priorityConfigs[i].Function != nil {
		wg.Add(1)
        // 注意这里的参数index，这里传入的实参是上面的i；
		go func(index int) {
			defer wg.Done()
			var err error
            // 所以这里的results[index]就好理解了；后面priorityConfigs[index]的索引也是index，
            // 这里表达的是第N个优选配置里有Function，那么这个Function的计算结果保存在
            // results的第N个格子里；
			results[index], err = priorityConfigs[index].Function(pod, nodeNameToInfo, nodes)
			if err != nil {
				appendError(err)
			}
		}(i)
	} else {
        // 如果没有定义Function，其实也就是使用了Map-Reduce方式的，这里先存个空的结构占位；
		results[i] = make(schedulerapi.HostPriorityList, len(nodes))
	}
}
```

上面这段代码逻辑还算好理解，唯一有点绕的还是前面强调的HostPriorityList相关类型的操作上。

### Map-Reduce

关于map-reduce思想我就不在这里赘述了，数据处理很流行的一种思想，百度一下（如果你能够google，自然更好咯）一大堆介绍的文章。

简单说map-reduce就是：Map是映射，Reduce是规约；map是统计一本书中的一页出现了多少次k8s这个词，reduce是将这些词加在一起得到最终结果。（map一般都是将一个算法作用于一堆数据集的每一个元素，得到一个结果集，reduce有各种形式，我没有深入看map-reduce的思想，也没有玩过大数据领域的Hadoop和MapReduce，这里可能理解有出入）

#### Map-reduce方式的优选过程

看看在Scheduler里面是怎么用Map-Reduce的吧：

```go
// 这个并发逻辑之前介绍过了，我们直接看ParallelizeUntil的最后一个参数就行，这里直接写了一个匿名函数；
workqueue.ParallelizeUntil(context.TODO(), 16, len(nodes), func(index int) {
    // 这里的index是[0，len(nodes)-1]，相当于遍历所有的nodes；
   nodeInfo := nodeNameToInfo[nodes[index].Name]
    // 这个for循环遍历的是所有的优选配置，如果有老Fun就跳过，新逻辑就继续；
   for i := range priorityConfigs {
      if priorityConfigs[i].Function != nil {
         continue
      }

      var err error
       // 这里的i和前面老Fun的互补，老Fun中没有赋值的results中的元素就在这里赋值了；
       // 注意到这里调用了一个Map函数就直接赋值给了results[i][index]，这里的index通过
       // ParallelizeUntil这个并发实现所有node对于一个优选算法的分值计算；
      results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
      if err != nil {
         appendError(err)
         results[i][index].Host = nodes[index].Name
      }
   }
})

for i := range priorityConfigs {
    // 没有定义Reduce函数就不处理；
   if priorityConfigs[i].Reduce == nil {
      continue
   }
   wg.Add(1)
   go func(index int) {
      defer wg.Done()
       // 调用Reduce函数
      if err := priorityConfigs[index].Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
         appendError(err)
      }
      if klog.V(10) {
         for _, hostPriority := range results[index] {
            klog.Infof("%v -> %v: %v, Score: (%d)", util.GetPodFullName(pod), hostPriority.Host, priorityConfigs[index].Name, hostPriority.Score)
         }
      }
   }(i)
}
// Wait for all computations to be finished.
wg.Wait()
if len(errs) != 0 {
   return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
}
```

看到这里我们可以发现老Fun和Map的区别不大，都是优选函数的执行过程。这里可以注意到有一个区别就是老Fun执行结果是给了`results[index]`，这个类型是`schedulerapi.HostPriorityList`；而Map的执行结果是给了`results[i][index]`，这个类型是`HostPriority`类型，开头的时候多次强调过这2个类型。但是Map本身在并发执行的时候，遍历的是nodes，所以Map的结果其实给了results[i]，也就是填充了`schedulerapi.HostPriorityList`给results. 

所以不管是老的Fun还是新的Map，做的事情是一样的，都是把1个算法作用于所有的nodes，从而得到1个result，也就是1个**schedulerapi.HostPriorityList**，无非是Map函数是一个一个node计算，Fun是一下子接受了nodes参数拿去计算。

开始时可能大家会以为Fun直接计算出了一个优选函数作用于所有node的结果，Map是一个优选函数作用于一个node的结果，然后使用reduce汇总，然而并不是这样。那Fun和Map-Reduce到底还有啥区别呢？继续看吧。

我们关注一下Reduce函数的一个参数是`results[index]`，这个index表达的是第index个优选算法，没有啥实质返回值，所以Reduce并不将什么数据整合了，最终results还是[]HostPriorityList类型，也就是\[\]\[\]HostPriority，存了每个算法作用于每个node的分值。

看来只能看几个Reduce函数的实例才能理解Reduce在这个场景的作用了～

#### Map-Reduce形式的priority函数

**CalculateAntiAffinityPriority**计算反亲的目的是让“同一个service下的pod尽量分散在给定的nodes上”。下面我们来看这个策略的Map和Reduce过程分别是怎么写的：

**map函数**

!FILENAME pkg/scheduler/algorithm/priorities/selector_spreading.go:221

```go
func (s *ServiceAntiAffinity) CalculateAntiAffinityPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
	var firstServiceSelector labels.Selector

	node := nodeInfo.Node()
	if node == nil {
		return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
	}
	priorityMeta, ok := meta.(*priorityMetadata)
	if ok {
		firstServiceSelector = priorityMeta.podFirstServiceSelector
	} else {
		firstServiceSelector = getFirstServiceSelector(pod, s.serviceLister)
	}
    // 查找给定node在给定namespace下符合selector的pod，返回值是[]*v1.Pod
	matchedPodsOfNode := filteredPod(pod.Namespace, firstServiceSelector, nodeInfo)

	return schedulerapi.HostPriority{
		Host:  node.Name,
        // 返回值中Score设置成上面找到的pod的数量
		Score: int(len(matchedPodsOfNode)),
	}, nil
}
```

这个函数比较短，可以看到在指定node上查询到匹配selector的pod越多，分值就越高。假设找到了20个，那么这里的分值就是20；假设找到的是2，那这里的分值就是2.

**reduce函数**

!FILENAME pkg/scheduler/algorithm/priorities/selector_spreading.go:245

```go
func (s *ServiceAntiAffinity) CalculateAntiAffinityPriorityReduce(pod *v1.Pod, meta interface{}, nodeNameToInfo map[string]*schedulercache.NodeInfo, result schedulerapi.HostPriorityList) error {
   var numServicePods int
   var label string
   podCounts := map[string]int{}
   labelNodesStatus := map[string]string{}
   maxPriorityFloat64 := float64(schedulerapi.MaxPriority)

   for _, hostPriority := range result {
       // Score为n就是map函数中写入的匹配pod的数量为n;
       // 累加也就得到了所有node的匹配pod数量之和；
      numServicePods += hostPriority.Score
       // 如果nodes上的labels中不包含当前服务指定的反亲和label，则continue；
      if !labels.Set(nodeNameToInfo[hostPriority.Host].Node().Labels).Has(s.label) {
         continue
      }
       // 这里也就是没有continue的逻辑，说明在当前node上找到了反亲和的label，通过Get获取label值；
      label = labels.Set(nodeNameToInfo[hostPriority.Host].Node().Labels).Get(s.label)
       // 存一个host name和label值的map；
      labelNodesStatus[hostPriority.Host] = label
       // label和pod数的map；
      podCounts[label] += hostPriority.Score
   }

   // 遍历result也就是遍历每个node
   for i, hostPriority := range result {
       // 如果这个node上有设置的label
      label, ok := labelNodesStatus[hostPriority.Host]
      if !ok {
         result[i].Host = hostPriority.Host
         result[i].Score = int(0)
         continue
      }
      // 设置fScore为默认最大值10.0
      fScore := maxPriorityFloat64
      if numServicePods > 0 {
         fScore = maxPriorityFloat64 * (float64(numServicePods-podCounts[label]) / float64(numServicePods))
      }
      result[i].Host = hostPriority.Host
      result[i].Score = int(fScore)
   }

   return nil
}
```