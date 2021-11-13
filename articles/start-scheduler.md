# Kube-Scheduler 启动

    Kubernetes Version: v1.22@392de8012eb4116
    Date: 2021.10.31

## 1. 开篇
[上篇文章](https://github.com/kerthcet/kube-scheduler-design/blob/main/articles/RunCommand.md)讲了`scheduler` 程序启动前的初始化流程，但是程序启动相关逻辑并没有讲，今天就围绕这一块代码展开，看看 `scheduler` 启动到底运行了哪些服务。

代码入口位于 `cmd/kube-scheduler/app/server.go:runCommand()`，大致代码如下，分为信号量处理，`scheduler` 初始化以及最后运行 `scheduler`。

    func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
        // ...

        ctx, cancel := context.WithCancel(context.Background())
        defer cancel()
        go func() {
            stopCh := server.SetupSignalHandler()
            <-stopCh
            cancel()
        }()

        cc, sched, err := Setup(ctx, opts, registryOptions...)
        if err != nil {
            return err
        }

        return Run(ctx, cc, sched)
    }

## 2. 信号量处理
我们先说说 `server.SetupSignalHandler()`，主要用于通过信号量控制程序运行，目前支持 `SIGTERM` 和 `SIGINT`。这段代码逻辑很清晰，但是我觉得有一些值得学习的地方:

    func SetupSignalContext() context.Context {
        close(onlyOneSignalHandler)

        shutdownHandler = make(chan os.Signal, 2)

        ctx, cancel := context.WithCancel(context.Background())
        signal.Notify(shutdownHandler, shutdownSignals...)
        go func() {
            <-shutdownHandler
            cancel()
            <-shutdownHandler
            os.Exit(1) // second signal. Exit directly.
        }()

        return ctx
    }

第一个是 `close(onlyOneSignalHandler)`， `onlyOneSignalHandler` 是全局申明的 `channel`，通过 `close()` 方法保证整个方法只能调用一次，第二次调用就会引发 `panic`。

    var onlyOneSignalHandler = make(chan struct{})

第二个是声明了长度为2的 channel `shutdownHandler`，用于接受信号量输入，第一次信号量输入会终止程序启动，第二次输入则会立即退出。可以学习一下来实现优雅的终止程序。

## 3. 初始化 `Scheduler`
配置的初始化操作主要是在 `Setup()` 方法中实现:

    func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {

        opts.Validate()

        c, err := opts.Config()

        // ...

        outOfTreeRegistry := make(runtime.Registry)
        for _, option := range outOfTreeRegistryOptions {
            if err := option(outOfTreeRegistry); err != nil {
                return nil, nil, err
            }
        }

        sched, err := scheduler.New(...)

        return &cc, sched, nil
    }

### 3.1 `ops.Validata()`
这个方法主要是做参数校验，本身并没有太多复杂的业务逻辑，但是有一点值得学习的是该方法将所有的错误返回，并在最后通过 `NewAggregate()` 聚合，这样可以尽可能多的收集错误。

    func (o *Options) Validate() []error {
        var errs []error

        // ...
        errs = append(errs, o.Logs.Validate()...)

        return errs
    }

### 3.2 `opts.Config()`
这个方法用于生成所有的配置信息，并返回 `Config` 实例，比如 `kubeconfig` 配置，`client` 配置，以及 `metric` 和 `log` 配置等，代码如下：

    func (o *Options) Config() (*schedulerappconfig.Config, error) {
        // ...

        c := &schedulerappconfig.Config{}
        if err := o.ApplyTo(c); err != nil {
            return nil, err
        }

        // Prepare kube config.
        kubeConfig, err := createKubeConfig(c.ComponentConfig.ClientConnection, o.Master)

        // Prepare kube clients.
        client, eventClient, err := createClients(kubeConfig)

        c.EventBroadcaster = events.NewEventBroadcasterAdapter(eventClient)
        c.InformerFactory = scheduler.NewInformerFactory(client, 0)

        return c, nil
    }


这里有两个配置项值得一提，第一个是 `EventBroadcaster`，这里我们只需要知道它负责 `scheduler` 模块所有事件的上报、保存等处理工作。


第二个配置项是 `InformerFactory`，它主要通过 `ListAndWatch` 机制监听 `apiserver`，获得所有的事件，如 `pod` 创建，更新删除事件等，后面我们也会单独出一篇文章详细的讲解。

### 3.3 插件配置
插件相关代码如下：

    outOfTreeRegistry := make(runtime.Registry)
    for _, option := range outOfTreeRegistryOptions {
        if err := option(outOfTreeRegistry); err != nil {
            return nil, nil, err
        }
    }

`kubernetes` 将插件分为两类，一类称为 `In-Tree plugin`，是核心代码的一部分，默认开启，另外还有一类叫 `OutOfTree Plugins`，`kubernetes` 官方托管了一个[插件库](https://github.com/kubernetes-sigs/scheduler-plugins)，如果你想要使用外部插件，需要自己编译运行，后面插件专栏我会着重讲。最终生成的 `outOfTreeRegistry` 对象会作为参数传入 `scheduler.New()` 方法。


### 3.4 构造scheduler对象
这里就是将前面配置好的各类参数传入 `scheduler.New()` 方法，构造 `scheduler` 对象。

    func New(client clientset.Interface,
        informerFactory informers.SharedInformerFactory,
        dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
        recorderFactory profile.RecorderFactory,
        stopCh <-chan struct{},
        opts ...Option) (*Scheduler, error) {

        // ...

        if options.applyDefaultProfile {
            var versionedCfg v1beta3.KubeSchedulerConfiguration
            scheme.Scheme.Default(&versionedCfg)
            cfg := config.KubeSchedulerConfiguration{}
            if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
                return nil, err
            }
            options.profiles = cfg.Profiles
        }
        schedulerCache := internalcache.New(30*time.Second, stopEverything)

        registry := frameworkplugins.NewInTreeRegistry()
        if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
            return nil, err
        }

        snapshot := internalcache.NewEmptySnapshot()
        clusterEventMap := make(map[framework.ClusterEvent]sets.String)

        configurator := &Configurator{
            // ...
        }

        metrics.Register()

        sched, err := configurator.create()
        if err != nil {
            return nil, fmt.Errorf("couldn't create scheduler: %v", err)
        }

        // Additional tweaks to the config produced by the configurator.
        sched.StopEverything = stopEverything
        sched.client = client

        addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))

        return sched, nil
    }

我们把重要的逻辑拆开来讲一下。

#### 3.4.1 API Convert
先看第一段逻辑：

    var versionedCfg v1beta3.KubeSchedulerConfiguration
    scheme.Scheme.Default(&versionedCfg)
    cfg := config.KubeSchedulerConfiguration{}
    if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
        return nil, err
    }
    options.profiles = cfg.Profiles

我们前面讲到 `KubeSchedulerConfiguration` 目前的版本是 `v1beta3`，这里就是将我们配置的 `yaml` 文件转化为 `internal KubeSchedulerConfiguration`，这是 `kubernetes` 多版本处理的主要逻辑，就是将不同的 `api` 版本的代码统一转化为内部统一的版本，类似于接口层设计，主要是通过 `convert()` 方法实现。

`profile` 用于配置插件的开关及相应的参数设置，可以丰富我们的调度方案，取代了以前多调度器的设计。可以通过直接定义 pod 的 `spec.schedulerName` 来选择你想要的调度方案。默认 `schedulerName` 是 `default-scheduler`。

#### 3.4.2 SchedulerCache
第二段逻辑是关于声明 `pod` 缓存，默认30s超时时间，后面讲调度队列会着重介绍，我们现在只需要知道它主要用来缓存 `assumed pods`。

    schedulerCache := internalcache.New(30*time.Second, stopEverything)

#### 3.4.3 InTree Plugins
第三段逻辑关于 `Plugin` 注册，我们前面已经注册了 `OutOfTree Plugin`，这里注册的是 `InTree Plugin`，并将它们合并。

    registry := frameworkplugins.NewInTreeRegistry()
        if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
            return nil, err
    }

#### 3.4.4 Snapshot
第四段逻辑是关于快照，声明了一个 `Snapshot` 对象，可以保存节点的信息。在每一轮调度前会调用快照方法保存当前所有节点的信息。

    snapshot := internalcache.NewEmptySnapshot()

#### 3.4.5 注册 `metrics server`

    metrics.Register()

#### 3.4.6 scheduler对象
第五段逻辑就是关于生成 `scheduler` 对象，我们看一下 `create()` 代码：

    func (c *Configurator) create() (*Scheduler, error) {
        // extender...

        nominator := internalqueue.NewPodNominator(c.informerFactory.Core().V1().Pods().Lister())
        profiles, err := profile.NewMap(c.profiles, c.registry, c.recorderFactory,
            // ...
        )

        podQueue := internalqueue.NewSchedulingQueue(
            // ...
        )

        algo := NewGenericScheduler(
            c.schedulerCache,
            c.nodeInfoSnapshot,
            c.percentageOfNodesToScore,
        )

        return &Scheduler{
            SchedulerCache:  c.schedulerCache,
            Algorithm:       algo,
            Extenders:       extenders,
            Profiles:        profiles,
            NextPod:         internalqueue.MakeNextPodFunc(podQueue),
            Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
            StopEverything:  c.StopEverything,
            SchedulingQueue: podQueue,
        }, nil
    }

最前面是一段关于 `extender` 的代码，`extender` 是 `kubernetes` 自定义调度策略的一种扩展方式，侵入性小，无需修改 `scheduler` 代码，但是功能有限，目前只支持 `filter`，`priority`，`bind`，这里简单提一下不做详细介绍。

接下来申明了一个 `PodNominator`，`PodNominator` 是 `scheduler` 为了实现抢占调度定义的一种提名管理器，他记录了所有抢占成功但需要等待被抢占Pod退出的Pod。

    nominator := internalqueue.NewPodNominator(c.informerFactory.Core().V1().Pods().Lister())

在 `3.4` 中我们讲到 `scheduler` 可以通过定义多个 `profile` 丰富调度的方案，每一个 `profile` 都对应一个 `framework`，`framework` 是实际的调度框架，管理所有的插件，在预选和优选过程中会被调用。这里，我们通过 `NewMap()` 方法返回了一个 `map` 对象，`key` 对应 `SchedulerName`，`value` 为 `framework`。

    type Map map[string]framework.Framework

随后，我们初始化了一个优先级队列，这个队列就是调度队列，所有没有被调度的 `pod` 都会被放到这个队列中等待调度。

    podQueue := internalqueue.NewSchedulingQueue(...)

最后，我们声明了一个 `genericScheduler`，并作为 `Scheduler` 的 `Algorithm` 字段对应的值，最后返回 `scheduler` 对象。

    algo := NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		c.percentageOfNodesToScore,
	)

#### 3.4.7 注册 eventHandler
`addAllEventHandlers()` 方法主要用于注册如 `pod` 创建、删除、更新等对应的 `handler` 方法。

### 3.5 完成 Setup
至此，`scheduler` 初始化完成。

## 4. 运行 `Scheduler`
`scheduler` 初始化完成，就到了 `Run()` 方法，代码如下，我们一一分析：

    func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
        // Configz registration.
        if cz, err := configz.New("componentconfig"); err == nil {
            cz.Set(cc.ComponentConfig)
        }

        // Prepare the event broadcaster.
        cc.EventBroadcaster.StartRecordingToSink(ctx.Done())

        // Setup healthz checks.
        var checks []healthz.HealthChecker
        if cc.ComponentConfig.LeaderElection.LeaderElect {
            checks = append(checks, cc.LeaderElection.WatchDog)
        }

        waitingForLeader := make(chan struct{})
        isLeader := func() bool {
            select {
            case _, ok := <-waitingForLeader:
                // if channel is closed, we are leading
                return !ok
            default:
                // channel is open, we are waiting for a leader
                return false
            }
        }

        // Start up the healthz server.
        if cc.SecureServing != nil {
            handler := buildHandlerChain(newHealthzAndMetricsHandler(&cc.ComponentConfig, cc.InformerFactory, isLeader, checks...), cc.Authentication.Authenticator, cc.Authorization.Authorizer)
            // TODO: handle stoppedCh returned by c.SecureServing.Serve
            if _, err := cc.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
                // fail early for secure handlers, removing the old error loop from above
                return fmt.Errorf("failed to start secure server: %v", err)
            }
        }

        // Start all informers.
        cc.InformerFactory.Start(ctx.Done())
        // DynInformerFactory can be nil in tests.
        if cc.DynInformerFactory != nil {
            cc.DynInformerFactory.Start(ctx.Done())
        }

        // Wait for all caches to sync before scheduling.
        cc.InformerFactory.WaitForCacheSync(ctx.Done())
        // DynInformerFactory can be nil in tests.
        if cc.DynInformerFactory != nil {
            cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
        }

        // If leader election is enabled, runCommand via LeaderElector until done and exit.
        if cc.LeaderElection != nil {
            cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
                OnStartedLeading: func(ctx context.Context) {
                    close(waitingForLeader)
                    sched.Run(ctx)
                },
                OnStoppedLeading: func() {
                    select {
                    case <-ctx.Done():
                        // We were asked to terminate. Exit 0.
                        klog.InfoS("Requested to terminate, exiting")
                        os.Exit(0)
                    default:
                        // We lost the lock.
                        klog.Exitf("leaderelection lost")
                    }
                },
            }
            leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
            if err != nil {
                return fmt.Errorf("couldn't create leader elector: %v", err)
            }

            leaderElector.Run(ctx)

            return fmt.Errorf("lost lease")
        }

        // Leader election is disabled, so runCommand inline until done.
        close(waitingForLeader)
        sched.Run(ctx)
        return fmt.Errorf("finished without leader elect")
    }
### 4.1 注册 `ComponentConfig`
首先是生成 `ComponentConfig` 代码：

    if cz, err := configz.New("componentconfig"); err == nil {
        cz.Set(cc.ComponentConfig)
    }

`ComponentConfig` 结构体包含了 `scheduler server` 的所有配置信息，它支持通过 `yaml` 的方式进行配置，目前 `API` 版本为 `kubescheduler.config.k8s.io/v1beta3`，`GVK` 定义如下。


    apiVersion: kubescheduler.config.k8s.io/v1beta3
    kind: KubeSchedulerConfiguration

这里，我们会将前面 `Setup()` 函数返回的 `ComponentConfig` 对象（`cc.ComponentConfig`）注册到一个全局的变量 `config` 中去，它是 `map[string]*Config{}` 结构，一些其他组件的配置信息也会注册到这个变量中，像 `ControllerManager`，`kube-proxy`， `kubelet`，主要作用是用于对外输出 `JSON` 配置信息，路由地址为 `/configz`。

### 4.2 开启 Event 监听并上报

    cc.EventBroadcaster.StartRecordingToSink(ctx.Done())

### 4.3 启动 https 服务器
其中 `metrics`  服务也是在这里启动的。

	if cc.SecureServing != nil {
		handler := buildHandlerChain(newHealthzAndMetricsHandler(...), cc.Authentication.Authenticator, cc.Authorization.Authorizer)

		if _, err := cc.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start secure server: %v", err)
		}
	}

### 4.4 `informer` 启动
这里不仅启动了 `informer`，而且还需要等待 `informer cache` 从 `apiserver` 同步全量数据完成，因为 `informer` 中需要有全量的数据，只有这样才可以不需要请求 `apiserver` 就知道当前集群的状态。

    cc.InformerFactory.Start(ctx.Done())
	if cc.DynInformerFactory != nil {
		cc.DynInformerFactory.Start(ctx.Done())
	}

	cc.InformerFactory.WaitForCacheSync(ctx.Done())
	if cc.DynInformerFactory != nil {
		cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
	}

### 4.5 `scheduler leader` 选举
如果 `Scheduler` 开启了 `LeaderElection` 机制，则需要先选举出 `leader`，具体逻辑我们在 `scheduler` 高可用章节中会详细描述。

### 4.6 启动 `scheduler`
最后，我们启动 `scheduler` 实例，一共分为3步：

    sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()

#### 4.6.1 启动优先级队列：

    sched.SchedulingQueue.Run()

这个优先级队列的 `Run()` 方法启动了两个循环队列，负责将调度失败的和无法调度的 `pod` 按照一定策略重新加入到调度队列中。一个是 `BackoffQ`，存储在多个 `schedulingCycle` 中依旧调度失败的 `pod`，并且有 `backOff` 机制，每次重新调度的时间周期会逐渐拉长。第二个队列是 `UnschedulableQ`，存储由于资源不足无法调度的 `pod`。

    func (p *PriorityQueue) Run() {
        go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
        go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
    }

#### 4.6.2 运行调度逻辑

    wait.UntilWithContext(ctx, sched.scheduleOne, 0)

这里我们深入的看一下 `UntilWithContext()` 方法，这个方法层层调用，最后调用的函数是 `BackoffUntil(f func(), backoff BackoffManager, sliding bool, stopCh <-chan struct{})`，其中：

`f` 是 `sched.ScheduleOne` 方法，并且把 `UntilWithContext` 的 `ctx` 值作为 `ScheduleOne` 的参数传入

`backoff` 是一个 `jitteredBackoffManagerImpl` 实例，它的实际值为：

    &jitteredBackoffManagerImpl{
		clock:        &clock.RealClock{},
		duration:     0,
		jitter:       0.0,
		backoffTimer: nil,
	}

`sliding` 等于 `true`，而 `stopCh` 是 `UntilWithContext` 参数 `ctx` 的 `Done()` 方法对应的 `channel` 。

搞清楚了各个参数的输入值，我们看一下 `BackoffUntil` 方法，看他是如何调用调度逻辑的，该方法代码如下：

    func BackoffUntil(f func(), backoff BackoffManager, sliding bool, stopCh <-chan struct{}) {
        var t clock.Timer
        for {
            select {
            case <-stopCh:
                return
            default:
            }

            if !sliding {
                t = backoff.Backoff()
            }

            func() {
                defer runtime.HandleCrash()
                f()
            }()

            if sliding {
                t = backoff.Backoff()
            }

            select {
            case <-stopCh:
                if !t.Stop() {
                    <-t.C()
                }
                return
            case <-t.C():
            }
        }
    }

首先申明了一个定时器，用来定义循环间隔，它实际值为 `backoff.Backoff()`，实际是一个间隔为0的定时器，所以该循环会不间断的运行 `scheduleOne` 方法，`sliding` 则定义了定时器是否需要包含 `scheduleOne` 的执行时间，`true` 则表示不需要，否则需要。最后 `stopCh` 则控制了何时终止程序运行。

### 4.6.3 终止运行

    sched.SchedulingQueue.Close()

一旦 `context.Done()` 结束运行上下文，则调用 `Close()` 方法关闭优先级队列，退出进程，调度器整个生命周期结束。

## 5. 总结
以上就是 `scheduler` 运行的全部流程，我们看到里面涵盖的东西很多，一篇文章无法 `cover` 所有内容，所以这篇文章主要是把涉及到的一些重要流程做一个介绍，后面我将分多次就某一个流程的完整生命周期进行详解解读， 比如 `scheduler` 的队列机制、`plugins` 机制，以及 `informer` 是如何运行的，等等。

`敬请期待！`