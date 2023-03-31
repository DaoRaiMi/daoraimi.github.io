# <center>Resource Management

当为Pod中的容器配置了**request**资源限制，**kube-scheduler**就会根据**request**来决定将这个Pod调度到哪个节点上。当为容器配置了资源限制**limit**时，**kubelet**会保证运行中的容器使用的资源不会超过你设定的限制。当然**kubelet**也会为容器至少预留**request**配置的量的相关资源。

## Requests and limits
如果在Pod运行的节点上有足够的资源可用，那么Pod中的容器可能并且也是被允许使用资源超过**request**指定的量的。然而，容器使用量是绝对不能超过**limit**指定的量的。当某个容器申请使用的量超过**limit**的配置时，该容器就会被OOM掉。

注意: 如果只为容器配置了**limit**限制，那么Kubernetes会把**limit**的值复制给**request**.

## 配置
```yaml
spec:
  containers:
  - name: nginx
    resources:
      requests:
        cpu: 50m
        memory: 100M
      limits:
        cpu: 100m
        memory: 500M
```
### CPU单位
在Kubernetes中1 CPU就表示一个物理的核,或者一个虚拟的核，这取决于这个节点是一个物理节点还是一个虚拟机。

1 CPU可以划分成1000m,这表示“一千微核”， Kubernetes不允许使用比微核更高的精确度。

### Memory单位
memory单位是以byte计算的。可以使用的单位：`E,P,T,G,M,K,`或者`Ei,Pi,Ti,Gi,Mi,Ki`，也可以不带单位，直接写数值，这就是使用了默认的单位`bytes`.

注意：如果配置的是`400m`，这表示的是`0.4 bytes`。这里的`m`和CPU中的具有相同的作用。

## 配置了requests资源限制的Pod是如何被调度的
当你创建一个Pod时，Kubernetes 调度器会选择一个节点来运行你的Pod。每个节点针对某种资源都有一个上限，也就是能够提供给Pod的CPU和Memory的上限。

Kubernetes的调度器会确保已经调度到单个节点上的Pod的**requests**的总量，不会超过该节点对应资源的上限。

注意：在capacity检查失败后，即使当前节点上CPU或memory的使用量很低，调度器仍然拒绝调度Pod到当前节点上。这么做的原因就是为了避免流量高峰来临时产生的资源紧缺。