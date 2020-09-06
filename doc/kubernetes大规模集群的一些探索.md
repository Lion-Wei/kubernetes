## kubernetes大规模集群

### 定义

- 节点：5000
- pod：150000
- pod/node：100

### 瓶颈点及解决思路

#### etcd

etcd面临的问题主要是读写性能的问题，性能问题受读写的请求数以及磁盘写入的性能影响；在不专门针对etcd本身调优的情况下可以通过下面的思路解决

1. 使用独立的ssd盘存储
2. etcd拆分，一般将etcd-event拆出去，另外crd之类的也可以单独拆分，`etcd-servers-overrides`可以支持多种形式的拆分
3. etcd独立于k8s node或master节点之外部署

#### apiserver

apiserver本身的限流可能会在大规模集群时面临压力，需要调整参数，放宽限流值

- `--max-mutating-requests-inflight`
- `--max-requests-inflight`
- `--watch-cache-size`，集群资源的watch cache大小

另外apiserver的多实例部署，以及连接数的负载均衡也是问题的解决方案之一。

#### 调度

##### 调优思路

调度是挺重要的一块，集群规模增大时对调度的压力尤为明显；可能的解决思路是：

1. 多实例并行调度：每个实例管理不同节点和pod，只在当前管理的部分节点中调度，实现局部最优（分解）
2. 同类合并调度/批调度：比如一个deploy扩容多个实例的场景，因为是明显的同样的实例，应该可以复用其中一个pod的调度结果。（批调度有点区别，但是更能参考类似的实现）

原生配置的调优方法：

1. `percentageOfNodesToScore`， 需要打分的节点比例，每次调度取一定比例和顺序取一部分节点打分。（如果判断可调度的节点数已经到达预置，则停止找后面的，然后进入打分阶段）
2. 通过taints/toleration，给pod划片；通过affinity/anti-affinity作亲和性调度。

##### 其他扩展

使用自定义调度插件，对插件作删减等，调度插件支持的阶段包括

![img](https://d33wubrfki0l68.cloudfront.net/4e9fa4651df31b7810c851b142c793776509e046/61a36/images/docs/scheduling-framework-extensions.png)



参考：[调度框架扩展点](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/scheduling-framework/#%E6%89%A9%E5%B1%95%E7%82%B9)

#### kube-proxy

主要是因为大规模集群场景所有节点都会同步整个集群的service规则，一方面svc数量巨大，一方面规则刷新也很成问题。

1. 使用ipvs模式
2. 使用endpointslice，替换endpoint
3. svc按照ns隔离

#### 其他

##### kubelet

- 节点上报心跳策略

##### controller

- 多实例部署













## 调度框架探究

并行阶段：节点打分阶段，由于是有挺多节点的，可以并行打分

