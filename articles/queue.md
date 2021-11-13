# Kube-Scheduler 调度队列

    Kubernetes Version: v1.22@3b76c758317b
    Date: 2021.11.08

## 1. Scheduler 优先级队列
在之前的文章 [Kube-Scheduler 启动](https://github.com/kerthcet/kube-scheduler-design/blob/main/articles/start-scheduler.md)中，我们初始化了一个优先级队列，当时由于篇幅有限，我们只是简单的提了一下。这篇文章，我们将对整个队列进行全方位的解读，包括入列出列操作，三级队列彼此之间如何协作等等。

### 1.1 初始化优先级队列

    podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		c.informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
		internalqueue.WithClusterEventMap(c.clusterEventMap),
	)

我们先看一下 `NewSchedulingQueue` 的几个入参：

第一个入参是 `lessFn`， 它是一个用来对 `scheduling queue` 中的 `pods` 进行排序的方法，定义如下：

    type LessFunc func(podInfo1, podInfo2 *QueuedPodInfo) bool

它的实际方法位于 `pkg/scheduler/framework/plugins/queuesort/priority_sort.go`，通过 `NewInTreeRegistry()` 方法完成注册，我们一起来看一下这个 `Less()` 方法：

    func (pl *PrioritySort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
        p1 := corev1helpers.PodPriority(pInfo1.Pod)
        p2 := corev1helpers.PodPriority(pInfo2.Pod)
        return (p1 > p2) || (p1 == p2 && pInfo1.Timestamp.Before(pInfo2.Timestamp))
    }

`PodPriority()` 方法获取 `pod.Spec.Priority` 值，如果 `priority` 是 `nil` 则返回0。`less()` 方法其实就是对两个 `pod` 的优先级进行比较，如果相等则比较他们加入队列的时间戳，创建时间越早的 `pod` 优先级越高。至于这个方法什么时候会被调用，下面我们会讲到。

第二个参数是之前定义过的 `SharedInformerFactory`，用来 `watch` `pod` 事件，并参与后面的调度。

第三个参数是 `podInitialBackoffDuration`，表示初始化等待时间，默认1s，第四个参数是 `WithPodMaxBackoffDuration`，表示最大等待时间，默认10s。这两个参数主要用于计算 `backupDuration`，所谓 `backupDuration` 是指带有指数回避策略的等待重试时间，当 `pod` 调度失败，会进行重试，为了避免频繁进行失败重试，所以每次重试间隔时间会指数级递增，最长不超过 `maxBackoffDuration`。

第五个参数是 `PodNominator`，主要用于抢占调度流程，这里不做过多介绍。

第六个参数是 `clusterEventMap`，主要是和 `informer` 交互，绑定具体的事件和对应的处理方法，我们会在 `informer` 章节中详细的介绍。

至此，优先级队列初始化完成。

### 1.2 启动优先级队列
启动优先级队列在之前的文章中我们已经提到过，通过调用 `Run()` 方法启动了两个后台循环队列，一个是 `backoffQ` 队列，还有一个 `unschedulableQ` 队列，这两个循环队列主要是存储调度失败的和无法调度的 `pod` 。

	func (p *PriorityQueue) Run() {
		go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
		go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
	}

#### 1.2.1 `flushBackoffQCompleted`
我们先看 `flushBackoffQCompleted`，每隔1s执行一次该方法，实现如下：

	func (p *PriorityQueue) flushBackoffQCompleted() {
		// 上锁
		p.lock.Lock()
		defer p.lock.Unlock()

		for {
			// 获取 heap 堆顶元素，如果不存在，表示队列为空，返回。平均时间复杂度 O(1)
			rawPodInfo := p.podBackoffQ.Peek()
			if rawPodInfo == nil {
				return
			}

			pod := rawPodInfo.(*framework.QueuedPodInfo).Pod
			boTime := p.getBackoffTime(rawPodInfo.(*framework.QueuedPodInfo))

			// 如果还没到重试时间，则继续等待，直接返回
			if boTime.After(p.clock.Now()) {
				return
			}

			// 否则，就弹出一个 pod
			_, err := p.podBackoffQ.Pop()
			if err != nil {
				return
			}

			// 加入 activeQ，并唤醒接收方 goroutine
			p.activeQ.Add(rawPodInfo)
			defer p.cond.Broadcast()
		}
	}


#### 1.2.2 `flushUnschedulableQLeftover`
`flushUnschedulableQLeftover` 方法逻辑更简单，每隔30s执行一次。

	func (p *PriorityQueue) flushUnschedulableQLeftover() {
		// 上锁
		p.lock.Lock()
		defer p.lock.Unlock()

		var podsToMove []*framework.QueuedPodInfo
		currentTime := p.clock.Now()
		for _, pInfo := range p.unschedulableQ.podInfoMap {
			// 获取 pod 入列时间
			lastScheduleTime := pInfo.Timestamp

			// 如果在 `unschedulableQ` 中存储时间超过60s，则将它转移到 `activeQ` 或者 `podBackoffQ`
			if currentTime.Sub(lastScheduleTime) > unschedulableQTimeInterval {
				podsToMove = append(podsToMove, pInfo)
			}
		}

		if len(podsToMove) > 0 {
			// 转移
			p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
		}
	}

## 2. `Scheduler` 三级队列
`scheduler` 的三级队列定义在优先级队列的数据结构中，我们一起看一下：

	type PriorityQueue struct {
		activeQ *heap.Heap
		podBackoffQ *heap.Heap
		unschedulableQ *UnschedulablePodsMap
		schedulingCycle int64
	}

1. `activeQ`

	存储所有等待调度的 `pod`，它是一个 `heap` 数据结构，准确的说是一个大顶堆，插入和更新队列平均时间复杂度是 `O(logN)`，获取队列元素的平均时间复杂度是 `O(1)`。

	`heap` 堆栈在进行构建优先级队列的时候有一个很重要的方法就是如何比较优先级，那这个方法是怎么定义的呢，其实就是我们在[初始化优先级队列](https://github.com/kerthcet/kube-scheduler-design/blob/main/articles/queue.md#11-%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BC%98%E5%85%88%E7%BA%A7%E9%98%9F%E5%88%97)中所讲的 `lessFn` 方法。

2. `podBackoffQ`

	存储所有调度失败的 `pod`，并且带有指数回避策略。同样，它也是一个 `heap` 堆栈。那 `podBackoffQ` 中元素的优先级是如何进行比较的呢，方法如下：

		func (p *priorityqueue) podscomparebackoffcompleted(podinfo1, podinfo2 interface{}) bool {
			pinfo1 := podinfo1.(*framework.queuedpodinfo)
			pinfo2 := podinfo2.(*framework.queuedpodinfo)
			bo1 := p.getbackofftime(pinfo1)
			bo2 := p.getbackofftime(pinfo2)
			return bo1.before(bo2)
		}

	方法很简单，谁距离下一次等待调度的时间越短，谁优先级越高。


3. `unschedulableQ`

	存储所有因为资源不够而排队的 `pod`，它是一个包装了 `map` 的数据结构

		type UnschedulablePodsMap struct {
			// pod 实际保存在这个 map 中
			podInfoMap map[string]*framework.QueuedPodInfo
			keyFunc    func(*v1.Pod) string
			metricRecorder metrics.MetricRecorder
		}
4. `schedulingCycle`

	逻辑时钟，表示调度周期。每次从优先级队列中 `Pop()` 一个元素出来，就累加。
## 3. 队列工作流程
### 3.1 入列 (activeQ)
优先级队列入列（这里主要指 `activeQ`）操作是通过 `Informer` 实现的，代码位于 `pkg/scheduler/eventhandler.go`，`addAllEventHandlers()` 有一段代码如下：

	Handler: cache.ResourceEventHandlerFuncs{
		AddFunc:    sched.addPodToSchedulingQueue,
		UpdateFunc: sched.updatePodInSchedulingQueue,
		DeleteFunc: sched.deletePodFromSchedulingQueue,
	}

所有的新增、更新、删除事件都会触发对应的入列，更新和出列操作，我们一起看一下这三种操作的具体代码逻辑。

`Add` 事件

	func (p *PriorityQueue) Add(pod *v1.Pod) error {
		p.lock.Lock()
		defer p.lock.Unlock()

		pInfo := p.newQueuedPodInfo(pod)
		if err := p.activeQ.Add(pInfo); err != nil {
			return err
		}

		if p.unschedulableQ.get(pod) != nil {
			p.unschedulableQ.delete(pod)
		}

		if err := p.podBackoffQ.Delete(pInfo); err == nil {
		}

		p.cond.Broadcast()

		return nil
	}

1. 首先通过加锁的方式锁住队列。`p.lock` 本身是一个读写锁 `sync.RWMutex`，并通过 `defer` 方法保证最后会释放锁。

2. 构造队列元素。`activeQ` 中元素的类型是 `podInfo`，数据结构如下：

		type QueuedPodInfo struct {
			*PodInfo
			Timestamp time.Time
			Attempts int
			InitialAttemptTimestamp time.Time
			UnschedulablePlugins sets.String
		}

	除了包括本身的一些 `pod` 信息，还有加入队列的时间，调度次数，因为过程中可能出现调度失败，需要重新调度。`UnschedulablePlugins` 则表示调度失败的插件名称。

3. 添加队列元素。 `p.activeQ.Add(pInfo)` 其实就是往底层的 `heap` 插入元素。同时，我们需要保证 `podBackoffQ` 和 `unschedulableQ` 中没有对应的元素，否则会重复调度。

4. 通知。 `activeQ` 使用条件变量实现通知功能，一旦入列完成，通过 `Broadcast()` 方法唤醒所有正在监听的 `goroutine`，后面我们讲出列逻辑的时候还会涉及到。

`Update` 事件

	func (sched *Scheduler) updatePodInSchedulingQueue(oldObj, newObj interface{}) {
		oldPod, newPod := oldObj.(*v1.Pod), newObj.(*v1.Pod)
		// Bypass update event that carries identical objects; otherwise, a duplicated
		// Pod may go through scheduling and cause unexpected behavior (see #96071).
		if oldPod.ResourceVersion == newPod.ResourceVersion {
			return
		}

		isAssumed, err := sched.SchedulerCache.IsAssumedPod(newPod)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("failed to check whether pod %s/%s is assumed: %v", newPod.Namespace, newPod.Name, err))
		}
		if isAssumed {
			return
		}

		if err := sched.SchedulingQueue.Update(oldPod, newPod); err != nil {
			utilruntime.HandleError(fmt.Errorf("unable to update %T: %v", newObj, err))
		}
	}

1. 对比 `oldPod` 和 `newPod` 的 `ResourceVersion`，如果不一致，则返回，保证更新的是同一个 `pod` 对象。

2. 判断 `pod` 是否是 `assumedPod`，`assumedPod` 你可以简单理解为已经找到对应调度节点的 `pod`，不再需要重新走调度流程。

3. 如果都满足条件，则走更新流程，代码如下。代码很长，我把具体逻辑用中文标出。

		func (p *PriorityQueue) Update(oldPod, newPod *v1.Pod) error {
			// 持有锁，防止冲突
			p.lock.Lock()
			defer p.lock.Unlock()

			if oldPod != nil {
				// 构造podInfo对象
				oldPodInfo := newQueuedPodInfoForLookup(oldPod)

				// 如果 `activeQ` 中存在，则更新该对象并重新添加到 `activeQ` 队列中，该添加方法是幂等的，不会重复创建。
				if oldPodInfo, exists, _ := p.activeQ.Get(oldPodInfo); exists {
					pInfo := updatePod(oldPodInfo, newPod)
					return p.activeQ.Update(pInfo)
				}

				// 如果 `podBackoffQ` 中存在，则同样更新对象并添加到 `podBackoffQ`，入列操作同样也是幂等的。
				if oldPodInfo, exists, _ := p.podBackoffQ.Get(oldPodInfo); exists {
					pInfo := updatePod(oldPodInfo, newPod)
					return p.podBackoffQ.Update(pInfo)
				}
			}

			// 判断 unschedulableQ 中是否存在该 pod
			if usPodInfo := p.unschedulableQ.get(newPod); usPodInfo != nil {
				// 如果存在就更新该 pod
				pInfo := updatePod(usPodInfo, newPod)

				// 判断该 pod 中是否有影响调度的字段发生了更新，如果有，则判断是否可以加入到 activeQ，否则重新加入 unschedulableQ
				if isPodUpdated(oldPod, newPod) {
					// 判断该 pod 是否已经结束了等待时间，如果还没有，就重新加入到 podBackoffQ 队列中，否则加入 activeQ 队列
					if p.isPodBackingoff(usPodInfo) {
						if err := p.podBackoffQ.Add(pInfo); err != nil {
							return err
						}
						p.unschedulableQ.delete(usPodInfo.Pod)
					} else {
						if err := p.activeQ.Add(pInfo); err != nil {
							return err
						}
						p.unschedulableQ.delete(usPodInfo.Pod)
						p.cond.Broadcast()
					}
				} else {
					p.unschedulableQ.addOrUpdate(pInfo)
				}

				return nil
			}


			// 如果任何队列中都没有找到，则直接加入到 activeQ 队列，并广播入列信息
			pInfo := p.newQueuedPodInfo(newPod)
			if err := p.activeQ.Add(pInfo); err != nil {
				return err
			}
			p.cond.Broadcast()
			return nil
		}

Delete 事件

	func (sched *Scheduler) deletePodFromSchedulingQueue(obj interface{}) {
		if err := sched.SchedulingQueue.Delete(pod); err != nil {
			utilruntime.HandleError(fmt.Errorf("unable to dequeue %T: %v", obj, err))
		}

		// ...

		if fwk.RejectWaitingPod(pod.UID) {
			sched.SchedulingQueue.MoveAllToActiveOrBackoffQueue(queue.AssignedPodDelete, nil)
		}
	}

1. 从优先级队列中删除 `pod`
2. 从 waitingPods 中删除该pod，如果成功，表示该 pod 之前已经成功 `assumed`，现在从优先级队列中删除，则会改变集群的资源利用情况，所以会立即触发 `MoveAllToActiveOrBackoffQueue()` 方法，将 `unschedulableQ` 中的元素全部转移至 `podBackoffQ` 或者 `activeQ` 中，我们看一下具体实现 `movePodsToActiveOrBackoffQueue`：

		func (p *PriorityQueue) movePodsToActiveOrBackoffQueue(podInfoList []*framework.QueuedPodInfo, event framework.ClusterEvent) {
			moved := false

			// podInfoList 是 unschedulableQ 中全部元素
			for _, pInfo := range podInfoList {

				// 如果该元素还在重试等待时间内，则加入 `podBackoffQ`，否则加入 `activeQ`
				if p.isPodBackingoff(pInfo) {
					if err := p.podBackoffQ.Add(pInfo); err != nil {
						klog.ErrorS(err, "Error adding pod to the backoff queue", "pod", klog.KObj(pod))
					} else {
						p.unschedulableQ.delete(pod)
					}
				} else {
					if err := p.activeQ.Add(pInfo); err != nil {
						klog.ErrorS(err, "Error adding pod to the scheduling queue", "pod", klog.KObj(pod))
					} else {
						p.unschedulableQ.delete(pod)
					}
				}
			}

			// ...

			// 如果 move 成功，则唤醒 goroutine
			if moved {
				p.cond.Broadcast()
			}
		}

### 3.2 出列
前面说到优先级队列通过条件变量实现通知功能，那消费端肯定也是通过条件变量被唤醒，调用代码入口位于 `pkg/scheduler/scheduler.go` 的 `scheduleOne()` 方法，具体实现代码如下：

	func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo {
		return func() *framework.QueuedPodInfo {
			podInfo, err := queue.Pop()
			// ...
		}
	}

其实实现很简单，就是调用优先级队列的 `Pop()` 方法，我们一起看一下：

	func (p *PriorityQueue) Pop() (*framework.QueuedPodInfo, error) {
		// 加锁
		p.lock.Lock()
		defer p.lock.Unlock()

		// 如果 activeQ 中存在元素，则不会等待，否则进入等待状态，直到被唤醒
		for p.activeQ.Len() == 0 {
			if p.closed {
				return nil, fmt.Errorf(queueClosed)
			}
			p.cond.Wait()
		}

		// 从 activeQ 中弹出元素
		obj, err := p.activeQ.Pop()
		if err != nil {
			return nil, err
		}
		pInfo := obj.(*framework.QueuedPodInfo)

		// ...

		return pInfo, err
	}

### 3.3 `unschedulableQ` 入列
`unschedulableQ` 入列主要在两个地方，第一个是在更新优先级队列的时候，即 `Update()` 方法，我们在[3.1 入列(activeQ)](https://github.com/kerthcet/kube-scheduler-design/blob/main/articles/queue.md#31-%E5%85%A5%E5%88%97-activeq) 讲 `Update` 事件的时候聊过了。第二个是在调用 `AddUnschedulableIfNotPresent()` 方法的时候，实现如下：

	func (p *PriorityQueue) AddUnschedulableIfNotPresent(pInfo *framework.QueuedPodInfo, podSchedulingCycle int64) error {
		p.lock.Lock()
		defer p.lock.Unlock()

		pod := pInfo.Pod
		if p.unschedulableQ.get(pod) != nil {
		}

		if _, exists, _ := p.activeQ.Get(pInfo); exists {
		}
		if _, exists, _ := p.podBackoffQ.Get(pInfo); exists {
		}

		if p.moveRequestCycle >= podSchedulingCycle {
			if err := p.podBackoffQ.Add(pInfo); err != nil {
			}
		} else {
			p.unschedulableQ.addOrUpdate(pInfo)
		}

		return nil
	}

其实逻辑很清晰，首先判断三级队列中是否存在该元素，如果存在，就返回，否则执行入列操作。但是这里我们做了一个逻辑判断，判断 `moveRequestCycle` 和 `podSchedulingCycle` 大小。`podSchedulingCycle` 我们知道它是一个逻辑概念，表示调度周期，那 `moveRequestCycle` 是什么意思呢？我们可以把它理解为一个缓存，每次执行 `movePodsToActiveOrBackoffQueue()` 方法，都会将 `podSchedulingCycle` 的值赋值给他，而触发 `movePodsToActiveOrBackoffQueue` 的时机除了后台定时任务，当集群资源发生变化的时候，也需要触发该方法，优化调度流程，让 `unschedulableQ` 中的 `pod` 尽快得到调度。

另外，调用 `AddUnschedulableIfNotPresent()` 的方法是 `MakeDefaultErrorFunc()`，是 `Scheduler` 处理 `error` 错误默认方法。

我们把这几个概念串起来，就理解了大致逻辑，当调度失败的时候，我们通过判断 `moveRequestCycle` 和 `podSchedulingCycle` 大小，就可以判断当前集群资源是否发生变化，如果没有，则重新加入 `unschedulableQ`，否则加入 `podBackoffQ`，有些同学问为什么不加入到 `activeQ`，因为我们前面刚调度失败，不应该再加入到 `activeQ`。

### 3.4 `podBackoffQ` 入列
`podBackoffQ` 入列操作其实都穿插在了之前的逻辑中，一个是 `AddUnschedulableIfNotPresent()` 方法，一个是优先级队列的 `Update()` 方法，还有一个是 `movePodsToActiveOrBackoffQueue()` 方法，这里就不展开了。

## 4. 总结
`Scheduler` 队列主要采用了三级队列加两个后台任务的方式来进行任务调度，同时采用锁的机制保证线程安全，使用条件变量进行信号通信，另外还加入了一些逻辑时钟来优化调度流程，整体逻辑还是有点复杂，而且代码冗余，所以社区也在想办法优化这几个队列。