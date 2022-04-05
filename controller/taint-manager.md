# 节点生命周期管理之 TaintManager

    Kubernetes Version: v1.23@8e0ac5b6
    Date: 2022.04.03

## 1. 开篇
Kubernetes 污点机制大家应该都是非常熟悉的了，其中污点有3种功效，分别是 `NoSchedule`，`NoExecute`，`PreferNoSchedule`，用于禁止调度，禁止运行和优先禁止调度等场景。今天我们讲一讲 `NoExecute` 是如何工作的。

## 2. 整体设计
负责管理 `NoExecute` 污点的是一个叫 `NoExecuteTaintManager` 的管理器，它属于 `NodeLifecycleController` 中的一个组件，随着 `NodeLifecycleController` 运行而运行。运行时，监听 node 和 pod 事件更新，触发对应的驱逐逻辑。

## 3. NewNoExecuteTaintManager 组件
`NoExecuteTaintManager` 数据结构如下：
```golang
type NoExecuteTaintManager struct {
	client                clientset.Interface
    // 负责记录事件
	recorder              record.EventRecorder
    // 监听 pod
	getPod                GetPodFunc
    // 监听 node
	getNode               GetNodeFunc
    // 获取 node 所有的 pods
	getPodsAssignedToNode GetPodsByNodeNameFunc

    // 驱逐队列
	taintEvictionQueue *TimedWorkerQueue
	taintedNodesLock sync.Mutex
	taintedNodes     map[string][]v1.Taint

    // node 更新 channel
	nodeUpdateChannels []chan nodeUpdateItem
    // pod 更新 channel
	podUpdateChannels  []chan podUpdateItem

    // node 更新队列
	nodeUpdateQueue workqueue.Interface
    // pod 更新队列
	podUpdateQueue  workqueue.Interface
}
```

实例初始化位于 `pkg/controller/nodelifecycle/scheduler/taint_manager.go:L158`，在 `NodeLifecycleController` 初始化的时候调用：

```golang
func NewNoExecuteTaintManager(ctx context.Context, c clientset.Interface, getPod GetPodFunc, getNode GetNodeFunc, getPodsAssignedToNode GetPodsByNodeNameFunc) *NoExecuteTaintManager {
    // 初始化 event 相关组件
	eventBroadcaster := record.NewBroadcaster()
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "taint-controller"})
	eventBroadcaster.StartStructuredLogging(0)
	if c != nil {
		klog.V(0).InfoS("Sending events to api server")
		eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: c.CoreV1().Events("")})
	} else {
		klog.Fatalf("kubeClient is nil when starting NodeController")
	}

    // 构造 NoExecuteTaintManager
	tm := &NoExecuteTaintManager{
		client:                c,
		recorder:              recorder,
		getPod:                getPod,
		getNode:               getNode,
		getPodsAssignedToNode: getPodsAssignedToNode,
		taintedNodes:          make(map[string][]v1.Taint),

        // 初始化 node 队列
		nodeUpdateQueue: workqueue.NewNamed("noexec_taint_node"),
        // 初始化 pod 队列
		podUpdateQueue:  workqueue.NewNamed("noexec_taint_pod"),
	}
    // 构造驱逐队列
	tm.taintEvictionQueue = CreateWorkerQueue(deletePodHandler(c, tm.emitPodDeletionEvent))

	return tm
}
```

整个 New 方法中最关键的是 `CreateWorkerQueue(deletePodHandler(c, tm.emitPodDeletionEvent))` 这个方法，我们展开看一下：

`CreateWorkerQueue` 方法其实就是构造了一个 `TimedWorkerQueue` 队列：
```golang
func CreateWorkerQueue(f func(ctx context.Context, args *WorkArgs) error) *TimedWorkerQueue {
	return &TimedWorkerQueue{
		workers:  make(map[string]*TimedWorker),
		workFunc: f,
		clock:    clock.RealClock{},
	}
}
```

其中，`clock.RealClock` 是一个实现了 `WithTicker` 接口的数据结构，可以实现定时执行的功能，这样就可以实现定时驱逐 pod。

另外我们观察到 `CreateWorkerQueue` 的参数是一个闭包，我们先看 `deletePodHandler` 方法：
```golang
func deletePodHandler(c clientset.Interface, emitEventFunc func(types.NamespacedName)) func(ctx context.Context, args *WorkArgs) error {
    // 返回实际的方法
	return func(ctx context.Context, args *WorkArgs) error {
		ns := args.NamespacedName.Namespace
		name := args.NamespacedName.Name
        // 这里的 NamespaceName.String() 返回 <namespace>/<pod> 这种格式的字符串
		klog.V(0).InfoS("NoExecuteTaintManager is deleting pod", "pod", args.NamespacedName.String())
		if emitEventFunc != nil {
			emitEventFunc(args.NamespacedName)
		}
		var err error
        // 定义了重拾机制（如果可以指数退避就更好了，后面考虑提交个PR）
		for i := 0; i < retries; i++ {
			err = c.CoreV1().Pods(ns).Delete(ctx, name, metav1.DeleteOptions{})
			if err == nil {
				break
			}
			time.Sleep(10 * time.Millisecond)
		}
		return err
	}
}
```

 该方法主要是负责执行删除 pod 的操作。再看 `emitPodDeletionEvent` 方法，主要负责输出事件：
 ```golang
 func (tc *NoExecuteTaintManager) emitPodDeletionEvent(nsName types.NamespacedName) {
	if tc.recorder == nil {
		return
	}
	ref := &v1.ObjectReference{
		Kind:      "Pod",
		Name:      nsName.Name,
		Namespace: nsName.Namespace,
	}
	tc.recorder.Eventf(ref, v1.EventTypeNormal, "TaintManagerEviction", "Marking for deletion Pod %s", nsName.String())
}
```
这样，`NewNoExecuteTaintManager` 就结束了。

## 4. Run 方法
看完了初始化方法，我们看一下 `NoExecuteTaintManager` 是如何执行的，代码位于同一个文件夹下：
```golang
func (tc *NoExecuteTaintManager) Run(ctx context.Context) {
	klog.V(0).InfoS("Starting NoExecuteTaintManager")

    // UpdateWorkerSize 表示 worker 数量，目前是8个
	for i := 0; i < UpdateWorkerSize; i++ {
		tc.nodeUpdateChannels = append(tc.nodeUpdateChannels, make(chan nodeUpdateItem, NodeUpdateChannelSize))
		tc.podUpdateChannels = append(tc.podUpdateChannels, make(chan podUpdateItem, podUpdateChannelSize))
	}
	go func(stopCh <-chan struct{}) {
		for {
            // 监听 nodeUpdateQueue，如果有新的元素加入队列，则从中获取该元素，这是一个 block 队列
			item, shutdown := tc.nodeUpdateQueue.Get()
            // 如果队列关闭，则退出该 goroutine
			if shutdown {
				break
			}
			nodeUpdate := item.(nodeUpdateItem)
            // 计算队列的 index
			hash := hash(nodeUpdate.nodeName, UpdateWorkerSize)
			select {
            // 监听 stop 信号
			case <-stopCh:
                // Done 方法表示 item 处理完成
				tc.nodeUpdateQueue.Done(item)
				return
            // 将 nodeUpdate 放到 channel中
			case tc.nodeUpdateChannels[hash] <- nodeUpdate:
				// tc.nodeUpdateQueue.Done is called by the nodeUpdateChannels worker
			}
		}
	}(ctx.Done())

    // 原理同 nodeUpdate 一模一样，只不过这里监听的是 podUpdate
	go func(stopCh <-chan struct{}) {
		for {
			item, shutdown := tc.podUpdateQueue.Get()
			if shutdown {
				break
			}
			podUpdate := item.(podUpdateItem)
			hash := hash(podUpdate.nodeName, UpdateWorkerSize)
			select {
			case <-stopCh:
				tc.podUpdateQueue.Done(item)
				return
			case tc.podUpdateChannels[hash] <- podUpdate:
				// tc.podUpdateQueue.Done is called by the podUpdateChannels worker
			}
		}
	}(ctx.Done())

	wg := sync.WaitGroup{}
	wg.Add(UpdateWorkerSize)
    // 前面我们监听了 nodeUpdate 和 podUpdate 事件，放到了对应的 channel 中，后续的处理则会由 worker() 负责。这里我们同样声明了 UpdateWorkerSize 个 worker
	for i := 0; i < UpdateWorkerSize; i++ {
		go tc.worker(ctx, i, wg.Done, ctx.Done())
	}
	wg.Wait()
}
```

我们再看一下 worker 方法：
```golang
func (tc *NoExecuteTaintManager) worker(ctx context.Context, worker int, done func(), stopCh <-chan struct{}) {
	defer done()

	for {
		select {
		case <-stopCh:
			return
        // 监听 nodeUpdateChannels，并进行处理
		case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
			tc.handleNodeUpdate(ctx, nodeUpdate)
			tc.nodeUpdateQueue.Done(nodeUpdate)
		case podUpdate := <-tc.podUpdateChannels[worker]:
			// If we found a Pod update we need to empty Node queue first.
		priority:
			for {
				select {
                // 优先处理 nodeUpdateChannel，一个是因为 node 优先级更高，另外，nodeUpdate 中也会进行 pod 的处理
				case nodeUpdate := <-tc.nodeUpdateChannels[worker]:
					tc.handleNodeUpdate(ctx, nodeUpdate)
					tc.nodeUpdateQueue.Done(nodeUpdate)
				default:
					break priority
				}
			}
			// nodeUpdateChannels 被清空后开始处理 podUpdate
			tc.handlePodUpdate(ctx, podUpdate)
			tc.podUpdateQueue.Done(podUpdate)
		}
	}
}
```

## 5. 驱逐逻辑
通过上面的 Run 方法和 worker() 方法的解读，我们知道了大体的处理逻辑，下面我们看一下具体的驱逐逻辑，代码实现位于上文提到的 `handleNodeUpdate` 和 `handlePodUpdate` 方法中。我们先看 `handleNodeUpdate` 方法：
```golang
func (tc *NoExecuteTaintManager) handleNodeUpdate(ctx context.Context, nodeUpdate nodeUpdateItem) {
    // 获取 node，其实就是调用的 NoExecuteTaintManager 的 getNode 方法
	node, err := tc.getNode(nodeUpdate.nodeName)
	if err != nil {
		if apierrors.IsNotFound(err) {
			// 如果没有找到该 node，需要从 taintedNodes 中删除
			tc.taintedNodesLock.Lock()
			defer tc.taintedNodesLock.Unlock()
			delete(tc.taintedNodes, nodeUpdate.nodeName)
			return
		}
        // 错误处理
		utilruntime.HandleError(fmt.Errorf("cannot get node %s: %v", nodeUpdate.nodeName, err))
		return
	}

	// Create or Update
	klog.V(4).InfoS("Noticed node update", "node", nodeUpdate)
    // 获取 node 所有的污点
	taints := getNoExecuteTaints(node.Spec.Taints)
	func() {
		tc.taintedNodesLock.Lock()
		defer tc.taintedNodesLock.Unlock()
		klog.V(4).InfoS("Updating known taints on node", "node", node.Name, "taints", taints)
		if len(taints) == 0 {
            // 如果没有污点，则从 taintNodes 中删除 node
			delete(tc.taintedNodes, node.Name)
		} else {
            // 否则，就更新 node 污点信息
			tc.taintedNodes[node.Name] = taints
		}
	}()

    // 获取 node 所有的 pod 信息
	pods, err := tc.getPodsAssignedToNode(node.Name)
	if err != nil {
		klog.ErrorS(err, "Failed to get pods assigned to node", "node", node.Name)
		return
	}
	if len(pods) == 0 {
		return
	}

	// 如果节点没有 taints，则取消 node 上 pod 的驱逐操作
	if len(taints) == 0 {
		klog.V(4).InfoS("All taints were removed from the node. Cancelling all evictions...", "node", node.Name)
		for i := range pods {
			tc.cancelWorkWithEvent(types.NamespacedName{Namespace: pods[i].Namespace, Name: pods[i].Name})
		}
		return
	}

	now := time.Now()
    // 否则，对每一个 pod 执行 processPodOnNode 逻辑
	for _, pod := range pods {
		podNamespacedName := types.NamespacedName{Namespace: pod.Namespace, Name: pod.Name}
		tc.processPodOnNode(ctx, podNamespacedName, node.Name, pod.Spec.Tolerations, taints, now)
	}
}
```
我们再看下 `processPodOnNode` 方法：
```golang
func (tc *NoExecuteTaintManager) processPodOnNode(
	ctx context.Context,
	podNamespacedName types.NamespacedName,
	nodeName string,
	tolerations []v1.Toleration,
	taints []v1.Taint,
	now time.Time,
) {
    // 又做了一次判断，如果污点为空，则执行取消 work
	if len(taints) == 0 {
		tc.cancelWorkWithEvent(podNamespacedName)
	}

    // 返回 pod 的 tolerations
	allTolerated, usedTolerations := v1helper.GetMatchingTolerations(taints, tolerations)
    // 如果 pod 没有完全容忍污点，则需要将 pod 立马进行驱逐
	if !allTolerated {
		klog.V(2).InfoS("Not all taints are tolerated after update for pod on node", "pod", podNamespacedName.String(), "node", klog.KRef("", nodeName))
        // 先从等待队列中驱逐，再立马执行
		tc.cancelWorkWithEvent(podNamespacedName)
        // 传入两个 time.Now 会立即执行驱逐逻辑，后面讲 AddWork 逻辑时会提到
		tc.taintEvictionQueue.AddWork(ctx, NewWorkArgs(podNamespacedName.Name, podNamespacedName.Namespace), time.Now(), time.Now())
		return
	}

    // 获取所有 tolerations 中最短的时间，驱逐时间以最短的时间为准
	minTolerationTime := getMinTolerationTime(usedTolerations)
    // 如果是负数，表示永远容忍，则不驱逐
	if minTolerationTime < 0 {
		klog.V(4).InfoS("Current tolerations for pod tolerate forever, cancelling any scheduled deletion", "pod", podNamespacedName.String())
		tc.cancelWorkWithEvent(podNamespacedName)
		return
	}

	startTime := now
	triggerTime := startTime.Add(minTolerationTime)
    // 这里对已经在驱逐队列中的元素进行了二次判断，表示是否需要延长或者缩短驱逐时间，但是存在 bug，具体可以看我提交的 PR: https://github.com/kubernetes/kubernetes/pull/109226，我就不再讲述逻辑
	scheduledEviction := tc.taintEvictionQueue.GetWorkerUnsafe(podNamespacedName.String())
	if scheduledEviction != nil {
		startTime = scheduledEviction.CreatedAt
		if startTime.Add(minTolerationTime).Before(triggerTime) {
			return
		}
		tc.cancelWorkWithEvent(podNamespacedName)
	}
    // 最后，将需要驱逐的 pod 加入队列中，等待驱逐
	tc.taintEvictionQueue.AddWork(ctx, NewWorkArgs(podNamespacedName.Name, podNamespacedName.Namespace), startTime, triggerTime)
}
```
队列逻辑差不多清楚了，但是加入到队列中后怎么样执行好像没有讲到，以及 `CancelWork` 和 `AddWork` 具体做什么的也没说。所以下面我们讲一下到底驱逐逻辑是什么时候执行已经怎么执行的。

说到如何执行驱逐，我们需要先看下这个队列到底是一个什么队列，数据结构如下：
```golang
type TimedWorkerQueue struct {
    // 锁
	sync.Mutex
    // 存放 worker 的字典
	workers  map[string]*TimedWorker
    // work 执行方法
	workFunc func(ctx context.Context, args *WorkArgs) error
	clock    clock.WithDelayedExecution
}

type TimedWorker struct {
	WorkItem  *WorkArgs
	CreatedAt time.Time
	FireAt    time.Time
	Timer     clock.Timer
}
```

我们看到 worker 是带有计时功能的，所以我们大胆猜测是通过计时器触发函数执行。我们继续看 `AddWork` 方法：
```golang
func (q *TimedWorkerQueue) AddWork(ctx context.Context, args *WorkArgs, createdAt time.Time, fireAt time.Time) {
    // 获取 key，各式就是 <namespace>/<podName>
	key := args.KeyFromWorkArgs()
	klog.V(4).Infof("Adding TimedWorkerQueue item %v at %v to be fired at %v", key, createdAt, fireAt)

	q.Lock()
	defer q.Unlock()
    // 如果 workers 中存在，则跳过，避免重复执行
	if _, exists := q.workers[key]; exists {
		klog.Warningf("Trying to add already existing work for %+v. Skipping.", args)
		return
	}
    // 创建 worker，并保存到 workers 中，这个 getWrappedWorkerFuc方法很有意思，后面我们在看，大体作用就是执行 TimedWorkerQueue 的 workFunc
	worker := createWorker(ctx, args, createdAt, fireAt, q.getWrappedWorkerFunc(key), q.clock)
	q.workers[key] = worker
}
```
接着看 createWorker：
```golang
func createWorker(ctx context.Context, args *WorkArgs, createdAt time.Time, fireAt time.Time, f func(ctx context.Context, args *WorkArgs) error, clock clock.WithDelayedExecution) *TimedWorker {
    // 我们前面在 processPodOnNode 方法中执行 createWorker 方法时，传入了两个 time.Now，目的就是触发 delay <=0, 立马执行函数
	delay := fireAt.Sub(createdAt)
	if delay <= 0 {
		go f(ctx, args)
		return nil
	}
    // AfterFunc 就是执行驱逐的关键方法，它是带有延迟执行功能的方法，delay 时间到了就会执行
	timer := clock.AfterFunc(delay, func() { f(ctx, args) })
	return &TimedWorker{
		WorkItem:  args,
		CreatedAt: createdAt,
		FireAt:    fireAt,
		Timer:     timer,
	}
}
```
所以逻辑就很清楚了，我们通过 AddWork 把带有计时器的 worker 加入队列中，一旦计时器到期，延迟函数就会立马执行，进行驱逐，驱逐逻辑就封装在 `NewNoExecuteTaintManager` 的 `deletePodHandler` 方法中。

我们前面提到 `getWrappedWorkerFunc` 函数，我们看一下这个函数做了什么：
```golang
func (q *TimedWorkerQueue) getWrappedWorkerFunc(key string) func(ctx context.Context, args *WorkArgs) error {
	return func(ctx context.Context, args *WorkArgs) error {
        // 实际执行 workFunc
		err := q.workFunc(ctx, args)
		q.Lock()
		defer q.Unlock()
		if err == nil {
            // 这里没有从 workers 中删除 key，而是赋值为 nil，主要原因是为了避免重复提交队列，
			// 我们在 AddWork 中会根据 key 校验是否是重复提交。但是有一个问题是 worker 中的 key 什么时候删除呢？
			// NoExecuteTaintManager 同样利用了 listWatch 机制，删除 pod 同样会推送事件过来，
			// 这个时候我们会删除对应的 key。
			q.workers[key] = nil
		} else {
            // 如果出错，则执行 delete 操作，可以再次提交事件
			delete(q.workers, key)
		}
		return err
	}
}
```

CancelWork 逻辑更简单，就是从队列中删除 worker：
```golang
func (q *TimedWorkerQueue) CancelWork(key string) bool {
	q.Lock()
	defer q.Unlock()
	worker, found := q.workers[key]
	result := false
    // 如果找到 worker，一是取消计时，二是直接从worker中删除
	if found {
		klog.V(4).Infof("Cancelling TimedWorkerQueue item %v at %v", key, time.Now())
		if worker != nil {
			result = true
			worker.Cancel()
		}
		delete(q.workers, key)
	}
	return result
}
```

## 6. 总结
`NoExecuteTaintManager` 进行 pod 驱逐的逻辑差不多就这么多，简单的说就是通过 listWatch 获取 node 和 pod 的更新事件，包装成带有计时器功能的数据结构放入驱逐队列中，一旦计时结束，则执行驱逐逻辑，并输出对应的事件。