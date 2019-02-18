# 源码准备

<!-- toc -->

## 环境准备

操作系统：我们使用Linux作为k8s源码分析和调试环境，fedora、centos、ubuntu都行，我这里使用fedora；

golang相关：

- GOROOT=/usr/local/lib/golang
- GOPATH=/root/go
- go version go1.10.3 linux/amd64

## 源码下载

```sh
mkdir -p /root/go/src/k8s.io
cd /root/go/src/k8s.io/
git clone https://github.com/kubernetes/kubernetes.git
```

下载后本地目录：

![1550208476439](./image/1550208476439.png)

## 源码编译

我们先看一下几个主要的目录：

| 目录名  | 用途                         |
| ------- | ---------------------------- |
| cmd     | 每个组件代码入口（main函数） |
| pkg     | 各个组件的具体功能实现       |
| staging | 已经分库的项目               |
| vendor  | 依赖                         |

考虑到国内网络环境等因素，我们不使用容器化方式构建。我们尝试在kubernetes项目cmd目录下构建一个组件（执行路径：`/root/go/src/k8s.io/kubernetes/cmd/kube-scheduler`）：

![1550221168405](./image/1550221168405.png)

这里需要注意一下，如果报依赖错误，找不到k8s.io下的某些项目，就到vendor下看一下软链接是不是都还在，如下：

![1550477991608](./image/1550477991608.png)

注意到k8s是使用这种方式解决k8s.io下的依赖问题的，如果我们在windows下下载的代码，然后copy到linux下，就很容易遇到这些软链接丢失的情况，导致go找不到依赖，编译失败。

## IDE

我们使用Goland看代码：

![1550328005342](./image/1550328005342.png)

最后，别忘了在正式研读源码前切换到`release-1.13`分支～