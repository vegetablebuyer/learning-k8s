### kubelet如何读取私有镜像仓库凭证文件

kubelet对私有仓库的认证主要由一个```providersDockerKeyring```的对象来实现，这个对象包含了一些```Provider```。

```golang
// kubernetes/pkg/kubelet/kuberuntime/kuberuntime_manager.go +155
func NewKubeGenericRuntimeManager(...) (KubeGenericRuntime, error) {
    kubeRuntimeManager := &kubeGenericRuntimeManager{
    ...
    }

    ...
    kubeRuntimeManager.keyring = credentialprovider.NewDockerKeyring()

    ...

    return kubeRuntimeManager, nil
}

// kubernetes/pkg/credentialprovider/plugins.go
func NewDockerKeyring() DockerKeyring {
    keyring := &providersDockerKeyring{
        Providers: make([]DockerConfigProvider, 0),
    }

    keys := reflect.ValueOf(providers).MapKeys()
    stringKeys := make([]string, len(keys))
    for ix := range keys {
        stringKeys[ix] = keys[ix].String()
    }
    sort.Strings(stringKeys)

    for _, key := range stringKeys {
        provider := providers[key]
        if provider.Enabled() {
            klog.V(4).Infof("Registering credential provider: %v", key)
            keyring.Providers = append(keyring.Providers, provider)
        }
    }

    return keyring
}

var providers = make(map[string]DockerConfigProvider)

func RegisterCredentialProvider(name string, provider DockerConfigProvider) {
    providersMutex.Lock()
    defer providersMutex.Unlock()
    _, found := providers[name]
    if found {
        klog.Fatalf("Credential provider %q was registered twice", name)
    }
    klog.V(4).Infof("Registered credential provider %q", name)
    providers[name] = provider
}

// kubernetes/pkg/credentialprovider/plugins.go
func init() {
    RegisterCredentialProvider(".dockercfg",
        &CachingDockerConfigProvider{
            Provider: &defaultDockerConfigProvider{},
            Lifetime: 5 * time.Minute,
        })
}

```
可见，kubelet的证书```Provider```为默认的```defaultDockerConfigProvider```
```golang
// kubernetes/pkg/credentialprovider/provider.go
// Enabled implements dockerConfigProvider
func (d *defaultDockerConfigProvider) Enabled() bool {
    return true
}

// Provide implements dockerConfigProvider
func (d *defaultDockerConfigProvider) Provide(image string) DockerConfig {
    // Read the standard Docker credentials from .dockercfg
    if cfg, err := ReadDockerConfigFile(); err == nil {
        return cfg
    } else if !os.IsNotExist(err) {
        klog.V(4).Infof("Unable to parse Docker config file: %v", err)
    }
    return DockerConfig{}
}
```
每次kubelet拉取镜像会调用```providersDockerKeyring```的```lookup()```方法，进而调用```Provider```的```Provide()```方法来查找镜像的认证
```golang
// kubernetes/pkg/kubelet/kuberuntime/kuberuntime_image.go +31
func (m *kubeGenericRuntimeManager) PullImage(image kubecontainer.ImageSpec, pullSecrets []v1.Secret, podSandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {
    img := image.Image
    repoToPull, _, _, err := parsers.ParseImageName(img)
    ...

    imgSpec := toRuntimeAPIImageSpec(image)

    creds, withCredentials := keyring.Lookup(repoToPull)
    ...

    // some pulling action
}

// kubernetes/pkg/credentialprovider/keyring.go +263
func (dk *providersDockerKeyring) Lookup(image string) ([]AuthConfig, bool) {
    keyring := &BasicDockerKeyring{}

    for _, p := range dk.Providers {
        keyring.Add(p.Provide(image))
    }

    return keyring.Lookup(image)
}

// kubernetes/pkg/credentialprovider/provider.go +76
func (d *defaultDockerConfigProvider) Provide(image string) DockerConfig {
    // Read the standard Docker credentials from .dockercfg
    if cfg, err := ReadDockerConfigFile(); err == nil {
        return cfg
    } else if !os.IsNotExist(err) {
        klog.V(4).Infof("Unable to parse Docker config file: %v", err)
    }
    return DockerConfig{}
}

// kubernetes/pkg/credentialprovider/config.go 
func ReadDockerConfigFile() (cfg DockerConfig, err error) {
    if cfg, err := ReadDockerConfigJSONFile(nil); err == nil {
        return cfg, nil
    }
    return ReadDockercfgFile(nil)
    }

func ReadDockerConfigJSONFile(searchPaths []string) (cfg DockerConfig, err error) {
    if len(searchPaths) == 0 {
        // searchPaths参数传进来是一个空列表，所以关键要看DefaultDockerConfigJSONPaths()返回的路径列表
        searchPaths = DefaultDockerConfigJSONPaths()
    }
    for _, configPath := range searchPaths {
        absDockerConfigFileLocation, err := filepath.Abs(filepath.Join(configPath, configJSONFileName))
		...
		cfg, err = ReadSpecificDockerConfigJSONFile(absDockerConfigFileLocation)
		...
        return cfg, nil
	}
	return nil, fmt.Errorf("couldn't find valid %s after checking in %v", configJSONFileName, searchPaths)

}

func ReadDockercfgFile(searchPaths []string) (cfg DockerConfig, err error) {
	if len(searchPaths) == 0 {
		searchPaths = DefaultDockercfgPaths()
	}

	for _, configPath := range searchPaths {
		absDockerConfigFileLocation, err := filepath.Abs(filepath.Join(configPath, configFileName))
		...
		contents, err := ioutil.ReadFile(absDockerConfigFileLocation)
		...
		cfg, err := readDockerConfigFileFromBytes(contents)
		...
		return cfg, nil

	}
	return nil, fmt.Errorf("couldn't find valid .dockercfg after checking in %v", searchPaths)
}


func DefaultDockerConfigJSONPaths() []string {
	return []string{GetPreferredDockercfgPath(), workingDirPath, homeJSONDirPath, rootJSONDirPath}
}

func GetPreferredDockercfgPath() string {
	preferredPathLock.Lock()
	defer preferredPathLock.Unlock()
	return preferredPath
}

```
所以总共有4个查找路径，并且查找的文件名变量为```configJSONFileName```，几个路径都查找不到json文件就开始找```configFileName```
- ```GetPreferredDockercfgPath()```函数返回的```preferredPath```
- ```workingDirPath```
- ```homeJSONDirPath```
- ```rootJSONDirPath```
除了```preferredPath```，其他3个为早就定义好的变量
```golang
var (
	preferredPath     = ""
	workingDirPath    = ""
	homeDirPath, _    = os.UserHomeDir()
	rootDirPath       = "/"
	homeJSONDirPath   = filepath.Join(homeDirPath, ".docker")
	rootJSONDirPath   = filepath.Join(rootDirPath, ".docker")

	configFileName     = ".dockercfg"
	configJSONFileName = "config.json"
)
```
其中```preferredPath```变量在一开始kubelet初始化的时候设置了
```golang
// kubernetes/cmd/kubelet/app/server.go
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
    ...
    credentialprovider.SetPreferredDockercfgPath(kubeServer.RootDirectory)
    ...
}
```
而```kubeServer.RootDirectory```的值为```/var/lib/kubelet``` \
综上可得，kubelet查找镜像仓库凭证文件的顺序为
1. /var/lib/kubelet/config.json
2. ./config.json
3. ${home}/.docker/config.json
4. /.docker/config.json
5. /var/lib/kubelet/.dockercfg
6. ./.dockercfg
7. ${home}/.docker/.dockercfg
8. /.docker/.dockercfg

### kubelet到containerd之间的调用链

kubelet ---> cri plugin ---> cri service ---> containerd ---> boltdb
- kubelet集成了cri plugin的代码
- cri service实际上是containerd的一部分，由containerd实现并维护启动
- cri plugin通过grpc调用来访问cri service
- cri service通过grpc调用来访问containerd
- containerd用boltdb来存储容器的元数据