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
