# 核心组件源码分析

## 概述

核心组件的源码分析主要包括：

1. [kube-scheduler](./scheduler/README.md)
2. [kube-controller-manager](./controller-manager/README.md)
3. apiserver
4. proxy
5. kubelet

在分析第一个组件的时候会穿插一些整体性的介绍，比如源码组织啊、使用的一些三方库啊……；后面有些组件比较依赖其他较大的项目的，比如一个核心组件依赖于对client-go项目的理解，那就会先介绍client-go，当然client-go的介绍不会混在核心组件分析的章节中，我会单独分一个大类“周边项目源码分析”中。