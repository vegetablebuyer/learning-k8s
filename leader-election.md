k8s的控制面组件中，除了```kube-apiserver```是多个副本一起工作之外，另外两个组件```kube-controller-manager```跟```kube-scheduler```都是只有一个```leader```在工作，其他节点都是```candidate```，等待着```leader```故障之后接替。
如何在多个副本中选举```leader```，在k8s中是基于```leader election```来实现的。简单点说多个副本都去竞争k8s中的同一个资源作为**锁**，先获取到**锁**的就成为```leader```，其他节点自动成为```candidate```。
```leader```节点需要不断地```renew```**锁**以维持自己```leader```的身份。其他```candidate```也会不断地去获取这个**锁**的信息，检查**锁**是否过期。如果**锁**过期，就认为```leader```已经故障，并尝试成为**锁**的```holder```从而成为```leader```

上面的这些逻辑的实现都是基于```client-go```中的```tools/leaderelection```来实现的

在k8s早期的版本中，```kube-scheduler```跟```kube-controller-manager```默认使用的**锁**是```endpoint```资源，而在后期较新的版本中**锁**资源换成了```lease```资源

### leader-election配置

```golang
// vendor/k8s.io/component-base/config/types.go +41
type LeaderElectionConfiguration struct {
	// 是否开启leader选举
	LeaderElect bool
	
	// candidate节点对leader进行探活的间隔时间
	LeaseDuration metav1.Duration

	// leader节点续约的间隔时间，必须小于LeaseDuration
	RenewDeadline metav1.Duration

	// candidate节点尝试获取锁的间隔时间
	RetryPeriod metav1.Duration
	
	// 锁资源的类型
	ResourceLock string

	// 锁资源的名字
	ResourceName string

	// 锁资源所在的命名空间
	ResourceNamespace string
}
```

### 锁

```bash
kubectl get ep -n kube-system kube-scheduler -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"kube-content-rc-live-10-161-19-241-node2_06b9ed11-14d2-428a-a98b-46f14b6e207d","leaseDurationSeconds":15,"acquireTime":"2022-04-27T10:45:38Z","renewTime":"2022-07-13T12:38:21Z","leaderTransitions":49}'
  name: kube-scheduler
  namespace: kube-system

```


```bash
kubectl get lease -n kube-system kube-scheduler -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  acquireTime: "2022-06-07T15:35:07.817712Z"
  holderIdentity: k8s-bke-sg-uat-core-10-165-55-17_38a700fa-0756-48cb-a23c-cd77879ecc01
  leaseDurationSeconds: 15
  leaseTransitions: 26
  renewTime: "2022-07-13T12:44:20.919959Z"
```

