# <center>Kubernetes Scheduler
## 简介
调度器会监测那些最近创建的并且还没有分配节点的Pod。对于调度器发现的每一个Pod来说，调度器都有责任为该Pod找到一个最合适的节点来运行该Pod。

## kube-scheduler
**kube-scheduler**是Kubernetes中默认的调度器，并且作为控制面的一部分来运行。kube-scheduler是被设计为可替换的，如果你需要，你可以自己实现调度组件来替换它。

Kube-scheduler会选择最佳的节点来运行新创建的和未被调度的Pod。由于Pod中的容器和Pod自己都可以有不同的要求，所以调度器会过滤掉那些不满足Pod要求的节点。另外也可以在创建Pod时指定它所在的节点，不过通常情况下并不会这么做，只有在特殊需求时会这样做。

在集群中，那些满足了Pod调度要求的节点被称为”可调度的节点(feasible nodes)“。如果没有一个节点合适的话，那么该Pod会一直处于待调度的状态，直到调度器能够为它找到一个合适的节点。

调度器为Pod寻找可调度的节点，然后再运行一系列的函数来给这些可调度的节点打分(score)，最后从所有可调度的节点中选择一个分数最高的节点来运行该Pod。接着，调度器会通知API Server这个选择的结果，该过程也就是所谓的“绑定(binding)”

### kube-scheduler的节点选择
kube-scheduler为Pod选择一个节点可以分为以下两个操作：

* Filtering
* Scoring

Filtering步骤寻找了一系列可以调度Pod的结点。例如，`PodFitsResources`会检查节点上是否有足够可用的资源来满足Pod的**requests**的资源量。在该步骤后，节点列表就会包含所有合适的节点，通常合适的节点会有多个。如果节点列表为空，那么该Pod目前是不可调度的。

在Scoring步骤中，调度器会给过滤出来的节点进行排名来选择最合适的节点。调度器会根据打分规则给每个过滤出来的节点打分。

最后，kube-scheduler把Pod分配给分数最高的这个节点。如果有多个节点的分数都是最高分，那么kube-scheduler会从这几个节点中随机选择一个。
