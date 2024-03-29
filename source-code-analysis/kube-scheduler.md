## kube-scheduler
### kube-scheduler配置文件
```golang
// pkg/scheduler/apis/config/types.go +54
// KubeSchedulerConfiguration configures a scheduler
type KubeSchedulerConfiguration struct {
	metav1.TypeMeta

	// Parallelism defines the amount of parallelism in algorithms for scheduling a Pods. Must be greater than 0. Defaults to 16
	Parallelism int32

	AlgorithmSource SchedulerAlgorithmSource

	LeaderElection componentbaseconfig.LeaderElectionConfiguration

	ClientConnection componentbaseconfig.ClientConnectionConfiguration

	HealthzBindAddress string

	MetricsBindAddress string

	componentbaseconfig.DebuggingConfiguration

	PercentageOfNodesToScore int32

	PodInitialBackoffSeconds int64

	PodMaxBackoffSeconds int64

	Profiles []KubeSchedulerProfile

	Extenders []Extender
}

// KubeSchedulerProfile is a scheduling profile.
type KubeSchedulerProfile struct {
	// SchedulerName is the name of the scheduler associated to this profile.
	// If SchedulerName matches with the pod's "spec.schedulerName", then the pod
	// is scheduled with this profile.
	SchedulerName string

	// Plugins specify the set of plugins that should be enabled or disabled.
	// Enabled plugins are the ones that should be enabled in addition to the
	// default plugins. Disabled plugins are any of the default plugins that
	// should be disabled.
	// When no enabled or disabled plugin is specified for an extension point,
	// default plugins for that extension point will be used if there is any.
	// If a QueueSort plugin is specified, the same QueueSort Plugin and
	// PluginConfig must be specified for all profiles.
	Plugins *Plugins

	// PluginConfig is an optional set of custom plugin arguments for each plugin.
	// Omitting config args for a plugin is equivalent to using the default config
	// for that plugin.
	PluginConfig []PluginConfig
}
```

scheduler配置文件的默认配置

```golang
// cmd/kub-scheduler/app/options/options.go +138
func newDefaultComponentConfig() (*kubeschedulerconfig.KubeSchedulerConfiguration, error) {
	versionedCfg := kubeschedulerconfigv1beta1.KubeSchedulerConfiguration{}
	versionedCfg.DebuggingConfiguration = *configv1alpha1.NewRecommendedDebuggingConfiguration()

	kubeschedulerscheme.Scheme.Default(&versionedCfg)
	cfg := kubeschedulerconfig.KubeSchedulerConfiguration{}
	if err := kubeschedulerscheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
		return nil, err
	}
	return &cfg, nil
}
// kube-scheduler的配置文件由v1beta1版本转换为v1版本，而其中v1beta1版本的默认配置见下面函数
// pkg/scheduler/apis/config/v1beta1/defaults.go +46
func SetDefaults_KubeSchedulerConfiguration(obj *v1beta1.KubeSchedulerConfiguration) {
    ...
}

```
```golang
// v1beta1版本相比于v1版本少了AlgorithmSource的字段，改字段的转换需要手动转换
// pkg/scheduler/apis/config/v1beta1/conversion.go +49
func Convert_v1beta1_KubeSchedulerConfiguration_To_config_KubeSchedulerConfiguration(in *v1beta1.KubeSchedulerConfiguration, out *config.KubeSchedulerConfiguration, s conversion.Scope) error {
	if err := autoConvert_v1beta1_KubeSchedulerConfiguration_To_config_KubeSchedulerConfiguration(in, out, s); err != nil {
		return err
	}
    // v1beta1.SchedulerDefaultProviderName = "DefaultProvider"
	out.AlgorithmSource.Provider = pointer.StringPtr(v1beta1.SchedulerDefaultProviderName)
	return convertToInternalPluginConfigArgs(out)
}

```

如果kube-scheduler的下面三个参数配置了其中一个，那么AlgorithmSource将会被重新赋值
```golang
// ApplyAlgorithmSourceTo sets cfg.AlgorithmSource from flags passed on the command line in the following precedence order:
//
// 1. --use-legacy-policy-config to use a policy file.
// 2. --policy-configmap to use a policy config map value.
// 3. --algorithm-provider to use a named algorithm provider.
func (o *DeprecatedOptions) ApplyAlgorithmSourceTo(cfg *kubeschedulerconfig.KubeSchedulerConfiguration) {
	if o == nil {
		return
	}

	switch {
	case o.UseLegacyPolicyConfig || (len(o.PolicyConfigFile) > 0 && o.PolicyConfigMapName == ""):
		cfg.AlgorithmSource = kubeschedulerconfig.SchedulerAlgorithmSource{
			Policy: &kubeschedulerconfig.SchedulerPolicySource{
				File: &kubeschedulerconfig.SchedulerPolicyFileSource{
					Path: o.PolicyConfigFile,
				},
			},
		}
	case len(o.PolicyConfigMapName) > 0:
		cfg.AlgorithmSource = kubeschedulerconfig.SchedulerAlgorithmSource{
			Policy: &kubeschedulerconfig.SchedulerPolicySource{
				ConfigMap: &kubeschedulerconfig.SchedulerPolicyConfigMapSource{
					Name:      o.PolicyConfigMapName,
					Namespace: o.PolicyConfigMapNamespace,
				},
			},
		}
	case len(o.AlgorithmProvider) > 0:
		cfg.AlgorithmSource = kubeschedulerconfig.SchedulerAlgorithmSource{
			Provider: &o.AlgorithmProvider,
		}
	}
}
```

scheduler默认的plugins插件配置

```golang
// pkg/scheduler/algorithmprovider/registry.go +71
func getDefaultConfig() *schedulerapi.Plugins {
	return &schedulerapi.Plugins{
		QueueSort: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: queuesort.Name},
			},
		},
		PreFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.FitName},
				{Name: nodeports.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
				{Name: volumebinding.Name},
			},
		},
		Filter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: nodeunschedulable.Name},
				{Name: nodename.Name},
				{Name: tainttoleration.Name},
				{Name: nodeaffinity.Name},
				{Name: nodeports.Name},
				{Name: noderesources.FitName},
				{Name: volumerestrictions.Name},
				{Name: nodevolumelimits.EBSName},
				{Name: nodevolumelimits.GCEPDName},
				{Name: nodevolumelimits.CSIName},
				{Name: nodevolumelimits.AzureDiskName},
				{Name: volumebinding.Name},
				{Name: volumezone.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
			},
		},
		PostFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultpreemption.Name},
			},
		},
		PreScore: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: interpodaffinity.Name},
				{Name: podtopologyspread.Name},
				{Name: tainttoleration.Name},
			},
		},
		Score: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.BalancedAllocationName, Weight: 1},
				{Name: imagelocality.Name, Weight: 1},
				{Name: interpodaffinity.Name, Weight: 1},
				{Name: noderesources.LeastAllocatedName, Weight: 1},
				{Name: nodeaffinity.Name, Weight: 1},
				{Name: nodepreferavoidpods.Name, Weight: 10000},
				// Weight is doubled because:
				// - This is a score coming from user preference.
				// - It makes its signal comparable to NodeResourcesLeastAllocated.
				{Name: podtopologyspread.Name, Weight: 2},
				{Name: tainttoleration.Name, Weight: 1},
			},
		},
		Reserve: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		PreBind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		Bind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultbinder.Name},
			},
		},
	}
}
```

```kube-scheduler```核心调度函数代码

整个调度的过程：preFilter -> filter -> preScore -> score -> selectHost -> assumePod -> 

```golang
// pkg/scheduler/scheduler.go +440
// scheduleOne does the entire scheduling workflow for a single pod. It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
	// pod could be nil when schedulerQueue is closed
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
	pod := podInfo.Pod
	// 根据Pod指定的调度器名字(Pod.Spec.SchedulerName)选择Framework
	fwk, err := sched.frameworkForPod(pod)
	if err != nil {
		// This shouldn't happen, because we only accept for scheduling the pods
		// which specify a scheduler name that matches one of the profiles.
		klog.Error(err)
		return
	}
	// 判断pod是否需要忽略
	// 1、已经被删除的pod需要忽略，即pod.DeletionTimestamp != nil
	// 2、pod已经被先假定已经调度成功了
	// 3、在调度期间，pod会因为被更新而重新置入调度队列(EventHandlers)，如果是因为绑定过程中更新的信息，则需要忽略
	//    需要忽略的信息ResourceVersion、Spec.NodeName、Annotations、ManagedFields、Finalizers、Status.Conditions
	if sched.skipPodSchedule(fwk, pod) {
		return
	}

	klog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	state := framework.NewCycleState()
	state.SetRecordPluginMetrics(rand.Intn(100) < pluginMetricsSamplePercent)
	schedulingCycleCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, fwk, state, pod)
	if err != nil {
		// Schedule() may have failed because the pod would not fit on any host, so we try to
		// preempt, with the expectation that the next time the pod is tried for scheduling it
		// will fit due to the preemption. It is also possible that a different pod will schedule
		// into the resources that were preempted, but this is harmless.
		// 调度失败则尝试运行postFilter的插件进行强占调度。
		// 需要注意的是，即便抢占成功，Pod当前依然是不可调度状态，因为需要等待被强占的Pod退出，所以nominatedNode是否为空就可以判断是否抢占成功了。
		// 强占的资源也有可能被其他优先级别更高的pod占用，但这是无害的
		nominatedNode := ""
		if fitError, ok := err.(*core.FitError); ok {
			if !fwk.HasPostFilterPlugins() {
				klog.V(3).Infof("No PostFilter plugins are registered, so no preemption will be performed.")
			} else {
				// Run PostFilter plugins to try to make the pod schedulable in a future scheduling cycle.
				result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.FilteredNodesStatuses)
				if status.Code() == framework.Error {
					klog.Errorf("Status after running PostFilter plugins for pod %v/%v: %v", pod.Namespace, pod.Name, status)
				} else {
					klog.V(5).Infof("Status after running PostFilter plugins for pod %v/%v: %v", pod.Namespace, pod.Name, status)
				}
				if status.IsSuccess() && result != nil {
					nominatedNode = result.NominatedNodeName
				}
			}
			// Pod did not fit anywhere, so it is counted as a failure. If preemption
			// succeeds, the pod should get counted as a success the next time we try to
			// schedule it. (hopefully)
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else if err == core.ErrNoNodesAvailable {
			// No nodes available is counted as unschedulable rather than an error.
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else {
			klog.ErrorS(err, "Error selecting node for pod", "pod", klog.KObj(pod))
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		}
		// recordSchedulingFailure()用于统一实现调度pod失败的处理。
		// 包括将调度失败的原因更新到pod的event，将调度失败的pod重新放入调度队列等
		sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
		return
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod
	// kube-scheduler是通过异步的方式实现Bind，在Bind完成前，调度器还要调度新的Pod，此时就先假定Pod调度完成了。
	// Bind可以简单理解为：需要将Pod的调度结果写入etcd，持久化调度结果，所以也是相对比较耗时的操作。
	// AssumePod会将Pod的资源需求累加到Node上，这样kube-scheduler在调度其他Pod的时候，就不会占用这部分资源。
	// assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// This is most probably result of a BUG in retrying logic.
		// We report an error here so that pod scheduling can be retried.
		// This relies on the fact that Error will check if the pod has been bound
		// to a node and if so will not add it back to the unscheduled pods queue
		// (otherwise this would cause an infinite loop).
		sched.recordSchedulingFailure(fwk, assumedPodInfo, err, SchedulerError, "")
		return
	}

	// Run the Reserve method of reserve plugins.
	if sts := fwk.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// trigger un-reserve to clean up state associated with the reserved Pod
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, sts.AsError(), SchedulerError, "")
		return
	}

	// Run "permit" plugins.
	runPermitStatus := fwk.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		var reason string
		if runPermitStatus.IsUnschedulable() {
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = v1.PodReasonUnschedulable
		} else {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = SchedulerError
		}
		// One of the plugins returned status different than success or wait.
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, runPermitStatus.AsError(), reason, "")
		return
	}
	// 创建一个异步协程执行绑定操作。异步的原因是因为绑定是一个相对比较耗时的操作，至少包含一次向apiserver写入绑定信息的操作
	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()
		metrics.SchedulerGoroutines.WithLabelValues("binding").Inc()
		defer metrics.SchedulerGoroutines.WithLabelValues("binding").Dec()

		// 等待pod批准通过，如果有permitPlugin返回等待，pod就会被放入waitingPodsMap直到所有的permitPlugin批准通过
		// k8s在1.20版本及之前都还没实现permitPlugin，不确定在后续的版本会不会实现
		waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
		if !waitOnPermitStatus.IsSuccess() {
			var reason string
			if waitOnPermitStatus.IsUnschedulable() {
				metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = SchedulerError
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, "")
			return
		}

		// 执行预绑定的插件
		// Run "prebind" plugins.
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, "")
			return
		}

		// 执行绑定的插件，主要是向apiserver写入pod的资资源bind
		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", err)
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, "")
		} else {
			// Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
			if klog.V(2).Enabled() {
				klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
			}
			metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
			metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
			metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

			// Run "postbind" plugins.
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}


```

```golang
// pkg/scheduler/core/generic_scheduler.go +128

// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

	// 从kube-scheduler的缓存中更新一份NodeInfo的快照，在每次调度之前都会更新一次
    // pkg/scheduler/internal/cache/cache.go +203
	if err := g.snapshot(); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}
    
	// 这里会执行preFilter跟filter的插件
	// preFilter: pkg/scheduler/framework/runtime/framework.go +419
	//     这里主要是在收集pod的request信息跟cluster节点的资源信息，为下一步的filter做准备
	//     某些插件的preFilter也可能将pod直接置为不可调度(如interpodaffinity、volumebinding插件)，此时整个调度直接结束
	// filter: pkg/scheduler/core/generic_scheduler.g +258
	feasibleNodes, filteredNodesStatuses, err := g.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")

	if len(feasibleNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           g.nodeInfoSnapshot.NumNodes(),
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}

	// When only one node after predicate, just use it.
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(filteredNodesStatuses),
			FeasibleNodes:  1,
		}, nil
	}
	// 给筛选之后的node节点打分
	// 这里会执行preScore跟Score的插件
	// preScore: pkg/scheduler/framework/runtime/framework.go +621
	//      这里主要是在收集pod的request信息跟cluster节点的资源信息，为下一步的score做准备
	//      这里的插件也有可能直接返回错误，让调度提前失败
	// score: 给筛选之后的node节点打分
	priorityList, err := g.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}
	// 从打完分的node列表中选出分数最高的node
	// 如有多个node的分数最高，则从中随机选一个（这里随机选的实现很优雅）
	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(filteredNodesStatuses),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}

```