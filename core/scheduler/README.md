# scheduler

**<u>本章 owner：farmer-hutao</u>**

Scheduler部分我们先从设计原理上介绍，然后分析源码，最后针对一些主要算法做专题分析。

## 本章规划

1. [调度器设计](./design.md)
2. [调度程序启动前逻辑](./before-scheduler-run.md)
3. [调度器框架](./scheduler-framework.md)
4. [一般调度过程](./generic-scheduler.md)
5. [预选过程](./predicate.md)
6. [优选过程](./priority.md)
7. [抢占调度](./preempt.md)
8. [调度器初始化](./init.md)
9. [专题-亲和性调度](./affinity.md)
## FAQ

**读者A提问**：如果一个pod的资源占用只有100M，能够运行在一个node上，但是配置成了1000M，这个时候node上其实没有1000M，那么predicate过程还能不能过滤通过？

**回答**：如果一个人需要100块钱，卡里有1000块钱，这时候找银行要10000块，银行会给吗？银行不会知道你实际需要多少，你告诉他10000，他就看你卡里有没有10000；同样对于k8s来说你配置了需要1000M，k8s就看node上有没有1000M，没有就调度失败。