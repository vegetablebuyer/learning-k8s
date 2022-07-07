## kube-scheduler
> kube-scheduler是k8s集群默认的调度器。这意味着，如果你愿意，可以自己实现一个调度器

kube-scheduler负责将每一个新建的且没有绑定节点的pod找到最合适的节点进行调度。\
宏观来说，kube-scheduler实现一次调度分两步：
1. Filtering: 从集群中找出满足pod要求的所有node节点，这些节点被称为***fesible nodes***
2. Scoring: 根据打分规则对每一个***fesible node***进行打分，选出分数最高的一个，将pod调度到该节点

## scheduler配置

### profiles
通过```kube-scheduder --config <filename>```来指定profile文件。一个简单的profile文件如下：
```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:
        disabled:
        - name: PodTopologySpread
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```
profile配置文件可以控制<u>kube-scheduler</u>调度过程中的每一个阶段(stage)。每个阶段都通过扩展点(extension point)公开。调度插件通过实现一个或者多个扩展点，来提供调度功能

### 扩展点
整个调度的行为是由一系列阶段组成的，这些阶段通过下面的扩展点公开：
1. ```queueSort```: 这些插件对调度队列中处于```pending```状态的pod进行排序。一次只能弃用一个队列排序的插件。
2. ```preFilter```: 这些插件在filtering预处理之前检查pod或集群的信息。这些插件可以将pod标记为**不可调度**。
3. ```filter```: 这些插件相当于调度策略中的**predicate**，用于过滤不能运行pod的节点。过滤器的调用顺序是可配置的。如果没有一个节点通过所有过滤器的筛选，pod将会被标记为不可调度。
4. ```postFilter```: 当pod无法找到可用节点时，这些插件将会被照配置的顺序被调用，如果其中一个```postFilter```插件将pod标记为**可调度**，剩余其他插件将不再调用。
5. ```preScore```: 信息扩展点，用于预打分工作。
6. ```score```: 这些插件给筛选通过的节点打分，调度器最终会选择打分最高的节点。
7. ```reserve```: 这是一个信息扩展点，当资源已经预留给 Pod 时，会通知插件。 这些插件还实现了 ```Unreserve``` 接口，在 ```Reserve``` 期间或之后出现故障时调用。
8. ```permit```: 这些插件可以阻止或延迟pod绑定。
9. ```preBind```: 这些插件在pod绑定节点之前执行。
10. ```bind```: 这些插件将pod与节点绑定。绑定插件是按顺序执行的，只要有一个插件完成了绑定，其余插件不再调用。绑定插件至少需要一个。
11. ```postBind```: 这是一个信息扩展点，在pod绑定节点之后调用。
12. ```multiPoint```: 一个仅供配置的字段，允许同时为所有适用的扩展点启用或禁用插件。

### 调度插件
下面默认启用的插件实现了一个或多个扩展点
1. ```ImageLocality```: 选择已经存在pod运行所需容器镜像的节点。\
    实现扩展点: ```score```
2. ```TaintToleration```: 实现了污点和容忍的功能。\
    实现扩展点: ```filter```，```preScore```，```score```
3. ```NodeName```: 检查pod指定的节点名称与当前节点是否匹配。\
    实现扩展点: ```filter```
4. ```NodePorts```: 检查pod请求的端口在节点上是否可用。\
    实现扩展点: ```preFilter```，```filter```
5. ```NodeAffinity```: 实现了节点选择器 和节点亲和性。\
    实现扩展点: ```filter```，```score```
6. ```PodTopologySpread```: 实现了pod拓扑分布。\
    实现扩展点: ```preFilter```，```filter```，```preScore```，```score```
7. ```NodeUnschedulable```: 过滤```.spec.unschedulable```值为true的节点。\
    实现的扩展点: ```filter```
8. ```NodeResourcesFit```: 检查节点是否拥有pod请求的所有资源。 得分可以使用以下三种策略之一：```LeastAllocated（默认）```、```MostAllocated``` 和```RequestedToCapacityRatio```。\
    实现扩展点: ```preFilter```，```filter```，```score```
9. ```NodeResourcesBalancedAllocation```: 调度pod时，选择资源使用更为均衡的节点。\
    实现扩展点: ```score```
10. ```VolumeBinding```: 检查节点是否有请求的卷，或是否可以绑定请求的卷。 \
    实现扩展点: ```preFilter```，```filter```，```reserve```，```preBind```，```score```
11. ```InterPodAffinity```: 实现pod间的亲和性与反亲和性。\
    实现扩展点: ```preFilter```，```filter```，```preScore```，```score```
12. ```PrioritySort```: 提供默认的基于优先级的排序。\
    实现扩展点: ```queueSort```
13. ```DefaultBinder```: 提供默认的绑定机制。\
    实现扩展点: ```bind```
14. ```DefaultPreemption```: 提供默认的强占机制。\
    实现扩展点: ```postFilter```