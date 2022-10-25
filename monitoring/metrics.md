## Prometheus指标类型

- ```Counter```: 单调递增的计数器，重启的时候重置为0
- ```Gauge```: 一个可增可减的数字变量，初始值为0
- ```Histogram```: 会对观测数据取样，然后将观测数据放入有数值上界的桶中，并记录各桶中数据的个数，所有数据的个数和数据数值总和
- ```Summary```: 与```Histogram```类似，除此之外还会取一个滑动窗口，计算窗口内样本数据的分位数

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
- ```apiserver_request_total```: apiserver的请求总数，是一个```Counter```指标，有下面label \
    ```code```，```component```，```contentType```，```dry_run```，```group```，```resource```，```scope```，```subresource```，```verb```，```version```
- ```apiserver_request_duration_seconds_bucket```: apiserver的请求延迟，是一个```Histogram```指标，有下面label \
    ```le```，```component```，```contentType```，```dry_run```，```group```，```resource```，```scope```，```subresource```，```verb```，```version```
- ```apiserver_current_inflight_requests```: apiserver当前正在并发处理的请求，是一个```Gauge```指标，有下面label \
    ```requestKind```，取值为```readOnly(只读请求)```、```mutating(写请求)```
- ```apiserver_current_inqueue_requests```: apiserver当前正在排队的请求，是一个```Gauge```指标，需要开启APF才有
### 监控的指标
1. 
```promql
sum(rate(apiserver_request_total[15s])) by (instance) > 0
```