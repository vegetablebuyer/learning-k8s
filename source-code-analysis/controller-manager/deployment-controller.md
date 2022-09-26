## deployment-controller
[上面提到](https://github.com/vegetablebuyer/learning-k8s/blob/main/k8s-controller.md) ，每一个资源的controller的核心调谐的逻辑代码在```c.syncHandler()```中，而对应deployment-controller的这段代码

```golang
// kubernetes/pkg/controller/deployment/deployment_controller.go +100
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
    ...   
    dc := &DeploymentController{
        client:        client,
        eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
        queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
    }
    ...
    // deployment-controller核心的调谐代码在dc.syncDeployment
    dc.syncHandler = dc.syncDeployment
    ...
    return dc, nil
}
```

```dc.syncDeployment()```
```golang
// kubernetes/pkg/controller/deployment/deployment_controller.go +566

func (dc *DeploymentController) syncDeployment(key string) error {
    startTime := time.Now()
    klog.V(4).Infof("Started syncing deployment %q (%v)", key, startTime)
    defer func() {
    klog.V(4).Infof("Finished syncing deployment %q (%v)", key, time.Since(startTime))
    }()

    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
    return err
    }
    // 从informer cache中获取deployment对象
    deployment, err := dc.dLister.Deployments(namespace).Get(name)
    if errors.IsNotFound(err) {
        klog.V(2).Infof("Deployment %v has been deleted", key)
        return nil
    }
    if err != nil {
        return err
    }

    // Deep-copy otherwise we are mutating our cache.
    // TODO: Deep-copy only when needed.
    // 深度拷贝deployment对象的副本，避免修改到informer cache的内容
    d := deployment.DeepCopy()

    everything := metav1.LabelSelector{}
    if reflect.DeepEqual(d.Spec.Selector, &everything) {
        dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
        // 假如deployment.status.observedGeneration小于deployment.generation，将前者的值更新为后者
        if d.Status.ObservedGeneration < d.Generation {
            d.Status.ObservedGeneration = d.Generation
            dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(context.TODO(), d, metav1.UpdateOptions{})
        }
        return nil
    }

    // List ReplicaSets owned by this Deployment, while reconciling ControllerRef
    // through adoption/orphaning.
    // 1、列出当前namespace下所有的rs
    // 2、将所有孤儿rs(没有ownerReferences[*].controller=true的rs)中label selector匹配的rs收养为当前deploy的rs，即绑定ownerReferences关系
    // 3、将ownerReferences[*].uid=deploy.uid但是label selector不匹配的rs解除ownerReferences关系
    // 4、返回跟当前deploy有ownerReferences关系的rs列表
    rsList, err := dc.getReplicaSetsForDeployment(d)
    if err != nil {
        return err
    }
    // List all Pods owned by this Deployment, grouped by their ReplicaSet.
    // Current uses of the podMap are:
    //
    // * check if a Pod is labeled correctly with the pod-template-hash label.
    // * check that no old Pods are running in the middle of Recreate Deployments.
    // 获取当前deployment下面所有的pod，以下面的数据结构用rs.UID 对 pod 进行分类
    // podMap = {
    //  rs1.uid: [pod1, pod2...],
    //  rs2.uid: [pod1, pod2...],
    //  ...
    // }
    podMap, err := dc.getPodMapForDeployment(d, rsList)
    if err != nil {
        return err
    }
    // 如果该deployment处于删除的状态(打上了DeletionTimestamp的值)，只更新deployment的status
    // 通过遍历deployment关联的所有rs，计算出deployment的status，并更新
    // kubernetes/pkg/controller/deployment/sync.go +496
    // status := apps.DeploymentStatus{
    //    TODO: Ensure that if we start retrying status updates, we won't pick up a new Generation value.
    //    ObservedGeneration:  deployment.Generation,
    //    Replicas:            deploymentutil.GetActualReplicaCountForReplicaSets(allRSs),
    //    UpdatedReplicas:     deploymentutil.GetActualReplicaCountForReplicaSets([]*apps.ReplicaSet{newRS}),
    //    ReadyReplicas:       deploymentutil.GetReadyReplicaCountForReplicaSets(allRSs),
    //    AvailableReplicas:   availableReplicas,
    //    UnavailableReplicas: unavailableReplicas,
    //    CollisionCount:      deployment.Status.CollisionCount,
    // }
    if d.DeletionTimestamp != nil {
        return dc.syncStatusOnly(d, rsList)
    }

    // Update deployment conditions with an Unknown condition when pausing/resuming
    // a deployment. In this way, we can be sure that we won't timeout when a user
    // resumes a Deployment with a set progressDeadlineSeconds.
    // 1、if deploy.spec.paused=true && deploy.status.conditions.(不存在DeploymentPaused) then 将DeploymentPaused的condition更新到deploy.status
    // 2、if deploy.spec.paused=false && deploy.status.conditions.(存在DeploymentPaused) then 将DeploymentPaused的condition更新为DeploymentResume
    if err = dc.checkPausedConditions(d); err != nil {
        return err
    }

    // 当deploy处于pause的状态时，是可以处理scale up/down的操作的
    // 根据设置好的滚动策略，将最新的replicaset scale到指定副本
    if d.Spec.Paused {
        return dc.sync(d, rsList)
    }

    // rollback is not re-entrant in case the underlying replica sets are updated with a new
    // revision so we should ensure that we won't proceed to update replica sets until we
    // make sure that the deployment has cleaned up its rollback spec in subsequent enqueues.
    // 检查deployment对象的annotations中deprecated.deployment.rollback.to有值且不为空
    // 则判断该deployment当前处于rollback阶段，执行rollback流程
    // 该annotation现在已经弃用
    if getRollbackTo(d) != nil {
        return dc.rollback(d, rsList)
    }
   
    // 判断当前是否有需要执行scale
    scalingEvent, err := dc.isScalingEvent(d, rsList)
    if err != nil {
        return err
    }
    if scalingEvent {
        return dc.sync(d, rsList)
    }

    switch d.Spec.Strategy.Type {
    case apps.RecreateDeploymentStrategyType:
        return dc.rolloutRecreate(d, rsList, podMap)
    case apps.RollingUpdateDeploymentStrategyType:
        return dc.rolloutRolling(d, rsList)
    }
    return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}


```