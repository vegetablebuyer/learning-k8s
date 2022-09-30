### kubelet读取私有镜像仓库认证的顺序

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