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

### cri service



```protobuf
service RuntimeService {
    // Version returns the runtime name, runtime version, and runtime API version.
    rpc Version(VersionRequest) returns (VersionResponse) {}

    // RunPodSandbox creates and starts a pod-level sandbox. Runtimes must ensure
    // the sandbox is in the ready state on success.
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
    // StopPodSandbox stops any running process that is part of the sandbox and
    // reclaims network resources (e.g., IP addresses) allocated to the sandbox.
    // If there are any running containers in the sandbox, they must be forcibly
    // terminated.
    // This call is idempotent, and must not return an error if all relevant
    // resources have already been reclaimed. kubelet will call StopPodSandbox
    // at least once before calling RemovePodSandbox. It will also attempt to
    // reclaim resources eagerly, as soon as a sandbox is not needed. Hence,
    // multiple StopPodSandbox calls are expected.
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse) {}
    // RemovePodSandbox removes the sandbox. If there are any running containers
    // in the sandbox, they must be forcibly terminated and removed.
    // This call is idempotent, and must not return an error if the sandbox has
    // already been removed.
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
    // PodSandboxStatus returns the status of the PodSandbox. If the PodSandbox is not
    // present, returns an error.
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
    // ListPodSandbox returns a list of PodSandboxes.
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {}

    // CreateContainer creates a new container in specified PodSandbox
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
    // StartContainer starts the container.
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
    // StopContainer stops a running container with a grace period (i.e., timeout).
    // This call is idempotent, and must not return an error if the container has
    // already been stopped.
    // TODO: what must the runtime do after the grace period is reached?
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
    // RemoveContainer removes the container. If the container is running, the
    // container must be forcibly removed.
    // This call is idempotent, and must not return an error if the container has
    // already been removed.
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
    // ListContainers lists all containers by filters.
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}
    // ContainerStatus returns status of the container. If the container is not
    // present, returns an error.
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse) {}
    // UpdateContainerResources updates ContainerConfig of the container.
    rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse) {}
    // ReopenContainerLog asks runtime to reopen the stdout/stderr log file
    // for the container. This is often called after the log file has been
    // rotated. If the container is not running, container runtime can choose
    // to either create a new log file and return nil, or return an error.
    // Once it returns error, new container log file MUST NOT be created.
    rpc ReopenContainerLog(ReopenContainerLogRequest) returns (ReopenContainerLogResponse) {}

    // ExecSync runs a command in a container synchronously.
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse) {}
    // Exec prepares a streaming endpoint to execute a command in the container.
    rpc Exec(ExecRequest) returns (ExecResponse) {}
    // Attach prepares a streaming endpoint to attach to a running container.
    rpc Attach(AttachRequest) returns (AttachResponse) {}
    // PortForward prepares a streaming endpoint to forward ports from a PodSandbox.
    rpc PortForward(PortForwardRequest) returns (PortForwardResponse) {}

    // ContainerStats returns stats of the container. If the container does not
    // exist, the call returns an error.
    rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse) {}
    // ListContainerStats returns stats of all running containers.
    rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse) {}

    // UpdateRuntimeConfig updates the runtime configuration based on the given request.
    rpc UpdateRuntimeConfig(UpdateRuntimeConfigRequest) returns (UpdateRuntimeConfigResponse) {}

    // Status returns the status of the runtime.
    rpc Status(StatusRequest) returns (StatusResponse) {}
}

// ImageService defines the public APIs for managing images.
service ImageService {
    // ListImages lists existing images.
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse) {}
    // ImageStatus returns the status of the image. If the image is not
    // present, returns a response with ImageStatusResponse.Image set to
    // nil.
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse) {}
    // PullImage pulls an image with authentication config.
    rpc PullImage(PullImageRequest) returns (PullImageResponse) {}
    // RemoveImage removes the image.
    // This call is idempotent, and must not return an error if the image has
    // already been removed.
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse) {}
    // ImageFSInfo returns information of the filesystem that is used to store images.
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse) {}
}

```
