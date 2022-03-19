# kubernetes-design ![](https://visitor-badge.glitch.me/badge?page_id=kerthcet.kubernetes-design)
Kubernetes 源码学习笔记📰。如有错误，欢迎指正📌。(持续更新🌱)

最新更新：2022-03-19 [Kube-Scheduler Event机制](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/event.md)

<!-- ![image](https://github.com/kerthcet/KubernetesSchedulingDesign/blob/main/snapshots/wechat.jpeg) -->

 ## 索引:

 ### scheduler
* [Kube-Scheduler 初始化](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/initialization.md)
* [Kube-Scheduler 启动](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/start-scheduler.md)
* [Kube-Scheduler 调度队列](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/queue.md)
* [Kube-Scheduler 优先级与抢占](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/priority-preemption.md)
* [Kube-Scheduler Framework调度框架](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/framework.md)
* [Kube-Scheduler Cache机制](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/cache.md)
* [Kube-Scheduler Event机制](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/event.md)
* Kube-Scheduler 插件机制
* Kube-Scheduler 如何手写一个插件
* Kube-Scheduler 多版本控制如何实现
* Kube-Scheduler Extender
* Kube-Scheduler Informer机制
* Kube-Scheduler Event处理机制
* Kube-Scheduler Metrics机制
* Kube-Scheduler 如何解决调度不均问题？
* Kube-Scheduler Descheduler 机制
* Kube-Scheduler PodNominator 机制
* Kube-Scheduler 高可用设计

### kube-apiserver
* AdmissionController 源码解析

### kubectl
* [Kubectl Builder & Visitor 设计模式解析](https://github.com/kerthcet/kubernetes-design/blob/main/kubectl/builder-visitor-pattern.md)