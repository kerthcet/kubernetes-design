# scheduler 优先级和抢占

    Kubernetes Version: v1.24@f1da8cd3e20
    Date: 2022.01.15

## 1. 开篇
Pod 可以有 优先级。 优先级表示一个 Pod 相对于其他 Pod 的重要性。 如果一个 Pod 无法被调度，调度程序会尝试抢占（驱逐）较低优先级的 Pod， 以使悬决 Pod 可以被调度。这就是 `kubernetes` 优先级于抢占机制。今天我们一起从源码角度看一下它是如何实现的。

## 2. 如何设置 `pod` 优先级
我们一般通过 [`PriorityClass`](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/pod-priority-preemption/) 来设置 `pod` 优先级的方法，它是通过 `AdmissionController` 来实现的，主要代码逻辑位于 `plugin/pkg/admission/priority` 的 `Admit()` 方法：

```golang
func (p *Plugin) admitPod(a admission.Attributes) error {
    operation := a.GetOperation()
    pod, ok := a.GetObject().(*api.Pod)

    // 更新操作
    if operation == admission.Update {
        oldPod, ok := a.GetOldObject().(*api.Pod)

        // 如果原pod有优先级，则需要保留
        if pod.Spec.Priority == nil && oldPod.Spec.Priority != nil {
            pod.Spec.Priority = oldPod.Spec.Priority
        }

        // 同样保留抢占策略
        if pod.Spec.PreemptionPolicy == nil && oldPod.Spec.PreemptionPolicy != nil {
            pod.Spec.PreemptionPolicy = oldPod.Spec.PreemptionPolicy
        }
        return nil
    }

    // 创建操作
    if operation == admission.Create {
        var priority int32
        var preemptionPolicy *apiv1.PreemptionPolicy

        // 如果没有配置优先级，则配置默认优先级
        if len(pod.Spec.PriorityClassName) == 0 {
            var err error
            var pcName string
            pcName, priority, preemptionPolicy, err = p.getDefaultPriority()
            pod.Spec.PriorityClassName = pcName
        } else {
            // 如果配置了优先级，则进行赋值操作
            pc, err := p.lister.Get(pod.Spec.PriorityClassName)

            priority = pc.Value
            preemptionPolicy = pc.PreemptionPolicy
        }

        // 设置pod优先级
        pod.Spec.Priority = &priority

        // 配置抢占策略
        if p.nonPreemptingPriority {
            var corePolicy core.PreemptionPolicy
            if preemptionPolicy != nil {
                corePolicy = core.PreemptionPolicy(*preemptionPolicy)

                // 设置pod抢占策略
                pod.Spec.PreemptionPolicy = &corePolicy
            }
        }
    }
    return nil
}
```

通过 `admitPod()` 方法，我们就完成对 `pod` 优先级和抢占策略的设置。

## 3. 优先级调度
`pod` 完成优先级设置，之后会进入到调度流程，调度队列会根据优先级进行排序，我们在[优先级调度](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/queue.md#2-scheduler-%E4%B8%89%E7%BA%A7%E9%98%9F%E5%88%97)中讲 `activeQ` 的时候已经介绍过，`activeQ` 是一个优先级队列，会根据 `pod.Spec.Priority` 值进行排序，`Priority` 值越大，优先级越高，排在队列的越前面，所以会被优先调度。

## 4. 抢占机制
`Preemption` 现在是以一个插件的形式工作在 `PostFilter` 这个扩展点，这个插件名叫 `DefaultPreemption`，是一个默认的内置插件，代码位于 `pkg/scheduler/framework/plugins/defaultpreemption:default_preemption.go`。

我们先看一下 `PostFilter` 扩展点代码逻辑（由于本篇文章着重介绍抢占，其他扩展点逻辑不展开）：
```golang
func (sched *Scheduler) scheduleOne(ctx context.Context) {
    // ...

    // 调度逻辑
    scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, sched.Extenders, fwk, state, pod)

    // 如果找不到合适的节点，就进行抢占调度
    if err != nil {
        // nominatingInfo 包括候选节点相关信息
		var nominatingInfo *framework.NominatingInfo
        if fitError, ok := err.(*framework.FitError); ok {
            // ...
            // 执行抢占逻辑
            result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatusMap)

            // 如果抢占成功，则将候选节点相关信息赋值给 nominatingInfo，如果失败，则直接返回
            if status.Code() == framework.Error {
                klog.ErrorS(nil, "Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", status)
            } else {
                klog.V(5).InfoS("Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", status)
            }
            if result != nil {
                nominatingInfo = result.NominatingInfo
            }
        }

        // 处理所有调度失败后续问题，包括抢占调度，后面我们还会介绍。
		sched.handleSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatingInfo)
		return
    }

    // ...省略其他调度逻辑
}
```

### 4.1 抢占流程之抢
抢占逻辑位于代码 `RunPostFilterPlugins()` 方法中，它会遍历每个 `Plugin`，并调用他们的 `PostFilter()` 方法，我们一起看一下默认 `PostFilter` 扩展点插件 `DefaultPreemption` 的方法：
```golang
func (pl *DefaultPreemption) PostFilter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, m framework.NodeToStatusMap) (*framework.PostFilterResult, *framework.Status) {
    // 构造 Evaluator
    pe := preemption.Evaluator{
        PluginName: names.DefaultPreemption,
        Handler:    pl.fh,
        PodLister:  pl.podLister,
        PdbLister:  pl.pdbLister,
        State:      state,
        Interface:  pl,
    }

    // 调用抢占逻辑
    return pe.Preempt(ctx, pod, m)
}
```

我们继续看 `pe.Preempt()` 方法：
```golang
func (ev *Evaluator) Preempt(ctx context.Context, pod *v1.Pod, m framework.NodeToStatusMap) (*framework.PostFilterResult, *framework.Status) {

    podNamespace, podName := pod.Namespace, pod.Name
    pod, err := ev.PodLister.Pods(pod.Namespace).Get(pod.Name)

    // 检查pod是否具备抢占条件，如果 pod 已经被提名过一次且提名节点上的 pod 正在终止，此时应该拒绝抢占。
    if !ev.PodEligibleToPreemptOthers(pod, m[pod.Status.NominatedNodeName]) {
        return nil, framework.NewStatus(framework.Unschedulable)
    }

    // 找到所有的候选节点
    candidates, nodeToStatusMap, err := ev.findCandidates(ctx, pod, m)
    if err != nil && len(candidates) == 0 {
        return nil, framework.AsStatus(err)
    }

    // 如果没有候选节点，则返回
    if len(candidates) == 0 {
        return nil, framework.NewStatus(framework.Unschedulable, fitError.Error())
    }

    // 找出最合适的那个节点
    bestCandidate := ev.SelectCandidate(candidates)
    if bestCandidate == nil || len(bestCandidate.Name()) == 0 {
        return nil, framework.NewStatus(framework.Unschedulable)
    }

    // 抢占准备工作
    if status := ev.prepareCandidate(bestCandidate, pod, ev.PluginName); !status.IsSuccess() {
        return nil, status
    }

    // 返回抢占节点，结束抢占流程
    return &framework.PostFilterResult{NominatedNodeName: bestCandidate.Name()}, framework.NewStatus(framework.Success)
}
```

这就是整个抢占机制的大致流程，下面我们就其中的 `findCandidates()`，`selectCandidate()` 和 `prepareCandidate()` 这三个最核心的方法分别进行讲解。

#### 4.1.1 findCandidates
`findCandidates()` 顾名思义就是寻找所有可能的候选节点，那它是基于何种规则进行删选呢，我们一起看一下：
```golang
func (ev *Evaluator) findCandidates(ctx context.Context, pod *v1.Pod, m framework.NodeToStatusMap) ([]Candidate, framework.NodeToStatusMap, error) {
    // 获取所有快照 nodes
    allNodes, err := ev.Handler.SnapshotSharedLister().NodeInfos().List()

    // 删选掉即使抢占也无法调度的节点
    potentialNodes, unschedulableNodeStatus := nodesWherePreemptionMightHelp(allNodes, m)
    if len(potentialNodes) == 0 {
        return nil, unschedulableNodeStatus, nil
    }

    // 获取pdb资源
    pdbs, err := getPodDisruptionBudgets(ev.PdbLister)

    // 获取一个偏移量和候选的节点数量
    offset, numCandidates := ev.GetOffsetAndNumCandidates(int32(len(potentialNodes)))

    // 模拟抢占事件，返回符合条件的候选节点及不符合条件的节点调度状态
    candidates, nodeStatuses, err := ev.DryRunPreemption(ctx, pod, potentialNodes, pdbs, offset, numCandidates)

    // nodeStatuses 负责存储所有不符合条件的节点调度状态，如果没有找到符合要求的节点，则输出失败记录
    for node, nodeStatus := range unschedulableNodeStatus {
        nodeStatuses[node] = nodeStatus
    }

    // 返回结果
    return candidates, nodeStatuses, err
}
```


findCandidates() 方法中最重要的逻辑位于 DryRunPreemption 中，我们一起看一下它的逻辑：
```golang
func (ev *Evaluator) DryRunPreemption(...) ([]Candidate, framework.NodeToStatusMap, error) {
    // ...

    checkNode := func(i int) {
        nodeInfoCopy := potentialNodes[(int(offset)+i)%len(potentialNodes)].Clone()
        stateCopy := ev.State.Clone()

        // SelectVictimsOnNode方法逻辑主要有2段：
        // 1. 模拟移除所有优先级低的pod，判断是否满足抢占要求
        // 2. 如果满足抢占要求，再模拟尽可能少的移除pod
        // 3. 返回的 pods 按照 priority 优先级降序完成了排序
        // pods 表示需要驱逐的pod，numPDBViolations 表示违反PDB约束且需要被驱逐的pod数
        pods, numPDBViolations, status := ev.SelectVictimsOnNode(ctx, stateCopy, pod, nodeInfoCopy, pdbs)

        // 删选 pod 状态成功并且需要被驱逐的 pod 数不为0
        if status.IsSuccess() && len(pods) != 0 {
            victims := extenderv1.Victims{
                Pods:             pods,
                NumPDBViolations: int64(numPDBViolations),
            }
            c := &candidate{
                victims: &victims,
                name:    nodeInfoCopy.Node().Name,
            }
            // 如果没有违反 PDB 约束，就加入到 nonViolatingCandidates 候选节点列表
            // 否则加入到 violatingCandidates 候选节点列表
            if numPDBViolations == 0 {
                nonViolatingCandidates.add(c)
            } else {
                violatingCandidates.add(c)
            }
            nvcSize, vcSize := nonViolatingCandidates.size(), violatingCandidates.size()
            // 找到符合候选节点数量的节点就终止，在超大规模集群下可以提高性能
            if nvcSize > 0 && nvcSize+vcSize >= numCandidates {
                cancel()
            }
            return
        }

        // 没有需要被驱逐的 pod, 则表明该节点在抢占机制中无法发挥作用
        if status.IsSuccess() && len(pods) == 0 {
            status = framework.AsStatus(fmt.Errorf("expected at least one victim pod on node %q", nodeInfoCopy.Node().Name))
        }

        // 对于出错的节点，记录错误原因
        statusesLock.Lock()
        if status.Code() == framework.Error {
            errs = append(errs, status.AsError())
        }
        nodeStatuses[nodeInfoCopy.Node().Name] = status
        statusesLock.Unlock()
    }

    // 异步执行并返回符合条件的节点切片
    fh.Parallelizer().Until(parallelCtx, len(potentialNodes), checkNode)
    return append(nonViolatingCandidates.get(), violatingCandidates.get()...), nodeStatuses, utilerrors.NewAggregate(errs)
}
```

#### 4.1.2 selectCandidate
找到所有的候选节点后，我们需要选择其中的一个节点作为最终的候选节点，方法 `selectCandidate()` 代码如下：
```golang
    func (ev *Evaluator) SelectCandidate(candidates []Candidate) Candidate {
        // 如果没有候选节点，则返回nil
        if len(candidates) == 0 {
            return nil
        }
        // 如果只有一个，则直接返回
        if len(candidates) == 1 {
            return candidates[0]
        }

        // 主要是进行转化，变成 map[nodeName]victims 这种结构
        victimsMap := ev.CandidatesToVictimsMap(candidates)

        // 从所有的节点中选出最终候选节点，PDB 和 victims 数量都是参考维度
        candidateNode := pickOneNodeForPreemption(victimsMap)

        // 如果从 victimsMap 中找到对应的节点，就返回
        if victims := victimsMap[candidateNode]; victims != nil {
            return &candidate{
                victims: victims,
                name:    candidateNode,
            }
        }

        // 否则就走这段逻辑，我理解这里是做了一层保护作用，理论上不应该到这里
        klog.ErrorS(errors.New("no candidate selected"), "Should not reach here", "candidates", candidates)
        // To not break the whole flow, return the first candidate.
        return candidates[0]
    }
```

#### 4.1.3 prepareCandidate
找到对应的抢占节点后就来到了最后我们的准备工作
```golang
func (ev *Evaluator) prepareCandidate(c Candidate, pod *v1.Pod, pluginName string) *framework.Status {
    // 获取 ClientSet
	fh := ev.Handler
	cs := ev.Handler.ClientSet()
	for _, victim := range c.Victims().Pods {
        // 如果 pod 在队列中处于等待状态，就拒绝该轮调度，否则就直接删除
		if waitingPod := fh.GetWaitingPod(victim.UID); waitingPod != nil {
			waitingPod.Reject(pluginName, "preempted")
		} else if err := util.DeletePod(cs, victim); err != nil {
			return framework.AsStatus(err)
		}
	}

    // 由于同一个节点上可能同时存在更低优先级的 pod 也处于抢占周期，此时他们是不应该抢占成功的，
    // 因为即使抢占成功了也会又被驱逐，所以此时清理这些低优先级的抢占 pod
	nominatedPods := getLowerPriorityNominatedPods(fh, pod, c.Name())
	if err := util.ClearNominatedNodeName(cs, nominatedPods...); err != nil {
		klog.ErrorS(err, "Cannot clear 'NominatedNodeName' field")
		// We do not return as this error is not critical.
	}

	return nil
}
```

### 4.2 抢占流程之占
至此，我们的抢逻辑执行完毕，驱逐了优先级更低且必要的 pod，那我们该如何调度呢。我们上面提到 `handleSchedulingFailure` 方法，其中的玄机就在该方法中：
```golang
func (sched *Scheduler) handleSchedulingFailure(fwk framework.Framework, podInfo *framework.QueuedPodInfo, err error, reason string, nominatingInfo *framework.NominatingInfo) {
    // 调度失败的 pod 可以重新入列
	sched.Error(podInfo, err)

    // 把抢占节点及 pod 信息存到 nominator 中
	sched.SchedulingQueue.AddNominatedPod(podInfo.PodInfo, nominatingInfo)

	pod := podInfo.Pod

    // 更新 pod 信息
	if err := updatePod(sched.client, pod, &v1.PodCondition{
		Type:    v1.PodScheduled,
		Status:  v1.ConditionFalse,
		Reason:  reason,
		Message: err.Error(),
	}, nominatingInfo); err != nil {
		klog.ErrorS(err, "Error updating pod", "pod", klog.KObj(pod))
	}
}
```
`handleSchedulingFailure` 方法会处理所有调度失败后续逻辑，其中 `sched.Error` 方法，他的原方法是 `MakeDefaultErrorFunc` 方法，该方法位于 `pkg/scheduler/factory.go` 文件，篇幅有限，我们提一下这个方法有一个很重要的逻辑是会将调度失败的 pod 重新放入调度队列中（后面章节我们会详细介绍该方法），这样调度失败的 pod 会进行重新调度。

`updatePod` 方法也很重要，他会更新 `pod.Status` 的 `NominatedNodeName` 为当前候选节点的 name，这样，在下一次调度的时候可以更快的进行调度，我们回过头去看 `genericScheduler` 的 `findNodesThatFitPod` 方法，其中有一段逻辑：
```golang

func (g *genericScheduler) findNodesThatFitPod(...) ([]*v1.Node, framework.Diagnosis, error) {
    // ...

    // 如果 `pod.Status.NominatedNodeName` 有值，表明这个 pod 经历过抢占流程，这里直接对候选节点进行直接甄别，而不需要重新走一次遍历所有节点的流程，这里的 `feasibleNodes` 看起来有很多节点，其实只有一个候选节点。如果候选节点不符合要求，`feasibleNodes` 则为空
	if len(pod.Status.NominatedNodeName) > 0 {
		feasibleNodes, err := g.evaluateNominatedNode(ctx, extenders, pod, fwk, state, diagnosis)
		if err != nil {
			klog.ErrorS(err, "Evaluation failed on nominated node", "pod", klog.KObj(pod), "node", pod.Status.NominatedNodeName)

        // 如果候选节点符合要求，则直接返回，不需要再去找其他节点了。
		if len(feasibleNodes) != 0 {
			return feasibleNodes, diagnosis, nil
		}
	}

    // ...
```
至此，整个抢占机制算是走通了。

## `nominator` 提名器
我们在 [`prepareCandidate`](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/priority-preemption.md#413-preparecandidate)章节遇到过一个方法叫 `getLowerPriorityNominatedPods`，他可以获取某一个节点上所有同样处于抢占周期且优先级较低的 pod，他是如何实现的，其实是通过一个叫 `NominatedPodsForNode` 的方法，说到这，就不得不提一个结构体变量，叫 `nominator`，我们看一下：
```golang
type nominator struct {
	// podLister is used to verify if the given pod is alive.
	podLister listersv1.PodLister
	// nominatedPods is a map keyed by a node name and the value is a list of
	// pods which are nominated to run on the node. These are pods which can be in
	// the activeQ or unschedulableQ.
	nominatedPods map[string][]*framework.PodInfo
	// nominatedPodToNode is map keyed by a Pod UID to the node name where it is
	// nominated.
	nominatedPodToNode map[types.UID]string

	sync.RWMutex
}
```
这个数据结构就是用来存储所有的抢占结果，它实现了 `PodNominator` 接口：
```golang
type PodNominator interface {
	// AddNominatedPod adds the given pod to the nominator or
	// updates it if it already exists.
	AddNominatedPod(pod *PodInfo, nominatingInfo *NominatingInfo)
	// DeleteNominatedPodIfExists deletes nominatedPod from internal cache. It's a no-op if it doesn't exist.
	DeleteNominatedPodIfExists(pod *v1.Pod)
	// UpdateNominatedPod updates the <oldPod> with <newPod>.
	UpdateNominatedPod(oldPod *v1.Pod, newPodInfo *PodInfo)
	// NominatedPodsForNode returns nominatedPods on the given node.
	NominatedPodsForNode(nodeName string) []*PodInfo
}
```
这个接口实现了对 `nominator` 的增删改操作，这样，当有需要时我们就可以通过 `NominatedPodsForNode` 获取到 node 节点所有的 nominatedPod。

## 总结
以上就是整个抢占调度的全部逻辑，如果有说的不对不好的，欢迎指正！