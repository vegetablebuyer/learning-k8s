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
加载插件的函数具体内容
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
	// 整理插件之间的依赖关系，将被依赖的插件放在依赖的插件之前
	return plugin.Graph(filter(config.DisabledPlugins)), nil
}
```