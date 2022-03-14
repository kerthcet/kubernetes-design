# Scheduler Cache 机制

    Kubernetes Version: v1.23@45f2c63d
    Date: 2022.03.13

## 1. 开篇
调度器整体架构如下图所示，我们看到 cache 层存在的价值就是帮助调度器更有效的进行调度。它是事件驱动的，通过 reflector 进行 list/watch，所以有可能出现网络问题导致事件丢失。

下面我们一起从源码层面学习一下。
![cache](../snapshots/scheduler-architecture.png)

## 2. 初始化
`Cache` 初始化时机是在 `scheduler` 初始化的时候，代码如下：
```golang
// ttl 表示过期时间，所有 cache 是一个带有过期时间的缓存层
func New(ttl time.Duration, stop <-chan struct{}) Cache {
	cache := newCache(ttl, cleanAssumedPeriod, stop)
	cache.run()
	return cache
}
```

`Cache` 是一个接口定义，它的具体实现在 `newCache` 中，是一个 `cacheImpl` 结构体：
```golang
type cacheImpl struct {
	stop   <-chan struct{} // 中断 channel 信号
	ttl    time.Duration // 表示过期时间
	period time.Duration // cache.run 中用到，表示方法运行的间隔时间
	mu sync.RWMutex // 读写锁
	assumedPods sets.String // 表示更新了 NodeName 的 pod，即找到了待调度的节点但是还未完成绑定
	podStates map[string]*podState // 存储所有的 pod，并表示 pod 当前状态，包括是否完成绑定，以及 pod 在 cache 中的过期时间
	nodes     map[string]*nodeInfoListItem // 节点双向链表
	headNode *nodeInfoListItem // 最近更新的 node
	nodeTree *nodeTree // 树状结构，根据 zone 存储节点
	imageStates map[string]*imageState // 镜像状态，包括镜像大小以及哪些节点已经存在该镜像
}
```

`run` 方法本质上就是循环调用 `cleanupExpiredAssumedPods` 方法，`period` 表示时间间隔：
```golang
func (cache *cacheImpl) run() {
	go wait.Until(cache.cleanupExpiredAssumedPods, cache.period, cache.stop)
}
```

`cleanupExpiredAssumedPods` 会循环调用 `cleanupAssumedPods` 方法来清理掉已经过期的 assumePod：
```golang
func (cache *cacheImpl) cleanupAssumedPods(now time.Time) {
	cache.mu.Lock()
	defer cache.mu.Unlock()

	for key := range cache.assumedPods {
		ps, ok := cache.podStates[key]
		if !ok {
			klog.ErrorS(nil, "Key found in assumed set but not in podStates, potentially a logical error")
			os.Exit(1)
		}

        // 如果 binding 还在进行中，则拒绝过期该 pod
		if !ps.bindingFinished {
			klog.V(5).InfoS("Could not expire cache for pod as binding is still in progress",
				"pod", klog.KObj(ps.pod))
			continue
		}

        // 如果 pod 已经过了最后期限，则从 cache中移除
		if now.After(*ps.deadline) {
			klog.InfoS("Pod expired", "pod", klog.KObj(ps.pod))
			if err := cache.removePod(ps.pod); err != nil {
				klog.ErrorS(err, "ExpirePod failed", "pod", klog.KObj(ps.pod))
			}
		}
	}
}
```

## 3. Cache 基本方法

### 3.1 AssumePod
`AssumePod` 在调度的 `assume` 阶段触发，假定 pod 已经完成调度。由于 `Assume` 发生在 `Add` 之前，因此也不存在更新和删除的操作，代码如下：
```golang
func (cache *cacheImpl) AssumePod(pod *v1.Pod) error {
    // 获取 pod 的 UID
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

    // podStates 不是并发安全的，所以需要加锁
	cache.mu.Lock()
	defer cache.mu.Unlock()
    // 如果 podStates 中存在 pod，则报错
	if _, ok := cache.podStates[key]; ok {
		return fmt.Errorf("pod %v is in the cache, so can't be assumed", key)
	}

    // 将 pod 存入 cache，注意此时 pod 变成了 assumePod
	return cache.addPod(pod, true)
}
```

### 3.2 AddPod
`AddPod` 方法负责将 pod 存入 cache 中，并从 assumedPods 中移除，源码如下：
```golang
func (cache *cacheImpl) AddPod(pod *v1.Pod) error {
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

	cache.mu.Lock()
	defer cache.mu.Unlock()

	currState, ok := cache.podStates[key]
	switch {
    // 如果 podStates 和 assumePods 中都有该 pod 记录，表示之前已经添加过
	case ok && cache.assumedPods.Has(key):
        // 如果 assume 和 assign 不是同一个 node，则需要更新
		if currState.pod.Spec.NodeName != pod.Spec.NodeName {
			if err = cache.updatePod(currState.pod, pod); err != nil {
				klog.ErrorS(err, "Error occurred while updating pod")
			}
		} else {
            // 否则需要将 assumePods 中的 pod 删除，重新进行调度。同时移除 deadline。本质和 updatePod 相差不大，但是代码效率更高
			delete(cache.assumedPods, key)
			cache.podStates[key].deadline = nil
			cache.podStates[key].pod = pod
		}
	case !ok:
        // 如果 podStates 没有 pod，则添加，注意这里 addPod 第二个参数为 false，表示并非是一个 assumePod
		if err = cache.addPod(pod, false); err != nil {
			klog.ErrorS(err, "Error occurred while adding pod")
		}
	default:
		// 其他情况入 podStates 中有值而 assumedPods 没有，表示可能是同一个 add 事件触发了两次，报错
		return fmt.Errorf("pod %v was already in added state", key)
	}
	return nil
}
```

`updatePod` 方法逻辑很简单，先 remove 再 add：
```golang
func (cache *cacheImpl) updatePod(oldPod, newPod *v1.Pod) error {
	if err := cache.removePod(oldPod); err != nil {
		return err
	}
	return cache.addPod(newPod, false)
}
```
我们先看一下 `removePod` 方法：
```golang
func (cache *cacheImpl) removePod(pod *v1.Pod) error {
    // 获取 pod UID
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

	n, ok := cache.nodes[pod.Spec.NodeName]
    // 如果 nodes 列表中没有该 node，打印错误日志，这里没有直接返回，因为后面的逻辑不会有副作用
	if !ok {
		klog.ErrorS(nil, "Node not found when trying to remove pod", "node", klog.KRef("", pod.Spec.NodeName), "pod", klog.KObj(pod))
	} else {
        // 从 node 的 中移除 pod 相关信息，包括资源使用和端口占用等
		if err := n.info.RemovePod(pod); err != nil {
			return err
		}
        // 如果此时 node 中已经没有其他 pod，则可以删除该 node 相关信息
		if len(n.info.Pods) == 0 && n.info.Node() == nil {
			cache.removeNodeInfoFromList(pod.Spec.NodeName)
		} else {
            // 否则，将 node 移到列表最前面，表示最近更新
			cache.moveNodeInfoToHead(pod.Spec.NodeName)
		}
	}

    // 从 podStates 和 assumePods 中删除相关元素
	delete(cache.podStates, key)
	delete(cache.assumedPods, key)
	return nil
}
```

我们再看一下 `addPod()` 方法，它基本上就是 `removePod` 的反向操作：
```golang
func (cache *cacheImpl) addPod(pod *v1.Pod, assumePod bool) error {
    // 获取 pod uid
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}
    // 如果 nodes 中还没有对应的值，则初始化一个节点列表
	n, ok := cache.nodes[pod.Spec.NodeName]
	if !ok {
		n = newNodeInfoListItem(framework.NewNodeInfo())
		cache.nodes[pod.Spec.NodeName] = n
	}
    // 添加 pod，完善节点信息，如计算 pod 资源占用率，端口使用情况等等
	n.info.AddPod(pod)
    // 将 node 添加到头部，表示最近更新
	cache.moveNodeInfoToHead(pod.Spec.NodeName)
	ps := &podState{
		pod: pod,
	}
    // 添加 pod 到 podStates 中
	cache.podStates[key] = ps
    // 如果是assumePod，则添加至 `assumedPods` 集合
	if assumePod {
		cache.assumedPods.Insert(key)
	}
	return nil
}
```

### 3.3 ForgetPod
`ForgetPod` 是 `assume` 的反向操作，负责删除 assumePod：
```golang
func (cache *cacheImpl) ForgetPod(pod *v1.Pod) error {
    // 获取 pod uid
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

    // 加锁
	cache.mu.Lock()
	defer cache.mu.Unlock()

	// 逻辑检查
	currState, ok := cache.podStates[key]
	if ok && currState.pod.Spec.NodeName != pod.Spec.NodeName {
		return fmt.Errorf("pod %v was assumed on %v but assigned to %v", key, pod.Spec.NodeName, currState.pod.Spec.NodeName)
	}

	// 从 assumePods 中删除 pod
	if ok && cache.assumedPods.Has(key) {
		return cache.removePod(pod)
	}
	return fmt.Errorf("pod %v wasn't assumed so cannot be forgotten", key)
}
```

### 3.4 UpdatePod
`UpdatePod` 负责更新 cache 中的 pod：
```golang
func (cache *cacheImpl) UpdatePod(oldPod, newPod *v1.Pod) error {
	// 获取 old pod UID
	key, err := framework.GetPodKey(oldPod)
	if err != nil {
		return err
	}

	// 加锁
	cache.mu.Lock()
	defer cache.mu.Unlock()

	currState, ok := cache.podStates[key]
	// AddPod 之后 assumedPods 中就没有该 pod 了，所以在这里做了一次检查。
	if ok && !cache.assumedPods.Has(key) {
		if currState.pod.Spec.NodeName != newPod.Spec.NodeName {
			klog.ErrorS(nil, "Pod updated on a different node than previously added to", "pod", klog.KObj(oldPod))
			klog.ErrorS(nil, "scheduler cache is corrupted and can badly affect scheduling decisions")
			os.Exit(1)
		}
		// 更新逻辑
		return cache.updatePod(oldPod, newPod)
	}
	return fmt.Errorf("pod %v is not added to scheduler cache, so cannot be updated", key)
}
```

### 3.5 FinishBinding
`FinishBinding` 在 scheduler bind 阶段负责结束绑定的相关逻辑：
```golang
func (cache *cacheImpl) finishBinding(pod *v1.Pod, now time.Time) error {
    // 获取 pod uid
	key, err := framework.GetPodKey(pod)
	if err != nil {
		return err
	}

    // 加锁
	cache.mu.RLock()
	defer cache.mu.RUnlock()

	klog.V(5).InfoS("Finished binding for pod, can be expired", "pod", klog.KObj(pod))
	currState, ok := cache.podStates[key]
    // 标记 bindingFinished 为 true，表示完成绑定
	// 更新 deadline 为 cache 默认的过期时间
	if ok && cache.assumedPods.Has(key) {
		dl := now.Add(cache.ttl)
		currState.bindingFinished = true
		currState.deadline = &dl
	}
	return nil
}
```

## 4. 机制

### 4.1 状态机
下图展示了 Cache 的状态机机制：

	   +-------------------------------------------+  +----+
	   |                            Add            |  |    |
	   |                                           |  |    | Update
	   +      Assume                Add            v  v    |
	Initial +--------> Assumed +------------+---> Added <--+
	   ^                +   +               |       +
	   |                |   |               |       |
	   |                |   |           Add |       | Remove
	   |                |   |               |       |
	   |                |   |               +       |
	   +----------------+   +-----------> Expired   +----> Deleted
	         Forget             Expire

1. pod 被成功调度后，会执行 assume 添加到 cache 中，此时状态为 Assumed
2. 异步 bind 操作，如果超时，则调起 forget 操作，将 pod 从 cache 中删除
3. cache 会在后台定期清理过期 assumed pod
4. 如果 bind 成功，则会收到 AddEvent，将 pod 从 assumedPods 中移除
5. 如果收到 UpdateEvent，则更新 pod
6. 如果收到 DeleteEvent，则从 cache 中删除 pod

### 4.2 具体事件
* 快照 snapshot

	每一轮调度周期开始时 scheduler 会进行一次快照操作，调用 Cache `UpdateSnapshot` 方法，对 Cache 的节点信息进行快照。

* list/watch 事件

	Pod:

		Pod:AddFunc       -> Cache:AddPod

		Pod:UpdateFunc    -> Cache:UpdatePod

		Pod:DeleteFunc    -> Cache:RemovePod

	Node:

		Node:AddFunc      -> Cache:AddNode

		Node:UpdateFunc   -> Cache:UpdateNode

		Node:DeleteFunc   -> Cache:RemoveNode

* 跳过调度周期

	如果 pod 是 assumePod，即在 cache 的 assumePods 中存在，则跳过调度周期。

* Unreserve

	所以调度失败进入 Unreserve 阶段的 pod，都会调用 cache 的 `ForgetPod` 方法。


## 5. 总结
以上就是 Cache 的大致工作逻辑，通过 reflector 监听 api 事件，其中 pod 大体经过了 assumed -> added -> updated -> deleted 这样一个流程，并在 cache 中进行了信息的整合。最后在调度的时候通过快照获得集群节点信息，提高调度的效率。