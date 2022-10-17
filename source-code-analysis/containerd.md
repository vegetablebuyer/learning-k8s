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
    }

}

// containerd/plugin/plugin.go +89
// Registration contains information for registering a plugin
type Registration struct {
    // Type of the plugin
    Type Type
    // ID of the plugin
    ID string
    // Config specific to the plugin
    Config interface{}
    // Requires is a list of plugins that the registered plugin requires to be available
    Requires []Type

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