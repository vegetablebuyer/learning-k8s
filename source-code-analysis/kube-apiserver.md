## kube-apiserver restful请求的处理流程
### kube-apiserver的组成
kube-apiserver由AggregatorServer、APIServer、APIExtensionServer三个组件构成，这三个组件由内部的delegationTarget关系串联起来，并依次处理请求
```golang
// kubernetes/cmd/kube-apiserver/app/server.go + 179
// Run runs the specified APIServer.  This should never exit.
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
    // To help debugging, immediately log version
    klog.Infof("Version: %+v", version.Get())

    server, err := CreateServerChain(completeOptions, stopCh)
    ...

    prepared, err := server.PrepareRun()
    ...

    return prepared.Run(stopCh)
}

// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
    ...
	
    apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
    ...

    kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)

    ...
    aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
    ...

    return aggregatorServer, nil
}

```

AggregatorServer 和 APIExtensionsServer 对应处理两种主要扩展资源，即分别是 AA 和 CRD。APIServer则对应处理内建资源。
三个Server的主要构成都有GenericAPIServer，并且创建调用的都是同一个函数
```golang
// kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go + 127
// New returns a new instance of CustomResourceDefinitions from the given config.
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
    genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
    ...
    s := &CustomResourceDefinitions{
        GenericAPIServer: genericServer,
    }
    ...
}

// kubernetes/pkg/controlplane/instance.go + 351
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
    ...
    s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
    ...
    m := &Instance{
        GenericAPIServer:          s,
        ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
    }
    ...
}

// kubernetes/vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go + 164
// NewWithDelegate returns a new instance of APIAggregator from the given config.
func (c completedConfig) NewWithDelegate(delegationTarget genericapiserver.DelegationTarget) (*APIAggregator, error) {
    genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
    ...
    s := &APIAggregator{
        GenericAPIServer:           genericServer,
        ...
    }
    ...
}
```
其中三者的delagation关系AggregatorServer -> APIServer -> APIExtensionServer。

### http.Handler
http.Handler为处理http restful请求的实例，处理请求的时候调用的是实例的ServeHTTP()方法
```golang
type http.Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```
AggregatorServer，APIServer，APIExtensionServer三个服务都创建了一个http.Handler的实例用于处理各自负责的资源的请求
```golang
// kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go +538
func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
	...
    handlerChainBuilder := func(handler http.Handler) http.Handler {
        return c.BuildHandlerChainFunc(handler, c.Config)
    }
    apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())

    s := &GenericAPIServer{
        ...

        Handler: apiServerHandler,

        ...
    }
}

// kubernetes/vendor/k8s.io/apiserver/pkg/server/handler.go +73
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
    nonGoRestfulMux := mux.NewPathRecorderMux(name)
    if notFoundHandler != nil {
        nonGoRestfulMux.NotFoundHandler(notFoundHandler)
    }

    gorestfulContainer := restful.NewContainer()
    gorestfulContainer.ServeMux = http.NewServeMux()
    gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
    gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
        logStackOnRecover(s, panicReason, httpWriter)
    })
    gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
        serviceErrorHandler(s, serviceErr, request, response)
    })

    director := director{
        name:               name,
        goRestfulContainer: gorestfulContainer,
        nonGoRestfulMux:    nonGoRestfulMux,
    }

    return &APIServerHandler{
        FullHandlerChain:   handlerChainBuilder(director),
        GoRestfulContainer: gorestfulContainer,
        NonGoRestfulMux:    nonGoRestfulMux,
        Director:           director,
    }
}
```
ServeHTTP()方法是真正处理请求的逻辑
```golang
// kubernetes/vendor/k8s.io/apiserver/pkg/server/handler.go +187
func (a *APIServerHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	a.FullHandlerChain.ServeHTTP(w, r)
}
```
FullHandlerChain是被handlerChainBuilder修饰的director，所以正常处理请求是director.ServeHTTP() \
处理的逻辑：
- 路径匹配，d.goRestfulContainer.Dispatch(w, req)
- 路径不匹配，d.nonGoRestfulMux.ServeHTTP(w, req)
```golang
// kubernetes/vendor/k8s.io/apiserver/pkg/server/handler.go +122
func (d director) ServeHTTP(w http.ResponseWriter, req *http.Request) {
 	path := req.URL.Path
 
 	// check to see if our webservices want to claim this path
 	for _, ws := range d.goRestfulContainer.RegisteredWebServices() {
 		switch {
 		case ws.RootPath() == "/apis":
 			if path == "/apis" || path == "/apis/" {
 				d.goRestfulContainer.Dispatch(w, req)
 				return
 			}
 
 		case strings.HasPrefix(path, ws.RootPath()):
 			// ensure an exact match or a path boundary match
 			if len(path) == len(ws.RootPath()) || path[len(ws.RootPath())] == '/' {
 				d.goRestfulContainer.Dispatch(w, req)
 				return
 			}
 		}
 	}
 
 	// if we didn't find a match, then we just skip gorestful altogether
 	klog.V(5).Infof("%v: %v %q satisfied by nonGoRestful", d.name, req.Method, path)
 	d.nonGoRestfulMux.ServeHTTP(w, req)
 }
```