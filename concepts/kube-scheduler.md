## kube-scheduler
> kube-scheduler是k8s集群默认的调度器。这意味着，如果你愿意，可以自己实现一个调度器

kube-scheduler负责将每一个新建的且没有绑定节点的pod找到最合适的节点进行调度。
一般来说，kube-scheduler实现一次调度分两步：
1. Filtering：从集群中找出满足pod要求的所有node节点（***fesible nodes***）
2. Scoring