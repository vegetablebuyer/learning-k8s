## containerd
containerd采用插件式的结构来组织不同的组件的代码，不同的插件之间存在依赖关系，被依赖插件需要先于依赖的插件进行初始化。

```golang
// containerd/services/server/server.go +77
// New creates and initializes a new containerd server
func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    ...
    // 加载插件
    plugins, err := LoadPlugins(ctx, config)
    ...
    var (
        ...
        initialized = plugin.NewPluginSet()
        ...
    )
    for _, p := range plugins {
        ...

        initContext := plugin.NewContext(
            ctx,
            p,
            initialized,
            config.Root,
            config.State,
        )
        ...
        // 执行插件的初始化
        result := p.Init(initContext)
        // 将已经初始化之后的插件保存在initialized中，方便后续的插件调用
        if err := initialized.Add(result); err != nil {
            return nil, errors.Wrapf(err, "could not add plugin result to plugin set")
        }

        instance, err := result.Instance()
        ...
        // 检查插件有没有实现Register()，如果有，就可以作为grpc的服务注册
        // type Service interface {
        //    Register(*grpc.Server) error
        // }
        if src, ok := instance.(plugin.Service); ok {
            grpcServices = append(grpcServices, src)
        }
    }
    for _, service := range grpcServices {
        if err := service.Register(grpcServer); err != nil {
            return nil, err
        }
    }
}

// containerd/plugin/plugin.go +89
// Registration contains information for registering a plugin
type Registration struct {
    // Type表示插件的类型
    Type Type
    // ID of the plugin
    ID string
    // Config specific to the plugin
    Config interface{}
    // Requires 表示当前插件依赖的插件的类型
    Requires []Type
    // 插件的初始化函数
    InitFn func(*InitContext) (interface{}, error)
	// Disable the plugin from loading
    Disable bool
}

// Init the registered plugin
func (r *Registration) Init(ic *InitContext) *Plugin {
    p, err := r.InitFn(ic)
    return &Plugin{
        Registration: r,
        Config:       ic.Config,
        Meta:         ic.Meta,
        instance:     p,
        err:          err,
    }
}

```
LoadPlugins()加载插件的函数具体内容
```golang
// containerd/services/server/server.go +304
func LoadPlugins(ctx context.Context, config *srvconfig.Config) ([]*plugin.Registration, error) {
    // load all plugins into containerd
    ...
    // 除了下面两个插件需要手动加载之后，其他的插件都是通过包的init()函数加载
    plugin.Register(&plugin.Registration{
        Type: plugin.ContentPlugin,
        ID:   "content",
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            ic.Meta.Exports["root"] = ic.Root
            return local.NewStore(ic.Root)
        },
    })
    // containerd的源数据用boltdb存储
    plugin.Register(&plugin.Registration{
        Type: plugin.MetadataPlugin,
        ID:   "bolt",
        Requires: []plugin.Type{
            plugin.ContentPlugin,
            plugin.SnapshotPlugin,
        },
        Config: &srvconfig.BoltConfig{
            ContentSharingPolicy: srvconfig.SharingPolicyShared,
        },
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            ...
            path := filepath.Join(ic.Root, "meta.db")
            ic.Meta.Exports["path"] = path
            
            db, err := bolt.Open(path, 0644, nil)
            ...
            mdb := metadata.NewDB(db, cs.(content.Store), snapshotters, dbopts...)
            if err := mdb.Init(ic.Context); err != nil {
                return nil, err
            }
            return mdb, nil
        },
    })
    ... 
    // 整理插件之间的依赖关系，返回一个插件的列表，将被依赖的插件放在依赖的插件之前
    return plugin.Graph(filter(config.DisabledPlugins)), nil
}
```
其他的插件的初始化跟加载，此部分代码在containerd的主程序二进制代码的入口
```golang
package main

// containerd/cmd/containerd/builtins.go
// register containerd builtins here
import (
    _ "github.com/containerd/containerd/diff/walking/plugin"
    _ "github.com/containerd/containerd/gc/scheduler"
    _ "github.com/containerd/containerd/runtime/restart/monitor"
    _ "github.com/containerd/containerd/services/containers"
    _ "github.com/containerd/containerd/services/content"
    _ "github.com/containerd/containerd/services/diff"
    _ "github.com/containerd/containerd/services/events"
    _ "github.com/containerd/containerd/services/healthcheck"
    _ "github.com/containerd/containerd/services/images"
    _ "github.com/containerd/containerd/services/introspection"
    _ "github.com/containerd/containerd/services/leases"
    _ "github.com/containerd/containerd/services/namespaces"
    _ "github.com/containerd/containerd/services/opt"
    _ "github.com/containerd/containerd/services/snapshots"
    _ "github.com/containerd/containerd/services/tasks"
    _ "github.com/containerd/containerd/services/version"
)

// containerd/cmd/containerd/builtins_cri.go
package main

import _ "github.com/containerd/containerd/pkg/cri"

// 其他的包import需要看containerd/cmd/containerd/下面的代码
```

### cri server
根据grpc的服务描述定义，cri service需要实现下面的方法供kubelet调用。\
这些方法的实现都在```github.com/containerd/containerd/pkg/cri/server```的包下面


```protobuf
// containerd/vendor/k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.proto

// Runtime service defines the public APIs for remote container runtimes
service RuntimeService {
    rpc Version(VersionRequest) returns (VersionResponse) {}
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse) {}
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {}
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse) {}
    rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse) {}
    rpc ReopenContainerLog(ReopenContainerLogRequest) returns (ReopenContainerLogResponse) {}
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse) {}
    rpc Exec(ExecRequest) returns (ExecResponse) {}
    rpc Attach(AttachRequest) returns (AttachResponse) {}
    rpc PortForward(PortForwardRequest) returns (PortForwardResponse) {}
    rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse) {}
    rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse) {}
    rpc UpdateRuntimeConfig(UpdateRuntimeConfigRequest) returns (UpdateRuntimeConfigResponse) {}
    rpc Status(StatusRequest) returns (StatusResponse) {}
}

// ImageService defines the public APIs for managing images.
service ImageService {
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse) {}
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse) {}
    rpc PullImage(PullImageRequest) returns (PullImageResponse) {}
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse) {}
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse) {}
}
```

#### cri service 检查runtime的状态
```protobuf
service RuntimeService {
    ...
    // Status returns the status of the runtime.
    rpc Status(StatusRequest) returns (StatusResponse) {}
}
```
cri server通过实现Status()函数来检查runtime的状态
```golang
// containerd/pkg/cri/server/status.go +32
func (c *criService) Status(ctx context.Context, r *runtime.StatusRequest) (*runtime.StatusResponse, error) {
    // runtime的状态检查的返回主要有两部分，
    // 1. 一个是runtime本身的状态，默认为true，因为如果该grpc调用能正常处理请求，那么containerd肯定是正常的
    // 2. cni插件的状态
    runtimeCondition := &runtime.RuntimeCondition{
        Type:   runtime.RuntimeReady,
        Status: true,
    }
    networkCondition := &runtime.RuntimeCondition{
        Type:   runtime.NetworkReady,
        Status: true,
    }
    // 检查cni插件的状态
    if err := c.netPlugin.Status(); err != nil {
        networkCondition.Status = false
        networkCondition.Reason = networkNotReadyReason
        networkCondition.Message = fmt.Sprintf("Network plugin returns error: %v", err)
    }

    resp := &runtime.StatusResponse{
        Status: &runtime.RuntimeStatus{Conditions: []*runtime.RuntimeCondition{
            runtimeCondition,
            networkCondition,
        }},
    }
    if r.Verbose {
        ...
    }
    return resp, nil
}

```
cni插件在cri server初始化的时候跟着初始化
```golang
// containerd/pkg/cri/server/server_linux.go +28
const networkAttachCount = 2

// initPlatform handles linux specific initialization for the CRI service.
func (c *criService) initPlatform() error {
    ...

    // 每一个pod最少需要一个loopback网卡跟一个非主机网卡，所以networkAttachCount为2
    c.netPlugin, err = cni.New(cni.WithMinNetworkCount(networkAttachCount),
        cni.WithPluginConfDir(c.config.NetworkPluginConfDir),
        cni.WithPluginMaxConfNum(c.config.NetworkPluginMaxConfNum),
        cni.WithPluginDir([]string{c.config.NetworkPluginBinDir}))
    ...

    return nil
}
```
其中```cni.New()```函数返回一个```criService.netPlugin```
```golang
// containerd/vendor/github.com/containerd/go-cni/cni.go +80
func defaultCNIConfig() *libcni {
    return &libcni{
        config: config{
            pluginDirs:       []string{DefaultCNIDir},  // "/opt/cni/bin"
            pluginConfDir:    DefaultNetDir,            // "/etc/cni/net.d"
            pluginMaxConfNum: DefaultMaxConfNum,        // 1
            prefix:           DefaultPrefix,            // "eth"
        },
        cniConfig: &cnilibrary.CNIConfig{
            Path: []string{DefaultCNIDir},              // "/opt/cni/bin"
        },
        networkCount: 1,
    }
}

// New creates a new libcni instance.
func New(config ...Opt) (CNI, error) {
    cni := defaultCNIConfig()
    var err error
    for _, c := range config {
        if err = c(cni); err != nil {
            return nil, err
        }
    }
    return cni, nil
}

// 所以Status()函数检查的逻辑很简单，只要c.networks小于2，证明cni插件的初始化就没有完成
func (c *libcni) Status() error {
	c.RLock()
	defer c.RUnlock()
	if len(c.networks) < c.networkCount {
		return ErrCNINotInitialized
	}
	return nil
}

```
其中loopback口的network配置代码里默认加载，其他的配置则在""/etc/cni/net.d/*.(conf|conflist|json}"的配置文件中
```golang
// containerd/pkg/cri/server/server_linux.go +75
func (c *criService) cniLoadOptions() []cni.Opt {
    return []cni.Opt{cni.WithLoNetwork, cni.WithDefaultConf}
}
```
```cni.WithLoNetwork()```跟```cni.WithDefaultConf()```两个函数如下
```golang

// containerd/vendor/github.com/containerd/go-cni/opts.go +80
 func WithLoNetwork(c *libcni) error {
 	loConfig, _ := cnilibrary.ConfListFromBytes([]byte(`{
 "cniVersion": "0.3.1",
 "name": "cni-loopback",
 "plugins": [{
   "type": "loopback"
 }]
 }`))
 
 	c.networks = append(c.networks, &Network{
 		cni:    c.cniConfig,
 		config: loConfig,
 		ifName: "lo",
 	})
 	return nil
 }

func WithDefaultConf(c *libcni) error {
	return loadFromConfDir(c, c.pluginMaxConfNum)
}

func loadFromConfDir(c *libcni, max int) error {
	files, err := cnilibrary.ConfFiles(c.pluginConfDir, []string{".conf", ".conflist", ".json"})
	...
	var networks []*Network
	for _, confFile := range files {
		...
		networks = append(networks, &Network{
			cni:    c.cniConfig,
			config: confList,
			ifName: getIfName(c.prefix, i),
		})
		i++
		if i == max {
			break
		}
	}
	if len(networks) == 0 {
		return errors.Wrapf(ErrCNINotInitialized, "no valid networks found in %s", c.pluginDirs)
	}
	c.networks = append(c.networks, networks...)
	return nil
}
```
