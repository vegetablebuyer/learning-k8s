## Prometheus

### 谷歌SRE监控的4个黄金指标
- 延迟：服务请求所需时间
- 通讯量：监控当前系统的流量，用于衡量服务的容量需求
- 错误：监控当前系统所有发生的错误请求，衡量当前系统错误发生的速率
- 饱和度：衡量当前服务的饱和度

### 指标类型

- ```Counter```: 单调递增的计数器，重启的时候重置为0
- ```Gauge```: 一个可增可减的数字变量，初始值为0
- ```Histogram```: 会对观测数据取样，然后将观测数据放入有数值上界的桶中，并记录各桶中数据的个数，所有数据的个数和数据数值总和
- ```Summary```: 与```Histogram```类似，除此之外还会取一个滑动窗口，计算窗口内样本数据的分位数

### 函数
- ```rate(promql[tw])```: 计算在tw的采样周期内每秒的增长率，promql需要是一个```Counter```类型
- ```irate(promql[tw])```: 计算在tw的采样周期内瞬时增长率，只使用采样周期内的最后两个样本，promql需要是一个```Counter```类型
- ```increase(promql[tw])```: 计算在tw的采样周期内的增长量，```increase(promql[15s]) / 15```等同于```rate(promql[15s])```

## Apiserver
Apiserver的指标有下面的label
- ```client```: 请求的客户端，1.16版本可用，1.20版本之后已不可用
- ```code```: 请求的返回码
- ```component```: 固定值，默认为```apiserver```
- ```dry_run```: 是否为空运行的的请求，空字符串代表不是空运行
- ```verb```: 请求的动作，有```DELETE、GET、LIST、PATCH、POST、WATCH、PUT```
- ```group```: 请求的资源所属的group
- ```version```: 请求的资源所属的version
- ```resource```: 请求的资源名称
- ```subresource```: 请求的子资源
- ```scope```: 请求的范围，有```cluster```、```namespace```、```scope```跟空字符串4种可能。
- ```le```: ```Histogram```类型专有，是less than的缩写，表示当前指标的累积量的桶的数值上限
- ```requestKind```: 请求的类型，有```readOnly```、```mutating```
    
```scope```的取值根据下面的逻辑
```golang
// CleanScope returns the scope of the request.
func CleanScope(requestInfo *request.RequestInfo) string {
    if requestInfo.Namespace != "" {
        // eg: kubectl get pod -n kube-system
        return "namespace"
    }
    if requestInfo.Name != "" {
        // eg: kubectl get clusterrole system:node
        return "resource"
    }
    if requestInfo.IsResourceRequest {
        // eg: kubectl get clusterrole
        // eg: kubectl get pod -A
        return "cluster"
    }
    // this is the empty scope
    return ""
}
```
| 指标                                            | 指标类型  | 指标说明                                                                                                                                                                                                                                                                                                                                                                                       |
|-------------------------------------------------|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| apiserver_request_duration_seconds_bucket       | Histogram | 该指标用于统计APIServer客户端对APIServer的访问时延。对APIServer不同请求的时延分布。请求的维度包括Verb、Group、Version、Resource、Subresource、Scope、Component和Client。 Histogram Bucket的阈值为： {0.05, 0.1, 0.15, 0.2, 0.25, 0.3, 0.35, 0.4, 0.45, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.25, 1.5, 1.75, 2.0, 2.5, 3.0, 3.5, 4.0, 4.5, 5, 6, 7, 8, 9, 10, 15, 20, 25, 30, 40, 50, 60}，单位：秒。                 |
| apiserver_request_total                         | Counter   | 对APIServer不同请求的计数。请求的维度包括Verb、Group、Version、Resource、Scope、Component、HTTP contentType、HTTP code和Client                                                                                                                                                                                                                                                                      |
| apiserver_request_no_resourceversion_list_total | Counter   | 对APIServer的请求参数中未配置ResourceVersion的LIST请求的计数。请求的维度包括Group、Version、Resource、Scope和Client。用来评估quorum read类型LIST请求的情况，用于发现是否存在过多quorum read类型LIST以及相应的客户端，以便优化客户端请求行为。                                                                                                                                                  |
| apiserver_current_inflight_requests             | Gauge     | APIServer当前处理的请求数。包括ReadOnly和Mutating两种。                                                                                                                                                                                                                                                                                                                                        |
| apiserver_dropped_requests_total                | Counter   | 限流丢弃掉的请求数。HTTP返回值是429 'Try again later'。                                                                                                                                                                                                                                                                                                                                        |
### 监控的关键指标
| promql                                                                                                                                                                           | 说明                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| sum(irate(apiserver_request_total[15s]))                                                                                                                                   | APIServer总QPS。                |
| sum(irate(apiserver_request_total{code=\~"20.*",verb=\~"GET\|LIST"}[15s]))/sum(irate(apiserver_request_total{verb=~"GET\|LIST"}[15s]))                                 | APIServer读请求成功率。         |
| sum(irate(apiserver_request_total{code=\~"20.*",verb!\~"GET\|LIST\|WATCH\|CONNECT"}[15s]))/sum(irate(apiserver_request_total{verb!~"GET\|LIST\|WATCH\|CONNECT"}[15s])) | APIServer写请求成功率。         |
| sum(apiserver_current_inflight_requests{requestKind="readOnly"})                                                                                                                 | APIServer当前在处理读请求数量。 |
| sum(apiserver_current_inflight_requests{requestKind="mutating"})                                                                                                                 | APIServer当前在处理写请求数量。 |
| sum(irate(apiserver_dropped_requests_total[15s]))                                                                                                                          | Dropped Request Rate。          |