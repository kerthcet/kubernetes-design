# Scheduler 调度框架

    Kubernetes Version: v1.23@4ade9f25
    Date: 2022.02.07

## 1. 开篇
调度框架是面向 Kubernetes 调度器的一种插件架构， 它定义了多个扩展点，插件注册后在一个或者多个扩展点被调用。整个调度周期可以分为两个阶段，分别是调度周期和绑定周期，官方文档见[这里](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/scheduling-framework/)。

下图显示了调度框架的扩展点:
![framework](../snapshots/scheduling-framework-extensions.png)
### 1.1 PreFilter
`PreFilter` 用于预处理 Pod 的相关信息，或者检查集群或 Pod 必须满足的某些条件。 如果 PreFilter 插件返回错误，则调度周期将终止。`PreFilterPlugin` 需要实现3个接口：
```golang
Name() string
PreFilter(ctx context.Context, state *CycleState, p *v1.Pod) *Status
PreFilterExtensions() PreFilterExtensions
```

### 1.2 Filter
`Filter` 用于过滤出不能运行该 Pod 的节点。对于每个节点， 调度器将按照其配置顺序调用这些过滤插件。如果任何过滤插件将节点标记为不可行， 则不会为该节点调用剩下的过滤插件。节点可以被同时进行评估。`Filter` 需要实现2个接口：
```golang
Name() string
Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
```

### 1.3 PreScore
`PreScore` 用于执行 “前置评分” 工作，即生成一个可共享状态供评分插件使用。 如果 PreScore 插件返回错误，则调度周期将终止。`PreScorePlugin` 需要实现3个接口：
```golang
Name() string
PreScore(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
```

### 1.4 Score
`Score` 用于对通过过滤阶段的节点进行排名。调度器将为每个节点调用每个评分插件。 将有一个定义明确的整数范围，代表最小和最大分数。 在标准化评分阶段之后，调度器将根据配置的插件权重 合并所有插件的节点分数。插件实现接口：
```golang
Name() string
Score(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (int64, *Status)
ScoreExtensions() ScoreExtensions
```

### 1.5 Normalize Score
`Normalize Score` 插件用于在调度器计算节点的排名之前修改分数。 在此扩展点注册的插件将使用同一插件的评分 结果被调用。 每个插件在每个调度周期调用一次。
`Normalize Score` 其实就是 score 插件实现的 `ScoreExtensions` 方法。

### 1.6 Reserve
Reserve 是一个信息性的扩展点。 管理运行时状态的插件（也称为“有状态插件”）应该使用此扩展点，以便 调度器在节点给指定 Pod 预留了资源时能够通知该插件。 这是在调度器真正将 Pod 绑定到节点之前发生的，并且它存在是为了防止 在调度器等待绑定成功时发生竞争情况。

### 1.7 UnReserve
这是个信息性的扩展点。 如果 Pod 被保留，然后在后面的阶段中被拒绝，则 Unreserve 插件将被通知。 Unreserve 插件应该清楚保留 Pod 的相关状态。

### 1.8 Permit
Permit 插件在每个 Pod 调度周期的最后调用，用于防止或延迟 Pod 的绑定。 一个允许插件可以做以下三件事之一：

1. 批准

	一旦所有 Permit 插件批准 Pod 后，该 Pod 将被发送以进行绑定。

2. 拒绝

	如果任何 Permit 插件拒绝 Pod，则该 Pod 将被返回到调度队列。 这将触发Unreserve 插件。

3. 等待（带有超时）

	如果一个 Permit 插件返回 “等待” 结果，则 Pod 将保持在一个内部的 “等待中” 的 Pod 列表，同时该 Pod 的绑定周期启动时即直接阻塞直到得到 批准。如果超时发生，等待 变成 拒绝，并且 Pod 将返回调度队列，从而触发 Unreserve 插件。

### 1.9 PreBind
PreBind 插件用于执行 Pod 绑定前所需的任何工作。 例如，一个预绑定插件可能需要提供网络卷并且在允许 Pod 运行在该节点之前 将其挂载到目标节点上。

如果任何 PreBind 插件返回错误，则 Pod 将被拒绝 并且 退回到调度队列中。

### 1.10 Bind
Bind 插件用于将 Pod 绑定到节点上。直到所有的 PreBind 插件都完成，Bind 插件才会被调用。 各绑定插件按照配置顺序被调用。绑定插件可以选择是否处理指定的 Pod。 如果绑定插件选择处理 Pod，剩余的绑定插件将被跳过。

### 1.11 PostBind
这是个信息性的扩展点。 绑定后插件在 Pod 成功绑定后被调用。这是绑定周期的结尾，可用于清理相关的资源。

下面，我们就从代码层面了解整个调度框架的工作机制。

## 2. 插件注册
首先是注册插件，我们以最新版本 `v1beta3` 为例，代码位于 `pkg/scheduler/apis/config/v1beta3/default_plugins.go`:
```golang
func getDefaultPlugins() *v1beta3.Plugins {
	plugins := &v1beta3.Plugins{
		MultiPoint: v1beta3.PluginSet{
			Enabled: []v1beta3.Plugin{
				{Name: names.PrioritySort},
				{Name: names.NodeUnschedulable},
				{Name: names.NodeName},
				{Name: names.TaintToleration, Weight: pointer.Int32(3)},
				{Name: names.NodeAffinity, Weight: pointer.Int32(2)},
				{Name: names.NodePorts},
				{Name: names.NodeResourcesFit, Weight: pointer.Int32(1)},
				{Name: names.VolumeRestrictions},
				{Name: names.EBSLimits},
				{Name: names.GCEPDLimits},
				{Name: names.NodeVolumeLimits},
				{Name: names.AzureDiskLimits},
				{Name: names.VolumeBinding},
				{Name: names.VolumeZone},
				{Name: names.PodTopologySpread, Weight: pointer.Int32(2)},
				{Name: names.InterPodAffinity, Weight: pointer.Int32(2)},
				{Name: names.DefaultPreemption},
				{Name: names.NodeResourcesBalancedAllocation, Weight: pointer.Int32(1)},
				{Name: names.ImageLocality, Weight: pointer.Int32(1)},
				{Name: names.DefaultBinder},
			},
		},
	}
	applyFeatureGates(plugins)

	return plugins
}
```

`MultiPoint` 定义了默认启用的 plugins，减少了用户心智，不需要在每个扩展点单独设置，但是也有其他问题，比如不清楚这个 plugin 到底实现了哪个扩展点。

`getDefaultPlugins` 方法何时调用，我们需要追溯到 `RegisterDefaults`，该方法会在初始化的时候调用，完成一些默认方法的注册，所谓默认方法，最常见的就是补充默认参数，`RegisterDefaults` 代码如下：
```golang
func RegisterDefaults(scheme *runtime.Scheme) error {
    // ...
	scheme.AddTypeDefaultingFunc(&v1beta3.InterPodAffinityArgs{}, func(obj interface{}) { SetObjectDefaults_InterPodAffinityArgs(obj.(*v1beta3.InterPodAffinityArgs)) })
	scheme.AddTypeDefaultingFunc(&v1beta3.KubeSchedulerConfiguration{}, func(obj interface{}) {
		SetObjectDefaults_KubeSchedulerConfiguration(obj.(*v1beta3.KubeSchedulerConfiguration))
	})
    // ...
	return nil
}
```

其中 `AddTypeDefaultingFunc`，会将默认方法存到 `map[reflect.Type]func(interface{})` 中：
```golang
func (s *Scheme) AddTypeDefaultingFunc(srcType Object, fn func(interface{})) {
	s.defaulterFuncs[reflect.TypeOf(srcType)] = fn
}
```
`RegisterDefaults` 方法中注册了 `SetObjectDefaults_KubeSchedulerConfiguration` 方法，该方法会完成 `KubeSchedulerConfiguration` 一些默认参数的配置工作，其中通过调用 `setDefaults_KubeSchedulerProfile` 方法完成 `getDefaultPlugins` 的调用
```golang
func setDefaults_KubeSchedulerProfile(prof *v1beta3.KubeSchedulerProfile) {
	// Set default plugins.
	prof.Plugins = mergePlugins(getDefaultPlugins(), prof.Plugins)
    // ...
}
```

我们看到该方法会将默认配置的 plugin 和我们在 profile 中自定义的 plugin 配置进行 merge 操作，完成最终的 plugin 设置。

注册完成后，何时调用该方法呢？我们在 scheduler server `Setup` 的时候会调用 `Default` 方法，获得 config 配置:
```golang
func Default() (*config.KubeSchedulerConfiguration, error) {
	versionedCfg := v1beta3.KubeSchedulerConfiguration{}
	versionedCfg.DebuggingConfiguration = *v1alpha1.NewRecommendedDebuggingConfiguration()

	scheme.Scheme.Default(&versionedCfg)
    // ...
}
```

另外，对于通过使用 `config file` 进行初始化的情况，也会调用下面的方法进行初始化：
```golang
cfg, err := loadConfigFromFile(o.ConfigFile)
```

scheme 通过调用 `Default()`方法，完成所有默认方法的调用，我们看一下 scheme 的 `Default` 方法，其实就是调用之前 map 中存储的方法：
```golang
func (s *Scheme) Default(src Object) {
	if fn, ok := s.defaulterFuncs[reflect.TypeOf(src)]; ok {
		fn(src)
	}
}
```

至此，我们明白了整个插件的注册及初始化流程。
## 3. 调度周期
调度周期可以分为两个阶段，也就是我们常说的预选和优选阶段。代码入口位于 `pkg/scheduler/scheduler.go:L455`:
```golang
scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, sched.Extenders, fwk, state, pod)
```

### 3.1 预选阶段
预选阶段代码位于 `pkg/scheduler/generic_scheduler.go:L213` ，我们一起看一下：
```golang
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, extenders []framework.Extender, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {
    // diagnosis 用于记录调度失败相关信息，方便后期进行分析
	diagnosis := framework.Diagnosis{
		NodeToStatusMap:      make(framework.NodeToStatusMap),
		UnschedulablePlugins: sets.NewString(),
	}

	// PreFilter 扩展点
	s := fwk.RunPreFilterPlugins(ctx, state, pod)

    // 状态返回失败
	if !s.IsSuccess() {
        // 状态返回失败并且不是不可调度错误，直接返回错误结果（不可调度错误一般是因为资源不够用，可以等待进行新一轮调度，否则就是其他错误，重新调度也是出错，所以可以直接返回）
		if !s.IsUnschedulable() {
			return nil, diagnosis, s.AsError()
		}

        // 记录节点状态
		for _, n := range allNodes {
			diagnosis.NodeToStatusMap[n.Node().Name] = s
		}

        // 记录失败插件信息
		diagnosis.UnschedulablePlugins.Insert(s.FailedPlugin())
		return nil, diagnosis, nil
	}

    // ... 抢占逻辑

    // Filter 扩展点
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, allNodes)
	if err != nil {
		return nil, diagnosis, err
	}

    // ... extender 逻辑，这里不展开

	return feasibleNodes, diagnosis, nil
}
```

接下来，我们重点看一下 `RunPreFilterPlugins` 和 `findNodesThatPassFilters` 方法。

`RunPreFilterPlugins` 位于 `pkg/scheduler/framework/runtime/framework.go`:
```golang
func (f *frameworkImpl) RunPreFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (status *framework.Status) {
    // 遍历所有 PreFilterPlugins
	for _, pl := range f.preFilterPlugins {
        // 执行每个 plugin 的 PreFilter 方法
		status = f.runPreFilterPlugin(ctx, pl, state, pod)
        // 如果返回状态失败，则直接返回，终止调度周期
		// 如果是不可调度原因，记录失败的插件名称，并直接返回 status, kubernetes 会将它包装成 FitError，FitError 会参与后续的 pod 重调度
		if !status.IsSuccess() {
			status.SetFailedPlugin(pl.Name())
			if status.IsUnschedulable() {
				return status
			}
			return framework.AsStatus(fmt.Errorf("running PreFilter plugin %q: %w", pl.Name(), status.AsError())).WithFailedPlugin(pl.Name())
		}
	}

	return nil
}
```
`RunPreFilterPlugins` 方法主要负责遍历所有的 PreFilter plugins 并进行一些预处理，如果有任何一个插件返回状态失败则终止调度。

`findNodesThatPassFilters` 方法位于 `pkg/scheduler/generic_scheduler.go`:
```golang
func (g *genericScheduler) findNodesThatPassFilters(
	ctx context.Context,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	diagnosis framework.Diagnosis,
	nodes []*framework.NodeInfo) ([]*v1.Node, error) {
    // ...

	checkNode := func(i int) {
		// ... 找到 nodeInfo

        // 执行 Filter 方法
		status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
        // 如果出错，则发送错误到 channel，并立马执行 cancel() 结束并发
		if status.Code() == framework.Error {
			errCh.SendErrorWithCancel(status.AsError(), cancel)
			return
		}

        // 如果成功，则添加到 feasibleNodes 中
		if status.IsSuccess() {
			length := atomic.AddInt32(&feasibleNodesLen, 1)
			// 如果查找的节点数达到要求，则立即终止 Filter 阶段
			if length > numNodesToFind {
				cancel()
				atomic.AddInt32(&feasibleNodesLen, -1)
			} else {
				feasibleNodes[length-1] = nodeInfo.Node()
			}
        // 如果错误，则记录调度失败相关信息
		} else {
			statusesLock.Lock()
			diagnosis.NodeToStatusMap[nodeInfo.Node().Name] = status
			diagnosis.UnschedulablePlugins.Insert(status.FailedPlugin())
			statusesLock.Unlock()
		}
	}

    // ...

    // 针对所有节点并发执行
	fwk.Parallelizer().Until(ctx, len(nodes), checkNode)

	// ...

    // 返回合适的节点列表
	return feasibleNodes, nil
}
```

这里我们需要对 `RunFilterPluginsWithNominatedPods` 方法再深入展开一下:
```golang
func (f *frameworkImpl) RunFilterPluginsWithNominatedPods(ctx context.Context, state *framework.CycleState, pod *v1.Pod, info *framework.NodeInfo) *framework.Status {
	var status *framework.Status

	podsAdded := false
	// 这里执行了两次 Filter
	// 第一次将节点上完成抢占但未调度成功，我们称之为 nominatedPod 加入到 state 进行 Filter 流程，
	// 之所以这么做原因是不希望本轮调度会让之前抢占成功的 pod 再次被调度的时候由于如资源不足等原因被拒绝
	// 第二次则不考虑该情况有走了一遍 Filter 流程是担心之前的 nominatedPod 会对 Filter 结果造成影响，尤其是一些亲和性相关的 FilterPlugin
	for i := 0; i < 2; i++ {
		stateToUse := state
		nodeInfoToUse := info
		if i == 0 {
			var err error
			podsAdded, stateToUse, nodeInfoToUse, err = addNominatedPods(ctx, f, pod, state, info)
			if err != nil {
				return framework.AsStatus(err)
			}
		} else if !podsAdded || !status.IsSuccess() {
			break
		}

		// RunFilterPlugins 会遍历每一个 plugin，返回一个 map。默认情况下，只要有一个 plugin 调度失败，就直接终止调度周期，除非设置 runAllFilters = true。
		statusMap := f.RunFilterPlugins(ctx, stateToUse, pod, nodeInfoToUse)
		// 这里对status进行了一次 merge 操作
		status = statusMap.Merge()
		if !status.IsSuccess() && !status.IsUnschedulable() {
			return status
		}
	}

	return status
}
```

### 3.2 优选阶段
优选代码位于 `pkg/scheduler/generic_scheduler.go:L396` 代码如下：
```golang
func prioritizeNodes(
	ctx context.Context,
	extenders []framework.Extender,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (framework.NodeScoreList, error) {
	// ...

	// PreScore 阶段
	preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		return nil, preScoreStatus.AsError()
	}

	// Score 阶段，返回score
	scoresMap, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	// result 用来记录所有的得分情况
	result := make(framework.NodeScoreList, 0, len(nodes))

	// ... 简单汇总所有的得分，其中涉及到 framework.Extender 逻辑，这里不展开，后面会有单独的文章

	return result, nil
}
```

我们继续看一下 `RunPreScorePlugins` 和 `RunScorePlugins` 两个方法：
```golang
func (f *frameworkImpl) RunPreScorePlugins(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (status *framework.Status) {
	// ...

	// 遍历 PreScore Plugins，执行 PreScore 方法
	for _, pl := range f.preScorePlugins {
		status = f.runPreScorePlugin(ctx, pl, state, pod, nodes)
		if !status.IsSuccess() {
			return framework.AsStatus(fmt.Errorf("running PreScore plugin %q: %w", pl.Name(), status.AsError()))
		}
	}

	return nil
}
```

`RunPreScorePlugins` 方法很简单，就是遍历各个 `PreScorePlugin` 进行前置的一些 state 计算，为后面的 `Score` 阶段使用。我们再看 `RunScorePlugins`:
```golang
func (f *frameworkImpl) RunScorePlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*v1.Node) (ps framework.PluginToNodeScores, status *framework.Status) {
	// ...

	// 申明 map 用于存储插件得分
	pluginToNodeScores := make(framework.PluginToNodeScores, len(f.scorePlugins))
	for _, pl := range f.scorePlugins {
		pluginToNodeScores[pl.Name()] = make(framework.NodeScoreList, len(nodes))
	}
	ctx, cancel := context.WithCancel(ctx)
	errCh := parallelize.NewErrorChannel()

	// 遍历所有的 scorePlugin 并发执行 Score 方法
	f.Parallelizer().Until(ctx, len(nodes), func(index int) {
		for _, pl := range f.scorePlugins {
			nodeName := nodes[index].Name
			s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
			// 如果 score() 失败，则终止调度周期
			if !status.IsSuccess() {
				err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			// 如果 score() 成功，则记录该 plugin 对应的分数
			pluginToNodeScores[pl.Name()][index] = framework.NodeScore{
				Name:  nodeName,
				Score: s,
			}
		}
	})

	// 如果 score() 发生错误，则终止
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("running Score plugins: %w", err))
	}

	// 为了避免得分超过基准，根据最大和最小得分进行标准化操作
	f.Parallelizer().Until(ctx, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		nodeScoreList := pluginToNodeScores[pl.Name()]
		if pl.ScoreExtensions() == nil {
			return
		}
		status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
		if !status.IsSuccess() {
			err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
	})

	// 同样如果发生错误，则终止
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("running Normalize on Score plugins: %w", err))
	}

	// 根据各个插件的权重计算最终得分
	f.Parallelizer().Until(ctx, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		// Score plugins' weight has been checked when they are initialized.
		weight := f.scorePluginWeight[pl.Name()]
		nodeScoreList := pluginToNodeScores[pl.Name()]

		for i, nodeScore := range nodeScoreList {
			// return error if score plugin returns invalid score.
			if nodeScore.Score > framework.MaxNodeScore || nodeScore.Score < framework.MinNodeScore {
				err := fmt.Errorf("plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), nodeScore.Score, framework.MinNodeScore, framework.MaxNodeScore)
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			nodeScoreList[i].Score = nodeScore.Score * int64(weight)
		}
	})
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("applying score defaultWeights on Score plugins: %w", err))
	}

	// 返回最后的得分情况
	return pluginToNodeScores, nil
}
```
优选阶段到这还没有结束，最后还需要通过 `selectHost` 从所有的节点中选出得分最高的节点作为调度节点，方法位于 `pkg/scheduler/generic_scheduler.go:144`，方法比较简单，这里就不展开了。

### 3.3 抢占阶段
调度结束，如果调度失败就会进入到我们的[抢占流程](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/priority-preemption.md)，由于之前的文章已经介绍过，这里就不介绍了，点击跳转可以看到关于抢占调度相关文章。

### 3.4 Reserve 阶段
代码位于 `pkg/scheduler/scheduler.go:L507`：
```golang
assumedPodInfo := podInfo.DeepCopy()
assumedPod := assumedPodInfo.Pod

// 将选出的节点绑定到 pod.Spec.NodeName
err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
if err != nil {
	// 如果出错，则进行错误处理
	sched.handleSchedulingFailure(fwk, assumedPodInfo, err, SchedulerError, clearNominatedNode)
	return
}

// 执行 Reserve Plugin
if sts := fwk.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
	// 如果执行出错，则进行回滚操作，执行 UnreservePlugins，这里需要注意的是 Unreserve 方法应该是幂等的
	fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)

	// 从 cache 中移除该 pod，我们在 assume 方法中会将 pod 信息存到 cache 中
	if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
		klog.ErrorS(forgetErr, "Scheduler cache ForgetPod failed")
	}
	// 出错时进行错误处理
	sched.handleSchedulingFailure(fwk, assumedPodInfo, sts.AsError(), SchedulerError, clearNominatedNode)
	return
}
```

我们再看一下 `Reserve` 阶段两个方法，`assume` 和 `RunReservePluginsReserve`，`assume` 方法如下：
```golang
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
	// 将节点绑定到 pod 上
	assumed.Spec.NodeName = host

	// AssumePod 会将 pod 信息存到 SchedulerCache 中
	if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
		klog.ErrorS(err, "Scheduler cache AssumePod failed")
		return err
	}
	if sched.SchedulingQueue != nil {
		// 如果 pod 是抢占提名的 pod，则将它移除，因为已经调度成功了，不需要再抢占
		sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
	}

	return nil
}
```

再看 `RunReservePluginsReserve` 方法，该方法位于 `pkg/scheduler/framework/runtime/framework.go:L1044`，其实逻辑也很简单：
```golang
func (f *frameworkImpl) RunReservePluginsReserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
	// ...

	// 遍历所有的 reservePlugins
	for _, pl := range f.reservePlugins {
		// 执行接口方法 runReservePluginReserve
		status = f.runReservePluginReserve(ctx, pl, state, pod, nodeName)
		// 如果失败，则直接返回失败结果，后续插件不再执行
		if !status.IsSuccess() {
			err := status.AsError()
			return framework.AsStatus(fmt.Errorf("running Reserve plugin %q: %w", pl.Name(), err))
		}
	}
	return nil
}
```

### 3.5 Permit

```golang
	// Run "permit" plugins.
	runPermitStatus := fwk.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	// 如果返回拒绝或者错误（既不是等待也不是成功），则走 Unreserve 流程
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		// 输出对应错误原因
		var reason string
		if runPermitStatus.IsUnschedulable() {
			reason = v1.PodReasonUnschedulable
		} else {
			reason = SchedulerError
		}

		// 顺序执行 Unreserve plugin
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)

		// 将 pod 从 cache 中移除
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.ErrorS(forgetErr, "Scheduler cache ForgetPod failed")
		}
		// 最后进行错误处理并返回
		sched.handleSchedulingFailure(fwk, assumedPodInfo, runPermitStatus.AsError(), reason, clearNominatedNode)
		return
	}

	// ...
```
Permit 阶段会返回3种状态，成功，拒绝，等待。成功就会走到下一个流程，拒绝刚才我们已经分析过了，那等待呢？我们会在下面的绑定周期中继续介绍。

## 4. 绑定周期
调度周期结束，就来到绑定周期，绑定阶段也有多个扩展点，分别是 `PreBind`，`Bind`，`PostBind`。绑定阶段的代码是通过 `go func()` 异步进行的。在讲这几个扩展点之前，我们先看一下 `Permit` 阶段的等待情况：

### 4.1 Permit 收尾阶段
```golang
bindingCycleCtx, cancel := context.WithCancel(ctx)
defer cancel()

// 对于 Permit 阶段需要等待的 Pod，一直等待直到返回或者超时
waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
if !waitOnPermitStatus.IsSuccess() {
	// 输出原因
	var reason string
	if waitOnPermitStatus.IsUnschedulable() {
		reason = v1.PodReasonUnschedulable
	} else {
		reason = SchedulerError
	}

	// 执行 Unreserve Plugins
	fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	// 将 pod 从 cache 中移除
	if forgetErr := sched.Cache.ForgetPod(assumedPod); forgetErr != nil {
		klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
	}

	// 处理错误情况
	sched.handleSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, clearNominatedNode)
	return
}
```

### 4.2 PreBind 阶段
当 Permit 成功，就进入到 `PreBind` 阶段：
```golang
		// 执行 PreBind Plugins
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)

		// 如果执行失败
		if !preBindStatus.IsSuccess() {
			// 触发执行 Unreserve Plugins
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)

			// 从 cache 中移除 pod
			if forgetErr := sched.Cache.ForgetPod(assumedPod); forgetErr != nil {
				klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
			}

			// 处理错误情况
			sched.handleSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, clearNominatedNode)
			return
		}
```

### 4.3 Bind/PostBind 阶段
最后执行 `Bind` 和 `PostBInd`，我们一起看一下：
```golang
// 执行 bind 流程
err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
if err != nil {
	// 如果失败执行 Unreserve Plugins
	fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	// 从 cache 中移除 pod
	if err := sched.Cache.ForgetPod(assumedPod); err != nil {
		klog.ErrorS(err, "scheduler cache ForgetPod failed")
	}
	// 处理错误情况
	sched.handleSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, clearNominatedNode)
} else {
	// 如果 bind 成功，则进行 PostBind 阶段
	fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)

	// ...
}
```
我们深入的看一下 `bind` 方法：
```golang
func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed *v1.Pod, targetNode string, state *framework.CycleState) (err error) {
	defer func() {
		// Binding 收尾工作，主要是将 cache 中 pod state 标记为结束绑定，并为它添加一个过期时间，后台会定期清理过期的 pod
		sched.finishBinding(fwk, assumed, targetNode, err)
	}()

	// extender binding

	// 执行 Bind Plugins，如果成功，则直接返回
	bindStatus := fwk.RunBindPlugins(ctx, state, assumed, targetNode)
	if bindStatus.IsSuccess() {
		return nil
	}
	// 如果失败，返回错误信息
	if bindStatus.Code() == framework.Error {
		return bindStatus.AsError()
	}
}
```
至此，绑定周期结束，也意味着整个调度流程结束了。

## 5. 总结
我们看到 `scheduling framework` 通过定义好扩展点接口的方式，使得整个调度流程看起来十分清晰，同时也易于扩展，不管是对于开发者或者终端用户而言，都非常容易集成新的功能到 `framework` 中。另外，这也是一个很好的工程示例，值得认真研究和学习。