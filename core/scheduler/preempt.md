# 抢占调度

<!-- toc -->

## Pod priority

Pod 有了 priority(优先级) 后才有优先级调度、抢占调度的说法，高优先级的 pod 可以在调度队列中排到前面，优先选择 node；另外当高优先级的 pod 找不到合适的 node 时，就会看 node 上低优先级的 pod 驱逐之后是否能够 run 起来，如果可以，那么 node 上的一个或多个低优先级的 pod 会被驱逐，然后高优先级的 pod 得以成功运行1个 node 上。

今天我们分析 pod 抢占相关的代码。开始之前我们看一下和 priority 相关的2个示例配置文件：

**PriorityClass 例子**

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

**使用上述 PriorityClass**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

这两个文件的内容这里不解释，Pod priority 相关知识点不熟悉的小伙伴请先查阅[官方文档](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)，我们下面看调度器中和 preempt 相关的代码逻辑。

## preempt 入口

在`pkg/scheduler/scheduler.go:513 scheduleOne()`方法中我们上一次关注的是`suggestedHost, err := sched.schedule(pod)`这行代码，也就是关注通常情况下调度器如何给一个 pod 匹配一个最合适的 node. 今天我们来看如果这一行代码返回的 `err != nil` 情况下，如何开始 preempt 过程。

!FILENAME pkg/scheduler/scheduler.go:529

```go
suggestedHost, err := sched.schedule(pod)
if err != nil {
   if fitError, ok := err.(*core.FitError); ok {
      preemptionStartTime := time.Now()
      sched.preempt(pod, fitError)
      metrics.PreemptionAttempts.Inc()
   } else {
      klog.Errorf("error selecting node for pod: %v", err)
      metrics.PodScheduleErrors.Inc()
   }
   return
}
```

当`schedule()`函数没有返回 host，也就是没有找到合适的 node 的时候，就会出发 preempt 过程。这时候代码逻辑进入`sched.preempt(pod, fitError)`这一行。我们先看一下这个函数的整体逻辑，然后深入其中涉及的子过程：

!FILENAME pkg/scheduler/scheduler.go:311

```go
func (sched *Scheduler) preempt(preemptor *v1.Pod, scheduleErr error) (string, error) {
    // 特性没有开启就返回 ""
	if !util.PodPriorityEnabled() || sched.config.DisablePreemption {
		return "", nil
	}
    // 更新 pod 信息；入参和返回值都是 *v1.Pod 类型
	preemptor, err := sched.config.PodPreemptor.GetUpdatedPod(preemptor)

    // preempt 过程，下文分析
	node, victims, nominatedPodsToClear, err := sched.config.Algorithm.Preempt(preemptor, sched.config.NodeLister, scheduleErr)
	
	var nodeName = ""
	if node != nil {
		nodeName = node.Name
		// 更新队列中“任命pod”队列
		sched.config.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

		// 设置pod的Status.NominatedNodeName
		err = sched.config.PodPreemptor.SetNominatedNodeName(preemptor, nodeName)
		if err != nil {
			// 如果出错就从 queue 中移除
			sched.config.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
			return "", err
		}

		for _, victim := range victims {
            // 将要驱逐的 pod 驱逐
			if err := sched.config.PodPreemptor.DeletePod(victim); err != nil {
				return "", err
			}
			sched.config.Recorder.Eventf(victim, v1.EventTypeNormal, "Preempted", "by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)
		}
	}
	// Clearing nominated pods should happen outside of "if node != nil". 
    // 这个清理过程在上面的if外部，我们回头从 Preempt() 的实现去理解
	for _, p := range nominatedPodsToClear {
		rErr := sched.config.PodPreemptor.RemoveNominatedNodeName(p)
		if rErr != nil {
			klog.Errorf("Cannot remove nominated node annotation of pod: %v", rErr)
			// We do not return as this error is not critical.
		}
	}
	return nodeName, err
}
```

## preempt 实现

上面 `preempt()` 函数中涉及到了一些值得深入看看的对象，下面我们逐个看一下这些对象的实现。

### SchedulingQueue

SchedulingQueue 表示的是一个存储待调度 pod 的队列

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:60

```go
type SchedulingQueue interface {
   Add(pod *v1.Pod) error
   AddIfNotPresent(pod *v1.Pod) error
   AddUnschedulableIfNotPresent(pod *v1.Pod) error
   Pop() (*v1.Pod, error)
   Update(oldPod, newPod *v1.Pod) error
   Delete(pod *v1.Pod) error
   MoveAllToActiveQueue()
   AssignedPodAdded(pod *v1.Pod)
   AssignedPodUpdated(pod *v1.Pod)
   NominatedPodsForNode(nodeName string) []*v1.Pod
   WaitingPods() []*v1.Pod
   Close()
   UpdateNominatedPodForNode(pod *v1.Pod, nodeName string)
   DeleteNominatedPodIfExists(pod *v1.Pod)
   NumUnschedulablePods() int
}
```

在 Scheduler 中 SchedulingQueue 接口对应两种实现：

- FIFO 先进先出队列
- PriorityQueue 优先级队列

#### FIFO

FIFO 结构是对 cache.FIFO 的简单包装，然后实现了 SchedulingQueue 接口。

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:97

```go
type FIFO struct {
   *cache.FIFO
}
```

cache.FIFO定义在`vendor/k8s.io/client-go/tools/cache/fifo.go:93`，这个先进先出队列的细节先不讨论。

#### PriorityQueue

PriorityQueue 同样实现了 SchedulingQueue 接口，PriorityQueue 的顶是最高优先级的 pending pod. 这里的PriorityQueue 有2个子 queue，activeQ 放的是等待调度的 pod，unschedulableQ 放的是已经尝试过调度，然后失败了，被标记为 unschedulable 的 pod.

我们看一下 PriorityQueue 结构的定义：

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:201

```go
type PriorityQueue struct {
   stop  <-chan struct{}
   clock util.Clock
   lock  sync.RWMutex
   cond  sync.Cond

   // heap 头节点存的是最高优先级的 pod
   activeQ *Heap
   // unschedulableQ holds pods that have been tried and determined unschedulable.
   unschedulableQ *UnschedulablePodsMap
   // 存储已经被指定好要跑在某个 node 的 pod
   nominatedPods *nominatedPodMap
   // 只要将 pod 从 unschedulableQ 移动到 activeQ，就设置为true；从 activeQ 中 pop 出来 pod的时候设置为 false. 这个字段表明一个 pod 在被调度的过程中是否接收到了队列 move 操作，如果发生了 move 操作，那么这个 pod 就算被认定为 unschedulable，也被放回到 activeQ.
   receivedMoveRequest bool
   closed bool
}
```

PriorityQueue 的方法比较好理解，我们看几个吧：

**1、`func (p *PriorityQueue) Add(pod *v1.Pod) error`** //在 active queue 中添加1个pod

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:276

```go
func (p *PriorityQueue) Add(pod *v1.Pod) error {
   p.lock.Lock()
   defer p.lock.Unlock()
    // 直接在 activeQ 中添加 pod
   err := p.activeQ.Add(pod)
   if err != nil {
      klog.Errorf("Error adding pod %v/%v to the scheduling queue: %v", pod.Namespace, pod.Name, err)
   } else {
       // 如果在 unschedulableQ 中找到这个 pod，抛错误日志后移除队列中该 pod
      if p.unschedulableQ.get(pod) != nil {
         klog.Errorf("Error: pod %v/%v is already in the unschedulable queue.", pod.Namespace, pod.Name)
         p.unschedulableQ.delete(pod)
      }
       // 队列的 nominatedPods 属性中标记该 pod 不指定到任何 node
      p.nominatedPods.add(pod, "")
      p.cond.Broadcast()
   }
   return err
}
```

**2、`func (p *PriorityQueue) AddIfNotPresent(pod *v1.Pod) error`**//如果2个队列中都不存在该 pod，那么就添加到 active queue 中

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:295

```go
func (p *PriorityQueue) AddIfNotPresent(pod *v1.Pod) error {
   p.lock.Lock()
   defer p.lock.Unlock()
    //如果队列 unschedulableQ 中有 pod，啥也不做
   if p.unschedulableQ.get(pod) != nil {
      return nil
   }
    //如果队列 activeQ 中有 pod，啥也不做
   if _, exists, _ := p.activeQ.Get(pod); exists {
      return nil
   }
    // 添加 pod 到 activeQ
   err := p.activeQ.Add(pod)
   if err != nil {
      klog.Errorf("Error adding pod %v/%v to the scheduling queue: %v", pod.Namespace, pod.Name, err)
   } else {
      p.nominatedPods.add(pod, "")
      p.cond.Broadcast()
   }
   return err
}
```

**3、`func (p *PriorityQueue) flushUnschedulableQLeftover()`**//刷新 unschedulableQ 中的 pod，如果一个 pod 的呆的时间超过了 durationStayUnschedulableQ，就移动到 activeQ 中

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:346

```go
func (p *PriorityQueue) flushUnschedulableQLeftover() {
   p.lock.Lock()
   defer p.lock.Unlock()

   var podsToMove []*v1.Pod
   currentTime := p.clock.Now()
    // 遍历 unschedulableQ 中的 pod
   for _, pod := range p.unschedulableQ.pods {
      lastScheduleTime := podTimestamp(pod)
       // 这里的默认值是 60s，所以超过 60s 的 pod 将得到进入 activeQ 的机会
      if !lastScheduleTime.IsZero() && currentTime.Sub(lastScheduleTime.Time) > unschedulableQTimeInterval {
         podsToMove = append(podsToMove, pod)
      }
   }

   if len(podsToMove) > 0 {
       // 全部移到 activeQ 中，又有机会被调度了
      p.movePodsToActiveQueue(podsToMove)
   }
}
```

**4、`func (p *PriorityQueue) Pop() (*v1.Pod, error)`**//从 activeQ 中 pop 一个 pod

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:367

```go
func (p *PriorityQueue) Pop() (*v1.Pod, error) {
   p.lock.Lock()
   defer p.lock.Unlock()
   for len(p.activeQ.data.queue) == 0 {
      // 当队列为空的时候会阻塞
      if p.closed {
         return nil, fmt.Errorf(queueClosed)
      }
      p.cond.Wait()
   }
   obj, err := p.activeQ.Pop()
   if err != nil {
      return nil, err
   }
   pod := obj.(*v1.Pod)
    // 标记 receivedMoveRequest 为 false，表示新的一次调度开始了
   p.receivedMoveRequest = false
   return pod, err
}
```

再看个别 `PriorityQueue.nominatedPods` 属性相关操作的方法，也就是 `preempt()` 函数中多次调用到的方法：

**5、`func (p *PriorityQueue) UpdateNominatedPodForNode(pod *v1.Pod, nodeName string)**`//pod 抢占的时候，确定一个 node 可以用于跑这个 pod 时，通过调用这个方法将 pod nominated 到 指定的 node 上。

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:567

```go
func (p *PriorityQueue) UpdateNominatedPodForNode(pod *v1.Pod, nodeName string) {
   p.lock.Lock()
    //逻辑在这里面
   p.nominatedPods.add(pod, nodeName) 
   p.lock.Unlock()
}
```

先看 nominatedPods 属性的类型，这个类型用于存储 pods 被 nominate 到 nodes 的信息：

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:822

```go
type nominatedPodMap struct {
   // key 是 node name，value 是 nominated 到这个 node 上的 pods
   nominatedPods map[string][]*v1.Pod
   // 和上面结构相反，key 是 pod 信息，值是 node 信息
   nominatedPodToNode map[ktypes.UID]string
}
```

在看一下`add()`方法的实现：

!FILENAME pkg/scheduler/internal/queue/scheduling_queue.go:832

```go
func (npm *nominatedPodMap) add(p *v1.Pod, nodeName string) {
   // 不管有没有，先删一下，防止重了
   npm.delete(p)

   nnn := nodeName
    // 如果传入的 nodeName 是 “”
   if len(nnn) == 0 {
       // 查询 pod 的 pod.Status.NominatedNodeName
      nnn = NominatedNodeName(p)
       // 如果 pod.Status.NominatedNodeName 也是 “”,return
      if len(nnn) == 0 {
         return
      }
   }
    // 逻辑到这里说明要么 nodeName 不为空字符串，要么 nodeName 为空字符串但是 pod 的 pod.Status.NominatedNodeName 不为空字符串，这时候开始下面的赋值
   npm.nominatedPodToNode[p.UID] = nnn
   for _, np := range npm.nominatedPods[nnn] {
      if np.UID == p.UID {
         klog.V(4).Infof("Pod %v/%v already exists in the nominated map!", p.Namespace, p.Name)
         return
      }
   }
   npm.nominatedPods[nnn] = append(npm.nominatedPods[nnn], p)
}
```

### PodPreemptor

**PodPreemptor** 用来驱逐 pods 和更新 pod annotations.

!FILENAME pkg/scheduler/factory/factory.go:145

```go
type PodPreemptor interface {
   GetUpdatedPod(pod *v1.Pod) (*v1.Pod, error)
   DeletePod(pod *v1.Pod) error
   SetNominatedNodeName(pod *v1.Pod, nominatedNode string) error
   RemoveNominatedNodeName(pod *v1.Pod) error
}
```

这个 interface 对应的实现类型是：

!FILENAME pkg/scheduler/factory/factory.go:1620

```go
type podPreemptor struct {
   Client clientset.Interface
}
```

这个类型绑定了4个方法：

!FILENAME pkg/scheduler/factory/factory.go:1624

```go
// 新获取一次 pod 的信息
func (p *podPreemptor) GetUpdatedPod(pod *v1.Pod) (*v1.Pod, error) {
   return p.Client.CoreV1().Pods(pod.Namespace).Get(pod.Name, metav1.GetOptions{})
}

// 删除一个 pod
func (p *podPreemptor) DeletePod(pod *v1.Pod) error {
   return p.Client.CoreV1().Pods(pod.Namespace).Delete(pod.Name, &metav1.DeleteOptions{})
}

// 设置pod.Status.NominatedNodeName 为指定的 node name
func (p *podPreemptor) SetNominatedNodeName(pod *v1.Pod, nominatedNodeName string) error {
   podCopy := pod.DeepCopy()
   podCopy.Status.NominatedNodeName = nominatedNodeName
   _, err := p.Client.CoreV1().Pods(pod.Namespace).UpdateStatus(podCopy)
   return err
}

// 清空 pod.Status.NominatedNodeName
func (p *podPreemptor) RemoveNominatedNodeName(pod *v1.Pod) error {
   if len(pod.Status.NominatedNodeName) == 0 {
      return nil
   }
   return p.SetNominatedNodeName(pod, "")
}
```

### xx.Algorithm.Preempt

#### 接口定义

我们回到挺久之前讲常规调度过程的时候提过的一个接口：

!FILENAME pkg/scheduler/algorithm/scheduler_interface.go:78

```go
type ScheduleAlgorithm interface {
   Schedule(*v1.Pod, NodeLister) (selectedMachine string, err error)
   // Preempt 在 pod 调度发生失败的时候尝试抢占低优先级的 pod.
   // 返回发生 preemption 的 node, 被 preempt的 pods 列表, 
   // nominated node name 需要被移除的 pods 列表，一个 error 信息.
   Preempt(*v1.Pod, NodeLister, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
  
   Predicates() map[string]FitPredicate
   Prioritizers() []PriorityConfig
}
```

这个接口上次我们讲到的时候关注了`Schedule()`、`Predicates()`和`Prioritizers()`，这次来看`Preempt()`是怎么实现的。

#### 整体流程

`Preempt()`同样由`genericScheduler`类型(`pkg/scheduler/core/generic_scheduler.go:98`)实现，方法前的一大串英文注释先来理解一下：

- Preempt 寻找一个在发生抢占之后能够成功调度“pod”的node.
- Preempt 选择一个 node 然后抢占上面的 pods 资源，返回：
  - 这个 node 信息
  - 被抢占的 pods 信息
  - nominated node name 需要被清理的 node 列表
  - 可能有的 error
- Preempt 过程不涉及快照更新（快照的逻辑以后再讲）
- 避免出现这种情况：preempt 发现一个不需要驱逐任何 pods 就能够跑“pod”的 node.
- 当有很多 pending pods 在调度队列中的时候，a nominated pod 会排到队列中相同优先级的 pod 后面. 
- The nominated pod 会阻止其他 pods 使用“指定”的资源，哪怕花费了很多时间来等待其他 pending 的 pod.

我们先过整体流程，然后逐个分析子流程调用：

!FILENAME pkg/scheduler/core/generic_scheduler.go:251

```go
func (g *genericScheduler) Preempt(pod *v1.Pod, nodeLister algorithm.NodeLister, scheduleErr error) (*v1.Node, []*v1.Pod, []*v1.Pod, error) {
   // 省略几行
   // 判断执行驱逐操作是否合适
   if !podEligibleToPreemptOthers(pod, g.cachedNodeInfoMap) {
      klog.V(5).Infof("Pod %v/%v is not eligible for more preemption.", pod.Namespace, pod.Name)
      return nil, nil, nil, nil
   }
    // 所有的 nodes
   allNodes, err := nodeLister.List()
   if err != nil {
      return nil, nil, nil, err
   }
   if len(allNodes) == 0 {
      return nil, nil, nil, ErrNoNodesAvailable
   }
    // 计算潜在的执行驱逐后能够用于跑 pod 的 nodes
   potentialNodes := nodesWherePreemptionMightHelp(allNodes, fitError.FailedPredicates)
   if len(potentialNodes) == 0 {
      klog.V(3).Infof("Preemption will not help schedule pod %v/%v on any node.", pod.Namespace, pod.Name)
      // In this case, we should clean-up any existing nominated node name of the pod.
      return nil, nil, []*v1.Pod{pod}, nil
   }
    // 列出 pdb 对象
   pdbs, err := g.pdbLister.List(labels.Everything())
   if err != nil {
      return nil, nil, nil, err
   }
    // 计算所有 node 需要驱逐的 pods 有哪些等，后面细讲
   nodeToVictims, err := selectNodesForPreemption(pod, g.cachedNodeInfoMap, potentialNodes, g.predicates,
      g.predicateMetaProducer, g.schedulingQueue, pdbs)
   if err != nil {
      return nil, nil, nil, err
   }

   // 拓展调度的逻辑
   nodeToVictims, err = g.processPreemptionWithExtenders(pod, nodeToVictims)
   if err != nil {
      return nil, nil, nil, err
   }

    // 选择1个 node 用于 schedule
   candidateNode := pickOneNodeForPreemption(nodeToVictims)
   if candidateNode == nil {
      return nil, nil, nil, err
   }

    // 低优先级的被 nominate 到这个 node 的 pod 很可能已经不再 fit 这个 node 了，所以
    // 需要移除这些 pod 的 nomination，更新这些 pod，挪动到 activeQ 中，让调度器
    // 得以寻找另外一个 node 给这些 pod
   nominatedPods := g.getLowerPriorityNominatedPods(pod, candidateNode.Name)
   if nodeInfo, ok := g.cachedNodeInfoMap[candidateNode.Name]; ok {
      return nodeInfo.Node(), nodeToVictims[candidateNode].Pods, nominatedPods, err
   }

   return nil, nil, nil, fmt.Errorf(
      "preemption failed: the target node %s has been deleted from scheduler cache",
      candidateNode.Name)
}
```

上面涉及到一些子过程调用，我们逐个来看～

1. `podEligibleToPreemptOthers()` // 如何判断是否适合抢占？
2. `nodesWherePreemptionMightHelp()` // 怎么寻找能够用于 preempt 的 nodes？
3. `selectNodesForPreemption()` // 这个过程计算的是什么？
4. `pickOneNodeForPreemption()` // 怎么从选择最合适被抢占的 node？

#### podEligibleToPreemptOthers

- `podEligibleToPreemptOthers` 做的事情是判断一个 pod 是否应该去抢占其他 pods. 如果这个 pod 已经抢占过其他 pods，那些 pods 还在 graceful termination period 中，那就不应该再次发生抢占。
- 如果一个 node 已经被这个 pod nominated，并且这个 node 上有处于 terminating 状态的 pods，那么就不考虑驱逐更多的 pods.

这个函数逻辑很简单，我们直接看源码：

!FILENAME pkg/scheduler/core/generic_scheduler.go:1110

```go
func podEligibleToPreemptOthers(pod *v1.Pod, nodeNameToInfo map[string]*schedulercache.NodeInfo) bool {
   nomNodeName := pod.Status.NominatedNodeName
    // 如果 pod.Status.NominatedNodeName 不是空字符串
   if len(nomNodeName) > 0 {
       // 被 nominate 的 node
      if nodeInfo, found := nodeNameToInfo[nomNodeName]; found {
         for _, p := range nodeInfo.Pods() {
             // 有低优先级的 pod 处于删除中状态，就返回 false
            if p.DeletionTimestamp != nil && util.GetPodPriority(p) < util.GetPodPriority(pod) {
               // There is a terminating pod on the nominated node.
               return false
            }
         }
      }
   }
   return true
}
```

#### nodesWherePreemptionMightHelp

`nodesWherePreemptionMightHelp` 要做的事情是寻找 predicates 阶段失败但是通过抢占也许能够调度成功的 nodes.

这个函数也不怎么长，看下代码：

!FILENAME pkg/scheduler/core/generic_scheduler.go:1060

```go
func nodesWherePreemptionMightHelp(nodes []*v1.Node, failedPredicatesMap FailedPredicateMap) []*v1.Node {
    // 潜力 node， 用于存储返回值的 slice
   potentialNodes := []*v1.Node{}
   for _, node := range nodes {
       // 这个为 true 表示一个 node 驱逐 pod 也不一定能适合当前 pod 运行
      unresolvableReasonExist := false
       // 一个 node 对应的所有失败的 predicates
      failedPredicates, _ := failedPredicatesMap[node.Name]
      // 遍历，看是不是再下面指定的这些原因中，如果在，就标记 unresolvableReasonExist = true
      for _, failedPredicate := range failedPredicates {
         switch failedPredicate {
         case
            predicates.ErrNodeSelectorNotMatch,
            predicates.ErrPodAffinityRulesNotMatch,
            predicates.ErrPodNotMatchHostName,
            predicates.ErrTaintsTolerationsNotMatch,
            predicates.ErrNodeLabelPresenceViolated,
            predicates.ErrNodeNotReady,
            predicates.ErrNodeNetworkUnavailable,
            predicates.ErrNodeUnderDiskPressure,
            predicates.ErrNodeUnderPIDPressure,
            predicates.ErrNodeUnderMemoryPressure,
            predicates.ErrNodeOutOfDisk,
            predicates.ErrNodeUnschedulable,
            predicates.ErrNodeUnknownCondition,
            predicates.ErrVolumeZoneConflict,
            predicates.ErrVolumeNodeConflict,
            predicates.ErrVolumeBindConflict:
            unresolvableReasonExist = true
             // 如果找到一个上述失败原因，说明这个 node 已经可以排除了，break 后继续下一个 node 的计算
            break
         }
      }
       // false 的时候，也就是这个 node 也许驱逐 pods 后有用，那就添加到 potentialNodes 中
      if !unresolvableReasonExist {
         klog.V(3).Infof("Node %v is a potential node for preemption.", node.Name)
         potentialNodes = append(potentialNodes, node)
      }
   }
   return potentialNodes
}
```

#### selectNodesForPreemption

这个函数会并发计算所有的 nodes 是否通过驱逐实现 pod 抢占。

看这个函数内容之前我们先看一下返回值的类型：

`map[*v1.Node]*schedulerapi.Victims` 的 key 很好理解，value 是啥呢：

```go
type Victims struct {
   Pods             []*v1.Pod
   NumPDBViolations int
}
```

这里的 **Pods** 是被选中准备要驱逐的；**NumPDBViolations** 表示的是要破坏多少个 PDB 限制。这里肯定也就是要尽量符合 PDB 要求，能不和 PDB 冲突就不冲突。

然后看一下这个函数的整体过程：

!FILENAME pkg/scheduler/core/generic_scheduler.go:895

```go
func selectNodesForPreemption(pod *v1.Pod,
   nodeNameToInfo map[string]*schedulercache.NodeInfo,
   potentialNodes []*v1.Node, // 上一个函数计算出来的 nodes
   predicates map[string]algorithm.FitPredicate,
   metadataProducer algorithm.PredicateMetadataProducer,
   queue internalqueue.SchedulingQueue, // 这里其实是前面讲的优先级队列 PriorityQueue
   pdbs []*policy.PodDisruptionBudget, // pdb 列表
) (map[*v1.Node]*schedulerapi.Victims, error) { 
   nodeToVictims := map[*v1.Node]*schedulerapi.Victims{}
   var resultLock sync.Mutex

   // We can use the same metadata producer for all nodes.
   meta := metadataProducer(pod, nodeNameToInfo)
    // 这种形式的并发已经不陌生了，前面遇到过几次了
   checkNode := func(i int) {
      nodeName := potentialNodes[i].Name
      var metaCopy algorithm.PredicateMetadata
      if meta != nil {
         metaCopy = meta.ShallowCopy()
      }
       // 这里有一个子过程调用，下面单独介绍
      pods, numPDBViolations, fits := selectVictimsOnNode(pod, metaCopy, nodeNameToInfo[nodeName], predicates, queue, pdbs)
      if fits {
         resultLock.Lock()
         victims := schedulerapi.Victims{
            Pods:             pods,
            NumPDBViolations: numPDBViolations,
         }
          // 如果 fit，就添加到 nodeToVictims 中，也就是最后的返回值
         nodeToVictims[potentialNodes[i]] = &victims
         resultLock.Unlock()
      }
   }
   workqueue.ParallelizeUntil(context.TODO(), 16, len(potentialNodes), checkNode)
   return nodeToVictims, nil
}
```

上面这个函数的核心逻辑在 **selectVictimsOnNode** 中，这个函数尝试在给定的 node 中寻找最少数量的需要被驱逐的 pods，同时需要保证驱逐了这些 pods 之后，这个 noode 能够满足“pod”运行需求。

这些被驱逐的 pods 计算同时需要满足一个约束，就是能够删除低优先级的 pod 绝不先删高优先级的 pod.

这个算法首选计算当这个 node 上所有的低优先级 pods 被驱逐之后能否调度“pod”. 如果可以，那就按照优先级排序，根据 PDB 是否破坏分成两组，一组是影响 PDB 限制的，另外一组是不影响 PDB. 两组各自按照优先级排序。然后开始逐渐释放影响 PDB 的 group 中的 pod，然后逐渐释放不影响 PDB 的 group 中的 pod，在这个过程中要保持“pod”能够 fit 这个 node. 也就是说一旦放过某一个 pod 导致“pod”不 fit 这个 node 了，那就说明这个 pod 不能放过，也就是意味着已经找到了最少 pods 集。

看一下具体的实现吧：

FILENAME pkg/scheduler/core/generic_scheduler.go:983

```go
func selectVictimsOnNode(
   pod *v1.Pod,
   meta algorithm.PredicateMetadata,
   nodeInfo *schedulercache.NodeInfo,
   fitPredicates map[string]algorithm.FitPredicate,
   queue internalqueue.SchedulingQueue,
   pdbs []*policy.PodDisruptionBudget,
) ([]*v1.Pod, int, bool) {
   if nodeInfo == nil {
      return nil, 0, false
   }
    // 排个序
   potentialVictims := util.SortableList{CompFunc: util.HigherPriorityPod}
   nodeInfoCopy := nodeInfo.Clone()

    // 定义删除 pod 函数
   removePod := func(rp *v1.Pod) {
      nodeInfoCopy.RemovePod(rp)
      if meta != nil {
         meta.RemovePod(rp)
      }
   }
    // 定义添加 pod 函数
   addPod := func(ap *v1.Pod) {
      nodeInfoCopy.AddPod(ap)
      if meta != nil {
         meta.AddPod(ap, nodeInfoCopy)
      }
   }
   // 删除所有的低优先级 pod 看是不是能够满足调度需求了
   podPriority := util.GetPodPriority(pod)
   for _, p := range nodeInfoCopy.Pods() {
      if util.GetPodPriority(p) < podPriority {
          // 删除的意思其实就是添加元素到 potentialVictims.Items
         potentialVictims.Items = append(potentialVictims.Items, p)
         removePod(p)
      }
   }
    // 排个序
   potentialVictims.Sort()
   // 如果删除了所有的低优先级 pods 之后还不能跑这个新 pod，那么差不多就可以判断这个 node 不适合 preemption 了，还有一点点需要考虑的是这个“pod”的不 fit 的原因是由于 pod affinity 不满足了。
    // 后续可能会增加当前 pod 和低优先级 pod 之间的 优先级检查。
    
    // 这个函数调用其实就是之前讲到过的预选函数的调用逻辑，判断这个 pod 是否合适跑在这个 node 上。
   if fits, _, err := podFitsOnNode(pod, meta, nodeInfoCopy, fitPredicates, nil, queue, false, nil); !fits {
      if err != nil {
         klog.Warningf("Encountered error while selecting victims on node %v: %v", nodeInfo.Node().Name, err)
      }
      return nil, 0, false
   }
   var victims []*v1.Pod
   numViolatingVictim := 0
   // 尝试尽量多地释放这些 pods，也就是说能少杀就少杀；这里先从 PDB violating victims 中释放，再从 PDB non-violating victims 中释放；两个组都是从高优先级的 pod 开始释放。
   violatingVictims, nonViolatingVictims := filterPodsWithPDBViolation(potentialVictims.Items, pdbs)
    // 释放 pods 的函数，来一个放一个
   reprievePod := func(p *v1.Pod) bool {
      addPod(p)
      fits, _, _ := podFitsOnNode(pod, meta, nodeInfoCopy, fitPredicates, nil, queue, false, nil)
      if !fits {
         removePod(p)
         victims = append(victims, p)
         klog.V(5).Infof("Pod %v is a potential preemption victim on node %v.", p.Name, nodeInfo.Node().Name)
      }
      return fits
   }
    // 释放 violatingVictims 中元素的同时会记录放了多少个
   for _, p := range violatingVictims {
      if !reprievePod(p) {
         numViolatingVictim++
      }
   }
   // 开始释放 non-violating victims.
   for _, p := range nonViolatingVictims {
      reprievePod(p)
   }
   return victims, numViolatingVictim, true
}
```

#### pickOneNodeForPreemption

`pickOneNodeForPreemption` 要从给定的 nodes 中选择一个 node，这个函数假设给定的 map 中 value 部分是以 priority 降序排列的。这里选择 node 的标准是：

1. 最少的 PDB violations
2. 最少的高优先级 victim
3. 优先级总数字最小
4. victim 总数最小
5. 直接返回第一个

!FILENAME pkg/scheduler/core/generic_scheduler.go:788

```go
func pickOneNodeForPreemption(nodesToVictims map[*v1.Node]*schedulerapi.Victims) *v1.Node {
   if len(nodesToVictims) == 0 {
      return nil
   }
    // 初始化为最大值
   minNumPDBViolatingPods := math.MaxInt32
   var minNodes1 []*v1.Node
   lenNodes1 := 0
    // 这个循环要找到 PDBViolatingPods 最少的 node，如果有多个，就全部存在 minNodes1 中
   for node, victims := range nodesToVictims {
      if len(victims.Pods) == 0 {
         // 如果发现一个不需要驱逐 pod 的 node，马上返回
         return node
      }
      numPDBViolatingPods := victims.NumPDBViolations
      if numPDBViolatingPods < minNumPDBViolatingPods {
         minNumPDBViolatingPods = numPDBViolatingPods
         minNodes1 = nil
         lenNodes1 = 0
      }
      if numPDBViolatingPods == minNumPDBViolatingPods {
         minNodes1 = append(minNodes1, node)
         lenNodes1++
      }
   }
    // 如果只找到1个 PDB violations 最少的 node，那就直接返回这个 node 就 ok 了
   if lenNodes1 == 1 {
      return minNodes1[0]
   }

   // 还剩下多个 node，那就寻找 highest priority victim 最小的 node
   minHighestPriority := int32(math.MaxInt32)
   var minNodes2 = make([]*v1.Node, lenNodes1)
   lenNodes2 := 0
    // 这个循环要做的事情是看2个 node 上 victims 中最高优先级的 pod 哪个优先级更高
   for i := 0; i < lenNodes1; i++ {
      node := minNodes1[i]
      victims := nodesToVictims[node]
      // highestPodPriority is the highest priority among the victims on this node.
      highestPodPriority := util.GetPodPriority(victims.Pods[0])
      if highestPodPriority < minHighestPriority {
         minHighestPriority = highestPodPriority
         lenNodes2 = 0
      }
      if highestPodPriority == minHighestPriority {
         minNodes2[lenNodes2] = node
         lenNodes2++
      }
   }
    // 发现只有1个，那就直接返回
   if lenNodes2 == 1 {
      return minNodes2[0]
   }

   // 这时候还没有抉择出一个 node，那就开始计算优先级总和了，看哪个更低
   minSumPriorities := int64(math.MaxInt64)
   lenNodes1 = 0
   for i := 0; i < lenNodes2; i++ {
      var sumPriorities int64
      node := minNodes2[i]
      for _, pod := range nodesToVictims[node].Pods {
         // 这里的累加考虑到了先把优先级搞成正数。不然会出现1个 node 上有1优先级为 -3 的 pod，另外一个 node 上有2个优先级为 -3 的 pod，结果 -3>-6，有2个 pod 的 node 反而被认为总优先级更低！
         sumPriorities += int64(util.GetPodPriority(pod)) + int64(math.MaxInt32+1)
      }
      if sumPriorities < minSumPriorities {
         minSumPriorities = sumPriorities
         lenNodes1 = 0
      }
      if sumPriorities == minSumPriorities {
         minNodes1[lenNodes1] = node
         lenNodes1++
      }
   }
   if lenNodes1 == 1 {
      return minNodes1[0]
   }

   // 还是没有分出胜负，于是开始用 pod 总数做比较
   minNumPods := math.MaxInt32
   lenNodes2 = 0
   for i := 0; i < lenNodes1; i++ {
      node := minNodes1[i]
      numPods := len(nodesToVictims[node].Pods)
      if numPods < minNumPods {
         minNumPods = numPods
         lenNodes2 = 0
      }
      if numPods == minNumPods {
         minNodes2[lenNodes2] = node
         lenNodes2++
      }
   }
   // 还是没有区分出来1个 node 的话，只能放弃区分了，直接返回第一个结果
   if lenNodes2 > 0 {
      return minNodes2[0]
   }
   klog.Errorf("Error in logic of node scoring for preemption. We should never reach here!")
   return nil
}
```

## 小结

咋个说呢，此处应该有总结的，抢占过程的逻辑比我想象中的复杂，设计很巧妙，行云流水，大快人心！preemption 可以简单说成再预选->再优选吧；还是不多说了，一天写这么多有点坐不住了，下回再继续聊调度器～