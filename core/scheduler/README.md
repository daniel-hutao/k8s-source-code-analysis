# scheduler

Scheduler部分我们先从设计原理上介绍，然后分析源码，再准备环境调试，最后修改一下源码，实现一个自己的调度器。

## 分析过程

- [design](./design.md)
- [scheduler-framework](./scheduler-framework.md)
- [before-scheduler-run](./before-scheduler-run.md)
- [generic-scheduler](./generic-scheduler.md)
- [predicate](./predicate.md)
- [priority](./priority.md)
- [preempt](./preempt.md)

## FAQ

**读者A提问**：如果一个pod的资源占用只有100M，能够运行在一个node上，但是配置成了1000M，这个时候node上其实没有1000M，那么predicate过程还能不能过滤通过？

**回答**：如果一个人需要100块钱，卡里有1000块钱，这时候找银行要10000块，银行会给吗？银行不会知道你实际需要多少，你告诉他10000，他就看你卡里有没有10000；同样对于k8s来说你配置了需要1000M，k8s就看node上有没有1000M，没有就调度失败。