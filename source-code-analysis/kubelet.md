### kubelet读取私有镜像仓库认证的顺序

```golang
// NewKubeGenericRuntimeManager creates a new kubeGenericRuntimeManager
func NewKubeGenericRuntimeManager(
    ...
) (KubeGenericRuntime, error) {
    kubeRuntimeManager := &kubeGenericRuntimeManager{
    ...
    }

    ...
    kubeRuntimeManager.keyring = credentialprovider.NewDockerKeyring()

    ...

    return kubeRuntimeManager, nil
}
```