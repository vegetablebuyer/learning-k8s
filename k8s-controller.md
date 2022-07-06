## controller

controller作为k8s集群中比较核心的组件，实现了k8s中资源的自愈功能。他的核心逻辑是对比k8s中资源的预期状态跟当前状态，若两个状态有所不同，则不断地**调谐（reconcile）**，最终让两个状态达到一致。

几乎每种资源都有特定的controller去承担该资源的**调谐（reconcile）** 工作，即使是自己定义的资源（CRD）也需要实现相应的controller。而controller-manager则是管理这些controller的控制器，controller-manager不仅管理者所有的controller，也为这些controller提供统一的k8s资源访问的入口，降低controller对apiserver访问的压力。

![alt text](./pictures/controller-manager.png)

## sample-controller

通过k8s官方提供的一个 [controller范例](https://github.com/kubernetes/sample-controller/) 来了解一下controller实现的流程

1、实例化一个sharedInformerFactory，sharedInformerFactory为SharedInformerFactory interface的一种实现。SharedInformerFactory为所有内嵌资源都实现了生成shared infomers的方法

```golang
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

2、通过实例化之后的sharedInformerFactory调用资源informer()的实例化方法，生成相应资源的informer
```golang
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
deployInformers := kubeInformerFactory.Apps().V1().Deployments().Informers()

// SharedInformerFactory().Apps().V1().Deployments().Informers()最终会调用到下面的代码
// client-go/informers/apps/v1/deployment.go +84
func (f *deploymentInformer) Informer() cache.SharedIndexInformer {
   return f.factory.InformerFor(&appsv1.Deployment{}, f.defaultInformer)
}

// 反过来会调用sharedInformerFactory的InformerFor()将当前实例化的informer添加到sharedInformerFactory.informers中
// client-go/informers/factory.go +162
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
   f.lock.Lock()
   defer f.lock.Unlock()

   informerType := reflect.TypeOf(obj)
   informer, exists := f.informers[informerType]
   if exists {
      return informer
   }

   resyncPeriod, exists := f.customResync[informerType]
   if !exists {
      resyncPeriod = f.defaultResync
   }

   informer = newFunc(f.client, resyncPeriod)
   f.informers[informerType] = informer

   return informer
}

```
3、informer调用AddEventHandler()方法注册资源变化的处理函数
```golang
deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
   AddFunc: controller.handleObject,
   ...
})

func (c *Controller) handleObject(obj interface{}) {
      ...
      c.enqueueFoo(obj)
      return
   }
}

func (c *Controller) enqueueFoo(obj interface{}) {
   var key string
   var err error
   if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
      utilruntime.HandleError(err)
      return
   }
   c.workqueue.Add(key)
}

// 这里handlerObject方法最关键的是把obj获取到的事件推送到workqueue的队列中，等待process将事件取出处理
```

4、sharedInformerFactory调用Start()方法将所有实例化的informer启动
```golang
kubeInformerFactory.Start(stopCh)

// client-go/informers/factory.go +127
// Start initializes all requested informers.
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
   f.lock.Lock()
   defer f.lock.Unlock()

   for informerType, informer := range f.informers {
      if !f.startedInformers[informerType] {
         go informer.Run(stopCh)
         f.startedInformers[informerType] = true
      }
   }
}

// infomer的Run()方法
// client-go/tools/cache/shared_informer.go +368
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()

   fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
      KnownObjects:          s.indexer,
      EmitDeltaTypeReplaced: true,
   })

   cfg := &Config{
      ...
   }

   func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()

      s.controller = New(cfg)
      s.controller.(*controller).clock = s.clock
      s.started = true
   }()

   ...
   s.controller.Run(stopCh)
}

```

5、启动controller，开始消费队列中出现的事件，其中c.syncHandler()便是整个controller核心的逻辑处理
```golang
func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
   defer utilruntime.HandleCrash()
   defer c.workqueue.ShutDown()

   
   klog.Info("Starting workers")
   // Launch two workers to process Foo resources
   for i := 0; i < threadiness; i++ {
      go wait.Until(c.runWorker, time.Second, stopCh)
   }

   klog.Info("Started workers")
   <-stopCh
   klog.Info("Shutting down workers")

   return nil
}

func (c *Controller) runWorker() {
   for c.processNextWorkItem() {
   }
}

// processNextWorkItem will read a single work item off the workqueue and
// attempt to process it, by calling the syncHandler.
func (c *Controller) processNextWorkItem() bool {
   obj, shutdown := c.workqueue.Get()

   if shutdown {
      return false
   }

   // We wrap this block in a func so we can defer c.workqueue.Done.
   err := func(obj interface{}) error {
     
      defer c.workqueue.Done(obj)
      ...
      if err := c.syncHandler(key); err != nil {
         // Put the item back on the workqueue to handle any transient errors.
         c.workqueue.AddRateLimited(key)
         return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
      }
      ...
      return nil
   }(obj)

   if err != nil {
      utilruntime.HandleError(err)
      return true
   }

   return true
}

```