# Scheduler 调度框架

    Kubernetes Version: v1.23@f1da8cd3e20
    Date: 2022.01.22

## 1. 开篇
调度框架是面向 Kubernetes 调度器的一种插件架构， 它为现有的调度器添加了一组新的“插件” API。