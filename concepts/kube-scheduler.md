## kube-scheduler
> kube-scheduler是k8s集群默认的调度器。这意味着，如果你愿意，可以自己实现一个调度器

kube-scheduler负责将每一个新建的且没有绑定节点的pod找到最合适的节点进行调度。\
宏观来说，kube-scheduler实现一次调度分两步：
1. Filtering: 从集群中找出满足pod要求的所有node节点，这些节点被称为***fesible nodes***
2. Scoring: 根据打分规则对每一个***fesible node***进行打分，选出分数最高的一个，将pod调度到该节点

## scheduler配置

### profiles
通过```kube-scheduder --config <filename>```来指定profile文件。一个简单的profile文件如下：
```vim
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