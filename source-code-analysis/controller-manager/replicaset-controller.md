## replicaset-controller
根据rs-contorller的初始化代码，可以看出rs-controller只监听rs对象跟pod对象的增删改。每种资源的变动都会触发不同的逻辑
```golang
// kubernetes/pkg/controller/replicaset/replica_set.go +127
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
    gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
    if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
        ratelimiter.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
    }

    rsc := &ReplicaSetController{
        GroupVersionKind: gvk,
        kubeClient:       kubeClient,
        podControl:       podControl,
        burstReplicas:    burstReplicas,
        expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
        queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
    }

    rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    rsc.addRS,
        UpdateFunc: rsc.updateRS,
        DeleteFunc: rsc.deleteRS,
    })
    rsc.rsLister = rsInformer.Lister()
    rsc.rsListerSynced = rsInformer.Informer().HasSynced

    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: rsc.addPod,
        // This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
        // overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
        // local storage, so it should be ok.
        UpdateFunc: rsc.updatePod,
        DeleteFunc: rsc.deletePod,
    })
    rsc.podLister = podInformer.Lister()
    rsc.podListerSynced = podInformer.Informer().HasSynced

    rsc.syncHandler = rsc.syncReplicaSet

    return rsc
}
```
### pod的修改
```golang
// kubernetes/pkg/controller/replicaset/replica_set.go +396
func (rsc *ReplicaSetController) updatePod(old, cur interface{})
    curPod := cur.(*v1.Pod)
    oldPod := old.(*v1.Pod)
    if curPod.ResourceVersion == oldPod.ResourceVersion {
    // 新旧pod的ResourceVersion一致，代表无变化，直接返回
    // Periodic resync will send update events for all known pods.
    // Two different versions of the same pod will always have different RVs.
        return
    }
    
    labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
    if curPod.DeletionTimestamp != nil {
    // pod的DeletionTimestamp不为空，表示pod处于删除状态，直接调用rsc.deletePod()处理
    // when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period, 
    // and after such time has passed, the kubelet actually deletes it from the store. We receive an update 
    // for modification of the deletion timestamp and expect an rs to create more replicas asap, not wait 
    // until the kubelet actually deletes the pod. This is different from the Phase of a pod changing, because 
    // an rs never initiates a phase change, and so is never asleep waiting for the same. 
        rsc.deletePod(curPod)
        if labelChanged {
            // we don't need to check the oldPod.DeletionTimestamp because DeletionTimestamp cannot be unset. 
            rsc.deletePod(oldPod)
        }
        return
    }
    
    curControllerRef := metav1.GetControllerOf(curPod)
    oldControllerRef := metav1.GetControllerOf(oldPod)
    controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
    if controllerRefChanged && oldControllerRef != nil {
        // The ControllerRef was changed. Sync the old controller, if any. 
        if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
            rsc.enqueueRS(rs)
        }
    }

    // If it has a ControllerRef, that's all that matters.
    if curControllerRef != nil {
        rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
        if rs == nil {
            return
        }
        klog.V(4).Infof("Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
        rsc.enqueueRS(rs)
        // TODO: MinReadySeconds in the Pod will generate an Available condition to be added in
        // the Pod status which in turn will trigger a requeue of the owning replica set thus
        // having its status updated with the newly available replica. For now, we can fake the
        // update by resyncing the controller MinReadySeconds after the it is requeued because
        // a Pod transitioned to Ready.
        // Note that this still suffers from #29229, we are just moving the problem one level
        // "closer" to kubelet (from the deployment to the replica set controller).
        if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
            // oldPod notready并且curPod ready，证明本次pod的update至少有status的更新
            // 如果MinReadySeconds等于0(默认值)，则需要马上更新，则需要马上将rs中available + 1，这一步会在当前函数的上一个rsc.enqueueRS(rs)中体现
            // 如果MinReadySeconds大于0，则需在等待(MinReadySeconds + 1)秒之后再rs中available + 1(这里加1s是为了避免偏差)。
            // 当前函数的上一个rsc.enqueueRS(rs)会自动判断这里的逻辑觉得要不要available + 1
            klog.V(2).Infof("%v %q will be enqueued after %ds for availability check", rsc.Kind, rs.Name, rs.Spec.MinReadySeconds)
            // Add a second to avoid milliseconds skew in AddAfter.
            // See https://github.com/kubernetes/kubernetes/issues/39785#issuecomment-279959133 for more info.
            rsc.enqueueRSAfter(rs, (time.Duration(rs.Spec.MinReadySeconds)*time.Second)+time.Second)
        }
        return
    }

    // Otherwise, it's an orphan. If anything changed, sync matching controllers
    // to see if anyone wants to adopt it now.
    if labelChanged || controllerRefChanged {
        rss := rsc.getPodReplicaSets(curPod)
        if len(rss) == 0 {
            return
        }
        klog.V(4).Infof("Orphan Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
        for _, rs := range rss {
            rsc.enqueueRS(rs)
        }
    }
}
```
### pod的删除
```golang
// When a pod is deleted, enqueue the replica set that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (rsc *ReplicaSetController) deletePod(obj interface{}) {
    pod, ok := obj.(*v1.Pod)

    // When a delete is dropped, the relist will notice a pod in the store not
    // in the list, leading to the insertion of a tombstone object which contains
    // the deleted key/value. Note that this value might be stale. If the pod
    // changed labels the new ReplicaSet will not be woken up till the periodic resync.
    if !ok {
        tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
            return
        }
        pod, ok = tombstone.Obj.(*v1.Pod)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %#v", obj))
            return
        }
    }

    controllerRef := metav1.GetControllerOf(pod)
    if controllerRef == nil {
        // No controller should care about orphans being deleted.
        return
    }
    rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
    if rs == nil {
        return
    }
    rsKey, err := controller.KeyFunc(rs)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
        return
    }
    klog.V(4).Infof("Pod %s/%s deleted through %v, timestamp %+v: %#v.", pod.Namespace, pod.Name, utilruntime.GetCaller(), pod.DeletionTimestamp, pod)
    rsc.expectations.DeletionObserved(rsKey, controller.PodKey(pod))
    rsc.queue.Add(rsKey)
}
```
### rs的expectations机制
rs的controller维护了一个```ControllerExpectations```的数据结构，存储的对象类型为```ControlleeExpectations```，其中key值为```<rs_namespace>/<rs_name>```
```golang
// ControllerExpectations is a cache mapping controllers to what they expect to see before being woken up for a sync.
type ControllerExpectations struct {
    cache.Store
}

// ControlleeExpectations track controllee creates/deletes.
type ControlleeExpectations struct {
    // Important: Since these two int64 fields are using sync/atomic, they have to be at the top of the struct due to a bug on 32-bit platforms
    // See: https://golang.org/pkg/sync/atomic/ for more information
    add       int64
    del       int64
    key       string
    timestamp time.Time
}
```



### 如何判断一个pod是否available
同时满足两个条件：
1. pod已经ready
2. deployment没设置minReadySeconds或者设置了minReadySeconds但是pod ready持续的时间已经超过了minReadySeconds
```golang
// kubernetes/pkg/api/v1/pod/util.go +281
// IsPodAvailable returns true if a pod is available; false otherwise.
// Precondition for an available pod is that it must be ready. On top
// of that, there are two cases when a pod can be considered available:
// 1. minReadySeconds == 0, or
// 2. LastTransitionTime (is set) + minReadySeconds < current time
func IsPodAvailable(pod *v1.Pod, minReadySeconds int32, now metav1.Time) bool {
    if !IsPodReady(pod) {
        return false
    }

    c := GetPodReadyCondition(pod.Status)
    minReadySecondsDuration := time.Duration(minReadySeconds) * time.Second
    if minReadySeconds == 0 || !c.LastTransitionTime.IsZero() && c.LastTransitionTime.Add(minReadySecondsDuration).Before(now.Time) {
        return true
    }
    return false
}
```

### 如果rs的replicas超过预期，k8s筛选删除pod的逻辑
根据下面的排序条件，从上到下进行排序，满足其中一个条件则排序完成：
1. 优先删除```unassigned```的pod，即没有绑定到node的pod
2. 优先删除处于```pending```状态的pod，然后是```unknow```，最后是```running```状态
3. 优先删除notready的pod
4. 按同node上所属replicaset的pod数量排序，优先删除所属replicaset的pod数量多的node上的pod
5. 优先删除pod ready持续时间最短的pod
6. 优先删除pod中容器重启次数较多的pod
7. 优先删除pod创建时间最短的pod
```golang
// kubernetes/pkg/controller/controller_utils.go +834
// Less compares two pods with corresponding ranks and returns true if the first
// one should be preferred for deletion.
func (s ActivePodsWithRanks) Less(i, j int) bool {
    // 1. Unassigned < assigned
    // If only one of the pods is unassigned, the unassigned one is smaller
    if s.Pods[i].Spec.NodeName != s.Pods[j].Spec.NodeName && (len(s.Pods[i].Spec.NodeName) == 0 || len(s.Pods[j].Spec.NodeName) == 0) {
        return len(s.Pods[i].Spec.NodeName) == 0
    }
    // 2. PodPending < PodUnknown < PodRunning
    if podPhaseToOrdinal[s.Pods[i].Status.Phase] != podPhaseToOrdinal[s.Pods[j].Status.Phase] {
        return podPhaseToOrdinal[s.Pods[i].Status.Phase] < podPhaseToOrdinal[s.Pods[j].Status.Phase]
    }
    // 3. Not ready < ready
    // If only one of the pods is not ready, the not ready one is smaller
    if podutil.IsPodReady(s.Pods[i]) != podutil.IsPodReady(s.Pods[j]) {
        return !podutil.IsPodReady(s.Pods[i])
    }
    // 4. Doubled up < not doubled up
    // If one of the two pods is on the same node as one or more additional
    // ready pods that belong to the same replicaset, whichever pod has more
    // colocated ready pods is less
    if s.Rank[i] != s.Rank[j] {
        return s.Rank[i] > s.Rank[j]
    }
    // TODO: take availability into account when we push minReadySeconds information from deployment into pods,
    //       see https://github.com/kubernetes/kubernetes/issues/22065
    // 5. Been ready for empty time < less time < more time
    // If both pods are ready, the latest ready one is smaller
    if podutil.IsPodReady(s.Pods[i]) && podutil.IsPodReady(s.Pods[j]) {
        readyTime1 := podReadyTime(s.Pods[i])
        readyTime2 := podReadyTime(s.Pods[j])
        if !readyTime1.Equal(readyTime2) {
            return afterOrZero(readyTime1, readyTime2)
        }
    }
    // 6. Pods with containers with higher restart counts < lower restart counts
    if maxContainerRestarts(s.Pods[i]) != maxContainerRestarts(s.Pods[j]) {
        return maxContainerRestarts(s.Pods[i]) > maxContainerRestarts(s.Pods[j])
    }
    // 7. Empty creation time pods < newer pods < older pods
    if !s.Pods[i].CreationTimestamp.Equal(&s.Pods[j].CreationTimestamp) {
        return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
    }
    return false
}
```