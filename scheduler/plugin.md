# Scheduler 插件

    Kubernetes Version: v1.22@80056f73a614
    Date: 2021.10.17

## 1. 开篇
Kube-Scheduler是Kubernetes架构设计中的核心模块，负责整个集群的容器编排和调度策略。得益于Kubernetes整体架构设计的一致性，我们知道所有的组件启动命令都位于 `cmd` 文件夹下各个子模块的 `main` 方法内，`scheduling` 模块的具体代码位置位于 `cmd/kube-scheduler/scheduler.go` ，代码如下：