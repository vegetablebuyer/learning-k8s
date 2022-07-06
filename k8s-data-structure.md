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

```shell script
# 一窗口执行：
kubectl proxy --port 8080
# 另外一个窗口访问：
curl http://localhost:8080/api/v1
```

### core资源组
**core资源组也就是Legacy Group** ，最为k8s最核心的资源，没有Group的概念 \
这类资源主要有pod、node、service、configmap、event等
```shell script
# non-namespaced
curl http://localhost:8080/api/<version>/<kind>s

# namespaced
curl http://localhost:8080/api/<version>/namespaces/<namespace>/<kind>s/
curl http://localhost:8080/api/<version>/namespaces/<namespace>/<kind>s/<resource-name>

```

### 非core资源组
```shell script
# non-namespaced
curl http://localhost:8080/apis/<group>/<version>/<kind>s

# namespaced
curl http://localhost:8080/apis/<group>/<version>/namespaces/<namespace>/<kind>s/
curl http://localhost:8080/apis/<group>/<version>/namespaces/<namespace>/<kind>s/<resource-name>

```

## 内部版本与外部版本
在k8s的设计中，资源版本分内部版本跟外部版本，外部版本提供给外部使用，内部版本只给apiserver内部使用。
区分内外版本的作用：
- 提供不同版本之间的转换功能，例如从v1beta1-->v1的过程实际是v1beta1--> internal -->v1，转换函数会注册到scheme表中
- 减少复杂度，方便版本维护，避免维护多个版本的对应代码，实际APIServer端处理的都是转换后的内部版本
- 不同外部版本资源之间的字段/功能可能存在些许差异，而内部版本包含所有版本的字段/功能，这为它作为外部资源版本之间转换的桥梁提供了基础

内部版本和外部版本对于资源结构体定义显著的区别是，**内部版本是不带json和proto标签的，因为其不需要结构化提供给外部**。\
以Deployment资源为例

内部版本的代码路径为：```pkg/apis/apps/types.go:267```
```golang
type Deployment struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus
}
```

外部版本代码路径：```vendor/k8s.io/api/apps/v1/types.go:254```
```golang
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object metadata.
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

## scheme注册表

每一种Resource都有对应的Kind，为了更便于分类管理这些资源，APIServer设计了一种名为scheme的结构体，类似于注册表，运行时数据存放内存中，提供给各种资源进行注册，scheme有如下作用：

- 提供资源的版本转换功能
- 提供资源的序列化/反序列化功能

### scheme结构体
```vendor/k8s.io/apimachinery/pkg/runtime/scheme.go:46```
```golang
type Scheme struct {
	// versionMap allows one to figure out the go type of an object with
	// the given version and name.
	gvkToType map[schema.GroupVersionKind]reflect.Type

	// typeToGroupVersion allows one to find metadata for a given go object.
	// The reflect.Type we index by should *not* be a pointer.
	typeToGVK map[reflect.Type][]schema.GroupVersionKind

	// unversionedTypes are transformed without conversion in ConvertToVersion.
	unversionedTypes map[reflect.Type]schema.GroupVersionKind

	// unversionedKinds are the names of kinds that can be created in the context of any group
	// or version
	// TODO: resolve the status of unversioned types.
	unversionedKinds map[string]reflect.Type

	// Map from version and resource to the corresponding func to convert
	// resource field labels in that version to internal version.
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc

	// defaulterFuncs is an array of interfaces to be called with an object to provide defaulting
	// the provided object must be a pointer.
	defaulterFuncs map[reflect.Type]func(interface{})

	// converter stores all registered conversion functions. It also has
	// default converting behavior.
	converter *conversion.Converter

	// versionPriority is a map of groups to ordered lists of versions for those groups indicating the
	// default priorities of these versions as registered in the scheme
	versionPriority map[string][]string

	// observedVersions keeps track of the order we've seen versions during type registration
	observedVersions []schema.GroupVersion

	// schemeName is the name of this scheme.  If you don't specify a name, the stack of the NewScheme caller will be used.
	// This is useful for error reporting to indicate the origin of the scheme.
	schemeName string
}
```
主要关注的字段：
- gvkToType ：用map结构存储gvk和Type的映射关系，一个gvk只会具体对应一个Type
- typeToGVK：用map结构存储type和gvk的映射关系，不同的是，一个type可能会对应多个gvk
- converter：map结构，存放资源版本转换的方法