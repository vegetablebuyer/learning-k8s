## 基础架构
etcd的基础架构图如下
![alt text](../pictures/etcd-architecture.png)

按照分层模型，etcd分为client层、api网络层、raft算法层、逻辑层、存储层。
- client层：包括client v2和v3两个大版本的API客户端库，支持负载均衡，节点间故障自动转移。
- API网络层：包括client访问server和server节点之间的通信协议。
- raft算法层：这一层实现了leader选举、日志复制、readindex等核心算法特性，用于保障etcd多个节点之间的数据一致性、提升服务可用性等。
- 功能逻辑层：etcd核心特性实现层，KVServer模块，MVCC模块，Auth鉴权模块，Lease租约模块，Compactor压缩模块等，其中MVCC模块由treeIndex模块和boltdb模块组成。
- 存储层：包括WAL日志模块，Snapshot快照模块，boltdb模块。其中WAL保障etcd crash后数据不丢失，boltdb保障集群元数据和用户写入的数据。

> client访问etcd server分为v2和v3两个版本。
>
> v2 API使用HTTP/1.x协议，v3 API使用基于HTTP/2的gRPC协议。
>
> 另一方面，server之间通信协议，是指节点间通过raft算法实现数据复制和leader选举等功能时使用的HTTP协议。
>
> HTTP/2 是基于二进制而不是文本、支持多路复用而不再有序且阻塞、支持数据压缩以减少包大小、支持 server push 等特性。
>
> 因此，基于 HTTP/2 的 gRPC 协议具有低延迟、高性能的特点，有效解决了我们在上一讲中提到的 etcd v2 中 HTTP/1.x 性能问题。


## MVCC
多版本并发控制(Multiversion concurrency control)模块是为了解决上一讲我们提到etcd v2不支持保存key的历史版本、不支持多key事务等问题而产生的。它核心由内存树形索引模块（treeIndex）和嵌入式的kv持久化存储库boltdb组成。
> boltdb，它是个基于 B+ tree 实现的 key-value 键值库，支持事务，提供 Get/Put 等简易 API 给 etcd 操作。

boltdb的key是全局递增的版本号 (revision)，value是用户key、value等字段组合成的结构体，然后通过treeIndex模块来保存用户key和版本号的映射关系。

treeIndex与boltdb关系如下面的读事务流程图所示，从treeIndex中获取key hello的版本号，再以版本号作为boltdb的key，从boltdb中获取其value信息。
![alt text](../pictures/etcd-treeindex.png)
在获取到版本号信息后，就可从boltdb模块中获取用户的key-value数据了。不过有一点你要注意，并不是所有请求都一定要从boltdb获取数据。etcd出于数据一致性、性能等考虑，在访问boltdb前，首先会从一个内存读事务buffer中，二分查找你要访问key是否在buffer里面，若命中则直接返回。

MVCC机制是基于多版本技术实现的一种乐观锁机制，它乐观地认为数据不会发生冲突，但是当事务提交时，具备检测数据是否冲突的能力。

更新一个 key-value 数据的时候，它并不会直接覆盖原数据，而是新增一个版本来存储新的数据，每个数据都有一个版本号。即使是删除数据的时候，它实际也是新增一条带删除标识的数据记录。<u>当你指定版本号读取数据时，它实际上访问的是版本号生成那个时间点的快照数据。</u>
### treeIndex的数据结构
在treeIndex中，每个节点的key是一个keyIndex结构，etcd就是通过它保存了用户的key与版本号的映射关系。
```golang
type keyIndex struct {
    key []byte // 用户的key名称
    modified revision // 最后一次修改key时的etcd版本号
    generations []generation // generation保存了一个key若干代版本号信息
}
```
generations表示一个key从创建到删除的过程，每代对应key的一个生命周期的开始与结束。当你第一次创建一个 key 时，会生成第0代，后续的修改操作都是在往第0代中追加修改版本号。当你把key删除重新创建之后就会生成第1代，以此类推。
```golang
type generation struct { 
    ver int64 // 表示此key的修改次数 
    created revision // 表示generation结构创建时的版本号 
    revs []revision // 每次修改key时的revision追加到此数组
}
```
revision版本号并不是一个简单的整数，而是一个结构体。
```golang
type revision struct {
    main int64 // 一个全局递增的主版本号，随put/txn/delete事务递增，一个事务内的key main版本号是一致的 
    sub int64 // 一个事务内的子版本号，从0开始随事务内put/delete操作递增
}
```
### boltdb的数据结构
boltdb是一个key-value的存储系统，key为revision结构体，value也是一个结构体，它是由用户key、value、create_revision、mod_revision、version、lease 组成
- create_revision表示此key创建时的版本号，也就是treeIndex中keyIndex.generations[i].created 字段
- mod_revision表示key最后一次修改时的版本号，即put操作发生时的全局版本号加1
- version表示此key的修改次数，也就是treeIndex中keyIndex.generations[i].ver 字段
### MVCC查询key的流程
1. treeIndex模块从B-tree中，根据key查找到keyIndex对象
2. keyIndex对象匹配有效的generation
3. 如果不带版本号读则读取最新的数据，返回generation中最后一个revision
4. 如果带了版本号查询（假如为N），则在generation中遍历所有revisions，返回小于等于N的最大revision（key在N的revision时候不一定有数据，所以是返回小于等于N中的最大值）
5. 根据返回的revision去boltdb中查询并返回数据
### MVCC删除key的流程
1. 与更新key不一样的是，生成的boltdb key版本号{N,0,t}追加了删除标识（tombstone, 简写t），boltdb value变成只含用户key的KeyValue结构体
2. treeIndex给key的keyIndex追加一个空的generation对象，表示此索引对应的key被删除了
3. 再次查询key的时候发现keyIndex存在空的generation对象，并且查询的版本大于等于被删除时候的版本号，则返回空
> tombstone删除标注有两个作用
> 1. 生成delete的event事件
> 2. 重启etcd，遍历boltdb中的key构建treeIndex内存树时，需要知道哪些key是被删除了的
> 3. 真正删除treeIndex中的索引对象、boltdb中的key是通过压缩 (compactor) 组件异步完成。
>
> 正因为 etcd的删除key操作是基于以上延期删除原理实现的，因此只要压缩组件未回收历史版本，我们就能从etcd中找回误删的数据。