### kube-scheduler

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

```kube-scheduler```核心调度代码
```golang
// pkg/scheduler/core/generic_scheduler.go +128

// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

	if err := g.snapshot(); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

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

	priorityList, err := g.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(filteredNodesStatuses),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}

```