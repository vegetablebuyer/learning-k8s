## GVK and Resource

k8s的核心组件Apiserver的主要工作就是围绕各种资源的管控，而k8s的资源是通过Group(G)、Version(V)、Kind(K)来区分跟组合资源的

- Group: 资源组，也称ApiGroup，如core、apps、batch等
- Version: 资源版本，也称ApiVersion，如v1、v1beta1、v1alpha1等
    - stable: 正式发布的稳定版本，如v1、v2
    - beta: 预发布版本，如v1beta1、v1beta2
    - alpha: 实验性版本，如v1alpha1、v1alpha2
- Kind: 某个<group>/<version>下的资源类型，如pod，deployment等
- Resource：某个资源类型的具体实例，如一个名为test的pod实例

## resource的请求路径

### 暴露一个无需认证的请求接口

```bash
# 一窗口执行：
kubectl proxy --port 8080
# 另外一个窗口访问：
curl http://localhost:8080/api/v1
```

### core资源组
**core资源组也就是Legacy Group** ，最为k8s最核心的资源，没有Group的概念。
这类资源主要有pod、node、service、configmap、event等