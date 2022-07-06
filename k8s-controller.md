## controller

controller作为k8s集群中比较核心的组件，实现了k8s中资源的自愈功能。他的核心逻辑是对比k8s中资源的预期状态跟当前状态，若两个状态有所不同，则不断地**调谐（reconcile）**，最终让两个状态达到一致。

几乎每种资源都有特定的controller去承担该资源的**调谐（reconcile）** 工作，即使是自己定义的资源（CRD）也需要实现相应的controller。而controller-manager则是管理这些controller的控制器，controller-manager不仅管理者所有的controller，也为这些controller提供统一的k8s资源访问的入口，降低controller对apiserver访问的压力。

![alt text](./pictures/controller-manager.png)

## sample-controller

通过k8s官方提供的一个 [controller范例](https://github.com/kubernetes/sample-controller/) 来了解一下controller实现的流程

1、实例化一个sharedInformerFactory，sharedInformerFactory为SharedInformerFactory interface的一种实现。SharedInformerFactory为所有内嵌资源都实现了生成shared infomers的方法

```go
// client-go/informers/factory.go +55
 
type sharedInformerFactory struct {
   client           kubernetes.Interface
   namespace        string
   tweakListOptions internalinterfaces.TweakListOptionsFunc
   lock             sync.Mutex
   defaultResync    time.Duration
   customResync     map[reflect.Type]time.Duration

   informers map[reflect.Type]cache.SharedIndexInformer
   // startedInformers is used for tracking which informers have been started.
   // This allows Start() to be called multiple times safely.
   startedInformers map[reflect.Type]bool
}


// client-go/informers/factory.go +185

// SharedInformerFactory provides shared informers for resources in all known
// API group versions.
type SharedInformerFactory interface {
   internalinterfaces.SharedInformerFactory
   ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
   WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

   Admissionregistration() admissionregistration.Interface
   Internal() apiserverinternal.Interface
   Apps() apps.Interface
   Autoscaling() autoscaling.Interface
   Batch() batch.Interface
   Certificates() certificates.Interface
   Coordination() coordination.Interface
   Core() core.Interface
   Discovery() discovery.Interface
   Events() events.Interface
   Extensions() extensions.Interface
   Flowcontrol() flowcontrol.Interface
   Networking() networking.Interface
   Node() node.Interface
   Policy() policy.Interface
   Rbac() rbac.Interface
   Scheduling() scheduling.Interface
   Storage() storage.Interface
}
```