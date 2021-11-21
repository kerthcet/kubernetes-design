# kubernetes-design ![](https://visitor-badge.glitch.me/badge?page_id=kerthcet.kubernetes-design)
Kubernetes 源码解析系列文章📰。如有错误，欢迎指正📌。(持续更新...)

<!-- ![image](https://github.com/kerthcet/KubernetesSchedulingDesign/blob/main/snapshots/wechat.jpeg) -->

 ## 目录:

 ### 1. scheduler
1. [Kube-Scheduler 初始化](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/initialization.md)
2. [Kube-Scheduler 启动](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/start-scheduler.md)
3. [Kube-Scheduler 调度队列](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/queue.md)
4. Kube-Scheduler 插件机制
5. Kube-Scheduler 如何手写一个插件
6. Kube-Scheduler 调度机制
7. Kube-Scheduler 多版本控制如何实现
8. Kube-Scheduler Framework调度框架
9. Kube-Scheduler Extender
10. Kube-Scheduler Informer机制
11. Kube-Scheduler Event处理机制
12. Kube-Scheduler Metrics机制
13. Kube-Scheduler 如何解决调度不均问题？
14. Kube-Scheduler Descheduler 机制
15. Kube-Scheduler PodNominator 机制
16. Kube-Scheduler 高可用设计
17. Kube-Scheduler 抢占调度

### 2. kube-apiserver
1. AdmissionController 源码解析

### 3. kubectl
1. [Kubectl Builder & Visitor 设计模式解析](https://github.com/kerthcet/kubernetes-design/blob/main/kubectl/builder-visitor-pattern.md)