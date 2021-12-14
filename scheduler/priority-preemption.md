# scheduler 优先级和抢占

## 1. 开篇
Pod 可以有 优先级。 优先级表示一个 Pod 相对于其他 Pod 的重要性。 如果一个 Pod 无法被调度，调度程序会尝试抢占（驱逐）较低优先级的 Pod， 以使悬决 Pod 可以被调度。这就是 `kubernetes` 优先级于抢占机制。今天我们一起从源码角度看一下它是如何实现的。

## 2. 如何设置 `pod` 优先级
我们一般通过 [`PriorityClass`](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/pod-priority-preemption/) 来设置 `pod` 优先级的方法，它是通过 `AdmissionController` 来实现的，主要代码逻辑位于 `plugin/pkg/admission/priority` 的 `Admit()` 方法：

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

通过 `admitPod()` 方法，我们就完成对 `pod` 优先级和抢占策略的设置。

## 3. 优先级调度
`pod` 完成优先级设置，之后会进入到调度流程，调度队列会根据优先级进行排序，我们在[优先级调度](https://github.com/kerthcet/kubernetes-design/blob/main/scheduler/queue.md#2-scheduler-%E4%B8%89%E7%BA%A7%E9%98%9F%E5%88%97)中讲 `activeQ` 的时候已经介绍过，`activeQ` 是一个优先级队列，会根据 `pod.Spec.Priority` 值进行排序，`Priority` 值越大，优先级越高，排在队列的越前面，所以会被优先调度。

## 4. 抢占机制
`Preemption` 现在是以一个插件的形式工作在 `PostFilter` 这个扩展点，这个插件名叫 `DefaultPreemption`，是一个默认的内置插件，代码位于 `pkg/scheduler/framework/plugins/defaultpreemption:default_preemption.go`。

我们先看一下 `PostFilter` 扩展点代码逻辑（由于本篇文章着重介绍抢占，其他扩展点逻辑不展开）：

    func (sched *Scheduler) scheduleOne(ctx context.Context) {
        // ...省略其他调度逻辑

        // 调度
        scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, sched.Extenders, fwk, state, pod)

        // 如果找不到合适的节点，就进行抢占调度
        if err != nil {
            // 声明提名节点
            nominatedNode := ""
            if fitError, ok := err.(*framework.FitError); ok {
                // 执行抢占逻辑
                result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatusMap)

                // 如果抢占成功，则提名节点
                if status.IsSuccess() && result != nil {
                    nominatedNode = result.NominatedNodeName
                }
            }

            // 处理调度失败相关逻辑，比如记录调度失败的 event，如果抢占成功，更新 pod 信息等，并且该方法通过 `sched.Error()` 触发重新入队的操作，这样，pod 可能在下个调度周期调度到抢占的节点，为什么是可能，我们后面讲抢占调度逻辑的时候会详细介绍。
            sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
            return
        }

        // ...省略其他调度逻辑
    }

### 4.1 抢占流程
抢占逻辑位于代码 `RunPostFilterPlugins()` 方法中，它会遍历每个 `Plugin`，并调用他们的 `PostFilter()` 方法，我们一起看一下默认 `PostFilter` 扩展点插件 `DefaultPreemption` 的方法：

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

我们继续看 `pe.Preempt()` 方法：

    func (ev *Evaluator) Preempt(ctx context.Context, pod *v1.Pod, m framework.NodeToStatusMap) (*framework.PostFilterResult, *framework.Status) {

        podNamespace, podName := pod.Namespace, pod.Name
        pod, err := ev.PodLister.Pods(pod.Namespace).Get(pod.Name)

        // 检查pod是否具备抢占条件，一是 pod.Status.NominatedNodeName 有值，二是此时集群中没有pod正在被删除
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

这就是整个抢占机制的大致流程，下面我们就其中的 `findCandidates()`，`selectCandidate()` 和 `prepareCandidate()` 这三个最核心的方法分别进行讲解。

### 4.2 findCandidates
`findCandidates()` 顾名思义就是寻找所有可能的候选节点，那它是基于何种规则进行删选呢，我们一起看一下：

    func (ev *Evaluator) findCandidates(ctx context.Context, pod *v1.Pod, m framework.NodeToStatusMap) ([]Candidate, framework.NodeToStatusMap, error) {
        // 获取所有nodes
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

#### 4.2.1 DryRunPreemption
findCandidates() 方法中最重要的逻辑位于 DryRunPreemption 中，我们一起看一下它的逻辑：

    func (ev *Evaluator) DryRunPreemption(...) ([]Candidate, framework.NodeToStatusMap, error) {
        // ...

        checkNode := func(i int) {
            // 前面的位移值在这里派上了用场
            nodeInfoCopy := potentialNodes[(int(offset)+i)%len(potentialNodes)].Clone()
            stateCopy := ev.State.Clone()

            // SelectVictimsOnNode方法逻辑主要有2段：
            // 1. 模拟移除所有优先级低的pod，判断是否满足抢占要求
            // 2. 如果满足抢占要求，再模拟尽可能少的移除pod
            pods, numPDBViolations, status := ev.SelectVictimsOnNode(ctx, stateCopy, pod, nodeInfoCopy, pdbs)

            // 模拟抢占成功并且驱逐pod数不为0
            if status.IsSuccess() && len(pods) != 0 {
                victims := extenderv1.Victims{
                    Pods:             pods,
                    NumPDBViolations: int64(numPDBViolations),
                }
                c := &candidate{
                    victims: &victims,
                    name:    nodeInfoCopy.Node().Name,
                }
                if numPDBViolations == 0 {
                    nonViolatingCandidates.add(c)
                } else {
                    violatingCandidates.add(c)
                }
                nvcSize, vcSize := nonViolatingCandidates.size(), violatingCandidates.size()
                if nvcSize > 0 && nvcSize+vcSize >= numCandidates {
                    cancel()
                }
                return
            }

            // 模拟抢占成功但是驱逐pod数为0，这显然不对
            if status.IsSuccess() && len(pods) == 0 {
                status = framework.AsStatus(fmt.Errorf("expected at least one victim pod on node %q", nodeInfoCopy.Node().Name))
            }

            // 模拟抢占失败，记录失败原因
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



## `nominator` 提名器