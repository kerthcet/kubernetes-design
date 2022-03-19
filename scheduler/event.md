# Scheduler Event机制

    Kubernetes Version: v1.23@a504daa0
    Date: 2022.03.19

## 1. 开篇
Kubernetes 通过 `informer` 的 `reflector` 组件监听 API 事件，这些事件在 scheduler 中会触发对应的 cache 操作和队列任务，今天我们就从源码层面看一下它们是如何工作的。

## 2. 数据结构
在将事件处理的具体逻辑之前，我们先捋一捋会涉及到的几种数据结构：
1. `ResourceEventHandler` 接口，定义如下：
	```golang
	type ResourceEventHandler interface {
		OnAdd(obj interface{})
		OnUpdate(oldObj, newObj interface{})
		OnDelete(obj interface{})
	}
	```
	这个接口定义了处理 Event 的3个方法，分别是 Add 事件的 `OnAdd` 方法，Update 事件的 `OnUpdate` 方法，以及 Delete 事件的 `OnDelete` 方法。

2. `FilteringResourceEventHandler` 是接口 `ResourceEventHandler` 的一个具体实现：
	```golang
	type FilteringResourceEventHandler struct {
		FilterFunc func(obj interface{}) bool
		Handler    ResourceEventHandler
	}
	```
	`FilterFunc` 用来过滤事件，`Handler` 则封装了具体的处理方法，我们一次看一下3个方法:

	OnAdd:
	```golang
	func (r FilteringResourceEventHandler) OnAdd(obj interface{}) {
		// 如果没有通过 FilterFunc，则直接返回
		// 否则执行 OnAdd 逻辑
		if !r.FilterFunc(obj) {
			return
		}
		r.Handler.OnAdd(obj)
	}
	```

	OnUpdate:
	```golang
	func (r FilteringResourceEventHandler) OnUpdate(oldObj, newObj interface{}) {
		newer := r.FilterFunc(newObj)
		older := r.FilterFunc(oldObj)
		switch {
		// 这里进行了聚合操作，虽然是 Update 事件，但是处理方式却不尽相同
		case newer && older:
			r.Handler.OnUpdate(oldObj, newObj)
		case newer && !older:
			r.Handler.OnAdd(newObj)
		case !newer && older:
			r.Handler.OnDelete(oldObj)
		default:
			// do nothing
		}
	}
	```

	OnDelete:
	```golang
	func (r FilteringResourceEventHandler) OnDelete(obj interface{}) {
		// 如果没有通过 Filter，则直接返回
		// 否则进行删除逻辑
		if !r.FilterFunc(obj) {
			return
		}
		r.Handler.OnDelete(obj)
	}
	```

## 3. 事件处理
`scheduler` 事件处理逻辑位于代码 `pkg/scheduler/eventhandlers.go:L251`，其中
```golang
func addAllEventHandlers(
	sched *Scheduler,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	gvkMap map[framework.GVK]framework.ActionType,
) {
	// 监听 pod 事件并触发 scheduler cache 对应的处理方法，详见 https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/cache.md
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		// 前面讲过 FilteringResourceEventHandler 是 ResourceEventHandler 的具体实现，其中 FilterFunc 用于过滤事件
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				// 如果是 pod，则判断 pod 是否为已调度 Pod，可以通过 pod.Spec.NodeName 是否有值判断
				case *v1.Pod:
					return assignedPod(t)
				// 处理丢失的删除事件
				case cache.DeletedFinalStateUnknown:
					if _, ok := t.Obj.(*v1.Pod); ok {
						return true
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			// 具体的处理逻辑，前面的文章已经讲过
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToCache,
				UpdateFunc: sched.updatePodInCache,
				DeleteFunc: sched.deletePodFromCache,
			},
		},
	)
	// 上面是触发 cache 相关操作，这里是触发 scheduler queue 相关逻辑
	informerFactory.Core().V1().Pods().Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					// 判断 pod 是否完成调度，即 pod.Spec.NodeName 有值
					// 另外，还需要该 pod SchedulerName 为已经注册的 调度器名称
					return !assignedPod(t) && responsibleForPod(t, sched.Profiles)
				// 处理丢失的删除事件
				case cache.DeletedFinalStateUnknown:
					if pod, ok := t.Obj.(*v1.Pod); ok {
						return responsibleForPod(pod, sched.Profiles)
					}
					utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
					return false
				default:
					utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
					return false
				}
			},
			// 丢到队列中进行处理，这里不详细介绍
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
		},
	)

	// 注册 Node 事件处理函数
	informerFactory.Core().V1().Nodes().Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sched.addNodeToCache,
			UpdateFunc: sched.updateNodeInCache,
			DeleteFunc: sched.deleteNodeFromCache,
		},
	)

	// 动态构造事件处理方法，其中 ActionType 表示事件类型，通过 & 进行与判断
	buildEvtResHandler := func(at framework.ActionType, gvk framework.GVK, shortGVK string) cache.ResourceEventHandlerFuncs {
		funcs := cache.ResourceEventHandlerFuncs{}
		// Add 事件
		if at&framework.Add != 0 {
			evt := framework.ClusterEvent{Resource: gvk, ActionType: framework.Add, Label: fmt.Sprintf("%vAdd", shortGVK)}
			funcs.AddFunc = func(_ interface{}) {
				sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(evt, nil)
			}
		}
		// Update 事件
		if at&framework.Update != 0 {
			evt := framework.ClusterEvent{Resource: gvk, ActionType: framework.Update, Label: fmt.Sprintf("%vUpdate", shortGVK)}
			funcs.UpdateFunc = func(_, _ interface{}) {
				sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(evt, nil)
			}
		}
		// Delete 事件
		if at&framework.Delete != 0 {
			evt := framework.ClusterEvent{Resource: gvk, ActionType: framework.Delete, Label: fmt.Sprintf("%vDelete", shortGVK)}
			funcs.DeleteFunc = func(_ interface{}) {
				sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(evt, nil)
			}
		}
		return funcs
	}

	// 处理各类事件
	for gvk, at := range gvkMap {
		switch gvk {
		case framework.Node, framework.Pod:
			// 前面已经单独处理过 Node 和 Pod 事件，这里就不再处理了
		case framework.CSINode:
			informerFactory.Storage().V1().CSINodes().Informer().AddEventHandler(
				buildEvtResHandler(at, framework.CSINode, "CSINode"),
			)
		case framework.CSIDriver:
			informerFactory.Storage().V1().CSIDrivers().Informer().AddEventHandler(
				buildEvtResHandler(at, framework.CSIDriver, "CSIDriver"),
			)
		case framework.CSIStorageCapacity:
			informerFactory.Storage().V1beta1().CSIStorageCapacities().Informer().AddEventHandler(
				buildEvtResHandler(at, framework.CSIStorageCapacity, "CSIStorageCapacity"),
			)
		case framework.PersistentVolume:
			informerFactory.Core().V1().PersistentVolumes().Informer().AddEventHandler(
				buildEvtResHandler(at, framework.PersistentVolume, "Pv"),
			)
		case framework.PersistentVolumeClaim:
			informerFactory.Core().V1().PersistentVolumeClaims().Informer().AddEventHandler(
				buildEvtResHandler(at, framework.PersistentVolumeClaim, "Pvc"),
			)
		case framework.StorageClass:
			if at&framework.Add != 0 {
				informerFactory.Storage().V1().StorageClasses().Informer().AddEventHandler(
					cache.ResourceEventHandlerFuncs{
						AddFunc: sched.onStorageClassAdd,
					},
				)
			}
			if at&framework.Update != 0 {
				informerFactory.Storage().V1().StorageClasses().Informer().AddEventHandler(
					cache.ResourceEventHandlerFuncs{
						UpdateFunc: func(_, _ interface{}) {
							sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.StorageClassUpdate, nil)
						},
					},
				)
			}
		default:
			// 使用 dynamic informer
			// 测试用例可能不注册 dynInformerFactory，所以这里跳过
			if dynInformerFactory == nil {
				continue
			}
			// 对 GVK 格式进行校验
			if strings.Count(string(gvk), ".") < 2 {
				klog.ErrorS(nil, "incorrect event registration", "gvk", gvk)
				continue
			}

			gvr, _ := schema.ParseResourceArg(string(gvk))
			dynInformer := dynInformerFactory.ForResource(*gvr).Informer()
			dynInformer.AddEventHandler(
				buildEvtResHandler(at, gvk, strings.Title(gvr.Resource)),
			)
		}
	}
}
```

我们看到所谓的事件处理，其实就是将捕获的事件转化为调度事件。另外，我们注意到方法中的 `gvkMap` 存储了所有的事件，那么这些事件是从何而来的呢，回溯 `addAllEventHandlers` 方法，发现在初始化 scheduler 的时候，会把对应的事件传入：

```golang
addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))
```
其中的 `clusterEventMap` 就是原始的事件集合，它是一个 `cluster event -> plugin names` 的字典，但奇怪的是，没有在 scheduler 初始化方法中看到填充字典的逻辑，既然 clusterEventMap 的值是插件名称，而 scheduler framework 负责管理所有的插件，是不是在 framework 初始化的时候处理了该逻辑呢，事实也确实如何，我们在之前讲解 framework 框架的时候也提到过。
```golang
clusterEventMap map[framework.ClusterEvent]sets.String
```

## 4. 事件注册
事件注册逻辑的入口位于 `pkg/scheduler/framework/runtime/framework.go:L330` `NewFramework` 方法中：
```golang
func fillEventToPluginMap(p framework.Plugin, eventToPlugins map[framework.ClusterEvent]sets.String) {
	// `EnqueueExtensions` 是一个接口，插件需要实现该接口的 `EventsToRegister` 方法
	ext, ok := p.(framework.EnqueueExtensions)
	if !ok {
		// 为了保持兼容性，如果插件没有实现接口，则注册所有的事件
		registerClusterEvents(p.Name(), eventToPlugins, allClusterEvents)
		return
	}

	// 获取需要注册的事件
	events := ext.EventsToRegister()
	// 如果没有，则返回
	if len(events) == 0 {
		klog.InfoS("Plugin's EventsToRegister() returned nil", "plugin", p.Name())
		return
	}
	// 注册事件
	registerClusterEvents(p.Name(), eventToPlugins, events)
}
```

如此，完成了事件的注册逻辑。我们以插件 `PodTopologySpread` 为例，看看他的 `EventsToRegister` 方法：
```golang
func (pl *PodTopologySpread) EventsToRegister() []framework.ClusterEvent {
	return []framework.ClusterEvent{
		{Resource: framework.Pod, ActionType: framework.All},
		{Resource: framework.Node, ActionType: framework.Add | framework.Delete | framework.UpdateNodeLabel},
	}
}
```
我们看到插件注册了 Pod 所有的事件，以及 Node 的增删事件和 label 更新事件。

## 5. 总结
至此，scheduler 事件机制就说完了。核心逻辑就是通过 informer reflector 机制监听 API 事件，并通过插件注册对应事件的监听规则，转化为调度事件，执行缓存和队列逻辑。