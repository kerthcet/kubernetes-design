## Scheduler Informer 机制 (InProgres)
第二个配置项是 `InformerFactory`，它是一个 `sharedInformerFactory` 实例，该实例结构体如下:

        type sharedInformerFactory struct {
            client           kubernetes.Interface
            namespace        string
            tweakListOptions internalinterfaces.TweakListOptionsFunc
            lock             sync.Mutex
            defaultResync    time.Duration
            customResync     map[reflect.Type]time.Duration
            informers map[reflect.Type]cache.SharedIndexInformer
            startedInformers map[reflect.Type]bool
        }
`sharedInformerFactory` 负责构造各种 `informer` 对象，我们看到他有一个 `informers` 的 `map` 对象，用于存放各种 `informer`，通过共享一个 `sharedInformerFactory` 实例，`informer` 之间可以实现信息互通，以 `replicaController` 为例， 它不仅需要初始化 `replicationInformer`，还需要初始化 `podInformer`，这样就可以同时获得 `pod` 的创建删除事件，代码见下。同时用 `informer` 类型作为 `key`，也不需要竞争锁。

    func startReplicationController(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
        go replicationcontroller.NewReplicationManager(
            controllerContext.InformerFactory.Core().V1().Pods(), // podInformer
            controllerContext.InformerFactory.Core().V1().ReplicationControllers(), // replicationInformer
        ).Run(ctx, int(controllerContext.ComponentConfig.ReplicationController.ConcurrentRCSyncs))
        return nil, true, nil
    }

另外，细心的同学会发现还有一个 `Config()` 方法中还初始化了一个 `DynInformerFactory`，它的作用主要是针对一些 `CR` 资源，更多关于 `informer` 的介绍后面也会单独写一篇文章，这里主要是把这些东西都串起来。



    map[framework.ClusterEvent]sets.String

它是一个 `map` 结构，`key` 是一个结构体 `ClusterEvent`，`value` 是一个 `plugins name` 的 `map` 集合，之所以用 `map`，是因为 `golang` 没有 `set` 集合的概念，所以用同样具有去重功能的 `map` 来替代了。

    type ClusterEvent struct {
        Resource   GVK
        ActionType ActionType
        Label      string
    }

那 `c.clusterEventMap` 值从哪里来呢，我们往前看，来到 `pkg/scheduler/framework/runtime/framework.go` 的 `NewFramework()` 方法。

	func NewFramework(r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {

		// ...

		for name, factory := range r {
			// ...
			fillEventToPluginMap(p, options.clusterEventMap)
		}
	}

这个方法调用了 `fillEventToPluginMap()`，其中 `p` 表示 `Plugin`，`clusterEventMap` 就是我们前面提到的参数，通过函数名我们判断是在这里进行了赋值操作。我们继续看这个方法的代码:

	func fillEventToPluginMap(p framework.Plugin, eventToPlugins map[framework.ClusterEvent]sets.String) {
		ext, ok := p.(framework.EnqueueExtensions)

		// ...

		events := ext.EventsToRegister()
		registerClusterEvents(p.Name(), eventToPlugins, events)
	}

核心代码就2行，`EventsToRegister()` 是一个接口方法，每个 `Plugin` 都需要实现这个方法，表示可能会引起该插件调度失败的事件，以 `NodeAffinity` 为例：TODO:。 `registerClusterEvents` 就会拼接我们最终想要的结果，其中一个 `event` 事件对应多个 `plugins` 名字。

	func (pl *NodeAffinity) EventsToRegister() []framework.ClusterEvent {
		return []framework.ClusterEvent{
			{Resource: framework.Node, ActionType: framework.Add | framework.UpdateNodeLabel},
		}
	}