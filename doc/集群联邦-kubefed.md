## 集群联邦

主要优势

- 跨集群分散工作负载，提升可靠性
- 扩集群服务发现，服务地理位置感知，降低延迟（这一条还不知道是怎么实现的）
- 多云，混合云部署
- 消减单集群规模过大带来的影响



### kubefed

##### v1版本

1.6进入beta阶段，之后一直没发展，到1.11被废弃，原因：

1. 控制面问题会影响租户数据面
2. 无法兼容多版本API，不支持crd，可扩展性差
3. 无法有效的多集群权限管理rbac（v2是怎么实现多集群权限管理的，这个也不清楚）

##### v2版本

1. API全部通过crd支持，模块化
2. 跨集群服务发现
3. 跨集群应用编排部署

##### v2版本使用

- 在host cluster中安装kubefed
- 将其他集群加入联邦中，其他集群作为`member cluster`，`host cluster`也是一个
- **需要联邦的api配置**：如果一个API需要联邦管理，那么需要针对它进行配置。默认配置了`ClusterRole、Configmap、Deployment、Ingress、Job、Namespace、ReplicaSet、Secrets、ServiceAccount、Service`
  - 这里配置了之后，针对不同的资源具体联邦策略怎样，这个不确定
  - 还可以针对自定义的crd资源作联邦，需要enable crd联邦
- **DNS的配置**，使用MCIDNS（mutil cluster ingress dns）
  - 这里不是太确定外部DNS到底是什么策略

##### 执行联邦及相关配置

执行联邦指将我们需要的资源按照一定的策略分发到不同的集群里面。比如联邦ns和configmap

```
kubefedctl federate namespace my-namespace --contents --skip-api-resources "configmaps,apps"

kubefedctl federate configmaps my-configmap -n my-namespace
```

可以通过联邦的资源对应的类型，如ns对应`FederatedNamespace`，查看联邦的状态



要执行联邦就需要有一定的联邦策略，有如下字段：

- `placement`：在集群之间的分发策略
- `overrides`：分发到某个集群时需要做的差异化修改

##### MCIDNS/MCSDNS工作流程

![ingressdns-with-externaldns](https://gmem.site/wp-content/uploads/2020/05/ingressdns-with-externaldns.png)

1. 用户创建FederatedDeployment、FederatedService、FederatedIngress资源
2. Kubefed的Sync Controller将对应的目标资源传播到成员K8S集群中
3. 用户创建一个IngressDNSRecord资源，此资源标识期望的DNS名，以及可选的DNS记录参数
4. MCIDNS的Ingress DNS Controller监控IngressDNSRecord，用匹配（成员集群中命名空间、名字与IngressDNSRecord一致的）的Ingress资源的IP地址（这个地址由Ingress Controller填充）来更新IngressDNSRecord的Status
5. MCIDNS的DNS Endpoint Controller监控IngressDNSRecord，创建对应的DNSEndpoint
6. DNSEndpoint包含必要的信息，外部DNS系统（例如ExternalDNS）负责监控此对象，利用其中的信息，在DNS Provider中创建DNS记录

##### 调度RSP

通过RSP`ReplicaSchedulingPreference`机制来实现集群中调度的偏好

1. 用户创建rsp，提供偏好信息
2. rsp controller监控rsp，根据ns/name找到对应的`FederatedDeployment`等资源，根据配置分发
3. 如果存在rsp，则`FederatedDeployment`中的副本数配置不会生效。
4. 如果配置了`spec.rebalance`，则会执行再平衡的动作，感知不可调度并重新调度到其他集群。



这里的调度策略，根据理解还是不能根据集群的实际资源量作自动化调度，个人理解这是比较有用的一块，不知道为啥没这么做。



### 集群实测

